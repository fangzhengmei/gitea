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
