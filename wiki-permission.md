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

## 2. 页面写入流程（存储机制）

### 2.1 存储模型

Wiki 页面存储在**独立的 Git 仓库**中，与代码仓库分离：

- **仓库路径**: `{owner}/{repo}.wiki.git` (`models/repo/wiki.go:78-86`)
- **默认分支**: 可配置，默认 `wiki` 分支 (`models/repo/wiki.go:84-85`)
- **文件格式**: Markdown 文件 (`.md` 后缀)

```go
// models/repo/wiki.go:78-86
func RelativeWikiPath(ownerName, repoName string) string {
    return strings.ToLower(ownerName) + "/" + strings.ToLower(repoName) + ".wiki.git"
}

func (repo *Repository) WikiStorageRepo() StorageRepo {
    return StorageRepo(RelativeWikiPath(repo.OwnerName, repo.Name))
}
```

### 2.2 写入核心流程

**入口函数**: `services/wiki/wiki.go:85-240` `updateWikiPage()`

```
┌─────────────────────────────────────────────────────────────┐
│ updateWikiPage(ctx, doer, repo, oldName, newName, content) │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 1. 仓库归档检查                   │
          │    repo.MustNotBeArchived()      │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 2. 全局锁防止并发写入             │
          │    globallock.Lock("wiki_working_{repoID}") │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 3. 初始化 Wiki 仓库（如不存在）   │
          │    InitWiki() → git init         │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 4. 克隆到临时目录                │
          │    CloneRepoToLocal(bare=true)   │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 5. 路径准备与冲突检查            │
          │    prepareGitPath()              │
          │    - 检查旧文件是否存在           │
          │    - 处理重命名场景               │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 6. Git 对象操作                  │
          │    HashObjectBytes(content)      │
          │    AddObjectToIndex(100644)      │
          │    WriteTree()                   │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 7. GPG 签名（可选）              │
          │    asymkey_service.SignWikiCommit() │
          └──────────────────────────────────┘
                           │
                           ▼
          ┌──────────────────────────────────┐
          │ 8. 提交与推送                    │
          │    CommitTree()                  │
          │    PushFromLocal()               │
          └──────────────────────────────────┘
```

### 2.3 关键实现细节

**并发控制** (`services/wiki/wiki.go:94-98`)：
```go
releaser, err := globallock.Lock(ctx, getWikiWorkingLockKey(repo.ID))
if err != nil {
    return err
}
defer releaser()
```

**路径转换** (`services/wiki/wiki_path.go:96-111`)：
- `WebPath`: URL 中的路径，如 `Home-Page`、`100%25+Free`
- `GitPath`: Git 仓库中的实际文件名，如 `Home-Page.md`、`100%25 Free.md`
- 支持 dash marker (`.-`) 控制空格与连字符的转换

```go
func WebPathToGitPath(s WebPath) string {
    // ... 路径转义逻辑
    return strings.Join(a, "/") + ".md"
}
```

## 3. 访问鉴权机制（权限边界）

### 3.1 权限模型

Wiki 权限基于 Gitea 的**单元权限系统** (`unit.TypeWiki`)：

**权限单元定义** (`models/unit/unit.go:286-293`)：
```go
UnitWiki = Unit{
    TypeWiki,
    "repo.wiki",
    "/wiki",
    "repo.wiki.desc",
    4,
    perm.AccessModeOwner,  // 最高权限等级
}
```

### 3.2 权限层级

```
AccessModeOwner (4)  ── 所有者/管理员
AccessModeAdmin (3)  ── 仓库管理员
AccessModeWrite (2)  ── 可写
AccessModeRead  (1)  ── 可读
AccessModeNone  (0)  ── 无权限
```

### 3.3 路由层权限中间件

**路由定义** (`routers/web/web.go:1571-1585`)：
```go
m.Group("/{username}/{reponame}/wiki", func() {
    m.Combo("").Get(repo.Wiki).Post(..., reqUnitWikiWriter, ..., repo.WikiPost)
    m.Combo("/*").Get(repo.Wiki).Post(..., reqUnitWikiWriter, ..., repo.WikiPost)
    // ... 其他路由
}, optSignIn, context.RepoAssignment, repo.MustEnableWiki, reqUnitWikiReader, ...)
```

**中间件定义** (`services/context/permission.go:69-91`)：
```go
// 读权限检查
func RequireUnitReader(unitTypes ...unit.Type) func(ctx *Context) {
    return func(ctx *Context) {
        for _, unitType := range unitTypes {
            if ctx.Repo.Permission.CanRead(unitType) {
                return
            }
        }
        ctx.NotFound(nil)
    }
}

// 写权限检查
func RequireUnitWriter(unitTypes ...unit.Type) func(ctx *Context) {
    return func(ctx *Context) {
        if slices.ContainsFunc(unitTypes, ctx.Repo.Permission.CanWrite) {
            return
        }
        ctx.NotFound(nil)
    }
}
```

### 3.4 权限计算核心

**权限检查入口** (`models/perm/access/repo_permission.go:92-105`)：
```go
func (p *Permission) UnitAccessMode(unitType unit.Type) perm_model.AccessMode {
    // 1. 优先使用 unitsMode 中的明确权限
    if m, ok := p.unitsMode[unitType]; ok {
        return util.Iif(p.AccessMode >= perm_model.AccessModeAdmin, p.AccessMode, m)
    }
    // 2. 考虑匿名用户和所有登录用户的公共权限
    unitDefaultAccessMode := p.AccessMode
    unitDefaultAccessMode = max(unitDefaultAccessMode, p.anonymousAccessMode[unitType])
    unitDefaultAccessMode = max(unitDefaultAccessMode, p.everyoneAccessMode[unitType])
    // 3. 检查单元是否启用
    hasUnit := slices.ContainsFunc(p.units, func(u *repo_model.RepoUnit) bool { return u.Type == unitType })
    return util.Iif(hasUnit, unitDefaultAccessMode, perm_model.AccessModeNone)
}
```

### 3.5 Wiki 专属权限检查

**MustEnableWiki** (`routers/web/repo/wiki.go:49-70`)：
- 检查 Wiki 单元是否启用（内置或外部）
- 如果配置了外部 Wiki，自动重定向

```go
func MustEnableWiki(ctx *context.Context) {
    if !ctx.Repo.Permission.CanRead(unit.TypeWiki) &&
        !ctx.Repo.Permission.CanRead(unit.TypeExternalWiki) {
        ctx.NotFound(nil)
        return
    }
    // 外部 Wiki 重定向
    repoUnit, err := ctx.Repo.Repository.GetUnit(ctx, unit.TypeExternalWiki)
    if err == nil {
        ctx.Redirect(repoUnit.ExternalWikiConfig().ExternalWikiURL)
        return
    }
}
```

### 3.6 操作级权限检查

在处理函数内部还会进行**二次权限检查**：

**新建/编辑/删除** (`routers/web/repo/wiki.go:420-443`)：
```go
func WikiPost(ctx *context.Context) {
    switch ctx.FormString("action") {
    case "_new":
        if !ctx.Repo.Permission.CanWrite(unit.TypeWiki) {
            ctx.NotFound(nil)
            return
        }
        NewWikiPost(ctx)
    case "_delete":
        if !ctx.Repo.Permission.CanWrite(unit.TypeWiki) {
            ctx.NotFound(nil)
            return
        }
        DeleteWikiPagePost(ctx)
    }
    // ... 编辑操作同样检查
}
```

**模板层权限控制** (`routers/web/repo/wiki.go:447`)：
```go
ctx.Data["CanWriteWiki"] = ctx.Repo.Permission.CanWrite(unit.TypeWiki) && !ctx.Repo.Repository.IsArchived
```

## 4. 历史回溯实现

### 4.1 版本计数

**FileCommitsCount** (`modules/gitrepo/commit.go:48-54`)：
```go
func FileCommitsCount(ctx context.Context, repo Repository, revision, file string) (int64, error) {
    return CommitsCount(ctx, repo,
        CommitsCountOptions{
            Revision: []string{revision},
            RelPath:  []string{file},
        })
}
```

底层调用：`git rev-list --count {branch} -- {file}`

### 4.2 历史列表查询

**CommitsByFileAndRange** (`modules/git/repo_commit.go:228-270`)：
```go
func (repo *Repository) CommitsByFileAndRange(opts CommitsByFileAndRangeOptions) ([]*Commit, error) {
    gitCmd := gitcmd.NewCommand("rev-list").
        AddOptionFormat("--max-count=%d", setting.Git.CommitsRangeSize).
        AddOptionFormat("--skip=%d", (opts.Page-1)*setting.Git.CommitsRangeSize).
        AddDynamicArguments(opts.Revision)
    // ... 支持 not/since/until 过滤
    gitCmd.AddDashesAndList(opts.File)
    // ... 流式读取提交 ID 并转换为 Commit 对象
}
```

### 4.3 Web 端历史页面

**renderRevisionPage** (`routers/web/repo/wiki.go:316-375`)：
```
┌─────────────────────────────────────────┐
│ renderRevisionPage(ctx)                 │
└─────────────────────────────────────────┘
                │
                ▼
  ┌───────────────────────────────┐
  │ 1. 获取 Wiki 仓库最新提交     │
  │    findWikiRepoCommit()       │
  └───────────────────────────────┘
                │
                ▼
  ┌───────────────────────────────┐
  │ 2. 查找页面文件入口           │
  │    wikiContentsByName()       │
  └───────────────────────────────┘
                │
                ▼
  ┌───────────────────────────────┐
  │ 3. 统计提交总数               │
  │    FileCommitsCount()         │
  └───────────────────────────────┘
                │
                ▼
  ┌───────────────────────────────┐
  │ 4. 分页查询历史记录           │
  │    CommitsByFileAndRange()    │
  └───────────────────────────────┘
                │
                ▼
  ┌───────────────────────────────┐
  │ 5. 转换为前端格式             │
  │    ConvertFromGitCommit()     │
  └───────────────────────────────┘
```

### 4.4 API 端历史查询

**ListPageRevisions** (`routers/api/v1/repo/wiki.go:380-452`)：
- 返回结构化的 `WikiCommitList` JSON 响应
- 支持分页参数
- 通过 `X-Total-Count` 响应头返回总数

## 5. 三者衔接关系

### 5.1 完整调用链路（写入场景）

```
HTTP Request
     │
     ▼
[路由层] routers/web/web.go:1571
     │  ├─ optSignIn                 # 登录可选
     │  ├─ context.RepoAssignment    # 加载仓库信息
     │  ├─ repo.MustEnableWiki       # 检查 Wiki 启用
     │  └─ reqUnitWikiWriter         # 检查写权限 (CanWrite)
     │
     ▼
[Handler] routers/web/repo/wiki.go:657 (NewWikiPost)
     │  ├─ 表单验证
     │  ├─ UserTitleToWebPath()      # 标题转 WebPath
     │  └─ wiki_service.AddWikiPage()
     │
     ▼
[服务层] services/wiki/wiki.go:243 (AddWikiPage)
     │  └─ updateWikiPage(isNew=true)
     │     ├─ 全局锁
     │     ├─ InitWiki()             # 初始化 Git 仓库
     │     ├─ prepareGitPath()       # WebPath → GitPath
     │     ├─ Git 对象操作
     │     ├─ CommitTree()           # 生成 Git 提交
     │     └─ PushFromLocal()        # 推送到存储仓库
     │
     ▼
[存储层] {owner}/{repo}.wiki.git
     │
     ▼
[通知] notify_service.NewWikiPage() # 发送通知
```

### 5.2 完整调用链路（访问场景）

```
HTTP Request: GET /{owner}/{repo}/wiki/Home?action=_revision
     │
     ▼
[路由层] routers/web/web.go:1571
     │  ├─ reqUnitWikiReader         # 检查读权限 (CanRead)
     │  └─ repo.MustEnableWiki
     │
     ▼
[Handler] routers/web/repo/wiki.go:446 (Wiki)
     │  └─ action="_revision" → WikiRevision()
     │
     ▼
[Handler] routers/web/repo/wiki.go:505 (WikiRevision)
     │  └─ renderRevisionPage()
     │     ├─ findWikiRepoCommit()   # 打开 Git 仓库
     │     ├─ FileCommitsCount()     # 历史计数
     │     └─ CommitsByFileAndRange() # 历史列表
     │
     ▼
[存储层] Git rev-list 命令
```

### 5.3 关键衔接点

| 衔接点 | 涉及模块 | 作用 |
|-------|---------|------|
| **WebPath ↔ GitPath 转换** | `services/wiki/wiki_path.go` | URL 路径与 Git 文件名的双向转换，处理特殊字符转义 |
| **WikiStorageRepo()** | `models/repo/wiki.go:84` | 统一提供 Wiki 仓库的路径标识 |
| **Permission.CanRead/CanWrite** | `models/perm/access/repo_permission.go` | 统一的权限检查入口，支持单元级细粒度控制 |
| **findWikiRepoCommit()** | `routers/web/repo/wiki.go:98` | 统一打开 Wiki 仓库并获取最新提交，处理默认分支同步 |
| **globallock** | `services/wiki/wiki.go:94` | 防止并发写入冲突 |

### 5.4 数据流向

```
写入流程:
User Title → UserTitleToWebPath() → WebPath → WebPathToGitPath() → GitPath → Git Object → Git Commit → Git Push → .wiki.git

读取流程:
URL Path → WebPathFromRequest() → WebPath → WebPathToGitPath() → GitPath → Git Tree → Git Blob → Content → Markdown Render → HTML

历史查询:
WebPath → WebPathToGitPath() → GitPath → git rev-list → Commit List → ConvertFromGitCommit() → UI/JSON
```

## 6. 安全边界总结

### 6.1 多层权限检查

1. **路由中间件层**: `reqUnitWikiReader` / `reqUnitWikiWriter`
2. **Wiki 启用检查**: `MustEnableWiki()`
3. **操作二次检查**: 处理函数内部再次检查 `CanWrite`
4. **仓库状态检查**: `RepoMustNotBeArchived()` 防止归档仓库写入

### 6.2 存储安全

- **独立仓库**: Wiki 与代码仓库分离，避免互相影响
- **Git 签名**: 支持 GPG 签名 Wiki 提交 (`services/wiki/wiki.go:202`)
- **并发控制**: 全局锁防止并发写入冲突
- **路径验证**: `validateWebPath()` 防止保留名称和路径遍历攻击

### 6.3 特殊文件保护

- `_Sidebar` 和 `_Footer` 为特殊页面，不显示在页面列表中
- 原始文件（非 `.md` 后缀）不允许编辑 (`routers/web/repo/wiki.go:403-404`)

## 7. 核心文件清单

| 文件路径 | 核心职责 |
|---------|---------|
| `models/repo/wiki.go` | Wiki 数据模型、仓库路径定义 |
| `services/wiki/wiki.go` | Wiki 核心业务逻辑（增删改、初始化） |
| `services/wiki/wiki_path.go` | WebPath ↔ GitPath 转换 |
| `routers/web/repo/wiki.go` | Web 端路由处理函数 |
| `routers/api/v1/repo/wiki.go` | API 端路由处理函数 |
| `routers/web/web.go:1571-1585` | Wiki 路由组注册与中间件 |
| `models/unit/unit.go` | 权限单元 TypeWiki 定义 |
| `models/perm/access/repo_permission.go` | 权限计算核心逻辑 |
| `services/context/permission.go` | 权限中间件实现 |
| `modules/gitrepo/commit.go` | 提交计数函数 |
| `modules/git/repo_commit.go` | 历史提交查询函数 |
