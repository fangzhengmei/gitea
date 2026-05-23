# 仓库镜像（Repository Mirror）流程分析

## 一、核心数据模型

### 1.1 拉取镜像模型 (`Mirror`)

**文件**: `models/repo/mirror.go:21-36`

```go
type Mirror struct {
    ID          int64
    RepoID      int64
    Interval    time.Duration    // 同步间隔
    EnablePrune bool             // 是否启用修剪
    UpdatedUnix    timeutil.TimeStamp
    NextUpdateUnix timeutil.TimeStamp  // 下次更新时间
    LastSyncUnix   timeutil.TimeStamp  // 上次同步时间
    LFS         bool             // 是否同步 LFS
    LFSEndpoint string           // LFS 端点
    RemoteAddress string         // 远程地址（已清除凭据）
}
```

### 1.2 推送镜像模型 (`PushMirror`)

**文件**: `models/repo/pushmirror.go:20-32`

```go
type PushMirror struct {
    ID            int64
    RepoID        int64
    RemoteName    string         // 远程名称
    RemoteAddress string         // 远程地址
    SyncOnCommit  bool           // 提交时自动同步
    Interval      time.Duration  // 同步间隔
    LastUpdateUnix timeutil.TimeStamp
    LastError     string         // 上次错误信息
}
```

### 1.3 全局配置

**文件**: `modules/setting/mirror.go:13-25`

```go
var Mirror = struct {
    Enabled         bool
    DisableNewPull  bool
    DisableNewPush  bool
    DefaultInterval time.Duration  // 默认 8 小时
    MinInterval     time.Duration  // 最小 10 分钟
}
```

---

## 二、拉取镜像（Pull Mirror）流程

### 2.1 核心入口函数

**文件**: `services/mirror/mirror_pull.go:269-427`

`SyncPullMirror(ctx, repoID)` 是拉取镜像的入口函数，执行流程：

1. **全局锁保护**: 使用 `globallock.Lock("repo_pull_mirror_{repoID}")` 防止并发同步
2. **进程上下文**: 通过 `process.GetManager().AddContext()` 跟踪同步进程
3. **执行同步**: 调用 `runSync()` 执行实际同步
4. **调度下次更新**: 成功后调用 `ScheduleNextUpdate()` 计算下次同步时间
5. **通知机制**: 对变更的引用发送通知（创建/删除/推送）

### 2.2 同步执行逻辑 (`runSync`)

**文件**: `services/mirror/mirror_pull.go:109-262`

#### 2.2.1 Git 拉取命令
```go
cmd := gitcmd.NewCommand("fetch", "--tags")
if m.EnablePrune {
    cmd.AddArguments("--prune")
}
cmd.AddDynamicArguments(remoteName)
```

#### 2.2.2 可恢复错误检测 (`checkRecoverableSyncError`)

**文件**: `services/mirror/mirror_pull.go:91-106`

检测以下可自动恢复的错误类型：
- `unable to resolve reference` + `reference broken` - 引用损坏
- `remote error` + `not our ref` - 远程引用在拉取过程中被删除（竞态条件）
- `cannot lock ref` + `but expected` - 强制推送导致的预期值不匹配
- `cannot lock ref` + `unable to resolve reference` - 引用解析失败
- `Unable to create` + `.lock` - 锁文件冲突（与本地 GC 等操作竞态）

#### 2.2.3 冲突处理与重试机制

```go
if checkRecoverableSyncError(fetchStderr) {
    // 尝试修剪损坏的引用
    pruneErr := pruneBrokenReferences(ctx, m, m.Repo, timeout)
    if pruneErr == nil {
        // 重新尝试拉取
        fetchStdout, fetchStderr, err = gitrepo.RunCmdString(ctx, m.Repo, cmdFetch())
    }
}
```

#### 2.2.4 Wiki 同步
如果仓库启用了 Wiki，会对 Wiki 仓库执行相同的同步流程。

#### 2.2.5 LFS 同步
```go
if m.LFS && setting.LFS.StartServer {
    endpoint := lfs.DetermineEndpoint(remoteURL.String(), m.LFSEndpoint)
    lfsClient := lfs.NewClient(endpoint, migrations.NewMigrationHTTPTransport())
    repo_module.StoreMissingLfsObjectsInRepository(ctx, m.Repo, gitRepo, lfsClient)
}
```

### 2.3 引用差异检测

#### 2.3.1 分支同步 (`SyncRepoBranchesWithRepo`)

**文件**: `modules/repository/branch.go:47-170`

检测逻辑：
1. 获取 Git 仓库中的所有分支（`allBranches`）
2. 获取数据库中的所有分支记录（`dbBranches`）
3. 三类差异：
   - **新增**: Git 中有，数据库中无 → `toAdd`
   - **更新**: Git 和数据库都有，但 CommitID 不同 → `toUpdate`
   - **删除**: 数据库中有，Git 中无且未标记删除 → `toRemove`

返回 `SyncResult` 数组：
```go
type SyncResult struct {
    RefName     git.RefName  // 引用全名
    OldCommitID string       // 旧 Commit ID（空表示新增）
    NewCommitID string       // 新 Commit ID（空表示删除）
}
```

#### 2.3.2 标签/Release 同步 (`SyncReleasesWithTags`)

**文件**: `modules/repository/repo.go:182-286`

核心算法 `calcSync` (`modules/repository/repo.go:288-315`)：
```go
func calcSync(destTags []*git.Tag, dbTags []*shortRelease) (
    inserted []*git.Tag,    // 新增标签
    deleted []int64,        // 删除的 Release ID
    updated []*git.Tag,     // 更新的标签
)
```

同步策略：
- **目标**: 使本地 Release 集合与上游 Tag 集合完全一致
- **优势**: 对于有大量 Tag 的仓库（如 vim 有 13000+ tags）效率更高
- **仅处理 IsTag=true 的 Release**，避免删除用户手动创建的 Release

### 2.4 空仓库处理

**文件**: `services/mirror/mirror_pull.go:429-484`

首次同步成功后：
1. 检查是否为空仓库（`IsEmpty`）
2. 自动设置默认分支优先级：配置的默认分支 > master > main > 第一个分支
3. 更新 `IsEmpty = false` 和 `default_branch`

---

## 三、推送镜像（Push Mirror）流程

### 3.1 核心入口函数

**文件**: `services/mirror/mirror_push.go:78-121`

`SyncPushMirror(ctx, mirrorID)` 执行流程：
1. 从数据库获取 PushMirror 记录
2. 清除上次错误信息（`LastError = ""`）
3. 调用 `runPushSync()` 执行推送
4. 记录错误信息（`stripExitStatus` 正则去除 "exit status X - " 前缀）
5. 更新 `LastUpdateUnix` 时间戳
6. 保存到数据库

### 3.2 推送执行逻辑 (`runPushSync`)

**文件**: `services/mirror/mirror_push.go:123-189`

#### 3.2.1 推送配置
推送镜像通过 `git remote add --mirror=push` 配置，额外配置推送规则：
```go
gitrepo.GitConfigAdd(ctx, storageRepo, "remote."+m.RemoteName+".push", "+refs/heads/*:refs/heads/*")
gitrepo.GitConfigAdd(ctx, storageRepo, "remote."+m.RemoteName+".push", "+refs/tags/*:refs/tags/*")
```
- `+` 前缀表示强制推送（允许非快进更新）
- 推送所有分支和标签

#### 3.2.2 推送命令
```go
gitrepo.PushToExternal(ctx, storageRepo, git.PushOptions{
    Remote:  m.RemoteName,
    Force:   true,
    Mirror:  true,
    Timeout: timeout,
    Env:     envs,
})
```

#### 3.2.3 LFS 推送 (`pushAllLFSObjects`)

**文件**: `services/mirror/mirror_push.go:191-254`

批量上传策略：
1. 遍历仓库所有 LFS 指针对象
2. 检查本地内容存储是否存在
3. 按 `BatchSize()` 批量上传（默认批处理）
4. 上传回调函数负责读取本地内容

### 3.3 提交触发同步

**文件**: `services/mirror/notifier.go:25-31`

通过注册 `mirrorNotifier` 监听推送事件：
```go
func (m *mirrorNotifier) PushCommits(ctx, ...) {
    syncPushMirrorWithSyncOnCommit(ctx, repo.ID)
}

func syncPushMirrorWithSyncOnCommit(ctx, repoID) {
    pushMirrors, _ := repo_model.GetPushMirrorsSyncedOnCommit(ctx, repoID)
    for _, mirror := range pushMirrors {
        AddPushMirrorToQueue(mirror.ID)  // 加入队列异步执行
    }
}
```

---

## 四、定时同步与任务调度

### 4.1 Cron 任务注册

**文件**: `services/cron/tasks_basic.go:24-43`

```go
func registerUpdateMirrorTask() {
    RegisterTaskFatal("update_mirrors", &UpdateMirrorTaskConfig{
        BaseConfig: BaseConfig{
            Enabled:    true,
            RunAtStart: false,
            Schedule:   "@every 10m",  // 每 10 分钟检查一次
        },
        PullLimit: 50,  // 每次最多处理 50 个拉取镜像
        PushLimit: 50,  // 每次最多处理 50 个推送镜像
    }, func(ctx context.Context, ...) error {
        return mirror_service.Update(ctx, umtc.PullLimit, umtc.PushLimit)
    })
}
```

### 4.2 镜像更新调度 (`Update`)

**文件**: `services/mirror/mirror.go:35-119`

执行流程：
1. 遍历拉取镜像：`MirrorsIterate()` - 查找 `next_update_unix <= now` 的镜像
2. 遍历推送镜像：`PushMirrorsIterate()` - 查找 `last_update + interval <= now` 的镜像
3. 将符合条件的镜像加入队列：`PushToQueue(mirrorType, referenceID)`

### 4.3 迭代查询条件

#### 拉取镜像迭代 (`models/repo/mirror.go:109-118`)
```go
Where("next_update_unix<=?", time.Now().Unix()).
And("next_update_unix!=0").
OrderBy("updated_unix ASC")
```

#### 推送镜像迭代 (`models/repo/pushmirror.go:140-152`)
```go
Where("`push_mirror`.last_update + (`push_mirror`.`interval` / ?) <= ?", time.Second, time.Now().Unix()).
And("`push_mirror`.`interval` != 0").
And("`repository`.is_archived = ?", false).
OrderBy("last_update ASC")
```

---

## 五、队列机制

### 5.1 唯一工作池队列

**文件**: `services/mirror/queue.go:13-77`

```go
var mirrorQueue *queue.WorkerPoolQueue[*SyncRequest]

type SyncRequest struct {
    Type        SyncType  // PullMirrorType / PushMirrorType
    ReferenceID int64     // 拉取: RepoID; 推送: MirrorID
}
```

### 5.2 队列特性
- **唯一队列**: 使用 `CreateUniqueQueue()` 确保同一镜像不会重复入队
- **去重检查**: `queue.ErrAlreadyInQueue` 时跳过并记录日志
- **异步入队**: `addMirrorToQueue()` 使用 goroutine 异步入队，避免阻塞
- **优雅关闭**: 通过 `graceful.GetManager().RunWithCancel()` 支持优雅关闭

---

## 六、凭据安全处理

### 6.1 凭据存储机制

Git 凭据通过 **git remote URL 内嵌** 方式存储，而非使用 credential helper：

1. **添加远程时包含凭据**:
   ```
   git remote add origin https://user:password@github.com/org/repo.git
   ```

2. **数据库存储时清除凭据**:
   **文件**: `services/mirror/mirror_pull.go:67-70`
   ```go
   u.User = nil  // 清除 URL 中的用户信息
   m.Repo.OriginalURL = u.String()
   ```

### 6.2 日志安全

**文件**: `modules/util/sanitize.go:37-127`

`SanitizeCredentialURLs()` 函数会：
- 识别 URL 中的 `user:pass@host` 模式
- 替换为 `(masked)@host`
- 支持 HTTP/HTTPS 协议、IPv4/IPv6 地址
- 符合 RFC 3986 userinfo 字符集规范

应用场景：
- Git 命令日志输出
- 错误消息（`SanitizeErrorCredentialURLs`）
- 所有可能包含 URL 的日志输出

### 6.3 禁用交互式凭据提示

**文件**: `modules/git/gitcmd/command.go:251`
```go
"GIT_TERMINAL_PROMPT=0"  // 全局环境变量，禁止 git 交互提示
```

---

## 七、失败处理与重试策略

### 7.1 可恢复错误自动重试

| 错误类型 | 处理方式 | 重试机制 |
|---------|---------|---------|
| 引用损坏 (`reference broken`) | `git remote prune` 清理后重试 | 同步内立即重试 |
| 远程引用消失 (`not our ref`) | 同上 | 同步内立即重试 |
| 锁文件冲突 (`.lock`) | 同上 | 同步内立即重试 |
| 强制推送竞态 (`but expected`) | 同上 | 同步内立即重试 |

### 7.2 不可恢复错误

| 错误类型 | 处理方式 |
|---------|---------|
| 认证失败 (`Authentication failed`) | 记录错误，等待下次 cron 调度 |
| 远程仓库不存在 | 记录错误，等待下次 cron 调度 |
| 代理连接失败 | 记录错误，等待下次 cron 调度 |
| 其他网络错误 | 记录错误，等待下次 cron 调度 |

### 7.3 重试机制总结

1. **同步内重试**: 可恢复错误在一次 `runSync` 内尝试 prune + 重跑
2. **定时调度重试**: 所有失败的镜像会在下次 cron 检查（每 10 分钟）时重新入队
3. **间隔控制**: 每个镜像有独立的 `Interval`，不会过于频繁重试
4. **错误记录**: 推送镜像会保存 `LastError` 供用户查看

---

## 八、关键代码位置索引

| 功能模块 | 文件位置 | 关键函数 |
|---------|---------|---------|
| 拉取镜像入口 | `services/mirror/mirror_pull.go` | `SyncPullMirror`, `runSync` |
| 推送镜像入口 | `services/mirror/mirror_push.go` | `SyncPushMirror`, `runPushSync` |
| 可恢复错误检测 | `services/mirror/mirror_pull.go` | `checkRecoverableSyncError` |
| 引用修剪 | `services/mirror/mirror_pull.go` | `pruneBrokenReferences` |
| 分支差异检测 | `modules/repository/branch.go` | `SyncRepoBranchesWithRepo` |
| 标签差异检测 | `modules/repository/repo.go` | `SyncReleasesWithTags`, `calcSync` |
| 定时调度 | `services/cron/tasks_basic.go` | `registerUpdateMirrorTask` |
| 调度主逻辑 | `services/mirror/mirror.go` | `Update` |
| 队列管理 | `services/mirror/queue.go` | `StartSyncMirrors`, `PushToQueue` |
| 凭据清理 | `modules/util/sanitize.go` | `SanitizeCredentialURLs` |
| 提交触发推送 | `services/mirror/notifier.go` | `mirrorNotifier` |
| 数据模型 | `models/repo/mirror.go` | `Mirror`, `PushMirror` |
| 全局配置 | `modules/setting/mirror.go` | `Mirror` |

---

## 九、测试用例

### 9.1 拉取镜像测试

**文件**: `tests/integration/mirror_pull_test.go:26-126`

测试场景：
- 创建拉取镜像仓库
- 源仓库新增 tag 后同步验证
- 源仓库删除 tag 后同步验证
- 无效远程地址的错误处理
- `LastSyncUnix` 时间戳验证

### 9.2 推送镜像测试

**文件**: `tests/integration/mirror_push_test.go:30-188`

测试场景：
- 创建推送镜像并同步验证
- Wiki 默认分支不匹配场景
- 推送镜像间隔更新
- 跨仓库操作权限控制（防止越权修改）
- 推送镜像删除验证

### 9.3 可恢复错误检测测试

**文件**: `services/mirror/mirror_pull_test.go:12-38`

测试 6 种可恢复错误 + 3 种不可恢复错误的识别准确性。
