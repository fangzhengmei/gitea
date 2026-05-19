# Issue 与 Pull Request 共享模型分析

## 一、数据模型层：共享底层结构

### 1. 核心设计思路

Issue 和 Pull Request (PR) 采用 **"主表扩展"** 模式共享底层数据结构：

- **主表**：`issue` 表，存储所有工单和 PR 共有的字段
- **扩展表**：`pull_request` 表，仅存储 PR 特有的字段
- 关联方式：`pull_request.issue_id` 外键关联到 `issue.id`

### 2. Issue 结构体（共享基础模型）

**文件位置**：`models/issues/issue.go:54-111`

```go
type Issue struct {
    ID                int64
    RepoID            int64
    Index             int64      // 仓库内序号
    PosterID          int64      // 创建者
    Title             string     // 标题
    Content           string     // 内容
    Labels            []*Label   // 标签
    MilestoneID       int64      // 里程碑
    Assignees         []*user_model.User // 负责人
    IsClosed          bool       // 是否关闭
    IsPull            bool       // 【关键区分字段】是否为 PR
    PullRequest       *PullRequest `xorm:"-"` // 关联的 PR 扩展数据
    NumComments       int
    DeadlineUnix      timeutil.TimeStamp
    CreatedUnix       timeutil.TimeStamp
    UpdatedUnix       timeutil.TimeStamp
    ClosedUnix        timeutil.TimeStamp
    IsLocked          bool
    TimeEstimate      int64
}
```

**关键点**：
- `IsPull` 字段（第80行）是区分工单和 PR 的核心标识
- 所有共享字段（标题、内容、状态、标签、里程碑、负责人等）都定义在此
- `PullRequest` 字段是懒加载的扩展数据指针

### 3. PullRequest 结构体（PR 扩展模型）

**文件位置**：`models/issues/pull.go:119-155`

```go
type PullRequest struct {
    ID              int64
    Type            PullRequestType
    Status          PullRequestStatus // 合并状态（冲突、可合并等）
    ConflictedFiles []string
    CommitsAhead    int
    CommitsBehind   int
    IssueID         int64           // 关联到 Issue 表
    Issue           *Issue `xorm:"-"`
    Index           int64           // 与 Issue.Index 同步
    HeadRepoID      int64           // 源仓库
    HeadBranch      string          // 源分支
    BaseRepoID      int64           // 目标仓库
    BaseBranch      string          // 目标分支
    HasMerged       bool            // 是否已合并
    MergedCommitID  string
    MergerID        int64
    MergedUnix      timeutil.TimeStamp
    Flow            PullRequestFlow
}
```

**关键点**：
- 仅包含 PR 特有的字段：分支信息、合并状态、代码审查相关等
- 通过 `IssueID` 与 Issue 表建立一对一关联
- `Index` 字段与关联 Issue 的 `Index` 保持一致

### 4. 双向关联机制

**Issue → PullRequest 加载**：`models/issues/issue.go:187-203`

```go
func (issue *Issue) LoadPullRequest(ctx context.Context) (err error) {
    if issue.IsPull {
        if issue.PullRequest == nil && issue.ID != 0 {
            issue.PullRequest, err = GetPullRequestByIssueID(ctx, issue.ID)
            // ...
        }
        if issue.PullRequest != nil {
            issue.PullRequest.Issue = issue
        }
    }
    return nil
}
```

**PullRequest → Issue 加载**：`models/issues/pull.go:329-339`

```go
func (pr *PullRequest) LoadIssue(ctx context.Context) (err error) {
    if pr.Issue != nil {
        return nil
    }
    pr.Issue, err = GetIssueByID(ctx, pr.IssueID)
    if err == nil {
        pr.Issue.PullRequest = pr
    }
    return err
}
```

**设计特点**：
- 双向引用，加载时自动建立关联
- 懒加载模式，按需加载扩展数据
- 避免循环加载问题

## 二、列表过滤层：区分两类对象

### 1. 过滤参数定义

**文件位置**：`models/issues/issue_search.go:28-57`

```go
type IssuesOptions struct {
    // ... 其他过滤条件
    IsClosed           optional.Option[bool]
    IsPull             optional.Option[bool]  // 【核心过滤字段】
    // ...
}
```

`IsPull` 使用 `optional.Option[bool]` 类型，支持三种状态：
- `Some(true)`：只查询 PR
- `Some(false)`：只查询工单
- `None()`：不区分，查询所有

### 2. SQL 条件构建

**文件位置**：`models/issues/issue.go:515-520`

```go
func isPullToCond(isPull optional.Option[bool]) builder.Cond {
    if isPull.Has() {
        return builder.Eq{"is_pull": isPull.Value()}
    }
    return builder.NewCond()
}
```

在查询时动态添加 `is_pull` 字段的过滤条件。

### 3. 路由层参数解析

**文件位置**：`routers/common/issue_filter.go:23-25`

```go
func ParseIssueFilterTypeIsPull(typ string) optional.Option[bool] {
    return optional.FromMapLookup(map[string]bool{"pulls": true, "issues": false}, typ)
}
```

**URL 路由映射**：
- `/owner/repo/issues` → `typ = "issues"` → `IsPull = Some(false)`
- `/owner/repo/pulls` → `typ = "pulls"` → `IsPull = Some(true)`

### 4. PR 独立列表查询

**文件位置**：`models/issues/pull_list.go:35-62`

PR 也有独立的查询接口，通过 JOIN 关联 issue 表：

```go
func listPullRequestStatement(ctx context.Context, baseRepoID int64, opts *PullRequestsOptions) *xorm.Session {
    sess := db.GetEngine(ctx).Where("pull_request.base_repo_id=?", baseRepoID)
    // ... 其他 PR 特有条件
    sess.Join("INNER", "issue", "pull_request.issue_id = issue.id")
    // 共享条件（状态、标签、里程碑等）直接使用 issue 表字段
    switch opts.State {
    case "closed", "open":
        sess.And("issue.is_closed=?", opts.State == "closed")
    }
    // ...
    return sess
}
```

## 三、操作路径：合并与关闭的分叉点

### 1. 共用关闭逻辑

**文件位置**：`models/issues/issue_update.go:51-86`

```go
func SetIssueAsClosed(ctx context.Context, issue *Issue, doer *user_model.User, isMergePull bool) (*Comment, error) {
    if issue.IsClosed {
        return nil, ErrIssueIsClosed{...}
    }
    // 检查依赖关系
    // ...
    
    issue.IsClosed = true
    issue.ClosedUnix = timeutil.TimeStampNow()
    
    // 更新数据库
    // ...
    
    // 【分叉点】根据 isMergePull 创建不同类型的评论
    return updateIssueNumbers(ctx, issue, doer, 
        util.Iif(isMergePull, CommentTypeMergePull, CommentTypeClose))
}
```

**关键参数**：`isMergePull` 决定了评论类型：
- `false` → `CommentTypeClose`：普通关闭
- `true` → `CommentTypeMergePull`：合并关闭

### 2. 普通 Issue 关闭路径

**调用链路**：
```
用户点击"关闭工单"按钮
    ↓
routers/web/repo/issue.go 路由处理
    ↓
issue_service.CloseIssue()  [services/issue/status.go:17-40]
    ↓
issues_model.CloseIssue()  [models/issues/issue_update.go:167-178]
    ↓
SetIssueAsClosed(..., isMergePull=false)
    ↓
创建 CommentTypeClose 评论
```

**关键代码**：`services/issue/status.go:17-40`

```go
func CloseIssue(ctx context.Context, issue *issues_model.Issue, doer *user_model.User, commitID string) error {
    var comment *issues_model.Comment
    if err := db.WithTx(ctx, func(ctx context.Context) error {
        var err error
        comment, err = issues_model.CloseIssue(ctx, issue, doer)
        // ...
    }); err != nil {
        return err
    }
    notify_service.IssueChangeStatus(ctx, doer, commitID, issue, comment, true)
    return nil
}
```

### 3. PR 合并路径

**调用链路**：
```
用户点击"合并 PR"按钮
    ↓
routers/web/repo/pull.go:MergePullRequest()  [第1034行]
    ↓
pull_service.Merge()  [services/pull/merge.go:223-300]
    ↓
doMergeAndPush() → 执行 Git 合并操作
    ↓
（Git 后接收钩子触发）SetMerged()  [services/pull/merge.go:682-736]
    ↓
SetIssueAsClosed(..., isMergePull=true)
    ↓
创建 CommentTypeMergePull 评论
```

**PR 合并的额外操作**：`services/pull/merge.go:682-736`

```go
func SetMerged(ctx context.Context, pr *issues_model.PullRequest, ...) (bool, error) {
    // 更新 PR 合并状态
    pr.HasMerged = true
    pr.MergedCommitID = mergedCommitID
    pr.MergedUnix = mergedTimeStamp
    pr.MergerID = merger.ID
    pr.Status = mergeStatus
    
    return db.WithTx2(ctx, func(ctx context.Context) (bool, error) {
        // 1. 关闭关联的 Issue（isMergePull=true）
        if _, err := issues_model.SetIssueAsClosed(ctx, pr.Issue, pr.Merger, true); err != nil {
            return false, fmt.Errorf("ChangeIssueStatus: %w", err)
        }
        
        // 2. 更新 PullRequest 表的合并字段
        if cnt, err := db.GetEngine(ctx).Where("id = ?", pr.ID).
            And("has_merged = ?", false).
            Cols("has_merged, status, merge_base, merged_commit_id, merger_id, merged_unix, conflicted_files").
            Update(pr); err != nil {
            // ...
        }
        return true, nil
    })
}
```

### 4. 分叉点对比

| 操作 | 调用函数 | isMergePull 参数 | 评论类型 | 额外操作 |
|------|---------|-----------------|----------|---------|
| 关闭工单 | `CloseIssue()` | `false` | `CommentTypeClose` | 无 |
| 合并 PR | `SetMerged()` → `SetIssueAsClosed()` | `true` | `CommentTypeMergePull` | 更新 PR 合并字段、执行 Git 合并 |

### 5. 评论类型区分

在界面展示时，通过评论类型区分不同的关闭原因：

**文件位置**：`models/issues/issue.go:462-470`

```go
func (issue *Issue) GetLastEventLabel() string {
    if issue.IsClosed {
        if issue.IsPull && issue.PullRequest.HasMerged {
            return "repo.pulls.merged_by"  // "已由 XX 合并"
        }
        return "repo.issues.closed_by"     // "已由 XX 关闭"
    }
    return "repo.issues.opened_by"
}
```

## 四、三段协作总结

### 1. 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    界面与接口层                               │
│  ┌─────────────┐          ┌─────────────┐                    │
│  │ /issues     │          │ /pulls      │                    │
│  │ 路由        │          │ 路由        │                    │
│  └──────┬──────┘          └──────┬──────┘                    │
│         │                        │                           │
│         ▼                        ▼                           │
│  ParseIssueFilterTypeIsPull()  MergePullRequest()             │
└─────────┬────────────────────────┬───────────────────────────┘
          │                        │
┌─────────▼────────────────────────▼───────────────────────────┐
│                    业务逻辑层（services）                      │
│  ┌─────────────┐          ┌─────────────┐                    │
│  │ issue       │          │ pull        │                    │
│  │ service     │          │ service     │                    │
│  │ CloseIssue()│          │ Merge()     │                    │
│  └──────┬──────┘          └──────┬──────┘                    │
│         │                        │                           │
│         ▼                        ▼                           │
│  CloseIssue()             SetMerged() + Git 合并              │
└─────────┬────────────────────────┬───────────────────────────┘
          │                        │
┌─────────▼────────────────────────▼───────────────────────────┐
│                    数据模型层（models）                        │
│  ┌───────────────────────────────────────────────────┐      │
│  │ Issue 结构体（共享字段）                            │      │
│  │ - ID, Title, Content, IsClosed, Labels...         │      │
│  │ - IsPull: 区分字段                                 │      │
│  │ - PullRequest: 扩展数据指针                        │      │
│  └───────────────────┬───────────────────────────────┘      │
│                      │                                      │
│  ┌───────────────────▼───────────────────────────────┐      │
│  │ PullRequest 结构体（PR 特有字段）                   │      │
│  │ - IssueID: 外键关联                               │      │
│  │ - HeadBranch, BaseBranch, HasMerged...            │      │
│  └───────────────────────────────────────────────────┘      │
│                                                             │
│  ┌───────────────────────────────────────────────────┐      │
│  │ SetIssueAsClosed(isMergePull)                     │      │
│  │  - 共用关闭逻辑                                   │      │
│  │  - 分叉点：根据参数创建不同评论类型                │      │
│  └───────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 2. 设计优点

1. **数据复用**：工单和 PR 共享大部分字段，避免数据冗余
2. **查询统一**：列表查询使用同一套过滤逻辑，通过 `IsPull` 字段区分
3. **扩展灵活**：PR 特有字段独立存储，不影响工单表结构
4. **操作复用**：关闭逻辑共用，通过参数区分具体操作类型

### 3. 关键代码位置索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| Issue 结构体定义 | `models/issues/issue.go` | 54-111 |
| PullRequest 结构体定义 | `models/issues/pull.go` | 119-155 |
| IsPull 过滤条件转换 | `models/issues/issue.go` | 515-520 |
| 路由层类型解析 | `routers/common/issue_filter.go` | 23-25 |
| 共用关闭逻辑 | `models/issues/issue_update.go` | 51-86 |
| 工单关闭服务 | `services/issue/status.go` | 17-40 |
| PR 合并服务 | `services/pull/merge.go` | 223-300 |
| PR 合并后状态设置 | `services/pull/merge.go` | 682-736 |
| 事件标签展示 | `models/issues/issue.go` | 462-470 |
