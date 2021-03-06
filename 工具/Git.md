# Git

## 注意

1. 如果想切回主分支，如果当前修改都已完成，那就提交后切回主分支，如果未完成，那就存储后再切回主分支。

## Git简介

### 创建版本库

首先这里再明确一下，所有的版本控制系统，其实只能跟踪文本文件的改动，比如TXT文件，网页，所有的程序代码等等，Git也不例外。版本控制系统可以告诉你每次的改动，比如在第5行加了一个单词“Linux”，在第8行删了一个单词“Windows”。**而图片、视频这些二进制文件，虽然也能由版本控制系统管理，但没法跟踪文件的变化，只能把二进制文件每次改动串起来，也就是只知道图片从100KB改成了120KB，但到底改了啥，版本控制系统不知道，也没法知道**。不幸的是，Microsoft的Word格式是二进制格式，因此，版本控制系统是没法跟踪Word文件的改动的，**如果要真正使用版本控制系统，就要以纯文本方式编写文件**。

```bash
# 把这个目录变成Git可以管理的仓库：
git init
# 把文件添加到仓库：
git add <file>
# 把文件提交到仓库：
git commit -m <message>
```

## 时光机穿梭

```bash
# 掌握仓库当前的状态：
git status
# 看看具体修改了什么内容：
git diff <file>
# 查看工作区和版本库里面最新版本的区别：
git diff HEAD -- <file>
```

- 提交过，被修改，未添加：Changes not staged for commit；
- 已添加，未提交：Changes to be committed；
- 从未被添加过：Untracked files。

### 版本回退

每当你觉得文件修改到一定程度的时候，就可以“保存一个快照”，这个快照在Git中被称为commit。一旦你把文件改乱了，或者误删了文件，还可以从最近的一个commit恢复，然后继续工作，而不是把几个月的工作成果全部丢失。

```bash
# 显示从最近到最远的提交日志：
git log
# 简化输出信息：
git log --pretty=oneline
```

一大串类似1094adb...的是commit id（版本号），是一个用SHA1计算出来的一个非常大的数字，用十六进制表示。

在Git中，用HEAD表示当前版本，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100。

```bash
# 回退到上一个版本：
git reset --hard HEAD^
# 指定回到某个版本：
git reset --hard <commit id>
```

Git的版本回退速度非常快，因为Git在内部有个指向当前版本的HEAD指针，当你回退版本的时候，Git仅仅是把HEAD从指向当前版本改为指向目标版本，然后顺便把工作区的文件更新了。

```bash
# 记录你的每一次版本变更：
git reflog
```

- 穿梭前，用git log可以查看提交历史，以便确定要回退到哪个版本。
- 要重返未来，用git reflog查看命令历史，以便确定要回到未来的哪个版本。

### 工作区和暂存区

*工作区*（Working Directory）：就是你在电脑里能看到的目录。

*版本库*（Repository）：工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的*暂存区*，还有Git为我们自动创建的第一个*分支*master，以及指向master的一个*指针*叫HEAD。

<img src="./image/工作区与版本库.png">

把文件往Git版本库里添加的时候，是分两步执行的：

  1. 用git add把文件添加进去，实际上就是把文件修改添加到暂存区；
  2. 用git commit提交更改，实际上就是把暂存区的所有内容提交到当前分支。

我们创建Git版本库时，Git自动为我们创建了唯一一个master分支，所以，现在，git commit就是往master分支上提交更改。简单理解为，需要提交的文件修改通通放到暂存区，然后，一次性提交暂存区的所有修改。

### 管理修改

Git跟踪并管理的是*修改*，而非文件。**即提交的是暂存区的修改而不是工作区的修改**。

### 撤销修改

```bash
# 丢弃工作区的修改，回到最近一次add或commit的状态：
git checkout -- <file>
```

如果add过，就回到最近一次add的状态，如果没有add过，就回到最近一次commit的状态。

```bash
# 从暂存区中撤销被添加的文件修改：
git reset HEAD <file>
```

因此，当你添加了某个文件修改，此时想丢弃工作区的这个文件修改，必须先从暂存区中撤销，再进行丢弃。

场景：

  1. 当你改乱了工作区某个文件的内容，想直接丢弃工作区的修改时，用命令git checkout -- file。
  2. 当你不但改乱了工作区某个文件的内容，还添加到了暂存区时，想丢弃修改，分两步，第一步用命令git reset HEAD file，就回到了场景1，第二步按场景1操作。
  3. 已经提交了不合适的修改到版本库时，想要撤销本次提交，参考版本回退一节，不过前提是没有推送到远程库。

### 删除文件

前提：删除了一个已经提交到版本库的文件，此时git status提示**该文件被修改但未添加**。

```bash
# 确实想从版本库中删除该文件：
git rm <file>
git commit -m <message>
```

实际上，先手动删除文件，然后使用git rm file和git add file效果是一样的。**删除也是一种修改**。

```bash
# 删除了，但是版本库还有，想恢复：
git checkout -- <file>
```

因为此次修改没有被add，因此使用checkout命令会回到最近一次commit的状态，而上一次的版本库里还留存有该文件，因此将该文件从版本库恢复到了工作区。

git checkout其实是用版本库里的版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

**注意：从来没有被添加到版本库就被删除的文件，是无法恢复的**！

命令git rm用于删除一个文件。如果一个文件已经被提交到版本库，那么你永远不用担心误删，但是要小心，**你只能恢复文件到最新版本，你会丢失最近一次提交后你修改的内容**。

## 远程仓库

为什么GitHub需要SSH Key。因为GitHub需要识别出你推送的提交确实是你推送的，而不是别人冒充的，而Git支持SSH协议，所以，GitHub只要知道了你的公钥，就可以确认只有你自己才能推送。

当然，GitHub允许你添加多个Key。假定你有若干电脑，你一会儿在公司提交，一会儿在家里提交，只要把每台电脑的Key都添加到GitHub，就可以在每台电脑上往GitHub推送了。

### 添加远程库

```bash
# 关联一个空的仓库：
git remote add origin git@github.com:Mccreo/learngit.git
# 第一次推送：
git push -u origin master
# 后续推送：
git push origin master
```

远程库的名字就是origin，这是Git默认的叫法，也可以改成别的，但是origin这个名字一看就知道是远程库。

把本地库的内容推送到远程，用git push命令，实际上是把当前分支master推送到远程。

由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送给远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。

### 从远程库克隆

假设我们从零开发，那么最好的方式是先创建远程库，然后，从远程库克隆。

```bash
# 克隆一个本地库：
git clone git@github.com:michaelliao/gitskills.git
```

## 分支管理

### 创建与合并分支

```bash
# 创建分支并切换到分支：
git switch -c dev
# 分两条语句：
git branch dev
git switch dev
# 查看所有分支：
git branch
# 合并指定分支到当前分支：
git merge dev
# 删除指定分支
git branch -d dev
```

假设将dev分支提交的修改合并到master分支并且master分支没有任何提交：

<img src="./image/合并-无冲突01.png"/>

<img src="./image/合并-无冲突02.png"/>

Git鼓励你使用分支完成某个任务，合并后再删掉分支，这和直接在master分支上工作效果是一样的，但过程更安全。

### 解决冲突

场景：新建了一个feature1分支，在该分支上修改了readme.txt文件的最后一行，**添加并提交**；回到master分支，在该分支上修改了readme.txt文件的最后一行，**添加并提交**。尝试合并feature1分支到master分支。

发生冲突：原因在于两个分支都修改了同一行，Git无法做出选择。

此时我们需要进入到readme.txt文件，将冲突内容修改为我们希望的内容，**添加并提交**。

```bash
# 可以查看分支的合并情况：
git log --graph --pretty=oneline --abbrev-commit
```

当Git无法自动合并分支时，就必须首先解决冲突。解决冲突后，再提交，合并完成。解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。

<img src="./image/合并-有冲突01.png">

<img src="./image/合并-有冲突02.png">

### 分支管理策略

通常，合并分支时，如果可能，Git会用Fast forward模式，但这种模式下，删除分支后，会丢掉分支信息。如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

```bash
git merge --no-ff -m "merge with no-ff" dev
```

因为本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去。

按照几个基本原则进行分支管理：

  1. 首先，master分支应该是非常稳定的，也就是仅用来发布新版本，平时不能在上面干活；
  2. 干活都在dev分支上，也就是说，dev分支是不稳定的，到某个时候，比如1.0版本发布时，再把dev分支合并到master上，在master分支发布1.0版本；

合并分支时，加上--no-ff参数就可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并，而fast forward合并就看不出来曾经做过合并。

**合并完分支后，建议删除该分支**。

### Bug分支

```bash
# 把当前工作现场“储藏”起来，等以后恢复现场后继续工作：
git stash
# 查看工作现场：
git stash list
# 恢复工作现场但不把工作现场从现场栈中删除：
git stash apply
# 将工作现场从现场栈中删除：
git stash drop
# 恢复的同时删除：
git stash pop
# 复制一个特定提交所作的修改到当前分支并提交：
git cherry-pick <commit id>
```

- 修复bug时，我们会通过创建新的bug分支进行修复，然后合并，最后删除；
- 当手头工作没有完成时，先把工作现场git stash一下，然后去修复bug，修复后，再git stash pop，回到工作现场；
- 在master分支上修复的bug，想要合并到当前dev分支，可以用git cherry-pick命令，把bug提交的修改“复制”到当前分支，避免重复劳动。

### Feature分支

```bash
# 强行删除一个没有被合并过的分支：
git brach -D <name>
```

开发一个新feature，最好新建一个分支；

### 多人协作

```bash
# 查看远程库的信息：
git remote
# 详细信息：
git remote -v
# origin是远程库的名称，推送master分支到远程库中对应的master分支：
git push origin master
# 创建本地的dev分支，并和远程库的dev分支对应起来：
git switch -c dev origin/dev
# 把当前分支的最新提交抓下来到本地合并：
git pull
# 将已存在的dev分支和远程的dev分支进行链接：
git branch --set-upstream-to=origin/dev dev
```

多人协作的工作模式通常是这样：

  1. 首先，可以试图用git push origin branch-name推送自己的修改；
  2. 如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；
  3. 如果合并有冲突，则解决冲突，并在本地提交；
  4. 没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！

如果git pull提示no tracking information，则说明本地分支和远程分支的链接关系没有创建，用命令git branch --set-upstream-to branch-name origin/branch-name。

### Rebase

```bash
git rebase
```

- rebase操作可以把**本地未push的分叉提交历史整理成直线**；
- rebase的目的是使得我们在查看历史提交的变化时更容易，因为分叉的提交需要三方对比。

## 标签管理

标签也是版本库的一个快照。标签就是一个让人容易记住的有意义的名字，它跟某个commit绑在一起。

### 创建标签

```bash
# 打在最新提交的commit上：
git tag <tag name>
# 查看所有标签：
git tag
# 打在指定commit上：
git tag <tag name> <commit id>
# 查看标签信息：
git show <tag name>
# 创建带有说明信息的标签：
git tag -a <tag name> -m <message> <commit id>
```

**标签总是和某个commit挂钩。如果这个commit既出现在master分支，又出现在dev分支，那么在这两个分支上都可以看到这个标签**。

### 操作标签

```bash
# 删除标签：
git tag -d <tag name>
# 推送标签到远程：
git push origin <tag name>
# 一次性推送全部尚未推送到远程的本地标签：
git push origin --tags
```

如果标签已经推送到远程，要删除远程标签就麻烦一点：

```bash
git tag -d <tag name>
git push origin :refs/tags/<tag name>
```

## 自定义Git

### 忽略特殊文件

忽略某些文件时，需要编写.gitignore提交到版本库。

```bash
# 检查忽略的规则：
git check-ignore -v <file name>
```
