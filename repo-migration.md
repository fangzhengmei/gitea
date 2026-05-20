# Gitea 仓库迁移流程代码分析

本文档对照源码分析 Gitea 从外部站点迁移仓库的完整流程，包括**源站抓取**、**对象转写**、**权限映射**和**后台任务推进**四个核心阶段的接力方式。

---

## 一、整体架构

迁移系统采用 **Downloader + Uploader** 的抽象设计模式：

```
[外部站点] → Downloader → [通用中间格式] → Uploader → [Gitea 本地]
```

- **Downloader**：从源站（GitHub/GitLab/Gitea 等）抓取数据
- **Uploader**：将通用中间格式数据写入本地 Gitea 实例

关键接口定义在 `modules/migration/` 目录：
- `Downloader` 接口：`modules/migration/downloader.go:14-27`
- `Uploader` 接口：`modules/migration/uploader.go:10-26`

---

## 二、后台任务推进（第一棒：任务调度）

### 2.1 任务入口

迁移任务由 `services/task/migrate.go` 中的 `runMigrateTask` 函数驱动。

**任务状态流转**：
```
TaskStatusQueued → TaskStatusRunning → TaskStatusSuccess
                          ↓
                     TaskStatusFailed
```

**关键代码**（`services/task/migrate.go:44-154`）：

```go
func runMigrateTask(ctx context.Context, t *admin_model.Task) (err error) {
    // 1. 加载关联数据：仓库、执行者、所有者
    t.LoadRepo(ctx)
    t.LoadDoer(ctx)
    t.LoadOwner(ctx)
    
    // 2. 解析迁移配置
    opts, _ := t.MigrateConfig()
    
    // 3. 更新任务状态为运行中
    t.Status = structs.TaskStatusRunning
    t.UpdateCols(ctx, "start_time", "status")
    
    // 4. 启动取消监听协程（每2秒检查一次任务状态）
    go func() {
        for {
            select {
            case <-time.After(2 * time.Second):
            case <-ctx.Done():
                return
            }
            task, _ := admin_model.GetMigratingTask(ctx, t.RepoID)
            if task.Status != structs.TaskStatusRunning {
                cancel()  // 取消迁移上下文
                return
            }
        }
    }()
    
    // 5. 调用核心迁移函数
    t.Repo, err = migrations.MigrateRepository(ctx, t.Doer, t.Owner.Name, *opts, messenger)
    
    // 6. 完成或失败处理
    if err == nil {
        admin_model.FinishMigrateTask(ctx, t)
        notify_service.MigrateRepository(ctx, t.Doer, t.Owner, t.Repo)
    }
}
```

**设计要点**：
- 独立的取消监听协程，支持用户主动取消迁移
- 迁移过程中的进度通过 `messenger` 回调实时更新任务消息
- 失败时保留仓库记录，用户可查看错误信息

---

## 三、源站抓取（第二棒：Downloader 工厂模式）

### 3.1 Downloader 工厂注册

每种源站类型实现一个 `DownloaderFactory`，在包初始化时注册：

**GitHub 示例**（`services/migrations/github.go:34-36`）：
```go
func init() {
    RegisterDownloaderFactory(&GithubDownloaderV3Factory{})
}
```

**工厂选择逻辑**（`services/migrations/migrate.go:142-174`）：
```go
func newDownloader(ctx context.Context, ownerName string, opts base.MigrateOptions) (base.Downloader, error) {
    // 遍历所有已注册的工厂，匹配 GitServiceType
    for _, factory := range factories {
        if factory.GitServiceType() == opts.GitServiceType {
            downloader, err = factory.New(ctx, opts)
            break
        }
    }
    
    // 无匹配工厂时，回退到纯 Git 克隆（只迁移代码）
    if downloader == nil {
        opts.Wiki = true
        opts.Milestones = false
        opts.Labels = false
        opts.Releases = false
        opts.Comments = false
        opts.Issues = false
        opts.PullRequests = false
        downloader = NewPlainGitDownloader(ownerName, opts.RepoName, opts.CloneAddr)
    }
    
    // 包装重试装饰器
    if setting.Migrations.MaxAttempts > 1 {
        downloader = base.NewRetryDownloader(downloader, ...)
    }
    return downloader, nil
}
```

### 3.2 GitHub 抓取器实现细节

以 `GithubDownloaderV3` 为例（`services/migrations/github.go:65-897`）：

**速率限制处理**：
```go
func (g *GithubDownloaderV3) waitAndPickClient(ctx context.Context) {
    // 1. 选择剩余配额最多的 client（支持多 token 轮询）
    for i := 0; i < len(g.clients); i++ {
        if g.rates[i].Remaining > maxRemaining {
            maxRemaining = g.rates[i].Remaining
            recentIdx = i
        }
    }
    g.curClientIdx = recentIdx
    
    // 2. 配额耗尽时等待重置
    for g.rates[g.curClientIdx].Remaining <= GithubLimitRateRemaining {
        timer := time.NewTimer(time.Until(g.rates[g.curClientIdx].Reset.Time))
        select {
        case <-ctx.Done():
            timer.Stop()
            return
        case <-timer.C:
        }
        g.RefreshRate(ctx)
    }
}
```

**对象抓取方法**：
| 方法 | 功能 | API 端点 |
|------|------|----------|
| `GetRepoInfo` | 获取仓库基本信息 | `GET /repos/{owner}/{repo}` |
| `GetMilestones` | 分页获取里程碑 | `GET /repos/{owner}/{repo}/milestones` |
| `GetLabels` | 获取标签列表 | `GET /repos/{owner}/{repo}/labels` |
| `GetReleases` | 获取发布版本 | `GET /repos/{owner}/{repo}/releases` |
| `GetIssues` | 分页获取 Issues | `GET /repos/{owner}/{repo}/issues` |
| `GetPullRequests` | 分页获取 PR | `GET /repos/{owner}/{repo}/pulls` |
| `GetAllComments` | 批量获取评论 | `GET /repos/{owner}/{repo}/issues/comments` |
| `GetReviews` | 获取 PR 评审 | `GET /repos/{owner}/{repo}/pulls/{n}/reviews` |

### 3.3 安全检查

在 `services/migrations/common.go:32-83` 中对 PR 数据进行安全校验：

```go
func CheckAndEnsureSafePR(pr *base.PullRequest, commonCloneBaseURL string, g base.Downloader) bool {
    // 1. 检查 PatchURL 是否与源站同域，防止开放重定向
    if pr.PatchURL != "" && !hasBaseURL(pr.PatchURL, commonCloneBaseURL) {
        pr.PatchURL = ""
        valid = false
    }
    
    // 2. 检查 Head.CloneURL 是否与源站同域
    if pr.Head.CloneURL != "" && !hasBaseURL(pr.Head.CloneURL, commonCloneBaseURL) {
        pr.Head.CloneURL = ""
        valid = false
    }
    
    // 3. 校验 SHA 格式有效性
    if pr.MergeCommitSHA != "" && !CommitType.IsValid(pr.MergeCommitSHA) {
        pr.MergeCommitSHA = ""
    }
    
    // 4. 校验 Ref 格式有效性
    if pr.Head.Ref != "" && !git.IsValidRefPattern(pr.Head.Ref) {
        pr.Head.Ref = ""
    }
    
    pr.EnsuredSafe = true
    return valid
}
```

---

## 四、核心迁移流程（第三棒：MigrateRepository）

`services/migrations/migrate.go:111-511` 中的 `migrateRepository` 函数是迁移的核心编排器，按严格顺序执行：

### 4.1 执行顺序

```
1. 获取仓库信息 → 2. 创建仓库 + 克隆 Git 数据 → 3. 迁移 Topics
   → 4. 迁移 Milestones → 5. 迁移 Labels → 6. 迁移 Releases
   → 7. 迁移 Issues + 评论 → 8. 迁移 PRs + 评论 + 评审 → 9. 完成
```

**关键代码**：
```go
func migrateRepository(ctx context.Context, doer *user_model.User, 
    downloader base.Downloader, uploader base.Uploader, 
    opts base.MigrateOptions, messenger base.Messenger) error {
    
    // 1. 获取仓库信息
    repo, _ := downloader.GetRepoInfo(ctx)
    repo.IsPrivate = opts.Private
    
    // 2. 创建仓库并克隆 Git 数据
    uploader.CreateRepo(ctx, repo, opts)
    defer uploader.Close()
    
    // 3. 迁移 Topics
    if topics, err := downloader.GetTopics(ctx); err == nil {
        uploader.CreateTopics(ctx, topics...)
    }
    
    // 4. 迁移 Milestones（分批插入）
    if opts.Milestones {
        milestones, _ := downloader.GetMilestones(ctx)
        msBatchSize := uploader.MaxBatchInsertSize("milestone")
        for len(milestones) > 0 {
            uploader.CreateMilestones(ctx, milestones[:msBatchSize]...)
            milestones = milestones[msBatchSize:]
        }
    }
    
    // 5. 迁移 Labels（分批插入）
    if opts.Labels { /* ... */ }
    
    // 6. 迁移 Releases（分批插入 + 同步标签）
    if opts.Releases {
        // ... 分批插入 releases
        uploader.SyncTags(ctx)  // 同步非 release 标签
    }
    
    // 7. 迁移 Issues + 评论
    if opts.Issues {
        for i := 1; ; i++ {
            issues, isEnd, _ := downloader.GetIssues(ctx, i, issueBatchSize)
            uploader.CreateIssues(ctx, issues...)
            
            // 每个 Issue 拉取评论（不支持批量时）
            if opts.Comments && !supportAllComments {
                for _, issue := range issues {
                    comments, _, _ := downloader.GetComments(ctx, issue)
                    uploader.CreateComments(ctx, comments...)
                }
            }
            if isEnd { break }
        }
    }
    
    // 8. 迁移 PRs + 评论 + 评审
    if opts.PullRequests {
        for i := 1; ; i++ {
            prs, isEnd, _ := downloader.GetPullRequests(ctx, i, prBatchSize)
            uploader.CreatePullRequests(ctx, prs...)
            
            // PR 评论
            if opts.Comments { /* ... */ }
            
            // PR 评审
            for _, pr := range prs {
                reviews, _ := downloader.GetReviews(ctx, pr)
                uploader.CreateReviews(ctx, reviews...)
            }
            if isEnd { break }
        }
        uploader.SyncBranches(ctx)  // PR 迁移后同步分支
    }
    
    // 9. 支持批量获取评论时，一次性迁移所有评论
    if opts.Comments && supportAllComments {
        for i := 1; ; i++ {
            comments, isEnd, _ := downloader.GetAllComments(ctx, i, commentBatchSize)
            uploader.CreateComments(ctx, comments...)
            if isEnd { break }
        }
    }
    
    return uploader.Finish(ctx)
}
```

### 4.2 去重机制

由于分页过程中源站数据可能变化（删除），导致重复获取，因此维护已插入索引集合：

```go
mapInsertedIssueIndexes := container.Set[int64]{}
for i := 1; ; i++ {
    issues, isEnd, _ := downloader.GetIssues(ctx, i, issueBatchSize)
    // 过滤已插入的 issue
    for i := 0; i < len(issues); i++ {
        if mapInsertedIssueIndexes.Contains(issues[i].Number) {
            issues = append(issues[:i], issues[i+1:]...)
            i--
            continue
        }
        mapInsertedIssueIndexes.Add(issues[i].Number)
    }
    uploader.CreateIssues(ctx, issues...)
}
```

---

## 五、对象转写（第四棒：GiteaLocalUploader）

`GiteaLocalUploader`（`services/migrations/gitea_uploader.go:43-57`）负责将通用中间格式转换为 Gitea 本地模型并写入数据库。

### 5.1 内部缓存结构

```go
type GiteaLocalUploader struct {
    doer           *user_model.User
    repo           *repo_model.Repository
    labels         map[string]*issues_model.Label     // 标签名 → 本地 Label
    milestones     map[string]int64                   // 里程碑名 → 本地 ID
    issues         map[int64]*issues_model.Issue      // issue 编号 → 本地 Issue
    userMap        map[int64]int64                    // 外部用户 ID → 本地用户 ID
    prCache        map[int64]*issues_model.PullRequest
    prHeadCache    map[string]string
}
```

### 5.2 典型对象转写流程

**Milestone 转写示例**（`services/migrations/gitea_uploader.go:180-228`）：
```go
func (g *GiteaLocalUploader) CreateMilestones(ctx context.Context, milestones ...*base.Milestone) error {
    mss := make([]*issues_model.Milestone, 0, len(milestones))
    for _, milestone := range milestones {
        // 字段映射：base.Milestone → issues_model.Milestone
        ms := issues_model.Milestone{
            RepoID:        g.repo.ID,
            Name:          milestone.Title,
            Content:       milestone.Description,
            IsClosed:      milestone.State == "closed",
            CreatedUnix:   timeutil.TimeStamp(milestone.Created.Unix()),
            DeadlineUnix:  deadline,
        }
        if ms.IsClosed && milestone.Closed != nil {
            ms.ClosedDateUnix = timeutil.TimeStamp(milestone.Closed.Unix())
        }
        mss = append(mss, &ms)
    }
    
    // 批量插入
    issues_model.InsertMilestones(ctx, mss...)
    
    // 缓存映射，供后续 issue 使用
    for _, ms := range mss {
        g.milestones[ms.Name] = ms.ID
    }
    return nil
}
```

**Issue 转写示例**（`services/migrations/gitea_uploader.go:380-461`）：
```go
func (g *GiteaLocalUploader) CreateIssues(ctx context.Context, issues ...*base.Issue) error {
    iss := make([]*issues_model.Issue, 0, len(issues))
    for _, issue := range issues {
        // 通过缓存查找关联的 Label 和 Milestone
        var labels []*issues_model.Label
        for _, label := range issue.Labels {
            if lb, ok := g.labels[label.Name]; ok {
                labels = append(labels, lb)
            }
        }
        milestoneID := g.milestones[issue.Milestone]
        
        // 用户映射
        is := issues_model.Issue{ /* ... 字段映射 ... */ }
        g.remapUser(ctx, issue, &is)  // 核心：权限映射
        
        // 表情反应映射
        for _, reaction := range issue.Reactions {
            res := issues_model.Reaction{Type: reaction.Content}
            g.remapUser(ctx, reaction, &res)
            is.Reactions = append(is.Reactions, &res)
        }
        iss = append(iss, &is)
    }
    
    issues_model.InsertIssues(ctx, iss...)
    // 缓存 issue 映射，供后续评论使用
    for _, is := range iss {
        g.issues[is.Index] = is
    }
    return nil
}
```

---

## 六、权限映射（第五棒：remapUser）

用户身份映射是迁移中最关键的权限相关逻辑，由 `remapUser` 函数处理（`services/migrations/gitea_uploader.go:972-1022`）。

### 6.1 映射策略

```go
func (g *GiteaLocalUploader) remapUser(ctx context.Context, 
    source user_model.ExternalUserMigrated, 
    target user_model.ExternalUserRemappable) error {
    
    var userID int64
    if g.sameApp {
        // 同实例迁移：直接用 ID 映射
        userID, _ = g.remapLocalUser(ctx, source)
    } else {
        // 跨实例迁移：通过外部用户 ID 查找
        userID, _ = g.remapExternalUser(ctx, source)
    }
    
    if userID > 0 {
        // 找到匹配用户 → 使用真实用户 ID
        return target.RemapExternalUser("", 0, userID)
    }
    // 未找到匹配用户 → 回退到执行者（doer），保留原始名称
    return target.RemapExternalUser(source.GetExternalName(), source.GetExternalID(), g.doer.ID)
}
```

### 6.2 同实例迁移（remapLocalUser）

```go
func (g *GiteaLocalUploader) remapLocalUser(ctx context.Context, source user_model.ExternalUserMigrated) (int64, error) {
    userid, ok := g.userMap[source.GetExternalID()]
    if !ok {
        user, err := user_model.GetUserByID(ctx, source.GetExternalID())
        if errors.Is(err, util.ErrNotExist) {
            // 用户不存在 → 映射为 0（后续用 doer 替代）
            userid = 0
        } else if user != nil {
            // 验证用户名一致性
            if !util.AsciiEqualFold(user.Name, source.GetExternalName()) {
                userid = 0  // 用户名不匹配 → 不直接映射
            } else {
                userid = source.GetExternalID()
            }
        }
        g.userMap[source.GetExternalID()] = userid
    }
    return userid, nil
}
```

### 6.3 跨实例迁移（remapExternalUser）

```go
func (g *GiteaLocalUploader) remapExternalUser(ctx context.Context, source user_model.ExternalUserMigrated) (int64, error) {
    userid, ok := g.userMap[source.GetExternalID()]
    if !ok {
        // 通过 external_user 表查找关联用户
        userid, err = user_model.GetUserIDByExternalUserID(
            ctx, 
            g.gitServiceType.Name(), 
            strconv.FormatInt(source.GetExternalID(), 10),
        )
        g.userMap[source.GetExternalID()] = userid
    }
    return userid, nil
}
```

### 6.4 回退机制

当无法匹配到本地用户时，系统采用以下策略：
1. **操作人**：使用执行迁移的用户（doer）ID
2. **显示信息**：保留原始用户的名称和外部 ID
3. **数据完整性**：所有内容（评论、PR、Issue 等）都会被保留，只是归属到 doer

这确保了即使外部用户在本地 Gitea 不存在，迁移也能完成且数据不丢失。

---

## 七、接力总结

整个迁移流程是一个精确的五棒接力：

| 阶段 | 执行者 | 核心职责 | 关键文件 |
|------|--------|----------|----------|
| **第一棒** | `runMigrateTask` | 任务生命周期管理、状态流转、取消监听 | `services/task/migrate.go` |
| **第二棒** | `newDownloader` + 具体 Downloader | 工厂选择、源站 API 抓取、速率控制、安全校验 | `services/migrations/migrate.go`<br>`services/migrations/github.go` |
| **第三棒** | `migrateRepository` | 流程编排、分批处理、去重、进度通知 | `services/migrations/migrate.go` |
| **第四棒** | `GiteaLocalUploader` | 对象转写、缓存维护、分批写入 | `services/migrations/gitea_uploader.go` |
| **第五棒** | `remapUser` | 用户身份映射、权限关联、回退策略 | `services/migrations/gitea_uploader.go:972-1022` |

### 7.1 关键设计亮点

1. **接口抽象**：Downloader/Uploader 接口使新增源站类型只需实现对应工厂
2. **分批处理**：所有批量操作都支持分批，避免大仓库内存溢出
3. **缓存机制**：Uploader 内部维护多级缓存，减少数据库查询
4. **安全防护**：URL 白名单、SHA 校验、Ref 格式校验多层安全
5. **优雅降级**：用户映射失败时回退到 doer，保证迁移不中断
6. **可观测性**：进度通过 messenger 实时更新，错误保留在任务记录中

### 7.2 失败处理

- 迁移失败时，仓库记录保留，状态为失败，用户可见错误信息
- `uploader.Rollback()` 尝试清理不完整的仓库数据
- `system_model.CreateRepositoryNotice` 记录系统级通知

---

## 八、核心数据流向图

```
用户提交迁移请求
    ↓
[Web/API] 创建 admin.Task（Status=Queued）
    ↓
[任务调度器] 拉取任务 → runMigrateTask
    ↓
┌─────────────────────────────────────────┐
│  migrations.MigrateRepository           │
│  ┌─────────┐    ┌──────────────────┐    │
│  │ newDownloader ─→ GithubDownloader │    │
│  └─────────┘    └──────────────────┘    │
│         ↓ 调用 GetXXX 方法               │
│  ┌──────────────────────────────────┐   │
│  │ GiteaLocalUploader               │   │
│  │  CreateRepo → 克隆 Git 数据       │   │
│  │  CreateMilestones                │   │
│  │  CreateLabels                    │   │
│  │  CreateIssues + remapUser        │   │
│  │  CreateComments                  │   │
│  │  CreatePullRequests              │   │
│  │  CreateReviews                   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
    ↓
更新 Task 状态（Success/Failed）
    ↓
发送通知事件
```

通过以上五棒接力，Gitea 实现了从外部站点到本地的完整仓库迁移，兼顾了灵活性、安全性和可靠性。
