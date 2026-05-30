# Webhook 投递失败链路状态表

## Deliver 函数执行顺序与关键检查点

**代码位置**: `services/webhook/deliver.go:148-272`

```
Deliver() 执行流程（行号标注）

[150] w, err := GetWebhookByID(t.HookID)
[152]   if err != nil { return err }                        ← 失败点 A

[156] defer func() {                                         ← 注册 PANIC RECOVER
[157]   err := recover()                                    ← 捕获 panic，仅日志，不重抛
[163] }()

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

[216] defer func() {                                         ← 注册 UPDATE DEFER（LIFO 后注册先执行）
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

[256] defer resp.Body.Close()                               ← 注册 RESP CLOSE

[259] t.IsSucceed = resp.StatusCode/100 == 2
[265] p, err := util.ReadWithLimit(resp.Body, ...)
[266]   if err != nil {                                     ← 失败点 H
[267]     t.ResponseInfo.Body = fmt.Sprintf("read body: %s", err)
[268]     return fmt.Errorf("unable to read response")
[269]   }
[270] t.ResponseInfo.Body = string(p)
[271] return nil                                            ← 成功
```

**defer 执行顺序说明（LIFO 后进先出）**:
1. 先注册 → 后执行：L156 recover defer
2. 后注册 → 先执行：L216 UpdateHookTask defer、L256 resp.Body.Close defer
3. panic 发生时，按注册逆序执行 defer

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

## 三、Panic 分支分析（原文档缺失部分）

### 3.0 两层 recover 机制

Webhook 投递链路有两层 panic 保护：

| 层级 | 代码位置 | 捕获范围 | 行为 |
|------|----------|----------|------|
| 外层队列 | `modules/queue/workerqueue.go:243-257` | 整个 handler 调用 | 捕获未被内层捕获的 panic，日志记录，返回 nil |
| 内层 Deliver | `services/webhook/deliver.go:156-163` | GetWebhookByID 成功后的所有代码 | 捕获 panic，仅日志记录，不重抛，返回 nil |

**关键特性**:
- 两层 recover 都只记录日志，不重新抛出 panic
- 捕获 panic 后函数返回值为 `nil`（error 零值）
- handler 无法区分"正常成功"和"panic 被捕获"
- Go defer 按 LIFO（后进先出）顺序执行

### 3.1 Panic 场景 P0：GetWebhookByID 中发生 panic

**触发位置**: `deliver.go:151`，recover defer（L156）**尚未注册**

| 项目 | 值 |
|------|----|
| **触发条件** | 数据库查询时发生 panic（如驱动 Bug、内存错误） |
| **代码位置** | `deliver.go:151`（GetWebhookByID 内部） |
| **相对边界** | **MarkTaskDelivered 之前，且 recover 未注册** |
| **哪层 recover 捕获** | 外层队列 recover（workerqueue.go:248） |
| **DB is_delivered** | `false` |
| **DB is_succeed** | `false` |
| **RequestInfo** | ❌ 未落库 |
| **ResponseInfo** | ❌ 未落库 |
| **defer 执行顺序** | 内层 recover 未注册，只执行外层队列 recover |
| **UpdateHookTask defer 注册** | ❌ 未注册，不执行 |
| **Deliver 返回值** | 函数未正常返回，外层 handler 也不会收到 error（队列层返回 nil） |
| **w.LastStatus** | ❌ 不更新 |
| **handler 日志级别** | 外层队列记录 `log.Error("Recovered from panic...")` |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **实际可恢复性** | 中 — 取决于 panic 根因是否已修复 |

**执行流程**:
```
GetWebhookByID() → panic!
  → recover defer 未注册，向外传播
  → 队列层 safeHandler 的 recover 捕获
  → 记录 PANIC 日志
  → 返回 unhandled=nil
  → 任务从队列消失，但 DB is_delivered=false
  → 下次重启时重新入队
```

### 3.2 Panic 场景 P1：MarkTaskDelivered 之前发生 panic

**触发位置**: `deliver.go:156-203` 之间，recover defer（L156）**已注册**，UpdateHookTask defer（L216）**未注册**

**可能的 panic 点**:
- `newRequest` 中（URL 解析 panic、JSON 编码 panic）
- `t.RequestInfo.Headers` 赋值时（nil map 操作）
- `w.HeaderAuthorization()` 中（解密算法 panic）
- `MarkTaskDelivered` 之前的任意代码

| 项目 | 值 |
|------|----|
| **触发条件** | newRequest 内部 panic、HeaderAuthorization 解密 panic、nil map 操作等 |
| **代码位置** | `deliver.go:156-203` |
| **相对边界** | **MarkTaskDelivered 之前** |
| **哪层 recover 捕获** | 内层 Deliver recover（deliver.go:157） |
| **DB is_delivered** | `false`（MarkTaskDelivered 未执行） |
| **DB is_succeed** | `false` |
| **RequestInfo** | ❌ 未落库（UpdateHookTask defer 未注册） |
| **ResponseInfo** | ❌ 未落库 |
| **defer 执行顺序** | 只有内层 recover defer 执行，UpdateHookTask defer 未注册不执行 |
| **UpdateHookTask defer 注册** | ❌ 未注册 |
| **Deliver 返回值** | `nil`（error 零值，因为 panic 被捕获） |
| **handler 对返回值的处理** | `err == nil`，**认为成功**，不记录任何 error 日志 |
| **实际日志** | 只有内层 recover 记录 `PANIC whilst trying to deliver...` |
| **w.LastStatus** | ❌ 不更新 |
| **恢复入口** | ✅ **系统重启时 `populateWebhookSendingQueue` 重新入队** |
| **实际可恢复性** | 中 — 取决于 panic 根因 |
| **隐蔽问题** | handler 认为成功，但任务实际未执行，且无 error 日志，仅 PANIC 日志 |

**执行流程**:
```
GetWebhookByID() → 成功
  → 注册 recover defer (L156)
  → newRequest() / HeaderAuthorization() → panic!
  → 按 LIFO 执行 defer：
     1. recover defer 捕获 panic → 记录 PANIC 日志
     2. UpdateHookTask defer 未注册，不执行
  → Deliver 返回 nil（error 零值）
  → handler 判断 err == nil，认为成功
  → 任务从队列消失，但 DB is_delivered=false
  → 下次重启时重新入队
```

### 3.3 Panic 场景 P2：MarkTaskDelivered 之后发生 panic

**触发位置**: `deliver.go:205-271` 之间，recover defer（L156）和 UpdateHookTask defer（L216）**都已注册**

**可能的 panic 点**:
- `setting.DisableWebhooks` 访问时（配置系统 panic）
- `webhookHTTPClient.Do` 中（网络库 panic）
- `resp.StatusCode` 访问时（resp 为 nil，但此处不会，因为前面有 err 检查）
- `util.ReadWithLimit` 中（读取时 panic）
- `t.ResponseInfo.Body` 赋值时（nil pointer）
- defer 内部（UpdateHookTask 中 panic）

| 项目 | 值 |
|------|----|
| **触发条件** | HTTP 请求发送中 panic、响应读取 panic、任意代码缺陷 |
| **代码位置** | `deliver.go:205-271` |
| **相对边界** | **MarkTaskDelivered 之后** |
| **哪层 recover 捕获** | 内层 Deliver recover（deliver.go:157） |
| **DB is_delivered** | `true`（MarkTaskDelivered 已原子写入） |
| **DB is_succeed** | `false`（从未设为 true，除非 panic 在 L259 之后） |
| **RequestInfo** | ✅ **defer UpdateHookTask 会落库** |
| **ResponseInfo** | ✅ **defer UpdateHookTask 会落库**（内容取决于 panic 位置） |
| **defer 执行顺序** | 按 LIFO 逆序执行：<br>1. UpdateHookTask defer（L216）→ 全量写回<br>2. resp.Body.Close defer（L256，若已注册）<br>3. recover defer（L156）→ 捕获并记录日志 |
| **UpdateHookTask defer 注册** | ✅ 已注册，必执行 |
| **Deliver 返回值** | `nil`（error 零值） |
| **handler 对返回值的处理** | `err == nil`，**认为成功** |
| **实际日志** | 内层 recover 记录 PANIC 日志，UpdateHookTask 中记录 "Hook delivery failed" 日志 |
| **w.LastStatus** | `HookStatusFail`（defer 中 is_succeed=false 分支） |
| **恢复入口** | ❌ 无自动恢复（is_delivered=true） |
| **手动重放** | ✅ 用户在 UI 点击"重放"创建新任务 |
| **隐蔽问题** | handler 认为成功，UI 显示失败状态（LastStatus=Fail），日志有 PANIC 但无 Deliver error |

**执行流程**:
```
GetWebhookByID() → 成功
  → 注册 recover defer (L156)
  → MarkTaskDelivered() → 成功，DB is_delivered=true
  → 注册 UpdateHookTask defer (L216)
  → webhookHTTPClient.Do() / ReadWithLimit() → panic!
  → 按 LIFO 逆序执行 defer：
     1. resp.Body.Close defer（若已注册）
     2. UpdateHookTask defer：
        - t.IsSucceed 保持 false（panic 在 L259 之前）
        - t.RequestInfo / t.ResponseInfo 已赋值的部分会落库
        - w.LastStatus = HookStatusFail
        - UpdateHookTask 写 DB
     3. recover defer：
        - 捕获 panic
        - 记录 PANIC 日志
  → Deliver 返回 nil（error 零值）
  → handler 判断 err == nil，认为成功
  → 任务从队列消失，DB is_delivered=true
  → 无法自动恢复，只能手动重放
```

### 3.4 Panic 在 defer 内部的特殊情况

如果 panic 发生在 **UpdateHookTask defer 内部**（如 `UpdateHookTask` 数据库操作 panic）：

```
panic 发生在 defer 内部 → defer 链继续执行
  → recover defer 仍能捕获
  → 但 UpdateHookTask 的写库操作可能部分完成或完全未完成
  → 结果不确定，取决于 panic 发生的具体位置
```

---

## 四、综合状态对比表

### 4.1 MarkTaskDelivered 之前 vs 之后（含 panic 场景）

| 维度 | 之前（A/B/C/D/P0/P1） | 之后（E/F/G/H/P2） |
|------|-----------------------|---------------------|
| **DB is_delivered** | `false` | `true` |
| **defer UpdateHookTask** | 未注册，不执行 | 已注册，必执行 |
| **RequestInfo 落库** | ❌ 不会（包括 P1 panic 场景） | ✅ 会（包括 P2 panic 场景） |
| **ResponseInfo 落库** | ❌ 不会 | ✅ 会 |
| **w.LastStatus 更新** | ❌ 不更新 | ✅ 更新为 Fail |
| **重启可恢复** | ✅ `populateWebhookSendingQueue` 重新入队 | ❌ is_delivered=true 被跳过 |
| **手动重放** | ✅ 也可，但重启已能自动恢复 | ✅ **唯一的恢复手段** |
| **UI 可见性** | 不出现在"最近投递"列表 | 出现在"最近投递"列表，标红失败 |
| **handler 感知** | 普通错误可见，P0/P1 panic 不可见（返回 nil） | 普通错误可见，P2 panic 不可见（返回 nil） |

### 4.2 四种指定场景 + panic 场景完整对照

| 场景 | 失败点 | 代码行 | DB is_delivered | DB is_succeed | Request 落库 | Response 落库 | LastStatus | 重启恢复 | 手动重放 |
|------|--------|--------|-----------------|---------------|-------------|--------------|------------|----------|----------|
| **请求构建失败** | B | `deliver.go:172` | `false` | `false` | ❌ | ❌ | 不更新 | ✅ | ✅ |
| **授权头解析失败** | C | `deliver.go:189` | `false` | `false` | ❌（内存有值但不落库） | ❌ | 不更新 | ✅ | ✅ |
| **未激活返回** | F | `deliver.go:246` | `true` | `false` | ✅ | ✅（空） | `Fail` | ❌ | ✅ |
| **HTTP 失败** | G | `deliver.go:251` | `true` | `false` | ✅ | ✅（含错误信息） | `Fail` | ❌ | ✅ |
| **Panic: GetWebhookByID 中** | P0 | `deliver.go:151` | `false` | `false` | ❌ | ❌ | 不更新 | ✅ | ✅ |
| **Panic: MarkTaskDelivered 之前** | P1 | `deliver.go:156-203` | `false` | `false` | ❌（内存有值但 defer 未注册） | ❌ | 不更新 | ✅ | ✅ |
| **Panic: MarkTaskDelivered 之后** | P2 | `deliver.go:205-271` | `true` | `false` | ✅（defer 落库） | ✅（内容取决于 panic 位置） | `Fail` | ❌ | ✅ |

### 4.3 所有失败点 + panic 完整对照

| 失败点 | 触发条件 | 行号 | 相对边界 | DB is_delivered | defer 注册 | 重启恢复 | 手动重放 |
|--------|----------|------|----------|-----------------|-----------|----------|----------|
| A | GetWebhookByID 失败 | 152 | **之前** | `false` | ❌ | ✅ | ✅ |
| P0 | GetWebhookByID 中 panic | 151 | **之前** | `false` | ❌（recover 也未注册） | ✅ | ✅ |
| B | 请求构建失败 | 174 | **之前** | `false` | ❌ | ✅ | ✅ |
| C | 授权头解析失败 | 191 | **之前** | `false` | ❌ | ✅ | ✅ |
| P1 | 请求构建/授权头解析中 panic | 156-203 | **之前** | `false` | ❌（recover 已注册，Update 未注册） | ✅ | ✅ |
| D | MarkTaskDelivered DB 错误 | 207 | **边界上** | `false` | ❌ | ✅ | ✅ |
| E | DisableWebhooks | 243 | **之后** | `true` | ✅ | ❌ | ✅ |
| F | Webhook 未激活 | 248 | **之后** | `true` | ✅ | ❌ | ✅ |
| G | HTTP 请求失败 | 254 | **之后** | `true` | ✅ | ❌ | ✅ |
| H | 响应读取失败 | 268 | **之后** | `true` | ✅ | ❌ | ✅ |
| P2 | HTTP/响应读取中 panic | 205-271 | **之后** | `true` | ✅（两个 defer 都已注册） | ❌ | ✅ |

### 4.4 Panic 场景专项对比

| 维度 | P0 (GetWebhookByID 中) | P1 (MarkTaskDelivered 之前) | P2 (MarkTaskDelivered 之后) |
|------|-------------------------|-----------------------------|-----------------------------|
| **recover defer 注册状态** | 未注册 | 已注册 | 已注册 |
| **UpdateHookTask defer 注册状态** | 未注册 | 未注册 | 已注册 |
| **哪层 recover 捕获** | 外层队列 recover | 内层 Deliver recover | 内层 Deliver recover |
| **defer 执行顺序** | 只有外层队列 recover | 只有内层 recover | Update → resp.Close → recover |
| **DB is_delivered** | `false` | `false` | `true` |
| **Request/Response 落库** | ❌ | ❌ | ✅ |
| **w.LastStatus 更新** | ❌ | ❌ | ✅（Fail） |
| **Deliver 返回值** | 函数未正常返回 | `nil`（error 零值） | `nil`（error 零值） |
| **handler 感知** | 不可见 | 不可见（err==nil） | 不可见（err==nil） |
| **日志特征** | 队列层 "Recovered from panic" | Deliver 层 "PANIC whilst trying to deliver" | 两条日志：PANIC + "Hook delivery failed" |
| **重启可恢复** | ✅ | ✅ | ❌ |
| **隐蔽性** | 中 | 高（handler 认为成功） | 最高（handler 认为成功 + UI 显示失败） |

---

## 五、恢复路径详解

### 5.1 重启恢复（仅限 is_delivered=false）

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

**适用场景**: A / B / C / D / P0 / P1

**限制**:
- 仅在启动时执行一次
- 若根因未修复，任务会在下次重启时再次失败，形成 **重启循环**
- 场景 A（webhook 已删除）属于永久性循环
- 场景 C（SECRET_KEY 变更）属于永久性循环
- 场景 P0/P1（panic）若为代码缺陷导致，也属于永久性循环

### 5.2 手动重放（唯一通用恢复方式）

**触发**: 用户在 Web UI 点击"重放"按钮

```
ReplayWebhook(ctx)
  └─ ReplayHookTask(ctx, hookID, uuid)
       └─ CreateHookTask(ctx, &HookTask{...})   ← 创建全新任务
            └─ enqueueHookTask(newTask.ID)       ← 新 ID 入队
```

**适用场景**: 所有（A/B/C/D/E/F/G/H/P0/P1/P2）

**特点**:
- 创建全新 HookTask（新 ID、新 UUID）
- 新任务 `is_delivered=false`
- 旧任务记录保持不变
- 需用户主动操作

### 5.3 队列退避（对 webhook 不生效）

```
handler() 总是返回 nil → unhandled 为空 → 退避条件不满足 → 不触发
```

**结论**: 队列退避机制与 webhook 投递完全无关，包括 panic 场景。

---

## 六、场景 F 与 P1/P2 的特殊语义对比

场景 F（`!w.IsActive`）是 MarkTaskDelivered 之后**唯一返回 `nil` 而非 error 的普通失败路径**。但 P1/P2 panic 场景也返回 `nil`，形成更隐蔽的失败模式：

| 对比维度 | 场景 F（未激活） | 场景 P1（边界前 panic） | 场景 P2（边界后 panic） | 其他之后场景（E/G/H） |
|----------|-----------------|-----------------------|-----------------------|---------------------|
| Deliver 返回值 | `nil` | `nil` | `nil` | `error` |
| handler 日志级别 | 无 error 日志 | 无 error 日志 | 无 error 日志 | `log.Error` |
| defer 日志内容 | "skipped as webhook is inactive" | 只有 PANIC 日志 | PANIC + "delivery failed" | "Hook delivery failed" |
| DB is_delivered | `true` | `false` | `true` | `true` |
| Request/Response 落库 | ✅ | ❌ | ✅ | ✅ |
| w.LastStatus | `Fail` | 不更新 | `Fail` | `Fail` |
| handler 感知 | 不可见（返回 nil） | 不可见（返回 nil） | 不可见（返回 nil） | 可见（返回 error） |
| 重启可恢复 | ❌ | ✅ | ❌ | ❌ |
| 实际意义 | 用户主动禁用，不算异常 | 代码缺陷，隐蔽失败 | 代码缺陷，UI 可见失败 | 投递异常 |

**关键点**: 虽然 Deliver 返回 `nil`，但 `is_succeed=false`，`w.LastStatus=HookStatusFail`（对于 F 和 P2）。这意味着：
- 禁用 webhook 期间的所有事件都会被标记为失败
- 重新激活后，`LastStatus` 仍为 `Fail`，直到下次成功投递才更新
- UI 上"最近投递"列表中会出现大量红色记录
- P2 场景最隐蔽：handler 认为成功，但 UI 显示失败，日志需要同时检查 PANIC 和 delivery failed 两条记录

---

## 七、代码溯源索引

| 代码位置 | 文件路径 | 行号 |
|----------|----------|------|
| GetWebhookByID | `services/webhook/deliver.go` | 151 |
| 内层 panic recover defer | `services/webhook/deliver.go` | 156-163 |
| 内存 t.IsDelivered=true | `services/webhook/deliver.go` | 165 |
| 请求构建 newRequest | `services/webhook/deliver.go` | 172 |
| 授权头解析 HeaderAuthorization | `services/webhook/deliver.go` | 189 |
| 解密实现 DecryptSecret | `models/webhook/webhook.go` | 211-216 |
| MarkTaskDelivered（边界） | `services/webhook/deliver.go` | 204 |
| MarkTaskDelivered DB 实现 | `models/webhook/hooktask.go` | 188-195 |
| defer UpdateHookTask 注册 | `services/webhook/deliver.go` | 216-240 |
| DisableWebhooks 检查 | `services/webhook/deliver.go` | 242 |
| IsActive 检查 | `services/webhook/deliver.go` | 246 |
| HTTP 请求发送 | `services/webhook/deliver.go` | 251 |
| resp.Body.Close defer | `services/webhook/deliver.go` | 256 |
| 响应读取 | `services/webhook/deliver.go` | 265 |
| handler 处理器 | `services/webhook/webhook.go` | 77-104 |
| 外层队列 panic recover | `modules/queue/workerqueue.go` | 243-257 |
| 重启恢复 populateWebhookSendingQueue | `services/webhook/deliver.go` | 338-366 |
| 查询未投递任务 | `models/webhook/hooktask.go` | 173-186 |
| 手动重放 ReplayHookTask | `models/webhook/hooktask.go` | 153-171 |
| UpdateHookTask 全量写回 | `models/webhook/hooktask.go` | 148-151 |
| BeforeUpdate 序列化 | `models/webhook/hooktask.go` | 73-80 |
