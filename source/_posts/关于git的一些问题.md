---
title: 关于git的一些问题
date: 2019-11-02 14:29:26
tags: ['git']
---

微小工具叫 git ~。~

<!--more-->



在本地有一个提交了很久的master分支，没有维护到远程服务器，突然想起github的私有仓库已经免费，于是准备提交到服务器上。

OK，在github上创建了一个repository，在网页上初始化了一个README文件，远程master分支有了第一次的initial commit。

在自己本地文件夹下添加远程库

```
git remote add origin git@github.com:<your account>/<your repository>.git
```

没有输出，成功添加远程库。

习惯性`git pull`

控制台回显如下：

```
warning: no common commits
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
From github.com:<your account>/<your repository>
 * [new branch]      master     -> origin/master
There is no tracking information for the current branch.
Please specify which branch you want to merge with.
See git-pull(1) for details.

    git pull <remote> <branch>

If you wish to set tracking information for this branch you can do so with:

    git branch --set-upstream-to=origin/<branch> master
```

OK，git好像不知道本地master与origin/master的关联关系

执行`git branch --set-upstream-to=origin/<branch填你自己远程的branch哦~> master`

没有输出，成功关联本地master与远程master

再执行`git pull`

控制台回显如下：

```
fatal: refusing to merge unrelated histories
```

嗷，fatal，拒绝merge不相关的历史

也是，远程master的提交记录只有initial commit，而我本地分支却有一长串自己的提交，git认为这是两个不同的项目，无法提交。

那就强制合并

执行`git pull origin master --allow-unrelated-histories`

将远程master pull到本地master，允许不相关历史的merge

没有输出，成功merge

此时输入`git log`，可以看到远程master的提交已经合并打本地分支

`git push`大功告成~

# 如何删除本地未追踪的文件

`git clean -fd`将未追踪的文件和目录都删除



>  大部分正义感都是虚伪的 聊胜于无 —————————————沃德彭尤·胡某

