# Gitea Organization Team 与仓库授权机制深度分析

## 一、核心数据模型与数据库约束

### 1.1 数据库表约束概览

| 表名 | 联合唯一约束 | 索引字段 | 位置 |
|------|-------------|---------|------|
| `org_user` | `(uid, org_id)` | `uid`, `org_id`, `is_public` | `models/organization/org_user.go:27-30` |
| `team` | - | `org_id` | `models/organization/team.go:76` |
| `team_user` | `(team_id, uid)` | `org_id` | `models/organization/team_user.go:18-20` |
| `team_repo` | `(team_id, repo_id)` | `org_id` | `models/organization/team_repo.go:19-21` |
| `team_unit` | `(team_id, type)` | `org_id` | `models/organization/team_unit.go:17-19` |
| `access` | `(user_id, repo_id)` | - | `models/perm/access/access.go:27-28` |

### 1.2 OrgUser 组织成员表 (`models/organization/org_user.go`)

```go
type OrgUser struct {
    ID       int64
    UID      int64  `xorm:"INDEX UNIQUE(s)"`   // 联合唯一约束 1/2
    OrgID    int64  `xorm:"INDEX UNIQUE(s)"`   // 联合唯一约束 2/2
    IsPublic bool   `xorm:"INDEX"`
}
```

**约束说明**：
- `UNIQUE(s)` 标记表示联合唯一约束组 `s`
- 同一个用户在同一个组织中只能有一条成员记录
- `IsPublic` 索引用于快速查询公开成员

### 1.3 Team 团队表 (`models/organization/team.go`)

```go
type Team struct {
    ID                      int64 `xorm:"pk autoincr"`
    OrgID                   int64 `xorm:"INDEX"`
    LowerName               string
    Name                    string
    Description             string
    AccessMode              perm.AccessMode    `xorm:"'authorize'"`
    Members                 []*user_model.User  `xorm:"-"`
    NumRepos                int
    NumMembers              int
    Units                   []*TeamUnit         `xorm:"-"`
    IncludesAllRepositories bool                `xorm:"NOT NULL DEFAULT false"`
    CanCreateOrgRepo        bool                `xorm:"NOT NULL DEFAULT false"`
}
```

**约束说明**：
- `OrgID` 有索引，用于按组织快速查询团队
- 团队名称在组织内唯一（通过业务逻辑保证，非数据库级约束）

### 1.4 TeamUser 团队成员关联表 (`models/organization/team_user.go`)

```go
type TeamUser struct {
    ID     int64
    OrgID  int64 `xorm:"INDEX"`
    TeamID int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 1/2
    UID    int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 2/2
}
```

**约束说明**：
- `(TeamID, UID)` 联合唯一：一个用户在同一个团队中只能有一条记录
- `OrgID` 有索引，用于按组织快速查询团队成员

### 1.5 TeamRepo 团队仓库关联表 (`models/organization/team_repo.go`)

```go
type TeamRepo struct {
    ID     int64
    OrgID  int64 `xorm:"INDEX"`
    TeamID int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 1/2
    RepoID int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 2/2
}
```

**约束说明**：
- `(TeamID, RepoID)` 联合唯一：一个团队与一个仓库只能有一条关联记录
- 这保证了团队不会被重复添加到同一个仓库

### 1.6 TeamUnit 团队单元权限表 (`models/organization/team_unit.go`)

```go
type TeamUnit struct {
    ID         int64
    OrgID      int64      `xorm:"INDEX"`
    TeamID     int64      `xorm:"UNIQUE(s)"`   // 联合唯一约束 1/2
    Type       unit.Type  `xorm:"UNIQUE(s)"`   // 联合唯一约束 2/2
    AccessMode perm.AccessMode
}
```

**约束说明**：
- `(TeamID, Type)` 联合唯一：一个团队对同一个单元类型只能有一条权限记录
- `Type` 表示不同的功能单元：代码、Issue、PR、Wiki 等

### 1.7 Access 聚合权限表 (`models/perm/access/access.go`)

```go
type Access struct {
    ID     int64
    UserID int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 1/2
    RepoID int64 `xorm:"UNIQUE(s)"`   // 联合唯一约束 2/2
    Mode   perm.AccessMode
}
```

**约束说明**：
- `(UserID, RepoID)` 联合唯一：一个用户对一个仓库只能有一条最终权限记录
- 这是权限计算的结果表，由 `RecalculateTeamAccesses` 动态维护

---

## 二、权限位定义

### 2.1 AccessMode 枚举 (`models/perm/access_mode.go`)

```go
type AccessMode int

const (
    AccessModeNone  AccessMode = iota // 0: 无访问权限
    AccessModeRead                    // 1: 只读权限
    AccessModeWrite                   // 2: 可写权限
    AccessModeAdmin                   // 3: 管理员权限
    AccessModeOwner                   // 4: 所有者权限
)
```

**比较规则**：数值越大权限越高，使用 `>=` 进行比较。

---

## 三、Team 成员归属判定

### 3.1 成员归属层级结构

```
组织成员 (org_user 表)
    ↓
    团队成员 (team_user 表)
        ↓
        用户 (user 表)
```

### 3.2 核心查询函数

**1. 检查组织成员** (`models/organization/org_user.go:87-93`)

```go
func IsOrganizationMember(ctx context.Context, orgID, uid int64) (bool, error) {
    return db.GetEngine(ctx).
        Where("uid=?", uid).
        And("org_id=?", orgID).
        Table("org_user").
        Exist()
}
```

**2. 检查团队成员** (`models/organization/team_user.go:24-31`)

```go
func IsTeamMember(ctx context.Context, orgID, teamID, userID int64) (bool, error) {
    return db.GetEngine(ctx).
        Where("org_id=?", orgID).
        And("team_id=?", teamID).
        And("uid=?", userID).
        Table("team_user").
        Exist()
}
```

**3. 获取用户组织内所有团队** (`models/organization/team_list.go:109-116`)

```go
func GetUserOrgTeams(ctx context.Context, orgID, userID int64) (teams TeamList, err error) {
    return teams, db.GetEngine(ctx).
        Join("INNER", "team_user", "team_user.team_id = team.id").
        Where("team.org_id = ?", orgID).
        And("team_user.uid=?", userID).
        Find(&teams)
}
```

**4. 获取用户对某仓库有访问权的团队** (`models/organization/team_list.go:118-127`)

```go
func GetUserRepoTeams(ctx context.Context, orgID, userID, repoID int64) (teams TeamList, err error) {
    return teams, db.GetEngine(ctx).
        Join("INNER", "team_user", "team_user.team_id = team.id").
        Join("INNER", "team_repo", "team_repo.team_id = team.id").
        Where("team.org_id = ?", orgID).
        And("team_user.uid=?", userID).
        And("team_repo.repo_id=?", repoID).
        Find(&teams)
}
```

---

## 四、权限位继承机制

### 4.1 权限聚合策略

权限判定采用 **"取最大值"** 的继承策略：
- 用户可能属于多个团队
- 每个团队对仓库有不同权限
- 用户的最终权限 = `max(所有相关团队权限, 协作者权限)`

### 4.2 RecalculateTeamAccesses 重算权限

**函数位置**：`models/perm/access/access.go:189-229`

```go
func RecalculateTeamAccesses(ctx context.Context, repo *repo_model.Repository, ignTeamID int64) error {
    accessMap := make(map[int64]*userAccess, 20)

    // 1. 先加载协作者权限
    if err = refreshCollaboratorAccesses(ctx, repo.ID, accessMap); err != nil {
        return err
    }

    // 2. 获取组织所有团队
    teams, err := organization.FindOrgTeams(ctx, repo.Owner.ID)
    if err != nil {
        return err
    }

    // 3. 遍历团队，聚合权限
    for _, t := range teams {
        if t.ID == ignTeamID {
            continue
        }

        // Owners 团队自动获得 Owner 权限
        if t.IsOwnerTeam() {
            t.AccessMode = perm.AccessModeOwner
        } else if !organization.HasTeamRepo(ctx, t.OrgID, t.ID, repo.ID) {
            continue // 跳过没有仓库关联的团队
        }

        // 加载团队成员
        if err = t.LoadMembers(ctx); err != nil {
            return err
        }

        // 对每个成员，取最大权限
        for _, m := range t.Members {
            updateUserAccess(accessMap, m, t.AccessMode)
        }
    }

    // 4. 写入 access 表
    return refreshAccesses(ctx, repo, accessMap)
}
```

### 4.3 updateUserAccess 取最大权限

```go
func updateUserAccess(accessMap map[int64]*userAccess, user *user_model.User, mode perm.AccessMode) {
    if ua, ok := accessMap[user.ID]; ok {
        ua.Mode = maxAccessMode(ua.Mode, mode) // 关键：取最大值
    } else {
        accessMap[user.ID] = &userAccess{User: user, Mode: mode}
    }
}
```

### 4.4 触发权限重算的场景

| 操作 | 触发函数 |
|------|---------|
| 添加团队成员 | `RecalculateUserAccess` |
| 移除团队成员 | `RecalculateUserAccess` |
| 团队添加仓库 | `RecalculateTeamAccesses` |
| 团队移除仓库 | `RecalculateTeamAccesses` |
| 团队权限变更 | `RecalculateTeamAccesses` |
| 仓库转移 | `RecalculateAccesses` |

---

## 五、跨组织 Fork 完整流程分析

### 5.1 API 入口层 (`routers/api/v1/repo/fork.go`)

**CreateFork 函数调用链**：

```
CreateFork(ctx)
    │
    ├─→ 解析参数：form.Organization
    │
    ├─→ prepareDoerCreateRepoInOrg(ctx, orgName)
    │    ├─→ GetOrgByName(ctx, orgName)
    │    ├─→ HasOrgOrUserVisible(ctx, org, doer)  // 可见性检查
    │    └─→ org.CanCreateOrgRepo(ctx, doer.ID)   // 创建仓库权限检查
    │
    ├─→ ForkRepository(ctx, doer, forkOwner, opts)
    │    └─→ (进入服务层)
    │
    └─→ 返回结果
```

**CreateFork 核心代码** (`routers/api/v1/repo/fork.go:116-179`)：

```go
func CreateFork(ctx *context.APIContext) {
    form := web.GetForm(ctx).(*api.CreateForkOption)
    forkOwner := ctx.Doer
    
    // 如果指定了组织作为 fork 目标
    if form.Organization != nil {
        org := prepareDoerCreateRepoInOrg(ctx, *form.Organization)
        if ctx.Written() {
            return
        }
        forkOwner = org.AsUser()
    }

    repo := ctx.Repo.Repository
    name := optional.FromPtr(form.Name).ValueOrDefault(repo.Name)
    
    // 调用服务层创建 fork
    fork, err := repo_service.ForkRepository(ctx, ctx.Doer, forkOwner, repo_service.ForkRepoOptions{
        BaseRepo:    repo,
        Name:        name,
        Description: repo.Description,
    })
    // ... 错误处理
}
```

### 5.2 prepareDoerCreateRepoInOrg 前置检查

**位置**：`routers/api/v1/repo/fork.go:86-113`

```go
func prepareDoerCreateRepoInOrg(ctx *context.APIContext, orgName string) *organization.Organization {
    org, err := organization.GetOrgByName(ctx, orgName)
    
    // 检查 1：组织对用户可见
    if !organization.HasOrgOrUserVisible(ctx, org.AsUser(), ctx.Doer) {
        ctx.APIErrorNotFound()
        return nil
    }

    // 检查 2：非管理员需要有创建仓库权限
    if !ctx.Doer.IsAdmin {
        canCreate, err := org.CanCreateOrgRepo(ctx, ctx.Doer.ID)
        if !canCreate {
            ctx.APIError(http.StatusForbidden, "User is not allowed to create repositories in this organization.")
            return nil
        }
    }
    return org
}
```

### 5.3 Web 入口层 (`routers/web/repo/fork.go`)

Web 端有额外的循环检查逻辑：

```go
func ForkPost(ctx *context.Context) {
    // ...
    
    // 遍历所有父仓库，检查：
    // 1. CanUserForkBetweenOwners - 不能在同一所有者间循环 fork
    // 2. HasForkedRepo - 不能重复 fork
    traverseParentRepo := forkRepo
    for {
        if !repository.CanUserForkBetweenOwners(ctxUser.ID, traverseParentRepo.OwnerID) {
            ctx.JSONError(ctx.Tr("repo.settings.new_owner_has_same_repo"))
            return
        }
        repo := repo_model.GetForkedRepo(ctx, ctxUser.ID, traverseParentRepo.ID)
        if repo != nil {
            ctx.JSONRedirect(...)  // 已 fork 过，直接跳转
            return
        }
        if !traverseParentRepo.IsFork {
            break
        }
        traverseParentRepo, err = repo_model.GetRepositoryByID(ctx, traverseParentRepo.ForkID)
    }

    // 检查目标组织的创建权限
    if ctxUser.IsOrganization() {
        isAllowedToFork, err := organization.OrgFromUser(ctxUser).CanCreateOrgRepo(ctx, ctx.Doer.ID)
        if !isAllowedToFork {
            ctx.HTTPError(http.StatusForbidden)
            return
        }
    }

    // 调用服务层
    repo := ForkRepoTo(ctx, ctxUser, forkOpts)
}
```

### 5.4 CanUserForkBetweenOwners 规则

**位置**：`modules/repository/fork.go:18-23`

```go
func CanUserForkBetweenOwners(id1, id2 int64) bool {
    if id1 != id2 {
        return true  // 不同所有者可以 fork
    }
    // 同一所有者默认不能 fork，除非开启实验性功能
    return setting.Repository.AllowForkIntoSameOwner
}
```

### 5.5 服务层 ForkRepository (`services/repository/fork.go:58-210`)

```go
func ForkRepository(ctx context.Context, doer, owner *user_model.User, opts ForkRepoOptions) (*repo_model.Repository, error) {
    // 检查 1：用户未被源仓库所有者封禁
    if user_model.IsUserBlockedBy(ctx, doer, opts.BaseRepo.Owner.ID) {
        return nil, user_model.ErrBlockedUser
    }

    // 检查 2：用户未达到仓库创建上限
    if !doer.CanForkRepoIn(owner) {
        return nil, repo_model.ErrReachLimitOfRepo{...}
    }

    // 检查 3：未 fork 过
    forkedRepo, err := repo_model.GetUserFork(ctx, opts.BaseRepo.ID, owner.ID)
    if forkedRepo != nil {
        return nil, ErrForkAlreadyExist{...}
    }

    // 关键：私有属性继承
    repo := &repo_model.Repository{
        OwnerID:       owner.ID,
        Name:          opts.Name,
        // 继承规则：源仓库私有 OR 源所有者是私有组织
        IsPrivate:     opts.BaseRepo.IsPrivate || opts.BaseRepo.Owner.Visibility == structs.VisibleTypePrivate,
        IsFork:        true,
        ForkID:        opts.BaseRepo.ID,
        // ...
    }

    // 数据库事务中创建
    err = db.WithTx(ctx, func(ctx context.Context) error {
        if err = createRepositoryInDB(ctx, doer, owner, repo, true); err != nil {
            return err
        }
        if err = repo_model.IncrementRepoForkNum(ctx, opts.BaseRepo.ID); err != nil {
            return err
        }
        return git_model.CopyLFS(ctx, repo, opts.BaseRepo)
    })

    // ... Git 克隆、钩子创建、分支同步等
    return repo, nil
}
```

### 5.6 createRepositoryInDB 创建流程

**位置**：`services/repository/create.go:339-420`

```go
func createRepositoryInDB(ctx context.Context, doer, u *user_model.User, repo *repo_model.Repository, isFork bool) error {
    // 1. 名称检查
    if err = repo_model.IsUsableRepoName(repo.Name); err != nil {
        return err
    }
    has, err := repo_model.IsRepositoryModelExist(ctx, u, repo.Name)
    if has {
        return repo_model.ErrRepoAlreadyExist{...}
    }

    // 2. 插入仓库记录
    if err = db.Insert(ctx, repo); err != nil {
        return err
    }

    // 3. 删除可能存在的重定向
    if err = repo_model.DeleteRedirect(ctx, u.ID, repo.Name); err != nil {
        return err
    }

    // 4. 初始化仓库单元（Fork 使用 DefaultForkRepoUnits）
    defaultUnits := unit.DefaultRepoUnits
    switch {
    case isFork:
        defaultUnits = unit.DefaultForkRepoUnits  // Fork 有特殊的单元配置
    case repo.IsMirror:
        defaultUnits = unit.DefaultMirrorRepoUnits
    case repo.IsTemplate:
        defaultUnits = unit.DefaultTemplateRepoUnits
    }
    
    // 5. 创建各单元配置（Issues、PR、Projects 等）
    units := make([]repo_model.RepoUnit, 0, len(defaultUnits))
    for _, tp := range defaultUnits {
        // ... 创建各单元的默认配置
    }
    if err = db.Insert(ctx, units); err != nil {
        return err
    }

    // 6. 组织仓库：添加到 Owners 团队
    if u.IsOrganization() {
        t, err := organization.OrgFromUser(u).GetOwnerTeam(ctx)
        if err != nil {
            return err
        }
        t.NumRepos++
        if err = organization.AddTeamRepo(ctx, t.OrgID, t.ID, repo.ID); err != nil {
            return err
        }
        // ... 重算权限
    }

    // 7. Star 自己的仓库
    if !doer.IsGhost() {
        if err = repo_model.StarRepo(ctx, doer.ID, repo.ID, true); err != nil {
            return err
        }
    }

    return nil
}
```

---

## 六、API 与 Web 端前置限制一致性深度分析

### 6.1 完整调用链对比

#### 6.1.1 API 端完整调用链

**路由定义** (`routers/api/v1/api.go:1239-1240`)：
```go
m.Combo("/forks").Get(repo.ListForks).
    Post(reqToken(), reqRepoReader(unit.TypeCode), bind(api.CreateForkOption{}), repo.CreateFork)
```

**中间件前置检查**：
1. `reqToken()` - 验证 API token
2. `reqRepoReader(unit.TypeCode)` - 验证用户有源仓库的代码读权限
   ```go
   func reqRepoReader(unitType unit.Type) func(ctx *context.APIContext) {
       return func(ctx *context.APIContext) {
           if !ctx.Repo.Permission.CanRead(unitType) && !ctx.IsUserRepoAdmin() && !ctx.IsUserSiteAdmin() {
               ctx.APIError(http.StatusForbidden, "...")
               return
           }
       }
   }
   ```
3. `bind()` - 参数绑定验证

**`CreateFork` 函数内检查** (`routers/api/v1/repo/fork.go:149-177`)：
```go
func CreateFork(ctx *context.APIContext) {
    form := web.GetForm(ctx).(*api.CreateForkOption)
    forkOwner := ctx.Doer
    
    // 仅当指定组织作为目标时才检查
    if form.Organization != nil {
        org := prepareDoerCreateRepoInOrg(ctx, *form.Organization)
        if ctx.Written() {
            return
        }
        forkOwner = org.AsUser()
    }
    
    // 直接调用服务层，无其他检查！
    fork, err := repo_service.ForkRepository(ctx, ctx.Doer, forkOwner, ...)
}
```

**`prepareDoerCreateRepoInOrg` 检查** (`routers/api/v1/repo/fork.go:86-113`)：
```go
func prepareDoerCreateRepoInOrg(ctx *context.APIContext, orgName string) *organization.Organization {
    org, err := organization.GetOrgByName(ctx, orgName)
    
    // 检查 1：组织对用户可见
    if !organization.HasOrgOrUserVisible(ctx, org.AsUser(), ctx.Doer) {
        ctx.APIErrorNotFound()
        return nil
    }
    
    // 检查 2：非管理员需要有创建仓库权限
    if !ctx.Doer.IsAdmin {
        canCreate, err := org.CanCreateOrgRepo(ctx, ctx.Doer.ID)
        if !canCreate {
            ctx.APIError(http.StatusForbidden, "...")
            return nil
        }
    }
    return org
}
```

**关键发现**：当 fork 到个人空间时（`form.Organization == nil`），API 端完全跳过组织可见性和创建权限检查！

#### 6.1.2 Web 端完整调用链

**路由定义** (`routers/web/web.go:1450`)：
```go
m.Combo("/fork").Get(repo.Fork).Post(web.Bind(forms.CreateRepoForm{}), repo.ForkPost)
```

**全局中间件前置检查**：
1. `reqSignIn` - 必须登录
2. `context.RepoAssignment` - 仓库上下文赋值（含权限检查）
3. `reqUnitCodeReader` - 验证有代码读权限

**`Fork` GET 渲染检查** (`routers/web/repo/fork.go:44-147`)：
```go
func Fork(ctx *context.Context) {
    // 加载可 fork 的组织列表
    orgs, err := organization.GetOrgsCanCreateRepoByUserID(ctx, ctx.Doer.ID)
    if err != nil {
        ctx.ServerError("GetOrgsCanCreateRepoByUserID", err)
        return
    }
    
    // 1. 检查每个潜在目标的同一所有者限制
    canForkToUser := true
    traverseParentRepo := forkRepo
    for {
        if !repository.CanUserForkBetweenOwners(ctx.Doer.ID, traverseParentRepo.OwnerID) {
            canForkToUser = false
        }
        if !traverseParentRepo.IsFork {
            break
        }
        traverseParentRepo, _ = repo_model.GetRepositoryByID(ctx, traverseParentRepo.ForkID)
    }
    ctx.Data["CanForkToUser"] = canForkToUser
    
    // 2. 检查每个组织的同一所有者限制
    for i := range orgs {
        canFork := true
        traverseParentRepo = forkRepo
        for {
            if !repository.CanUserForkBetweenOwners(orgs[i].ID, traverseParentRepo.OwnerID) {
                canFork = false
                break
            }
            if !traverseParentRepo.IsFork {
                break
            }
            traverseParentRepo, _ = repo_model.GetRepositoryByID(ctx, traverseParentRepo.ForkID)
        }
        orgs[i].CanForkTo = canFork
    }
}
```

**`ForkPost` POST 提交检查** (`routers/web/repo/fork.go:150-202`)：
```go
func ForkPost(ctx *context.Context) {
    form := web.GetForm(ctx).(*forms.CreateRepoForm)
    forkRepo := ctx.Repo.Repository
    
    // 确定目标所有者
    ctxUser := checkContextUser(ctx, form.Uid)
    if ctx.Written() {
        return
    }
    
    // ===== 关键循环：遍历整个祖先链 =====
    traverseParentRepo := forkRepo
    for {
        // 检查 1：同一所有者限制（对每个祖先）
        if !repository.CanUserForkBetweenOwners(ctxUser.ID, traverseParentRepo.OwnerID) {
            ctx.JSONError(ctx.Tr("repo.settings.new_owner_has_same_repo"))
            return
        }
        // 检查 2：重复 fork（对每个祖先）
        repo := repo_model.GetForkedRepo(ctx, ctxUser.ID, traverseParentRepo.ID)
        if repo != nil {
            ctx.JSONRedirect(ctxUser.HomeLink() + "/" + url.PathEscape(repo.Name))
            return
        }
        // 继续向上遍历
        if !traverseParentRepo.IsFork {
            break
        }
        traverseParentRepo, err = repo_model.GetRepositoryByID(ctx, traverseParentRepo.ForkID)
        if err != nil {
            ctx.ServerError("GetRepositoryByID", err)
            return
        }
    }
    
    // 检查 3：目标组织的创建权限（在 checkContextUser 中）
    if ctxUser.IsOrganization() {
        isAllowedToFork, err := organization.OrgFromUser(ctxUser).CanCreateOrgRepo(ctx, ctx.Doer.ID)
        if !isAllowedToFork {
            ctx.HTTPError(http.StatusForbidden)
            return
        }
    }
    
    // 调用服务层
    repo := ForkRepoTo(ctx, ctxUser, ...)
}
```

**`checkContextUser` 检查** (`routers/web/repo/repo.go:82-130`)：
```go
func checkContextUser(ctx *context.Context, uid int64) *user_model.User {
    // 1. 加载用户可创建仓库的组织列表
    orgs, err := organization.GetOrgsCanCreateRepoByUserID(ctx, ctx.Doer.ID)
    // 2. 进一步过滤（CanCreateRepoIn）
    for i := range orgs {
        if ctx.Doer.CanCreateRepoIn(orgs[i].AsUser()) {
            orgsAvailable = append(orgsAvailable, orgs[i])
        }
    }
    
    // 3. 如果选择了组织
    if uid != ctx.Doer.ID && uid != 0 {
        org, err := user_model.GetUserByID(ctx, uid)
        // 4. 必须是组织
        if !org.IsOrganization() {
            ctx.HTTPError(http.StatusForbidden)
            return nil
        }
        // 5. 非管理员需要创建权限
        if !ctx.Doer.IsAdmin {
            canCreate, err := organization.OrgFromUser(org).CanCreateOrgRepo(ctx, ctx.Doer.ID)
            if !canCreate {
                ctx.HTTPError(http.StatusForbidden)
                return nil
            }
        }
    }
}
```

### 6.2 检查项详细对比表（经代码逐项校准）

> 以下每一行均通过源码核实。API 端的 `repoAssignment()` 中间件和 `reqToken()` 已计入对应列。

| 检查阶段 | 检查项 | Web 端 | API 端 (fork到个人) | API 端 (fork到组织) | 服务层 |
|---------|--------|--------|---------------------|---------------------|--------|
| **身份与源仓库** | 必须认证 | ✅ `reqSignIn` | ✅ `reqToken()` | ✅ `reqToken()` | ❌ |
| | 源仓库代码读权限 | ✅ `reqUnitCodeReader` | ✅ `reqRepoReader(TypeCode)` | ✅ `reqRepoReader(TypeCode)` | ❌ |
| | 源仓库上下文赋值 | ✅ `RepoAssignment` + `GetDoerRepoPermission` | ✅ `repoAssignment()` + `GetDoerRepoPermission` + `HasAnyUnitAccessOrPublicAccess` | ✅ 同左 | ❌ |
| **目标所有者** | 目标所有者存在 | ✅ `checkContextUser` → `GetUserByID` | ✅ 个人即 `ctx.Doer`，必然存在 | ✅ `GetOrgByName` | ❌ |
| | 目标对用户可见 | ✅ `checkContextUser` → `CanCreateOrgRepo` 隐含可见 | ✅ 个人空间即自己，天然可见 | ✅ `HasOrgOrUserVisible` | ❌ |
| | 目标组织创建权限 | ✅ `CanCreateOrgRepo` | N/A（个人空间无需） | ✅ `CanCreateOrgRepo` | ❌ |
| **业务规则** | 同一所有者限制 | ✅ 对每个祖先检查 `CanUserForkBetweenOwners` | ❌ **完全不检查** | ❌ **完全不检查** | ❌ |
| | 祖先链重复 fork | ✅ 遍历所有祖先 `GetForkedRepo` | ❌ **完全不检查** | ❌ **完全不检查** | ❌ |
| | 直接父仓库重复 fork | ✅ 祖先链遍历中覆盖 | ✅ 服务层 `GetUserFork` 兜底 | ✅ 服务层 `GetUserFork` 兜底 | ✅ `GetUserFork` |
| | 同一所有者同名仓库 | ✅ 服务层 `IsRepositoryModelExist` | ✅ 服务层 `IsRepositoryModelExist` | ✅ 服务层 `IsRepositoryModelExist` | ✅ |
| **安全边界** | 用户未被源所有者封禁 | ❌ | ❌ | ❌ | ✅ `IsUserBlockedBy` |
| | 未达创建上限 | ❌ | ❌ | ❌ | ✅ `CanForkRepoIn` |
| | 仓库名可用 | ✅ 服务层 `IsUsableRepoName` | ✅ 服务层 `IsUsableRepoName` | ✅ 服务层 `IsUsableRepoName` | ✅ |

**校准说明**：
- "必须登录"：API 端路由注册了 `reqToken()`（`api.go:1240`），该函数验证请求携带有效 token，等价于必须认证。之前版本标 ❌ 不准确，已修正为 ✅。
- "源仓库上下文赋值"：API 端 `repoAssignment()` 中间件（`api.go:134-220`）做了完整的 owner 查找、repo 查找、`GetDoerRepoPermission` 计算权限、`HasAnyUnitAccessOrPublicAccess` 检查访问性。之前版本对此未充分标注。
- "直接父仓库重复 fork"：服务层 `ForkRepository`（`fork.go:74-84`）调用 `GetUserFork(opts.BaseRepo.ID, owner.ID)` 检查 owner 是否已 fork 了直接父仓库。Web 和 API 端均经过此路径，因此 API 端对直接重复 fork 并非"完全不检查"，而是由服务层兜底。之前版本标 ❌ 不准确，已修正为 ✅。

### 6.3 同一所有者限制详细分析

**规则定义** (`modules/repository/fork.go:18-23`)：
```go
func CanUserForkBetweenOwners(id1, id2 int64) bool {
    if id1 != id2 {
        return true  // 不同所有者 → 允许
    }
    // 同一所有者 → 检查配置
    return setting.Repository.AllowForkIntoSameOwner
}
```

**Web 端执行位置**（3次检查）：
1. GET `/fork` 渲染时 - `getForkRepository` 中对当前用户检查（`fork.go:52`）
2. GET `/fork` 渲染时 - 遍历祖先链排除不可 fork 的组织（`fork.go:68-89`）
3. POST `/fork` 提交时 - `ForkPost` 中遍历祖先链再次验证（`fork.go:160-164`）

**API 端执行位置**：0 次。`CreateFork`（`api/v1/repo/fork.go:149-178`）中没有调用 `CanUserForkBetweenOwners`。

#### 6.3.1 同一所有者 + 默认仓库名：重名失败的天然屏障

当同一所有者 fork 到自己名下且使用**默认仓库名**时，存在一道隐式屏障：同名仓库检测。

**API 端默认名称逻辑** (`api/v1/repo/fork.go:160`)：
```go
name := optional.FromPtr(form.Name).ValueOrDefault(repo.Name)
```
当 `form.Name` 为 nil（未指定）时，`name` 默认等于源仓库名。

**服务层重名检测** (`services/repository/create.go:344-352`)：
```go
has, err := repo_model.IsRepositoryModelExist(ctx, u, repo.Name)
if has {
    return repo_model.ErrRepoAlreadyExist{Uname: u.Name, Name: repo.Name}
}
```

**同一所有者 + 默认名称的执行路径**：
```
源仓库：org1/repo
目标：org1（同一所有者），名称：repo（默认）

服务层 ForkRepository
    → GetUserFork(org1, repoID) → 已存在（就是自己）
    → 返回 ErrForkAlreadyExist ← 第一道拦截
```

**因此，同一所有者场景下的风险路径实际是：**

| 场景 | 默认名称 | 自定义名称（`form.Name = "repo-copy"`） |
|------|---------|----------------------------------------|
| 同一所有者 fork 到自己 | ❌ `GetUserFork` 返回已有 fork → `ErrForkAlreadyExist` | ⚠️ `GetUserFork` 检查通过（新名称）→ `IsRepositoryModelExist` 通过（不同名）→ **创建成功** |
| 同一所有者（不同仓库）| N/A | ⚠️ 如果祖先链中有同一所有者的仓库，`GetUserFork` 只检查直接父仓库，可能漏检 |

**关键结论**：API 端缺少 `CanUserForkBetweenOwners` 检查，但同一所有者 + 默认名称时会被 `GetUserFork` 拦截（返回 `ErrForkAlreadyExist`）。**只有使用自定义名称时**，才能绕过此屏障，在同一所有者下创建出多余的 fork。而 Web 端的 `CanUserForkBetweenOwners` 检查不依赖仓库名，无论默认还是自定义名称都会拦截。

#### 6.3.2 风险场景修正

**风险场景 1：fork 到个人空间（使用自定义名称）**
```
源仓库：userA/repo (个人仓库)
API 请求：POST /repos/userA/repo/forks, body: {"name": "repo-copy"}
```
- Web 端：`CanUserForkBetweenOwners(userA, userA) = false` → 被阻止
- API 端：无 `CanUserForkBetweenOwners` 检查 → `GetUserFork` 检查直接父仓库 → 通过（新名称） → **允许创建**
- 如果使用默认名称：`GetUserFork` 发现已 fork → `ErrForkAlreadyExist` → 被拦截

**风险场景 2：fork 到同一组织（使用自定义名称）**
```
源仓库：org1/repo
目标组织：org1
API 请求：POST /repos/org1/repo/forks, body: {"organization": "org1", "name": "repo-copy"}
```
- Web 端：`CanUserForkBetweenOwners(org1, org1) = false` → 被阻止
- API 端：无检查 → `GetUserFork(baseRepoID, org1)` → 已存在 → `ErrForkAlreadyExist` → 被拦截
- 但如果之前已删除了旧的 fork：`GetUserFork` 检查通过 → **允许创建**

### 6.4 祖先链重复 fork 检查详细分析

**Web 端祖先链遍历** (`routers/web/repo/fork.go:159-178`)：
```go
traverseParentRepo := forkRepo  // 从当前仓库开始
for {
    // 检查：目标所有者是否已 fork 了这个祖先
    repo := repo_model.GetForkedRepo(ctx, ctxUser.ID, traverseParentRepo.ID)
    if repo != nil {
        ctx.JSONRedirect(...)  // 已存在，跳转
        return
    }
    // 向上遍历父仓库
    if !traverseParentRepo.IsFork {
        break  // 到达原始仓库，结束
    }
    traverseParentRepo, _ = repo_model.GetRepositoryByID(ctx, traverseParentRepo.ForkID)
}
```

**GetForkedRepo 查询** (`models/repo/repo.go:713-718`)：
```go
func GetForkedRepo(ctx context.Context, uid, repoID int64) *Repository {
    var repo Repository
    has, _ := db.GetEngine(ctx).
        Where("owner_id=? AND fork_id=?", uid, repoID).
        Get(&repo)
    if has {
        return &repo
    }
    return nil
}
```

**服务层检查** (`services/repository/fork.go:74-84`)：
```go
// 只检查直接父仓库，不遍历祖先链
forkedRepo, err := repo_model.GetUserFork(ctx, opts.BaseRepo.ID, owner.ID)
if forkedRepo != nil {
    return nil, ErrForkAlreadyExist{
        Uid:        owner.ID,
        BaseRepoID: opts.BaseRepo.ID,
    }
}
```

**风险场景演示**：
```
原始仓库: org1/repo (ID=1)
    ↳ Fork: org2/repo (ID=2, ForkID=1)
        ↳ Fork: org3/repo (ID=3, ForkID=2)
```

用户想把 org3/repo (ID=3) fork 到 org1：

| 检查层级 | 检查内容 | 结果 |
|---------|---------|------|
| Web 端遍历 | GetForkedRepo(org1, 3) → 无 | ✅ 继续 |
| Web 端遍历 | GetForkedRepo(org1, 2) → 无 | ✅ 继续 |
| Web 端遍历 | GetForkedRepo(org1, 1) → 有（就是 org1/repo 自己）| ❌ 拒绝 |
| API 端检查 | GetUserFork(3, org1) → 无 | ✅ 允许 |
| 服务层检查 | GetUserFork(3, org1) → 无 | ✅ 允许 |

**最终结果**：
- Web 端：❌ 拒绝（正确）
- API 端：✅ 允许（错误，造成 org1 下有两个相同来源的仓库）

### 6.5 两层防御架构分析（校准后）

```
┌─────────────────────────────────────────────────────────────────────────┐
│  入口层检查 (Router Layer)                                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────────────────┐   ┌──────────────────────────────┐   │
│  │  Web 端 (ForkPost)           │   │  API 端 (CreateFork)         │   │
│  ├──────────────────────────────┤   ├──────────────────────────────┤   │
│  │ ✅ reqSignIn                 │   │ ✅ reqToken                  │   │
│  │ ✅ RepoAssignment + 权限     │   │ ✅ repoAssignment + 权限     │   │
│  │ ✅ reqUnitCodeReader         │   │ ✅ reqRepoReader(TypeCode)   │   │
│  │                              │   │                              │   │
│  │ ✅ 目标可见性 checkContextUser│   │ ✅ 个人: 天然可见            │   │
│  │ ✅ 目标创建权限              │   │ ✅ 组织: HasOrgOrUserVisible │   │
│  │                              │   │ ✅ 组织: CanCreateOrgRepo    │   │
│  │                              │   │                              │   │
│  │ ✅ 同一所有者限制 (祖先链)    │   │ ❌ 完全缺失                  │   │
│  │ ✅ 祖先链重复 fork           │   │ ❌ 完全缺失                  │   │
│  │ ✅ 直接重复 fork (祖先链)    │   │ ✅ 服务层兜底 GetUserFork    │   │
│  └──────────────────────────────┘   └──────────────────────────────┘   │
│                                       │                                  │
└───────────────────────────────────────┼──────────────────────────────────┘
                                        ↓                                  
┌─────────────────────────────────────────────────────────────────────────┐
│  服务层检查 (Service Layer) - ForkRepository                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ✅ IsUserBlockedBy(doer, baseRepoOwner)      - 用户封禁检查              │
│  ✅ CanForkRepoIn(doer, owner)               - 创建上限检查              │
│  ✅ GetUserFork(baseRepoID, ownerID)         - 直接父仓库重复检查        │
│  ✅ IsRepositoryModelExist(owner, name)       - 同名仓库检查              │
│  ✅ IsUsableRepoName(name)                   - 仓库名可用性检查          │
│  ❌ 同一所有者限制                           - 完全缺失                   │
│  ❌ 祖先链重复 fork                          - 完全缺失                   │
│  ❌ 目标可见性                               - 完全缺失                   │
│  ❌ 目标创建权限                             - 完全缺失                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**设计缺陷总结**：
1. **职责划分不清晰**：入口层承担了过多业务规则检查，服务层缺少核心防御
2. **API 端缺少同一所有者和祖先链检查**：但服务层的 `GetUserFork` + `IsRepositoryModelExist` 在默认名称场景下提供了隐式屏障，自定义名称场景才是真正的风险路径
3. **服务层信任过度**：假定入口层已做了所有检查，导致防御缺口

---

## 七、私有属性继承与可见性边界

### 7.1 IsPrivate 继承规则

**位置**：`services/repository/fork.go:98`

```go
IsPrivate: opts.BaseRepo.IsPrivate || opts.BaseRepo.Owner.Visibility == structs.VisibleTypePrivate,
```

**规则**：Fork 仓库的私有属性 = `源仓库私有 OR 源所有者是私有组织`

| 源仓库状态 | 源所有者可见性 | Fork 后 IsPrivate |
|-----------|--------------|-----------------|
| 公开 | Public | false |
| 公开 | Limited | false |
| 公开 | Private | true |
| 私有 | Public | true |
| 私有 | Limited | true |
| 私有 | Private | true |

### 7.2 源仓库变私有后的级联更新

**位置**：`services/repository/repository.go:287-291`

```go
// 当源仓库变为私有时，所有 fork 也同步变为私有
for i := range forkRepos {
    forkRepos[i].IsPrivate = repo.IsPrivate || repo.Owner.Visibility == structs.VisibleTypePrivate
    if err = updateRepository(ctx, forkRepos[i], true); err != nil {
        return fmt.Errorf("updateRepository[%d]: %w", forkRepos[i].ID, err)
    }
}
```

### 7.3 HasOrgOrUserVisible 可见性判定

**位置**：`models/organization/org.go:420-439`

```go
func HasOrgOrUserVisible(ctx context.Context, orgOrUser, user *user_model.User) bool {
    // 规则 1：匿名用户只能看到 Public 的组织
    if user == nil || user.IsGhost() {
        return orgOrUser.Visibility == structs.VisibleTypePublic
    }

    // 规则 2：管理员 OR 自己 总能看到
    if user.IsAdmin || orgOrUser.ID == user.ID {
        return true
    }

    // 规则 3：非严格模式下，Public 的组织所有人都能看到
    if !setting.Service.RequireSignInViewStrict && orgOrUser.Visibility == structs.VisibleTypePublic {
        return true
    }

    // 规则 4：私有组织 OR 受限用户 → 必须是组织成员才能看到
    if (orgOrUser.Visibility == structs.VisibleTypePrivate || user.IsRestricted) && 
        !OrgFromUser(orgOrUser).hasMemberWithUserID(ctx, user.ID) {
        return false
    }
    
    return true
}
```

### 7.4 可见性决策树

```
HasOrgOrUserVisible(orgOrUser, user)
    │
    ├─→ user 是匿名/幽灵？
    │    ├─ 是 → 仅 Public 可见
    │    └─ 否 → 继续
    │
    ├─→ user 是管理员 OR 是自己？
    │    └─ 是 → 可见
    │
    ├─→ 非严格模式 AND 组织是 Public？
    │    └─ 是 → 可见
    │
    ├─→ (组织是 Private OR 用户是受限) AND 不是成员？
    │    └─ 是 → 不可见
    │
    └─→ 其他情况 → 可见
```

### 7.5 最终可见性边界

| 组织可见性 | 用户身份 | 是否可见 |
|-----------|---------|---------|
| Public | 匿名 | ✅ 是 |
| Public | 登录用户 | ✅ 是 |
| Public | 受限用户 | ✅ 是（非严格模式） |
| Limited | 匿名 | ❌ 否 |
| Limited | 登录用户 | ✅ 是 |
| Limited | 受限用户 | ❌ 否（非成员） |
| Private | 匿名 | ❌ 否 |
| Private | 非成员 | ❌ 否 |
| Private | 成员 | ✅ 是 |

---

## 八、Forks 列表访问条件与可见范围深度分析

### 8.1 Web 端与 API 端入口对比

#### 8.1.1 Web 端 Forks 页面

**路由** (`routers/web/web.go:1719`)：
```go
m.Get("/forks", repo.Forks)
```

**中间件**：`optSignIn`, `context.RepoAssignment`, `reqUnitCodeReader`

**实现** (`routers/web/repo/view.go:382-412`)：
```go
func Forks(ctx *context.Context) {
    page := ctx.FormInt("page")
    if page <= 0 {
        page = 1
    }
    pageSize := setting.ItemsPerPage

    // 调用服务层查询
    forks, total, err := repo_service.FindForks(ctx, ctx.Repo.Repository, ctx.Doer, db.ListOptions{
        Page:     page,
        PageSize: pageSize,
    })
    if err != nil {
        ctx.ServerError("FindForks", err)
        return
    }

    // 加载 fork 仓库的所有者信息
    if err := repo_model.RepositoryList(forks).LoadOwners(ctx); err != nil {
        ctx.ServerError("LoadAttributes", err)
        return
    }

    ctx.Data["Repos"] = forks
    ctx.Data["Total"] = total
    ctx.HTML(http.StatusOK, tplForks)
}
```

#### 8.1.2 API 端 ListForks

**路由** (`routers/api/v1/api.go:1239`)：
```go
m.Combo("/forks").Get(repo.ListForks).Post(...)
```

**实现** (`routers/api/v1/repo/fork.go:27-83`)：
```go
func ListForks(ctx *context.APIContext) {
    forks, total, err := repo_service.FindForks(ctx, ctx.Repo.Repository, ctx.Doer, utils.GetListOptions(ctx))
    if err != nil {
        ctx.APIError(http.StatusInternalServerError, "FindForks", err)
        return
    }

    if err = repo_model.RepositoryList(forks).LoadAttributes(ctx); err != nil {
        ctx.APIError(http.StatusInternalServerError, "LoadAttributes", err)
        return
    }

    ctx.SetTotalCountHeader(total)
    ctx.JSON(http.StatusOK, convert.ToRepositories(ctx, forks))
}
```

**关键发现**：Web 端和 API 端使用完全相同的 `repo_service.FindForks` 函数，因此过滤逻辑完全一致。

### 8.2 FindForks 核心查询逻辑

**位置**：`services/repository/fork.go:235-275`

```go
type findForksOptions struct {
    db.ListOptions
    RepoID int64
    Doer   *user_model.User
}

func (opts findForksOptions) ToConds() builder.Cond {
    // 基础条件：直接 fork 关系（fork_id = 源仓库 ID）
    cond := builder.Eq{"fork_id": opts.RepoID}

    // 管理员：无条件看到所有 fork
    if opts.Doer != nil && opts.Doer.IsAdmin {
        return cond
    }

    // 非管理员：加上可见性条件
    // unit.TypeInvalid = 不要求特定单元权限，只要能访问仓库即可
    return cond.And(repo_model.AccessibleRepositoryCondition(opts.Doer, unit.TypeInvalid))
}

func (opts findForksOptions) ToOrders() string {
    return "`repository`.created_unix DESC"
}

func FindForks(ctx context.Context, repo *repo_model.Repository, doer *user_model.User, listOptions db.ListOptions) ([]*repo_model.Repository, int64, error) {
    return db.FindAndCount[repo_model.Repository](ctx, findForksOptions{
        ListOptions: listOptions,
        RepoID:      repo.ID,
        Doer:        doer,
    })
}
```

**重要注意**：`fork_id` 只记录**直接**父仓库。多级 fork 链中，每个 fork 只指向它的直接父仓库，不指向原始仓库。

### 8.3 AccessibleRepositoryCondition 完整解析

**位置**：`models/repo/repo_list.go:663-720`

```go
func AccessibleRepositoryCondition(user *user_model.User, unitType unit.Type) builder.Cond {
    cond := builder.NewCond()

    // =====================================================================
    // 条件组 1：公开仓库可见性（基于用户类型）
    // =====================================================================
    if user == nil || !user.IsRestricted || user.ID <= 0 {
        // 匿名用户 OR 非受限用户
        orgVisibilityLimit := []structs.VisibleType{structs.VisibleTypePrivate}
        
        // 匿名用户额外排除 Limited 组织
        if user == nil || user.ID <= 0 {
            orgVisibilityLimit = append(orgVisibilityLimit, structs.VisibleTypeLimited)
        }
        
        // 可见：非指定可见性组织下的所有非私有仓库
        cond = userAllPublicRepoCond(cond, orgVisibilityLimit)
    }

    if user != nil {
        // =====================================================================
        // 条件组 2：通过权限系统获得的访问权
        // =====================================================================
        if unitType == unit.TypeInvalid {
            // 不要求特定单元权限 → 只要能访问仓库即可
            cond = cond.Or(
                // a) access 表中的直接权限（个人权限、团队权限的预计算结果）
                UserAccessRepoCond("`repository`.id", user.ID),
                // b) 通过团队成员身份获得权限（关联 team_user → team_repo）
                UserOrgTeamRepoCond("`repository`.id", user.ID),
            )
        } else {
            // 要求特定单元权限（如 TypeCode, TypeIssues 等）
            cond = cond.Or(
                // a) 作为协作者被直接添加
                UserCollaborationRepoCond("`repository`.id", user.ID),
                // b) 团队对该单元有权限
                userOrgTeamUnitRepoCond("`repository`.id", user.ID, unitType),
            )
        }

        // =====================================================================
        // 条件组 3：用户是仓库所有者
        // =====================================================================
        cond = cond.Or(builder.Eq{"`repository`.owner_id": user.ID})

        // =====================================================================
        // 条件组 4：所属组织的公开仓库
        // =====================================================================
        if !user.IsRestricted {
            // 非受限用户：能看到自己所属组织下的公开仓库（即使组织是 Private）
            cond = cond.Or(userOrgPublicRepoCond(user.ID))
        } else if !setting.Service.RequireSignInViewStrict {
            // 受限用户 + 非严格模式：放宽公开仓库可见性
            orgVisibilityLimit := []structs.VisibleType{structs.VisibleTypePrivate, structs.VisibleTypeLimited}
            cond = userAllPublicRepoCond(cond, orgVisibilityLimit)
        }
    }

    return cond
}
```

### 8.4 关键辅助条件函数解析

#### 8.4.1 UserAccessRepoCond - 直接权限

**位置**：`models/repo/repo_list.go:653-655`
```go
func UserAccessRepoCond(col, userID int64) builder.Cond {
    return builder.In(col,
        builder.Select("repo_id").From("access").Where(builder.Eq{"user_id": userID}),
    )
}
```
- 查询 `access` 表：用户-仓库的预计算权限记录
- 由 `RecalculateTeamAccesses` 维护

#### 8.4.2 UserOrgTeamRepoCond - 团队成员权限

**位置**：`models/repo/repo_list.go:657-661`
```go
func UserOrgTeamRepoCond(col, userID int64) builder.Cond {
    return builder.In(col,
        builder.Select("tr.repo_id").
            From("team_repo tr").
            Join("INNER", "team_user tu", "tu.team_id = tr.team_id").
            Where(builder.Eq{"tu.uid": userID}),
    )
}
```
- 通过 `team_user` → `team_repo` 关联查询
- 只要是团队成员且团队关联了仓库，就可见

#### 8.4.3 userAllPublicRepoCond - 公开仓库条件

```go
func userAllPublicRepoCond(cond builder.Cond, orgVisibilityLimit []structs.VisibleType) builder.Cond {
    return cond.Or(
        // 个人用户的公开仓库
        builder.And(
            builder.Eq{"`repository`.is_private": false},
            builder.NotIn("`repository`.owner_id",
                builder.Select("id").From("user").
                    Where(builder.In("visibility", orgVisibilityLimit)),
            ),
        ),
        // 组织用户的公开仓库（组织可见性不在限制列表中）
        builder.And(
            builder.Eq{"`repository`.is_private": false},
            builder.In("`repository`.owner_id",
                builder.Select("id").From("user").
                    Where(builder.NotIn("visibility", orgVisibilityLimit)),
            ),
        ),
    )
}
```

### 8.5 可见范围边界矩阵

#### 8.5.1 按用户类型

| 用户类型 | 可见范围条件 |
|---------|-------------|
| **管理员** | ✅ 所有 fork（无条件） |
| **非受限登录用户** | OR(条件组1, 条件组2, 条件组3, 条件组4)<br>= 公开仓库 + 有权限的私有 + 自己的 + 所属组织的公开 |
| **受限登录用户** | OR(条件组2, 条件组3) [+ 条件组1(非严格模式)]<br>= 有权限的私有 + 自己的 [+ 公开仓库] |
| **匿名用户** | 条件组1（排除 Private 和 Limited 组织）<br>= 仅 Public 组织下的公开仓库 |

#### 8.5.2 按仓库类型

| 仓库类型 | 可见条件 |
|---------|---------|
| **公开仓库 (Public 组织)** | ✅ 所有人（包括匿名） |
| **公开仓库 (Limited 组织)** | ✅ 登录用户可见，❌ 匿名不可见 |
| **公开仓库 (Private 组织)** | ✅ 组织成员可见，❌ 非成员不可见 |
| **私有仓库** | ✅ 有访问权限的用户可见（access 表或团队成员） |

### 8.6 跨组织 Fork 可见性场景深度分析

#### 场景 1：多级 Fork 链的可见性

```
原始仓库: orgA/repo (ID=1, is_private=false, orgA=Public)
    ↳ Fork1: orgB/repo (ID=2, fork_id=1, is_private=true, orgB=Private)
        ↳ Fork2: orgC/repo (ID=3, fork_id=2, is_private=false, orgC=Public)
            ↳ Fork3: user1/repo (ID=4, fork_id=3, is_private=true)
```

**查看 orgA/repo 的 forks 列表**：
- 只有 `fork_id=1` 的仓库会被列出 → **只有 Fork1 (orgB/repo)**
- Fork2/Fork3 不会出现在 orgA/repo 的列表中（它们的 fork_id 不是 1）

**不同用户看到的结果**：

| 用户 | 看到 orgA/repo 的 forks | 看到 orgB/repo 的 forks | 看到 orgC/repo 的 forks |
|-----|------------------------|------------------------|------------------------|
| 管理员 | ✅ orgB/repo | ✅ orgC/repo | ✅ user1/repo |
| orgA 成员（非其他组织） | ❌ （orgB 是私有，无权限） | ❌ （无权访问 orgB/repo） | ✅ orgC 是 Public + 公开仓库 |
| orgB 成员 | ✅ orgB/repo | ✅ orgC/repo | ❌ user1 是私有 |
| orgC 成员 | ❌ （无权访问 orgB/repo） | ✅ orgC/repo | ✅ user1/repo（如果有访问权） |
| user1 | ❌ （无权访问 orgB/repo） | ❌ （无权访问 orgB/repo） | ✅ user1/repo（自己的） |
| 匿名用户 | ❌ （orgB 是私有） | ❌ （无权访问 orgB/repo） | ✅ orgC 是 Public |

#### 场景 2：跨组织 fork 后源仓库变私有

```
初始状态：
orgA/repo (public) → orgB/repo (public)

变化：orgA 将 repo 改为 private
级联更新：所有 fork 的 is_private 被强制设为 true
最终状态：
orgA/repo (private) → orgB/repo (private)
```

**对 forks 列表的影响**：
- 只有 orgA 成员且有 orgA/repo 访问权的用户才能看到源仓库
- 能看到源仓库的用户中，只有同时有 orgB/repo 访问权的才能看到这个 fork
- 结果：大部分用户看不到这个 fork 了

#### 场景 3：fork 到个人空间后的可见性

```
源仓库：orgA/repo (private, orgA 成员: user1, user2)
Fork1：user1/repo (private)
Fork2：user2/repo (public)
```

**查看 orgA/repo 的 forks 列表**：

| 用户 | 看到的 forks | 原因 |
|-----|-------------|------|
| user1 | ✅ user1/repo, ✅ user2/repo | 自己的仓库 + user2 的是公开 |
| user2 | ✅ user1/repo?, ✅ user2/repo | user1 的是私有，需要有访问权 |
| orgA 成员 user3 | ❌ user1/repo, ✅ user2/repo | user1 没加 user3 为协作者 |
| 管理员 | ✅ user1/repo, ✅ user2/repo | 管理员看到所有 |
| 匿名用户 | ❌ 都看不到 | 源仓库是私有，先被 RepoAssignment 拦截 |

### 8.7 权限传播与可见性边界总结

1. **Fork 创建后权限独立**
   - 源仓库的团队成员不会自动加入 fork 仓库
   - fork 仓库有自己的团队/协作者权限体系
   - 但 `is_private` 属性会随源仓库级联更新

2. **Fork 链是扁平化的**
   - 每个 fork 只记录直接父仓库（`fork_id`）
   - 查看 forks 列表时只显示直接 fork，不显示间接 fork
   - 多级 fork 的可见性需要逐层判断

3. **可见性是动态计算的**
   - 每次查询 forks 列表时实时应用 `AccessibleRepositoryCondition`
   - 源仓库的访问权不代表 fork 仓库的访问权
   - 即使能看到源仓库，也可能看不到它的某些 fork

4. **PR 跨 fork 的特殊权限**
   - 从 fork 提 PR 到源仓库：只需要 fork 用户有源仓库的**读权限**
   - 源仓库审查者不需要有 fork 仓库的权限
   - 这是为了支持开源协作模式：任何人 fork 后都能提 PR

---

## 九、GetIndividualUserRepoPermission 详细流程

### 9.1 Permission 结构

```go
type Permission struct {
    AccessMode perm.AccessMode  // 整体权限
    units      []*repo_model.RepoUnit
    unitsMode  map[unit.Type]perm.AccessMode  // 各单元细粒度权限

    everyoneAccessMode  map[unit.Type]perm.AccessMode  // 登录用户默认权限
    anonymousAccessMode map[unit.Type]perm.AccessMode  // 匿名用户默认权限
}
```

### 9.2 12 步判定流程

```
GetIndividualUserRepoPermission(ctx, repo, user)
    │
    ├─→ 1. 加载仓库单元 (repo.LoadUnits)
    │
    ├─→ 2. 匿名用户访问私有仓库 → AccessModeNone
    │
    ├─→ 3. 检查是否为协作者 (IsCollaborator)
    │
    ├─→ 4. 检查组织/用户可见性 (HasOrgOrUserVisible)
    │    └─→ 不可见且非协作者 → AccessModeNone
    │
    ├─→ 5. 匿名用户访问公开仓库 → AccessModeRead
    │
    ├─→ 6. 管理员或仓库所有者 → AccessModeOwner
    │
    ├─→ 7. 调用 accessLevel 从 access 表查权限
    │
    ├─→ 8. 非组织仓库 → 直接返回
    │
    ├─→ 9. 组织仓库最小权限 (公开+非受限用户 → Read)
    │
    ├─→ 10. 获取用户团队权限 (GetUserRepoTeams)
    │
    ├─→ 11. Owner 团队成员 → AccessModeOwner (提前返回)
    │
    └─→ 12. 按单元取最大权限 (max(unitsMode, minAccessMode, teamMode))
```

---

## 十、关键数据表关系图

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   org_user   │      │     team     │      │  repository  │
│ (组织成员表)  │      │   (团队表)    │      │   (仓库表)    │
├──────────────┤      ├──────────────┤      ├──────────────┤
│ uid    [U]   │◄────►│ id    [PK]   │◄────►│ id     [PK]  │
│ org_id [U]   │      │ org_id [I]   │      │ owner_id [I] │
│ is_public [I]│      │ name         │      │ is_private   │
└──────────────┘      └──────────────┘      │ fork_id      │
         │                   │              └──────────────┘
         │                   │                   │
         ▼                   ▼                   ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   team_user  │      │  team_repo   │      │    access    │
│ (团队成员表)  │      │ (团队仓库表)  │      │ (权限表)     │
├──────────────┤      ├──────────────┤      ├──────────────┤
│ uid    [U]   │      │ team_id [U]  │      │ user_id [U]  │
│ team_id [U]  │      │ repo_id [U]  │      │ repo_id [U]  │
│ org_id [I]   │      │ org_id [I]   │      │ mode         │
└──────────────┘      └──────────────┘      └──────────────┘
                              ▲
                              │
                    ┌──────────────┐
                    │  team_unit   │
                    │ (团队单元表)  │
                    ├──────────────┤
                    │ team_id [U]  │
                    │ type    [U]  │
                    │ access_mode  │
                    └──────────────┘

注：[PK] 主键, [U] 联合唯一, [I] 索引
```

---

## 十一、调试技巧

### 11.1 常用查询 SQL

```sql
-- 查看用户最终权限
SELECT * FROM access WHERE user_id = ? AND repo_id = ?;

-- 查看用户所在团队及权限
SELECT t.name, t.authorize, tu.uid 
FROM team t
JOIN team_user tu ON tu.team_id = t.id
WHERE tu.uid = ? AND t.org_id = ?;

-- 查看团队-仓库关联
SELECT t.name, r.name 
FROM team t
JOIN team_repo tr ON tr.team_id = t.id
JOIN repository r ON r.id = tr.repo_id
WHERE t.org_id = ?;

-- 查看组织成员
SELECT u.name, ou.is_public 
FROM org_user ou
JOIN user u ON u.id = ou.uid
WHERE ou.org_id = ?;
```

### 11.2 关键调试日志

**位置**：`models/perm/access/repo_permission.go:399`

```go
log.Trace("Permission Loaded for user %-v in repo %-v, permissions: %-+v", user, repo, perm)
```

### 11.3 access 表清理规则

`refreshAccesses` 函数会智能清理不必要的记录：
- 公开仓库的非受限用户：Read 权限不写入（默认公开）
- 个人仓库的非受限用户：Write 及以下不写入
- 只有高于默认权限的才会显式记录

---

## 十二、代码路径速查

| 功能 | 文件 | 核心函数 |
|------|------|---------|
| Team 模型 | `models/organization/team.go` | `IsMember`, `HasAdminAccess` |
| Team 成员 | `models/organization/team_user.go` | `IsTeamMember`, `GetTeamMembers` |
| Team-Repo 关联 | `models/organization/team_repo.go` | `HasTeamRepo` |
| Team 单元权限 | `models/organization/team_unit.go` | `getUnitsByTeamID` |
| 团队列表查询 | `models/organization/team_list.go` | `GetUserRepoTeams` |
| 可见性判定 | `models/organization/org.go` | `HasOrgOrUserVisible` |
| Access 模式 | `models/perm/access_mode.go` | `AccessMode` 定义 |
| 权限计算核心 | `models/perm/access/repo_permission.go` | `GetIndividualUserRepoPermission` |
| Access 表维护 | `models/perm/access/access.go` | `RecalculateTeamAccesses` |
| 仓库可见性条件 | `models/repo/repo_list.go` | `AccessibleRepositoryCondition` |
| Fork API | `routers/api/v1/repo/fork.go` | `CreateFork` |
| Fork Web | `routers/web/repo/fork.go` | `ForkPost` |
| Fork 服务 | `services/repository/fork.go` | `ForkRepository` |
| 仓库创建 | `services/repository/create.go` | `createRepositoryInDB` |
| Fork 边界检查 | `modules/repository/fork.go` | `CanUserForkBetweenOwners` |
