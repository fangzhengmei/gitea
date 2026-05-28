# Gitea Issue / PR 通知订阅与收件人计算代码链路分析

## 一、整体架构：观察者模式的多 Notifier 并行分发

Gitea 的通知系统基于 **Notifier 接口** 设计，采用观察者模式实现事件的多通道并行分发。所有 notifier 在系统启动时注册，事件发生时同步广播到所有 notifier。

### 核心入口

**`services/notify/notifier.go:20-85`** 定义了 `Notifier` 接口，包含所有可通知的事件类型：

```go
type Notifier interface {
    Run()
    NewIssue(ctx, issue, mentions)
    CreateIssueComment(ctx, doer, repo, issue, comment, mentions)
    IssueChangeStatus(ctx, ...)
    NewPullRequest(ctx, pr, mentions)
    PullRequestReview(ctx, ...)
    // ... 其他 30+ 事件方法
}
```

**`services/notify/notify.go:20-26`** 实现 notifier 注册与事件分发：

```go
var notifiers []Notifier

func RegisterNotifier(notifier Notifier) {
    go notifier.Run()
    notifiers = append(notifiers, notifier)
}

func NewIssue(ctx, issue, mentions) {
    for _, notifier := range notifiers {
        notifier.NewIssue(ctx, issue, mentions)  // 同步广播到所有 notifier
    }
}
```

### 已注册的 Notifier 实现

| Notifier | 注册位置 | 作用 |
|---------|---------|------|
| `uinotification` | `services/uinotification/notify.go:36` | 站内信通知 |
| `mailNotifier` | `services/mailer/mailer.go:34` | 邮件通知（需 `EnableNotifyMail` 配置） |
| `webhook` | `services/webhook/notifier.go` | Webhook 推送 |
| `feed` | `services/feed/notifier.go` | RSS/Atom 订阅 |
| `indexer` | `services/indexer/indexer.go` | 搜索索引更新 |
| `mirror` | `services/mirror/notifier.go` | 镜像同步 |
| `actions` | `services/actions/init.go` | CI/CD 流水线触发 |

---

## 二、关注状态聚合：三级订阅体系

收件人计算基于 **三级订阅体系**，站内信与邮件各自独立计算但逻辑相似。

### 2.1 数据模型

#### Issue 级订阅 (`issue_watch` 表)

**`models/issues/issue_watch.go:16-23`**

```go
type IssueWatch struct {
    ID         int64
    UserID     int64  `xorm:"UNIQUE(watch) NOT NULL"`
    IssueID    int64  `xorm:"UNIQUE(watch) NOT NULL"`
    IsWatching bool   `xorm:"NOT NULL"`  // true=关注, false=忽略
}
```

- **显式关注**：`IsWatching = true`，用户主动点击"Subscribe"
- **显式忽略**：`IsWatching = false`，用户主动点击"Unsubscribe"，优先级最高

#### 仓库级订阅 (`watch` 表)

**`models/repo/watch.go:15-37`**

```go
type WatchMode int8
const (
    WatchModeNone   WatchMode = iota  // 0: 未订阅
    WatchModeNormal                   // 1: 主动关注
    WatchModeDont                     // 2: 明确不自动关注
    WatchModeAuto                     // 3: 自动关注（提交代码后自动）
)

func IsWatchMode(mode WatchMode) bool {
    return mode != WatchModeNone && mode != WatchModeDont
}
```

#### 参与者 (Participants)

无需显式订阅，以下用户自动成为参与者：
- Issue 作者 (`PosterID`)
- 评论者
- PR 审查者
- 被分配人 (Assignee)

### 2.2 关注状态检查逻辑

**`models/issues/issue_watch.go:72-85`**

```go
func CheckIssueWatch(ctx, user, issue) (bool, error) {
    iw, exist, _ := GetIssueWatch(ctx, user.ID, issue.ID)
    if exist {
        return iw.IsWatching, nil  // Issue 级显式设置优先级最高
    }
    w, _ := repo_model.GetWatch(ctx, user.ID, issue.RepoID)
    // 仓库级关注 OR 是参与者
    return repo_model.IsWatchMode(w.Mode) || IsUserParticipantsOfIssue(ctx, user, issue), nil
}
```

### 2.3 站内信收件人聚合

**`models/activities/notification_list.go:78-126`** (`createOrUpdateIssueNotifications`)

```go
// 当 receiverID = 0 时（广播给所有关注者）
toNotify = make(container.Set[int64], 32)

// 1. Issue 级关注者
issueWatches, _ := issues_model.GetIssueWatchersIDs(ctx, issueID, true)
toNotify.AddMultiple(issueWatches...)

// 2. 仓库级关注者（WIP PR 排除）
if !(issue.IsPull && HasWorkInProgressPrefix(issue.Title)) {
    repoWatches, _ := repo_model.GetRepoWatchersIDs(ctx, issue.RepoID)
    toNotify.AddMultiple(repoWatches...)
}

// 3. 参与者（评论者、审查者等）
issueParticipants, _ := issue.GetParticipantIDsByIssue(ctx)
toNotify.AddMultiple(issueParticipants...)

// 4. 排除操作者本人
delete(toNotify, notificationAuthorID)

// 5. 排除 Issue 级显式忽略者
issueUnWatches, _ := issues_model.GetIssueWatchersIDs(ctx, issueID, false)
for _, id := range issueUnWatches {
    toNotify.Remove(id)
}
```

### 2.4 邮件收件人聚合

**`services/mailer/mail_issue.go:27-104`** (`mailIssueCommentToParticipants`)

```go
unfiltered := make([]int64, 1, 64)

// 1. 原始作者
unfiltered[0] = comment.Issue.PosterID

// 2. 被分配人
ids, _ := issues_model.GetAssigneeIDsByIssue(ctx, comment.Issue.ID)
unfiltered = append(unfiltered, ids...)

// 3. 参与者
ids, _ = issues_model.GetParticipantsIDsByIssueID(ctx, comment.Issue.ID)
unfiltered = append(unfiltered, ids...)

// 4. Issue 级关注者
ids, _ = issues_model.GetIssueWatchersIDs(ctx, comment.Issue.ID, true)
unfiltered = append(unfiltered, ids...)

// 5. 仓库级关注者（放最后，数量最多）
if !(issue.IsPull && IsWorkInProgress && ActionType != CreatePullRequest) {
    ids, _ = repo_model.GetRepoWatchersIDs(ctx, comment.Issue.RepoID)
    unfiltered = append(ids, unfiltered...)
}

// 6. 排除操作者本人（除非设置了 "Send me own activity notifications"）
if comment.Doer.EmailNotificationsPreference != EmailNotificationsAndYourOwn {
    visited.Add(comment.Doer.ID)
}

// 7. 先处理 @mentions（单独发邮件）
mailIssueCommentBatch(ctx, comment, mentions, visited, true)

// 8. 排除 Issue 级显式忽略者
ids, _ = issues_model.GetIssueWatchersIDs(ctx, comment.Issue.ID, false)
visited.AddMultiple(ids...)

// 9. 批量处理其余收件人
unfilteredUsers, _ := user_model.GetMailableUsersByIDs(ctx, unfiltered, false)
mailIssueCommentBatch(ctx, comment, unfilteredUsers, visited, false)
```

---

## 三、@mention 解析：从文本到用户列表

### 3.1 解析流程

```
文本内容
    ↓
mdstripper.StripMarkdownBytes  # 剥离 Markdown 标记
    ↓
references.FindAllMentionsBytes  # 正则匹配 @user 和 @org/team
    ↓
ResolveIssueMentionsByVisibility  # 解析用户、检查可见性、展开团队成员
    ↓
FindAndUpdateIssueMentions  # 过滤被封禁用户、写入 DB
    ↓
得到最终 mentions []*user_model.User
```

### 3.2 正则匹配

**`modules/references/references.go:31`**

```go
mentionPattern = regexp.MustCompile(
    `(?:\s|^|\(|\[)(@[-\w][-.\w]*?|@[-\w][-.\w]*?/[-\w][-.\w]*?)(?:\s|$|[:,;.?!](\s|$)|'|\)|\])`,
)
```

支持两种格式：
- `@username` - 单个用户
- `@org/teamname` - 组织团队（会展开为所有成员）

### 3.3 可见性解析

**`models/issues/issue_update.go:543-620`** (`ResolveIssueMentionsByVisibility`)

```go
func ResolveIssueMentionsByVisibility(ctx, issue, doer, mentions) (users, err) {
    // 1. 分离用户提及和团队提及
    for _, name := range mentions {
        if strings.Contains(name, "/") {  // @org/team
            mentionTeams = append(mentionTeams, names[1])
        } else {  // @user
            resolved[name] = false  // 待验证
        }
    }

    // 2. 团队展开：查询有仓库访问权限的团队，获取其所有成员
    if len(mentionTeams) > 0 {
        teams, _ := db.GetEngine(ctx).
            Join("INNER", "team_repo", ...).
            Where("team_repo.repo_id=?", issue.Repo.ID).
            In("team.lower_name", mentionTeams).
            Find(&teams)

        teamusers, _ := db.GetEngine(ctx).
            Join("INNER", "team_user", ...).
            In("`team_user`.team_id", checked).
            Find(&teamusers)

        users = append(users, teamusers...)
    }

    // 3. 用户可见性检查
    usernames := keysOfMap(resolved)
    mentionedUsers, _ := user_model.GetManyUsersByNames(ctx, usernames)
    for _, u := range mentionedUsers {
        // 检查是否能访问 Issues/Pull Requests 单元
        if access_model.CheckRepoUnitUser(ctx, issue.Repo, u, unitType) {
            users = append(users, u)
        }
    }
    return users, nil
}
```

### 3.4 调用时机

`FindAndUpdateIssueMentions` 在以下场景被调用：

| 场景 | 调用位置 |
|-----|---------|
| Issue 创建 | `services/issue/issue.go:55` |
| PR 创建 | `services/pull/pull.go:178` |
| 评论创建 | `services/issue/comments.go:79` |
| PR 审查 | `services/pull/review.go` |

---

## 四、并行分发：站内信与邮件的双轨处理

站内信 (`uinotification`) 和邮件 (`mailer`) 是两个完全独立的 Notifier，各自维护队列、各自计算收件人。

### 4.1 站内信分发流程

**`services/uinotification/notify.go:66-88`**

```go
func (ns *notificationService) CreateIssueComment(ctx, doer, repo, issue, comment, mentions) {
    // 1. 广播给所有关注者（receiverID = 0）
    opts := issueNotificationOpts{
        IssueID:              issue.ID,
        NotificationAuthorID: doer.ID,
        CommentID:            comment.ID,
        ReceiverID:           0,  // 0 = 所有关注者
    }
    _ = ns.issueQueue.Push(opts)

    // 2. @mention 用户单独推送（确保即使没关注也能收到）
    for _, mention := range mentions {
        _ = ns.issueQueue.Push(issueNotificationOpts{
            IssueID:              issue.ID,
            NotificationAuthorID: doer.ID,
            ReceiverID:           mention.ID,  // 单独推送给这个用户
            CommentID:            comment.ID,
        })
    }
}
```

**队列处理** (`services/uinotification/notify.go:53-60`)：
```go
func handler(items ...issueNotificationOpts) []issueNotificationOpts {
    for _, opts := range items {
        activities_model.CreateOrUpdateIssueNotifications(
            ctx, opts.IssueID, opts.CommentID,
            opts.NotificationAuthorID, opts.ReceiverID,
        )
    }
    return nil
}
```

**通知创建/更新** (`models/activities/notification.go:151-190`)：
```go
func createIssueNotification(ctx, userID, issue, commentID, updatedByID) error {
    notification := &Notification{
        UserID:    userID,
        RepoID:    issue.RepoID,
        Status:    NotificationStatusUnread,
        IssueID:   issue.ID,
        CommentID: commentID,
        UpdatedBy: updatedByID,
        Source:    if(issue.IsPull, PullRequest, Issue),
    }
    return db.Insert(ctx, notification)
}

func updateIssueNotification(ctx, userID, issueID, commentID, updatedByID) error {
    notification, _ := GetIssueNotification(ctx, userID, issueID)
    if notification.Status == NotificationStatusRead {
        notification.Status = NotificationStatusUnread
        notification.CommentID = commentID
    }
    notification.UpdatedBy = updatedByID
    // 更新以重新排序通知列表
    _, err = db.GetEngine(ctx).ID(notification.ID).Cols(cols...).Update(notification)
    return err
}
```

### 4.2 邮件分发流程

**`services/mailer/notify.go:31-51`**

```go
func (m *mailNotifier) CreateIssueComment(ctx, doer, repo, issue, comment, mentions) {
    var act activities_model.ActionType
    switch comment.Type {
    case issues_model.CommentTypeClose:
        act = activities_model.ActionCloseIssue
    case issues_model.CommentTypeComment:
        act = activities_model.ActionCommentIssue
    // ...
    }
    MailParticipantsComment(ctx, comment, act, issue, mentions)
}
```

**批量发送** (`services/mailer/mail_issue.go:106-154`)：
```go
func mailIssueCommentBatch(ctx, comment, users, visited, fromMention) error {
    // 按用户语言分组，实现多语言邮件模板
    langMap := make(map[string][]*user_model.User)
    for _, user := range users {
        // 过滤：非活跃用户
        if !user.IsActive { continue }

        // 过滤：邮件通知偏好检查
        if !(user.EmailNotificationsPreference == Enabled ||
             user.EmailNotificationsPreference == AndYourOwn ||
             fromMention && user.EmailNotificationsPreference == OnMention) {
            continue
        }

        // 过滤：已处理过的用户
        if !visited.Add(user.ID) { continue }

        // 过滤：仓库单元权限检查
        if !access_model.CheckRepoUnitUser(ctx, issue.Repo, user, checkUnit) {
            continue
        }

        langMap[user.Language] = append(langMap[user.Language], user)
    }

    // 按语言批量发送，每批最多 100 人（MailBatchSize）
    for lang, receivers := range langMap {
        for i := 0; i < len(receivers); i += MailBatchSize {
            msgs, _ := composeIssueCommentMessages(ctx, comment, lang, receivers[i:i+100], fromMention)
            SendAsync(msgs...)  // 异步发送到邮件队列
        }
    }
    return nil
}
```

**邮件队列** (`services/mailer/mailer.go:48-63`)：
```go
mailQueue = queue.CreateSimpleQueue(..., "mail", func(items ...*Message) []*Message {
    for _, msg := range items {
        if err := sender_service.Send(sender, msg); err != nil {
            log.Error("Failed to send emails: %v", err)
        }
    }
    return nil
})
```

### 4.3 站内信 vs 邮件：关键差异

| 维度 | 站内信 (uinotification) | 邮件 (mailer) |
|-----|------------------------|--------------|
| 收件人计算位置 | `models/activities/notification_list.go` | `services/mailer/mail_issue.go` |
| 队列 | `notification-service` | `mail` |
| @mention 处理 | 单独推送（receiverID 设为用户 ID） | 单独批次发送（`fromMention=true`） |
| 邮件偏好 | 不检查（始终发送） | 检查 `EmailNotificationsPreference` |
| 语言分组 | 不需要 | 按语言分组，使用对应模板 |
| 批处理 | 无 | 每批 100 人 |
| 去重逻辑 | 使用 `container.Set[int64]` | 使用 `container.Set[int64]` + visited |

---

## 五、完整调用链路

### 5.1 Issue 创建流程

```
services/issue/issue.go:NewIssue()
    │
    ├─ issues_model.NewIssue()            # 写入数据库
    │
    ├─ issues_model.FindAndUpdateIssueMentions()
    │   ├─ references.FindAllMentionsMarkdown()    # 文本解析 @mention
    │   ├─ ResolveIssueMentionsByVisibility()      # 检查可见性、展开团队
    │   └─ UpdateIssueMentions()                   # 写入 issue_mention 表
    │
    └─ notify_service.NewIssue(ctx, issue, mentions)  # 广播到所有 notifier
        ├─ uinotification.NewIssue()
        │   └─ issueQueue.Push(ReceiverID=0)           # 推送给所有关注者
        │   └─ for mention ∈ mentions:
        │       issueQueue.Push(ReceiverID=mention.ID) # 单独推送给被@用户
        │
        └─ mailNotifier.NewIssue()
            └─ MailParticipants()
                └─ mailIssueCommentToParticipants()
                    ├─ 收集：作者 + 负责人 + 参与者 + Issue关注者 + Repo关注者
                    ├─ mailIssueCommentBatch(mentions, fromMention=true)   # 先发送给被@用户
                    └─ mailIssueCommentBatch(others, fromMention=false)    # 再发送给其他关注者
```

### 5.2 评论创建流程

```
services/issue/comments.go:CreateComment()
    │
    ├─ issues_model.CreateComment()        # 写入数据库
    │
    ├─ issues_model.FindAndUpdateIssueMentions()
    │   └─ (同上 mention 解析流程)
    │
    └─ notify_service.CreateIssueComment(ctx, doer, repo, issue, comment, mentions)
        ├─ uinotification.CreateIssueComment()
        │   └─ issueQueue.Push(ReceiverID=0)
        │   └─ for mention ∈ mentions: issueQueue.Push(ReceiverID=mention.ID)
        │
        └─ mailNotifier.CreateIssueComment()
            └─ MailParticipantsComment()
                └─ mailIssueCommentToParticipants()
                    └─ (同上述邮件收件人计算)
```

### 5.3 PR 审查流程

```
services/pull/review.go:SubmitReview()
    │
    ├─ issues_model.CreateReview()         # 创建审查记录
    │
    ├─ issues_model.FindAndUpdateIssueMentions()
    │   └─ (mention 解析)
    │
    └─ notify_service.PullRequestReview(ctx, pr, review, comment, mentions)
        ├─ uinotification.PullRequestReview()
        │   └─ issueQueue.Push(ReceiverID=0)
        │   └─ for mention ∈ mentions: issueQueue.Push(ReceiverID=mention.ID)
        │
        └─ mailNotifier.PullRequestReview()
            └─ MailParticipantsComment()
                └─ mailIssueCommentToParticipants()
```

---

## 六、关键设计洞察

### 6.1 并行分发的优势与代价

**优势**：
- 解耦：站内信和邮件逻辑完全独立，互不影响
- 容错：邮件发送失败不影响站内信送达
- 可扩展：新增通知渠道只需实现 Notifier 接口

**代价**：
- 收件人计算重复执行两次，存在冗余
- 两份代码需保持逻辑一致，维护成本高

### 6.2 @mention 的特殊处理

@mention 用户被特殊对待：
- **站内信**：即使关闭了通知，被 @ 时也会单独推送（`ReceiverID` 直接指定）
- **邮件**：即使设置为 "仅在被 @ 时发送邮件"，也能收到
- 这是因为 `FindAndUpdateIssueMentions` 在事件广播前就已解析好 mentions 列表，并作为参数传递给所有 notifier

### 6.3 显式忽略的优先级

Issue 级的 `IsWatching=false` 优先级最高，会在：
- 站内信：`createOrUpdateIssueNotifications` L119-125 中被移除
- 邮件：`mailIssueCommentToParticipants` L89-93 中被加入 visited 集合

确保用户明确取消订阅后不会收到任何通知。

### 6.4 WIP PR 的通知抑制

**`services/mailer/mail_issue.go:68`** 和 **`models/activities/notification_list.go:103`** 都对 WIP (Work In Progress) PR 进行了特殊处理：
- 非创建动作的 WIP PR 不通知仓库关注者
- 减少不必要的打扰

---

## 七、PR 状态变更的触发链路与通知路径

PR 有四种关键状态变更——合并、关闭、重开、草稿/Ready 切换——各自走不同的 notify 事件，收件人集合与排除规则也存在微妙差异。

### 7.1 PR 合并（手动 & 自动）

#### 触发入口

| 入口 | 代码位置 |
|-----|---------|
| Web UI 手动合并 | `routers/web/repo/pull.go` → `services/pull/merge.go:Merge()` |
| API 手动合并 | `routers/api/v1/repo/pull.go` → `services/pull/merge.go:Merge()` |
| 手动标记已合并 | `services/pull/merge.go:MergedManually()` |
| 自动合并 | `services/automerge/automerge.go:handlePullRequestAutoMerge()` → `services/pull/merge.go:Merge(wasAutoMerged=true)` |

#### 核心逻辑：合并与通知是分离的

**关键断点**：`Merge()` 并不通过 `IssueChangeStatus` 发通知，而是直接调用专属的 `MergePullRequest` / `AutoMergePullRequest`。

```
services/pull/merge.go:Merge()  (L223-300)
    │
    ├─ doMergeAndPush()               # git push 到 base 分支
    │
    ├─ pr, _ = GetPullRequestByID()   # 重新加载（post-receive hook 已更新 DB）
    │
    ├─ if wasAutoMerged:
    │   notify_service.AutoMergePullRequest(ctx, doer, pr)    # ← L291
    │ else:
    │   notify_service.MergePullRequest(ctx, doer, pr)        # ← L293
    │
    └─ handleCloseCrossReferences()   # 关闭关联的 Issue（会触发 IssueChangeStatus）
```

**注意**：合并时 `SetMerged()` 内部调用 `SetIssueAsClosed()` 关闭关联 Issue，但该操作发生在 DB 事务内（`services/pull/merge.go:700-736`），不会直接触发通知。关联 Issue 的关闭通知由 `handleCloseCrossReferences` → `issue_service.CloseIssue()` → `notify_service.IssueChangeStatus()` 单独触发。

#### 站内信路径（合并）

```
notify_service.MergePullRequest(ctx, doer, pr)
    │
    └─ uinotification.MergePullRequest()             # notify.go:125-130
        └─ issueQueue.Push({
               IssueID:              pr.Issue.ID,
               NotificationAuthorID: doer.ID,
               ReceiverID:           0,   # 广播：走 createOrUpdateIssueNotifications
               CommentID:            0,   # 无 Comment
           })
```

**收件人集合**（receiverID=0，走 `createOrUpdateIssueNotifications`）：
1. Issue 级关注者 (`issue_watch.IsWatching=true`)
2. 仓库级关注者 — **合并不受 WIP 抑制**（`HasWorkInProgressPrefix` 仅在 `createOrUpdateIssueNotifications` 里检查，合并后标题已不含 WIP 前缀）
3. 参与者（评论者、审查者等）
4. 排除操作者本人 (`delete(toNotify, notificationAuthorID)`)
5. 排除 Issue 级显式忽略者
6. 权限检查：`CheckRepoUnitUser(unit.TypePullRequests)`

#### 邮件路径（合并）

```
notify_service.MergePullRequest(ctx, doer, pr)
    │
    └─ mailNotifier.MergePullRequest()               # notify.go:138-146
        └─ MailParticipants(ctx, pr.Issue, doer, ActionMergePullRequest, nil)
            └─ mailIssueCommentToParticipants()
```

**特殊处理**：
- `mentions=nil`：合并操作不携带 @mention 列表
- `content=""`：合并邮件不含 Issue 原文内容（`mail_issue.go:165-168`，对 `ActionMergePullRequest` 清空 content）
- **WIP 抑制**：合并时 WIP 检查逻辑为 `comment.Issue.IsPull && IsWorkInProgress && ActionType != CreatePullRequest`。对合并操作 `ActionType=ActionMergePullRequest`，如果标题仍含 WIP 前缀，仓库关注者仍会被排除——但正常合并后 PR 标题不含 WIP 前缀，所以实际不受影响。

#### 自动合并 vs 手动合并的差异

| 维度 | `MergePullRequest` | `AutoMergePullRequest` |
|-----|--------------------|-----------------------|
| 站内信 | `ReceiverID=0` 广播 | 同 `MergePullRequest`（`ns.AutoMergePullRequest` 直接调用 `ns.MergePullRequest`） |
| 邮件 | `ForceDoerNotification=false` | **`ForceDoerNotification=true`**（`mail_issue.go:170`） |
| 操作者邮件 | 正常排除 | **自动合并的操作者也会收到邮件**（除非偏好为 `Disabled`） |
| 邮件内容 | `content=""` | `content=""` |
| mentions | `nil` | `nil` |

`ForceDoerNotification` 的效果（`mail_issue.go:79`）：
```go
if comment.Doer.EmailNotificationsPreference != EmailNotificationsAndYourOwn && !comment.ForceDoerNotification {
    visited.Add(comment.Doer.ID)  // ForceDoerNotification=true 时跳过此排除
}
```

### 7.2 PR/Issue 关闭

#### 触发入口

| 入口 | 代码位置 |
|-----|---------|
| Web UI 评论关闭 | `routers/web/repo/issue_comment.go:163` → `issue_service.CloseIssue()` |
| API 关闭 | `routers/api/v1/repo/issue.go:706` / `closeOrReopenIssue()` → `issue_service.CloseIssue()` |
| 提交关键词关闭 | `services/issue/commit.go:221` → `CloseIssue(ctx, refIssue, doer, c.Sha1)` |
| PR 合并联动关闭 | `services/pull/merge.go:318` → `issue_service.CloseIssue()` |

#### 通知链路

```
services/issue/status.go:CloseIssue()     (L17-40)
    │
    ├─ issues_model.CloseIssue()          # DB: SetIssueAsClosed → CreateComment(Close)
    │
    └─ notify_service.IssueChangeStatus(ctx, doer, commitID, issue, comment, true)
        │
        ├─ uinotification.IssueChangeStatus()       # notify.go:104-110
        │   └─ issueQueue.Push({
        │          IssueID:              issue.ID,
        │          NotificationAuthorID: doer.ID,
        │          CommentID:            actionComment.ID,  # ← 关闭操作的 Comment
        │          ReceiverID:           0,
        │      })
        │
        └─ mailNotifier.IssueChangeStatus()          # notify.go:59-78
            └─ MailParticipants(ctx, issue, doer, ActionClosePullRequest/ActionCloseIssue, nil)
                └─ mailIssueCommentToParticipants()
```

#### 关键差异：关闭走 `IssueChangeStatus` 而非 `MergePullRequest`

PR 的关闭与合并是**两个独立的通知事件**：
- **合并**：`notify_service.MergePullRequest` / `AutoMergePullRequest`
- **关闭**（不合并）：`notify_service.IssueChangeStatus(closeOrReopen=true)`

关闭一个 PR 不会合并代码，所以走 `IssueChangeStatus` 通道。

#### 收件人集合（关闭）

**站内信**：同 `createOrUpdateIssueNotifications` 通用逻辑。

**邮件**：
- `mentions=nil`：关闭操作无 @mention
- `content=""`：关闭邮件不含原文（`ActionCloseIssue/ActionClosePullRequest` 清空 content）
- 收集：作者 + Assignee + 参与者 + Issue 关注者 + 仓库关注者
- WIP 检查：**关闭 PR 时，如果标题仍含 WIP 前缀，仓库关注者不会收到邮件**
- `ForceDoerNotification=false`：操作者本人正常排除

### 7.3 PR/Issue 重开

#### 触发入口

| 入口 | 代码位置 |
|-----|---------|
| Web UI 重开 | `routers/web/repo/issue_comment.go:180` → `issue_service.ReopenIssue()` |
| API 重开 | `routers/api/v1/repo/pull.go:1066` → `issue_service.ReopenIssue()` |
| PR 合并联动重开 | `services/pull/merge.go:325` → `issue_service.ReopenIssue()` |
| 提交关键词重开 | `services/issue/commit.go:225` → `ReopenIssue()` |

#### 通知链路

```
services/issue/status.go:ReopenIssue()    (L44-52)
    │
    ├─ issues_model.ReopenIssue()         # DB: setIssueAsReopen → CreateComment(Reopen)
    │
    └─ notify_service.IssueChangeStatus(ctx, doer, commitID, issue, comment, false)
        │
        ├─ uinotification.IssueChangeStatus()       # 同关闭
        └─ mailNotifier.IssueChangeStatus()          # → MailParticipants(ActionReopen*)
```

#### 收件人集合（重开）

与关闭逻辑基本相同，但：
- `ActionType = ActionReopenIssue / ActionReopenPullRequest`
- `content=""`：重开邮件同样不含原文
- **WIP 不再适用**：重开时 Issue/PR 已非关闭状态，标题不含 WIP 的可能性更高；但若标题仍含 WIP，仓库关注者仍被排除

### 7.4 PR 草稿(WIP)与 Ready 切换

Gitea 没有独立的"WIP 状态"字段——WIP 完全由**标题前缀**决定。切换仅通过修改标题实现。

#### WIP 前缀判断

**`models/issues/pull.go:660-663`**

```go
func HasWorkInProgressPrefix(title string) bool {
    _, ok := CutWorkInProgressPrefix(title)
    return ok
}
```

前缀列表由 `setting.Repository.PullRequest.WorkInProgressPrefixes` 配置，默认 `[WIP:, [WIP], Draft:, (Draft)]`。

#### 触发链路：标题变更

```
services/issue/issue.go:ChangeTitle()     (L72-111)
    │
    ├─ issues_model.ChangeIssueTitle(ctx, issue, doer, oldTitle)
    │
    ├─ if IsPull && HasWorkInProgressPrefix(oldTitle) && !HasWorkInProgressPrefix(newTitle):
    │   ├─ PullRequestCodeOwnersReview()    # 触发 code owner 审查
    │   └─ reviewNotifiers = ...
    │
    └─ notify_service.IssueChangeTitle(ctx, doer, issue, oldTitle)
```

#### 站内信路径（WIP→Ready）

**`services/uinotification/notify.go:112-123`**

```go
func (ns *notificationService) IssueChangeTitle(ctx, doer, issue, oldTitle) {
    issue.LoadPullRequest(ctx)
    if issue.IsPull && HasWorkInProgressPrefix(oldTitle) && !issue.PullRequest.IsWorkInProgress(ctx) {
        issueQueue.Push({
            IssueID:              issue.ID,
            NotificationAuthorID: doer.ID,
            CommentID:            0,   # 无 Comment
            ReceiverID:           0,   # 广播
        })
    }
}
```

**关键断点**：只有 **WIP→Ready** 才发站内信，Ready→WIP 不发。且 `CommentID=0`，这意味着站内信只能更新已有通知的排序，无法创建关联到具体评论的通知。

#### 邮件路径（WIP→Ready）

**`services/mailer/notify.go:80-90`**

```go
func (m *mailNotifier) IssueChangeTitle(ctx, doer, issue, oldTitle) {
    issue.LoadPullRequest(ctx)
    if issue.IsPull && HasWorkInProgressPrefix(oldTitle) && !issue.PullRequest.IsWorkInProgress(ctx) {
        MailParticipants(ctx, issue, doer, ActionPullRequestReadyForReview, nil)
    }
}
```

同样 **只有 WIP→Ready 才发邮件**，Ready→WIP 不发。

#### 收件人集合（WIP→Ready）

**站内信**：走 `createOrUpdateIssueNotifications(ReceiverID=0)`

此时 PR 标题已不含 WIP 前缀，所以：
1. Issue 级关注者 ✓
2. **仓库级关注者 ✓**（`HasWorkInProgressPrefix(issue.Title)` 为 false，不再抑制）
3. 参与者 ✓
4. 排除操作者本人 ✓
5. 排除 Issue 级显式忽略者 ✓

**邮件**：走 `mailIssueCommentToParticipants()`

- `ActionType = ActionPullRequestReadyForReview`
- `mentions=nil`：无 @mention
- **content=issue.Content**：Ready for Review 邮件**包含 Issue 原文内容**（只有 Close/Reopen/Merge 才清空 content）
- 仓库关注者 ✓（标题已不含 WIP）
- `ForceDoerNotification=false`

#### Ready→WIP：无通知

这是容易忽视的断点：**将 PR 标题改回 WIP 不会触发任何通知**。用户不会被通知"PR 又变成草稿了"。这可能导致以下问题：
- 已经收到"Ready for Review"邮件的审查者，不知道 PR 又回到了草稿状态
- 站内信列表中仍显示之前的"Ready"通知，不会更新

---

## 八、收件人集合对比：来源、去重顺序与排除规则

### 8.1 站内信收件人计算统一流程

所有站内信事件最终都走同一个函数 `createOrUpdateIssueNotifications`（`models/activities/notification_list.go:78-162`），但入口参数不同：

| 事件 | ReceiverID | CommentID | 效果 |
|-----|-----------|-----------|------|
| 创建 Issue/评论 | 0 + 各 mention.ID | comment.ID | 广播 + mention 单独推送 |
| 合并 PR | 0 | 0 | 广播，无关联 Comment |
| 自动合并 PR | 0 | 0 | 同上 |
| 关闭/重开 | 0 | actionComment.ID | 广播，关联关闭/重开 Comment |
| WIP→Ready | 0 | 0 | 广播，无关联 Comment |
| Assignee 变更 | assignee.ID | comment.ID | 仅指定用户 |
| Review 请求 | reviewer.ID | comment.ID | 仅指定用户 |

**站内信去重与排除顺序**（`createOrUpdateIssueNotifications`）：

```
① 收集 Issue 级关注者 (IsWatching=true)    → Set.AddMultiple
② 收集仓库级关注者 (mode≠Dont)             → Set.AddMultiple（WIP PR 跳过）
③ 收集参与者                                 → Set.AddMultiple
④ 排除操作者本人                             → Set.Delete(notificationAuthorID)
⑤ 排除 Issue 级显式忽略者 (IsWatching=false) → Set.Remove(各 ID)
⑥ 对集合中每个用户：权限检查 CheckRepoUnitUser
⑦ 已有通知 → updateIssueNotification
   无通知 → createIssueNotification
```

**注意**：当 `ReceiverID > 0` 时（如 @mention、Assignee 变更），跳过 ①-⑤，直接为指定用户创建/更新通知。但 **Issue 级显式忽略者的排除也被跳过**——这意味着被 @mention 时即使用户显式 unwatch 了一个 Issue，仍会收到站内信通知。

### 8.2 邮件收件人计算统一流程

所有邮件事件最终都走 `mailIssueCommentToParticipants`（`services/mailer/mail_issue.go:27-104`），但入参 `mailComment` 不同：

| 事件 | ActionType | mentions | content | ForceDoerNotification |
|-----|-----------|----------|---------|----------------------|
| 创建 Issue | ActionCreateIssue | 解析结果 | issue.Content | false |
| 创建评论 | ActionCommentIssue | 解析结果 | comment.Content | false |
| 合并 PR | ActionMergePullRequest | nil | "" | false |
| 自动合并 PR | ActionAutoMergePullRequest | nil | "" | **true** |
| 关闭 | ActionCloseIssue/PR | nil | "" | false |
| 重开 | ActionReopenIssue/PR | nil | "" | false |
| WIP→Ready | ActionPullRequestReadyForReview | nil | issue.Content | false |

**邮件去重与排除顺序**（`mailIssueCommentToParticipants`）：

```
① 收集候选 ID 列表 (unfiltered[]):
   a. 原始作者 (PosterID)
   b. Assignee IDs
   c. 参与者 IDs
   d. Issue 级关注者 (IsWatching=true)
   e. 仓库级关注者 (mode≠Dont)  ← WIP PR 且非创建动作时跳过

② 初始化 visited 集合

③ 排除操作者本人:
   if Doer.EmailNotificationsPreference ≠ AndYourOwn && !ForceDoerNotification:
       visited.Add(Doer.ID)

④ 先处理 @mentions (fromMention=true):
   mailIssueCommentBatch(mentions, visited, true)
   ├─ 过滤: 非活跃用户
   ├─ 过滤: 邮件偏好 (Enabled || AndYourOwn || (OnMention && fromMention))
   ├─ 过滤: visited 去重
   ├─ 过滤: 仓库单元权限
   └─ 按语言分组 → 批量发送 (每批 100)

⑤ 排除 Issue 级显式忽略者 (IsWatching=false):
   visited.AddMultiple(issueUnWatchIDs...)

⑥ 处理其余收件人 (fromMention=false):
   unfilteredUsers = GetMailableUsersByIDs(unfiltered)
   mailIssueCommentBatch(unfilteredUsers, visited, false)
   ├─ 过滤: 非活跃用户
   ├─ 过滤: 邮件偏好 (Enabled || AndYourOwn)  ← 不含 OnMention
   ├─ 过滤: visited 去重 (已被 mention 或 unwatch 的不会重复)
   ├─ 过滤: 仓库单元权限
   └─ 按语言分组 → 批量发送
```

### 8.3 关键差异：站内信 vs 邮件的排除规则

| 规则 | 站内信 | 邮件 |
|-----|--------|------|
| 操作者本人排除 | **始终排除**（`delete(toNotify, authorID)`） | 排除，但 `EmailNotificationsAndYourOwn` 或 `ForceDoerNotification` 时不排除 |
| 显式忽略者 | 先加入集合再移除（最终排除） | 加入 visited 集合（在 mentions 处理之后） |
| @mention 用户是否受显式忽略影响 | **否**（ReceiverID>0 时跳过 unwatch 排除） | **是**（unwatch 排除在 mention 处理之后，visited 包含 unwatcher） |
| 邮件偏好 | 不检查 | 检查 `EmailNotificationsPreference` |
| 活跃状态 | 由 JOIN `user.is_active AND !prohibit_login` 保证 | 显式检查 `user.IsActive` |
| 权限检查 | `CheckRepoUnitUser` | `CheckRepoUnitUser` |

**最重要的不对称**：@mention 用户在站内信中**不受显式忽略影响**，但在邮件中**受显式忽略影响**。

这意味着：
- 用户 A unwatch 了 Issue #1，有人在 #1 中 @A → A **会收到**站内信，但**不会收到**邮件
- 这是站内信与邮件之间最大的行为不一致

### 8.4 各事件收件人集合一览

| 事件 | 站内信候选 | 邮件候选 | mentions | WIP 抑制 |
|-----|----------|---------|----------|---------|
| 创建 Issue | Issue 关注者 + 仓库关注者 + 参与者 + @mention | 作者 + Assignee + 参与者 + Issue 关注者 + 仓库关注者 + @mention | ✓ | 否（非 PR） |
| 创建评论 | 同上 | 同上 | ✓ | WIP PR 不通知仓库关注者 |
| 合并 PR | Issue 关注者 + 仓库关注者 + 参与者 | 作者 + Assignee + 参与者 + Issue 关注者 + 仓库关注者 | ✗ | 合并后不含 WIP，不受影响 |
| 自动合并 PR | 同合并 | 同合并 | ✗ | 同合并；**操作者收到邮件** |
| 关闭 PR/Issue | Issue 关注者 + 仓库关注者 + 参与者 | 作者 + Assignee + 参与者 + Issue 关注者 + 仓库关注者 | ✗ | WIP PR 不通知仓库关注者 |
| 重开 PR/Issue | 同关闭 | 同关闭 | ✗ | WIP PR 不通知仓库关注者 |
| WIP→Ready | Issue 关注者 + **仓库关注者** + 参与者 | 作者 + Assignee + 参与者 + Issue 关注者 + **仓库关注者** | ✗ | 否（标题已不含 WIP） |
| Ready→WIP | **不发送** | **不发送** | ✗ | N/A |

---

## 九、Code Owner 审查请求通知链路

Code Owner 审查请求是 PR 从 WIP 切到 Ready 时自动触发的特殊通知路径，与普通评论/状态变更的广播模式不同，采用**定向分发**模式——只通知指定的评审人，不通知全体关注者。

### 9.1 触发入口与路径

Code Owner 审查请求有两个触发时机：

| 触发场景 | 代码位置 |
|---------|---------|
| PR 创建时（非 WIP） | `services/pull/pull.go:144-149` |
| WIP → Ready 切换时 | `services/issue/issue.go:95-104` |

**核心调用链路**：

```
PR 创建 或 WIP→Ready
    │
    └─ issue_service.PullRequestCodeOwnersReview(ctx, pr)
        │
        ├─ ① 读取 CODEOWNERS 文件（查找顺序：CODEOWNERS → docs/CODEOWNERS → .gitea/CODEOWNERS）
        ├─ ② 解析规则：glob 模式匹配变更文件
        ├─ ③ 匹配得到 uniqUsers 和 uniqTeams
        ├─ ④ 过滤：排除 PR 作者、排除已有审查记录的用户
        ├─ ⑤ 为每个用户调用 AddReviewRequest()
        │      ├─ 创建 review 记录（ReviewTypeRequest）
        │      └─ 创建 CommentTypeReviewRequest 评论
        ├─ ⑥ 为每个团队调用 AddTeamReviewRequest()
        │      └─ 创建团队 review 记录 + 评论
        └─ 返回 ReviewRequestNotifier 列表（每个待通知的评审人/团队一条）
    │
    └─ issue_service.ReviewRequestNotify(ctx, issue, poster, notifiers)
        │
        └─ for each notifier:
            if 单个用户:
                notify_service.PullRequestReviewRequest(reviewer=notifier.Reviewer)
            if 团队:
                teamReviewRequestNotify(notifier.ReviewTeam)
                    └─ GetTeamMembers()
                       for each member:
                           if member.ID != PosterID:
                               notify_service.PullRequestReviewRequest(reviewer=member)
```

### 9.2 关键数据结构

**`services/issue/pull.go:20-25`**

```go
type ReviewRequestNotifier struct {
    Comment    *issues_model.Comment   // ReviewRequest 类型的评论
    IsAdd      bool                    // true=添加审查请求
    Reviewer   *user_model.User        // 单个评审人（互斥）
    ReviewTeam *org_model.Team         // 评审团队（互斥）
}
```

### 9.3 CODEOWNERS 文件解析与匹配

**`services/issue/pull.go:33-166`** (`PullRequestCodeOwnersReview`)

```go
// 前置条件检查
if pr.IsWorkInProgress(ctx)    { return nil, nil }  // WIP 不触发
if pr.BaseRepo.IsFork            { return nil, nil }  // Fork 仓库不触发

// ① 读取 CODEOWNERS（默认分支）
for _, file := range ["CODEOWNERS", "docs/CODEOWNERS", ".gitea/CODEOWNERS"] {
    if blob, err := commit.GetBlobByPath(file); err == nil {
        data = blob.GetBlobContent()
        break
    }
}

// ② 解析规则
rules, _ := issues_model.GetCodeOwnersFromContent(ctx, data)

// ③ 获取变更文件列表（mergeBase 与 head 之间）
mergeBase, _ := gitrepo.MergeBase(ctx, pr.BaseRepo, baseBranch, headRef)
changedFiles, _ := repo.GetFilesChangedBetween(mergeBase, pr.GetGitHeadRefName())

// ④ 匹配变更文件，收集用户与团队
uniqUsers := make(map[int64]*user_model.User)
uniqTeams := make(map[string]*org_model.Team)  // key: "orgID/teamID"
for _, rule := range rules {
    for _, f := range changedFiles {
        matched, _ := rule.Rule.MatchString(f)
        if matched {
            for _, u := range rule.Users { uniqUsers[u.ID] = u }
            for _, t := range rule.Teams { uniqTeams[key(t)] = t }
        }
    }
}

// ⑤ 排除 PR 作者与已有审查记录的用户
for _, u := range uniqUsers {
    if u.ID != issue.Poster.ID && !contain(latestReviews, u) {
        comment, _ := issues_model.AddReviewRequest(ctx, issue, u, poster, true)
        if comment != nil {
            notifiers = append(notifiers, &ReviewRequestNotifier{
                Comment: comment, IsAdd: true, Reviewer: u,
            })
        }
    }
}

// ⑥ 团队审查请求
for _, t := range uniqTeams {
    comment, _ := issues_model.AddTeamReviewRequest(ctx, issue, t, poster, true)
    if comment != nil {
        notifiers = append(notifiers, &ReviewRequestNotifier{
            Comment: comment, IsAdd: true, ReviewTeam: t,
        })
    }
}
```

**Code Owner 元数据标记**：`AddReviewRequest` 的最后一个参数 `isCodeOwners=true` 时，会在评论元数据中标记 `SpecialDoerNameCodeOwners`，用于前端展示"Requested review by code owners"。

### 9.4 团队审查请求的成员展开

**`services/issue/review_request.go:210-233`** (`teamReviewRequestNotify`)

```go
func teamReviewRequestNotify(ctx, issue, doer, reviewerTeam, isAdd, comment) error {
    // ① 获取团队所有成员
    members, _ := organization.GetTeamMembers(ctx, &SearchMembersOptions{
        TeamID: reviewerTeam.ID,
    })

    // ② 逐个成员发送审查请求通知
    for _, member := range members {
        if member.ID == comment.Issue.PosterID {
            continue  // 排除 PR 作者
        }
        comment.AssigneeID = member.ID  // 复用 Comment，设置 AssigneeID
        notify_service.PullRequestReviewRequest(ctx, doer, issue, member, isAdd, comment)
    }
    return nil
}
```

**注意**：团队审查请求不直接通知团队，而是**展开为每个成员的单独通知**。这与站内信/邮件的实现一致——两者都只处理单个 `reviewer` 用户，不处理团队概念。

### 9.5 站内信：定向通知

**`services/uinotification/notify.go:238-252`**

```go
func (ns *notificationService) PullRequestReviewRequest(ctx, doer, issue, reviewer, isRequest, comment) {
    if isRequest {
        opts := issueNotificationOpts{
            IssueID:              issue.ID,
            NotificationAuthorID: doer.ID,
            ReceiverID:           reviewer.ID,  // 定向推送
        }
        if comment != nil {
            opts.CommentID = comment.ID       // 关联到 ReviewRequest 评论
        }
        _ = ns.issueQueue.Push(opts)
    }
}
```

**站内信关键特性**：
- **定向推送**：`ReceiverID = reviewer.ID`，跳过通用的广播逻辑
- **跳过 unwatch 检查**：因为 `ReceiverID > 0`，`createOrUpdateIssueNotifications` 会直接创建/更新通知，不检查 Issue 级显式忽略
- **关联评论**：`CommentID = comment.ID`，站内信可定位到具体的审查请求评论
- **去重由上游保证**：`AddReviewRequest` 在 DB 层面会检查 reviewer 是否已存在审查请求，重复添加会返回 `comment=nil`，不会触发通知

### 9.6 邮件：定向通知

**`services/mailer/notify.go:129-136`**

```go
func (m *mailNotifier) PullRequestReviewRequest(ctx, doer, issue, reviewer, isRequest, comment) {
    if isRequest && doer.ID != reviewer.ID && reviewer.EmailNotificationsPreference != EmailNotificationsDisabled {
        ct := fmt.Sprintf("Requested to review %s.", issue.HTMLURL(ctx))
        if err := SendIssueAssignedMail(ctx, issue, doer, ct, comment, []*user_model.User{reviewer}); err != nil {
            log.Error("Error in SendIssueAssignedMail ...: %v", err)
        }
    }
}
```

**邮件关键特性**：
- **定向推送**：收件人列表 `[]*user_model.User{reviewer}` 只有一人
- **前置过滤三层**：
  1. `isRequest`：只有添加审查请求才发，移除不发
  2. `doer.ID != reviewer.ID`：排除自己请求自己审查
  3. `reviewer.EmailNotificationsPreference != Disabled`：检查邮件偏好
- **复用 Assigned 邮件模板**：邮件主题/正文模板与分配 Issue 相同，仅 content 文本不同（`"Requested to review ..."`）

### 9.7 邮件发送流程：`SendIssueAssignedMail`

**`services/mailer/mail_issue.go:186-217`**

```go
func SendIssueAssignedMail(ctx, issue, doer, content, comment, recipients) error {
    // ① 按语言分组，与批量邮件一致
    langMap := make(map[string][]*user_model.User)
    for _, user := range recipients {
        if !user.IsActive {
            continue  // 过滤非活跃用户
        }
        langMap[user.Language] = append(langMap[user.Language], user)
    }

    // ② 逐语言发送
    for lang, tos := range langMap {
        msgs, err := composeIssueCommentMessages(ctx, &mailComment{
            Issue:      issue,
            Doer:       doer,
            ActionType: ActionType(0),  // 特殊：无 ActionType
            Content:    content,        // "Requested to review ..."
            Comment:    comment,
        }, lang, tos, false, "issue assigned")  // 使用 issue assigned 模板
        if err == nil {
            SendAsync(msgs...)
        }
    }
    return nil
}
```

### 9.8 去重逻辑对比

| 层级 | 站内信 | 邮件 |
|-----|--------|------|
| **DB 层面** | `AddReviewRequest` 检查 reviewer 是否已有审查记录，重复添加返回 `comment=nil` | 同站内信（共享同一 DB 检查） |
| **用户排除** | 无（因为 `ReceiverID>0` 跳过 unwatch 排除） | `doer.ID != reviewer.ID` + `EmailNotificationsPreference != Disabled` |
| **团队成员展开** | 无（团队在上游已展开为成员） | 无（团队在上游已展开为成员） |
| **作者排除** | `teamReviewRequestNotify` 中排除 `member.ID == PosterID` | 同站内信（共享同一上游） |
| **非活跃用户** | 由 DB JOIN 条件保证 | `SendIssueAssignedMail` 中显式检查 `user.IsActive` |

### 9.9 三种审查请求入口的一致性

审查请求有三个独立入口，最终都走到同一个通知路径：

| 入口 | 触发位置 | 通知路径 |
|-----|---------|---------|
| Code Owner 自动请求 | `PullRequestCodeOwnersReview` | `ReviewRequestNotify` → `PullRequestReviewRequest` |
| 用户手动添加评审人 | `services/issue/review_request.go:ReviewRequest` | 直接 `notify_service.PullRequestReviewRequest` |
| 用户手动添加评审团队 | `services/issue/review_request.go:TeamReviewRequest` | `teamReviewRequestNotify` → `PullRequestReviewRequest` |

三者共享同一个 `PullRequestReviewRequest` notifier 事件，所以站内信和邮件的行为完全一致。

### 9.10 关键断点与注意事项

1. **WIP 不触发 Code Owner**：`PullRequestCodeOwnersReview` L38-39 明确检查 `pr.IsWorkInProgress(ctx)`，WIP PR 直接返回 nil
2. **Fork 仓库不触发 Code Owner**：L49-51 检查 `pr.BaseRepo.IsFork`，fork 的目标仓库不触发
3. **团队审查请求只发添加通知**：`TeamReviewRequest` L191-193 中 `if comment == nil || !isAdd` 直接 return，移除团队审查请求不发通知
4. **代码作者排除**：团队成员展开时排除 PR 作者（`member.ID == comment.Issue.PosterID`），但 Code Owner 用户过滤时也排除了作者，所以是双重保险
5. **已有审查者不再通知**：`PullRequestCodeOwnersReview` L133 的 `!contain(latestReviews, u)` 检查避免重复通知已有审查记录的用户

---

## 十、核心文件索引

| 文件 | 职责 |
|-----|------|
| `services/notify/notifier.go` | Notifier 接口定义 |
| `services/notify/notify.go` | Notifier 注册与事件广播 |
| `services/uinotification/notify.go` | 站内信 Notifier 实现 |
| `services/mailer/notify.go` | 邮件 Notifier 实现 |
| `services/mailer/mail_issue.go` | 邮件收件人计算与批量发送 (`SendIssueAssignedMail`) |
| `services/mailer/mail_comment.go` | 评论邮件发送 (`MailParticipantsComment`, `MailMentionsComment`) |
| `services/mailer/mail_issue_common.go` | 邮件公共结构 (`mailComment`, `composeIssueCommentMessages`) |
| `services/mailer/mailer.go` | 邮件队列与发送 |
| `services/pull/merge.go` | PR 合并逻辑 (`Merge`, `MergedManually`, `SetMerged`) |
| `services/pull/pull.go` | PR 创建与 code owner 审查触发 |
| `services/automerge/automerge.go` | PR 自动合并逻辑 |
| `services/issue/status.go` | Issue/PR 关闭与重开 (`CloseIssue`, `ReopenIssue`) |
| `services/issue/issue.go` | Issue 创建与标题变更 (`NewIssue`, `ChangeTitle`) |
| `services/issue/pull.go` | Code Owner 审查逻辑 (`PullRequestCodeOwnersReview`, `ReviewRequestNotifier`) |
| `services/issue/review_request.go` | 审查请求处理 (`ReviewRequest`, `TeamReviewRequest`, `ReviewRequestNotify`, `teamReviewRequestNotify`) |
| `services/issue/commit.go` | 提交关键词关闭/重开 Issue |
| `models/activities/notification.go` | 通知数据模型与 CRUD |
| `models/activities/notification_list.go` | 站内信收件人聚合 (`CreateOrUpdateIssueNotifications`) |
| `models/issues/issue_watch.go` | Issue 级关注模型 |
| `models/issues/issue_update.go` | @mention 解析 + Issue 关闭/重开 DB 操作 |
| `models/issues/pull.go` | PR 模型 + WIP 前缀判断 |
| `models/issues/review.go` | 审查请求 DB 操作 (`AddReviewRequest`, `AddTeamReviewRequest`) |
| `models/repo/watch.go` | 仓库级关注模型与 WatchMode |
| `modules/references/references.go` | @mention 正则匹配与 Markdown 剥离 |
