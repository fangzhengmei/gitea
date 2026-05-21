# Gitea 制品仓库（Package Registry）代码链路分析

## 一、整体架构概览

Gitea 的制品仓库系统在代码中被称为 **Package Registry**，支持 23 种包管理类型。整个系统可以划分为以下核心层次：

```
┌─────────────────────────────────────────────────────────┐
│  协议适配层 (routers/api/packages/*)                    │
│  每种包类型独立路由 + Handler，适配原生协议               │
├─────────────────────────────────────────────────────────┤
│  认证准入层 (services/packages/auth.go + auth/)         │
│  JWT Bearer Token / Basic Auth / OAuth2 / 反向代理       │
├─────────────────────────────────────────────────────────┤
│  业务服务层 (services/packages/packages.go)             │
│  包的创建/删除/下载、配额校验、通知分发                   │
├─────────────────────────────────────────────────────────┤
│  数据模型层 (models/packages/*.go)                      │
│  Package / PackageVersion / PackageFile / PackageBlob   │
├─────────────────────────────────────────────────────────┤
│  内容存储层 (modules/packages/content_store.go)         │
│  ContentStore → ObjectStorage → Local/MinIO/Azure       │
└─────────────────────────────────────────────────────────┘
```

---

## 二、路由入口与挂载

### 2.1 路由注册

在 `routers/init.go:189-195`，路由根据 `setting.Packages.Enabled` 配置条件挂载：

```go
// 绝大多数包管理器的路由挂载在 /api/packages 下
r.Mount("/api/packages", packages_router.CommonRoutes())
// OCI 容器镜像协议必须挂载在根路径 /v2（符合 OCI 分发规范）
r.Mount("/v2", packages_router.ContainerRoutes())
```

### 2.2 CommonRoutes 与 ContainerRoutes

两条路由构建函数定义在 `routers/api/packages/api.go`：

- **`CommonRoutes()`** — 挂载 22 种包管理器的路由（除 Container 外），统一前缀 `/api/packages/{username}/`
- **`ContainerRoutes()`** — OCI 容器镜像协议，挂载在 `/v2/`，支持 Docker/Containerd 等客户端

两者共享相同的中间件组合模式：
1. `context.PackageContexter()` — 创建基础 Web Context
2. `verifyAuth()` — 执行认证，组装 `auth.Method` 组
3. 路径分组下叠加 `reqPackageAccess(perm.AccessModeRead/Write)` — 细粒度权限校验

---

## 三、认证准入链路

### 3.1 认证方法组合

在 `routers/api/packages/api.go:96-128`，`verifyAuth` 函数通过 `auth.NewGroup` 组装多种认证方法：

```go
verifyAuth(r, []auth.Method{
    &auth.OAuth2{},       // OAuth2 Token
    &auth.Basic{},        // Basic Auth（用户名/密码 或 用户名/Token）
    &nuget.Auth{},        // NuGet 专用的 API Key 认证
    &Auth{},              // 自定义 JWT Bearer Token（conan/container 共用）
    &chef.Auth{},         // Chef 专用认证
}, verifyAuthOptions)
```

Container 路由使用不同的认证组：
```go
verifyAuth(r, []auth.Method{
    &auth.Basic{},
    &Auth{AllowGhostUser: true},  // 允许匿名 Ghost 用户
}, ...)
```

### 3.2 Auth.Group 执行流程

`services/auth/group.go:44-72` 定义了 `Group.Verify`：

```
for each method in methods:
    user, err = method.Verify(req, w, store, sess)
    if err != nil: continue（尝试下一个方法）
    if user != nil: return user, nil（认证成功）
return nil, retErr（全部失败）
```

关键设计：**多种方法串行尝试**，这使得 OAuth2 和 conan.Auth 都能从 `Authorization: Bearer <token>` 头部读取 token——前者先尝试解析为 OAuth2，失败后后者再尝试解析为自定义 JWT。

### 3.3 各认证方法的细节

| 方法 | 提取凭证的位置 | 验证方式 | Scope 来源 |
|------|--------------|----------|-----------|
| `auth.OAuth2` | `Authorization: Bearer <token>` | OAuth2 access_token → DB 查询 → 用户 | token 自身的 scope |
| `auth.Basic` | `Authorization: Basic <base64(user:pass)>` | 优先尝试作为 access_token → 失败后尝试用户名/密码登录 | access_token 的 scope 或 `AccessTokenScopeAll` |
| `nuget.Auth` | `X-NuGet-ApiKey` 请求头 | 作为 access_token 查询 DB | access_token 的 scope |
| `packages.Auth` | `Authorization: Bearer <token>`（JWT） | HS256 签名验证，Claims 中含 `UserID` 和 `Scope` | JWT Claims 中的 `PackageMeta.Scope` |
| `chef.Auth` | 请求签名（`X-Ops-Sign`、`X-Ops-Timestamp` 等头） | 用户公钥验证签名 | 空（chef 不使用 scope） |

**auth.Basic.Verify 执行顺序** (`services/auth/basic.go:125-175`):
1. 解析 `Authorization: Basic` 头
2. 尝试将密码部分作为 **OAuth2 access_token** 查找 (`GetOAuthAccessTokenScopeAndUserID`)
3. 失败后尝试将密码部分作为 **Personal Access Token** 查找 (`GetAccessTokenBySHA`)
4. 失败后尝试使用 **用户名/密码登录** (`UserSignIn`)，此时需 `EnableBasicAuth` 开启

### 3.4 自定义 JWT Token（packages.Auth）

定义于 `routers/api/packages/auth.go:17-60` 和 `services/packages/auth.go:21-88`。

**Token 签发** (`CreateAuthorizationToken`):
- Claims 结构：`{ UserID, Scope, ActionsUserTaskID }` + JWT 标准字段
- 签名算法：HS256，密钥取自 `setting.GetGeneralTokenSigningSecret()`
- 有效期：24 小时

**Token 验证** (`ParseAuthorizationRequest`):
1. 从 `Authorization` 头部提取 Bearer Token
2. 解析 JWT 得到 `PackageMeta{UserID, Scope, ActionsUserTaskID}`
3. 根据 UserID 查找用户（支持 GhostUser、ActionsUser）
4. 将 `Scope` 写入 `store.GetData()["ApiTokenScope"]`，供后续权限校验使用

**`AllowGhostUser` 的作用** (`routers/api/packages/auth.go:40-44`):
- Container 路由设置 `AllowGhostUser: true`
- 当 JWT 中 `UserID == GhostUserID`（-1）时，若 `AllowGhostUser` 为 `true` 则返回 GhostUser，否则返回 nil（继续尝试其他方法）
- 这使得匿名用户可通过 `/v2/token` 端点获取 GhostUser 的 JWT Token

### 3.5 从 username 到权限决策的完整流转

这是整个认证链路中最容易混淆的部分。以下逐步拆解一个请求如 `GET /api/packages/alice/npm/mypackage` 的完整处理流程：

```
Step 1: 路由匹配
        web.Router 将路径匹配到 r.Group("/{username}", ...)
        → 触发 AfterRouting 中间件链

Step 2: PackageContexter (AfterRouting #1)
        创建基础 Web Context（含 BaseContext、渲染器等）

Step 3: verifyAuth (AfterRouting #2)
        3a. 遍历 auth.Method 列表（OAuth2 → Basic → NuGet → JWT → Chef）
        3b. 第一个成功的方法返回 ctx.Doer（当前操作用户）
        3c. 将 Scope 写入 ctx.Data["ApiTokenScope"]，标记 IsApiToken
        3d. 若全部失败：ctx.Doer = nil, ctx.IsSigned = false

Step 4: 进入 r.Group("/{username}", ...) 的处理链
        顺序执行以下中间件：
        4a. UserAssignmentWeb()
            - 从 URL 提取 username = "alice"
            - 若 ctx.Doer 的 LowerName == username → ctx.ContextUser = ctx.Doer（跳过 DB 查询）
            - 否则 → user_model.GetUserByName(ctx, "alice") → ctx.ContextUser = 查到的用户
            - 若用户不存在 → 尝试 LookupUserRedirect（处理用户名变更）→ 仍不存在则 404
        4b. PackageAssignment()
            - 用 ctx.ContextUser 作为 pkgOwner
            - determineAccessMode(base, pkgOwner, ctx.Doer) → 计算访问模式
            - 根据 URL 中的 type/name/version 参数加载 PackageDescriptor
            - ctx.Package = {Owner, AccessMode, Descriptor}
        4c. reqPackageAccess(perm.AccessModeRead)
            - 第一级：若 IsApiToken → 检查 Scope 包含 ReadPackage → 若 PublicOnly 且 Owner 私有则 403
            - 第二级：若 ctx.Package.AccessMode < Read 且非管理员 → 401
```

**关键区分：`ctx.Doer` vs `ctx.ContextUser` vs `ctx.Package.Owner`**

| 变量 | 含义 | 来源 |
|------|------|------|
| `ctx.Doer` | 当前操作的用户（可能为 nil = 匿名） | auth.Group 认证结果 |
| `ctx.ContextUser` | URL 中 `{username}` 对应的用户（包的拥有者） | `user_model.GetUserByName` 或直接复用 `ctx.Doer` |
| `ctx.Package.Owner` | 同 `ctx.ContextUser`，用于包权限计算 | `PackageAssignment` 从 `ctx.ContextUser` 传递 |

**`userAssignment` 的短路优化** (`services/context/user.go:40-63`):
```go
if doer != nil && strings.EqualFold(doer.LowerName, username) {
    contextUser = doer  // 操作用户就是包拥有者，跳过 DB 查询
}
```

### 3.6 权限校验中间件 `reqPackageAccess`

定义于 `routers/api/packages/api.go:41-90`，执行两级检查：

**第一级：Token Scope 检查**（仅当 `IsApiToken == true` 时）
```go
case perm.AccessModeRead:
    scopeMatched, err = scope.HasScope(auth_model.AccessTokenScopeReadPackage)
case perm.AccessModeWrite:
    scopeMatched, err = scope.HasScope(auth_model.AccessTokenScopeWritePackage)
```
- 若 scope 限制为 "仅公开资源" (`PublicOnly`)，且目标包为私有，则拒绝

**第二级：访问模式检查**
```go
if ctx.Package.AccessMode < accessMode && !ctx.IsUserSiteAdmin() {
    // 返回 401 Unauthorized
}
```

### 3.7 访问模式计算 `determineAccessMode`

在 `services/context/package.go:116-170`：

| 场景 | 访问模式 |
|------|----------|
| `RequireSignInViewStrict` 启用且用户未登录/Ghost | `AccessModeNone` |
| 用户被禁用/禁止登录 | `AccessModeNone` |
| 组织 + 已登录成员 | 取组织团队最高授权 + `TypePackages` 单位的最大访问模式 |
| 组织 + 未登录 + Public | `AccessModeRead` |
| 组织 + 未登录 + Limited/Private | `AccessModeNone` |
| 组织 + 已登录非成员 + Public/Limited | `AccessModeRead` |
| 组织 + 已登录非成员 + Private | `AccessModeNone` |
| 个人 + 本人 | `AccessModeOwner` |
| 个人 + 公开/有限可见的他人 | `AccessModeRead` |
| 个人 + 公开 + 未登录 | `AccessModeRead` |

**关键修正**：
- 组织可见性 `Limited` 的定义是 **"Visible for every connected user"**（仅已登录用户可见），而非对匿名用户可见
- 匿名用户对 `Limited` 组织的访问模式为 `AccessModeNone`
- 这由 `HasOrgOrUserVisible` (`models/organization/org.go:423-425`) 保证：
  ```go
  if user == nil || user.IsGhost() {
      return orgOrUser.Visibility == structs.VisibleTypePublic  // 仅 Public
  }
  ```

---

## 四、匿名访问权限边界校准

### 4.1 RequireSignInViewStrict 开关的精确影响

`RequireSignInViewStrict` 是制品仓库匿名访问的总闸，配置项 `[service] REQUIRE_SIGNIN_VIEW = true/false`，定义于 `modules/setting/service.go:46,173-179`。

**开关的三个检查点**：

| 检查点 | 文件 | 逻辑 |
|--------|------|------|
| **1. determineAccessMode** | `services/context/package.go:117-119` | 若开关开启且 doer 为 nil/Ghost → `AccessModeNone` |
| **2. ReqContainerAccess** | `routers/api/packages/container/container.go:142-145` | 若开关开启且 doer 为 nil/Ghost → 直接 401 |
| **3. container.Authenticate** | `routers/api/packages/container/container.go:168-174` | 若开关开启且 doer 为 nil → 拒绝签发 Ghost Token |

**开关前后的匿名访问差异对照表**：

| 场景 | `RequireSignInViewStrict = false`（默认） | `RequireSignInViewStrict = true` |
|------|------------------------------------------|----------------------------------|
| **匿名用户（doer = nil）** | | |
| CommonRoutes 个人公开包 | `AccessModeRead` → 可读取 | `AccessModeNone` → 401 |
| CommonRoutes 个人私有包 | `AccessModeNone` → 401 | `AccessModeNone` → 401 |
| CommonRoutes 组织 Public 包 | `AccessModeRead` → 可读取 | `AccessModeNone` → 401 |
| CommonRoutes 组织 Limited 包 | **`AccessModeNone` → 401**（仅已登录可见） | `AccessModeNone` → 401 |
| CommonRoutes 组织 Private 包 | `AccessModeNone` → 401 | `AccessModeNone` → 401 |
| Container 组织 Public 包 | 可通过 `/v2/token` 获取 Ghost Token → 可读取 | `/v2/token` 返回 401 → 完全无法访问 |
| Container 组织 Limited 包 | **获取 Ghost Token 后仍 `AccessModeNone` → 401** | `/v2/token` 返回 401 → 完全无法访问 |
| Container 私有包 | 即使获取 Ghost Token → `AccessModeNone` → 401 | `/v2/token` 返回 401 → 完全无法访问 |
| **Ghost 用户（doer = GhostUser）** | | |
| CommonRoutes 个人/组织 Public 包 | `AccessModeRead` → 可读取 | `AccessModeNone` → 401 |
| CommonRoutes 组织 Limited 包 | **`AccessModeNone` → 401**（Ghost 视为匿名） | `AccessModeNone` → 401 |
| Container 组织 Public 包 | `AccessModeRead` → 可读取 | `ReqContainerAccess` 直接 401 |
| Container 组织 Limited 包 | **`AccessModeNone` → 401** | `ReqContainerAccess` 直接 401 |
| **写操作（PUT/POST/DELETE）** | 任何开关下，匿名用户/Ghost 用户均无法通过写操作的 `AccessMode >= Write` 检查 | 同左 |

**关键澄清**：
- 即使 `RequireSignInViewStrict = false`，**匿名用户也绝对无法写入任何包**（公开包也不行）。写入需要 `AccessMode >= Write`，而匿名用户最多只有 `AccessModeRead`。
- 开关只影响**读权限**的判定。
- `AccessMode` 枚举值：`None(0) < Read(1) < Write(2) < Admin(3) < Owner(4)`。

### 4.2 公开包场景下的实际读写边界

以下是匿名用户（未登录）对**公开个人包**（`pkgOwner.Visibility = Public`）的实际权限：

| 操作 | CommonRoutes（22 种类型） | Container |
|------|--------------------------|-----------|
| **读取包元数据** | ✅ 允许（`AccessModeRead`） | ✅ 允许（需先获取 Ghost Token） |
| **下载包文件** | ✅ 允许（`AccessModeRead`） | ✅ 允许（需先获取 Ghost Token） |
| **搜索/列举包** | ✅ 允许（`AccessModeRead`） | ✅ 允许（`/_catalog` 可见公开包） |
| **上传新版本** | ❌ 拒绝（`AccessModeRead < Write`） | ❌ 拒绝（`AccessModeRead < Write`） |
| **删除包/版本** | ❌ 拒绝（`AccessModeRead < Write`） | ❌ 拒绝（`AccessModeRead < Write`） |
| **覆盖文件** | ❌ 拒绝（`AccessModeRead < Write`） | ❌ 拒绝（`AccessModeRead < Write`） |
| **修改元数据** | ❌ 拒绝（`AccessModeRead < Write`） | ❌ 拒绝（`AccessModeRead < Write`） |
| **获取 NuGet 服务索引** | ✅ 允许（无保护端点） | N/A |
| **获取 Swift 认证质询** | ✅ 允许（无保护端点） | N/A |
| **获取 Vagrant 认证质询** | ✅ 允许（无保护端点） | N/A |

**对组织包的匿名访问**（基于 `HasOrgOrUserVisible` 精确逻辑）：
- 组织可见性为 Public → 匿名用户有 `AccessModeRead`（同公开个人包）
- 组织可见性为 Limited → **未登录匿名用户 = `AccessModeNone`**（Limited 仅对已登录用户可见），**已登录非成员 = `AccessModeRead`**
- 组织可见性为 Private → 匿名用户 `AccessModeNone`，仅成员可见

**错误点纠正**：原描述将 Limited 的匿名访问与已登录访问颠倒了。`Limited` 的定义是 "Visible for every connected user"（`modules/structs/visible_type.go:13`），即**仅已登录用户可见**，匿名用户无法访问 Limited 组织的任何包。

### 4.3 写操作的双重限制机制

写操作（PUT/POST/DELETE）受到**鉴权中间件**与**访问模式判定**的双重保护，即使访问模式计算出错，中间件仍然会拦截。

**第一重限制：`determineAccessMode` 计算出的 `AccessMode` 本身就不允许写入**

匿名用户对任何包（包括公开包）的 `AccessMode` 最大只能是 `Read`：
```go
// services/context/package.go:157-166
if pkgOwner.IsOrganization() {
    // ...
} else {
    if doer != nil && !doer.IsGhost() {
        if doer.ID == pkgOwner.ID {
            accessMode = perm.AccessModeOwner  // 仅本人能到 Owner
        } else if pkgOwner.Visibility == structs.VisibleTypePublic || pkgOwner.Visibility == structs.VisibleTypeLimited {
            accessMode = perm.AccessModeRead   // 他人最多 Read
        }
    } else if pkgOwner.Visibility == structs.VisibleTypePublic {
        accessMode = perm.AccessModeRead       // 未登录 + 公开 → Read
    }
}
```

**第二重限制：`reqPackageAccess(Write)` 中间件强制检查**

定义于 `routers/api/packages/api.go:41-90`：
```go
func reqPackageAccess(accessMode perm.AccessMode) func(ctx *context.Context) {
    return func(ctx *context.Context) {
        // Token Scope 检查（仅当 IsApiToken == true 时执行）
        if ctx.Data["IsApiToken"] == true {
            scope, ok := ctx.Data["ApiTokenScope"].(auth_model.AccessTokenScope)
            if ok {
                // 检查 scope 包含 ReadPackage/WritePackage
                // 检查 PublicOnly 限制
            }
        }
        // 访问模式检查（始终执行）
        if ctx.Package.AccessMode < accessMode && !ctx.IsUserSiteAdmin() {
            ctx.Resp.Header().Set("WWW-Authenticate", `Basic realm="Gitea Package API"`)
            ctx.HTTPError(http.StatusUnauthorized, "reqPackageAccess", "user should have specific permission or be a site admin")
            return
        }
    }
}
```

**错误点纠正：Scope 校验的门控条件**
- `IsApiToken` 仅在认证成功且使用了 Token（OAuth2/PAT/HTTP Sign）时才会被设置为 `true`
- **匿名用户（无 Token）→ `IsApiToken` 不存在或为 `false` → Scope 校验块完全跳过**
- 拦截匿名用户写操作的是**第二级的 `AccessMode` 检查**，而非 Scope 检查
- `IsApiToken` 的设置位置（搜索 `IsApiToken`）：
  - `services/auth/oauth2.go:124,145` — OAuth2 Token
  - `services/auth/basic.go:83,104` — Basic Auth 用 Token 认证
  - `services/auth/httpsign.go:81` — HTTP Sign 认证

**写操作的完整拦截逻辑（匿名用户）**：

```
匿名用户发起 PUT /api/packages/alice/generic/pkg/1.0/file
        │
        ▼
1. verifyAuth → ctx.Doer = nil, ctx.IsSigned = false
   └─ ctx.Data["IsApiToken"] 未设置 → Scope 校验将被跳过
        │
        ▼
2. UserAssignmentWeb → ctx.ContextUser = alice（公开用户）
        │
        ▼
3. PackageAssignment
   └─ determineAccessMode(base, alice, nil)
      └─ pkgOwner.Public + doer=nil → AccessModeRead
        │
        ▼
4. reqPackageAccess(perm.AccessModeWrite)
   ├─ Scope 检查：IsApiToken != true → 跳过
   └─ AccessMode 检查：Read(1) < Write(2) → 401 Unauthorized
        │
        ▼
   请求被拦截，Handler 永不执行
```

**校验逻辑的真实结构（而非双重冗余）**：
- 若 `determineAccessMode` 因 bug 给匿名用户错误返回 `AccessModeWrite`，则**两级检查都不会拦截**
  - Scope 检查因 `IsApiToken != true` 跳过
  - AccessMode 检查因 `Write < Write` 为 false 也通过
  - 此时匿名用户将能执行写操作（这是一个真实的安全风险点，而非冗余设计）
- 安全保障实际依赖于：`determineAccessMode` 的正确性 + 路由配置中 `reqPackageAccess(Write)` 中间件的正确叠加
- 所有 23 种包类型的 50+ 个写操作路径均显式叠加了 `reqPackageAccess(Write)` 中间件

**各类型写操作的中间件保护情况**（从 `routers/api/packages/api.go` 提取）：

| 包类型 | 写操作路径 | 中间件保护 |
|--------|----------|-----------|
| Alpine | `PUT /{branch}/{repository}` | `reqPackageAccess(Write)` |
| | `DELETE /.../{filename}` | `reqPackageAccess(Write)` |
| Arch | `PUT /*`, `PUT /<repository:*>` | `reqPackageAccess(Write)` |
| | `DELETE /.../<architecture>` | `reqPackageAccess(Write)` |
| Cargo | `PUT /api/v1/crates/new` | `reqPackageAccess(Write)` |
| | `DELETE /.../yank`, `PUT /.../unyank` | `reqPackageAccess(Write)` |
| Chef | `POST /api/v1/cookbooks` | `reqPackageAccess(Write)` |
| | `DELETE /.../{version}` | `reqPackageAccess(Write)` |
| Composer | `PUT /` | `reqPackageAccess(Write)` |
| Conan | `DELETE /.../delete`, `POST /.../upload_urls` | `reqPackageAccess(Write)` |
| | `PUT /.../upload` | `reqPackageAccess(Write)` |
| Conda | `PUT /<channel:*>/<filename>` | `reqPackageAccess(Write)` |
| CRAN | `PUT /src`, `PUT /bin` | `reqPackageAccess(Write)` |
| Debian | `PUT /pool/.../upload` | `reqPackageAccess(Write)` |
| | `DELETE /pool/.../{architecture}` | `reqPackageAccess(Write)` |
| Generic | `PUT /{packagename}/{packageversion}/{filename}` | `reqPackageAccess(Write)` |
| | `DELETE /.../{filename}` | `reqPackageAccess(Write)` |
| Go | `PUT /upload` | `reqPackageAccess(Write)` |
| Helm | `POST /api/charts` | `reqPackageAccess(Write)` |
| Maven | `PUT /*` | `reqPackageAccess(Write)` |
| npm | `PUT /@{scope}/{id}`, `PUT /{id}` | `reqPackageAccess(Write)` |
| | `DELETE /.../-rev/{revision}` | `reqPackageAccess(Write)` |
| NuGet | `PUT /`, `PUT /symbolpackage` | `reqPackageAccess(Write)` |
| | `DELETE /{id}/{version}` | `reqPackageAccess(Write)` |
| Pub | 整个 `/versions/new` Group | `reqPackageAccess(Write)` |
| PyPI | `POST /` | `reqPackageAccess(Write)` |
| RPM | `PUT /<group:*>/upload` | `reqPackageAccess(Write)` |
| | `DELETE /.../<architecture>` | `reqPackageAccess(Write)` |
| RubyGems | `POST /api/v1/gems`, `DELETE /.../yank` | `reqPackageAccess(Write)` |
| Swift | `PUT /{scope}/{name}/<version>` | `reqPackageAccess(Write)` |
| Terraform | `POST`, `DELETE /state/{name}` | `reqPackageAccess(Write)` |
| | `POST/DELETE /lock` | `reqPackageAccess(Write)` |
| Vagrant | `PUT /{name}/{version}/{provider}` | `reqPackageAccess(Write)` |
| Container | `POST /.../blobs/uploads` | `reqPackageAccess(Write)` |
| | `PATCH/PUT/DELETE /.../uploads/{uuid}` | `reqPackageAccess(Write)` |
| | `DELETE /.../blobs/<digest>` | `reqPackageAccess(Write)` |
| | `PUT/DELETE /.../manifests/<reference>` | `reqPackageAccess(Write)` |

**结论**：所有 23 种包类型的所有写操作路径（共 50+ 个）都显式叠加了 `reqPackageAccess(Write)` 中间件，无任何例外。匿名用户即使绕过认证（如通过无保护的协议端点），也无法通过第二重访问模式检查。

### 4.4 本次纠正的错误点汇总

| # | 原错误描述 | 正确代码逻辑 | 代码位置 |
|---|-----------|-------------|----------|
| **1** | `determineAccessMode` 表格中"组织 + 未登录/非成员 + 组织可见"合并为一行，未区分 Public/Limited | 需分四行区分：未登录+Public→Read，未登录+Limited→None，已登录非成员+Public/Limited→Read，已登录非成员+Private→None | `services/context/package.go:203-214` |
| **2** | 组织 Limited → "未登录匿名用户有 AccessModeRead，已登录非成员无" — 完全颠倒 | `HasOrgOrUserVisible` 中 `user == nil` 时仅返回 `Visibility == Public`，Limited 对匿名用户不可见；已登录非成员可见 Limited | `models/organization/org.go:423-425`; `modules/structs/visible_type.go:13` |
| **3** | "双重校验安全冗余：即使 determineAccessMode 返回 Write，Scope 检查也会拦截" | Scope 检查仅当 `IsApiToken == true` 时执行，匿名用户无 Token → `IsApiToken` 非 true → Scope 检查完全跳过；若 AccessMode 错误返回 Write，则两级都不拦截，是风险点而非冗余 | `routers/api/packages/api.go:43`; `services/auth/oauth2.go:124,145`; `services/auth/basic.go:83,104` |
| **4** | RequireSignInViewStrict 对照表未包含组织 Limited 场景 | 补充 6 行：CommonRoutes/Container 的组织 Limited 包在开关关闭时匿名用户仍为 `AccessModeNone` | `services/context/package.go:152-155` |
| **5** | 写操作时序图中未体现 Scope 检查跳过的情况 | 补充 Scope 检查判断分支，明确标注"IsApiToken != true → 跳过" | `routers/api/packages/api.go:43-82` |
| **6** | `reqPackageAccess` 代码注释写"Token Scope 检查（第一级）"暗示始终执行 | 正确注释应为"Token Scope 检查（仅当 IsApiToken == true 时执行）" | `routers/api/packages/api.go:43` |

---

## 五、各协议的匿名与认证例外路径

不同包类型因协议规范差异，在认证上有各自的例外。以下从路由注册代码 (`routers/api/packages/api.go`) 中提取：

### 5.1 标准模式（reqPackageAccess Read 在 Group 上）

22 种包类型（除 Container）的路由采用以下标准模式：

```go
r.Group("/{username}", func() {
    r.Group("/alpine", func() { ... }, reqPackageAccess(perm.AccessModeRead))
    r.Group("/arch",   func() { ... }, reqPackageAccess(perm.AccessModeRead))
    // ... 其他包类型
}, context.UserAssignmentWeb(), context.PackageAssignment())
```

在这种模式下，`reqPackageAccess(Read)` 作为 **Group 级中间件** 应用于每个包类型的所有路由。写操作路径再叠加 `reqPackageAccess(Write)`。

这意味着：**所有读操作必须至少有 Read 权限，匿名用户仅能访问公开包（AccessMode = Read）**。

### 5.2 例外协议详解

#### 5.2.1 NuGet — 服务发现端点无需认证

```go
r.Group("/nuget", func() {
    r.Group("", func() { // 无 reqPackageAccess，完全匿名
        r.Get("/", nuget.ServiceIndexV2)
        r.Get("/index.json", nuget.ServiceIndexV3)
        r.Get("/$metadata", nuget.FeedCapabilityResource)
    })
    r.Group("", func() { // reqPackageAccess(Read)
        r.Get("/query", nuget.SearchServiceV3)
        r.Get("/registration/{id}/index.json", nuget.RegistrationIndex)
        // ... 其他读操作
        r.Group("", func() { // reqPackageAccess(Write)
            r.Put("/", nuget.UploadPackage)
            r.Delete("/{id}/{version}", nuget.DeletePackage)
        }, reqPackageAccess(perm.AccessModeWrite))
    }, reqPackageAccess(perm.AccessModeRead))
})
```

**原因**：NuGet 客户端需要先获取服务索引（Service Index）来发现可用的 API 端点，此步骤按协议规范无需认证。

#### 5.2.2 Swift — 认证检查端点无需认证

```go
r.Group("/swift", func() {
    r.Group("", func() { // 无 reqPackageAccess
        r.Post("", swift.CheckAuthenticate)       // POST /{scope}/{name}
        r.Post("/login", swift.CheckAuthenticate)  // POST /{scope}/{name}/login
    })
    r.Group("", func() { // reqPackageAccess(Read)
        // ... 包元数据、下载、上传等操作
    }, reqPackageAccess(perm.AccessModeRead))
})
```

**原因**：Swift Package Manager 的认证协议要求先 `POST` 检查认证状态，即使匿名用户也需要得到 401 响应以触发客户端认证流程。

#### 5.2.3 Container（OCI）— 独立的认证体系

Container 使用完全不同的认证流程：

```go
// 认证方法组：仅 Basic + JWT（无 OAuth2/NuGet/Chef）
verifyAuth(r, []auth.Method{
    &auth.Basic{},
    &Auth{AllowGhostUser: true},  // ← 关键：允许 Ghost 用户
}, ...)

// 第一层中间件：ReqContainerAccess（替代 reqPackageAccess）
r.Get("", container.ReqContainerAccess, container.DetermineSupport)
r.Group("/token", func() { ... })
r.Get("/_catalog", container.ReqContainerAccess, ...)

// 第二层中间件：在 Group 中
r.Group("/{username}", func() { ... },
    container.ReqContainerAccess,
    context.UserAssignmentWeb(),
    context.PackageAssignment(),
    reqPackageAccess(perm.AccessModeRead))
```

**Container 匿名访问机制：**

1. `Auth{AllowGhostUser: true}` 允许 JWT 中 `UserID = GhostUserID` 通过
2. 匿名用户访问 `/v2/token` → `container.Authenticate` 签发 GhostUser 的 JWT
3. 后续请求携带此 JWT → `Auth.Verify` 返回 GhostUser → `ctx.Doer = GhostUser`
4. `ReqContainerAccess` 检查：若 `RequireSignInViewStrict` 未启用，允许 GhostUser 通过
5. `determineAccessMode` 计算：GhostUser + 公开包 → `AccessModeRead`
6. `reqPackageAccess(Read)` 检查通过

| 配置 | 匿名访问结果 |
|------|-------------|
| `RequireSignInViewStrict = false`（默认） | 匿名用户可**读**公开包（写操作仍被拒绝） |
| `RequireSignInViewStrict = true` | 匿名用户收到 401 + `WWW-Authenticate: Bearer` 头，完全无法访问 |

#### 5.2.4 Conan — 认证端点在 Read Group 内，但实际无保护

```go
r.Group("/conan", func() {
    r.Group("/v1", func() {
        r.Get("/ping", conan.Ping)
        r.Group("/users", func() {
            r.Get("/authenticate", conan.Authenticate)      // 返回 JWT Token
            r.Get("/check_credentials", conan.CheckCredentials)
        })
        // ...
    })
}, reqPackageAccess(perm.AccessModeRead))
```

Conan 的 `authenticate` 端点返回用于后续请求的 Bearer Token。由于它在 `reqPackageAccess(Read)` 组内，匿名用户也能访问（公开包即可读）。首次认证通过 Basic Auth，后续请求使用返回的 JWT Token。

#### 5.2.5 Vagrant — 认证检查端点无保护

```go
r.Group("/vagrant", func() {
    r.Group("/authenticate", func() {
        r.Get("", vagrant.CheckAuthenticate)  // 无 reqPackageAccess
    })
    r.Group("/{name}", func() { ... }, reqPackageAccess(perm.AccessModeRead))
}, reqPackageAccess(perm.AccessModeRead))
```

**原因**：Vagrant 客户端需要先检查认证状态，然后根据响应决定是否携带凭证。

#### 5.2.6 Terraform — 写操作在 Read Group 内单独保护

```go
r.Group("/terraform/state/{name}", func() {
    r.Get("", terraform.GetTerraformState)                    // 读：仅需 Group 级 Read
    r.Get("/versions/{serial}", terraform.GetTerraformStateBySerial)  // 读：仅需 Group 级 Read
    r.Group("", func() {
        r.Post("", terraform.UploadState)                     // 写：需 Write
        r.Delete("", terraform.DeleteState)                   // 写：需 Write
    }, reqPackageAccess(perm.AccessModeWrite))
    r.Group("/lock", func() {
        r.Post("", terraform.LockState)                       // 写：需 Write
        r.Delete("", terraform.UnlockState)                   // 写：需 Write
    }, reqPackageAccess(perm.AccessModeWrite))
}, reqPackageAccess(perm.AccessModeRead))
```

#### 5.2.7 Pub (Dart/Flutter) — 上传路径的 Write 保护在子 Group

```go
r.Group("/pub", func() {
    r.Group("/api/packages", func() {
        r.Group("/versions/new", func() {
            r.Get("", pub.RequestUpload)          // 读：获取上传临时 URL
            r.Post("/upload", pub.UploadPackageFile) // 写：实际上传
            r.Get("/finalize/{id}/{version}", pub.FinalizePackage)
        }, reqPackageAccess(perm.AccessModeWrite)) // 整个子 Group 需 Write
        r.Group("/{id}", func() {
            r.Get("", pub.EnumeratePackageVersions)    // 读
            r.Get("/files/{version}", pub.DownloadPackageFile) // 读
            r.Get("/{version}", pub.PackageVersionMetadata)   // 读
        })
    })
}, reqPackageAccess(perm.AccessModeRead))
```

注意：Pub 的 `RequestUpload`（GET）获取上传临时 URL 也在 Write 保护下，因为获取上传凭证需要写权限。

### 5.3 例外汇总表

| 包类型 | 无认证保护的端点 | 原因 |
|--------|-----------------|------|
| **NuGet** | `GET /`, `GET /index.json`, `GET /$metadata` | 服务发现协议要求 |
| **Swift** | `POST /`, `POST /login` | 认证检查协议要求 |
| **Container** | `GET /v2/token`（签发 GhostUser Token） | OCI Token 认证流程 |
| **Conan** | `GET /v1/ping`, `GET /v1/users/authenticate`, `GET /v2/ping`, `GET /v2/users/authenticate` | 认证握手协议要求（实际在 Read Group 内，匿名用户可访问公开包） |
| **Vagrant** | `GET /authenticate` | 认证检查协议要求 |
| **Terraform** | 无 | 全部在 Read Group 内，写操作单独加 Write |
| **Pub** | 无 | 上传操作在 Write Group 内 |

其余所有包类型（Alpine, Arch, Cargo, Chef, Composer, Conda, CRAN, Debian, Go, Generic, Helm, Maven, npm, PyPI, RPM, RubyGems）的所有端点均在 `reqPackageAccess(Read)` 保护下，无匿名例外。

---

## 六、协议适配层

### 6.1 支持的包类型（23 种）

在 `models/packages/package.go:32-56` 定义：

| 常量 | 类型 | 路由前缀 |
|------|------|----------|
| `TypeAlpine` | Alpine Linux 包 | `/alpine` |
| `TypeArch` | Arch Linux 包 | `/arch` |
| `TypeCargo` | Rust Cargo | `/cargo` |
| `TypeChef` | Chef Cookbook | `/chef` |
| `TypeComposer` | PHP Composer | `/composer` |
| `TypeConan` | C/C++ Conan | `/conan` |
| `TypeConda` | Conda 包 | `/conda` |
| `TypeContainer` | OCI 容器镜像 | `/v2`（独立） |
| `TypeCran` | R CRAN | `/cran` |
| `TypeDebian` | Debian 包 | `/debian` |
| `TypeGeneric` | 通用文件 | `/generic` |
| `TypeGo` | Go Modules | `/go` |
| `TypeHelm` | Kubernetes Helm | `/helm` |
| `TypeMaven` | Java Maven | `/maven` |
| `TypeNpm` | JavaScript npm | `/npm` |
| `TypeNuGet` | .NET NuGet | `/nuget` |
| `TypePub` | Dart/Flutter Pub | `/pub` |
| `TypePyPI` | Python PyPI | `/pypi` |
| `TypeRpm` | RPM 包 | `/rpm` |
| `TypeRubyGems` | Ruby Gems | `/rubygems` |
| `TypeSwift` | Swift Package | `/swift` |
| `TypeTerraformState` | Terraform State | `/terraform` |
| `TypeVagrant` | Vagrant Box | `/vagrant` |

### 6.2 Generic 包上传链路（典型示例）

以 `routers/api/packages/generic/generic.go` 的 `UploadPackage` 为例，展示标准上传流程：

```
1. 参数校验（正则匹配包名/文件名/版本号）
2. ctx.UploadStream() → 获取上传流
3. packages_module.CreateHashedBufferFromReader(upload) → 计算多哈希 + 写入 FileBackedBuffer
4. packages_service.CreatePackageOrAddFileToExisting() → 核心创建逻辑
5. 返回 201 Created
```

### 6.3 Container 包上传链路（OCI 协议）

容器镜像走 OCI 分发规范，分为 Blob Upload 和 Manifest Push：

**Blob 上传（`routers/api/packages/container/container.go`）:**

1. **POST `/v2/{image}/blobs/uploads`** — 创建上传会话
   - `mount` 参数：尝试从另一个仓库挂载已有 blob（跨仓库复用）
   - `digest` 参数：monolithic upload（单请求直接提交）
   - 默认：返回 `Location` 和 `Docker-Upload-Uuid`，等待后续 PATCH/PUT

2. **PATCH `/v2/{image}/blobs/uploads/{uuid}`** — 分块追加数据
   - 通过 `container_service.NewBlobUploader` 创建上传器
   - 支持 `Content-Range` 校验，保证顺序追加
   - 数据写入临时文件 + 同步计算哈希

3. **PUT `/v2/{image}/blobs/uploads/{uuid}?digest=...`** — 完成上传
   - 校验客户端提供的 digest 与服务端计算的哈希一致
   - 调用 `saveAsPackageBlob` 持久化
   - 删除 `package_blob_upload` 记录

**Manifest 推送（`PutManifest`）:**
- 读取最多 10MB 的 manifest 内容
- `processManifest` 解析并创建/关联包版本、文件、tag
- 支持按 digest 和按 tag 两种引用方式

### 6.4 各包类型 Handler 的共性模式

每种包类型的 Handler 都遵循以下模式：

```
请求进入 → 协议特定解析 → packages_service 调用 → 协议特定响应
```

核心服务调用集中在 `services/packages/packages.go` 中，典型函数：

| 函数 | 用途 |
|------|------|
| `CreatePackageAndAddFile` | 创建包+版本+文件（拒绝重复版本） |
| `CreatePackageOrAddFileToExisting` | 创建或追加文件到已有版本 |
| `AddFileToExistingPackage` | 向已存在的包版本追加文件 |
| `RemovePackageVersionByNameAndVersion` | 按名称删除版本 |
| `OpenFileForDownloadByPackageNameAndVersion` | 下载文件（含配额、计数、直链） |
| `RemovePackage` | 删除整个包及其所有版本 |

---

## 七、业务服务层

### 7.1 包创建核心流程 `createPackageAndAddFile`

定义于 `services/packages/packages.go:85-127`，是所有包上传操作的最终汇聚点：

```
1. db.TxContext() → 开启数据库事务
2. createPackageAndVersion() → 创建/获取 Package + PackageVersion
   ├─ TryInsertPackage() → 幂等插入 Package（按 OwnerID+Type+LowerName 唯一）
   ├─ 插入 PackageProperties
   ├─ GetOrInsertVersion() → 幂等插入 PackageVersion（按 PackageID+LowerVersion 唯一）
   ├─ CheckCountQuotaExceeded() → 校验版本数量配额
   └─ 插入 VersionProperties
3. addFileToPackageVersion() → 创建文件记录 + 存储 Blob
   ├─ CheckSizeQuotaExceeded() → 校验大小配额
   ├─ packages_model.GetOrInsertBlob() → 幂等插入 PackageBlob（按全部哈希唯一）
   │   └─ 若 blob 不存在 → contentStore.Save() → 写入底层存储
   ├─ 处理 OverwriteExisting 逻辑（删除旧文件记录）
   └─ TryInsertFile() → 插入 PackageFile
4. committer.Commit() → 提交事务
5. 若新建版本 → notify_service.PackageCreate() → 发送通知
```

**关键设计：**
- **Blob 去重**：`PackageBlob` 表通过 `(size, hash_md5, hash_sha1, hash_sha256, hash_sha512)` 联合唯一约束实现内容寻址，相同内容只存一次
- **延迟存储删除**：删除包版本时不立即删除底层 blob，由 `cleanup_packages` 定时任务清理（参见 `services/packages/packages.go:487` 注释）
- **事务保护**：整个创建过程在一个数据库事务中，任何步骤失败都会回滚，已写入的 blob 会被清理

### 7.2 配额校验

**数量配额** (`CheckCountQuotaExceeded`, `packages.go:330-350`)：
```go
if setting.Packages.LimitTotalOwnerCount > -1 {
    totalCount := CountVersions(ownerID, isInternal=false)
    if totalCount > LimitTotalOwnerCount → ErrQuotaTotalCount
}
```

**大小配额** (`CheckSizeQuotaExceeded`, `packages.go:354-426`)：
- 按包类型检查单独大小限制：`LimitSizeAlpine`, `LimitSizeContainer` 等
- 检查全局总大小：`LimitTotalOwnerSize`
- 管理员跳过所有检查

### 7.3 包下载核心流程 `OpenBlobForDownload`

定义于 `services/packages/packages.go:609-637`：

```
1. key = BlobHash256Key(pb.HashSHA256) → 构造存储键
2. cs = NewContentStore()
3. if cs.ShouldServeDirect():
       u = cs.GetServeDirectURL(key, filename, method, opts)
       → 生成预签名直链（仅 MinIO/Azure 实现，Local 返回 ErrURLNotSupported）
4. if u == nil:
       s = cs.OpenBlob(key) → 打开本地文件流
5. if pf.IsLead && method == GET:
       IncrementDownloadCounter() → 增加下载计数
6. return (s, u, pf, nil)
```

**直链模式**：当存储为 MinIO/Azure 且 `ServeDirect` 启用时，生成 5 分钟有效的预签名 URL，客户端直接从对象存储下载，不经过 Gitea 服务器。

### 7.4 包版本描述符 `PackageDescriptor`

`models/packages/descriptor.go:56-71` 定义了完整的包版本描述结构：

```go
type PackageDescriptor struct {
    Package           *Package
    Owner             *user_model.User
    Repository        *repo_model.Repository  // 可选关联仓库
    Version           *PackageVersion
    SemVer            *version.Version        // 语义版本解析
    Creator           *user_model.User
    PackageProperties PackagePropertyList
    VersionProperties PackagePropertyList
    Metadata          any                     // 类型特定的元数据
    Files             []*PackageFileDescriptor
}
```

元数据按包类型动态反序列化（`descriptor.go:172-226`），每种类型对应独立的元数据结构体，如 `alpine.VersionMetadata`、`container.Metadata` 等。

---

## 八、直链存储支持范围核对

### 8.1 直链机制总览

直链（Serve Direct）允许客户端直接从对象存储下载文件，而不经过 Gitea 服务器中转。核心决策点在 `services/packages/packages.go:609-637` 的 `OpenBlobForDownload`：

```go
if cs.ShouldServeDirect() {
    u, err = cs.GetServeDirectURL(key, pf.Name, method, serveDirectReqParams)
    // 若返回 ErrURLNotSupported → 回退到本地读取
}
```

### 8.2 支持直链的存储后端

| 后端 | 文件 | `ServeDirectURL` 实现 | 预签名有效期 | 配置项 |
|------|------|----------------------|-------------|--------|
| **MinioStorage** | `modules/storage/minio.go` | `PresignedGetObject` / `PresignedHeadObject` | 5 分钟 (`ServeDirectDefaultMinExpiry`) | `SERVE_DIRECT` |
| **AzureBlobStorage** | `modules/storage/azureblob.go` | SAS Token 生成 | 5 分钟 | `SERVE_DIRECT` |
| **LocalStorage** | `modules/storage/local.go` | 返回 `ErrURLNotSupported` | N/A | N/A |

### 8.3 各包类型的直链支持情况

所有包类型的下载最终都汇聚到 `OpenBlobForDownload`，因此**理论上所有包类型都支持直链**，前提是存储后端支持。但存在以下差异：

#### 8.3.1 标准包类型（Generic, npm, Maven, PyPI, Alpine, Arch, Cargo, Chef, Composer, Conan, Conda, CRAN, Debian, Go, Helm, Pub, RPM, RubyGems, Swift, Terraform, Vagrant）

这些类型通过 `packages_service.OpenFileForDownloadByPackageNameAndVersion` → `OpenFileForDownload` → `OpenBlobForDownload` 调用链获取文件。`helper.ServePackageFile` 处理结果：

```go
func ServePackageFile(ctx, s, u, pf) {
    if u != nil {
        ctx.Redirect(u.String())  // 302 重定向到直链
        return
    }
    // 本地读取
    ctx.ServeContent(s, opts)
}
```

#### 8.3.2 Container（OCI）

Container 的 blob/manifest 下载通过自己的 `serveBlob` 函数 (`routers/api/packages/container/container.go:717-745`)，**不经过** `helper.ServePackageFile`：

```go
func serveBlob(ctx, pfd) {
    s, u, _, err := packages_service.OpenBlobForDownload(ctx, pfd.File, pfd.Blob,
        ctx.Req.Method, &storage.ServeDirectOptions{
            ContentType: pfd.Properties.GetByName(container_module.PropertyMediaType),
        })
    if u != nil {
        // 307 Temporary Redirect（OCI 规范要求）
        headers.Status = http.StatusTemporaryRedirect
        headers.Location = u.String()
        return
    }
    // 本地读取
    io.Copy(ctx.Resp, s)
}
```

**Container 直链的特殊之处：**
- 重定向使用 **307 Temporary Redirect**（OCI 分发规范要求），而非 302
- 传递 `ContentType` 参数到 `ServeDirectOptions`，影响预签名 URL 中的 `response-content-type`
- 同样汇聚到 `OpenBlobForDownload`，因此后端支持情况与标准类型一致

#### 8.3.3 Head 请求（HEAD Method）

`HEAD` 请求也支持直链。在 `OpenBlobForDownload` 中，`method` 参数传递给 `GetServeDirectURL`：

```go
// MinioStorage 中的实现
case http.MethodHead:
    u, err = s.core.PresignedHeadObject(s.bucket, path, opts)
```

这意味着 `HEAD /api/packages/alice/generic/.../file` 也会返回预签名 URL（作为 `Location` 头），客户端可直接用 `HEAD` 请求对象存储。

### 8.4 直链启用条件

直链需要同时满足以下所有条件：

| 条件 | 检查位置 | 说明 |
|------|----------|------|
| `setting.Packages.Storage.ServeDirect()` | `ShouldServeDirect()` | 存储配置中 `SERVE_DIRECT = true` |
| 存储后端实现 `ServeDirectURL` | `GetServeDirectURL()` | LocalStorage 始终返回 `ErrURLNotSupported` |
| 无错误 | `OpenBlobForDownload` | 若生成预签名 URL 出错，回退到本地读取 |

**配置示例（MinIO）：**
```ini
[packages]
STORAGE_TYPE = minio
MINIO_ENDPOINT = s3.amazonaws.com
MINIO_ACCESS_KEY_ID = ...
MINIO_SECRET_ACCESS_KEY = ...
MINIO_BUCKET = packages
SERVE_DIRECT = true
```

### 8.5 直链与下载计数

在 `OpenBlobForDownload` (`packages.go:631-636`)：

```go
if pf.IsLead && method == http.MethodGet {
    if err := packages_model.IncrementDownloadCounter(ctx, pf.VersionID); err != nil {
        log.Error("Error incrementing download counter: %v", err)
    }
}
```

**重要**：直链模式下下载计数仍然准确——计数在返回预签名 URL 之前就已增加，无论客户端是否实际完成下载。

---

## 九、数据模型层

### 9.1 ER 关系

```
Package (1) ──── (N) PackageVersion
                      │
                      ├─ (N) PackageFile ── (1) PackageBlob
                      │
                      └─ (N) PackageProperty (ref_type=version)

Package (1) ──── (N) PackageProperty (ref_type=package)
PackageFile (1) ─ (N) PackageProperty (ref_type=file)
```

### 9.2 核心表结构

| 表 | 关键字段 | 说明 |
|----|----------|------|
| `package` | `id`, `owner_id`, `type`, `name`, `lower_name`, `semver_compatible`, `repo_id` | 包基本信息，按 `(owner_id, type, lower_name)` 唯一 |
| `package_version` | `id`, `package_id`, `creator_id`, `version`, `lower_version`, `metadata_json`, `download_count`, `is_internal` | 版本信息，按 `(package_id, lower_version)` 唯一 |
| `package_file` | `id`, `version_id`, `blob_id`, `name`, `lower_name`, `composite_key`, `is_lead` | 文件记录，按 `(version_id, lower_name, composite_key)` 唯一 |
| `package_blob` | `id`, `size`, `hash_md5`, `hash_sha1`, `hash_sha256`, `hash_sha512` | 内容寻址 blob，按全部哈希联合唯一 |
| `package_property` | `id`, `ref_type`, `ref_id`, `name`, `value` | 通用 KV 属性表，支持 package/version/file 三种引用 |
| `package_blob_upload` | 容器 blob 分块上传的会话状态 | 仅 Container 类型使用 |

### 9.3 Property 系统

`models/packages/package_property.go` 中的三态引用：

```go
const (
    PropertyTypePackage = "package"
    PropertyTypeVersion = "version"
    PropertyTypeFile    = "file"
)
```

属性用于存储类型特定的元数据字段（如 npm 的 `dist-tags`、container 的 `digest`），避免为每种包类型创建独立数据表。

---

## 十、内容存储层

### 10.1 ContentStore 抽象

`modules/packages/content_store.go` 是 Package Registry 专有的存储门面：

```go
type ContentStore struct {
    store storage.ObjectStorage  // 底层为 storage.Packages（全局单例）
}
```

核心方法：
- `Save(key BlobHash256Key, r io.Reader, size int64)` — 存储 blob
- `OpenBlob(key)` — 打开 blob 读取流
- `Delete(key)` — 删除 blob
- `GetServeDirectURL(...)` — 生成预签名直链
- `ShouldServeDirect()` — 检查是否启用直链模式

### 10.2 存储路径映射

键值 `aabb000000...` 被映射为目录结构 `aa/bb/aabb000000...`（`KeyToRelativePath`），避免单目录文件过多：

```go
func KeyToRelativePath(key BlobHash256Key) string {
    return path.Join(string(key)[0:2], string(key)[2:4], string(key))
}
```

### 10.3 ObjectStorage 接口

`modules/storage/storage.go:75-100` 定义了统一的存储接口：

```go
type ObjectStorage interface {
    Open(path string) (Object, error)
    Save(path string, r io.Reader, size int64) (int64, error)
    Stat(path string) (os.FileInfo, error)
    Delete(path string) error
    ServeDirectURL(path, name, method string, opt *ServeDirectOptions) (*url.URL, error)
    IterateObjects(basePath string, iterator func(fullPath string, obj Object) error) error
}
```

三种实现：

| 实现 | 文件 | 存储类型 | 直链支持 |
|------|------|----------|----------|
| `LocalStorage` | `modules/storage/local.go` | 本地文件系统 | 否（返回 `ErrURLNotSupported`） |
| `MinioStorage` | `modules/storage/minio.go` | S3 兼容对象存储 | 是（预签名 URL，5 分钟有效） |
| `AzureBlobStorage` | `modules/storage/azureblob.go` | Azure Blob Storage | 是 |

存储初始化在 `modules/storage/storage.go:235-243`：

```go
func initPackages() (err error) {
    if !setting.Packages.Enabled {
        Packages = discardStorage("Packages isn't enabled")
        return nil
    }
    Packages, err = NewStorage(setting.Packages.Storage.Type, setting.Packages.Storage)
    return err
}
```

### 10.4 LocalStorage 实现要点

`modules/storage/local.go`：
- 根目录：`setting.Packages.Storage.Path`（绝对路径）
- 临时目录：`{storage_root}/tmp`
- `Save`：先写入临时文件 → `os.Rename` 原子替换 → 应用 umask（去除执行位）
- `Delete`：删除文件后递归清理空父目录
- `IterateObjects`：`filepath.WalkDir` 遍历

### 10.5 MinioStorage 实现要点

`modules/storage/minio.go`：
- 使用 `minio-go/v7` SDK
- 支持静态凭证、环境变量、IAM 角色等多种认证方式（`buildMinioCredentials`）
- 连接测试：`GetBucketVersioning` 检查参数正确性
- `ServeDirectURL`：`PresignedGetObject` / `PresignedHeadObject` 生成 5 分钟预签名 URL，可携带 `response-content-type` 和 `response-content-disposition` 参数

### 10.6 Blob 上传期间的哈希计算

`modules/packages/hashed_buffer.go` 定义 `HashedBuffer`：

```go
type HashedBuffer struct {
    *filebuffer.FileBackedBuffer  // 内存→磁盘自动切换（默认 32MB 阈值）
    hash *MultiHasher             // 同时计算 MD5/SHA1/SHA256/SHA512
    combinedWriter io.Writer      // MultiWriter：同时写入 buffer 和 hasher
}
```

这确保了上传流只需被读取一次，就能同时完成存储和哈希计算。

---

## 十一、清理与维护机制

### 11.1 延迟存储删除（Deferred Storage Delete）

在 `services/packages/packages.go:487-488` 和 `services/packages/cleanup/cleanup.go:188-209`：

```go
// HINT: PACKAGE-DEFER-STORAGE-DELETE: Blobs are not deleted immediately,
// instead they are deleted by the cleanup_packages cron task.
```

当删除包版本时：
1. 只删除数据库中的 `package_version`、`package_file`、属性等记录
2. **不删除** `package_blob` 和底层存储文件
3. 由 `CleanupExpiredData` 定时任务扫描过期的无引用 blob

### 11.2 清理任务 `CleanupTask`

`services/packages/cleanup/cleanup.go:27-33` 定义了两阶段清理：

```go
func CleanupTask(ctx context.Context, olderThan time.Duration) error {
    if err := ExecuteCleanupRules(ctx); err != nil {  // 阶段一：用户定义的清理规则
        return err
    }
    return CleanupExpiredData(ctx, olderThan)       // 阶段二：过期数据清理
}
```

**阶段一：清理规则** (`ExecuteCleanupRules`)
- 遍历每个启用的 `PackageCleanupRule`
- 按包类型过滤，应用 `KeepCount`、`KeepPattern`、`RemoveDays`、`RemovePattern` 等条件
- 对 Debian/Alpine/RPM/Arch 类型，删除后重建仓库索引文件

**阶段二：过期数据清理** (`CleanupExpiredData`)
1. `container_service.Cleanup` — 清理容器上传会话
2. `FindUnreferencedPackages` — 查找无版本引用的 Package 并删除
3. `FindExpiredUnreferencedBlobs` — 查找 `olderThan` 之前创建且无文件引用的 Blob
4. 从数据库删除 blob 记录 → 从存储中删除物理文件（`contentStore.Delete`）

---

## 十二、完整链路时序图

### 12.1 包上传时序

```
Client                    Router                Service                 Model              Storage
  │                        │                     │                      │                   │
  │  PUT /api/packages/    │                     │                      │                   │
  │ {user}/generic/...     │                     │                      │                   │
  │───────────────────────>│                     │                      │                   │
  │                        │  verifyAuth         │                      │                   │
  │                        │  (OAuth2/Basic/     │                      │                   │
  │                        │   JWT Bearer)       │                      │                   │
  │                        │────────────────────>│                      │                   │
  │                        │                     │  user_model.GetUser  │                   │
  │                        │                     │─────────────────────>│                   │
  │                        │                     │                      │                   │
  │                        │  UserAssignmentWeb  │                      │                   │
  │                        │  (username→user)    │                      │                   │
  │                        │                     │                      │                   │
  │                        │  PackageAssignment  │                      │                   │
  │                        │  (determineAccess-  │                      │                   │
  │                        │   Mode)             │                      │                   │
  │                        │                     │                      │                   │
  │                        │  reqPackageAccess   │                      │                   │
  │                        │  (AccessModeWrite)  │                      │                   │
  │                        │                     │                      │                   │
  │                        │  UploadPackage      │                      │                   │
  │                        │────────────────────>│                      │                   │
  │                        │                     │  CreateHashedBuffer  │                   │
  │                        │                     │  (读取+算哈希)        │                   │
  │                        │                     │                      │                   │
  │                        │                     │  CreatePackageOr     │                   │
  │                        │                     │  AddFileToExisting   │                   │
  │                        │                     │─────────────────────>│                   │
  │                        │                     │                      │  TxContext         │
  │                        │                     │                      │  TryInsertPackage  │
  │                        │                     │                      │  GetOrInsertVersion│
  │                        │                     │                      │  GetOrInsertBlob   │
  │                        │                     │                      │                   │
  │                        │                     │                      │  Save blob         │
  │                        │                     │                      │──────────────────>│
  │                        │                     │                      │  TryInsertFile     │
  │                        │                     │                      │                   │
  │                        │                     │                      │  Commit            │
  │                        │                     │                      │                   │
  │                        │                     │  PackageCreate (通知) │                   │
  │                        │                     │─────────────────────>│                   │
  │  201 Created           │                     │                      │                   │
  │<───────────────────────│                     │                      │                   │
```

### 12.2 包下载时序

```
Client                    Router                Service                 Model              Storage
  │                        │                     │                      │                   │
  │  GET /api/packages/    │                     │                      │                   │
  │ {user}/generic/...     │                     │                      │                   │
  │───────────────────────>│                     │                      │                   │
  │                        │  verifyAuth         │                      │                   │
  │                        │  UserAssignmentWeb  │                      │                   │
  │                        │  PackageAssignment  │                      │                   │
  │                        │  reqPackageAccess   │                      │                   │
  │                        │  (AccessModeRead)   │                      │                   │
  │                        │                      │                      │                   │
  │                        │  DownloadPackageFile│                      │                   │
  │                        │────────────────────>│                      │                   │
  │                        │                     │  OpenFileForDownload │                   │
  │                        │                     │─────────────────────>│                   │
  │                        │                     │                      │ GetVersionByName   │
  │                        │                     │                      │ GetFileForVersion  │
  │                        │                     │                      │ GetBlobByID        │
  │                        │                     │                      │                   │
  │                        │                     │  OpenBlobForDownload │                   │
  │                        │                     │─────────────────────>│                   │
  │                        │                     │  ShouldServeDirect?  │                   │
  │                        │                     │                      │                   │
  │                        │                     │  [直链模式]           │                   │
  │                        │                     │  GetServeDirectURL   │                   │
  │                        │                     │─────────────────────────────────────────>│
  │                        │                     │  返回预签名 URL       │                   │
  │  302 Redirect          │                     │                      │                   │
  │  (标准类型)            │                     │                      │                   │
  │  307 Redirect          │                     │                      │                   │
  │  (Container)           │                     │                      │                   │
  │<───────────────────────│                     │                      │                   │
  │                        │                     │                      │                   │
  │  [或: 本地模式]         │                     │                      │                   │
  │                        │                     │  OpenBlob            │                   │
  │                        │                     │─────────────────────────────────────────>│
  │                        │                     │  ServeContent        │                   │
  │                        │                     │  IncrementDownload   │                   │
  │                        │                     │  Counter             │                   │
  │  200 OK + file body    │                     │                      │                   │
  │<───────────────────────│                     │                      │                   │
```

### 12.3 username → 权限决策流转图

```
URL: /api/packages/{username}/{type}/{name}...
         │
         ▼
  ┌──────────────────────┐
  │ 1. AfterRouting      │ 路由匹配成功，按顺序执行 AfterRouting 中间件
  │ ├─ PackageContexter  │ 创建 BaseContext + WebContext
  │ └─ verifyAuth        │ auth.Group 串行尝试 → ctx.Doer, ctx.IsSigned
  └──────────────────────┘
         │
         ▼
  ┌──────────────────────┐
  │ 2. Group("{username}")│ 进入包所有者路由组
  │ ├─ UserAssignmentWeb │
  │ │   ├─ ctx.Doer.Name │
  │ │   │   == username? │ → ctx.ContextUser = ctx.Doer（短路优化）
  │ │   └─ 否则          │ → user_model.GetUserByName(username)
  │ │                     │   失败 → 404 Not Found
  │ │
  │ ├─ PackageAssignment │
  │ │   └─ determineAccessMode(base, pkgOwner, doer)
  │ │       ├─ RequireSignInViewStrict + 未登录 → None
  │ │       ├─ 组织 + 已登录 → 团队最高权限 + TypePackages
  │ │       ├─ 组织 + 未登录 + 可见 → Read
  │ │       ├─ 个人 + 本人 → Owner
  │ │       ├─ 个人 + 公开他人 → Read
  │ │       └─ 个人 + 公开 + 未登录 → Read
  │ │
  │ └─ reqPackageAccess   │
  │     ├─ IsApiToken?    │ → 检查 Scope (ReadPackage/WritePackage)
  │     │   └─ PublicOnly + 私有Owner → 403
  │     └─ AccessMode <   │ → 401 Unauthorized
  │       required?       │   (除非 SiteAdmin)
  └──────────────────────┘
         │
         ▼
  ┌──────────────────────┐
  │ 3. Handler 执行       │ 协议特定逻辑
  │   └─ 调用 services/   │
  │     packages/*        │
  └──────────────────────┘
```

---

## 十三、关键文件索引

| 文件路径 | 职责 |
|----------|------|
| `routers/init.go` | 路由注册入口，挂载 `/api/packages` 和 `/v2` |
| `routers/api/packages/api.go` | CommonRoutes/ContainerRoutes 路由构建 + reqPackageAccess/verifyAuth 中间件 |
| `routers/api/packages/auth.go` | 自定义 JWT Bearer Token 认证（用于 conan/container） |
| `routers/api/packages/helper/helper.go` | 统一错误处理 `ProcessErrorForUser` + 文件下载 `ServePackageFile` |
| `routers/api/packages/generic/generic.go` | Generic 包类型 Handler（最简单的参考实现） |
| `routers/api/packages/container/container.go` | OCI 容器镜像协议完整实现 + ReqContainerAccess + Authenticate |
| `routers/api/packages/nuget/auth.go` | NuGet 专用 API Key 认证（X-NuGet-ApiKey 头） |
| `routers/api/packages/chef/auth.go` | Chef 专用签名认证 |
| `services/packages/auth.go` | JWT Token 签发/验证 (`CreateAuthorizationToken`/`ParseAuthorizationRequest`) |
| `services/packages/packages.go` | 核心业务逻辑：包创建、删除、下载、配额校验 |
| `services/packages/spec.go` | 包类型特化接口 `Specialization` + `SpecManager` 注册表 |
| `services/packages/package_update.go` | 包与仓库的关联/解除关联 |
| `services/packages/cleanup/cleanup.go` | 清理规则执行 + 过期数据清理 |
| `services/context/package.go` | Package 上下文构建 + `determineAccessMode` 权限计算 |
| `services/context/user.go` | `UserAssignmentWeb` + `userAssignment` username → ContextUser 映射 |
| `services/auth/group.go` | `auth.Group` 串行尝试多认证方法 |
| `services/auth/basic.go` | Basic Auth 实现（含 `GetAccessScope` scope 推断） |
| `services/auth/oauth2.go` | OAuth2 Token 认证实现 |
| `models/packages/package.go` | Package 模型 + 类型定义 + CRUD |
| `models/packages/package_version.go` | PackageVersion 模型 + 搜索/计数 + 版本管理 |
| `models/packages/package_file.go` | PackageFile 模型 + 文件搜索/配额计算 |
| `models/packages/package_blob.go` | PackageBlob 模型 + 内容寻址 + 过期 blob 查找 |
| `models/packages/package_property.go` | PackageProperty KV 属性 + 三态引用类型 |
| `models/packages/descriptor.go` | PackageDescriptor 聚合视图 + 元数据反序列化 |
| `modules/packages/content_store.go` | ContentStore 门面 + 键到路径映射 |
| `modules/packages/hashed_buffer.go` | HashedBuffer + MultiHasher（流式多哈希计算） |
| `modules/storage/storage.go` | ObjectStorage 接口定义 + 全局单例（Packages/LFS/Attachments...） |
| `modules/storage/local.go` | LocalStorage 实现 |
| `modules/storage/minio.go` | MinioStorage 实现（S3 兼容） |
| `modules/storage/azureblob.go` | AzureBlobStorage 实现 |
