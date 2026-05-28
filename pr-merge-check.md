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

### 8.2 Commit-level 阻断（不可绕过）

**位置**: `routers/web/repo/pull_merge_box.go:109-144`

| 状态 | 判断逻辑 | UI 文案 |
|------|---------|---------|
| 冲突 | `pull.IsFilesConflicted()` | "This branch has conflicts that must be resolved" + 文件列表 |
| 数据损坏 | `prInfo.IsPullRequestBroken` | "Data broken" |
| 检查中 | `pull.IsChecking()` | "This pull request is still being checked" |
| 祖先提交 | `pull.IsAncestor()` | "The head commit is already in the base branch" |
| 不可合并 | `!pull.IsStatusMergeable() && !pull.IsEmpty()` | "This pull request can't be merged" + "Ask someone with write access to merge this pull request manually" |
| 空 PR | `pull.IsEmpty()` | "This pull request is empty" |

**关键区别**: Commit-level 阻断（冲突、检查中、祖先）属于 `infoCommitBlockers`，**不可通过管理员权限绕过**；Protection-level 阻断（审批、status check 等）属于 `infoProtectionBlockers`，**可绕过**。

### 8.3 按钮禁用状态逻辑

**前端组件**: `web_src/js/components/PullRequestMergeForm.vue`

**核心变量传递**:
```go
// routers/web/repo/issue_view.go:968-970
data.canMergeNow = (!data.hasOverridableBlockers || data.canBypassProtection) &&
    (!data.requireSigned || data.willSign)
```

| 场景 | `canMergeNow` | 按钮行为 | 可用操作 |
|------|--------------|---------|---------|
| 无任何阻断 | `true` | 启用（绿色） | 立即合并 |
| 有阻断但可绕过 | `true` | 启用（红色 "Force Merge"） | 强制合并 |
| 有阻断不可绕过 | `false` | 禁用/隐藏 | 仅可设置自动合并 |
| 正在检查中 | 取决于 `hasOverridableBlockers` | 检查中状态 | 等待 |
| 有冲突 | `false` (commit blocker) | 禁用 | 手动合并（如启用） |

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

### 8.4 Merge Box 图标颜色规则

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

## 十一、关键代码位置速查表

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
