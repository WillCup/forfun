
title: git-同时push到两个远程仓库
date: 2017-02-08 14:14:54
tags: [youdaonote]
---

```
root@will-vm:~/gitLearning/dobuleremote# git remote -v
origin	git@github.com:WillCup/dobuleremote.git (fetch)
origin	git@github.com:WillCup/dobuleremote.git (push)
```


```
git remote add ink git@10.1.5.83:/server/gitrepo/dpush.git
```

之后就可以看到
```
root@will-vm:~/gitLearning/dobuleremote# git remote -v
ink	git@10.1.5.83:/server/gitrepo/dpush.git (fetch)
ink	git@10.1.5.83:/server/gitrepo/dpush.git (push)
origin	git@github.com:WillCup/dobuleremote.git (fetch)
origin	git@github.com:WillCup/dobuleremote.git (push)
```

可以通过push ink推送到新的远程仓库。


但是执行git status的时候发现，他是以origin为准的
```
root@will-vm:~/gitLearning/dobuleremote# git st
位于分支 master
您的分支与上游分支 'origin/master' 一致。
未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）

	both1

提交为空，但是存在尚未跟踪的文件（使用 "git add" 建立跟踪）

```

新建一个both文件，然后不加参数提交：
```
root@will-vm:~/gitLearning/dobuleremote# ll
总用量 12
drwxr-xr-x  3 root root 4096 2月   8 14:17 ./
drwxr-xr-x 29 root root 4096 2月   8 14:21 ../
-rw-r--r--  1 root root    0 2月   8 14:17 both1
drwxr-xr-x  8 root root 4096 2月   8 14:20 .git/
-rw-r--r--  1 root root    0 2月   8 14:05 githubside
-rw-r--r--  1 root root    0 2月   8 14:06 githubside2

root@will-vm:~/gitLearning/dobuleremote# git push
warning: push.default 尚未设置，它的默认值在 Git 2.0 已从 'matching'
变更为 'simple'。若要不再显示本信息并保持传统习惯，进行如下设置：

  git config --global push.default matching

若要不再显示本信息并从现在开始采用新的使用习惯，设置：

  git config --global push.default simple

当 push.default 设置为 'matching' 后，git 将推送和远程同名的所有
本地分支。

从 Git 2.0 开始，Git 默认采用更为保守的 'simple' 模式，只推送当前
分支到远程关联的同名分支，即 'git push' 推送当前分支。

参见 'git help config' 并查找 'push.default' 以获取更多信息。
（'simple' 模式由 Git 1.7.11 版本引入。如果您有时要使用老版本的 Git，
为保持兼容，请用 'current' 代替 'simple'）

对象计数中: 2, 完成.
Delta compression using up to 2 threads.
压缩对象中: 100% (2/2), 完成.
写入对象中: 100% (2/2), 231 bytes | 0 bytes/s, 完成.
Total 2 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local objects.
To git@github.com:WillCup/dobuleremote.git
   2ba4199..8697e51  master -> master

```

可以看到是推送到了github端。到dpush那边也查看了一下，最新版本并没有出现这个文件
```
root@will-vm:~/gitLearning/dpush# ll
总用量 12
drwxr-xr-x  3 root root 4096 2月   8 14:21 ./
drwxr-xr-x 29 root root 4096 2月   8 14:21 ../
drwxr-xr-x  8 root root 4096 2月   8 14:21 .git/
-rw-r--r--  1 root root    0 2月   8 14:21 githubside
-rw-r--r--  1 root root    0 2月   8 14:21 githubside2

```

其实也可以只为某个remote添加push，而并不添加pull：
```
root@will-vm:~/gitLearning/dobuleremote# git remote set-url --add --push ink git@github.com:WillCup/dobuleremote.git
root@will-vm:~/gitLearning/dobuleremote# git remote -v
ink	git@10.1.5.83:/server/gitrepo/dpush.git (fetch)
ink	git@github.com:WillCup/dobuleremote.git (push)
origin	git@github.com:WillCup/dobuleremote.git (fetch)
origin	git@github.com:WillCup/dobuleremote.git (push)
root@will-vm:~/gitLearning/dobuleremote# git --version
git version 2.7.4
root@will-vm:~/gitLearning/dobuleremote# git remote set-url --add --push ink git@10.1.5.83:/server/gitrepo/dpush.git
root@will-vm:~/gitLearning/dobuleremote# git remote -v
ink	git@10.1.5.83:/server/gitrepo/dpush.git (fetch)
ink	git@github.com:WillCup/dobuleremote.git (push)
ink	git@10.1.5.83:/server/gitrepo/dpush.git (push)
origin	git@github.com:WillCup/dobuleremote.git (fetch)
origin	git@github.com:WillCup/dobuleremote.git (push)
```


再添加一个文件both2，然后再push测试一下
```
root@will-vm:~/gitLearning/dobuleremote# touch both2
root@will-vm:~/gitLearning/dobuleremote# git st
位于分支 master
您的分支与上游分支 'origin/master' 一致。
未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）

	both2

提交为空，但是存在尚未跟踪的文件（使用 "git add" 建立跟踪）
root@will-vm:~/gitLearning/dobuleremote# git add .
root@will-vm:~/gitLearning/dobuleremote# git ci -am "both test 2"
[master 40cdebb] both test 2
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 both2
root@will-vm:~/gitLearning/dobuleremote# git push ink master
对象计数中: 2, 完成.
Delta compression using up to 2 threads.
压缩对象中: 100% (2/2), 完成.
写入对象中: 100% (2/2), 223 bytes | 0 bytes/s, 完成.
Total 2 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local objects.
To git@github.com:WillCup/dobuleremote.git
   8697e51..40cdebb  master -> master
对象计数中: 4, 完成.
Delta compression using up to 2 threads.
压缩对象中: 100% (4/4), 完成.
写入对象中: 100% (4/4), 414 bytes | 0 bytes/s, 完成.
Total 4 (delta 1), reused 0 (delta 0)
To git@10.1.5.83:/server/gitrepo/dpush.git
   2ba4199..40cdebb  master -> master

```

从上面的输出就可以看到，是push到了两个remote端，刚才落下的提交也一并被push过去了。
