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
4. **调度下次更新**:
   - **成功**: 调用 `ScheduleNextUpdate()` 计算下次同步时间（`Now + Interval`），并更新 `LastSyncUnix = UpdatedUnix`
   - **失败**: 仅调用 `TouchMirror()` 更新 `UpdatedUnix`，**不修改 `NextUpdateUnix`**，保持原调度时间不变
5. **通知机制**: 对变更的引用发送通知（创建/删除/推送）

#### 失败后的调度重试时机
**文件**: `services/mirror/mirror_pull.go:299-303`

```go
if !ok {
    if err = repo_model.TouchMirror(ctx, m); err != nil {
        log.Error("SyncMirrors [repo: %-v]: failed to TouchMirror: %v", m.Repo, err)
    }
    return false
}
```

- `TouchMirror()` 只更新 `updated_unix` 时间戳，不改变 `next_update_unix`
- 因此失败后，下次 cron 检查（每 10 分钟）时，若 `next_update_unix <= now` 会自动重新入队重试
- 无论成功失败，**都不会立即重新入队**，必须等到原调度时间或下一次 cron 检查

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

**实现与测试对照**:
- 实现中有 **5 个 case** 返回 `true`（可恢复）
- 测试覆盖 **5 个可恢复场景 + 3 个不可恢复场景** (`services/mirror/mirror_pull_test.go:12-38`)

检测以下 **5 种** 可自动恢复的错误模式：

| 错误模式 | 检测条件 | 典型场景 |
|---------|---------|---------|
| 引用损坏 | `unable to resolve reference` + `reference broken` | 引用在拉取过程中损坏 |
| 远程引用消失 | `remote error` + `not our ref` | 远程引用在拉取过程中被删除（竞态） |
| 强制推送竞态 | `cannot lock ref` + `but expected` | 强制推送导致预期值不匹配 |
| 引用解析失败 | `cannot lock ref` + `unable to resolve reference` | 无法解析引用 |
| 锁文件冲突 | `Unable to create` + `.lock` | 与本地 GC/维护操作的锁竞争 |

**不可恢复错误（测试覆盖 3 种）**:
- 认证失败 (`Authentication failed`)
- 远程仓库不存在 (`Could not read from remote repository`)
- 代理连接失败 (`Failed to connect to configured-https-proxy`)

#### 2.2.3 冲突处理与重试机制

**可恢复错误计数**: 代码中**没有显式的重试计数器**，在一次 `runSync` 调用内最多只重试 **1 次**：

```go
if checkRecoverableSyncError(fetchStderr) {
    // 尝试修剪损坏的引用
    pruneErr := pruneBrokenReferences(ctx, m, m.Repo, timeout)
    if pruneErr == nil {
        // 重新尝试拉取（仅此一次）
        fetchStdout, fetchStderr, err = gitrepo.RunCmdString(ctx, m.Repo, cmdFetch())
    }
    // 如果 prune 失败或再次 fetch 失败，不会继续重试，直接返回失败
}
```

- 重试次数限制：**0 或 1 次**（prune 成功才重试，不成功则直接失败）
- 若 prune 后 fetch 再次失败，不会继续重试，按普通失败处理
- 需等待下次 cron 调度（每 10 分钟）才有机会再次尝试

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

> **注意**: `And("`push_mirror`.`interval` != 0")` 条件意味着 **interval=0 的推送镜像永远不会被定时调度触发**。

### 4.4 推送镜像 interval=0 时的触发路径

当 `Interval = 0` 时，推送镜像不会被定时 cron 任务调度，只能通过以下三种路径触发：

| 触发路径 | 代码位置 | 说明 |
|---------|---------|------|
| **提交自动触发** | `services/mirror/notifier.go:25-31` | 需满足 `SyncOnCommit = true`（默认值）。每次有代码推送时，`mirrorNotifier.PushCommits()` 被调用，筛选出 `SyncOnCommit=true` 的推送镜像并加入队列。 |
| **Web UI 手动触发** | `routers/web/repo/setting/setting.go:378-393` | 用户在仓库设置页面点击 "Sync" 按钮，调用 `handleSettingsPostPushMirrorSync()` → `AddPushMirrorToQueue(m.ID)`。 |
| **API 同步触发** | `routers/api/v1/repo/mirror.go:77-124` | 调用 `POST /repos/{owner}/{repo}/push_mirrors-sync`，会遍历该仓库所有推送镜像并**同步执行** `SyncPushMirror()`（不走队列）。 |

**关键代码 - 提交触发** (`services/mirror/mirror_push.go:256-265`):
```go
func syncPushMirrorWithSyncOnCommit(ctx context.Context, repoID int64) {
    pushMirrors, err := repo_model.GetPushMirrorsSyncedOnCommit(ctx, repoID)
    // 筛选 SyncOnCommit = true 的镜像
    for _, mirror := range pushMirrors {
        AddPushMirrorToQueue(mirror.ID)
    }
}
```

**数据库查询** (`models/repo/pushmirror.go:132-136`):
```go
func GetPushMirrorsSyncedOnCommit(ctx context.Context, repoID int64) ([]*PushMirror, error) {
    return db.Find[PushMirror](ctx, findPushMirrorOptions{
        RepoID:       repoID,
        SyncOnCommit: optional.Some(true),  // 只返回 SyncOnCommit=true 的
    })
}
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

### 5.3 队列去重在出队后的生效边界

Gitea 支持三种底层队列实现：**LevelDB (level)**、**Redis** 和 **内存 Channel (channel)**，它们在出队后的去重行为**各不相同**。

#### 5.3.1 三类队列的去重差异证据

Gitea 支持三种底层队列实现，它们在出队后的去重行为**各不相同**：

| 操作 | Level 队列 (`baseLevelQueueUnique`) | Redis 队列 (`baseRedis`) | Channel 队列 (`baseChannel`) |
|-----|-----------------------------------|--------------------------|-------------------------------|
| **底层存储** | LevelDB（磁盘） | Redis（内存/持久化） | Go Channel（内存） |
| **PushItem** | `RPush` + `SAdd` 到 Set | `RPush` + `SADD` 到 Set | `channel<-` + `set.Add` |
| **出队** | `LPop()` 仅移除 Queue，**Set 不移除** | `LPop()` + `SREM` ✅ 移除 Set | `<-channel` + `set.Remove` ✅ 移除 Set |
| **出队后 HasItem** | `true`（仍在 Set 中） | `false`（已从 Set 移除） | `false`（已从 Set 移除） |
| **出队后能否再次入队** | ❌ 不能 | ✅ 可以 | ✅ 可以 |
| **清空去重方式** | 调用 `RemoveAll()` | 出队自动清理 / `RemoveAll()` | 出队自动清理 / `RemoveAll()` |
| **重启后去重集合** | ✅ **保留**（LevelDB 持久化） | ✅ **保留**（Redis 持久化） | ❌ 丢失（内存 channel） |

**关键结论**：
```
Level 队列：一旦入队，除非 RemoveAll() 手动清理，否则永远无法再次入队
Redis / Channel 队列：出队后自动清理去重 Set，可以立即再次入队
```

#### 5.3.1.2 Redis 队列实现 (`modules/queue/base_redis.go`)

**出队时同时清空去重** (`modules/queue/base_redis.go:81-98`):
```go
func (q *baseRedis) PopItem(ctx) ([]byte, error) {
    return backoffRetErr(ctx, ..., func() (retry bool, data []byte, err error) {
        data, err = q.client.LPop(ctx, q.cfg.QueueFullName).Bytes()
        if q.isUnique {
            // the data has been popped, even if there is any error we can't do anything
            _ = q.client.SRem(ctx, q.cfg.SetFullName, data).Err()  // ✅ 从 Set 移除！
        }
        return false, data, err
    })
}
```

**入队去重** (`modules/queue/base_redis.go:68-76`):
```go
if q.isUnique {
    added, err := q.client.SAdd(ctx, q.cfg.SetFullName, data).Result()
    if added == 0 {
        return false, ErrAlreadyInQueue  // Set 中已存在，拒绝入队
    }
}
q.client.RPush(ctx, q.cfg.QueueFullName, data)
```

**⚠️ Redis 队列重启行为**:
- Redis 数据默认持久化，重启后去重集合保留
- 但由于出队时已 `SREM` 移除，已处理完成的项不会影响后续入队
- 只有那些**入队后未出队就崩溃**的项会残留在 Set 中，无法再次入队

#### 5.3.1.1 Level 队列重启后去重集合保留的证据

**证据 1 - LevelDB 持久化特性** (`modules/queue/queue.go:34`):
> "LevelDB: Especially useful in persistent queues for single instances."

**证据 2 - NewUniqueQueue 不清空已有数据** (`modules/queue/base_levelqueue_unique.go:36`):
```go
lq, err := levelqueue.NewUniqueQueue(db, []byte(cfg.QueueFullName), []byte(cfg.SetFullName), false)
// 第三个参数 false = 不清空已有数据，保留历史 Set
```

**证据 3 - 损坏恢复测试** (`modules/queue/base_levelqueue_test.go:29-76`):
```go
// 1. 创建队列并写入数据
lq, _ := levelqueue.NewUniqueQueue(db, nameQueuePrefix, nameSetPrefix, false)
lq.RPush([]byte("item-1"))
lq.Close()  // 关闭但不清空数据

// 2. 删除部分数据模拟损坏
db.Delete(itemKey, nil)

// 3. RemoveAll 是唯一清空方式
lqinternal.RemoveLevelQueueKeys(db, nameQueuePrefix)  // 仅删除 Queue 前缀的 key
lqinternal.RemoveLevelQueueKeys(db, nameSetPrefix)    // 仅删除 Set 前缀的 key

// 4. 重新创建队列，从空开始
lq, _ = levelqueue.NewUniqueQueue(db, nameQueuePrefix, nameSetPrefix, false)
lq.RPush([]byte("item-new-1"))  // 正常工作
```

**⚠️ 重要事实校正**: 之前的"系统重启会清空去重集合"是**错误推断**。实际上：
- 系统重启不会清空 LevelDB 中的 Set 数据
- 重启后去重集合仍然存在，之前入队过的项仍然无法再次入队
- `RemoveAll()` 是代码中**唯一**显式清空去重集合的方式
- 如果队列数据损坏（如测试中所示），RemoveAll 也是唯一的恢复手段

#### 5.3.2 Level 队列实现 (`modules/queue/base_levelqueue_unique.go`)

**出队时不清空去重** (`base_levelqueue_common.go:52-68`):
```go
func (q *baseLevelQueueCommonImpl) PopItem(ctx) ([]byte, error) {
    return backoffRetErr(ctx, ..., func() (retry bool, data []byte, err error) {
        data, err = q.internalFunc().LPop()  // 只从 Queue 移除
        // 不对 Set 做任何操作
        return false, data, err
    })
}
```

**唯一清空去重的方式** (`base_levelqueue_unique.go:75-87`):
```go
func (q *baseLevelQueueUnique) RemoveAll(ctx) error {
    // 删除 Queue 和 Set 的所有 LevelDB key
    lqinternal.RemoveLevelQueueKeys(q.db, []byte(q.cfg.QueueFullName))
    lqinternal.RemoveLevelQueueKeys(q.db, []byte(q.cfg.SetFullName))
    // 重新创建 queue 和 set
}
```

#### 5.3.3 Channel 队列实现 (`modules/queue/base_channel.go:72-85`)

**出队时同时清空去重**:
```go
func (q *baseChannel) PopItem(ctx context.Context) ([]byte, error) {
    select {
    case data, ok := <-q.c:
        if !ok {
            return nil, errChannelClosed
        }
        q.mu.Lock()
        q.set.Remove(string(data))  // ✅ 从 Set 中移除！
        q.mu.Unlock()
        return data, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

#### 5.3.4 共同的入队去重逻辑

两类队列的入队去重逻辑相同：
```go
// PushItem 时检查 Set
if q.isUnique && q.set.Contains(string(data)) {
    return ErrAlreadyInQueue
}
```

#### 5.3.5 实际影响

- **生产环境（默认 Level 队列）**:
  - 镜像同步任务正在执行中或刚执行完，**无法立即再次入队**
  - 即使同步失败，也无法立即重试，**必须等待 `RemoveAll()` 手动清理**
  - **⚠️ 系统重启不能解决问题**：重启后去重集合仍然保留在 LevelDB 中
- **集群环境（Redis 队列）**:
  - 出队后自动清理去重 Set，可以立即再次入队
  - **异常崩溃场景**：入队后未出队就崩溃的项会残留在 Redis Set 中，需手动 `RemoveAll()`
- **测试/开发环境（可能使用 Channel 队列）**:
  - 出队后可以立即再次入队
  - 重启后去重集合完全清空（内存 channel 不持久化）

- `modules/queue/workerqueue.go:178-182` 注释明确说明：
  > "Has only works for unique queues. Keep in mind that this check may not be reliable (due to lacking of proper transaction support). There could be a small chance that duplicate items appear in the queue"
- 由于缺乏事务支持，三类队列都有极小概率出现重复项

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

### 6.2 凭据更新回填逻辑

凭据实际存储在 git remote 配置中，数据库只存储清除了凭据的地址。更新凭据需要重新构造带凭据的 URL 并更新 git remote。

#### 6.2.1 拉取镜像凭据更新

**文件**: `routers/web/repo/setting/setting.go:309-327`

```go
// 特殊回填逻辑：如果用户名与当前用户相同且密码为空，自动回填用户密码
if form.MirrorPassword == "" && form.MirrorUsername == u.User.Username() {
    form.MirrorPassword, _ = u.User.Password()
}

// ParseRemoteAddr 组合用户名和密码到 URL 中
address, err := git.ParseRemoteAddr(form.MirrorAddress, form.MirrorUsername, form.MirrorPassword)

// UpdateAddress 更新 git remote URL（含凭据）
if err := mirror_service.UpdateAddress(ctx, pullMirror, address); err != nil {
    ctx.ServerError("UpdateAddress", err)
    return
}
```

**URL 组合逻辑** (`modules/git/remote.go:82-110`):
```go
func ParseRemoteAddr(remoteAddr, authUsername, authPassword string) (string, error) {
    u, err := url.Parse(remoteAddr)
    if len(authUsername)+len(authPassword) > 0 {
        // 覆盖 URL 中的 userinfo
        u.User = url.UserPassword(authUsername, authPassword)
    }
    return u.String(), nil
}
```

**UpdateAddress 真实实现** (`services/mirror/mirror_pull.go:33-71`):
```go
func UpdateAddress(ctx context.Context, m *repo_model.Mirror, addr string) error {
    // 解析 URL 用于后续清除凭据
    u, err := giturl.ParseGitURL(addr)
    remoteName := m.GetRemoteName()  // 通常为 "origin"
    repo := m.GetRepository(ctx)

    // 先删除旧 remote
    err = gitrepo.GitRemoteRemove(ctx, repo, remoteName)
    if err != nil && !git.IsRemoteNotExistError(err) {
        return err
    }

    // 重新添加带新凭据的 remote（使用 RemoteOptionMirrorFetch）
    err = gitrepo.GitRemoteAdd(ctx, repo, remoteName, addr, gitrepo.RemoteOptionMirrorFetch)

    // 更新 Wiki remote
    if repo_service.HasWiki(ctx, m.Repo) {
        wikiRemotePath := repo_module.WikiRemoteURL(ctx, addr)
        gitrepo.GitRemoteRemove(ctx, repo.WikiStorageRepo(), remoteName)
        gitrepo.GitRemoteAdd(ctx, repo.WikiStorageRepo(), remoteName, wikiRemotePath, gitrepo.RemoteOptionMirrorFetch)
    }

    // 清除 URL 中的用户信息后保存到数据库
    u.User = nil
    m.Repo.OriginalURL = u.String()
    return repo_model.UpdateRepositoryColsNoAutoTime(ctx, m.Repo, "original_url")
}
```

#### 6.2.2 推送镜像凭据更新

**⚠️ 重要事实校正**: 推送镜像的地址**创建后无法通过 Web UI 更新**。`handleSettingsPostPushMirrorUpdate` 只更新 `Interval`，不更新地址。

**创建时** (`routers/web/repo/setting/setting.go:474-535`):
```go
// 组合带凭据的地址
address, err := git.ParseRemoteAddr(form.PushMirrorAddress, form.PushMirrorUsername, form.PushMirrorPassword)
// 添加到 git remote
if err = mirror_service.AddPushMirrorRemote(ctx, m, address); err != nil {
    ctx.ServerError("AddPushMirrorRemote", err)
    return
}
```

**更新间隔时** (`routers/web/repo/setting/setting.go:399-438`):
```go
// handleSettingsPostPushMirrorUpdate 只更新 Interval，不更新地址！
func handleSettingsPostPushMirrorUpdate(ctx *context.Context) {
    interval, err := time.ParseDuration(form.PushMirrorInterval)
    m, _, _ := repo_model.GetPushMirrorByIDAndRepoID(ctx, form.PushMirrorID, repo.ID)

    m.Interval = interval
    // 只更新间隔，不涉及地址/凭据更新
    if err := repo_model.UpdatePushMirrorInterval(ctx, m); err != nil {
        ctx.ServerError("UpdatePushMirrorInterval", err)
        return
    }

    // 可选立即同步
    if !ctx.FormBool("push_mirror_defer_sync") {
        mirror_service.AddPushMirrorToQueue(m.ID)
    }
}
```

**地址更新的唯一方式**: 目前代码中**不存在 `UpdatePushMirrorAddress` 函数**。若需更新推送镜像地址，只能：
1. 删除旧的推送镜像
2. 重新创建新的推送镜像（带新地址和凭据）

#### 6.2.3 凭据更新数据流

**拉取镜像更新流**:
```
用户表单输入 (username/password)
       ↓
ParseRemoteAddr() → 组合成 https://user:pass@host/repo.git
       ↓
UpdateAddress(mirror, addr)  [services/mirror/mirror_pull.go:33]
       ↓
├─ git remote remove origin
├─ git remote add origin <new_addr_with_creds>  # --mirror=fetch
└─ (若有Wiki) wiki remote 同步更新
       ↓
数据库保存: OriginalURL = SanitizeURL(address)  // 清除凭据后保存
```

**推送镜像创建流**（地址更新只能通过删除重建）:
```
用户表单输入 (address, username, password)
       ↓
ParseRemoteAddr() → 组合成 https://user:pass@host/repo.git
       ↓
AddPushMirrorRemote(m, addr)  [services/mirror/mirror_push.go:33]
       ↓
├─ git remote add <remote_name> <addr_with_creds>  # --mirror=push
├─ git config remote.<remote_name>.push +refs/heads/*:refs/heads/*
├─ git config remote.<remote_name>.push +refs/tags/*:refs/tags/*
└─ (若有Wiki) wiki remote 同步配置
       ↓
数据库保存: RemoteAddress = SanitizeURL(address)
```

### 6.3 日志安全

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

### 6.4 禁用交互式凭据提示

**文件**: `modules/git/gitcmd/command.go:251`
```go
"GIT_TERMINAL_PROMPT=0"  // 全局环境变量，禁止 git 交互提示
```

---

## 七、失败处理与重试策略

### 7.1 可恢复错误自动重试

| 错误类型 | 检测条件 | 处理方式 | 重试机制 |
|---------|---------|---------|---------|
| 引用损坏 | `unable to resolve reference` + `reference broken` | `git remote prune` 清理后重试 | 同步内立即重试 |
| 远程引用消失 | `remote error` + `not our ref` | 同上 | 同步内立即重试 |
| 强制推送竞态 | `cannot lock ref` + `but expected` | 同上 | 同步内立即重试 |
| 引用解析失败 | `cannot lock ref` + `unable to resolve reference` | 同上 | 同步内立即重试 |
| 锁文件冲突 | `Unable to create` + `.lock` | 同上 | 同步内立即重试 |

> 以上 **5 种** 可恢复错误与 `checkRecoverableSyncError()` 实现完全一致，测试覆盖全部 5 种场景。

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
| 拉取凭据更新 | `services/mirror/mirror_pull.go` | `UpdateAddress` |
| 推送间隔更新 | `models/repo/pushmirror.go` | `UpdatePushMirrorInterval` |
| URL 组合 | `modules/git/remote.go` | `ParseRemoteAddr` |
| Level 队列去重 | `modules/queue/base_levelqueue_unique.go` | `RemoveAll` |
| Redis 队列去重 | `modules/queue/base_redis.go` | `PopItem` (自动 `SREM`) |
| Channel 队列去重 | `modules/queue/base_channel.go` | `PopItem` (自动清理) |

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

测试 **5 种可恢复错误** + **3 种不可恢复错误**的识别准确性，与 `checkRecoverableSyncError()` 实现完全一致。
