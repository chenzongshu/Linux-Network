
Git作为近年来很火的版本管理工具,逐渐超越了SVN,下面记录下git的使用命令

# 安装

如果你是linux用户,可以通过yum或者apt来安装;windows用户可以安装msysGit

# 配置

使用Git的第一件事就是设置你的名字和email,这些就是你在提交commit时的签名。

```
$ git config --global user.name "czs"
$ git config --global user.email "czs@xx.com"
```

执行了上面的命令后,会在你的主目录(home directory)建立一个叫~/.gitconfig 的文件. 内容一般像下面这样:

```
[user]
name = Robin Hu
email = hudashi@gmail.com
```

这样的设置是全局设置,会影响此用户建立的每个项目.
如果你想使项目里的某个值与前面的全局设置有区别(例如把私人邮箱地址改为工作邮箱);你可以在项目中使用git config 命令不带--global 选项来设置.

# 初始化和创建

## git init
新建一个文件夹`project`,使用`git init`初始化

```
$ cd project
$ git init

Initialized empty Git repository in .git/  #输出
```

上述操作的结果是在project 目录下创建了一个.git 隐藏目录，它就是所谓的Git 仓库，不过现在它还是空的。

另外project 目录也不再是普通的文档目录了，今后我们将其称为工作树。每个工作树又包含着一个Git仓库

## git add
把工作树中的一些文档存储至Git仓库中。由于Git在向仓库中添加文档时并非是简单地文档复制过去，势必要将所添加文档进行一番处理，生成Git 仓库所能接受的数据格式，Git 称这个过程为"take a snapshot"(生成快照),就是使用`git add`命令

```
$ git add .   # 这里的 . 表示当前目录
$ git add a.txt b.txt #只add单独文件
```
git add命令所生成的快照被存放到一个临时的存储区域，Git 称该区域为索引

## git commit

使用git commit 命令可将索引提交至仓库中，这个过程称为提交，每一次提交都意味着
版本在进行一次更新

```
git commit -m "你的版本更新信息"
```

如果工作目录中有些文档是你不想提交的,可以使用忽略机制,将不希望接受Git管理的文档信息写到同一目录下的.gitignore 文件中.

```
$ echo "out" > .gitignore
```

每次`git commit`之前,需要`git add`操作,如果不想`git add`操作,可以使用

```
git commit -am "你的版本更新信息"
```
"-a"这个参数可以提交已经纳入stage区管理的文件,简单的说,就是如果有a.txt这个文件,如果第一次git add到了stage,然后修改了之后,不需要再git add,直接`git commit -am`


# 克隆(clone)项目

git clone命令可以使用git协议或者http协议

```
git clone git://git.kernel.org/pub/scm/git/git.git
git clone http://www.kernel.org/pub/scm/git/git.git
```

如果只想克隆某一分支,可以使用

```
git clone -b <branch> <remote_repo>
```

# 其他基础操作

## git status
列出当前目录所有还没有被git管理的文件和被git管理且被修改但还未提交(git commit)的文件

## git rm
把一个文件删除，并把它从git的仓库管理系统中移除。但是注意最后要执行git commit才真正提交到git仓库,删除文件带 '-r'参数

```
git rm -r myFolder
```

## git push
把本地仓库的更新推到服务器仓库

## git pull
把服务器仓库的更新拉到本地仓库中

## git log
查看项目日志

## git tag
打标签

## git show
显示代码改动情况

```
git show -U9999 > diff #显示代码改动情况,输出到diff文件
$ git show 214cf # 一般只使用版本号的前几个字符即可
$ git show HEAD #显示当前分支的最新版本的更新细节
$ git show # 显示当前分支的最新版本的更新细节，它相当于git show HEAD
每一个项目版本号通常都对应存在一个父版本号，也就是项目的前一次版本状态。
可使用如下命令查看当前项目版本的父版本更新细节：
$ git show HEAD^ # 查看HEAD 的父版本更新细节
$ git show HEAD^^ # 查看HEAD 的祖父版本更新细节
$ git show HEAD~4 # 查看HEAD 的祖父之祖父的版本更新细节
```

## git diff
显示代码改动情况

# 撤销

我们可以使用git reset或git checkout或git revert来撤销我们的修改。

## 撤销未提交的修改

如果你现在的工作目录(work tree)里搞的一团乱麻, 但是你现在还没有把它们提交; 你可以通过下面的命令, 让工作目录回到上次提交时的状态(last committed state):

```
$ git reset --hard HEAD
```

你这条件命令会把你所以工作目录中所有未提交的内容清空(当然这不包括未置于版控制下的文件 untracked files). 从另一种角度来说, 这会让"git diff" 和"git diff --cached"命令的显示法都变为空.

如果你只是要恢复一个文件,如"hello.rb", 你就要使用git checkout

```
$ git checkout -- hello.rb
```

这条命令把hello.rb从HEAD中签出并且把它恢复成未修改时的样子.
如果你想要恢复当前目录所有修改的文件,你可以使用

```
$ git checkout .
```
这条命令把 当前目录所有修改的文件 从HEAD中签出并且把它恢复成未修改时的样子.

## 撤销已提交的修改
如果你已经做了一个提交(commit),但是你马上后悔了, 

这里有两种截然不同的方法去处理这个问题:

- 创建一个新的提交(commit),在新的提交里撤消老的提交所作的修改.这种作法在你已经把代码发布(git push)到服务器的情况下十分正确.

- 你也可以去修改你的老提交(old commit). 但是如果你已经把代码发布(git push)到了服务器,那么千万别这么做;git不会处理项目的历史会改变的情况,如果一个分支的历史被改变了那以后就不能正常的合并.

### 1.创建新提交来撤销前期提交的修改 
创建一个新的提交来撤消(revert)前期某个提交(commit)的修改是很容易的;只要把你想撤销修改的某个提交(commit)的名字(reference)做为参数传给命令: git revert就可以了; 下面这条命令就演示了如何撤消最近的一个提交:

```
$ git revert HEAD
```

这样就创建了一个撤消了上次提交(HEAD)修改的新提交, 你就有机会来修改新提交(new commit)里的提交注释信息.你也可撤消更早期的修改,下面这条命令就是撤消“上上次”(next-to-last)的提交:

```
$ git revert HEAD^
```

撤销某个commit的提交

```
$git revert 4ab494a0bf5c5b09267a01ec03b587731d3034b4
```

这样就创建了一个撤消提交4ab494a0bf5c5b09267a01ec03b587731d3034b4修改的新提交,
在执行git revert 时,git尝试去撤消老提交的修改,如果你最近的修改和要撤消的修改有重叠(overlap),那么就会被求手工解决冲突(conflicts),　就像解决合并(merge)时出现的冲突一样.

另外， git revert 其实不会直接创建一个提交(commit),把撤消扣的文件内容放到索引(index)里,你需要再执行git commit命令，它们才会成为真正的提交(commit).

### 2.撤销旧提交但不保留修改
如果你刚刚做了一个或多个提交(commit), 但是你又想撤销这些提交，而且不保留这些提交的修改和文件系统中的修改，可以使用`git reset --hard`来达到该目的。


### 3.撤销旧提交但保留修改
如果你刚刚做了一个或多个提交(commit),但是你又想撤销这些提交，但是仍然在文件系统中保留这些修改，然后做修改或不做修改再次提交的话，可以使用`git reset --mixed`或`git reset --soft`来达到该目的。


### 4.追加提交来修改提交
如果你刚刚做了某个提交(commit), 但是你这里又想来马上修改这个提交; git commit 现在支持一个叫--amend的参数，你能让修改刚才的这个提交(HEAD commit). 这项机制能让你在代码发布前,添加一些新的文件或是修改你的提交注释(commit message).
  
  另外、如果你在老提交(older commit)里发现一个错误, 但是现在还没有发布(git push)到代码服务器上. 你可以使用git rebase命令的交互模式, "git rebase -i"会提示你在编辑中做相关的修改. 这样其实就是让你在rebase的过程来修改提交.
  

# 分支管理

前所讲内容未有提及项目分支问题，但事实上是有一个分支存在的，那就是master 分支（主分支），该分支是由Git自动产生的。在此之前，我们针对项目版本的各种操作都是在主分支上进行的，只是我们未察觉它的存在而已。

## 1.分支的创建

Git 可以轻松地产生新的项目分支，譬如下面操作可添加一个名曰“local”的新的项目分支：

```
$ git branch local
```

对于新产生的local 分支，初始时是完全等同于主分支的。但是，在local分支所进行的所有版本更新工作都不影响主分支，这意味着作为项目的参与者，可以在local中开始各种各样的更新尝试。


## 2.查看分支
查看项目仓库中存在多少分支，可直接使用git-branch命令，譬如使用该命令查看项目分支列表：

```
$ git branch
local
* master
```

在上述操作输出结果中，若分支名之前存在* 符号，表示此分支为当前分支。其实Git 各分支不存在尊卑之别，只存在哪个分支是当前分支的区别。为了某种良好的秩序，很多人默认是将master 分支视为主分支，本文也将沿用这一潜在规则。

## 3.分支的切换
由上述操作输出的分支列表可以看出，虽然使用git-branch 命令产生了local 分支，但是Git 不会自动将当前分支切换到local下。可使用git-checkout命令实现分支切换，下面操作将当前分支切换为前文所产生的local 分支：

```
$ git checkout local
```

## 4.分支的合并
我们产生了local 分支，并在该分支下进行了诸多修改与数次的版本更新提交，但是该如何将这一分支的最终状态提交到master 分支中呢？

`git-merge` 命令可实现两个分支的合并。譬如我们将local分支与master分支合并，操作如下：

```
$ git checkout master # 将当前分支切换为master
$ git merge local # 将local分支与当前分支合并
```

当一个分支检查无误并且与master分支成功合并完毕后，那么这一分支基本上就没有存在的必要性了，可以删除掉：

```
$ git branch -d local
```

注意：git-branch 的-d选项只能删除已经参与了合并的分支，对于未有合并的分支是无法删除的。如果想不问青红皂白地删除一个分支，可以使用git-branch 的-D 选项。

另外我们还可以通过git rebase来实现分支的合并。

在你经常使用的命令当中有一个`Git branch –a`用来查看所有的分支，包括本地和远程的。但是时间长了你会发现有些分支在远程其实早就被删除了，但是在你本地依然可以看见这些被删除的分支。

你可以通过命令，git remote show origin来查看有关于origin的一些信息，包括分支是否tracking。

另外可用git remote prune origin对本分支做清理，即远程已经删除了的分支，本地做清理，也删除掉。
