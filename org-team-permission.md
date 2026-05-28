# Gitea Organization Team 与仓库授权机制分析

## 一、核心数据模型

### 1.1 Team 模型 (`models/organization/team.go`)

```go
type Team struct {
    ID                      int64
    OrgID                   int64
    LowerName               string
    Name                    string
    Description             string
    AccessMode              perm.AccessMode    // 团队整体权限位
    Members                 []*user_model.User `xorm:"-"`
    NumRepos                int
    NumMembers              int
    Units                   []*TeamUnit `xorm:"-"` // 各单元细粒度权限
    IncludesAllRepositories bool
    CanCreateOrgRepo        bool
}
```

**关键设计要点：**
- `AccessMode` 字段 (`authorize` 列) 存储团队的整体权限级别
- `Units` 字段是细粒度的按单元权限控制，通过 `TeamUnit` 表存储
- `IncludesAllRepositories` 表示团队是否自动拥有组织内所有仓库的访问权

### 1.2 TeamUser 团队成员关联 (`models/organization/team_user.go`)

```go
type TeamUser struct {
    ID     int64
    OrgID  int64
    TeamID int64 `xorm:"UNIQUE(s)"`
    UID    int64 `xorm:"UNIQUE(s)"`
}
```

- 多对多关系表，通过 `OrgID + TeamID + UID` 唯一确定成员归属
- 成员归属查询：`IsTeamMember(ctx, orgID, teamID, userID)`

### 1.3 TeamRepo 团队仓库关联 (`models/organization/team_repo.go`)

```go
type TeamRepo struct {
    ID     int64
    OrgID  int64
    TeamID int64 `xorm:"UNIQUE(s)"`
    RepoID int64 `xorm:"UNIQUE(s)"`
}
```

- 关联团队与仓库，一个团队可以管理多个仓库
- 查询：`HasTeamRepo(ctx, orgID, teamID, repoID)`

### 1.4 TeamUnit 团队单元权限 (`models/organization/team_unit.go`)

```go
type TeamUnit struct {
    ID         int64
    OrgID      int64
    TeamID     int64     `xorm:"UNIQUE(s)"`
    Type       unit.Type `xorm:"UNIQUE(s)"` // 单元类型（代码、Issue、PR等）
    AccessMode perm.AccessMode
}
```

- 细粒度权限控制，按单元（Code、Issues、PullRequests、Wiki 等）设置权限
- 每个团队可以对不同单元有不同的访问级别

### 1.5 Access 权限表 (`models/perm/access/access.go`)

```go
type Access struct {
    ID     int64
    UserID int64 `xorm:"UNIQUE(s)"`
    RepoID int64 `xorm:"UNIQUE(s)"`
    Mode   perm.AccessMode
}
```

- **关键表**：存储用户对仓库的最终聚合权限
- 此表是通过 `RecalculateTeamAccesses` 动态计算和维护的

---

## 二、权限位定义 (`models/perm/access_mode.go`)

```go
type AccessMode int

const (
    AccessModeNone  AccessMode = iota // 0: 无访问
    AccessModeRead                    // 1: 只读
    AccessModeWrite                   // 2: 可写
    AccessModeAdmin                   // 3: 管理员
    AccessModeOwner                   // 4: 所有者
)
```

**权限比较规则**：数值越大权限越高，使用 `>=` 比较。例如：
- `mode >= AccessModeRead` 表示有读权限
- `mode >= AccessModeAdmin` 表示有管理员权限

---

## 三、Team 成员归属判定

### 3.1 成员归属层级结构

```
Organization (org_user 表)
    ↓
    Team (team_user 表)
        ↓
        User
```

### 3.2 核心查询函数

**1. 检查是否为组织成员** (`models/organization/org_user.go:87-93`)

```go
func IsOrganizationMember(ctx context.Context, orgID, uid int64) (bool, error) {
    return db.GetEngine(ctx).
        Where("uid=?", uid).
        And("org_id=?", orgID).
        Table("org_user").
        Exist()
}
```

**2. 检查是否为团队成员** (`models/organization/team_user.go:24-31`)

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

**3. 获取用户在组织中的所有团队** (`models/organization/team_list.go:109-116`)

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

### 4.1 权限聚合的核心流程

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

| 操作 | 触发函数 | 位置 |
|------|---------|------|
| 添加团队成员 | `RecalculateUserAccess` | services/org/team.go |
| 移除团队成员 | `RecalculateUserAccess` | services/org/team.go |
| 团队添加仓库 | `RecalculateTeamAccesses` | services/repository/repo_team.go |
| 团队移除仓库 | `RecalculateTeamAccesses` | services/repository/repo_team.go |
| 团队权限变更 | `RecalculateTeamAccesses` | services/org/team.go |
| 仓库转移 | `RecalculateAccesses` | services/repository/transfer.go |

---

## 五、GetIndividualUserRepoPermission 详细分析

**位置**：`models/perm/access/repo_permission.go:394-495`

这是权限判定的核心入口函数，返回完整的 `Permission` 结构体。

### 5.1 Permission 结构

```go
type Permission struct {
    AccessMode perm.AccessMode  // 整体权限
    units      []*repo_model.RepoUnit
    unitsMode  map[unit.Type]perm.AccessMode  // 各单元细粒度权限

    everyoneAccessMode  map[unit.Type]perm.AccessMode  // 登录用户默认权限
    anonymousAccessMode map[unit.Type]perm.AccessMode  // 匿名用户默认权限
}
```

### 5.2 判定流程详解

```
┌─────────────────────────────────────────────────────────┐
│ GetIndividualUserRepoPermission(ctx, repo, user)        │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 1: 加载仓库单元 (repo.LoadUnits)                   │
│         perm.units = repo.Units                         │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 2: 匿名用户访问私有仓库 → AccessModeNone           │
│         (user == nil && repo.IsPrivate)                 │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 3: 检查是否为协作者 (IsCollaborator)               │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 4: 检查组织/用户可见性 (HasOrgOrUserVisible)       │
│         不可见且非协作者 → AccessModeNone                │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 5: 匿名用户访问公开仓库 → AccessModeRead            │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 6: 管理员或仓库所有者 → AccessModeOwner            │
│         (user.IsAdmin || user.ID == repo.OwnerID)       │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 7: 调用 accessLevel 从 access 表查权限             │
│         (access 表是预计算的聚合结果)                    │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 8: 非组织仓库 → 直接返回结果                        │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 9: 组织仓库最小权限 (公开+非受限用户 → Read)        │
│         minAccessMode = max(current, public_access)     │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 10: 获取用户团队权限 (GetUserRepoTeams)            │
│          遍历团队，取各单元权限最大值                     │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 11: Owner 团队成员 → AccessModeOwner               │
│         (team.HasAdminAccess() → 直接返回 Owner)        │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│ Step 12: 计算各单元权限                                  │
│         for each unit + team:                           │
│             unitAccessMode = max(unitsMode, min, team)  │
└─────────────────────────────────────────────────────────┘
```

### 5.3 关键代码片段

**团队权限判定** (`repo_permission.go:459-492`)：

```go
// 获取用户对该仓库有访问权的所有团队
teams, err := organization.GetUserRepoTeams(ctx, repo.OwnerID, user.ID, repo.ID)
if len(teams) == 0 {
    return perm, nil
}

perm.unitsMode = make(map[unit.Type]perm.AccessMode)

// 协作者权限优先
if isCollaborator {
    for _, u := range repo.Units {
        perm.unitsMode[u.Type] = perm.AccessMode
    }
}

// Owner 团队直接升级为 Owner 权限
for _, team := range teams {
    if team.HasAdminAccess() {
        perm.AccessMode = perm_model.AccessModeOwner
        perm.unitsMode = nil
        return perm, nil
    }
}

// 按单元取最大权限
for _, u := range repo.Units {
    for _, team := range teams {
        teamMode, _ := team.UnitAccessModeEx(ctx, u.Type)
        unitAccessMode := max(perm.unitsMode[u.Type], minAccessMode, teamMode)
        perm.unitsMode[u.Type] = unitAccessMode
    }
}
```

---

## 六、跨组织 Fork 权限判定

### 6.1 Fork 操作的权限检查

**位置**：`routers/api/v1/repo/fork.go:116-179`

```go
func CreateFork(ctx *context.APIContext) {
    form := web.GetForm(ctx).(*api.CreateForkOption)
    forkOwner := ctx.Doer
    
    // 如果指定了组织作为 fork 目标...
    if form.Organization != nil {
        org := prepareDoerCreateRepoInOrg(ctx, *form.Organization)
        forkOwner = org.AsUser()
    }

    // 检查：用户未被源仓库所有者封禁
    if user_model.IsUserBlockedBy(ctx, doer, opts.BaseRepo.Owner.ID) {
        return nil, user_model.ErrBlockedUser
    }

    // 检查：用户未达到仓库创建上限
    if !doer.CanForkRepoIn(owner) {
        return nil, repo_model.ErrReachLimitOfRepo{...}
    }

    // 检查：未 fork 过
    forkedRepo, err := repo_model.GetUserFork(ctx, opts.BaseRepo.ID, owner.ID)
    if forkedRepo != nil {
        return nil, ErrForkAlreadyExist{...}
    }

    // 创建 fork
    fork, err := repo_service.ForkRepository(ctx, ctx.Doer, forkOwner, ...)
}
```

### 6.2 prepareDoerCreateRepoInOrg 检查

**位置**：`routers/api/v1/repo/fork.go:86-113`

```go
func prepareDoerCreateRepoInOrg(ctx *context.APIContext, orgName string) *organization.Organization {
    org, err := organization.GetOrgByName(ctx, orgName)
    
    // 1. 组织对用户可见
    if !organization.HasOrgOrUserVisible(ctx, org.AsUser(), ctx.Doer) {
        ctx.APIErrorNotFound()
        return nil
    }

    // 2. 非管理员需要检查是否有创建仓库权限
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

### 6.3 CanCreateOrgRepo 权限判定

**位置**：`models/organization/org_user.go:106-113`

```go
func CanCreateOrgRepo(ctx context.Context, orgID, uid int64) (bool, error) {
    return db.GetEngine(ctx).
        Where(
            builder.Eq{"team.can_create_org_repo": true}.
                Or(builder.Eq{"team.authorize": perm.AccessModeOwner}),
        ).
        Join("INNER", "team_user", "team_user.team_id = team.id").
        And("team_user.uid = ?", uid).
        And("team_user.org_id = ?", orgID).
        Exist(new(Team))
}
```

**满足条件之一即可**：
1. 所在团队的 `can_create_org_repo = true`
2. 所在团队的 `authorize = AccessModeOwner`（即 Owners 团队）

### 6.4 Fork 仓库列表可见性

**位置**：`services/repository/fork.go:235-256`

```go
func FindForks(ctx context.Context, repo *repo_model.Repository, doer *user_model.User, ...) {
    return db.FindAndCount[repo_model.Repository](ctx, findForksOptions{
        RepoID: repo.ID,
        Doer:   doer,
    })
}

func (opts findForksOptions) ToConds() builder.Cond {
    cond := builder.Eq{"fork_id": opts.RepoID}
    if opts.Doer != nil && opts.Doer.IsAdmin {
        return cond
    }
    // 关键：非管理员只能看到自己有权限访问的 fork
    return cond.And(repo_model.AccessibleRepositoryCondition(opts.Doer, unit.TypeInvalid))
}
```

### 6.5 AccessibleRepositoryCondition 访问条件

**位置**：`models/repo/repo_list.go:663-704`

```go
func AccessibleRepositoryCondition(user *user_model.User, unitType unit.Type) builder.Cond {
    cond := builder.NewCond()

    // 1. 非受限用户：能看到所有非私有仓库
    if user == nil || !user.IsRestricted {
        cond = userAllPublicRepoCond(cond, orgVisibilityLimit)
    }

    if user != nil {
        // 2. 有独立访问权限的仓库（access 表）
        // 3. 通过团队成员能访问的仓库
        if unitType == unit.TypeInvalid {
            cond = cond.Or(
                UserAccessRepoCond("`repository`.id", user.ID),
                UserOrgTeamRepoCond("`repository`.id", user.ID),
            )
        } else {
            // 特定单元权限检查
            cond = cond.Or(
                UserCollaborationRepoCond("`repository`.id", user.ID),
                userOrgTeamUnitRepoCond("`repository`.id", user.ID, unitType),
            )
        }
        // 4. 自己拥有的仓库
        cond = cond.Or(builder.Eq{"`repository`.owner_id": user.ID})
        
        // 5. 非受限用户：能看到所属私有组织的公开仓库
        if !user.IsRestricted {
            cond = cond.Or(userOrgPublicRepoCond(user.ID))
        }
    }
    return cond
}
```

### 6.6 跨组织 Fork 的权限边界

| 场景 | 权限判定 |
|------|---------|
| 用户 A 能否 fork 组织 B 的公开仓库 | ✅ 能（公开仓库默认可读） |
| 用户 A 能否 fork 组织 B 的私有仓库 | ✅ 能（只要 A 有该仓库读权限） |
| Fork 到用户 A 的个人空间 | ✅ 不需要额外权限（A 是所有者） |
| Fork 到组织 C | 需要 A 在 C 中有创建仓库权限 |
| Fork 后的可见性 | 遵循目标所有者（用户/组织）的权限规则 |
| PR 从 fork 提交到源仓库 | 需要 fork 仓库用户有源仓库的读权限 |

---

## 七、关键数据表关系图

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   org_user   │      │     team     │      │  repository  │
│ (组织成员表)  │      │   (团队表)    │      │   (仓库表)    │
├──────────────┤      ├──────────────┤      ├──────────────┤
│ uid          │      │ id           │      │ id           │
│ org_id       │      │ org_id       │      │ owner_id     │
│ is_public    │      │ name         │      │ is_private   │
└──────────────┘      │ authorize    │      │ fork_id      │
         │            └──────────────┘      └──────────────┘
         │                   │  │                   │
         │                   │  │                   │
         ▼                   ▼  ▼                   ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   team_user  │      │  team_repo   │      │    access    │
│ (团队成员表)  │      │ (团队仓库表)  │      │ (权限表)     │
├──────────────┤      ├──────────────┤      ├──────────────┤
│ uid          │      │ team_id      │      │ user_id      │
│ team_id      │      │ repo_id      │      │ repo_id      │
│ org_id       │      │ org_id       │      │ mode         │
└──────────────┘      └──────────────┘      └──────────────┘
                                                         ↑
                                                         │
                                            ┌──────────────┐
                                            │ team_unit    │
                                            │ (团队单元表)  │
                                            ├──────────────┤
                                            │ team_id      │
                                            │ type         │
                                            │ access_mode  │
                                            └──────────────┘
```

---

## 八、常见问题与调试技巧

### 8.1 权限未生效？检查是否触发了重算

```sql
-- 查看用户最终权限
SELECT * FROM access WHERE user_id = ? AND repo_id = ?;

-- 查看用户所在团队
SELECT t.name, t.authorize FROM team t
JOIN team_user tu ON tu.team_id = t.id
WHERE tu.uid = ? AND t.org_id = ?;

-- 查看团队-仓库关联
SELECT t.name FROM team t
JOIN team_repo tr ON tr.team_id = t.id
WHERE tr.repo_id = ?;
```

### 8.2 关键调试日志

在 `repo_permission.go:399` 有 Trace 日志：
```go
log.Trace("Permission Loaded for user %-v in repo %-v, permissions: %-+v", user, repo, perm)
```

开启 Trace 日志可以看到完整的权限计算过程。

### 8.3 access 表清理规则

`refreshAccesses` 函数会智能清理不必要的记录：
- 公开仓库的非受限用户：Read 权限不写入 access 表（默认公开）
- 个人仓库的非受限用户：Write 及以下不写入 access 表
- 只有高于默认权限的才会显式记录

---

## 九、代码路径速查

| 功能 | 文件 | 核心函数 |
|------|------|---------|
| Team 模型 | `models/organization/team.go` | `IsMember`, `HasAdminAccess` |
| Team 成员 | `models/organization/team_user.go` | `IsTeamMember`, `GetTeamMembers` |
| Team-Repo 关联 | `models/organization/team_repo.go` | `HasTeamRepo` |
| Team 单元权限 | `models/organization/team_unit.go` | `getUnitsByTeamID` |
| 团队列表查询 | `models/organization/team_list.go` | `GetUserRepoTeams` |
| Access 模式 | `models/perm/access_mode.go` | `AccessMode` 定义 |
| 权限计算核心 | `models/perm/access/repo_permission.go` | `GetIndividualUserRepoPermission` |
| Access 表维护 | `models/perm/access/access.go` | `RecalculateTeamAccesses` |
| 仓库可见性 | `models/repo/repo_list.go` | `AccessibleRepositoryCondition` |
| Fork API | `routers/api/v1/repo/fork.go` | `CreateFork` |
| Fork 服务 | `services/repository/fork.go` | `ForkRepository` |
