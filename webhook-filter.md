# Webhook 事件投递与过滤协作流程分析

## 整体架构概览

Webhook 系统采用**事件捕获 → 规则匹配 → 任务入队 → 异步投递 → 重试节流**的分层协作模式。核心代码分布在四个关键文件：

| 层级 | 文件路径 | 核心职责 |
|------|----------|----------|
| 事件捕获 | `services/webhook/notifier.go` | 监听系统事件，转换为 webhook 载荷 |
| 规则匹配 | `services/webhook/webhook.go` | 事件过滤、分支匹配、任务创建 |
| 任务模型 | `models/webhook/hooktask.go` | 投递任务存储、状态管理 |
| 投递执行 | `services/webhook/deliver.go` | HTTP 请求构建、发送、结果记录 |
| 队列节流 | `modules/queue/workerqueue.go` | 异步处理、去重、重试退避 |

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
    IsDelivered    bool
    Delivered      timeutil.TimeStampNano
    IsSucceed      bool
    RequestContent string                         `xorm:"LONGTEXT"`
    ResponseContent string                        `xorm:"LONGTEXT"`
}
```

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

---

## 四、重试节流机制

### 4.1 队列初始化

**代码位置**: `services/webhook/deliver.go:308-336`

```go
func Init() error {
    // HTTP 客户端配置（超时、TLS、代理、主机白名单）
    timeout := time.Duration(setting.Webhook.DeliverTimeout) * time.Second
    webhookHTTPClient = &http.Client{
        Timeout: timeout,
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                InsecureSkipVerify: setting.Webhook.SkipTLSVerify,
            },
            Proxy: webhookProxy(allowedHostMatcher),
            DialContext: hostmatcher.NewDialContext(...),
        },
    }
    
    // 创建唯一队列（去重）
    hookQueue = queue.CreateUniqueQueue(
        graceful.GetManager().ShutdownContext(),
        "webhook_sender",
        handler,
    )
    go graceful.GetManager().RunWithCancel(hookQueue)
    
    // 系统启动时恢复未投递任务
    go graceful.GetManager().RunWithShutdownContext(
        populateWebhookSendingQueue,
    )
    
    return nil
}
```

**关键配置**:
- `DeliverTimeout`: HTTP 请求超时（默认 5 秒）
- `SkipTLSVerify`: 是否跳过 TLS 证书验证
- `AllowedHostList`: 允许访问的主机白名单
- `ProxyURL`: 代理配置

### 4.2 唯一队列去重机制

**代码位置**: `modules/queue/queue.go:48-51`

```
A queue can be "simple" or "unique". 
A unique queue will try to avoid duplicate items.
Unique queue's "Has" function can be used to check whether an item 
is already in the queue, although it's not 100% reliable due to 
the lack of proper transaction support.
```

**去重效果**:
- 相同 `taskID` 不会重复入队（`ErrAlreadyInQueue` 被静默忽略）
- 避免网络抖动导致的重复投递

### 4.3 任务处理器

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
    return nil
}
```

**设计要点**:
- 返回 `nil` 表示所有任务已处理（无论成功失败都不自动重试）
- `HammerContext` 确保关机时仍能完成正在处理的任务

### 4.4 重试机制（队列层）

**代码位置**: `modules/queue/workergroup.go:103-125`

```go
func (q *WorkerPoolQueue[T]) doWorkerHandle(batch []T) {
    unhandled := q.safeHandler(batch...)
    
    // 全部失败时退避重试
    if len(unhandled) == len(batch) && unhandledItemRequeueDuration.Load() != 0 {
        log.Error("Queue %q failed to handle batch of %d items, backoff", 
            q.GetName(), len(batch))
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

**重试策略**:
- **退避时间**: 默认 1 秒（`unhandledItemRequeueDuration`）
- **重试条件**: 仅当整批全部失败时才退避
- **重试次数**: 无上限，直到成功或系统关闭
- **持久化**: 基于底层队列存储（LevelDB/Redis）确保不丢失

### 4.5 启动时恢复未投递任务

**代码位置**: `services/webhook/deliver.go:338-366`

```go
func populateWebhookSendingQueue(ctx context.Context) {
    lowerID := int64(0)
    for {
        // 批量查询未投递任务（每次 100 条）
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
```

---

## 五、投递执行阶段

### 5.1 投递主流程

**代码位置**: `services/webhook/deliver.go:148-272`

```go
func Deliver(ctx context.Context, t *webhook_model.HookTask) error {
    w, err := webhook_model.GetWebhookByID(ctx, t.HookID)
    
    t.IsDelivered = true
    
    // 根据 webhook 类型选择请求构建器
    newRequest := webhookRequesters[w.Type]
    if t.PayloadVersion == 1 || newRequest == nil {
        newRequest = newDefaultRequest
    }
    
    // 构建 HTTP 请求
    req, body, err := newRequest(ctx, w, t)
    
    // 记录请求信息
    t.RequestInfo = &webhook_model.HookRequest{
        URL:        req.URL.String(),
        HTTPMethod: req.Method,
        Headers:    map[string]string{},
        Body:       string(body),
    }
    
    // 原子标记: 确保只投递一次
    updated, err := webhook_model.MarkTaskDelivered(ctx, t)
    if !updated {
        log.Trace("Webhook Task[%d] already delivered", t.ID)
        return nil
    }
    
    // 更新状态的 defer 块
    defer func() {
        t.Delivered = timeutil.TimeStampNanoNow()
        // 更新任务记录
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
    
    // 发送 HTTP 请求
    resp, err := webhookHTTPClient.Do(req.WithContext(ctx))
    
    // 判断成功（2xx 状态码）
    t.IsSucceed = resp.StatusCode/100 == 2
    t.ResponseInfo.Status = resp.StatusCode
    
    // 读取响应（限制 1MB）
    p, err := util.ReadWithLimit(resp.Body, 1024*1024)
    t.ResponseInfo.Body = string(p)
    
    return nil
}
```

### 5.2 幂等性保障

**双重检查机制**:
1. **内存检查**: `handler` 函数中检查 `task.IsDelivered`
2. **数据库原子操作**: `MarkTaskDelivered` 使用条件更新

```go
// models/webhook/hooktask.go:188-195
func MarkTaskDelivered(ctx context.Context, task *HookTask) (bool, error) {
    count, err := db.GetEngine(ctx).ID(task.ID).
        Where("is_delivered = ?", false).
        Cols("is_delivered").
        Update(&HookTask{ID: task.ID, IsDelivered: true})
    return count != 0, err
}
```

### 5.3 请求签名与 Headers

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
    // ... GitHub 兼容 headers
    return nil
}
```

---

## 六、全流程时序图

```
用户操作 → 领域事件触发
    ↓
notify_service 广播事件
    ↓
[webhookNotifier.PushCommits] 事件捕获
    ├─ 转换为 API Payload
    └─ 调用 PrepareWebhooks
            ↓
[PrepareWebhooks] 收集 Webhook
    ├─ 仓库级 Webhooks
    ├─ 所有者级 Webhooks
    └─ 系统级 Webhooks
            ↓
[PrepareWebhook] 规则匹配（逐个过滤）
    ├─ ✗ DisableWebhooks? → 终止
    ├─ ✗ HasEvent? → 终止
    ├─ ✗ 空 Commit 优化? → 终止
    ├─ ✗ 分支过滤? → 终止
    └─ ✓ → 创建 HookTask 并入库
            ↓
[enqueueHookTask] 入队
    └─ hookQueue.Push(taskID)
            ↓
[队列 worker 池] 异步处理
    ├─ 批量拉取任务
    ├─ 去重检查
    └─ 调用 handler
            ↓
[handler] 任务处理
    ├─ 检查 IsDelivered（幂等）
    └─ 调用 Deliver
            ↓
[Deliver] 实际投递
    ├─ 构建 HTTP 请求（签名、Headers）
    ├─ MarkTaskDelivered（原子标记）
    ├─ 发送 HTTP 请求
    ├─ 记录请求/响应
    └─ 更新 Webhook 最后状态
            ↓
失败处理:
    ├─ handler 返回 nil（不自动重试）
    └─ 系统重启时 populateWebhookSendingQueue 重新入队
```

---

## 七、关键设计决策

### 7.1 无重试循环设计

**现象**: `handler` 函数总是返回 `nil`，不会触发队列层的自动重试。

**原因**:
- Webhook 投递失败通常是目标服务问题，立即重试成功率低
- 避免因大量失败请求耗尽系统资源
- 失败任务保留在数据库中，可通过 UI 手动重发或系统重启时恢复

### 7.2 两层幂等保护

| 层级 | 实现 | 作用 |
|------|------|------|
| 应用层 | `task.IsDelivered` 检查 | 快速跳过已处理任务 |
| 数据库层 | `MarkTaskDelivered` 条件更新 | 并发场景下确保唯一投递 |

### 7.3 唯一队列去重

**效果**: 相同 `taskID` 重复入队时返回 `ErrAlreadyInQueue` 并被静默忽略。

**适用场景**:
- 系统重启时 `populateWebhookSendingQueue` 可能重复入队
- 手动重发功能可能重复触发
- 网络抖动导致队列 Push 重试

### 7.4 Payload 版本设计

- **Version 1**: 存储发送给 URL 的最终 JSON（已按 webhook 类型转换）
- **Version 2**: 存储原始事件数据，投递时按 webhook 类型动态转换

**优势**: Version 2 支持重新投递时升级转换逻辑。

---

## 八、代码溯源索引

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
| 任务处理器 | `services/webhook/webhook.go` | 77-104 |
| 投递主函数 | `services/webhook/deliver.go` | 148-272 |
| 原子标记投递 | `models/webhook/hooktask.go` | 188-195 |
| 启动恢复任务 | `services/webhook/deliver.go` | 338-366 |
| 队列重试机制 | `modules/queue/workergroup.go` | 103-125 |
