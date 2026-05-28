# Gitea 服务端 Git Hook 在 Push 操作中的执行机制

## 一、总体架构：Hook 进程与 Gitea 主进程的协作

Gitea 的服务端 Git hook 并非在 Gitea 主进程内直接执行，而是采用 **"Git hook 脚本 → gitea hook 子命令 → 内部 HTTP API → Gitea 主进程"** 的架构：

```
  git push (SSH/HTTP)
        │
        ▼
  git-receive-pack 调用仓库 hooks/ 下的脚本
        │
        ▼
  hooks/pre-receive  ─── 遍历 hooks/pre-receive.d/* 调用各脚本
        │                     └── hooks/pre-receive.d/gitea
        │                           └── gitea hook --config=... pre-receive
        ▼
  cmd/hook.go::runHookPreReceive()
        │  从环境变量读取 pusher/repo 信息
        │  批量收集 ref 更新（每批 500 条）
        │  通过内部 HTTP POST 调用 Gitea 主进程
        ▼
  POST /api/internal/hook/pre-receive/{owner}/{repo}
        │
        ▼
  routers/private/hook_pre_receive.go::HookPreReceive()
        │  权限校验、签名检查、强推检查等
        ▼
  返回结果 → hook 子命令 → git-receive-pack → 客户端
```

关键设计点：
- Hook 子进程通过 **环境变量** 获取 pusher 身份信息（由 `gitea serv` 或内部推送环境注入）
- Hook 子进程通过 **内部 HTTP API** 与 Gitea 主进程通信（`setting.LocalURL` + `api/internal/...`）
- 内部 API 使用 `newInternalRequestAPI` 发送请求，带有内部认证 token

---

## 二、Hook 安装机制

### 2.1 Hook 脚本生成

**代码位置**：`modules/gitrepo/hooks.go::getHookTemplates()`

Gitea 为每个仓库生成两层 hook 脚本：

| 层级 | 路径 | 作用 |
|------|------|------|
| 外层委托脚本 | `hooks/pre-receive` | 遍历 `hooks/pre-receive.d/*` 下所有可执行文件，依次执行并收集退出码 |
| Gitea 内部脚本 | `hooks/pre-receive.d/gitea` | 调用 `gitea hook --config=<path> pre-receive` |

四个 hook 均按此模式安装：`pre-receive`、`update`、`post-receive`、`proc-receive`。

外层脚本的设计允许用户在 `hooks/*.d/` 下放置自定义 hook 脚本，Gitea 的脚本只是其中之一。任一脚本非零退出则整个 hook 失败。

### 2.2 gitea hook 子命令注册

**代码位置**：`cmd/hook.go::newHookCommand()`

```
gitea hook
  ├── pre-receive   → runHookPreReceive
  ├── update        → runHookUpdate
  ├── post-receive  → runHookPostReceive
  └── proc-receive  → runHookProcReceive
```

---

## 三、各 Hook 阶段触发机制

### 3.1 pre-receive

**触发时机**：`git-receive-pack` 在接收完所有 ref 更新数据后、实际写入 ref 之前。

**代码位置**：`cmd/hook.go::runHookPreReceive()` → `routers/private/hook_pre_receive.go::HookPreReceive()`

**执行流程**：

1. **内部推送跳过**：检查 `GITEA_INTERNAL_PUSH` 环境变量，若为 true 则直接返回（Gitea 内部的推送操作如合并 PR 已在事务内完成，无需重复校验）
2. **SSH 环境检查**：
   ```go
   // cmd/hook.go:191-198
   if len(os.Getenv("SSH_ORIGINAL_COMMAND")) == 0 {
       if setting.OnlyAllowPushIfGiteaEnvironmentSet {
           return fail(ctx, `Rejecting changes as Gitea environment not set...`)
       }
       return nil  // 否则不校验，直接通过
   }
   ```
   > **修正**：默认行为是宽松的——当 `SSH_ORIGINAL_COMMAND` 为空且配置**未启用**时，**直接通过**而非拒绝。只有配置 `ONLY_ALLOW_PUSH_IF_GITEA_ENVIRONMENT_SET=true`（默认值）时才会拒绝。HTTP 推送通过手动设置 `SSH_ORIGINAL_COMMAND=receive-pack` 来规避此检查。
3. **环境变量读取**：从环境变量获取 pusher 身份信息：
   - `GITEA_PUSHER_ID` — 推送者用户 ID
   - `GITEA_REPO_USER_NAME` / `GITEA_REPO_NAME` — 仓库所有者/仓库名
   - `GITEA_REPO_IS_WIKI` — 是否 wiki
   - `GITEA_PR_ID` — 关联的 PR ID（UI 合并时设置）
   - `GITEA_DEPLOY_KEY_ID` — Deploy Key ID
   - `GITEA_ACTIONS_TASK_ID` — Actions 任务 ID
4. **批量收集 ref 更新**：从 stdin 逐行读取 `<old-oid> <new-oid> <ref>` 格式的数据，按每 500 条一批向内部 API 发起检查
5. **ref 过滤**：仅对分支（`refs/heads/`）和标签（`refs/tags/`）执行检查；若 Git 支持 `proc-receive`，则所有 ref 都进行检查
6. **调用内部 API**：`POST /api/internal/hook/pre-receive/{owner}/{repo}`

### 3.2 update

**触发时机**：`git-receive-pack` 对每一个被更新的 ref 调用一次。

**代码位置**：`cmd/hook.go::runHookUpdate()`

**执行逻辑**：当前 update hook 几乎为空操作，仅做兼容保留：
- 内部推送直接跳过
- 若 ref 是 `refs/pull/` 前缀则 `os.Exit(1)` 阻止直接写入 pull ref
- 其余情况直接返回 0

注释说明：避免在 update 中做重操作，因为它对每个 ref 调用一次，不像 pre-receive/post-receive 是批量处理。

### 3.3 post-receive

**触发时机**：所有 ref 更新成功写入后。

**代码位置**：`cmd/hook.go::runHookPostReceive()` → `routers/private/hook_post_receive.go::HookPostReceive()`

**执行流程**：

1. **无条件执行** `git update-server-info`（智能 HTTP 协议所需）
2. 内部推送跳过后续处理
3. 批量收集 ref 更新（与 pre-receive 类似）
4. 调用内部 API `POST /api/internal/hook/post-receive/{owner}/{repo}`

**post-receive 服务端处理**（`hook_post_receive.go::HookPostReceive()`）：
- 将分支变更同步到数据库（`SyncBranchesToDB`）
- 删除的分支标记为已删除（`MarkBranchAsDeleted`）
- 更新关联的 PR 引用（`UpdatePullsRefs`）
- 处理 PR 合并（`PushTriggerPRMergeToBase`）
- 处理 Push Options（如 `repo-private`、`repo-template`）
- 生成 PR 创建/访问提示链接

### 3.4 proc-receive

**触发时机**：Git ≥ 2.29 支持，当 `refs/for/` 前缀的 ref 推送时由 Git 路由到此 hook（AGit Flow）。

**代码位置**：`cmd/hook.go::runHookProcReceive()` → `routers/private/hook_proc_receive.go::HookProcReceive()` → `services/agit/agit.go::ProcReceive()`

**执行流程**：

1. 与 Git 进行 pkt-line 协议协商（版本号 + 特性）
2. 接收 ref 更新命令和 push options
3. 调用内部 API `POST /api/internal/hook/proc-receive/{owner}/{repo}`
4. 服务端将 `refs/for/<branch>` 重写为 `refs/pull/<id>/head`（创建或更新 PR）
5. 通过 pkt-line 协议返回结果（ok/ng + option 指令）

---

## 四、环境变量注入机制

### 4.1 SSH 推送（gitea serv）

**代码位置**：`cmd/serv.go::runServ()`

当用户通过 SSH 推送时，`gitea serv` 子命令：
1. 从 SSH key ID 解析用户身份
2. 调用内部 API `ServCommand` 进行初步权限校验
3. 启动 `git-receive-pack` 子进程，注入以下环境变量：

| 环境变量 | 值 | 来源 |
|---------|-----|------|
| `GITEA_REPO_IS_WIKI` | bool | ServCommand 结果 |
| `GITEA_REPO_NAME` | 仓库名 | ServCommand 结果 |
| `GITEA_REPO_USER_NAME` | 所有者名 | ServCommand 结果 |
| `GITEA_PUSHER_NAME` | 推送者用户名 | ServCommand 结果 |
| `GITEA_PUSHER_EMAIL` | 推送者邮箱 | ServCommand 结果 |
| `GITEA_PUSHER_ID` | 推送者 ID | ServCommand 结果 |
| `GITEA_REPO_ID` | 仓库 ID | ServCommand 结果 |
| `GITEA_PR_ID` | PR ID（初始为 0） | 硬编码 |
| `GITEA_DEPLOY_KEY_ID` | Deploy Key ID | ServCommand 结果 |
| `GITEA_KEY_ID` | 公钥 ID | ServCommand 结果 |
| `GITEA_ROOT_URL` | App URL | setting |
| `SSH_ORIGINAL_COMMAND` | 原始 SSH 命令 | SSH 服务设置 |

### 4.2 Gitea 内部推送

**代码位置**：`modules/repository/env.go`

Gitea 内部操作（如 UI 合并 PR）推送时使用 `InternalPushingEnvironment()` 或 `PushingEnvironment()`：

- `InternalPushingEnvironment`：在 `PushingEnvironment` 基础上额外设置 `GITEA_INTERNAL_PUSH=true`，使所有 hook 直接跳过
- `FullPushingEnvironment`：设置完整的 Git author/committer 信息 + `SSH_ORIGINAL_COMMAND=gitea-internal`

### 4.3 ServCommand 中的权限校验

**代码位置**：`routers/private/serv.go::ServCommand()`

`ServCommand` 是 SSH 推送的第一道关卡，在 hook 执行之前：

1. **解析身份**：根据 key ID 找到用户或 Deploy Key
2. **仓库状态检查**：仓库是否在创建中、是否损坏、是否是镜像（镜像只读）
3. **账号状态检查**：用户是否被禁用/禁止登录
4. **归档仓库检查**：归档仓库不允许推送
5. **权限校验**：
   - Deploy Key：检查 `deployKey.Mode >= requestedMode`
   - 普通用户：`access_model.GetDoerRepoPermission` → `perm.UnitAccessMode(unitType)`
   - **AGit 特殊处理**：若 Git 支持 `proc-receive` 且 verb 为 `receive-pack`，则将写权限降级为读权限（因为 `refs/for` 推送的真正写入由 `proc-receive` hook 控制）
6. **Push-to-create**：仓库不存在时，若配置允许则自动创建

### 4.4 HTTP receive-pack 推送（完整闭环）

**代码位置**：`routers/web/repo/githttp.go::ServiceReceivePack()` → `serviceRPC()` → `httpBase()`

与 SSH 推送通过 `gitea serv` 中转不同，HTTP 推送的权限校验和环境变量注入完全在 Gitea Web 进程内完成。

**路由**：
```
POST /{username}/{reponame}/git-receive-pack  →  repo.ServiceReceivePack
GET  /{username}/{reponame}/info/refs         →  repo.GetInfoRefs (带 ?service=git-receive-pack 查询参数)
```

> **修正**：`/info/refs` 使用 `GET` 方法（带 `?service=git-receive-pack` 查询参数），而非 `POST`。`git-receive-pack` 本身才是 `POST`。对应源码：`routers/web/githttp.go:16` → `m.Methods("GET,OPTIONS", "/info/refs", repo.GetInfoRefs)`。

**完整执行流程**：

```
  git push http://...
        │
        ▼
  ServiceReceivePack(ctx)
        │
        ▼
  serviceRPC(ctx, "receive-pack")
        │
        ├── httpBase(ctx, "git-receive-pack")
        │    │
        │    ├── 解析服务类型：git-receive-pack → ServiceTypeReceivePack, accessMode=Write
        │    ├── 身份认证：
        │    │   ├── 公开仓库 pull 操作免认证
        │    │   ├── 否则需 Basic Auth / OAuth / PAT
        │    │   ├── 2FA 用户不能用密码（需 PAT）
        │    │   └── 检查 repo-scoped token 权限
        │    │
        │    ├── 仓库加载与状态检查：镜像只读、归档不可推
        │    ├── AGit 权限降级（同 SSH）：ReceivePack + SupportProcReceive → accessMode 改为 Read
        │    ├── 权限校验：GetDoerRepoPermission → CanAccess(accessMode, unitType)
        │    ├── Push-to-create（仓库不存在时）
        │    │
        │    └── 环境变量注入：
        │         h.environ = repo_module.DoerPushingEnvironment(ctx.Doer, repo, isWiki)
        │             │
        │             ├── GITEA_ROOT_URL=<AppURL>
        │             ├── GITEA_REPO_NAME=<repo> + (".wiki" if wiki)
        │             ├── GITEA_REPO_USER_NAME=<owner>
        │             ├── GITEA_REPO_ID=<repoID>
        │             ├── GITEA_REPO_IS_WIKI=true/false
        │             ├── GITEA_PUSHER_NAME=<doer.Name>
        │             ├── GITEA_PUSHER_ID=<doer.ID>
        │             ├── GITEA_PUSHER_EMAIL=<doer.Email>  # 若 !KeepEmailPrivate
        │             └── GITEA_ACTIONS_TASK_ID=<taskID>    # 若为 Actions 用户
        │
        ├── 额外环境变量追加：
        │    h.environ = append(h.environ, "SSH_ORIGINAL_COMMAND=receive-pack")  # 用于 hook 层判断非匿名
        │    h.environ = append(h.environ, "GIT_PROTOCOL=<version>")            # 若请求头有 Git-Protocol
        │
        ├── 构建 git 命令（serviceRPC 内）：
        │    // 先校验 Content-Type 必须为 application/x-git-receive-pack-request
        │    // 不匹配则返回 400 Bad Request
        │    // 支持 gzip 压缩的请求体（Content-Encoding: gzip）
        │    // 运行命令：
        │    git receive-pack --stateless-rpc .
        │        ├── WithEnv(append(os.Environ(), h.environ...))
        │        ├── WithStdinCopy(reqBody)           # 客户端推送的 pack 数据（可能解压）
        │        └── WithStdoutCopy(ctx.Resp)          # 结果直接写回 HTTP 响应
        │
        └── 运行命令：
             gitrepo.RunCmdWithStderr(ctx, repo, cmd)

        // info/refs 流程类似，但不启动 hook，仅 advertise refs：
        //   git receive-pack --stateless-rpc --advertise-refs .
        //   结果包装为 application/x-git-receive-pack-advertisement
        │
        ▼
  git-receive-pack 启动
        │
        ▼
  调用 hooks/pre-receive
        │
        ▼
  调用 gitea hook pre-receive （cmd/hook.go::runHookPreReceive）
        │  从环境变量读取上述注入的所有变量
        │  同 SSH 路径一致：批量收集 → 内部 API → 权限校验 → 返回结果
        ▼
  后续流程与 SSH 完全一致
```

**HTTP vs SSH 环境变量对比**：

| 环境变量 | SSH 推送 | HTTP 推送 | 说明 |
|---------|---------|----------|------|
| `GITEA_PUSHER_ID` | ✓ | ✓ | 推送者 ID |
| `GITEA_PUSHER_NAME` | ✓ | ✓ | 推送者用户名 |
| `GITEA_PUSHER_EMAIL` | ✓ | ✓ | 推送者邮箱（非私密时） |
| `GITEA_REPO_USER_NAME` | ✓ | ✓ | 仓库所有者 |
| `GITEA_REPO_NAME` | ✓ | ✓ | 仓库名 |
| `GITEA_REPO_IS_WIKI` | ✓ | ✓ | 是否 wiki |
| `GITEA_REPO_ID` | ✓ | ✓ | 仓库 ID |
| `GITEA_ROOT_URL` | ✓ | ✓ | App URL |
| `GITEA_ACTIONS_TASK_ID` | ✓ | ✓ | Actions 用户任务 ID |
| `GITEA_PR_ID` | ✓ (初始=0) | ✗ | PR ID（HTTP 场景不需要，内部推送通过 FullPushingEnvironment 设置） |
| `GITEA_DEPLOY_KEY_ID` | ✓ | ✗ | Deploy Key ID（HTTP 不使用 Deploy Key） |
| `GITEA_KEY_ID` | ✓ | ✗ | SSH 公钥 ID |
| `SSH_ORIGINAL_COMMAND` | 完整命令（如 `git-receive-pack...`） | `"receive-pack"` | HTTP 场景仅设置为服务名，用于 hook 层跳过匿名检查 |
| `GIT_PROTOCOL` | ✗ | ✓（若有请求头） | Git 协议版本（通过 Git-Protocol 请求头传递） |

**info/refs 请求的环境变量注入**：

`GetInfoRefs` 同样会调用 `httpBase` 并注入环境变量（`routers/web/repo/githttp.go:455-483`），用于在 `--advertise-refs` 阶段正确设置 hook 环境。但该阶段不启动 hook，仅用于获取引用列表。

**关键设计点**：
1. HTTP 场景复用 `SSH_ORIGINAL_COMMAND` 环境变量来标记"非匿名推送"，但值仅为服务名（`"receive-pack"`）而非完整命令
2. `runHookPreReceive` 中的检查逻辑 `len(os.Getenv("SSH_ORIGINAL_COMMAND")) == 0` 对 HTTP 推送同样生效
3. HTTP 推送不经过 `gitea serv`，直接由 Web 进程启动 `git-receive-pack`，权限校验在 `httpBase` 中完成
4. HTTP 推送的环境变量注入入口是 `repo_module.DoerPushingEnvironment`（返回基础环境），再在 `serviceRPC` 中追加 `SSH_ORIGINAL_COMMAND` 和 `GIT_PROTOCOL`。SSH 推送是在 `runServ` 中手动拼接完整环境。
5. `DoerPushingEnvironment` 不含 `SSH_ORIGINAL_COMMAND`，`FullPushingEnvironment` 会设置 `SSH_ORIGINAL_COMMAND=gitea-internal`（内部推送场景）

---

## 五、Hook 进程内权限校验实现

### 5.1 preReceiveContext

**代码位置**：`routers/private/hook_pre_receive.go::preReceiveContext`

`preReceiveContext` 封装了推送者的身份和权限信息，按需加载（lazy load）：

```go
type preReceiveContext struct {
    user                *user_model.User
    userPerm            access_model.Permission
    deployKeyAccessMode perm_model.AccessMode
    canWriteCode        bool
    canCreatePullRequest bool
}
```

### 5.2 loadPusherAndPermission

**代码位置**：`routers/private/hook_pre_receive.go::loadPusherAndPermission()`

根据 `opts.UserID` 加载推送者身份和权限：

- **Actions 用户**（`user_model.ActionsUserID`）：创建 Actions 虚拟用户，通过 `access_model.GetActionsUserRepoPermission` 获取权限
- **普通用户**：通过 `user_model.GetUserByID` + `access_model.GetDoerRepoPermission` 获取权限
- **Deploy Key**：额外加载 `deployKey.Mode` 作为权限补充

### 5.3 CanWriteCode — 代码写入权限

**代码位置**：`routers/private/hook_pre_receive.go::CanWriteCode()`

```
canWriteCode = CanMaintainerWriteToBranch(userPerm, branchName, user) || deployKeyAccessMode >= Write
```

**函数名误导性说明**：`CanMaintainerWriteToBranch` 名称具有误导性，代码注释（`services/context/repo.go:152`）明确指出：
> "CanMaintainerWriteToBranch is a bad name, but it really does what 'CanWriteToBranch' does"

实际上该函数的作用是**判断用户在任何前提下是否对分支有写入权限**，而不仅限于 "maintainer" 场景。它在多处被用作通用的分支可写性判断（如 `Repository.CanWriteToBranch` 直接调用它）。

`CanMaintainerWriteToBranch`（`models/issues/pull_list.go:73`）的真实判断逻辑：

```
1. 快速路径：headPerm.CanWrite(unit.TypeCode) == true
   │
   ├── 直接返回 true（99% 场景在此返回）
   │
   └── false → 进入 maintainer 编辑模式检查

2. Maintainer 编辑模式检查（仅当用户无直接 Write 权限时触发）：
   │
   ├── 从 headPerm 中提取仓库 ID（GetFirstUnitRepoID）
   ├── 查询该仓库中 headBranch 作为源分支的所有未合并 PR
   │   过滤条件：has_merged=false, is_closed=false, **flow=PullRequestFlowGithub**
   │   （即**只匹配 GitHub Flow 类型 PR，排除 AGit Flow PR**）
   │
   └── 遍历 PR，寻找满足以下所有条件的 PR：
       │
       ├── PR.AllowMaintainerEdit == true
       ├── PR.Issue.Poster 对 HeadRepo 有 Code Write 权限（防止 Poster 是 Reader 的情况）
       └── 当前 doer 对 BaseRepo 有 Code Write 权限
       │
       └── 找到任一满足的 PR → 返回 true

3. 以上均不满足 → 返回 false
```

> **修正**：`GetUnmergedPullRequestsByHeadInfo` 的 SQL 查询带有 `flow = PullRequestFlowGithub` 过滤条件（`models/issues/pull_list.go:142`），因此 AGit Flow 的 PR 不会触发 Maintainer 编辑路径。

**Hook 场景下的上下文**：
- 在 `pre-receive` hook 中，`headPerm` 是 doer 对**目标仓库**（被 push 的仓库）的权限，不是 PR HeadRepo 的权限
- 因此 "maintainer 编辑" 路径只有在以下极特殊场景才会触发：
  - doer 对目标仓库只有 Read 权限
  - 但存在一个从 fork 仓库的 `branchName` 指向目标仓库的未合并 PR（GitHub Flow 类型）
  - PR 开启了 `AllowMaintainerEdit`
  - PR 发布者对 fork 仓库（HeadRepo）有 Code Write 权限
  - doer 对目标仓库（BaseRepo）有 Code Write 权限

> **补充**：在 `services/context/repo.go::CanWriteToBranch` 场景中，`headPerm` 同样是 doer 对当前仓库（HeadRepo）的权限，此时 Maintainer 路径才更有意义——用于判断上游仓库的维护者是否有权限修改当前 fork 仓库的源分支。

**完整调用链**：
- `routers/private/hook_pre_receive.go:62` — hook 中的分支写权限判断
- `services/context/repo.go:107` — 通用 `CanWriteToBranch`
- `services/context/repo.go:153` — web 编辑器分支可写性判断
- `services/context/repo.go:1110` — PR 上下文分支可写性
- `routers/web/repo/editor.go:144` — 在线编辑分支权限
- `routers/web/repo/pull.go:925` — PR 页面分支可写性
- `services/convert/convert.go:79` — API 响应分支可写性

### 5.4 按 ref 类型的权限分发

**代码位置**：`routers/private/hook_pre_receive.go::HookPreReceive()`

```go
switch {
case refFullName.IsBranch():
    preReceiveBranch(...)
case refFullName.IsTag():
    preReceiveTag(...)
case SupportProcReceive && refFullName.IsFor():
    preReceiveFor(...)
default:
    AssertCanWriteCode()
}
```

---

## 六、分支保护与强推检查

### 6.1 preReceiveBranch 完整检查流程

**代码位置**：`routers/private/hook_pre_receive.go::preReceiveBranch()`

对受保护分支的检查按严格到宽松的顺序进行：

```
1. CanWriteCode 权限检查
        │
2. 默认分支不可删除
        │
3. 获取保护规则（GetFirstMatchProtectedBranchRule）
        │ 无保护规则 → 直接通过
        ▼ 有保护规则
4. 禁止删除受保护分支
        │
5. 强推检测（git rev-list --max-count=1 old ^new）
        │ 非强推 → 跳到步骤 6
        │ 是强推：
        ├── CanForcePush == false → 拒绝
        └── CanForcePush == true → isForcePush = true, 继续
        │
6. 签名验证（RequireSignedCommits）
        │
7. 受保护文件检查（GetProtectedFilePatterns）
        │
8. 推送者白名单检查（CanUserPush / CanUserForcePush）
        │ canPush == true → 通过
        │ canPush == false:
        ├── PR 合并（opts.PullRequestID != 0）→ 检查合并权限
        └── 非合并推送:
            ├── 修改了受保护文件 → 拒绝
            ├── 仅修改非保护文件 → 通过
            └── 普通推送 → 拒绝
```

### 6.2 强推检测算法

**代码位置**：`routers/private/hook_pre_receive.go:197-221`

```bash
git rev-list --max-count=1 <oldCommitID> ^<newCommitID>
```

该命令检查是否存在从 `oldCommitID` 可达但不可从 `newCommitID` 可达的提交。若输出非空，说明存在被丢弃的提交，即为强推。

**关键点**：
- 仅在 `oldCommitID != EmptyObjectID`（即非新建分支）时检查
- 该命令在 quarantine 环境下运行，通过 `ctx.env` 传入 `GIT_OBJECT_DIRECTORY`、`GIT_QUARANTINE_PATH` 等
- 检测到强推后，若 `protectBranch.CanForcePush` 为 false 则直接拒绝；为 true 则标记 `isForcePush = true` 继续后续检查

### 6.3 CanUserPush / CanUserForcePush

**代码位置**：`models/git/protected_branch.go`

`CanUserPush` 逻辑：
```
CanPush == false                    → 拒绝
EnableWhitelist == false            → 检查用户对仓库 Code 单元是否有 Write 权限
EnableWhitelist == true             → 检查用户/团队是否在白名单中
```

`CanUserForcePush` 逻辑：
```
CanForcePush == false               → 拒绝
EnableForcePushAllowlist == false   → 复用 CanUserPush 的结果（所有能 push 的用户都能 force push）
EnableForcePushAllowlist == true    → 检查用户/团队是否在强推白名单中 且 CanUserPush == true
```

### 6.4 Deploy Key 的推送权限

**代码位置**：`routers/private/hook_pre_receive.go:268-274`

Deploy Key 的权限判断与普通用户不同：

```go
if ctx.opts.DeployKeyID != 0 {
    if isForcePush {
        canPush = !changedProtectedfiles &&
                  protectBranch.CanPush &&
                  (!protectBranch.EnableForcePushAllowlist || protectBranch.ForcePushAllowlistDeployKeys)
    } else {
        canPush = !changedProtectedfiles &&
                  protectBranch.CanPush &&
                  (!protectBranch.EnableWhitelist || protectBranch.WhitelistDeployKeys)
    }
}
```

Deploy Key 的推送权限不依赖白名单中的用户/团队 ID，而是通过 `WhitelistDeployKeys` 和 `ForcePushAllowlistDeployKeys` 两个布尔开关控制。

### 6.5 AGit Flow 中的强推检查

**代码位置**：`services/agit/agit.go:227-243`

AGit proc-receive 中的强推检查逻辑：

```go
if !forcePush.Value() {
    output, _, err := gitrepo.RunCmdString(ctx, repo,
        gitcmd.NewCommand("rev-list", "--max-count=1").
            AddDynamicArguments(oldCommitID, "^"+opts.NewCommitIDs[i]),
    )
    if len(output) > 0 {
        results = append(results, ...{Err: "request `force-push` push option"})
        continue
    }
}
```

与 pre-receive 中的算法相同，但额外要求通过 push option 声明 `force-push`。

---

## 七、签名验证（Require Signed Commits）

### 7.1 触发条件

**代码位置**：`routers/private/hook_pre_receive.go:224-241`

仅当受保护分支规则 `RequireSignedCommits == true` 时触发。

### 7.2 验证流程

**代码位置**：`routers/private/hook_verification.go`

```
verifyCommits(oldCommitID, newCommitID, gitRepo, env)
    │
    ├── 新建分支（oldCommitID == EmptyObjectID）:
    │   git rev-list <newCommitID> --not --all
    │   列出从 newCommitID 可达、但**无法从仓库中任何已存在 ref（分支/标签）** 可达的提交
    │   （即排除所有已存在 ref 指向的提交及其所有祖先）
    │   确保只验证真正通过此次推送新接收的提交
    │
    └── 已有分支:
        git rev-list <old>...<new>
        列出 old 与 new 的对称差集
    │
    ▼ 逐个提交验证
readAndVerifyCommit(sha, repo, env)
    │
    git cat-file commit <sha>
    │
    解析 commit 对象 → git.CommitFromReader
    │
    ParseCommitWithSignature(commit)
    │
    ├── 无签名 → Verified = false → 拒绝
    ├── GPG 签名 → parseCommitWithGPGSignature
    │   提取签名 → 查找 key ID → 验证签名与 committer 是否匹配
    └── SSH 签名 → parseCommitWithSSHSignature
        验证 SSH 签名与 committer 是否匹配
    │
    verification.Verified == false → 返回 errUnverifiedCommit{sha}
```

#### `old...new` 三点范围的语义澄清

**代码位置**：`routers/private/hook_verification.go:27`

`git rev-list <old>...<new>` 使用**三点范围**（三重点号 `...`），其语义是**对称差集**（symmetric difference）：

```
     old                         new
      o                           o
     / \                         / \
    o   o                       o   o
     \ /                         \ /
      o                           o
       \                         /
        o---------o---------o---o
        A         B         C   D

old...new = {A} ∪ {D} （同时包含两侧独有的提交）
old..new  = {D}        （仅 new 侧独有的提交）
```

具体来说：
- **提交可达性**：列出所有可从 `old` 或 `new` 到达，但**不能同时**从两者到达的提交
- **等价命令**：`git rev-list old new --not $(git merge-base old new)`

**在不同推送场景下的实际范围**：

| 推送场景 | old...new 实际包含 | 语义合理性 |
|---------|-------------------|-----------|
| **Fast-forward 推送**（普通推送，old 是 new 的祖先） | 仅 new 侧的新提交（等同于 `old..new`） | ✅ 合理 |
| **Force push**（非 Fast-forward，old 不是 new 的祖先） | old 侧被丢弃的提交 + new 侧新引入的提交 | ⚠️ 过于宽泛 |
| **新建分支**（old = 0000000） | 使用 `new --not --all`（完全独立逻辑） | ✅ 合理 |

**与强推检测的顺序依赖**：

在 `preReceiveBranch` 中，强推检测（步骤 5）在签名验证（步骤 6）**之前**执行：
```go
// hook_pre_receive.go:197-221
if len(output) > 0 {  // 检测到强推
    if protectBranch.CanForcePush {
        isForcePush = true  // 标记，继续后续检查
    } else {
        return error  // 拒绝，不执行签名验证
    }
}
// hook_verification.go:29 注释："This is safe as force pushes are already forbidden"
```

**注释的局限性与潜在问题**：`hook_verification.go:29` 的注释 "This is safe as force pushes are already forbidden" 只说对了一半：

| 场景 | CanForcePush | 强推检测结果 | 是否执行签名验证 | old...new 范围 | 实际行为 |
|------|-------------|-------------|-----------------|---------------|---------|
| Fast-forward 推送 | 任意 | false | 是 | 仅 new 侧新提交 | ✅ 合理 |
| Force push | false | true | **否**（强推检测直接拒绝） | N/A | ✅ 注释成立 |
| Force push | true | true | **是** | old 侧被丢弃提交 + new 侧新提交 | ⚠️ 注释不成立，存在问题 |

**`CanForcePush == true` 时的问题**：

当强推被允许时，`old...new` 包含即将被丢弃的 old 侧提交。对这些提交做签名验证存在两个不合理之处：
1. **范围过大**：old 侧的提交是即将被丢弃的历史，不应该要求它们都有签名（可能是多年前的历史提交）
2. **校验对象错误**：`verifyCommits` 通过 `git cat-file commit <sha>` 读取的是**仓库中已存在**的 commit 对象（强推的 old 侧提交必然已存在于仓库中），而非 quarantine 中即将写入的新对象
3. **不一致性**：`newCommitID --not --all` 逻辑只校验新对象，而 `old...new` 在校验已存在对象

> **修正**：合理的实现应该是在强推场景下改用 `new --not --all`（仅校验新引入的提交），或使用 `old..new`（两点范围，仅 new 侧独有的提交）。当前实现会在允许强推时对将被丢弃的历史提交也做签名检查，可能造成不必要的拒绝。

**代码位置**：`services/asymkey/commit.go::ParseCommitWithSignature()`

签名验证的核心逻辑：
1. 根据 committer 邮箱查找 Gitea 用户
2. 若 commit 无签名 → `Verified: false, Reason: "not_signed_commit"`
3. 若签名为 SSH 格式 → `parseCommitWithSSHSignature`
4. 若签名为 GPG 格式 → `parseCommitWithGPGSignature`
5. 验证结果中 `Verified == true` 才通过

### 7.3 验证失败处理

在 `preReceiveBranch` 中，签名验证失败后：
- 若是 `errUnverifiedCommit` 类型 → 返回 403，提示具体未验证的 commit SHA
- 若是其他错误（如 git 命令失败）→ 返回 500

---

## 八、Tag 保护检查

**代码位置**：`routers/private/hook_pre_receive.go::preReceiveTag()`

```
1. CanWriteCode 权限检查
2. 获取仓库所有保护标签规则（GetProtectedTags，仅加载一次）
3. IsUserAllowedToControlTag(tags, tagName, userID)
    │
    遍历所有保护标签规则:
    ├── 规则不匹配 tagName → 跳过
    └── 规则匹配:
        └── IsUserAllowedModifyTag(tag, userID)
            ├── 无白名单 → 仓库 Write 权限
            └── 有白名单 → 检查用户/团队是否在白名单中
    │
    无匹配规则 → isAllowed = true（默认允许）
    首个匹配规则允许 → isAllowed = true（短路）
    匹配规则全部拒绝 → isAllowed = false
```

---

## 九、AGit Flow（refs/for）检查

**代码位置**：`routers/private/hook_pre_receive.go::preReceiveFor()`

```
1. CanCreatePullRequest 权限检查（仅需 PR 单元读权限）
2. 空仓库不允许创建 PR
3. Wiki 不支持 PR
4. 验证目标分支是否存在（GetAgitBranchInfo）
```

AGit Flow 的权限特点是：在 `ServCommand` 阶段，`git-receive-pack` 的权限被降级为 Read，因此 Reader 也可以推送 `refs/for/`。真正的写入由 `proc-receive` hook 处理，它将推送重写为 `refs/pull/<id>/head`（PR 引用），而非直接写入分支。

---

## 十、内部推送的豁免机制

Gitea 内部操作（UI 合并 PR、同步分支等）通过 `GITEA_INTERNAL_PUSH=true` 环境变量标记：

- **pre-receive**：检查到该变量后直接返回 0，跳过所有校验
- **update**：同上
- **post-receive**：执行 `git update-server-info` 后跳过后续处理

内部推送环境的构建（`modules/repository/env.go`）：
- `InternalPushingEnvironment()`：在完整推送环境上添加 `GITEA_INTERNAL_PUSH=true`
- `FullPushingEnvironment()`：设置 author/committer 信息 + `SSH_ORIGINAL_COMMAND=gitea-internal`

**设计意图**：内部推送已在业务逻辑层做过权限校验（如合并 PR 前检查合并权限），hook 层无需重复校验，避免循环检查和事务一致性问题。

---

## 十一、关键源码索引

| 功能 | 文件 | 函数/行 |
|------|------|---------|
| Hook 子命令入口 | `cmd/hook.go` | `runHookPreReceive:184`, `runHookUpdate:314`, `runHookPostReceive:331`, `runHookProcReceive:518` |
| 内部 API 客户端 | `modules/private/hook.go` | `HookPreReceive:96`, `HookPostReceive:103`, `HookProcReceive:109` |
| 内部 API 路由 | `routers/private/internal.go` | L67-69 |
| pre-receive 权限校验 | `routers/private/hook_pre_receive.go` | `HookPreReceive:109`, `preReceiveBranch:142`, `preReceiveTag:406`, `preReceiveFor:442`, `loadPusherAndPermission:493` |
| 签名验证 | `routers/private/hook_verification.go` | `verifyCommits:18`, `readAndVerifyCommit:58` |
| 签名解析 | `services/asymkey/commit.go` | `ParseCommitWithSignature:26` |
| post-receive 处理 | `routers/private/hook_post_receive.go` | `HookPostReceive:33` |
| proc-receive / AGit | `services/agit/agit.go` | `ProcReceive:65` |
| SSH Serv 命令 | `cmd/serv.go` | `runServ:135` |
| SSH ServCommand 权限 | `routers/private/serv.go` | `ServCommand:79` |
| **HTTP receive-pack** | `routers/web/repo/githttp.go` | `ServiceReceivePack:437`, `serviceRPC:369`, `httpBase:60`, `GetInfoRefs:454` |
| **HTTP 路由注册** | `routers/web/githttp.go` | `addOwnerRepoGitHTTPRouters:11` |
| 受保护分支模型 | `models/git/protected_branch.go` | `CanUserPush:125`, `CanUserForcePush:162`, `CanBypassBranchProtection:212` |
| 受保护标签模型 | `models/git/protected_tag.go` | `IsUserAllowedToControlTag:127` |
| Maintainer 写入权限 | `models/issues/pull_list.go` | `CanMaintainerWriteToBranch:73`, `canMaintainerWriteToBranch:82` |
| Maintainer 权限调用链 | `services/context/repo.go` | `CanWriteToBranch:106`, L153, L1110 |
| Doer 推送环境 | `modules/repository/env.go` | `DoerPushingEnvironment:58` |
| Hook 脚本生成 | `modules/gitrepo/hooks.go` | `getHookTemplates:17`, `CreateDelegateHooks:110` |
| 环境变量常量 | `modules/repository/env.go` | L19-33 |
| 推送环境构建 | `modules/repository/env.go` | `InternalPushingEnvironment:47`, `FullPushingEnvironment:78` |
