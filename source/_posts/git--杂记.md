
title: git--杂记
date: 2016-12-05 17:05:43
tags: [youdaonote]
---

配置别名
---
编辑~/.gitconfig文件, 添加
```
[alias]
    last = log -1
    co = checkout
    ci = commit
    br = branch
    st = status
    lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit

```




case
---
有时本地修改了，然后直接checkout线上的某个branch:
```
 Your local changes to the following files would be overwritten by checkout:
    app/models/VO/Request/ServiceSchedulerConfigExt.php
    app/models/purchase/InvoiceApplyModel.php
```
两个方案:

- 丢掉本地修改： git reset --hard, 之后重新执行
- 保留本地修改：git stash ： Stash the changes in a dirty working directory away。之后git pull,之后 git stash pop
                             
**说明**：git stash是当我们对项目进行了修改，但是想在不丢失当前修改的前提下，到commit的某个版本【HEAD】那儿去。这时候调用这个命令，其实相当于调用了git stash save，它会将当前所有的修改存放到一个stash list里去，可以理解为一个队列。当我们到了【HEAD】那里，然后执行git stash pop，就会从stash list里移除最前面的修改信息，并把这个修改应用到当前情境之中，这个修改必须是前面刚刚git stash save存储的。


常用操作
---

- 取消对文件的修改。还原到最近的版本，废弃本地做的修改。git checkout -- <file>
- 取消已经暂存的文件。即，撤销先前"git add"的操作git reset HEAD <file>...
- 修改最后一次提交。用于修改上一次的提交信息，或漏提交文件等情况。git commit --amend
- 回退所有内容到上一个版本git reset HEAD^
- 回退a.py这个文件的版本到上一个版本  git reset HEAD^ a.py  
- 向前回退到第3个版本  git reset –soft HEAD~3  
- 将本地的状态回退到和远程的一样  git reset –hard origin/master  
- 回退到某个版本  git reset 057d  
- 回退到上一次提交的状态，按照某一次的commit完全反向的进行一次commit.(代码回滚到上个版本，并提交git)git revert HEAD
回复git reset --hard之前的数据
- 使用git reflog查看所有的提交记录，
    ```
    1d2b028 HEAD@{0}: reset: moving to 1d2b02818e8fa
    9510636 HEAD@{1}: commit: update pci driver and add network stuff
    1d2b028 HEAD@{2}: commit: update pci driver
    e4e042b HEAD@{3}: commit: add infrastructure of Linux pci driver
    eb7ac57 HEAD@{4}: commit: update i2c driver - add at24 device
    034034f HEAD@{5}: commit: update i2c linux driver - example of mini2440
    17e0bfb HEAD@{6}: commit: add some diagram for i2c driver
    1f382d7 HEAD@{7}: commit: update i2c driver - overview
    ```
    找到对应版本的commit后进行git reset 1d2b028 

- 删除远程分支上的提交
    ```
    git reset --hard HEAD~1 // 在本地回退一次git提交，会提示回退到那个commit上了
    git push --force //将本地回退强制推送到远程
    ```

