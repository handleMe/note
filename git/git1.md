

## git配置

Git 提供了一个叫做 git config 的工具，专门用来配置或读取相应的工作环境变量。

这些环境变量，决定了 Git 在各个环节的具体工作方式和行为。这些变量可以存放在以下三个不同的地方：

- /etc/gitconfig 文件：系统中对所有用户都普遍适用的配置。若使用 git config 时用 --system 选项，读写的就是这个文件。
- ~/.gitconfig 文件：用户目录下的配置文件只适用于该用户。若使用 git config 时用 --global 选项，读写的就是这个文件。
- 当前项目的 Git 目录中的配置文件（也就是工作目录中的 .git/config 文件）：这里的配置仅仅针对当前项目有效。每一个级别的配置都会覆盖上层的相同配置，所以 .git/config 里的配置会覆盖 /etc/gitconfig 中的同名变量。

### 配置个人的用户名称和电子邮件地址：
$ git config --global user.name "runoob"
$ git config --global user.email test@runoob.com

### 设置Git默认使用的文本编辑器, 
一般可能会是 Vi 或者 Vim。如果你有其他偏好，比如 Emacs 的话，可以重新设置：:
$ git config --global core.editor emacs

### 差异分析工具
还有一个比较常用的是，在解决合并冲突时使用哪种差异分析工具。比如要改用 vimdiff 的话：
$ git config --global merge.tool vimdiff

### 查看配置信息
要检查已有的配置信息，可以使用 git config --list 命令
也可以直接查阅某个环境变量的设定，只要把特定的名字跟在后面即可，像这样：
$ git config user.name
runoob

##　基本概念
工作区：就是你在电脑里能看到的目录。
暂存区：英文叫 stage 或 index。一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
版本库：工作区有一个隐藏目录 .git，这个不算工作区，而是 Git 的版本库。

## 创建仓库
git init
Git 使用 git init 命令来初始化一个 Git 仓库，Git 的很多命令都需要在 Git 的仓库中运行，所以 git init 是使用 Git 的第一个命令。
在执行完成 git init 命令后，Git 仓库会生成一个 .git 目录，该目录包含了资源的所有元数据，其他的项目目录保持不变

### 使用方式
使用当前目录作为Git仓库，我们只需使它初始化。该命令执行完后会在当前目录生成一个 .git 目录。
git init

使用我们指定目录作为Git仓库。
git init newrepo

初始化后，会在 newrepo 目录下会出现一个名为 .git 的目录，所有 Git 需要的数据和资源都存放在这个目录中。

如果当前目录下有几个文件想要纳入版本控制，需要先用 git add 命令告诉 Git 开始对这些文件进行跟踪，然后提交：
将以.c结尾以及README文件提交到仓库中
```
$ git add *.c
$ git add README
$ git commit -m '初始化项目版本'
```
## git clone
克隆仓库的命令格式为：
```git clone <repo>```
如果我们需要克隆到指定的目录，可以使用以下命令格式：
```git clone <repo> <directory>```

## 配置
编辑 git 配置文件:
$ git config -e    # 针对当前仓库 

或者：
$ git config -e --global   # 针对系统上所有仓库

## 提交与修改
### git add 命令
可将该文件添加到暂存区。
添加一个或多个文件到暂存区：
git add [file1] [file2] ...

添加指定目录到暂存区，包括子目录：
git add [dir]

添加当前目录下的所有文件到暂存区：
git add .

### git status 
命令用于查看项目的当前状态。

```
$ git status
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

    new file:   README
    new file:   hello.php
```

使用 **-s** 参数来获得简短的输出结果：

```
$ git status -s
?? README
?? hello.php
```

### git diff 命令

git diff 命令比较文件的不同，即比较文件在暂存区和工作区的差异。

- 尚未缓存的改动：**git diff**
- 查看已缓存的改动： **git diff --cached**
- 查看已缓存的与未缓存的所有改动：**git diff HEAD**
- 显示摘要而非整个 diff：**git diff --stat**

### git commit 命令
git commit 命令将暂存区内容添加到本地仓库中。
提交暂存区到本地仓库中:
```git commit -m [message]```
[message] 可以是一些备注信息。

提交暂存区的指定文件到仓库区：
```$ git commit [file1] [file2] ... -m [message]```

-a 参数设置修改文件后不需要执行 git add 命令，直接来提交
$ git commit -a

### git reset 命令
git reset 命令语法格式如下：
```git reset [--soft | --mixed | --hard] [HEAD]```
--mixed 为默认，可以不用带该参数，用于重置暂存区的文件与上一次的提交(commit)保持一致，工作区文件内容保持不变。

--soft 参数用于回退到某个版本：
git reset --soft HEAD
```$ git reset --soft HEAD~3 # 回退上上上一个版本```

--hard 参数撤销工作区中所有未提交的修改内容，将暂存区与工作区都回到上一次版本，并删除之前的所有信息提交：
git reset --hard HEAD

```
$ git reset HEAD^            # 回退所有内容到上一个版本  
$ git reset HEAD^ hello.php  # 回退 hello.php 文件的版本到上一个版本  
$ git  reset  052e           # 回退到指定版本
```
**谨慎使用 –hard 参数，它会删除回退点之前的所有信息。**

#### HEAD 说明：
HEAD 表示当前版本
HEAD^ 上一个版 本
HEAD^^ 上上一个版本
HEAD^^^ 上上上一个版本
以此类推...
可以使用 ～数字表示
HEAD~0 表示当前版本
HEAD~1 上一个版本
HEAD^2 上上一个版本
HEAD^3 上上上一个版本
以此类推.
#### git reset HEAD 
命令用于取消已缓存的内容。
```$ git reset HEAD hello.php ```

### git rm 命令
git rm 命令用于删除文件。

将文件从暂存区和工作区中删除：
```git rm <file>```
可以递归删除，即如果后面跟的是一个目录做为参数，则会递归删除整个目录中的所有子目录和文件：
```git rm –r * ```

以下实例从暂存区和工作区中删除 runoob.txt 文件：
```git rm runoob.txt ```

如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除选项 -f。
强行从暂存区和工作区中删除修改后的 runoob.txt 文件：
```git rm -f runoob.txt ```

如果想把文件从暂存区域移除，但仍然希望保留在当前工作目录中，换句话说，仅是从跟踪清单中删除，使用 --cached 选项即可：
```git rm --cached <file>```

以下实例从暂存区中删除 runoob.txt 文件：
```git rm --cached runoob.txt```

### git mv 命令
git mv 命令用于移动或重命名一个文件、目录或软连接。
```git mv [file] [newfile]```

如果新但文件名已经存在，但还是要重命名它，可以使用 -f 参数：
```git mv -f [file] [newfile]```

我们可以添加一个 README 文件（如果没有的话）：
```$ git add README ```
然后对其重命名:
```
$ git mv README  README.md
$ ls
README.md
```

## Git 查看提交历史


### git log - 查看历史提交记录。

```git log --oneline```可以查看历史记录的简洁版本

我们还可以用 --graph 选项，查看历史中什么时候出现了分支、合并。以下为相同的命令，开启了拓扑图选项：
```
*   d5e9fc2 (HEAD -> master) Merge branch 'change_site'
|\  
| * 7774248 (change_site) changed the runoob.php
* | c68142b 修改代码
|/  
* c1501a2 removed test.txt、add runoob.php
* 3e92c19 add test.txt
* 3b58100 第一次版本提交
```
可以用 --reverse 参数来逆向显示所有日志。
```
$ git log --reverse --oneline
3b58100 第一次版本提交
3e92c19 add test.txt
c1501a2 removed test.txt、add runoob.php
7774248 (change_site) changed the runoob.php
c68142b 修改代码
d5e9fc2 (HEAD -> master) Merge branch 'change_site'
```
如果只想查找指定用户的提交日志可以使用命令：git log --author
```
$ git log --author=Linus --oneline -5
81b50f3 Move 'builtin-*' into a 'builtin/' subdirectory
3bb7256 make "index-pack" a built-in
377d027 make "git pack-redundant" a built-in
b532581 make "git unpack-file" a built-in
112dd51 make "mktag" a built-in
```
如果你要指定日期，可以执行几个选项：--since 和 --before，但是你也可以用 --until 和 --after。
例如，如果我要看 Git 项目中三周前且在四月十八日之后的所有提交，我可以执行这个（我还用了 --no-merges 选项以隐藏合并提交）：
````
$ git log --oneline --before={3.weeks.ago} --after={2010-04-18} --no-merges
5469e2d Git 1.7.1-rc2
d43427d Documentation/remote-helpers: Fix typos and improve language
272a36b Fixup: Second argument may be any arbitrary string
b6c8d2d Documentation/remote-helpers: Add invocation section
5ce4f4e Documentation/urls: Rewrite to accomodate transport::address
00b84e9 Documentation/remote-helpers: Rewrite description
03aa87e Documentation: Describe other situations where -z affects git diff
77bc694 rebase-interactive: silence warning when no commits rewritten
636db2c t3301: add tests to use --format="%N"
```

### git blame 查看文件修改记录
```git blame <file>```
git blame 命令是以列表形式显示修改记录，如下实例：
```
$ git blame README 
^d2097aa (tianqixin 2020-08-25 14:59:25 +0800 1) # Runoob Git 测试
db9315b0 (runoob    2020-08-25 16:00:23 +0800 2) # 菜鸟教程
```

##　远程操作

### git remote 命令

显示所有远程仓库：
```git remote -v```

显示某个远程仓库的信息：
```git remote show [remote]```

添加远程版本库：
```git remote add [shortname] [url]```

shortname 为本地的版本库，例如：

提交到 Github
$ git remote add origin git@github.com:tianqixin/runoob-git-test.git
$ git push -u origin master
其他相关命令：

git remote rm name  # 删除远程仓库
git remote rename old_name new_name  # 修改仓库名


### git fetch 命令
git fetch 命令用于从远程获取代码库。

该命令执行完后需要执行 git merge 远程分支到你所在的分支。
从远端仓库提取数据并尝试合并到当前分支：
git merge

该命令就是在执行 git fetch 之后紧接着执行 git merge 远程分支到你所在的任意分支。
假设你配置好了一个远程仓库，并且你想要提取更新的数据，你可以首先执行:
git fetch [alias]
以上命令告诉 Git 去获取它有你没有的数据，然后你可以执行：
git merge [alias]/[branch]
以上命令将服务器上的任何更新（假设有人这时候推送到服务器了）合并到你的当前分支。

### git pull 命令
git pull 命令用于从远程获取代码并合并本地的版本。
git pull 其实就是 git fetch 和 git merge FETCH_HEAD 的简写。 命令格式如下：
git pull <远程主机名> <远程分支名>:<本地分支名>

更新操作：
```
$ git pull
$ git pull origin
```
将远程主机 origin 的 master 分支拉取过来，与本地的 brantest 分支合并。
```git pull origin master:brantest```

如果远程分支是与当前分支合并，则冒号后面的部分可以省略。
git pull origin master

### git push 命令

git push 命用于从将本地的分支版本上传到远程并合并。
命令格式如下：
```git push <远程主机名> <本地分支名>:<远程分支名>```

如果本地分支名与远程分支名相同，则可以省略冒号：
```git push <远程主机名> <本地分支名>```

以下命令将本地的 master 分支推送到 origin 主机的 master 分支。
```$ git push origin master```
相等于：
```$ git push origin master:master```

如果本地版本与远程版本有差异，但又要强制推送可以使用``` --force``` 参数：
```git push --force origin master```

删除主机但分支可以使用 ```--delete``` 参数，以下命令表示删除 origin 主机的 master 分支：
```git push origin --delete master```

## git 分支管理
几乎每一种版本控制系统都以某种形式支持分支。使用分支意味着你可以从开发主线上分离开来，然后在不影响主线的同时继续工作。

### 创建分支命令：
git branch (branchname)
### 切换分支命令:
git checkout (branchname)
当你切换分支的时候，Git 会用该分支的最后提交的快照替换你的工作目录的内容， 所以多个分支不需要多个目录。
合并分支命令:
git merge 

### 列出/创建分支 git branch
没有参数时，git branch 会列出你在本地的分支。

如果我们要手动创建一个分支。执行 git branch (branchname) 即可。
```
$ git branch testing
$ git branch
* master
  testing
```
使用 git checkout -b (branchname) 命令来创建新分支并立即切换到该分支下，从而在该分支中操作。
```
$ git checkout -b newtest
Switched to a new branch 'newtest'
$ git rm test.txt 
rm 'test.txt'
$ ls
README
$ touch runoob.php
$ git add .
$ git commit -am 'removed test.txt、add runoob.php'
[newtest c1501a2] removed test.txt、add runoob.php
 2 files changed, 1 deletion(-)
 create mode 100644 runoob.php
 delete mode 100644 test.txt
$ ls
README        runoob.php
$ git checkout master
Switched to branch 'master'
$ ls
README        test.txt
```

### 删除分支
删除分支命令：
git branch -d (branchname)

### 分支合并
git merge

将 newtest 分支合并到主分支去```$ git merge newtest```

##### 合并冲突
合并并不仅仅是简单的文件添加、移除的操作，Git 也会合并修改。

## git标签
如果你达到一个重要的阶段，并希望永远记住那个特别的提交快照，你可以使用 git tag 给它打上标签。
### 创建标签
比如说，我们想为我们的 runoob 项目发布一个"1.0"版本。 我们可以用 git tag -a v1.0 命令给最新一次提交打上（HEAD）"v1.0"的标签。
-a 选项意为"创建一个带注解的标签"。 不用 -a 选项也可以执行的，但它不会记录这标签是啥时候打的，谁打的，也不会让你添加个标签的注解。 我推荐一直创建带注解的标签。
```$ git tag -a v1.0 ```

指定标签信息命令：
```git tag -a <tagname> -m "runoob.com标签"```
PGP签名标签命令：
```git tag -s <tagname> -m "runoob.com标签"```

### 删除标签
```git tag -d v1.1```

### 查看标签所在的版本
```
[root@Git git]# git show v1.0
commit 91388f0883903ac9014e006611944f6688170ef4
Author: "syaving" <"819044347@qq.com">
Date: Fri Dec 16 02:32:05 2016 +0800
commit dir
diff –git a/readme b/readme
index 7a3d711..bfecb47 100644
— a/readme
+++ b/readme
@@ -1,2 +1,3 @@
text
hello git
+use commit
[root@Git git]# git log –oneline
91388f0 commit dir
e435fe8 add readme
2525062 add readme
```