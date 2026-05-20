# OAuth2 登录后用户会话接续分析

本文档分析 Gitea 中第三方 OAuth2 登录后的授权回调、账号绑定和会话写入的接续关系。

---

## 整体流程概览

```
用户点击 OAuth2 登录按钮
        ↓
SignInOAuth() → 重定向到第三方提供商
        ↓
用户在第三方授权
        ↓
第三方回调到 SignInOAuthCallback()
        ├─→ 情况1: 找到匹配用户 → handleOAuth2SignIn()
        │                           ├─ oauth2SignInSync 同步信息
        │                           ├─ 检查 2FA 状态（不验证）
        │                           ├─ ⚠️ EnsureLinkExternalToUser() [在2FA验证之前执行]
        │                           ├─ 无2FA → updateSession 写入登录态
        │                           └─ 有2FA → 存 twofaUid → 跳转2FA页面 → 验证后登录
        ├─→ 情况2: 已登录用户 → LinkAccountToUser() 直接绑定 → 跳转设置页
        ├─→ 情况3: 自动注册 → createAndHandleCreatedUser()
        │                           ├─ 成功 → handleUserCreated()
        │                           │               ├─ AccountLinking != disabled → EnsureLinkExternalToUser()
        │                           │               └─ AccountLinking = disabled → ⚠️ 不绑定
        │                           └─ 重名/邮箱冲突 → 按 AccountLinking 配置回落
        │                                               ├─ auto → 自动绑定已有用户
        │                                               ├─ login → 跳转绑定页面
        │                                               └─ disabled → 直接报错
        └─→ 情况4: 需手动绑定 → 存 LinkAccountData → 跳转 link_account 页面
                                                        ├─ 登录已有账号 → oauth2LinkAccount()
                                                        │                           ├─ oauth2SignInSync 同步信息
                                                        │                           ├─ 无2FA → LinkAccountToUser() + 登录
                                                        │                           └─ 有2FA → 存 twofaUid+linkAccount → 跳转2FA
                                                        │                                                                   ↓
                                                        │                                                           2FA验证通过
                                                        │                                                                   ↓
                                                        │                                                           TOTP/WebAuthn/Passkey: linkAccountFromContext()
                                                        │                                                           备用码: ⚠️ 不执行绑定
                                                        │                                                                   ↓
                                                        │                                                           登录
                                                        └─ 注册新账号 → 创建用户 → handleUserCreated()
                                                                                └─ linkAccountData != nil → EnsureLinkExternalToUser()
```

---

## 1. 授权回调处理 (SignInOAuthCallback)

**文件**: `routers/web/auth/oauth.go:74-216`

### 核心流程

```go
func SignInOAuthCallback(ctx *context.Context) {
    // 1. 验证错误响应
    if ctx.Req.FormValue("error") != "" { ... }

    // 2. 获取 OAuth2 认证源
    authSource, err := auth.GetActiveOAuth2SourceByAuthName(ctx, authName)

    // 3. 处理回调，获取用户信息
    u, gothUser, err := oAuth2UserLoginCallback(ctx, authSource, ctx.Req, ctx.Resp)

    // 4. 分支处理
    if u == nil {
        if ctx.Doer != nil {
            // 已登录用户：绑定外部账号
            externalaccount.LinkAccountToUser(ctx, authSource.ID, ctx.Doer, gothUser)
            ctx.Redirect(setting.AppSubURL + "/user/settings/security")
            return
        } else if autoRegistrationEnabled {
            // 自动注册：创建新用户
            var missingFields []string
            // ... 检查必填字段 ...
            if len(missingFields) > 0 {
                // 字段缺失：仍然跳转到绑定页面
                showLinkingLogin(ctx, authSource.ID, gothUser)
                return
            }
            u = &user_model.User{...}
            linkAccountData := &LinkAccountData{authSource.ID, gothUser}
            if !createAndHandleCreatedUser(ctx, "", nil, u, overwriteDefault, linkAccountData) {
                return // 内部已处理回落逻辑
            }
            syncGroupsToTeams(...)
        } else {
            // 需要手动绑定：跳转到 link_account 页面
            showLinkingLogin(ctx, authSource.ID, gothUser)
            return
        }
    }

    // 5. 处理登录（如找到用户或创建用户成功）
    handleOAuth2SignIn(ctx, authSource, u, gothUser)
}
```

### 关键函数 oAuth2UserLoginCallback

**文件**: `routers/web/auth/oauth.go:453-532`

```go
func oAuth2UserLoginCallback(...) (*user_model.User, goth.User, error) {
    // 1. 通过 goth 获取第三方用户信息
    gothUser, err := oauth2Source.Callback(request, response)

    // 2. 检查必填声明（如 required claim）
    if oauth2Source.RequiredClaimName != "" { ... }

    // 3. 按 LoginName + LoginSource 查找用户
    user := &user_model.User{
        LoginName:   gothUser.UserID,
        LoginType:   auth.OAuth2,
        LoginSource: authSource.ID,
    }
    hasUser, err := user_model.GetIndividualUser(ctx, user)
    if hasUser { return user, gothUser, nil }

    // 4. 按 ExternalLoginUser 查找（外部关联表）
    externalLoginUser := &user_model.ExternalLoginUser{
        ExternalID:    gothUser.UserID,
        LoginSourceID: authSource.ID,
    }
    hasUser, err = user_model.GetExternalLogin(request.Context(), externalLoginUser)
    if hasUser {
        user, err = user_model.GetUserByID(request.Context(), externalLoginUser.UserID)
        return user, gothUser, err
    }

    // 5. 未找到用户，返回 nil
    return nil, gothUser, nil
}
```

**注意**: 两级用户查找：
- 第一级：通过 `LoginName` + `LoginSource` 直接匹配用户表
- 第二级：通过 `ExternalLoginUser` 关联表查找（支持多 OAuth 源绑定到同一用户）

---

## 2. 外部账号关联的触发时机与模式

### 2.1 LinkExternalToUser vs EnsureLinkExternalToUser

**文件**: `models/user/external_login_user.go:88-168`

```go
// LinkExternalToUser: 绑定外部账号到用户，如果已存在则报错（严格模式）
func LinkExternalToUser(ctx context.Context, user *User, externalLoginUser *ExternalLoginUser) error {
    has, err := db.Exist[ExternalLoginUser](ctx, ...)
    if has {
        return ErrExternalLoginUserAlreadyExist{...}
    }
    _, err = db.GetEngine(ctx).Insert(externalLoginUser)
    return err
}

// EnsureLinkExternalToUser: 确保外部账号已绑定（幂等模式）
// 已存在则更新，不存在则插入
func EnsureLinkExternalToUser(ctx context.Context, external *ExternalLoginUser) error {
    has, err := db.Exist[ExternalLoginUser](ctx, ...)
    if has {
        _, err = db.GetEngine(ctx).Where(...).AllCols().Update(external)
        return err
    }
    _, err = db.GetEngine(ctx).Insert(external)
    return err
}
```

### 2.2 模式 A：已有匹配用户 OAuth2 登录

**文件**: `routers/web/auth/oauth.go:344-389`

```go
func handleOAuth2SignIn(ctx *context.Context, authSource *auth.Source, u *user_model.User, gothUser goth.User) {
    // 1. 同步用户信息（头像、SSH Key 等）
    oauth2SignInSync(ctx, authSource.ID, u, gothUser)
    if ctx.Written() { return }

    // 2. 检查是否需要 2FA（仅检查是否已启用，不验证）
    needs2FA := false
    if !authSource.TwoFactorShouldSkip() {
        _, err := auth.GetTwoFactorByUID(ctx, u.ID)
        needs2FA = err == nil
    }

    // 3. 同步组声明和团队映射
    // ...

    // ⚠️ 4. 外部账号关联在 2FA 验证之前执行！
    // 即使用户最终没有通过 2FA，外部账号信息也已经建立/更新
    if err := externalaccount.EnsureLinkExternalToUser(ctx, authSource.ID, u, gothUser); err != nil {
        ctx.ServerError("EnsureLinkExternalToUser", err)
        return
    }

    // 5. 分支处理
    if !needs2FA {
        // 无 2FA：写入登录态
        updateSession(ctx, nil, map[string]any{
            session.KeyUID:                  u.ID,
            session.KeyUname:                u.Name,
            session.KeyUserHasTwoFactorAuth: userHasTwoFactorAuth,
        })
        redirectAfterAuth(ctx)
        return
    }

    // 有 2FA：保存 twofaUid，跳转验证页面（此时绑定已经完成）
    updateSession(ctx, nil, map[string]any{
        "twofaUid":      u.ID,
        "twofaRemember": false,
    })
    // 跳转到 2FA 或 WebAuthn 页面
}
```

**关键点**: 已有用户 OAuth2 登录时，`EnsureLinkExternalToUser` 在 **2FA 验证之前** 就执行了。这是因为这个用户已经是系统的合法用户，之前已经完成了账号绑定，这里只是更新外部账号信息。

### 2.3 模式 B：新账号绑定流程（通过 link_account 页面）

**文件**: `routers/web/auth/linkaccount.go:141-185`

```go
func oauth2LinkAccount(ctx *context.Context, u *user_model.User, linkAccountData *LinkAccountData, remember bool) {
    // 1. 同步用户信息
    oauth2SignInSync(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)
    if ctx.Written() { return }

    // 2. 检查 2FA 状态
    _, err := auth.GetTwoFactorByUID(ctx, u.ID)
    if err != nil {
        if !auth.IsErrTwoFactorNotEnrolled(err) { ... }

        // === 无 2FA：立即绑定并登录 ===
        err = externalaccount.LinkAccountToUser(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)
        handleSignIn(ctx, u, remember)
        return
    }

    // === 有 2FA：延迟绑定，只保存状态到 session ===
    // 注意：这里不调用 LinkAccountToUser！
    // 因为绑定是敏感操作，必须在 2FA 验证通过后才能执行
    if err := updateSession(ctx, nil, map[string]any{
        "twofaUid":      u.ID,
        "twofaRemember": remember,
        "linkAccount":   true,  // 标记：2FA 通过后需要执行绑定
    }); err != nil { ... }

    // 跳转到 2FA 或 WebAuthn 页面
}
```

**设计意图**: 新账号绑定是敏感操作（将第三方账号关联到本地账号），必须在用户通过密码 + 2FA 双重验证后才能执行。

### 2.4 模式 C：新用户注册后绑定

**文件**: `routers/web/auth/auth.go:691-716`

```go
func handleUserCreated(ctx *context.Context, u *user_model.User, possibleLinkAccountData *LinkAccountData) (ok bool) {
    // ... 设置首个用户为管理员 ...

    // update external user information
    // ⚠️ 只有当 possibleLinkAccountData != nil 时才执行绑定
    // 当 AccountLinking = disabled 时，possibleLinkAccountData 为 nil，跳过绑定
    if possibleLinkAccountData != nil {
        if err := externalaccount.EnsureLinkExternalToUser(ctx, possibleLinkAccountData.AuthSourceID, u, possibleLinkAccountData.GothUser); err != nil {
            log.Error("EnsureLinkExternalToUser failed: %v", err)
        }
    }

    // ... 发送激活邮件 ...
}
```

---

## 3. 2FA 验证后的绑定操作（各分支对比）

### 3.1 各分支绑定操作一致性对比

| 2FA 验证方式 | 文件 | 检查 `linkAccount` | 调用 `linkAccountFromContext` | 绑定是否执行 | 适用场景 |
|-------------|------|-------------------|-----------------------------|-------------|---------|
| **TOTP 验证码** | `2fa.go:76-82` | ✓ | ✓ | ✓ | 新账号绑定流程 |
| **WebAuthn 硬件密钥** | `webauthn.go:264-270` | ✓ | ✓ | ✓ | 新账号绑定流程 |
| **Passkey 登录** | `webauthn.go:151-156` | ✓ | ✓ | ✓ | 新账号绑定流程 |
| **备用码 (Scratch Code)** | `2fa.go:116-161` | ✗ | ✗ | **✗ 不执行！** | 新账号绑定流程 |
| **已有用户登录** | `oauth.go:386` | - | - | ✓（在2FA前执行） | 已有用户 OAuth2 登录 |

### 3.2 TOTP 验证后的绑定

**文件**: `routers/web/auth/2fa.go:43-96`

```go
func TwoFactorPost(ctx *context.Context) {
    // ... 验证 TOTP 密码 ...
    if ok && twofa.LastUsedPasscode != form.Passcode {
        remember := ctx.Session.Get("twofaRemember").(bool)
        u, err := user_model.GetUserByID(ctx, id)

        // === 关键：检查 linkAccount 标记（仅新账号绑定流程使用）===
        if ctx.Session.Get("linkAccount") != nil {
            // 从 session 读取 LinkAccountData，执行绑定
            err = linkAccountFromContext(ctx, u)
            if err != nil { ... }
        }

        twofa.LastUsedPasscode = form.Passcode
        auth.UpdateTwoFactor(ctx, twofa)

        handleSignIn(ctx, u, remember)
        return
    }
}
```

### 3.3 WebAuthn 验证后的绑定

**文件**: `routers/web/auth/webauthn.go:264-276`

```go
func WebAuthnLoginAssertionPost(ctx *context.Context) {
    // ... WebAuthn 验证 ...

    // === 关键：检查 linkAccount 标记（仅新账号绑定流程使用）===
    if ctx.Session.Get("linkAccount") != nil {
        if err := linkAccountFromContext(ctx, user); err != nil {
            ctx.ServerError("LinkAccountFromStore", err)
            return
        }
    }

    remember := ctx.Session.Get("twofaRemember").(bool)
    handleSignInFull(ctx, user, remember)
    _ = ctx.Session.Delete("twofaUid")
    ctx.JSONRedirect(consumeAuthRedirectLink(ctx))
}
```

### 3.4 Passkey 登录后的绑定

**文件**: `routers/web/auth/webauthn.go:150-161`

```go
func WebAuthnPasskeyLogin(ctx *context.Context) {
    // ... Passkey 验证 ...

    // === 关键：检查 linkAccount 标记（仅新账号绑定流程使用）===
    if ctx.Session.Get("linkAccount") != nil {
        if err := linkAccountFromContext(ctx, user); err != nil {
            ctx.ServerError("LinkAccountFromStore", err)
            return
        }
    }

    remember := false // TODO: implement remember me
    handleSignInFull(ctx, user, remember)
    ctx.JSONRedirect(consumeAuthRedirectLink(ctx))
}
```

### 3.5 备用码验证：⚠️ 不执行绑定

**文件**: `routers/web/auth/2fa.go:116-164`

```go
func TwoFactorScratchPost(ctx *context.Context) {
    // ... 验证备用码 ...
    if twofa.VerifyScratchToken(form.Token) {
        // ... 作废备用码，生成新的 ...

        remember := ctx.Session.Get("twofaRemember").(bool)
        u, err := user_model.GetUserByID(ctx, id)

        // ⚠️ 注意：这里没有检查 linkAccount 标记！
        // 如果是新账号绑定流程，绑定操作不会执行

        handleSignInFull(ctx, u, remember)
        ctx.Redirect(setting.AppSubURL + "/user/settings/security")
        return
    }
}
```

**⚠️ 重要发现**: 如果用户使用备用码进行 2FA 验证，新账号绑定操作不会执行！这是一个不一致性。

### 3.6 linkAccountFromContext 辅助函数

**文件**: `routers/web/auth/linkaccount.go:275-281`

```go
func linkAccountFromContext(ctx *context.Context, user *user_model.User) error {
    linkAccountData := oauth2GetLinkAccountData(ctx)
    if linkAccountData == nil {
        return errors.New("not in LinkAccount session")
    }
    return externalaccount.LinkAccountToUser(ctx, linkAccountData.AuthSourceID, user, linkAccountData.GothUser)
}
```

---

## 4. 账号绑定逻辑

### 4.1 会话中的 LinkAccountData

**文件**: `routers/web/auth/oauth.go:274-303`

```go
type LinkAccountData struct {
    AuthSourceID int64
    GothUser     goth.User
}

// 存储到 session
func showLinkingLogin(ctx *context.Context, authSourceID int64, gothUser goth.User) {
    Oauth2SetLinkAccountData(ctx, LinkAccountData{authSourceID, gothUser})
    ctx.Redirect(setting.AppSubURL + "/user/link_account")
}

// 从 session 读取
func oauth2GetLinkAccountData(ctx *context.Context) *LinkAccountData {
    v, ok := ctx.Session.Get("linkAccountData").(LinkAccountData)
    if !ok { return nil }
    return &v
}
```

### 4.2 LinkAccount 页面展示

**文件**: `routers/web/auth/linkaccount.go:45-95`

用户看到两个选项：
1. **登录已有账号** → 调用 `LinkAccountPostSignIn`
2. **注册新账号** → 调用 `LinkAccountPostRegister`

### 4.3 绑定已有账号 (分阶段落地)

**文件**: `routers/web/auth/linkaccount.go:114-185`

```go
func LinkAccountPostSignIn(ctx *context.Context) {
    // 1. 验证用户输入的用户名密码
    u, _, err := auth_service.UserSignIn(ctx, signInForm.UserName, signInForm.Password)

    // 2. 执行绑定和登录
    oauth2LinkAccount(ctx, u, linkAccountData, signInForm.Remember)
}
```

### 4.4 创建新账号并绑定

**文件**: `routers/web/auth/linkaccount.go:188-273`

```go
func LinkAccountPostRegister(ctx *context.Context) {
    // 1. 创建用户
    u := &user_model.User{
        Name:        form.UserName,
        Email:       form.Email,
        Passwd:      form.Password,
        LoginType:   auth.OAuth2,
        LoginSource: linkAccountData.AuthSourceID,
        LoginName:   linkAccountData.GothUser.UserID,
    }

    // 2. 创建用户并处理绑定（内部会处理重名回落）
    if !createAndHandleCreatedUser(ctx, tplLinkAccount, form, u, nil, linkAccountData) {
        return
    }

    // 3. 同步信息
    oauth2SignInSync(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)

    // 4. 同步团队映射
    syncGroupsToTeams(...)

    // 5. 登录
    handleSignIn(ctx, u, false)
}
```

### 4.5 自动注册遇重名的回落机制

**文件**: `routers/web/auth/auth.go:621-647` (createUserInContext)

```go
func createUserInContext(ctx *context.Context, tpl templates.TplName, form any, u *user_model.User,
    overwrites *user_model.CreateUserOverwriteOptions, possibleLinkAccountData *LinkAccountData) (ok bool) {

    meta := &user_model.Meta{...}
    if err := user_model.CreateUser(ctx, u, meta, overwrites); err != nil {
        // === 关键：如果创建失败是因为用户名/邮箱已存在，且有 LinkAccountData ===
        if possibleLinkAccountData != nil &&
            (user_model.IsErrUserAlreadyExist(err) || user_model.IsErrEmailAlreadyUsed(err)) {

            switch setting.OAuth2Client.AccountLinking {
            case setting.OAuth2AccountLinkingAuto:
                // === 自动模式：自动查找已有用户并绑定 ===
                var user *user_model.User
                user = &user_model.User{Name: u.Name}
                hasUser, err := user_model.GetIndividualUser(ctx, user)
                if !hasUser || err != nil {
                    user = &user_model.User{Email: u.Email}
                    hasUser, err = user_model.GetIndividualUser(ctx, user)
                    if !hasUser || err != nil { ... }
                }
                // 直接走绑定流程
                oauth2LinkAccount(ctx, user, possibleLinkAccountData, true)
                return false

            case setting.OAuth2AccountLinkingLogin:
                // === 登录模式：跳转到绑定页面，让用户手动确认 ===
                showLinkingLogin(ctx, possibleLinkAccountData.AuthSourceID, possibleLinkAccountData.GothUser)
                return false

            case setting.OAuth2AccountLinkingDisabled:
                // === 禁用模式：不进入 switch，直接报错 ===
            }
        }

        // ... 其他错误处理 ...
    }
    log.Trace("Account created: %s", u.Name)
    return true
}
```

### 4.6 AccountLinking = disabled 的实际行为边界

**文件**: `routers/web/auth/oauth.go:195-198`

```go
linkAccountData := &LinkAccountData{authSource.ID, gothUser}
if setting.OAuth2Client.AccountLinking == setting.OAuth2AccountLinkingDisabled {
    linkAccountData = nil  // 不传递 linkAccountData
}
if !createAndHandleCreatedUser(ctx, "", nil, u, overwriteDefault, linkAccountData) {
    return
}
```

**行为边界总结**:

| 场景 | AccountLinking = disabled |
|------|--------------------------|
| 自动注册遇重名 | ✗ 不回落，直接报错（因为 linkAccountData 是 nil，不会进入 switch） |
| 已有 OAuth2 用户登录 | ✓ 允许，正常调用 `EnsureLinkExternalToUser`（在 handleOAuth2SignIn 中，不依赖 linkAccountData） |
| 已登录用户主动绑定 | ✓ 允许，`SignInOAuthCallback` 中直接调用 `LinkAccountToUser` |
| link_account 页面手动绑定 | ✓ 允许，`linkaccount.go` 不检查 AccountLinking 设置 |
| 新用户注册后自动绑定 | ✗ 不绑定（因为 linkAccountData 是 nil，`handleUserCreated` 中跳过） |

**AccountLinking 配置说明** (定义于 `modules/setting/oauth2.go`):
- `OAuth2AccountLinkingDisabled` (disabled): 仅禁止**自动注册流程中的重名回落**和**新用户自动绑定**，不影响已有用户登录和主动绑定
- `OAuth2AccountLinkingLogin` (login, **默认**): 跳转到登录页面，让用户手动登录已有账号进行绑定
- `OAuth2AccountLinkingAuto` (auto): 自动查找匹配的用户（按用户名或邮箱），直接执行绑定流程

### 4.7 自动注册流程

**文件**: `routers/web/auth/oauth.go:141-207`

```go
if !setting.Service.AllowOnlyInternalRegistration && setting.OAuth2Client.EnableAutoRegistration {
    // 提取用户名和邮箱
    uname, err := extractUserNameFromOAuth2(&gothUser)

    // 检查必填字段
    var missingFields []string
    if gothUser.UserID == "" { missingFields = append(missingFields, "sub") }
    if gothUser.Email == "" { missingFields = append(missingFields, "email") }
    if uname == "" { /* 根据配置检查 nickname 或 preferred_username */ }

    if len(missingFields) > 0 {
        // 字段缺失：跳转到绑定页面
        gothUser.RawData["__giteaAutoRegMissingFields"] = missingFields
        showLinkingLogin(ctx, authSource.ID, gothUser)
        return
    }

    // 创建用户
    u := &user_model.User{
        Name:        uname,
        Email:       gothUser.Email,
        LoginType:   auth.OAuth2,
        LoginSource: authSource.ID,
        LoginName:   gothUser.UserID,
    }

    // 根据组声明设置管理员/受限用户
    isAdmin, isRestricted := getUserAdminAndRestrictedFromGroupClaims(source, &gothUser)

    // 创建并处理（内部处理重名回落）
    linkAccountData := &LinkAccountData{authSource.ID, gothUser}
    if setting.OAuth2Client.AccountLinking == setting.OAuth2AccountLinkingDisabled {
        linkAccountData = nil
    }
    if !createAndHandleCreatedUser(ctx, "", nil, u, overwriteDefault, linkAccountData) {
        return // 内部已处理回落
    }

    // 同步团队
    syncGroupsToTeams(ctx, source, &gothUser, u)
}
```

---

## 5. 会话写入机制与状态清理

### 5.1 updateSession 核心函数

**文件**: `routers/web/auth/auth.go:934-954`

```go
func updateSession(ctx *context.Context, deletes []string, updates map[string]any) error {
    // 1. 重新生成 session ID（防止会话固定攻击）
    if _, err := session.RegenerateSession(ctx.Resp, ctx.Req); err != nil {
        return fmt.Errorf("regenerate session: %w", err)
    }

    sess := ctx.Session
    sessID := sess.ID()

    // 2. 删除指定的 session 键
    for _, k := range deletes {
        if err := sess.Delete(k); err != nil {
            return fmt.Errorf("delete %v in session[%s]: %w", k, sessID, err)
        }
    }

    // 3. 设置新的 session 键值
    for k, v := range updates {
        if err := sess.Set(k, v); err != nil {
            return fmt.Errorf("set %v in session[%s]: %w", k, sessID, err)
        }
    }

    // 4. 持久化 session
    if err := sess.Release(); err != nil {
        return fmt.Errorf("store session[%s]: %w", sessID, err)
    }
    return nil
}
```

### 5.2 各分支的会话状态清理对比表

| 场景 | 调用位置 | 删除的 session 键 | 写入的 session 键 |
|------|---------|------------------|------------------|
| **已有用户登录（无2FA）** | `oauth.go:407` | 无（nil） | `uid`, `uname`, `userHasTwoFactorAuth` |
| **已有用户登录（需2FA）** | `oauth.go:432` | 无（nil） | `twofaUid`, `twofaRemember` |
| **绑定流程需2FA** | `linkaccount.go:167` | 无（nil） | `twofaUid`, `twofaRemember`, `linkAccount=true` |
| **最终登录完成** | `auth.go:401` | `openid_verified_uri`, `openid_signin_remember`, `openid_determined_email`, `openid_determined_username`, `twofaUid`, `twofaRemember`, `linkAccount`, `linkAccountData` | `uid`, `uname`, `userHasTwoFactorAuth` |
| **自动登录（remember）** | `auth.go:121` | 无（nil） | `uid`, `uname`, `userHasTwoFactorAuth` |
| **存储绑定数据** | `oauth.go:292` | 无（nil） | `linkAccountData` |

### 5.3 handleSignInFull 的完整清理

**文件**: `routers/web/auth/auth.go:384-442`

```go
func handleSignInFull(ctx *context.Context, u *user_model.User, remember bool) {
    // 1. 设置 remember me cookie（如果勾选）
    if remember {
        nt, token, err := auth_service.CreateAuthTokenForUserID(ctx, u.ID)
        ctx.SetSiteCookie(setting.CookieRememberName, nt.ID+":"+token, ...)
    }

    // 2. 检查 2FA 状态
    userHasTwoFactorAuth, err := auth.HasTwoFactorOrWebAuthn(ctx, u.ID)

    // 3. 完整清理所有认证中间状态，写入最终登录态
    if err := updateSession(ctx, []string{
        // OpenID 相关
        "openid_verified_uri",
        "openid_signin_remember",
        "openid_determined_email",
        "openid_determined_username",
        // 2FA 相关
        "twofaUid",
        "twofaRemember",
        // 账号绑定相关
        "linkAccount",
        "linkAccountData",
    }, map[string]any{
        session.KeyUID:                  u.ID,
        session.KeyUname:                u.Name,
        session.KeyUserHasTwoFactorAuth: userHasTwoFactorAuth,
    }); err != nil { ... }

    // 4. 设置语言
    resetLocale(ctx, u)

    // 5. 更新最后登录时间
    user_service.UpdateUser(ctx, u, &user_service.UpdateOptions{SetLastLogin: true})
}
```

**会话键说明** (定义于 `modules/session/key.go`):
- `KeyUID = "uid"`: 用户 ID
- `KeyUname = "uname"`: 用户名
- `KeyUserHasTwoFactorAuth`: 用户是否启用了双因素认证

### 5.4 2FA 场景下的会话流转

```
已有用户 OAuth2 登录
        ↓
handleOAuth2SignIn()
        ├─ oauth2SignInSync 同步信息
        ├─ 检查 2FA 状态
        ├─ EnsureLinkExternalToUser() [绑定在 2FA 验证之前完成]
        ├─ 无2FA → updateSession 写入登录态
        └─ 有2FA → updateSession 写入 twofaUid → 跳转2FA页面 → 验证后登录

新账号绑定流程（通过 link_account 页面）
        ↓
oauth2LinkAccount()
        ├─ oauth2SignInSync 同步信息
        ├─ 无2FA → LinkAccountToUser() + 登录
        └─ 有2FA → updateSession 写入 twofaUid + linkAccount=true → 跳转2FA
                                                                ↓
                                                        2FA 验证通过
                                                                ↓
                                                        TOTP/WebAuthn/Passkey: linkAccountFromContext()
                                                        备用码: ⚠️ 跳过绑定
                                                                ↓
                                                        handleSignInFull() → 清理所有中间状态
```

---

## 6. 关键接续点总结（修正版）

### 6.1 回调 → 查找用户
```
SignInOAuthCallback()
    ↓
oAuth2UserLoginCallback() → 返回 (user, gothUser, error)
    ├─ 找到 user → handleOAuth2SignIn()
    │                   ├─ oauth2SignInSync 同步信息
    │                   ├─ 检查 2FA 状态
    │                   ├─ EnsureLinkExternalToUser() [在2FA验证前执行!]
    │                   ├─ 无2FA → updateSession 写入登录态
    │                   └─ 有2FA → updateSession 写入 twofaUid → 跳转2FA
    └─ 未找到 user → 进入绑定/注册流程
```

### 6.2 绑定流程接续（含 2FA 分阶段）
```
showLinkingLogin() → 保存 LinkAccountData 到 session
    ↓
用户跳转到 /user/link_account
    ├─ 登录 → LinkAccountPostSignIn() → oauth2LinkAccount()
    │                                       ├─ 同步信息
    │                                       ├─ 无2FA → LinkAccountToUser() + handleSignIn()
    │                                       └─ 有2FA → updateSession(linkAccount=true) → 跳转2FA
    │                                                                   ↓
    │                                                           2FA 验证通过
    │                                                                   ↓
    │                                                           TOTP/WebAuthn/Passkey: linkAccountFromContext()
    │                                                           备用码: ⚠️ 跳过绑定
    │                                                                   ↓
    │                                                           handleSignIn()
    └─ 注册 → LinkAccountPostRegister() → createAndHandleCreatedUser()
                                                    ├─ 成功 → handleUserCreated()
                                                    │               └─ linkAccountData != nil → EnsureLinkExternalToUser()
                                                    └─ 重名 → 按 AccountLinking 回落
```

### 6.3 自动注册回落接续
```
createUserInContext() 尝试创建用户
    ↓
创建失败（用户名/邮箱已存在）
    ↓
检查 possibleLinkAccountData != nil
    ↓
AccountLinking 配置
    ├─ auto → 按用户名/邮箱查找已有用户 → oauth2LinkAccount()
    ├─ login → showLinkingLogin() → 跳转绑定页面
    └─ disabled → 直接报错（不进入 switch）
```

### 6.4 登录会话接续
```
认证通过（密码/2FA/WebAuthn/Passkey）
    ↓
updateSession() 执行：
    1. RegenerateSession()       - 重新生成 session ID（防固定攻击）
    2. 清理 8 个认证中间状态键    - 防止状态残留
    3. 写入 uid / uname 等        - 建立最终登录态
    4. Release()                  - 持久化
    ↓
后续请求通过 session 中的 uid 识别用户
```

---

## 7. 关键代码位置速查

| 功能 | 文件 | 函数 |
|------|------|------|
| OAuth2 回调入口 | `routers/web/auth/oauth.go` | `SignInOAuthCallback` |
| 用户查找逻辑 | `routers/web/auth/oauth.go` | `oAuth2UserLoginCallback` |
| 已有用户登录处理（含 2FA 分支） | `routers/web/auth/oauth.go` | `handleOAuth2SignIn` |
| 已有用户外部账号关联 | `routers/web/auth/oauth.go:386` | `EnsureLinkExternalToUser` 调用 |
| 绑定已有账号（分阶段） | `routers/web/auth/linkaccount.go` | `oauth2LinkAccount` |
| TOTP 2FA 验证后绑定 | `routers/web/auth/2fa.go` | `TwoFactorPost` |
| WebAuthn 验证后绑定 | `routers/web/auth/webauthn.go` | `WebAuthnLoginAssertionPost` |
| Passkey 登录后绑定 | `routers/web/auth/webauthn.go` | `WebAuthnPasskeyLogin` |
| 备用码验证（无绑定） | `routers/web/auth/2fa.go` | `TwoFactorScratchPost` |
| 从 session 执行绑定 | `routers/web/auth/linkaccount.go` | `linkAccountFromContext` |
| 绑定页面展示 | `routers/web/auth/linkaccount.go` | `LinkAccount` |
| 绑定已有账号入口 | `routers/web/auth/linkaccount.go` | `LinkAccountPostSignIn` |
| 注册新账号入口 | `routers/web/auth/linkaccount.go` | `LinkAccountPostRegister` |
| 自动注册重名回落 | `routers/web/auth/auth.go` | `createUserInContext` |
| 用户创建后处理 | `routers/web/auth/auth.go` | `handleUserCreated` |
| 严格绑定（存在则报错） | `models/user/external_login_user.go` | `LinkExternalToUser` |
| 幂等绑定（存在则更新） | `models/user/external_login_user.go` | `EnsureLinkExternalToUser` |
| 通用登录处理 | `routers/web/auth/auth.go` | `handleSignIn` / `handleSignInFull` |
| 会话更新核心 | `routers/web/auth/auth.go` | `updateSession` |
| 会话键定义 | `modules/session/key.go` | `KeyUID` / `KeyUname` |
| 用户信息同步 | `routers/web/auth/oauth_signin_sync.go` | `oauth2SignInSync` |
| AccountLinking 配置 | `modules/setting/oauth2.go` | `OAuth2AccountLinkingType` |

---

## 8. 重要发现与注意事项

### ⚠️ 备用码绑定不一致
使用备用码（Scratch Code）进行 2FA 验证时，**不会执行新账号绑定操作**。这是因为 `TwoFactorScratchPost` (`routers/web/auth/2fa.go:116`) 没有检查 `linkAccount` 标记，与 TOTP、WebAuthn、Passkey 三个分支不一致。

### ⚠️ 已有用户登录时绑定时机
已有 OAuth2 用户登录时，`EnsureLinkExternalToUser` 在 **2FA 验证之前** 就执行了（`oauth.go:386`）。这意味着即使用户最终没有通过 2FA，外部账号信息也已经被更新。这是合理的，因为这个用户已经是系统的合法用户，之前已经完成了账号绑定，这里只是更新外部账号信息。

### ⚠️ AccountLinking = disabled 边界有限
`AccountLinking = disabled` **仅**影响：
- 自动注册流程中的重名回落（不回落，直接报错）
- 新用户注册后自动绑定（不执行绑定）

但**不限制**：
- 已有 OAuth2 用户正常登录（仍然调用 `EnsureLinkExternalToUser`）
- 已登录用户主动绑定新的 OAuth2 账号（`SignInOAuthCallback` 中直接绑定）
- 用户通过 link_account 页面手动绑定（`linkaccount.go` 不检查此设置）

### 绑定时机区分原则
- **已有用户登录**：绑定在 2FA 验证前执行（更新已有绑定关系）
- **新账号绑定**：绑定在 2FA 验证后执行（建立新的绑定关系，敏感操作）
