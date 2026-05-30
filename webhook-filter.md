# Webhook 投递失败链路状态表

## Deliver 函数执行顺序与关键检查点

**代码位置**: `services/webhook/deliver.go:148-272`

```
Deliver() 执行流程（行号标注）

[150] w, err := GetWebhookByID(t.HookID)
[152]   if err != nil { return err }                        ← 失败点 A

[165] t.IsDelivered = true                                  ← 内存标记（仅进程内）

[167] newRequest := webhookRequesters[w.Type]
[172] req, body, err := newRequest(ctx, w, t)
[173]   if err != nil { return fmt.Errorf("cannot create") } ← 失败点 B

[178] t.RequestInfo = &HookRequest{...}

[189] authorization, err := w.HeaderAuthorization()
[190]   if err != nil { return fmt.Errorf("cannot get Auth") } ← 失败点 C

[198] t.ResponseInfo = &HookResponse{...}

════════════════════════════════════════════════════════════
                    MarkTaskDelivered 边界
════════════════════════════════════════════════════════════

[204] updated, err := MarkTaskDelivered(ctx, t)             ← 原子写 DB
[206]   if err != nil { return fmt.Errorf("unable to mark") } ← 失败点 D
[209]   if !updated { return nil }                           ← 幂等短路

[216] defer func() {                                         ← 注册最终落库
[217]   t.Delivered = timeutil.TimeStampNanoNow()
[226]   UpdateHookTask(ctx, t)                              ← 全量写回
[231-235] UpdateWebhookLastStatus(ctx, w)                   ← 更新状态
[240] }()

[242] if setting.DisableWebhooks { return fmt.Errorf("...") }  ← 失败点 E

[246] if !w.IsActive { return nil }                         ← 失败点 F

[251] resp, err := webhookHTTPClient.Do(req)
[252]   if err != nil {                                     ← 失败点 G
[253]     t.ResponseInfo.Body = fmt.Sprintf("Delivery: %v", err)
[254]     return fmt.Errorf("unable to deliver")
[255]   }

[259] t.IsSucceed = resp.StatusCode/100 == 2
[265] p, err := util.ReadWithLimit(resp.Body, ...)
[266]   if err != nil {                                     ← 失败点 H
[267]     t.ResponseInfo.Body = fmt.Sprintf("read body: %s", err)
[268]     return fmt.Errorf("unable to read response")
[269]   }
[270] t.ResponseInfo.Body = string(p)
[271] return nil                                            ← 成功
```

---

## 一、MarkTaskDelivered 之前的失败场景

此阶段特征：`MarkTaskDelivered` 尚未执行或执行失败，数据库 `is_delivered` 仍为 `false`。defer 块未注册，不会有 `UpdateHookTask` 调用。

### 场景 A：GetWebhookByID 失败

| 项目 | 值 |
|------|----|
| **触发条件** | webhook 配置被删除，或数据库故障 |
| **代码位置** | `deliver.go:151-154` |
| **内存 t.IsDelivered** | `false`（此行在 L165 之前，未执行到） |
| **DB is_delivered** | `false` |
| **DB is_succeed** | `false` |
| **RequestInfo** | `nil`，未落库 |
| **ResponseInfo** | `nil`，未落库 |
| **defer 是否注册** | 否 |
| **handler 对 error 的处理** | 仅 log.Error，返回 nil |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **恢复前提** | webhook 记录需已恢复，否则同错循环 |
| **实际可恢复性** | 低 — 若 webhook 已被删除，每次重启都会重试并失败 |

### 场景 B：请求构建失败（newRequest 返回 error）

| 项目 | 值 |
|------|----|
| **触发条件** | URL 格式无效、Content-Type 不合法、HTTP Method 不支持、Matrix txnID 计算失败 |
| **代码位置** | `deliver.go:172-175` |
| **内存 t.IsDelivered** | `true`（L165 已执行，但仅内存） |
| **DB is_delivered** | `false`（MarkTaskDelivered 未执行） |
| **DB is_succeed** | `false` |
| **RequestInfo** | `nil`（L178 未执行到），未落库 |
| **ResponseInfo** | `nil`，未落库 |
| **defer 是否注册** | 否 |
| **handler 对 error 的处理** | 仅 log.Error，返回 nil |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **恢复前提** | 请求构建的根因已修复（如 URL 格式修正） |
| **实际可恢复性** | 中 — URL 格式错误属持久性问题，重启后仍会失败；Matrix txnID 等瞬时错误可恢复 |

**newRequest 可能失败的具体子因**:

| 子因 | 来源函数 | 持久性 |
|------|----------|--------|
| URL 解析失败 | `url.Parse(w.URL)` | 持久 |
| http.NewRequest 失败 | Go 标准库 | 持久（非法 URL） |
| Content-Type 无效 | `newDefaultRequest:60` | 持久 |
| HTTP Method 不支持 | `newDefaultRequest:87,90` | 持久 |
| Matrix txnID 计算失败 | `getMatrixTxnID` | 瞬时（payload 编码） |

### 场景 C：授权头解析失败（HeaderAuthorization 返回 error）

| 项目 | 值 |
|------|----|
| **触发条件** | `secret.DecryptSecret` 解密失败（SECRET_KEY 变更、密文损坏） |
| **代码位置** | `deliver.go:189-191` |
| **内存 t.IsDelivered** | `true`（仅内存） |
| **DB is_delivered** | `false` |
| **DB is_succeed** | `false` |
| **RequestInfo** | 已在内存赋值（L178-186），但 **未落库**（defer 未注册） |
| **ResponseInfo** | `nil` |
| **defer 是否注册** | 否 |
| **handler 对 error 的处理** | 仅 log.Error，返回 nil |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **恢复前提** | SECRET_KEY 已恢复或 Authorization 头已重设 |
| **实际可恢复性** | 低 — SECRET_KEY 变更后所有加密字段均不可逆，需手动重设 |

**HeaderAuthorization 失败的具体原因**:

```go
// models/webhook/webhook.go:211-216
func (w Webhook) HeaderAuthorization() (string, error) {
    if w.HeaderAuthorizationEncrypted == "" {
        return "", nil    // 空 → 直接返回，不会失败
    }
    return secret.DecryptSecret(setting.SecretKey, w.HeaderAuthorizationEncrypted)
    // 失败条件: setting.SecretKey 与加密时不同，或密文被篡改
}
```

### 场景 D：MarkTaskDelivered 自身数据库操作失败

| 项目 | 值 |
|------|----|
| **触发条件** | 数据库连接中断、事务冲突等 |
| **代码位置** | `deliver.go:204-208` |
| **内存 t.IsDelivered** | `true`（仅内存） |
| **DB is_delivered** | `false`（UPDATE 失败，行未变更） |
| **DB is_succeed** | `false` |
| **RequestInfo** | 已在内存赋值，但 **未落库** |
| **ResponseInfo** | 已在内存赋值，但 **未落库** |
| **defer 是否注册** | 否（L216 在此之后） |
| **handler 对 error 的处理** | 仅 log.Error，返回 nil |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **实际可恢复性** | 高 — 数据库临时故障恢复后即可成功 |

---

## 二、MarkTaskDelivered 之后的失败场景

此阶段特征：`MarkTaskDelivered` 已成功（`is_delivered=true` 已写入 DB），defer 块已注册。无论后续发生什么，defer 都会执行 `UpdateHookTask` 全量写回和 `UpdateWebhookLastStatus` 状态更新。

### 场景 E：全局 Webhook 被禁用（DisableWebhooks = true）

| 项目 | 值 |
|------|----|
| **触发条件** | `setting.DisableWebhooks = true` |
| **代码位置** | `deliver.go:242-244` |
| **内存 t.IsDelivered** | `true` |
| **DB is_delivered** | `true`（MarkTaskDelivered 已写入） |
| **DB is_succeed** | `false`（从未设为 true） |
| **RequestInfo** | 已在内存赋值 → **defer 会落库** |
| **ResponseInfo** | 已在内存赋值 → **defer 会落库**（空 Body） |
| **defer 是否注册** | ✅ 是，会执行 |
| **Deliver 返回值** | `fmt.Errorf("webhook task skipped (webhooks disabled)")` |
| **handler 对 error 的处理** | log.Error，返回 nil |
| **w.LastStatus** | `HookStatusFail`（defer 中 is_succeed=false 分支） |
| **恢复入口** | ❌ 无自动恢复（is_delivered=true） |
| **手动重放** | ✅ 用户在 UI 点击"重放"创建新任务 |
| **注意** | 即使重新启用 Webhooks，此任务也不会被重试 |

### 场景 F：Webhook 未激活（IsActive = false）

| 项目 | 值 |
|------|----|
| **触发条件** | webhook 被用户禁用 |
| **代码位置** | `deliver.go:246-249` |
| **内存 t.IsDelivered** | `true` |
| **DB is_delivered** | `true` |
| **DB is_succeed** | `false` |
| **RequestInfo** | 已落库 |
| **ResponseInfo** | 已落库（空 Body） |
| **defer 是否注册** | ✅ 是，会执行 |
| **Deliver 返回值** | `nil`（不是 error！） |
| **handler 对 error 的处理** | 无 error，正常结束 |
| **w.LastStatus** | `HookStatusFail`（defer 中 is_succeed=false 分支） |
| **恢复入口** | ❌ 无自动恢复 |
| **手动重放** | ✅ 用户在 UI 点击"重放"创建新任务 |
| **特殊之处** | 返回 `nil` 而非 error，handler 不记录任何错误日志；defer 中日志为 "Hook delivery skipped as webhook is inactive" |

### 场景 G：HTTP 请求失败（网络错误 / 超时 / DNS 解析失败）

| 项目 | 值 |
|------|----|
| **触发条件** | 目标服务器不可达、超时、TLS 错误、代理失败、主机白名单拒绝 |
| **代码位置** | `deliver.go:251-255` |
| **内存 t.IsDelivered** | `true` |
| **DB is_delivered** | `true` |
| **DB is_succeed** | `false`（从未设为 true） |
| **RequestInfo** | 已落库（包含完整请求头和 Body） |
| **ResponseInfo.Body** | `"Delivery: <error message>"` → **defer 会落库** |
| **ResponseInfo.Status** | `0`（未赋值，无 HTTP 响应） |
| **defer 是否注册** | ✅ 是，会执行 |
| **Deliver 返回值** | `fmt.Errorf("unable to deliver webhook task[%d]...")` |
| **handler 对 error 的处理** | log.Error，返回 nil |
| **w.LastStatus** | `HookStatusFail` |
| **恢复入口** | ❌ 无自动恢复 |
| **手动重放** | ✅ 用户在 UI 点击"重放"创建新任务 |

**HTTP 失败的具体子因**:

| 子因 | 错误来源 | 可恢复性 |
|------|----------|----------|
| 目标服务器宕机 | `net/http: connection refused` | 瞬时 |
| 请求超时 | `context deadline exceeded` | 瞬时 |
| DNS 解析失败 | `lookup: no such host` | 持久或瞬时 |
| TLS 证书错误 | `certificate verify failed`（SkipTLSVerify=false） | 持久 |
| 主机白名单拒绝 | `webhook can only call allowed HTTP servers` | 持久 |
| 代理错误 | proxy 配置错误 | 持久 |

### 场景 H：HTTP 响应读取失败

| 项目 | 值 |
|------|----|
| **触发条件** | 响应 Body 超过 1MB 限制、连接中断 |
| **代码位置** | `deliver.go:265-269` |
| **内存 t.IsDelivered** | `true` |
| **DB is_delivered** | `true` |
| **DB is_succeed** | `false` |
| **RequestInfo** | 已落库 |
| **ResponseInfo.Body** | `"read body: <error>"` → 落库 |
| **ResponseInfo.Status** | HTTP 状态码已赋值 → 落库 |
| **defer 是否注册** | ✅ 是 |
| **Deliver 返回值** | `fmt.Errorf("unable to deliver...unable to read response body")` |
| **w.LastStatus** | `HookStatusFail` |
| **恢复入口** | ❌ 无自动恢复 |
| **手动重放** | ✅ |
| **注意** | 请求实际已到达目标服务器，目标可能已处理（非幂等接口有重复风险） |

---

## 三、综合状态对比表

### 3.1 MarkTaskDelivered 之前 vs 之后

| 维度 | 之前（A/B/C/D） | 之后（E/F/G/H） |
|------|-----------------|-----------------|
| **DB is_delivered** | `false` | `true` |
| **defer UpdateHookTask** | 未注册，不执行 | 已注册，必执行 |
| **RequestInfo 落库** | ❌ 不会 | ✅ 会 |
| **ResponseInfo 落库** | ❌ 不会 | ✅ 会 |
| **w.LastStatus 更新** | ❌ 不更新 | ✅ 更新为 Fail |
| **重启可恢复** | ✅ `populateWebhookSendingQueue` 重新入队 | ❌ is_delivered=true 被跳过 |
| **手动重放** | ✅ 也可，但重启已能自动恢复 | ✅ **唯一的恢复手段** |
| **UI 可见性** | 不出现在"最近投递"列表 | 出现在"最近投递"列表，标红失败 |

### 3.2 四种指定场景完整对照

| 场景 | 失败点 | 代码行 | DB is_delivered | DB is_succeed | Request 落库 | Response 落库 | LastStatus | 重启恢复 | 手动重放 |
|------|--------|--------|-----------------|---------------|-------------|--------------|------------|----------|----------|
| **请求构建失败** | B | `deliver.go:172` | `false` | `false` | ❌ | ❌ | 不更新 | ✅ | ✅ |
| **授权头解析失败** | C | `deliver.go:189` | `false` | `false` | ❌（内存有值但不落库） | ❌ | 不更新 | ✅ | ✅ |
| **未激活返回** | F | `deliver.go:246` | `true` | `false` | ✅ | ✅（空） | `Fail` | ❌ | ✅ |
| **HTTP 失败** | G | `deliver.go:251` | `true` | `false` | ✅ | ✅（含错误信息） | `Fail` | ❌ | ✅ |

### 3.3 所有失败点完整对照

| 失败点 | 触发条件 | 行号 | 相对边界 | DB is_delivered | defer 注册 | 重启恢复 | 手动重放 |
|--------|----------|------|----------|-----------------|-----------|----------|----------|
| A | GetWebhookByID 失败 | 152 | **之前** | `false` | ❌ | ✅ | ✅ |
| B | 请求构建失败 | 174 | **之前** | `false` | ❌ | ✅ | ✅ |
| C | 授权头解析失败 | 191 | **之前** | `false` | ❌ | ✅ | ✅ |
| D | MarkTaskDelivered DB 错误 | 207 | **边界上** | `false` | ❌ | ✅ | ✅ |
| E | DisableWebhooks | 243 | **之后** | `true` | ✅ | ❌ | ✅ |
| F | Webhook 未激活 | 248 | **之后** | `true` | ✅ | ❌ | ✅ |
| G | HTTP 请求失败 | 254 | **之后** | `true` | ✅ | ❌ | ✅ |
| H | 响应读取失败 | 268 | **之后** | `true` | ✅ | ❌ | ✅ |

---

## 四、恢复路径详解

### 4.1 重启恢复（仅限 is_delivered=false）

**触发**: Gitea 进程启动时

```
Init()
  └─ populateWebhookSendingQueue()
       └─ FindUndeliveredHookTaskIDs(ctx, lowerID)
            └─ WHERE is_delivered = false   ← 只查未投递的
                 └─ enqueueHookTask(taskID)
                      └─ hookQueue.Push(taskID)
                           └─ handler() → Deliver() 重新执行
```

**适用场景**: A / B / C / D

**限制**:
- 仅在启动时执行一次
- 若根因未修复，任务会在下次重启时再次失败，形成 **重启循环**
- 场景 A（webhook 已删除）属于永久性循环
- 场景 C（SECRET_KEY 变更）属于永久性循环

### 4.2 手动重放（唯一通用恢复方式）

**触发**: 用户在 Web UI 点击"重放"按钮

```
ReplayWebhook(ctx)
  └─ ReplayHookTask(ctx, hookID, uuid)
       └─ CreateHookTask(ctx, &HookTask{...})   ← 创建全新任务
            └─ enqueueHookTask(newTask.ID)       ← 新 ID 入队
```

**适用场景**: 所有（A/B/C/D/E/F/G/H）

**特点**:
- 创建全新 HookTask（新 ID、新 UUID）
- 新任务 `is_delivered=false`
- 旧任务记录保持不变
- 需用户主动操作

### 4.3 队列退避（对 webhook 不生效）

```
handler() 总是返回 nil → unhandled 为空 → 退避条件不满足 → 不触发
```

**结论**: 队列退避机制与 webhook 投递完全无关。

---

## 五、场景 F 的特殊语义

场景 F（`!w.IsActive`）是 MarkTaskDelivered 之后**唯一返回 `nil` 而非 error 的失败路径**。这带来几个特殊后果：

| 对比维度 | 场景 F（未激活） | 其他之后场景（E/G/H） |
|----------|-----------------|---------------------|
| Deliver 返回值 | `nil` | `error` |
| handler 日志级别 | 无 error 日志 | `log.Error` |
| defer 日志内容 | "Hook delivery skipped as webhook is inactive" | "Hook delivery failed" |
| 实际意义 | 用户主动禁用，不算异常 | 属于投递异常 |

**关键点**: 虽然 Deliver 返回 `nil`，但 `is_succeed=false`，`w.LastStatus=HookStatusFail`。这意味着：
- 禁用 webhook 期间的所有事件都会被标记为失败
- 重新激活后，`LastStatus` 仍为 `Fail`，直到下次成功投递才更新
- UI 上"最近投递"列表中会出现大量红色记录

---

## 六、代码溯源索引

| 代码位置 | 文件路径 | 行号 |
|----------|----------|------|
| GetWebhookByID | `services/webhook/deliver.go` | 151 |
| 内存 t.IsDelivered=true | `services/webhook/deliver.go` | 165 |
| 请求构建 newRequest | `services/webhook/deliver.go` | 172 |
| 授权头解析 HeaderAuthorization | `services/webhook/deliver.go` | 189 |
| 解密实现 DecryptSecret | `models/webhook/webhook.go` | 211-216 |
| MarkTaskDelivered（边界） | `services/webhook/deliver.go` | 204 |
| MarkTaskDelivered DB 实现 | `models/webhook/hooktask.go` | 188-195 |
| defer 注册（落库） | `services/webhook/deliver.go` | 216-240 |
| DisableWebhooks 检查 | `services/webhook/deliver.go` | 242 |
| IsActive 检查 | `services/webhook/deliver.go` | 246 |
| HTTP 请求发送 | `services/webhook/deliver.go` | 251 |
| 响应读取 | `services/webhook/deliver.go` | 265 |
| handler 处理器 | `services/webhook/webhook.go` | 77-104 |
| 重启恢复 populateWebhookSendingQueue | `services/webhook/deliver.go` | 338-366 |
| 查询未投递任务 | `models/webhook/hooktask.go` | 173-186 |
| 手动重放 ReplayHookTask | `models/webhook/hooktask.go` | 153-171 |
| UpdateHookTask 全量写回 | `models/webhook/hooktask.go` | 148-151 |
| BeforeUpdate 序列化 | `models/webhook/hooktask.go` | 73-80 |
