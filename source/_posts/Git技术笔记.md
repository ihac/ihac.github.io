---
title: Git技术笔记
date: 2017-05-11 16:47:54
tags: [Study Notes, Git]
---

　　这篇博文主要是对Git（而非Github）背景知识、管理操作的摘抄与整理，内容来源包括但不限于[Pro Git](https://git-scm.com/book/en/v2)、[Stack Overflow](https://stackoverflow.com/questions/tagged/git)。

## 版本控制系统的演变

- 保存文件修改记录 --> 本地版本控制系统，如RCS。
- 与其他开发者合作 --> 中心化版本控制系统，如CVS、Subversion、Perforce。
- 去除单点故障 --> 分布式版本控制系统，如Git、Mercurial、Bazaar、Darcs。

## Git历史

1. 2002年开始，Linux内核社区使用BitKeeper对代码进行版本控制。
2. 2005年，Linux内核社区与BitKeeper社区闹翻，Linus开始设计新版本系统。
3. Git最初的设计目标：
	- 速度
	- 简单
	- 支持非线性开发（多个并行分支）
	- 分布式
	- 支持大工程

## Git基本特性

- 其他版本控制系统（如CVS、Subversion、Perforce、Bazaar等等）主要记录文件及对文件的一系列修改操作；而Git记录文件在不同版本下的快照。
- Git将工程的完整历史都保存在本地，因此受网络延迟的影响非常小。
- Git对所有文件计算SHA-1校验和，用于区分文件，因此不可能绕开Git对文件进行修改，这保证了文件的完整性。
- Git通常只允许添加数据，很难让系统执行不可逆操作或者删除数据。
- Git拥有三个主要状态，分别为committed、modified与staged。

## Git首次配置

### 配置文件
- `/etc/gitconfig` 全局配置文件
- `~/.gitconfig`、`~/.config/git/config` 用户配置文件
- `.git/config` 项目配置文件

### 配置命令

``` bash
$ git config --global user.name <user_name>
$ git config --global user.email <user_email>
$ git config --global core.editor <editor_name>
$ git config -l
```

## 基本操作

1. 初始化一个仓库
``` bash
$ git init
$ git add *.c
$ git commit -m "initial project version"
```

2. 克隆已存在的仓库
``` bash
$ git clone https://github.com/xxx/yyy.git [rename_dir]
```

3. 检查文件状态
``` bash
$ git status [-s]
 M README
MM Rakefile
A  lib/git.rb
M  lib/simplegit.rb
?? LICENSE.txt
```
    当使用`-s`简化输出时，文件状态用两列字符来表示，其中左列表示staging area的状态，右列表示working tree的状态。对于字符而言，`??`表示untracked文件，`M`表示modified文件，`A`表示added文件。

    因此可以看出，`README`已被修改，但是并没有添加到staging area中，而`Rakefile`在修改并添加到staging area后，再次被修改。

4. 忽略文件
```
# ignore all .a files
*.a
# but do track lib.a, even though you're ignoring .a files above
!lib.a
# only ignore the TODO file in the current directory, not subdir/TODO
/TODO
# ignore all files in the build/ directory
build/
# ignore doc/notes.txt, but not doc/server/arch.txt
doc/*.txt
# ignore all .pdf files in the doc/ directory and any of its subdirectories
doc/**/*.pdf
```
	规则如下：
	- 匹配符
	- `#`表示注释
	-  `!`取反
	-  `/`+pattern用于防止递归匹配
	-  pattern+`/`用于表示目录

5. 查看修改内容
``` bash
$ git diff [--staged|cached]
```

6. 提交修改
``` bash
$ git commit [-v|s|a] [-m "message"]
# -v对修改内容进行diff，打印出来
# -a提交所有修改内容，不管是否已经staged
```

7. 删除文件
``` bash
$ git rm [--cached] [-f]
# --cached表示从staged area中删除
```

8. 移动文件
``` bash
$ git mv file_from file_to
```

9. 查看提交历史
``` bash
$ git log [-p] [-<num>] [--stat] [--pretty=<format>] [--graph]
# -p对提交内容进行diff
# --stat查看统计内容
```

    使用`git log -S symbol_name`查看对该符号进行过修改的提交记录

10. 重新提交，覆盖上一次commit
``` bash
$ git commit --amend
```

11. 撤销staged状态
``` bash
$ git reset HEAD <file>
```

12. 撤销对文件的修改
``` bash
$ git checkout -- <file>
```

13. 打印远程仓库地址
``` bash
$ git remote -v
```

14. 添加远程仓库
``` bash
$ git remote add <shortname> <url>
```

15. 从远程仓库获取更新
``` bash
$ git fetch <remote-name>
# 不会自动和本地分支合并
$ git pull <remote-name>
# 自动fetch并和本地当前分支合并
```

16. 更新远程仓库
``` bash
$ git push [remote-name] [branch-name]
```

17. 检查远程仓库
``` bash
$ git remote show [remote-name]
```

18. 修改/删除远程仓库名字
``` bash
$ git remote rename <old-name> <new-name>
$ git remote remove <remote-name>
```

19. 打印标签
``` bash
$ git tag [-l "v1.8.5*"]
```

20. 新建标签
``` bash
# annotated tags
$ git tag -a v1.4 -m "my version 1.4"
# lightweight tags
$ git tag v1.4-lw

$ git show v1.4
```

21. 为之前的提交历史添加标签
``` bash
$ git tag -a v1.2 <checksum>
```

22. 提交标签
`git push`默认不会上传标签，必须显式地说明。
``` bash
$ git push origin [tagname]
# or
$ git push origin --tags
```

23. 将工作目录切换到特定标签下
``` bash
# 需要新建一个分支
$ git checkout -b version2 v2.0.0
```
    需要注意的是，当该分支有了新的修改及提交后，原标签不会同步这一改动。

24. 命令别名
``` bash
$ git config --global alias.co checkout
$ git config --global alias.br branch
$ git config --global alias.unstage 'reset HEAD --'
# use '!' to run an external command
$ git config --global alias.visual '!gitk'
```

## 分支管理

1. 创建新分支
``` bash
git branch <new_branch>
# or
git checkout -b <new_branch>
# == git branch + git checkout
git checkout -b <new_branch> <remote>/<branch>
# 以远端分支为base
```
2. 切换分支
``` bash
git checkout <branch_name>
```
3. 查看分支历史
``` bash
git log --oneline --decorate --graph --all
```
4. 删除分支
``` bash
git checkout -d/D <branch_name>
# -D 强制删除，不会检测是否已经被合并
```
5. 合并分支
```bash
git merge <branch_name>
# 将<branch_name>合并到HEAD指向的分支
```
6. 解决分支合并冲突
``` bash
# 查看冲突文件
git status 
# 手动修改冲突内容
# vim ...
# 完成merge
git add && git commit
```
7. 列举分支
``` bash
git branch [-v] [--merged/no-merged] [<branch_name>]
# -v 是否显示各分支最后一次commit
# --merged/no-merged 仅显示已经（或尚未）合并到当前分支的分支
# <branch_name> 配合上一选项使用，以指定分支（而非当前分支）作为基准
```
8. 列举远端分支
```bash
# 远端分支在本地用指针<remote>/<branch>来表示
git ls-remote [remote]
# or
git remote show [remote]
```
9. 同步远端分支
``` bash
git fetch origin
# 更新本地缓存的远端分支
```
10. 更新远端分支
```bash
git push <remote> <local_branch>:<remote_branch>
# 将本地的<local_branch>分支push到远端的<remote_branch>分支
# or
git push <remote> <branch>
# 将本地的<branch>分支push到远端的<branch>分支
```
11. 缓存认证信息
```bash
# 短时间内省去密码输入操作
git config --global credential.helper cache
```
12. tracking branch
tracking branch是与远端分支直接对应的本地分支，进行pull操作时，git能够自动提供server地址与需要merge 的分支名。
``` bash
git checkout -b <new_branch> <remote>/<branch>
# or
git checkout --track <remote>/<branch>
# or
git checkout <new_branch>
# 当<new_branch>与本地缓存的远端分支同名时，可以自动新建一个tracking branch
```
13. 修改upstream branch
``` bash
git branch -u/--set-upstream-to <remote>/<branch>
# 修改当前tracking branch的upstream branch
```
14. upstream branch别名
当前tracking branch的upstream branch可以用@{u}、@{upstream}来表示。
``` bash
# @{u}, @{upstream}
git merge @{u}
# or
git checkout @{u}
```
15. 列举track关系
``` bash
git branch -vv
```
16. 删除远端分支
``` bash
git push <remote> --delete <branch>
```
17. rebase
``` bash
git rebase <branch>
# 将当前分支rebase到<branch>上
git rebase <base_branch> <topic_branch>
# 将<topic_branch>分支rebase到<base_branch>上
git rebase --onto <base_branch> <new_branch1> <new_branch2>
# 以<base_branch>作为base，将<new_branch2>从<new_branch1>分叉后的所有commit重新apply一遍，具体应用见下
git rebase -i <commit_hash_value>
# 基于<commit_hash_value>对应的commit，对后续commit进行交互式rebase操作，用于调整git历史
```
    初始状态
    ![before](https://git-scm.com/book/en/v2/images/interesting-rebase-1.png)
rebase命令：`$ git rebase --onto master server client
`
    结果：
    ![](https://git-scm.com/book/en/v2/images/interesting-rebase-2.png)
18. 自动rebase
``` bash
git pull --rebase
# 采用rebase，而非merge
# == git fetch + git rebase <remote>/<branch>
git config --global pull.rebase true
# 设置默认采用rebase
```
19. 注意事项
**Never rebase anything you’ve pushed somewhere**.
If you treat rebasing as a way to clean up and work with commits before you push them, and if you **only rebase commits that have never been available publicly**, then you’ll be fine. If you rebase commits that have already been pushed publicly, and people may have based work on those commits, then you may be in for some frustrating trouble, and the scorn of your teammates.

---

*To Be Continued*


