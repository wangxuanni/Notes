## add与commit

首先，选择一个合适的地方，创建一个空目录

通过`git init`命令把这个目录变成Git可以管理的仓库。可以发现当前目录下多了一个`.git`的目录，这个目录是Git来跟踪管理版本库的。

第一步，用命令`git add`告诉Git，把文件添加到仓库：

第二步，用命令`git commit`告诉Git，把文件提交到仓库：

```
git add readme.txt
git commit -m "wrote a readme file"
```

为什么Git添加文件需要`add`，`commit`一共两步呢？因为`commit`可以一次提交很多文件，所以你可以多次`add`不同的文件,也可以对一个文件进行不同更改



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



## 分支

因为创建、合并和删除分支非常快，所以Git鼓励你使用分支完成某个任务，合并后再删掉分支，这和直接在`master`分支上工作效果是一样的，但过程更安全。

切换分支和撤销修改是同一个关键词，确实有点令人迷惑，不过切换分支的checkout也可以用switch替换

```
git checkout -b dev #创建并切换到dev分支
git checkout master #切换回master分支
git merge dev #在master分支上合并dev
git branch -d dev #删除dev分支
git branch #查看分支
```

## 冲突

1. 如果两个分支的修改内容不一样，Git无法执行“快速合并”，必须手动选择。
2. git status查看是哪个文件出现了冲突
3. 打开这个文件，发现git会`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容，它保留了两个分支冲突的地方，需要手动进行修改。
4. 在提交就ok了， 删除`feature1`分支。git log --graph --pretty=oneline --abbrev-commit查看分支合并情况。

```
$ git merge feature1
Auto-merging readme.txt
CONFLICT (content): Merge conflict in readme.txt
Automatic merge failed; fix conflicts and then commit the result.
```



快速和并是看不出来曾经做过合并，禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

```
git merge --no-ff -m "merge with no-ff" dev
```



## stash

bug分支，工作写到一半还不能提交，但是bug得马上先修的尴尬局面。

1. git stash可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作：
2. `git checkout -b issue-101`在修复bug的分支上创建临时bug分支
3. 修完后add+提交，切换会原来的分支，合并，最后删除`issue-101`分支。
4. `git stash list`查看之前储藏起来的内容。用`git stash apply`或者`git stash pop`，恢复，后者会在恢复的同时删除stash。（list中的id号可以让你恢复指定的stash）
5. 如果其他分支也有该bug，git cherry-pick 4c805e2，`cherry-pick`能复制一个特定的提交到当前分支：

```
git stash
```

如果要丢弃一个没有被合并过的分支，可以通过`git branch -D <name>`强行删除。