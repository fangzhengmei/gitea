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
| 列表可见性 | 仅写入权限用户可见 | 所有用户可见 |

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

### 单个 Release 访问控制

**草稿保护** (`routers/api/v1/repo/release.go:80-83`):
```go
if release.IsDraft && !canAccessReleaseDraft(ctx) {
    ctx.APIErrorNotFound()  // 对无权限用户返回 404
    return
}
```

**附件访问** (`routers/api/v1/repo/release_attachment.go:37-40`):
```go
if release.IsDraft && !canAccessReleaseDraft(ctx) {
    ctx.APIErrorNotFound()  // 草稿的附件也受保护
    return
}
```

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
2. **附件权限继承**：草稿 Release 的附件也受草稿权限保护，无权限用户无法访问
3. **通知时机**：只有非草稿状态才会触发通知，草稿转正时触发 `NewRelease` 而非 `UpdateRelease`
4. **附件关联校验**：附件必须属于同一仓库且未被其他 Release 使用才能关联
