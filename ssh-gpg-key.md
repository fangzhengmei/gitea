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

## 四、签名徽章展示的数据来源及调用路径

### 4.1 数据流向总图

```
HTTP 请求 (GET /repo/commits 或 /repo/commit/{sha})
│
├── 后端控制器层
│   ├── 提交列表: routers/web/repo/commit.go:Commits
│   │   └── processGitCommits
│   │       └── services/git/commit.go:ConvertFromGitCommit
│   │           ├── 步骤1: ValidateCommitsWithEmails (邮箱 -> 用户)
│   │           ├── 步骤2: ParseCommitsWithSignature (签名验证)
│   │           └── 步骤3: ParseCommitsWithStatus (CI 状态)
│   │
│   └── 单提交详情: routers/web/repo/commit.go:Diff
│       ├── asymkey_service.ParseCommitWithSignature (签名验证)
│       ├── user_model.ValidateCommitWithEmail (作者邮箱验证)
│       └── CalculateTrustStatus (信任状态计算)
│
├── 模板层
│   ├── 列表页: templates/repo/commits_list.tmpl
│   │   └── range .Commits 循环
│   │       └── 调用: {{template "repo/commit_sign_badge" dict ... .Verification}}
│   │
│   └── 详情页: templates/repo/commit_page.tmpl
│       ├── 作者信息区: {{.Author}} (ValidateCommitWithEmail 结果)
│       ├── 提交者信息区: {{if .Verification.CommittingUser}}
│       └── 签名徽章: {{template "repo/commit_sign_badge" .Verification}}
│
└── 徽章组件: templates/repo/commit_sign_badge.tmpl
    ├── 输入: CommitSignVerification (CommitVerification)
    └── 输出: 签名徽章 HTML + 头像 + Tooltip 提示
```

### 4.2 提交列表完整调用链

**入口**: `routers/web/repo/commit.go:Commits` (行 60)

```
Commits(ctx)
├── 获取提交: ctx.Repo.Commit.CommitsByRange(...)
├── 处理提交: processGitCommits(ctx, commits)
│   └── services/git/commit.go:ConvertFromGitCommit
│       ├── 第一步: 邮箱验证
│       │   └── user_model.ValidateCommitsWithEmails(ctx, commits)
│       │       ├── 收集所有 Author.Email 到 emailSet
│       │       ├── user_model.GetUsersByEmails(ctx, emails)
│       │       └── 为每个 commit 匹配 User -> []*UserCommit
│       │
│       ├── 第二步: 签名验证
│       │   └── ParseCommitsWithSignature(ctx, repo, validatedCommits, trustModel)
│       │       ├── 收集所有 Committer.Email
│       │       ├── 批量获取用户: GetUsersByEmails
│       │       └── 逐个 commit 验证: ParseCommitWithSignatureCommitter
│       │           ├── 注意: 这里使用 Committer.Email 而非 Author.Email
│       │
│       └── 第三步: 信任状态计算
│           └── CalculateTrustStatus(verification, trustModel, isOwnerMemberCollaborator, &keyMap)
│               ├── 检查 SigningUser 是否为仓库成员
│               └── 设置 TrustStatus: trusted/untrusted/unmatched
│
├── 获取标签映射: FindTagsByCommitIDs
└── 渲染模板: ctx.HTML(http.StatusOK, tplCommits)
```

**关键发现**:
- `ValidateCommitsWithEmails` 使用 **Author** 邮箱匹配用户
- `ParseCommitsWithSignature` 使用 **Committer** 邮箱匹配用户
- 代码中存在 FIXME 注释：`FIXME: why ValidateCommitsWithEmails uses "Author", but ParseCommitsWithSignature uses "Committer"?`

### 4.3 单提交详情调用链

**入口**: `routers/web/repo/commit.go:Diff` (行 263)

```
Diff(ctx)
├── 获取提交: gitRepo.GetCommit(commitID)
├── 获取 diff: GetDiffForRender
├── 获取 CI 状态: GetLatestCommitStatus
├── 签名验证: asymkey_service.ParseCommitWithSignature(ctx, commit)  // 行 382
├── 作者验证: user_model.ValidateCommitWithEmail(ctx, commit)  // 行 384
│   └── GetUserByEmail(ctx, commit.Author.Email)  // 使用 Author 邮箱
├── 信任状态: CalculateTrustStatus(verification, trustModel, ...)  // 行 388-393
├── 渲染模板: ctx.HTML(http.StatusOK, tplCommitPage)
└── 模板中展示:
    ├── 作者: {{if .Author}} -> 展示系统用户头像和链接
    ├── 提交者: {{if ne Committer.Name Author.Name}} 单独展示
    │   └── 使用 {{.Verification.CommittingUser}}
    └── 签名徽章: {{template "repo/commit_sign_badge" .Verification}}
```

### 4.4 模板数据绑定

**提交列表模板** (`templates/repo/commits_list.tmpl` 行 36):
```html
{{template "repo/commit_sign_badge" dict 
    "Commit" . 
    "CommitBaseLink" $commitBaseLink 
    "CommitSignVerification" .Verification
}}
```

**单提交详情模板** (`templates/repo/commit_page.tmpl` 行 158-160):
```html
{{if .Verification}}
    {{template "repo/commit_sign_badge" dict "CommitSignVerification" .Verification}}
{{end}}
```

**徽章模板** 接收三个参数：
1. `Commit` - Git 提交对象（可选，用于显示提交 ID）
2. `CommitBaseLink` - 链接基础路径（可选）
3. `CommitSignVerification` - **核心验证数据** (CommitVerification 结构体)

## 五、Noreply 邮箱映射及作者/提交者差异影响

### 5.1 Noreply 邮箱格式

**生成函数**: `models/user/user.go:GetPlaceholderEmail` (行 218)

```go
func (u *User) GetPlaceholderEmail() string {
    return fmt.Sprintf("%d+%s@%s", u.ID, u.LowerName, setting.Service.NoReplyAddress)
}
```

**格式**: `{user_id}+{lower_username}@{noreply_domain}`

**示例**:
- 用户 ID: 123
- 用户名: johndoe
- NoReplyAddress: noreply.gitea.io
- 结果: `123+johndoe@noreply.gitea.io`

### 5.2 Noreply 邮箱解析流程

**解析函数**: `models/user/user.go:parseLocalPartToNameID` (行 1293)

```go
func parseLocalPartToNameID(localPart string) (string, int64) {
    idstr, name, hasPlus := strings.Cut(localPart, "+")
    if hasPlus {
        id, _ = strconv.ParseInt(idstr, 10, 64)
    } else {
        name = idstr
    }
    return name, id
}
```

**匹配优先级**:
1. 优先使用 `ID` 查找（如果格式是 `id+name`）
2. 降级使用 `name` 查找（如果格式只有 `name`）

### 5.3 GetUserByEmail 完整查找逻辑

**文件**: `models/user/user.go:GetUserByEmail` (行 1305)

```
GetUserByEmail(ctx, email)
├── 邮箱转小写
├── 第一步: 查找激活的邮箱地址
│   └── EmailAddress{LowerEmail: email, IsActivated: true}
│       └── 找到 -> 返回对应用户
├── 第二步: 检查是否为 noreply 格式
│   ├── 匹配后缀: @setting.Service.NoReplyAddress
│   ├── 解析 local-part: name, id = parseLocalPartToNameID(localPart)
│   ├── 如果有 ID -> GetUserByID(ctx, id)
│   └── 否则 -> GetIndividualUserByName(ctx, name)
└── 都没找到 -> 返回 ErrUserNotExist
```

**批量优化版本**: `GetUsersByEmails` (行 1203)
- 批量收集所有需要检查的邮箱
- 一次 SQL 查询获取所有激活邮箱
- 一次 SQL 查询获取所有 noreply 对应的用户（按 ID 或用户名）
- 构建邮箱 -> 用户的 map 缓存

### 5.4 Author vs Committer 差异分析

#### 5.4.1 Git 提交中的双重身份

Git 提交包含两个独立的身份字段：

| 字段 | 含义 | 存储位置 |
|------|------|----------|
| **Author** | 代码的实际编写者 | Git 对象内部 |
| **Committer** | 将代码提交到仓库的人 | Git 对象内部 |

**典型场景**:
- **正常开发**: Author == Committer（自己写自己提交）
- **Patch 工作流**: Author != Committer（维护者应用他人补丁）
- **GitHub PR 合并**: Author != Committer（GitHub 机器人或仓库管理员提交）
- **Cherry-pick**: Author != Committer（原作者不变，新的提交者）

#### 5.4.2 Gitea 中的不一致处理

**关键发现**（代码注释中的 FIXME）:

在 `services/git/commit.go:37`：
```go
committerUser := emailUsers.GetByEmail(c.Committer.Email)
// FIXME: why ValidateCommitsWithEmails uses "Author", 
//        but ParseCommitsWithSignature uses "Committer"?
```

**实际行为对比**:

| 功能模块 | 使用的邮箱字段 | 影响范围 |
|---------|---------------|----------|
| `ValidateCommitsWithEmails` | Author.Email | 作者头像、作者链接显示 |
| `ValidateCommitWithEmail` | Author.Email | 单提交页作者信息 |
| `ParseCommitsWithSignature` | Committer.Email | 签名验证的用户匹配 |
| `ParseCommitWithSignature` | Committer.Email | API 单提交验证 |

#### 5.4.3 对签名者识别的影响

**场景 1: Author != Committer，但只有 Author 上传了密钥**

```
提交信息:
  Author: alice@example.com (有 GPG 密钥)
  Committer: bob@example.com (没有密钥)
  签名: 使用 alice 的密钥签名

验证流程:
  parseCommitWithSignature 使用 Committer.Email (bob@...)
  -> 查找 bob 的密钥 -> 没找到
  -> 尝试按 KeyID 查找 -> 如果 alice 的密钥 KeyID 匹配
     -> 但验证邮箱时检查的是 bob 的邮箱
  -> 结果: 可能验证失败或显示错误的用户
```

**场景 2: 使用 noreply 邮箱提交**

```
提交信息:
  Author: 123+alice@noreply.gitea.io
  Committer: 456+bob@noreply.gitea.io

验证流程:
  1. 解析 noreply 邮箱 -> ID 123 和 456
  2. 按 ID 查找用户 -> 准确匹配
  3. 使用该用户的密钥进行签名验证
```

**最佳实践**:
- 如果使用 Gitea Web UI 提交，会自动使用用户的 noreply 邮箱
- 命令行提交时，应配置 `user.email` 为 Gitea noreply 邮箱
- 签名密钥的邮箱应与提交者邮箱（Committer.Email）匹配

## 六、信任状态判定分支与页面文案对应关系

### 6.1 信任状态判定逻辑

**文件**: `models/asymkey/gpg_key_commit_verification.go:CalculateTrustStatus` (行 117)

#### 6.1.1 CommitterTrustModel 模式

```go
if repoTrustModel == repo_model.CommitterTrustModel {
    verification.TrustStatus = "unmatched"  // 默认
    
    // 检查签名用户与提交用户是否匹配
    if (SigningUser.ID != 0 && CommittingUser.ID == SigningUser.ID) ||
       (SigningUser.ID == 0 && CommittingUser.ID == 0 && 
        SigningUser.Email == CommittingUser.Email) {
        verification.TrustStatus = "trusted"
    }
    return nil
}
```

**判定逻辑**:
- 签名用户 ID == 提交用户 ID → `trusted`
- 或两者都是外部用户（ID=0）且邮箱相同 → `trusted`
- 其他情况 → `unmatched`

#### 6.1.2 其他信任模式（CollaboratorCommitterTrustModel 等）

```go
// 默认假设是可信的
verification.TrustStatus = "trusted"

// 1. 实例默认密钥签名（无系统用户）
if verification.SigningUser.ID == 0 {
    if repoTrustModel == CollaboratorCommitterTrustModel &&
       (CommittingUser.ID != 0 || SigningUser.Email != CommittingUser.Email) {
        verification.TrustStatus = "untrusted"
    }
    return nil
}

// 2. 检查签名用户是否为仓库成员
isMember, _ := isOwnerMemberCollaborator(SigningUser)
if !isMember {
    verification.TrustStatus = "untrusted"
    if CommittingUser.ID != SigningUser.ID {
        verification.TrustStatus = "unmatched"  // 更严重
    }
}

// 3. 检查提交者与签名者是否匹配
else if repoTrustModel == CollaboratorCommitterTrustModel &&
        CommittingUser.ID != SigningUser.ID {
    verification.TrustStatus = "unmatched"
}
```

### 6.2 判定分支全景图

```
CalculateTrustStatus(verification, trustModel, isOwnerMemberCollaborator)
│
├── 未验证 (Verified=false) → 直接返回，TrustStatus 为空
│
├── CommitterTrustModel 模式
│   ├── 签名用户与提交用户匹配 → TrustStatus = "trusted"
│   └── 不匹配 → TrustStatus = "unmatched"
│
└── 其他模式（如 CollaboratorCommitterTrustModel）
    ├── 默认 → TrustStatus = "trusted"
    │
    ├── 签名用户为实例密钥（ID=0）
    │   └── 提交用户不匹配 → TrustStatus = "untrusted"
    │
    └── 签名用户为系统用户（ID≠0）
        ├── 不是仓库成员
        │   ├── 且提交者 != 签名者 → TrustStatus = "unmatched"
        │   └── 提交者 == 签名者 → TrustStatus = "untrusted"
        │
        └── 是仓库成员
            ├── Collaborator 模式 且 提交者 != 签名者 → TrustStatus = "unmatched"
            └── 其他情况 → 保持 "trusted"
```

### 6.3 三种 TrustState 含义

| TrustStatus | 含义 | CSS 类 |
|------------|------|--------|
| `trusted` | 签名可信：密钥属于仓库成员，且与提交者匹配 | `sign-trusted` |
| `untrusted` | 签名不可信：密钥不属于仓库成员 | `sign-untrusted` |
| `unmatched` | 签名者与提交者不匹配 | `sign-unmatched` |

### 6.4 前端模板分支与文案映射

**文件**: `templates/repo/commit_sign_badge.tmpl` (行 21-56)

#### 6.4.1 Verified = true 分支

```go
{{- if $verification.Verified -}}
    {{- $msgReason = $verification.Reason -}}  // "{name} / {key-id}"
    {{- $verified = true -}}
    
    {{- if eq $verification.TrustStatus "trusted" -}}
        {{- $extraClass = "sign-trusted" -}}
        {{- /* 无前缀，直接显示原因 */ -}}
    {{- else if eq $verification.TrustStatus "untrusted" -}}
        {{- $extraClass = "sign-untrusted" -}}
        {{- $msgReasonPrefix = ctx.Locale.Tr "repo.commits.signed_by_untrusted_user" -}}
    {{- else -}}
        {{- $extraClass = "sign-unmatched" -}}
        {{- $msgReasonPrefix = ctx.Locale.Tr "repo.commits.signed_by_untrusted_user_unmatched" -}}
    {{- end -}}
{{- end -}}
```

#### 6.4.2 Verified = false 分支

```go
{{- else -}}
    {{- if $verification.Warning -}}
        {{- $extraClass = "sign-warning" -}}
    {{- else -}}
        {{- $extraClass = "" -}}  // 未签名，不显示徽章
    {{- end -}}
    {{- $msgReason = ctx.Locale.Tr $verification.Reason -}}
    {{- /* Reason 是翻译 key，如 "gpg.error.not_signed_commit" */ -}}
{{- end -}}
```

#### 6.4.3 文案拼接逻辑

```go
{{- if $msgReasonPrefix -}}
    {{- $msgReason = print $msgReasonPrefix ": " $msgReason -}}
{{- end -}}
```

**示例结果**:
- `trusted`: `"Alice / 3AA5C34371567BD2"`
- `untrusted`: `"Signed by untrusted user: Alice / 3AA5C34371567BD2"`
- `unmatched`: `"Signed by untrusted user who does not match committer: Alice / 3AA5C34371567BD2"`

### 6.5 本地化文案对照表

| 文案 Key | 英文原文 | CSS 状态 |
|---------|---------|----------|
| **已签名且可信** | | |
| `repo.commits.signed_by` | Signed by | (无额外类) |
| `repo.commits.signed_by_untrusted_user` | Signed by untrusted user | `sign-untrusted` |
| `repo.commits.signed_by_untrusted_user_unmatched` | Signed by untrusted user who does not match committer | `sign-unmatched` |
| **验证失败** | | |
| `gpg.error.not_signed_commit` | Not a signed commit | (无徽章) |
| `gpg.error.no_gpg_keys_found` | No known key found for this signature in database | (无徽章或警告) |
| `gpg.error.probable_bad_signature` | WARNING! Although there is a key with this ID in the database, it does not verify this commit! This commit is SUSPICIOUS. | `sign-warning` |
| `gpg.error.extract_sign` | Failed to extract signature | `sign-warning` |
| `gpg.error.generate_hash` | Failed to generate hash of commit | `sign-warning` |
| `gpg.error.failed_retrieval_gpg_keys` | Failed to retrieve any key attached to the committer's account | `sign-warning` |
| `gpg.error.no_committer_account` | No account linked to committer's email address | `sign-warning` |

### 6.6 完整视觉映射矩阵

| 状态 | Verified | Warning | TrustStatus | CSS 类 | 图标 | Tooltip |
|------|----------|---------|------------|--------|------|---------|
| **可信签名** | true | - | trusted | `sign-trusted` | 🔒 (gitea-lock) | 用户名 / KeyID |
| **不可信签名** | true | - | untrusted | `sign-untrusted` | 🔒 | Signed by untrusted user: ... |
| **不匹配签名** | true | - | unmatched | `sign-unmatched` | 🔒 | Signed by untrusted user who does not match committer: ... |
| **可疑签名** | false | true | - | `sign-warning` | 🔓 | WARNING! Although there is a key... |
| **提取失败** | false | true | - | `sign-warning` | 🔓 | Failed to extract signature |
| **未签名** | false | false | - | (无) | (无) | Not a signed commit |
| **无密钥** | false | false | - | (无) | 🔓 | No known key found... |

## 七、代码调用关系总图

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
├── 信任状态: CalculateTrustStatus
│   └── 输出: TrustStatus (trusted/untrusted/unmatched)
└── 模板层
    ├── 数据: .Verification (CommitVerification)
    └── templates/repo/commit_sign_badge.tmpl
        ├── 分支: Verified ?
        │   ├── true → 信任状态分支
        │   └── false → Warning ? 显示/隐藏徽章
        └── 输出: HTML 签名徽章
```
