# Gitea 用户活动与通知派发同步呈现逻辑分析

## 概述

Gitea 系统中存在两条并行但相关的数据链路：**用户活动（Activity/Action）** 和 **系统通知（Notification）**。两者共享相同的事件源，但通过不同的 Notifier 实现进行独立处理，最终在不同的展现入口呈现给用户。

---

## 一、整体架构

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
      │                               │
      ▼                               ▼
  Action 表                    Notification 表
(用户活动时间线)             (系统通知中心)
      │                               │
      ▼                               ▼
  展现入口                          展现入口
- 仓库活动页 /activity           - 通知中心 /notifications
- 用户首页 Dashboard            - API /notifications/new
```

---

## 二、第一阶段：事件采集触发

### 2.1 事件源定义

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

### 2.2 事件触发入口

所有业务操作通过 `services/notify/notify.go` 中的统一函数触发事件。该文件定义了 40+ 个事件触发函数，例如：

- `NewIssue()` - 新建 Issue
- `CreateIssueComment()` - 创建评论
- `NewPullRequest()` - 新建 PR
- `MergePullRequest()` - 合并 PR
- `PushCommits()` - 推送提交
- `IssueChangeStatus()` - Issue 状态变更

**核心代码示例**（`services/notify/notify.go:73-78`）：
```go
func NewIssue(ctx context.Context, issue *issues_model.Issue, mentions []*user_model.User) {
    for _, notifier := range notifiers {
        notifier.NewIssue(ctx, issue, mentions)
    }
}
```

### 2.3 Notifier 接口

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

系统启动时通过 `RegisterNotifier()` 注册所有实现，每个 Notifier 独立处理事件。

---

## 三、第二阶段：订阅匹配筛选

系统有两个核心 Notifier 实现，分别处理**活动记录**和**系统通知**。

### 3.1 活动（Action）订阅匹配 - `actionNotifier`

**位置**：`services/feed/notifier.go`

**职责**：为每个符合条件的用户创建活动时间线记录。

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

**异步处理架构**：
```go
ns.issueQueue = queue.CreateSimpleQueue(..., "notification-service", handler)
```

通知通过队列异步处理，避免阻塞业务流程。

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

## 四、第三阶段：展现入口

### 4.1 活动（Action）展现

#### 4.1.1 仓库活动页
**路由**：`/{owner}/{repo}/activity`  
**代码**：`routers/web/repo/activity.go:22-81`

展示仓库维度的统计数据：
- 提交数、PR 数、Issue 数
- 贡献者排行
- 按时间周期筛选（日/周/月/季/年）

#### 4.1.2 用户首页 Dashboard
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

### 4.2 通知（Notification）展现

#### 4.2.1 Web 通知中心
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

#### 4.2.2 未读数查询（AJAX 轮询）
**路由**：`/user/notifications/new`  
**代码**：`routers/web/user/notification.go:388-399`

```go
func NewAvailable(ctx *context.Context) {
    total, _ := db.Count[activities_model.Notification](ctx, activities_model.FindNotificationOptions{
        UserID: ctx.Doer.ID,
        Status: []activities_model.NotificationStatus{NotificationStatusUnread},
    })
    ctx.JSON(http.StatusOK, structs.NotificationCount{New: total})
}
```

#### 4.2.3 API 接口
**路由**：`GET /api/v1/notifications/new`  
**代码**：`routers/api/v1/notify/notifications.go:18-36`

提供与 Web 端一致的未读数查询能力。

---

## 五、数据模型对比

### 5.1 Action（活动）模型
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

### 5.2 Notification（通知）模型
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

## 六、关键设计决策

### 6.1 双轨并行设计
- **Action** 面向"发生了什么"，是公开的时间线记录
- **Notification** 面向"需要我关注什么"，是个人的待处理提醒
- 两者独立存储、独立展现，但共享相同的事件源

### 6.2 性能优化手段
1. **批量预计算权限**：通知匹配前一次性计算所有关注者权限
2. **异步队列处理**：通知创建通过队列异步化，不阻塞业务流程
3. **批量数据加载**：展现层使用 `LoadRepos`/`LoadIssues` 批量加载关联数据，避免 N+1 查询
4. **联合索引**：数据库索引针对查询模式优化

### 6.3 通知去重与更新
- 同一 Issue 对同一用户只有一条通知记录
- 新活动触发时更新现有通知的时间戳和状态
- 避免通知列表被同一 Issue 的多次更新淹没

---

## 七、核心接力流程总结

```
用户发表评论
    │
    ▼
services/notify/notify.go:CreateIssueComment()
    │
    ├─► actionNotifier.CreateIssueComment()
    │      │
    │      ├─ 构造 Action 对象
    │      ├─ NotifyWatchers()
    │      │    ├─ 获取仓库关注者
    │      │    ├─ 预计算权限
    │      │    ├─ 按动作类型过滤
    │      │    └─ 批量插入 Action 表
    │      └─ 完成（活动时间线更新）
    │
    └─► notificationService.CreateIssueComment()
           │
           ├─ 构造 issueNotificationOpts
           ├─ 加入 notification-service 队列
           │    └─ 异步处理 handler
           │         ├─ CreateOrUpdateIssueNotifications()
           │         │    ├─ 收集候选接收者（Issue关注+仓库关注+参与者）
           │         │    ├─ 排除触发者和取消关注者
           │         │    ├─ 权限校验
           │         │    └─ 创建/更新 Notification 记录
           │         └─ 完成（系统通知更新）
           │
           ▼
用户访问 /notifications
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

## 八、关键代码文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 事件分发中心 | `services/notify/notify.go` |
| Notifier 接口 | `services/notify/notifier.go` |
| 活动 Notifier 实现 | `services/feed/notifier.go` |
| 活动匹配逻辑 | `services/feed/feed.go` |
| 通知 Notifier 实现 | `services/uinotification/notify.go` |
| 通知匹配逻辑 | `models/activities/notification_list.go` |
| Action 模型 | `models/activities/action.go` |
| Notification 模型 | `models/activities/notification.go` |
| Web 通知展现 | `routers/web/user/notification.go` |
| Web 活动展现 | `routers/web/repo/activity.go` |
| API 通知接口 | `routers/api/v1/notify/notifications.go` |
