---
title: "Git 常用命令和工作流笔记"
date: 2026-06-24 21:30:00 +0800
tags: [Git, GitHub, 版本控制, 命令行, 开发工具]
main_category: "技术随笔"
discipline: "其他"
course: "Git"
material_type: "学习笔记"
description: "从安装配置、工作区/暂存区、本地提交、撤销修改、远程仓库、分支协作到 .gitignore，整理一份日常够用的 Git 复习笔记。"
---

这篇是我给自己整理的 Git 基础笔记。目标不是把所有 Git 命令背完，而是先掌握日常开发最常用的工作流：初始化仓库、提交修改、查看历史、撤销错误、连接远程、管理分支和处理协作。

<!--more-->

## Git 的核心模型

Git 最重要的是先理解三个区域：

```text
工作区 Working Tree
  -> 当前文件夹里正在编辑的文件

暂存区 Staging Area / Index
  -> 已经 git add、准备进入下一次 commit 的修改

本地仓库 Repository
  -> 已经 git commit 保存下来的版本历史
```

日常提交通常就是这条链路：

```text
编辑文件
-> git status 查看状态
-> git diff 查看具体改了什么
-> git add 放入暂存区
-> git commit 保存到本地仓库
-> git push 推送到远程仓库
```

## 安装和基础配置

macOS 可以用 Homebrew 安装：

```bash
brew install git
```

第一次使用 Git，先配置用户名和邮箱。它们会写入 commit 作者信息：

```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
```

查看当前配置：

```bash
git config --global --list
```

## 创建版本库

进入项目目录后执行：

```bash
git init
```

这会在当前目录下创建 `.git` 隐藏目录。`.git` 是 Git 保存版本历史和内部数据的地方，不要手动改里面的文件。

如果看不到 `.git`，可以用：

```bash
ls -ah
```

Finder 里显示隐藏文件的快捷键是：

```text
Command + Shift + .
```

## 第一次提交

添加单个文件：

```bash
git add README.md
```

添加当前目录下所有变动：

```bash
git add .
```

提交：

```bash
git commit -m "init project"
```

常用检查命令：

```bash
git status
git diff
```

`git status` 看文件处于什么状态，`git diff` 看工作区还没暂存的具体改动。

## 查看历史

查看提交历史：

```bash
git log
```

更紧凑的单行历史：

```bash
git log --pretty=oneline
```

我更常用这一版：

```bash
git log --oneline --graph --decorate --all
```

它能同时看到提交、分支和合并关系。

## 版本回退

`HEAD` 表示当前版本，也就是当前分支最新提交。

```text
HEAD      当前提交
HEAD^     上一个提交
HEAD^^    上上个提交
HEAD~100  往前 100 个提交
```

回退到上一个提交：

```bash
git reset --hard HEAD^
```

回退到指定提交：

```bash
git reset --hard <commit_id>
```

`reset` 的三种常见模式：

| 命令 | 影响 commit | 影响暂存区 | 影响工作区 | 适合场景 |
| --- | --- | --- | --- | --- |
| `git reset --soft <id>` | 回退 | 保留 | 保留 | 想撤销 commit，但保留已暂存状态 |
| `git reset --mixed <id>` | 回退 | 重置 | 保留 | 想撤销 commit 和 add，保留文件修改 |
| `git reset --hard <id>` | 回退 | 重置 | 重置 | 想完全回到某个版本 |

`--hard` 会丢掉工作区修改，用之前先确认没有未保存的重要内容。

如果回退后后悔了，可以用 `reflog` 找回之前的 commit：

```bash
git reflog
git reset --hard <commit_id>
```

`git log` 看提交历史，`git reflog` 看本地 HEAD 移动历史。只要记录还在，很多误操作都能救回来。

## 撤销修改

撤销要先判断修改在哪一层。

| 场景 | 命令 |
| --- | --- |
| 工作区改乱了，还没 `git add` | `git restore <file>` |
| 已经 `git add`，想取消暂存 | `git restore --staged <file>` |
| 取消暂存后还想丢弃工作区修改 | `git restore <file>` |
| 已经 commit，但还没 push | `git reset --soft/--mixed/--hard <id>` |
| 已经 push，不想改写历史 | `git revert <commit_id>` |

老命令也能用：

```bash
git checkout -- <file>
git reset HEAD <file>
```

但现在更推荐用 `git restore` 表达“恢复文件”，用 `git switch` 表达“切换分支”，语义更清楚。

## 删除文件

删除文件并把删除操作放入暂存区：

```bash
git rm <file>
git commit -m "remove file"
```

如果只是手动删了文件，也可以：

```bash
git add <file>
git commit -m "remove file"
```

如果误删了还没提交，可以恢复：

```bash
git restore <file>
```

## 连接远程仓库

先在 GitHub 创建仓库，然后在本地关联远程：

```bash
git remote add origin git@github.com:<user>/<repo>.git
```

查看远程地址：

```bash
git remote -v
```

第一次推送并建立 upstream：

```bash
git push -u origin main
```

如果你的默认分支叫 `master`，就用：

```bash
git push -u origin master
```

后续已经绑定 upstream 后，通常只需要：

```bash
git push
git pull
```

删除远程关联：

```bash
git remote rm origin
```

从远程克隆仓库：

```bash
git clone git@github.com:<user>/<repo>.git
```

## 配置 SSH Key

先检查有没有已有 key：

```bash
ls -ah ~/.ssh
```

生成 SSH Key。现在更推荐 `ed25519`：

```bash
ssh-keygen -t ed25519 -C "email@example.com"
```

如果环境较老，也可以用 RSA：

```bash
ssh-keygen -t rsa -C "email@example.com"
```

复制公钥到剪贴板：

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

如果你生成的是 RSA：

```bash
pbcopy < ~/.ssh/id_rsa.pub
```

然后去 GitHub 的 SSH Keys 页面添加公钥。注意：`id_xxx` 是私钥，不能泄露；`id_xxx.pub` 是公钥，可以提交给 GitHub。

## 分支管理

查看分支：

```bash
git branch
```

创建分支：

```bash
git branch <name>
```

切换分支：

```bash
git switch <name>
```

创建并切换分支：

```bash
git switch -c <name>
```

老命令也可以：

```bash
git checkout <name>
git checkout -b <name>
```

合并某个分支到当前分支：

```bash
git merge <name>
```

删除已经合并的分支：

```bash
git branch -d <name>
```

强制删除未合并分支：

```bash
git branch -D <name>
```

`-D` 会丢掉该分支上未合并的提交，使用前要确认这些提交不再需要。

## 解决冲突

当 Git 无法自动合并时，会出现 conflict。处理流程：

```bash
git status
```

打开冲突文件，手动编辑成想要的最终内容。冲突标记通常长这样：

```text
<<<<<<< HEAD
当前分支内容
=======
被合并分支内容
>>>>>>> feature
```

解决后：

```bash
git add <file>
git commit
```

查看分支图：

```bash
git log --graph --oneline --decorate --all
```

## 临时保存工作现场

如果手头工作没做完，但要临时切去别的分支修 bug：

```bash
git stash
```

查看 stash：

```bash
git stash list
```

恢复最近一次 stash，并删除这条 stash 记录：

```bash
git stash pop
```

只恢复，不删除 stash 记录：

```bash
git stash apply
```

## cherry-pick

如果某个 bug fix 已经提交在另一个分支上，想把这个提交复制到当前分支：

```bash
git cherry-pick <commit_id>
```

典型场景：

```text
master 上修了 bug
dev 分支也需要这个 bug fix
-> 切到 dev
-> git cherry-pick <bugfix_commit>
```

`cherry-pick` 复制的是一次提交的修改，不是合并整个分支。

## 多人协作

查看远程分支：

```bash
git branch -r
```

拉取远程更新：

```bash
git pull
```

推送当前分支：

```bash
git push origin <branch-name>
```

基于远程分支创建本地分支：

```bash
git switch -c <branch-name> origin/<branch-name>
```

设置本地分支和远程分支的 upstream：

```bash
git branch --set-upstream-to=origin/<branch-name> <branch-name>
```

常见协作规则：

- 本地新建分支如果不 push，别人看不到。
- push 失败通常是远程已有新提交，需要先 pull 或 fetch + rebase。
- pull 后有冲突，要先解决冲突再继续。

## rebase

`rebase` 可以把本地未 push 的分叉提交整理成更直的历史：

```bash
git fetch origin
git rebase origin/main
```

适合场景：

- 本地分支还没有 push。
- 想让提交历史更线性。
- 想减少无意义的 merge commit。

不建议对已经推送并且被别人基于其开发的公共分支随便 rebase，因为它会改写提交历史。

## .gitignore

`.gitignore` 用来忽略不该进入版本库的文件。

常见应该忽略的内容：

- 操作系统自动生成文件，例如 `.DS_Store`。
- 编译产物、缓存目录、临时文件。
- 本地虚拟环境，例如 `.venv/`、`node_modules/`。
- 带密码、token、密钥的本地配置文件。

`.gitignore` 文件本身应该提交到 Git，保证团队使用同一套忽略规则。

示例：

```gitignore
.DS_Store
node_modules/
.venv/
__pycache__/
*.log
.env
```

如果某个文件已经被 Git 跟踪，后来再写进 `.gitignore` 不会自动取消跟踪。需要：

```bash
git rm --cached <file>
```

## macOS 命令行补充

跳转用户主目录：

```bash
cd ~
```

Finder 直接跳到用户主目录：

```text
Command + Shift + H
```

查看隐藏文件：

```bash
ls -ah
```

按修改时间倒序列出详细信息，最新在底部：

```bash
ls -ltr
```

查看文件内容：

```bash
cat <file>
```

只列出目录本身，不展开目录内容：

```bash
ls -d .ssh
```

## 我的日常 Git 流程

单人开发：

```bash
git status
git diff
git add .
git commit -m "describe change"
git push
```

新功能开发：

```bash
git switch main
git pull
git switch -c feature/xxx

# 开发、提交
git add .
git commit -m "add xxx"

git push -u origin feature/xxx
```

修 bug：

```bash
git stash
git switch main
git pull
git switch -c fix/bug-name

# 修复并提交
git add .
git commit -m "fix bug-name"
git push -u origin fix/bug-name

# 回到原工作分支
git switch <old-branch>
git stash pop
```

最该记住的几句话：

- `git status` 先看状态，不确定时别急着操作。
- `git add` 只是放入暂存区，不等于保存进历史。
- `git commit` 才是在本地仓库生成一个版本。
- `git push` 才会同步到远程。
- `git reflog` 是很多本地误操作的后悔药。
- `reset --hard` 和 `branch -D` 都可能丢工作，执行前要确认。
