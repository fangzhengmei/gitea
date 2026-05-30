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

通过 `ChangeDefaultWikiBranch` (`services/wiki/wiki.go:382-410`) 修改。**注意：数据库事务只能回滚数据库操作，无法回滚 Git 操作！**

```go
func ChangeDefaultWikiBranch(ctx context.Context, repo *repo_model.Repository, newBranch string) error {
    if !git.IsValidRefPattern(newBranch) {
        return fmt.Errorf("invalid branch name: %s", newBranch)
    }
    return db.WithTx(ctx, func(ctx context.Context) error {
        // 步骤1: 先更新数据库
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
        // 步骤2: 再重命名 Git 分支（这一步不在数据库事务范围内！）
        err = gitrepo.RenameBranch(ctx, repo.WikiStorageRepo(), oldDefBranch, newBranch)
        if err != nil { return fmt.Errorf("unable to rename default branch: %w", err) }
        return nil
    })
}
```

### 2.5 事务边界与失败场景分析

#### 2.5.1 基础事实

**确定事实 1**：`db.WithTx` 只能保证数据库操作的原子性，**Git 分支重命名不在数据库事务范围内**
- 源码依据：`services/wiki/wiki.go:387-409` → 数据库更新在 `db.WithTx` 内，Git 操作也在其中但无法回滚
- 有效条件：所有调用 `ChangeDefaultWikiBranch` 的场景
- 置信度：✅ 100%

**确定事实 2**：数据库更新是函数的第一步，Git 重命名是最后一步
- 源码依据：`services/wiki/wiki.go:388-391`（数据库更新）vs `services/wiki/wiki.go:402-404`（Git 重命名）
- 有效条件：所有调用 `ChangeDefaultWikiBranch` 的场景
- 置信度：✅ 100%

**确定事实 3**：Web 端 `findWikiRepoCommit` 有自动分支同步逻辑，API 端没有
- 源码依据：`routers/web/repo/wiki.go:105-119`（有同步）vs `routers/api/v1/repo/wiki.go:473-497`（无同步）
- 有效条件：所有 Web/API Wiki 页面访问
- 置信度：✅ 100%

---

#### 2.5.2 失败场景 1：数据库更新成功 + Git 重命名失败

**触发条件（确定事实）**：
1. 数据库 `UPDATE repository SET default_wiki_branch = ?` 执行成功
2. `gitrepo.RenameBranch` 执行失败（权限、网络、磁盘 IO 等原因）

**结果（确定事实）**：
- 数据库状态：`default_wiki_branch = newBranch`（已提交，无法回滚）
- Git 仓库状态：分支仍为 `oldDefBranch`
- 数据库与 Git 状态不一致

**Web 端恢复机制（确定事实）**：
| 步骤 | 操作 | 结果 |
|-----|------|------|
| 1 | 用户访问 Wiki 页面 | 触发 `findWikiRepoCommit` |
| 2 | `GetBranchCommit(DefaultWikiBranch)` | 返回 `git.IsErrNotExist` |
| 3 | 触发分支同步逻辑 | ✅ 触发 |
| 4 | `GetDefaultBranch()` 读取 Git 真实分支 | 得到 `oldDefBranch` |
| 5 | 回写数据库 `default_wiki_branch` | 更新成功 |
| 6 | 重试 `GetBranchCommit` | 成功返回 commit |
| 7 | 最终结果 | ✅ 状态恢复一致，页面正常显示 |

**Web 端恢复触发条件（确定事实）**：
- 前置条件：Wiki 仓库存在但数据库记录的分支在 Git 中不存在
- 触发点：任何 Web 端 Wiki 页面访问（查看页面、页面列表、历史等）
- 恢复成本：一次访问即可完成恢复

**API 端行为（确定事实）**：
| 步骤 | 操作 | 结果 |
|-----|------|------|
| 1 | 调用 Wiki API | 触发 API 端 `findWikiRepoCommit` |
| 2 | `GetBranchCommit(DefaultWikiBranch)` | 返回 `git.IsErrNotExist` |
| 3 | API 端处理 | ❌ 直接返回 404，无同步逻辑 |
| 4 | 最终结果 | 所有 Wiki API 返回 404，直到手动干预 |

**API 端恢复条件（逻辑推论）**：
- 方式 1：通过 Web 端访问一次 Wiki 页面触发自动恢复
- 方式 2：管理员重新调用 `ChangeDefaultWikiBranch` API
- 方式 3：直接修改数据库 `default_wiki_branch` 字段

---

#### 2.5.3 失败场景 2：Git 重命名成功但事务后续失败

**分析（确定事实）**：
- Git 重命名是函数的最后一步（`services/wiki/wiki.go:402-404`）
- 之后只有 `return nil`，没有其他可能失败的操作
- **结论**：此场景理论上不可能发生

---

#### 2.5.4 分支同步触发条件汇总

| 条件 | Web 端 | API 端 | 置信度 |
|-----|--------|--------|--------|
| `GetBranchCommit` 返回 `git.IsErrNotExist` | ✅ 触发同步 | ❌ 直接 404 | ✅ 100% |
| 其他错误（权限、IO 等） | ❌ 返回错误 | ❌ 返回错误 | ✅ 100% |
| 正常分支存在 | ✅ 正常访问 | ✅ 正常访问 | ✅ 100% |

---

#### 2.5.5 Web 端 vs API 端分支不一致行为对比

| 维度 | Web 端 | API 端 | 置信度 |
|-----|--------|--------|--------|
| 数据库分支不存在于 Git | 自动读取真实分支并回写数据库，重试成功 | 返回 404 错误 | ✅ 100% |
| 自动恢复 | ✅ 一次访问即可恢复 | ❌ 永久失败直到手动干预 | ✅ 100% |
| 状态最终一致性 | 最终一致 | 需要外部干预 | ✅ 100% |
| 自愈机制来源 | `findWikiRepoCommit` 中的错误处理逻辑 | 无 | ✅ 100% |

**设计意图推论**：Web 端和 API 端在分支同步行为上的差异是有意设计的——Web 面向普通用户需要更高容错性，API 面向集成需要确定性和可预测的行为。

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

### 6.3 `findEntryForFile` 的 QueryUnescape 回退机制

#### 6.3.1 基础事实

**确定事实 1**：`GetTreeEntryByPath` 在文件不存在时返回 `(nil, ErrNotExist)`，而非 `(nil, nil)`
- 源码依据：`modules/git/tree_blob_gogit.go:59` → `return nil, ErrNotExist{"", relpath}`
- 有效条件：所有平台、所有 Wiki 仓库
- 置信度：✅ 100%（直接从源码验证）

**确定事实 2**：Web 端和 API 端的 `findEntryForFile` 错误处理逻辑完全不同
- 源码依据：`routers/web/repo/wiki.go:83` vs `routers/api/v1/repo/wiki.go:457`
- 有效条件：所有平台、所有 Wiki 仓库
- 置信度：✅ 100%（直接从源码验证）

---

#### 6.3.2 Web 端回退机制（确定事实）

**Web 端 `findEntryForFile`** (`routers/web/repo/wiki.go:81-96`)：
```go
func findEntryForFile(commit *git.Commit, target string) (*git.TreeEntry, error) {
    entry, err := commit.GetTreeEntryByPath(target)
    // 关键：只有非 IsErrNotExist 错误才返回
    if err != nil && !git.IsErrNotExist(err) {
        return nil, err
    }
    if entry != nil {
        return entry, nil
    }
    // 回退：文件不存在时继续尝试 QueryUnescape
    unescapedTarget, err := url.QueryUnescape(target)
    if err != nil { return nil, err }
    return commit.GetTreeEntryByPath(unescapedTarget)
}
```

**Web 端回退触发条件（确定事实）**：
| 条件 | 是否触发回退 | 说明 |
|-----|-------------|------|
| `err == git.IsErrNotExist` 且 `entry == nil` | ✅ 触发 | 文件不存在是正常场景，触发 QueryUnescape 回退 |
| `err == nil` 且 `entry == nil` | ✅ 触发 | 理论上不可能，但代码逻辑上会触发 |
| `err != nil` 且 `!git.IsErrNotExist(err)` | ❌ 不触发 | 其他错误（权限、IO 错误等）直接返回 |

---

#### 6.3.3 API 端回退机制（确定事实）

**API 端 `findEntryForFile`** (`routers/api/v1/repo/wiki.go:455-470`)：
```go
func findEntryForFile(commit *git.Commit, target string) (*git.TreeEntry, error) {
    entry, err := commit.GetTreeEntryByPath(target)
    // 关键：任何错误都直接返回，包括 IsErrNotExist
    if err != nil {
        return nil, err
    }
    if entry != nil {
        return entry, nil
    }
    // 回退：只有 err == nil && entry == nil 才触发（理论上不可能）
    unescapedTarget, err := url.QueryUnescape(target)
    if err != nil { return nil, err }
    return commit.GetTreeEntryByPath(unescapedTarget)
}
```

**API 端回退触发条件（确定事实）**：
| 条件 | 是否触发回退 | 说明 |
|-----|-------------|------|
| `err == git.IsErrNotExist` 且 `entry == nil` | ❌ 不触发 | 直接返回错误，不执行回退代码 |
| `err == nil` 且 `entry == nil` | ✅ 触发 | 理论上不可能（`GetTreeEntryByPath` 从不返回此组合） |
| `err != nil` 且 `!git.IsErrNotExist(err)` | ❌ 不触发 | 直接返回错误 |

**确定结论**：
- API 端的 QueryUnescape 回退逻辑**在实践中永不触发**
- 因为 `GetTreeEntryByPath` 找不到文件时返回 `ErrNotExist` 错误，被 API 端直接返回
- 这是 API 端与 Web 端的**行为不一致缺陷**

---

### 6.4 对页面定位的影响：三种场景分析

#### 6.4.1 场景 A：Gitea 创建的含 %2F 页面（确定事实）

**适用范围**：通过 Gitea Web UI 或 API 创建的 Wiki 页面
**Web 端和 API 端表现一致**

| 步骤 | 操作 | 结果 |
|-----|------|------|
| 1 | URL 请求 | `/wiki/some%2Fpath` 被框架解码，路径参数为 `some/path` |
| 2 | 路径规范化 | `WebPathFromRequest` → `some%2Fpath` |
| 3 | Git 路径转换 | `WebPathToGitPath` → `some%2Fpath.md` |
| 4 | 第一遍查找 | `findEntryForFile("some%2Fpath.md")` → 找到文件 |
| 5 | 最终结果 | ✅ 页面正常显示 |

---

#### 6.4.2 场景 B：Git CLI 创建的含真实斜杠页面（Web 端）

**适用范围**：通过 `git clone` + 文件系统操作创建的含真实 `/` 的 Wiki 页面
**确定事实 + 逻辑推论**

| 步骤 | 操作 | 结果（确定事实） |
|-----|------|-----------------|
| 1 | URL 请求 | `/wiki/some/path` → 路径参数为 `some/path` |
| 2 | 路径规范化 | `WebPathFromRequest` → `some%2Fpath` |
| 3 | Git 路径转换 | `WebPathToGitPath` → `some%2Fpath.md` |
| 4 | 第一遍查找 | `findEntryForFile("some%2Fpath.md")` → `ErrNotExist`（Git 中是 `some/path.md`） |
| 5 | 第二遍查找 | QueryUnescape 后 `some/path.md` → ✅ 找到文件（确定事实） |
| 6 | 最终结果 | ✅ 页面内容正常显示（确定事实） |

**逻辑推论**：此场景下页面能正常显示，但历史统计可能失败（见 §6.7）。

---

#### 6.4.3 场景 C：Git CLI 创建的含真实斜杠页面（API 端）

**适用范围**：通过 `git clone` + 文件系统操作创建的含真实 `/` 的 Wiki 页面
**确定事实**

| 步骤 | 操作 | 结果（确定事实） |
|-----|------|-----------------|
| 1 | API 请求 | `GET /repos/{owner}/{repo}/wiki/page/some/path` → 路径参数为 `some/path` |
| 2 | 路径规范化 | `WebPathFromRequest` → `some%2Fpath` |
| 3 | Git 路径转换 | `WebPathToGitPath` → `some%2Fpath.md` |
| 4 | 第一遍查找 | `findEntryForFile("some%2Fpath.md")` → `ErrNotExist`（Git 中是 `some/path.md`） |
| 5 | 错误处理 | `err == IsErrNotExist` → **API 端直接返回 404**（确定事实） |
| 6 | 最终结果 | ❌ 页面不存在（确定事实） |

**确定结论**：通过 Git CLI 创建的含真实 `/` 的 Wiki 页面，在 API 端**完全无法访问**。

---

#### 6.4.4 场景对比总结

| 场景 | Web 端 | API 端 | 置信度 |
|-----|--------|--------|--------|
| Gitea 创建的 `%2F` 文件 | ✅ 正常 | ✅ 正常 | ✅ 100% |
| Git CLI 创建的真实 `/` 文件 | ✅ 正常 | ❌ 404 | ✅ 100% |

**关键发现（确定事实）**：
- Web 端 QueryUnescape 回退正常工作，兼容 Git CLI 创建的真实目录文件
- API 端 QueryUnescape 回退永不触发，真实 `/` 文件完全无法访问
- 但 Web 端即使页面能显示，**历史统计仍然可能失败**（见 §6.7）

### 6.5 `pageFilename` 与 `entry.Name()` 的使用区别

#### 6.5.1 基础事实

**确定事实 1**：`pageFilename` 和 `entry.Name()` 在代码中有明确的使用分工
- 源码依据：`routers/web/repo/wiki.go:310`, `routers/web/repo/wiki.go:357`, `routers/web/repo/wiki.go:525`
- 有效条件：所有 Wiki 页面访问场景
- 置信度：✅ 100%

**确定事实 2**：`wikiEntryByName` 返回的 `gitFilename` 始终是 `WebPathToGitPath(wikiName)` 的结果，与实际找到的文件路径无关
- 源码依据：`routers/web/repo/wiki.go:149` vs `routers/web/repo/wiki.go:168`
- 有效条件：所有 Wiki 页面查找场景
- 置信度：✅ 100%

---

#### 6.5.2 使用分工对比（确定事实）

| 用途 | 使用变量 | 来源 | 源码位置 |
|-----|---------|------|---------|
| 历史查询 `FileCommitsCount` | `pageFilename` | `WebPathToGitPath(wikiName)`（预期路径） | `routers/web/repo/wiki.go:310` |
| 历史查询 `CommitsByFileAndRange` | `pageFilename` | `WebPathToGitPath(wikiName)`（预期路径） | `routers/web/repo/wiki.go:357` |
| 遍历文件列表转换 WebPath | `entry.Name()` | Git TreeEntry 的实际文件名 | `routers/web/repo/wiki.go:577` |
| 检测渲染类型 `DetectRendererTypeByFilename` | `entry.Name()` | Git TreeEntry 的实际文件名 | `routers/web/repo/wiki.go:488` |
| 获取最后提交 `GetCommitByPath` | `entry.Name()` | Git TreeEntry 的实际文件名 | `routers/web/repo/wiki.go:525` |
| `PageMeta.GitEntryName` 字段 | `entry.Name()` | Git TreeEntry 的实际文件名 | `routers/web/repo/wiki.go:589` |

**关键代码（确定事实）**：
```go
// Web 端 wikiEntryByName (routers/web/repo/wiki.go:147-169)
func wikiEntryByName(...) (*git.TreeEntry, string, bool, bool) {
    gitFilename := wiki_service.WebPathToGitPath(wikiName)  // 预期路径
    entry, err := findEntryForFile(commit, gitFilename)    // 实际查找
    // 即使 findEntryForFile 通过回退找到真实路径的文件
    return entry, gitFilename, false, isRaw  // 返回的仍然是预期路径！
}

// 使用 entry.Name() 的场景（遍历文件列表）
for _, entry := range entries {
    wikiName, err := wiki_service.GitPathToWebPath(entry.Name())  // ← 用真实文件名
    pages = append(pages, PageMeta{
        GitEntryName: entry.Entry.Name(),  // ← 存储真实文件名
    })
}
```

---

### 6.6 历史查询参数不一致问题

#### 6.6.1 问题根因（确定事实）

**确定结论**：历史查询使用的是 `pageFilename`（预期路径），而不是实际找到的 `entry.Name()`（真实路径）。

**源码依据**：
- `wikiEntryByName` 返回：`(entry, gitFilename, ...)` → `gitFilename` 来自 `WebPathToGitPath(wikiName)`
- 历史查询调用：`FileCommitsCount(..., pageFilename)` → `pageFilename` 是 `gitFilename`
- 实际文件路径：`entry.Name()` → 来自 Git TreeEntry

**适用范围**：所有需要历史查询的 Wiki 页面访问场景（页面查看、历史页面等）

---

#### 6.6.2 不一致场景的完整流程

**适用范围**：Web 端访问通过 Git CLI 创建的含真实 `/` 的 Wiki 页面
**确定事实 + 逻辑推论**

| 步骤 | 操作 | 路径值 | 确定性 |
|-----|------|--------|--------|
| 1 | URL 请求 | `/wiki/some/path` | ✅ 确定 |
| 2 | `WebPathFromRequest` | 转为 `some%2Fpath` | ✅ 确定 |
| 3 | `WebPathToGitPath` | 转为 `some%2Fpath.md` | ✅ 确定 |
| 4 | `findEntryForFile` 第一遍查找 | 尝试 `some%2Fpath.md` → 返回 `ErrNotExist` | ✅ 确定 |
| 5 | `findEntryForFile` QueryUnescape 回退 | 尝试 `some/path.md` → ✅ 成功找到 | ✅ 确定 |
| 6 | `wikiEntryByName` 返回 | `entry`(真实文件对象), `gitFilename`=`some%2Fpath.md` | ✅ 确定 |
| 7 | `FileCommitsCount` 查询参数 | `some%2Fpath.md` → **Git 中不存在此文件！** | ✅ 确定 |
| 8 | 历史计数结果 | 0 | ⚠️ 逻辑推论（需实际测试） |
| 9 | 历史列表结果 | 空列表 | ⚠️ 逻辑推论（需实际测试） |

---

#### 6.6.3 具体影响分析

| 功能 | 预期路径 vs 真实路径一致 | 预期路径 vs 真实路径不一致 | 确定性 |
|-----|-----------------------|-------------------------|--------|
| 页面内容显示 | ✅ 正常 | ✅ 正常（通过回退找到） | ✅ 确定 |
| 历史版本计数 | ✅ 正确 | ⚠️ 可能为 0 | ⚠️ 推论 |
| 历史版本列表 | ✅ 正确 | ⚠️ 可能为空 | ⚠️ 推论 |
| 编辑保存 | ✅ 正常 | ✅ 正常（操作 TreeEntry 对象） | ✅ 确定 |
| 重命名页面 | ✅ 正常 | ⚠️ 新文件以 `%2F` 形式创建 | ⚠️ 推论 |
| 最后提交信息 | ✅ 正确 | ✅ 正确（使用 `entry.Name()`） | ✅ 确定 |

**修正建议（逻辑推论）**：
- 应使用 `entry.Name()` 而非 `gitFilename` 作为历史查询参数
- 但需要考虑对已有 `%2F` 格式文件的历史查询兼容性
- 需要验证 `entry.Name()` 返回的路径格式是否与 Git 命令行期望的格式一致

### 6.8 Dash Marker 机制

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

**正常情况（Gitea 创建的文件）**：
- 写入：UserTitle → WebPath → WebPathToGitPath → GitPath → Git Commit
- 读取：URL → WebPathFromRequest → WebPath → WebPathToGitPath → GitPath → 查找
- 历史查询：GitPath → `git rev-list` → 结果正确

`%2F` 约束确保了写入和读取的 GitPath **始终一致**，历史查询参数与写入时的文件名一致。

**异常情况（Git CLI 创建的含真实 `/` 文件）**：
- 页面内容查找：✅ 成功（通过 `findEntryForFile` 的 QueryUnescape 回退）
- 历史查询参数：❌ 错误（使用 `WebPathToGitPath` 转换后的路径，而非真实文件路径）
- 历史查询结果：❌ 空列表 / 0 条记录

### 8.4 页面命中与历史统计的路径不一致问题详解

**不一致的根因**：`wikiContentsByName` / `wikiEntryByName` 返回的 `gitFilename` 始终是 `WebPathToGitPath(wikiName)` 的结果，而非实际找到的文件的 `entry.Name()`。

```go
// 代码中的问题：
func wikiEntryByName(...) (*git.TreeEntry, string, bool, bool) {
    gitFilename := wiki_service.WebPathToGitPath(wikiName)  // 预期路径
    entry, err := findEntryForFile(commit, gitFilename)    // 实际查找（可能通过回退找到不同路径）
    // 返回的仍然是预期路径，而非实际找到的文件路径！
    return entry, gitFilename, noEntry, isRaw
}
```

**不一致的影响矩阵**：

| 功能模块 | 正常 %2F 文件 | Git CLI 创建的真实 `/` 文件 |
|---------|--------------|-------------------------|
| 页面内容显示 | ✅ | ✅ |
| 历史版本数显示 | ✅ | ❌ 显示 0 |
| 历史版本列表 | ✅ | ❌ 空列表 |
| 编辑保存 | ✅ | ✅（覆盖真实路径） |
| 重命名页面 | ✅ | ❌（新文件以 %2F 形式创建） |
| 删除页面 | ✅ | ✅（通过 TreeEntry 操作） |

**潜在修复方案**：
1. **方案 A**：在 `findEntryForFile` 中同时返回实际命中的路径
2. **方案 B**：在 `wikiEntryByName` 中通过 `entry.Name()` 获取真实路径
3. **方案 C**：在调用历史查询的地方使用 `entry.Name()` 而非传入的 `pageFilename`

**注意**：修复需要考虑对已有 `%2F` 格式文件的历史查询兼容性。

## 9. 安全边界总结

### 9.1 多层权限检查

1. **路由中间件层**: `reqUnitWikiReader` / `reqUnitWikiWriter` (Web) / `mustEnableWiki` + `reqRepoWriter` (API)
2. **Wiki 启用检查**: `MustEnableWiki()` (Web) / `mustEnableWiki()` (API)
3. **操作二次检查**: 处理函数内部再次检查 `CanWrite`
4. **仓库状态检查**: `RepoMustNotBeArchived()` / `mustNotBeArchived()` 防止归档仓库写入

### 9.2 Web 与 API 行为差异清单

| 差异点 | Web 端 | API 端 |
|-------|--------|--------|
| 外部 Wiki 支持 | 检查 `TypeExternalWiki` + 自动重定向 | 不检查，直接 404 |
| 默认分支同步 | 自动修正数据库与 Git 仓库的不一致 | 不修正，直接失败 |
| 写操作认证 | `reqSignIn` (session) | `reqToken()` (API token) |
| `_Sidebar`/`_Footer` API 可见性 | 不涉及 JSON API | `ListWikiPages` 会暴露 |
| `findEntryForFile` 错误处理 | `IsErrNotExist` 时触发 QueryUnescape 回退 | **任何错误都直接返回**，回退几乎永不触发 |
| Git CLI 创建的真实 `/` 文件 | ✅ 可访问（通过回退） | ❌ 完全无法访问（无回退） |
| Wiki 原始文件支持 | 支持（`.md` 回退） | 不支持（无二次查找逻辑） |
| `ListWikiPages` 计数口径 | 无 TotalCount 头 | **TotalCount = 未过滤的 entries 总数**，与实际返回数不一致 |

### 9.3 `ListWikiPages` 计数口径与过滤口径不一致问题

**API 端 `ListWikiPages`** (`routers/api/v1/repo/wiki.go:310-338`) 存在**计数口径与过滤口径不一致**的问题：

```go
// 过滤逻辑
for i, entry := range entries {
    if i < skip || i >= maxNum || !entry.IsRegular() {
        continue  // 跳过分页外的和非普通文件
    }
    wikiName, err := wiki_service.GitPathToWebPath(entry.Name())
    if err != nil {
        if repo_model.IsErrWikiInvalidFileName(err) {
            continue  // 跳过无效文件名
        }
        // ...
    }
    pages = append(pages, ...)
}

// 计数逻辑
ctx.SetLinkHeader(int64(len(entries)), limit)       // ← 使用未过滤的 entries 总数
ctx.SetTotalCountHeader(int64(len(entries)))        // ← 使用未过滤的 entries 总数
```

**问题**：
- **过滤口径**：排除非普通文件（目录、符号链接）、无效文件名
- **计数口径**：`len(entries)`（所有条目，包括目录、符号链接等）
- **结果**：`X-Total-Count` 头返回的总数大于实际返回的页面数，分页信息不准确

**示例场景**：
- Git 仓库中有 10 个条目：8 个 `.md` 文件 + 1 个目录 + 1 个无效文件名
- API 返回 `X-Total-Count: 10`
- 实际返回 `pages` 数组只有 8 个元素
- 影响：客户端分页逻辑混乱，最后一页可能为空

### 9.4 存储安全

- **独立仓库**: Wiki 与代码仓库分离，避免互相影响
- **Git 签名**: 支持 GPG 签名 Wiki 提交 (`services/wiki/wiki.go:202`)
- **并发控制**: 全局锁防止并发写入冲突
- **路径验证**: `validateWebPath()` 防止保留名称（`_pages`、`_new`、`_edit`、`raw`）和路径遍历攻击

### 9.5 已知边界问题与风险

---

#### 9.5.1 事务边界风险（确定事实）

**问题描述**：`ChangeDefaultWikiBranch` 的数据库事务无法回滚 Git 操作

**触发条件（确定事实）**：
1. 数据库 `UPDATE repository SET default_wiki_branch = ?` 执行成功
2. `gitrepo.RenameBranch` 执行失败（权限、网络、磁盘 IO 等原因）

**结果（确定事实）**：
- 数据库：`default_wiki_branch = newBranch`（已提交，无法回滚）
- Git：分支仍为 `oldDefBranch`
- 状态不一致

**恢复机制（确定事实）**：
- Web 端：可通过任意 Wiki 页面访问触发自动同步自愈
- API 端：永久失败，直到手动干预

**源码依据**：`services/wiki/wiki.go:387-409`
**有效条件**：所有调用 `ChangeDefaultWikiBranch` 的场景
**置信度**：✅ 100%

---

#### 9.5.2 `findEntryForFile` 回退机制不一致（确定事实）

**问题描述**：Web 端和 API 端的 `findEntryForFile` 错误处理逻辑不同

**差异对比（确定事实）**：
| 维度 | Web 端 | API 端 |
|-----|--------|--------|
| 错误处理条件 | `if err != nil && !git.IsErrNotExist(err)` | `if err != nil` |
| `IsErrNotExist` 时行为 | 触发 QueryUnescape 回退 | 直接返回错误，不触发回退 |
| 回退触发场景 | 文件不存在时触发 | 实践中永不触发 |
| 真实 `/` 文件可访问性 | ✅ 可访问 | ❌ 完全无法访问 |

**源码依据**：`routers/web/repo/wiki.go:83` vs `routers/api/v1/repo/wiki.go:457`
**有效条件**：所有 Wiki 页面查找场景
**置信度**：✅ 100%

---

#### 9.5.3 路径查找与历史统计不一致（确定事实 + 逻辑推论）

**问题描述**：`findEntryForFile` 的 QueryUnescape 回退机制与历史查询参数不匹配

**确定事实部分**：
1. `wikiEntryByName` 返回的 `gitFilename` 始终是 `WebPathToGitPath(wikiName)` 的结果
2. 历史查询使用 `pageFilename`（即 `gitFilename`）作为参数
3. 实际找到的文件路径是 `entry.Name()`，可能与 `gitFilename` 不同（通过回退找到真实路径时）
4. `GetCommitByPath` 使用 `entry.Name()`，因此最后提交信息正确

**逻辑推论部分**：
1. 当通过回退找到真实路径文件时，历史版本统计可能返回 0
2. 历史版本列表可能为空
3. 重命名后新文件可能以 `%2F` 形式创建

**源码依据**：`routers/web/repo/wiki.go:149`, `routers/web/repo/wiki.go:310`
**有效条件**：Web 端访问通过 Git CLI 创建的含真实 `/` 的文件
**置信度**：⚖️ 混合（部分确定，部分推论）

---

#### 9.5.4 `ListWikiPages` 计数口径不一致（确定事实）

**问题描述**：API 端 `ListWikiPages` 的 TotalCount 使用未过滤的 entries 总数

**计数口径对比（确定事实）**：
| 口径 | 逻辑 | 包含内容 |
|-----|------|---------|
| 过滤口径 | `if i < skip || i >= maxNum || !entry.IsRegular()` → continue | 仅普通文件、有效文件名 |
| 计数口径 | `len(entries)` | 所有条目（含目录、符号链接、无效文件名） |

**结果（确定事实）**：
- `X-Total-Count` 头返回的总数大于实际返回的页面数
- 客户端分页逻辑可能混乱，最后一页可能为空

**示例场景（确定事实）**：
- Git 仓库中有 10 个条目：8 个 `.md` 文件 + 1 个目录 + 1 个无效文件名
- API 返回 `X-Total-Count: 10`
- 实际返回 `pages` 数组只有 8 个元素

**源码依据**：`routers/api/v1/repo/wiki.go:310-338`
**有效条件**：API 端 `ListWikiPages` 调用
**置信度**：✅ 100%

---

#### 9.5.5 API 端不支持 Wiki 原始文件（确定事实）

**问题描述**：Web 端支持访问非 `.md` 格式的原始文件，API 端不支持

**实现差异（确定事实）**：
- Web 端 `wikiEntryByName`：有二次查找逻辑，去掉 `.md` 后缀重试
- API 端 `wikiContentsByName`：只有一次查找，不支持原始文件

**源码依据**：`routers/web/repo/wiki.go:155-163` vs `routers/api/v1/repo/wiki.go:514-528`
**有效条件**：访问非 `.md` 格式的 Wiki 文件
**置信度**：✅ 100%

---

#### 9.5.6 路径穿透风险（逻辑推论）

**问题描述**：`findEntryForFile` 的 QueryUnescape 机制可能被用于绕过路径验证

**理论风险（逻辑推论）**：
- 可通过精心构造的 `%2e%2e%2f` 尝试路径遍历
- `%2e%2e%2f` QueryUnescape 后为 `../`，可能访问上级目录

**缓解因素（确定事实）**：
- `WebPathFromRequest` 会先经过 `util.PathJoinRelX` 规范化
- 实际风险较低

**源码依据**：`services/wiki/wiki_path.go:144-149`
**有效条件**：理论上的攻击场景
**置信度**：⚠️ 逻辑推论（需安全审计验证）

### 9.6 特殊文件处理

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
