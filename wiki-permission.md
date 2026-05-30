# Wiki 存储与权限边界分析

## 1. 核心架构概览

Gitea Wiki 采用 **Git 仓库作为存储后端**，**单元权限系统**作为访问控制，**Git 提交历史**作为版本回溯机制，三者通过统一的路径转换层和服务层衔接。

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  路由层 (Router)│────▶│  服务层 (Service)│────▶│  存储层 (Git)   │
│  - 权限中间件   │     │  - wiki.go      │     │  - .wiki.git   │
│  - 参数绑定     │     │  - wiki_path.go │     │  - Git 命令    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
          │                        │                        │
          ▼                        ▼                        ▼
  ┌───────────────┐      ┌───────────────┐        ┌───────────────┐
  │ 权限校验      │      │ 路径转换      │        │ 历史回溯      │
  │ CanRead/Write │      │ WebPath ↔ GitPath │    │ Git log/rev-list │
  └───────────────┘      └───────────────┘        └───────────────┘
```

## 2. 默认分支来源与同步机制

### 2.1 数据库字段

默认分支存储在 `repository` 表的 `default_wiki_branch` 列中 (`models/repo/repo.go:166`)：

```go
type Repository struct {
    // ...
    DefaultBranch       string
    DefaultWikiBranch   string  // 独立于代码仓库的默认分支
    // ...
}
```

### 2.2 初始值与回退

当 `DefaultWikiBranch` 为空时，回退到全局配置 (`models/repo/repo.go:326-328`)：

```go
func (repo *Repository) LoadAttributes() {
    // ...
    if repo.DefaultWikiBranch == "" {
        repo.DefaultWikiBranch = setting.Repository.DefaultBranch
    }
}
```

数据库迁移 v289 为旧仓库补设初始值 (`models/migrations/v1_22/v289.go:23`)：

```go
_, err := x.Exec("UPDATE `repository` SET default_wiki_branch = 'master' WHERE (default_wiki_branch IS NULL) OR (default_wiki_branch = '')")
```

**结论**：旧仓库默认分支为 `master`，新仓库默认分支由 `setting.Repository.DefaultBranch`（通常为 `main`）决定。

### 2.3 分支同步机制（仅 Web 端）

Web 端的 `findWikiRepoCommit` 在访问 Wiki 时会检测数据库记录与 Git 仓库的默认分支是否一致，不一致则自动修正数据库 (`routers/web/repo/wiki.go:98-125`)：

```go
func findWikiRepoCommit(ctx *context.Context) (*git.Repository, *git.Commit, error) {
    wikiGitRepo, errGitRepo := gitrepo.RepositoryFromRequestContextOrOpen(ctx, ctx.Repo.Repository.WikiStorageRepo())
    // ...
    commit, errCommit := wikiGitRepo.GetBranchCommit(ctx.Repo.Repository.DefaultWikiBranch)
    if git.IsErrNotExist(errCommit) {
        // 数据库记录的分支在 Git 仓库中不存在 → 从 Git 仓库读取真实默认分支
        gitRepoDefaultBranch, errBranch := gitrepo.GetDefaultBranch(ctx, ctx.Repo.Repository.WikiStorageRepo())
        // ...
        // 回写数据库
        errDb := repo_model.UpdateRepositoryColsNoAutoTime(ctx, &repo_model.Repository{
            ID: ctx.Repo.Repository.ID,
            DefaultWikiBranch: gitRepoDefaultBranch,
        }, "default_wiki_branch")
        // 更新内存中的值
        ctx.Repo.Repository.DefaultWikiBranch = gitRepoDefaultBranch
        // 用修正后的分支重试
        commit, errCommit = wikiGitRepo.GetBranchCommit(ctx.Repo.Repository.DefaultWikiBranch)
    }
    return wikiGitRepo, commit, nil
}
```

**关键差异**：API 端的 `findWikiRepoCommit` (`routers/api/v1/repo/wiki.go:474-495`) **没有此同步逻辑**，直接使用数据库中的 `DefaultWikiBranch`。如果数据库记录与 Git 仓库实际默认分支不一致（例如通过 Git 命令行直接修改了 Wiki 仓库的默认分支），API 端将返回 404 而非自动修正。

### 2.4 分支变更入口

通过 `ChangeDefaultWikiBranch` (`services/wiki/wiki.go:382-410`) 修改，操作原子性由数据库事务保证：

```go
func ChangeDefaultWikiBranch(ctx context.Context, repo *repo_model.Repository, newBranch string) error {
    if !git.IsValidRefPattern(newBranch) {
        return fmt.Errorf("invalid branch name: %s", newBranch)
    }
    return db.WithTx(ctx, func(ctx context.Context) error {
        repo.DefaultWikiBranch = newBranch
        if err := repo_model.UpdateRepositoryColsNoAutoTime(ctx, repo, "default_wiki_branch"); err != nil {
            return fmt.Errorf("unable to update database: %w", err)
        }
        if !repo_service.HasWiki(ctx, repo) {
            return nil
        }
        oldDefBranch, err := gitrepo.GetDefaultBranch(ctx, repo.WikiStorageRepo())
        if err != nil { return fmt.Errorf("unable to get default branch: %w", err) }
        if oldDefBranch == newBranch { return nil }
        err = gitrepo.RenameBranch(ctx, repo.WikiStorageRepo(), oldDefBranch, newBranch)
        if err != nil { return fmt.Errorf("unable to rename default branch: %w", err) }
        return nil
    })
}
```

此函数同时：1) 更新数据库字段；2) 在 Git 仓库中重命名分支。两者在同一事务中完成。

## 3. 页面写入流程（存储机制）

### 3.1 存储模型

Wiki 页面存储在**独立的 Git 仓库**中，与代码仓库分离：

- **仓库路径**: `{owner}/{repo}.wiki.git` (`models/repo/wiki.go:78-86`)
- **默认分支**: 由 `DefaultWikiBranch` 数据库字段控制（见第 2 节）
- **文件格式**: Markdown 文件 (`.md` 后缀)

### 3.2 写入核心流程

**入口函数**: `services/wiki/wiki.go:85-240` `updateWikiPage()`

写入步骤：
1. 仓库归档检查 `repo.MustNotBeArchived()`
2. 全局锁防止并发写入 `globallock.Lock("wiki_working_{repoID}")`
3. 初始化 Wiki 仓库（如不存在）`InitWiki()` → `git init` + `SetDefaultBranch()`
4. 克隆到临时目录 `CloneRepoToLocal(bare=true)`
5. 路径准备与冲突检查 `prepareGitPath()`
6. Git 对象操作 `HashObjectBytes` → `AddObjectToIndex(100644)` → `WriteTree`
7. GPG 签名（可选）`asymkey_service.SignWikiCommit()`
8. 提交与推送 `CommitTree()` → `PushFromLocal()`

### 3.3 并发控制

`services/wiki/wiki.go:94-98`：通过 `globallock.Lock` 以仓库 ID 为粒度加锁，保证同一仓库的 Wiki 写入操作串行执行。

## 4. 访问鉴权机制（权限边界）

### 4.1 权限模型

Wiki 权限基于 Gitea 的**单元权限系统**，涉及两个权限单元：

```go
// models/unit/unit.go
UnitWiki = Unit{TypeWiki, "repo.wiki", "/wiki", "repo.wiki.desc", 4, perm.AccessModeOwner}
UnitExternalWiki = Unit{TypeExternalWiki, "repo.ext_wiki", "/wiki", "repo.ext_wiki.desc", 102, perm.AccessModeRead}
```

**注意**：`UnitExternalWiki` 的 `MaxAccessMode` 为 `AccessModeRead`，即外部 Wiki 只读，无法通过 Gitea 写入。

### 4.2 权限层级

```
AccessModeOwner (4)  ── 所有者/管理员
AccessModeAdmin (3)  ── 仓库管理员
AccessModeWrite (2)  ── 可写
AccessModeRead  (1)  ── 可读
AccessModeNone  (0)  ── 无权限
```

### 4.3 路由层权限中间件

**Web 端路由定义** (`routers/web/web.go:1571-1585`)：

```go
m.Group("/{username}/{reponame}/wiki", func() {
    m.Combo("").Get(repo.Wiki).Post(..., reqUnitWikiWriter, ..., repo.WikiPost)
    m.Combo("/*").Get(repo.Wiki).Post(..., reqUnitWikiWriter, ..., repo.WikiPost)
    // ...
}, optSignIn, context.RepoAssignment, repo.MustEnableWiki, reqUnitWikiReader, ...)
```

**API 端路由定义** (`routers/api/v1/api.go:1309-1317`)：

```go
m.Group("/wiki", func() {
    m.Combo("/page/{pageName}").
        Get(repo.GetWikiPage).
        Patch(mustNotBeArchived, reqToken(), reqRepoWriter(unit.TypeWiki), ..., repo.EditWikiPage).
        Delete(mustNotBeArchived, reqToken(), reqRepoWriter(unit.TypeWiki), repo.DeleteWikiPage)
    m.Get("/revisions/{pageName}", repo.ListPageRevisions)
    m.Post("/new", reqToken(), mustNotBeArchived, reqRepoWriter(unit.TypeWiki), ..., repo.NewWikiPage)
    m.Get("/pages", repo.ListWikiPages)
}, mustEnableWiki)
```

### 4.4 Web 与 API 对外部 Wiki 的鉴权边界差异

这是 Web 和 API 之间**最关键的鉴权差异**：

| 维度 | Web 端 `MustEnableWiki` | API 端 `mustEnableWiki` |
|-----|------------------------|------------------------|
| **位置** | `routers/web/repo/wiki.go:49-70` | `routers/api/v1/api.go:727-732` |
| **检查 TypeWiki** | ✅ `CanRead(unit.TypeWiki)` | ✅ `CanRead(unit.TypeWiki)` |
| **检查 TypeExternalWiki** | ✅ `CanRead(unit.TypeExternalWiki)` | ❌ **未检查** |
| **外部 Wiki 重定向** | ✅ 自动重定向到外部 URL | ❌ 无此逻辑 |

```go
// Web 端：同时检查内置和外部 Wiki
func MustEnableWiki(ctx *context.Context) {
    if !ctx.Repo.Permission.CanRead(unit.TypeWiki) &&
        !ctx.Repo.Permission.CanRead(unit.TypeExternalWiki) {
        ctx.NotFound(nil)
        return
    }
    repoUnit, err := ctx.Repo.Repository.GetUnit(ctx, unit.TypeExternalWiki)
    if err == nil {
        ctx.Redirect(repoUnit.ExternalWikiConfig().ExternalWikiURL)
        return
    }
}

// API 端：仅检查内置 Wiki
func mustEnableWiki(ctx *context.APIContext) {
    if !(ctx.Repo.Permission.CanRead(unit.TypeWiki)) {
        ctx.APIErrorNotFound()
        return
    }
}
```

**影响**：当仓库配置了外部 Wiki（且禁用了内置 Wiki）时：
- **Web 端**：用户访问 `/wiki` 会被自动重定向到外部 Wiki URL，体验正常
- **API 端**：所有 Wiki API 返回 404，因为 `mustEnableWiki` 只检查了 `TypeWiki`，不识别 `TypeExternalWiki`

此外，Web 端写操作需要 `reqSignIn` + `reqUnitWikiWriter` + `RepoMustNotBeArchived` 三重检查；API 端写操作则需要 `reqToken()` + `reqRepoWriter(unit.TypeWiki)` + `mustNotBeArchived`。

### 4.5 操作级权限检查

Web 端在处理函数内部进行二次权限检查 (`routers/web/repo/wiki.go:420-443`)：

```go
func WikiPost(ctx *context.Context) {
    switch ctx.FormString("action") {
    case "_new":
        if !ctx.Repo.Permission.CanWrite(unit.TypeWiki) {
            ctx.NotFound(nil); return
        }
        NewWikiPost(ctx)
    case "_delete":
        if !ctx.Repo.Permission.CanWrite(unit.TypeWiki) {
            ctx.NotFound(nil); return
        }
        DeleteWikiPagePost(ctx)
    }
    // 编辑操作同样检查
}
```

模板层权限控制 (`routers/web/repo/wiki.go:447`)：
```go
ctx.Data["CanWriteWiki"] = ctx.Repo.Permission.CanWrite(unit.TypeWiki) && !ctx.Repo.Repository.IsArchived
```

## 5. `_Sidebar` 和 `_Footer` 的展示差异

### 5.1 三种页面列表场景的对比

| 场景 | 过滤 `_Sidebar`/`_Footer` | 代码位置 |
|-----|---------------------------|---------|
| Web 页面查看时的侧边栏页面列表 (`renderViewPage`) | ✅ **过滤掉** | `routers/web/repo/wiki.go:208` |
| Web 页面列表页 (`WikiPages`) | ❌ **不过滤** | `routers/web/repo/wiki.go:572-592` |
| API 页面列表 (`ListWikiPages`) | ❌ **不过滤** | `routers/api/v1/repo/wiki.go:316-334` |

### 5.2 具体实现

**Web 侧边栏列表 — 过滤** (`routers/web/repo/wiki.go:196-217`)：

```go
// renderViewPage 中构建侧边栏页面列表
for _, entry := range entries {
    if !entry.IsRegular() { continue }
    wikiName, err := wiki_service.GitPathToWebPath(entry.Name())
    // ...
    } else if wikiName == "_Sidebar" || wikiName == "_Footer" {
        continue  // ← 过滤掉特殊页面
    }
    pages = append(pages, PageMeta{...})
}
```

**Web 页面列表页 — 不过滤** (`routers/web/repo/wiki.go:572-592`)：

`WikiPages` 函数遍历所有 `entries`，仅跳过非普通文件和无效文件名，**不对 `_Sidebar`/`_Footer` 做特殊处理**。

**API 页面列表 — 不过滤** (`routers/api/v1/repo/wiki.go:316-334`)：

`ListWikiPages` 同样遍历所有 `entries`，仅跳过非普通文件、超出分页范围和无效文件名的条目。

### 5.3 内容读取差异

Web 端查看页面时 (`renderViewPage`)，`_Sidebar` 和 `_Footer` 的内容会被单独渲染为侧边栏和页脚 (`routers/web/repo/wiki.go:285-307`)：

```go
if !isSideBar {
    sidebarContent, _, _, _ := wikiContentsByName(ctx, commit, "_Sidebar")
    ctx.Data["WikiSidebarHTML"], _ = renderFn(sidebarContent)
}
if !isFooter {
    footerContent, _, _, _ := wikiContentsByName(ctx, commit, "_Footer")
    ctx.Data["WikiFooterHTML"], _ = renderFn(footerContent)
}
```

API 端获取单页面时 (`getWikiPage`)，`_Sidebar` 和 `_Footer` 的内容作为 `sidebar` 和 `footer` 字段内联返回 (`routers/api/v1/repo/wiki.go:185-190`)：

```go
sidebarContent, _ := wikiContentsByName(ctx, commit, "_Sidebar", true)
footerContent, _ := wikiContentsByName(ctx, commit, "_Footer", true)
```

注意 API 的 `wikiContentsByName` 有第二个参数 `isSidebarOrFooter`，当为 `true` 时，如果页面不存在不会返回 404 错误而是静默跳过。

### 5.4 编辑限制

当 `_Sidebar` 或 `_Footer` 为非 `.md` 后缀的原始文件时，编辑页面会返回 403 (`routers/web/repo/wiki.go:403-404`)：

```go
if isRaw {
    ctx.HTTPError(http.StatusForbidden, "Editing of raw wiki files is not allowed")
}
```

## 6. `%2F` 路径兼容约束对页面定位和历史查询的影响

### 6.1 路径转换体系

Wiki 使用三层路径表示 (`services/wiki/wiki_path.go:18-39`)：

| 层级 | 含义 | 示例 |
|-----|------|------|
| **Display Segment** | 用户看到的标题 | `Home Page`、`2000-01-02 meeting` |
| **WebPath** | URL 中的路径 | `Home-Page`、`2000-01-02+meeting.-` |
| **GitPath** | Git 仓库中的文件名 | `Home-Page.md`、`2000-01-02 meeting.-.md` |

### 6.2 `%2F` 约束的来源

`WebPathFromRequest` (`services/wiki/wiki_path.go:144-149`) 在解析请求路径时，**强制将 `/` 替换为 `%2F`**：

```go
func WebPathFromRequest(s string) WebPath {
    s = util.PathJoinRelX(s)
    // The old wiki code's behavior is always using %2F, instead of subdirectory.
    s = strings.ReplaceAll(s, "/", "%2F")
    return WebPath(s)
}
```

**原因**：旧版 Wiki 代码总是使用 `%2F` 表示路径中的斜杠，而非子目录。许多用户仓库中已存在包含 `%2F` 的 Wiki 文件名。注释明确指出：

> Although this package now has the ability to support subdirectory, but the route package doesn't:
> - Double-escaping problem: the URL `/wiki/abc%2Fdef` becomes `/wiki/abc/def` by ctx.PathParam, which is incorrect
> - This problem should have been 99% fixed, but it needs more tests.
> - The old wiki code's behavior is always using %2F, instead of subdirectory, so there are a lot of legacy "%2F" files in user wikis.

### 6.3 对页面定位的影响

当 Wiki 页面名包含 `/` 或 `%2F` 时：

1. **请求阶段**：URL `/wiki/some/path` 被 Web 框架拆分为路径段，`ctx.PathParamRaw("*")` 得到 `some/path`
2. **规范化**：`WebPathFromRequest` 将其转为 `some%2Fpath`，确保路径不被子目录化
3. **Git 查找**：`WebPathToGitPath` 将 `some%2Fpath` 转为 Git 文件名 `some%2Fpath.md`
4. **实际文件**：Git 仓库中文件名为 `some%2Fpath.md`，而非 `some/path.md`

**潜在问题**：如果用户通过 Git 命令行直接创建了包含真实 `/` 的 Wiki 文件（如 `some/path.md`），Gitea 的路径转换机制无法定位到该文件，因为 `WebPathFromRequest` 会把 `/` 转为 `%2F`。

### 6.4 对历史查询的影响

历史查询依赖 `GitPath` 定位文件 (`routers/web/repo/wiki.go:310`, `routers/api/v1/repo/wiki.go:438-443`)：

```go
// Web 端
commitsCount, _ := gitrepo.FileCommitsCount(ctx, ctx.Repo.Repository.WikiStorageRepo(),
    ctx.Repo.Repository.DefaultWikiBranch, pageFilename)

// API 端
commitsHistory, err := wikiGitRepo.CommitsByFileAndRange(
    git.CommitsByFileAndRangeOptions{
        Revision: ctx.Repo.Repository.DefaultWikiBranch,
        File:     pageFilename,   // ← 这个值来自 WebPathToGitPath 的转换结果
        Page:     page,
    })
```

`pageFilename` 来自 `wikiContentsByName` / `wikiEntryByName` 的返回值，该值由 `WebPathToGitPath` 转换得到。因此：

- **正常路径**（如 `Home-Page`）：`WebPathToGitPath("Home-Page")` → `Home-Page.md` → 历史查询正常
- **含 `%2F` 的路径**（如 `some%2Fpath`）：`WebPathToGitPath("some%2Fpath")` → `some%2Fpath.md` → 历史查询正常，前提是文件确实以此名存储
- **含真实 `/` 的路径**（如通过 Git CLI 创建的 `dir/page.md`）：路径转换后变为 `%2F` 形式，导致 `git rev-list` 找不到文件，历史查询返回 0 条记录

### 6.5 Dash Marker 机制

为处理标题中包含连字符 `-` 与空格转换的歧义，引入了 dash marker (`.-`) (`services/wiki/wiki_path.go:53-63`)：

```go
func hasDashMarker(s string) bool    { return strings.HasSuffix(s, ".-") }
func removeDashMarker(s string) string { return strings.TrimSuffix(s, ".-") }
func addDashMarker(s string) string    { return s + ".-" }
```

规则：
- 如果 WebPath 段以 `.-` 结尾，表示**不进行 dash-space 转换**，原样保留
- 否则，`-` 会被视为空格的 URL 编码形式

示例：`2000-01-02+meeting.-` → 显示标题为 `2000-01-02 meeting`（保留连字符，不转为空格）

## 7. 历史回溯实现

### 7.1 版本计数

`FileCommitsCount` (`modules/gitrepo/commit.go:48-54`)，底层调用 `git rev-list --count {branch} -- {file}`

### 7.2 历史列表查询

`CommitsByFileAndRange` (`modules/git/repo_commit.go:228-270`)，使用 `git rev-list` + 分页参数，流式读取提交 ID 并转换为 Commit 对象。

### 7.3 Web 端历史页面

`renderRevisionPage` (`routers/web/repo/wiki.go:316-375`) 流程：
1. `findWikiRepoCommit()` — 获取 Wiki 仓库最新提交（含默认分支同步）
2. `wikiContentsByName()` — 查找页面文件入口，获取 GitPath
3. `FileCommitsCount()` — 统计提交总数
4. `CommitsByFileAndRange()` — 分页查询历史记录
5. `ConvertFromGitCommit()` — 转换为前端格式

### 7.4 API 端历史查询

`ListPageRevisions` (`routers/api/v1/repo/wiki.go:380-452`)：
- 返回结构化的 `WikiCommitList` JSON 响应
- 支持分页参数
- 通过 `X-Total-Count` 响应头返回总数

## 8. 权限与历史回溯的关联

### 8.1 权限决定历史可见性

历史查询的入口始终受到读权限控制：

```
Web: optSignIn → RepoAssignment → MustEnableWiki → reqUnitWikiReader → Handler
API: TokenAuth → RepoAssignment → mustEnableWiki → Handler
```

没有 `CanRead(unit.TypeWiki)` 权限的用户无法触发任何历史查询。

### 8.2 默认分支作为权限与历史的桥梁

默认分支贯穿权限检查和历史查询：
- **写入**时推送到 `refs/heads/{DefaultWikiBranch}` (`services/wiki/wiki.go:222`)
- **读取**时从 `DefaultWikiBranch` 获取最新提交 (`findWikiRepoCommit`)
- **历史查询**时以 `DefaultWikiBranch` 为 revision 参数 (`FileCommitsCount`, `CommitsByFileAndRange`)

如果默认分支不一致（数据库与 Git 仓库不同步），**Web 端会自动修正**，**API 端则直接失败**。这意味着在极端情况下，API 用户可能因默认分支不同步而无法查看历史，而 Web 用户不受影响。

### 8.3 路径转换对历史一致性的影响

历史查询依赖 `GitPath` 定位文件，而 `GitPath` 由 `WebPathToGitPath` 从 `WebPath` 转换得到。`%2F` 约束确保了：
- 写入时的 GitPath 和读取时的 GitPath **始终一致**（都经过 `WebPathToGitPath` 转换）
- 历史查询使用的 `pageFilename` 与写入时的文件名**一致**

但如果通过 Git CLI 绕过 Gitea 创建了包含真实 `/` 的文件，路径转换将导致读取/历史查询无法定位到该文件。

## 9. 安全边界总结

### 9.1 多层权限检查

1. **路由中间件层**: `reqUnitWikiReader` / `reqUnitWikiWriter` (Web) / `mustEnableWiki` + `reqRepoWriter` (API)
2. **Wiki 启用检查**: `MustEnableWiki()` (Web) / `mustEnableWiki()` (API)
3. **操作二次检查**: 处理函数内部再次检查 `CanWrite`
4. **仓库状态检查**: `RepoMustNotBeArchived()` / `mustNotBeArchived()` 防止归档仓库写入

### 9.2 Web 与 API 鉴权差异清单

| 差异点 | Web 端 | API 端 |
|-------|--------|--------|
| 外部 Wiki 支持 | 检查 `TypeExternalWiki` + 自动重定向 | 不检查，直接 404 |
| 默认分支同步 | 自动修正数据库与 Git 仓库的不一致 | 不修正，直接失败 |
| 写操作认证 | `reqSignIn` (session) | `reqToken()` (API token) |
| `_Sidebar`/`_Footer` API 可见性 | 不涉及 JSON API | `ListWikiPages` 会暴露 |

### 9.3 存储安全

- **独立仓库**: Wiki 与代码仓库分离，避免互相影响
- **Git 签名**: 支持 GPG 签名 Wiki 提交 (`services/wiki/wiki.go:202`)
- **并发控制**: 全局锁防止并发写入冲突
- **路径验证**: `validateWebPath()` 防止保留名称（`_pages`、`_new`、`_edit`、`raw`）和路径遍历攻击

### 9.4 特殊文件处理

| 特殊页面 | 侧边栏页面列表 | 页面列表页 | API 列表 | 内容渲染 |
|---------|--------------|----------|---------|---------|
| `_Sidebar` | 不显示 | 显示 | 显示 | 作为侧边栏渲染 |
| `_Footer` | 不显示 | 显示 | 显示 | 作为页脚渲染 |
| 非 `.md` 文件 | 显示 | 显示 | 显示 | 不允许编辑 (403) |

## 10. 核心文件清单

| 文件路径 | 核心职责 |
|---------|---------|
| `models/repo/repo.go:166,326-328` | `DefaultWikiBranch` 字段定义与回退逻辑 |
| `models/repo/wiki.go` | Wiki 数据模型、仓库路径定义 |
| `models/migrations/v1_22/v289.go` | `default_wiki_branch` 列的数据库迁移 |
| `services/wiki/wiki.go` | Wiki 核心业务逻辑（增删改、初始化、分支变更） |
| `services/wiki/wiki_path.go` | WebPath ↔ GitPath 转换，%2F 兼容逻辑 |
| `services/repository/repository.go:302-308` | `HasWiki()` 判断 Wiki 仓库是否存在 |
| `routers/web/repo/wiki.go` | Web 端路由处理函数，含默认分支同步 |
| `routers/api/v1/repo/wiki.go` | API 端路由处理函数，无默认分支同步 |
| `routers/web/web.go:1571-1585` | Wiki Web 路由组注册与中间件 |
| `routers/api/v1/api.go:727-732,1309-1317` | Wiki API 路由组注册与中间件 |
| `models/unit/unit.go` | 权限单元 TypeWiki/TypeExternalWiki 定义 |
| `models/perm/access/repo_permission.go` | 权限计算核心逻辑 |
| `services/context/permission.go` | 权限中间件实现 |
| `modules/gitrepo/commit.go` | 提交计数函数 |
| `modules/git/repo_commit.go` | 历史提交查询函数 |
