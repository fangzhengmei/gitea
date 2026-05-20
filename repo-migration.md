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

### 2.1 任务入口与失败收敛全链路

迁移任务由 `services/task/migrate.go` 中的 `runMigrateTask` 函数驱动，该函数构建了完整的**失败收敛闭环**。

**任务状态流转**：
```
TaskStatusQueued → TaskStatusRunning → TaskStatusSuccess
                          ↓
                     TaskStatusFailed
```

#### 2.1.1 核心执行流程

```go
func runMigrateTask(ctx context.Context, t *admin_model.Task) (err error) {
    // ========== 【第一关：Panic 恢复 + 失败兜底】 ==========
    defer func(ctx context.Context) {
        // 1. Panic 捕获与恢复
        if e := recover(); e != nil {
            err = fmt.Errorf("PANIC whilst trying to do migrate task: %v", e)
            log.Error("PANIC during runMigrateTask[%d] ... Stacktrace: %v", t.ID, e, log.Stack(2))
        }
        
        // 2. 成功路径
        if err == nil {
            err = admin_model.FinishMigrateTask(ctx, t)
            if err == nil {
                notify_service.MigrateRepository(ctx, t.Doer, t.Owner, t.Repo)
                return
            }
        }
        
        // ========== 【失败状态回写】 ==========
        log.Error("runMigrateTask[%d] failed: %v", t.ID, err)
        
        t.EndTime = timeutil.TimeStampNow()
        t.Status = structs.TaskStatusFailed
        t.Message = err.Error()  // 此时 err 已经过脱敏处理
        t.UpdateCols(ctx, "status", "message", "end_time")
        
        // 关键注释：do not delete the repository, otherwise the users won't be able to see the last error
    }(graceful.GetManager().ShutdownContext())
    
    // 加载关联数据
    t.LoadRepo(ctx)
    t.LoadDoer(ctx)
    t.LoadOwner(ctx)
    opts, _ := t.MigrateConfig()
    
    // 更新任务状态为运行中
    t.Status = structs.TaskStatusRunning
    t.UpdateCols(ctx, "start_time", "status")
    
    // 启动取消监听协程（每2秒检查一次任务状态）
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
    
    // ========== 【调用核心迁移函数】 ==========
    t.Repo, err = migrations.MigrateRepository(ctx, t.Doer, t.Owner.Name, *opts, messenger)
    
    if err == nil {
        return nil  // 成功路径，defer 中完成收尾
    }
    
    // ========== 【第二关：错误脱敏与归一化】 ==========
    
    // 1. 错误脱敏：移除 URL 中的凭证信息
    err = util.SanitizeErrorCredentialURLs(err)
    
    // 2. 鉴权失败归一
    if strings.Contains(err.Error(), "Authentication failed") ||
       strings.Contains(err.Error(), "could not read Username") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    
    // 3. Git 致命错误归一
    if strings.Contains(err.Error(), "fatal:") {
        return fmt.Errorf("migration failed: %w", err)
    }
    
    // 4. 仓库创建相关错误归一
    err = handleCreateError(t.Owner, err)
    
    return err  // 最终 err 会被 defer 函数写入 Task.Message
}
```

#### 2.1.2 失败收敛四层防御

| 层级 | 机制 | 代码位置 | 作用 |
|------|------|---------|------|
| **第一层** | Panic 恢复 | `defer` 中的 `recover()` | 捕获代码 panic，避免进程崩溃，转为普通 error |
| **第二层** | 错误脱敏 | `util.SanitizeErrorCredentialURLs(err)` | 移除错误信息中的密码、token 等敏感信息 |
| **第三层** | 错误归一 | 鉴权失败 / Git fatal / 创建错误匹配 | 将底层错误转换为用户友好的统一格式 |
| **第四层** | 状态兜底 | `defer` 中的状态回写 | 无论何种退出路径（正常 return / panic），都保证任务状态被更新 |

#### 2.1.3 深度校验：错误归一是否掩盖底层故障定位？

**结论**：**不会掩盖**。Gitea 采用了"**双轨记录**"策略，兼顾用户友好性和运维可观测性。

让我们分析具体实现：

```go
// ========== 用户可见的错误（经过归一化） ==========
// 1. 错误脱敏：移除 URL 中的凭证
err = util.SanitizeErrorCredentialURLs(err)

// 2. 错误类型归一（用户看到的是这些消息）
if strings.Contains(err.Error(), "Authentication failed") {
    return fmt.Errorf("authentication failed: %w", err)  // 用 %w 包装，保留原始错误链
} else if strings.Contains(err.Error(), "fatal:") {
    return fmt.Errorf("migration failed: %w", err)       // 用 %w 包装，保留原始错误链
}
err = handleCreateError(t.Owner, err)

// ========== 运维可见的错误（完整原始信息） ==========
// 在 defer 函数中：
log.Error("runMigrateTask[%d] by DoerID[%d] to RepoID[%d] for OwnerID[%d] failed: %v", 
    t.ID, t.DoerID, t.RepoID, t.OwnerID, err)  // 这里记录的 err 包含完整错误链
```

**双轨记录策略详解**：

| 维度 | 用户可见（Task.Message） | 运维可见（服务日志） |
|------|-------------------------|---------------------|
| **错误内容** | 归一化后的友好消息，如 `authentication failed: ...` | 完整原始错误，包含堆栈、底层错误详情 |
| **敏感信息** | 已脱敏（`SanitizeErrorCredentialURLs`） | 已脱敏（`SanitizeErrorCredentialURLs`） |
| **错误链** | 通过 `%w` 包装保留，可通过 `errors.Unwrap()` 追溯 | 完整保留 |
| **存储位置** | 数据库 `task.message` 字段，Web 界面可见 | 服务日志文件，仅运维可查 |
| **详细程度** | 中等，足够用户理解失败原因 | 完整，包含文件名、行号、堆栈 |

**关键技术点**：
1. **`%w` 错误包装**：使用 `fmt.Errorf("...: %w", err)` 而非 `%v`，保留错误链
2. **`SanitizeErrorCredentialURLs` 实现**：返回 `sanitizedError{err: err}`，`Error()` 方法返回脱敏后的字符串，但内部保留原始 error
3. **日志记录时机**：`log.Error` 在 defer 中调用，此时 err 已经过 `%w` 包装，包含完整上下文

**代码证据**（`modules/util/sanitize.go:25-27`）：
```go
func SanitizeErrorCredentialURLs(err error) error {
    return sanitizedError{err: err}  // 内部持有原始 error，不丢失
}
```

**运维定位流程**：
1. 用户报告迁移失败，看到归一化错误消息
2. 运维根据 Task ID 在日志中搜索 `runMigrateTask[12345] failed`
3. 日志中包含完整错误链：`authentication failed: clone failed: HTTP 401: invalid token`
4. 可通过 `errors.As()` / `errors.Is()` 进一步提取特定错误类型

这种设计既保证了用户界面的友好性，又不丢失运维所需的故障定位信息。

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
    
    // 无匹配工厂时，回退到纯 Git 克隆（只迁移代码 + Wiki）
    if downloader == nil {
        // ========== Plain-Git 降级：被关闭的对象迁移范围 ==========
        opts.Wiki = true              // ✅ 保留：Wiki 迁移
        opts.Milestones = false       // ❌ 关闭：里程碑
        opts.Labels = false           // ❌ 关闭：标签
        opts.Releases = false         // ❌ 关闭：发布版本
        opts.Comments = false         // ❌ 关闭：评论
        opts.Issues = false           // ❌ 关闭：Issues
        opts.PullRequests = false     // ❌ 关闭：Pull Requests
        // =========================================================
        
        downloader = NewPlainGitDownloader(ownerName, opts.RepoName, opts.CloneAddr)
        log.Trace("Will migrate from git: %s", opts.OriginalURL)
    }
    
    // ========== Plain-Git 降级：可恢复性与数据完整性的取舍 ==========
    // 取舍原则：宁可丢失协作数据，也要保证 Git 代码本身能迁移成功
    //
    // ✅ 可恢复性增强（保留的能力）：
    // 1. Git 完整历史：所有分支、标签、提交历史都保留
    // 2. Wiki 数据：如果源站支持 Wiki，也会被克隆
    // 3. 零外部依赖：不依赖源站 API，只需要 Git 协议可用
    // 4. 成功率最高：Git 协议是最基础、最兼容的能力
    //
    // ❌ 数据完整性牺牲（丢失的能力）：
    // 1. 所有 Issue/PR 元数据：标题、内容、状态、标签、里程碑
    // 2. 所有评论：Issue 评论、PR 评论、代码评审评论
    // 3. 所有发布版本：Release 标题、说明、附件
    // 4. 用户关联：Issue/PR 的创建者、评论者都无法映射
    // 5. 评审历史：PR 的审批状态、评审意见
    //
    // 设计意图：当源站类型无法识别或 API 不可用时，
    // 确保至少能拿到最核心的资产——Git 代码库本身。
    // =========================================================
    
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

### 3.3 安全防护体系

Gitea 构建了多层安全防护体系，确保迁移过程的安全性。

#### 3.3.1 第一层：URL 白名单校验

迁移开始前，首先通过 `IsMigrateURLAllowed` 函数（`services/migrations/migrate.go:44-87`）对克隆地址进行严格校验：

```go
func IsMigrateURLAllowed(remoteURL string, doer *user_model.User) error {
    u, err := url.Parse(remoteURL)
    
    // 1. 本地文件系统访问控制
    if u.Scheme == "file" || u.Scheme == "" {
        if !doer.CanImportLocal() {  // 检查用户是否有权限导入本地路径
            return &git.ErrInvalidCloneAddr{IsPermissionDenied: true}
        }
        // 验证是绝对路径且为目录
        isAbs := filepath.IsAbs(u.Host + u.Path)
        isDir, _ := util.IsDir(u.Host + u.Path)
        if !isAbs || !isDir {
            return &git.ErrInvalidCloneAddr{IsInvalidPath: true}
        }
        return nil
    }
    
    // 2. 协议校验：只允许 http/https/git
    if u.Scheme != "http" && u.Scheme != "https" && u.Scheme != "git" {
        return &git.ErrInvalidCloneAddr{IsProtocolInvalid: true}
    }
    
    // 3. Git 协议注入防护：阻止 CRLF 注入
    if u.Scheme == "git" && u.Port() != "" && 
       (strings.Contains(remoteURL, "%0d") || strings.Contains(remoteURL, "%0a")) {
        return &git.ErrInvalidCloneAddr{IsURLError: true}
    }
    
    // 4. 主机名 + IP 白名单/黑名单校验
    hostName, _, _ := net.SplitHostPort(u.Host)
    addrList, _ := net.LookupIP(hostName)  // 解析 IP 进行双重校验
    return checkByAllowBlockList(hostName, addrList)
}
```

**黑白名单校验逻辑**（`services/migrations/migrate.go:89-108`）：
```go
func checkByAllowBlockList(hostName string, addrList []net.IP) error {
    // 1. 检查 IP 是否在白/黑名单中
    for _, addr := range addrList {
        ipAllowed = ipAllowed || allowList.MatchIPAddr(addr)
        ipBlocked = ipBlocked || blockList.MatchIPAddr(addr)
    }
    
    // 2. 黑名单优先：主机名或 IP 任一被阻止即拒绝
    if blockList.MatchHostName(hostName) || ipBlocked {
        blockedError = &git.ErrInvalidCloneAddr{IsPermissionDenied: true}
    }
    
    // 3. 白名单校验（如果配置了白名单）
    if !allowList.IsEmpty() {
        if !allowList.MatchHostName(hostName) && !ipAllowed {
            return &git.ErrInvalidCloneAddr{IsPermissionDenied: true}
        }
    }
    return blockedError
}
```

**白名单初始化**（`services/migrations/migrate.go:514-531`）：
```go
func Init() error {
    blockList = hostmatcher.ParseSimpleMatchList("migrations.BLOCKED_DOMAINS", setting.Migrations.BlockedDomains)
    allowList = hostmatcher.ParseSimpleMatchList("migrations.ALLOWED_DOMAINS/ALLOW_LOCALNETWORKS", setting.Migrations.AllowedDomains)
    
    if allowList.IsEmpty() {
        allowList.AppendBuiltin(hostmatcher.MatchBuiltinExternal)  // 默认允许外部网络
    }
    if setting.Migrations.AllowLocalNetworks {
        allowList.AppendBuiltin(hostmatcher.MatchBuiltinPrivate)   // 允许私有网络
        allowList.AppendBuiltin(hostmatcher.MatchBuiltinLoopback)  // 允许回环地址
    }
    return nil
}
```

#### 3.3.2 第二层：克隆 URL 二次校验

在获取仓库信息后，还会对 Downloader 返回的 CloneURL 进行二次校验（`services/migrations/migrate.go:200-219`）：

```go
// SECURITY: If the downloader is not a RepositoryRestorer then we need to recheck the CloneURL
if _, ok := downloader.(*RepositoryRestorer); !ok {
    // 重新校验 CloneURL（Downloader 可能重写了 URL）
    if err := IsMigrateURLAllowed(repo.CloneURL, doer); err != nil {
        return err
    }
    
    // 防止从外部 URL 重定向到本地文件系统
    cloneAddrURL, _ := url.Parse(opts.CloneAddr)
    cloneURL, _ := url.Parse(repo.CloneURL)
    if cloneURL.Scheme == "file" && cloneAddrURL.Scheme != "file" {
        return errors.New("repo info has changed from external to local filesystem")
    }
}
```

**RepositoryRestorer 例外的因果关系**：

| 问题 | 说明 |
|------|------|
| **为什么 RepositoryRestorer 不需要二次校验？** | `RepositoryRestorer` 是从**本地备份目录**恢复仓库（`services/migrations/restore.go:19-39`），它返回的 `CloneURL` 是本地路径：`filepath.Join(r.baseDir, "git")`。这个路径已经在创建时验证过，且不涉及外部网络请求，不存在被重定向或篡改的风险。 |
| **为什么其他 Downloader 需要二次校验？** | 外部 Downloader（如 GitHub/GitLab）的 `GetRepoInfo()` 可能返回被源站重写的 CloneURL。例如：<br>1. 用户输入 `https://github.com/user/repo` <br>2. GitHub API 可能返回 `https://oauth2:xxx@github.com/user/repo.git` <br>3. 甚至可能被恶意源站重定向到 `file:///etc/passwd` <br>因此必须重新校验最终的 CloneURL |
| **校验了什么？** | 1. 通过 `IsMigrateURLAllowed` 再次走完整的白名单校验流程<br>2. 防止协议降级：从 http/https 变为 file 协议（SSRF 防护） |

#### 3.3.2.1 深度校验：RepositoryRestorer 与普通 Downloader 的输入信任边界差异

RepositoryRestorer 和普通 Downloader（如 GithubDownloaderV3）处于完全不同的信任边界，这决定了它们的安全策略差异：

| 维度 | RepositoryRestorer | 普通 Downloader（GitHub/GitLab 等） |
|------|-------------------|-----------------------------------|
| **输入来源** | 本地文件系统路径（`baseDir`） | 外部网络 URL（`CloneAddr`） |
| **调用入口** | 内部 `dump.go:720` 直接创建，绕过 `newDownloader` 工厂 | 外部 API/Web 调用，经 `newDownloader` 工厂创建 |
| **前置校验** | `NewRepositoryRestorer` 中调用 `filepath.Abs()` 验证路径 | `IsMigrateURLAllowed` 完整校验（协议/主机/IP 黑白名单） |
| **CloneURL 来源** | 静态拼接：`filepath.Join(r.baseDir, "git")` | 外部 API 返回，可能被源站重写 |
| **PatchURL 来源** | 静态拼接：`"file://" + filepath.Join(r.baseDir, pr.PatchURL)` | 外部 API 返回，可能指向任意 URL |
| **用户权限** | 仅管理员可调用（`user_model.GetAdminUser(ctx)`） | 普通登录用户即可调用 |
| **数据可信性** | 从 Gitea 自身导出的备份格式，数据结构可控 | 第三方 API 返回，数据结构和内容不可控 |

**关键代码证据**（`services/migrations/dump.go:715-737`）：
```go
// 仅管理员可调用恢复功能
doer, err := user_model.GetAdminUser(ctx)
if err != nil {
    return err
}

// 直接创建 RepositoryRestorer，绕过 newDownloader 工厂
downloader, err := NewRepositoryRestorer(ctx, baseDir, ownerName, repoName, validation)
if err != nil {
    return err
}

// 直接调用 migrateRepository，不走 MigrateRepository（跳过 IsMigrateURLAllowed）
if err = migrateRepository(ctx, doer, downloader, uploader, migrateOpts, nil); err != nil {
    // ...
}
```

**信任边界总结**：
- RepositoryRestorer 处于**高信任边界**内：管理员操作 + 本地文件 + 已知导出格式
- 普通 Downloader 处于**低信任边界**：普通用户 + 外部网络 + 不可控 API 返回
- 这就是为什么 RepositoryRestorer 可以跳过 CloneURL 二次校验，而普通 Downloader 必须严格校验

#### 3.3.3 第三层：PR 数据安全校验

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
| **第一棒** | `runMigrateTask` | 任务生命周期管理、Panic 恢复、错误脱敏归一、状态兜底 | `services/task/migrate.go` |
| **第二棒** | `newDownloader` + 具体 Downloader | 工厂选择、Plain-Git 降级、源站 API 抓取、速率控制 | `services/migrations/migrate.go`<br>`services/migrations/github.go` |
| **第三棒** | `migrateRepository` | 流程编排、URL 二次校验（RepositoryRestorer 例外）、分批处理、去重 | `services/migrations/migrate.go` |
| **第四棒** | `GiteaLocalUploader` | 对象转写、缓存维护、分批写入、Rollback（仅关闭句柄） | `services/migrations/gitea_uploader.go` |
| **第五棒** | `remapUser` | 用户身份映射、权限关联、失败回退 doer | `services/migrations/gitea_uploader.go:972-1022` |

### 7.1 关键设计亮点

1. **接口抽象**：Downloader/Uploader 接口使新增源站类型只需实现对应工厂
2. **分批处理**：所有批量操作都支持分批，避免大仓库内存溢出
3. **缓存机制**：Uploader 内部维护多级缓存，减少数据库查询
4. **多层安全防护**：
   - URL 白名单校验（协议、主机名、IP 双重校验）
   - CloneURL 二次校验（防止重定向攻击，RepositoryRestorer 例外）
   - PR 数据校验（PatchURL/HeadCloneURL 同域、SHA/Ref 格式）
5. **优雅降级**：
   - 无匹配 Downloader 时降级为 Plain-Git 模式（仅代码 + Wiki）
   - 用户映射失败时回退到 doer，保证迁移不中断
6. **失败收敛闭环**：
   - Panic 捕获恢复，避免进程崩溃
   - 错误脱敏（移除凭证）+ 归一化（统一错误格式）
   - defer 兜底，无论何种退出路径都更新任务状态
7. **失败不删库**：Rollback() 仅关闭 Git 句柄，不删除任何数据，确保用户可见错误信息
8. **可观测性**：进度通过 messenger 实时更新，错误保留在任务记录中

### 7.2 失败处理

#### 7.2.1 核心原则：失败不删库

代码中有两处关键注释明确了这一设计决策：
1. `services/task/migrate.go:69`: `// then, do not delete the repository, otherwise the users won't be able to see the last error`
2. `services/migrations/gitea_uploader.go:948`: `// do not delete the repository, otherwise the end users won't be able to see the last error message`

#### 7.2.2 Rollback() 的真实行为

**重要更正**：`uploader.Rollback()` **并不会清理仓库数据**。让我们看实际代码（`services/migrations/gitea_uploader.go:944-951`）：

```go
func (g *GiteaLocalUploader) Rollback() error {
    if g.repo != nil && g.repo.ID > 0 {
        g.gitRepo.Close()  // 仅关闭 Git 仓库句柄，不删除任何数据
        
        // 明确注释：do not delete the repository
    }
    return nil  // 永远返回 nil，不做任何清理
}
```

**Rollback() 实际只做了一件事**：关闭 `gitRepo` 文件句柄，释放资源。仓库目录、数据库记录、已写入的 Issue/PR 等数据都会被完整保留。

#### 7.2.3 完整失败处理链路

```
迁移失败
    ↓
1. MigrateRepository 中调用 uploader.Rollback() → 仅关闭 Git 句柄
    ↓
2. 创建系统通知：system_model.CreateRepositoryNotice(...)
    ↓
3. 错误向上返回给 runMigrateTask
    ↓
4. runMigrateTask 的 defer 函数：
   - 错误脱敏（SanitizeErrorCredentialURLs）
   - 错误归一（鉴权失败 / Git fatal 等）
   - 更新 Task 状态为 Failed，写入错误信息
   - 不删除仓库
    ↓
5. 用户在界面看到：仓库存在 + 状态为失败 + 可查看具体错误信息
```

#### 7.2.4 设计意图

这种设计的好处是：
- **可追溯**：用户总能看到最后一条错误信息，便于排查问题
- **可恢复**：已迁移的部分数据不会丢失，理论上可以基于已写入的数据继续迁移
- **避免误删**：防止因临时网络波动等原因导致已迁移的大量数据被误删

代价是可能留下不完整的仓库，需要用户手动判断是否删除。

---

## 八、核心数据流向图

```
用户提交迁移请求
    ↓
[Web/API] 创建 admin.Task（Status=Queued）
    ↓
[任务调度器] 拉取任务 → runMigrateTask
    │
    ├─► defer 注册 Panic 恢复 + 状态兜底
    ├─► 加载 Repo/Doer/Owner 数据
    ├─► 启动取消监听协程（每 2s 轮询）
    ↓
┌───────────────────────────────────────────────────────────┐
│  migrations.MigrateRepository                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ newDownloader                                       │  │
│  │  ├─ 匹配 GitServiceType → GithubDownloader          │  │
│  │  └─ 无匹配 → Plain-Git 降级（仅代码+Wiki，关闭其他） │  │
│  └─────────────────────────────────────────────────────┘  │
│                           ↓                               │
│  1. GetRepoInfo → CloneURL 二次校验（RepositoryRestorer 跳过）│
│  2. CreateRepo → 克隆 Git 数据                            │
│  3. CreateTopics / Milestones / Labels / Releases        │
│  4. CreateIssues + Comments（每批 remapUser）             │
│  5. CreatePullRequests + Reviews（SHA/Ref 格式校验）      │
│                           ↓                               │
│  成功 → uploader.Finish() / 失败 → Rollback（仅关闭句柄）  │
└───────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│ runMigrateTask defer 执行               │
│  ├─ 成功：FinishMigrateTask + 通知       │
│  └─ 失败：                               │
│      ├─ SanitizeErrorCredentialURLs     │
│      ├─ 错误归一（鉴权/Git fatal/创建）  │
│      ├─ 更新 Task.Status = Failed       │
│      └─ 不删除仓库（用户可见错误）        │
└─────────────────────────────────────────┘
```

## 九、关键事实速查表

| 说法 | 事实 | 代码依据 |
|------|------|---------|
| ❌ Rollback() 会清理仓库数据 | ✅ Rollback() 仅关闭 Git 句柄，不删除任何数据 | `gitea_uploader.go:944-951` |
| ❌ RepositoryRestorer 也需要 CloneURL 校验 | ✅ 本地恢复场景跳过二次校验（路径已验证） | `migrate.go:201` |
| ❌ Plain-Git 降级只迁移代码 | ✅ 保留 Wiki，关闭 Milestones/Labels/Issues/PRs/Comments/Releases | `migrate.go:169-176` |
| ✅ 失败时保留仓库 | ✅ 两处代码注释明确：do not delete the repository | `task/migrate.go:69`<br>`gitea_uploader.go:948` |
| ✅ 用户映射失败回退 doer | ✅ 未匹配用户时，操作人用 doer ID，保留原始名称 | `gitea_uploader.go:984-987` |
| ✅ Panic 会被捕获 | ✅ defer 中 recover()，转为 error 写入任务 | `task/migrate.go:45-49` |
| ✅ 错误信息会脱敏 | ✅ SanitizeErrorCredentialURLs 移除 URL 中的凭证 | `task/migrate.go:143` |
| ❌ 错误归一掩盖底层故障 | ✅ 双轨记录：用户看到归一化消息，日志保留完整错误链 | `task/migrate.go:145-156` |
| ✅ RepositoryRestorer 仅管理员可用 | ✅ NewRepositoryRestorer 调用前先 `GetAdminUser(ctx)` | `dump.go:715` |
| ✅ Plain-Git 是优雅降级 | ✅ 宁可丢失协作数据，也要保证 Git 代码本身迁移成功 | `migrate.go:167-176` |

通过以上五棒接力，Gitea 实现了从外部站点到本地的完整仓库迁移，兼顾了灵活性、安全性和可靠性。
