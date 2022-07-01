---
layout: post
title: 常用Git命令
date: 2021-08-14 20:30:32 +0800
categories: 学习笔记
---

Git版本管理原理如图所示：

<div align=center><img src="/images/posts/learning-notes/git/git_principle.bmp" width="80%"><br>图1 Git版本管理原理框图</div>

##### git clone

克隆远程仓库的默认分支到本地。
```
git clone <远程仓库>
```

克隆远程仓库的指定分支到本地。
```
git clone -b <远程分支名> <远程仓库>
```

##### git init

在当前目录下创建新的Git仓库。
```
git init
```

##### git status

查看上次提交之后，在工作区中对哪些文件做了修改，还未添加到暂存区，同时查看暂存区中的哪些文件还没有提交到本地分支。
```
git status
```

##### git diff

查看在工作区中对哪些文件做了哪些修改，还未添加到暂存区。
```
git diff
```

查看在工作区中对指定文件做了哪些修改，还未添加到暂存区。
```
git diff <文件名>
```

##### git add

将工作区中对文件的修改添加到暂存区。
```
git add <文件名1> <文件名2> ...
```

将工作区中对已经追踪(Tracked)的文件的修改添加到暂存区，但是未追踪(Untracked)的文件的修改不会被添加到暂存区，必须明确使用文件名进行添加。
```
git add -u
```

##### git commit

将暂存区的文件修改提交到本地分支。
```
git commit -m "<本次提交的说明信息>" 
```

##### git log
查看当前分支的历史提交记录。
```
git log
```

##### git push

关联远程仓库，设置本地分支关联的远程分支，并将本地分支push上去 （本地仓库关联远程仓库之后首次push代码）。
```
git push --set-upstream <远程主机名> <远程分支名>
```

在远程仓库创建一个与本地分支同名的远程分支，并将本地分支push上去（并非本地仓库关联远程仓库之后首次push代码）。
```
git push -u <远程主机名>
```

将本地分支的提交push到远程关联的分支上。
```
git push
```

 在远程仓库创建一个远程分支，并将本地的分支push上去。（并非本地仓库关联远程仓库之后首次push代码）。
```
git push -u <远程主机名> <远程分支名>
```

将本地分支强制push（覆盖）到远程仓库关联的远程分支。
```
git push --force
```

将本地指定的分支强制push（覆盖）到远程指定的某个分支。
```
git push <远程主机名> <本地分支名>:<远程分支名> --force
```

##### git pull

同步远程仓库的关联分支到本地分支。
```
git pull
```

##### git revert

撤销特定的提交。执行`git revert`命令会在当前版本之后追加一条新的提交，用于记录撤销操作，而并非从原有的版本中删除指定的提交。
```
git revert <commit id>
```

##### git checkout

基于当前分支，在本地仓库创建一个新的分支，并切换到新创建的本地分支。
```
git checkout -b <本地分支名>
```

从当前分支切换到本地仓库的其他分支。
```
git checkout <本地分支名>
```

撤销文件在工作区的修改。一种是文件自修改后还没有被添加到暂存区，撤销修改就回到和版本库一模一样的状态；一种是文件已经添加到暂存区后又作了修改，撤销修改就回到添加到暂存区后的状态。总之，就是让文件回到最近一次git commit或git add时的状态。
```
git checkout -- <文件名1> <文件名2> ...
```

##### git branch

列出所有本地分支（当前分支会用*标识）
```
git branch
```

列出所有本地分支和远程分支（当前的本地分支会用*标识）
```
git branch -a
```

查看本地分支与远程分支的关联关系（当前的本地分支会用*标识）
```
git branch -vv 
```

修改本地分支名称
```
git branch -m <分支原名称> <分支新名称>
```

设置本地与远程分支的追踪关系
```
git  branch --set-upstream-to= <远程主机名>/<远程分支名>  <本地分支名>
```

删除本地分支（会在删除前检查merge状态）
```
git branch -d <分支名>
```

强制删除本地分支（git branch --delete --force的缩写）
```
git branch -D <分支名>
```

删除远程分支
```
git push <远程主机名> --delete <远程分支名>
```

##### git reset

回退当前分支到指定版本。
```
git reset [--soft | --mixed | --hard] [HEAD]
```
其中，"HEAD"表示当前版本，"HEAD^"表示上一个版本，"HEAD^^"表示上上一个版本，以此类推；"HEAD\~0"表示当前版本，"HEAD\~1"表示上一个版本；"HEAD^2"表示上上一个版本，"HEAD^3"标识上上上一个版本，以此类推。
* 参数“--soft”用于将HEAD引用指向给定提交，暂存区和工作目录的内容是不变的，在三个命令中对现有版本库状态改动最小。
* 参数“--mixed”为默认，可以省略，用于将HEAD引用指向给定提交，并且暂存区内容也跟着改变，工作目录内容不变；这个命令会将暂存区变成刚刚暂存该提交全部变化时的状态，会显示工作目录中有什么修改。
* 参数“--hard”用于将HEAD引用指向给定提交，暂存区和工作目录的内容都会变为给定提交时的状态，也就是在给定提交后所修改的内容都会丢失(新文件会被删除，不在工作目录中的文件恢复，未清除回收站的前提)。

回退指定文件到指定版本。
```
git reset [--soft | --mixed | --hard] [HEAD] <文件名> 
```

例如：

回退当前分支到上一个版本。
```
git reset HEAD^
```

回退指定文件到上一个版本。
```
git reset HEAD^ <文件名>
```

##### git remote

查看本地仓库与远程仓库的关联关系。
```
git remote -vv
```

给本地仓库添加远程仓库关联。
```
git remote add <远程主机名> <远程仓库名>
```

给本地仓库取消远程仓库关联。
```
git remote remove <远程主机名>
```

显示远程追踪关系。
```
git remote show <远程主机名>
```

##### git config

安装完 Git 之后，需要设置你的用户名和邮件地址，每一次 Git 提交都会使用这些信息，它们会写入到你的每一次提交中，不可更改。如果使用了 `--global` 选项，那么该命令只需要运行一次，之后无论在该系统上使用Git做任何事情， 都会使用这些信息。 如果想针对特定项目使用不同的用户名称与邮件地址时，可以在特定项目目录下运行没有 `--global` 选项的命令来配置。

 配置Git的用户名称。
```
git config --global user.name "<用户名称>" 
```

配置Git的用户邮箱。
```
git config --global user.email "<邮箱>"
```

配置Git的默认编辑器为vim。
```
git config --global core.editor vim
```

##### git rebase

（1）合并官方仓库的master分支到本地指定分支。

* 切换到本地master分支。
```
git checkout master
```
本地master分支与官方仓库的master分支关联，一般不建议在本地master分支进行代码修改，本地master分支只用于同步官方仓库的master分支。

* 同步官方仓库的master分支到本地的master分支。
```
git pull
```

* 切换到需要执行rebase操作的本地分支。
```
git checkout <本地分支名>
```

* 合并本地分支的多次提交。
```
git rebase -i <commit id>
```

将`<commit id>`对应的版本之后的所有版本进行合并，不包含`<commit id>`对应的版本。执行完`git rebase -i <commit id>`命令之后会弹出文档，只保留文档中第一个commit的 "pick"， 将其他commit的"pick"改为 "s"（此时也可以只合并其中的部分版本，将需要合并到前一个版本的"pick"修改为"s"，"s"是"squash的简写"）。例如：
```
pick 8953da2ff [DORIS] init branch-0.12 from doris branch-0.11 version
s 27a89de56 Add CD for deploy doris 0.12
s 804e0224b Improve CI resource request
s f82fa9260 Remove unnecessary CI build stage
s 55fffd0cf Fix CI bug
s 5e49254c7 Fix CI failed cause by TimeZone
s 8ca6b29c7  Add LSAN and ASAN for CI
s 02f7a22d1  fix CI config & use small limit config in ut
s 0823cf4b3 update CI for doris-0.13-branch

# 变基 406c30cca..0823cf4b3 到 406c30cca（9 个提交）
#
# 命令:
# p, pick = 使用提交
# r, reword = 使用提交，但修改提交说明
# e, edit = 使用提交，但停止以便进行提交修补
# s, squash = 使用提交，但和前一个版本融合
# f, fixup = 类似于 "squash"，但丢弃提交说明日志
# x, exec = 使用 shell 运行命令（此行剩余部分）
# d, drop = 删除提交
#
# 这些行可以被重新排序；它们会被从上至下地执行。
#
# 如果您在这里删除一行，对应的提交将会丢失。
#
# 然而，如果您删除全部内容，变基操作将会终止。
#
# 注意空提交已被注释掉
```

保存文档并退出之后，会弹出一个log文档，编辑文档只保留第一个commit的message log说明，删除其他commit的log说明（当然此时也可以保留或修改各个commit的commit message）。

* 执行`git log`命令查看当前分支，发现需要合并的多个版本已经合并。

* 将本地分支与官方仓库master分支执行rebase操作。
```
git rebase <官方主机名>/master
```

`<官方主机名>`是执行`git remote add <远程主机名> <远程仓库名>`命令时指定的，或者是执行`git clone <远程仓库>`命令自动生成的。如果本地提交与官方仓库的代码不存在冲突，则执行`git rebase <官方主机名>/master`命令之后，官方仓库的master分支将被成功地合并到本地分支；否则需要手动处理冲突。手动处理冲突的方法如下：
a. 执行`git status`命令，查看冲突文件。
b. 编辑冲突文件，解决代码冲突。
解决完代码冲突之后，执行`git rebase --continue`命令继续进行rebase操作；否则，执行`git rebase --abort`命令，放弃本次rebase操作。

* 执行`git log`命令查看rebase结果。

（2）修改已经提交的Commit Message。

* 与需要修改message的第一个commit的前一个commit执行rebase操作。
```
git rebase -i <commit id>
```

执行完`git rebase -i <commit id>`命令之后会弹出文档，编辑文档将需要修改message的commit项的 “pick” 修改为 “e”。例如：
```
e 2135f1e1d update CI for doris-0.13-branch
pick 69a0b6326 [Doirs] Intergretion GA&&MiStat UDF to doris 
e 00902308d update download source of thirdparty-brpc to internal brpc branch
pick dd4b7c0ae modify log for ~Compaction()
e 246b87efb Remove valid IP to get FE image for cloud-doris verify

# 变基 406c30cca..246b87efb 到 406c30cca（5 个提交）
#
# 命令:
# p, pick = 使用提交
# r, reword = 使用提交，但修改提交说明
# e, edit = 使用提交，但停止以便进行提交修补
# s, squash = 使用提交，但和前一个版本融合
# f, fixup = 类似于 "squash"，但丢弃提交说明日志
# x, exec = 使用 shell 运行命令（此行剩余部分）
# d, drop = 删除提交
#
# 这些行可以被重新排序；它们会被从上至下地执行。
#
# 如果您在这里删除一行，对应的提交将会丢失。
#
# 然而，如果您删除全部内容，变基操作将会终止。
#
# 注意空提交已被注释掉
```

* 保存并退出文档，返回如下提示信息。

```
停止在 2135f1e1d... update CI for doris-0.13-branch
You can amend the commit now, with

  git commit --amend 

Once you are satisfied with your changes, run

  git rebase --continue
```

* 根据提示信息执行 `git commit --amend`命令，在弹出的log文档中修改对应commit的Message，保存并退出log文档。

* 执行 `git rebase --continue`命令，继续修改下一个commit的Message。

* 根据提示信息执行 `git commit --amend`命令，在弹出的log文档中修改对应commit的Message，保存并退出log文档。

* 多次执行`git commit --amend`和执行 `git rebase --continue`命令，直到出现`Successfully rebased and updated refs/heads/...`信息，表示需要修改的所有commit的message均已修改完成。

##### git cherry-pick

将github官方仓库的一次特定commit提交到gitlab官方仓库。

* 克隆gitlab官方仓库的master分支到本地仓库。
```
git clone <gitlab官方仓库>
```

* 基于gitlab官方仓库的master分支在本地创建一个新的gitlab代码分支。
```
git checkout -b <本地新分支>
```

* 将github官方仓库添加为本地仓库的一个远程仓库。
```
git remote add <github官方远程主机> <github官方仓库>
```

* 同步github官方仓库的提交到本地仓库。
```
git fetch <github官方远程主机>
```

* 将github官方仓库的一次提交cherry-pick到本地新创建的gitlab代码分支。
```
git cherry-pick <github上的commit id>
```

如果github官方仓库的这一次commit与gitlab官方仓库的代码不存在冲突，则执行`git cherry-pick <github上的commit id>`命令之后，github官方仓库的这一次commit将被成功地提交到本地分支；否则需要手动处理冲突。手动处理冲突的方法如下：
a.  执行`git status`命令，查看冲突文件。
b. 编辑冲突文件，解决代码冲突。
解决完代码冲突之后，执行`git cherry-pick --continue`命令继续进行cherry-pick操作；否则，执行`git cherry-pick --abort`命令，放弃本次cherry-pick操作。

* 执行`git log`命令查看cherry-pick结果。

* 将gitlab个人远程仓库添加为本地仓库的一个远程仓库。
```
git remote add <gitlab个人远程主机> <gitlab个人远程仓库>
```

* 将cherry-pick之后的本地分支push到gitlab个人远程仓库的同名分支
```
git push --set-upstream <gitlab个人远程主机> <远程分支名>
```

* 向gitlab官方仓库提交PR，等待代码合并。
