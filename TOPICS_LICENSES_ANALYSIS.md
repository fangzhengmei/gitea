# Gitea 话题标签与许可信息全链路源码分析

本文档对 Gitea 仓库中话题标签（Topics）与许可信息（Licenses）从用户输入、规范化、搜索索引到对外接口的完整链路进行源码级拆解。

---

## 一、话题标签（Topics）链路分析

### 1.1 数据模型定义

**核心模型**位于 `models/repo/topic.go`：

```go
// Topic represents a topic of repositories
type Topic struct {
    ID          int64  `xorm:"pk autoincr"`
    Name        string `xorm:"UNIQUE VARCHAR(50)"`  // 唯一索引，最大50字符
    RepoCount   int                                   // 使用该标签的仓库数量
    CreatedUnix timeutil.TimeStamp `xorm:"INDEX created"`
    UpdatedUnix timeutil.TimeStamp `xorm:"INDEX updated"`
}

// RepoTopic 仓库与标签的关联表（多对多关系）
type RepoTopic struct {
    RepoID  int64 `xorm:"pk"`
    TopicID int64 `xorm:"pk"`
}
```

**Repository 模型**中的冗余字段（用于性能优化）：
```go
// models/repo/repo.go:211
Topics []string `xorm:"TEXT JSON"`  // 以JSON数组形式缓存标签列表
```

### 1.2 用户输入与规范化策略

#### 1.2.1 验证规则

标签验证函数 `ValidateTopic` (`models/repo/topic.go:57-59`)：
```go
var topicPattern = regexp.MustCompile(`^[a-z0-9][-.a-z0-9]*$`)

func ValidateTopic(topic string) bool {
    return len(topic) <= 35 && topicPattern.MatchString(topic)
}
```

**验证规则**：
- 长度限制：≤ 35 字符
- 正则匹配：`^[a-z0-9][-.a-z0-9]*$`
  - 必须以小写字母或数字开头
  - 后续字符只能是小写字母、数字、连字符（-）或点（.）
  - 不允许大写字母（输入时自动转换）

#### 1.2.2 归一化策略

`SanitizeAndValidateTopics` 函数 (`models/repo/topic.go:62-86`) 实现完整的归一化流程：

```go
func SanitizeAndValidateTopics(topics []string) (validTopics, invalidTopics []string) {
    validTopics = make([]string, 0)
    mValidTopics := make(container.Set[string])  // 用于去重
    invalidTopics = make([]string, 0)

    for _, topic := range topics {
        // 步骤1: 去除首尾空白 + 转换为小写
        topic = strings.TrimSpace(strings.ToLower(topic))
        
        // 步骤2: 忽略空字符串
        if len(topic) == 0 {
            continue
        }
        
        // 步骤3: 去重（忽略重复标签）
        if mValidTopics.Contains(topic) {
            continue
        }
        
        // 步骤4: 格式验证
        if ValidateTopic(topic) {
            validTopics = append(validTopics, topic)
            mValidTopics.Add(topic)
        } else {
            invalidTopics = append(invalidTopics, topic)
        }
    }
    return validTopics, invalidTopics
}
```

**归一化四步曲**：
1. **空白清理**：`strings.TrimSpace()` 去除首尾空白
2. **大小写归一**：`strings.ToLower()` 全部转换为小写
3. **空值过滤**：跳过空字符串
4. **去重处理**：使用 `container.Set[string]` 确保唯一性

#### 1.2.3 数量限制

每个仓库最多允许 25 个标签，限制在以下位置检查：
- API 层：`routers/api/v1/repo/topic.go:108-114`（UpdateTopics）
- API 层：`routers/api/v1/repo/topic.go:184-189`（AddTopic）
- Web 层：`routers/web/repo/topic.go:32-38`（TopicsPost）

### 1.3 标签词典管理

#### 1.3.1 标签创建与计数

`addTopicByNameToRepo` 函数 (`models/repo/topic.go:101-129`)：
- 若标签不存在则创建，`RepoCount` 初始化为 1
- 若标签已存在则 `RepoCount++`
- 创建 `RepoTopic` 关联记录

#### 1.3.2 标签删除与计数

`removeTopicFromRepo` 函数 (`models/repo/topic.go:132-147`)：
- `RepoCount--`（不自动删除标签，由定期任务清理）
- 删除 `RepoTopic` 关联记录

#### 1.3.3 孤儿标签清理

```go
// 统计无关联仓库的标签
func CountOrphanedTopics(ctx context.Context) (int64, error) {
    return db.GetEngine(ctx).Where("repo_count = 0").Count(new(Topic))
}

// 删除无关联仓库的标签
func DeleteOrphanedTopics(ctx context.Context) (int64, error) {
    return db.GetEngine(ctx).Where("repo_count = 0").Delete(new(Topic))
}
```

### 1.4 数据同步机制

`syncTopicsInRepository` 函数 (`models/repo/topic.go:340-354`) 确保数据一致性：

```go
func syncTopicsInRepository(ctx context.Context, repoID int64) error {
    // 从关联表查询所有标签名称
    topicNames := make([]string, 0, 25)
    if err := db.GetEngine(ctx).Table("topic").Cols("name").
        Join("INNER", "repo_topic", "repo_topic.topic_id = topic.id").
        Where("repo_topic.repo_id = ?", repoID).Asc("topic.name").Find(&topicNames); err != nil {
        return err
    }

    // 更新 repository 表的 topics 冗余字段（JSON数组）
    if _, err := db.GetEngine(ctx).ID(repoID).Cols("topics").Update(&Repository{
        Topics: topicNames,
    }); err != nil {
        return err
    }
    return nil
}
```

**同步触发时机**：
- `AddTopic` → `syncTopicsInRepository`
- `DeleteTopic` → `syncTopicsInRepository`
- `SaveTopics` → `syncTopicsInRepository`
- `GenerateTopics`（模板仓库生成）→ `syncTopicsInRepository`

---

## 二、许可信息（Licenses）链路分析

### 2.1 数据模型定义

**核心模型**位于 `models/repo/license.go`：

```go
type RepoLicense struct {
    ID          int64 `xorm:"pk autoincr"`
    RepoID      int64 `xorm:"UNIQUE(s) NOT NULL"`     // 联合唯一索引
    CommitID    string                                 // 检测时的提交ID
    License     string `xorm:"VARCHAR(255) UNIQUE(s) NOT NULL"`  // 许可证标识
    CreatedUnix timeutil.TimeStamp `xorm:"INDEX CREATED"`
    UpdatedUnix timeutil.TimeStamp `xorm:"INDEX UPDATED"`
}
```

**设计特点**：
- `RepoID + License` 联合唯一索引
- 支持一个仓库多个许可证（双重许可等场景）
- 记录检测时的 `CommitID` 用于版本追踪

### 2.2 许可识别器实现

许可识别核心位于 `services/repository/license.go`，使用 **Google License Classifier v2** 库。

#### 2.2.1 分类器初始化

`InitLicenseClassifier` 函数 (`services/repository/license.go:39-56`)：

```go
func InitLicenseClassifier() error {
    // 阈值设置为 0.85（0.84~0.86之间，测试要求）
    classifier = licenseclassifier.NewClassifier(.85)
    
    // 加载内置许可证模板库
    licenseFiles, err := options.AssetFS().ListFiles("license", true)
    if err != nil {
        return err
    }

    for _, licenseFile := range licenseFiles {
        licenseName := licenseFile
        data, err := options.License(licenseFile)
        if err != nil {
            return err
        }
        // 将许可证文本添加到分类器
        classifier.AddContent("License", licenseName, licenseName, data)
    }
    return nil
}
```

**初始化时机**：系统启动时在 `routers/init.go:170` 调用：
```go
mustInit(repo_service.InitLicenseClassifier)
```

#### 2.2.2 许可检测流程

`detectLicense` 函数 (`services/repository/license.go:149-167`)：

```go
func detectLicense(r io.Reader) ([]string, error) {
    if r == nil {
        return nil, nil
    }

    // 调用分类器进行匹配
    matches, err := classifier.MatchFrom(r)
    if err != nil {
        return nil, err
    }
    
    if len(matches.Matches) > 0 {
        results := make(container.Set[string], len(matches.Matches))
        for _, r := range matches.Matches {
            // 只保留类型为"License"且去重
            if r.MatchType == "License" && !results.Contains(r.Variant) {
                results.Add(r.Variant)
            }
        }
        return results.Values(), nil
    }
    return nil, nil
}
```

**检测过滤条件**：
- `MatchType == "License"`：只保留许可证类型匹配
- 去重处理：使用 `container.Set[string]` 避免重复
- 阈值过滤：由分类器内部处理（阈值 0.85）

#### 2.2.3 内置许可证模板库

许可证模板存储在 `options/license/` 目录，通过 `options.License(name)` 读取：
```go
// modules/options/base.go:35-37
func License(name string) ([]byte, error) {
    return AssetFS().ReadFile("license", name)
}
```

#### 2.2.4 异步更新队列

为避免阻塞主流程，许可证更新通过异步队列处理：

```go
// services/repository/license.go:28-29
var licenseUpdaterQueue *queue.WorkerPoolQueue[*LicenseUpdaterOptions]

// services/repository/repository.go:100
licenseUpdaterQueue = queue.CreateUniqueQueue(
    graceful.GetManager().ShutdownContext(), 
    "repo_license_updater", 
    repoLicenseUpdater,
)
```

**队列触发场景**：
1. 仓库推送后：`services/repository/migrate.go:172`
2. 镜像同步后：`services/mirror/mirror_pull.go:420`
3. 定时同步：`services/repository/license.go:94-115` 的 `SyncRepoLicenses`

#### 2.2.5 检测执行流程

`UpdateRepoLicenses` 函数 (`services/repository/license.go:118-146`)：

```go
func UpdateRepoLicenses(ctx context.Context, repo *repo_model.Repository, commit *git.Commit) error {
    // 1. 获取 LICENSE 文件 blob
    b, err := commit.GetBlobByPath(LicenseFileName)
    if err != nil && !git.IsErrNotExist(err) {
        return fmt.Errorf("GetBlobByPath: %w", err)
    }

    // 2. 若 LICENSE 文件不存在，清除该仓库的许可记录
    if git.IsErrNotExist(err) {
        return repo_model.CleanRepoLicenses(ctx, repo)
    }

    // 3. 读取文件内容并检测
    licenses := make([]string, 0)
    if b != nil {
        r, err := b.DataAsync()
        if err != nil {
            return err
        }
        defer r.Close()

        licenses, err = detectLicense(r)
        if err != nil {
            return fmt.Errorf("detectLicense: %w", err)
        }
    }
    
    // 4. 更新数据库
    return repo_model.UpdateRepoLicenses(ctx, repo, commit.ID.String(), licenses)
}
```

### 2.3 创建仓库时许可证参数处理流程

创建仓库时，许可证参数从 API 层到数据库的完整链路：

#### 2.3.1 API 层参数接收

`CreateUserRepo` 函数 (`routers/api/v1/repo/repo.go:229-249`)：
```go
func CreateUserRepo(ctx *context.APIContext, owner *user_model.User, opt api.CreateRepoOption) {
    // ...
    repo, err := repo_service.CreateRepository(ctx, ctx.Doer, owner, repo_service.CreateRepoOptions{
        // ...
        License:          opt.License,  // 许可证参数传递
        // ...
    })
}
```

#### 2.3.2 服务层参数传递

`CreateRepoOptions` 结构 (`services/repository/create.go:38-57`)：
```go
type CreateRepoOptions struct {
    // ...
    License          string  // 用户选择的许可证
    AutoInit         bool    // 是否自动初始化
    // ...
}
```

#### 2.3.3 初始化提交写入 LICENSE 文件

`prepareRepoCommit` 函数 (`services/repository/create.go:126-141`)：
```go
// LICENSE
if len(opts.License) > 0 {
    // 1. 获取许可证模板并填充占位符
    data, err = repo_module.GetLicense(opts.License, &repo_module.LicenseValues{
        Owner: repo.OwnerName,      // 所有者名称
        Email: authorSig.Email,     // 邮箱
        Repo:  repo.Name,           // 仓库名称
        Year:  time.Now().Format("2006"),  // 当前年份
    })
    if err != nil {
        return fmt.Errorf("getLicense[%s]: %w", opts.License, err)
    }

    // 2. 写入 LICENSE 文件到临时目录
    if err = os.WriteFile(filepath.Join(tmpDir, "LICENSE"), data, 0o644); err != nil {
        return fmt.Errorf("write LICENSE: %w", err)
    }
}
```

**许可证模板占位符填充** (`modules/repository/license.go:23-56`)：
```go
func GetLicense(name string, values *LicenseValues) ([]byte, error) {
    data, err := options.License(name)
    if err != nil {
        return nil, fmt.Errorf("GetLicense[%s]: %w", name, err)
    }
    return fillLicensePlaceholder(name, values, data), nil
}
```

支持的占位符包括：
- `Owner`: `<name of author>`, `<owner>`, `[NAME]` 等
- `Email`: `[EMAIL]`
- `Repo`: `<program>` 等
- `Year`: `<year>`, `[YEAR]`, `{YEAR}` 等

#### 2.3.4 创建仓库时直接写入许可证记录

**关键**：创建仓库时不经过异步队列，直接同步写入数据库！

`CreateRepositoryDirectly` 函数 (`services/repository/create.go:311-325`)：
```go
// 6 - update licenses
var licenses []string
if len(opts.License) > 0 {
    licenses = append(licenses, opts.License)

    // 获取当前 HEAD commit ID
    var stdout string
    stdout, _, err = gitrepo.RunCmdString(ctx, repo, gitcmd.NewCommand("rev-parse", "HEAD"))
    if err != nil {
        log.Error("CreateRepository(git rev-parse HEAD) in %v: Stdout: %s\nError: %v", repo, stdout, err)
        return nil, fmt.Errorf("CreateRepository(git rev-parse HEAD): %w", err)
    }
    
    // 直接写入 repo_license 表（同步操作）
    if err = repo_model.UpdateRepoLicenses(ctx, repo, stdout, licenses); err != nil {
        return nil, err
    }
}
```

> **重要区别**：创建仓库时选择的许可证使用用户指定的 license name 直接写入，**不经过 Google License Classifier 检测**。这与后续推送代码时的自动检测流程不同。

---

### 2.4 数据库更新策略

`UpdateRepoLicenses` 模型函数 (`models/repo/license.go:47-90`)：

```go
func UpdateRepoLicenses(ctx context.Context, repo *Repository, commitID string, licenses []string) error {
    oldLicenses, err := GetRepoLicenses(ctx, repo)
    if err != nil {
        return err
    }
    
    // 更新或插入新许可证
    for _, license := range licenses {
        upd := false
        for _, o := range oldLicenses {
            if o.License == license {
                // 已存在则更新 CommitID
                o.CommitID = commitID
                if _, err := db.GetEngine(ctx).ID(o.ID).Cols("`commit_id`").Update(o); err != nil {
                    return err
                }
                upd = true
                break
            }
        }
        if !upd {
            // 不存在则插入新记录
            if err := db.Insert(ctx, &RepoLicense{
                RepoID:   repo.ID,
                CommitID: commitID,
                License:  license,
            }); err != nil {
                return err
            }
        }
    }
    
    // 删除不在新列表中的旧许可证（通过CommitID判断）
    licenseToDelete := make([]int64, 0, len(oldLicenses))
    for _, o := range oldLicenses {
        if o.CommitID != commitID {
            licenseToDelete = append(licenseToDelete, o.ID)
        }
    }
    if len(licenseToDelete) > 0 {
        if _, err := db.GetEngine(ctx).In("`id`", licenseToDelete).Delete(&RepoLicense{}); err != nil {
            return err
        }
    }
    return nil
}
```

---

## 三、搜索索引实现

### 3.1 搜索选项结构

`SearchRepoOptions` 位于 `models/repo/repo_list.go:154-213`：

```go
type SearchRepoOptions struct {
    db.ListOptions
    Actor           *user_model.User
    Keyword         string
    // ... 其他字段
    
    // 仅搜索话题名称
    TopicOnly bool
    // 按主语言过滤
    Language string
    // 关键词搜索包含描述
    IncludeDescription bool
    // ... 其他字段
}
```

### 3.2 话题搜索实现

`SearchRepositoryCondition` 函数 (`models/repo/repo_list.go:456-492`) 中的话题搜索逻辑：

```go
if opts.Keyword != "" {
    // 关键词按逗号分隔（支持多标签搜索）
    subQueryCond := builder.NewCond()
    for v := range strings.SplitSeq(opts.Keyword, ",") {
        if opts.TopicOnly {
            // 精确匹配
            subQueryCond = subQueryCond.Or(builder.Eq{"topic.name": strings.ToLower(v)})
        } else {
            // 模糊匹配
            subQueryCond = subQueryCond.Or(builder.Like{"topic.name", strings.ToLower(v)})
        }
    }
    
    // 子查询：通过关联表找到匹配的仓库ID
    subQuery := builder.Select("repo_topic.repo_id").From("repo_topic").
        Join("INNER", "topic", "topic.id = repo_topic.topic_id").
        Where(subQueryCond).
        GroupBy("repo_topic.repo_id")

    keywordCond := builder.In("id", subQuery)
    
    // 非 TopicOnly 模式下还匹配仓库名称和描述
    if !opts.TopicOnly {
        likes := builder.NewCond()
        for v := range strings.SplitSeq(opts.Keyword, ",") {
            likes = likes.Or(builder.Like{"lower_name", strings.ToLower(v)})
            // ... 描述匹配等
        }
        keywordCond = keywordCond.Or(likes)
    }
    cond = cond.And(keywordCond)
}
```

**搜索逻辑**：
1. **多标签支持**：关键词按逗号 `,` 分隔
2. **匹配模式**：
   - `TopicOnly=true`：精确匹配 `topic.name = keyword`
   - `TopicOnly=false`：模糊匹配 `topic.name LIKE %keyword%`
3. **子查询优化**：通过 `repo_topic` 和 `topic` 关联表查询匹配的仓库ID
4. **结果合并**：非 TopicOnly 模式下，结果合并仓库名称和描述匹配

### 3.3 "仅显示相关" 过滤器

`OnlyShowRelevant` 选项 (`models/repo/repo_list.go:533-554`)：

```go
if opts.OnlyShowRelevant {
    subQueryCond := builder.NewCond()
    
    // 检查是否有标签
    if setting.Database.Type.IsPostgreSQL() {
        subQueryCond = subQueryCond.Or(builder.And(builder.NotNull{"topics"}, builder.Neq{"(topics)::text": "[]"}))
    } else {
        subQueryCond = subQueryCond.Or(builder.And(builder.Neq{"topics": "null"}, builder.Neq{"topics": "[]"}))
    }
    
    // 检查是否有描述
    subQueryCond = subQueryCond.Or(builder.Neq{"description": ""})
    
    // 检查是否有头像
    subQueryCond = subQueryCond.Or(builder.Neq{"avatar": ""})
    
    // 排除空仓库
    subQueryCond = subQueryCond.And(builder.Eq{"is_empty": false})
    
    cond = cond.And(subQueryCond)
}
```

**注意**：PostgreSQL 和其他数据库使用不同的 JSON 字段检查语法。

### 3.4 话题搜索 API

独立的话题搜索接口 (`routers/api/v1/repo/topic.go:258-306`)：

```go
func TopicSearch(ctx *context.APIContext) {
    opts := &repo_model.FindTopicOptions{
        Keyword:     ctx.FormString("q"),
        ListOptions: utils.GetListOptions(ctx),
    }

    topics, total, err := db.FindAndCount[repo_model.Topic](ctx, opts)
    // ... 转换为 TopicResponse 并返回
}
```

`FindTopicOptions` 查询条件 (`models/repo/topic.go:175-186`)：
```go
func (opts *FindTopicOptions) ToConds() builder.Cond {
    cond := builder.NewCond()
    if opts.RepoID > 0 {
        cond = cond.And(builder.Eq{"repo_topic.repo_id": opts.RepoID})
    }
    if opts.Keyword != "" {
        cond = cond.And(builder.Like{"topic.name", opts.Keyword})
    }
    return cond
}
```

---

### 3.5 SearchRepository 结果组装时许可字段读取流程

搜索结果组装时，**Topics 字段从冗余字段直接读取，Licenses 字段需要额外查询数据库**。

#### 3.5.1 API 层搜索入口

`Search` 函数 (`routers/api/v1/repo/repo.go:46-227`)：
```go
func Search(ctx *context.APIContext) {
    // 1. 构建搜索选项
    opts := repo_model.SearchRepoOptions{
        ListOptions:        utils.GetListOptions(ctx),
        Actor:              ctx.Doer,
        Keyword:            ctx.FormTrim("q"),
        TopicOnly:          ctx.FormBool("topic"),  // 仅按话题搜索
        // ... 其他选项
    }

    // 2. 执行搜索（仅查询 repository 表）
    repos, count, err := repo_model.SearchRepository(ctx, opts)

    // 3. 结果组装（转换为 API 格式）
    results := make([]*api.Repository, len(repos))
    for i, repo := range repos {
        if err = repo.LoadOwner(ctx); err != nil {
            // ... 错误处理
        }
        permission, err := access_model.GetDoerRepoPermission(ctx, repo, ctx.Doer)
        // 关键：调用 ToRepo 进行转换
        results[i] = convert.ToRepo(ctx, repo, permission)
    }
    // ...
}
```

#### 3.5.2 ToRepo 转换中的许可字段读取

`ToRepo` 函数 (`services/convert/repository.go:188-264`)：
```go
func innerToRepo(ctx context.Context, repo *repo_model.Repository, permissionInRepo access_model.Permission, isParent bool) *api.Repository {
    // ... 其他字段处理

    // Topics：直接从 repository 表的 JSON 冗余字段读取（性能优化）
    // 位置: services/convert/repository.go:262
    Topics: util.SliceNilAsEmpty(repo.Topics),

    // Licenses：需要额外查询 repo_license 表（N+1 查询问题）
    // 位置: services/convert/repository.go:188-191
    repoLicenses, err := repo_model.GetRepoLicenses(ctx, repo)
    if err != nil {
        return nil
    }
    // ...
    // 位置: services/convert/repository.go:264
    Licenses: util.SliceNilAsEmpty(repoLicenses.StringList()),
}
```

**性能对比**：
| 字段 | 读取方式 | 性能特点 |
|------|---------|---------|
| Topics | `repo.Topics`（JSON 字段） | 单次查询，高性能 |
| Licenses | `repo_model.GetRepoLicenses(ctx, repo)`（额外查询） | 每个结果一次查询，N+1 性能问题 |

---

### 3.6 许可字段未参与仓库搜索过滤的原因分析

**代码入口**：`models/repo/repo_list.go:154-213` (`SearchRepoOptions` 结构定义)

#### 3.6.1 现状：无 License 过滤选项

`SearchRepoOptions` 结构中缺少 License 相关字段：
```go
type SearchRepoOptions struct {
    // ... 现有字段
    TopicOnly bool          // ✅ 支持按话题过滤
    Language string         // ✅ 支持按语言过滤
    // License 字段缺失 ❌
}
```

`SearchRepositoryCondition` 函数（`models/repo/repo_list.go:369-557`）中：
- ✅ 有话题搜索逻辑（`TopicOnly` 模式）
- ✅ 有语言过滤逻辑（`Language` 字段）
- ❌ 无任何 License 相关过滤逻辑

#### 3.6.2 根本原因分析

**原因 1：许可证识别的异步性**
```
代码推送 → 异步队列（licenseUpdaterQueue）→ Google Classifier 检测 → 写入 repo_license 表
```
- 许可证识别是**异步**的，不是实时的
- 搜索时许可证信息可能尚未更新
- 基于异步数据的过滤可能导致结果不一致

**原因 2：数据存储结构的性能问题**

- **Topics**：`repository.topics` JSON 冗余字段 → 可直接过滤
- **Licenses**：存储在独立 `repo_license` 表 → 需要 JOIN 查询

如果要实现 License 过滤，SQL 会是这样：
```sql
SELECT * FROM repository
INNER JOIN repo_license ON repository.id = repo_license.repo_id
WHERE repo_license.license = 'MIT'
```
- JOIN 查询影响搜索性能
- 多许可证场景需要 DISTINCT 或 GROUP BY，进一步降低性能

**原因 3：缺少冗余字段设计**

- Repository 表中没有 `licenses` JSON 冗余字段
- 每次查询都需要 JOIN repo_license 表
- 为了性能而未实现（话题有冗余字段，许可没有）

#### 3.6.3 代码位置总结

| 检查点 | 文件路径 | 行号 | 状态 |
|--------|---------|------|------|
| SearchRepoOptions 结构 | `models/repo/repo_list.go` | 154-213 | 无 License 字段 |
| SearchRepositoryCondition 函数 | `models/repo/repo_list.go` | 369-557 | 无 License 过滤逻辑 |
| API Search 函数参数 | `routers/api/v1/repo/repo.go` | 46-134 | 无 license 查询参数 |

---

## 四、对外接口暴露

### 4.1 RESTful API 接口

API 路由定义位于 `routers/api/v1/api.go`：

#### 4.1.1 话题标签接口

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| `GET` | `/repos/{owner}/{repo}/topics` | `repo.ListTopics` | 获取仓库标签列表 |
| `PUT` | `/repos/{owner}/{repo}/topics` | `repo.UpdateTopics` | 替换仓库所有标签 |
| `PUT` | `/repos/{owner}/{repo}/topics/{topic}` | `repo.AddTopic` | 添加单个标签 |
| `DELETE` | `/repos/{owner}/{repo}/topics/{topic}` | `repo.DeleteTopic` | 删除单个标签 |
| `GET` | `/topics/search` | `repo.TopicSearch` | 搜索话题（全局） |

**接口详情**：

1. **ListTopics** (`routers/api/v1/repo/topic.go:21-72`)
   - 支持分页（`page`, `limit`）
   - 返回：`{"topics": ["topic1", "topic2"], ...}`
   - Header：`X-Total-Count` 总数

2. **UpdateTopics** (`routers/api/v1/repo/topic.go:75-132`)
   - 请求体：`{"topics": ["topic1", "topic2"]}`
   - 先调用 `SanitizeAndValidateTopics` 验证
   - 超过25个返回 422 错误
   - 无效标签返回 422 错误及无效列表

3. **AddTopic** (`routers/api/v1/repo/topic.go:135-199`)
   - 路径参数 `topic` 自动 `TrimSpace` + `ToLower`
   - 验证标签格式
   - 检查数量限制（≤25）

4. **DeleteTopic** (`routers/api/v1/repo/topic.go:202-255`)
   - 路径参数 `topic` 自动规范化
   - 标签不存在返回 404

5. **TopicSearch** (`routers/api/v1/repo/topic.go:258-306`)
   - 查询参数 `q` 关键词
   - 返回 `TopicResponse` 数组，包含 ID、名称、仓库计数等

#### 4.1.2 许可信息接口

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| `GET` | `/repos/{owner}/{repo}/licenses` | `repo.GetLicenses` | 获取仓库许可证列表 |
| `GET` | `/licenses` | `misc.ListLicenseTemplates` | 获取许可证模板列表 |
| `GET` | `/licenses/{name}` | `misc.GetLicenseTemplateInfo` | 获取许可证模板详情 |

**GetLicenses** 实现 (`routers/api/v1/repo/license.go:15-51`)：
```go
func GetLicenses(ctx *context.APIContext) {
    licenses, err := repo_model.GetRepoLicenses(ctx, ctx.Repo.Repository)
    // ...
    resp := make([]string, len(licenses))
    for i := range licenses {
        resp[i] = licenses[i].License
    }
    ctx.JSON(http.StatusOK, resp)
}
```

### 4.2 Web 界面接口

#### 4.2.1 话题标签 Web 接口

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| `POST` | `/repo/{owner}/{repo}/topics` | `repo.TopicsPost` | Web端更新标签 |
| `GET` | `/explore/topics/search` | `explore.TopicSearch` | 探索页面标签搜索 |

**TopicsPost** (`routers/web/repo/topic.go:16-60`)：
- 表单参数 `topics`（逗号分隔字符串）
- 返回 JSON 格式响应

#### 4.2.2 仓库主页展示

`prepareHomeSidebarRepoTopics` 和 `prepareHomeSidebarLicenses` 函数（`routers/web/repo/view_home.go:58-134`）：

```go
// 加载仓库标签供侧边栏显示
func prepareHomeSidebarRepoTopics(ctx *context.Context) {
    topics, err := db.Find[repo_model.Topic](ctx, &repo_model.FindTopicOptions{
        RepoID: ctx.Repo.Repository.ID,
    })
    ctx.Data["Topics"] = topics
}

// 加载检测到的许可证供侧边栏显示
func prepareHomeSidebarLicenses(ctx *context.Context) {
    repoLicenses, err := repo_model.GetRepoLicenses(ctx, ctx.Repo.Repository)
    ctx.Data["DetectedRepoLicenses"] = repoLicenses.StringList()
    ctx.Data["LicenseFileName"] = repo_service.LicenseFileName
}
```

### 4.3 数据转换层（Convert）

#### 4.3.1 Topic 转换

`ToTopicResponse` (`services/convert/convert.go:819-827`)：
```go
func ToTopicResponse(topic *repo_model.Topic) *api.TopicResponse {
    return &api.TopicResponse{
        ID:        topic.ID,
        Name:      topic.Name,
        RepoCount: topic.RepoCount,
        Created:   topic.CreatedUnix.AsTime(),
        Updated:   topic.UpdatedUnix.AsTime(),
    }
}
```

#### 4.3.2 Repository 转换

`ToRepo` 函数 (`services/convert/repository.go:262-264`) 中包含标签和许可：
```go
return &api.Repository{
    // ... 其他字段
    Topics:   util.SliceNilAsEmpty(repo.Topics),      // 从冗余字段读取
    Licenses: util.SliceNilAsEmpty(repoLicenses.StringList()),  // 查询获取
    // ...
}
```

**注意**：
- `Topics` 从 `repository.topics` JSON 字段直接读取（性能优化）
- `Licenses` 需额外查询 `repo_license` 表（`repo_model.GetRepoLicenses`）

### 4.4 API 结构体定义

#### 4.4.1 话题相关结构体（`modules/structs/repo_topic.go`）

```go
type TopicResponse struct {
    ID        int64     `json:"id"`
    Name      string    `json:"topic_name"`
    RepoCount int       `json:"repo_count"`
    Created   time.Time `json:"created"`
    Updated   time.Time `json:"updated"`
}

type TopicName struct {
    TopicNames []string `json:"topics"`
}

type RepoTopicOptions struct {
    Topics []string `json:"topics"`
}
```

#### 4.4.2 仓库结构体中的字段（`modules/structs/repo.go:134-135`）

```go
type Repository struct {
    // ... 其他字段
    Topics   []string `json:"topics"`
    Licenses []string `json:"licenses"`
    // ...
}
```

#### 4.4.3 创建仓库时的许可选项（`modules/structs/repo.go:159`）

```go
type CreateRepoOption struct {
    // ...
    License string `json:"license" binding:"MaxSize(100)"`  // 创建仓库时指定初始LICENSE
    // ...
}
```

---

## 五、完整调用链路总结

### 5.1 话题标签完整链路

```
用户输入
    ↓
[Web/API层]
    ├─ Web: routers/web/repo/topic.go TopicsPost
    └─ API: routers/api/v1/repo/topic.go UpdateTopics/AddTopic
    ↓
[规范化层] models/repo/topic.go
    ├─ SanitizeAndValidateTopics
    │   ├─ strings.TrimSpace()
    │   ├─ strings.ToLower()
    │   ├─ 空值过滤
    │   └─ 去重处理
    └─ ValidateTopic
        └─ 正则匹配 + 长度检查
    ↓
[数据层] models/repo/topic.go
    ├─ SaveTopics / AddTopic
    │   ├─ addTopicByNameToRepo (创建/更新 Topic，RepoCount++)
    │   ├─ 创建 RepoTopic 关联
    │   └─ syncTopicsInRepository (同步 repository.topics 字段)
    └─ DeleteTopic
        ├─ removeTopicFromRepo (RepoCount--，删除 RepoTopic)
        └─ syncTopicsInRepository
    ↓
[搜索层] models/repo/repo_list.go
    └─ SearchRepositoryCondition
        ├─ TopicOnly 精确匹配
        └─ 模糊匹配 topic.name
    ↓
[展示层]
    ├─ API: ToRepo() → topics 字段
    └─ Web: prepareHomeSidebarRepoTopics → 侧边栏展示
```

### 5.2 许可信息完整链路

许可信息有两条独立的进入路径：**创建仓库时的同步写入** 和 **代码推送后的异步检测**。

#### 5.2.1 创建仓库时的同步写入链路

```
用户创建仓库（选择许可证）
    ↓
[API层] routers/api/v1/repo/repo.go CreateUserRepo
    └─ 接收 api.CreateRepoOption.License 参数
    ↓
[服务层] services/repository/create.go CreateRepositoryDirectly
    ├─ prepareRepoCommit
    │   ├─ repo_module.GetLicense(licenseName)  // 获取模板+填充占位符
    │   │   └─ fillLicensePlaceholder(Owner/Email/Repo/Year)
    │   └─ os.WriteFile("LICENSE")  // 写入临时目录
    ├─ initRepoCommit  // 执行初始化提交
    └─ 步骤6：同步写入 repo_license 表
        ├─ git rev-parse HEAD  // 获取提交ID
        └─ repo_model.UpdateRepoLicenses(ctx, repo, commitID, [license])
            └─ 直接写入，不经过 Classifier 检测！
```

#### 5.2.2 代码推送后的异步检测链路

```
系统启动
    ↓
[初始化] routers/init.go
    └─ repo_service.InitLicenseClassifier
        └─ licenseclassifier.NewClassifier(.85) + 加载内置许可证模板
    ↓
[触发检测]
    ├─ 代码推送
    ├─ 镜像同步
    └─ 定时任务 SyncRepoLicenses
    ↓
[异步队列] services/repository/license.go
    └─ licenseUpdaterQueue → repoLicenseUpdater
        ↓
[检测层] services/repository/license.go
    ├─ UpdateRepoLicenses
    │   ├─ GetBlobByPath("LICENSE")  // 读取 Git 仓库文件
    │   ├─ detectLicense (调用 google classifier)
    │   └─ repo_model.UpdateRepoLicenses
    └─ detectLicense
        └─ classifier.MatchFrom(r) → 过滤 MatchType="License" + 去重
    ↓
[数据层] models/repo/license.go
    └─ UpdateRepoLicenses
        ├─ 更新已有记录的 CommitID
        ├─ 插入新许可证
        └─ 删除过时许可证（CommitID 不匹配）
    ↓
[展示层]
    ├─ API: GET /repos/{owner}/{repo}/licenses
    ├─ API: ToRepo() → licenses 字段（需额外查询）
    └─ Web: prepareHomeSidebarLicenses → 侧边栏展示
```

> **重要区别**：创建仓库时使用用户指定的 license name 直接写入；推送后使用 Google Classifier 自动检测。两者可能不一致！

---

## 六、关键技术点总结

| 技术点 | 话题标签 | 许可信息 |
|--------|---------|---------|
| **验证规则** | 正则 `^[a-z0-9][-.a-z0-9]*$`，长度≤35 | 无需用户输入，自动检测 |
| **归一化策略** | 小写转换、去空、去重 | 自动检测，无需归一化 |
| **识别技术** | 规则匹配（正则） | Google License Classifier v2，阈值0.85 |
| **数据存储** | Topic + RepoTopic（多对多）+ repository.topics 冗余 | RepoLicense（联合唯一索引） |
| **数据冗余字段** | `repository.topics`（JSON 数组） | 无冗余字段 |
| **更新方式** | 同步更新 | 创建仓库：同步写入；代码推送：异步队列 |
| **搜索过滤** | 支持 TopicOnly 精确匹配 + 模糊匹配 | 不支持搜索过滤 |
| **搜索结果读取** | 直接读取 repository.topics 字段（高性能） | 额外查询 repo_license 表（N+1 问题） |
| **未实现搜索的原因** | - | 1. 异步识别非实时 2. 无冗余字段需 JOIN 3. 性能考虑 |
| **数量限制** | 每仓库最多25个 | 无限制（支持多许可） |
| **API 接口** | 5个（CRUD + 搜索） | 3个（查询 + 模板） |
| **性能优化** | topics JSON 冗余字段 | 异步队列处理 |
| **创建时写入** | SaveTopics 同步写入 | 直接写入 repo_license 表（绕过 Classifier） |
| **占位符填充** | 不适用 | Owner/Email/Repo/Year 自动替换 |

---

### 6.1 两个重要的设计决策对比

| 设计决策 | 话题标签 | 许可信息 |
|---------|---------|---------|
| **冗余字段设计** | ✅ 有（repository.topics） | ❌ 无 |
| **搜索过滤支持** | ✅ 支持 | ❌ 不支持 |
| **创建时检测** | ✅ 同步验证 | ✅ 同步写入（用户指定） |
| **推送后检测** | 不适用 | ✅ 异步自动检测 |
| **性能权衡** | 以存储空间换查询性能 | 以查询性能（N+1）换取实现简单 |

---

## 七、相关文件索引

| 模块 | 文件路径 | 关键函数/结构 |
|------|---------|--------------|
| 话题模型 | `models/repo/topic.go` | `Topic`, `RepoTopic`, `SanitizeAndValidateTopics`, `SaveTopics`, `syncTopicsInRepository` |
| 许可模型 | `models/repo/license.go` | `RepoLicense`, `UpdateRepoLicenses`, `GetRepoLicenses` |
| 话题API | `routers/api/v1/repo/topic.go` | `ListTopics`, `UpdateTopics`, `AddTopic`, `DeleteTopic`, `TopicSearch` |
| 许可API | `routers/api/v1/repo/license.go` | `GetLicenses` |
| 仓库搜索API | `routers/api/v1/repo/repo.go` | `Search`, `CreateUserRepo` |
| 话题Web | `routers/web/repo/topic.go` | `TopicsPost` |
| 话题搜索Web | `routers/web/explore/topic.go` | `TopicSearch` |
| 许可检测服务 | `services/repository/license.go` | `InitLicenseClassifier`, `detectLicense`, `UpdateRepoLicenses`, `SyncRepoLicenses` |
| 仓库创建服务 | `services/repository/create.go` | `CreateRepositoryDirectly`, `prepareRepoCommit`, `initRepository` |
| 许可模板处理 | `modules/repository/license.go` | `GetLicense`, `fillLicensePlaceholder`, `LicenseValues` |
| 仓库主页展示 | `routers/web/repo/view_home.go` | `prepareHomeSidebarRepoTopics`, `prepareHomeSidebarLicenses` |
| 搜索逻辑 | `models/repo/repo_list.go` | `SearchRepositoryCondition`, `SearchRepoOptions` |
| 数据转换 | `services/convert/repository.go` | `ToRepo`, `innerToRepo` |
| 话题转换 | `services/convert/convert.go` | `ToTopicResponse` |
| API结构体 | `modules/structs/repo_topic.go` | `TopicResponse`, `RepoTopicOptions` |
| 仓库结构体 | `modules/structs/repo.go` | `Repository.Topics`, `Repository.Licenses`, `CreateRepoOption.License` |
| 许可模板 | `modules/options/base.go` | `License()` |
| 路由注册 | `routers/api/v1/api.go` | 话题和许可API路由定义 |
| 系统初始化 | `routers/init.go` | `InitLicenseClassifier` 调用 |
| 队列初始化 | `services/repository/repository.go` | `licenseUpdaterQueue` 创建 |
