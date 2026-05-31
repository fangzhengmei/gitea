# Gitea 软件包注册服务核心机制分析报告

## 1. 概述

Gitea 软件包注册服务是一个多格式的包管理系统，支持包括 Container、Maven、npm、PyPI、NuGet、RPM、Debian 等在内的 20+ 种包格式。本报告深入分析其核心协作机制：凭据校验、命名空间隔离、二进制对象存储、去重与垃圾回收、仓库布局以及配额约束。

---

## 2. 凭据校验与鉴权流程

### 2.1 认证方式

系统支持多种认证方式，定义于 `routers/api/packages/api.go:96-113`：

| 认证方式 | 适用场景 | 核心实现 |
|---------|---------|---------|
| **Basic Auth** | 大多数包管理器 | `services/auth/basic.go` |
| **OAuth2 Token** | API 访问 | `services/auth/oauth2.go` |
| **JWT Token** | Container 镜像、Conan | `services/packages/auth.go` |
| **Reverse Proxy** | 反向代理认证 | `services/auth/reverseproxy.go` |
| **NuGet 专用** | NuGet 客户端 | `routers/api/packages/nuget/auth.go` |
| **Chef 专用** | Chef 客户端 | `routers/api/packages/chef/auth.go` |

### 2.2 JWT Token 机制

**Token 创建流程** (`services/packages/auth.go:31-54`)：

```go
// Token 结构
type packageClaims struct {
    jwt.RegisteredClaims
    PackageMeta  // 包含 UserID、Scope、ActionsUserTaskID
}

// Token 有效期：24 小时
// 签名算法：HS256
```

**Token 解析流程** (`services/packages/auth.go:56-88`)：

1. 从 `Authorization` 头提取 Bearer token
2. 使用 `GetGeneralTokenSigningSecret()` 验证签名
3. 解析 `UserID`、`Scope`、`ActionsUserTaskID`
4. 支持 Ghost User（匿名访问）和 Actions 服务账号

### 2.3 权限验证中间件

**核心函数** `reqPackageAccess(accessMode)` (`routers/api/packages/api.go:41-90`)：

```go
func reqPackageAccess(accessMode perm.AccessMode) func(ctx *context.Context) {
    return func(ctx *context.Context) {
        // 1. API Token 范围检查
        if ctx.Data["IsApiToken"] == true {
            // 检查 token 是否有 read:package / write:package 权限
            scopeMatched, _ := scope.HasScope(auth_model.AccessTokenScopeReadPackage)
            // 检查 public-only 限制
            publicOnly, _ := scope.PublicOnly()
        }
        
        // 2. 用户实际权限检查
        if ctx.Package.AccessMode < accessMode && !ctx.IsUserSiteAdmin() {
            ctx.HTTPError(http.StatusUnauthorized, ...)
        }
    }
}
```

### 2.4 上传与下载鉴权流程

**上传流程**：
```
请求到达
    ↓
verifyAuth() 中间件 (支持 Basic/OAuth2/JWT)
    ↓
PackageContexter() 初始化包上下文
    ↓
UserAssignmentWeb() 获取用户信息
    ↓
PackageAssignment() 计算用户对包的访问权限
    ↓
reqPackageAccess(Write) 检查写入权限
    ↓
包格式特定处理器执行上传
```

**下载流程**：
```
请求到达
    ↓
verifyAuth() 中间件 (匿名用户也可通过)
    ↓
PackageAssignment() 计算用户对包的访问权限
    ↓
reqPackageAccess(Read) 检查读取权限
    ↓
OpenBlobForDownload() 打开文件流
    ↓
ServePackageFile() 提供下载 (支持直接重定向到存储)
```

---

## 3. 命名空间隔离机制

### 3.1 核心数据模型

**包的三层命名空间**（基于数据库唯一约束）：

| 层级 | 字段 | 说明 |
|-----|------|------|
| **第一层** | `OwnerID` | 用户/组织 ID，实现租户隔离 |
| **第二层** | `Type` | 包类型 (TypeMaven / TypeNpm 等) |
| **第三层** | `LowerName` | 包名（小写，大小写不敏感） |

**数据库唯一约束** (`models/packages/package.go:191-200`)：
```go
type Package struct {
    ID               int64
    OwnerID          int64  `xorm:"UNIQUE(s) INDEX NOT NULL"`
    Type             Type   `xorm:"UNIQUE(s) INDEX NOT NULL"`
    LowerName        string `xorm:"UNIQUE(s) INDEX NOT NULL"`
    // ...
}
```

### 3.2 版本与文件隔离

**版本隔离** (`models/packages/package_version.go:27-37`)：
```go
type PackageVersion struct {
    ID            int64
    PackageID     int64  `xorm:"UNIQUE(s) INDEX NOT NULL"`
    LowerVersion  string `xorm:"UNIQUE(s) INDEX NOT NULL"`
    // ...
}
```

**文件隔离** (`models/packages/package_file.go:34-43`)：
```go
type PackageFile struct {
    ID           int64
    VersionID    int64  `xorm:"UNIQUE(s) INDEX NOT NULL"`
    LowerName    string `xorm:"UNIQUE(s) INDEX NOT NULL"`
    CompositeKey string `xorm:"UNIQUE(s) INDEX"`  // 额外的复合键维度
    // ...
}
```

### 3.3 访问权限计算

**核心函数** `determineAccessMode()` (`services/context/package.go:116-170`)：

```
用户访问权限计算逻辑：
├─ 组织包 (Owner 是 Organization)
│   ├─ 检查用户在组织中的最高授权级别
│   ├─ 检查各团队对 Packages 单元的访问权限
│   └─ 组织可见则至少有 Read 权限
└─ 个人包 (Owner 是 User)
    ├─ 包所有者拥有 Owner 权限
    ├─ 用户公开/受限可见则有 Read 权限
    └─ 仅公开用户允许匿名访问
```

### 3.4 Blob 访问控制

**Blob 级别的权限检查** (`models/packages/package_blob.go:125-161`)：

```go
func IsBlobAccessibleForUser(ctx context.Context, blobID int64, user *user_model.User) (bool, error) {
    // 通过 JOIN 查询验证：
    // package_blob → package_file → package_version → package → user
    // 确保 blob 关联的包对用户可见
}
```

---

## 4. 二进制对象存储、去重与垃圾回收

### 4.1 内容存储 (ContentStore)

**存储抽象层** (`modules/packages/content_store.go:21-74`)：

```go
type ContentStore struct {
    store storage.ObjectStorage  // 支持 Local / MinIO
}

// 存储路径：aa/bb/aabb0000... (SHA256 前四位分目录)
func KeyToRelativePath(key BlobHash256Key) string {
    return path.Join(key[0:2], key[2:4], string(key))
}
```

**支持的存储后端**（通过 `RegisterStorageType` 注册机制动态注册）：
- **本地文件系统** (`modules/storage/local.go`)
- **MinIO / S3 兼容** (`modules/storage/minio.go`)
- **Azure Blob Storage** (`modules/storage/azureblob.go`)

**存储类型常量** (`modules/setting/storage.go:17-23`)：
```go
const (
    LocalStorageType     StorageType = "local"
    MinioStorageType     StorageType = "minio"
    AzureBlobStorageType StorageType = "azureblob"
)
```

### 4.1.1 存储类型注册机制

**动态注册** (`modules/storage/storage.go:32-34`)：
```go
var storageMap = map[Type]NewStorageFunc{}

func RegisterStorageType(typ Type, fn NewStorageFunc) {
    storageMap[typ] = fn
}
```

**各存储类型的 init() 注册**：
- `modules/storage/local.go:175-177`: `RegisterStorageType(setting.LocalStorageType, NewLocalStorage)`
- `modules/storage/minio.go:320-322`: `RegisterStorageType(setting.MinioStorageType, NewMinioStorage)`
- `modules/storage/azureblob.go:344-346`: `RegisterStorageType(setting.AzureBlobStorageType, NewAzureBlobStorage)`

### 4.1.2 存储初始化入口

**全局初始化流程** (`modules/storage/storage.go:167-182`)：
```go
func Init() error {
    for _, f := range []func() error{
        initAttachments,
        initAvatars,
        initRepoAvatars,
        initLFS,
        initRepoArchives,
        initPackages,      // 包存储初始化
        initActions,
    } {
        if err := f(); err != nil {
            return err
        }
    }
    return nil
}
```

**包存储初始化** (`modules/storage/storage.go:235-243`)：
```go
func initPackages() (err error) {
    if !setting.Packages.Enabled {
        Packages = discardStorage("Packages isn't enabled")
        return nil
    }
    log.Info("Initialising Packages storage with type: %s", setting.Packages.Storage.Type)
    Packages, err = NewStorage(setting.Packages.Storage.Type, setting.Packages.Storage)
    return err
}
```

**通用存储工厂** (`modules/storage/storage.go:185-195`)：
```go
func NewStorage(typStr Type, cfg *setting.Storage) (ObjectStorage, error) {
    if len(typStr) == 0 {
        typStr = setting.LocalStorageType
    }
    fn, ok := storageMap[typStr]
    if !ok {
        return nil, fmt.Errorf("Unsupported storage type: %s", typStr)
    }
    return fn(context.Background(), cfg)
}
```

### 4.1.3 配置入口与层次化配置

**配置加载入口** (`modules/setting/packages.go:52-66`)：
```go
func loadPackagesFrom(rootCfg ConfigProvider) (err error) {
    sec, _ := rootCfg.GetSection("packages")
    if sec == nil {
        Packages.Storage, err = getStorage(rootCfg, "packages", "", nil)
        return err
    }
    if err = sec.MapTo(&Packages); err != nil {
        return fmt.Errorf("failed to map Packages settings: %v", err)
    }
    Packages.Storage, err = getStorage(rootCfg, "packages", "", sec)
    return err
}
```

**层次化配置优先级**（从低到高）：
1. **全局默认**：`[storage]` 节，默认 `STORAGE_TYPE = local`
2. **存储类型默认**：`[storage.local]` / `[storage.minio]` / `[storage.azureblob]` 节
3. **特定存储命名**：`[storage.packages]` 节（可独立配置包存储）
4. **业务模块配置**：`[packages]` 节中的存储相关配置（覆盖上层）

**配置解析逻辑** (`modules/setting/storage.go:126-149`)：
```go
func getStorage(rootCfg ConfigProvider, name, typ string, sec ConfigSection) (*Storage, error) {
    targetSec, tp, err := getStorageTargetSection(rootCfg, name, typ, sec)
    overrideSec := getStorageOverrideSection(rootCfg, sec, tp, name)
    
    targetType := targetSec.Key("STORAGE_TYPE").String()
    switch targetType {
    case string(LocalStorageType):
        return getStorageForLocal(targetSec, overrideSec, tp, name)
    case string(MinioStorageType):
        return getStorageForMinio(targetSec, overrideSec, tp, name)
    case string(AzureBlobStorageType):
        return getStorageForAzureBlob(targetSec, overrideSec, tp, name)
    }
}
```

### 4.1.4 各存储类型配置参数

**本地存储配置**：
| 参数 | 说明 | 默认值 |
|-----|------|--------|
| `STORAGE_TYPE` | 必须为 `local` | `local` |
| `PATH` | 存储根目录（绝对路径） | `{AppDataPath}/packages/` |
| `TEMPORARY_PATH` | 临时文件目录 | `{PATH}/tmp` |

**MinIO/S3 存储配置**：
| 参数 | 说明 | 默认值 |
|-----|------|--------|
| `STORAGE_TYPE` | 必须为 `minio` | `local` |
| `MINIO_ENDPOINT` | MinIO 服务器地址 | `localhost:9000` |
| `MINIO_ACCESS_KEY_ID` | 访问密钥 ID | - |
| `MINIO_SECRET_ACCESS_KEY` | 秘密访问密钥 | - |
| `MINIO_BUCKET` | Bucket 名称 | `gitea` |
| `MINIO_LOCATION` | 区域 | `us-east-1` |
| `MINIO_BASE_PATH` | 基础路径前缀 | `packages/` |
| `MINIO_USE_SSL` | 是否使用 SSL | `false` |
| `MINIO_INSECURE_SKIP_VERIFY` | 跳过证书验证 | `false` |
| `MINIO_CHECKSUM_ALGORITHM` | 校验算法 | `default` |
| `MINIO_BUCKET_LOOKUP_TYPE` | Bucket 查找方式 | `auto` |
| `SERVE_DIRECT` | 是否直接重定向到存储 | `false` |

**Azure Blob 存储配置**：
| 参数 | 说明 | 默认值 |
|-----|------|--------|
| `STORAGE_TYPE` | 必须为 `azureblob` | `local` |
| `AZURE_BLOB_ENDPOINT` | Azure Blob 端点 | - |
| `AZURE_BLOB_ACCOUNT_NAME` | 账户名称 | - |
| `AZURE_BLOB_ACCOUNT_KEY` | 账户密钥 | - |
| `AZURE_BLOB_CONTAINER` | 容器名称 | `gitea` |
| `AZURE_BLOB_BASE_PATH` | 基础路径前缀 | `packages/` |
| `SERVE_DIRECT` | 是否直接重定向到存储 | `false` |

**MinIO 认证链** (`modules/storage/minio.go:164-193`)：
```
1. 静态凭据 (MINIO_ACCESS_KEY_ID + MINIO_SECRET_ACCESS_KEY)
2. MINIO_ 环境变量
3. AWS_ 环境变量
4. MINIO 共享凭据文件
5. AWS 共享凭据文件
6. EC2 IAM 角色元数据
```

**配置示例** (app.ini)：
```ini
# 全局存储配置
[storage]
STORAGE_TYPE = minio
MINIO_ENDPOINT = minio.example.com:9000
MINIO_ACCESS_KEY_ID = my-access-key
MINIO_SECRET_ACCESS_KEY = my-secret-key
MINIO_BUCKET = gitea
MINIO_USE_SSL = true

# 包存储独立配置（可选，覆盖全局）
[storage.packages]
MINIO_BASE_PATH = gitea-packages/
SERVE_DIRECT = true

# 包服务配置
[packages]
ENABLED = true
LIMIT_TOTAL_OWNER_SIZE = 10GB
```

### 4.1.5 存储接口能力对比

| 能力 | Local | MinIO | Azure Blob |
|-----|-------|-------|------------|
| Open/Read | ✓ | ✓ | ✓ |
| Save/Write | ✓ | ✓ | ✓ |
| Stat | ✓ | ✓ | ✓ |
| Delete | ✓ | ✓ | ✓ |
| IterateObjects | ✓ | ✓ | ✓ |
| ServeDirectURL | ✗ | ✓ (5分钟预签名) | ✓ (5分钟SAS) |
| 临时文件写入 | ✓ (原子重命名) | ✓ (流式上传) | ✓ (流式上传) |
| 自动创建目录 | ✓ (MkdirAll) | ✓ (MakeBucket) | ✓ (CreateContainer) |

> **ServeDirectURL 注意**：生成的 URL 有效期为 5 分钟，允许浏览器直接从对象存储下载文件，绕过 Gitea 服务器，减轻带宽压力。

---

### 4.2 Blob 去重机制

**多哈希唯一约束** (`models/packages/package_blob.go:29-38`)：

```go
type PackageBlob struct {
    ID          int64
    Size        int64
    HashMD5     string  `xorm:"char(32) UNIQUE(md5) INDEX NOT NULL"`
    HashSHA1    string  `xorm:"char(40) UNIQUE(sha1) INDEX NOT NULL"`
    HashSHA256  string  `xorm:"char(64) UNIQUE(sha256) INDEX NOT NULL"`
    HashSHA512  string  `xorm:"char(128) UNIQUE(sha512) INDEX NOT NULL"`
    CreatedUnix timeutil.TimeStamp
}
```

**原子去重操作** (`models/packages/package_blob.go:41-70`)：

```go
func GetOrInsertBlob(ctx context.Context, pb *PackageBlob) (*PackageBlob, bool, error) {
    // 1. SELECT 查询匹配 (size + 所有哈希)
    // 2. 存在则直接返回
    // 3. 不存在则 INSERT
    // 4. 处理竞态条件：INSERT 失败则重试 SELECT
}
```

**去重效果**：
- 相同内容的文件在整个系统中只存储一次
- 不同包、不同版本可以引用同一个 Blob
- 通过 `package_file.blob_id` 建立引用关系

### 4.3 延迟删除策略

**设计注释** (`services/packages/packages.go:493`, `services/packages/packages.go:555`)：

> **PACKAGE-DEFER-STORAGE-DELETE**: Blobs 不立即删除，而是由 `cleanup_packages` 定时任务清理。

**删除流程对比**：

| 操作 | 立即删除 | 延迟删除 |
|-----|---------|---------|
| PackageVersion 删除 | 删除 package_version 记录 | ✓ |
| PackageFile 删除 | 删除 package_file 记录 | ✓ |
| PackageBlob 删除 | ❌ | ✓ (通过清理任务) |
| 存储文件删除 | ❌ | ✓ (通过清理任务) |

### 4.4 垃圾回收机制

**清理任务入口** (`services/packages/cleanup/cleanup.go:27-33`)：

```go
func CleanupTask(ctx context.Context, olderThan time.Duration) error {
    // 1. 执行清理规则 (删除过期版本)
    if err := ExecuteCleanupRules(ctx); err != nil {
        return err
    }
    // 2. 清理过期数据 (未引用的 Blob)
    return CleanupExpiredData(ctx, olderThan)
}
```

**未引用 Blob 查找** (`models/packages/package_blob.go:94-101`)：

```go
func FindExpiredUnreferencedBlobs(ctx context.Context, olderThan time.Duration) ([]*PackageBlob, error) {
    // LEFT JOIN package_file
    // WHERE package_file.id IS NULL (无引用)
    //   AND package_blob.created_unix < 阈值 (避免删除刚上传的)
}
```

**Blob 清理流程** (`services/packages/cleanup/cleanup.go:167-211`)：

```
1. 数据库事务内：
   ├─ 清理 Container 专用数据
   ├─ 删除无版本的 Package
   ├─ 查找过期未引用 Blob
   └─ 删除 package_blob 记录
   
2. 事务提交后：
   └─ 从 ContentStore 物理删除文件
```

### 4.5 清理规则引擎

**规则模型** (`models/packages/package_cleanup_rule.go:25-39`)：

| 字段 | 作用 |
|-----|------|
| `KeepCount` | 保留最近 N 个版本 |
| `KeepPattern` | 版本匹配正则，匹配则保留 |
| `RemoveDays` | 超过 N 天的版本删除 |
| `RemovePattern` | 版本匹配正则，匹配则删除 |
| `MatchFullName` | 匹配 "包名/版本" 还是仅 "版本" |

**规则执行逻辑** (`services/packages/cleanup/cleanup.go:35-85`)：

```
对每个包版本：
    1. 若在 KeepCount 内 → 保留
    2. 若匹配 KeepPattern → 保留
    3. 若创建时间 < RemoveDays 阈值 → 候选删除
    4. 若匹配 RemovePattern → 执行删除
```

---

## 5. 不同包格式对仓库布局的影响

### 5.1 URL 路由架构

**基础路由结构** (`routers/api/packages/api.go:130-547`)：

```
/api/packages/{username}/{type}/...
```

### 5.2 典型包格式的 URL 布局

#### 5.2.1 Generic (通用格式)

**路由** (`routers/api/packages/generic/generic.go:30-210`)：

```
GET  /{packagename}/{packageversion}/{filename}  # 下载
PUT  /{packagename}/{packageversion}/{filename}  # 上传
DELETE  /{packagename}/{packageversion}/{filename}  # 删除文件
DELETE  /{packagename}/{packageversion}  # 删除整个版本
```

**包名规则**：`[-_+.\w]+`，简单扁平结构

#### 5.2.2 Maven

**路由** (`routers/api/packages/maven/maven.go:56-477`)：

```
# Maven 标准层级结构
GET  /{groupIdPath}/{artifactId}/{version}/{filename}
# 例如: /com/example/myapp/1.0.0/myapp-1.0.0.jar

# 元数据文件
GET  /{groupIdPath}/{artifactId}/maven-metadata.xml
GET  /{groupIdPath}/{artifactId}/{version}/maven-metadata.xml
```

**内部包名转换**：
- URL: `/com/example/myapp/1.0.0/...`
- 内部包名: `com.example:myapp` (GroupID:ArtifactID)
- 版本: `1.0.0`

#### 5.2.3 npm

**路由定义** (`routers/api/packages/api.go:403-446`)：

npm 路由系统采用**双轨设计**，同时支持作用域包和非作用域包，每种都有完整独立的路由组。

##### 5.2.3.1 路由组结构

```
/api/packages/{username}/npm
├─ /@{scope}/{id}              # 作用域包路由组
│  ├─ GET  ""                  # 获取元数据
│  ├─ PUT  ""                  # 发布包 (需 Write 权限)
│  ├─ /-/{version}/{filename}  # 版本文件组
│  │  ├─ GET  ""               # 下载指定版本文件
│  │  └─ DELETE /-rev/{revision}  # 删除版本 (需 Write)
│  ├─ GET  /-/{filename}       # 按文件名自动查找版本下载
│  └─ /-rev/{revision}         # 修订组 (需 Write)
│     ├─ DELETE ""             # 删除整个包
│     └─ PUT    ""             # 删除预览 (空操作)
│
├─ /{id}                       # 非作用域包路由组 (与上面对称)
│  ├─ GET  ""                  # 获取元数据
│  ├─ PUT  ""                  # 发布包 (需 Write 权限)
│  ├─ /-/{version}/{filename}  # 版本文件组
│  │  ├─ GET  ""               # 下载指定版本文件
│  │  └─ DELETE /-rev/{revision}  # 删除版本 (需 Write)
│  ├─ GET  /-/{filename}       # 按文件名自动查找版本下载
│  └─ /-rev/{revision}         # 修订组 (需 Write)
│     ├─ DELETE ""             # 删除整个包
│     └─ PUT    ""             # 删除预览 (空操作)
│
├─ /-/package/@{scope}/{id}/dist-tags  # 作用域包 dist-tags
│  ├─ GET  ""                  # 列出所有标签
│  └─ /{tag}                   # 标签操作 (需 Write)
│     ├─ PUT  ""               # 添加标签
│     └─ DELETE ""             # 删除标签
│
├─ /-/package/{id}/dist-tags   # 非作用域包 dist-tags
│  ├─ GET  ""                  # 列出所有标签
│  └─ /{tag}                   # 标签操作 (需 Write)
│     ├─ PUT  ""               # 添加标签
│     └─ DELETE ""             # 删除标签
│
└─ /-/v1/search
   └─ GET  ""                  # 搜索包
```

##### 5.2.3.2 包名解析机制

**核心函数** `packageNameFromParams()` (`routers/api/packages/npm/npm.go:42-51`)：

```go
// packageNameFromParams gets the package name from the url parameters
// Variations: /name/, /@scope/name/, /@scope%2Fname/
func packageNameFromParams(ctx *context.Context) string {
    scope := ctx.PathParam("scope")
    id := ctx.PathParam("id")
    if scope != "" {
        return fmt.Sprintf("@%s/%s", scope, id)
    }
    return id
}
```

**三种包名变体**（内部存储格式统一）：

| URL 形式 | 路由参数 | 内部包名 | 说明 |
|---------|---------|---------|------|
| `/mypackage` | `scope=""`, `id="mypackage"` | `mypackage` | 非作用域包 |
| `/@myorg/mypackage` | `scope="myorg"`, `id="mypackage"` | `@myorg/mypackage` | 作用域包（路径分隔） |
| `/@myorg%2Fmypackage` | 特殊编码 | `@myorg/mypackage` | 作用域包（URL 编码） |

> **注意**：第三种变体 `/@scope%2Fname/` 是为了兼容某些 npm 客户端将 `@scope/name` 作为单个路径段编码的情况，实际路由匹配时由 `/{id}` 组捕获。

##### 5.2.3.3 下载路由详解

npm 提供**两种下载模式**，作用域包和非作用域包均支持：

**模式一：按版本号精确下载**

- **非作用域包**：`GET /{id}/-/{version}/{filename}`
  - 示例：`/mypackage/-/1.0.0/mypackage-1.0.0.tgz`
  - 处理函数：`npm.DownloadPackageFile`

- **作用域包**：`GET /@{scope}/{id}/-/{version}/{filename}`
  - 示例：`/@myorg/mypackage/-/1.0.0/mypackage-1.0.0.tgz`
  - 处理函数：`npm.DownloadPackageFile`

**处理逻辑** (`routers/api/packages/npm/npm.go:82-110`)：
```go
func DownloadPackageFile(ctx *context.Context) {
    packageName := packageNameFromParams(ctx)  // 解析作用域
    packageVersion := ctx.PathParam("version")
    filename := ctx.PathParam("filename")
    
    // 按包名+版本精确查找
    s, u, pf, err := packages_service.OpenFileForDownloadByPackageNameAndVersion(
        ctx,
        &packages_service.PackageInfo{
            Owner:       ctx.Package.Owner,
            PackageType: packages_model.TypeNpm,
            Name:        packageName,
            Version:     packageVersion,
        },
        &packages_service.PackageFileInfo{
            Filename: filename,
        },
        ctx.Req.Method,
    )
    // ...
    helper.ServePackageFile(ctx, s, u, pf)
}
```

**模式二：按文件名自动查找版本**

- **非作用域包**：`GET /{id}/-/{filename}`
  - 示例：`/mypackage/-/mypackage-1.0.0.tgz`
  - 处理函数：`npm.DownloadPackageFileByName`

- **作用域包**：`GET /@{scope}/{id}/-/{filename}`
  - 示例：`/@myorg/mypackage/-/mypackage-1.0.0.tgz`
  - 处理函数：`npm.DownloadPackageFileByName`

**处理逻辑** (`routers/api/packages/npm/npm.go:112-153`)：
```go
func DownloadPackageFileByName(ctx *context.Context) {
    filename := ctx.PathParam("filename")
    
    // 按包名+文件名搜索，自动匹配版本
    pvs, _, err := packages_model.SearchVersions(ctx, &packages_model.PackageSearchOptions{
        OwnerID: ctx.Package.Owner.ID,
        Type:    packages_model.TypeNpm,
        Name: packages_model.SearchValue{
            ExactMatch: true,
            Value:      packageNameFromParams(ctx),
        },
        HasFileWithName: filename,  // 关键：通过文件名反向查找版本
        IsInternal:      optional.Some(false),
    })
    if len(pvs) != 1 {
        apiError(ctx, http.StatusNotFound, nil)
        return
    }
    // 找到唯一版本后提供下载
    s, u, pf, err := packages_service.OpenFileForDownloadByPackageVersion(ctx, pvs[0], ...)
    // ...
}
```

**两种下载模式对比**：

| 特性 | 按版本精确下载 | 按文件名自动查找 |
|-----|--------------|----------------|
| URL 结构 | `/-/{version}/{filename}` | `/-/{filename}` |
| 需要参数 | version + filename | filename |
| 查找方式 | 包名+版本精确匹配 | 包名+文件名搜索 |
| 性能 | 快（索引直接查询） | 稍慢（需要搜索） |
| 多版本冲突 | 无（版本唯一） | 需确保文件名唯一（返回 404 如不唯一） |
| 适用场景 | npm install 标准流程 | 旧版客户端兼容 |

##### 5.2.3.4 dist-tags 路由

**非作用域包**：
```
GET  /-/package/{id}/dist-tags            # 列出所有标签
PUT  /-/package/{id}/dist-tags/{tag}      # 添加标签 (需 Write)
DELETE /-/package/{id}/dist-tags/{tag}     # 删除标签 (需 Write)
```

**作用域包**：
```
GET  /-/package/@{scope}/{id}/dist-tags   # 列出所有标签
PUT  /-/package/@{scope}/{id}/dist-tags/{tag}   # 添加标签 (需 Write)
DELETE /-/package/@{scope}/{id}/dist-tags/{tag}  # 删除标签 (需 Write)
```

**标签存储机制**：通过 `PackageProperty` 存储，属性名为 `npm.TagProperty`
- 键：`npm.TagProperty` = `"npm.tag"`
- 值：标签名（如 `latest`, `beta`, `v1.x`）
- 关联：`PropertyTypeVersion` + `VersionID`

**标签约束** (`routers/api/packages/npm/npm.go:385-392`)：
```go
func setPackageTag(ctx std_ctx.Context, tag string, pv *packages_model.PackageVersion, deleteOnly bool) error {
    if tag == "" {
        return errInvalidTagName
    }
    // 标签名不能是有效的 SemVer 版本号
    _, err := version.NewVersion(tag)
    if err == nil {
        return errInvalidTagName
    }
    // ...
}
```

> **标签约束**：标签名不能是空字符串，也不能是有效的 SemVer 版本号（如 `1.0.0` 不能作为标签名）。

##### 5.2.3.5 完整 URL 示例

**非作用域包 `mypackage`**：
```
# 元数据
GET  /api/packages/user1/npm/mypackage

# 发布
PUT  /api/packages/user1/npm/mypackage

# 下载 (指定版本)
GET  /api/packages/user1/npm/mypackage/-/1.0.0/mypackage-1.0.0.tgz

# 下载 (自动查找版本)
GET  /api/packages/user1/npm/mypackage/-/mypackage-1.0.0.tgz

# dist-tags
GET  /api/packages/user1/npm/-/package/mypackage/dist-tags
PUT  /api/packages/user1/npm/-/package/mypackage/dist-tags/latest
```

**作用域包 `@myorg/mypackage`**：
```
# 元数据
GET  /api/packages/user1/npm/@myorg/mypackage

# 发布
PUT  /api/packages/user1/npm/@myorg/mypackage

# 下载 (指定版本)
GET  /api/packages/user1/npm/@myorg/mypackage/-/1.0.0/mypackage-1.0.0.tgz

# 下载 (自动查找版本)
GET  /api/packages/user1/npm/@myorg/mypackage/-/mypackage-1.0.0.tgz

# dist-tags
GET  /api/packages/user1/npm/-/package/@myorg/mypackage/dist-tags
PUT  /api/packages/user1/npm/-/package/@myorg/mypackage/dist-tags/beta
```

#### 5.2.4 Container (OCI)

**特殊路由** - 挂载在根路径 `/v2/`：

```
# 符合 OCI Distribution Spec
GET  /v2/
GET  /v2/_catalog
GET  /v2/{username}/{image}/tags/list
GET  /v2/{username}/{image}/manifests/{reference}
GET  /v2/{username}/{image}/blobs/{digest}
POST /v2/{username}/{image}/blobs/uploads
```

**Container 认证特殊处理** (`routers/api/packages/api.go:560-571`)：
- 允许 Ghost User（匿名）访问
- 专用的 token 认证端点 `/v2/token`

### 5.3 包格式特性对比

| 格式 | 命名层级 | 元数据 | 版本语义 | 索引文件 | 专用认证 |
|-----|---------|--------|---------|---------|---------|
| Generic | 2 层 (name/version) | 无 | 任意 | 无 | - |
| Maven | 4 层+ (group/artifact/version) | POM XML | 任意 | maven-metadata.xml | - |
| npm | 2-3 层 (@scope/name) | package.json | SemVer | - | - |
| PyPI | 2 层 (name/version) | setup.py | 任意 | simple 索引 | - |
| Container | N 层 (path/image) | Manifest JSON | tag/digest | - | JWT |
| Debian | 3 层 (dist/component/arch) | DEB 控制信息 | 任意 | Packages/Sources | - |
| RPM | 3 层 (group/name/version) | RPM 头 | 任意 | repodata | - |
| Conan | 4 层 (name/version/user/channel) | conanfile.py | 任意 | 搜索 API | - |

---

## 6. 配额与速率约束

### 6.1 配额配置

**配置结构** (`modules/setting/packages.go:14-94`)：

```go
var Packages = struct {
    // 总配额
    LimitTotalOwnerCount  int64  // 所有者包版本总数
    LimitTotalOwnerSize   int64  // 所有者总存储大小
    
    // 单包类型大小限制
    LimitSizeAlpine       int64
    LimitSizeArch         int64
    LimitSizeCargo        int64
    LimitSizeContainer    int64
    // ... 20+ 种类型
}{
    Enabled:              true,
    LimitTotalOwnerCount: -1,  // -1 表示无限制
}
```

**配置示例** (app.ini)：
```ini
[packages]
LIMIT_TOTAL_OWNER_COUNT = 1000
LIMIT_TOTAL_OWNER_SIZE = 10GB
LIMIT_SIZE_CONTAINER = 10GB
LIMIT_SIZE_NPM = 500MB
```

### 6.2 配额检查时机

**数量配额检查** (`services/packages/packages.go:335-356`)：

```go
func CheckCountQuotaExceeded(ctx context.Context, doer, owner *user_model.User) error {
    // 跳过管理员
    if doer.IsAdmin { return nil }
    
    // 检查总版本数
    if setting.Packages.LimitTotalOwnerCount > -1 {
        totalCount, _ := packages_model.CountVersions(...)
        if totalCount > setting.Packages.LimitTotalOwnerCount {
            return ErrQuotaTotalCount
        }
    }
    return nil
}

// 检查时机：创建新版本时 (createPackageAndVersion)
```

**大小配额检查** (`services/packages/packages.go:360-432`)：

```go
func CheckSizeQuotaExceeded(ctx context.Context, doer, owner *user_model.User, packageType Type, uploadSize int64) error {
    // 1. 检查单类型大小限制
    if typeSpecificSize > -1 && typeSpecificSize < uploadSize {
        return ErrQuotaTypeSize
    }
    
    // 2. 检查所有者总大小限制
    if setting.Packages.LimitTotalOwnerSize > -1 {
        totalSize, _ := packages_model.CalculateFileSize(...)
        if totalSize + uploadSize > setting.Packages.LimitTotalOwnerSize {
            return ErrQuotaTotalSize
        }
    }
    return nil
}

// 检查时机：添加文件时 (addFileToPackageVersion)
```

### 6.3 配额错误处理

**HTTP 响应映射** (`routers/api/packages/generic/generic.go:129-133`)：

| 错误类型 | HTTP 状态码 |
|---------|------------|
| `ErrQuotaTotalCount` | 403 Forbidden |
| `ErrQuotaTotalSize` | 403 Forbidden |
| `ErrQuotaTypeSize` | 403 Forbidden |

### 6.4 速率约束说明

**注意**：当前代码中未发现专门针对包注册服务的速率限制中间件。速率限制依赖于：
1. 全局 API 速率限制（如 Nginx 反向代理层）
2. 全局认证失败速率限制
3. 具体包管理器客户端的重试机制

---

## 7. 核心协作流程图

### 7.1 包上传完整流程

```
客户端请求
    ↓
[HTTP 路由匹配]
    ↓
[认证中间件] verifyAuth()
    ├─ Basic Auth 验证
    ├─ OAuth2 Token 验证
    └─ JWT Token 验证 (Container/Conan)
    ↓
[包上下文] PackageContexter()
    ↓
[用户上下文] UserAssignmentWeb()
    └─ 从 URL 解析 {username}
    ↓
[包权限计算] PackageAssignment()
    ├─ 确定 Owner (User/Organization)
    └─ 计算 AccessMode (None/Read/Write/Owner)
    ↓
[权限检查] reqPackageAccess(Write)
    ├─ 检查 Token Scope (read:package / write:package)
    └─ 检查用户实际权限 >= Write
    ↓
[包格式处理]
    ├─ 解析包特定元数据 (pom.xml/package.json 等)
    ├─ 计算文件哈希 (MD5/SHA1/SHA256/SHA512)
    ↓
[事务开始]
    ├─ GetOrInsertBlob() → 去重检查
    │   ├─ 已存在 → 复用 Blob
    │   └─ 不存在 → ContentStore.Save()
    ├─ TryInsertPackage()
    ├─ GetOrInsertVersion() → 检查数量配额
    ├─ TryInsertFile() → 检查大小配额
    └─ [事务提交]
    ↓
[后处理]
    ├─ 发送通知 (PackageCreate 事件)
    └─ 更新索引 (Maven/Debian/RPM 等)
    ↓
HTTP 201 Created
```

### 7.2 Blob 引用关系图

```
package (owner_id + type + name)
    │
    ├─ 1:N → package_version (package_id + version)
    │         │
    │         └─ 1:N → package_file (version_id + name + composite_key)
    │                   │
    │                   └─ N:1 → package_blob (blob_id)
    │                             │
    └─────────────────────────────└─ ContentStore: aa/bb/aabbxx... (SHA256)


```

### 7.3 垃圾回收流程

```
定时任务触发 (cleanup_packages cron)
    ↓
ExecuteCleanupRules()
    └─ 对每个启用的规则:
        ├─ 获取该类型所有包
        └─ 对每个包的版本:
            ├─ 应用 KeepCount
            ├─ 应用 KeepPattern
            ├─ 应用 RemoveDays
            ├─ 应用 RemovePattern
            ├─ DeletePackageVersionAndReferences()
            └─ 更新索引文件 (Debian/RPM/Arch 等)
    ↓
CleanupExpiredData(olderThan = 24h)
    ├─ [事务]
    │   ├─ Container.Cleanup()
    │   ├─ 删除无版本的 Package
    │   ├─ FindExpiredUnreferencedBlobs()
    │   │   └─ LEFT JOIN package_file IS NULL
    │   └─ 删除 package_blob 记录
    └─ [事务外]
        └─ ContentStore.Delete() → 物理删除文件
```

---

## 8. 关键设计总结

### 8.1 设计优势

1. **多层去重**：基于 4 种哈希算法保证内容唯一性
2. **延迟删除**：Blob 延迟删除避免误删风险，便于事务回滚
3. **租户隔离**：基于 OwnerID 的命名空间隔离，支持多租户
4. **多格式统一**：统一的数据模型 + 格式特定路由适配器
5. **灵活配额**：支持总数、总大小、单类型大小三级配额
6. **可扩展存储**：抽象 ObjectStorage 接口，支持本地/MinIO/S3

### 8.2 核心数据模型关系

```
┌─────────────────────────────────────────────────────────┐
│                   package_blob (去重层)                  │
├─────────────────────────────────────────────────────────┤
│ id, size, hash_md5, hash_sha1, hash_sha256, hash_sha512 │
└─────────────┬───────────────────────────────────────────┘
              │ 1
              │
              │ N
┌─────────────▼───────────────────────────────────────────┐
│                     package_file                         │
├─────────────────────────────────────────────────────────┤
│ id, version_id, blob_id, name, composite_key, is_lead   │
└─────────────┬───────────────────────────────────────────┘
              │ N
              │
              │ 1
┌─────────────▼───────────────────────────────────────────┐
│                  package_version                         │
├─────────────────────────────────────────────────────────┤
│ id, package_id, version, metadata_json, download_count  │
└─────────────┬───────────────────────────────────────────┘
              │ N
              │
              │ 1
┌─────────────▼───────────────────────────────────────────┐
│                      package                             │
├─────────────────────────────────────────────────────────┤
│ id, owner_id, repo_id, type, name                       │
└─────────────┬───────────────────────────────────────────┘
              │ N
              │
              │ 1
┌─────────────▼──────────┐
│         user           │  (Owner - User/Organization)
└────────────────────────┘
```

### 8.3 代码位置索引

| 功能 | 主要文件 |
|-----|---------|
| **认证授权** | `services/packages/auth.go` <br> `routers/api/packages/auth.go` <br> `routers/api/packages/api.go` |
| **命名空间隔离** | `services/context/package.go` <br> `models/packages/package.go` |
| **Blob 存储/去重** | `models/packages/package_blob.go` <br> `modules/packages/content_store.go` |
| **垃圾回收** | `services/packages/cleanup/cleanup.go` <br> `models/packages/package_cleanup_rule.go` |
| **配额管理** | `services/packages/packages.go` <br> `modules/setting/packages.go` |
| **通用服务** | `services/packages/packages.go` |
| **数据模型** | `models/packages/*.go` |
| **路由处理** | `routers/api/packages/*/*.go` |

---

## 9. 附录：支持的包格式列表

| 类型常量 | 显示名称 | 路径 |
|---------|---------|------|
| `TypeAlpine` | Alpine | `/alpine` |
| `TypeArch` | Arch | `/arch` |
| `TypeCargo` | Cargo | `/cargo` |
| `TypeChef` | Chef | `/chef` |
| `TypeComposer` | Composer | `/composer` |
| `TypeConan` | Conan | `/conan` |
| `TypeConda` | Conda | `/conda` |
| `TypeContainer` | Container | `/v2` (OCI 规范) |
| `TypeCran` | CRAN | `/cran` |
| `TypeDebian` | Debian | `/debian` |
| `TypeGeneric` | Generic | `/generic` |
| `TypeGo` | Go | `/go` |
| `TypeHelm` | Helm | `/helm` |
| `TypeMaven` | Maven | `/maven` |
| `TypeNpm` | npm | `/npm` |
| `TypeNuGet` | NuGet | `/nuget` |
| `TypePub` | Pub | `/pub` |
| `TypePyPI` | PyPI | `/pypi` |
| `TypeRpm` | RPM | `/rpm` |
| `TypeRubyGems` | RubyGems | `/rubygems` |
| `TypeSwift` | Swift | `/swift` |
| `TypeTerraformState` | Terraform State | `/terraform/state` |
| `TypeVagrant` | Vagrant | `/vagrant` |

---

**报告生成时间**：2026-05-31
**代码版本**：Gitea 174-gitea
**分析范围**：凭据校验、命名空间隔离、二进制对象存储、去重与垃圾回收、仓库布局、配额约束
