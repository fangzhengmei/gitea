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

##### 2.1.2.1 `tasksVersion=0` 兼容分支深度分析

**问题背景**: 系统首次运行或某作用域从未产生过任务时，`GetTasksVersionByScope` 返回 0。

**兼容处理** (`routers/api/actions/runner/runner.go:188-196`):
```go
latestVersion, err := actions_model.GetTasksVersionByScope(ctx, runner.OwnerID, runner.RepoID)
if err != nil {
    return nil, status.Errorf(codes.Internal, "query tasks version failed: %v", err)
} else if latestVersion == 0 {
    if err := actions_model.IncreaseTaskVersion(ctx, runner.OwnerID, runner.RepoID); err != nil {
        return nil, status.Errorf(codes.Internal, "fail to increase task version: %v", err)
    }
    // if we don't increase the value of `latestVersion` here,
    // the response of FetchTask will return tasksVersion as zero.
    // and the runner will treat it as an old version of Gitea.
    latestVersion++
}
```

**关键设计点**:
1. **自动初始化**: 当 `latestVersion == 0` 时，立即调用 `IncreaseTaskVersion` 初始化版本记录
2. **手动递增**: `IncreaseTaskVersion` 执行后不返回更新后的值，需手动 `latestVersion++`
3. **版本语义**: 避免返回 0 给 Runner，因为旧版 Gitea 没有 tasksVersion 机制，Runner 会将 0 误认为是不支持该特性的旧服务端

##### 2.1.2.2 三层版本递增对任务可见性的影响

**递增链路** (`models/actions/tasks_version.go:75-101`):
```go
func IncreaseTaskVersion(ctx context.Context, ownerID, repoID int64) error {
    return db.WithTx(ctx, func(ctx context.Context) error {
        // 1. 递增全局版本 (OwnerID=0, RepoID=0)
        if err := increaseTasksVersionByScope(ctx, 0, 0); err != nil {
            return err
        }

        // 2. 递增组织版本 (OwnerID>0, RepoID=0)
        if ownerID > 0 {
            if err := increaseTasksVersionByScope(ctx, ownerID, 0); err != nil {
                return err
            }
        }

        // 3. 递增仓库版本 (OwnerID=0, RepoID>0)
        if repoID > 0 {
            if err := increaseTasksVersionByScope(ctx, 0, repoID); err != nil {
                return err
            }
        }
        return nil
    })
}
```

**可见性影响矩阵**:

| 触发场景 | 递增层级 | 受影响 Runner 范围 |
|---------|---------|------------------|
| 仓库作业就绪 | 全局 + 仓库 | 全局 Runner + 该仓库专属 Runner |
| 组织下某仓库作业就绪 | 全局 + 组织 + 仓库 | 全局 Runner + 该组织 Runner + 该仓库 Runner |
| Runner 读取版本 | 仅读取自身作用域 | 按 Runner.OwnerID/RepoID 匹配对应层级 |

**设计意图**:
- **全局递增**: 确保全局 Runner 能感知所有新任务
- **组织/仓库递增**: 实现"任务可见性隔离"，专属 Runner 只需要感知自己作用域内的任务
- **事务性**: 三层递增在同一事务中完成，保证版本变更的原子性
- **通知广播**: 任何作业就绪都会触发全局版本递增，确保全局 Runner 不会错过任务

**实际运行示例**:
1. 组织 `Org1` 下的仓库 `Repo1` 有新作业就绪
2. 调用 `IncreaseTaskVersion(Org1.ID, Repo1.ID)`
3. 事务内依次递增：
   - `increaseTasksVersionByScope(0, 0)` → 全局版本 v100 → v101
   - `increaseTasksVersionByScope(Org1.ID, 0)` → 组织版本 v50 → v51
   - `increaseTasksVersionByScope(0, Repo1.ID)` → 仓库版本 v20 → v21
4. 各 Runner 下次 FetchTask 时：
   - 全局 Runner (OwnerID=0, RepoID=0)：比较全局版本 100≠101 → 尝试 PickTask
   - Org1 Runner (OwnerID=Org1.ID, RepoID=0)：比较组织版本 50≠51 → 尝试 PickTask
   - Repo1 Runner (OwnerID=0, RepoID=Repo1.ID)：比较仓库版本 20≠21 → 尝试 PickTask
   - 其他组织/仓库 Runner：版本不变 → 跳过 PickTask

##### 2.1.2.3 Owner 作用域版本递增与可消费 Runner 关系的边界说明

**仓内代码可证的事实** (`models/actions/tasks_version.go:75-101`):
```go
func IncreaseTaskVersion(ctx context.Context, ownerID, repoID int64) error {
    return db.WithTx(ctx, func(ctx context.Context) error {
        // 1. 全局版本始终递增
        if err := increaseTasksVersionByScope(ctx, 0, 0); err != nil {
            return err
        }
        // 2. ownerID>0 时递增组织版本
        if ownerID > 0 {
            if err := increaseTasksVersionByScope(ctx, ownerID, 0); err != nil {
                return err
            }
        }
        // 3. repoID>0 时递增仓库版本
        if repoID > 0 {
            if err := increaseTasksVersionByScope(ctx, 0, repoID); err != nil {
                return err
            }
        }
        return nil
    })
}
```

**边界说明（避免误推）**:

1. ✅ **仓内代码可证**：`IncreaseTaskVersion(ownerID, repoID)` 被调用时：
   - 全局版本 **一定** 递增（无条件）
   - 组织版本 **仅当** `ownerID > 0` 时递增
   - 仓库版本 **仅当** `repoID > 0` 时递增

2. ❌ **常见误推**："组织版本递增意味着一定有组织级 Runner 存在并能消费任务"
   - **边界**：版本递增是 **事实**，但是否有对应 Owner 作用域的 Runner 存在并能消费任务是 **另一个独立条件**
   - 可能出现的情况：
     - 组织版本递增了，但没有注册任何组织级 Runner → 任务只能由全局 Runner 或仓库级 Runner 消费
     - 有组织级 Runner，但标签不匹配 → Runner 版本变化触发 PickTask，但 `CanMatchLabels` 失败，任务仍无法分配
     - 组织级 Runner 被禁用 → 版本递增了，但 Runner 被 `IsDisabled` 检查过滤

3. ✅ **仓内代码可证**：`GetTasksVersionByScope` 的查询逻辑
   ```go
   // models/actions/tasks_version.go:56-73
   func GetTasksVersionByScope(ctx context.Context, ownerID, repoID int64) (int64, error) {
       cond := builder.NewCond()
       if ownerID > 0 {
           cond = cond.And(builder.Eq{"owner_id": ownerID})
       } else {
           cond = cond.And(builder.Eq{"owner_id": 0})
       }
       if repoID > 0 {
           cond = cond.And(builder.Eq{"repo_id": repoID})
       } else {
           cond = cond.And(builder.Eq{"repo_id": 0})
       }
       var v ActionTasksVersion
       has, err := db.GetEngine(ctx).Where(cond).Get(&v)
       if err != nil {
           return 0, err
       } else if !has {
           return 0, nil // 记录不存在时返回 0
       }
       return v.Version, nil
   }
   ```
   - 全局 Runner 查 `owner_id=0, repo_id=0`
   - 组织 Runner 查 `owner_id=?, repo_id=0`
   - 仓库 Runner 查 `owner_id=0, repo_id=?`
   - 三者完全隔离，互不干扰

4. **版本递增 ≠ 任务可分配**：
   版本递增只是"通知信号"，任务能否实际分配还需经过 `PickTask` 中的多重检查：
   - Runner 是否禁用
   - Runner 是否已有运行任务（Ephemeral Runner 检查）
   - 标签是否匹配 (`CanMatchLabels`)
   - 作业是否仍为 `Waiting` 状态且 `task_id=0`（乐观锁检查）

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

##### 3.3.3 Task Token 与 `gitea_runtime_token` 生成注入完整流程

**完整时序链**:
```
PickTask 分配任务
    ↓
① task.GenerateAndFillToken() 生成 Task Token
    ↓
② CreateTaskForRunner 持久化 Task（含 TokenHash/TokenSalt/TokenLastEight）
    ↓
③ generateTaskContext() 构建上下文
    ├─ 调用 CreateAuthorizationToken() 生成 JWT (gitea_runtime_token)
    ├─ 注入 Task Token 到 gitCtx["token"]
    ├─ 注入 JWT 到 gitCtx["gitea_runtime_token"]
    ↓
④ structpb.NewStruct(gitCtx) 序列化为 protobuf Struct
    ↓
⑤ task.Context 字段设置为序列化结果
    ↓
⑥ Task 对象通过 FetchTaskResponse 返回给 Runner
    ↓
⑦ Runner 解析 task.Context 获取两个 Token
    ├─ token: 用于 Git 认证 (actions/checkout)
    └─ gitea_runtime_token: 用于 Gitea API 调用 (artifact/cache)
```

**步骤 1：Task Token 生成** (`models/actions/utils.go:21-27` → `models/actions/task.go:147-149`)
```go
func generateSaltedToken() (string, string, string, string) {
    salt := util.CryptoRandomString(10)           // 10 位随机盐
    buf := util.CryptoRandomBytes(20)             // 20 字节随机数
    token := hex.EncodeToString(buf)              // 40 位十六进制字符串（明文，仅返回一次给 Runner）
    hash := auth_model.HashToken(token, salt)      // SHA256(token + salt)
    return token, salt, hash, token[len(token)-8:] // 明文、盐、哈希、后8位
}

func (task *ActionTask) GenerateAndFillToken() {
    task.Token, task.TokenSalt, task.TokenHash, task.TokenLastEight = generateSaltedToken()
}
```

**存储策略**:
- `task.Token`: 明文 Token，**仅在内存中短暂存在**，序列化到 Context 后不再返回
- `task.TokenHash`: SHA256 哈希，持久化到数据库
- `task.TokenSalt`: 盐值，持久化到数据库
- `task.TokenLastEight`: 后 8 位，用于快速查询过滤

**步骤 2：JWT (`gitea_runtime_token`) 生成** (`services/actions/auth.go:41-73`)
```go
func CreateAuthorizationToken(taskID, runID, jobID int64) (string, error) {
    now := time.Now()

    // 缓存权限序列化
    ac, err := json.Marshal(&[]actionsCacheScope{
        {
            Scope:      "",
            Permission: actionsCachePermissionWrite, // 写权限
        },
    })
    if err != nil {
        return "", err
    }

    claims := actionsClaims{
        RegisteredClaims: jwt.RegisteredClaims{
            Issuer:    "gitea",
            Subject:   strconv.FormatInt(taskID, 10),
            ExpiresAt: jwt.NewNumericDate(now.Add(1*time.Hour + setting.Actions.EndlessTaskTimeout)),
            IssuedAt:  jwt.NewNumericDate(now),
            ID:        util.CryptoRandomString(10),
        },
        Scp:    fmt.Sprintf("Actions.Results:%d:%d", runID, jobID), // 作用域声明
        TaskID: taskID,
        RunID:  runID,
        JobID:  jobID,
        Ac:     string(ac), // 缓存权限
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(setting.GetGeneralTokenSigningSecret())
}
```

**JWT 验证** (`services/actions/auth.go:75-111`):
```go
func ParseAuthorizationToken(req *http.Request) (int64, error) {
    authHeader := req.Header.Get("Authorization")
    // Bearer 前缀解析 ...

    token, err := jwt.ParseWithClaims(tokenStr, &actionsClaims{}, func(t *jwt.Token) (any, error) {
        // 验证签名算法
        if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return setting.GetGeneralTokenSigningSecret(), nil
    })

    if claims, ok := token.Claims.(*actionsClaims); ok && token.Valid {
        return claims.TaskID, nil // 返回 TaskID，用于查找任务
    }
    return 0, errors.New("invalid token")
}
```

**步骤 3：上下文构建与注入** (`services/actions/task.go:114-125` → `services/actions/context.go:26-121`)
```go
func generateTaskContext(ctx context.Context, t *actions_model.ActionTask) (*structpb.Struct, error) {
    // 生成 JWT
    giteaRuntimeToken, err := CreateAuthorizationToken(t.ID, t.Job.RunID, t.JobID)
    if err != nil {
        return nil, err
    }

    // 生成基础上下文（不含 token）
    gitCtx := GenerateGiteaContext(ctx, t.Job.Run, nil, t.Job)

    // 注入两个 Token 到上下文
    gitCtx["token"] = t.Token                // Task Token，用于 Git 认证
    gitCtx["gitea_runtime_token"] = giteaRuntimeToken // JWT，用于 API 调用

    // 序列化为 protobuf Struct
    return structpb.NewStruct(gitCtx)
}
```

**步骤 4：Runner 侧使用场景（仓内代码可证 vs 推断边界）**
1. **仓内代码可证**：两个 Token 被注入到 `task.Context` 的 `gitCtx` map 中
   - `gitCtx["token"]` = Task Token 明文
   - `gitCtx["gitea_runtime_token"]` = JWT 字符串
   - 见 `services/actions/task.go:114-125`

2. **仓内代码可证**：服务端 Token 验证逻辑
   - Task Token 验证：`GetRunningTaskByToken(token)` → 恒定时间哈希比较
     见 `models/actions/task.go:163-214`
   - JWT 验证：`ParseAuthorizationToken(req)` → HS256 验签
     见 `services/actions/auth.go:75-111`

3. **⚠️ 【推断】** Runner 侧 Token 使用方式：
   - ⚠️ **【推断】** Runner 将 `gitCtx["token"]` 设置为环境变量 `GIT_AUTH_TOKEN`
   - ⚠️ **【推断】** `actions/checkout` 使用该 Token 通过 HTTPS 克隆代码
   - ⚠️ **【推断】** Runner 将 `gitCtx["gitea_runtime_token"]` 设置为环境变量 `GITEA_RUNTIME_TOKEN`
   - ⚠️ **【推断】** `actions/upload-artifact` / `actions/cache` 等 Action 使用该 JWT 调用 Gitea API
   - > 注：以上 Runner 侧和第三方 Action 的行为不在当前仓内代码范围内，属于合理推断

**双 Token 职责分离**:

| 特性 | Task Token | gitea_runtime_token (JWT) |
|------|-----------|--------------------------|
| **生成方式** | 随机字符串 + Salted SHA256 | JWT HS256 签名 |
| **存储方式** | 仅存哈希（不可逆） | 无需存储（自包含） |
| **认证方式** | 哈希比较 | JWT 验签 |
| **有效条件** | Task 状态为 Running/Cancelling | 签名有效 + 未过期 |
| **主要用途** | Git 代码克隆认证 | Gitea API 调用授权 |
| **权限范围** | 仅限对应 Task 的代码仓库 | 对应 Task 的 Actions 权限（artifact/cache 等） |
| **失效时机** | Task 终态（Success/Failure/Cancelled/Skipped） | 1 小时 + EndlessTaskTimeout 后过期 |

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

**仓内代码可证的状态流转**：
```
StatusWaiting/Blocked → CancelJobs() → StatusCancelled (直接终态)
                       见 models/actions/run.go:310-328

StatusRunning → StopTask(StatusCancelling) → StatusCancelling
                       见 models/actions/task.go:431-505

StatusRunning → StopTask(StatusCancelled) → StatusCancelled
                       (不支持 Cancelling 时直接终态)
                       见 models/actions/task.go:460-466
```

**推断边界标注**：
```
StatusCancelling (服务端状态)
    ↓ ⚠️ 【推断】Runner 支持 Cancelling 时
    ↓ ⚠️ 【推断】执行 post-step cleanup
    ↓
    UpdateTask(Result=*) → StatusCancelled (保留用户意图)
                       见 models/actions/task.go:395-398
```
> 注："执行 post-step cleanup" 是 Runner 侧行为，不在当前仓内代码范围内，属于合理推断

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

##### 5.2.2.1 任务取消信号返回路径修正

**主通道：`UpdateTaskResponse`（正在执行任务的取消信号回传）**
这是 **正在运行任务的唯一取消信号回传通道**，仓内代码可证。

**仓内代码可证的链路** (`routers/api/actions/runner/runner.go:281-287` → `models/actions/status.go:102-115`):
```go
// 1. 仓内可证：StopTask 将 task.Status 设置为 StatusCancelling 或 StatusCancelled
//    见 models/actions/task.go:431-499 StopTask

// 2. 仓内可证：UpdateTask 响应返回 task.Status.AsResult()
func (s *Service) UpdateTask(ctx, req) (*Response[UpdateTaskResponse], error) {
    // ...
    return connect.NewResponse(&runnerv1.UpdateTaskResponse{
        State: &runnerv1.TaskState{
            Id:     req.Msg.State.Id,
            Result: task.Status.AsResult(), // 仓内可证：状态映射
        },
        SentOutputs: sentOutputs,
    }), nil
}

// 3. 仓内可证：StatusCancelling 和 StatusCancelled 都映射为 RESULT_CANCELLED
func (s Status) AsResult() runnerv1.Result {
    switch s {
    case StatusSuccess:
        return runnerv1.Result_RESULT_SUCCESS
    case StatusFailure:
        return runnerv1.Result_RESULT_FAILURE
    case StatusCancelled, StatusCancelling:  // 仓内可证：两个状态都映射
        return runnerv1.Result_RESULT_CANCELLED
    case StatusSkipped:
        return runnerv1.Result_RESULT_SKIPPED
    default:
        return runnerv1.Result_RESULT_UNSPECIFIED
    }
}
```

**仓内代码可证的机制**:
- `StatusCancelling` 和 `StatusCancelled` 在 `AsResult()` 中都映射为 `runnerv1.Result_RESULT_CANCELLED`
- 设计意图（仓内可证）：通过 Runner 的正常心跳（`UpdateTask`）通道返回取消信号，无需额外长连接或推送机制

**Runner 收到 CANCELLED 后的行为（仓内代码可证 vs 推断边界）**:
- ✅ **仓内代码可证**：`UpdateTaskResponse.State.Result` 字段会返回 `RESULT_CANCELLED` 给调用方
- ⚠️ **【推断】** Runner 收到 `RESULT_CANCELLED` 后会终止当前步骤执行
- ⚠️ **【推断】** Runner 会执行 `post:` 和 `always:` 步骤进行清理
- ⚠️ **【推断】** Runner 会通过下一次 `UpdateTask` 调用上报清理后的最终结果
- > 注：以上三点 Runner 侧行为不在当前仓内代码范围内，属于基于 Gitea Actions 协议约定的合理推断

**FetchTask 不是正在运行任务的取消信号回传通道**
- ✅ **仓内代码可证**：`FetchTask` 仅返回 `tasksVersion` 不一致时新分配的任务对象
  见 `routers/api/actions/runner/runner.go:177-220`
- ✅ **仓内代码可证**：`FetchTask` 响应结构中没有"当前运行任务状态"字段
  仅包含 `Task task` 和 `int64 tasksVersion`
- ✅ **仓内代码可证**：对于未分配 Task 的 Job（`task_id=0`），`CancelJobs` 直接设置为 `StatusCancelled`
  见 `models/actions/run.go:310-328`

**FetchTask 的作用域（仓内代码可证）**:
- 仅在 Runner 空闲、正在轮询新任务时工作
- 不参与任何正在运行任务的取消信号回传
- 未分配 Task 的 Job 取消直接在服务端完成，无需通知 Runner

**状态映射关系表** (`models/actions/status.go:13-27, 102-130`):
| 内部 Status | proto Result | 含义 |
|------------|-------------|------|
| `StatusWaiting` (5) | `UNSPECIFIED` (0) | 非终态 |
| `StatusRunning` (6) | `UNSPECIFIED` (0) | 非终态 |
| `StatusBlocked` (7) | `UNSPECIFIED` (0) | 非终态 |
| `StatusCancelling` (8) | `CANCELLED` (3) | 过渡态，映射为取消 |
| `StatusSuccess` (1) | `SUCCESS` (1) | 成功 |
| `StatusFailure` (2) | `FAILURE` (2) | 失败 |
| `StatusCancelled` (3) | `CANCELLED` (3) | 已取消 |
| `StatusSkipped` (4) | `SKIPPED` (4) | 已跳过 |

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

**仓内代码可证部分**：
```
User/System                             Gitea                                  Runner
  |                                        |                                        |
  | 取消操作                                |                                        |
  |--------------------------------------->|                                        |
  |                                        | StopTask(StatusCancelling)             |
  |                                        | 检查 Runner Cancelling 支持              |
  |                                        | 更新 task.Status = Cancelling           |
  |                                        |                                        |
  |                                        | 🔄 Runner 定期调用 UpdateTask 上报状态    |
  |                                        |<---------------------------------------|
  |                                        |                                        |
  |                                        | UpdateTaskResponse.State.Result        |
  |                                        |   = task.Status.AsResult()              |
  |                                        |   = RESULT_CANCELLED                    |
  |                                        |--------------------------------------->|
```

**推断边界标注**：
- ⚠️ **【推断】** Runner 收到 `RESULT_CANCELLED` 后会终止当前步骤执行
- ⚠️ **【推断】** Runner 会执行 `post:` 和 `always:` 清理步骤
- ⚠️ **【推断】** Runner 会通过下一次 `UpdateTask` 上报最终结果
- > 注：正在运行任务的取消信号 **仅通过 UpdateTaskResponse 返回**，FetchTask 不参与此流程

**后续服务端处理（仓内代码可证）**：
```
  |                                        |                                        |
  |                                        | UpdateTask(Result=*)                   |
  |                                        |<---------------------------------------|
  |                                        | 🔹 如原状态为 Cancelling                 |
  |                                        |    → 强制设置为 Cancelled               |
  |                                        |    (保留用户取消意图，不被 Runner 结果覆盖)|
  |                                        | 🔹 归档日志                              |
  |                                        | 🔹 触发后续作业调度                      |
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

### 8.1 仓内代码可证的设计特点

1. **拉模式调度**：Runner 主动轮询，避免服务端主动连接问题
2. **三级乐观锁版本控制**：通过 global/owner/repo 三层 `tasks_version` 减少无效轮询
3. **三级令牌体系**：注册令牌、Runner 身份令牌、任务运行令牌职责分离
4. **两级日志存储**：DBFS 高效追加写 + 对象存储长期归档
5. **取消信号主通道**：正在运行任务的取消信号 **仅通过 `UpdateTaskResponse` 返回**，`FetchTask` 不参与
6. **状态映射设计**：`StatusCancelling` 和 `StatusCancelled` 都映射为 `RESULT_CANCELLED`
7. **防时序攻击**：所有令牌比较使用 `subtle.ConstantTimeCompare`
8. **去重机制**：`jobEmitterQueue` 使用 `ErrAlreadyInQueue` 防止重复调度
9. **版本递增原子性**：三层版本递增在同一数据库事务中完成

### 8.2 推断边界说明（仓内代码不可证部分）

以下内容基于 Gitea Actions 协议约定的合理推断，不在当前仓内代码范围内：
- Runner 收到 `RESULT_CANCELLED` 后会终止当前步骤并执行 `post:`/`always:` 清理
- Runner 将 `gitCtx["token"]` 设为 `GIT_AUTH_TOKEN` 环境变量
- Runner 将 `gitCtx["gitea_runtime_token"]` 设为 `GITEA_RUNTIME_TOKEN` 环境变量
- `actions/checkout`/`actions/upload-artifact` 等第三方 Action 使用相应 Token

### 8.3 关键边界说明（避免误推）

1. **版本递增 ≠ 任务可分配**：版本递增只是"通知信号"，任务能否分配还需经过 Runner 禁用、标签匹配、乐观锁等多重检查
2. **组织版本递增 ≠ 存在组织级 Runner**：版本递增与 Runner 存在是两个独立条件，无组织级 Runner 时任务可由全局或仓库级 Runner 消费
3. **`tasksVersion=0` 特殊处理**：首次访问时自动初始化版本记录，避免返回 0 被误认为旧版 Gitea
