---
title: 令人又爱又恨的git
date: 2019-12-10 22:19:50
categories: 工具
description: 实习期间所学的一些开发技巧。
---

前言：从一个学生到进入公司，一个很大的不同就是从单独开发者转变为需要团队合作的工程师。像我，之前用过git，但最多就是往github上我自己的仓库提交代码，基本没有使用过分支的经验。刚实习的时候，被告知要在分支上开发，开发完成再合并主分支。

我因为不熟悉分支，遇到过这些窘境：

比如：commit提交到本地分支了，切换到远程分支全没有了。解决办法：把本地和远程的分支和一下。

再比如：在我的分支上有的方法，与其他的预发分支合并也没问题。发上去应用构建居然报错找不方法？原来是别人发布了一次，此时主分支就和我不一样了。本地分支当然能看到该方法，但切到构建分支会发现并没有这个方法，这是因为构建的时候自动合并了。解决办法：把我的分支和主分支合并一下

再再比如：

Commit之后切换分支了，就commit不了，显示没有变更。解决方法：嘿嘿还是分支合并就vans了。



熟悉分支上开发后，在回头看当时的这些问题只是小case。👇这些可能的分支开发最常见的情况和对应的步骤：

情况：有人发布更新了主分支，此时最好把自己的分支merge一下主分支，相当于是更新一下。

对应的步骤：

1. git checkout master
2. git pull
3. git checkout 分支一
4. git merge master



情况：分支有冲突，要手动解决，比如上面的最后一步git merge master就有可能出现这种情况，这可能是最常见的情况了。

merge时如果有冲突是会自动合并的，但有些冲突git不知选择哪一个，只能手动解决冲突了。在你的合并命令后，**Auto-merging**的字样表示它自动合并了，而当出现了**大写的CONFLICT (content): Merge conflict in**的字样，你需要根据它的提示，去对应的页面解决冲突，**解决完成之后务必重新add一下文件，commit并且push到自己的分支。**

对应的步骤：

1. git fetch

2. git checkout 分支一

3. git merge origin/分支二

4. 解决冲突，提示CONFLICT即需要手动解决的会在控制台提示是什么类。并且在这个类上加上`<<<<<<<`，`=======`，`>>>>>>>`这种，跟着报错提示很容易找到地方。

5. git add -u  记得重新add一下否则会提示冲突没有解决

6. git commit -m 'XXXXX'

7. git push



顺便说一下pull和fetch的区别

`*git fetch`是将远程主机的最新内容拉到本地，用户在检查了以后决定是否合并到工作本机分支中。*

*而`git pull` 则是将远程主机的最新内容拉下来后直接合并，即：`git pull = git fetch + git merge`，这样可能会产生冲突，需要手动解决。*

为什么解决冲突第一步是要git fetch呢，这一步的目的主要是把刚创建的分支拉倒本地，因为分支冲突发生在你想和构建分支合并的时候，而这个构建分支一般是刚创建出来的，你不得更新一下？如果你要合并的两个分支已经在本地了，则可以跳过这一步。







## stash

stash的意思就是暂存，在你切换分支的时候，但你还有没有提交的更改，此时你有三种处理方法：1.commit（更改） 2.revert（复原）3.stash。

想象一下，你在自己的分支开发，突然要紧急处理一个bug，于是你新建了一个bug分支，工作写到一半还不想提交，但是bug得马上先修的尴尬局面。

1. git stash可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作：
2. `git checkout -b issue-101`在修复bug的分支上创建临时bug分支
3. 修完后add+提交，切换会原来的分支，合并，最后删除`issue-101`分支。
4. `git stash list`查看之前储藏起来的内容。用`git stash apply`或者`git stash pop`，恢复，后者会在恢复的同时删除stash。（list中的id号可以让你恢复指定的stash）
5. 如果其他分支也有该bug，git cherry-pick 4c805e2，`cherry-pick`能复制一个特定的提交到当前分支：

```
git stash
```

如果要丢弃一个没有被合并过的分支，可以通过`git branch -D <name>`强行删除。



## 版本

`git log`命令显示从最近到最远的提交日志，

```
git log --pretty=oneline
```

在Git中，用`HEAD`表示当前版本，上一个版本就是`HEAD^`，上上一个版本就是`HEAD^^`，当然往上100个版本写100个`^`比较容易数不过来，所以写成`HEAD~100`。

如果回到了原来的版本，git log看不到之前最新的提交记录了，但是git reflog还可以。只有知道版本号，就可还原之前的任何一个版本。

```
git reset --hard HEAD^
git reset --hard 1094a
```



`git add`命令实际上就是把要提交的所有修改放到暂存区（Stage），然后，执行`git commit`就可以一次性把暂存区的所有修改提交到分支。一旦提交后，如果你又没有对工作区做任何修改，那么工作区就是“干净”的。

`git status`可以查看一下状态，如果工作区进行了修改有没有add就会显示Untracked files

一个重要的概念是:Git跟踪并管理的是修改，而非文件。把文件git add readme.txt添加到了暂存区，暂存区也只是跟踪了这次修改，而不是跟踪了这个文件。换句话说：add不是一劳永逸之后对这个文件的更改都会被commit。如果再对readme.txt进行修改，就需要在进行一次git add，如果不再add，commit提交就不会提交这次修改。**简而言之：每次修改都要进行git add。**

删除文件：

```
 git rm test.txt
 git commit -m "remove test.txt"
```

如果是误删，版本库里还有，想恢复.`git checkout`其实是用版本库里的**最新**版本替换工作区的版本，无论工作区是修改还是删除，都可以“一键还原”。

```
git checkout -- test.txt
```



![git-repo](https://www.liaoxuefeng.com/files/attachments/919020037470528/0)



## 附注常用git命令

```
git checkout -b dev #创建并切换到dev分支
git checkout master #切换回master分支
git merge dev #在master分支上合并dev
git branch -d dev #删除dev分支
git branch #查看分支
```





## 冷知识

code review的时候一般代码已经提交了，此时可以通过和主分支比较看到两者的不同，点击主分支右键compare with current。

![Pasted Graphic](/Users/wangnini/Desktop/Pasted Graphic.png)

为什么Git添加文件需要`add`，`commit`一共两步呢？因为`commit`可以一次提交很多文件，所以你可以多次`add`不同的文件,也可以对一个文件进行不同更改.

切换分支和撤销修改是同一个关键词checkout，确实有点令人迷惑，不过切换分支的checkout也可以用switch替换.





强烈推荐这个教程，深入浅出.
https://backlog.com/git-tutorial/cn/stepup/stepup7_5.html