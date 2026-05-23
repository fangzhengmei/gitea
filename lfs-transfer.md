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
Pure SSH 传输底层**复用大部分 HTTP API 的业务逻辑**，但存在重要差异：

| SSH 操作 | 调用的内部 HTTP API | 复用的业务逻辑 | 重要差异 |
|----------|-------------------|----------------|----------|
| `Batch` | `POST /api/internal/repo/{repo}/info/lfs/objects/batch` | `BatchHandler` - 认证、配额校验、对象存在性检查 | 无 |
| `Download` | `GET /api/internal/repo/{repo}/info/lfs/objects/{oid}` | `DownloadHandler` - 内容读取 | ⚠️ **不传递 Range 头**，不支持断点续传 |
| `Upload` | `PUT /api/internal/repo/{repo}/info/lfs/objects/{oid}/{size}` | `UploadHandler` - 哈希校验、大小校验、存储写入 | ⚠️ **不使用 Transfer-Encoding: chunked**，改用 Content-Length |
| `Verify` | `POST /api/internal/repo/{repo}/info/lfs/verify` | `VerifyHandler` - 存在性验证 | 无 |
| `Lock` | `POST /api/internal/repo/{repo}/info/lfs/locks` | 锁管理逻辑 | 无 |

- **内部认证**: 通过 `X-Gitea-Internal-Auth` 头携带 `setting.InternalToken` 进行内部服务认证
- **URL 转换**: `modules/lfstransfer/backend/util.go:toInternalLFSURL` 将外部 URL 转换为内部 API 路径
- **权限检查差异**: Pure SSH 路径下**不触发仓库 Scope 检查**（非 API Token 场景），但仍会执行其他所有权限校验

### 5.5 上传校验流程
Pure SSH 上传与 HTTP 上传**在核心校验逻辑层面走相同的代码路径**（满足条件时）：

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

### 5.7 Pure SSH 上传链路与 HTTP 路径的异同

虽然两者最终都调用 `UploadHandler` 执行业务逻辑，但在请求头策略和内容长度处理上存在显著差异。

#### 5.7.1 请求头对比

| 头字段 | HTTP 上传路径 | Pure SSH 上传路径 | 说明 |
|--------|--------------|-------------------|------|
| `Authorization` | ✅ Bearer JWT | ✅ Bearer JWT | 相同，都使用 LFS JWT Token 进行认证 |
| `Transfer-Encoding` | ✅ `chunked` (在 batch 响应中通知) | ❌ 不设置 | HTTP 路径通过 batch 响应主动通知客户端使用分块编码 |
| `Content-Length` | ❌ 依赖 chunked，不设置 | ✅ `strconv.FormatInt(size, 10)` | SSH 路径显式设置精确的内容长度 |
| `Content-Type` | 由客户端决定 (通常 `application/octet-stream`) | ✅ `application/octet-stream` | SSH 路径显式指定 MIME 类型 |
| `X-Gitea-Internal-Auth` | ❌ 不需要 | ✅ `setting.InternalToken` | SSH 内部调用专用认证头 |
| `Accept` | 由客户端决定 | ❌ 不设置 | - |

**代码实现对比**:

HTTP 上传 batch 响应 (`services/lfs/server.go:504-508`):
```go
rep.Actions["upload"] = lfs_module.NewLink(rc.UploadLink(pointer)).
    WithHeader("Authorization", rc.Authorization).
    WithHeader("Transfer-Encoding", "chunked") // 通知客户端使用分块上传
```

Pure SSH 上传 (`modules/lfstransfer/backend/backend.go:233-241`):
```go
headers := map[string]string{
    headerAuthorization:     g.authToken,
    headerGiteaInternalAuth: g.internalAuth,  // 内部认证
    headerContentType:       mimeOctetStream,
    headerContentLength:     strconv.FormatInt(size, 10), // 显式设置 Content-Length
}
req := newInternalRequestLFS(g.ctx, toInternalLFSURL(action.Href), http.MethodPut, headers, nil)
req.Body(r) // 流式传输数据
```

#### 5.7.2 内容长度策略对比

| 特性 | HTTP 上传路径 | Pure SSH 上传路径 |
|------|--------------|-------------------|
| **size 来源** | URL 路径参数 `/objects/{oid}/{size}` | `git-lfs-transfer` 协议参数 + URL 路径参数 |
| **传输编码** | `Transfer-Encoding: chunked` (通知客户端) | `Content-Length: {size}` (内部调用使用) |
| **服务端解析** | `UploadHandler` 从 URL 路径解析 size | `UploadHandler` 从 URL 路径解析 size (复用相同逻辑) |
| **Go HTTP 客户端行为** | 由外部 git-lfs 客户端决定 | 根据 Content-Length 自动判断是否 chunked |

**关键差异分析**:

1. **HTTP 路径的 chunked 设计**：
   - 在 batch 响应中设置 `Transfer-Encoding: chunked` 是为了兼容 git-lfs 客户端的行为
   - 参考：[git-lfs basic_upload.go#L58-59](https://github.com/git-lfs/git-lfs/blob/main/tq/basic_upload.go#L58-59)
   - 这样客户端可以在不知道总大小时流式上传
   - 但 Gitea 实际上在 URL 路径中已经包含了 size，所以这是"带长度的分块上传"

2. **Pure SSH 路径的 Content-Length 设计**：
   - `git-lfs-transfer` 协议在调用 Upload 方法时已经传递了精确的 size
   - GiteaBackend 直接使用这个 size 设置 `Content-Length` 头
   - 内部 HTTP 调用时，Go 标准库会根据 Content-Length 决定传输方式
   - 避免了分块编码的额外开销，传输效率更高

3. **服务端处理一致性**：
   - 无论哪种路径，`UploadHandler` 都从 URL 路径参数解析 size (`services/lfs/server.go:315`)
   - 无论哪种传输编码，`hashingReader` 都会实时校验实际传输字节数与声明 size 是否一致
   - **在数据完整性校验层面**，最终的哈希和大小校验逻辑走相同的代码路径

#### 5.7.3 完整链路对比表

| 阶段 | HTTP 上传链路 | Pure SSH 上传链路 |
|------|--------------|-------------------|
| **1. 批请求** | `POST /info/lfs/objects/batch` (HTTP) | `Processor.Batch()` → 内部 HTTP API |
| **2. 认证** | `Authorization` 头 → `authenticate()` | `Authorization` + `X-Gitea-Internal-Auth` → `authenticate()` |
| **3. 配额校验** | `BatchHandler` L215/L258 | 复用 `BatchHandler` 相同逻辑 |
| **4. 上传数据** | git-lfs 客户端 → `PUT /objects/{oid}/{size}` | `Processor.Upload()` → 内部 `PUT /api/internal/repo/...` |
| **5. 数据读取** | `ctx.Req.Body` (由 net/http 处理 chunked) | `req.Body(r)` (io.Reader 流式传输) |
| **6. 哈希校验** | `hashingReader` 实时计算 SHA256 | 复用 `hashingReader` 相同逻辑 |
| **7. 大小校验** | `hashingReader` 检查 written == size | 复用 `hashingReader` 相同逻辑 |
| **8. 跨仓库检查** | `LFSObjectAccessible` | 复用 `LFSObjectAccessible` 相同逻辑 |
| **9. 元数据创建** | `NewLFSMetaObject` | 复用 `NewLFSMetaObject` 相同逻辑 |

**核心结论**：Pure SSH 传输在**传输层**（请求头、编码方式）与 HTTP 路径不同，在**核心业务层**（配额校验、哈希校验、大小校验、存储操作）**在满足特定条件时**复用相同的代码逻辑；但在部分权限检查（仓库 Scope）和高级功能（ServeDirect、断点续传）层面存在差异，不能简单认为"行为完全一致"。

### 5.7.4 ServeDirect 直链与 Pure SSH 内部 URL 的兼容性边界

⚠️ **重要发现**：`SERVE_DIRECT = true` 与 Pure SSH 传输**完全不兼容**，同时启用会导致下载失败。

#### 兼容性问题分析

**URL 转换逻辑** (`modules/lfstransfer/backend/util.go:92-108`):
```go
func toInternalLFSURL(s string) string {
    pos1 := strings.Index(s, "://")
    if pos1 == -1 { return "" }
    
    appSubURLWithSlash := setting.AppSubURL + "/"
    pos2 := strings.Index(s[pos1+3:], appSubURLWithSlash)
    if pos2 == -1 { return "" }  // ⚠️  URL 不包含 AppSubURL 则返回空
    
    routePath := s[pos1+3+pos2+len(appSubURLWithSlash):]
    fields := strings.SplitN(routePath, "/", 3)
    if len(fields) < 3 || !strings.HasPrefix(fields[2], "info/lfs") { return "" }
    return setting.LocalURL + "api/internal/repo/" + routePath
}
```

**不兼容的完整链路**：
```
SERVE_DIRECT = true
    ↓
buildObjectResponse() 生成 MinIO/Azure 预签名 URL
    (如: https://minio.example.com/gitea-lfs/ab/cd/abcdef1234...)
    ↓
Batch API 返回给 GiteaBackend.Batch()
    ↓
GiteaBackend.Download() 获取 action.Href = 预签名 URL
    ↓
toInternalLFSURL(预签名 URL) → 返回空字符串
    (因为预签名 URL 不包含 setting.AppSubURL)
    ↓
newInternalRequestLFS() → 检查 isInternalLFSURL("") → 返回 nil
    ↓
req.Response() → nil 指针调用，panic！💥
```

#### 兼容性矩阵

| 配置组合 | HTTP 下载 | Pure SSH 下载 | 说明 |
|---------|----------|---------------|------|
| `SERVE_DIRECT = false` | ✅ Gitea 代理 | ✅ 正常工作 | 推荐组合 |
| `SERVE_DIRECT = true` | ✅ 对象存储直链 | ❌ panic/失败 | 不兼容，必须关闭 ServeDirect 才能使用 Pure SSH |
| `LFS_ALLOW_PURE_SSH = true` + `SERVE_DIRECT = true` | ✅ 对象存储直链 | ❌ 完全不可用 | 危险组合，会导致 SSH 下载失败 |

#### 内部 URL 安全检查

为防止 SSRF 攻击，`newInternalRequestLFS` 严格限制只能访问内部 URL：
```go
// modules/lfstransfer/backend/util.go:130-133
func newInternalRequestLFS(ctx context.Context, internalURL, method string, ...) *httplib.Request {
    if !isInternalLFSURL(internalURL) {
        return nil  // ⚠️  非内部 URL 直接拒绝
    }
    // ...
}
```

**安全设计权衡**：
- ✅ 防止 SSRF 攻击，确保 Pure SSH 只能调用 Gitea 内部 API
- ❌ 导致无法使用对象存储直链，必须经过 Gitea 代理下载

### 5.7.5 HTTP 与 Pure SSH 共用配额校验机制的条件

两种传输方式**在满足以下全部条件时**，可视为共用同一套配额和校验机制：

| 条件 | 说明 |
|------|------|
| **1. HTTP 路径不使用 API Token 认证** | 使用 JWT/Session/Basic Auth，而非 Personal Access Token |
| **2. 关闭 SERVE_DIRECT** | 不使用对象存储直链下载 |
| **3. 不依赖 HTTP Range 断点续传** | 接受从头开始下载整个文件 |
| **4. 核心校验逻辑位于 handler 内部** | 而非路由中间件 |

#### 共用的配额和校验机制（满足条件时）

| 校验项 | HTTP 路径 | Pure SSH 路径 | 复用位置 |
|--------|----------|---------------|----------|
| MaxBatchSize | ✅ | ✅ | `BatchHandler` L215 |
| MaxFileSize | ✅ | ✅ | `BatchHandler` L258 |
| Actions 用户权限 | ✅ | ✅ | `authenticate` L544 |
| 常规用户权限 | ✅ | ✅ | `authenticate` L554 |
| LFS JWT Token 校验 | ✅ | ✅ | `parseToken` / `handleLFSToken` |
| 令牌写约束 | ✅ | ✅ | `handleLFSToken` L600 |
| 仓库绑定校验 | ✅ | ✅ | `handleLFSToken` L596 |
| 用户状态校验 | ✅ | ✅ | `handleLFSToken` L609 |
| 最终权限校验 | ✅ | ✅ | `handleLFSToken` L613 |
| 跨仓库访问检查 | ✅ | ✅ | `UploadHandler` L340 |
| 哈希实时校验 | ✅ | ✅ | `contentStore.Put` |
| 大小完整性校验 | ✅ | ✅ | `contentStore.Put` |
| 对象存在性检查 | ✅ | ✅ | `BatchHandler` 内 |

#### 不共用的机制（始终存在差异）

| 校验项 | HTTP 路径 | Pure SSH 路径 | 差异原因 |
|--------|----------|---------------|----------|
| 仓库 Scope 检查 | ✅ API Token 场景 | ❌ 不生效 | Pure SSH 使用 LFS JWT Token，`IsApiToken != true` |
| Range 断点续传 | ✅ 支持 | ❌ 不支持 | `GiteaBackend.Download` 不传递 Range 头 |
| ServeDirect 直链 | ✅ 支持 | ❌ 不支持 | 预签名 URL 无法转换为内部 URL |
| Transfer-Encoding: chunked | ✅ 通知客户端 | ❌ 不使用 | Pure SSH 显式设置 Content-Length |

#### 关键判定标准

> **可视为共用一套机制的判定条件**：当且仅当 HTTP 路径使用 JWT/Session/Basic Auth 认证，且不依赖 ServeDirect 和 Range 断点续传时，两种传输方式在配额校验、权限认证、数据完整性校验等核心业务逻辑层面才是等价的。

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
- **完整流程包含三层权限检查**：Actions 用户权限 → 常规用户权限 → LFS JWT Token 认证

```go
func authenticate(ctx *context.Context, repository *repo_model.Repository, authorization string, requireSigned, requireWrite bool) bool {
    // 1. 优先检查 Actions 用户权限 (CI/CD 场景)
    if taskID, ok := user_model.GetActionsUserTaskID(ctx.Doer); ok {
        perm, err := access_model.GetActionsUserRepoPermission(ctx, repository, ctx.Doer, taskID)
        if err != nil {
            log.Error("Unable to GetActionsUserRepoPermission for task[%d] Error: %v", taskID, err)
            return false
        }
        return perm.CanAccess(accessMode, unit.TypeCode)
    }

    // 2. 检查常规用户权限 (登录用户 / 匿名访问)
    perm, _ := access_model.GetDoerRepoPermission(ctx, repository, ctx.Doer)
    canAccess := perm.CanAccess(accessMode, unit.TypeCode)
    if canAccess && (!requireSigned || ctx.IsSigned) {
        return true
    }
    
    // 3. 回退到 LFS JWT Token 认证 (git-lfs 客户端场景)
    user, err := parseToken(ctx, authorization, repository, accessMode)
    if err != nil {
        return false
    }
    ctx.Doer = user
    return true
}
```

#### 6.2.1 Actions 用户权限生效点
- **触发条件**: `ctx.Doer` 是 Actions 系统用户（ID = `user_model.ActionsUserID`）
- **生效位置**: `services/lfs/server.go:authenticate` L544-551 (最优先检查)
- **权限来源**: `GetActionsUserRepoPermission` 从 Actions Task 的 Job 信息中获取权限范围
- **检查内容**:
  1. 验证 Task 所属 Job 的仓库权限
  2. 考虑 Actions Token 的权限级别（读/写/管理）
  3. 验证 Task 与目标仓库的关联关系

#### 6.2.2 仓库 Scope 检查生效点
- **触发条件**: **仅在使用 API Token 认证时** (`ctx.Data["IsApiToken"] == true`)
- **生效位置**: `services/lfs/server.go:getAuthenticatedRepository` L469-473 (认证通过后)
- **检查函数**: `context.CheckRepoScopedToken` (`services/context/permission.go:94-96`)
- **检查逻辑**:
  ```go
  func CheckTokenScopes(ctx *Context, repo *repo_model.Repository, scopes ...auth_model.AccessTokenScope) {
      if ctx.Data["IsApiToken"] != true {
          return  // ⚠️  不是 API Token 直接返回，不做任何检查
      }
      // 后续 scope 检查...
  }
  ```
- **实际生效场景**:
  - ✅ HTTP 路径使用个人访问令牌 (PAT) 认证时
  - ✅ HTTP 路径使用 OAuth2 Token 认证时
  - ❌ Pure SSH 路径（使用 LFS JWT Token，不设置 `IsApiToken`）
  - ❌ Session/浏览器登录场景
- **额外限制**: 如果 Token 只有 public scope，不能访问私有仓库 (`PublicOnly` 检查)

#### 6.2.3 Pure SSH 路径下的权限门禁真实生效条件

Pure SSH 路径与 HTTP 路径的权限检查存在差异，以下是真实生效的门禁：

| 权限检查 | HTTP 路径 (API Token) | HTTP 路径 (JWT/Session) | Pure SSH 路径 |
|---------|----------------------|------------------------|---------------|
| Actions 用户权限 | ✅ | ✅ | ✅ |
| 常规用户权限 (GetDoerRepoPermission) | ✅ | ✅ | ✅ |
| LFS JWT Token 校验 | ✅ | ✅ | ✅ |
| **仓库 Scope 检查** | ✅ | ❌ | ❌ |
| 令牌读写约束 | ✅ | ✅ | ✅ |
| 仓库绑定校验 | ✅ | ✅ | ✅ |

**Pure SSH 权限链路详解**：
```
SSH 连接建立
    ↓
1. SSH Key 认证 (cmd/serv.go 前置认证)
    ↓
2. 生成 LFS JWT Token (绑定 RepoID + Op + UserID)
    ↓
3. lfstransfer.Processor 启动
    ↓
4. 内部 API 调用 (携带 X-Gitea-Internal-Auth + Authorization)
    ↓
5. authenticate() 函数执行:
   a. 检查是否是 Actions 用户 → 如果是，用 Actions 权限
   b. 检查常规用户权限 (通过 JWT 解析出的 UserID)
   c. 如果失败，解析 LFS JWT Token 并校验
    ↓
6. getAuthenticatedRepository() 执行:
   a. CheckRepoScopedToken() → 因 IsApiToken != true 直接返回
   b. ⚠️  仓库 Scope 检查在此处被跳过
```

**关键结论**：Pure SSH 路径下**不会触发仓库 Scope 检查**，因为内部 API 调用使用 LFS JWT Token 认证，而非 API Token。权限主要通过以下机制保障：
1. SSH Key 强身份认证
2. LFS JWT Token 的 RepoID 绑定（跨仓库拒绝）
3. LFS JWT Token 的 Op 约束（写操作必须是 upload）
4. 最终 `GetDoerRepoPermission` 的用户仓库权限校验

### 6.3 Token 解析与校验
- **函数**: `services/lfs/server.go:handleLFSToken` (L580-622)
- **关键特性**: 读写路径上存在不对称约束

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
    
    // 3. 校验操作权限 - 读写路径不对称约束
    // ⚠️  注意：只有写路径有严格的 Op 约束，读路径没有 Op 约束！
    if mode == perm_model.AccessModeWrite && claims.Op != "upload" {
        return nil, errors.New("invalid token claim") // 写操作必须使用 upload 令牌
    }
    // 读路径：download 令牌 和 upload 令牌 都可以用于下载！
    
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

#### 6.3.1 令牌读写约束差异详解

| 令牌 Op | 用于读操作 (下载) | 用于写操作 (上传) |
|---------|------------------|------------------|
| `"download"` | ✅ 允许 | ❌ 拒绝 |
| `"upload"` | ✅ 允许 | ✅ 允许 |

**设计意图分析**:
- **写路径严格约束**：防止下载令牌被滥用进行上传操作
- **读路径宽松约束**：upload 令牌可以用于下载，避免客户端需要同时维护两种令牌
- **安全边界**：最终都会通过 `GetDoerRepoPermission` 再次校验用户的实际仓库权限

**令牌生成位置**:
- **download 令牌**: `BatchHandler` 处理下载批请求时生成 (`op = "download"`)
- **upload 令牌**: `BatchHandler` 处理上传批请求时生成 (`op = "upload"`)
- **SSH authenticate**: `cmd/serv.go:274-294` 根据操作类型生成对应令牌
- **Pure SSH**: `cmd/serv.go:264-271` 根据操作类型生成对应令牌

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

| 配置项 | 默认值 | 说明 | 兼容性说明 |
|--------|--------|------|------------|
| `LFS_START_SERVER` | false | 是否启用 LFS 服务 | - |
| `LFS_ALLOW_PURE_SSH` | false | 是否启用 Pure SSH LFS 传输 | ⚠️ 与 `SERVE_DIRECT = true` 不兼容 |
| `LFS_MAX_FILE_SIZE` | 0 (无限制) | 单文件最大大小（字节） | HTTP 与 Pure SSH 共用 |
| `LFS_MAX_BATCH_SIZE` | 0 (无限制) | 单次批量请求最大对象数 | HTTP 与 Pure SSH 共用 |
| `LFS_HTTP_AUTH_EXPIRY` | 24h | JWT Token 有效期 | HTTP 与 Pure SSH 共用 |
| `SERVE_DIRECT` | false | 是否启用对象存储直链 | ⚠️ 与 `LFS_ALLOW_PURE_SSH = true` 不兼容 |
| `LFSClient.BATCH_SIZE` | 20 | 内部客户端每批处理对象数 | 仅内部镜像同步使用 |
| `LFSClient.BATCH_OPERATION_CONCURRENCY` | 8 | 内部客户端并发传输数 | 仅内部镜像同步使用 |

> **重要兼容性警告**：`LFS_ALLOW_PURE_SSH = true` 与 `SERVE_DIRECT = true` 不能同时启用。同时启用会导致 Pure SSH 下载时 `toInternalLFSURL` 无法转换对象存储预签名 URL，最终引发 nil 指针 panic。

## 十、安全边界总结

1. **凭据隔离**: JWT Token 与仓库 ID、操作类型、用户 ID 三重绑定，防止跨仓库越权
2. **令牌读写约束不对称**: 写操作强制要求 `Op="upload"`，读操作允许多种令牌，兼顾安全与便利
3. **Actions 用户权限**: CI/CD 场景下优先使用 Task/Job 级别的细粒度权限控制
4. **仓库 Scope 检查**: **仅在 API Token 场景生效**，必须具备 `write_repo`/`read_repo` scope，且 Public scope 不能访问私有仓库（Pure SSH 路径不触发此检查）
5. **对象访问控制**: `LFSObjectAccessible` 确保用户只能访问其有权限仓库中的对象
6. **配额防护**: 入口处校验批量大小和单文件大小，防止滥用
7. **完整性校验**: 上传过程中实时计算 SHA256 哈希和大小，确保数据完整性
8. **预签名 URL**: 对象存储签名 URL 有效期仅 5 分钟，且不暴露 Gitea 凭据
9. **内部 API 认证**: Pure SSH 传输通过 `X-Gitea-Internal-Auth` 头进行内部服务间认证
10. **传输通道安全**: SSH 传输使用 pkt-line 格式在加密通道内传输，避免额外暴露 HTTP 端点
11. **配置兼容性**: `SERVE_DIRECT = true` 与 Pure SSH 完全不兼容，同时启用会导致下载 panic

## 十一、传输机制对比总结

| 特性 | HTTP 传输 | Pure SSH 传输 |
|------|----------|---------------|
| **协议** | HTTP/HTTPS | SSH (git-lfs-transfer) |
| **批接口** | 独立 HTTP Batch API | 通过 GiteaBackend 调用内部 Batch API |
| **Range 下载 (断点续传)** | ✅ 支持（Gitea 处理 / 对象存储处理） | ❌ 不支持（GiteaBackend 不传递 Range 头） |
| **Chunked 上传** | ✅ Batch 响应通知客户端使用 chunked | ❌ 内部调用使用 Content-Length，不使用 chunked |
| **上传请求头** | 由 git-lfs 客户端决定 | 显式设置 Content-Length / Content-Type |
| **内部认证** | 不需要 | `X-Gitea-Internal-Auth` 头 |
| **认证方式** | Basic Auth / JWT Token / Session / API Token | SSH Key + 内部 JWT Token |
| **Actions 支持** | ✅ 支持（authenticate 优先检查） | ✅ 支持（调用相同 authenticate 函数） |
| **仓库 Scope 检查** | ✅ API Token 场景生效 | ❌ 不生效（非 API Token 场景） |
| **凭据有效期** | JWT 默认 24h | JWT 默认 24h |
| **配额校验** | ✅ BatchHandler 入口 | ✅ 调用相同 BatchHandler 代码 |
| **完整性校验** | ✅ hashingReader 实时校验 | ✅ 调用相同 hashingReader 代码 |
| **ServeDirect 兼容** | ✅ 支持对象存储直链 | ❌ 不兼容（会导致 panic） |
| **适用场景** | 大部分用户、CI/CD | SSH 环境偏好者、纯内网部署 |
| **客户端配置** | `git config lfs.url` | `git config lfs.sshtransfer always` |

### 11.1 关键代码路径对照

| 功能 | HTTP 路径 | SSH 路径 | 复用逻辑 | 条件性说明 |
|------|----------|----------|----------|------------|
| 批量请求 | `POST /info/lfs/objects/batch` | `GiteaBackend.Batch()` | `BatchHandler` | 核心逻辑相同，URL 不同 |
| 下载 | `GET /info/lfs/objects/{oid}` | `GiteaBackend.Download()` | `DownloadHandler` | 仅数据读取相同，Range 不传递 |
| 上传 | `PUT /info/lfs/objects/{oid}/{size}` | `GiteaBackend.Upload()` | `UploadHandler` | 仅业务逻辑相同，请求头不同 |
| 验证 | `POST /info/lfs/verify` | `GiteaBackend.Verify()` | `VerifyHandler` | 核心逻辑相同 |
| 权限校验 | `getAuthenticatedRepository()` | `getAuthenticatedRepository()` | 大部分相同 | 仓库 Scope 检查在 Pure SSH 下不生效 |
| 配额校验 | `BatchHandler` L215/L258 | `BatchHandler` L215/L258 | 核心逻辑相同 | 触发条件一致 |

> **重要说明**："复用逻辑"仅指最终调用的 handler 函数相同，不代表行为完全一致。实际行为还受传输层参数（如请求头、URL 类型）的影响。

### 11.2 Pure SSH 下载链路中 Range 请求的实际传递分析

⚠️ **重要修正**：虽然 `DownloadHandler` 本身支持 Range 请求（断点续传），但 Pure SSH 传输链路**实际上不支持断点续传**。

#### 11.2.1 代码证据

**GiteaBackend.Download 实现** (`modules/lfstransfer/backend/backend.go:159-209`):
```go
func (g *GiteaBackend) Download(oid string, args transfer.Args) (_ io.ReadCloser, _ int64, retErr error) {
    // ... 解析 action href ...
    
    headers := map[string]string{
        headerAuthorization:     g.authToken,
        headerGiteaInternalAuth: g.internalAuth,
        headerAccept:            mimeOctetStream,
        // ⚠️  注意：这里没有设置 Range 头！
    }
    req := newInternalRequestLFS(g.ctx, toInternalLFSURL(action.Href), http.MethodGet, headers, nil)
    resp, err := req.Response()
    
    // ⚠️  只接受 200 OK，不接受 206 Partial Content
    if resp.StatusCode != http.StatusOK {
        return nil, 0, statusCodeToErr(resp.StatusCode)
    }
    
    // 通过自定义头获取大小，而不是 Content-Range
    respSize, err := strconv.ParseInt(resp.Header.Get("X-Gitea-LFS-Content-Length"), 10, 64)
    return resp.Body, respSize, nil
}
```

#### 11.2.2 不支持断点续传的三层原因

| 层面 | 原因 | 代码位置 |
|------|------|----------|
| **协议层** | `git-lfs-transfer` 协议的 `Backend` 接口没有定义 `offset`/`size` 参数，无法传递断点信息 | `transfer.Backend` 接口定义 |
| **实现层** | `GiteaBackend.Download` 没有设置 `Range` 请求头 | `modules/lfstransfer/backend/backend.go:181-185` |
| **响应处理** | 只接受 `200 OK`，不处理 `206 Partial Content` 状态码 | `modules/lfstransfer/backend/backend.go:200-202` |

#### 11.2.3 对比：HTTP 路径 vs Pure SSH 路径的下载流程

```
HTTP 下载路径 (支持断点续传):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ git-lfs     │     │ Gitea       │     │ Content     │
│ Client      │────▶│ Download    │────▶│ Store       │
│  (支持)     │     │ Handler     │     │             │
│ Range Header│     │ (解析 Range)│     │ (Seek offset)│
└─────────────┘     └─────────────┘     └─────────────┘

Pure SSH 下载路径 (不支持断点续传):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ git-lfs     │     │ GiteaBackend│     │ Gitea       │     │ Content     │
│ Client      │────▶│ .Download() │────▶│ Download    │────▶│ Store       │
│  (支持)     │     │ (❌ 不传递) │     │ Handler     │     │             │
│ Range Header│     │  Range      │     │ (Range 无用)│     │ (Seek 0)    │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

#### 11.2.4 关键结论

1. **"复用 DownloadHandler" ≠ "支持断点续传"**：虽然底层 `DownloadHandler` 代码支持 Range，但上层 `GiteaBackend` 没有传递 Range 请求的能力
2. **git-lfs-transfer 协议限制**：该协议的 `transfer.Backend` 接口设计不支持断点续传参数
3. **实际行为**：Pure SSH 下载总是从字节 0 开始传输整个文件，不支持从断点继续
4. **X-Gitea-LFS-Content-Length 头**：仅用于传递内容长度，不用于断点续传（避免被反向代理或压缩修改）


## 十二、认证与授权体系完整总结

### 12.1 完整认证流程栈

```
请求进入
    ↓
┌─────────────────────────────────────────┐
│ 1. Actions 用户权限检查                  │
│    (authenticate L544-551)              │
│    - GetActionsUserTaskID               │
│    - GetActionsUserRepoPermission       │
│    - 基于 Task/Job 的权限范围校验        │
└─────────────────────────────────────────┘
    ↓ 不是 Actions 用户
┌─────────────────────────────────────────┐
│ 2. 常规用户权限检查                      │
│    (authenticate L553-565)              │
│    - GetDoerRepoPermission              │
│    - 支持匿名访问 / Session 登录         │
└─────────────────────────────────────────┘
    ↓ 认证失败
┌─────────────────────────────────────────┐
│ 3. LFS JWT Token 认证                   │
│    (authenticate L567-577)              │
│    - parseToken → handleLFSToken        │
│    - 令牌读写约束校验 (L600-602)        │
│    - 仓库绑定校验 (L596-598)            │
│    - 用户状态校验 (L609-611)            │
│    - 最终权限校验 (L613-620)            │
└─────────────────────────────────────────┘
    ↓ 认证通过
┌─────────────────────────────────────────┐
│ 4. 仓库 Scope 检查                       │
│    (getAuthenticatedRepository L469-473)│
│    - CheckRepoScopedToken               │
│    - 仅对 API Token 生效                │
│    - 检查 write_repo / read_repo scope  │
│    - PublicOnly 限制私有仓库访问         │
└─────────────────────────────────────────┘
    ↓
业务逻辑执行
```

### 12.2 令牌读写约束不对称性总结

| 令牌类型 | 生成时机 | 下载权限 | 上传权限 | 约束代码 |
|---------|---------|---------|---------|---------|
| `op="download"` | 下载批请求 | ✅ 允许 | ❌ 拒绝 | `mode == Write && Op != "upload"` |
| `op="upload"` | 上传批请求 | ✅ 允许 | ✅ 允许 | 无约束（读路径宽松） |

**设计权衡**：
- 安全：写操作严格限制，防止下载令牌被滥用
- 便利：读操作宽松，避免客户端维护两种令牌
- 兜底：最终都会通过 `GetDoerRepoPermission` 校验用户实际权限

### 12.3 上传内容长度策略对比总结

| 策略维度 | HTTP 上传 | Pure SSH 上传 |
|---------|----------|---------------|
| **Batch 响应头** | `Transfer-Encoding: chunked` | 无（不经过 Batch 响应） |
| **实际请求头** | 由 git-lfs 客户端决定 | `Content-Length: {size}` |
| **size 传递** | URL 路径 `/objects/{oid}/{size}` | 协议参数 + URL 路径 |
| **传输效率** | chunked 编码有额外开销 | Content-Length 更高效 |
| **完整性校验** | `hashingReader` 实时校验 | 复用 `hashingReader` |
| **内部认证头** | 无 | `X-Gitea-Internal-Auth` |

### 12.4 关键生效点速查表

| 检查项 | 生效位置 | 触发条件 |
|--------|----------|---------|
| MaxBatchSize | `BatchHandler` L215 | 批量请求入口 |
| MaxFileSize | `BatchHandler` L258 | 对象不存在且需上传 |
| Actions 权限 | `authenticate` L544 | ctx.Doer 是 Actions 用户 |
| 仓库 Scope 检查 | `getAuthenticatedRepository` L469 | **仅**使用 API Token 认证时 |
| 令牌写约束 | `handleLFSToken` L600 | mode == AccessModeWrite |
| 令牌读约束 | 无 | 读路径宽松，download 和 upload 令牌均可 |
| 跨仓库访问 | `UploadHandler` L340 | 对象已存在但当前仓库无关联 |
| 哈希/大小校验 | `contentStore.Put` | 所有上传场景 |
| 内部 API 认证 | `newInternalRequestLFS` | Pure SSH 传输场景 |
| Range 请求处理 | `DownloadHandler` L126-147 | **仅** HTTP 路径，Pure SSH 不传递 |

### 12.5 边界修正总结

| 之前的错误结论 | 修正后的正确结论 | 证据代码 |
|---------------|-----------------|----------|
| 仓库 Scope 检查在所有场景生效 | **仅在 API Token 场景生效** | `services/context/permission.go:17` `if ctx.Data["IsApiToken"] != true { return }` |
| Pure SSH 路径支持断点续传 | **不支持**，GiteaBackend 不传递 Range 头 | `modules/lfstransfer/backend/backend.go:181-185` 未设置 Range 头 |
| 复用 DownloadHandler 就支持断点续传 | 上层协议不传递 Range，底层能力无法发挥 | `modules/lfstransfer/backend/backend.go:200-202` 只接受 200 OK |
| Pure SSH 与 HTTP 权限检查完全相同 | 仓库 Scope 检查在 Pure SSH 下被跳过 | 对比 `ctx.Data["IsApiToken"]` 设置场景 |
| ServeDirect 与 Pure SSH 可以同时启用 | **完全不兼容**，同时启用会导致下载 panic | `modules/lfstransfer/backend/util.go:92-108` `toInternalLFSURL` 无法转换预签名 URL |
| 业务层 100% 复用相同代码逻辑 | **在满足特定条件时**核心业务逻辑复用相同代码 | 见 5.7.5 节共用条件判定标准 |
| 两种传输方式行为完全一致 | 仅在配额/校验/存储层面等价，传输层/部分权限/高级功能存在差异 | 见 11.2 节 Range 传递分析 |

### 12.6 最终可判定标准汇总

#### 12.6.1 ServeDirect 与 Pure SSH 兼容性判定

**可同时启用**：❌ 绝对不可

| 判定项 | 结果 |
|--------|------|
| `SERVE_DIRECT = true` + `LFS_ALLOW_PURE_SSH = true` | ❌ 危险组合，SSH 下载会 panic |
| `SERVE_DIRECT = false` + `LFS_ALLOW_PURE_SSH = true` | ✅ 安全组合，推荐 |
| `SERVE_DIRECT = true` + `LFS_ALLOW_PURE_SSH = false` | ✅ 安全组合 |
| `SERVE_DIRECT = false` + `LFS_ALLOW_PURE_SSH = false` | ✅ 默认安全 |

**根本原因**：Pure SSH 的 URL 转换机制无法处理对象存储预签名 URL，导致 nil 指针 panic。

#### 12.6.2 配额与校验机制等价性判定

**可视为共用一套机制**：✅ 当且仅当以下全部条件满足

1. ✅ HTTP 路径使用 JWT/Session/Basic Auth 认证（非 API Token）
2. ✅ `SERVE_DIRECT = false`（不使用对象存储直链）
3. ✅ 不依赖 HTTP Range 断点续传功能
4. ✅ 核心校验逻辑位于 handler 函数内部（非路由中间件）

**满足条件时，以下校验完全等价**：
- MaxBatchSize / MaxFileSize 配额检查
- Actions 用户权限检查
- 常规用户仓库权限检查
- LFS JWT Token 仓库绑定与操作约束
- 跨仓库对象访问控制
- SHA256 哈希与文件大小完整性校验
- 对象存储读写操作

**不满足条件时的差异**：
| 不满足的条件 | 差异表现 |
|-------------|----------|
| 使用 API Token | HTTP 路径额外触发仓库 Scope 检查，Pure SSH 不触发 |
| `SERVE_DIRECT = true` | HTTP 路径使用对象存储直链，Pure SSH 完全不可用 |
| 依赖 Range 续传 | HTTP 路径支持断点续传，Pure SSH 从头下载 |

#### 12.6.3 传输行为一致性判定

**完全一致**：❌ 不存在这样的场景

**核心业务层一致**：✅ 满足 12.6.2 节全部条件时，配额、校验、存储行为一致

**始终存在差异**：
| 层面 | 差异项 | HTTP 路径 | Pure SSH 路径 |
|------|--------|----------|---------------|
| 传输层 | 请求头设置 | 由客户端决定 | 固定设置 Content-Length |
| 传输层 | 传输编码 | Transfer-Encoding: chunked | Content-Length: {size} |
| 传输层 | 断点续传 | 支持 Range | 不支持 |
| 传输层 | URL 类型 | 可能是外部预签名 URL | 始终是内部 API URL |
| 权限层 | 仓库 Scope | API Token 场景生效 | 永不生效 |
| 安全层 | 内部认证头 | 不需要 | 需要 X-Gitea-Internal-Auth |
