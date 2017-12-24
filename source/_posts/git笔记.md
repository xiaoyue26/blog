---
title: git笔记
date: 2016-12-30 19:15:35
tags: git
categories: git
---

git教程:
[http://rogerdudler.github.io/git-guide/index.zh.html][1]

工作流:
>本地仓库由 git 维护的三棵“树”组成。第一个是你的 工作目录，它持有实际文件；第二个是 暂存区（Index），它像个缓存区域，临时保存你的改动；最后是 HEAD，它指向你最后一次提交的结果。
![此处输入图片的描述][2]

- 创建新仓库
创建新文件夹，打开，然后执行 
`git init`
以创建新的 git 仓库。

- 设置用户名和邮箱
```
git config --global user.name xiaoyue26
git config --global user.email 296671657@qq.com
git config --global color.ui true
```

## 在项目目录下`ls -a`查看隐藏的git配置文件
## 修改.gitignore文件，例如在里面另起一行，添加：`*.zip`,则可以忽略zip后缀名的文件。
```
-- 忽略除了lib.a以外的.a后缀文件
*.a
!lib.a
```

`git status`命令看到的绿色的就是已经stage的文件，
红色的就是还没有`git add`的文件(unstaged)。
##　unstage 删除远端没有 本地stage的文件。
```
 git rm --cached <file>... 
```
## 暂存(慎用)
想暂存到目前为止作出的修改，但是不想提交到版本库，git提供了一个命令git stash，所有修改会提交到栈中

## git commit 的时候把时间改成明天：
```
-- 网上查到明天的unix时间戳是1451750400
git commit --date 1451750400
```

## 撤销最后一次commit:
`git reset --soft HEAD~1`

## 从远端还原一个文件：
`git checkout config.rb`

## 查看远端信息：
```shell
git remote -v
git pull origin master
githug
```

## 向本地仓库添加远程版本库，名叫origin
`git remote add origin https://github.com/githug/githug`

>合并分支，我们可以使用rebase，这个题目里，本地master提交了Thrid commit，而远程分支也提交了Fourth commit，现在需要把远程更新合并到本地，可以使用下面的命令。
```
git rebase origin/master
git push origin master
```

## 查看`config.rb`文件每一行的作者：
`git blame config.rb`

##检出仓库
执行如下命令以创建一个本地仓库的克隆版本：
```git clone /path/to/repository ```
如果是远端服务器上的仓库，你的命令会是这个样子：
```git clone username@host:/path/to/repository```

# 用git定位代码从哪一次commit后开始出错：
原理：二分查找
背景条件：
1. `ruby prog.rb 5`命令在当前代码条件下应该返回正确答案15.
2. 从过去的某一次commit后的代码之后，这个结果就错了。
现在要用git bisect找出到底是哪一次之后就出错了。

```shell

# 查找正确那次的commit hash:
git log
# 例如已知f608824那次commit运行`ruby prog.rb 5`命令的结果确实是15.
# 告诉git bisect查找的起点和终点：
git bisect start -- 开始查找
git bisect good f608824 -- 起点(正确)
git bisect bad master -- 终点(错误)
#  交互地验证(类似监督学习)
ruby prog.rb 5 #看结果是否 15 
# 1. 如果上述命令结果正确(15)，则输入：
git bisect good
# 3. 如果上述命令结果错误(不是15)，则输入：
git bisect bad

# 反复进行上述步骤，直到出现提示：
>18ed2ac1522a014412d4303ce7c8db39becab076 is the first bad commit
commit 18ed2ac1522a014412d4303ce7c8db39becab076
Author: Robert Bittle <guywithnose@gmail.com>
Date:   Mon Apr 23 06:52:10 2012 -0400

```










##添加和提交
你可以提出更改（把它们添加到暂存区），使用如下命令：
```git add <filename>
git add *```
这是 git 基本工作流程的第一步；使用如下命令以实际提交改动：
```git commit -m "代码提交信息"```
现在，你的改动已经提交到了 `HEAD`，但是还没到你的远端仓库。

##推送改动
你的改动现在已经在本地仓库的 HEAD 中了。执行如下命令以将这些改动提交到远端仓库：
```git push origin master```
可以把 master 换成你想要推送的任何分支。 

如果你还没有克隆现有仓库，并欲将你的仓库连接到某个远程服务器，你可以使用如下命令添加：
```git remote add origin <server>```
如此你就能够将你的改动推送到所添加的服务器上去了。

## 切换到tag v1.2 :
`git checkout v1.2`
或者明确指定是tag：
`git checkout tags/v1.2`

###分支
创建一个叫做`my_branch`的分支：
`git branch my_branch`
创建一个叫做“feature_x”的分支，并切换过去：
```git checkout -b feature_x```
上述都是在当前代码基础上创建分支，也可以在任意一次commit的基础上创建分支：
```
# 先查看commit的hash标记
git log
# 创建分支
git branch test_branch -v 27b1836a65de099f6ebddb5998
```

切换回主分支：
```git checkout master```
查看当前分支情况：(类似git status)
`git branch `
push到指定分支上：
`git push --set-upstream origin test_branch`
其中：
>origin 是默认的远程版本库名称
可以在 .git/config 之中进行修改

把`feature`分支合并到当前分支:
`git merge feature`

不merge但是获取远端的东西：
`git fetch`
>git fetch：相当于是从远程获取最新版本到本地，不会自动merge
git pull：相当于是从远程获取最新版本并merge到本地

git rebase和git merge区别：
>1. 在B分支上执行 git merge A 后 A就被合到B上了
2. 在B分支上执行 git rebase B A 后，效果与merge是一样的，但是 A就没有了，两个分支就合在一起了。

git rebase分支：
`git rebase master feature`

用`git gc`代替`git repack`命令。

# cherry-pick 从别的分支提取commit(提取文件)。
```
# 切换到feature分支:
git checkout feature
# 查看log树:
git log
# 找到添加和修改想要文件那次commit的hash ca32a6...
# 切换回master分支:
git checkout master
# 提取文件：
git cherry-pick ca32a6dac7b6f97
```

查看还有多少个`TODO`:
`git grep TODO`

修改很久以前的注释,(修改上一次的直接git commit --amend -m就行了)
```
# 往前倒到上两次的：
git rebase -i HEAD~2
# 把pick改成reword,并修改注释。
```

合并前四次commit:
```
# 通过git log确定要合并的commit有几次
git log
# 修改前四次的：
git rebase -i HEAD~4
# 这时的顺序是从按时间最早到最晚排列的，所以把后面几个pick都改成s然后退出保存即可。
```

合并分支时，把分支的多次提交变成一次提交。
```
# 跟上面的类似，但是从别的分支取出几个commit,然后在本分支上体现为一个commit:
git merge --squash long-feature-branch
git commit -m "commit from merge branch"
```

修改commit的顺序：
```
git rebase -i HEAD~3
然后把里面的几行顺序换一下就好了。
```

再把新建的分支删掉：
```git branch -d feature_x```
除非你将分支推送到远端仓库，不然该分支就是 不为他人所见的：
```git push origin <branch>```
>origin 是默认的远程版本库名称你可以在 `.git/config` 之中进行修改.
>事实上 `git push origin master` 的意思是 `git push origin master:master` （将本地的 master 分支推送至远端的 master 分支，如果没有就新建一个）

##更新与合并
要更新你的本地仓库至最新改动，执行：
```git pull```
以在你的工作目录中 获取（fetch） 并 合并（merge） 远端的改动。
要合并其他分支到你的当前分支（例如 master），执行：
```git merge <branch>```
在这两种情况下，git 都会尝试去自动合并改动。遗憾的是，这可能并非每次都成功，并可能出现冲突（conflicts）。 这时候就需要你修改这些文件来手动合并这些冲突（conflicts）。改完之后，你需要执行如下命令以将它们标记为合并成功：
```git add <filename>```
在合并改动之前，你可以使用如下命令预览差异：
```git diff <source_branch> <target_branch>```

##标签
为软件发布创建标签是推荐的。这个概念早已存在，在 SVN 中也有。你可以执行如下命令创建一个叫做 1.0.0 的标签：
```git tag 1.0.0 1b2e1d63ff```
`1b2e1d63ff` 是你想要标记的提交 ID 的前 10 位字符。可以使用下列命令获取提交 ID：
```git log```
你也可以使用少一点的提交 ID 前几位，只要它的指向具有唯一性。

##替换本地改动
假如你操作失误（当然，这最好永远不要发生），你可以使用如下命令替换掉本地改动：
```git checkout -- <filename>```
此命令会使用 HEAD 中的最新内容替换掉你的工作目录中的文件。已添加到暂存区的改动以及新文件都不会受到影响。

假如你想丢弃你在本地的所有改动与提交，可以到服务器上获取最新的版本历史，并将你本地主分支指向它：
```
git fetch origin
git reset --hard origin/master
```

##实用小贴士
内建的图形化 git：
`gitk`
彩色的 git 输出：
`git config color.ui true`
显示历史记录时，每个提交的信息只显示一行：
`git config format.pretty oneline`
交互式添加文件到暂存区：
`git add -i`

进阶教程:
[http://marklodato.github.io/visual-git-guide/index-zh-cn.html][3]










1. 开分支:
``` git branch 新分支名```
 如新建一个开发分支:
``` git branch dev```
2. 切换到某分支:
``` git checkout 分支名```
3. 合并上述两个命令:
``` git checkout -b 新分支名```
4. 合并分支:
``` git merge 需要合并的分支名```
5. 查看本地分支列表:
``` git branch -a ```
6. 查看远端分支列表:
```git branch -r```
7. 向远程分支提交本地新开的分支:
```git push origin 新分支名```
8. 删除远程分支:(多一个空格和冒号)
```git push origin :远程分支名```
原理就是把一个空的branch赋值给已有的branch，这样就删除了。
9. 删除本地分支:
```git branch 分支名称 -d```
10. 拉取远端:
```git pull origin master```
11. 更新分支列表信息:
```git fetch -p```
12. 查看帮助:
```shell
git -h
git branch -h
git pull -h
```

- 其他
```shell
git push origin :branch_you_want_to_delete
#注意空格
#下述命令没使用过,不知道含义 :
#查看git branch -h时得知-r表示对远端生效，-d表示删除。
#git branch -r -d origin/branch-name
```



竟然没有commit和pull.
```
git commit fileA;
git pull ;
```



```shell
#安装brew:
curl -LsSf http://github.com/mxcl/homebrew/tarball/master | sudo tar xvz -C/usr/local --strip 1
#查看brew软件列表:
sudo brew search /apache*/
#安装软件:
sudo brew install wget  
```
- 给git log 上颜色
`git log --decorate --graph`
git commit --amend
直接删掉changeid
会自动生成新的changeid

- git add时(stage)先进行diff和修改
```
git add feature.rb -e
```

查看切换分支的记录log:
 
`git reflog`
 
结果：
>894a16d HEAD@{0}: commit: commit another todo
6876e5b HEAD@{1}: checkout: moving from solve_world_hunger to kill_the_batman
324336a HEAD@{2}: commit: commit todo
6876e5b HEAD@{3}: checkout: moving from blowup_sun_for_ransom to solve_world_hunger
6876e5b HEAD@{4}: checkout: moving from kill_the_batman to blowup_sun_for_ransom
6876e5b HEAD@{5}: checkout: moving from cure_common_cold to kill_the_batman
6876e5b HEAD@{6}: commit (initial): initial commit

然后可以切换到想去的分支:
`git checkout solve_world_hunger`

取消一个已经提交(push)的commit,原理其实是提交一次revert操作的commit.
通过`git log`查看想要取消的那次commit的hash值。
```
git revert d71adf7ad90cc0206c2076b
# 或者如果是倒数第二次的commit:
git revert HEAD~1
```

删除最近一次提交:
`git reset --hard HEAD^`
取消删除最近一次提交:
`git reflog`查看到倒数第二个commit的hash然后：
`git checkout b5ed1ea`


git submodule add 仓库地址 路径



  [1]: http://rogerdudler.github.io/git-guide/index.zh.html
  [2]: http://rogerdudler.github.io/git-guide/img/trees.png
  [3]: http://marklodato.github.io/visual-git-guide/index-zh-cn.html