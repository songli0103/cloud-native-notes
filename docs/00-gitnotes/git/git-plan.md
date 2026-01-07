# 🚀 Git 系统学习路线图 (From Zero to Hero)

> **学习核心原则**：不要死记硬背命令，要构建“心理模型”。  
> 始终思考：**“当前 HEAD 指针在哪里？我的文件现在处于哪一个区域（工作区/暂存区/仓库）？”**

---

## 🟢 第一阶段：核心概念构建 (Mental Model)
**目标**：理解 Git 的数据存储方式，这是所有操作的基础。

- [ ] **版本控制演变**
    - [ ] 集中式 (SVN) vs 分布式 (Git) 的区别
- [ ] **Git 的三个核心区域 (重点)**
    - [ ] **工作区 (Working Directory)**：你编辑文件的地方
    - [ ] **暂存区 (Staging Area / Index)**：临时保存改动的地方
    - [ ] **本地仓库 (Local Repository)**：安全存档的地方 (`.git` 目录)
- [ ] **Git 的底层原理**
    - [ ] **快照 (Snapshot)** vs 差异 (Delta)
    - [ ] **Commit 对象**：包含指向父提交的指针、作者、时间、快照哈希
    - [ ] **HEAD 指针**：当前你所在的位置
    - [ ] **状态生命周期**：Untracked / Modified / Staged / Committed

---

## 🔵 第二阶段：基础生存技能 (Basic Operations)
**目标**：能够独立在本地进行代码管理。

- [ ] **环境配置**
    - [ ] `git config --global user.name "Your Name"`
    - [ ] `git config --global user.email "email@example.com"`
    - [ ] `git config --list`
- [ ] **仓库初始化**
    - [ ] `git init` (新项目)
    - [ ] `git clone <url>` (已有项目)
- [ ] **日常提交流 (肌肉记忆)**
    - [ ] `git status` (**最重要命令**：时刻检查当前状态)
    - [ ] `git add <file>` / `git add .` (工作区 -> 暂存区)
    - [ ] `git commit -m "feat: message"` (暂存区 -> 仓库)
- [ ] **查看历史**
    - [ ] `git log` / `git log --oneline --graph`
    - [ ] `git diff` (查看工作区与暂存区的差异)
    - [ ] `git diff --staged` (查看暂存区与仓库的差异)
- [ ] **忽略文件**
    - [ ] 编写 `.gitignore` 文件 (忽略 `.env`, `node_modules/`, `*.log` 等)

---

## 🟡 第三阶段：分支与合并 (Branching & Merging)
**目标**：掌握并行开发的能力，理解 Git 极其轻量的分支模型。

- [ ] **分支操作**
    - [ ] `git branch` (查看/创建)
    - [ ] `git checkout <branch>` 或 `git switch <branch>` (切换分支)
    - [ ] `git checkout -b <branch>` 或 `git switch -c <branch>` (创建并切换)
    - [ ] `git branch -d <branch>` (删除已合并分支)
- [ ] **合并策略**
    - [ ] `git merge <branch>`
    - [ ] 理解 **Fast-forward** (快进模式)
    - [ ] 理解 **Recursive / Merge Commit** (三方合并模式)
- [ ] **解决冲突 (难点)**
    - [ ] 识别冲突标记 (`<<<<<<<`, `=======`, `>>>>>>>`)
    - [ ] 手动修改冲突文件
    - [ ] 重新 `add` 和 `commit` 完成合并

---

## 🟠 第四阶段：远程协作 (Remote Collaboration)
**目标**：与 GitHub/GitLab/Gitee 交互，进行团队合作。

- [ ] **远程仓库管理**
    - [ ] `git remote add origin <url>`
    - [ ] `git remote -v`
- [ ] **同步数据**
    - [ ] `git push -u origin <branch>` (推送)
    - [ ] `git fetch` (下载代码但不合并，**推荐**)
    - [ ] `git pull` (下载并自动合并，等同于 fetch + merge)
- [ ] **协作流程**
    - [ ] **Pull Request (PR) / Merge Request (MR)** 流程
    - [ ] Fork 模式与 Upstream 同步
    - [ ] Code Review 基本操作

---

## 🔴 第五阶段：后悔药与时光机 (Undo & Recovery)
**目标**：在犯错时能够安全地撤销操作，而不丢失重要数据。

- [ ] **撤销修改**
    - [ ] `git checkout -- <file>` 或 `git restore <file>` (丢弃工作区修改)
    - [ ] `git reset HEAD <file>` 或 `git restore --staged <file>` (将文件移出暂存区)
- [ ] **版本回退 (Reset)**
    - [ ] `git reset --soft <commit>` (回退提交，保留更改在暂存区)
    - [ ] `git reset --mixed <commit>` (回退提交，保留更改在工作区，默认模式)
    - [ ] `git reset --hard <commit>` (**危险**：彻底丢弃更改)
- [ ] **安全撤销 (Revert)**
    - [ ] `git revert <commit>` (生成反向提交，适合公共分支)
- [ ] **临时保存 (Stash)**
    - [ ] `git stash` (保存现场)
    - [ ] `git stash pop` (恢复现场)

---

## 🟣 第六阶段：高级进阶 (Advanced Techniques)
**目标**：保持提交历史整洁，处理复杂场景。

- [ ] **变基 (Rebase)**
    - [ ] `git rebase <branch>` (线性化历史)
    - [ ] **黄金法则**：永远不要在公共分支上 rebase！
- [ ] **交互式 Rebase (整理历史)**
    - [ ] `git rebase -i <commit>`
    - [ ] pick / squash (合并提交) / drop (删除提交) / reword (修改注释)
- [ ] **修改最近一次提交**
    - [ ] `git commit --amend`
- [ ] **精准提取**
    - [ ] `git cherry-pick <commit>` (跨分支捡代码)
- [ ] **打标签**
    - [ ] `git tag -a v1.0 -m "Release version"`
- [ ] **终极救援 (Reflog)**
    - [ ] `git reflog` (找回被 reset 掉的孤魂野鬼 commit)

---

## 🟤 第七阶段：规范与工作流 (Workflow & Best Practices)
**目标**：像专业团队一样工作。

- [ ] **分支管理模型**
    - [ ] **Git Flow** (经典的 master/develop/feature/release 结构)
    - [ ] **GitHub Flow** (轻量级，以 master 为主，适合 CI/CD)
    - [ ] **Trunk Based** (主干开发)
- [ ] **Commit Message 规范**
    - [ ] 学习 **Conventional Commits** (`feat:`, `fix:`, `docs:`, `refactor:`)
- [ ] **SSH Key 配置**
    - [ ] 配置 SSH 公钥/私钥，实现免密 push/pull

---

## 📚 推荐学习资源 (必须要看)

1.  **[Learn Git Branching](https://learngitbranching.js.org/)** (⭐⭐⭐⭐⭐)
    *   *评价*：最好的 Git 可视化学习游戏。通关它，你的 Git 水平超过 90% 的开发者。
2.  **Pro Git (官方书籍)**
    *   *评价*：Git 官网免费提供的电子书，权威指南。
3.  **Oh My Zsh + Git 插件**
    *   *建议*：如果你用 Mac/Linux，配置这个可以在终端直观看到当前分支和状态。