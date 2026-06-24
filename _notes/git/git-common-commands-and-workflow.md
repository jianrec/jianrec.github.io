---
title: "Git 常用命令和工作流笔记"
date: 2026-06-24 19:25:00 +0800
tags: [Git, GitHub, 版本控制, 命令行, 开发工具]
main_category: "技术随笔"
discipline: "其他"
course: "Git"
material_type: "学习笔记"
description: "从工作区、暂存区、本地仓库、远程仓库和分支协作几条主线整理 Git 日常命令。"
---

学 Git 不能只背命令。真正容易混的是：文件现在在哪一层、下一步要把它放到哪里、这条命令会不会丢东西。

这份笔记按日常使用顺序整理。先把工作区、暂存区、本地仓库和远程仓库这条线跑通，再补上回退、撤销、分支和协作。

<!--more-->

## Git 先看三层

Git 的日常操作基本都围绕这三层转：

```text
工作区 Working Tree
  当前文件夹里正在编辑的文件

暂存区 Staging Area / Index
  已经 git add，准备放进下一次 commit 的修改

本地仓库 Repository
  已经 git commit 保存下来的版本历史
```

平时提交就是：

```bash
git status
git diff
git add .
git commit -m "describe change"
git push
```

不确定时先 `git status`。这条命令最便宜，也最能防止误操作。

## 安装和初始化

macOS 安装：

```bash
brew install git
```

第一次用先配置身份：

```bash
git config --global user.name "Your Name"
git config --global user.email "email@example.com"
git config --global --list
```

进入项目目录后初始化仓库：

```bash
git init
```

初始化后会出现 `.git` 目录，这是 Git 保存版本信息的地方。它默认隐藏，可以这样看：

```bash
ls -ah
```

Finder 显示隐藏文件：

```text
Command + Shift + .
```

## 第一次提交

添加一个文件：

```bash
git add README.md
```

添加当前目录下所有变化：

```bash
git add .
```

提交到本地仓库：

```bash
git commit -m "init project"
```

`git add` 只是放到暂存区，`git commit` 才会生成一个本地版本。

## 查看状态和历史

看当前状态：

```bash
git status
```

看工作区还没暂存的改动：

```bash
git diff
```

看提交历史：

```bash
git log
```

更常用的简洁版本：

```bash
git log --oneline --graph --decorate --all
```

这条命令能同时看到提交、分支和合并关系。

## 回退版本

`HEAD` 指当前提交：

```text
HEAD      当前提交
HEAD^     上一个提交
HEAD^^    上上个提交
HEAD~10   往前 10 个提交
```

回退到上一个提交：

```bash
git reset --hard HEAD^
```

回退到指定提交：

```bash
git reset --hard <commit_id>
```

`reset` 的三种模式要分清：

| 命令 | commit | 暂存区 | 工作区 | 适合情况 |
| --- | --- | --- | --- | --- |
| `git reset --soft <id>` | 回退 | 保留 | 保留 | 撤销 commit，但保留 staged 状态 |
| `git reset --mixed <id>` | 回退 | 重置 | 保留 | 撤销 commit 和 add，保留文件修改 |
| `git reset --hard <id>` | 回退 | 重置 | 重置 | 完全回到某个版本 |

`--hard` 会直接改工作区，没保存的修改会丢。用之前先看 `git status`。

如果回退之后后悔，可以用：

```bash
git reflog
git reset --hard <commit_id>
```

`git log` 看提交历史，`git reflog` 看本地 `HEAD` 移动记录。很多本地误操作都靠它找回来。

## 撤销修改

按修改所在位置处理：

| 场景 | 命令 |
| --- | --- |
| 工作区改乱了，还没 add | `git restore <file>` |
| 已经 add，想取消暂存 | `git restore --staged <file>` |
| 取消暂存后还想丢掉文件修改 | `git restore <file>` |
| 已经 commit，还没 push | `git reset --soft/--mixed/--hard <id>` |
| 已经 push，不想改写远程历史 | `git revert <commit_id>` |

旧命令也能用：

```bash
git checkout -- <file>
git reset HEAD <file>
```

现在我更倾向用 `restore` 和 `switch`，因为语义清楚：`restore` 管文件恢复，`switch` 管分支切换。

## 删除文件

删除文件并把删除动作放进暂存区：

```bash
git rm <file>
git commit -m "remove file"
```

如果只是手动删除，也可以：

```bash
git add <file>
git commit -m "remove file"
```

误删且还没提交：

```bash
git restore <file>
```

## 连接 GitHub

关联远程仓库：

```bash
git remote add origin git@github.com:<user>/<repo>.git
git remote -v
```

第一次推送并绑定 upstream：

```bash
git push -u origin main
```

如果默认分支叫 `master`：

```bash
git push -u origin master
```

绑定后日常同步直接：

```bash
git push
git pull
```

删除远程关联：

```bash
git remote rm origin
```

克隆远程仓库：

```bash
git clone git@github.com:<user>/<repo>.git
```

## SSH Key

先看本机有没有现成 key：

```bash
ls -ah ~/.ssh
```

生成新 key：

```bash
ssh-keygen -t ed25519 -C "email@example.com"
```

老环境可以用 RSA：

```bash
ssh-keygen -t rsa -C "email@example.com"
```

复制公钥：

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

RSA 对应：

```bash
pbcopy < ~/.ssh/id_rsa.pub
```

`id_xxx` 是私钥，不要泄露；`id_xxx.pub` 是公钥，可以放到 GitHub 的 SSH Keys 页面。

## 分支

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

创建并切换：

```bash
git switch -c <name>
```

老命令：

```bash
git checkout <name>
git checkout -b <name>
```

把某分支合并到当前分支：

```bash
git merge <name>
```

删除已合并分支：

```bash
git branch -d <name>
```

强删未合并分支：

```bash
git branch -D <name>
```

`-D` 会丢掉这个分支上未合并的提交，确认不需要再用。

## 冲突

合并失败时先看状态：

```bash
git status
```

冲突文件里会出现：

```text
<<<<<<< HEAD
当前分支内容
=======
被合并分支内容
>>>>>>> feature
```

手动改成最终想要的内容，然后：

```bash
git add <file>
git commit
```

看分支图：

```bash
git log --graph --oneline --decorate --all
```

## stash

手头工作没做完，但要临时切分支：

```bash
git stash
```

查看：

```bash
git stash list
```

恢复并删除最近一次 stash：

```bash
git stash pop
```

只恢复，不删除记录：

```bash
git stash apply
```

## cherry-pick

把另一个分支上的某个提交复制到当前分支：

```bash
git cherry-pick <commit_id>
```

典型场景是 bug fix 已经进了 `main`，但 `dev` 也需要同一份修改。`cherry-pick` 复制的是一次提交，不是合并整个分支。

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

设置 upstream：

```bash
git branch --set-upstream-to=origin/<branch-name> <branch-name>
```

本地新分支不 push，别人看不到。push 失败一般是远程已有新提交，先 pull 或 fetch，再处理冲突。

## rebase

整理本地未 push 的提交历史：

```bash
git fetch origin
git rebase origin/main
```

适合本地分支还没推送、想让历史更直的时候。已经推到远程并且被别人基于它开发的公共分支，不要随便 rebase。

## .gitignore

`.gitignore` 放不该进仓库的文件：

```gitignore
.DS_Store
node_modules/
.venv/
__pycache__/
*.log
.env
```

常见忽略对象：

- 操作系统自动生成文件，比如 `.DS_Store`。
- 编译产物、缓存、临时文件。
- 本地虚拟环境。
- 带密码、token、密钥的本地配置。

`.gitignore` 本身要提交到 Git，团队才能共享同一套规则。

如果文件已经被 Git 跟踪，后来再加进 `.gitignore` 不会自动失效，需要：

```bash
git rm --cached <file>
```

## macOS 顺手命令

回到用户主目录：

```bash
cd ~
```

Finder 跳到用户主目录：

```text
Command + Shift + H
```

查看隐藏文件：

```bash
ls -ah
```

按修改时间列出文件，最新在底部：

```bash
ls -ltr
```

看文件内容：

```bash
cat <file>
```

只列目录本身，不展开里面内容：

```bash
ls -d .ssh
```

## 我实际常用的几套流程

单人开发：

```bash
git status
git diff
git add .
git commit -m "describe change"
git push
```

新功能：

```bash
git switch main
git pull
git switch -c feature/xxx

git add .
git commit -m "add xxx"
git push -u origin feature/xxx
```

临时修 bug：

```bash
git stash
git switch main
git pull
git switch -c fix/bug-name

git add .
git commit -m "fix bug-name"
git push -u origin fix/bug-name

git switch <old-branch>
git stash pop
```

最后记住几条就够用：

- 不确定先 `git status`。
- `git add` 只是暂存，不是保存历史。
- `git commit` 保存到本地。
- `git push` 同步到远程。
- `git reflog` 能救很多本地误操作。
- `reset --hard`、`branch -D` 用之前先确认工作区没有重要改动。
