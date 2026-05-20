# OAuth2 登录后用户会话接续分析

本文档分析 Gitea 中第三方 OAuth2 登录后的授权回调、账号绑定和会话写入的接续关系。

## 整体流程概览

```
用户点击 OAuth2 登录按钮
        ↓
SignInOAuth() → 重定向到第三方提供商
        ↓
用户在第三方授权
        ↓
第三方回调到 SignInOAuthCallback()
        ├─→ 情况1: 找到匹配用户 → 直接登录
        ├─→ 情况2: 已登录用户 → 绑定账号
        ├─→ 情况3: 自动注册 → 创建新用户 → 登录
        └─→ 情况4: 需手动绑定 → 跳转到 link_account 页面
                                                        ├─→ 用户登录已有账号 → 绑定后登录
                                                        └─→ 用户注册新账号 → 创建后登录
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
        } else if autoRegistrationEnabled {
            // 自动注册：创建新用户
            createAndHandleCreatedUser(...)
        } else {
            // 需要手动绑定：跳转到 link_account 页面
            showLinkingLogin(ctx, authSource.ID, gothUser)
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

**注意**: 这里有两级用户查找：
- 第一级：通过 `LoginName` + `LoginSource` 直接匹配用户表
- 第二级：通过 `ExternalLoginUser` 关联表查找（支持多 OAuth 源绑定到同一用户）

---

## 2. 账号绑定逻辑

### 2.1 会话中的 LinkAccountData

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

### 2.2 LinkAccount 页面展示

**文件**: `routers/web/auth/linkaccount.go:45-95`

用户看到两个选项：
1. **登录已有账号** → 调用 `LinkAccountPostSignIn`
2. **注册新账号** → 调用 `LinkAccountPostRegister`

### 2.3 绑定已有账号

**文件**: `routers/web/auth/linkaccount.go:114-185`

```go
func LinkAccountPostSignIn(ctx *context.Context) {
    // 1. 验证用户输入的用户名密码
    u, _, err := auth_service.UserSignIn(ctx, signInForm.UserName, signInForm.Password)

    // 2. 执行绑定和登录
    oauth2LinkAccount(ctx, u, linkAccountData, signInForm.Remember)
}

func oauth2LinkAccount(ctx *context.Context, u *user_model.User, linkAccountData *LinkAccountData, remember bool) {
    // 1. 同步用户信息（头像、SSH Key 等）
    oauth2SignInSync(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)

    // 2. 检查 2FA
    _, err := auth.GetTwoFactorByUID(ctx, u.ID)
    if err != nil && auth.IsErrTwoFactorNotEnrolled(err) {
        // 无 2FA：直接绑定并登录
        externalaccount.LinkAccountToUser(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)
        handleSignIn(ctx, u, remember)
        return
    }

    // 有 2FA：保存会话，跳转到 2FA 页面
    updateSession(ctx, nil, map[string]any{
        "twofaUid":      u.ID,
        "twofaRemember": remember,
        "linkAccount":   true,
    })
    // 跳转到 2FA 或 WebAuthn 页面
}
```

### 2.4 创建新账号并绑定

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

    // 2. 创建用户并处理绑定
    createAndHandleCreatedUser(ctx, tplLinkAccount, form, u, nil, linkAccountData)

    // 3. 同步信息
    oauth2SignInSync(ctx, linkAccountData.AuthSourceID, u, linkAccountData.GothUser)

    // 4. 同步团队映射
    syncGroupsToTeams(...)

    // 5. 登录
    handleSignIn(ctx, u, false)
}
```

### 2.5 自动注册流程

**文件**: `routers/web/auth/oauth.go:141-207`

```go
if !setting.Service.AllowOnlyInternalRegistration && setting.OAuth2Client.EnableAutoRegistration {
    // 提取用户名和邮箱
    uname, err := extractUserNameFromOAuth2(&gothUser)

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

    // 创建并处理
    linkAccountData := &LinkAccountData{authSource.ID, gothUser}
    createAndHandleCreatedUser(ctx, "", nil, u, overwriteDefault, linkAccountData)

    // 同步团队
    syncGroupsToTeams(ctx, source, &gothUser, u)
}
```

### 2.6 外部账号关联核心函数

**文件**: `services/externalaccount/user.go`

```go
// LinkAccountToUser: 绑定外部账号到用户
func LinkAccountToUser(ctx context.Context, authSourceID int64, user *user_model.User, gothUser goth.User) error {
    externalLoginUser := toExternalLoginUser(authSourceID, user, gothUser)
    return user_model.LinkExternalToUser(ctx, user, externalLoginUser)
}

// EnsureLinkExternalToUser: 确保外部账号已绑定（幂等）
func EnsureLinkExternalToUser(ctx context.Context, authSourceID int64, user *user_model.User, gothUser goth.User) error {
    externalLoginUser := toExternalLoginUser(authSourceID, user, gothUser)
    return user_model.EnsureLinkExternalToUser(ctx, externalLoginUser)
}
```

---

## 3. 会话写入机制

### 3.1 updateSession 核心函数

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
        if err := sess.Delete(k); err != nil { ... }
    }

    // 3. 设置新的 session 键值
    for k, v := range updates {
        if err := sess.Set(k, v); err != nil { ... }
    }

    // 4. 持久化 session
    if err := sess.Release(); err != nil { ... }

    return nil
}
```

### 3.2 正常登录时的会话写入

**文件**: `routers/web/auth/oauth.go:407-414`

```go
func handleOAuth2SignIn(...) {
    // ... 2FA 检查 ...

    if !needs2FA {
        // 更新用户最后登录时间
        opts.SetLastLogin = true
        user_service.UpdateUser(ctx, u, opts)

        // 写入会话
        updateSession(ctx, nil, map[string]any{
            session.KeyUID:                  u.ID,      // "uid"
            session.KeyUname:                u.Name,    // "uname"
            session.KeyUserHasTwoFactorAuth: userHasTwoFactorAuth,
        })

        resetLocale(ctx, u)
        redirectAfterAuth(ctx)
        return
    }
}
```

**会话键说明** (定义于 `modules/session/key.go`):
- `KeyUID = "uid"`: 用户 ID
- `KeyUname = "uname"`: 用户名
- `KeyUserHasTwoFactorAuth`: 用户是否启用了双因素认证

### 3.3 通用登录 handleSignIn

**文件**: `routers/web/auth/auth.go:376-425`

```go
func handleSignIn(ctx *context.Context, u *user_model.User, remember bool) {
    handleSignInFull(ctx, u, remember)
    redirectAfterAuth(ctx)
}

func handleSignInFull(ctx *context.Context, u *user_model.User, remember bool) {
    // 1. 设置 remember me cookie（如果勾选）
    if remember {
        nt, token, err := auth_service.CreateAuthTokenForUserID(ctx, u.ID)
        ctx.SetSiteCookie(setting.CookieRememberName, nt.ID+":"+token, ...)
    }

    // 2. 检查 2FA 状态
    userHasTwoFactorAuth, err := auth.HasTwoFactorOrWebAuthn(ctx, u.ID)

    // 3. 清除旧的认证相关 session，写入新的
    updateSession(ctx, []string{
        "openid_verified_uri",
        "openid_signin_remember",
        "twofaUid",
        "twofaRemember",
        "linkAccount",
        "linkAccountData",  // 清除绑定数据
    }, map[string]any{
        session.KeyUID:                  u.ID,
        session.KeyUname:                u.Name,
        session.KeyUserHasTwoFactorAuth: userHasTwoFactorAuth,
    })

    // 4. 设置语言
    resetLocale(ctx, u)
}
```

### 3.4 2FA 场景下的会话

**文件**: `routers/web/auth/oauth.go:432-448`

```go
if needs2FA {
    // 保存用户 ID 和 remember 选项到 session
    updateSession(ctx, nil, map[string]any{
        "twofaUid":      u.ID,
        "twofaRemember": false,
    })

    // 跳转到 2FA 或 WebAuthn 页面
    regs, err := auth.GetWebAuthnCredentialsByUID(ctx, u.ID)
    if err == nil && len(regs) > 0 {
        ctx.Redirect(setting.AppSubURL + "/user/webauthn")
    } else {
        ctx.Redirect(setting.AppSubURL + "/user/two_factor")
    }
}
```

2FA 验证通过后，会从 session 读取 `twofaUid`，完成最终登录。

---

## 4. 关键接续点总结

### 4.1 回调 → 查找用户
```
SignInOAuthCallback()
    ↓
oAuth2UserLoginCallback() → 返回 (user, gothUser, error)
    ├─ 找到 user → handleOAuth2SignIn() → 登录
    └─ 未找到 user → 进入绑定/注册流程
```

### 4.2 绑定流程接续
```
showLinkingLogin() 保存 LinkAccountData 到 session
    ↓
用户跳转到 /user/link_account
    ├─ 登录 → LinkAccountPostSignIn() → 从 session 读回 LinkAccountData
    └─ 注册 → LinkAccountPostRegister() → 从 session 读回 LinkAccountData
```

### 4.3 登录会话接续
```
认证通过
    ↓
updateSession() 执行：
    1. RegenerateSession()  - 重新生成 session ID
    2. 清理旧的认证数据     - 删除 linkAccountData 等
    3. 写入 uid / uname     - 建立登录态
    4. Release()            - 持久化
    ↓
后续请求通过 session 中的 uid 识别用户
```

### 4.4 2FA 接续
```
需要 2FA
    ↓
updateSession() 写入 twofaUid
    ↓
跳转到 2FA 页面
    ↓
2FA 验证通过
    ↓
从 session 读 twofaUid，调用 handleSignIn() 完成登录
```

---

## 5. 关键代码位置速查

| 功能 | 文件 | 函数 |
|------|------|------|
| OAuth2 回调入口 | `routers/web/auth/oauth.go` | `SignInOAuthCallback` |
| 用户查找逻辑 | `routers/web/auth/oauth.go` | `oAuth2UserLoginCallback` |
| 登录处理（含会话写入） | `routers/web/auth/oauth.go` | `handleOAuth2SignIn` |
| 绑定页面展示 | `routers/web/auth/linkaccount.go` | `LinkAccount` |
| 绑定已有账号 | `routers/web/auth/linkaccount.go` | `LinkAccountPostSignIn` |
| 注册新账号 | `routers/web/auth/linkaccount.go` | `LinkAccountPostRegister` |
| 外部账号关联 | `services/externalaccount/user.go` | `LinkAccountToUser` / `EnsureLinkExternalToUser` |
| 通用登录处理 | `routers/web/auth/auth.go` | `handleSignIn` / `handleSignInFull` |
| 会话更新核心 | `routers/web/auth/auth.go` | `updateSession` |
| 会话键定义 | `modules/session/key.go` | `KeyUID` / `KeyUname` |
| 用户信息同步 | `routers/web/auth/oauth_signin_sync.go` | `oauth2SignInSync` |
