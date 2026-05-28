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

## 七、核心文件索引

| 文件 | 职责 |
|-----|------|
| `services/notify/notifier.go` | Notifier 接口定义 |
| `services/notify/notify.go` | Notifier 注册与事件广播 |
| `services/uinotification/notify.go` | 站内信 Notifier 实现 |
| `services/mailer/notify.go` | 邮件 Notifier 实现 |
| `services/mailer/mail_issue.go` | 邮件收件人计算与批量发送 |
| `services/mailer/mailer.go` | 邮件队列与发送 |
| `models/activities/notification.go` | 通知数据模型与 CRUD |
| `models/activities/notification_list.go` | 站内信奉件人聚合 (`CreateOrUpdateIssueNotifications`) |
| `models/issues/issue_watch.go` | Issue 级关注模型 |
| `models/repo/watch.go` | 仓库级关注模型与 WatchMode |
| `models/issues/issue_update.go` | @mention 解析 (`FindAndUpdateIssueMentions`, `ResolveIssueMentionsByVisibility`) |
| `modules/references/references.go` | @mention 正则匹配与 Markdown 剥离 |
