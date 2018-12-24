---
title: 你真的了解 Git 吗？
date: 2018-04-08 16:31:19
tags:
---

看了几遍廖雪峰的git教程和阮一峰的git教程之后，觉得自己使用git已经是得心应手了，脑中也构建出了一副关于git操作的图像。


> 学习一个新东西的时候我总是喜欢把知识形象化出一个图谱在脑中，这样记忆的更加深刻。

但是随着使用的深入，我发现我脑中的图像与git的实际行为存在出入。

<!-- more -->

稍微进入一下正题。假如你的仓库关联了远程仓库，当我们执行`>git status`时，可能会有如下提示

```
# 您的分支与origin/master 一致
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean

```
当我的队友commit and push 之后我再次执行 `>git status`，我以为会有如下提示

```
# 你的分支落后于 origin/master一次提交.并且可以快速合并.执行git pull可以进行快速合并
On branch master
Your branch is behind 'origin/master' by 1 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)
nothing to commit, working tree clean

```

这是阮一峰大神一篇博客中看到的git操作示意图
> 网上流传的大部分资料都类似这样

![](http://asset.eienao.com/15231690297894.jpg)

通过这幅图，我理所当然的认为在队友push之后，我执行git status命令会提示`Your branch is behind`
我相信很多人都是这么认为的。

但实际产生的行为是前者，为什么会出现这样的情况？ 这就是我的问题。
再抽象一下这个问题，就是 **git status 是对哪两个仓库进行的比较?**

我们分析一下提示语句 `Your branch is behind 'origin/master' ` 。不难看出是 `master`分支和 `origin/master`进行的比较

按照上面的示意图，也就是 Remote(origin/master)和Repository进行了一个比较。但是这与实际情不符。

如果你有心尝试，你会发现`git status`是一个断网之后也可以执行的命令。所以不可能是Remote和Repository进行的比较。

于是经过简单的测试，我脑海中又重新构建出一份git示意图。


### 没有远程仓库时，我们常执行的命令有两条

`git add`和`git commit` 通过这两条命令绘制出如下图像
![](http://asset.eienao.com/15231725127941.jpg)


### 当关联远程仓库后，得到如下示意图

![](http://asset.eienao.com/15231740463034.jpg)

这张图是我根据git命令的一些行为得到，非权威。和上面阮一峰的示意图相比，本地增加了一个 origin/master。

再稍微介绍几个命令

### git pull

该命令没有在图上体现，其是一条复合命令

```
git pull = git fetch + git merge origin/master 
```
git fetch操作将remote同步到本地的origin/master

git merge origin/master 将 origin的master分支 合并到 仓库的master分支。


### git push
这条命令我没法确切的理解。我的猜测是， git push操作是将local的master push到了local的origin/master
git检测到origin变化之后，进行了一个origin到remote的同步操作

### git status

该命令中对比的两个仓库如图示。通过上图就能够解释，为什么 队友提交到remote之后,我执行git status没有落后提示。至此我上文提到的问题得到了解决。


## 结
我认为local和remote之间的行为，用推送和拉取来描述并不那么优雅。这两个词很容易会误解成，remote是一个控制中心。

实际上local和remote之间进行的只是简单的同步操作，无论是示意图还是我在介绍git fetch和git push都刻意体现出了这种同步。

在我看来，同步这个字眼更能体现git设计的初衷 —— 分布式版本控制。
