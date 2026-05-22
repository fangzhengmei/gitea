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

> **代码证实**：Notifier 按注册顺序被调用，`actionNotifier` 先于 `notificationService` 执行。但由于通知使用异步队列，两者的 DB 写入顺序由调度决定。

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

**处理方式**：**同步**执行，在请求 goroutine 中直接写入 DB，使用 `db.WithTx` 事务包裹。

**匹配流程**（`services/feed/feed.go:97-162`）：

1. **事务包裹**：
   ```go
   func NotifyWatchers(ctx context.Context, acts ...*activities_model.Action) error {
       return db.WithTx(ctx, func(ctx context.Context) error {
           // ... 所有操作在同一事务内
       })
   }
   ```

2. **获取关注者列表**：
   ```go
   watchers, err := repo_model.GetWatchers(ctx, repoID)
   ```

3. **批量预计算权限**（在 `NotifyWatchers` 内完成，`services/feed/feed.go:124-145`）：
   ```go
   permCode := make([]bool, len(watchers))   // 代码可读权限
   permIssue := make([]bool, len(watchers))  // Issue 可读权限
   permPR := make([]bool, len(watchers))     // PR 可读权限
   
   for i, watcher := range watchers {
       user, err := user_model.GetUserByID(ctx, watcher.UserID)
       if err != nil {
           permCode[i] = false
           permIssue[i] = false
           permPR[i] = false
           continue
       }
       perm, err := access_model.GetIndividualUserRepoPermission(ctx, repo, user)
       permCode[i] = perm.CanRead(unit.TypeCode)
       permIssue[i] = perm.CanRead(unit.TypeIssues)
       permPR[i] = perm.CanRead(unit.TypePullRequests)
   }
   ```

4. **按动作类型过滤**（在内部 `notifyWatchers` 函数中，`services/feed/feed.go:72-86`）：
   ```go
   switch act.OpType {
   case activities_model.ActionCommitRepo, activities_model.ActionPushTag, activities_model.ActionDeleteTag:
       if !permCode[i] { continue }
   case activities_model.ActionCreateIssue, activities_model.ActionCommentIssue:
       if !permIssue[i] { continue }
   case activities_model.ActionCreatePullRequest, activities_model.ActionCommentPull:
       if !permPR[i] { continue }
   }
   ```

5. **排除操作者自身**：
   ```go
   if act.ActUserID == watcher.UserID {
       continue
   }
   ```

6. **插入记录**（`services/feed/feed.go:49-91`）：
   - 为操作者本人插入一条（UserID = ActUserID）
   - 为仓库所属组织插入一条（如果是组织仓库且 ActUserID != 组织 ID）
   - 为每个符合条件的关注者插入一条

   均使用 `db.Insert(ctx, act)` 同步写入。

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

1. **批量收集防抖**（代码常量，`modules/queue/workergroup.go:18`）：
   ```go
   batchDebounceDuration = 100 * time.Millisecond
   ```

2. **动态扩缩容**：
   ```go
   if full || noWorker {
       if q.workerNum < q.workerMaxNum || noWorker && q.workerMaxNum <= 0 {
           q.workerNum++
           q.doStartNewWorker(wg)
       }
   }
   ```

3. **批量处理**（`services/uinotification/notify.go:53-60`）：
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
       continue
   }
   if !issue.IsPull && !access_model.CheckRepoUnitUser(ctx, issue.Repo, user, unit.TypeIssues) {
       continue
   }
   ```

4. **创建或更新通知**：
   ```go
   if notificationExists(notifications, issue.ID, userID) {
       updateIssueNotification(ctx, userID, issue.ID, commentID, notificationAuthorID)
   } else {
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

1. **定时轮询**（间隔由配置决定，默认 10 秒，`modules/setting/ui.go:118`）：
   ```go
   timer := time.NewTicker(setting.UI.Notification.EventSourceUpdateTime)
   // 默认值: EventSourceUpdateTime: 10 * time.Second
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
    
    // 心跳（30秒，代码中硬编码）
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

**轮询配置**（默认值，`modules/setting/ui.go:115-117`）：
```go
MinTimeout:  10 * time.Second,  // 最小间隔
TimeoutStep: 10 * time.Second,  // 退避步长
MaxTimeout:  60 * time.Second,  // 最大间隔
```

**渐进式退避**（`web_src/js/features/notification.ts:55-76`）：
```typescript
async function updateNotificationCountWithCallback(...) {
    if (lastCount !== newCount) {
        timeout = notificationSettings.MinTimeout;  // 有变化，使用最小间隔
    } else if (timeout < notificationSettings.MaxTimeout) {
        timeout += notificationSettings.TimeoutStep;  // 无变化，增加间隔
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

**Web 路由**（`routers/web/web.go:1743-1750`）：
```go
m.Group("/notifications", func() {
    m.Get("", user.Notifications)
    // ...
    m.Get("/new", user.NewAvailable)  // 完整路径: /notifications/new
}, reqSignIn)
```

> **路由核查说明**：
> - 第 630 行 `m.Group("/user/settings/notifications", ...)` - 用户通知设置页面（父路径 `/user/settings`）
> - 第 1743 行 `m.Group("/notifications", ...)` - 通知中心（根路径 `/notifications`）
> - 两者位于不同的父分组下，路径不冲突。未读数路由 `/notifications/new` 定义正确。

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
**代码**：`services/feed/feed.go:21-25`

```go
func GetFeedsForDashboard(ctx context.Context, opts activities_model.GetFeedsOptions) (activities_model.ActionList, int64, error) {
    opts.DontCount = opts.RequestedTeam == nil && opts.Date == ""
    results, cnt, err := activities_model.GetFeeds(ctx, opts)
    return results, util.Iif(opts.DontCount, -1, cnt), err
}
```

> **事实校准**：`GetFeedsForDashboard` 是一个包装函数，设置 `DontCount` 标志后直接调用 `activities_model.GetFeeds`。当 `RequestedTeam` 为空且 `Date` 为空时，跳过计数查询，返回 `-1` 作为计数值。

**实际查询**在 `models/activities/action_list.go:207` 中进行，查询时进行可见性过滤：
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

## 七、活动流 vs 通知流：代码可证实的差异对比

### 7.1 处理模型对比

| 维度 | 活动流 (Action) | 通知流 (Notification) |
|------|---------------|---------------------|
| **处理方式** | 同步，请求 goroutine 内直接调用 `db.Insert` | 异步，先 `Push` 到 `WorkerPoolQueue`，由 worker 后续处理 |
| **事务** | 使用 `db.WithTx` 包裹所有写入操作 | 队列 handler 内独立事务 |
| **执行时机** | Notifier 方法返回时已完成写入 | Notifier 方法返回时仅完成入队，写入时机由队列调度决定 |
| **Notifier 调用顺序** | 先调用（注册顺序在前） | 后调用（注册顺序在后） |
| **错误处理** | 写入错误返回给调用方，事务回滚 | handler 内记录错误日志，不返回给调用方 |

### 7.2 处理流程代码证据

**活动流同步证据**（`services/feed/feed.go:97-98`）：
```go
func NotifyWatchers(ctx context.Context, acts ...*activities_model.Action) error {
    return db.WithTx(ctx, func(ctx context.Context) error {
        // ... 循环内直接调用 db.Insert
    })
}
```

**通知流异步证据**（`services/uinotification/notify.go:76`）：
```go
_ = ns.issueQueue.Push(opts)  // 仅入队，立即返回
```

**队列工作机制证据**（`modules/queue/workergroup.go:18, 19`）：
```go
batchDebounceDuration  = 100 * time.Millisecond  // 批量收集防抖
workerIdleDuration     = 1 * time.Second         // worker 空闲超时
```

**默认轮询间隔证据**（`modules/setting/ui.go:118`）：
```go
EventSourceUpdateTime: 10 * time.Second
```

### 7.3 触发顺序代码证据

**注册顺序**（`routers/init.go:130-131`）：
```go
mustInit(feed_service.Init)          // 1. actionNotifier 先注册
mustInit(uinotification.Init)       // 2. notificationService 后注册
```

**调用顺序**（`services/notify/notify.go:73-77`）：
```go
func NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User) {
    for _, notifier := range notifiers {  // 按切片顺序遍历
        notifier.NewIssue(ctx, issue, mentions)
    }
}
```

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

## 九、关键设计决策（代码可证实）

### 9.1 双轨并行设计
- **Action**：使用同步事务写入，与业务操作同生命周期
- **Notification**：使用异步队列解耦，不阻塞业务流程
- 两者独立存储、独立展现，但共享相同的事件源

### 9.2 实时推送架构
- **SharedWorker + EventSource**：多标签页共享单一连接，代码见 `web_src/js/eventsource.sharedworker.ts`
- **渐进式降级**：从 EventSource → 轮询，代码见 `web_src/js/features/notification.ts:34-49`
- **序列号防乱序**：通知列表刷新使用递增序列号，代码见 `web_src/js/features/notification.ts:6, 94`

### 9.3 性能优化手段
1. **批量预计算权限**：活动匹配前一次性计算所有关注者权限，代码见 `services/feed/feed.go:124-145`
2. **异步队列处理**：通知创建通过队列异步化，不阻塞业务流程，代码见 `services/uinotification/notify.go:46`
3. **批量数据加载**：展现层使用 `LoadRepos`/`LoadIssues` 批量加载关联数据，避免 N+1 查询
4. **联合索引**：数据库索引针对查询模式优化
5. **队列防抖**：100ms 批量收集，减少 DB 写入次数，代码见 `modules/queue/workergroup.go:18`

### 9.4 通知去重与更新
- 同一 Issue 对同一用户只有一条通知记录，代码见 `models/activities/notification_list.go:154`
- 新活动触发时更新现有通知的时间戳和状态，代码见 `models/activities/notification.go:170-190`

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
    │      │    ├─ db.WithTx 开启事务
    │      │    ├─ 获取仓库关注者
    │      │    ├─ 批量预计算权限 (permCode/permIssue/permPR)
    │      │    ├─ notifyWatchers() 内部函数
    │      │    │    ├─ 按动作类型过滤权限
    │      │    │    ├─ 排除操作者自身
    │      │    │    └─ db.Insert 写入 Action 表
    │      │    └─ 事务提交
    │      └─ 返回
    │
    └─► ② notificationService.CreateIssueComment() ── 异步
           │
           ├─ 构造 issueNotificationOpts
           ├─ issueQueue.Push(opts) ── 入队立即返回
           │
           └─ 队列后台处理
                │
                └─ handler()
                     └─ CreateOrUpdateIssueNotifications()
                          ├─ 收集候选接收者
                          ├─ 排除过滤
                          ├─ 权限校验
                          └─ 创建/更新 Notification 表
                                 │
                                 ▼
后端 EventSource Manager 定时轮询 (默认 10s)
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
    │           └─ 序列号校验通过后替换 #notification_div DOM
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
    ├─ GetFeedsForDashboard() 设置 DontCount 标志
    ├─ activities_model.GetFeeds() 查询 Action 表
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
| 活动匹配逻辑（含权限预计算） | `services/feed/feed.go` |
| GetFeedsForDashboard 包装函数 | `services/feed/feed.go:21-25` |
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
| Worker 组调度（含防抖常量） | `modules/queue/workergroup.go` |
| Action 模型 | `models/activities/action.go` |
| Notification 模型 | `models/activities/notification.go` |
| Action 查询逻辑 | `models/activities/action_list.go:207` |
| Web 通知展现 | `routers/web/user/notification.go` |
| Web 活动展现 | `routers/web/repo/activity.go` |
| API 通知接口 | `routers/api/v1/notify/notifications.go` |
| 系统初始化顺序 | `routers/init.go` |
| 路由定义 | `routers/web/web.go` |
| 通知配置默认值 | `modules/setting/ui.go:109-119` |
