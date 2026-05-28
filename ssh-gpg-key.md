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

### 6.1 四种信任模型

**文件**: `models/repo/repo.go:105`

```go
const (
    DefaultTrustModel               = iota  // 回退为 CollaboratorTrustModel
    CommitterTrustModel                      // 只看签名者与提交者是否匹配
    CollaboratorTrustModel                   // 签名者必须是仓库成员
    CollaboratorCommitterTrustModel          // 签名者必须是仓库成员且与提交者匹配
)
```

**DefaultTrustModel 的解析**（`models/repo/repo.go:731`）：
```
repo.GetTrustModel()
├── 如果 repo.TrustModel == DefaultTrustModel
│   ├── 读取全局配置: setting.Repository.Signing.DefaultTrustModel
│   │   └── 默认值: "collaborator"（modules/setting/repository.go:285）
│   ├── 如果全局配置仍为 "default" → 回退为 CollaboratorTrustModel
│   └── 否则使用全局配置值
└── 否则使用 repo.TrustModel
```

### 6.2 CalculateTrustStatus 逐行精讲

**文件**: `models/asymkey/gpg_key_commit_verification.go:117`

函数签名：
```go
func CalculateTrustStatus(
    verification *CommitVerification,
    repoTrustModel repo_model.TrustModelType,
    isOwnerMemberCollaborator func(*user_model.User) (bool, error),
    keyMap *map[string]bool,
) error
```

**参数说明**：
- `isOwnerMemberCollaborator`：闭包函数，调用 `repo_model.IsOwnerMemberCollaborator`，
  判断用户是否为仓库所有者 / 有 Code 权限的团队成员 / 协作者
- `keyMap`：`map[string]bool` 缓存，以 GPG KeyID 为 key 避免重复查库

#### 6.2.1 前置守卫

```go
if !verification.Verified {
    return nil  // Verified=false 时直接返回，TrustStatus 保持空字符串
}
```

**关键**：只有密码学验证通过的提交才进入信任判定。

#### 6.2.2 CommitterTrustModel 分支（行 125-138）

```go
if repoTrustModel == repo_model.CommitterTrustModel {
    verification.TrustStatus = "unmatched"  // 默认不匹配

    if (verification.SigningUser.ID != 0 &&
        verification.CommittingUser.ID == verification.SigningUser.ID) ||
       (verification.SigningUser.ID == 0 && verification.CommittingUser.ID == 0 &&
        verification.SigningUser.Email == verification.CommittingUser.Email) {
        verification.TrustStatus = "trusted"
    }
    return nil
}
```

**判定逻辑**：

| 条件 | SigningUser.ID | CommittingUser.ID | 结果 |
|------|---------------|-------------------|------|
| 系统用户匹配 | ≠0 | == SigningUser.ID | `trusted` |
| 外部用户邮箱匹配 | 0 | 0 | 且 Email 相同 → `trusted` |
| 其他 | 任意 | 任意 | `unmatched` |

**CommitterTrustModel 不关心成员关系**，只看签名者是否就是提交者。
实例密钥签名（SigningUser.ID==0）时，如果提交者也不是系统用户，且邮箱相同，也算 trusted。

#### 6.2.3 非 CommitterTrustModel 分支（行 140-183）

此分支覆盖 `CollaboratorTrustModel` 和 `CollaboratorCommitterTrustModel`。

```go
verification.TrustStatus = "trusted"  // 默认可信
```

**第一层：实例密钥签名（SigningUser.ID == 0）**（行 143-153）

```go
if verification.SigningUser.ID == 0 {
    if repoTrustModel == CollaboratorCommitterTrustModel &&
       (verification.CommittingUser.ID != 0 ||
        verification.SigningUser.Email != verification.CommittingUser.Email) {
        verification.TrustStatus = "untrusted"
    }
    return nil
}
```

| 信任模型 | CommittingUser.ID | 邮箱匹配 | 结果 |
|---------|-------------------|---------|------|
| CollaboratorTrustModel | 任意 | 任意 | `trusted` |
| CollaboratorCommitterTrustModel | 0 | == | `trusted` |
| CollaboratorCommitterTrustModel | 0 | ≠ | `untrusted` |
| CollaboratorCommitterTrustModel | ≠0 | 任意 | `untrusted` |

**设计意图**：CollaboratorTrustModel 下实例密钥总是可信的；
CollaboratorCommitterTrustModel 要求实例密钥的邮箱与提交者邮箱一致（且提交者不是系统用户）。

**第二层：系统用户签名（SigningUser.ID ≠ 0）+ 成员校验**（行 155-181）

```go
if verification.SigningKey != nil {
    var isMember bool
    if keyMap != nil {
        var has bool
        isMember, has = (*keyMap)[verification.SigningKey.KeyID]
        if !has {
            isMember, err = isOwnerMemberCollaborator(verification.SigningUser)
            (*keyMap)[verification.SigningKey.KeyID] = isMember
        }
    } else {
        isMember, err = isOwnerMemberCollaborator(verification.SigningUser)
    }
    // ... 后续判定
}
```

**⚠️ 关键发现：成员校验以 `verification.SigningKey != nil` 为守卫条件**

这意味着：
- **GPG 签名**：`HashAndVerifyWithSubKeysCommitVerification` 在 `Verified=true` 时一定设置 `SigningKey`，
  所以 GPG 签名**总是会进入成员校验**
- **SSH 签名**：`verifySSHCommitVerification` 在 `Verified=true` 时设置的是 `SigningSSHKey`（而非 `SigningKey`），
  所以 `verification.SigningKey` 为 **nil**，**整个成员校验分支被跳过**！

这是 SSH 签名和 GPG 签名在信任判定上的根本差异。

**成员校验后的判定逻辑**：

```go
if !isMember {
    verification.TrustStatus = "untrusted"
    if verification.CommittingUser.ID != verification.SigningUser.ID {
        verification.TrustStatus = "unmatched"
    }
} else if repoTrustModel == CollaboratorCommitterTrustModel &&
          verification.CommittingUser.ID != verification.SigningUser.ID {
    verification.TrustStatus = "unmatched"
}
```

| 成员状态 | 信任模型 | CommittingUser == SigningUser | 结果 |
|---------|---------|-------------------------------|------|
| 不是成员 | 任意 | == (同一人) | `untrusted` |
| 不是成员 | 任意 | ≠ (不同人) | `unmatched` |
| 是成员 | Collaborator | 任意 | `trusted` |
| 是成员 | CollaboratorCommitter | == | `trusted` |
| 是成员 | CollaboratorCommitter | ≠ | `unmatched` |

### 6.3 SigningKey vs SigningSSHKey 对信任判定的影响

#### 6.3.1 GPG 签名验证成功时的 CommitVerification 构造

**`HashAndVerifyWithSubKeysCommitVerification`**（行 92-113）：
```go
return &CommitVerification{
    CommittingUser: committer,
    Verified:       true,
    Reason:         fmt.Sprintf("%s / %s", signer.Name, key.KeyID),
    SigningUser:    signer,
    SigningKey:     key,       // ← GPGKey 非 nil
    SigningEmail:   email,
}
```

`SigningKey` 字段**非 nil** → `CalculateTrustStatus` 中 `if verification.SigningKey != nil` 为 true → 进入成员校验分支。

#### 6.3.2 SSH 签名验证成功时的 CommitVerification 构造

**`verifySSHCommitVerification`**（行 437-450）：
```go
return &CommitVerification{
    CommittingUser: committer,
    Verified:       true,
    Reason:         fmt.Sprintf("%s / %s", signer.Name, k.Fingerprint),
    SigningUser:    signer,
    SigningSSHKey:  k,           // ← PublicKey 非 nil
    SigningEmail:   email,
    // SigningKey 未设置 → 默认 nil
}
```

`SigningKey` 字段为 **nil** → `if verification.SigningKey != nil` 为 false → **跳过整个成员校验分支**。

#### 6.3.3 SSH 签名的信任状态实际结果

因为成员校验分支被跳过，SSH 签名的信任状态**只受前面两层逻辑影响**：

1. CommitterTrustModel → 按 SigningUser vs CommittingUser 匹配
2. SigningUser.ID == 0（实例密钥） → 按 CollaboratorCommitterTrustModel 邮箱匹配
3. SigningUser.ID ≠ 0（系统用户） → **直接保持默认的 `"trusted"`**

**结论**：SSH 签名由系统用户签发时，无论该用户是否为仓库成员，TrustStatus 始终为 `"trusted"`。
这是当前代码的一个潜在问题——SSH 签名绕过了成员校验。

### 6.4 实例密钥场景 vs 仓库成员场景的差异

#### 6.4.1 实例密钥签名的身份构造

实例密钥签名时，SigningUser 由配置构造，而非从数据库查询：

**GPG 实例密钥**（`verifyWithGPGSettings` 行 332-335）：
```go
signer := &user_model.User{
    Name:  gpgSettings.Name,   // setting.Repository.Signing.SigningName
    Email: gpgSettings.Email,  // setting.Repository.Signing.SigningEmail
}
```

**SSH 实例密钥**（`parseCommitWithSSHSignature` 行 416-418）：
```go
signerUser := &user_model.User{
    Name:  gpgSettings.Name,   // setting.Repository.Signing.SigningName
    Email: gpgSettings.Email,  // setting.Repository.Signing.SigningEmail
}
```

两种实例密钥构造的 signer 都**没有数据库 ID**（`ID == 0`），因此 SigningUser.ID == 0。

#### 6.4.2 信任状态来源对比

| 维度 | 实例密钥场景 | 仓库成员场景 |
|------|------------|------------|
| **SigningUser 来源** | 配置文件构造（无 DB ID） | 数据库查询（有 DB ID） |
| **SigningUser.ID** | 0 | > 0 |
| **SigningKey** | GPG:非 nil / SSH:nil | GPG:非 nil / SSH:nil |
| **CommitterTrustModel** | CommittingUser.ID==0 且邮箱匹配 → trusted；否则 unmatched | SigningUser.ID==CommittingUser.ID → trusted；否则 unmatched |
| **CollaboratorTrustModel** | **始终 trusted**（SigningUser.ID==0 提前返回） | 进入成员校验 → 成员→trusted / 非成员→untrusted/unmatched |
| **CollaboratorCommitterTrustModel** | CommittingUser.ID==0 且邮箱匹配 → trusted；否则 untrusted | 进入成员校验 + 提交者匹配双重检查 |
| **SSH 签名 + 系统用户** | 不适用 | **SigningKey==nil，跳过成员校验，始终 trusted** |

#### 6.4.3 差异总结图

```
实例密钥签名 (SigningUser.ID == 0)
│
├── CommitterTrustModel
│   ├── CommittingUser.ID == 0 且邮箱匹配 → trusted
│   └── 其他 → unmatched
│
├── CollaboratorTrustModel
│   └── 始终 trusted（不检查成员关系）
│
└── CollaboratorCommitterTrustModel
    ├── CommittingUser.ID == 0 且邮箱匹配 → trusted
    └── 其他 → untrusted

仓库成员签名 (SigningUser.ID != 0)
│
├── CommitterTrustModel
│   ├── SigningUser.ID == CommittingUser.ID → trusted
│   └── 其他 → unmatched
│
├── CollaboratorTrustModel
│   ├── [GPG] SigningKey!=nil → 进入成员校验
│   │   ├── 是成员 → trusted
│   │   └── 不是成员 → untrusted / unmatched
│   └── [SSH] SigningKey==nil → 跳过成员校验 → 始终 trusted ⚠️
│
└── CollaboratorCommitterTrustModel
    ├── [GPG] SigningKey!=nil → 成员校验 + 提交者匹配
    │   ├── 是成员 且 提交者==签名者 → trusted
    │   ├── 是成员 且 提交者≠签名者 → unmatched
    │   ├── 不是成员 且 提交者==签名者 → untrusted
    │   └── 不是成员 且 提交者≠签名者 → unmatched
    └── [SSH] SigningKey==nil → 跳过成员校验 → 始终 trusted ⚠️
```

### 6.5 IsOwnerMemberCollaborator 成员判定

**文件**: `models/repo/collaboration.go:161`

```go
func IsOwnerMemberCollaborator(ctx context.Context, repo *Repository, userID int64) (bool, error) {
    // 1. 仓库所有者
    if repo.OwnerID == userID {
        return true, nil
    }
    // 2. 有 Code 权限的团队成员
    teamMember, _ := db.GetEngine(ctx).
        Join("INNER", "team_repo", "team_repo.team_id = team_user.team_id").
        Join("INNER", "team_unit", "team_unit.team_id = team_user.team_id").
        Where("team_repo.repo_id = ?", repo.ID).
        And("team_unit.`type` = ?", unit.TypeCode).
        And("team_user.uid = ?", userID).
        Table("team_user").Exist()
    if teamMember {
        return true, nil
    }
    // 3. 直接协作者
    return db.GetEngine(ctx).Get(&Collaboration{RepoID: repo.ID, UserID: userID})
}
```

判定优先级：仓库所有者 → 有 Code 单元权限的团队成员 → 直接协作者。

### 6.6 keyMap 缓存机制

`CalculateTrustStatus` 的 `keyMap` 参数用于在同一次请求中批量处理多个提交时缓存成员判定结果：

```go
if keyMap != nil {
    isMember, has = (*keyMap)[verification.SigningKey.KeyID]
    if !has {
        isMember, _ = isOwnerMemberCollaborator(verification.SigningUser)
        (*keyMap)[verification.SigningKey.KeyID] = isMember
    }
}
```

- **提交列表页**（`ParseCommitsWithSignature`）：传入 `&keyMap`，同一 KeyID 只查一次
- **提交详情页**（`routers/web/repo/commit.go:388`）：传入 `nil`，不使用缓存
- **仓库首页**（`routers/web/repo/view.go:129`）：传入 `nil`，不使用缓存
- **Graph 页面**（`gitgraph/graph_models.go:120`）：传入 `&keyMap`，使用缓存

### 6.7 三种 TrustStatus 含义

| TrustStatus | 含义 | CSS 类 |
|------------|------|--------|
| `trusted` | 签名可信：密钥属于仓库成员，且与提交者匹配 | `sign-trusted` |
| `untrusted` | 签名不可信：密钥不属于仓库成员 | `sign-untrusted` |
| `unmatched` | 签名者与提交者不匹配 | `sign-unmatched` |

### 6.8 前端模板分支与文案映射

**文件**: `templates/repo/commit_sign_badge.tmpl` (行 21-56)

#### 6.8.1 Verified = true 分支

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

#### 6.8.2 Verified = false 分支

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

#### 6.8.3 文案拼接逻辑

```go
{{- if $msgReasonPrefix -}}
    {{- $msgReason = print $msgReasonPrefix ": " $msgReason -}}
{{- end -}}
```

**示例结果**:
- `trusted`: `"Alice / 3AA5C34371567BD2"`
- `untrusted`: `"Signed by untrusted user: Alice / 3AA5C34371567BD2"`
- `unmatched`: `"Signed by untrusted user who does not match committer: Alice / 3AA5C34371567BD2"`

### 6.9 本地化文案对照表

| 文案 Key | 英文原文 | Warning | CSS 状态 |
|---------|---------|---------|----------|
| **已签名且可信** | | | |
| `repo.commits.signed_by` | Signed by | - | `sign-trusted` |
| `repo.commits.signed_by_untrusted_user` | Signed by untrusted user | - | `sign-untrusted` |
| `repo.commits.signed_by_untrusted_user_unmatched` | Signed by untrusted user who does not match committer | - | `sign-unmatched` |
| **签名可疑（Warning=true）** | | | |
| `gpg.error.probable_bad_signature` | WARNING! Although there is a key with this ID in the database, it does not verify this commit! This commit is SUSPICIOUS. | **true** | `sign-warning` |
| **无签名信息（Warning=false，列表页不渲染徽章）** | | | |
| `gpg.error.not_signed_commit` | Not a signed commit | false | 列表页:无 / 详情页:默认灰 |
| `gpg.error.no_gpg_keys_found` | No known key found for this signature in database | false | 列表页:无 / 详情页:默认灰 |
| `gpg.error.extract_sign` | Failed to extract signature | false | 列表页:无 / 详情页:默认灰 |
| `gpg.error.generate_hash` | Failed to generate hash of commit | false | 列表页:无 / 详情页:默认灰 |
| `gpg.error.failed_retrieval_gpg_keys` | Failed to retrieve any key attached to the committer's account | false | 列表页:无 / 详情页:默认灰 |
| `gpg.error.no_committer_account` | No account linked to committer's email address | false | 列表页:无 / 详情页:默认灰 |

> **注意**：原文档将 `extract_sign`/`generate_hash`/`failed_retrieval_gpg_keys`/`no_committer_account`
> 错误标注为 `sign-warning`。实际上这些场景的后端代码未设置 `Warning=true`，
> 模板中 `$extraClass` 被重置为空字符串，不会触发 `sign-warning` 样式。
> 唯一触发 `Warning=true` 的原因是 `BadSignature`（密钥在 DB 中找到但签名不匹配）。

### 6.10 原视觉映射矩阵的修正（三场景逐项验证）

#### 6.10.1 模板核心判断条件还原

`commit_sign_badge.tmpl` 的渲染逻辑分三层嵌套：

```
1. if $verification （CommitVerification 非 nil）
   ├── 进入后先设 $extraClass = "commit-is-signed"
   ├── if Verified
   │   └── 按TrustStatus 分 sign-trusted / sign-untrusted / sign-unmatched
   └── else（Verified=false）
       ├── if Warning  → 追加 "sign-warning"
       └── else        → $extraClass = ""（重置为空，撤销 "commit-is-signed"）

2. 徽章元素是否渲染
   if or (not $commit) $extraClass
   └── $extraClass 非空时才渲染 <span class="commit-sign-badge">

3. 徽章内部图标选择
   ├── if $verified  → gitea-lock / gitea-lock-cog + 头像
   └── else          → gitea-unlock（无头像）
```

**关键发现**：当 `Verified=false && Warning=false` 时，`$extraClass` 被重置为空字符串 `""`，
导致 `or (not $commit) $extraClass` 为 false（在列表页 $commit 非 nil），
**徽章元素完全不渲染**——既不显示图标也不显示 Tooltip。

---

#### 6.10.2 场景 A：未签名（c.Signature == nil）

**后端触发路径**：
```
services/asymkey/commit.go:ParseCommitWithSignatureCommitter (行 44)
└── c.Signature == nil →
    return &CommitVerification{
        CommittingUser: committer,
        Verified:       false,
        Warning:        false,      // 默认零值
        Reason:         "gpg.error.not_signed_commit",
    }
```

**Verified=false, Warning=false**

**模板推导**：
1. 进入 `if $verification` → `$extraClass = "commit-is-signed"`
2. `if $verification.Verified` → false，进入 else
3. `if $verification.Warning` → false → `$extraClass = ""`（重置）
4. `$msgReason = ctx.Locale.Tr "gpg.error.not_signed_commit"` → "Not a signed commit"

**渲染结果**：

| 页面 | $commit | `or (not $commit) $extraClass` | 徽章 | 图标 | Tooltip |
|------|---------|-------------------------------|------|------|---------|
| 提交列表 | 非 nil | `or false ""` = false | **不渲染** | 无 | 无 |
| 提交详情 | nil | `or true ""` = true | **渲染** | 🔓 gitea-unlock | "Not a signed commit" |
| 仓库首页 latest_commit | 非 nil | `or false ""` = false | **不渲染** | 无 | 无 |
| PR 提交列表 | 非 nil | `or false ""` = false | **不渲染** | 无 | 无 |

**修正**：原文档标注"未签名 → (无) 图标"，在列表页正确，但详情页**会**显示 unlock 图标。

---

#### 6.10.3 场景 B：无密钥（签名存在但数据库中找不到匹配密钥）

**GPG 后端触发路径**：
```
services/asymkey/commit.go:parseCommitWithGPGSignature (行 180)
└── 所有验证步骤都没匹配 → 兜底返回
    return &CommitVerification{
        CommittingUser: committer,
        Verified:       false,
        Warning:        defaultReason != NoKeyFound,  // NoKeyFound时Warning=false
        Reason:         defaultReason,                // "gpg.error.no_gpg_keys_found"
        SigningKey:     &GPGKey{KeyID: keyID},
    }
```

**SSH 后端触发路径**：
```
services/asymkey/commit.go:parseCommitWithSSHSignature (行 430)
└── 用户密钥/可信密钥/实例密钥都没匹配 →
    return &CommitVerification{
        CommittingUser: committerUser,
        Verified:       false,
        Reason:         NoKeyFound,      // "gpg.error.no_gpg_keys_found"
        // Warning 未显式设置 → 默认零值 false
    }
```

**Verified=false, Warning=false**

**模板推导**：与场景 A 完全相同——`$extraClass` 被重置为 `""`，徽章不渲染。

**渲染结果**：

| 页面 | $commit | 徽章 | 图标 | Tooltip |
|------|---------|------|------|---------|
| 提交列表 | 非 nil | **不渲染** | 无 | 无 |
| 提交详情 | nil | **渲染** | 🔓 gitea-unlock | "No known key found for this signature in database" |
| 仓库首页 | 非 nil | **不渲染** | 无 | 无 |
| PR 提交列表 | 非 nil | **不渲染** | 无 | 无 |

**修正**：原文档标注"无密钥 → 🔓 图标"，在列表页是**错误**的，列表页不渲染任何徽章元素；
仅在详情页（$commit=nil）才会显示 unlock 图标和 Tooltip。

**注意**：此场景 CommitVerification.SigningKey 不为 nil（含 KeyID），但模板中
`$verification.SigningKey.PaddedKeyID` 只在 Verified=true 时才被读取到 `$msgSigningKey`，
所以 SigningKey 信息不会出现在 Tooltip 中。

---

#### 6.10.4 场景 C：Warning（签名存在，密钥在 DB 中找到但验证失败）

**GPG 触发路径 1** — HashAndVerifyForKeyID 返回 BadSignature：
```
services/asymkey/commit.go:HashAndVerifyForKeyID (行 278)
└── 密钥 KeyID 在 DB 中找到，但签名不匹配 →
    return &CommitVerification{
        CommittingUser: committer,
        Verified:       false,
        Warning:        true,
        Reason:         BadSignature,  // "gpg.error.probable_bad_signature"
    }
```

**GPG 触发路径 2** — verifyWithGPGSettings 中默认密钥 KeyID 匹配但签名不匹配：
```
services/asymkey/commit.go:verifyWithGPGSettings (行 340)
└── keyID == k.KeyID 但签名不匹配 →
    return &CommitVerification{
        CommittingUser: committer,
        Verified:       false,
        Warning:        true,
        Reason:         BadSignature,  // 同样是 "gpg.error.probable_bad_signature"
    }
```

**其他 Warning 场景**：
| 触发条件 | Reason | Warning |
|---------|--------|---------|
| ExtractSignature 失败 | `gpg.error.extract_sign` | false (隐式) |
| GetUserByEmail 内部错误 | `gpg.error.no_committer_account` | false (隐式) |
| LoadSubKeys 失败 | `gpg.error.failed_retrieval_gpg_keys` | false (隐式) |
| CheckArmoredGPGKeyString 失败 | `gpg.error.generate_hash` | false (隐式) |
| DB 中有 KeyID 但签名不匹配 | `gpg.error.probable_bad_signature` | **true** |
| 默认密钥 KeyID 匹配但签名不匹配 | `gpg.error.probable_bad_signature` | **true** |

**重要发现**：除了 `BadSignature` 外，其他 Verified=false 的返回中 Warning 均为 false（零值）。
只有密钥在数据库中存在但签名验证失败时 Warning=true——这才是 `sign-warning` 的真正触发条件。

**Verified=false, Warning=true**

**模板推导**：
1. 进入 `if $verification` → `$extraClass = "commit-is-signed"`
2. `if $verification.Verified` → false，进入 else
3. `if $verification.Warning` → true → `$extraClass = "commit-is-signed sign-warning"`
4. `$msgReason = ctx.Locale.Tr "gpg.error.probable_bad_signature"`
   → "WARNING! Although there is a key with this ID in the database, it does not verify this commit! This commit is SUSPICIOUS."

**渲染结果**：

| 页面 | $commit | `or (not $commit) $extraClass` | 徽章 | 图标 | Tooltip |
|------|---------|-------------------------------|------|------|---------|
| 提交列表 | 非 nil | `or false "commit-is-signed sign-warning"` = true | **渲染** | 🔓 gitea-unlock | "WARNING! Although there is a key..." |
| 提交详情 | nil | `or true "..."` = true | **渲染** | 🔓 gitea-unlock | "WARNING! Although there is a key..." |
| 仓库首页 | 非 nil | true | **渲染** | 🔓 gitea-unlock | "WARNING! Although there is a key..." |
| PR 提交列表 | 非 nil | true | **渲染** | 🔓 gitea-unlock | "WARNING! Although there is a key..." |

**CSS 效果**：`commit-is-signed sign-warning` → 红色边框 + 红色背景（`--color-red-badge` / `--color-red-badge-bg`）

---

#### 6.10.5 修正后的完整视觉映射矩阵

| 状态 | Verified | Warning | $extraClass | 列表页徽章 | 详情页徽章 | 图标 | Tooltip |
|------|----------|---------|-------------|-----------|-----------|------|---------|
| **可信签名** | true | - | `commit-is-signed sign-trusted` | ✅ 绿底 | ✅ 绿底 | 🔒 lock | 用户名 / KeyID |
| **不可信签名** | true | - | `commit-is-signed sign-untrusted` | ✅ 黄底 | ✅ 黄底 | 🔒 lock | Signed by untrusted user: ... |
| **不匹配签名** | true | - | `commit-is-signed sign-unmatched` | ✅ 橙底 | ✅ 橙底 | 🔒 lock | ...who does not match committer: ... |
| **可疑签名** | false | true | `commit-is-signed sign-warning` | ✅ 红底 | ✅ 红底 | 🔓 unlock | WARNING! Although there is a key... |
| **未签名** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | Not a signed commit |
| **无密钥(GPG)** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | No known key found... |
| **无密钥(SSH)** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | No known key found... |
| **提取签名失败** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | Failed to extract signature |
| **获取密钥失败** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | Failed to retrieve any key... |
| **无提交者账户** | false | false | `""` (重置) | ❌ 不渲染 | ✅ 无底色 | 🔓 unlock | No account linked to committer's... |

**列表页 vs 详情页差异的根本原因**：

模板行 62：
```html
{{- if or (not $commit) $extraClass -}}
```
- **列表页/仓库首页/PR提交列表**：传入 `Commit` 字段（非 nil），条件变为 `or false $extraClass`
  - 当 $extraClass 为空时（未签名/无密钥/提取失败等），条件为 false → 徽章不渲染
  - 当 $extraClass 非空时（Warning/Verified），条件为 true → 徽章渲染
- **详情页**：不传 `Commit` 字段（nil），条件变为 `or true $extraClass`
  - 始终为 true → 徽章始终渲染，包括未签名/无密钥场景

---

#### 6.10.6 Warning=false 的 Verified=false 场景为何不显示徽章

设计意图解读（模板行 42 注释：`the commit is not signed`）：

1. **未签名提交**：`c.Signature == nil`，本质上没有签名 → 不需要展示签名状态
2. **无密钥提交**：有签名但找不到密钥 → 在列表页也被视为"无有效签名信息可展示"，与未签名同等处理
3. **提取/获取失败**：内部错误，不应向普通用户展示为签名问题 → 同等处理

**统一规则**：只有 `Verified=true`（签名有效）或 `Warning=true`（签名可疑）时才在列表页展示徽章；
其余 Verified=false + Warning=false 的情况只在详情页（独立渲染时）显示 unlock 图标。

#### 6.10.7 CSS 样式细节

**文件**: `web_src/css/repo/commit-sign.css`

| CSS 类组合 | 边框色 | 背景色 | 使用场景 |
|-----------|--------|--------|---------|
| `.commit-is-signed.sign-trusted` | `--color-green-badge` | `--color-green-badge-bg` | 签名可信 |
| `.commit-is-signed.sign-untrusted` | `--color-yellow-badge` | `--color-yellow-badge-bg` | 签名者非成员 |
| `.commit-is-signed.sign-unmatched` | `--color-orange-badge` | `--color-orange-badge-bg` | 签名者与提交者不匹配 |
| `.commit-is-signed.sign-warning` | `--color-red-badge` | `--color-red-badge-bg` | 签名可疑 |
| `.commit-sign-badge`（无 sign-* 附加类） | `--color-light-border` | 默认 | 详情页的未签名/无密钥 |

**注意**：所有 sign-* 状态样式都要求同时具有 `commit-is-signed` 类。
当 $extraClass 被重置为空时，`commit-is-signed` 也被移除，所以即使徽章元素被渲染（详情页），
也不会命中任何颜色样式，回退到默认的浅灰边框。

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

Git 提交展示（提交列表页）
├── routers/web/repo/commit.go:Commits
│   └── processGitCommits → ConvertFromGitCommit
│       ├── ValidateCommitsWithEmails (Author.Email → User)
│       ├── ParseCommitsWithSignature (Committer.Email → CommitVerification)
│       │   └── ParseCommitWithSignatureCommitter
│       │       ├── c.Signature==nil → {Verified:false, Warning:false, Reason:"not_signed_commit"}
│       │       ├── GPG签名 → parseCommitWithGPGSignature
│       │       │   ├── 验证成功 → {Verified:true, ...}
│       │       │   ├── BadSignature → {Verified:false, Warning:true, Reason:"probable_bad_signature"}
│       │       │   └── 无密钥兜底 → {Verified:false, Warning:false, Reason:"no_gpg_keys_found"}
│       │       └── SSH签名 → parseCommitWithSSHSignature
│       │           ├── 验证成功 → {Verified:true, ...}
│       │           └── 无密钥兜底 → {Verified:false, Warning:false, Reason:"no_gpg_keys_found"}
│       └── CalculateTrustStatus → TrustStatus
└── 模板渲染
    ├── commits_list.tmpl → commit_sign_badge(Commit=有值, ..., Verification)
    ├── graph/commits.tmpl → commit_sign_badge(Commit=有值, ..., Verification)
    ├── commits_list_small.tmpl → commit_sign_badge(Commit=有值, ..., Verification)
    └── latest_commit.tmpl → commit_sign_badge(Commit=有值, ..., Verification)

Git 提交展示（单提交详情页）
├── routers/web/repo/commit.go:Diff
│   ├── ParseCommitWithSignature → CommitVerification
│   ├── ValidateCommitWithEmail (Author.Email → User)
│   └── CalculateTrustStatus → TrustStatus
└── 模板渲染
    └── commit_page.tmpl → commit_sign_badge(Commit=nil, CommitSignVerification=Verification)
        └── Commit=nil → or(not nil, $extraClass) 恒为 true → 徽章始终渲染
```
