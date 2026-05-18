# 自托管运行器通信机制概念报告（基于Gitea Actions代码可验证实现）

## 1. 架构概述

Gitea Actions 采用**运行器主动拉取**的通信模式。运行器（act_runner）部署在用户环境中，通过 gRPC/Connect 协议与服务端通信，所有连接均由运行器发起，服务端仅响应请求。

核心通信接口（代码可验证）：
- `Register`：运行器注册
- `Declare`：运行器声明标签与版本
- `FetchTask`：拉取任务
- `UpdateTask`：上报任务状态
- `UpdateLog`：上传执行日志

## 2. 注册鉴权机制（代码可验证链路）

### 2.1 注册令牌体系

注册采用**两级令牌**机制：

**注册令牌（Registration Token）**
- 由管理员在服务端生成，分三个作用域：全局（OwnerID=0, RepoID=0）、组织（OwnerID>0, RepoID=0）、仓库（OwnerID=0, RepoID>0）
- 新令牌生成时自动将同一作用域的旧令牌标记为非激活
- 注册令牌仅用于注册流程，注册成功后不再使用

**运行器身份凭证**
- 注册成功后，服务端为运行器颁发：
  - `UUID`：运行器唯一标识（36字符）
  - `Token`：运行器认证令牌
- 令牌采用加盐哈希存储（`TokenHash` + `TokenSalt`），数据库中不保存明文

### 2.2 日常通信鉴权头

运行器在除 `Register` 外的所有请求中必须携带两个 HTTP 头（`interceptor.go` 可验证）：

```
x-runner-uuid: <runner-uuid>
x-runner-token: <runner-token>
```

**鉴权流程**（interceptor.go 可验证）：
1. 服务端通过 UUID 查找运行器记录
2. 使用存储的 `TokenSalt` 对请求 Token 进行哈希计算
3. 采用**恒定时间比较**（`subtle.ConstantTimeCompare`）防止时序攻击
4. 鉴权通过后，运行器对象注入请求上下文供后续处理

### 2.3 自动状态续活（代码可验证）

鉴权中间件在每次请求时自动更新运行器状态（`interceptor.go` 可验证）：
- 所有请求更新 `LastOnline` 时间戳
- `UpdateTask` 和 `UpdateLog` 请求额外更新 `LastActive` 时间戳
- 这两个时间戳是判断运行器在线状态的核心依据

## 3. 任务分发与版本门控（代码可验证链路）

### 3.1 任务版本机制

服务端维护 `ActionTasksVersion` 表，为三个层级维护递增版本号：
- 全局（OwnerID=0, RepoID=0）
- 组织（OwnerID>0, RepoID=0）
- 仓库（OwnerID=0, RepoID>0）

每次 `IncreaseTaskVersion` 调用会同时递增**全局+作用域**两级版本号（`tasks_version.go:75-100` 可验证）。

### 3.2 版本号递增触发点（代码可验证调用点）

以下路径会触发 `IncreaseTaskVersion` 递增：

| 调用点 | 触发条件 | 文件位置 |
|--------|---------|---------|
| 新Run入队 | 创建新Run且存在Waiting状态的Job | `services/actions/run.go:206` |
| Job状态变更 | Job状态变为Waiting时 | `models/actions/run_job.go:244` |
| 重跑（Rerun） | 重跑任务且存在Waiting状态的Job | `services/actions/rerun.go:262` |
| 运行器禁用切换 | 运行器启用/禁用状态变更时 | `models/actions/runner.go:316` |
| 版本初始化 | FetchTask时版本号为0，首次初始化 | `routers/api/actions/runner/runner.go:146` |

### 3.3 版本门控分发流程（代码可验证）

```
运行器 FetchTask 请求（runner.go:134-177 可验证）
    ↓
携带 tasksVersion（本地缓存的版本号）
    ↓
服务端获取 latestVersion（数据库中的版本号）
    ↓
┌─ 版本号相同？ ──┐
│                │
是                否
│                │
返回空任务        重新加载运行器状态（避免禁用竞态）
                调用 PickTask 尝试为运行器挑选任务
    ↓
返回任务（或空）+ latestVersion
    ↓
运行器更新本地缓存的 tasksVersion
```

**设计优势**：
- 大部分空轮询请求仅需一次版本号查询，数据库压力极小
- 版本号变更时才触发完整的任务匹配逻辑
- 有效避免了"惊群效应"，大量运行器同时轮询时不会造成数据库雪崩

### 3.4 任务挑选与绑定（代码可验证）

**PickTask 前置检查**（`services/actions/task.go:19-48` 可验证）：
1. 若运行器已禁用（`IsDisabled = true`），直接返回空
2. 若为一次性运行器（`Ephemeral = true`）：
   - 查询该运行器是否已有关联任务
   - 如有且任务状态为 Waiting/Running/Blocked：不分配新任务（单任务约束）
   - 如有但任务已结束：删除该运行器记录并返回错误

**CreateTaskForRunner 真实挑选逻辑**（`models/actions/task.go:229-333` 可验证，数据库事务内）：
1. 按运行器作用域过滤可执行的 Job
2. 查询条件：`task_id = 0 AND status = Waiting`（未被认领、等待中）
3. **排序规则**：按 `updated ASC, id ASC`（先按更新时间升序，再按ID升序）
4. 遍历符合条件的 Job，按顺序进行标签匹配
5. 找到第一个匹配的 Job 后，创建 `ActionTask` 记录并与运行器绑定
6. 将 Job 状态从 `Waiting` 改为 `Running`，关联 TaskID

> 注：代码中无"按优先级调度"逻辑，任务挑选严格遵循更新时间+ID的升序排列。

## 4. 任务归属校验（代码可验证链路）

运行器只能更新自己认领的任务和日志，服务端在两个接口进行严格校验：

### 4.1 UpdateTask 归属校验

`UpdateTaskByState` 中校验（`models/actions/task.go:366` 可验证）：
```go
} else if runnerID != task.RunnerID {
    return nil, errors.New("invalid runner for task")
}
```

**校验时机**：在事务内查询到 Task 记录后立即校验
**失败处理**：直接返回错误，不执行任何状态更新

### 4.2 UpdateLog 归属校验

`UpdateLog` 接口中校验（`routers/api/actions/runner/runner.go:259` 可验证）：
```go
} else if runner.ID != task.RunnerID {
    return nil, status.Errorf(codes.Internal, "invalid runner for task")
}
```

**校验时机**：在查询到 Task 记录后、处理日志数据前校验
**失败处理**：直接返回错误，不写入任何日志数据

### 4.3 校验设计意图

- 防止运行器越权操作其他运行器的任务
- 防止日志数据被恶意篡改或污染
- 确保任务状态变更的可信度

## 5. 日志回传与 ACK 断点处理（代码可验证链路）

### 5.1 日志上传协议

运行器通过 `UpdateLog` 接口分片上传日志（`runner.go:248-314` 可验证），请求参数：
- `TaskId`：任务 ID
- `Index`：本批次日志的起始行号（从 0 开始）
- `Rows`：日志行数组
- `NoMore`：是否为最后一批（日志结束标记）

### 5.2 服务端 ACK 机制（代码可验证）

服务端维护 `LogLength` 字段，表示已成功持久化的日志行数。

**断点处理逻辑**（`runner.go:264-276` 可验证）：
```
服务端 ack = task.LogLength
    ↓
如果请求 Index ≤ ack 且 Index + len(Rows) > ack
    → 裁剪 Rows，只保留 [ack - Index : ] 部分
    → 即丢弃已确认的重复行，只追加新行
    ↓
写入新日志行到存储
更新 LogLength += len(rows)
    ↓
返回 AckIndex = task.LogLength
```

**边界场景处理**（代码可验证）：
- 运行器重复发送已确认的日志：服务端直接返回当前 AckIndex，不重复写入（`runner.go:266-267`）
- 运行器发送的日志存在 gap（Index > ack）：即使带 NoMore 标记也拒绝，要求重试（`runner.go:273`）
- 日志已归档（`LogInStorage = true`）：拒绝写入，返回 `AlreadyExists`（`runner.go:278-280`）

### 5.3 日志归档（代码可验证）

当 `NoMore = true` 时（`runner.go:298-304` 可验证）：
1. 将日志从临时存储转移到永久存储（`TransferLogs`）
2. 标记 `LogInStorage = true`
3. 返回最终 AckIndex
4. 清理临时存储资源

## 6. 状态上报与续活机制（代码可验证链路）

### 6.1 任务状态更新

运行器通过 `UpdateTask` 接口上报任务执行状态（`runner.go:180-245` 可验证），核心字段：
- `State.Id`：任务 ID
- `State.Result`：任务结果（成功/失败/取消/超时）
- `State.Steps`：各步骤的状态、日志索引
- `Outputs`：任务输出变量

### 6.2 隐含续活设计（代码可验证）

`UpdateTaskByState` 的关键设计（`models/actions/task.go:350-396` 可验证）：
> 即使任务状态没有变化，也强制更新 `ActionTask.Updated` 时间戳
> 目的是避免任务被判定为"僵尸任务"

这意味着运行器定期调用 `UpdateTask` 即可维持任务的"存活"状态。

### 6.3 运行器状态判定（代码可验证）

基于两个时间戳计算运行器状态（`models/actions/runner.go:105-113` 可验证）：

| 状态 | 判定条件 |
|------|---------|
| ACTIVE（活跃） | `LastActive` 在 10 秒内 |
| IDLE（空闲） | `LastActive` 超过 10 秒，但 `LastOnline` 在 1 分钟内 |
| OFFLINE（离线） | `LastOnline` 超过 1 分钟 |

常量定义（`models/actions/runner.go:73-76` 可验证）：
- `RunnerOfflineTime = 1 * time.Minute`
- `RunnerIdleTime = 10 * time.Second`

## 7. 一次性运行器（Ephemeral Runner）约束与边界语义（代码可验证链路）

### 7.1 单任务约束

一次性运行器（`Ephemeral = true`）严格遵循"一生只执行一个任务"的约束，在 `PickTask` 中强制执行（`services/actions/task.go:30-48` 可验证）：

```
一次性运行器 FetchTask → PickTask
    ↓
查询该运行器是否已有关联任务
    ↓
┌─ 已有任务？ ──┐
│              │
是              否
│              │
├─ 任务状态为 Waiting/Running/Blocked？
│      │              │
│      是              否
│      │              │
│  返回空任务     删除该运行器
│              返回错误 "runner has been removed"
    ↓
正常挑选任务
```

**约束目的**：确保一次性运行器不会被重复使用，任务执行环境完全隔离。

### 7.2 完成后自动移除

任务完成时自动触发一次性运行器删除（`models/actions/task.go:343-345` 可验证）：
```go
if err == nil && task.Status.IsDone() && util.SliceContainsString(cols, "status") {
    return DeleteEphemeralRunner(ctx, task.RunnerID)
}
```

**触发条件**：
- 更新 Task 记录时 status 字段被修改
- 修改后的状态为"已完成"（成功/失败/取消/超时）

### 7.3 残留清理机制

为防止异常场景下一次性运行器残留，提供两层清理：

**自动清理**（`services/actions/cleanup.go:142-156` 可验证）：
- 定时任务 `CleanupEphemeralRunners` 定期扫描
- 删除条件：运行器为一次性，且关联的任务已结束（非 Waiting/Running/Blocked）

**仓库级清理**（`services/actions/cleanup.go:158-172` 可验证）：
- 删除仓库时触发 `CleanupEphemeralRunnersByPickedTaskOfRepo`
- 删除所有在该仓库执行过任务的一次性运行器

### 7.4 边界语义总结

| 场景 | 行为 |
|------|------|
| 一次性运行器已有活跃任务 | 拒绝分配新任务 |
| 一次性运行器已有已结束任务 | 删除运行器，拒绝新任务 |
| 任务状态变更为已完成 | 自动删除关联的一次性运行器 |
| 定时扫描 | 清理残留的一次性运行器 |
| 仓库删除 | 清理该仓库关联的所有一次性运行器 |

## 8. 异常断线与超时回收（代码可验证链路）

### 8.1 超时检测机制（代码可验证）

服务端通过三个独立的后台定时任务进行异常检测（`services/actions/clear_tasks.go` 可验证）：

**StopZombieTasks（僵尸任务清理）**
- 目标：`Status = Running` 且 `Updated` 超过 `ZombieTaskTimeout` 的任务
- 判定：运行器虽然认领了任务，但长时间未上报任何状态更新
- 处理：强制标记为失败，触发日志归档

**StopEndlessTasks（无限任务清理）**
- 目标：`Status = Running` 且 `Started` 超过 `EndlessTaskTimeout` 的任务
- 判定：任务虽然在续活，但执行时间超过合理上限
- 处理：强制标记为失败

**CancelAbandonedJobs（废弃任务清理）**
- 目标：`Status = Waiting/Blocked` 且 `Updated` 超过 `AbandonedJobTimeout` 的任务
- 判定：任务长时间未被任何运行器认领
- 处理：标记为已取消

### 8.2 任务回收流程（代码可验证）

当检测到异常任务时（`services/actions/clear_tasks.go:117-156` 可验证）：
1. 数据库事务内将任务状态改为 `Failure`
2. 更新关联的 `ActionRunJob` 状态
3. 将所有未完成的步骤也标记为失败
4. 触发日志归档（`TransferLogs`）
5. 发送通知（Webhook、邮件等）
6. 触发后续依赖任务的调度

### 8.3 运行器重连（代码可验证链路）

运行器断线重连后的唯一代码可验证流程：
1. 使用原有 UUID 和 Token 重新认证（无需重新注册）
2. 服务端更新 `LastOnline`，运行器重新变为在线状态
3. 重新加入任务池，通过 FetchTask 拉取新任务

> 注：关于"重连后查询既有任务并继续执行"属于实现外推，代码中无直接验证链路，因此不纳入本报告。

## 9. 设计总结（代码可验证）

### 9.1 核心设计原则

1. **极简轮询模型**：运行器主动拉取，适配各种网络环境（NAT、防火墙）
2. **版本门控优化**：通过版本号大幅降低空轮询的数据库压力，仅在特定事件时递增版本
3. **隐含式续活**：业务请求自带续活效果，无需单独的心跳接口
4. **三层超时防护**：僵尸任务、无限任务、废弃任务分别检测
5. **断点安全**：日志 ACK 机制确保数据不丢不重
6. **归属严格校验**：运行器只能更新自己的任务和日志
7. **一次性隔离**：Ephemeral 运行器单任务约束，执行完即销毁

### 9.2 可靠性保障

- 所有关键操作均在数据库事务内完成
- 任务状态机单向流转，防止状态混乱
- 运行器身份凭证加盐哈希，防止泄露
- 超时参数均可配置，适配不同规模的部署场景
