# Gitea SSH 公钥与 GPG 签名校验代码分析

## 一、SSH 公钥写入 authorized_keys 实现

### 1.1 核心数据模型

**文件**: `models/asymkey/ssh_key.go`

`PublicKey` 结构体定义了 SSH 公钥的数据模型：

```go
type PublicKey struct {
    ID            int64
    OwnerID       int64           // 所有者用户ID
    Name          string          // 密钥名称
    Fingerprint   string          // 密钥指纹
    Content       string          // 密钥内容
    Mode          perm.AccessMode // 访问模式
    Type          KeyType         // 密钥类型（用户/部署/Principal）
    LoginSourceID int64           // 登录源ID
    Verified      bool            // 是否已验证
    // ...
}
```

密钥类型分为三种：
- `KeyTypeUser` - 用户密钥
- `KeyTypeDeploy` - 部署密钥
- `KeyTypePrincipal` - 授权主体密钥

### 1.2 authorized_keys 文件格式

**文件**: `models/asymkey/ssh_key_authorized_keys.go`

每个公钥在 `authorized_keys` 文件中的格式由 `writeAuthorizedStringForKey` 函数生成：

```go
const AuthorizedStringCommentPrefix = `# gitea public key`
const tpl = AuthorizedStringCommentPrefix + "\n" + 
    `command=%s,no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty,no-user-rc,restrict %s` + "\n"
```

每行密钥包含以下安全限制：
- `command=` - 强制执行 Gitea 的 SSH 命令处理程序
- `no-port-forwarding` - 禁止端口转发
- `no-X11-forwarding` - 禁止 X11 转发
- `no-agent-forwarding` - 禁止代理转发
- `no-pty` - 禁止分配伪终端
- `no-user-rc` - 禁止执行用户 rc 文件
- `restrict` - 启用所有限制

### 1.3 公钥添加流程

**入口函数**: `models/asymkey/ssh_key.go:AddPublicKey`

```
AddPublicKey
├── 计算指纹: CalcFingerprint(content)
├── 事务内操作
│   ├── 检查指纹是否已存在: checkKeyFingerprint
│   ├── 检查同一用户下密钥名称是否重复
│   ├── 创建 PublicKey 对象
│   └── 调用 addKey
└── addKey
    ├── 数据库插入: db.Insert(ctx, key)
    └── 写入 authorized_keys: appendAuthorizedKeysToFile(key)
```

### 1.4 追加写入 authorized_keys

**函数**: `models/asymkey/ssh_key_authorized_keys.go:appendAuthorizedKeysToFile`

```go
func appendAuthorizedKeysToFile(keys ...*PublicKey) error {
    // 跳过条件：内置 SSH 服务器 或 不创建 authorized_keys 文件
    if setting.SSH.StartBuiltinServer || !setting.SSH.CreateAuthorizedKeysFile {
        return nil
    }
    
    sshOpLocker.Lock()  // 互斥锁保护
    defer sshOpLocker.Unlock()
    
    // 确保 .ssh 目录存在 (权限 0700)
    os.MkdirAll(setting.SSH.RootPath, 0o700)
    
    // 以追加模式打开 authorized_keys (权限 0600)
    f, _ := os.OpenFile(fPath, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0o600)
    
    // 非 Windows 系统检查并修正文件权限
    if !setting.IsWindows {
        fi.Mode().Perm() > 0o600 时执行 Chmod(0o600)
    }
    
    // 写入每个密钥
    for _, key := range keys {
        WriteAuthorizedStringForValidKey(key, f)
    }
}
```

### 1.5 公钥删除流程

**入口函数**: `services/asymkey/ssh_key.go:DeletePublicKey`

```
DeletePublicKey
├── 权限检查：管理员或密钥所有者
├── 数据库删除: db.DeleteByID[PublicKey]
└── 如果是 Principal 密钥
    ├── RewriteAllPrincipalKeys
    └── 否则 RewriteAllPublicKeys
```

**注意**: 删除操作不做增量修改，而是全量重写 `authorized_keys` 文件。

### 1.6 全量重写 authorized_keys

**入口函数**: `services/asymkey/ssh_key_authorized_keys.go:RewriteAllPublicKeys`

```
RewriteAllPublicKeys
├── 跳过条件：内置 SSH 服务器 或 不创建 authorized_keys 文件
└── WithSSHOpLocker 保护下执行 rewriteAllPublicKeys
    ├── 创建 .ssh 目录（权限 0700）
    ├── 创建临时文件 authorized_keys.tmp
    ├── 可选：备份原文件为 authorized_keys_{timestamp}.gitea_bak
    ├── 从数据库重新生成所有密钥: RegeneratePublicKeys
    ├── 关闭临时文件
    └── 原子重命名: tmpPath -> fPath
```

**RegeneratePublicKeys** 函数的特殊处理：
```go
func RegeneratePublicKeys(ctx context.Context, t io.Writer) error {
    // 1. 从数据库写入所有非 Principal 类型的密钥
    db.GetEngine(ctx).Where("type != ?", KeyTypePrincipal).Iterate(...)
    
    // 2. 保留原 authorized_keys 中非 Gitea 管理的密钥
    // 扫描原文件，跳过以 "# gitea public key" 开头的行及其下一行
    // 将非 Gitea 管理的密钥行复制到新文件
}
```

### 1.7 外部认证源密钥同步

**函数**: `models/asymkey/ssh_key.go:SynchronizePublicKeys`

用于与外部认证源（如 LDAP）同步 SSH 公钥：
1. 获取当前用户在 DB 中属于该认证源的所有密钥
2. 处理外部提供的密钥列表（去重、提取类型和内容）
3. 比较两组密钥，添加新增的密钥
4. 删除不再存在于外部源的密钥
5. 返回是否需要更新 authorized_keys

## 二、GPG 签名校验实现

### 2.1 GPG 密钥数据模型

**文件**: `models/asymkey/gpg_key.go`

```go
type GPGKey struct {
    ID                int64
    OwnerID           int64           // 所有者用户ID
    KeyID             string          // 密钥ID (16字符)
    PrimaryKeyID      string          // 主密钥ID（子密钥使用）
    Content           string          // Base64 编码的公钥内容
    SubsKey           []*GPGKey       // 子密钥列表（xorm:"-"）
    Emails            []*user_model.EmailAddress  // 关联的邮箱地址
    Verified          bool            // 是否已验证
    CanSign           bool            // 可签名
    CanEncryptComms   bool            // 可加密通信
    CanEncryptStorage bool            // 可加密存储
    CanCertify        bool            // 可认证
}
```

### 2.2 签名验证数据结构

**文件**: `models/asymkey/gpg_key_commit_verification.go`

```go
type CommitVerification struct {
    Verified       bool              // 是否已验证
    Warning        bool              // 是否有警告
    Reason         string            // 验证结果原因/错误信息
    SigningUser    *user_model.User  // 签名用户
    CommittingUser *user_model.User  // 提交用户
    SigningEmail   string            // 签名邮箱
    SigningKey     *GPGKey           // 签名GPG密钥
    SigningSSHKey  *PublicKey        // 签名SSH密钥
    TrustStatus    string            // 信任状态
}
```

**信任状态** (`TrustStatus`):
- `trusted` - 可信
- `untrusted` - 不可信
- `unmatched` - 签名者与提交者不匹配
- `unmatched` - 不匹配

### 2.3 签名验证主流程

**入口函数**: `services/asymkey/commit.go:ParseCommitWithSignature`

```
ParseCommitWithSignature
├── 通过邮箱获取提交者用户: GetUserByEmail
└── ParseCommitWithSignatureCommitter
    ├── 无签名 -> 返回未签名验证结果
    ├── SSH 签名 -> parseCommitWithSSHSignature
    └── GPG 签名 -> parseCommitWithGPGSignature
```

### 2.4 GPG 签名验证流程

**函数**: `services/asymkey/commit.go:parseCommitWithGPGSignature`

```
parseCommitWithGPGSignature
├── 提取签名数据包: ExtractSignature
│   └── 失败 -> 返回 "gpg.error.extract_sign"
├── 从签名中获取 KeyID: TryGetKeyIDFromSignature
├── 第一步：按 KeyID 查找密钥验证: HashAndVerifyForKeyID
│   ├── 成功 -> 返回验证结果
│   └── 失败但密钥存在 -> 标记 BadSignature
├── 第二步：按提交者用户的所有密钥验证
│   ├── 获取用户所有 GPG 密钥
│   ├── 加载子密钥
│   ├── 检查密钥邮箱与提交者邮箱匹配
│   └── 逐个验证: HashAndVerifyWithSubKeysCommitVerification
├── 第三步：尝试实例配置的默认签名密钥
│   ├── setting.Repository.Signing.SigningKey
│   └── 系统全局 GPG 设置: GetDefaultPublicGPGKey
└── 最终返回验证结果
    ├── 匹配成功 -> Verified=true
    └── 匹配失败 -> Verified=false, Reason=BadSignature/NoKeyFound
```

### 2.5 核心验证函数

**函数**: `models/asymkey/gpg_key_commit_verification.go:hashAndVerifyWithSubKeys`

```go
func hashAndVerifyWithSubKeys(sig *packet.Signature, payload string, k *GPGKey) (*GPGKey, error) {
    // 1. 先用主密钥验证
    verified, err := hashAndVerify(sig, payload, k)
    
    // 2. 主密钥失败时，逐个尝试子密钥
    for _, sk := range k.SubsKey {
        verified, err := hashAndVerify(sig, payload, sk)
        if err != nil || verified != nil {
            return verified, err
        }
    }
    return nil, nil
}

func hashAndVerify(sig *packet.Signature, payload string, k *GPGKey) (*GPGKey, error) {
    // 生成提交内容的哈希
    hash, _ := populateHash(sig.Hash, []byte(payload))
    // 验证签名
    err = verifySign(sig, hash, k)
}

func verifySign(s *packet.Signature, h hash.Hash, k *GPGKey) error {
    // 检查密钥是否可签名
    if !k.CanSign { return errors.New("key can not sign") }
    // Base64 解码公钥内容
    pkey, _ := base64DecPubKey(k.Content)
    // 执行密码学验证
    return pkey.VerifySignature(h, s)
}
```

### 2.6 SSH 签名验证流程

**函数**: `services/asymkey/commit.go:parseCommitWithSSHSignature`

```
parseCommitWithSSHSignature
├── 第一步：按提交者用户的 SSH 公钥验证
│   ├── 获取用户所有非 Principal 类型的 SSH 公钥
│   └── 逐个已验证密钥验证: verifySSHCommitVerification
├── 第二步：尝试预配置的可信 SSH 密钥
│   └── setting.Repository.Signing.TrustedSSHKeys
├── 第三步：尝试实例配置的 SSH 签名密钥
│   └── setting.Repository.Signing.SigningKey (SSH 格式)
└── 验证成功返回 Verified=true，否则返回 NoKeyFound
```

**SSH 签名验证核心**:
```go
func verifySSHCommitVerification(sig, payload string, k *asymkey_model.PublicKey, ...) {
    // 使用 sshsig 库验证 SSH 签名
    err := sshsig.Verify(strings.NewReader(payload), []byte(sig), []byte(k.Content), "git")
}
```

### 2.7 信任状态计算

**函数**: `models/asymkey/gpg_key_commit_verification.go:CalculateTrustStatus`

根据仓库的信任模型 (`TrustModel`) 计算签名的信任状态：

1. **CommitterTrustModel** 模式：
   - 只比较签名用户与提交用户是否匹配
   - 匹配则 `trusted`，否则 `unmatched`

2. **CollaboratorCommitterTrustModel** 等更严格模式：
   - 检查签名用户是否为仓库协作者/成员/所有者
   - 非成员标记为 `untrusted`
   - 签名者与提交者不匹配标记为 `unmatched`

## 三、提交者标识展示实现

### 3.1 Git 提交数据结构

**文件**: `modules/git/commit.go`

```go
type Commit struct {
    ID        ObjectID
    Author    *Signature  // 作者信息（永不 nil）
    Committer *Signature  // 提交者信息（永不 nil）
    Signature *CommitSignature  // GPG/SSH 签名
    // ...
}

type Signature struct {
    Name  string  // 姓名
    Email string  // 邮箱
    When  time.Time  // 时间
}

type CommitSignature struct {
    Signature string  // 签名内容
    Payload   string  // 签名的原始载荷
}
```

### 3.2 签名解析

**文件**: `modules/git/signature.go`

`parseSignatureFromCommitLine` 函数解析 Git 提交行格式：
```
full name <user@example.com> 1378823654 +0200
```

解析为 `Signature` 结构体：
- 姓名：`<` 之前的部分
- 邮箱：`<` 和 `>` 之间的部分
- 时间：Unix 时间戳 + 时区

### 3.3 API 层转换

**文件**: `services/convert/git_commit.go` 和 `services/convert/convert.go`

#### 3.3.1 提交用户转换

```go
func ToCommitUser(sig *git.Signature) *api.CommitUser {
    return &api.CommitUser{
        Identity: api.Identity{
            Name:  sig.Name,
            Email: sig.Email,
        },
        Date: sig.When.UTC().Format(time.RFC3339),
    }
}
```

#### 3.3.2 签名验证信息转换

```go
func ToVerification(ctx context.Context, c *git.Commit) *api.PayloadCommitVerification {
    // 调用签名验证服务
    verif := asymkey_service.ParseCommitWithSignature(ctx, c)
    
    return &api.PayloadCommitVerification{
        Verified:  verif.Verified,
        Reason:    verif.Reason,
        Signature: c.Signature.Signature,  // 原始签名
        Payload:   c.Signature.Payload,    // 原始载荷
        Signer:    &api.PayloadUser{...},  // 签名用户信息
    }
}
```

#### 3.3.3 完整提交转换

```go
func ToCommit(ctx context.Context, repo *repo_model.Repository, ...) (*api.Commit, error) {
    // 1. 通过邮箱查找 Author 和 Committer 对应的系统用户
    author, _ := user_model.GetUserByEmail(ctx, commit.Author.Email)
    committer, _ := user_model.GetUserByEmail(ctx, commit.Committer.Email)
    
    // 2. 转换为 API 用户对象
    apiAuthor = ToUser(ctx, author, nil)
    apiCommitter = ToUser(ctx, committer, nil)
    
    // 3. 构造提交信息
    res := &api.Commit{
        Author:    apiAuthor,     // 系统用户（可能为 nil）
        Committer: apiCommitter,  // 系统用户（可能为 nil）
        RepoCommit: &api.RepoCommit{
            Author:    ToCommitUser(commit.Author),     // Git 记录的作者
            Committer: ToCommitUser(commit.Committer),  // Git 记录的提交者
            Verification: ToVerification(ctx, commit),  // 签名验证信息
        },
    }
}
```

**关键点**:
- `Author`/`Committer` 字段：通过邮箱匹配到的 Gitea 系统用户
- `RepoCommit.Author`/`Committer` 字段：Git 提交记录中的原始信息
- 两者可能不匹配（Git 提交邮箱未在 Gitea 注册时）

### 3.4 前端展示模板

**文件**: `templates/repo/commit_sign_badge.tmpl`

#### 3.4.1 签名徽章展示逻辑

```
CommitSignVerification
├── Verified = true
│   ├── TrustStatus = "trusted"
│   │   └── class: sign-trusted, 显示 gitea-lock 图标
│   ├── TrustStatus = "untrusted"
│   │   └── class: sign-untrusted, 提示 "signed_by_untrusted_user"
│   └── TrustStatus = "unmatched"（其他）
│       └── class: sign-unmatched, 提示 "signed_by_untrusted_user_unmatched"
├── Verified = false
│   ├── Warning = true
│   │   └── class: sign-warning
│   └── Warning = false
│       └── class: 空（未签名）
└── 展示签名密钥信息
    ├── SSH 密钥 -> 显示指纹
    └── GPG 密钥 -> 显示 KeyID（PaddedKeyID）
```

#### 3.4.2 关键模板代码

```html
{{- if $verification.Verified -}}
    {{- if eq $verification.TrustStatus "trusted" -}}
        {{- $extraClass = print $extraClass " sign-trusted" -}}
    {{- else if eq $verification.TrustStatus "untrusted" -}}
        {{- $extraClass = print $extraClass " sign-untrusted" -}}
    {{- else -}}
        {{- $extraClass = print $extraClass " sign-unmatched" -}}
    {{- end -}}
{{- else -}}
    {{- if $verification.Warning -}}
        {{- $extraClass = print $extraClass " sign-warning" -}}
    {{- end -}}
{{- end -}}

<!-- 签名用户头像 -->
{{- if and $signingUser $signingUser.ID -}}
    {{ctx.AvatarUtils.Avatar $signingUser 16}}
{{- else -}}
    {{ctx.AvatarUtils.AvatarByEmail $signingEmail "" 16}}
{{- end -}}
```

#### 3.4.3 图标含义

| 图标 | 类名 | 含义 |
|------|------|------|
| `gitea-lock` | `sign-trusted` | 已签名且可信 |
| `gitea-lock-cog` | `sign-trusted` | 实例密钥签名 |
| `gitea-unlock` | `sign-warning` 或 无 | 未签名或签名有问题 |

## 四、代码调用关系总图

```
用户添加 SSH 公钥
└── models/asymkey/ssh_key.go:AddPublicKey
    ├── 数据库插入
    └── models/asymkey/ssh_key_authorized_keys.go:appendAuthorizedKeysToFile
        └── writeAuthorizedStringForKey

用户删除 SSH 公钥
└── services/asymkey/ssh_key.go:DeletePublicKey
    ├── 数据库删除
    └── services/asymkey/ssh_key_authorized_keys.go:RewriteAllPublicKeys
        └── models/asymkey/ssh_key_authorized_keys.go:RegeneratePublicKeys

Git 提交展示
├── modules/git/commit.go:Commit (数据结构)
├── services/convert/git_commit.go:ToCommit
│   ├── 查找 Author/Committer 对应的系统用户
│   └── services/convert/convert.go:ToVerification
│       └── services/asymkey/commit.go:ParseCommitWithSignature
│           ├── GPG: parseCommitWithGPGSignature
│           │   ├── HashAndVerifyForKeyID
│           │   └── models/asymkey/gpg_key_commit_verification.go:hashAndVerifyWithSubKeys
│           │       └── verifySign (密码学验证)
│           └── SSH: parseCommitWithSSHSignature
│               └── verifySSHCommitVerification
│                   └── sshsig.Verify (密码学验证)
└── templates/repo/commit_sign_badge.tmpl (前端展示)
```
