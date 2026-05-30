# Wiki 分支变更与历史回溯代码分析

## 一、数据库事务与 Git 分支重命名失败的边界情况

### 1.1 核心代码分析

**代码位置**: `services/wiki/wiki.go:382-410`

```go
func ChangeDefaultWikiBranch(ctx context.Context, repo *repo_model.Repository, newBranch string) error {
    if !git.IsValidRefPattern(newBranch) {
        return fmt.Errorf("invalid branch name: %s", newBranch)
    }
    return db.WithTx(ctx, func(ctx context.Context) error {
        // 步骤1: 更新数据库中的 default_wiki_branch
        repo.DefaultWikiBranch = newBranch
        if err := repo_model.UpdateRepositoryColsNoAutoTime(ctx, repo, "default_wiki_branch"); err != nil {
            return fmt.Errorf("unable to update database: %w", err)
        }

        if !repo_service.HasWiki(ctx, repo) {
            return nil
        }

        // 步骤2: 获取当前 Git 默认分支
        oldDefBranch, err := gitrepo.GetDefaultBranch(ctx, repo.WikiStorageRepo())
        if err != nil {
            return fmt.Errorf("unable to get default branch: %w", err)
        }
        if oldDefBranch == newBranch {
            return nil
        }

        // 步骤3: 执行 Git 分支重命名
        err = gitrepo.RenameBranch(ctx, repo.WikiStorageRepo(), oldDefBranch, newBranch)
        if err != nil {
            return fmt.Errorf("unable to rename default branch: %w", err)
        }
        return nil
    })
}
```

### 1.2 数据库事务机制

**代码位置**: `models/db/context.go:143-176`

```go
func WithTx(parentCtx context.Context, f func(ctx context.Context) error) error {
    if sess := getTransactionSession(parentCtx); sess != nil {
        err := f(withContextEngine(parentCtx, sess))
        if err != nil {
            _ = sess.Close()  // 回滚事务
        }
        return err
    }
    return txWithNoCheck(parentCtx, f)
}

func txWithNoCheck(parentCtx context.Context, f func(ctx context.Context) error) error {
    sess := xormEngine.NewSession()
    defer sess.Close()
    if err := sess.Begin(); err != nil {
        return err
    }

    if err := f(withContextEngine(parentCtx, sess)); err != nil {
        return err  // 错误返回时, defer sess.Close() 会触发回滚
    }

    return sess.Commit()
}
```

### 1.3 边界情况分析

⚠️ **关键问题**: Git 操作是外部系统调用，不在数据库事务控制范围内。

| 执行顺序 | 操作 | 成功/失败 | 结果 |
|----------|------|-----------|------|
| 1 | 数据库更新 default_wiki_branch | ✅ 成功 | 数据库变更暂存于事务 |
| 2 | GetDefaultBranch | ✅ 成功 | - |
| 3 | RenameBranch | ❌ 失败 | 返回错误，事务回滚 |

**边界情况 1: Git 重命名失败，数据库事务正确回滚**
- 场景：`RenameBranch` 执行失败（如目标分支已存在、权限不足等）
- 结果：`WithTx` 捕获错误，`sess.Close()` 触发回滚
- 结论：**此场景数据一致**，数据库和 Git 均未变更

---

| 执行顺序 | 操作 | 成功/失败 | 结果 |
|----------|------|-----------|------|
| 1 | 数据库更新 default_wiki_branch | ✅ 成功 | 数据库变更暂存于事务 |
| 2 | GetDefaultBranch | ✅ 成功 | - |
| 3 | RenameBranch | ✅ 成功 | Git 分支已重命名 |
| 4 | return nil | - | 事务提交 |

**边界情况 2: Git 重命名成功，事务提交成功**
- 场景：所有操作正常完成
- 结果：`sess.Commit()` 提交事务，数据库和 Git 均更新
- 结论：**此场景数据一致**

---

| 执行顺序 | 操作 | 成功/失败 | 结果 |
|----------|------|-----------|------|
| 1 | 数据库更新 default_wiki_branch | ✅ 成功 | 数据库变更暂存于事务 |
| 2 | GetDefaultBranch | ✅ 成功 | - |
| 3 | RenameBranch | ✅ 成功 | **Git 分支已物理重命名** |
| 4 | defer 或其他代码 | ❌ Panic/进程崩溃 | 事务未提交，自动回滚 |

⚠️ **边界情况 3: Git 重命名成功，但事务未提交（严重不一致）**
- 场景：Git 重命名成功后，进程崩溃或发生 panic
- 结果：
  - Git 分支已实际重命名（不可逆，已执行的系统调用无法回滚）
  - 数据库事务因会话关闭而回滚（`default_wiki_branch` 仍为旧值）
  - **数据库与 Git 出现严重不一致**

### 1.4 不一致后的自动恢复

**代码位置**: `routers/web/repo/wiki.go:98-125`

```go
func findWikiRepoCommit(ctx *context.Context) (*git.Repository, *git.Commit, error) {
    wikiGitRepo, errGitRepo := gitrepo.RepositoryFromRequestContextOrOpen(...)
    
    // 尝试按数据库记录的 default_wiki_branch 获取 commit
    commit, errCommit := wikiGitRepo.GetBranchCommit(ctx.Repo.Repository.DefaultWikiBranch)
    
    if git.IsErrNotExist(errCommit) {
        // 数据库记录的分支不存在，重新同步
        gitRepoDefaultBranch, errBranch := gitrepo.GetDefaultBranch(ctx, ctx.Repo.Repository.WikiStorageRepo())
        
        // 用 Git 实际的默认分支更新数据库
        errDb := repo_model.UpdateRepositoryColsNoAutoTime(ctx, 
            &repo_model.Repository{
                ID: ctx.Repo.Repository.ID, 
                DefaultWikiBranch: gitRepoDefaultBranch
            }, 
            "default_wiki_branch")
        
        ctx.Repo.Repository.DefaultWikiBranch = gitRepoDefaultBranch
        
        // 重试获取 commit
        commit, errCommit = wikiGitRepo.GetBranchCommit(ctx.Repo.Repository.DefaultWikiBranch)
    }
    return wikiGitRepo, commit, nil
}
```

**恢复机制**:
- 当数据库记录的 `default_wiki_branch` 在 Git 中不存在时
- 自动从 Git 读取实际默认分支并更新数据库
- 这是一个"最终一致"的恢复机制，但只在访问 Wiki 时触发

---

## 二、findEntryForFile 函数中 QueryUnescape 回退机制

### 2.1 核心代码分析

**代码位置**: `routers/web/repo/wiki.go:80-96`

```go
// findEntryForFile finds the tree entry for a target filepath.
func findEntryForFile(commit *git.Commit, target string) (*git.TreeEntry, error) {
    // 第一步: 尝试原始路径
    entry, err := commit.GetTreeEntryByPath(target)
    if err != nil && !git.IsErrNotExist(err) {
        return nil, err
    }
    if entry != nil {
        return entry, nil
    }

    // 第二步: 回退到 QueryUnescape 后的路径
    var unescapedTarget string
    if unescapedTarget, err = url.QueryUnescape(target); err != nil {
        return nil, err
    }
    return commit.GetTreeEntryByPath(unescapedTarget)
}
```

### 2.2 %2F 与实际斜杠的歧义问题

**URL 编码背景**:
- `%2F` 是 `/` 字符的 URL 编码形式
- 在 Git 树中，`/` 表示目录分隔符
- 但 Wiki 文件名可能字面包含 `%2F` 字符（历史遗留问题）

**歧义场景**:

| Git 中实际文件名 | URL 请求路径 | 第一次查找 | 第二次查找（QueryUnescape 后） | 结果 |
|-----------------|-------------|-----------|-------------------------------|------|
| `foo%2Fbar.md`（字面 %2F） | `/wiki/foo%2Fbar` | `foo%2Fbar.md` ✅ 找到 | 不执行 | ✅ 正确 |
| `foo/bar.md`（子目录） | `/wiki/foo%2Fbar` | `foo%2Fbar.md` ❌ 未找到 | `foo/bar.md` ✅ 找到 | ✅ 正确 |
| `foo%2Fbar.md`（字面 %2F） | `/wiki/foo%252Fbar` | `foo%252Fbar.md` ❌ 未找到 | `foo%2Fbar.md` ✅ 找到 | ✅ 正确 |
| `foo/bar.md`（子目录） 和 `foo%2Fbar.md`（字面 %2F） 同时存在 | `/wiki/foo%252Fbar` | `foo%252Fbar.md` ❌ 未找到 | `foo%2Fbar.md` ✅ 找到 | ⚠️ 有歧义但可接受 |

⚠️ **真正的问题场景**:

| 场景 | Git 文件 | URL 路径 | 第一次查找 | 第二次查找 | 结果 |
|------|---------|---------|-----------|-----------|------|
| 只有 `foo/bar.md` | `foo/bar.md` | `/wiki/foo/bar` | `foo/bar.md` ✅ 找到 | 不执行 | ✅ 正确 |
| 只有 `foo%2Fbar.md` | `foo%2Fbar.md` | `/wiki/foo/bar` | `foo/bar.md` ❌ 未找到 | `foo/bar.md` ❌ 未找到 | ✅ 正确（确实不存在） |
| 两个文件都存在 | `foo/bar.md` 和 `foo%2Fbar.md` | `/wiki/foo%2Fbar` | `foo%2Fbar.md` ✅ 先找到 | 不执行 | ⚠️ **优先级问题** |

### 2.3 优先级问题分析

**问题描述**:
- 当 Git 中同时存在 `foo%2Fbar.md`（字面 %2F）和 `foo/bar.md`（子目录）时
- 访问 `/wiki/foo%2Fbar` 会优先匹配到 `foo%2Fbar.md`
- 但用户可能实际想访问的是 `foo/bar.md`（子目录中的 bar.md）

**代码优先级逻辑**:
1. 先尝试原始路径 `foo%2Fbar.md` → 如果存在，直接返回
2. 只有不存在时，才尝试解码后的路径 `foo/bar.md`

**历史背景**（代码注释说明）:
- 旧版 Wiki 代码总是使用 `%2F` 而不是子目录
- 导致用户 Wiki 中存在大量遗留的 `%2F` 字面值文件
- 回退机制是为了兼容这些历史文件

---

## 三、页面命中路径与历史统计参数不一致问题

### 3.1 代码调用链分析

#### 3.1.1 页面命中路径获取

**代码位置**: `routers/web/repo/wiki.go:144-169`

```go
func wikiEntryByName(ctx *context.Context, commit *git.Commit, wikiName wiki_service.WebPath) (*git.TreeEntry, string, bool, bool) {
    isRaw := false
    // 转换 WebPath 为 Git 路径
    gitFilename := wiki_service.WebPathToGitPath(wikiName)
    
    // ⚠️ 这里可能通过 QueryUnescape 回退找到不同的文件
    entry, err := findEntryForFile(commit, gitFilename)
    
    if entry == nil {
        // 尝试无 .md 后缀（raw 文件）
        gitFilename := strings.TrimSuffix(gitFilename, ".md")
        entry, err = findEntryForFile(commit, gitFilename)
        isRaw = true
    }
    
    // 返回实际找到的 entry 和 gitFilename
    return entry, gitFilename, false, isRaw
}
```

#### 3.1.2 历史统计参数传递

**代码位置**: `routers/web/repo/wiki.go:309-311`

```go
// 获取页面内容时返回 pageFilename
_, entry, pageFilename, noEntry := wikiContentsByName(ctx, commit, pageName)

// ...

// 使用 pageFilename 统计提交历史
commitsCount, _ := gitrepo.FileCommitsCount(ctx, 
    ctx.Repo.Repository.WikiStorageRepo(), 
    ctx.Repo.Repository.DefaultWikiBranch, 
    pageFilename)  // ⚠️ 这里使用的 pageFilename 可能与实际找到的路径不一致
```

### 3.2 不一致的具体情形

#### 情形 1: QueryUnescape 回退导致路径不一致

| 步骤 | 变量 | 值 | 说明 |
|------|------|----|------|
| 1 | 传入 wikiName | `foo%252Fbar`（URL 编码的 `foo%2Fbar`） | - |
| 2 | `WebPathToGitPath(wikiName)` | `foo%252Fbar.md` | 编码后文件名 |
| 3 | 第一次 `findEntryForFile` | `foo%252Fbar.md` ❌ 未找到 | Git 中实际是 `foo%2Fbar.md` |
| 4 | 第二次 `findEntryForFile(QueryUnescape)` | `foo%2Fbar.md` ✅ 找到 | 解码后匹配成功 |
| 5 | 返回的 pageFilename | `foo%252Fbar.md` | ⚠️ 仍然是原始转换值！ |
| 6 | `FileCommitsCount` 参数 | `foo%252Fbar.md` | ❌ Git 中不存在此路径 |

**结果**: 页面内容正确显示，但历史统计为 0 或错误。

#### 情形 2: 去掉 .md 后缀导致路径不一致

| 步骤 | 变量 | 值 | 说明 |
|------|------|----|------|
| 1 | 传入 wikiName | `image.png` | - |
| 2 | `WebPathToGitPath(wikiName)` | `image.png.md` | 自动加 .md 后缀 |
| 3 | 第一次 `findEntryForFile` | `image.png.md` ❌ 未找到 | - |
| 4 | TrimSuffix 后查找 | `image.png` ✅ 找到 | 实际是 raw 文件 |
| 5 | 返回的 pageFilename | `image.png` | ✅ 这次是正确的（因为重新赋值了） |
| 6 | `FileCommitsCount` 参数 | `image.png` | ✅ 正确 |

**注意**: 情形 2 没有问题，因为代码在 TrimSuffix 后重新赋值了 `gitFilename`。

#### 情形 3: QueryUnescape + 无 .md 后缀双重回退

| 步骤 | 变量 | 值 | 说明 |
|------|------|----|------|
| 1 | 传入 wikiName | `foo%252Fimage.png` | - |
| 2 | `WebPathToGitPath(wikiName)` | `foo%252Fimage.png.md` | - |
| 3 | 第一次 findEntryForFile | `foo%252Fimage.png.md` ❌ | - |
| 4 | QueryUnescape 回退 | `foo%2Fimage.png.md` ❌ | - |
| 5 | TrimSuffix 后 | `foo%252Fimage.png` | - |
| 6 | 再次 findEntryForFile | `foo%252Fimage.png` ❌ | - |
| 7 | QueryUnescape 回退 | `foo%2Fimage.png` ✅ 找到 | - |
| 8 | 返回的 pageFilename | `foo%252Fimage.png` | ⚠️ 不是实际找到的路径！ |
| 9 | FileCommitsCount 参数 | `foo%252Fimage.png` | ❌ 错误路径 |

### 3.3 根因分析

`wikiEntryByName` 函数的返回值设计问题：

```go
func wikiEntryByName(...) (*git.TreeEntry, string, bool, bool) {
    gitFilename := wiki_service.WebPathToGitPath(wikiName)
    entry, err := findEntryForFile(commit, gitFilename)
    
    if entry == nil {
        gitFilename := strings.TrimSuffix(gitFilename, ".md")  // 重新赋值了局部变量
        entry, err = findEntryForFile(commit, gitFilename)
    }
    
    // 返回的 gitFilename 可能与 entry 实际路径不一致
    return entry, gitFilename, false, isRaw
}
```

⚠️ **关键 Bug**: `findEntryForFile` 可能通过 QueryUnescape 回退找到文件，但返回的 `gitFilename` 仍然是原始值，而不是实际找到的路径。

**对比**: 去掉 `.md` 后缀时重新赋值了 `gitFilename`（`gitFilename := strings.TrimSuffix(...)`），但 QueryUnescape 回退时没有更新 `gitFilename`。

---

## 四、完整代码溯源索引

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| Wiki 分支重命名 | `services/wiki/wiki.go` | 382-410 |
| 数据库事务 WithTx | `models/db/context.go` | 143-176 |
| Git 分支重命名 | `modules/gitrepo/branch.go` | 93-96 |
| findWikiRepoCommit 自动恢复 | `routers/web/repo/wiki.go` | 98-125 |
| findEntryForFile 回退机制 | `routers/web/repo/wiki.go` | 80-96 |
| wikiEntryByName 路径返回 | `routers/web/repo/wiki.go` | 144-169 |
| FileCommitsCount 统计 | `modules/gitrepo/commit.go` | 48-54 |
| WebPathToGitPath 转换 | `services/wiki/wiki_path.go` | 96-111 |
| WebPathFromRequest 转换 | `services/wiki/wiki_path.go` | 144-149 |
| 页面渲染与历史统计 | `routers/web/repo/wiki.go` | 309-311 |

---

## 五、问题总结表

| 问题分类 | 具体问题 | 影响程度 | 代码位置 |
|----------|----------|----------|----------|
| 事务一致性 | Git 操作成功后进程崩溃，数据库回滚导致不一致 | 高 | `services/wiki/wiki.go:382-410` |
| 路径解析 | QueryUnescape 回退导致 `%2F` 与 `/` 歧义 | 中 | `routers/web/repo/wiki.go:80-96` |
| 统计不一致 | 页面命中路径与 FileCommitsCount 参数不一致 | 高 | `routers/web/repo/wiki.go:144-169` |
| 自动恢复 | 分支不一致仅在访问时触发恢复，可能长期不一致 | 低 | `routers/web/repo/wiki.go:98-125` |
