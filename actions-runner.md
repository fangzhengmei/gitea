# CI 流水线下发任务到 Runner 握手链路分析

## 一、整体架构概览

Gitea Actions 的任务调度采用 **"拉模式"** 设计，即 Runner 主动轮询 Gitea 服务端获取任务，而非服务端主动推送。核心握手链路如下：

```
Runner 注册 → Runner 心跳/声明 → FetchTask 轮询 → PickTask 分配 → 任务执行 → UpdateTask/UpdateLog 回传
```

---

## 二、作业队列实现机制

### 2.1 队列层级设计

Gitea Actions 采用 **"数据库级队列 + 内存级调度"** 的双层队列架构：

#### 2.1.1 数据库级作业池 (`ActionRunJob`)
- **表模型**: `models/actions/run_job.go:27`
- **核心字段**:
  - `Status`: 作业状态（`Blocked` → `Waiting` → `Running` → 终态）
  - `TaskID`: 关联的执行任务 ID，为 0 表示未分配
  - `RunsOn`: 标签匹配条件
  - `Needs`: 依赖的前置作业

#### 2.1.2 任务版本信号量 (`ActionTasksVersion`)
- **表模型**: `models/actions/tasks_version.go:18`
- **作用**: 避免无效轮询的乐观锁机制
- **三级作用域**:
  - 全局 (`OwnerID=0, RepoID=0`)
  - 组织级 (`OwnerID>0, RepoID=0`)
  - 仓库级 (`OwnerID=0, RepoID>0`)

**核心逻辑** (`routers/api/actions/runner/runner.go:177-220`):
```go
func (s *Service) FetchTask(ctx, req) {
    latestVersion, _ := GetTasksVersionByScope(ctx, runner.OwnerID, runner.RepoID)
    if tasksVersion != latestVersion {
        // 版本不一致时才尝试分配任务
        task, ok, _ := PickTask(ctx, freshRunner)
    }
    return FetchTaskResponse{Task: task, TasksVersion: latestVersion}
}
```

**版本递增触发点** (`models/actions/run_job.go:245-250`):
```go
if slices.Contains(cols, "status") && job.Status.IsWaiting() {
    // 作业状态变为 Waiting 时递增版本，通知 Runner 有新任务
    IncreaseTaskVersion(ctx, job.OwnerID, job.RepoID)
}
```

### 2.2 任务分配流程 (`PickTask`)

**入口**: `services/actions/task.go:19-112` → `models/actions/task.go:230-334`

#### 2.2.1 前置检查
1. **Runner 禁用检查**: `runner.IsDisabled` 为 true 时直接返回
2. **Ephemeral Runner 检查**: 临时 Runner 只能分配一个任务，已有任务且未完成时拒绝
3. **数据库事务**: 使用 `db.WithTx` 保证任务分配原子性

#### 2.2.2 作业筛选逻辑 (`CreateTaskForRunner`)
```go
// 1. 按 Runner 权限范围筛选作业
jobCond := builder.NewCond()
if runner.RepoID != 0 {
    jobCond = builder.Eq{"repo_id": runner.RepoID}
} else if runner.OwnerID != 0 {
    jobCond = builder.In("repo_id", ...) // 组织下所有启用 Actions 的仓库
}

// 2. 查找未分配且状态为 Waiting 的作业
e.Where("task_id=? AND status=?", 0, StatusWaiting).And(jobCond).Asc("updated", "id").Find(&jobs)

// 3. 标签匹配（runner.AgentLabels 需包含 job.RunsOn 所有标签）
for _, v := range jobs {
    if runner.CanMatchLabels(v.RunsOn) {
        job = v
        break
    }
}
```

#### 2.2.3 任务创建与绑定
```go
// 更新 Job 状态
job.Started = now
job.Status = StatusRunning
job.TaskID = task.ID

// 创建 Task
task := &ActionTask{
    JobID:    job.ID,
    RunnerID: runner.ID,
    Status:   StatusRunning,
    Started:  now,
    // ...
}
task.GenerateAndFillToken() // 生成任务专属 Token

// 创建步骤记录
for i, v := range workflowJob.Steps {
    steps[i] = &ActionTaskStep{
        Name:   makeTaskStepDisplayName(v, 255),
        TaskID: task.ID,
        Index:  int64(i),
        Status: StatusWaiting,
    }
}
```

#### 2.2.4 乐观锁保障
```go
// 使用 task_id=0 作为 CAS 条件，防止并发分配
if n, err := UpdateRunJob(ctx, job, builder.Eq{"task_id": 0}); err != nil {
    return nil, false, err
} else if n != 1 {
    return nil, false, nil // 并发冲突，返回无任务
}
```

### 2.3 作业就绪调度 (`job_emitter`)

**组件**: `services/actions/job_emitter.go`

#### 2.3.1 就绪队列
```go
var jobEmitterQueue *queue.WorkerPoolQueue[*jobUpdate]

func EmitJobsIfReadyByRun(runID int64) error {
    err := jobEmitterQueue.Push(&jobUpdate{RunID: runID})
    if errors.Is(err, queue.ErrAlreadyInQueue) {
        return nil // 去重，避免重复处理
    }
    return err
}
```

#### 2.3.2 依赖解析 (`jobStatusResolver`)
```go
func (r *jobStatusResolver) resolve(ctx) map[int64]Status {
    for id, status := range r.statuses {
        if status != StatusBlocked {
            continue
        }
        // 检查所有依赖是否完成
        allDone, allSucceed := r.resolveCheckNeeds(id)
        if !allDone {
            continue
        }
        // 评估并发组
        updateConcurrencyEvaluationForJobWithNeeds(ctx, job, r.vars)
        // 决定新状态：有 if 条件则 Waiting，否则 Skipped
        newStatus := util.Iif(shouldStartJob, StatusWaiting, StatusSkipped)
        // 并发组检查，可能取消同组其他作业
        newStatus, cancelledJobs, _ = PrepareToStartJobWithConcurrency(ctx, job)
        ret[id] = newStatus
    }
    return ret
}
```

---

## 三、令牌时效管理机制

Gitea Actions 采用 **三层令牌体系**，各层职责分明：

### 3.1 注册令牌 (`ActionRunnerToken`)
- **模型**: `models/actions/runner_token.go:29`
- **用途**: Runner 首次注册时的身份凭证
- **生命周期**:
  - 手动生成，40 位随机字符串
  - 同一作用域（全局/组织/仓库）只有一个活跃令牌
  - 新令牌生成时自动失效旧令牌 (`NewRunnerTokenWithValue:87-97`)
- **使用场景**: 仅用于 `Register` 接口

### 3.2 Runner 身份令牌 (`ActionRunner.Token`)
- **模型**: `models/actions/runner.go:53-56`
- **生成**: `GenerateAndFillToken()` 注册时生成
- **存储**: 仅存储 `TokenHash`（SHA256）和 `TokenSalt`，不存明文
- **认证流程** (`routers/api/actions/runner/interceptor.go:28-61`):
  ```go
  // 拦截器中验证
  uuid := request.Header().Get("x-runner-uuid")
  token := request.Header().Get("x-runner-token")
  
  runner, _ := GetRunnerByUUID(ctx, uuid)
  // 使用恒定时间比较防止时序攻击
  if subtle.ConstantTimeCompare([]byte(runner.TokenHash), 
      []byte(auth_model.HashToken(token, runner.TokenSalt))) != 1 {
      return Unauthenticated
  }
  
  // 更新在线状态
  runner.LastOnline = timeutil.TimeStampNow()
  if methodName == "UpdateTask" || methodName == "UpdateLog" {
      runner.LastActive = timeutil.TimeStampNow()
  }
  ```
- **时效**: 长期有效，除非 Runner 被删除

### 3.3 任务运行时令牌 (`ActionTask.Token` + JWT)

#### 3.3.1 任务专属 Token
- **模型**: `models/actions/task.go:46-49`
- **生成**: `GenerateAndFillToken()` 任务分配时生成
- **存储**: SHA256 哈希 + Salt，保留后 8 位用于快速查询
- **认证缓存**: LRU 缓存成功认证的 Token，减少哈希计算
  ```go
  // models/actions/task.go:163-214
  func GetRunningTaskByToken(ctx, token) (*ActionTask, error) {
      // 先查缓存
      if id := getTaskIDFromCache(token); id > 0 {
          // ...
      }
      // 后 8 位快速过滤
      lastEight := token[len(token)-8:]
      db.Where("token_last_eight = ? AND status IN (?, ?)", 
          lastEight, StatusRunning, StatusCancelling).Find(&tasks)
      // 恒定时间比较哈希
      for _, t := range tasks {
          tempHash := auth_model.HashToken(token, t.TokenSalt)
          if subtle.ConstantTimeCompare([]byte(t.TokenHash), []byte(tempHash)) == 1 {
              successfulTokenTaskCache.Add(token, t.ID)
              return t, nil
          }
      }
  }
  ```
- **有效状态**: 仅 `StatusRunning` 和 `StatusCancelling` 状态可认证

#### 3.3.2 JWT 运行时授权令牌
- **生成**: `services/actions/auth.go:41-73`
- **结构**:
  ```go
  type actionsClaims struct {
      jwt.RegisteredClaims
      Scp    string // "Actions.Results:{runID}:{jobID}"
      TaskID int64
      RunID  int64
      JobID  int64
      Ac     string // JSON 序列化的缓存权限
  }
  ```
- **时效配置**:
  ```go
  ExpiresAt: jwt.NewNumericDate(now.Add(1*time.Hour + setting.Actions.EndlessTaskTimeout))
  ```
  - 基础时效：1 小时
  - 附加时效：`EndlessTaskTimeout`（防止超长任务令牌过期）
- **用途**: Runner 调用 Gitea API 时的 Authorization 头，用于 artifact 上传、cache 操作等

---

## 四、运行日志回传机制

### 4.1 日志传输协议

**API 接口**: `routers/api/actions/runner/runner.go:291-357`

#### 4.1.1 请求结构
```protobuf
message UpdateLogRequest {
    int64 task_id = 1;
    int64 index = 2;          // 本次上传起始行号
    repeated LogRow rows = 3; // 日志行
    bool no_more = 4;         // 是否结束
}

message LogRow {
    google.protobuf.Timestamp time = 1;
    string content = 2;
}
```

#### 4.1.2 去重与 ACK 机制
```go
ack := task.LogLength // 服务端已确认的行数

// 裁剪已 ACK 的行
var rows []*runnerv1.LogRow
if req.Msg.Index <= ack && int64(len(req.Msg.Rows))+req.Msg.Index > ack {
    rows = req.Msg.Rows[ack-req.Msg.Index:]
}

// 无新行且非结束标记时直接返回 ACK
if len(rows) == 0 && (!req.Msg.NoMore || req.Msg.Index > ack) {
    res.Msg.AckIndex = ack
    return res, nil
}
```

### 4.2 两级存储架构

#### 4.2.1 第一级：DBFS 临时存储
**实现**: `modules/actions/log.go:35-78`

```go
func WriteLogs(ctx, filename string, offset int64, rows []*runnerv1.LogRow) ([]int, error) {
    name := DBFSPrefix + filename // "actions_log/" + filename
    
    // offset=0 时才允许创建文件，防止内容空洞
    flag := os.O_WRONLY
    if offset == 0 {
        flag |= os.O_CREATE
    }
    
    f, _ := dbfs.OpenFile(ctx, name, flag)
    defer f.Close()
    
    // 检查文件大小，防止空洞
    stat, _ := f.Stat()
    if stat.Size() < offset {
        return nil, fmt.Errorf("size of %q is less than offset", name)
    }
    
    f.Seek(offset, io.SeekStart)
    writer := bufio.NewWriterSize(f, defaultBufSize)
    
    // 格式化每行日志：时间戳 + 内容
    for _, row := range rows {
        n, _ := writer.WriteString(FormatLog(row.Time.AsTime(), row.Content) + "\n")
        ns = append(ns, n)
    }
    writer.Flush()
    return ns, nil
}
```

**日志格式**:
```
2024-01-01T12:00:00.0000000Z 日志内容
```
- 时间戳格式：`2006-01-02T15:04:05.0000000Z07:00`
- 单行最大：64KB (`MaxLineSize`)
- 换行符转义：`\n` → `\\n`

#### 4.2.2 第二级：对象存储归档
**实现**: `modules/actions/log.go:126-162`

```go
func TransferLogs(ctx, filename string) (func(), error) {
    name := DBFSPrefix + filename
    // 延迟删除 DBFS 文件
    remove := func() {
        dbfs.Remove(ctx, name)
    }
    
    f, _ := dbfs.Open(ctx, name)
    defer f.Close()
    
    var reader io.Reader = f
    // 可选 ZSTD 压缩（文件名以 .zst 结尾时）
    if strings.HasSuffix(filename, ".zst") {
        r, w := io.Pipe()
        reader = r
        zstdWriter, _ := zstd.NewSeekableWriter(w, logZstdBlockSize) // 128KB 块
        go func() {
            io.Copy(zstdWriter, f)
            w.CloseWithError(zstdWriter.Close())
        }()
    }
    
    // 保存到对象存储
    storage.Actions.Save(filename, reader, -1)
    return remove, nil
}
```

**触发归档时机**:
1. Runner 发送 `NoMore=true` 标记时
2. 任务被终止时（Zombie/Endless 清理）

### 4.3 索引与元数据
- `LogLength`: 已确认的日志行数
- `LogSize`: 日志文件字节大小
- `LogIndexes`: 行号到字节偏移的索引数组，用于快速定位
- `LogInStorage`: 是否已归档到对象存储

---

## 五、作业取消机制

### 5.1 取消触发场景

#### 5.1.1 用户主动取消
- Web/API 调用取消操作
- 触发 `StopTask` 接口

#### 5.1.2 并发组自动取消
**实现**: `services/actions/clear_tasks.go:80-95`
```go
func PrepareToStartJobWithConcurrency(ctx, job) (Status, []*ActionRunJob, error) {
    // 取消同并发组中之前的 Waiting/Blocked 作业
    // 如果 concurrency.cancel=true，还会取消 Running/Cancelling 作业
    jobs, err := CancelPreviousJobsByJobConcurrency(ctx, job)
    return newStatus, jobs, nil
}
```

#### 5.1.3 同分支新提交取消旧工作流
**实现**: `services/actions/clear_tasks.go:48-53`
```go
func CancelPreviousJobs(ctx, repoID, ref, workflowID, event) error {
    jobs, _ := actions_model.CancelPreviousJobs(ctx, repoID, ref, workflowID, event)
    NotifyWorkflowJobsAndRunsStatusUpdate(ctx, jobs)
    EmitJobsIfReadyByJobs(jobs)
    return nil
}
```

#### 5.1.4 超时自动清理
- **Zombie Tasks**: `StopZombieTasks` - 长时间无更新的 Running/Cancelling 任务
  ```go
  UpdatedBefore: time.Now().Add(-setting.Actions.ZombieTaskTimeout)
  ```
- **Endless Tasks**: `StopEndlessTasks` - 运行时间过长的任务
  ```go
  StartedBefore: time.Now().Add(-setting.Actions.EndlessTaskTimeout)
  ```
- **Abandoned Jobs**: `CancelAbandonedJobs` - 长时间未被 Runner 拾取的 Waiting/Blocked 作业
  ```go
  UpdatedBefore: timeutil.TimeStampNow().AddDuration(-setting.Actions.AbandonedJobTimeout)
  ```

### 5.2 取消状态流转

#### 5.2.1 能力协商
**Runner 能力声明** (`routers/api/actions/runner/runner.go:117-139`):
```go
const runnerCapabilityCancelling = "cancelling"

func runnerRequestHasCancellingCapability(req proto.Message) (bool, bool) {
    if typedReq, ok := any(req).(capabilityGetter); ok {
        return slices.Contains(typedReq.GetCapabilities(), runnerCapabilityCancelling), true
    }
    return false, false
}
```

#### 5.2.2 状态机设计
```
StatusWaiting/Blocked → CancelJobs() → StatusCancelled (直接终态)

StatusRunning → StopTask(StatusCancelling) → StatusCancelling
    ↓ (Runner 支持 Cancelling)
    执行 post-step cleanup
    ↓
    UpdateTask(Result=*) → StatusCancelled (保留用户意图)

StatusRunning → StopTask(StatusCancelled) → StatusCancelled (不支持 Cancelling 时直接终态)
```

**核心逻辑** (`models/actions/task.go:431-499`):
```go
func StopTask(ctx, taskID int64, status Status) error {
    if status == StatusCancelling {
        runner, err := GetRunnerByID(ctx, task.RunnerID)
        if err != nil || !runner.HasCancellingSupport {
            // Runner 不存在或不支持 Cancelling，直接终态
            status = StatusCancelled
        }
    }
    
    if status == StatusCancelling {
        // 过渡状态，等待 Runner 上报清理结果
        task.Status = StatusCancelling
        UpdateRunJob(ctx, &ActionRunJob{ID: task.JobID, Status: StatusCancelling}, nil, "status")
        return UpdateTask(ctx, task, "status")
    }
    
    // 终态处理
    task.Status = status
    task.Stopped = now
    UpdateRunJob(ctx, &ActionRunJob{...})
    UpdateTask(ctx, task, "status", "stopped")
    
    // 终止未完成步骤
    for _, step := range task.Steps {
        if !step.Status.IsDone() {
            step.Status = status
            step.Stopped = now
        }
    }
}
```

#### 5.2.3 Runner 上报处理
**实现** (`models/actions/task.go:353-429`):
```go
func UpdateTaskByState(ctx, runnerID int64, state *runnerv1.TaskState) (*ActionTask, error) {
    if state.Result != runnerv1.Result_RESULT_UNSPECIFIED {
        if task.Status == StatusCancelling {
            // 保留用户取消意图，不被 Runner 的 SUCCESS/FAILURE 覆盖
            task.Status = StatusCancelled
        } else {
            task.Status = StatusFromResult(state.Result)
        }
        task.Stopped = timeutil.TimeStamp(state.StoppedAt.AsTime().Unix())
        UpdateTask(ctx, task, "status", "stopped")
        UpdateRunJob(ctx, &ActionRunJob{...})
    } else {
        // 非终态，仅更新 Updated 时间戳防止被判定为 Zombie
        task.Updated = timeutil.TimeStampNow()
        UpdateTask(ctx, task, "updated")
    }
}
```

### 5.3 清理收尾
```go
// services/actions/clear_tasks.go:153-163
// 终止任务时归档日志
remove, err := actions.TransferLogs(ctx, task.LogFilename)
if err == nil {
    task.LogInStorage = true
    UpdateTask(ctx, task, "log_in_storage")
    remove() // 删除 DBFS 临时文件
}

// 终止后触发后续作业调度
NotifyWorkflowJobsAndRunsStatusUpdate(ctx, jobs)
EmitJobsIfReadyByJobs(jobs)
```

---

## 六、关键交互时序

### 6.1 Runner 注册流程
```
Runner                                  Gitea
  |                                        |
  | POST /.../Register(token, name)        |
  |--------------------------------------->|
  |                                        | 验证注册令牌有效性
  |                                        | 生成 Runner UUID + 身份 Token
  |                                        | 持久化 Runner 信息
  |                                        |
  | 响应 {id, uuid, token, ...}            |
  |<---------------------------------------|
```

### 6.2 任务拉取与执行流程
```
Runner                                  Gitea
  |                                        |
  | POST /.../FetchTask(tasksVersion)      |
  |--------------------------------------->|
  |                                        | 比较 tasksVersion
  | 如无新任务，直接返回最新版本号            |
  |<---------------------------------------|
  |                                        |
  | 轮询等待 (版本一致时)                   |
  |                                        |
  | POST /.../FetchTask(tasksVersion)      |
  |--------------------------------------->|
  |                                        | 版本不一致 → PickTask
  |                                        | 数据库事务分配任务
  |                                        | 生成任务 Token + JWT
  |                                        |
  | 响应 {task, tasksVersion}              |
  |<---------------------------------------|
  |                                        |
  | 执行任务                                |
  |                                        |
  | POST /.../UpdateTask(state)            |
  |--------------------------------------->| 更新任务状态
  |                                        |
  | POST /.../UpdateLog(rows)              |
  |--------------------------------------->| 追加日志到 DBFS
  |                                        |
  | POST /.../UpdateLog(no_more=true)      |
  |--------------------------------------->| 归档日志到对象存储
  |                                        |
  | POST /.../UpdateTask(result=SUCCESS)   |
  |--------------------------------------->| 标记任务完成
  |                                        | 触发后续作业调度
```

### 6.3 任务取消流程
```
User/System                             Gitea                                  Runner
  |                                        |                                        |
  | 取消操作                                |                                        |
  |--------------------------------------->|                                        |
  |                                        | StopTask(StatusCancelling)             |
  |                                        | 检查 Runner Cancelling 支持              |
  |                                        | 更新任务状态为 Cancelling                |
  |                                        |                                        |
  |                                        | 下次 FetchTask 或 UpdateTask 时         |
  |                                        | 通知 Runner 任务已取消                   |
  |<---------------------------------------|--------------------------------------->|
  |                                        |                                        | 执行清理步骤
  |                                        |                                        |
  |                                        | UpdateTask(Result=*)                   |
  |                                        |<---------------------------------------|
  |                                        | 状态强制为 Cancelled (保留用户意图)      |
  |                                        | 归档日志                                |
  |                                        | 触发后续作业调度                        |
```

---

## 七、关键配置项

| 配置项 | 含义 | 默认值 | 位置 |
|--------|------|--------|------|
| `RunnerOfflineTime` | Runner 离线判定时间 | 1 分钟 | `models/actions/runner.go:76` |
| `RunnerIdleTime` | Runner 空闲判定时间 | 10 秒 | `models/actions/runner.go:77` |
| `Actions.ZombieTaskTimeout` | Zombie 任务超时 | - | `setting` |
| `Actions.EndlessTaskTimeout` | 超长任务超时 | - | `setting` |
| `Actions.AbandonedJobTimeout` | 作业遗弃超时 | - | `setting` |
| `SuccessfulTokensCacheSize` | Token 认证缓存大小 | - | `setting` |
| `MaxLineSize` | 单条日志最大长度 | 64KB | `modules/actions/log.go:25` |
| `logZstdBlockSize` | ZSTD 压缩块大小 | 128KB | `modules/actions/log.go:119` |
| `MaxJobNumPerRun` | 单 Run 最大作业数 | 256 | `models/actions/run_job.go:24` |

---

## 八、设计特点总结

1. **拉模式调度**：Runner 主动轮询，避免服务端主动连接问题
2. **乐观锁版本控制**：通过 `tasks_version` 减少无效轮询
3. **三级令牌体系**：注册令牌、Runner 身份令牌、任务运行令牌职责分离
4. **两级日志存储**：DBFS 高效追加写 + 对象存储长期归档
5. **优雅取消机制**：支持 `Cancelling` 过渡状态，允许 Runner 执行清理
6. **并发控制**：作业级和工作流级并发组，支持取消进行中任务
7. **防时序攻击**：所有令牌比较使用 `subtle.ConstantTimeCompare`
8. **去重机制**：`jobEmitterQueue` 使用 `ErrAlreadyInQueue` 防止重复调度
