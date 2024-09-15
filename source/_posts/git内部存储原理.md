---
title: git内部存储原理
date: 2024-06-10 13:47:40
categories: 
- git
---
cs61b的一个实验是mini-git，借此看看git内部是如何存储数据的。
# Git对象
## blob object
Git内部对象的存储是一个kv库。key为SHA-1的哈希值，value为具体文件。接下来我们通过一个小小的实验来看看。
首先需要初始化一个git仓库。
```console
$  go init test
Initialized empty Git repository in /demo/test/.git/
$  cd test
$  find .git/objects
.git/objects #用于保存blob对象
.git/objects/info
.git/objects/pack
```
接着我们可以用`git hash-object`向这个kv库中添加数据对象。 `-w` 选项会指示该命令不要只返回键，还要将该对象写入数据库中。 最后，`--stdin` 选项则指示该命令从标准输入读取内容；若不指定此选项，则须在命令尾部给出待存储文件的路径。
```console
echo 'test content' | git hash-object -w --stdin
d670460b4b4aece5915caf5c68d12f560a9fe3e4
```
接着我们可以用`find .git/objects -type f`查看具体有些什么对象了。
```console
$ find .git/objects -type f
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```
接着你可以用`git cat-file -p `通过key查看具体blob内容。
```
$ git cat-file -p d670460b4b4aece5915caf5c68d12f560a9fe3e4
test content
```
现在我们已经大致明白这个kv库的工作模式。我们开始尝试对文件进行版本管理。首先我们先创建一个test.txt，并写入一定的内容。
```
$ echo 'version 1' > test.txt
$ git hash-object -w test.txt
83baae61804e65cc73a7201a7252750c76066a30
```
这里我们可以再试试`git cat-file -p`命令，
```
$ git cat-file -p 83baae61804e65cc73a7201a7252750c76066a30
test content
```

接着我们创建版本2
```
$ echo 'version 2' > test.txt
$ git hash-object -w test.txt
1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
```

可以尝试用`find .git/objects -type f`命令查看一些有几个对象了。
```
$ find .git/objects -type f
.git/objects/1f/7a7a472abf3dd9643fd615f6da379c4acb3e3a
.git/objects/83/baae61804e65cc73a7201a7252750c76066a30
.git/objects/d6/70460b4b4aece5915caf5c68d12f560a9fe3e4
```
以上这些对象都只是一个内容的存储者，我们可以称之为blob object，其并没有保存比如文件名之类的信息。我们可以使用`git cat-file -t`查看blob类型。
```
$ git cat-file -t 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
blob
```
## tree object
tree object的出现就是为了解决文件名的存储问题。具体我们来看看。
通常，Git 根据某一时刻暂存区所表示的状态创建并记录一个对应的树对象。 因此，为创建一个树对象，首先需要通过暂存一些文件来创建一个暂存区。
可以通过底层命令 `git update-index` 为一个单独文件——我们的 test.txt 文件的首个版本——创建一个暂存区。
- `--add`：将指定的文件信息添加到暂存区中。
-  `--cacheinfo`：手工指定文件的模式(权限)、SHA-1 哈希值和文件名。
- `100644` 指这是一个普通文件。
```console
$ git update-index --add --cacheinfo 100644 \
83baae61804e65cc73a7201a7252750c76066a30 test.txt
```
现在，可以通过 `git write-tree` 命令将暂存区内容写入一个树对象。
```console
$ git write-tree
d8329fc1cc938780ffdd9f94e0d364e0ea74f579
$ git cat-file -p d8329fc1cc938780ffdd9f94e0d364e0ea74f579
100644 blob 83baae61804e65cc73a7201a7252750c76066a30    test.txt
```
我们可以使用`git cat-file -t`验证其为tree object。
```console
git cat-file -t d8329fc1cc938780ffdd9f94e0d364e0ea74f579
```
接着我们来创建一个新的树对象，它包括 `test.txt` 文件的第二个版本，以及一个新的文件：
```console
$ echo 'new file' > new.txt
$ git update-index --add --cacheinfo 100644 \
  1f7a7a472abf3dd9643fd615f6da379c4acb3e3a test.txt
$ git update-index --add new.txt
```
再跑一跑上面的步骤创建tree object。
```console
$ git write-tree
0155eb4229851634a0f03eb265b69f5a2d56f341
$ git cat-file -p 0155eb4229851634a0f03eb265b69f5a2d56f341
100644 blob fa49b077972391ad58037050f2a75f74e3671e92    new.txt
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a    test.txt
```
当然也支持将第一个tree object1添加到tree object2
```console
$ git read-tree --prefix=bak d8329fc1cc938780ffdd9f94e0d364e0ea74f579
$ git write-tree
3c4e9cd789d88d8d89c1073707c3585e41b0e614
$ git cat-file -p 3c4e9cd789d88d8d89c1073707c3585e41b0e614
040000 tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579      bak
100644 blob fa49b077972391ad58037050f2a75f74e3671e92      new.txt
100644 blob 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a      test.txt
```
现在你的tree object大概是这样：
![](images/git暂存区.png)
## commit 对象
如果你做完了以上所有操作，那么现在就有了三个树对象，分别代表我们想要跟踪的不同项目快照。 然而问题依旧：若想重用这些快照，你必须记住所有三个 SHA-1 哈希值。 并且，你也完全不知道是谁保存了这些快照，在什么时刻保存的，以及为什么保存这些快照。 而以上这些，正是commit object能为你保存的基本信息。
可以通过调用 `commit-tree` 命令创建一个提交对象，为此需要指定一个树对象的 SHA-1 值，以及该提交的父提交对象（如果有的话）。 我们从之前创建的第一个树对象开始：
```
$ echo 'first commit' | git commit-tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
2f907b2fd7fef414713baabbd6db08cfcee8bdff
$ git cat-file -p 2f907b2fd7fef414713baabbd6db08cfcee8bdff
tree d8329fc1cc938780ffdd9f94e0d364e0ea74f579
author silky1313 <silky1313@gmail.com> 1723259642 +0800
committer silky1313 <silky1313@gmail.com> 1723259642 +0800

first commit

```

提交对象的格式很简单：它先指定一个顶层树对象，代表当前项目快照； 然后是可能存在的父提交（前面描述的提交对象并不存在任何父提交）； 之后是作者/提交者信息（依据你的 `user.name` 和 `user.email` 配置来设定，外加一个时间戳）； 留空一行，最后是提交注释。
接着，我们将创建另两个提交对象，它们分别引用各自的上一个提交（作为其父提交对象）：
```
$ echo 'second commit' | git commit-tree  0155eb4229851634a0f03eb265b69f5a2d56f341 -p 2f907b2fd7fef414713baabbd6db08cfcee8bdff
cf5d1d819f739c7e4a124c94996735b942bd2ea8
$ echo 'second commit' | git commit-tree  0155eb4229851634a0f03eb265b69f5a2d56f341 -p cf5d1d819f739c7e4a124c94996735b942bd2ea8
e3d828ca5b56c97911317033bf8243af31c15fc2
```
这三个提交对象分别指向之前创建的三个树对象快照中的一个。 现在，如果对最后一个提交的 SHA-1 值运行 `git log` 命令，会出乎意料的发现，你已有一个货真价实的、可由 `git log` 查看的 Git 提交历史了：
```console
$ git log --stat e3d828ca5b56c97911317033bf8243af31c15fc2
commit e3d828ca5b56c97911317033bf8243af31c15fc2
Author: silky1313 <silky1313@gmail.com>
Date:   Sat Aug 10 11:16:53 2024 +0800

    second commit

commit cf5d1d819f739c7e4a124c94996735b942bd2ea8
Author: silky1313 <silky1313@gmail.com>
Date:   Sat Aug 10 11:16:26 2024 +0800

    second commit

 new.txt  | 1 +
 test.txt | 2 +-
 2 files changed, 2 insertions(+), 1 deletion(-)

commit 2f907b2fd7fef414713baabbd6db08cfcee8bdff
Author: silky1313 <silky1313@gmail.com>
Date:   Sat Aug 10 11:14:02 2024 +0800

    first commit

 test.txt | 1 +
 1 file changed, 1 insertion(+)
```
现在内部的对象关系大概是这样：
![](images/git内部关系图.png)

# 备忘录
test content -> d670460b4b4aece5915caf5c68d12f560a9fe3e4
test.txt -> version 1 -> 83baae61804e65cc73a7201a7252750c76066a30
test.txt -> version 2 -> 1f7a7a472abf3dd9643fd615f6da379c4acb3e3a
tree object 1-> test.txt version1  -> d8329fc1cc938780ffdd9f94e0d364e0ea74f579
tree object2 ->test.txt version2 new.txt -> 0155eb4229851634a0f03eb265b69f5a2d56f341
tree object3 -> tree1 + tree2 -> 3c4e9cd789d88d8d89c1073707c3585e41b0e614
commit1 -> tree1 -> 2f907b2fd7fef414713baabbd6db08cfcee8bdff
commit2 -> tree2 -> cf5d1d819f739c7e4a124c94996735b942bd2ea8
commit3 -> tree3 -> e3d828ca5b56c97911317033bf8243af31c15fc2

# 参考资料
[10.2 Git 内部原理 - Git 对象](https://git-scm.com/book/zh/v2/Git-%E5%86%85%E9%83%A8%E5%8E%9F%E7%90%86-Git-%E5%AF%B9%E8%B1%A1)

