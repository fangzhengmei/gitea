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

## 六、私有属性继承与可见性边界

### 6.1 IsPrivate 继承规则

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

### 6.2 源仓库变私有后的级联更新

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

### 6.3 HasOrgOrUserVisible 可见性判定

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

### 6.4 可见性决策树

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

### 6.5 最终可见性边界

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

## 七、GetIndividualUserRepoPermission 详细流程

### 7.1 Permission 结构

```go
type Permission struct {
    AccessMode perm.AccessMode  // 整体权限
    units      []*repo_model.RepoUnit
    unitsMode  map[unit.Type]perm.AccessMode  // 各单元细粒度权限

    everyoneAccessMode  map[unit.Type]perm.AccessMode  // 登录用户默认权限
    anonymousAccessMode map[unit.Type]perm.AccessMode  // 匿名用户默认权限
}
```

### 7.2 12 步判定流程

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

## 八、关键数据表关系图

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

## 九、调试技巧

### 9.1 常用查询 SQL

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

### 9.2 关键调试日志

**位置**：`models/perm/access/repo_permission.go:399`

```go
log.Trace("Permission Loaded for user %-v in repo %-v, permissions: %-+v", user, repo, perm)
```

### 9.3 access 表清理规则

`refreshAccesses` 函数会智能清理不必要的记录：
- 公开仓库的非受限用户：Read 权限不写入（默认公开）
- 个人仓库的非受限用户：Write 及以下不写入
- 只有高于默认权限的才会显式记录

---

## 十、代码路径速查

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
