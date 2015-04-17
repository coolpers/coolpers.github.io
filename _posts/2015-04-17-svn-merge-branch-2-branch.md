---
layout: post
title:  "SVN分支间Merge注意事项"
date:   2015-04-17
categories: svn
tags: svn branch merge
---

# 遇到的问题 #

根据开发需求，需要把一个开发分支所有的代码merge到另外一个分支。因为连个代码都还没有开发完成，都还没有同步到Trunk。所以临时的出现这个branches间的merge。


但是使用工具merge完后，发现合并的不完整，有部分代码并不是最新的。百思不得解。


于是分析问题，就有了这篇文章。


# Svn 分支关系 #

为了调查此问题，使用测试分支模拟了一下真是环境，得以复原案件现场。

Svn branch 间的关系如下：

![](/assets/posts/2014-08-25-view-hprof-bitmap/revision-graph.PNG)

- 16781 AppSearch_test 为主干代码
- 16808 在主干进行了代码修改，添加了一行代码 : modify by trunk
- 16809 基于修改后的主干代码创建 branch 1 分支
- 16811 在主干进行了代码修改，添加一行代码：modify by trunk2
- 16812 在branch 1 进行了修改，添加一行代码：modify by branch 1
- 16813 基于主干 16811 基线创建branch 2 分支
- 16814 在 branch 2 进行修改，添加一行代码：modify by branch 2


最后三个分支的代码如下：

Trunk ：

	……
    modify by trunk
    modify by trunk 2
	……


Branch 1：

	……
    modify by trunk
    modify by branch 1
	……


Branch 2：

	……
    modify by trunk
    modify by trunk 2
	modify by branch 2
	……


# 进行合并 #

工具：tortoisesvn  1.6，或者 eclipse 插件 svn 1.6 版本
合并过程使用 non-interactive 模式，遇到冲突继续，等合并完一并解决冲突。


使用 merge a range of revisions方式

![](/assets/posts/2014-08-25-view-hprof-bitmap/select-merge-type.png)


合并后工具的提示：

![](/assets/posts/2014-08-25-view-hprof-bitmap/merge-warn-1.6.png)


冲突目录：
![](/assets/posts/2014-08-25-view-hprof-bitmap/confict-1.6.png)

confict-1.6.PNG


合并后手工打开冲突文件：
这个看起来还是没问题的，代码全都在，如果采用手工解决是没问题的。。
![](/assets/posts/2014-08-25-view-hprof-bitmap/merge-result-1.6-conflict.png)


使用 tortoise merge 工具合并（问题所在）

![](/assets/posts/2014-08-25-view-hprof-bitmap/conflict-edit.1.6.png)


使用工具合并后，发现 branch 2 修改的代码丢了。

因为工具是根据 confict-1.6.PNG 图中的 16808-16811 的冲突进行的合并。这两个之间的修改为主干代码的修改，而没有 branch 2 的修改。所以使用工具后进行的merge，把 branch 2 的修改丢了。


# 问题分析 #


查看 merge 工具帮助文档能找到一些线索，其中关于遇到冲突是否立即解决还是以后解决：

> You can choose to do that for the current conflicted file, or for all files in the rest of the merge. However, if there are further changes in that file, it will not be possible to complete the merge.

另外一段

> If you do not want to use this interactive callback, there is a checkbox in the merge progress dialog Merge non-interactive. If this is set for a merge and the merge would result in a conflict, the file is marked as in conflict and the merge goes on. You will have to resolve the conflicts after the whole merge is finished. If it is not set, then before a file is marked as conflicted you get the chance to resolve the conflict during the merge. This has the advantage that if a file gets multiple merges (multiple revisions apply a change to that file), subsequent merges might succeed depending on which lines are affected. But of course you can't walk away to get a coffee while the merge is running ;)


也就是说，使用 non-interative (resolve conflicts later) 模式，可能导致无法完全合并。

但是如何解决呢。。。


# 解决办法 #

办法1：

根据提示，我们采用 interactive 模式，遇到一个 conflict 就 resolve 一个，这样发现没有问题。但是缺点就是合并过程比较痛苦，不能离开。


办法2：

根据冲突文件，手工解决冲突。这种办法也可以。但是那是相当痛苦。

那有没有别的办法呢？


# Merge工具版本太低，需要升级 #


我们把merge工具升级到 1.8，看下合并结果。

同样采用和 1.6 上述步骤一样的办法，采用 non-interactive 模式，先合并，随后一并解决冲突。

![](/assets/posts/2014-08-25-view-hprof-bitmap/resolve-dialog.png)

从合并的提示来看，跟1.6 有不同。多了一句话：

> Resolve all conflicts and return the merge to apply the remaining.


也就是新版本发现了多个冲突，当无法解决冲突时，主动终止，让我们先把当前冲突解决，然后来回再次merge剩余的修改。


好，我们按照提示，先解决冲突。


解决冲突，这个和 1.6 的一样，还是合并 16808-16811 之间的冲突。

此时合并完并没有branch 2的修改。


然后再次merge。此时 branch 2 的修改才被完全的merge进来。


使用新工具我们采用了多次 merge 的方式来达到目的。


# 结论 #

至此真相大白。

当我们合并多个 branch 的时候，如果两个branch的 base 不是主干上的同一个revision，此时需要注意。可能会遇到同一个文件多次冲突导致无法正确合并的问题。


解决有以下几个办法

1、	分段merge，先merge主干部分差异，然后再merge 分支。这其实是一个正确的思路。就跟我们平时 branch 只跟 trunk 合并。很少 branch间合并一样。

2、	遇到conflict立马解决，避免以后的冲突依赖于当前。

3、	把 merge工具升级到新版本，这样能避免使用不当导致的问题。




