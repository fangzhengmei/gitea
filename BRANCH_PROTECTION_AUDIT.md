# 分支保护合并链路审计报告

## 概述

本文档审计 Gitea 分支保护规则在合并请求闭合过程中的各类绕过路径，重点分析**网页合并校验**与**推送钩子校验**之间的差异，以及**手动标记合并**、**自动合并**、**管理员覆盖**三种特殊场景在不同入口的生效边界。

---

## 一、核心检查入口对比

### 1.1 检查层级分布

Gitea 采用**双重检查机制**，在两个独立层级执行分支保护校验：

| 检查层级 | 执行时机 | 主要文件 | 核心函数 |
|----------|----------|----------|----------|
| UI/API 层 | 用户点击合并按钮时 | `services/pull/check.go` | `CheckPullMergeable` |
| 钩子层 | Git 实际推送前 | `routers/private/hook_pre_receive.go` | `preReceiveBranch` |

### 1.2 两层检查的差异分析

#### UI/API 层检查（`CheckPullMergeable`）

**检查内容**：
- 基础状态检查（已合并、已关闭、WIP、可合并状态等）
- 合并权限检查
- 分支保护规则检查（状态、评审、受保护文件、过时分支）
- 签名要求检查
- 依赖检查

**绕过机制**：
- `MergeCheckTypeManually`：跳过所有检查
- `MergeCheckTypeAuto`：跳过分支保护检查
- `adminForceMerge=true`：管理员可绕过（受 `BlockAdminMergeOverride` 限制）

#### 钩子层检查（`preReceiveBranch`）

**检查内容**：
- 写入权限检查
- 默认分支删除保护
- 分支删除保护
- 强制推送保护
- 签名提交验证
- 受保护文件检查
- 推送权限检查
- PR 合并时的分支保护检查

**绕过机制**：
- 管理员权限：直接跳过所有分支保护检查（**无条件**）
- 仅修改未保护文件：允许推送

---

## 二、三类绕过路径的详细分析

### 2.1 手动标记合并（Manually Merged）

#### 入口代码路径

**Web 入口**：`routers/web/repo/pull.go:1038-1097`
```go
manuallyMerged := repo_model.MergeStyle(form.Do) == repo_model.MergeStyleManuallyMerged
mergeCheckType := pull_service.MergeCheckTypeGeneral
if manuallyMerged {
    mergeCheckType = pull_service.MergeCheckTypeManually
}
```

**API 入口**：`routers/api/v1/repo/pull.go:957-1007`（逻辑同上）

#### UI/API 层行为

在 `CheckPullMergeable` 中（`services/pull/check.go:162-165`）：
```go
if mergeCheckType == MergeCheckTypeManually {
    // if doer is doing "manually merge" (mark as merged manually), do not check anything
    return nil
}
```

**关键发现**：只要用户有合并权限，`MergeCheckTypeManually` 会**跳过所有检查**，包括：
- 分支保护规则（状态检查、评审要求、受保护文件等）
- 签名要求检查
- 依赖检查

#### 执行流程

手动标记合并不经过 Git 推送流程，因此**不触发 pre-receive 钩子**：

```
用户点击 "Manually Merge"
    │
    ▼
CheckPullMergeable(mergeCheckType=Manually)
    │  → 跳过所有检查
    ▼
MergedManually()
    │
    ├─ 检查手动合并是否在仓库配置中允许
    ├─ 验证 commit ID 是否存在且在目标分支中
    └─ 直接更新数据库标记 PR 为已合并
    │
    ▼
完成（无 git push，无钩子触发）
```

#### 生效边界

| 检查项 | 是否生效 | 说明 |
|--------|----------|------|
| 合并权限 | ✅ | 仍需通过 `IsUserAllowedToMerge` 检查 |
| 手动合并功能开关 | ✅ | 仓库需允许手动合并 |
| Commit ID 验证 | ✅ | 必须是目标分支中存在的合法 commit |
| 状态检查 | ❌ | 完全跳过 |
| 评审要求 | ❌ | 完全跳过 |
| 签名要求 | ❌ | 完全跳过 |
| 受保护文件 | ❌ | 完全跳过 |
| 钩子层检查 | ❌ | 不触发钩子 |

#### 风险点

- **审计绕过**：手动标记合并的 commit 可能未经过任何 CI/CD 检查和代码评审
- **签名绕过**：即使分支要求签名提交，手动合并也不验证
- **无钩子执行**：pre-receive 钩子中的自定义逻辑完全被跳过

---

### 2.2 自动合并（Auto Merge / Merge When Checks Succeed）

#### 入口代码路径

**Web 入口**：`routers/web/repo/pull.go:1041-1042`
```go
if form.MergeWhenChecksSucceed {
    mergeCheckType = pull_service.MergeCheckTypeAuto
}
```

**API 入口**：`routers/api/v1/repo/pull.go:960-962`（逻辑同上）

#### 调度阶段行为

在 `CheckPullMergeable` 中（`services/pull/check.go:187-190`）：
```go
// * when doing Auto Merge (Scheduled Merge After Checks Succeed), skip the branch protection check
if mergeCheckType == MergeCheckTypeAuto {
    err = nil
}
```

**调度时的检查**：
- ✅ 基础状态检查（已合并、已关闭、WIP 等）
- ✅ 合并权限检查
- ❌ 分支保护检查（被跳过）
- ❌ 签名要求检查（被跳过）
- ❌ 依赖检查（被跳过）

调度成功后，PR 进入等待状态，直到所有检查通过后由自动合并队列处理。

#### 自动合并队列执行阶段

在 `services/automerge/automerge.go:254`：
```go
if err := pull_service.CheckPullMergeable(ctx, doer, &perm, pr, pull_service.MergeCheckTypeGeneral, scheduledPRM.MergeStyle, false); err != nil {
```

**关键发现**：自动合并队列执行时，使用 `MergeCheckTypeGeneral` 而非 `MergeCheckTypeAuto`，因此会执行**完整的分支保护检查**。

#### 完整执行流程

```
用户点击 "Merge When Checks Succeed"
    │
    ▼
CheckPullMergeable(mergeCheckType=Auto)
    │  → 跳过分支保护检查（仅做基础检查）
    ▼
ScheduleAutoMerge()
    │
    ├─ 写入 scheduled_merge 表
    └─ 添加到自动合并队列
    │
    ▼
[ 等待所有状态检查通过 ]
    │
    ▼
自动合并队列触发 handlePullRequestAutoMerge()
    │
    ├─ 验证 head commit SHA 匹配
    ├─ 检查 IsPullCommitStatusPass
    └─ CheckPullMergeable(mergeCheckType=General, adminForceMerge=false)
        │  → 完整检查所有分支保护规则
        ▼
    Merge()
        │
        └─ git push → 触发 pre-receive 钩子
            │
            ▼
        preReceiveBranch()
            ├─ 检查合并权限
            ├─ 管理员可跳过所有检查
            └─ CheckPullBranchProtections (非管理员)
```

#### 生效边界

| 检查项 | 调度时 | 执行时（队列） | 钩子层 |
|--------|--------|----------------|--------|
| 合并权限 | ✅ | ✅ | ✅ |
| 基础状态 | ✅ | ✅ | - |
| 状态检查 | ❌ | ✅ | ✅ |
| 评审要求 | ❌ | ✅ | ✅ |
| 签名要求 | ❌ | ✅ | ✅ |
| 受保护文件 | ❌ | ✅ | ✅ |
| 管理员覆盖 | - | ❌ | ✅（无条件） |

#### 风险点

- **调度时权限滥用**：用户在调度时可能满足权限条件，但执行时权限已被撤销（但执行时会重新检查）
- **管理员钩子豁免**：即使分支保护设置了 `BlockAdminMergeOverride`，钩子层管理员仍可无条件绕过

---

### 2.3 管理员覆盖（Admin Force Merge）

#### 入口代码路径

**Web 入口**：`routers/web/repo/pull.go:1049`
```go
if err := pull_service.CheckPullMergeable(ctx, ctx.Doer, &ctx.Repo.Permission, pr, mergeCheckType, repo_model.MergeStyle(form.Do), form.ForceMerge); err != nil {
```

**API 入口**：`routers/api/v1/repo/pull.go:968`（逻辑同上）

#### UI/API 层行为

在 `CheckPullMergeable` 中（`services/pull/check.go:193-209`）：
```go
if adminForceMerge {
    isRepoAdmin, errForceMerge := access_model.IsUserRepoAdmin(ctx, pr.BaseRepo, doer)
    // ...
    protectedBranchRule, errForceMerge := git_model.GetFirstMatchProtectedBranchRule(ctx, pr.BaseRepoID, pr.BaseBranch)
    // ...
    blockAdminForceMerge := protectedBranchRule != nil && protectedBranchRule.BlockAdminMergeOverride
    if isRepoAdmin && !blockAdminForceMerge {
        err = nil
    }
}
```

**UI/API 层的管理员覆盖受 `BlockAdminMergeOverride` 限制**。

#### 钩子层行为

在 `preReceiveBranch` 中（`routers/private/hook_pre_receive.go:374-377`）：
```go
// If we're an admin for the repository we can ignore status checks, reviews and override protected files
if ctx.userPerm.IsAdmin() {
    return
}
```

**关键发现**：钩子层的管理员豁免是**无条件**的，不检查 `BlockAdminMergeOverride`！

#### 生效边界对比

| 检查层级 | `BlockAdminMergeOverride=true` | `BlockAdminMergeOverride=false` |
|----------|--------------------------------|---------------------------------|
| UI/API 层 | ❌ 管理员不能绕过 | ✅ 管理员可以绕过 |
| 钩子层 | ✅ 管理员仍可绕过 | ✅ 管理员可以绕过 |

#### 完整执行流程

```
管理员勾选 "Force Merge" 并点击合并
    │
    ▼
CheckPullMergeable(adminForceMerge=true)
    │
    ├─ 检查是否为仓库管理员
    ├─ 检查 BlockAdminMergeOverride
    │   ├─ true：阻止绕过，返回错误
    │   └─ false：允许绕过，清除错误
    ▼
Merge()
    │
    └─ git push → 触发 pre-receive 钩子
        │
        ▼
    preReceiveBranch()
        │
        ├─ 检查是否为管理员
        │   └─ 是：直接返回，跳过所有分支保护检查（忽略 BlockAdminMergeOverride！）
        └─ 非管理员：执行完整检查
```

#### 风险点

- **`BlockAdminMergeOverride` 不完全生效**：该设置仅在 UI/API 层生效，钩子层管理员仍可无条件绕过
- **不一致的安全模型**：用户在 UI 上看到 "Force Merge 被阻止"，但实际上管理员仍可通过直接推送绕过
- **审计盲点**：管理员的强制合并在钩子层不留下任何"被阻止"的日志

---

## 三、两层检查的绕过路径矩阵

| 绕过方式 | UI/API 层 | 钩子层 | 备注 |
|----------|-----------|--------|------|
| 手动标记合并 | ✅ 完全绕过 | ❌ 不触发 | 直接数据库操作，无 git push |
| 自动合并（调度时） | ✅ 跳过分支保护 | - | 执行时会重新检查 |
| 自动合并（执行时） | ❌ 完整检查 | ⚠️ 管理员可绕过 | |
| 管理员 Force Merge | ⚠️ 受 BlockAdminMergeOverride 限制 | ✅ 无条件绕过 | 两层行为不一致 |
| 仅修改未保护文件 | ❌ 仍需检查 | ✅ 允许推送 | 仅钩子层有此逻辑 |
| 部署密钥白名单 | - | ✅ 可配置 | 仅钩子层检查 |

---

## 四、关键风险点汇总

### 4.1 手动标记合并风险

1. **完全绕过所有保护**：包括 CI/CD 检查、代码评审、签名验证
2. **无钩子执行**：pre-receive 钩子中的自定义安全逻辑无法执行
3. **Commit 验证不充分**：仅验证 commit 存在于目标分支，不验证其合规性

### 4.2 自动合并风险

1. **调度与执行的时间差**：调度时有权限但执行时可能已无权限（但执行时会重新检查）
2. **管理员钩子豁免**：即使设置了 `BlockAdminMergeOverride`，自动合并执行时如果 doer 是管理员，钩子层仍会放行

### 4.3 管理员覆盖风险

1. **`BlockAdminMergeOverride` 的虚假安全感**：该设置仅阻止 UI 操作，不阻止直接推送
2. **两层检查不一致**：UI 显示被阻止，但实际可通过 git push 绕过
3. **无审计追踪**：钩子层的管理员绕过不记录"本应被阻止"的信息

### 4.4 架构设计风险

1. **双重检查逻辑不一致**：UI/API 层和钩子层使用不同的绕过逻辑
2. **权限检查重复但不统一**：合并权限在两层都检查，但规则不同
3. **可配置项生效边界不清晰**：`BlockAdminMergeOverride` 等配置的作用范围容易产生误解

---

## 五、代码位置索引表

| 功能点 | 文件路径 | 行号 |
|--------|----------|------|
| CheckPullMergeable 主入口 | `services/pull/check.go` | 142-229 |
| MergeCheckTypeManually 跳过 | `services/pull/check.go` | 162-165 |
| MergeCheckTypeAuto 跳过 | `services/pull/check.go` | 187-190 |
| adminForceMerge 逻辑 | `services/pull/check.go` | 193-209 |
| MergedManually 实现 | `services/pull/merge.go` | 617-679 |
| 自动合并队列处理 | `services/automerge/automerge.go` | 157-279 |
| 自动合并执行时检查 | `services/automerge/automerge.go` | 254 |
| pre-receive 钩子主入口 | `routers/private/hook_pre_receive.go` | 109-140 |
| 钩子层 PR 合并检查 | `routers/private/hook_pre_receive.go` | 292-403 |
| 钩子层管理员豁免 | `routers/private/hook_pre_receive.go` | 374-377 |
| BlockAdminMergeOverride 字段 | `models/git/protected_branch.go` | 65 |
