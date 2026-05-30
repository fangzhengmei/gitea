# Gitea 发行版与附件发布流程梳理

## 核心数据模型

### Release 模型 (`models/repo/release.go:67-90`)

```go
type Release struct {
    ID               int64
    RepoID           int64
    PublisherID      int64
    TagName          string
    Target           string       // 目标分支/commit
    Title            string
    Sha1             string       // Git tag 的 commit hash
    Note             string       // 发行说明
    IsDraft          bool         // 是否为草稿 (核心状态字段)
    IsPrerelease     bool         // 是否为预发布
    IsTag            bool         // 是否仅为 tag（无关联 release）
    Attachments      []*Attachment `xorm:"-"`
    CreatedUnix      timeutil.TimeStamp
}
```

### Attachment 模型 (`models/repo/attachment.go:23-36`)

```go
type Attachment struct {
    ID                int64
    UUID              string    // 唯一标识
    RepoID            int64
    IssueID           int64     // 关联 issue（可选）
    ReleaseID         int64     // 关联 release（可选）
    UploaderID        int64
    CommentID         int64
    Name              string    // 文件名
    DownloadCount     int64
    Size              int64
    CreatedUnix       timeutil.TimeStamp
}
```

---

## 第一阶段：草稿创建流程

### 1. Web 端入口 (`routers/web/repo/release.go:417-542`)

**函数**: `NewReleasePost()`

用户在 Web 界面填写表单后提交：
- 检查表单验证（标题非空、目标分支存在等）
- 解析 `form.Draft` 字段决定是否创建草稿
- 收集附件 UUIDs (`form.Files`)

### 2. 创建 Release - 核心服务 (`services/release/release.go:169-198`)

**函数**: `CreateRelease(gitRepo, rel, attachmentUUIDs, msg)`

流程步骤：
1. **检查 Release 是否已存在** (`IsReleaseExist`)
2. **调用 `createTag()` 处理 Git tag 逻辑**：
   - **关键分支判断** (`services/release/release.go:80`):
     ```go
     if !rel.IsDraft {
         // 非草稿：立即创建 Git tag
         if !gitrepo.IsTagExist(...) {
             // 创建 annotated tag 或 lightweight tag
             gitRepo.CreateTag(rel.TagName, commit.ID.String())
             // 触发通知：PushCommits、CreateRef
             notify_service.PushCommits(...)
             notify_service.CreateRef(...)
             rel.Sha1 = commit.ID.String()
             rel.NumCommits = ...
         }
     } else {
         // 草稿：仅设置创建时间，不创建 Git tag
         rel.CreatedUnix = timeutil.TimeStampNow()
     }
     ```
3. **数据库插入 Release 记录**
4. **关联附件** (`AddReleaseAttachments`)
5. **非草稿才发送通知**：
   ```go
   if !rel.IsDraft {
       notify_service.NewRelease(gitRepo.Ctx, rel)
   }
   ```

### 草稿的关键特性

| 特性 | 草稿 (IsDraft=true) | 正式发布 (IsDraft=false) |
|------|---------------------|--------------------------|
| Git Tag 创建 | ❌ 不创建 | ✅ 立即创建 |
| Sha1 字段 | 空字符串 | 实际 commit hash |
| 通知发送 | ❌ 不发送 | ✅ NewRelease 通知 |
| Release 元数据可见性 | 仅写入权限用户可见 | 所有用户可见 |
| 附件可见性（Release 上下文） | 仅写入权限用户可见 | 所有用户可见 |
| 附件可见性（UUID 直链） | ⚠️ 不受草稿保护（见第六阶段详解） | 所有有仓库读权限的用户可见 |

---

## 第二阶段：二进制附件上传流程

### 两种上传方式

#### 方式一：创建 Release 时同时上传 (Web 端)

**流程**：
1. 前端先通过 AJAX 上传文件，获得附件 UUID 列表
2. 提交 Release 表单时，将 UUIDs 传给后端
3. 后端在 `CreateRelease()` 中调用 `AddReleaseAttachments()` 关联附件

#### 方式二：Release 创建后单独上传 (API 端)

**API 入口**: `routers/api/v1/repo/release_attachment.go:160-267`

**函数**: `CreateReleaseAttachment()`

流程步骤：
1. **权限检查** (`checkReleaseMatchRepo`):
   - 验证 Release 存在且属于当前仓库
   - **草稿检查**: 若 `release.IsDraft`，调用者必须有 release 写入权限
2. **读取上传文件**:
   - 支持 `multipart/form-data` 和 `application/octet-stream`
3. **调用上传服务**:
   ```go
   attachment_service.UploadAttachmentForRelease(ctx, uploaderFile, &repo_model.Attachment{
       Name:       filename,
       UploaderID: ctx.Doer.ID,
       RepoID:     ctx.Repo.Repository.ID,
       ReleaseID:  releaseID,  // 直接关联 Release
   })
   ```

### 附件上传核心 (`services/attachment/attachment.go:70-93`)

**函数**: `uploadAttachment()`

流程步骤：
1. **文件类型验证** (`upload.Verify`)
2. **文件大小检查**
3. **存储到文件系统** (`storage.Attachments.Save`)
4. **数据库插入 Attachment 记录**
   - 生成 UUID
   - `ReleaseID` 字段直接关联到 Release

### 附件关联逻辑 (`models/repo/release.go:179-215`)

**函数**: `AddReleaseAttachments(releaseID, attachmentUUIDs)`

关键检查：
1. 检查附件是否属于同一仓库 (`attach.RepoID == rel.RepoID`)
2. 检查附件是否已被其他 Release 使用 (`attach.ReleaseID != 0`)
3. 更新 `attachment.ReleaseID = releaseID`

---

## 第三阶段：草稿 → 正式发布 状态转换

### 触发方式

1. **Web 端**: 编辑 Release 时取消勾选 "This is a pre-release" 旁边的 "Save as draft"
2. **API 端**: `PATCH /repos/{owner}/{repo}/releases/{id}` 中设置 `is_draft: false`

### 核心更新逻辑 (`services/release/release.go:259-355`)

**函数**: `UpdateRelease()`

关键代码流程：
1. **调用 `createTag()`**：
   - 当 `IsDraft` 从 `true` → `false` 时，`createTag()` 内部会执行正式发布逻辑
   - 创建 Git tag（如果不存在）
   - 设置 `rel.Sha1` 和 `rel.NumCommits`
2. **数据库事务更新**：
   - 更新 Release 基本信息
   - 处理附件增删改 (`addAttachmentUUIDs`, `delAttachmentUUIDs`, `editAttachments`)
3. **通知触发判断** (`services/release/release.go:347-353`):
   ```go
   if !rel.IsDraft {
       if !isTagCreated && !isConvertedFromTag {
           // 已发布过的更新 → UpdateRelease 通知
           notify_service.UpdateRelease(gitRepo.Ctx, doer, rel)
       } else {
           // 首次发布（草稿转正）→ NewRelease 通知
           notify_service.NewRelease(gitRepo.Ctx, rel)
       }
   }
   ```

### 状态转换时的关键变化

| 阶段 | 操作 |
|------|------|
| **Git 层** | 创建真实的 Git tag（如不存在） |
| **数据库** | `IsDraft = false`, `Sha1` 被填充, `NumCommits` 被计算 |
| **通知层** | 发送 `NewRelease` 通知（首次发布）或 `UpdateRelease` 通知 |
| **可见性** | 对所有用户可见 |

---

## 第四阶段：可见性控制

### 列表查询过滤

**API 列表** (`routers/api/v1/repo/release.go:134-208`):
```go
opts := repo_model.FindReleasesOptions{
    IncludeDrafts: canAccessReleaseDraft(ctx),  // 仅写入权限用户能看到草稿
    ...
}
```

**Web 列表** (`routers/web/repo/release.go:167-172`):
```go
IncludeDrafts: writeAccess,  // 同样仅写入权限可见
```

### 单个 Release 访问控制（API 层）

**草稿保护** (`routers/api/v1/repo/release.go:80-83`):
```go
if release.IsDraft && !canAccessReleaseDraft(ctx) {
    ctx.APIErrorNotFound()  // 对无权限用户返回 404
    return
}
```

**附件访问（API 层）** (`routers/api/v1/repo/release_attachment.go:37-40`):
```go
if release.IsDraft && !canAccessReleaseDraft(ctx) {
    ctx.APIErrorNotFound()  // 草稿的附件也受保护
    return
}
```

### 可见性小结：API 层 vs UUID 直链

| 访问路径 | 草稿附件受保护？ | 保护机制 | 代码依据 |
|---------|-----------------|---------|---------|
| API: `GET /repos/{owner}/{repo}/releases/{id}` | ✅ 受保护 | `IsDraft && !canAccessReleaseDraft` 显式检查 | `routers/api/v1/repo/release.go:80` |
| API: `GET /repos/{owner}/{repo}/releases/{id}/assets` | ✅ 受保护 | `checkReleaseMatchRepo` 中检查草稿 | `routers/api/v1/repo/release_attachment.go:37` |
| Web: `/releases/download/{tag}/{file}` | ✅ 受保护 | `IncludeDrafts` 查询过滤 | `routers/web/repo/repo.go:325` |
| Web: `/attachments/{uuid}` | ⚠️ **不受保护** | 仅检查仓库读权限，不检查 `IsDraft` | `routers/web/repo/attachment.go:184` |

---

## 第五阶段：附件下载入口鉴权深度分析

### 5.1 Web 端附件下载入口的鉴权流程

**下载入口路由** (`routers/web/web.go:1474`):
```go
m.Get("/releases/download/{vTag}/{fileName}", webAuth.AllowBasic, webAuth.AllowOAuth2, repo.RedirectDownload)
```

**核心鉴权函数** (`routers/web/repo/repo.go:317-363`):

#### 步骤 1：查询 Release 时复用草稿过滤规则
```go
releases, err := db.Find[repo_model.Release](ctx, repo_model.FindReleasesOptions{
    IncludeDrafts: ctx.Repo.Permission.CanWrite(unit.TypeReleases),  // 关键：复用写入权限判断
    RepoID:        curRepo.ID,
    TagNames:      tagNames,
})
```

**复用机制解析**：
- `IncludeDrafts` 参数直接绑定到用户的仓库写入权限
- 无写入权限的用户：`IncludeDrafts = false`，查询结果自动过滤草稿
- 有写入权限的用户：`IncludeDrafts = true`，可以看到草稿
- **这是草稿可见规则的第一层复用**

#### 步骤 2：通过 tag 名隐式过滤草稿
- 如果用户无写入权限，查询结果中不包含草稿 Release
- 即使攻击者知道草稿的 tag 名，也无法通过 tag 名查询到草稿 Release
- 最终效果：`len(releases) == 0`，返回 404

#### 步骤 3：获取并返回附件
```go
if len(releases) == 1 {
    release := releases[0]
    att, err := repo_model.GetAttachmentByReleaseIDFileName(ctx, release.ID, fileName)
    if att != nil {
        ServeAttachment(ctx, att.UUID)  // 进入通用附件服务逻辑
        return
    }
}
```

### 5.2 通用附件服务的二次鉴权

**函数** (`routers/web/repo/attachment.go:135-195`): `ServeAttachment()`

#### 权限检查链路：
1. **仓库归属检查**：
   ```go
   if attach.CreatedUnix > repo_model.LegacyAttachmentMissingRepoIDCutoff && 
      ctx.Repo.Repository != nil && ctx.Repo.Repository.ID != attach.RepoID {
       ctx.HTTPError(http.StatusNotFound)
       return
   }
   ```

2. **关联类型与权限检查**：
   ```go
   unitType, repoID, err := repo_service.GetAttachmentLinkedTypeAndRepoID(ctx, attach)
   // unitType = unit.TypeReleases (当 attach.ReleaseID != 0 时)
   
   if !perm.CanRead(unitType) {  // 检查 Releases 单元的读取权限
       ctx.HTTPError(http.StatusNotFound)
       return
   }
   ```

3. **Token Scope 检查**（API Token 访问时）：
   ```go
   if requiredScope, ok := attachmentReadScope(unitType); ok {
       context.CheckTokenScopes(ctx, repo, requiredScope)
   }
   ```

**attachmentReadScope 映射规则** (`routers/web/repo/attachment.go:25-34`):
```go
func attachmentReadScope(unitType unit.Type) (auth_model.AccessTokenScope, bool) {
    switch unitType {
    case unit.TypeIssues, unit.TypePullRequests:
        return auth_model.AccessTokenScopeReadIssue, true
    case unit.TypeReleases:
        return auth_model.AccessTokenScopeReadRepository, true  // Release 附件需要 read:repository
    default:
        return "", false
    }
}
```
- Release 附件下载的 Token Scope 是 `read:repository`，而非 `write:repository`
- 草稿保护不依赖 Scope，而是依赖查询层的 `IncludeDrafts` 过滤

### 5.3 附件下载入口的鉴权复用总结

| 鉴权层级 | 复用机制 | 代码位置 |
|---------|---------|---------|
| **第一层** | 查询时通过 `IncludeDrafts` 过滤草稿（关键防护） | `routers/web/repo/repo.go:325` |
| **第二层** | 仅能查询到有权限的 Release（隐式过滤） | `FindReleasesOptions.ToConds()` |
| **第三层** | 通用附件服务的仓库归属检查 | `routers/web/repo/attachment.go:148` |
| **第四层** | Releases 单元读取权限检查 | `routers/web/repo/attachment.go:184` |
| **第五层** | API Token Scope 验证（read:repository） | `routers/web/repo/attachment.go:189-194` |

**鉴权缺陷说明**：
- 前两层仅适用于 `/releases/download/{tag}/{filename}` 友好 URL 路径，通过 `IncludeDrafts` 实现草稿过滤
- 全局 UUID 直链 `/attachments/{uuid}` 绕过了前两层过滤，`ServeAttachment()` 只做仓库读权限检查
- 因此 UUID 直链下草稿附件的可见性仅取决于仓库可见性（公开/私有），与 `IsDraft` 无关

---

## 第六阶段：草稿转正后附件可见性变化

### 6.1 数据库层面的变化

**草稿转正操作**：`UpdateRelease()` 将 `IsDraft` 从 `true` 改为 `false`

**关键观察**：
- **附件记录本身不发生任何变化**
- `Attachment.ReleaseID` 保持不变
- 附件与 Release 的关联关系在草稿阶段就已建立

### 6.2 可见性变化的触发点

**变化的核心是查询条件，而非数据本身**：

| 阶段 | 查询条件 `IncludeDrafts` | 结果 |
|------|-------------------------|------|
| **草稿时** | 无写入权限用户：`false` | 附件不可见 |
| **正式发布后** | 所有用户：`true`（因为 `IsDraft=false` 不依赖此 flag） | 附件对所有人可见 |

### 6.3 各入口的可见性变化详解

以下所有入口均遵循同一套草稿可见性规则：**草稿 Release 及其附件仅对有仓库写入权限的用户可见**。区别在于各入口实现此规则的方式不同。

#### 1. Release 列表 API (`GET /repos/{owner}/{repo}/releases`)
- **控制逻辑**：`FindReleasesOptions` 中的 `IncludeDrafts` 参数 (`routers/api/v1/repo/release.go:178`)
- **草稿时**：`IncludeDrafts = canAccessReleaseDraft(ctx)`，仅写入权限用户能看到 Release 及其附件列表
- **转正后**：`IsDraft = false`，不依赖 `IncludeDrafts` 过滤，所有用户都能看到

#### 2. 单个 Release API (`GET /repos/{owner}/{repo}/releases/{id}`)
- **控制逻辑**：显式 `IsDraft` 检查 (`routers/api/v1/repo/release.go:80-83`)
- **草稿时**：无写入权限用户返回 404
- **转正后**：所有用户都能获取详情，包含附件 URL

#### 3. Web 端友好下载 URL (`/releases/download/{tag}/{filename}`)

**普通 tag 下载分支** (`routers/web/repo/repo.go:324-328`):
```go
releases, err := db.Find[repo_model.Release](ctx, repo_model.FindReleasesOptions{
    IncludeDrafts: ctx.Repo.Permission.CanWrite(unit.TypeReleases),
    RepoID:        curRepo.ID,
    TagNames:      tagNames,
})
```
- **控制逻辑**：`IncludeDrafts` 查询过滤（与入口1一致）
- **草稿时**：无写入权限用户查询不到 Release，返回 404
- **转正后**：所有有仓库读权限的用户可下载

**latest 别名下载分支** (`routers/web/repo/repo.go:344-360`):
```go
} else if len(releases) == 0 && vTag == "latest" {
    // GitHub supports the alias "latest" for the latest release
    // We only fetch the latest release if the tag is "latest" and no release with the tag "latest" exists
    release, err := repo_model.GetLatestReleaseByRepoID(ctx, ctx.Repo.Repository.ID)
    ...
}
```

**`GetLatestReleaseByRepoID` 完整实现** (`models/repo/release.go:329-349`):
```go
// GetLatestReleaseByRepoID returns the latest release for a repository
func GetLatestReleaseByRepoID(ctx context.Context, repoID int64) (*Release, error) {
    cond := builder.NewCond().
        And(builder.Eq{"repo_id": repoID}).
        And(builder.Eq{"is_draft": false}).      // 硬编码过滤草稿: 无论权限都看不到
        And(builder.Eq{"is_prerelease": false}).  // 硬编码过滤预发布
        And(builder.Eq{"is_tag": false})           // 排除纯 tag

    rel := new(Release)
    has, err := db.GetEngine(ctx).
        Desc("created_unix", "id").  // 按创建时间倒序，取最新的
        Where(cond).
        Get(rel)
    ...
}
```

**latest 分支特性**：
- 查询条件**硬编码** `is_draft: false`，与用户权限无关
- 即使用户有写入权限，latest 路径也永远不会返回草稿 Release
- 草稿必须转正（`IsDraft` 变为 `false`）后才会出现在 latest 查询结果中
- 同时过滤预发布版本和纯 tag

#### 4. UUID 直接访问 (`/attachments/{uuid}`)

**实际权限检查流程** (`routers/web/repo/attachment.go:153-195`):
```go
unitType, repoID, err := repo_service.GetAttachmentLinkedTypeAndRepoID(ctx, attach)
// unitType = unit.TypeReleases (当 attach.ReleaseID != 0 时)

if repo == nil {  // 全局路由访问时 ctx.Repo.Repository 为 nil
    repo, err = repo_model.GetRepositoryByID(ctx, repoID)
    perm, err = access_model.GetDoerRepoPermission(ctx, repo, ctx.Doer)
}

if !perm.CanRead(unitType) {  // 仅检查 Releases 读权限
    ctx.HTTPError(http.StatusNotFound)
    return
}
// ⚠️ 此处没有检查 release.IsDraft
```

**未登录用户权限处理** (`models/perm/access/repo_permission.go:407-437`):
```go
// anonymous user visit private repo → AccessModeNone
if user == nil && repo.IsPrivate {
    perm.AccessMode = perm_model.AccessModeNone
    return perm, nil
}
// anonymous visit public repo → AccessModeRead
if user == nil {
    perm.AccessMode = perm_model.AccessModeRead
    return perm, nil
}
```

**可见性分析（区分公开/私有仓库）**：

| 仓库类型 | 草稿时 | 转正后 | 根因 |
|---------|--------|--------|------|
| **公开仓库** | ⚠️ **所有人均可下载**（包括未登录用户） | 所有人均可下载 | `CanRead(TypeReleases)` 在公开仓库对所有用户返回 true，且不检查 `IsDraft` |
| **私有仓库** | 仅仓库读权限用户可下载 | 仅仓库读权限用户可下载 | 未登录用户返回 `AccessModeNone`，已登录用户需有读权限 |

**问题总结**：
- UUID 直链通过 `ServeAttachment()` 处理，该函数**完全不检查**关联 Release 的 `IsDraft` 状态
- 对于公开仓库：草稿附件和已发布附件的可见性完全相同——均对所有用户开放
- 对于私有仓库：虽然未登录用户被 `AccessModeNone` 拒绝，但有仓库读权限的用户可以访问草稿附件
- 这与 Release 上下文（API/Web 列表）中的草稿保护逻辑不一致

### 6.4 安全注意事项

**UUID 直链的草稿保护缺失**：

`ServeAttachment()` 的权限检查链路 (`routers/web/repo/attachment.go:153-195`)：
1. 仓库归属检查 → 防止跨仓库访问
2. `perm.CanRead(unit.TypeReleases)` → 检查仓库读权限
3. `attachmentReadScope` → Token Scope 检查 (`read:repository`)

**缺失的检查**：关联 Release 的 `IsDraft` 状态

**影响范围（按仓库类型区分）**：

| 仓库类型 | 影响 | 说明 |
|---------|------|------|
| 公开仓库 | 🔴 严重 | 任何人（含未登录）均可通过 UUID 直链下载草稿附件，与 Release 列表中的草稿保护完全矛盾 |
| 私有仓库 | 🟡 中等 | 已登录且有仓库读权限的用户可下载草稿附件，但该用户本来也能看到仓库内容；未登录用户被拒绝 |

**修复建议（需考虑全局路由上下文，与真实函数签名一致）**：
```go
// 在 ServeAttachment() 的 perm.CanRead 检查之后，增加草稿状态检查
if unitType == unit.TypeReleases {
    rel, err := repo_model.GetReleaseByID(ctx, attach.ReleaseID)
    if err == nil && rel.IsDraft {
        // 复用与 canAccessReleaseDraft 一致的写入权限判断
        var canWrite bool
        if ctx.Repo.Repository != nil {
            canWrite = ctx.Repo.Permission.CanWrite(unit.TypeReleases)
        } else {
            // 全局路由下 ctx.Repo.Repository 可能为 nil，需动态获取
            // GetRepositoryByID 签名: (ctx, id int64) (*Repository, error)  — models/repo/repo.go:835
            r, rErr := repo_model.GetRepositoryByID(ctx, rel.RepoID)
            if rErr == nil {
                // GetDoerRepoPermission 签名: (ctx, repo *Repository, user) (Permission, error)  — models/perm/access/repo_permission.go:384
                p, pErr := access_model.GetDoerRepoPermission(ctx, r, ctx.Doer)
                if pErr == nil {
                    canWrite = p.CanWrite(unit.TypeReleases)
                }
            }
        }
        if !canWrite {
            ctx.HTTPError(http.StatusNotFound)
            return
        }
    }
}
```

---

## 第七阶段：API Token 与登录态在草稿访问上的差异

### 7.1 草稿访问权限函数分析

**函数** (`routers/api/v1/repo/release.go:24-37`): `canAccessReleaseDraft()`

```go
func canAccessReleaseDraft(ctx *context.APIContext) bool {
    // 第一步：基础权限检查
    if !ctx.IsSigned || !ctx.Repo.Permission.CanWrite(unit.TypeReleases) {
        return false
    }
    
    // 第二步：区分登录态与 API Token
    if ctx.Data["IsApiToken"] != true {
        // 分支 A：用户登录态（非 API Token）
        return true
    }
    
    // 分支 B：API Token 访问
    scope := ctx.Data["ApiTokenScope"].(auth_model.AccessTokenScope)
    requiredScopes := auth_model.GetRequiredScopes(auth_model.Write, auth_model.AccessTokenScopeCategoryRepository)
    allow, _ := scope.HasScope(requiredScopes...)
    return allow
}
```

### 7.2 两种访问方式的对比

| 维度 | 用户登录态（Session/Cookie） | API Token 访问 |
|------|-----------------------------|----------------|
| **识别标志** | `ctx.Data["IsApiToken"] != true` | `ctx.Data["IsApiToken"] == true` |
| **权限判断** | 只需有仓库写入权限 | 写入权限 + Token Scope 检查 |
| **Required Scope** | 不需要 | `write:repository` |
| **典型场景** | Web 界面操作、浏览器访问 | 第三方工具、CI/CD |

### 7.3 访问场景详解

#### 场景 1：Web 浏览器访问（登录态）
- 用户通过账号密码登录，持有 Session Cookie
- `IsApiToken = false`
- 只要有仓库写入权限，即可访问草稿
- **无需额外 Scope 检查**

#### 场景 2：使用 Personal Access Token 调用 API
- 请求头：`Authorization: token <personal-access-token>`
- `IsApiToken = true`
- 双重检查：
  1. Token 所属用户有仓库写入权限
  2. Token 具有 `write:repository` Scope

#### 场景 3：使用 OAuth2 Token 调用 API
- 请求头：`Authorization: Bearer <oauth2-token>`
- `IsApiToken = true`
- 同样需要双重检查

### 7.4 Scope 详细说明

**Required Scopes 计算** (`routers/api/v1/repo/release.go:34`):
```go
requiredScopes := auth_model.GetRequiredScopes(
    auth_model.Write,                          // 权限级别：Write
    auth_model.AccessTokenScopeCategoryRepository  // 类别：Repository
)
```

**Scope 常量定义** (`models/auth/access_token_scope.go:80-81`):
```go
AccessTokenScopeReadRepository  AccessTokenScope = "read:repository"
AccessTokenScopeWriteRepository AccessTokenScope = "write:repository"
```

**Scope 含义**：
- `write:repository`：完整的仓库读写权限
- `read:repository`：仓库只读权限
- 草稿管理属于写入操作范畴，因此需要 Write 级别的 Scope

### 7.5 权限矩阵

| 访问方式 | 已登录 | 仓库写入权限 | Token Scope | 可访问草稿 |
|---------|--------|-------------|-------------|-----------|
| 浏览器（未登录） | ❌ | - | - | ❌ |
| 浏览器（只读用户） | ✅ | ❌ | - | ❌ |
| 浏览器（写入用户） | ✅ | ✅ | - | ✅ |
| API Token（只读 Scope） | ✅ | ✅ | `read:repository` | ❌ |
| API Token（写入 Scope） | ✅ | ✅ | `write:repository` | ✅ |

---

## 完整流程图

```
用户操作
   ↓
[Web/API 入口]
   │
   ├─ 新建草稿 Release
   │      ↓
   │  services/release/CreateRelease()
   │      ├─ IsDraft = true
   │      ├─ 不创建 Git Tag
   │      ├─ Sha1 = ""
   │      ├─ 数据库插入
   │      └─ 不发通知
   │
   ├─ 上传附件（草稿时也可上传）
   │      ↓
   │  services/attachment/UploadAttachmentForRelease()
   │      ├─ 文件类型/大小校验
   │      ├─ storage 存储
   │      ├─ 数据库插入 Attachment
   │      └─ ReleaseID 关联
   │
   └─ 草稿转正 / 编辑发布
          ↓
    services/release/UpdateRelease()
          ├─ IsDraft = false
          ├─ createTag() → 创建 Git Tag
          ├─ 填充 Sha1 / NumCommits
          ├─ 数据库更新
          ├─ 附件增删改
          └─ NewRelease / UpdateRelease 通知
                ↓
          [正式发布，全员可见]
```

---

## 关键文件与代码位置

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| Release 数据模型 | `models/repo/release.go` | `Release` struct |
| Attachment 数据模型 | `models/repo/attachment.go` | `Attachment` struct |
| Release 核心服务 | `services/release/release.go` | `CreateRelease`, `UpdateRelease`, `createTag` |
| 附件上传服务 | `services/attachment/attachment.go` | `UploadAttachmentForRelease`, `NewAttachment` |
| Web 端路由 | `routers/web/repo/release.go` | `NewReleasePost`, `EditReleasePost` |
| API 端 Release | `routers/api/v1/repo/release.go` | `CreateRelease`, `EditRelease` |
| API 端附件 | `routers/api/v1/repo/release_attachment.go` | `CreateReleaseAttachment` |

---

## 注意事项

1. **草稿与 Git Tag 的分离**：草稿 Release 不创建真实的 Git tag，只有发布时才创建
2. **附件权限保护的分层**：草稿附件在 Release 上下文（API 列表、Web 列表、友好下载 URL）中受 `IsDraft` 保护，但在 UUID 直链路径中**不检查草稿状态**，公开仓库的草稿附件对所有人可下载
3. **通知时机**：只有非草稿状态才会触发通知，草稿转正时触发 `NewRelease` 而非 `UpdateRelease`
4. **附件关联校验**：附件必须属于同一仓库且未被其他 Release 使用才能关联
5. **latest 路径的特殊性**：`GetLatestReleaseByRepoID` 硬编码 `is_draft: false`，无论用户权限如何都不返回草稿
