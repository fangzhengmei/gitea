# Gitea 用户活动与通知派发同步呈现逻辑分析

## 概述

Gitea 系统中存在两条并行但相关的数据链路：**用户活动（Activity/Action）** 和 **系统通知（Notification）**。两者共享相同的事件源，但通过不同的 Notifier 实现进行独立处理，最终在不同的展现入口呈现给用户。

通知系统采用**后端 EventSource 定时推送 + 前端 SharedWorker 多标签共享**的架构实现实时同步，同时辅以轮询作为降级方案。

---

## 一、整体架构

### 1.1 双轨数据链路

```
业务操作触发事件
      │
      ▼
services/notify/notify.go ── 事件分发中心
      │
      ├───────────────────────────────┐
      │                               │
      ▼                               ▼
actionNotifier                    notificationService
(activities/feed)              (services/uinotification)
      │  同步写入                     │  异步队列
      ▼                               ▼
  Action 表                    Notification 表
(用户活动时间线)             (系统通知中心)
      │                               │
      ▼                               ▼
  展现入口                          展现入口
- 仓库活动页 /activity           - 通知中心 /notifications
- 用户首页 Dashboard            - 顶部导航未读红点
                              - API /notifications/new
```

### 1.2 实时推送架构

```
后端 EventSource Manager
      │
      ├─ 定时轮询 DB 获取通知计数变化
      │   (GetUIDsAndNotificationCounts)
      └─ 通过 SSE 推送到各用户连接
                │
                ▼
        前端 SharedWorker
          (跨标签共享连接)
                │
                ├─ 标签页 1: 更新未读红点 + 刷新列表
                ├─ 标签页 2: 更新未读红点 + 刷新列表
                └─ ...
```

---

## 二、第一阶段：事件采集触发

### 2.1 初始化与注册顺序

**初始化顺序**（`routers/init.go:130-131, 158`）：
```go
mustInit(feed_service.Init)          // 1. 活动 Notifier 先初始化
mustInit(uinotification.Init)       // 2. 通知 Notifier 后初始化
// ...
eventsource.GetManager().Init()     // 3. 最后启动 EventSource
```

**Notifier 注册**（`services/notify/notify.go:22-26`）：
```go
var notifiers []Notifier

func RegisterNotifier(notifier Notifier) {
    go notifier.Run()                          // 异步启动 Notifier 的 Run 方法
    notifiers = append(notifiers, notifier)    // 按注册顺序加入切片
}
```

> **关键观察**：Notifier 按注册顺序被调用，`actionNotifier` 先于 `notificationService` 执行。但由于通知使用异步队列，两者的 DB 写入顺序**不保证**。

### 2.2 事件源定义

事件类型在 `models/activities/action.go:37-65` 中定义，共 27 种动作类型：

```go
const (
    ActionCreateRepo                ActionType = iota + 1  // 1
    ActionRenameRepo                                      // 2
    ActionStarRepo                                        // 3
    ActionWatchRepo                                       // 4
    ActionCommitRepo                                      // 5
    ActionCreateIssue                                     // 6
    ActionCreatePullRequest                               // 7
    // ... 共 27 种
)
```

### 2.3 事件触发入口

所有业务操作通过 `services/notify/notify.go` 中的统一函数触发事件。该文件定义了 40+ 个事件触发函数，例如：

- `NewIssue()` - 新建 Issue
- `CreateIssueComment()` - 创建评论
- `NewPullRequest()` - 新建 PR
- `MergePullRequest()` - 合并 PR
- `PushCommits()` - 推送提交
- `IssueChangeStatus()` - Issue 状态变更

**核心分发逻辑**（`services/notify/notify.go:73-78`）：
```go
func NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User) {
    for _, notifier := range notifiers {  // 按注册顺序遍历
        notifier.NewIssue(ctx, issue, mentions)
    }
}
```

### 2.4 Notifier 接口

事件分发基于 `Notifier` 接口（`services/notify/notifier.go:20-85`），支持多实现：

```go
type Notifier interface {
    Run()
    NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User)
    CreateIssueComment(ctx context.Context, doer *user_model.User, repo *repo_model.Repository,
        issue *issues_model.Issue, comment *issues_model.Comment, mentions []*user_model.User)
    // ... 40+ 个方法
}
```

---

## 三、第二阶段：订阅匹配筛选

系统有两个核心 Notifier 实现，分别处理**活动记录**和**系统通知**。

### 3.1 活动（Action）订阅匹配 - `actionNotifier`

**位置**：`services/feed/notifier.go`

**职责**：为每个符合条件的用户创建活动时间线记录。

**处理方式**：**同步**执行，在请求 goroutine 中直接写入 DB。

**匹配流程**（`services/feed/feed.go:96-161`）：

1. **获取关注者列表**：
   ```go
   watchers, err := repo_model.GetWatchers(ctx, repoID)
   ```

2. **预计算权限**（批量处理，性能优化）：
   ```go
   permCode := make([]bool, len(watchers))   // 代码可读权限
   permIssue := make([]bool, len(watchers))  // Issue 可读权限
   permPR := make([]bool, len(watchers))     // PR 可读权限
   
   for i, watcher := range watchers {
       perm, _ := access_model.GetIndividualUserRepoPermission(ctx, repo, user)
       permCode[i] = perm.CanRead(unit.TypeCode)
       permIssue[i] = perm.CanRead(unit.TypeIssues)
       permPR[i] = perm.CanRead(unit.TypePullRequests)
   }
   ```

3. **按动作类型过滤**：
   ```go
   switch act.OpType {
   case ActionCommitRepo, ActionPushTag, ActionDeleteTag:
       if !permCode[i] { continue }  // 需要代码权限
   case ActionCreateIssue, ActionCommentIssue:
       if !permIssue[i] { continue } // 需要 Issue 权限
   case ActionCreatePullRequest, ActionCommentPull:
       if !permPR[i] { continue }    // 需要 PR 权限
   }
   ```

4. **排除操作者自身**：
   ```go
   if act.ActUserID == watcher.UserID {
       continue  // 不通知自己
   }
   ```

5. **插入重复记录**（设计特性）：
   - 为操作者本人插入一条（UserID = ActUserID）
   - 为仓库所属组织插入一条（如果是组织仓库）
   - 为每个关注者插入一条

### 3.2 通知（Notification）订阅匹配 - `notificationService`

**位置**：`services/uinotification/notify.go`

**职责**：为用户创建系统通知（带未读状态）。

#### 3.2.1 异步队列架构

**队列创建**（`services/uinotification/notify.go:44-51`）：
```go
func NewNotifier() notify_service.Notifier {
    ns := &notificationService{}
    ns.issueQueue = queue.CreateSimpleQueue(
        graceful.GetManager().ShutdownContext(),
        "notification-service",
        handler,
    )
    return ns
}
```

**入队操作**（`services/uinotification/notify.go:76`）：
```go
_ = ns.issueQueue.Push(issueNotificationOpts{
    IssueID:              issue.ID,
    NotificationAuthorID: doer.ID,
    CommentID:            comment.ID,
})
```

#### 3.2.2 WorkerPoolQueue 工作机制

**核心结构**（`modules/queue/workerqueue.go:22-45`）：
```go
type WorkerPoolQueue[T any] struct {
    batchChan     chan []T        // 批处理通道
    workerNum     int             // 当前 worker 数量
    workerMaxNum  int             // 最大 worker 数
    workerActiveNum int           // 活跃 worker 数
    baseQueue     baseQueue       // 底层队列（channel/leveldb/redis）
    // ...
}
```

**消息分发流程**（`modules/queue/workergroup.go:48-88`）：

1. **批量收集**（100ms 防抖）：
   ```go
   batchDebounceDuration = 100 * time.Millisecond
   ```

2. **动态扩缩容**：
   ```go
   if full || noWorker {
       if q.workerNum < q.workerMaxNum || noWorker && q.workerMaxNum <= 0 {
           q.workerNum++
           q.doStartNewWorker(wg)  // 启动新 worker
       }
   }
   ```

3. **批量处理**：
   ```go
   func handler(items ...issueNotificationOpts) []issueNotificationOpts {
       for _, opts := range items {
           CreateOrUpdateIssueNotifications(ctx, opts.IssueID, ...)
       }
       return nil
   }
   ```

#### 3.2.3 订阅匹配逻辑

**匹配流程**（`models/activities/notification_list.go:72-162`）：

1. **收集候选接收者**：
   ```go
   if receiverID > 0 {
       // 指定接收者（如 @提及、被指派）
       toNotify.Add(receiverID)
   } else {
       // 1. Issue 关注者（显式关注）
       issueWatches, _ := issues_model.GetIssueWatchersIDs(ctx, issueID, true)
       toNotify.AddMultiple(issueWatches...)
       
       // 2. 仓库关注者（排除 WIP PR）
       if !(issue.IsPull && issues_model.HasWorkInProgressPrefix(issue.Title)) {
           repoWatches, _ := repo_model.GetRepoWatchersIDs(ctx, issue.RepoID)
           toNotify.AddMultiple(repoWatches...)
       }
       
       // 3. Issue 参与者（发帖人、评论人等）
       issueParticipants, _ := issue.GetParticipantIDsByIssue(ctx)
       toNotify.AddMultiple(issueParticipants...)
   }
   ```

2. **排除过滤**：
   ```go
   delete(toNotify, notificationAuthorID)  // 不通知触发者
   
   // 移除显式取消关注 Issue 的用户
   issueUnWatches, _ := issues_model.GetIssueWatchersIDs(ctx, issueID, false)
   for _, id := range issueUnWatches {
       toNotify.Remove(id)
   }
   ```

3. **权限二次校验**：
   ```go
   if issue.IsPull && !access_model.CheckRepoUnitUser(ctx, issue.Repo, user, unit.TypePullRequests) {
       continue  // PR 需要 PR 权限
   }
   if !issue.IsPull && !access_model.CheckRepoUnitUser(ctx, issue.Repo, user, unit.TypeIssues) {
       continue  // Issue 需要 Issue 权限
   }
   ```

4. **创建或更新通知**：
   ```go
   if notificationExists(notifications, issue.ID, userID) {
       // 已存在：更新状态为未读，更新时间戳
       updateIssueNotification(ctx, userID, issue.ID, commentID, notificationAuthorID)
   } else {
       // 不存在：创建新通知
       createIssueNotification(ctx, userID, issue, commentID, notificationAuthorID)
   }
   ```

**通知更新策略**（`models/activities/notification.go:170-190`）：
```go
// 如果之前通知已读，更新为未读并记录新评论
if notification.Status == NotificationStatusRead {
    notification.Status = NotificationStatusUnread
    notification.CommentID = commentID
}
// 无论如何更新 updated_by 用于排序
notification.UpdatedBy = updatedByID
```

---

## 四、第三阶段：事件流推送与实时同步

### 4.1 后端 EventSource Manager

**核心结构**（`modules/eventsource/manager.go:11-16`）：
```go
type Manager struct {
    mutex      sync.Mutex
    messengers map[int64]*Messenger  // 每个用户一个 Messenger
    connection chan struct{}        // 新连接信号
}
```

**每个用户的 Messenger**（`modules/eventsource/messenger.go:9-13`）：
```go
type Messenger struct {
    mutex    sync.Mutex
    uid      int64
    channels []chan *Event  // 多标签页共享，每个连接一个 channel
}
```

**定时推送流程**（`modules/eventsource/manager_run.go:31-120`）：

1. **定时轮询**（间隔由配置决定）：
   ```go
   timer := time.NewTicker(setting.UI.Notification.EventSourceUpdateTime)
   ```

2. **获取变化用户的通知计数**：
   ```go
   // 查询 [then, now) 时间窗口内有更新的用户及其未读数
   uidCounts, err := activities_model.GetUIDsAndNotificationCounts(ctx, then, now)
   ```

3. **向在线用户推送**：
   ```go
   for _, uidCount := range uidCounts {
       m.SendMessage(uidCount.UserID, &Event{
           Name: "notification-count",
           Data: uidCount,
       })
   }
   ```

4. **消息分发到各标签页**（`modules/eventsource/messenger.go:58-68`）：
   ```go
   func (m *Messenger) SendMessage(message *Event) {
       m.mutex.Lock()
       defer m.mutex.Unlock()
       for i := range m.channels {
           select {
           case m.channels[i] <- message:  // 非阻塞发送
           default:  // 通道满则丢弃
           }
       }
   }
   ```

### 4.2 HTTP 长连接端点

**路由**（`routers/web/web.go:592`）：
```go
m.Any("/user/events", routing.MarkLongPolling(), events.Events)
```

**SSE 握手**（`routers/web/events/events.go:17-65`）：
```go
func Events(ctx *context.Context) {
    // SSE 响应头
    ctx.Resp.Header().Set("Content-Type", "text/event-stream")
    ctx.Resp.Header().Set("Cache-Control", "no-cache")
    ctx.Resp.Header().Set("Connection", "keep-alive")
    ctx.Resp.WriteHeader(http.StatusOK)
    
    // 注册到 EventSource Manager
    messageChan := eventsource.GetManager().Register(uid)
    
    // 心跳（30秒）
    timer := time.NewTicker(30 * time.Second)
    
    // 事件循环
    for {
        select {
        case <-timer.C:  // 发送 ping 心跳
        case event, ok := <-messageChan:  // 接收推送事件
            event.WriteTo(ctx.Resp)
            ctx.Resp.Flush()
        }
    }
}
```

### 4.3 前端 SharedWorker 多标签共享

**Worker 初始化**（`web_src/js/modules/worker.ts:6-29`）：
```typescript
export class UserEventsSharedWorker {
    sharedWorker: SharedWorker;
    
    constructor(options?: string | WorkerOptions) {
        const worker = new SharedWorker(sharedWorkerUri, options);
        worker.port.postMessage({
            type: 'start',
            url: `${window.location.origin}${appSubUrl}/user/events`,
        });
    }
}
```

**Worker 内部管理**（`web_src/js/eventsource.sharedworker.ts:1-145`）：

1. **Source 复用**：
   ```typescript
   const sourcesByUrl = new Map<string, Source | null>();
   const sourcesByPort = new Map<MessagePort, Source | null>();
   ```

2. **Source 类**（每个 URL 一个 EventSource 连接）：
   ```typescript
   class Source {
     url: string;
     eventSource: EventSource | null;
     clients: Array<MessagePort>;  // 多个标签页端口
     
     notifyClients(event: {type: string, data: any}) {
       for (const client of this.clients) {
         client.postMessage(event);  // 广播给所有标签页
       }
     }
   }
   ```

3. **事件监听**：
   ```typescript
   source.listen('notification-count');
   source.listen('logout');
   source.listen('stopwatches');
   ```

### 4.4 降级方案：轮询机制

当浏览器不支持 SharedWorker 或 EventSource 时，使用渐进式轮询：

**轮询逻辑**（`web_src/js/features/notification.ts:21-49`）：
```typescript
function initNotificationCount() {
    // 优先尝试 SharedWorker + EventSource
    if (notificationSettings.EventSourceUpdateTime > 0 && 
        window.EventSource && window.SharedWorker) {
        const worker = new UserEventsSharedWorker('notification-worker');
        worker.addMessageEventListener((event) => {
            if (event.data.type === 'notification-count') {
                receiveUpdateCount(event);
            }
        });
        return;
    }
    
    // 降级：渐进式轮询
    startPeriodicPoller(notificationSettings.MinTimeout);
}
```

**渐进式退避**（`web_src/js/features/notification.ts:55-76`）：
```typescript
async function updateNotificationCountWithCallback(...) {
    if (lastCount !== newCount) {
        timeout = notificationSettings.MinTimeout;  // 有变化，加快轮询
    } else if (timeout < notificationSettings.MaxTimeout) {
        timeout += notificationSettings.TimeoutStep;  // 无变化，减慢轮询
    }
}
```

---

## 五、第四阶段：计数与列表刷新的协同

### 5.1 未读数更新流程

**推送接收**（`web_src/js/features/notification.ts:8-19`）：
```typescript
async function receiveUpdateCount(event: MessageEvent) {
    const data = JSON.parse(event.data.data);
    
    // 1. 更新顶部导航未读红点
    for (const count of document.querySelectorAll('.notification_count')) {
        count.classList.toggle('tw-hidden', data.Count === 0);
        count.textContent = `${data.Count}`;
    }
    
    // 2. 异步刷新通知列表
    await updateNotificationTable();
}
```

### 5.2 通知列表刷新

**序列防乱序**（`web_src/js/features/notification.ts:6, 78-100`）：
```typescript
let notificationSequenceNumber = 0;

async function updateNotificationTable() {
    const notificationDiv = document.querySelector('#notification_div');
    
    params.set('sequence-number', String(++notificationSequenceNumber));
    const response = await GET(`${appSubUrl}/notifications?${params.toString()}`);
    const data = await response.text();
    const el = createElementFromHTML(data);
    
    // 只接受更新的序列号，防止旧响应覆盖新内容
    if (parseInt(el.getAttribute('data-sequence-number')!) === notificationSequenceNumber) {
        notificationDiv.outerHTML = data;
    }
}
```

### 5.3 手动查询未读数

**Web 路由**（`routers/web/web.go:1749`）：
```go
m.Get("/new", user.NewAvailable)  // 完整路径: /notifications/new
```

> ⚠️ **路由检查说明**：
> - 第 630 行 `/user/settings/notifications` - 用户通知设置页面
> - 第 1743 行 `/notifications` - 通知中心（根路径）
> - 两者在不同分组下，无冲突。未读数路由 `/notifications/new` 定义正确。

**未读数查询实现**（`routers/web/user/notification.go:388-399`）：
```go
func NewAvailable(ctx *context.Context) {
    total, _ := db.Count[activities_model.Notification](ctx, activities_model.FindNotificationOptions{
        UserID: ctx.Doer.ID,
        Status: []activities_model.NotificationStatus{activities_model.NotificationStatusUnread},
    })
    ctx.JSON(http.StatusOK, structs.NotificationCount{New: total})
}
```

### 5.4 完整协同流程

```
后端 EventSource 定时轮询 DB
      │
      ▼
发现通知更新 (updated_unix 变化)
      │
      ▼
向在线用户推送 notification-count 事件
      │
      ▼
SharedWorker 接收事件，广播给所有标签页
      │
      ├─► 标签页 1: receiveUpdateCount()
      │       ├─ 更新未读红点 (同步)
      │       └─ updateNotificationTable() (异步)
      │           ├─ 请求 /notifications?div-only=true&sequence-number=N
      │           └─ 替换 #notification_div DOM
      │
      └─► 标签页 2: (同标签页 1)
```

---

## 六、第五阶段：展现入口

### 6.1 活动（Action）展现

#### 6.1.1 仓库活动页
**路由**：`/{owner}/{repo}/activity`  
**代码**：`routers/web/repo/activity.go:22-81`

展示仓库维度的统计数据：
- 提交数、PR 数、Issue 数
- 贡献者排行
- 按时间周期筛选（日/周/月/季/年）

#### 6.1.2 用户首页 Dashboard
**代码**：`services/feed/feed.go:21-30`

```go
func GetFeedsForDashboard(ctx context.Context, opts activities_model.GetFeedsOptions) (activities_model.ActionList, int64, error) {
    return activities_model.GetFeeds(ctx, opts)
}
```

查询时进行可见性过滤（`models/activities/action.go:469-552`）：
- 检查用户活动隐私设置
- 检查仓库可读权限
- 按时间、用户、仓库等维度过滤

### 6.2 通知（Notification）展现

#### 6.2.1 Web 通知中心
**路由**：`/notifications`  
**代码**：`routers/web/user/notification.go:37-136`

**数据准备流程**：
```go
// 1. 分页查询通知
nls, _ := db.Find[activities_model.Notification](ctx, activities_model.FindNotificationOptions{
    UserID: ctx.Doer.ID,
    Status: []activities_model.NotificationStatus{queryStatus, NotificationStatusPinned},
})

// 2. 批量加载关联数据（性能优化，避免 N+1）
notifications.LoadRepos(ctx)
notifications.LoadIssues(ctx)
notifications.LoadIssuePullRequests(ctx)
notifications.LoadComments(ctx)
```

#### 6.2.2 未读数查询（AJAX 轮询/EventSource 推送）
**Web 路由**：`/notifications/new`  
**代码**：`routers/web/user/notification.go:388-399`

#### 6.2.3 API 接口
**路由**：`GET /api/v1/notifications/new`  
**代码**：`routers/api/v1/notify/notifications.go:18-36`

提供与 Web 端一致的未读数查询能力。

---

## 七、活动流 vs 通知流：一致性与延迟对比

### 7.1 处理模型对比

| 维度 | 活动流 (Action) | 通知流 (Notification) |
|------|---------------|---------------------|
| **处理方式** | 同步，请求 goroutine 内直接执行 | 异步，通过 WorkerPoolQueue 队列处理 |
| **事务保证** | 与业务操作在同一 DB 事务 | 独立事务，最终一致 |
| **写入时机** | 业务操作提交时同时写入 | 队列消费时写入（延迟不确定） |
| **失败处理** | 失败则业务操作回滚 | 失败重试，可能丢失 |

### 7.2 延迟特性对比

**活动流延迟**：
- **典型延迟**：< 10ms（同步写入）
- **影响因素**：DB 写入速度、关注者数量（批量插入）
- **可见性**：页面刷新立即可见，无推送机制

**通知流延迟**：
- **典型延迟**：100ms ~ 数秒（队列 + EventSource 轮询间隔）
- **延迟组成**：
  1. 队列防抖：100ms (`batchDebounceDuration`)
  2. 队列处理：取决于积压情况
  3. EventSource 轮询：配置项 `EventSourceUpdateTime`（通常 1-5 秒）
  4. 前端刷新：额外的 HTTP 请求
- **最坏情况**：队列严重积压时，延迟可达分钟级

### 7.3 一致性分析

**强一致性场景**：
- 活动流与业务操作强一致，操作成功则活动必然存在
- 适合作为"事实记录"的时间线

**最终一致性场景**：
- 通知流最终一致，可能出现：
  - **顺序不一致**：活动已显示，但通知红点未更新
  - **状态不一致**：通知已读/未读状态短暂不同步
  - **丢失风险**：队列处理失败可能导致通知丢失

**已知不一致窗口**：
1. **操作→活动**：同步，无窗口
2. **操作→通知**：队列处理时间（~100ms+）
3. **通知→推送**：EventSource 轮询间隔（配置项，如 2s）
4. **推送→列表刷新**：前端 HTTP 请求时间（~几十 ms）

> **设计权衡**：通知流选择最终一致性是为了不阻塞业务操作。活动流选择强一致是因为它是"历史记录"，必须可靠。

### 7.4 触发顺序分析

**代码执行顺序**（`services/notify/notify.go:29-33`）：
```go
func NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User) {
    for _, notifier := range notifiers {
        notifier.NewIssue(ctx, issue, mentions)  // 按注册顺序调用
    }
}
```

**注册顺序**（`routers/init.go:130-131`）：
```go
mustInit(feed_service.Init)          // 1. actionNotifier 先注册
mustInit(uinotification.Init)       // 2. notificationService 后注册
```

**因此调用顺序为**：
1. `actionNotifier.NewIssue()` - 同步写入 Action 表
2. `notificationService.NewIssue()` - 入队，立即返回

**DB 写入顺序**：
- Action 表：业务事务提交时写入（确定）
- Notification 表：队列消费时写入（不确定，可能在 Action 之后几十~几百 ms）

---

## 八、数据模型对比

### 8.1 Action（活动）模型
**位置**：`models/activities/action.go:135-151`

```go
type Action struct {
    ID          int64
    UserID      int64       // 接收者用户 ID
    OpType      ActionType  // 动作类型
    ActUserID   int64       // 执行者用户 ID
    RepoID      int64       // 仓库 ID
    CommentID   int64       // 评论 ID（可选）
    RefName     string      // 分支/标签名
    Content     string      // 动作内容（JSON 或 分隔字符串）
    IsPrivate   bool        // 是否私有
    CreatedUnix timeutil.TimeStamp
}
```

- **特点**：时间序、不可变、用于展示动态
- **索引**：`(user_id, created_unix)` 用于首页时间线查询

### 8.2 Notification（通知）模型
**位置**：`models/activities/notification.go:52-73`

```go
type Notification struct {
    ID        int64
    UserID    int64              // 接收者用户 ID
    RepoID    int64              // 仓库 ID
    Status    NotificationStatus // 未读/已读/置顶
    Source    NotificationSource // Issue/PR/Commit/Repository
    IssueID   int64              // Issue ID
    CommitID  string             // Commit ID
    CommentID int64              // 评论 ID
    UpdatedBy int64              // 最后更新人
    UpdatedUnix timeutil.TimeStamp
}
```

- **特点**：可变状态（未读→已读）、可置顶、按更新时间排序
- **索引**：`(user_id, status, updated_unix)` 用于通知列表查询

---

## 九、关键设计决策

### 9.1 双轨并行设计
- **Action** 面向"发生了什么"，是公开的时间线记录，要求强一致
- **Notification** 面向"需要我关注什么"，是个人的待处理提醒，可接受最终一致
- 两者独立存储、独立展现，但共享相同的事件源

### 9.2 实时推送架构
- **SharedWorker + EventSource**：多标签页共享单一连接，降低服务器压力
- **渐进式降级**：从 EventSource → 轮询，适应不同浏览器环境
- **序列号防乱序**：通知列表刷新使用递增序列号，避免旧响应覆盖新内容

### 9.3 性能优化手段
1. **批量预计算权限**：通知匹配前一次性计算所有关注者权限
2. **异步队列处理**：通知创建通过队列异步化，不阻塞业务流程
3. **批量数据加载**：展现层使用 `LoadRepos`/`LoadIssues` 批量加载关联数据，避免 N+1 查询
4. **联合索引**：数据库索引针对查询模式优化
5. **队列防抖**：100ms 批量收集，减少 DB 写入次数

### 9.4 通知去重与更新
- 同一 Issue 对同一用户只有一条通知记录
- 新活动触发时更新现有通知的时间戳和状态
- 避免通知列表被同一 Issue 的多次更新淹没

### 9.5 延迟与一致性权衡
- 活动流：同步写入，强一致，低延迟
- 通知流：异步队列，最终一致，较高延迟
- 设计目标：业务操作不被通知系统阻塞

---

## 十、完整接力流程总结

### 10.1 端到端实时同步流程

```
用户发表评论 (业务操作)
    │
    ▼
services/notify/notify.go:CreateIssueComment()
    │
    ├─► ① actionNotifier.CreateIssueComment() ── 同步
    │      │
    │      ├─ 构造 Action 对象
    │      ├─ NotifyWatchers()
    │      │    ├─ 获取仓库关注者
    │      │    ├─ 预计算权限
    │      │    ├─ 按动作类型过滤
    │      │    └─ 批量插入 Action 表 ──┐
    │      └─ 完成（活动时间线更新）      │  同步，<10ms
    │                                    │
    └─► ② notificationService.CreateIssueComment() ── 异步
           │
           ├─ 构造 issueNotificationOpts
           ├─ issueQueue.Push(opts) ── 入队立即返回
           │
           └─ 队列后台处理 (~100ms 防抖 + 批量)
                │
                └─ handler()
                     ├─ CreateOrUpdateIssueNotifications()
                     │    ├─ 收集候选接收者
                     │    ├─ 排除过滤
                     │    ├─ 权限校验
                     │    └─ 创建/更新 Notification 表 ──┐
                     └─ 完成（系统通知更新）               │  异步，100ms+
                                                            │
                                                            ▼
后端 EventSource Manager 定时轮询 (~2s)
    │
    ├─ GetUIDsAndNotificationCounts(then, now)
    └─ SendMessage(uid, notification-count 事件)
           │
           ▼
前端 SharedWorker 接收事件
    │
    ├─ 广播给所有标签页的 MessagePort
    │
    ├─► 标签页 1: receiveUpdateCount()
    │       ├─ 更新未读红点 (同步)
    │       └─ updateNotificationTable() (异步)
    │           ├─ GET /notifications?div-only=true&sequence-number=N
    │           └─ 替换 #notification_div DOM
    │
    └─► 标签页 2: (同标签页 1)
```

### 10.2 页面访问流程

```
用户访问 /notifications (通知中心)
    │
    ├─ 查询 Notification 表（按 user_id + status）
    ├─ 批量加载 Repo/Issue/Comment 关联数据
    └─ 渲染通知列表

用户访问首页 Dashboard
    │
    ├─ 查询 Action 表（按 user_id + created_unix）
    ├─ 可见性过滤（隐私、权限）
    └─ 渲染活动时间线
```

---

## 十一、关键代码文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 事件分发中心 | `services/notify/notify.go` |
| Notifier 接口 | `services/notify/notifier.go` |
| 活动 Notifier 实现 | `services/feed/notifier.go` |
| 活动匹配逻辑 | `services/feed/feed.go` |
| 通知 Notifier 实现 | `services/uinotification/notify.go` |
| 通知匹配逻辑 | `models/activities/notification_list.go` |
| 后端 EventSource Manager | `modules/eventsource/manager.go` |
| EventSource 运行循环 | `modules/eventsource/manager_run.go` |
| 用户 Messenger | `modules/eventsource/messenger.go` |
| EventSource HTTP 端点 | `routers/web/events/events.go` |
| 前端 SharedWorker | `web_src/js/eventsource.sharedworker.ts` |
| 前端通知计数更新 | `web_src/js/features/notification.ts` |
| Worker 封装类 | `web_src/js/modules/worker.ts` |
| WorkerPoolQueue 实现 | `modules/queue/workerqueue.go` |
| Worker 组调度 | `modules/queue/workergroup.go` |
| Action 模型 | `models/activities/action.go` |
| Notification 模型 | `models/activities/notification.go` |
| Web 通知展现 | `routers/web/user/notification.go` |
| Web 活动展现 | `routers/web/repo/activity.go` |
| API 通知接口 | `routers/api/v1/notify/notifications.go` |
| 系统初始化顺序 | `routers/init.go` |
| 路由定义 | `routers/web/web.go` |
