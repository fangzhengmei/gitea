# Gitea PR Merge 检查与状态汇总代码走向

## 一、核心模块概览

| 模块 | 文件路径 | 主要职责 |
|------|---------|---------|
| Merge 核心 | `services/pull/merge.go` | 执行合并操作、合并消息生成 |
| 合并前检查 | `services/pull/check.go` | 合并条件总入口、冲突检测队列 |
| 审查逻辑 | `services/pull/review.go` | Review 提交、驳回、过期处理 |
| CI 状态检查 | `services/pull/commit_status.go` | 提交状态汇总、必填 context 匹配 |
| 分支保护 | `services/pull/protected_branch.go` | 分支保护规则更新后触发 PR 重检 |
| PR 模型 | `models/issues/pull.go` | Review 计数、审批统计、合并阻塞判断 |
| Review 模型 | `models/issues/review.go` | Review 数据结构与基础操作 |
| 分支保护模型 | `models/git/protected_branch.go` | 保护规则数据结构、权限判断 |
| 提交状态模型 | `models/git/commit_status.go` | 提交状态存储与计算 |
| 状态合并逻辑 | `modules/commitstatus/commit_status.go` | 多状态合并算法 |
| Web 入口 | `routers/web/repo/pull.go` | 接收前端合并请求 |

---

## 二、合并检查主调用链

### 2.1 入口：CheckPullMergeable

**位置**: `services/pull/check.go:142`

```go
func CheckPullMergeable(stdCtx context.Context, doer *user_model.User, perm *access_model.Permission,
    pr *issues_model.PullRequest, mergeCheckType MergeCheckType, mergeStyle repo_model.MergeStyle, forceMerge bool) error
```

**检查顺序**：

1. **基础状态检查**（`services/pull/check.go:144-177`）：
   - 是否已合并 (`HasMerged`)
   - 是否已关闭 (`Issue.IsClosed`)
   - 用户是否有合并权限 (`IsUserAllowedToMerge`)
   - 是否为 WIP (`IsWorkInProgress`)
   - 是否处于可合并状态 (`IsStatusMergeable`)
   - 是否正在检查中 (`IsChecking`)

2. **分支保护检查**（`services/pull/check.go:179`）：
   ```go
   if errProtection := CheckPullBranchProtections(ctx, pr, false); errProtection != nil {
       // 处理自动合并跳过、强制合并绕过逻辑
   }
   ```

3. **签名要求检查**（`services/pull/check.go:219`）：
   ```go
   if err := checkSigningRequirements(ctx, pr, doer, mergeStyle); err != nil {
       return err
   }
   ```

4. **依赖检查**（`services/pull/check.go:223`）：
   ```go
   if noDeps, err := issues_model.IssueNoDependenciesLeft(ctx, pr.Issue); err != nil {
       return err
   } else if !noDeps {
       return ErrDependenciesLeft
   }
   ```

---

### 2.2 核心：CheckPullBranchProtections

**位置**: `services/pull/merge.go:570`

```go
func CheckPullBranchProtections(ctx context.Context, pr *issues_model.PullRequest, skipProtectedFilesCheck bool) (err error)
```

**检查顺序与逻辑**：

```
CheckPullBranchProtections
├─ 加载保护规则 (GetFirstMatchProtectedBranchRule)
├─ 1. Status Check 检查 (IsPullCommitStatusPass)
│   └─ 失败 → "Not all required status checks successful"
├─ 2. 审批数量检查 (HasEnoughApprovals)
│   └─ 失败 → "Does not have enough approvals"
├─ 3. 拒绝审查检查 (MergeBlockedByRejectedReview)
│   └─ 失败 → "There are requested changes"
├─ 4. 官方审查请求检查 (MergeBlockedByOfficialReviewRequests)
│   └─ 失败 → "There are official review requests"
├─ 5. 分支过时检查 (MergeBlockedByOutdatedBranch)
│   └─ 失败 → "The head branch is behind the base branch"
└─ 6. 受保护文件检查 (MergeBlockedByProtectedFiles)
    └─ 失败 → "Changed protected files"
```

**任一检查失败** → 包装 `ErrNotReadyToMerge` 错误返回。

---

## 三、Status Check（CI 状态检查）实现

### 3.1 调用链

```
IsPullCommitStatusPass (services/pull/commit_status.go:69)
└─ GetPullRequestCommitStatusState (services/pull/commit_status.go:86)
   ├─ 获取 HEAD commit SHA
   ├─ 获取该 SHA 的所有最新状态 (GetLatestCommitStatus)
   └─ MergeRequiredContextsCommitStatus (services/pull/commit_status.go:22)
      ├─ 编译 required contexts 的 glob 模式
      ├─ 匹配 commit status 的 context 字段
      └─ 调用 CalcCommitStatus 合并状态
```

### 3.2 MergeRequiredContextsCommitStatus 逻辑

**位置**: `services/pull/commit_status.go:22`

**核心规则**：

1. **无状态** → `pending`
2. **无必填 context** → 使用所有状态的合并结果
3. **有必填 context**：
   - 用 glob 匹配每个必填 context
   - 收集所有匹配到的 commit status
   - 如果有必填 context 未匹配到任何状态：
     - 已有匹配状态是 `failure` → 返回 `failure`
     - 否则返回 `pending`（等待未出现的检查）

### 3.3 多状态合并算法 (Combine)

**位置**: `modules/commitstatus/commit_status.go:67`

```go
func (css CommitStatusStates) Combine() CommitStatusState
```

**优先级规则**（按优先级从高到低）：

| 状态 | 优先级 | 说明 |
|------|--------|------|
| `error` / `failure` / `warning` | 最高 | 任一出现 → 整体 `failure` |
| `pending` | 中 | 存在但无更高优先级 → 整体 `pending` |
| `success` / `skipped` | 低 | 全部为此类 → 整体 `success` |

**示例**：
- `[success, pending]` → `pending`
- `[success, error]` → `failure`
- `[success, success, warning]` → `failure`
- `[success, success, success]` → `success`

---

## 四、Review 计数（审批机制）实现

### 4.1 核心数据结构

**位置**: `models/issues/review.go:123`

```go
type Review struct {
    ID         int64
    Type       ReviewType       // Approve / Reject / Comment / Request
    ReviewerID int64
    IssueID    int64
    Official   bool             // 是否为官方审查（计入审批）
    Stale      bool             // 是否因新提交而过期
    Dismissed  bool             // 是否已被驳回
    // ...
}
```

### 4.2 审批数量统计

**位置**: `models/issues/pull.go:761-784`

```go
func HasEnoughApprovals(ctx context.Context, protectBranch *git_model.ProtectedBranch, pr *PullRequest) bool {
    if protectBranch.RequiredApprovals == 0 {
        return true
    }
    return GetGrantedApprovalsCount(ctx, protectBranch, pr) >= protectBranch.RequiredApprovals
}

func GetGrantedApprovalsCount(ctx context.Context, protectBranch *git_model.ProtectedBranch, pr *PullRequest) int64 {
    sess := db.GetEngine(ctx).Where("issue_id = ?", pr.IssueID).
        And("type = ?", ReviewTypeApprove).
        And("official = ?", true).
        And("dismissed = ?", false)
    if protectBranch.IgnoreStaleApprovals {
        sess = sess.And("stale = ?", false)
    }
    approvals, err := sess.Count(new(Review))
    // ...
}
```

**统计条件**：
- `type = Approve`（批准类型）
- `official = true`（官方审查，由审批白名单用户提交）
- `dismissed = false`（未被驳回）
- 可选：`stale = false`（未过期，当 `IgnoreStaleApprovals` 开启时）

### 4.3 其他 Review 相关检查

| 检查函数 | 位置 | 触发条件 |
|---------|------|---------|
| `MergeBlockedByRejectedReview` | `models/issues/pull.go:787` | `BlockOnRejectedReviews=true` 且存在 `type=Reject, official=true, dismissed=false` 的 review |
| `MergeBlockedByOfficialReviewRequests` | `models/issues/pull.go:806` | `BlockOnOfficialReviewRequests=true` 且存在 `type=Request, official=true` 的 review 请求 |
| `MergeBlockedByOutdatedBranch` | `models/issues/pull.go:823` | `BlockOnOutdatedBranch=true` 且 `pr.CommitsBehind > 0` |

### 4.4 Stale Review 机制

**位置**: `services/pull/review.go:317-338`

当用户提交审批/拒绝时：
1. 获取当前 HEAD commit ID
2. 与 review 记录的 commit ID 比较
3. 如不一致，调用 `checkIfPRContentChanged` 检查内容是否真的变化
4. 标记 `stale = true`（如内容变化）

---

## 五、分支保护规则汇总

### 5.1 ProtectedBranch 核心字段

**位置**: `models/git/protected_branch.go:30`

| 字段 | 类型 | 说明 |
|------|------|------|
| `EnableStatusCheck` | `bool` | 是否启用状态检查 |
| `StatusCheckContexts` | `[]string` | 必填的 status check context（支持 glob） |
| `RequiredApprovals` | `int64` | 需要的审批数量 |
| `EnableApprovalsWhitelist` | `bool` | 是否启用审批白名单 |
| `ApprovalsWhitelistUserIDs` | `[]int64` | 审批用户白名单 |
| `ApprovalsWhitelistTeamIDs` | `[]int64` | 审批团队白名单 |
| `BlockOnRejectedReviews` | `bool` | 拒绝审查时阻止合并 |
| `BlockOnOfficialReviewRequests` | `bool` | 有待官方审查时阻止合并 |
| `BlockOnOutdatedBranch` | `bool` | 分支过时时阻止合并 |
| `DismissStaleApprovals` | `bool` | 新提交时自动驳回过期审批 |
| `IgnoreStaleApprovals` | `bool` | 统计时忽略过期审批 |
| `RequireSignedCommits` | `bool` | 需要签名提交 |
| `ProtectedFilePatterns` | `string` | 受保护文件模式（分号分隔） |
| `EnableMergeWhitelist` | `bool` | 是否启用合并白名单 |
| `MergeWhitelistUserIDs` | `[]int64` | 合并用户白名单 |
| `MergeWhitelistTeamIDs` | `[]int64` | 合并团队白名单 |
| `EnableBypassAllowlist` | `bool` | 是否允许绕过分支保护 |
| `BypassAllowlistUserIDs` | `[]int64` | 绕过白名单用户 |
| `BypassAllowlistTeamIDs` | `[]int64` | 绕过白名单团队 |
| `BlockAdminMergeOverride` | `bool` | 阻止管理员强制合并 |

### 5.2 权限判断

**合并权限判断** (`services/pull/merge.go:548-567`)：
```go
func IsUserAllowedToMerge(ctx context.Context, pr *issues_model.PullRequest, p access_model.Permission, user *user_model.User) (bool, error) {
    pb, err := git_model.GetFirstMatchProtectedBranchRule(ctx, pr.BaseRepoID, pr.BaseBranch)
    if (p.CanWrite(unit.TypeCode) && pb == nil) || (pb != nil && git_model.IsUserMergeWhitelisted(ctx, pb, user.ID, p)) {
        return true, nil
    }
    return false, nil
}
```

**强制合并权限** (`services/pull/check.go:193-211`)：
```go
canForceMerge := isRepoAdmin
if protectedBranchRule != nil {
    canForceMerge = git_model.CanBypassBranchProtection(ctx, protectedBranchRule, doer, isRepoAdmin)
}
```

**官方审查者判断** (`models/git/protected_branch.go:234`)：
```go
func IsUserOfficialReviewer(ctx context.Context, protectBranch *ProtectedBranch, user *user_model.User) (bool, error) {
    if !protectBranch.EnableApprovalsWhitelist {
        // 有写权限即为官方审查者
        writeAccess, err := access_model.HasAccessUnit(ctx, user, repo, unit.TypeCode, perm.AccessModeWrite)
        return writeAccess, err
    }
    // 检查是否在审批白名单中
    return inWhitelist, nil
}
```

### 5.3 绕过分支保护权限矩阵

**核心函数**: `CanBypassBranchProtection` (`models/git/protected_branch.go:212-231`)

```go
func CanBypassBranchProtection(ctx context.Context, protectBranch *ProtectedBranch, user *user_model.User, isRepoAdmin bool) bool {
    if isRepoAdmin && !protectBranch.BlockAdminMergeOverride {
        return true  // 管理员且未被阻止 → 可以绕过
    }
    if !protectBranch.EnableBypassAllowlist {
        return false  // 未启用绕过白名单 → 不能绕过
    }
    // 检查用户/团队是否在绕过白名单中
    return inBypassAllowlist
}
```

**完整条件矩阵**：

| 用户角色 | `BlockAdminMergeOverride` | `EnableBypassAllowlist` | 在绕过白名单中 | `canBypassProtection` | 说明 |
|---------|--------------------------|------------------------|---------------|----------------------|------|
| 非管理员 | 任意 | `false` | 任意 | `false` | 非管理员且白名单禁用 → 不能绕过 |
| 非管理员 | 任意 | `true` | 否 | `false` | 不在白名单中 → 不能绕过 |
| 非管理员 | 任意 | `true` | 是 | `true` | 在白名单中 → 可以绕过 |
| 管理员 | `false` | 任意 | 任意 | `true` | 管理员未被阻止 → 可以绕过 |
| 管理员 | `true` | `false` | 任意 | `false` | 管理员被阻止且白名单禁用 → 不能绕过 |
| 管理员 | `true` | `true` | 否 | `false` | 管理员被阻止且不在白名单 → 不能绕过 |
| 管理员 | `true` | `true` | 是 | `true` | 管理员被阻止但在白名单 → 可以绕过 |

**关键点**：
- 管理员默认可以绕过，除非 `BlockAdminMergeOverride=true`
- 非管理员只能通过绕过白名单获得绕过权限
- 绕过权限只影响 `canMergeNow`，不影响 `mergeStyles` 生成（仍需 `IsStatusMergeable()==true`）

---

## 六、PR 状态流转与检查队列

### 6.1 PR 状态枚举

**位置**: `models/issues/pull.go:98-106`

```go
const (
    PullRequestStatusConflict PullRequestStatus = iota  // 有冲突
    PullRequestStatusChecking                           // 检查中
    PullRequestStatusMergeable                          // 可合并
    PullRequestStatusManuallyMerged                     // 已手动合并
    PullRequestStatusError                              // 检查出错
    PullRequestStatusEmpty                              // 空 PR
    PullRequestStatusAncestor                           // HEAD 已在目标分支中
)
```

### 6.2 检查队列机制

**位置**: `services/pull/check.go:38-534`

1. **队列初始化** (`Init()`):
   - 创建唯一队列 `pr_patch_checker`（避免重复任务）
   - 启动 worker 处理 `checkPullRequestMergeable`

2. **任务触发**：
   - `StartPullRequestCheckImmediately` - 立即检查（PR 更新时）
   - `StartPullRequestCheckDelayable` - 延迟检查（目标分支更新时，对不活跃 PR 延迟）
   - `StartPullRequestCheckOnView` - 查看时触发检查

3. **检查流程** (`checkPullRequestMergeable`):
   ```
   checkPullRequestMergeable(id)
   ├─ 获取全局锁 (globallock.Lock)
   ├─ 检查是否已合并
   ├─ 检查是否手动合并 (manuallyMerged)
   ├─ 检查分支是否可合并 (checkPullRequestBranchMergeable)
   │   ├─ 冲突检测
   │   ├─ CommitsAhead/CommitsBehind 计算
   │   └─ 受保护文件变更检测
   └─ 标记状态 (markPullRequestAsMergeable)
       ├─ 设置状态为 Mergeable/Conflict
       └─ 如有自动合并调度，触发自动合并检查
   ```

---

## 七、错误类型汇总

**位置**: `services/pull/check.go:40-50`

| 错误变量 | 说明 |
|---------|------|
| `ErrIsClosed` | PR 已关闭 |
| `ErrNoPermissionToMerge` | 无合并权限 |
| `ErrNotReadyToMerge` | 未满足合并条件（分支保护检查失败） |
| `ErrHasMerged` | 已合并 |
| `ErrIsWorkInProgress` | WIP 状态 |
| `ErrIsChecking` | 正在检查中 |
| `ErrNotMergeableState` | 非可合并状态（有冲突等） |
| `ErrDependenciesLeft` | 有关联依赖未解决 |
| `ErrHeadCommitsNotAllVerified` | 提交未全部签名 |

---

## 八、Merge Box 状态映射与按钮逻辑

### 8.1 后端状态 → UI 文案映射

**核心位置**: `routers/web/repo/issue_view.go:1040-1083`

| 阻断器变量 | 判断逻辑 | UI 文案 | 图标 |
|-----------|---------|---------|------|
| `isBlockedByApprovals` | `!HasEnoughApprovals(ctx, pb, pull)` | "1/2 Approvals" (实际数量) | ✗ 红色 |
| `isBlockedByRejection` | `MergeBlockedByRejectedReview(ctx, pb, pull)` | "Blocked by rejection" | ✗ 红色 |
| `isBlockedByOfficialReviewRequests` | `MergeBlockedByOfficialReviewRequests(ctx, pb, pull)` | "Blocked by official review requests" | ✗ 红色 |
| `isBlockedByOutdatedBranch` | `MergeBlockedByOutdatedBranch(pb, pull)` | "The head branch is behind the base branch" | ✗ 红色 |
| `isBlockedByChangedProtectedFiles` | `len(pull.ChangedProtectedFiles) != 0` | "Changed 1 protected file" (列表) | ✗ 红色 |
| `hasStatusCheckBlocker` | `enableStatusCheck && !RequiredChecksState.IsSuccess()` | 各 status check 单独显示 | ✗/○/✓ |

**附加信息提示**:
- `hasPermToMerge=false` → "You're not authorized to merge this pull request" (ℹ️ 灰色)
- `hasOverridableBlockers && canBypassProtectionAsAdmin` → "Only administrators can merge this pull request with failing checks" (⚫)
- `hasOverridableBlockers && !canBypassProtectionAsAdmin` → "Only users in the bypass allowlist can merge this pull request with failing checks" (⚫)
- `canMergeNow && !hasOverridableBlockers` → "This pull request can be merged automatically" (✓ 绿色)

### 8.2 Blocker 分类与绕过能力

#### 8.2.1 两类 Blocker 的定义

| Blocker 类型 | 包含情况 | 判定代码 | 能否绕过 |
|-------------|---------|---------|---------|
| **Commit Blocker** | 冲突、数据损坏、检查中、祖先提交 | `pull_merge_box.go:109-144` | **不可绕过** |
| **Protection Blocker** | 审批不足、拒绝审查、Status Check 失败、分支过时、受保护文件变更、签名要求不满足 | `issue_view.go:957-959` + `pull_merge_box.go:146-163` | **可绕过**（需要权限） |

**重要修正**：`!IsStatusMergeable() && !IsEmpty()` 被添加到 `infoProtectionBlockers`（代码注释说"can be bypassed by admin"），但这是**半真半假**的说法：
- 从 `canMergeNow` 的计算看，它确实**不把**非 Mergeable 状态作为 overridable blocker 考虑
- 但 `mergeStyles` 的生成**要求** `IsStatusMergeable() == true`（`pull_merge_form.go:109`）
- 实际效果：即使 `canMergeNow == true`，如果 `IsStatusMergeable() == false`，也**没有可用的合并方式**（除手动合并外）

#### 8.2.2 commit blocker 详细列表

**位置**: `routers/web/repo/pull_merge_box.go:109-144`

| 状态 | 判断逻辑 | UI 文案 |
|------|---------|---------|
| 冲突 | `pull.IsFilesConflicted()` | "This branch has conflicts that must be resolved" + 文件列表 |
| 数据损坏 | `prInfo.IsPullRequestBroken` | "Data broken" |
| 检查中 | `pull.IsChecking()` | "This pull request is still being checked" |
| 祖先提交 | `pull.IsAncestor()` | "The head commit is already in the base branch" |

#### 8.2.3 protection blocker 详细列表

**位置**: `routers/web/repo/pull_merge_box.go:146-170` + `issue_view.go:1040-1080`

| 状态 | 判断逻辑 | UI 文案 | 能否绕过 |
|------|---------|---------|---------|
| 不可合并 | `!pull.IsStatusMergeable() && !pull.IsEmpty()` | "This pull request can't be merged" + "Ask someone with write access to merge this pull request manually" | 否¹ |
| 空 PR | `pull.IsEmpty()` | "This pull request is empty" | 否² |
| 无合并权限 | `!hasPermToMerge` | "You're not authorized to merge this pull request" | 否 |
| Status Check 失败 | `enableStatusCheck && !RequiredChecksState.IsSuccess()` | "Required status checks have failed" | 是 |
| 审批不足 | `!HasEnoughApprovals()` | "1/2 Approvals" | 是 |
| 拒绝审查 | `MergeBlockedByRejectedReview()` | "Blocked by rejection" | 是 |
| 分支过时 | `MergeBlockedByOutdatedBranch()` | "The head branch is behind the base branch" | 是 |
| 受保护文件变更 | `ChangedProtectedFiles` | "Changed 1 protected file" | 是 |
| 签名要求不满足 | `requireSigned && !willSign` | "Requires signed commits" | **否³** |

**重要注释**：
¹ 注释说"can be bypassed by admin"，但实际受 `mergeStyles` 生成限制
² 空 PR 状态下 `mergeStyles` 不会生成正常合并选项
³ **签名要求不能被绕过** - 检查在 `CheckPullMergeable` 的 forceMerge 逻辑之后执行

---

#### 8.2.4 签名要求的特殊处理

**⚠️ 关键发现**：签名要求不被算作 `hasOverridableBlockers`，但通过独立的与运算影响结果。

**代码位置**: `routers/web/repo/issue_view.go:968-970`
```go
data.canMergeNow = (!data.hasOverridableBlockers || data.canBypassProtection) &&
    (!data.requireSigned || data.willSign)
```

**执行顺序** (`services/pull/check.go:179-221`):
```
CheckPullMergeable
├─ CheckPullBranchProtections → forceMerge 可跳过此检查
└─ checkSigningRequirements   → forceMerge 不可跳过此检查（在绕过逻辑之后）
```

**结论**：即使启用了 forceMerge，签名要求仍然必须满足。签名检查是**唯一不能被绕过**的保护规则。

### 8.3 按钮可用性与 `canMergeNow` 深层解析

**⚠️ 关键修正**：`canMergeNow` 只是一个标志，**不直接决定合并按钮是否显示**。真正决定按钮可用性的是 `mergeStyles` 的生成逻辑。

#### 8.3.1 canMergeNow 的真实含义

**代码位置**: `routers/web/repo/issue_view.go:968-970`
```go
// CanMergeNow means: if the doer has write permission, whether the PR can be merged now
data.canMergeNow = (!data.hasOverridableBlockers || data.canBypassProtection) && // status checks are satisfied
    (!data.requireSigned || data.willSign) // signing requirement is satisfied
```

**注意**：`canMergeNow` **不检查** `IsStatusMergeable()`！它只检查 overridable blockers 和签名要求。

#### 8.3.2 mergeStyles 生成条件（真正的可用性闸口）

**代码位置**: `routers/web/repo/pull_merge_form.go:109-168`

| 合并方式 | 生成条件 |
|---------|---------|
| 正常合并 (merge/rebase/squash/ff) | `IsStatusMergeable() == true` + 仓库启用对应方式 |
| 手动合并 (manually-merged) | `!IsWorkInProgress()` + `!IsChecking()` + `AllowManualMerge == true` |

**结论**：即使 `canMergeNow == true`，如果 `IsStatusMergeable() == false`，也只能使用手动合并方式。

#### 8.3.3 前端按钮行为逻辑

**前端组件**: `web_src/js/components/PullRequestMergeForm.vue`

| `canMergeNow` | `IsStatusMergeable()` | `mergeStyles` 非空 | 按钮行为 | 可用操作 |
|--------------|----------------------|-------------------|---------|---------|
| `true` | `true` | 是 | 启用 | 立即合并 + 自动合并（可选） |
| `true` | `false` | 仅手动合并 | 启用 | 手动标记为已合并 |
| `false` | `true` | 是 | 下拉仅显示「自动合并」 | 只能设置自动合并 |
| `false` | `false` | 仅手动合并 | 启用（如允许） | 手动标记为已合并 |
| 任意 | 任意 | 否 | Vue 组件不挂载 | 无合并按钮 |

**按钮样式逻辑** (`PullRequestMergeForm.vue:33-37`):
```typescript
const mergeButtonStyleClass = computed(() => {
  if (mergeStyle.value === mergeStyleManuallyMerged) return 'red';
  if (mergeForm.allOverridableChecksOk) return 'primary';
  return autoMergeWhenSucceed.value ? 'primary' : 'red';
});
```

**强制合并标志** (`PullRequestMergeForm.vue:45-47`):
```typescript
const forceMerge = computed(() => {
  return mergeForm.canMergeNow && !mergeForm.allOverridableChecksOk;
});
```

---

#### 8.3.3.1 红色按钮原因的澄清

**⚠️ 常见误判：签名要求不满足导致红色按钮？**

| 状态 | 结果 | 原因 |
|------|------|------|
| `requireSigned=true` 且 `willSign=false` | `canMergeNow=false` | 按钮不显示或只显示自动合并（非红色） |
| `allOverridableChecksOk=false` 且 `canMergeNow=true` | 红色按钮（强制合并） | 有 protection blocker 但可绕过 |
| `mergeStyle=manually-merged` | 红色按钮 | 手动合并方式 |

**关键结论**：
- 签名要求不满足 **不会** 导致红色按钮，而是导致 `canMergeNow=false`
- 红色按钮只在两种情况下出现：
  1. 有 protection blocker（审批/CI/分支过时等）但可以绕过（forceMerge）
  2. 选择了手动合并方式（manually-merged）
- 签名要求不满足时，用户甚至看不到合并按钮（只能设置自动合并）

#### 8.3.4 `hasOverridableBlockers` 的注释提示

**代码位置**: `routers/web/repo/issue_view.go:954-959`

```go
// HINT: if a PR's status is not mergeable, then it is a non-overridable blocker, such logic is handled separately (see IsStatusMergeable)
data.hasOverridableBlockers = data.isBlockedByApprovals || data.isBlockedByRejection ||
    data.isBlockedByOfficialReviewRequests || data.isBlockedByOutdatedBranch || data.isBlockedByChangedProtectedFiles ||
    data.hasStatusCheckBlocker
```

**这段注释非常重要**：它明确说明非 Mergeable 状态是**不可绕过**的，由 `IsStatusMergeable` 单独处理（即通过 `mergeStyles` 生成逻辑）。

### 8.4 完整条件决策矩阵

#### 8.4.1 合并可用性总表

```
场景 1: 完美可合并
├─ IsStatusMergeable()  = true
├─ 无 commit blocker
├─ hasOverridableBlockers = false
├─ canMergeNow          = true
└─ 可用方式: 全部正常合并方式 (merge/rebase/squash/ff)

场景 2: 可强制合并
├─ IsStatusMergeable()  = true
├─ 无 commit blocker
├─ hasOverridableBlockers = true
├─ canBypassProtection  = true
├─ canMergeNow          = true
└─ 可用方式: 强制合并（红色按钮）

场景 3: 仅能自动合并
├─ IsStatusMergeable()  = true
├─ 无 commit blocker
├─ hasOverridableBlockers = true
├─ canBypassProtection  = false
├─ canMergeNow          = false
└─ 可用方式: 下拉菜单仅显示「自动合并」

场景 4: 有 commit blocker
├─ IsStatusMergeable()  = false 或 检查中/冲突
├─ 有 commit blocker
└─ 可用方式: 仅手动合并（如启用）

场景 5: 能 bypass 但状态非 Mergeable
├─ IsStatusMergeable()  = false
├─ 无 commit blocker
├─ canBypassProtection  = true
├─ canMergeNow          = true
└─ 可用方式: 仅手动合并（无正常合并选项）

场景 6: 完全无法合并
├─ IsStatusMergeable()  = false
├─ 无 commit blocker
├─ canBypassProtection  = false
├─ canMergeNow          = false
└─ 可用方式: 仅手动合并（如启用）
```

**记忆口诀**：
- `IsStatusMergeable() == false` → 无正常合并选项（只能手动）
- `canMergeNow == false` → 无法立即合并（只能自动合并）

#### 8.4.2 常见排查场景速查表

| 现象 | 可能原因 | 验证方法 |
|------|---------|---------|
| 🔴 红色图标但只有 "Checking" 文字 | 有 protection blocker 但被 commit blocker 隐藏 | 检查 `infoCommitBlockers` 是否非空 |
| ⚪ 有审批但计数不足 | Review 的 `official=false` | 检查审查者是否在审批白名单 |
| 🟢 CI 全绿但按钮禁用 | 必填 context 配置了但未上报 | 检查 `MissingRequiredChecks` |
| 🔴 管理员无法强制合并 | `BlockAdminMergeOverride=true` 或 `IsStatusMergeable()==false` | 检查保护规则配置 + PR 状态 |
| ⚪ 能看到按钮但只有 "Auto-merge" | `canMergeNow==false` 但 `mergeStyles` 非空 | 检查 overridable blockers |
| ❌ 完全看不到合并按钮 | `mergeStyles` 为空 | 检查 `IsStatusMergeable()` 和合并方式配置 |
| 🟢 显示 "Can be merged" 但按钮红色 | `canMergeNow==true` 但 `allOverridableChecksOk==false` | 这是正常的强制合并状态 |

### 8.5 Merge Box 图标颜色规则

**位置**: `routers/web/repo/pull_merge_box.go:54-88`

```
颜色优先级（从高到低）：
├─ 已合并 → 紫色
├─ 已关闭 / WIP / 空 PR / 有冲突 → 灰色
├─ Status Check 失败 (error/failure) → 红色
├─ 有 commit/protection blockers → 红色
├─ 检查中 / Status Check pending/warning → 黄色
├─ 可合并 (IsStatusMergeable) → 绿色
└─ 其他 → 灰色
```

---

## 九、状态断点排查清单

### 9.1 Status Check 失败排查

**现象**: 合并按钮禁用，显示 "Required status checks have failed"

| 检查项 | 位置 | 验证方法 |
|-------|------|---------|
| 保护规则是否启用 | `ProtectedBranch.EnableStatusCheck` | 仓库设置 → 分支保护 → 状态检查 |
| 必填 Contexts 配置 | `ProtectedBranch.StatusCheckContexts` | 检查 glob 模式是否匹配实际 status |
| Commit SHA 正确性 | `pr.CompareInfo.HeadCommitID` | 确认 status 是针对最新 HEAD 的 |
| 状态合并结果 | `MergeRequiredContextsCommitStatus()` | 调试：检查各必填 context 的匹配结果 |
| 是否有未出现的检查 | - | 如配置了 context 但无对应 status，整体为 pending |

**代码断点**:
- `services/pull/commit_status.go:69` - `IsPullCommitStatusPass` 入口
- `services/pull/commit_status.go:22` - `MergeRequiredContextsCommitStatus` 匹配逻辑
- `routers/web/repo/pull.go:452-470` - 缺失 required checks 收集

### 9.2 审批数量不足排查

**现象**: 显示 "1/2 Approvals" 但看起来有足够审批

| 检查项 | 位置 | 验证方法 |
|-------|------|---------|
| Review 是否为 Official | `Review.Official=true` | 审查者需在审批白名单中 或 有写权限 |
| Review 是否已被驳回 | `Review.Dismissed=false` | 检查是否有 Dismiss Review 操作 |
| 是否忽略过期审批 | `ProtectedBranch.IgnoreStaleApprovals` | 新提交后旧审批是否被标记为 stale |
| Review 类型 | `Review.Type=Approve` | Comment/Request 类型不计入 |
| 审批白名单配置 | `ProtectedBranch.EnableApprovalsWhitelist` | 审查者是否在白名单中 |

**代码断点**:
- `models/issues/pull.go:769` - `GetGrantedApprovalsCount` SQL 查询
- `models/git/protected_branch.go:234` - `IsUserOfficialReviewer` 判断

### 9.3 按钮禁用但无明确错误

**现象**: 合并按钮灰色/隐藏，无明显错误提示

| 检查项 | 位置 | 验证方法 |
|-------|------|---------|
| 合并权限 | `hasPermToMerge` | 用户是否在合并白名单 或 有写权限 |
| Commit 状态 | `pull.Status` | 是否为 `Conflict`/`Error`/`Checking` |
| WIP 状态 | `pr.IsWorkInProgress()` | 标题是否含 WIP 标记 |
| PR 是否已关闭 | `issue.IsClosed` | |
| 合并方式配置 | `prConfig.AllowMerge/AllowRebase/...` | 仓库是否启用了合并方式 |
| 签名要求 | `requireSigned && !willSign` | 是否配置了签名要求但无法签名 |

**代码断点**:
- `routers/web/repo/issue_view.go:910` - `IsUserAllowedToMerge` 权限判断
- `routers/web/repo/pull_merge_form.go:18-48` - 合并方式筛选逻辑

### 9.4 强制合并不可用

**现象**: 有阻断但无 "Force Merge" 选项

| 检查项 | 位置 | 验证方法 |
|-------|------|---------|
| 是否为仓库管理员 | `isRepoAdmin` | 用户是否是仓库 admin 或 site admin |
| 绕过白名单配置 | `ProtectedBranch.EnableBypassAllowlist` | 是否启用绕过白名单 |
| 管理员绕过阻止 | `ProtectedBranch.BlockAdminMergeOverride` | 是否阻止管理员强制合并 |
| 阻断类型 | `infoCommitBlockers` vs `infoProtectionBlockers` | Commit 级阻断（冲突）不可绕过 |

**代码断点**:
- `routers/web/repo/issue_view.go:961-966` - `canBypassProtection` 判断
- `models/git/protected_branch.go:212` - `CanBypassBranchProtection`

### 9.5 自动合并不工作

**现象**: 设置了 "Merge when checks succeed" 但未自动合并

| 检查项 | 位置 | 验证方法 |
|-------|------|---------|
| 自动合并调度记录 | `pull_model.GetScheduledMergeByPullID` | DB 中是否有 auto_merge 记录 |
| Status Check 是否全部通过 | `IsPullCommitStatusPass` | 所有必填检查是否 success |
| 审批是否满足 | `HasEnoughApprovals` | 审批数量是否达标 |
| 检查队列是否运行 | `prPatchCheckerQueue` | 后台 worker 是否正常 |
| 合并权限 | `IsUserAllowedToMerge` | 调度者是否仍有合并权限 |

**代码断点**:
- `services/pull/check.go:298-305` - 自动合并触发逻辑
- `services/automergequeue/` - 自动合并队列处理

---

## 十、完整状态决策树

```
PR 页面加载
│
├─ 加载 PR 基础信息
│  ├─ HasMerged? → 显示已合并状态
│  ├─ IsClosed? → 显示已关闭状态
│  └─ IsWorkInProgress? → 标记 WIP
│
├─ prepareMergeBoxProtectionChecks()
│  ├─ 加载 ProtectedBranch 规则
│  ├─ prepareMergeBoxStatusCheckData() → 计算 status check 状态
│  └─ prepareMergeBoxProtectedRules() → 计算各阻断标志
│
├─ 计算核心标志
│  ├─ hasStatusCheckBlocker = enableStatusCheck && !RequiredChecksState.IsSuccess()
│  ├─ hasOverridableBlockers = isBlockedByApprovals || isBlockedByRejection || ...
│  ├─ canBypassProtection = isRepoAdmin || (in bypass allowlist)
│  └─ canMergeNow = (!hasOverridableBlockers || canBypassProtection) && (!requireSigned || willSign)
│
├─ 组装信息条目
│  ├─ infoCommitBlockers (冲突、检查中、祖先、空PR、数据损坏)
│  ├─ infoProtectionBlockers (审批、拒绝、status check 等)
│  └─ infoMergePrompts (可合并提示、可绕过提示)
│
└─ 渲染 Merge Box
   ├─ 图标颜色判定 (prepareMergeBoxIconColor)
   ├─ 合并表单属性 (prepareMergeBoxFormProps)
   └─ 信息条目展示 (prepareMergeBoxInfoItems)
```

---

## 十一、InfoSections 选择规则与展示优先级

### 11.1 InfoSections 构建逻辑

**位置**: `routers/web/repo/pull_merge_box.go:190-195`

```go
if len(data.infoCommitBlockers.items) > 0 {
    data.InfoSections = append(data.InfoSections, &pullInfoSection{data.infoCommitBlockers.items})
} else {
    data.InfoSections = append(data.InfoSections, &pullInfoSection{data.infoProtectionBlockers.items})
}
data.InfoSections = append(data.InfoSections, &pullInfoSection{data.infoMergePrompts.items})
```

**核心规则：commit blocker 与 protection blocker 互斥展示**

| 条件 | Section 1 | Section 2 |
|------|-----------|-----------|
| 有 commit blocker | `infoCommitBlockers` | `infoMergePrompts` |
| 无 commit blocker | `infoProtectionBlockers` | `infoMergePrompts` |

**设计意图**：commit blocker（冲突/损坏/检查中/祖先）意味着 PR 尚未完成基础检查，此时展示 protection blocker 无意义；反之若 commit 层面没问题，才需展示保护规则阻断。

### 11.2 三个集合的填充顺序

**prepareMergeBoxInfoItems** (`pull_merge_box.go:91-195`) 按以下顺序填充：

```
1. infoCommitBlockers（按顺序添加）
   ├─ IsFilesConflicted()     → 冲突文件列表
   ├─ IsPullRequestBroken     → 数据损坏
   ├─ IsChecking()            → 检查中
   └─ IsAncestor()            → 祖先提交

2. infoProtectionBlockers（按顺序添加）
   ├─ !IsStatusMergeable()    → 不可合并 / 空 PR
   ├─ !hasPermToMerge         → 无合并权限
   ├─ [prepareMergeBoxStatusCheckData 添加]
   │   ├─ RequiredChecksState.IsError/Failure  → "Required status checks have failed"
   │   └─ !RequiredChecksState.IsSuccess       → "Required status checks are missing"
   ├─ [prepareMergeBoxProtectedRules 添加]
   │   ├─ isBlockedByApprovals                → 审批不足
   │   ├─ isBlockedByRejection                → 被拒绝审查阻塞
   │   ├─ isBlockedByOfficialReviewRequests   → 官方审查请求阻塞
   │   ├─ isBlockedByOutdatedBranch           → 分支过时
   │   └─ isBlockedByChangedProtectedFiles    → 受保护文件变更
   └─ [checkSigningRequirements 添加]
       └─ requireSigned && !willSign          → 签名要求未满足

3. infoMergePrompts（按顺序添加）
   ├─ [checkSigningRequirements 添加]
   │   ├─ willSign                            → "Will sign with ..."
   │   └─ !requireSigned && wontSignReason    → "Won't sign: ..."
   ├─ canMergeNow && hasOverridableBlockers   → 管理员/绕过白名单提示
   └─ canMergeNow && !hasOverridableBlockers  → "Can be merged automatically"
```

### 11.3 模板渲染顺序

**位置**: `templates/repo/issue/view_content/pull_merge_box.tmpl`

```
Merge Box DOM 结构（从上到下）：
┌─────────────────────────────────────────────┐
│ ① Timeline Icon (颜色由 prepareMergeBoxIconColor 决定)    │
├─────────────────────────────────────────────┤
│ ② Status Check 摘要区 (ShowStatusCheck 时)                 │
│   ├─ CommitStatusCheckPrompt 文案                          │
│   ├─ RequireApprovalRunCount 提示                          │
│   └─ 各 commit status 详情列表                             │
├─────────────────────────────────────────────┤
│ ③ ClosedInfo (已合并/已关闭时，替代后续内容)               │
├─────────────────────────────────────────────┤
│ ④ InfoSections[0] (commit blocker 或 protection blocker)  │
│   └─ 每个 InfoItem: 图标 + 文案 + 可选列表                 │
├─────────────────────────────────────────────┤
│ ⑤ InfoSections[1] (infoMergePrompts)                      │
│   └─ 可合并/可绕过提示                                     │
├─────────────────────────────────────────────┤
│ ⑥ Update Branch (ShowUpdatePullInfo 时)                    │
├─────────────────────────────────────────────┤
│ ⑦ WIP 提示 (IsPullWorkInProgress 时)                      │
├─────────────────────────────────────────────┤
│ ⑧ Merge Form (MergeFormProps 不为空时)                     │
│   └─ Vue 组件 PullRequestMergeForm                         │
├─────────────────────────────────────────────┤
│ ⑨ Pull Commands (ShowPullCommands 时)                      │
└─────────────────────────────────────────────┘
```

**关键渲染规则**：
- ② Status Check 区域**独立于** InfoSections，始终在最上方
- ③ ClosedInfo 存在时**替代** ④⑤⑥⑦⑧⑨
- ④ 互斥：commit blocker 存在时不显示 protection blocker

### 11.4 Status Check 摘要文案决策

**位置**: `routers/web/repo/pull.go:521-537` (`CommitStatusCheckPrompt`)

```
CommitStatusCheckPrompt 决策树：
├─ RequiredChecksState.IsPending() || len(MissingRequiredChecks) > 0
│   → "Waiting for status checks to succeed"        (黄色)
├─ RequiredChecksState.IsSuccess()
│   ├─ pullCommitStatusState.IsFailure()
│   │   → "All required checks have passed, but some other checks are failing"  (绿色+警告)
│   └─ 否则
│       → "All checks have passed"                   (绿色)
├─ RequiredChecksState.IsWarning()
│   → "All required checks have passed, but some with warnings"  (黄色)
├─ RequiredChecksState.IsFailure()
│   → "Required status checks have failed"           (红色)
├─ RequiredChecksState.IsError()
│   → "Required status checks have errored"          (红色)
└─ 其他
    → "Waiting for status checks to succeed"          (黄色)
```

**注意**：`RequiredChecksState` 与 `pullCommitStatusState` 是两个独立状态：
- `RequiredChecksState` = 只看**必填** context 的合并状态（决定是否阻断合并）
- `pullCommitStatusState` = **所有** commit status 的合并状态（仅用于信息展示）

---

## 十二、排查误判场景

### 12.1 "明明有审批却显示审批不足"

**根因**：审批计数只统计 `official=true` 的 review

**排查步骤**：

1. **确认审查者是否在审批白名单**
   - `EnableApprovalsWhitelist=false` → 所有有写权限的用户都是官方审查者
   - `EnableApprovalsWhitelist=true` → 只有白名单内的用户/团队才是官方审查者
   - 代码位置：`models/git/protected_branch.go:234` `IsUserOfficialReviewer`

2. **确认 review 的 official 字段**
   - review 创建时根据 `IsUserOfficialReviewer` 结果设置 `official`
   - **如果后续修改了白名单，已存在的 review 不会重新计算 official**
   - 需要审查者重新提交 review

3. **确认是否有 dismissed review**
   - `DismissStaleApprovals=true` 时，新提交后旧 review 被自动 dismissed
   - 手动 Dismiss Review 也会设置 `dismissed=true`
   - 代码位置：`models/issues/pull.go:782` `GetGrantedApprovalsCount`

4. **确认 IgnoreStaleApprovals 的影响**
   - `IgnoreStaleApprovals=true` 时，stale review 不计入
   - 与 `DismissStaleApprovals` 不同：stale 只是标记，dismissed 才是驳回
   - 代码位置：`models/issues/pull.go:788`

### 12.2 "Status Check 通过了但合并按钮仍禁用"

**根因**：`pullCommitStatusState`（全部状态）与 `RequiredChecksState`（必填状态）不一致

**排查步骤**：

1. **确认 RequiredChecksState 的值**
   - 前端 Status Check 摘要文案由 `RequiredChecksState` 决定
   - 即使 `pullCommitStatusState=success`，如果必填 context 中有 pending 的，`RequiredChecksState` 仍为 pending
   - 代码位置：`routers/web/repo/pull.go:485`

2. **检查 glob 匹配问题**
   - 必填 context 支持 glob 模式（如 `ci-*`）
   - 旧版存储的 context 可能不是合法 glob，会触发 `log.Error` 但不中断匹配
   - 代码位置：`routers/web/repo/pull.go:474`

3. **检查缺失的 required checks**
   - `MissingRequiredChecks` 列出了配置了但没找到匹配 status 的 context
   - 只要有一个缺失，整体为 pending
   - 代码位置：`routers/web/repo/pull.go:452-466`

4. **检查 commit SHA 是否正确**
   - status 是按 commit SHA 查询的，如果 PR 有新推送但 status 还在旧 SHA 上，看不到
   - 代码位置：`routers/web/repo/pull.go:423`

### 12.3 "Merge Box 显示红色图标但找不到错误"

**根因**：图标颜色逻辑与 InfoSections 展示逻辑是独立的

**排查步骤**：

1. **检查 hasBlockers 变量**
   - 图标颜色：`hasBlockers = len(infoCommitBlockers.items) > 0 || len(infoProtectionBlockers.items) > 0`
   - 即使有 blocker，如果属于 commit blocker 类型，InfoSections 只展示 commit blocker
   - protection blocker 的内容可能被隐藏
   - 代码位置：`pull_merge_box.go:71,82-83`

2. **检查 IsStatusMergeable 状态**
   - `!pull.IsStatusMergeable() && !pull.IsEmpty()` 会添加 "can't be merged" 到 protection blocker
   - 但如果同时有 commit blocker（如检查中），只会展示 commit blocker
   - 这导致用户看到红色图标，但展示的是 "still being checked" 而非 "can't be merged"

3. **检查 Status Check 状态对图标的影响**
   - 图标颜色优先看 `statusCheckData`，即使 `hasBlockers=false` 也会因 CI 失败变红
   - 但 CI 失败的信息在 Status Check 区域展示，不在 InfoSections
   - 代码位置：`pull_merge_box.go:62-68`

### 12.4 "管理员无法强制合并"

**根因**：`canBypassProtection` 和 `canMergeNow` 计算链中的多个条件

**排查步骤**：

1. **区分 commit blocker vs protection blocker**
   - 只有 protection blocker 可绕过
   - 冲突、检查中、数据损坏 = commit blocker = 不可绕过
   - 代码位置：`pull_merge_box.go:190-194`

2. **检查 BlockAdminMergeOverride**
   - `BlockAdminMergeOverride=true` → 即使是仓库管理员也不能绕过
   - 代码位置：`models/git/protected_branch.go:213`

3. **检查 EnableBypassAllowlist**
   - `EnableBypassAllowlist` 只影响**非管理员**用户的绕过权限
   - 管理员的绕过权由 `BlockAdminMergeOverride` 单独控制，不受 `EnableBypassAllowlist` 影响
   - 代码位置：`models/git/protected_branch.go:212-215`

4. **检查 IsStatusMergeable 状态**
   - **最常见原因**：`IsStatusMergeable()==false` 时 `mergeStyles` 不会生成正常合并选项
   - 即使 `canMergeNow==true`，如果 `IsStatusMergeable()==false`，也只能用手动合并
   - 代码位置：`routers/web/repo/pull_merge_form.go:109`

5. **检查签名要求的与运算**
   - `canMergeNow = (!hasOverridableBlockers || canBypassProtection) && (!requireSigned || willSign)`
   - 签名要求是独立条件，即使能绕过保护规则，签名不满足也无法合并
   - 代码位置：`routers/web/repo/issue_view.go:969`

### 12.5 "Status Check 摘要文案与详情不一致"

**根因**：摘要由 `CommitStatusCheckPrompt` 决定，详情由 `status_items` 模板渲染

**排查步骤**：

1. **摘要说 "All checks have passed" 但详情有红色项**
   - `RequiredChecksState=success` 且 `pullCommitStatusState=failure`
   - 摘要文案："All required checks have passed, but some other checks are failing"
   - 如果该文案被截断或用户没注意后半句，会误以为全部通过
   - 代码位置：`routers/web/repo/pull.go:525-527`

2. **摘要说 "Waiting" 但详情全部绿色**
   - 可能存在 `MissingRequiredChecks`，即配置了必填 context 但还没上报
   - 摘要由 `RequiredChecksState.IsPending() || len(MissingRequiredChecks) > 0` 决定
   - 即使所有已上报的 check 都是 success，缺失的 check 仍使整体为 pending
   - 代码位置：`routers/web/repo/pull.go:522`

3. **Status Check 摘要与 InfoSections 阻断项重复**
   - Status Check 失败时，既在 Status Check 区域展示详情，又在 InfoSections[0] 展示 "Required status checks have failed"
   - 这是**双重展示**而非 bug，前者是详细信息，后者是阻断汇总
   - 代码位置：`routers/web/repo/pull.go:487-498` 和 `pull_merge_box.go:190-194`

### 12.6 "PR 可合并但合并表单不显示"

**根因**：MergeFormProps 为空导致 Vue 组件不挂载

**排查步骤**：

1. **检查 canMergeNow 和 hasPermToMerge**
   - `MergeFormProps` 在 `canMergeNow || hasPermToMerge` 时才生成
   - 代码位置：`routers/web/repo/pull_merge_form.go`

2. **检查仓库合并方式配置**
   - `AllowMerge`/`AllowRebase`/`AllowSquash`/`AllowFastForwardOnly` 至少一个为 true
   - 全部为 false 时不生成合并表单
   - 代码位置：`routers/web/repo/pull_merge_form.go:18-48`

3. **检查 Manually Merged 权限**
   - 即使 `canMergeNow=false`，如有手动合并权限，仍可显示手动合并选项
   - 但需 `AllowManualMerge=true` 且用户有写权限

### 12.7 "分支保护规则更新后合并状态未刷新"

**根因**：保护规则变更后的重检机制

**排查步骤**：

1. **检查保护规则更新是否触发了 PR 重检**
   - `services/pull/protected_branch.go` 中的 `OnProtectedBranchRuleChange`
   - 应对所有受影响的 PR 调用 `StartPullRequestCheckImmediately`

2. **检查前端自动刷新**
   - Merge Box 有 `data-pull-merge-box-reloading-interval` 属性
   - PR 处于 `IsChecking` 状态时，前端会定时拉取最新状态
   - 代码位置：`pull_merge_box.tmpl:5-8`

3. **检查缓存问题**
   - `GetLatestCommitStatus` 查询结果可能受 DB 缓存影响
   - 保护规则变更不会使 commit status 缓存失效

---

### 12.8 "签名要求导致按钮不可用"

**根因**：签名要求不被算作 overridable blocker，但通过独立的与运算影响结果

**排查步骤**：

1. **检查 `requireSigned` 是否为 true**
   - 确认分支保护规则中 `RequireSignedCommits` 是否启用
   - 代码位置：`issue_view.go:493`

2. **检查 `willSign` 的值**
   - 调用 `asymkey_service.SignMerge()` 检查 Gitea 是否能为当前用户签名
   - 代码位置：`issue_view.go:497`

3. **检查 wontSign 原因**
   - `ErrWontSign` 的可能原因：
     - `no-key`: 用户没有配置 GPG 密钥
     - `not-signing`: Gitea 未配置签名密钥
     - `email-mismatch`: 用户邮箱与 GPG 密钥邮箱不匹配
     - `error`: 其他错误
   - 代码位置：`issue_view.go:501-506`

4. **检查合并方式的签名要求差异**
   | 合并方式 | 签名要求 |
   |---------|---------|
   | fast-forward-only | 用户提交必须全部已签名 |
   | merge | 用户提交必须全部已签名 + Gitea 必须能签名合并提交 |
   | rebase/rebase-merge/squash | Gitea 必须能签名生成的提交 |
   - 代码位置：`services/pull/check.go:256-269`

---

### 12.9 签名问题最小排查流程

**当遇到签名相关的按钮不可用时，按以下顺序快速排查**：

```
第 1 步：确认现象
├─ 按钮完全不显示？ → 检查 hasPermToMerge
├─ 只有 "Auto-merge" 选项？ → canMergeNow=false，检查签名要求
└─ 按钮显示红色？ → 不是签名问题，检查 overridable blockers

第 2 步：检查签名配置
├─ 仓库：Repo Settings → Branch Protection → Require Signed Commits
├─ 用户：User Settings → GPG Keys → 确认密钥存在且邮箱匹配
└─ Gitea 服务器：app.ini → [repository.signing] 配置

第 3 步：代码层验证（快速断点）
├─ issue_view.go:493 → data.requireSigned = ?
├─ issue_view.go:498 → data.willSign = ?
├─ issue_view.go:502 → wontSignReason = ?
└─ issue_view.go:969 → canMergeNow 计算结果

第 4 步：验证后端检查
└─ services/pull/check.go:219 → checkSigningRequirements 能否通过
```

**常见错误原因对照表**：

| 现象 | 最可能原因 | 解决方法 |
|------|-----------|---------|
| `wontSignReason = "no-key"` | 用户未配置 GPG 密钥 | 用户在设置中添加 GPG 密钥 |
| `wontSignReason = "email-mismatch"` | GPG 密钥邮箱与提交邮箱不匹配 | 更新密钥邮箱或使用匹配的邮箱提交 |
| `wontSignReason = "not-signing"` | Gitea 未配置签名密钥 | 管理员配置 Gitea 的 [repository.signing] |
| `requireSigned=true` 但 FF 合并不可用 | 用户提交未全部签名 | 用户需要重新签名提交 |
| 管理员强制合并也不可用 | 签名要求不能被绕过 | 必须满足签名要求或使用手动合并标记 |

---

## 十三、关键代码位置速查表

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 合并检查主入口 | `services/pull/check.go` | 142 |
| 分支保护检查汇总 | `services/pull/merge.go` | 570 |
| Status Check 通过判断 | `services/pull/commit_status.go` | 69 |
| 必填 context 状态合并 | `services/pull/commit_status.go` | 22 |
| 多状态合并算法 | `modules/commitstatus/commit_status.go` | 67 |
| 审批数量统计 | `models/issues/pull.go` | 769 |
| 拒绝审查阻塞 | `models/issues/pull.go` | 787 |
| 分支过时阻塞 | `models/issues/pull.go` | 823 |
| 受保护文件阻塞 | `models/git/protected_branch.go` | 287 |
| 合并权限判断 | `services/pull/merge.go` | 548 |
| 强制合并绕过判断 | `models/git/protected_branch.go` | 212 |
| 官方审查者判断 | `models/git/protected_branch.go` | 234 |
| 检查队列 worker | `services/pull/check.go` | 462 |
| 执行合并操作 | `services/pull/merge.go` | 223 |
| Merge Box 阻断器计算 | `routers/web/repo/issue_view.go` | 1040 |
| canMergeNow 计算 | `routers/web/repo/issue_view.go` | 969 |
| Merge Box 图标颜色 | `routers/web/repo/pull_merge_box.go` | 54 |
| Merge Box 信息条目 | `routers/web/repo/pull_merge_box.go` | 91 |
| 合并表单属性 | `routers/web/repo/pull_merge_form.go` | 18 |
| 前端 Merge Box 组件 | `web_src/js/components/PullRequestMergeForm.vue` | 1 |
| InfoSections 互斥选择 | `routers/web/repo/pull_merge_box.go` | 190 |
| Status Check 摘要文案 | `routers/web/repo/pull.go` | 521 |
| Status Check 缺失检查收集 | `routers/web/repo/pull.go` | 452 |
| Status Check glob 匹配 | `routers/web/repo/pull.go` | 469 |
| RequiredChecksState 计算 | `routers/web/repo/pull.go` | 485 |
| 保护规则变更触发重检 | `services/pull/protected_branch.go` | - |
| Merge Box 模板 | `templates/repo/issue/view_content/pull_merge_box.tmpl` | 1 |
| Status Check 详情模板 | `templates/repo/issue/view_content/pull_merge_status_checks.tmpl` | 1 |
| 签名要求检查 | `routers/web/repo/issue_view.go` | 488 |
| 签名要求后端校验 | `services/pull/check.go` | 219 |
| 合并方式签名差异 | `services/pull/check.go` | 233 |
| SignMerge 函数 | `services/asymkey/sign.go` | - |
| ErrWontSign 定义 | `services/asymkey/sign.go` | - |
