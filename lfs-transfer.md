# LFS 大对象传输协同机制分析

## 一、整体架构概览

Gitea LFS 系统由四大核心组件协同工作：
1. **大对象批接口 (Batch API)** - 客户端与服务端交互的入口，批量获取对象上传/下载链接
2. **对象存储签名** - 当使用 MinIO/Azure Blob 等对象存储时，生成预签名 URL 实现直接访问
3. **HTTP 传输机制** - 通过 HTTP Range 头实现下载断点续传，通过 Transfer-Encoding: chunked 实现流式上传
4. **Pure SSH 传输链路** - 基于 git-lfs-transfer 协议的 SSH 原生传输通道

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  LFS Client     │────▶│  Batch API      │────▶│  Object Storage │
│  (git-lfs)      │     │  (批处理接口)   │     │  (签名/直传)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
          │                       │
          │                       ▼
          │               ┌─────────────────┐
          ├──────────────▶│  HTTP Transfer  │
          │               │  (Range/chunked)│
          │               └─────────────────┘
          │
          │               ┌─────────────────┐
          └──────────────▶│  SSH Transfer   │
                          │  (git-lfs-transfer) │
                          └─────────────────┘
```

## 二、大对象批接口 (Batch API) 实现

### 2.1 路由入口
- **文件**: `routers/common/lfs.go:18`
- **端点**: `POST /{username}/{reponame}/info/lfs/objects/batch`
- **处理函数**: `services/lfs/server.go:BatchHandler` (L187-307)

### 2.2 核心流程

```go
// BatchHandler 核心逻辑
func BatchHandler(ctx *context.Context) {
    // 1. 解析批量请求 (upload/download)
    var br lfs_module.BatchRequest
    decodeJSON(ctx.Req, &br)
    
    // 2. 认证与仓库权限校验
    repository := getAuthenticatedRepository(ctx, rc, isUpload)
    
    // 3. 配额校验 - 批量大小限制
    if setting.LFS.MaxBatchSize != 0 && len(br.Objects) > setting.LFS.MaxBatchSize {
        writeStatus(ctx, http.StatusRequestEntityTooLarge)
        return
    }
    
    // 4. 逐个处理对象
    for _, p := range br.Objects {
        // 4.1 校验对象有效性
        if !p.IsValid() { ... }
        
        // 4.2 检查对象是否已存在
        exists, _ := contentStore.Exists(p)
        
        // 4.3 配额校验 - 单文件大小限制 (仅上传时)
        if !exists && setting.LFS.MaxFileSize > 0 && p.Size > setting.LFS.MaxFileSize {
            err = &lfs_module.ObjectError{
                Code:    http.StatusUnprocessableEntity,
                Message: fmt.Sprintf("Size must be less than or equal to %d", setting.LFS.MaxFileSize),
            }
        }
        
        // 4.4 构建响应对象
        responseObject = buildObjectResponse(rc, p, true, !exists, err)
    }
    
    // 5. 返回批量响应
    enc.Encode(&lfs_module.BatchResponse{Objects: responseObjects})
}
```

### 2.3 响应构建与对象存储签名
- **函数**: `services/lfs/server.go:buildObjectResponse` (L482-518)
- **关键逻辑**:

```go
func buildObjectResponse(rc *requestContext, pointer Pointer, download, upload bool, err *ObjectError) *ObjectResponse {
    if download {
        if setting.LFS.Storage.ServeDirect() {
            // 生成对象存储预签名 URL，绕过 Gitea 服务器
            u, err := storage.LFS.ServeDirectURL(pointer.RelativePath(), pointer.Oid, http.MethodGet, nil)
            if u != nil && err == nil {
                link = lfs_module.NewLink(u.String()) // 预签名 URL 不需要 Authorization 头
            }
        }
        if link == nil {
            // 回退到通过 Gitea 服务器代理下载
            link = lfs_module.NewLink(rc.DownloadLink(pointer)).WithHeader("Authorization", rc.Authorization)
        }
        rep.Actions["download"] = link
    }
    
    if upload {
        // 上传始终通过 Gitea 服务器（不支持直传到对象存储）
        rep.Actions["upload"] = lfs_module.NewLink(rc.UploadLink(pointer)).
            WithHeader("Authorization", rc.Authorization).
            WithHeader("Transfer-Encoding", "chunked") // 启用分块传输
        
        rep.Actions["verify"] = lfs_module.NewLink(rc.VerifyLink(pointer)).
            WithHeader("Authorization", rc.Authorization)
    }
}
```

## 三、对象存储签名机制

### 3.1 接口定义
- **文件**: `modules/storage/storage.go:93`
- **接口**: `ServeDirectURL(path, name, method string, opt *ServeDirectOptions) (*url.URL, error)`

### 3.2 MinIO 签名实现
- **文件**: `modules/storage/minio.go:ServeDirectURL` (L277-296)
- **有效期**: 5 分钟
- **签名方法**: 使用 MinIO SDK 的 `PresignedGetObject` / `PresignedHeadObject`

```go
func (m *MinioStorage) ServeDirectURL(storePath, name, method string, opt *ServeDirectOptions) (*url.URL, error) {
    expires := 5 * time.Minute
    if method == http.MethodHead {
        return m.client.PresignedHeadObject(m.ctx, m.bucket, m.buildMinioPath(storePath), expires, reqParams)
    }
    return m.client.PresignedGetObject(m.ctx, m.bucket, m.buildMinioPath(storePath), expires, reqParams)
}
```

### 3.3 Azure Blob 签名实现
- **文件**: `modules/storage/azureblob.go:ServeDirectURL` (L281-303)
- **有效期**: 5 分钟
- **签名方法**: 使用 SAS (Shared Access Signature) 令牌

```go
func (a *AzureBlobStorage) ServeDirectURL(storePath, name, method string, reqParams *ServeDirectOptions) (*url.URL, error) {
    u, err := a.getSasURL(blobClient, sas.BlobSignatureValues{
        Permissions: (&sas.BlobPermissions{
            Read:  method == http.MethodGet || method == http.MethodHead,
            Write: method == http.MethodPut,
        }).String(),
        StartTime:  time.Now().UTC(),
        ExpiryTime: time.Now().UTC().Add(5 * time.Minute),
        ContentDisposition: param.ContentDisposition,
        ContentType:        param.ContentType,
    })
    return url.Parse(u)
}
```

### 3.4 本地存储
- **文件**: `modules/storage/local.go:ServeDirectURL` (L136-138)
- 不支持直链，返回 `ErrURLNotSupported`

## 四、HTTP 传输机制详解

### 4.1 下载断点续传 (Range 请求)
- **文件**: `services/lfs/server.go:DownloadHandler` (L112-185)
- **实现原理**: 解析 HTTP `Range` 头，支持 `bytes=start-end` 格式
- **适用场景**: 下载中断后从断点继续，不需要重新下载整个文件

```go
func DownloadHandler(ctx *context.Context) {
    // 解析 Range 头
    var fromByte, toByte int64
    toByte = meta.Size - 1
    statusCode := http.StatusOK
    
    if rangeHdr := ctx.Req.Header.Get("Range"); rangeHdr != "" {
        match := rangeHeaderRegexp.FindStringSubmatch(rangeHdr)
        if len(match) > 1 {
            statusCode = http.StatusPartialContent
            fromByte, _ = strconv.ParseInt(match[1], 10, 32)
            
            if fromByte >= meta.Size {
                writeStatus(ctx, http.StatusRequestedRangeNotSatisfiable)
                return
            }
            
            // 设置响应头
            ctx.Resp.Header().Set("Content-Range", fmt.Sprintf("bytes %d-%d/%d", fromByte, toByte, meta.Size))
            ctx.Resp.Header().Set("Access-Control-Expose-Headers", "Content-Range")
        }
    }
    
    // 从指定位置开始读取
    if fromByte > 0 {
        _, err = content.Seek(fromByte, io.SeekStart)
    }
    
    // 返回部分内容
    ctx.Resp.WriteHeader(statusCode)
    io.CopyN(ctx.Resp, content, contentLength)
}
```

### 4.2 上传分块传输 (Transfer-Encoding: chunked)
- **触发点**: `services/lfs/server.go:504-508`
- **实现原理**: 通过在 upload action 的 header 中设置 `Transfer-Encoding: chunked`，通知 git-lfs 客户端可以使用流式分块上传
- **与断点续传的区别**:
  - **Transfer-Encoding: chunked** 是 HTTP/1.1 的传输编码方式，用于在不知道总大小时将数据分成多个块流式传输
  - **Range 断点续传** 是 HTTP 协议的内容范围请求，用于从指定偏移量继续传输
  - 两者是完全独立的机制：chunked 解决"不知道大小时如何流式上传"，Range 解决"中断后如何继续"
  - **注意**: Gitea 的上传接口目前不支持 Range 方式的断点续传，仅通过 chunked 编码支持流式传输

```go
// services/lfs/server.go:504-508
// Set Transfer-Encoding header to enable chunked uploads. Required by git-lfs client to do chunked transfer.
// See: https://github.com/git-lfs/git-lfs/blob/main/tq/basic_upload.go#L58-59
rep.Actions["upload"] = lfs_module.NewLink(rc.UploadLink(pointer)).
    WithHeader("Authorization", rc.Authorization).
    WithHeader("Transfer-Encoding", "chunked")
```

### 4.3 ServeDirect 模式下的 Range 处理
当 `SERVE_DIRECT = true` 时，下载流程发生本质变化：

1. **非 ServeDirect 模式**（默认）:
   - 客户端 → Gitea `DownloadHandler` → Gitea 解析 Range 头 → Gitea Seek 到指定位置 → 返回数据
   - Range 请求完全由 Gitea 服务器处理

2. **ServeDirect 模式**（对象存储直链）:
   - 客户端 → 预签名 URL → **直接访问对象存储服务**
   - Range 请求由 **对象存储侧（MinIO/Azure Blob）** 直接处理，Gitea 服务器不参与数据传输
   - 预签名 URL 包含访问凭证，有效期仅 5 分钟
   - 对象存储原生支持 Range 请求，性能更优且不占用 Gitea 带宽

```
非 ServeDirect 模式:
┌──────────┐  Range  ┌─────────┐  Seek  ┌──────────┐
│ Client   │────────▶│ Gitea   │────────▶│ Storage  │
│          │◀────────│ Server  │◀────────│          │
└──────────┘  Data   └─────────┘  Data  └──────────┘

ServeDirect 模式:
┌──────────┐  Range  ┌──────────────┐
│ Client   │────────▶│ Object Store │
│          │◀────────│ (MinIO/Azure)│
└──────────┘  Data   └──────────────┘
       ▲
       │ 预签名 URL
┌─────────┐
│ Gitea   │  (仅签名，不参与数据传输)
└─────────┘
```

### 4.4 并发控制：内部 LFSClient 与外部 git-lfs 的区别

| 配置项 | 作用对象 | 生效场景 | 默认值 |
|--------|----------|----------|--------|
| `LFSClient.BATCH_OPERATION_CONCURRENCY` | Gitea 内部 LFSClient | Gitea 作为客户端镜像同步 LFS 对象时 | 8 |
| `lfs.concurrenttransfers` | 外部 git-lfs 客户端 | 用户本地 git-lfs 上传/下载时 | 8 |

- **Gitea 内部 LFSClient** (`modules/lfs/http_client.go`):
  - 用于 Gitea 自身作为 LFS 客户端的场景，如仓库镜像同步
  - 并发控制在 `performOperation` 函数中通过 `errgroup.SetLimit()` 实现

```go
// modules/lfs/http_client.go:146-158
if setting.LFSClient.BatchOperationConcurrency <= 0 {
    panic("BatchOperationConcurrency must be greater than 0, forgot to init?")
}
errGroup, groupCtx := errgroup.WithContext(ctx)
errGroup.SetLimit(setting.LFSClient.BatchOperationConcurrency)
for _, object := range result.Objects {
    errGroup.Go(func() error {
        return performSingleOperation(groupCtx, object, dc, uc, transferAdapter)
    })
}
return errGroup.Wait()
```

- **外部 git-lfs 客户端**:
  - 用户通过 `git config lfs.concurrenttransfers 8` 配置
  - 由 git-lfs 客户端自身控制，Gitea 服务器无法直接限制
  - Gitea 通过 `MaxBatchSize` 限制单次批请求的对象数来间接控制并发压力

## 五、Pure SSH 传输链路

### 5.1 协议概述
- **配置项**: `LFS_ALLOW_PURE_SSH` (默认 `false`)
- **协议**: 基于 `git-lfs-transfer` 协议，使用 pkt-line 格式在 SSH 通道上传输
- **依赖库**: `github.com/charmbracelet/git-lfs-transfer`

### 5.2 触发流程
- **文件**: `cmd/serv.go:264-271`
- 当 SSH 命令为 `git-lfs-transfer <repo> <upload|download>` 时触发

```go
// cmd/serv.go:264-271
// LFS SSH protocol
if verb == git.CmdVerbLfsTransfer {
    token, err := lfs.GetLFSAuthTokenWithBearer(lfs.AuthTokenOptions{Op: lfsVerb, UserID: results.UserID, RepoID: results.RepoID})
    if err != nil {
        return err
    }
    return lfstransfer.Main(ctx, repoPath, lfsVerb, token)
}
```

### 5.3 核心架构
- **入口**: `modules/lfstransfer/main.go:Main`
- **后端适配器**: `modules/lfstransfer/backend/backend.go:GiteaBackend`
- **通信方式**: 通过 SSH 的 stdin/stdout 使用 pkt-line 格式通信

```
┌──────────────┐  SSH  ┌──────────┐  Internal API  ┌────────────┐
│ git-lfs      │──────▶│ serv     │───────────────▶│ Batch API  │
│ (lfs.sshtransfer)    │ (SSH)    │                │ Upload API │
└──────────────┘       └──────────┘                │ Download   │
       ▲                                              │ Verify API │
       │ pkt-line                                     └────────────┘
       ▼
┌───────────────────────────────────┐
│ lfstransfer.Processor             │
│  └─ GiteaBackend (HTTP 适配器)    │
└───────────────────────────────────┘
```

### 5.4 与 HTTP 批接口的关联
Pure SSH 传输底层**复用所有 HTTP API 的业务逻辑**：

| SSH 操作 | 调用的内部 HTTP API | 复用的业务逻辑 |
|----------|-------------------|----------------|
| `Batch` | `POST /api/internal/repo/{repo}/info/lfs/objects/batch` | `BatchHandler` - 认证、配额校验、对象存在性检查 |
| `Download` | `GET /api/internal/repo/{repo}/info/lfs/objects/{oid}` | `DownloadHandler` - Range 处理、内容读取 |
| `Upload` | `PUT /api/internal/repo/{repo}/info/lfs/objects/{oid}/{size}` | `UploadHandler` - 哈希校验、大小校验、存储写入 |
| `Verify` | `POST /api/internal/repo/{repo}/info/lfs/verify` | `VerifyHandler` - 存在性验证 |
| `Lock` | `POST /api/internal/repo/{repo}/info/lfs/locks` | 锁管理逻辑 |

- **内部认证**: 通过 `X-Gitea-Internal-Auth` 头携带 `setting.InternalToken` 进行内部服务认证
- **URL 转换**: `modules/lfstransfer/backend/util.go:toInternalLFSURL` 将外部 URL 转换为内部 API 路径

### 5.5 上传校验流程
Pure SSH 上传与 HTTP 上传走完全相同的校验链路：

```
SSH Upload Flow:
1. 客户端 → SSH → lfstransfer.Main()
2. Processor.Batch() → 调用内部 Batch API
   ↳ BatchHandler: 认证、配额校验(MaxFileSize/MaxBatchSize)、对象存在性检查
3. Processor.Upload() → 调用内部 Upload API
   ↳ UploadHandler: 
     - getAuthenticatedRepository (权限校验)
     - contentStore.Put(): hashingReader 实时计算 SHA256 和大小
     - uploadOrVerify(): 跨仓库访问控制(LFSObjectAccessible)
     - NewLFSMetaObject(): 创建元数据关联
4. Processor.Verify() → 调用内部 Verify API (可选)
```

### 5.6 SSH Token 认证流程
- **文件**: `cmd/serv.go:274-294`
- 当 SSH 命令为 `git-lfs-authenticate <repo> <upload|download>` 时，返回 HTTP API 的访问令牌
- 这是 git-lfs 通过 SSH 获取 HTTP 凭据的标准流程，与 Pure SSH 传输是两个不同的机制

```go
// cmd/serv.go:274-294
// LFS token authentication
if verb == git.CmdVerbLfsAuthenticate {
    url := fmt.Sprintf("%s%s/%s.git/info/lfs", setting.AppURL, ...)
    token, err := lfs.GetLFSAuthTokenWithBearer(...)
    tokenAuthentication := &git_model.LFSTokenResponse{
        Header: map[string]string{"Authorization": token},
        Href:   url,
    }
    enc.Encode(tokenAuthentication) // 返回给客户端用于后续 HTTP 请求
}
```

## 六、凭据隔离机制

### 6.1 JWT Token 认证体系
- **Token 生成**: `services/lfs/server.go:GetLFSAuthTokenWithBearer` (L63-82)
- **Claims 结构**:

```go
type Claims struct {
    RepoID int64  // 仓库 ID - 凭据与仓库绑定
    Op     string // 操作类型: upload/download
    UserID int64  // 用户 ID
    jwt.RegisteredClaims
}
```

- **有效期**: `setting.LFS.HTTPAuthExpiry` (默认 24 小时)

### 6.2 认证流程
- **入口函数**: `services/lfs/server.go:authenticate` (L538-578)

```go
func authenticate(ctx *context.Context, repository *repo_model.Repository, authorization string, requireSigned, requireWrite bool) bool {
    // 1. 首先检查常规用户权限
    perm, _ := access_model.GetDoerRepoPermission(ctx, repository, ctx.Doer)
    canAccess := perm.CanAccess(accessMode, unit.TypeCode)
    if canAccess && (!requireSigned || ctx.IsSigned) {
        return true
    }
    
    // 2. 回退到 LFS JWT Token 认证
    user, err := parseToken(ctx, authorization, repository, accessMode)
    if err != nil {
        return false
    }
    ctx.Doer = user
    return true
}
```

### 6.3 Token 解析与校验
- **函数**: `services/lfs/server.go:handleLFSToken` (L580-622)

```go
func handleLFSToken(ctx stdCtx.Context, tokenSHA string, target *repo_model.Repository, mode perm_model.AccessMode) (*user_model.User, error) {
    // 1. 解析 JWT Token
    token, err := jwt.ParseWithClaims(tokenSHA, &Claims{}, func(t *jwt.Token) (any, error) {
        return setting.LFS.JWTSecretBytes, nil
    })
    
    // 2. 校验仓库绑定 - 凭据隔离核心
    if claims.RepoID != target.ID {
        return nil, errors.New("invalid token claim") // Token 只能用于指定仓库
    }
    
    // 3. 校验操作权限
    if mode == perm_model.AccessModeWrite && claims.Op != "upload" {
        return nil, errors.New("invalid token claim") // 上传 Token 不能用于下载
    }
    
    // 4. 校验用户状态
    u, _ := user_model.GetUserByID(ctx, claims.UserID)
    if !u.IsActive || u.ProhibitLogin {
        return nil, util.NewPermissionDeniedErrorf("not allowed to access any repository")
    }
    
    // 5. 再次校验仓库权限
    perm, _ := access_model.GetDoerRepoPermission(ctx, target, u)
    if !perm.CanAccess(mode, unit.TypeCode) {
        return nil, util.NewPermissionDeniedErrorf("no permission to access the repository")
    }
    
    return u, nil
}
```

### 6.4 跨仓库对象访问控制
当对象已存在但当前仓库没有关联记录时，需要校验用户是否有权访问该对象：

- **函数**: `models/git/lfs.go:LFSObjectAccessible` (L193-202)

```go
func LFSObjectAccessible(ctx context.Context, user *user_model.User, oid string) (bool, error) {
    if user.IsAdmin {
        // 管理员可以访问所有对象
        count, err := db.GetEngine(ctx).Count(&LFSMetaObject{Pointer: lfs.Pointer{Oid: oid}})
        return count > 0, err
    }
    // 普通用户只能访问其有权限的仓库中的对象
    cond := repo_model.AccessibleRepositoryCondition(user, unit.TypeInvalid)
    count, err := db.GetEngine(ctx).Where(cond).
        Join("INNER", "repository", "`lfs_meta_object`.repository_id = `repository`.id").
        Count(&LFSMetaObject{Pointer: lfs.Pointer{Oid: oid}})
    return count > 0, err
}
```

**调用位置**:
- `services/lfs/server.go:BatchHandler` (L266-271) - 批量处理时
- `services/lfs/server.go:UploadHandler` (L340-345) - 上传验证时

## 七、配额校验生效位置

### 7.1 批量大小限制
- **配置项**: `setting.LFS.MaxBatchSize`
- **生效位置**: `services/lfs/server.go:215-218` (BatchHandler 入口)

### 7.2 单文件大小限制
- **配置项**: `setting.LFS.MaxFileSize`
- **生效位置**: `services/lfs/server.go:258-262` (BatchHandler 中逐个对象校验)
- **注意**: 仅在对象不存在且需要上传时校验

### 7.3 哈希与大小完整性校验
- **实现**: `modules/lfs/content_store.go:hashingReader` (L112-163)
- **触发点**: `modules/lfs/content_store.go:Put` (L48-78)

```go
func (s *ContentStore) Put(pointer Pointer, r io.Reader) error {
    // 包装读取器，实时计算哈希和大小
    wrappedRd := newHashingReader(pointer.Size, pointer.Oid, r)
    
    // 写入存储，过程中实时校验
    written, err := s.Save(p, wrappedRd, pointer.Size)
    
    // 二次校验（部分存储 SDK 可能忽略读取错误）
    if wrappedRd.lastError != nil && !errors.Is(wrappedRd.lastError, io.EOF) {
        err = wrappedRd.lastError
    } else if written != pointer.Size {
        err = ErrSizeMismatch
    }
    
    // 上传失败则清理文件
    if err != nil {
        s.Delete(p)
    }
    return err
}
```

### 7.4 上传验证流程
- **函数**: `services/lfs/server.go:uploadOrVerify` (L338-368)

```go
uploadOrVerify := func() error {
    if exists {
        // 对象已存在，检查用户是否有权访问
        accessible, _ := git_model.LFSObjectAccessible(ctx, ctx.Doer, p.Oid)
        if !accessible {
            // 无权限则要求用户重新上传完整内容进行验证
            hash := sha256.New()
            written, _ := io.Copy(hash, ctx.Req.Body)
            
            if written != p.Size {
                return lfs_module.ErrSizeMismatch
            }
            if hex.EncodeToString(hash.Sum(nil)) != p.Oid {
                return lfs_module.ErrHashMismatch
            }
        }
    } else if err := contentStore.Put(p, ctx.Req.Body); err != nil {
        // 对象不存在，执行上传
        return err
    }
    // 创建元数据关联
    _, err := git_model.NewLFSMetaObject(ctx, repository.ID, p)
    return err
}
```

## 八、关键数据结构

### 8.1 LFS 指针 (Pointer)
- **文件**: `modules/lfs/pointer.go`
- **存储路径**: `{oid[0:2]}/{oid[2:4]}/{oid[4:]}`

```go
type Pointer struct {
    Oid  string `json:"oid"`  // SHA256 哈希，64 字符十六进制
    Size int64  `json:"size"` // 文件大小（字节）
}
```

### 8.2 批量请求/响应
- **文件**: `modules/lfs/shared.go`

```go
type BatchRequest struct {
    Operation string     `json:"operation"` // "upload" 或 "download"
    Transfers []string   `json:"transfers,omitempty"`
    Ref       *Reference `json:"ref,omitempty"`
    Objects   []Pointer  `json:"objects"`
}

type BatchResponse struct {
    Transfer string            `json:"transfer,omitempty"`
    Objects  []*ObjectResponse `json:"objects"`
}

type ObjectResponse struct {
    Pointer
    Actions map[string]*Link `json:"actions,omitempty"` // 包含 upload/download/verify 链接
    Error   *ObjectError     `json:"error,omitempty"`
}

type Link struct {
    Href      string            `json:"href"`
    Header    map[string]string `json:"header,omitempty"` // 包含 Authorization 等头
    ExpiresAt *time.Time        `json:"expires_at,omitempty"`
}
```

## 九、配置项汇总

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `LFS_START_SERVER` | false | 是否启用 LFS 服务 |
| `LFS_ALLOW_PURE_SSH` | false | 是否启用 Pure SSH LFS 传输 |
| `LFS_MAX_FILE_SIZE` | 0 (无限制) | 单文件最大大小（字节） |
| `LFS_MAX_BATCH_SIZE` | 0 (无限制) | 单次批量请求最大对象数 |
| `LFS_HTTP_AUTH_EXPIRY` | 24h | JWT Token 有效期 |
| `SERVE_DIRECT` | false | 是否启用对象存储直链 |
| `LFSClient.BATCH_SIZE` | 20 | 内部客户端每批处理对象数 |
| `LFSClient.BATCH_OPERATION_CONCURRENCY` | 8 | 内部客户端并发传输数 |

## 十、安全边界总结

1. **凭据隔离**: JWT Token 与仓库 ID、操作类型、用户 ID 三重绑定，防止跨仓库越权
2. **对象访问控制**: `LFSObjectAccessible` 确保用户只能访问其有权限仓库中的对象
3. **配额防护**: 入口处校验批量大小和单文件大小，防止滥用
4. **完整性校验**: 上传过程中实时计算 SHA256 哈希和大小，确保数据完整性
5. **预签名 URL**: 对象存储签名 URL 有效期仅 5 分钟，且不暴露 Gitea 凭据
6. **内部 API 认证**: Pure SSH 传输通过 `X-Gitea-Internal-Auth` 头进行内部服务间认证
7. **传输通道安全**: SSH 传输使用 pkt-line 格式在加密通道内传输，避免额外暴露 HTTP 端点

## 十一、传输机制对比总结

| 特性 | HTTP 传输 | Pure SSH 传输 |
|------|----------|---------------|
| **协议** | HTTP/HTTPS | SSH (git-lfs-transfer) |
| **批接口** | 独立 HTTP Batch API | 通过 GiteaBackend 调用内部 Batch API |
| **Range 下载** | Gitea 处理 (非 ServeDirect) / 对象存储处理 (ServeDirect) | 通过内部 API 复用 DownloadHandler |
| **Chunked 上传** | 支持 | 通过内部 API 复用 UploadHandler |
| **认证方式** | Basic Auth / JWT Token / Session | SSH Key + 内部 JWT Token |
| **凭据有效期** | JWT 默认 24h | JWT 默认 24h |
| **配额校验** | BatchHandler 入口 | 复用 BatchHandler 逻辑 |
| **完整性校验** | hashingReader 实时校验 | 复用 hashingReader 逻辑 |
| **适用场景** | 大部分用户、CI/CD | SSH 环境偏好者、纯内网部署 |
| **客户端配置** | `git config lfs.url` | `git config lfs.sshtransfer always` |

### 关键代码路径对照

| 功能 | HTTP 路径 | SSH 路径 | 复用逻辑 |
|------|----------|----------|----------|
| 批量请求 | `POST /info/lfs/objects/batch` | `GiteaBackend.Batch()` | `BatchHandler` |
| 下载 | `GET /info/lfs/objects/{oid}` | `GiteaBackend.Download()` | `DownloadHandler` |
| 上传 | `PUT /info/lfs/objects/{oid}/{size}` | `GiteaBackend.Upload()` | `UploadHandler` |
| 验证 | `POST /info/lfs/verify` | `GiteaBackend.Verify()` | `VerifyHandler` |
| 权限校验 | `getAuthenticatedRepository()` | `getAuthenticatedRepository()` | 完全相同 |
| 配额校验 | `BatchHandler` L215/L258 | `BatchHandler` L215/L258 | 完全相同 |
