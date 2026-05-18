# 分支保护规则在合并请求闭合前的全链路检查机制

## 概述

Gitea 的分支保护规则（Branch Protection Rule）在合并请求（Pull Request）闭合前需要经过多层检查，包括规则匹配、权限验证、评审检查、状态检查、签名验证以及与 Git 钩子脚本的协作。本文档详细说明这一全链路流程。

---

## 一、规则匹配机制

### 1.1 规则数据结构

分支保护规则定义在 `models/git/protected_branch.go:29-69`，核心字段包括：

- `RuleName`：规则名称，可以是精确分支名或 glob 模式
- `Priority`：规则优先级，数值越小优先级越高
- `EnableWhitelist`：是否启用推送白名单
- `EnableMergeWhitelist`：是否启用合并白名单
- `EnableStatusCheck`：是否启用状态检查
- `RequiredApprovals`：所需的批准数量
- `BlockOnRejectedReviews`：是否阻止有拒绝评审的合并
- `RequireSignedCommits`：是否要求签名提交

### 1.2 规则匹配算法

规则匹配流程在 `models/git/protected_branch_list.go:17-24` 中实现：

```
1. 加载仓库所有保护规则
2. 按优先级排序：
   - 首先按 Priority 字段排序（升序）
   - 优先级相同时，精确名称规则优先于 glob 规则
   - 仍相同时，按创建时间排序（先创建的优先）
3. 遍历排序后的规则，返回第一个匹配的规则
```

匹配逻辑在 `models/git/protected_branch.go:103-111`：

- **精确匹配**：如果规则名不含特殊字符，使用大小写不敏感的字符串比较
- **Glob 匹配**：如果规则名含特殊字符（`*`, `?`, `[` 等），使用 glob 模式匹配

核心匹配函数：`GetFirstMatchProtectedBranchRule(ctx, repoID, branchName)`

---

## 二、合并请求闭合前的主检查流程

主检查入口在 `services/pull/check.go:142-229` 的 `CheckPullMergeable` 函数。

### 2.1 基础检查（前置条件）

| 检查项 | 代码位置 | 说明 |
|--------|----------|------|
| 是否已合并 | `check.go:144-146` | `pr.HasMerged` 为 true 时返回 `ErrHasMerged` |
| 是否已关闭 | `check.go:148-153` | 关联 issue 已关闭时返回 `ErrIsClosed` |
| 合并权限 | `check.go:155-160` | 调用 `IsUserAllowedToMerge` 检查用户是否有权限合并 |
| 手动合并标记 | `check.go:162-165` | `MergeCheckTypeManually` 类型跳过后续检查 |
| WIP 状态 | `check.go:167-169` | 工作进行中 PR 返回 `ErrIsWorkInProgress` |
| 可合并状态 | `check.go:171-173` | 非可合并状态且非空 PR 返回 `ErrNotMergeableState` |
| 检查中状态 | `check.go:175-177` | 正在检查冲突时返回 `ErrIsChecking` |

### 2.2 分支保护规则检查

调用 `CheckPullBranchProtections` 函数（`services/pull/merge.go:569-614`）进行详细检查。

#### 2.2.1 状态检查

`IsPullCommitStatusPass`（`services/pull/commit_status.go:69-83`）：

1. 获取匹配的保护规则
2. 如果未启用状态检查，直接通过
3. 获取 PR 头部提交的所有状态检查结果
4. 调用 `MergeRequiredContextsCommitStatus` 计算最终状态
5. 只有所有必需的上下文状态都为 `success` 时才通过

#### 2.2.2 评审检查

| 检查项 | 代码位置 | 说明 |
|--------|----------|------|
| 足够的批准数 | `pull.go:761-766` | `HasEnoughApprovals` 统计官方且未被驳回的批准数 |
| 拒绝评审阻止 | `pull.go:787-802` | `MergeBlockedByRejectedReview` 检查是否存在未驳回的拒绝评审 |
| 官方评审请求 | `pull.go:806-820` | `MergeBlockedByOfficialReviewRequests` 检查是否有待处理的官方评审请求 |
| 分支过时检查 | `pull.go:823-825` | `MergeBlockedByOutdatedBranch` 检查 head 分支是否落后于 base 分支 |

批准数统计逻辑（`GetGrantedApprovalsCount`）：
- 只统计 `official = true` 且 `dismissed = false` 的批准
- 如果 `IgnoreStaleApprovals` 为 true，还需满足 `stale = false`

#### 2.2.3 受保护文件检查

`MergeBlockedByProtectedFiles`（`protected_branch.go:261-268`）：
- 检查 PR 是否修改了受保护的文件
- 受保护文件模式支持 glob 匹配，多个模式用分号分隔

### 2.3 签名检查

`checkSigningRequirements`（`services/pull/check.go:239-270`）根据合并策略进行不同的签名检查：

| 合并策略 | 签名要求 |
|----------|----------|
| Fast-Forward Only | 所有头部提交必须已验证签名 |
| Merge | 头部提交已验证 + Gitea 能签名合并提交 |
| Rebase/Rebase-Merge/Squash | Gitea 能签名重写后的提交 |

### 2.4 依赖检查

`IssueNoDependenciesLeft` 检查是否所有依赖项都已完成。

---

## 三、绕过权限机制

### 3.1 管理员强制合并

在 `CheckPullMergeable` 中（`check.go:192-209`），当分支保护检查失败时：

1. 如果是自动合并（`MergeCheckTypeAuto`），跳过保护检查
2. 如果是管理员强制合并（`adminForceMerge = true`）：
   - 检查用户是否为仓库管理员
   - 检查保护规则是否设置了 `BlockAdminMergeOverride`
   - 只有管理员且未阻止管理员覆盖时，才允许绕过

### 3.2 合并白名单

`IsUserMergeWhitelisted`（`protected_branch.go:185-205`）：

- 如果未启用合并白名单，只要有代码写入权限即可
- 如果启用了白名单，用户 ID 必须在 `MergeWhitelistUserIDs` 中，或所属团队在 `MergeWhitelistTeamIDs` 中

### 3.3 官方评审者白名单

`IsUserOfficialReviewer`（`protected_branch.go:208-233`）：

- 如果未启用批准白名单，任何有写入权限的用户都可作为官方评审者
- 如果启用了白名单，用户必须在批准白名单中

---

## 四、与 Git 钩子脚本的协作

### 4.1 Pre-Receive 钩子

`routers/private/hook_pre_receive.go:109-140` 是推送时的核心检查点，在 Git 接收推送前执行。

#### 4.1.1 分支推送检查流程

`preReceiveBranch` 函数（`hook_pre_receive.go:142-404`）：

```
1. 检查写入权限
2. 检查是否为默认分支删除（禁止）
3. 获取匹配的分支保护规则
   ├─ 无规则：直接通过
   └─ 有规则：继续检查
4. 禁止删除受保护分支
5. 检测并阻止强制推送（除非规则允许）
6. 签名提交验证（如果要求）
7. 受保护文件变更检查
8. 推送权限检查：
   ├─ 部署密钥：检查白名单设置
   └─ 普通用户：调用 CanUserPush/CanUserForcePush
9. 如果无直接推送权限：
   ├─ 非 PR 合并：拒绝推送
   └─ PR 合并：检查合并权限和分支保护
```

#### 4.1.2 PR 合并时的钩子检查

当推送是 PR 合并操作时（`PullRequestID != 0`）：

1. 调用 `IsUserAllowedToMerge` 检查合并权限
2. 如果是仓库管理员，跳过后续状态和评审检查
3. 非管理员则调用 `CheckPullBranchProtections` 进行完整检查

### 4.2 钩子与 UI 检查的关系

**双重检查机制**：

1. **UI/API 层检查**：在用户点击合并按钮时，`CheckPullMergeable` 进行完整检查
2. **钩子层检查**：在实际推送（`git push`）时，`preReceiveBranch` 再次检查

这种设计确保了：
- UI 能提前给出友好的错误提示
- 即使绕过 UI 直接推送，钩子层仍能强制执行保护规则

---

## 五、全链路时序图

```
用户点击合并按钮
    │
    ▼
CheckPullMergeable ──────────┐
    │                        │
    ├─ 基础状态检查          │
    ├─ 合并权限检查          │
    ├─ CheckPullBranchProtections │
    │   ├─ 状态检查          │
    │   ├─ 评审检查          │
    │   ├─ 受保护文件检查    │
    │   └─ 过时分支检查      │
    ├─ 签名要求检查          │
    └─ 依赖检查              │
    │                        │
    ▼                        │
Merge 函数                   │  UI/API 层
    │                        │
    ├─ 合并策略验证          │
    ├─ 创建临时仓库          │
    ├─ 执行合并操作          │
    └─ git push origin       │
    │                        │
    ▼                        │
Git 服务器接收推送           │
    │                        │
    ▼                        │
pre-receive 钩子 ◄───────────┘
    │
    ├─ 获取分支保护规则
    ├─ 禁止删除检查
    ├─ 强制推送检查
    ├─ 签名验证
    ├─ 受保护文件检查
    ├─ 推送权限检查
    └─ PR 合并检查（再次调用 CheckPullBranchProtections）
    │
    ▼
post-receive 钩子
    │
    ├─ 更新 PR 状态为已合并
    ├─ 发送通知
    └─ 触发相关 Webhook
```

---

## 六、关键代码路径汇总

| 功能模块 | 主要文件 | 核心函数 |
|----------|----------|----------|
| 规则匹配 | `models/git/protected_branch.go` | `Match`, `CanUserPush`, `IsUserMergeWhitelisted` |
| 规则列表 | `models/git/protected_branch_list.go` | `GetFirstMatchProtectedBranchRule`, `sort` |
| 合并检查 | `services/pull/check.go` | `CheckPullMergeable`, `checkSigningRequirements` |
| 分支保护检查 | `services/pull/merge.go` | `CheckPullBranchProtections` |
| 状态检查 | `services/pull/commit_status.go` | `IsPullCommitStatusPass`, `MergeRequiredContextsCommitStatus` |
| 评审检查 | `models/issues/pull.go` | `HasEnoughApprovals`, `MergeBlockedByRejectedReview` |
| 钩子协作 | `routers/private/hook_pre_receive.go` | `HookPreReceive`, `preReceiveBranch` |
| 合并执行 | `services/pull/merge.go` | `Merge`, `doMergeAndPush` |

---

## 七、设计特点

1. **规则优先级机制**：通过 Priority 字段和精确/glob 排序，确保最具体的规则先匹配
2. **双重检查保障**：UI 层和钩子层分别检查，防止绕过
3. **灵活的白名单体系**：支持用户和团队维度的推送、合并、评审白名单
4. **管理员覆盖机制**：允许仓库管理员在必要时绕过保护规则（可配置是否阻止）
5. **合并策略感知**：不同合并策略对应不同的签名检查要求
6. **可扩展性**：通过 glob 模式支持复杂的分支和文件匹配规则
