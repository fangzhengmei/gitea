# 自动合并全链路复核报告

## 概述

本文档对 Gitea 自动合并（Auto Merge）从**调度登记** → **队列执行** → **钩子落地**的完整链路进行深度复核，重点校正：
1. 签名检查在各阶段的生效情况
2. 依赖检查在各阶段的生效情况
3. 管理员在钩子层绕过分支保护时与 `BlockAdminMergeOverride` 配置的冲突边界

---

## 一、自动合并完整链路全景

### 1.1 链路阶段划分

自动合并流程分为三个明确阶段：

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  调度登记阶段   │───▶│  队列执行阶段   │───▶│  钩子落地阶段   │
│ (用户点击触发)  │    │ (后台异步处理)  │    │ (git push 触发) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 1.2 关键代码路径索引

| 阶段 | 主要文件 | 核心函数 |
|------|----------|----------|
| 调度登记 | `routers/web/repo/pull.go` / `routers/api/v1/repo/pull.go` | `MergeWhenChecksSucceed` 表单处理 |
| 调度登记 | `services/pull/check.go` | `CheckPullMergeable(mergeCheckType=Auto)` |
| 调度登记 | `services/automerge/automerge.go` | `ScheduleAutoMerge` |
| 队列执行 | `services/automerge/automerge.go` | `handlePullRequestAutoMerge` |
| 队列执行 | `services/pull/check.go` | `CheckPullMergeable(mergeCheckType=General, adminForceMerge=false)` |
| 合并执行 | `services/pull/merge.go` | `Merge` |
| 钩子落地 | `routers/private/hook_pre_receive.go` | `preReceiveBranch` |

---

## 二、各阶段检查项详细复核

### 2.1 调度登记阶段

**入口**：用户在 Web/API 点击 "Merge When Checks Succeed"

#### 2.1.1 CheckPullMergeable 调用参数

```go
// routers/web/repo/pull.go:1049
pull_service.CheckPullMergeable(
    ctx, 
    ctx.Doer, 
    &ctx.Repo.Permission, 
    pr, 
    pull_service.MergeCheckTypeAuto,  // 关键：使用 Auto 类型
    repo_model.MergeStyle(form.Do), 
    form.ForceMerge
)
```

#### 2.1.2 检查执行流程

在 `services/pull/check.go:142-228` 中，`MergeCheckTypeAuto` 的行为：

```go
if err := CheckPullBranchProtections(ctx, pr, false); err != nil {
    if !errors.Is(err, ErrNotReadyToMerge) {
        return err
    }
    
    // * when doing Auto Merge (Scheduled Merge After Checks Succeed), skip the branch protection check
    if mergeCheckType == MergeCheckTypeAuto {
        err = nil  // 清除分支保护检查错误
    }
    // ...
}

// 以下检查不受 mergeCheckType 影响，始终执行：
if err := checkSigningRequirements(ctx, pr, doer, mergeStyle); err != nil {
    return err  // ✅ 签名检查始终执行
}

if noDeps, err := issues_model.IssueNoDependenciesLeft(ctx, pr.Issue); err != nil {
    return err
} else if !noDeps {
    return ErrDependenciesLeft  // ✅ 依赖检查始终执行
}
```

**关键校正**：
- ❌ **分支保护检查**（状态、评审、受保护文件、过时分支）：被跳过
- ✅ **签名检查**：**始终执行**，不受 `MergeCheckTypeAuto` 影响
- ✅ **依赖检查**：**始终执行**，不受 `MergeCheckTypeAuto` 影响
- ✅ **基础检查**（已合并、已关闭、WIP、可合并状态、检查中）：始终执行
- ✅ **合并权限检查**：始终执行

#### 2.1.3 ScheduleAutoMerge 登记

`services/automerge/automerge.go:59-76`：
```go
func ScheduleAutoMerge(ctx context.Context, doer *user_model.User, pull *issues_model.PullRequest, style repo_model.MergeStyle, message string, deleteBranchAfterMerge bool) (scheduled bool, err error) {
    err = db.WithTx(ctx, func(ctx context.Context) error {
        if err := pull_model.ScheduleAutoMerge(ctx, doer, pull.ID, style, message, deleteBranchAfterMerge); err != nil {
            return err
        }
        _, err = issues_model.CreateAutoMergeComment(ctx, issues_model.CommentTypePRScheduledToAutoMerge, pull, doer)
        return err
    })
    // ...
    automergequeue.StartPRCheckAndAutoMerge(ctx, pull)
}
```

**关键点**：
- 仅写入数据库和创建评论，不做额外检查
- 立即触发一次自动合并队列检查

#### 2.1.4 调度阶段检查矩阵

| 检查项 | 是否执行 | 说明 |
|--------|----------|------|
| 合并权限 | ✅ | `IsUserAllowedToMerge` |
| 基础状态（已合并/关闭/WIP等） | ✅ | 始终执行 |
| 签名要求 | ✅ | `checkSigningRequirements` 始终执行 |
| 依赖检查 | ✅ | `IssueNoDependenciesLeft` 始终执行 |
| 状态检查（CI/CD） | ❌ | 被 `MergeCheckTypeAuto` 跳过 |
| 评审要求 | ❌ | 被 `MergeCheckTypeAuto` 跳过 |
| 受保护文件 | ❌ | 被 `MergeCheckTypeAuto` 跳过 |
| 过时分支 | ❌ | 被 `MergeCheckTypeAuto` 跳过 |

---

### 2.2 队列执行阶段

**入口**：自动合并队列触发 `handlePullRequestAutoMerge`

#### 2.2.1 执行前预处理

`services/automerge/automerge.go:157-239`：
```go
func handlePullRequestAutoMerge(pullID int64, sha string) {
    // 1. 加载 PR 信息
    pr, err := issues_model.GetPullRequestByID(ctx, pullID)
    
    // 2. 验证调度记录存在
    exists, scheduledPRM, err := pull_model.GetScheduledMergeByPullID(ctx, pr.ID)
    
    // 3. 验证 head commit SHA 匹配（防止并发问题）
    headCommitID, err := baseGitRepo.GetRefCommitID(pr.GetGitHeadRefName())
    if headCommitID != sha {
        return  // SHA 不匹配，忽略此次请求
    }
    
    // 4. 检查 head 分支存在
    // ...
    
    // 5. 检查状态检查是否通过
    pass, err := pull_service.IsPullCommitStatusPass(ctx, pr)
    if !pass {
        return  // 状态检查未通过，等待下一次
    }
```

**关键点**：
- 队列执行前会先单独调用 `IsPullCommitStatusPass` 检查状态
- 这是调度阶段跳过的检查在执行前的补全

#### 2.2.2 CheckPullMergeable 调用参数

```go
// services/automerge/automerge.go:254
pull_service.CheckPullMergeable(
    ctx, 
    doer, 
    &perm, 
    pr, 
    pull_service.MergeCheckTypeGeneral,  // 关键：使用 General 类型
    scheduledPRM.MergeStyle, 
    false  // adminForceMerge = false
)
```

**关键校正**：
- 使用 `MergeCheckTypeGeneral` 而非 `MergeCheckTypeAuto`
- `adminForceMerge` 硬编码为 `false`，不允许管理员强制合并

#### 2.2.3 检查执行流程

由于使用 `MergeCheckTypeGeneral`，所有检查完整执行：
- ✅ 基础状态检查
- ✅ 合并权限检查
- ✅ 分支保护检查（状态、评审、受保护文件、过时分支）
- ✅ 签名检查
- ✅ 依赖检查

#### 2.2.4 队列执行阶段检查矩阵

| 检查项 | 是否执行 | 说明 |
|--------|----------|------|
| 合并权限 | ✅ | 重新检查（可能与调度时权限不同） |
| 基础状态 | ✅ | 始终执行 |
| 签名要求 | ✅ | 重新检查 |
| 依赖检查 | ✅ | 重新检查 |
| 状态检查 | ✅ | 先单独检查，再在 CheckPullMergeable 中检查 |
| 评审要求 | ✅ | `HasEnoughApprovals` 等 |
| 受保护文件 | ✅ | `MergeBlockedByProtectedFiles` |
| 过时分支 | ✅ | `MergeBlockedByOutdatedBranch` |
| 管理员强制合并 | ❌ | `adminForceMerge` 硬编码为 false |

---

### 2.3 钩子落地阶段

**入口**：`Merge` 函数执行 `git push` 触发 `pre-receive` 钩子

#### 2.3.1 钩子层检查流程

`routers/private/hook_pre_receive.go:142-404` 中，PR 合并时的检查：

```go
func preReceiveBranch(ctx *preReceiveContext, oldCommitID, newCommitID string, refFullName git.RefName) {
    // ...
    
    // 3. Enforce require signed commits
    if protectBranch.RequireSignedCommits {
        err := verifyCommits(oldCommitID, newCommitID, gitRepo, ctx.env)
        // 签名检查失败直接返回错误
        // ✅ 签名检查在所有绕过逻辑之前执行
    }
    
    // 4. 受保护文件检查
    // ...
    
    // 5. 推送权限检查
    if !canPush {
        if ctx.opts.PullRequestID == 0 {
            // 非 PR 合并的推送检查
        } else {
            // PR 合并检查
            pr, err := issues_model.GetPullRequestByID(ctx, ctx.opts.PullRequestID)
            
            // 检查合并权限
            allowedMerge, err := pull_service.IsUserAllowedToMerge(ctx, pr, ctx.userPerm, ctx.user)
            
            // If we're an admin for the repository we can ignore status checks, reviews and override protected files
            if ctx.userPerm.IsAdmin() {
                return  // ⚠️ 管理员直接返回，跳过所有后续检查
            }
            
            // 非管理员才执行 CheckPullBranchProtections
            if err := pull_service.CheckPullBranchProtections(ctx, pr, true); err != nil {
                // 检查失败返回错误
            }
        }
    }
}
```

#### 2.3.2 钩子层检查执行顺序

**关键发现**：签名检查的执行位置在所有绕过逻辑之前！

```
钩子执行顺序：
1. 写入权限检查
2. 分支删除保护
3. 强制推送保护
4. ✅ 签名提交验证（RequireSignedCommits）← 此处执行，管理员无法绕过！
5. 受保护文件检查
6. 推送权限检查
7. 合并权限检查
8. ⚠️ 管理员豁免（IsAdmin()）← 此处才跳过后续检查
9. 非管理员：CheckPullBranchProtections（状态、评审等）
```

#### 2.3.3 钩子层检查矩阵

| 检查项 | 普通用户 | 管理员 | 说明 |
|--------|----------|--------|------|
| 写入权限 | ✅ | ✅ | 基础权限 |
| 分支删除保护 | ✅ | ✅ | 阻止删除受保护分支 |
| 强制推送保护 | ✅ | ✅ | 除非规则明确允许 |
| 签名要求 | ✅ | ✅ | **在管理员豁免之前执行，管理员也无法绕过** |
| 受保护文件 | ✅ | ❌ | 管理员豁免后跳过 |
| 合并权限 | ✅ | ✅ | 仍需检查 |
| 状态检查 | ✅ | ❌ | 管理员豁免后跳过 |
| 评审要求 | ✅ | ❌ | 管理员豁免后跳过 |
| 过时分支 | ✅ | ❌ | 管理员豁免后跳过 |

---

## 三、管理员绕过与 BlockAdminMergeOverride 的冲突边界

### 3.1 BlockAdminMergeOverride 配置说明

字段定义在 `models/git/protected_branch.go:65`：
```go
BlockAdminMergeOverride bool `xorm:"NOT NULL DEFAULT false"`
```

**设计意图**：当设置为 `true` 时，阻止仓库管理员使用 "Force Merge" 功能绕过分支保护规则。

### 3.2 UI/API 层的生效逻辑

在 `services/pull/check.go:193-209`：
```go
if adminForceMerge {
    isRepoAdmin, errForceMerge := access_model.IsUserRepoAdmin(ctx, pr.BaseRepo, doer)
    // ...
    protectedBranchRule, errForceMerge := git_model.GetFirstMatchProtectedBranchRule(ctx, pr.BaseRepoID, pr.BaseBranch)
    // ...
    blockAdminForceMerge := protectedBranchRule != nil && protectedBranchRule.BlockAdminMergeOverride
    if isRepoAdmin && !blockAdminForceMerge {
        err = nil  // 仅当 BlockAdminMergeOverride 为 false 时才允许绕过
    }
}
```

**UI/API 层行为**：
- `BlockAdminMergeOverride=true` → 管理员不能使用 Force Merge 绕过
- `BlockAdminMergeOverride=false` → 管理员可以使用 Force Merge 绕过

### 3.3 钩子层的实际行为

在 `routers/private/hook_pre_receive.go:374-377`：
```go
// If we're an admin for the repository we can ignore status checks, reviews and override protected files
if ctx.userPerm.IsAdmin() {
    return  // 无条件返回，不检查 BlockAdminMergeOverride！
}
```

**关键校正**：钩子层的管理员豁免是**无条件**的，`BlockAdminMergeOverride` 配置在钩子层**完全不生效**！

### 3.4 冲突边界分析

| 场景 | UI/API 层（CheckPullMergeable） | 钩子层（preReceiveBranch） | 实际结果 |
|------|--------------------------------|-----------------------------|----------|
| 管理员 Force Merge，`BlockAdminMergeOverride=true` | ❌ 阻止 | ✅ 放行 | 不一致：UI 阻止但可通过 git push 绕过 |
| 管理员 Force Merge，`BlockAdminMergeOverride=false` | ✅ 放行 | ✅ 放行 | 一致 |
| 自动合并（doer 是管理员），`BlockAdminMergeOverride=true` | ✅ 执行完整检查（adminForceMerge=false） | ✅ 管理员豁免 | 不一致：队列执行时检查通过，但钩子层直接放行 |
| 普通用户合并 | ✅ 完整检查 | ✅ 完整检查 | 一致 |

### 3.5 风险场景演示

**场景 1：UI 显示被阻止，但实际可绕过**
```
1. 仓库设置：BlockAdminMergeOverride = true
2. 管理员在 UI 点击 Force Merge
3. UI 层 CheckPullMergeable 阻止操作，显示错误
4. 管理员通过 git push 直接推送合并结果
5. 钩子层检查到是管理员，无条件放行
6. 结果：合并成功，绕过了所有分支保护规则
```

**场景 2：自动合并中管理员豁免**
```
1. 仓库设置：BlockAdminMergeOverride = true，RequireSignedCommits = true
2. 管理员 A 调度自动合并（签名检查在调度时已通过）
3. 其他用户 B 推送了未签名的新提交，导致签名检查失败
4. 自动合并队列执行：CheckPullMergeable 发现签名检查失败，本应阻止
5. 但 Merge 执行 git push 到钩子层
6. 钩子层检查：签名验证在管理员豁免之前，所以未签名提交仍会被阻止！
7. 结果：签名检查被强制执行，管理员也无法绕过
```

---

## 四、签名检查全链路穿透分析

### 4.1 签名检查的三个执行点

| 执行点 | 位置 | 检查逻辑 | 管理员能否绕过 |
|--------|------|----------|----------------|
| 调度登记 | `CheckPullMergeable` → `checkSigningRequirements` | 根据合并策略检查 | ❌ 不能绕过（始终执行） |
| 队列执行 | `CheckPullMergeable` → `checkSigningRequirements` | 根据合并策略检查 | ❌ 不能绕过（始终执行，adminForceMerge=false） |
| 钩子落地 | `preReceiveBranch` → `verifyCommits` | 验证所有推送的 commit | ❌ 不能绕过（在管理员豁免之前执行） |

**关键结论**：签名检查在自动合并全链路中**始终生效**，管理员在任何阶段都无法绕过签名要求。

### 4.2 不同合并策略的签名检查差异

`checkSigningRequirements` 函数（`services/pull/check.go:239-270`）：

| 合并策略 | 检查要求 |
|----------|----------|
| Fast-Forward Only | 所有头部提交必须已验证签名 |
| Merge | 头部提交已验证 + Gitea 能签名合并提交 |
| Rebase/Rebase-Merge/Squash | Gitea 能签名重写后的提交 |

**注意**：钩子层的 `verifyCommits` 会验证所有推送的 commit，与合并策略无关。

---

## 五、依赖检查全链路穿透分析

### 5.1 依赖检查的两个执行点

| 执行点 | 位置 | 检查逻辑 | 管理员能否绕过 |
|--------|------|----------|----------------|
| 调度登记 | `CheckPullMergeable` → `IssueNoDependenciesLeft` | 检查 PR 所有依赖是否已关闭 | ❌ 不能绕过（始终执行） |
| 队列执行 | `CheckPullMergeable` → `IssueNoDependenciesLeft` | 检查 PR 所有依赖是否已关闭 | ❌ 不能绕过（始终执行） |
| 钩子落地 | 无 | 不执行 | - |

**关键校正**：依赖检查**不在钩子层执行**。但由于：
1. 自动合并队列执行时 `adminForceMerge=false`
2. 普通合并时如果 Force Merge 被 `BlockAdminMergeOverride` 阻止

因此依赖检查在大多数场景下是有效的。

### 5.2 依赖检查绕过的唯一可能

只有当管理员通过直接 git push 的方式合并 PR 时，才能绕过依赖检查：
- 绕过 UI/API 层的 `CheckPullMergeable`
- 钩子层不执行 `IssueNoDependenciesLeft` 检查

---

## 六、复核结论与风险矩阵

### 6.1 签名检查复核结论

✅ **签名检查在自动合并全链路始终生效**
- 调度登记：执行
- 队列执行：执行
- 钩子落地：执行（且在管理员豁免之前）

### 6.2 依赖检查复核结论

⚠️ **依赖检查在 UI/API 层生效，但钩子层不执行**
- 调度登记：执行
- 队列执行：执行
- 钩子落地：不执行
- 风险：管理员通过直接 git push 可绕过

### 6.3 BlockAdminMergeOverride 复核结论

❌ **BlockAdminMergeOverride 仅在 UI/API 层生效，钩子层完全不生效**
- 配置为 true 时，UI 阻止管理员 Force Merge
- 但管理员仍可通过直接 git push 绕过所有分支保护（签名检查除外）

### 6.4 最终风险矩阵

| 检查项 | 调度阶段 | 队列执行 | 钩子层 | 管理员能否绕过 |
|--------|----------|----------|--------|----------------|
| 签名要求 | ✅ | ✅ | ✅ | ❌ 不能 |
| 依赖检查 | ✅ | ✅ | ❌ | ⚠️ 通过直接 push 可绕过 |
| 状态检查 | ❌ | ✅ | ⚠️ 管理员可绕过 | ⚠️ 钩子层管理员可绕过 |
| 评审要求 | ❌ | ✅ | ⚠️ 管理员可绕过 | ⚠️ 钩子层管理员可绕过 |
| 受保护文件 | ❌ | ✅ | ⚠️ 管理员可绕过 | ⚠️ 钩子层管理员可绕过 |
| 过时分支 | ❌ | ✅ | ⚠️ 管理员可绕过 | ⚠️ 钩子层管理员可绕过 |

---

## 七、代码位置速查表

| 复核点 | 文件路径 | 行号 |
|--------|----------|------|
| 自动合并调度时跳过分支保护 | `services/pull/check.go` | 187-190 |
| 自动合并调度时仍检查签名 | `services/pull/check.go` | 217-219 |
| 自动合并调度时仍检查依赖 | `services/pull/check.go` | 221-225 |
| 自动合并队列执行时完整检查 | `services/automerge/automerge.go` | 254 |
| 钩子层签名检查位置 | `routers/private/hook_pre_receive.go` | 223-241 |
| 钩子层管理员豁免位置 | `routers/private/hook_pre_receive.go` | 374-377 |
| 钩子层不检查 BlockAdminMergeOverride | `routers/private/hook_pre_receive.go` | 374-377 |
| BlockAdminMergeOverride 字段 | `models/git/protected_branch.go` | 65 |
| UI 层 BlockAdminMergeOverride 检查 | `services/pull/check.go` | 204-208 |
