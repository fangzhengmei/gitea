# Webhook 事件投递与过滤协作流程分析

## 整体架构概览

Webhook 系统采用**事件捕获 → 规则匹配 → 任务入队 → 异步投递**的分层协作模式。与之前理解不同的是：**失败任务没有自动重试机制**，所有任务一旦被 worker 取出处理，就会被标记为 `is_delivered=true`（无论成功失败）。

| 层级 | 文件路径 | 核心职责 |
|------|----------|----------|
| 事件捕获 | `services/webhook/notifier.go` | 监听系统事件，转换为 webhook 载荷 |
| 规则匹配 | `services/webhook/webhook.go` | 事件过滤、分支匹配、任务创建 |
| 任务模型 | `models/webhook/hooktask.go` | 投递任务存储、状态管理 |
| 投递执行 | `services/webhook/deliver.go` | HTTP 请求构建、发送、结果记录 |
| 队列管理 | `modules/queue/workerqueue.go` | 异步处理、去重 |

---

## 一、事件捕获阶段

### 1.1 注册监听入口

**代码位置**: `services/webhook/notifier.go:32-45`

```go
func init() {
    notify_service.RegisterNotifier(NewNotifier())
}

type webhookNotifier struct {
    notify_service.NullNotifier
}

func NewNotifier() notify_service.Notifier {
    return &webhookNotifier{}
}
```

**机制说明**:
- `webhookNotifier` 实现了 `notify_service.Notifier` 接口
- 通过 `init()` 函数在包加载时自动注册到全局通知服务
- 采用观察者模式，系统所有事件会广播给所有已注册的 notifier

### 1.2 事件处理方法集

**代码位置**: `services/webhook/notifier.go:47-1063`

系统为每种事件类型提供了对应的处理方法，部分关键方法包括：

| 事件类型 | 处理方法 | 触发时机 |
|----------|----------|----------|
| 代码推送 | `PushCommits` | 用户 push 代码时 |
| Issue 创建 | `NewIssue` | 新建 Issue 时 |
| PR 创建 | `NewPullRequest` | 新建 Pull Request 时 |
| PR 合并 | `MergePullRequest` | PR 被合并时 |
| 评论创建 | `CreateIssueComment` | 新增评论时 |
| 标签变更 | `IssueChangeLabels` | Issue/PR 标签变化时 |
| 里程碑变更 | `IssueChangeMilestone` | 里程碑变化时 |
| 代码评审 | `PullRequestReview` | PR 评审提交时 |
| Release 发布 | `NewRelease` | Release 发布时 |

**典型事件处理流程**（以 PushCommits 为例）：

```go
// services/webhook/notifier.go:646-668
func (m *webhookNotifier) PushCommits(ctx context.Context, pusher *user_model.User, 
    repo *repo_model.Repository, opts *repository.PushUpdateOptions, 
    commits *repository.PushCommits) {
    
    apiPusher := convert.ToUser(ctx, pusher, nil)
    apiCommits, apiHeadCommit, err := commits.ToAPIPayloadCommits(ctx, repo)
    
    if err := PrepareWebhooks(ctx, EventSource{Repository: repo}, 
        webhook_module.HookEventPush, &api.PushPayload{
            Ref:          opts.RefFullName.String(),
            Before:       opts.OldCommitID,
            After:        opts.NewCommitID,
            Commits:      apiCommits,
            TotalCommits: commits.Len,
            HeadCommit:   apiHeadCommit,
            Repo:         convert.ToRepo(ctx, repo, ...),
            Pusher:       apiPusher,
            Sender:       apiPusher,
        }); err != nil {
        log.Error("PrepareWebhooks: %v", err)
    }
}
```

**设计要点**:
1. 每个事件方法负责将领域模型转换为 API 载荷结构
2. 统一调用 `PrepareWebhooks(ctx, source, eventType, payload)` 进入下一阶段
3. 事件失败仅记录日志，不阻塞主业务流程

---

## 二、规则匹配阶段

### 2.1 Webhook 收集与分发

**代码位置**: `services/webhook/webhook.go:179-227`

```go
func PrepareWebhooks(ctx context.Context, source EventSource, 
    event webhook_module.HookEventType, p api.Payloader) error {
    
    var ws []*webhook_model.Webhook
    
    // 1. 收集仓库级 webhook
    if source.Repository != nil {
        repoHooks, err := db.Find[webhook_model.Webhook](ctx, 
            webhook_model.ListWebhookOptions{
                RepoID:   source.Repository.ID,
                IsActive: optional.Some(true),
            })
        ws = append(ws, repoHooks...)
    }
    
    // 2. 收集所有者（用户/组织）级 webhook
    if owner != nil {
        ownerHooks, err := db.Find[webhook_model.Webhook](ctx, 
            webhook_model.ListWebhookOptions{
                OwnerID:  owner.ID,
                IsActive: optional.Some(true),
            })
        ws = append(ws, ownerHooks...)
    }
    
    // 3. 收集系统级 webhook
    systemHooks, err := webhook_model.GetSystemWebhooks(ctx, optional.Some(true))
    ws = append(ws, systemHooks...)
    
    // 4. 逐个 webhook 进行过滤
    for _, w := range ws {
        if err := PrepareWebhook(ctx, w, event, p); err != nil {
            return err
        }
    }
    return nil
}
```

**三级 Webhook 生效范围**:
1. **系统级**: 管理员配置，对所有仓库生效
2. **所有者级**: 用户/组织配置，对其下所有仓库生效
3. **仓库级**: 具体仓库配置，仅对当前仓库生效

### 2.2 单 Webhook 过滤逻辑

**代码位置**: `services/webhook/webhook.go:135-177`

```go
func PrepareWebhook(ctx context.Context, w *webhook_model.Webhook, 
    event webhook_module.HookEventType, p api.Payloader) error {
    
    // 过滤器1: 全局开关
    if setting.DisableWebhooks {
        return nil
    }
    
    // 过滤器2: 事件类型匹配
    if !w.HasEvent(event) {
        return nil
    }
    
    // 过滤器3: 空 commit 推送优化
    if pushEvent, ok := p.(*api.PushPayload); ok &&
        w.Type != webhook_module.GITEA && w.Type != webhook_module.GOGS &&
        len(pushEvent.Commits) == 0 {
        return nil
    }
    
    // 过滤器4: 分支过滤
    if ref := getPayloadRef(p); ref != "" {
        if !checkBranchFilter(w.BranchFilter, ref) {
            return nil
        }
    }
    
    // 过滤通过: 创建任务并入队
    payload, err := p.JSONPayload()
    task, err := webhook_model.CreateHookTask(ctx, &webhook_model.HookTask{
        HookID:         w.ID,
        PayloadContent: string(payload),
        EventType:      event,
        PayloadVersion: 2,
    })
    
    return enqueueHookTask(task.ID)
}
```

### 2.3 事件类型匹配详解

**代码位置**: `models/webhook/webhook.go:171-184`

```go
func (w *Webhook) HasEvent(evt webhook_module.HookEventType) bool {
    // 模式1: 发送所有事件
    if w.SendEverything {
        return true
    }
    // 模式2: 仅推送事件
    if w.PushOnly {
        return evt == webhook_module.HookEventPush
    }
    // 模式3: 事件组映射（评审子事件归组）
    checkEvt := evt
    switch evt {
    case webhook_module.HookEventPullRequestReviewApproved,
         webhook_module.HookEventPullRequestReviewRejected,
         webhook_module.HookEventPullRequestReviewComment:
        checkEvt = webhook_module.HookEventPullRequestReview
    }
    // 模式4: 精确事件匹配
    return w.HookEvents[checkEvt]
}
```

### 2.4 分支过滤逻辑

**代码位置**: `services/webhook/webhook.go:114-130`

```go
func checkBranchFilter(branchFilter string, ref git.RefName) bool {
    // 空或通配符直接通过
    if branchFilter == "" || branchFilter == "*" || branchFilter == "**" {
        return true
    }
    // 编译 glob 模式
    g, err := glob.Compile(branchFilter)
    // 匹配分支名或完整 ref 名
    if ref.IsBranch() && g.Match(ref.BranchName()) {
        return true
    }
    return g.Match(ref.String())
}
```

**适用事件类型**（通过 `getPayloadRef` 提取 ref）:
- `HookEventPush` - 推送事件
- `HookEventCreate` - 创建分支/标签
- `HookEventDelete` - 删除分支/标签

---

## 三、任务创建与入队

### 3.1 HookTask 数据模型

**代码位置**: `models/webhook/hooktask.go:44-65`

```go
type HookTask struct {
    ID             int64                          `xorm:"pk autoincr"`
    HookID         int64                          `xorm:"index"`
    UUID           string                         `xorm:"unique"`
    PayloadContent string                         `xorm:"LONGTEXT"`
    PayloadVersion int                            `xorm:"DEFAULT 1"`  // v2: 原始事件
    EventType      webhook_module.HookEventType
    IsDelivered    bool                           // 关键状态字段
    Delivered      timeutil.TimeStampNano
    IsSucceed      bool                           // 投递结果
    RequestContent string                         `xorm:"LONGTEXT"`
    ResponseContent string                        `xorm:"LONGTEXT"`
}
```

**状态字段说明**:
- `IsDelivered`: 是否已被投递处理（**只要被 worker 取出处理就设为 true，无论成功失败**）
- `IsSucceed`: 投递是否成功（HTTP 2xx 视为成功）

### 3.2 任务创建

**代码位置**: `models/webhook/hooktask.go:119-130`

```go
func CreateHookTask(ctx context.Context, t *HookTask) (*HookTask, error) {
    t.UUID = gouuid.New().String()
    if t.Delivered == 0 {
        t.Delivered = timeutil.TimeStampNanoNow()
    }
    if t.PayloadVersion == 0 {
        return nil, errors.New("missing HookTask.PayloadVersion")
    }
    return t, db.Insert(ctx, t)
}
```

**新建任务初始状态**:
- `IsDelivered = false` - 未被处理
- `IsSucceed = false` - 未成功

### 3.3 入队操作

**代码位置**: `services/webhook/webhook.go:106-112`

```go
func enqueueHookTask(taskID int64) error {
    err := hookQueue.Push(taskID)
    if err != nil && err != queue.ErrAlreadyInQueue {
        return err
    }
    return nil
}
```

**队列特性**:
- 使用 `CreateUniqueQueue` 创建唯一队列，相同 taskID 不会重复入队
- `ErrAlreadyInQueue` 错误被静默忽略

---

## 四、任务状态流转与失败处理（核心修正部分）

### 4.1 队列初始化

**代码位置**: `services/webhook/deliver.go:308-336`

```go
func Init() error {
    // HTTP 客户端配置
    timeout := time.Duration(setting.Webhook.DeliverTimeout) * time.Second
    webhookHTTPClient = &http.Client{
        Timeout: timeout,
        Transport: &http.Transport{...},
    }
    
    // 创建唯一队列（去重）
    hookQueue = queue.CreateUniqueQueue(
        graceful.GetManager().ShutdownContext(),
        "webhook_sender",
        handler,
    )
    go graceful.GetManager().RunWithCancel(hookQueue)
    
    // 系统启动时恢复未被处理过的任务
    go graceful.GetManager().RunWithShutdownContext(
        populateWebhookSendingQueue,
    )
    
    return nil
}
```

### 4.2 任务处理器（handler）

**代码位置**: `services/webhook/webhook.go:77-104`

```go
func handler(items ...int64) []int64 {
    ctx := graceful.GetManager().HammerContext()
    
    for _, taskID := range items {
        task, err := webhook_model.GetHookTaskByID(ctx, taskID)
        if err != nil {
            continue
        }
        // 幂等检查: 已投递则跳过
        if task.IsDelivered {
            log.Trace("Task[%d] has already been delivered", task.ID)
            continue
        }
        // 实际投递
        if err := Deliver(ctx, task); err != nil {
            log.Error("Unable to deliver webhook task[%d]: %v", task.ID, err)
        }
    }
    
    return nil  // 关键点: 总是返回 nil，所有任务都视为"已处理"
}
```

⚠️ **关键发现 1**: handler 总是返回 `nil`，意味着：
- 无论任务成功或失败，都不会返回给队列层
- **队列的退避重试机制对 webhook 完全不生效**
- 不会有任何自动重试

### 4.3 Deliver 函数中的状态标记

**代码位置**: `services/webhook/deliver.go:148-272`

```go
func Deliver(ctx context.Context, t *webhook_model.HookTask) error {
    w, err := webhook_model.GetWebhookByID(ctx, t.HookID)
    
    // ⚠️ 关键发现 2: 内存中先设为 true（第 165 行）
    t.IsDelivered = true
    
    // ... 构建 HTTP 请求 ...
    
    t.ResponseInfo = &webhook_model.HookResponse{...}
    
    // ⚠️ 关键发现 3: 在发送 HTTP 请求之前，原子标记数据库（第 204 行）
    updated, err := webhook_model.MarkTaskDelivered(ctx, t)
    if !updated {
        log.Trace("Webhook Task[%d] already delivered", t.ID)
        return nil
    }
    
    // defer 块: 更新任务状态和 webhook 最后状态
    defer func() {
        t.Delivered = timeutil.TimeStampNanoNow()
        // 更新任务记录（包含 IsSucceed 状态）
        if err := webhook_model.UpdateHookTask(ctx, t); err != nil {
            log.Error("UpdateHookTask [%d]: %v", t.ID, err)
        }
        // 更新 webhook 最后状态
        if t.IsSucceed {
            w.LastStatus = webhook_module.HookStatusSucceed
        } else {
            w.LastStatus = webhook_module.HookStatusFail
        }
        webhook_model.UpdateWebhookLastStatus(ctx, w)
    }()
    
    // 检查 webhook 是否激活
    if !w.IsActive {
        return nil  // IsSucceed 保持 false，但不算"失败"
    }
    
    // 发送 HTTP 请求
    resp, err := webhookHTTPClient.Do(req.WithContext(ctx))
    if err != nil {
        t.ResponseInfo.Body = fmt.Sprintf("Delivery: %v", err)
        return err  // IsSucceed 保持 false
    }
    defer resp.Body.Close()
    
    // 判断成功（2xx 状态码）
    t.IsSucceed = resp.StatusCode/100 == 2
    t.ResponseInfo.Status = resp.StatusCode
    
    // 读取响应
    p, err := util.ReadWithLimit(resp.Body, 1024*1024)
    t.ResponseInfo.Body = string(p)
    
    return nil
}
```

⚠️ **关键发现 4**: `MarkTaskDelivered` 在 HTTP 请求发送**之前**执行！

```go
// models/webhook/hooktask.go:188-195
func MarkTaskDelivered(ctx context.Context, task *HookTask) (bool, error) {
    count, err := db.GetEngine(ctx).ID(task.ID).
        Where("is_delivered = ?", false).  // 只有 false 才更新
        Cols("is_delivered").
        Update(&HookTask{ID: task.ID, IsDelivered: true})  // 设为 true
    return count != 0, err
}
```

**状态流转结论**:
- 只要任务被 worker 取出并进入 `Deliver` 函数，`is_delivered` 就会被设为 `true`
- 这个标记发生在网络请求发送之前
- **无论投递成功还是失败，is_delivered 永远是 true**

### 4.4 启动时任务恢复

**代码位置**: `services/webhook/deliver.go:338-366`

```go
func populateWebhookSendingQueue(ctx context.Context) {
    lowerID := int64(0)
    for {
        // ⚠️ 关键发现 5: 只查询 is_delivered=false 的任务
        taskIDs, err := webhook_model.FindUndeliveredHookTaskIDs(ctx, lowerID)
        if err != nil {
            return
        }
        if len(taskIDs) == 0 {
            return
        }
        lowerID = taskIDs[len(taskIDs)-1]
        
        for _, taskID := range taskIDs {
            if err := enqueueHookTask(taskID); err != nil {
                log.Error("Unable to push HookTask[%d] to queue: %v", taskID, err)
            }
        }
    }
}

// models/webhook/hooktask.go:173-186
func FindUndeliveredHookTaskIDs(ctx context.Context, lowerID int64) ([]int64, error) {
    return tasks, db.GetEngine(ctx).
        Select("id").
        Table(new(HookTask)).
        Where("is_delivered=?", false).  // 只查未投递的
        And("id > ?", lowerID).
        Asc("id").
        Limit(batchSize).
        Find(&tasks)
}
```

⚠️ **关键发现 6**: `populateWebhookSendingQueue` 的真实作用
- 不是恢复"失败任务"，而是恢复**从未被处理过的任务**
- 场景：系统崩溃时，有些任务已创建但未被 worker 取走（is_delivered=false）
- 失败任务的 is_delivered=true，不会被这个函数处理

### 4.5 队列层的退避机制（与 webhook 无关）

**代码位置**: `modules/queue/workergroup.go:103-125`

```go
func (q *WorkerPoolQueue[T]) doWorkerHandle(batch []T) {
    unhandled := q.safeHandler(batch...)
    
    // 全部失败时退避重试
    if len(unhandled) == len(batch) && unhandledItemRequeueDuration.Load() != 0 {
        log.Error("Queue %q failed to handle batch, backoff", q.GetName())
        select {
        case <-q.ctxRun.Done():
        case <-time.After(time.Duration(unhandledItemRequeueDuration.Load())):
        }
    }
    
    // 未处理项重新入队
    for _, item := range unhandled {
        if err := q.Push(item); err != nil {
            log.Error("Failed to requeue item: %v", err)
        }
    }
}
```

⚠️ **关键发现 7**: 这个退避机制对 webhook **不生效**
- webhook 的 handler 总是返回 `nil`（`unhandled` 为空）
- 所以退避条件 `len(unhandled) == len(batch)` 永远为 false
- webhook 投递失败不会触发任何队列层的重试

### 4.6 手动重放机制（唯一的"重试"方式）

**代码位置**: `routers/web/repo/setting/webhook.go:721-740`

```go
// ReplayWebhook replays a webhook
func ReplayWebhook(ctx *context.Context) {
    hookTaskUUID := ctx.PathParam("uuid")
    
    orCtx, w := checkWebhook(ctx)
    if ctx.Written() {
        return
    }
    
    // 调用重放服务
    if err := webhook_service.ReplayHookTask(ctx, w, hookTaskUUID); err != nil {
        ctx.ServerError("ReplayHookTask", err)
        return
    }
    
    ctx.Flash.Success(ctx.Tr("repo.settings.webhook.delivery.success"))
    ctx.Redirect(fmt.Sprintf("%s/%d", orCtx.Link, w.ID))
}
```

**重放实现**:

```go
// models/webhook/hooktask.go:153-171
func ReplayHookTask(ctx context.Context, hookID int64, uuid string) (*HookTask, error) {
    task, exist, err := db.Get[HookTask](ctx, builder.Eq{"hook_id": hookID, "uuid": uuid})
    if !exist {
        return nil, ErrHookTaskNotExist{...}
    }
    
    // ⚠️ 关键: 创建一个全新任务，而不是修改旧的
    return CreateHookTask(ctx, &HookTask{
        HookID:         task.HookID,
        PayloadContent: task.PayloadContent,
        EventType:      task.EventType,
        PayloadVersion: task.PayloadVersion,
    })
}
```

**重放机制说明**:
- **触发方式**: 用户在 Web UI 点击"重放"按钮
- **实现原理**: 基于旧任务的 payload 创建一个**全新的 HookTask**
- **新任务状态**: `is_delivered=false`, `is_succeed=false`
- **旧任务状态**: 保持不变（历史记录）

---

## 五、投递执行阶段

### 5.1 请求签名与 Headers

**代码位置**: `services/webhook/deliver.go:97-146`

```go
func addDefaultHeaders(req *http.Request, secret []byte, 
    w *webhook_model.Webhook, t *webhook_model.HookTask, 
    payloadContent []byte) error {
    
    // HMAC 签名（SHA1 和 SHA256）
    if len(secret) > 0 {
        sig1 := hmac.New(sha1.New, secret)
        sig256 := hmac.New(sha256.New, secret)
        io.MultiWriter(sig1, sig256).Write(payloadContent)
        signatureSHA1 = hex.EncodeToString(sig1.Sum(nil))
        signatureSHA256 = hex.EncodeToString(sig256.Sum(nil))
    }
    
    // 标准 Headers（兼容 Gitea/Gogs/GitHub）
    req.Header.Add("X-Gitea-Delivery", t.UUID)
    req.Header.Add("X-Gitea-Event", event)
    req.Header.Add("X-Gitea-Signature", signatureSHA256)
    req.Header.Add("X-Hub-Signature", "sha1="+signatureSHA1)
    req.Header.Add("X-Hub-Signature-256", "sha256="+signatureSHA256)
    return nil
}
```

---

## 六、完整状态流转图（修正版）

```
任务创建 → HookTask{is_delivered: false, is_succeed: false}
    ↓
入队 → hookQueue.Push(taskID)
    ↓
队列 worker 取出
    ↓
handler() 调用 Deliver()
    ├─ 内存中设置 is_delivered=true
    └─ MarkTaskDelivered() 原子更新数据库
            ↓
            │ [is_delivered 现在一定是 true 了]
            ↓
发送 HTTP 请求
    ├─ 成功 (2xx) → is_succeed=true
    └─ 失败 (非2xx/网络错误) → is_succeed=false
            ↓
defer 块更新数据库状态
    ↓
最终状态: is_delivered=true, is_succeed=?


失败后的可能性:
├─ 队列层: 不会自动重试（handler 返回 nil）
├─ 系统重启: 不会恢复（is_delivered=true）
└─ 手动重放: 用户点击 → 创建新任务（全新的生命周期）
```

---

## 七、关键设计决策与真相

### 7.1 无自动重试设计

**代码事实**:
- `handler` 总是返回 `nil` → 队列层不重试
- `is_delivered` 在发送前就标记为 true → 重启也不会恢复
- 失败任务只能通过手动重放来"重试"

**设计意图推测**:
1. Webhook 投递失败通常是目标服务问题，立即重试成功率极低
2. 避免因大量失败请求耗尽系统资源（连接、内存、CPU）
3. 失败保留在数据库中，管理员可查看并决定是否手动重发
4. 手动重放创建新任务，便于追踪每次投递历史

### 7.2 提前标记 is_delivered

**代码事实**: `MarkTaskDelivered` 在 HTTP 请求发送之前执行

**设计意图**:
1. **幂等性保障**: 避免并发场景下重复投递
2. **防止雪崩**: 系统重启后不会重复处理已经在投递中的任务
3. **代价**: 任务处理过程中崩溃会导致该任务"丢失"（is_delivered=true 但实际未完成）

### 7.3 唯一队列去重

**效果**: 相同 `taskID` 重复入队时返回 `ErrAlreadyInQueue` 并被静默忽略。

**适用场景**:
- 系统重启时 `populateWebhookSendingQueue` 可能重复入队
- 手动重放功能不会触发（创建的是新任务，ID 不同）

### 7.4 Payload 版本设计

- **Version 1**: 存储发送给 URL 的最终 JSON（已按 webhook 类型转换）
- **Version 2**: 存储原始事件数据，投递时按 webhook 类型动态转换

**优势**: Version 2 支持重新投递时升级转换逻辑。

---

## 八、常见误解澄清表

| 误解 | 代码事实 | 证据位置 |
|------|----------|----------|
| 失败任务会自动重试 | handler 返回 nil，不会触发队列重试 | `services/webhook/webhook.go:103` |
| 系统重启会恢复失败任务 | 只恢复 is_delivered=false 的任务，失败任务 is_delivered=true | `models/webhook/hooktask.go:181` |
| is_delivered 表示"投递成功" | is_delivered 表示"已被处理"，无论成功失败 | `services/webhook/deliver.go:165,204` |
| 队列退避用于 webhook 失败 | webhook handler 总是返回 nil，退避逻辑不触发 | `modules/queue/workergroup.go:103` |
| 重放是重试旧任务 | 重放创建全新的 HookTask，旧任务保持不变 | `models/webhook/hooktask.go:165-170` |
| is_succeed=false 会被重试 | is_succeed 仅用于展示，不影响重试逻辑 | 全代码搜索无相关逻辑 |

---

## 九、代码溯源索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| Notifier 注册 | `services/webhook/notifier.go` | 32-45 |
| Push 事件处理 | `services/webhook/notifier.go` | 646-668 |
| Webhook 收集 | `services/webhook/webhook.go` | 179-227 |
| 单 Webhook 过滤 | `services/webhook/webhook.go` | 135-177 |
| 事件类型匹配 | `models/webhook/webhook.go` | 171-184 |
| 分支过滤 | `services/webhook/webhook.go` | 114-130 |
| HookTask 创建 | `models/webhook/hooktask.go` | 119-130 |
| 队列初始化 | `services/webhook/deliver.go` | 308-336 |
| 任务处理器 handler | `services/webhook/webhook.go` | 77-104 |
| 投递主函数 | `services/webhook/deliver.go` | 148-272 |
| 提前标记 is_delivered | `services/webhook/deliver.go` | 165,204 |
| 原子标记投递 | `models/webhook/hooktask.go` | 188-195 |
| 启动恢复任务 | `services/webhook/deliver.go` | 338-366 |
| 查询未投递任务 | `models/webhook/hooktask.go` | 173-186 |
| 队列退避机制 | `modules/queue/workergroup.go` | 103-125 |
| 手动重放路由 | `routers/web/repo/setting/webhook.go` | 721-740 |
| 重放实现 | `models/webhook/hooktask.go` | 153-171 |
