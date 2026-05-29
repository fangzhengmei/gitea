# Gitea 页面模板渲染与接口数据来源脉络

## 一、核心架构分层

```
┌───────────────────────────────────────────────────────────────────┐
│                      HTTP Request / Response                       │
└─────────────────────────────────────┬─────────────────────────────┘
                                      │
                ┌─────────────────────┼─────────────────────┐
                │                     │                     │
         ┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
         │  Web 路由   │       │ API 路由    │       │  中间件层   │
         │ routers/web │       │ routers/api │       │ services/ctx│
         └──────┬──────┘       └──────┬──────┘       └──────┬──────┘
                │                     │                     │
                │                     │                     │
         ┌──────▼──────┐       ┌──────▼──────┐              │
         │  Web Handler│       │ API Handler │              │
         └──────┬──────┘       └──────┬──────┘              │
                │                     │                     │
                │              ┌──────▼──────┐              │
                │              │ Convert 层  │              │
                │              │ services/   │              │
                │              │  convert    │              │
                │              └──────┬──────┘              │
                │                     │                     │
         ┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
         │ ctx.Data    │       │ API Struct  │       │  公共数据   │
         │ (map[any]   │       │ modules/    │       │  注入       │
         │  any)       │       │  structs    │       │             │
         └──────┬──────┘       └──────┬──────┘       └─────────────┘
                │                     │
         ┌──────▼──────┐       ┌──────▼──────┐
         │ 模板引擎    │       │ JSON 序列化 │
         │ modules/    │       │ encoding/json│
         │ templates   │       └─────────────┘
         └──────┬──────┘
                │
         ┌──────▼──────┐
         │ HTML 输出   │
         └─────────────┘
```

---

## 二、Web 页面渲染完整链路

### 2.1 链路概览

```
HTTP Request
    ↓
[路由匹配] routers/web/web.go
    ↓
[中间件链] Contexter() → services/context/context.go:160
    │  ├─ 创建 Context 结构体 (services/context/context.go:41)
    │  ├─ 初始化 TemplateContext (services/context/context.go:101)
    │  ├─ 注入公共数据: CommonTemplateContextData()
    │  ├─ 注入系统配置、Flash 消息、PageData
    │  └─ ctx.Data = map[string]any{}
    ↓
[Handler] 例如 Home() → routers/web/repo/view_home.go:389
    │  ├─ 执行业务逻辑，查询 DB Model
    │  │   topics, _ := db.Find[repo_model.Topic](...)
    │  ├─ 直接注入 ctx.Data (无转换层)
    │  │   ctx.Data["Topics"] = topics          // view_home.go:66
    │  │   ctx.Data["LatestRelease"] = release  // view_home.go:183
    │  │   ctx.Data["LanguageStats"] = langs    // view_home.go:164
    │  └─ 调用 ctx.HTML() 触发渲染
    ↓
[模板渲染] ctx.HTML() → services/context/context_response.go:81
    │  ├─ 传入模板名和 ctx.Data
    │  └─ PageRenderer.HTML() → modules/templates/page.go:47
    ↓
[模板执行] Go Template Engine
    │  ├─ 模板通过 .Data.Key 访问数据
    │  │   例如: {{range .Data.Topics}} ... {{end}}
    │  └─ 输出 HTML
    ↓
HTTP Response (text/html)
```

### 2.2 关键代码详解

#### 2.2.1 上下文创建与公共数据注入

**文件**: `services/context/context.go:160`

```go
func Contexter() func(next http.Handler) http.Handler {
    rnd := templates.PageRenderer()
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(resp http.ResponseWriter, req *http.Request) {
            base := NewBaseContext(resp, req)
            ctx := NewWebContext(base, rnd, session.GetContextSession(req))
            
            // 注入公共模板数据
            ctx.Data.MergeFrom(middleware.CommonTemplateContextData())
            ctx.Data["CurrentURL"] = setting.AppSubURL + req.URL.RequestURI()
            ctx.Data["Link"] = ctx.Link
            
            // PageData 传递给 JavaScript (window.config.pageData)
            ctx.PageData = map[string]any{}
            ctx.Data["PageData"] = ctx.PageData
            
            // 注入系统配置
            ctx.Data["SystemConfig"] = setting.Config()
            ctx.Data["DisableMigrations"] = setting.Repository.DisableMigrations
            ctx.Data["EnableActions"] = setting.Actions.Enabled && ...
            ctx.Data["AllLangs"] = translation.AllLangs()
            
            next.ServeHTTP(ctx.Resp, ctx.Req)
        })
    }
}
```

#### 2.2.2 Handler 填充数据示例

**文件**: `routers/web/repo/view_home.go:389`

```go
func Home(ctx *context.Context) {
    // ... 前置检查 ...
    
    // 直接将 DB Model 注入 ctx.Data，无转换层
    prepareFuncs := []func(*context.Context){
        prepareOpenWithEditorApps,     // ctx.Data["OpenWithEditorApps"] = ...
        prepareHomeSidebarRepoTopics,  // ctx.Data["Topics"] = topics
        checkOutdatedBranch,
        prepareToRenderDirOrFile(entry),
        prepareRecentlyPushedNewBranches,
        prepareUpstreamDivergingInfo,  // ctx.Data["UpstreamDivergingInfo"] = ...
        prepareHomeSidebarLicenses,    // ctx.Data["DetectedRepoLicenses"] = ...
        prepareHomeSidebarLanguageStats, // ctx.Data["LanguageStats"] = langs
        prepareHomeSidebarLatestRelease, // ctx.Data["LatestRelease"] = release
    }
    
    for _, prepare := range prepareFuncs {
        prepare(ctx)
    }
    
    // 触发渲染
    ctx.HTML(http.StatusOK, tplRepoHome)
}
```

#### 2.2.3 模板渲染入口

**文件**: `services/context/context_response.go:81`

```go
func (ctx *Context) HTML(status int, name templates.TplName) {
    tmplStartTime := time.Now()
    ctx.Data["TemplateName"] = name
    ctx.Data["TemplateLoadTimes"] = func() string {
        return strconv.FormatInt(time.Since(tmplStartTime).Nanoseconds()/1e6, 10) + "ms"
    }
    
    // 核心：传入 ctx.Data 作为模板数据
    err := ctx.Render.HTML(ctx.Resp, status, name, ctx.Data, ctx.TemplateContext)
    // ...
}
```

#### 2.2.4 TemplateContext 辅助函数注入

**文件**: `services/context/context.go:101`

```go
func NewTemplateContextForWeb(ctx reqctx.RequestContext, req *http.Request, locale translation.Locale) TemplateContext {
    tmplCtx := NewTemplateContext(ctx, req)
    tmplCtx["Locale"] = locale
    tmplCtx["AvatarUtils"] = templates.NewAvatarUtils(ctx)
    tmplCtx["RenderUtils"] = templates.NewRenderUtils(ctx)
    tmplCtx["MiscUtils"] = templates.NewMiscUtils(ctx)
    tmplCtx["ActionsUtils"] = templates.NewActionsUtils(ctx)
    tmplCtx["RootData"] = ctx.GetData()
    tmplCtx["Consts"] = map[string]any{
        "RepoUnitTypeCode": unit.TypeCode,
        "RepoUnitTypeIssues": unit.TypeIssues,
        // ...
    }
    return tmplCtx
}
```

---

## 三、API 接口数据完整链路

### 3.1 链路概览

```
HTTP Request
    ↓
[路由匹配] routers/api/v1
    ↓
[中间件链] APIContexter() → services/context/api.go:225
    │  └─ 创建 APIContext 结构体 (services/context/api.go:34)
    ↓
[Handler] 例如 Get() → routers/api/v1/repo/repo.go:500
    │  ├─ 查询 DB Model
    │  │   repo, _ := repo_model.GetRepositoryByID(ctx, ...)
    │  └─ 加载关联属性
    │       ctx.Repo.Repository.LoadAttributes(ctx)
    ↓
[字段裁剪] convert.ToRepo() → services/convert/repository.go:22
    │  ├─ 显式字段映射（逐字段赋值）
    │  │   return &api.Repository{
    │  │       ID:          repo.ID,
    │  │       Owner:       ToUser(...),
    │  │       Name:        repo.Name,
    │  │       FullName:    repo.FullName(),
    │  │       Description: repo.Description,
    │  │       // ... 只保留 API 需要的字段
    │  │   }
    │  └─ 过滤内部字段（如 CreatedUnix 转为 Created 等）
    ↓
[API 结构体] modules/structs/repo.go:59
    │  ├─ 带 json tag 控制序列化
    │  │   type Repository struct {
    │  │       ID    int64  `json:"id"`
    │  │       Owner *User  `json:"owner"`
    │  │       Name  string `json:"name"`
    │  │       // ...
    │  │   }
    │  └─ json tag 决定最终 JSON 输出字段
    ↓
[JSON 序列化] ctx.JSON() → encoding/json
    ↓
HTTP Response (application/json)
```

### 3.2 关键代码详解

#### 3.2.1 API 上下文创建

**文件**: `services/context/api.go:225`

```go
func APIContexter() func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
            base := NewBaseContext(w, req)
            ctx := &APIContext{
                Base:  base,
                Cache: cache.GetCache(),
                Repo:  &Repository{},
                Org:   &APIOrganization{},
            }
            ctx.SetContextValue(apiContextKey, ctx)
            next.ServeHTTP(ctx.Resp, ctx.Req)
        })
    }
}
```

#### 3.2.2 API Handler 示例

**文件**: `routers/api/v1/repo/repo.go:500`

```go
// Get 获取仓库信息
func Get(ctx *context.APIContext) {
    // 加载关联属性
    if err := ctx.Repo.Repository.LoadAttributes(ctx); err != nil {
        ctx.APIErrorInternal(err)
        return
    }
    
    // 核心：通过 convert 层进行字段裁剪，然后 JSON 序列化
    ctx.JSON(http.StatusOK, convert.ToRepo(ctx, ctx.Repo.Repository, ctx.Repo.Permission))
}
```

#### 3.2.3 Convert 层 - 字段裁剪核心

**文件**: `services/convert/repository.go:22`

```go
// ToRepo converts a Repository to api.Repository
func ToRepo(ctx context.Context, repo *repo_model.Repository, permissionInRepo access_model.Permission) *api.Repository {
    return innerToRepo(ctx, repo, permissionInRepo, false)
}

func innerToRepo(ctx context.Context, repo *repo_model.Repository, permissionInRepo access_model.Permission, isParent bool) *api.Repository {
    // ... 权限处理、单元检查 ...
    
    // 显式字段映射：只保留 API 需要的字段
    return &api.Repository{
        ID:                            repo.ID,
        Owner:                         ToUserWithAccessMode(ctx, repo.Owner, permissionInRepo.AccessMode),
        Name:                          repo.Name,
        FullName:                      repo.FullName(),
        Description:                   repo.Description,
        Private:                       repo.IsPrivate,
        Empty:                         repo.IsEmpty,
        Archived:                      repo.IsArchived,
        Size:                          int(repo.Size / 1024),
        Fork:                          repo.IsFork,
        HTMLURL:                       repo.HTMLURL(ctx),
        SSHURL:                        cloneLink.SSH,
        CloneURL:                      cloneLink.HTTPS,
        Stars:                         repo.NumStars,
        Forks:                         repo.NumForks,
        Watchers:                      repo.NumWatches,
        OpenIssues:                    repo.NumOpenIssues,
        DefaultBranch:                 repo.DefaultBranch,
        Created:                       repo.CreatedUnix.AsTime(),  // 时间格式转换
        Updated:                       repo.UpdatedUnix.AsTime(),
        Permissions:                   permission,
        HasCode:                       hasCode,
        HasIssues:                     hasIssues,
        HasWiki:                       hasWiki,
        HasPullRequests:               hasPullRequests,
        // ... 更多字段
    }
}
```

#### 3.2.4 API 结构体定义 - JSON tag 控制输出

**文件**: `modules/structs/repo.go:59`

```go
// Repository represents a repository
type Repository struct {
    ID          int64  `json:"id"`
    Owner       *User  `json:"owner"`
    Name        string `json:"name"`
    FullName    string `json:"full_name"`
    Description string `json:"description"`
    Empty       bool   `json:"empty"`
    Private     bool   `json:"private"`
    Fork        bool   `json:"fork"`
    Template    bool   `json:"template"`
    Parent      *Repository `json:"parent,omitempty"`  // omitempty: 空值不输出
    Mirror      bool        `json:"mirror"`
    Size        int         `json:"size"`
    Language    string      `json:"language"`
    HTMLURL     string      `json:"html_url"`
    URL         string      `json:"url"`
    SSHURL      string      `json:"ssh_url"`
    CloneURL    string      `json:"clone_url"`
    Stars       int         `json:"stars_count"`
    Forks       int         `json:"forks_count"`
    Watchers    int         `json:"watchers_count"`
    OpenIssues  int         `json:"open_issues_count"`
    DefaultBranch string    `json:"default_branch"`
    Created     time.Time   `json:"created_at"`
    Updated     time.Time   `json:"updated_at"`
    Permissions *Permission `json:"permissions,omitempty"`
    HasCode     bool        `json:"has_code"`
    HasIssues   bool        `json:"has_issues"`
    HasWiki     bool        `json:"has_wiki"`
    // ...
}
```

---

## 四、Web 页面 vs API 接口：核心差异对比

| 对比维度 | Web 页面渲染 | API 接口输出 |
|---------|-------------|-------------|
| **上下文类型** | `Context` (services/context/context.go:41) | `APIContext` (services/context/api.go:34) |
| **数据载体** | `ctx.Data map[string]any` 动态映射 | 强类型结构体 (modules/structs) |
| **数据转换层** | ❌ 无，直接使用 DB Model | ✅ `services/convert.ToXxx()` 显式转换 |
| **字段裁剪时机** | 模板渲染时按需取用 | Convert 层显式映射 + JSON tag 过滤 |
| **DB Model 暴露** | ✅ 模板可直接访问所有字段 | ❌ 必须经过 Convert 层过滤 |
| **输出格式** | HTML (text/html) | JSON (application/json) |
| **渲染引擎** | Go Template Engine | encoding/json |
| **性能特点** | 模板解析开销，无额外转换开销 | JSON 序列化开销，有字段映射开销 |
| **安全性** | 依赖模板作者不输出敏感字段 | Convert 层统一管控，安全可控 |
| **典型代码** | `ctx.Data["Topics"] = topics` | `ctx.JSON(200, convert.ToRepo(ctx, repo, perm))` |

---

## 五、完整调用链代码溯源

### 5.1 Web 页面渲染调用链

```
1. 路由注册
   routers/web/web.go:257 → Web 路由初始化
   │
   ├─ 中间件注入
   │  services/context/context.go:160 → Contexter()
   │  ├─ services/context/context.go:124 → NewWebContext()
   │  └─ services/context/context.go:101 → NewTemplateContextForWeb()
   │
   ├─ Handler 执行
   │  routers/web/repo/view_home.go:389 → Home()
   │  ├─ 填充 ctx.Data (多个 prepare 函数)
   │  │  ├─ view_home.go:66 → ctx.Data["Topics"] = topics
   │  │  ├─ view_home.go:183 → ctx.Data["LatestRelease"] = release
   │  │  └─ view_home.go:164 → ctx.Data["LanguageStats"] = langs
   │  └─ view_home.go:463 → ctx.HTML(http.StatusOK, tplRepoHome)
   │
   └─ 模板渲染
      services/context/context_response.go:81 → Context.HTML()
      └─ modules/templates/page.go:47 → pageRenderer.HTML()
         ├─ 查找模板
         └─ 执行模板: t.Execute(w, data) → data 即 ctx.Data
```

### 5.2 API 接口调用链

```
1. 路由注册
   routers/api/v1/repo/repo.go → API 路由初始化
   │
   ├─ 中间件注入
   │  services/context/api.go:225 → APIContexter()
   │  └─ 创建 APIContext
   │
   ├─ Handler 执行
   │  routers/api/v1/repo/repo.go:500 → Get()
   │  ├─ 加载 DB Model 关联
   │  │  repo_model.Repository.LoadAttributes(ctx)
   │  └─ 字段裁剪转换
   │     convert.ToRepo(ctx, ctx.Repo.Repository, ctx.Repo.Permission)
   │
   ├─ Convert 层映射
   │  services/convert/repository.go:22 → ToRepo()
   │  └─ services/convert/repository.go:26 → innerToRepo()
   │     └─ 逐字段赋值给 api.Repository
   │
   ├─ API 结构体定义
   │  modules/structs/repo.go:59 → Repository struct
   │  └─ json tag 控制输出字段
   │
   └─ JSON 序列化
      ctx.JSON(http.StatusOK, apiRepo)
      └─ encoding/json.Marshal(apiRepo)
```

---

## 六、核心组件文件索引

| 组件 | 文件路径 | 关键行号 | 说明 |
|-----|---------|---------|------|
| Web 上下文 | `services/context/context.go` | 41, 124, 160 | Context 结构体、NewWebContext、Contexter 中间件 |
| API 上下文 | `services/context/api.go` | 34, 225 | APIContext 结构体、APIContexter 中间件 |
| Web 渲染 | `services/context/context_response.go` | 81 | Context.HTML() 渲染入口 |
| 模板引擎 | `modules/templates/page.go` | 23, 47, 62 | PageRenderer、HTML 渲染方法 |
| 模板上下文 | `services/context/context_template.go` | 23, 101 | TemplateContext、辅助函数注入 |
| Convert 层入口 | `services/convert/convert.go` | 44 | ToEmail、ToBranch 等转换函数 |
| Convert 层仓库 | `services/convert/repository.go` | 22, 26 | ToRepo、innerToRepo 字段映射 |
| API 结构体 | `modules/structs/repo.go` | 59 | Repository 结构体定义 |
| Web Handler 示例 | `routers/web/repo/view_home.go` | 389 | Home() 仓库首页 |
| API Handler 示例 | `routers/api/v1/repo/repo.go` | 500, 526 | Get() 获取仓库信息 |

---

## 七、字段裁剪机制总结

### 7.1 Web 页面：隐式裁剪

Web 页面**没有显式的字段裁剪层**，DB Model 直接注入 `ctx.Data`，裁剪发生在模板层面：

```go
// Handler 中直接注入完整 DB Model
ctx.Data["LatestRelease"] = release  // release 是 *repo_model.Release

// 模板中按需取用（隐式裁剪）
// templates/repo/home.tmpl
{{if .Data.LatestRelease}}
    <div class="release-info">
        <span class="tag">{{.Data.LatestRelease.TagName}}</span>
        <span class="title">{{.Data.LatestRelease.Title}}</span>
        {{/* 模板不访问的字段不会被输出 */}}
    </div>
{{end}}
```

**优点**：灵活，模板按需取用  
**风险**：敏感字段可能被模板意外输出

### 7.2 API 接口：显式裁剪

API 接口有**两层显式裁剪**：

#### 第一层：Convert 层代码裁剪

```go
// services/convert/repository.go:195
return &api.Repository{
    ID:          repo.ID,           // ✅ 保留
    Name:        repo.Name,         // ✅ 保留
    // 👇 内部字段被裁剪，不传递给 API
    // repo.CreatedUnix → 不直接传递，转为 Created 后传递
    Created:     repo.CreatedUnix.AsTime(),
    // repo.UpdatedUnix → 同上
    Updated:     repo.UpdatedUnix.AsTime(),
    // repo.IsFork → 保留为 Fork
    Fork:        repo.IsFork,
    // 👇 完全裁剪的字段（不出现）
    // repo.OwnerID, repo.Units, 等内部字段
}
```

#### 第二层：JSON tag 裁剪

```go
// modules/structs/repo.go
type Repository struct {
    ID       int64  `json:"id"`
    Name     string `json:"name"`
    Parent   *Repository `json:"parent,omitempty"`  // omitempty: nil 不输出
    // 👇 没有 json tag 的字段不会被序列化
    InternalField string  // 不会出现在 JSON 输出中
}
```

**优点**：安全可控，统一管理  
**代价**：需要维护 convert 层代码，新增字段需同步更新

---

## 八、数据流向对比图

### 8.1 Web 页面数据流

```
DB Model (repo_model.Topic)
    │
    └─ ctx.Data["Topics"] = topics ─┐
                                  │
DB Model (repo_model.Release)     │
    │                             │
    └─ ctx.Data["LatestRelease"] ─┤
                                  │
DB Model (git_model.LanguageStat) │
    │                             │
    └─ ctx.Data["LanguageStats"] ─┤
                                  │
                                  ▼
                          ctx.Data (map[string]any)
                                  │
                                  ▼
                          Go Template Engine
                                  │
                                  ▼
                              HTML 输出
```

### 8.2 API 接口数据流

```
DB Model (repo_model.Repository)
    │
    ├─ repo.ID
    ├─ repo.Name
    ├─ repo.Description
    ├─ repo.IsPrivate
    ├─ repo.CreatedUnix
    └─ repo.NumStars
          │
          ▼
  convert.ToRepo() ──── 字段逐一映射
          │
          ▼
  api.Repository (modules/structs)
    ├─ ID: repo.ID
    ├─ Name: repo.Name
    ├─ Description: repo.Description
    ├─ Private: repo.IsPrivate
    ├─ Created: repo.CreatedUnix.AsTime()  ← 格式转换
    └─ Stars: repo.NumStars
          │
          ▼
  json.Marshal() ← 根据 json tag 输出
          │
          ▼
      JSON 输出
```

---

## 九、关键设计决策分析

### 9.1 为什么 Web 页面不经过 Convert 层？

1. **性能考量**：模板渲染本身需要遍历数据，多一层转换会增加开销
2. **灵活性**：页面可能需要展示 DB Model 的任意字段，统一转换层会成为瓶颈
3. **历史原因**：Gitea 继承自 Gogs，早期设计就是模板直接访问模型
4. **开发效率**：新增页面字段无需修改 convert 层，直接在模板取用

### 9.2 为什么 API 接口必须经过 Convert 层？

1. **契约稳定**：API 是公开契约，需要保持字段稳定，不能随 DB Model 变化而变化
2. **安全可控**：统一过滤敏感字段（如密码哈希、内部状态字段）
3. **版本兼容**：API 字段可以独立演进，不与 DB Model 强绑定
4. **格式标准化**：统一时间格式、枚举值映射、URL 生成等

### 9.3 `ctx.Data` vs `ctx.PageData` 的区别

| 特性 | `ctx.Data` | `ctx.PageData` |
|-----|-----------|---------------|
| 用途 | 模板渲染数据 | JavaScript 模块数据 |
| 访问方式 | 模板中 `.Data.Key` | `window.config.pageData.Key` |
| 传递方式 | 模板执行时传入 | 渲染到 `head.tmpl` 的 inline script |
| 数据类型 | 任意 Go 类型 | 需可序列化为 JSON |
| 典型数据 | 列表、对象、HTML 字符串 | 配置项、初始状态、API 响应 |

```go
// ctx.Data - 给 Go 模板用
ctx.Data["Topics"] = topics  // topics 是 Go 结构体切片

// ctx.PageData - 给 JavaScript 用
ctx.PageData["citationFileContent"] = content  // 会被 JSON 序列化
```

---

## 十、RepoAssignment 中间件：模板变量取值的前置注入

Web 页面渲染前，`RepoAssignment` 是最关键的仓库上下文准备中间件，决定了 `ctx.Data` 中所有仓库相关变量的可用性。

### 10.1 中间件执行链

**文件**: `services/context/repo.go:792`

```go
func RepoAssignment(ctx *Context) {
    repoAssignmentPreCheck(ctx)
    prepareData := repoAssignmentPrepareData(ctx)
    funcs := []func(ctx *Context, data *repoAssignmentPrepareDataStruct){
        repoAssignmentPrepareOwner,       // 1. 解析 Owner 用户
        repoAssignmentAutoRedirectWiki,   // 2. .wiki 后缀自动重定向
        repoAssignmentPrepareRepo,        // 3. 查找 Repository DB Model
        repoAssignmentLegacy,             // 4. 权限校验 + 核心注入
        repoAssignmentPrepareTemplateData,// 5. 批量注入模板变量
        repoAssignmentAutoRedirectNotReady,// 6. 迁移中/损坏仓库重定向
        repoAssignmentPrepareGitRepo,     // 7. 打开 GitRepo
        repoAssignmentPrepareRepoTransfer,// 8. 仓库转让信息
        repoAssignmentPrepareBranches,    // 9. 分支计数
        repoAssignmentPreparePullRequests,// 10. PR 上下文
        repoAssignmentHandleGoGet,        // 11. go-get meta
    }
    for _, f := range funcs {
        f(ctx, prepareData)
        if ctx.Written() { return }
    }
}
```

### 10.2 权限校验流程

**核心代码**: `repoAssignmentLegacy` → `services/context/repo.go:424`

```
请求进入
    ↓
repo.LoadOwner(ctx)          // 加载仓库 Owner
    ↓
┌─ ctx.DoerNeedTwoFactorAuth()?
│   YES → ctx.Repo.Permission = PermissionNoAccess()   // 2FA 未完成 = 无权限
│   NO  → ctx.Repo.Permission = GetDoerRepoPermission(ctx, repo, ctx.Doer)
    ↓
┌─ ctx.Repo.Permission.HasAnyUnitAccessOrPublicAccess()?
│   NO → ┌─ go-get=1? → EarlyResponseForGoGetMeta()    // go get 特殊处理
│        └─ 否则 → ctx.NotFound(nil)                    // 404
│   YES → 继续
    ↓
ctx.Data["Permission"] = &ctx.Repo.Permission           // 注入权限对象到模板
```

**关键点**：
- 2FA 强制模式下，未完成二次认证的用户被设为 `NoAccess`
- `canWriteAsMaintainer` 是额外的旁路检查：即使整体无 UnitAccess，如果用户作为 Maintainer 可写某分支，也允许访问
- `HasAnyUnitAccessOrPublicAccess` 对公开仓库的未登录用户也放行

### 10.3 模板变量注入完整清单

`repoAssignmentPrepareTemplateData` → `services/context/repo.go:587` 是**模板变量注入的核心函数**，在权限校验通过后执行：

| ctx.Data Key | 来源 | 代码行 | 说明 |
|-------------|------|-------|------|
| `RepoLink` | `repo.Link()` | 590 | 仓库链接 |
| `FeedURL` | `repo.Link()` | 591 | Feed URL |
| `RepoExternalIssuesLink` | `unit.ExternalTrackerConfig()` | 595 | 外部 Issue 跟踪链接 |
| `NumTags` | `db.Count[repo_model.Release]` | 598 | 标签数量 |
| `NumReleases` | `db.Count[repo_model.Release]` | 608 | Release 数量（受权限控制：无写权限则不含 Draft） |
| `Title` | `repo.Owner.Name + "/" + repo.Name` | 618 | 页面标题 |
| `PageTitleCommon` | `repo.Name + " - " + setting.AppName` | 619 | 通用标题 |
| `Repository` | `repo` (repo_model.Repository) | 620 | **完整 DB Model** |
| `Owner` | `ctx.Repo.Repository.Owner` | 621 | 仓库 Owner |
| `CanWriteCode` | `Permission.CanWrite(TypeCode)` | 622 | 是否可写代码 |
| `CanWriteIssues` | `Permission.CanWrite(TypeIssues)` | 623 | 是否可写 Issue |
| `CanWritePulls` | `Permission.CanWrite(TypePullRequests)` | 624 | 是否可写 PR |
| `CanWriteActions` | `Permission.CanWrite(TypeActions)` | 625 | 是否可写 Actions |
| `CanSignedUserFork` | `repo_module.CanUserForkRepo()` | 632 | 当前用户是否可 Fork |
| `UserAndOrgForks` | `repo_model.GetForksByUserAndOrgs()` | 639 | 用户/组织的 Fork 列表 |
| `ShowForkModal` | `len(userAndOrgForks) > 1 \|\| ...` | 644 | 是否显示 Fork 选择弹窗 |
| `RepoCloneLink` | `repo.CloneLink(ctx, ctx.Doer)` | 646 | 克隆链接（含用户信息） |
| `CloneButtonShowHTTPS` | `!setting.Repository.DisableHTTPGit` | 648 | 是否显示 HTTPS 克隆 |
| `CloneButtonShowSSH` | `!setting.SSH.Disabled && ...` | 649 | 是否显示 SSH 克隆 |
| `CloneButtonOriginLink` | `ctx.Data["RepoCloneLink"]` | 656 | 克隆按钮原始链接 |
| `RepoSearchEnabled` | `setting.Indexer.RepoIndexerEnabled` | 658 | 代码搜索是否启用 |
| `CodeIndexerUnavailable` | `!code_indexer.IsAvailable()` | 660 | 代码索引是否不可用 |
| `IsWatchingRepo` | `repo_model.IsWatching()` | 664 | 是否关注（需登录） |
| `IsStaringRepo` | `repo_model.IsStaring()` | 665 | 是否 Star（需登录） |
| `Permission` | `&ctx.Repo.Permission` | 450 | 权限对象（repoAssignmentLegacy 注入） |
| `RepoName` | `ctx.Repo.Repository.Name` | 463 | 仓库名 |
| `IsEmptyRepo` | `ctx.Repo.Repository.IsEmpty` | 464 | 是否空仓库 |
| `PullMirror` | `repo_model.GetMirrorByRepoID()` | 454 | Mirror 信息 |
| `BranchesCount` | `db.Count[git_model.Branch]` | 743 | 分支总数 |
| `BaseRepo` | `repo.BaseRepo` 或 `repo` | 754/758 | PR 基础仓库 |
| `PullRequestCtx` | `InitRepoPullRequestCtx()` | 755/759 | PR 上下文 |
| `RepoTransfer` | `repo_model.GetPendingRepositoryTransfer()` | 776 | 待转让信息 |
| `CanUserAcceptOrRejectTransfer` | `repoTransfer.CanUserAcceptOrRejectTransfer()` | 777 | 是否可接受/拒绝转让 |

### 10.4 模板中的权限变量取值路径

模板中权限检查主要通过 `.Permission` 对象和预计算的布尔值两条路径：

**路径 1：直接使用预计算布尔值**（由 `repoAssignmentPrepareTemplateData` 注入）

```
模板取值                  注入来源                              代码行
.CanWriteCode         ← Permission.CanWrite(TypeCode)        repo.go:622
.CanWriteIssues       ← Permission.CanWrite(TypeIssues)      repo.go:623
.CanWritePulls        ← Permission.CanWrite(TypePullRequests) repo.go:624
.CanWriteActions      ← Permission.CanWrite(TypeActions)     repo.go:625
```

模板示例：`templates/repo/issue/milestones.tmpl:10`
```html
{{if and (or .CanWriteIssues .CanWritePulls) (not .Repository.IsArchived)}}
```

**路径 2：通过 Permission 对象动态调用**（注入 `ctx.Data["Permission"]`）

```
模板取值                                  对应方法
.Permission.IsAdmin                   ← access_model.Permission.IsAdmin
.Permission.IsOwner                   ← access_model.Permission.IsOwner
.Permission.CanRead ctx.Consts.RepoUnitTypeCode     ← Permission.CanRead(TypeCode)
.Permission.CanWrite ctx.Consts.RepoUnitTypeCode    ← Permission.CanWrite(TypeCode)
.Permission.HasAnyUnitPublicAccess    ← Permission.HasAnyUnitPublicAccess
```

模板示例：`templates/repo/pulse.tmpl:21`
```html
{{if (or (.Permission.CanRead ctx.Consts.RepoUnitTypeIssues) (.Permission.CanRead ctx.Consts.RepoUnitTypePullRequests))}}
```

---

## 十一、RepoRefByType 中间件：Git 引用解析与变量注入

`RepoRefByType` 在 `RepoAssignment` 之后执行，负责解析当前查看的 Git 引用（分支/标签/Commit），并将结果注入 `ctx.Repo` 和 `ctx.Data`。

### 11.1 执行流程

**文件**: `services/context/repo.go:937`

```
请求进入 RepoRefByType(detectRefType)
    ↓
┌─ 仓库为空?
│   YES → 设置默认分支名，注入 BranchName/TreePath，返回
│   NO  → 继续
    ↓
┌─ 仓库迁移中/损坏?
│   YES → 返回（显示迁移中 UI）
│   NO  → 继续
    ↓
┌─ reqPath (路径参数 *) 为空?
│   YES → 使用默认分支
│   │     RefFullName = refs/heads/{DefaultBranch}
│   │     Commit = GetBranchCommit(DefaultBranch)
│   NO  → 解析 refShortName
│         ├─ detectRefType == "" (Legacy) → getRefNameLegacy()
│         │   依次尝试: Branch → Tag → Commit ID → 回退默认分支
│         └─ detectRefType != "" → getRefName(ctx, repo, path, refType)
│             精确匹配指定类型的引用
    ↓
根据解析结果设置 ctx.Repo 字段:
    ├─ ctx.Repo.RefFullName   // 如 refs/heads/main
    ├─ ctx.Repo.BranchName    // 如 main
    ├─ ctx.Repo.Commit        // *git.Commit
    ├─ ctx.Repo.CommitID      // 如 a1b2c3d4
    ├─ ctx.Repo.TreePath      // 如 docs/README.md
    └─ ctx.Repo.CommitsCount  // int64
    ↓
注入到 ctx.Data:
    ├─ ctx.Data["RefFullName"]        // 完整引用名
    ├─ ctx.Data["RefTypeNameSubURL"]  // URL 子路径如 "branch/main"
    ├─ ctx.Data["TreePath"]           // 文件树路径
    ├─ ctx.Data["BranchName"]         // 分支名
    ├─ ctx.Data["CommitID"]           // Commit SHA
    ├─ ctx.Data["CanCreateBranch"]    // 是否可创建分支
    └─ ctx.Data["CommitsCount"]       // 提交数
```

### 11.2 Legacy 引用解析策略

`getRefNameLegacy` → `services/context/repo.go:832` 是 URL 中未明确指定 ref 类型时的回退解析：

```
输入: reqPath = "master/docs/README.md"
    ↓
1. 尝试作为 Branch: getRefName(ctx, repo, path, RefTypeBranch)
   ├─ "master" 是分支? → YES → RefFullName=refs/heads/master, TreePath=docs/README.md
   └─ NO → 继续尝试 "master/docs" 是否分支...
    ↓
2. 尝试作为 Tag: getRefName(ctx, repo, path, RefTypeTag)
   ├─ "master" 是标签? → YES → ...
   └─ NO → 继续尝试
    ↓
3. 尝试作为 Commit ID: IsStringLikelyCommitID("master")
   └─ 不是合法 SHA → 跳过
    ↓
4. 回退到默认分支: RefFullName=refs/heads/{DefaultBranch}
   TreePath = reqPath（整个路径作为文件路径）
```

**重命名分支处理**：如果引用名匹配了重命名分支，会自动重定向到新分支名。

---

## 十二、API repoAssignment 中间件：Token 可见性与权限校验

### 12.1 执行流程

**文件**: `routers/api/v1/api.go:134`

```
HTTP Request (API)
    ↓
[tokenRequiresScopes] → api.go:319
    │  ├─ 解析 API Token Scope
    │  ├─ 验证 Scope 包含必需的 Category (如 Repository)
    │  ├─ 检查 Read/Write 级别与 HTTP Method 匹配
    │  └─ 设置 ctx.PublicOnly = scope.PublicOnly()
    ↓
[repoAssignment] → api.go:134
    │
    ├─ 1. 解析 Owner 用户
    │     ├─ 已登录且名字匹配 → ctx.Doer
    │     └─ 否则 → GetUserByName()，处理重定向
    │
    ├─ 2. 查找 Repository
    │     └─ repo_model.GetRepositoryByName()
    │        └─ 不存在则尝试重定向
    │
    ├─ 3. 权限计算（三层判断）
    │     ├─ Actions Task 用户 → GetActionsUserRepoPermission()
    │     ├─ 需要 2FA → PermissionNoAccess()
    │     └─ 普通用户 → GetDoerRepoPermission(ctx, repo, ctx.Doer)
    │
    ├─ 4. 访问控制（两道关卡）
    │     ├─ !Permission.HasAnyUnitAccessOrPublicAccess() → 404
    │     └─ !TokenCanAccessRepo(repo) → 404
    │
    └─ 5. 注入到 ctx.Repo
          ctx.Repo.Owner = owner
          ctx.Repo.Repository = repo
          ctx.Repo.Permission = permission
```

### 12.2 Token 可见性限制详解

API 的 Token 可见性限制与 Web 页面的 Session 认证有本质区别：

#### `ctx.PublicOnly` 的设置链路

```
tokenRequiresScopes() → api.go:319
    ↓
检查 ctx.Data["IsApiToken"] == true?
    ↓ YES
解析 scope = ctx.Data["ApiTokenScope"]
    ↓
scope.PublicOnly() → 判断 Scope 是否仅限公共资源
    ↓
ctx.PublicOnly = publicOnly  // 赋值到 APIContext
```

**`PublicOnly` 的来源**：Token 的 Scope 中可能包含 `public-only` 限定符，表示该 Token 只能访问公开资源。

#### `TokenCanAccessRepo` 的检查逻辑

**文件**: `services/context/api.go:53`

```go
func (ctx *APIContext) TokenCanAccessRepo(repo *repo_model.Repository) bool {
    return repo == nil || !ctx.PublicOnly || !repo.IsPrivate
}
```

```
TokenCanAccessRepo(repo) 判定:
    ├─ repo == nil → true (无仓库限制)
    ├─ !ctx.PublicOnly → true (非 Public-Only Token，不受限)
    └─ !repo.IsPrivate → true (公开仓库，可访问)
    
    仅当 PublicOnly=true && repo.IsPrivate=true 时 → false (拒绝)
```

#### `checkTokenPublicOnly` 中间件

**文件**: `routers/api/v1/api.go:246`

在 `repoAssignment` 之后执行，针对特定端点进行更细粒度的 Public-Only 检查：

```go
func checkTokenPublicOnly() func(ctx *context.APIContext) {
    return func(ctx *context.APIContext) {
        if !ctx.PublicOnly { return }  // 非 Public-Only Token，放行
        // 检查请求的 Scope Category 是否允许访问私有资源
        // 例如: 写操作在 Public-Only Token 下被拒绝
    }
}
```

#### `rejectPublicOnly` 中间件

某些端点完全禁止 Public-Only Token 访问：

```go
func rejectPublicOnly() func(ctx *context.APIContext) {
    return func(ctx *context.APIContext) {
        if !ctx.PublicOnly { return }
        ctx.APIError(http.StatusForbidden, "this endpoint is not available for public-only tokens")
    }
}
```

### 12.3 Web 与 API 权限校验对比

| 校验维度 | Web (`RepoAssignment`) | API (`repoAssignment`) |
|---------|----------------------|----------------------|
| **认证方式** | Session + Cookie | Bearer Token / OAuth2 |
| **2FA 处理** | `DoerNeedTwoFactorAuth()` → NoAccess | `doerNeedTwoFactorAuth()` → NoAccess |
| **权限计算** | `GetDoerRepoPermission(ctx, repo, doer)` | `GetDoerRepoPermission(ctx, repo, doer)` 或 `GetActionsUserRepoPermission()` |
| **访问拒绝** | `ctx.NotFound(nil)` (404 页面) | `ctx.APIErrorNotFound()` (404 JSON) |
| **Token Scope** | ❌ 不适用 | ✅ `tokenRequiresScopes()` 检查 Category |
| **Public-Only** | ❌ 不适用 | ✅ `TokenCanAccessRepo()` + `checkTokenPublicOnly()` |
| **go-get 特例** | ✅ `go-get=1` 返回 meta | ❌ 无 |
| **canWriteAsMaintainer** | ✅ 旁路检查 | ❌ 无 |
| **权限对象注入** | `ctx.Data["Permission"] = &ctx.Repo.Permission` | 不注入模板，通过 convert 层使用 |

### 12.4 API 路由中间件栈典型示例

```
/repos/{username}/{reponame} 组路由:
    ↓
tokenRequiresScopes(AccessTokenScopeCategoryRepository)   // 检查 Token 有 Repository Scope
    ↓
repoAssignment()                                           // 解析仓库、计算权限
    ↓
checkTokenPublicOnly()                                     // 检查 Public-Only 限制
    ↓
具体 Handler:
    ├─ Get()         → ctx.JSON(200, convert.ToRepo(...))  // 读取
    ├─ Edit()        → ctx.JSON(200, convert.ToRepo(...))  // 修改
    └─ Delete()      → ...                                  // 删除
```

---

## 十三、模板取数路径与 Convert 字段裁剪对应关系

### 13.1 同一 DB Model 在 Web 模板与 API 中的取值路径对比

以 `repo_model.Repository` 为例，展示模板直接访问与 Convert 裁剪后的对应关系：

| DB Model 字段 | Web 模板取值路径 | API Convert 裁剪 | 差异说明 |
|-------------|----------------|-----------------|---------|
| `repo.ID` | `.Repository.ID` | `api.Repository.ID` → `json:"id"` | 直传 |
| `repo.Name` | `.Repository.Name` / `.RepoName` | `api.Repository.Name` → `json:"name"` | 模板有额外 `.RepoName` 快捷方式 |
| `repo.Owner` | `.Repository.Owner` / `.Owner` | `api.Repository.Owner` → `ToUser()` 裁剪 | 模板直接用 DB Model；API 经过 ToUser 裁剪 |
| `repo.IsPrivate` | `.Repository.IsPrivate` | `api.Repository.Private` → `json:"private"` | 字段名重命名 Is→无前缀 |
| `repo.IsFork` | `.Repository.IsFork` | `api.Repository.Fork` → `json:"fork"` | 字段名重命名 |
| `repo.IsArchived` | `.Repository.IsArchived` | `api.Repository.Archived` → `json:"archived"` | 字段名重命名 |
| `repo.IsEmpty` | `.Repository.IsEmpty` / `.IsEmptyRepo` | `api.Repository.Empty` → `json:"empty"` | 模板有额外 `.IsEmptyRepo` |
| `repo.NumStars` | `.Repository.NumStars` | `api.Repository.Stars` → `json:"stars_count"` | 字段名+json tag 双重重命名 |
| `repo.NumForks` | `.Repository.NumForks` | `api.Repository.Forks` → `json:"forks_count"` | 同上 |
| `repo.NumWatches` | - | `api.Repository.Watchers` → `json:"watchers_count"` | Web 不直接用，用 `.IsWatchingRepo` 布尔值 |
| `repo.NumOpenIssues` | - | `api.Repository.OpenIssues` → `json:"open_issues_count"` | Web 不直接用此字段 |
| `repo.NumOpenPulls` | - | `api.Repository.OpenPulls` → `json:"open_pr_counter"` | Web 不直接用此字段 |
| `repo.DefaultBranch` | `.Repository.DefaultBranch` | `api.Repository.DefaultBranch` → `json:"default_branch"` | 直传 |
| `repo.CreatedUnix` | `.Repository.CreatedUnix` (调用方法) | `api.Repository.Created` → `json:"created_at"` (AsTime()) | 模板用 Unix+方法；API 转为 time.Time |
| `repo.UpdatedUnix` | `.Repository.UpdatedUnix` (调用方法) | `api.Repository.Updated` → `json:"updated_at"` (AsTime()) | 同上 |
| `repo.Description` | `.Repository.Description` | `api.Repository.Description` → `json:"description"` | 直传 |
| `repo.Website` | `.Repository.Website` | `api.Repository.Website` → `json:"website"` | 直传 |
| `repo.OwnerID` | ❌ 不暴露 | ❌ 裁剪掉 | 内部字段，两边都不暴露 |
| `repo.Units` | 通过方法间接访问 | ❌ 裁剪掉，转为 `HasXxx` 布尔值 | 模板用 `Repository.UnitEnabled()`；API 用 `HasIssues`/`HasWiki` 等 |
| 权限信息 | `.Permission` (完整对象) | `api.Repository.Permissions` → `json:"permissions"` (简化为 admin/push/pull) | 模板可调用 `.Permission.CanWrite(TypeCode)` 等；API 只有三个布尔 |
| 克隆链接 | `.RepoCloneLink` / `.CloneButtonOriginLink` | `api.Repository.SSHURL` / `CloneURL` | 模板注入 CloneLink 对象；API 拆为两个字段 |
| 分支数 | `.BranchesCount` | `api.Repository.BranchCount` → `json:"branch_count"` | 模板从 `repoAssignmentPrepareBranches` 注入；API 从 `CountBranches` |
| Release 数 | `.NumReleases` | `api.Repository.Releases` → `json:"release_counter"` | 模板受权限控制（无写权限不含 Draft）；API 含非 Draft |
| 标签数 | `.NumTags` | ❌ API 无此字段 | Web 侧独立计算注入 |
| Star/Watch | `.IsStaringRepo` / `.IsWatchingRepo` (布尔) | ❌ API Repository 无此字段 | Web 直接计算；API 需单独端点 |
| Fork 相关 | `.CanSignedUserFork` / `.ShowForkModal` / `.UserAndOrgForks` | ❌ API 无此字段 | Web 侧 UI 专用计算 |

### 13.2 Permission 对象在两条路径中的形态差异

**Web 模板路径** — 完整 `access_model.Permission` 对象：

```
ctx.Data["Permission"] = &ctx.Repo.Permission

模板可调用:
    .Permission.CanWrite(unit.TypeCode)      // 细粒度单元权限
    .Permission.CanRead(unit.TypeIssues)      // 细粒度单元权限
    .Permission.IsAdmin                       // 管理员
    .Permission.IsOwner                       // 所有者
    .Permission.HasAnyUnitPublicAccess        // 公共访问
```

**API Convert 路径** — 简化为三个布尔：

```
convert.ToRepo() →
    permission := &api.Permission{
        Admin: permissionInRepo.AccessMode >= perm.AccessModeAdmin,
        Push:  permissionInRepo.UnitAccessMode(TypeCode) >= perm.AccessModeWrite,
        Pull:  permissionInRepo.UnitAccessMode(TypeCode) >= perm.AccessModeRead,
    }

JSON 输出:
    "permissions": {
        "admin": false,
        "push": true,
        "pull": true
    }
```

**核心差异**：
- Web 模板的 Permission 支持按**单元类型**（Code/Issues/PR/Wiki/Actions 等）分别检查读写权限
- API 的 Permission 只输出 Code 单元级别的 Admin/Push/Pull 三个布尔值
- 模板可做 `Permission.CanRead(TypeIssues)` 细粒度控制；API 消费者需自行从 `has_issues` 等布尔字段推断

### 13.3 动态字段裁剪对应关系总结

```
                    DB Model 层
                        │
          ┌─────────────┼─────────────┐
          │                           │
    Web 模板路径                  API Convert 路径
          │                           │
   RepoAssignment 注入            convert.ToRepo()
          │                           │
   ┌──────▼──────┐              ┌──────▼──────┐
   │ ctx.Data    │              │ api.Repo    │
   │ (无裁剪)    │              │ (逐字段映射) │
   └──────┬──────┘              └──────┬──────┘
          │                           │
   ┌──────▼──────┐              ┌──────▼──────┐
   │ 模板按需    │              │ JSON tag    │
   │ 隐式裁剪    │              │ + omitempty │
   └──────┬──────┘              └──────┬──────┘
          │                           │
          ▼                           ▼
     HTML 输出                    JSON 输出
  (只含模板引用的)           (只含 Convert 映射的
   DB Model 字段)             + omitempty 过滤)
```

**典型裁剪差异**：

| 裁剪类型 | Web 模板 | API Convert |
|---------|---------|-------------|
| 字段名映射 | 无（保持 DB 原名） | `IsPrivate` → `Private`，`NumStars` → `Stars` |
| 时间格式 | `CreatedUnix` (Unix 时间戳+模板方法) | `Created` (time.Time，JSON 自动 RFC3339) |
| 关联对象 | 直接访问 `.Repository.Owner.*` | `ToUser()` 裁剪后注入 `Owner` |
| 权限粒度 | 按 Unit Type 细粒度检查 | 简化为 Admin/Push/Pull |
| 计数字段 | `.NumReleases`（受权限控 Draft） | `Releases`（不含 Draft） |
| UI 专用字段 | `.CanSignedUserFork`/`.ShowForkModal` | 完全不存在 |
| 内部字段 | 模板不引用则不输出 | Convert 层直接不映射 |
| omitempty | 不适用 | `Parent`/`Permissions` 等空值不输出 |
