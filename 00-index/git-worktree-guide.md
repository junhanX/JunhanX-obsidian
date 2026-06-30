# Git Worktree 使用指南

## 是什么

Git Worktree 允许在**同一个仓库**中，把不同分支检出到**不同目录**，并行工作互不干扰。

```
/项目/
├── .git/                        ← 共享的 git 仓库
├── src/                         ← 目录A：main 分支
│   └── ...
└── .worktrees/
    └── feature-dev/             ← 目录B：feature-dev 分支
        └── ...
```

## 为什么用它

| 场景 | 传统方式 | 用 Worktree |
|------|---------|------------|
| 开发中需要修 hotfix | stash → switch → fix → switch → pop | 新终端直接修 |
| 同时开发两个功能 | 反复切换分支 | 两个目录并行 |
| 跑长时间测试 | 干等，不能写代码 | 一个跑测试，一个继续写 |
| Code Review | 切分支看代码 | 检出到独立目录对比 |

**一句话：免去 stash/switch 的来回折腾。**

## 常用命令

### 创建 Worktree

```bash
# 创建新分支并检出
git worktree add ../path/to/dir -b new-branch

# 检出已有分支
git worktree add ../path/to/dir existing-branch
```

### 查看所有 Worktree

```bash
git worktree list

# 输出示例：
# /project                          abc123 [main]
# /project/.worktrees/feature-dev   abc123 [feature-dev]
```

### 删除 Worktree

```bash
# 正常删除
git worktree remove ../path/to/dir

# 强制删除（目录已手动删除时）
git worktree prune
```

## 操作流程

### 1. 初始化

```bash
mkdir my-project && cd my-project
git init
git commit --allow-empty -m "Initial commit"

# 确保 worktree 目录不被追踪
echo ".worktrees/" >> .gitignore
git add .gitignore && git commit -m "Add .gitignore"
```

### 2. 创建隔离开发环境

```bash
git worktree add .worktrees/feature-dev -b feature-dev
cd .worktrees/feature-dev
```

### 3. 并行开发

```
终端A: cd /project             → main 分支
终端B: cd /project/.worktrees/feature-dev → feature-dev 分支
```

### 4. 合并回主分支

```bash
cd /project                    # 回到主目录
git checkout main
git merge feature-dev          # 合并

# 清理
git branch -d feature-dev
git worktree remove .worktrees/feature-dev
```

## 注意事项

| 规则 | 说明 |
|------|------|
| 同一分支不能检出两次 | `git worktree add` 会拒绝 |
| `.git` 是文件不是目录 | 子 worktree 中是 `.git` 文件指向主仓库 |
| 确保目录被 gitignore | 防止 worktree 内容被意外提交 |
| 共享同一个仓库 | 提交、分支、标签在所有 worktree 间立即可见 |

## 与其他方式的对比

| | Worktree | `git checkout` | 手动 `cp` |
|------|----------|---------------|-----------|
| 并行开发 | ✅ 天然支持 | ❌ 来回切换 | ⚠️ 代码分散 |
| 磁盘占用 | 少（共享 .git） | 少 | 多（每份都有 .git） |
| 分支同步 | 自动 | 自动 | 手动 |
| 防止冲突 | 锁定同分支 | 无保护 | 无保护 |
