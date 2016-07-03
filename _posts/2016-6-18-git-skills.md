---
layout: post
title: 一些git实用技巧
---

这篇文章将通过场景分析的方式讲解一些git的实用技巧。这些技巧的使用频率或许不是很高，但真正需要用到的时候，你可能会感慨相见恨晚。

<!-- more -->

### 场景一、罪魁祸首

>程序员小A发现源文件B.java中的某一行引发了一个bug。现在小A要找出是谁在什么时间提交了这次修改。那么小A应该怎么做呢？

最直接的想法应该是察看B.java这个文件的提交历史，在一个个commit中找出导致bug的修改。高级一点的话可以使用`git bisect`进行二分查找。其实git提供了一个简单直接的命令`git blame`可以帮助我们快速地找到那个“罪魁祸首”。`git blame`命令会输出指定文件中每一行的最后一次修改的作者、时间以及commit hash. 例如执行`git blame B.java`的输出如下：

	^87fe8e2 (little A 2016-06-18 14:21:23 +0800 1) public class B {
	^87fe8e2 (little A 2016-06-18 14:21:23 +0800 2) 
	^87fe8e2 (little A 2016-06-18 14:21:23 +0800 3)         public static void main(String[] args) {
	51497e8a (someone  2016-06-18 14:22:21 +0800 4)       System.out.println("It is me! Haha!");
	^87fe8e2 (little A 2016-06-18 14:21:23 +0800 5)   }
	^87fe8e2 (little A 2016-06-18 14:21:23 +0800 6) }

可以看到someone就是那个“罪魁祸首”！（好吧，其实我们还是不知道他是谁）

### 场景二、亡羊补牢1.0

>小A刚刚提交了一个commit，但发现提交信息写错了。小A该如何修改这次的提交信息呢？

补救的方法有不少，这里列举两个：

1. 执行命令`git commit --amend`, 在vim编辑界面修改提交信息后保存即可。这算是最简单的方法了。
2. 依次执行

		git reset HEAD~1
		git commit -am "new message"
	
	这个方法首先将当前的working tree还原到最后一次commit之前的状态，然后再进行一次新的commit.

### 场景三、亡羊补牢2.0

>现在难度升级，如果小A想修改中间某个commit的提交信息，该怎么办呢？

这时可以使用强大的`git rebase -i`命令。输入`git rebase -i`加一个commit hash之后回车，进入一个vim编辑界面。界面上按序列出了指定commit之后的所有commit，类似下面这样：

	pick 383fd55 llal pick 1ddcdab update new.txt
	pick 1519f1f update new.txt 2
	pick fdc6370 delete new.txt
	pick 0a8bcf8 add back new.txt
	# Rebase 2aa6567..0a8bcf8 onto 2aa6567 (5 command(s))
	# # Commands:
	# p, pick = use commit
	# r, reword = use commit, but edit the commit message
	# e, edit = use commit, but stop for amending
	# s, squash = use commit, but meld into previous commit
	# f, fixup = like "squash", but discard this commit's log message
	# x, exec = run command (the rest of the line) using shell
	# d, drop = remove commit
	
在每一行的开头可以指定一个你想执行的命令。默认是`pick`, 也就是不做任何修改。现在我们想修改中间的某个commit信息，只需把该commit所在行的`pick`替换成`reword`或者简写`r`，然后保存。接下来会出现新的编辑界面，然后我们就可以修改提交信息了。

可以看到除了`reword`还有其他一些命令，后面会用到其中的某几个。

### 场景四、追悔莫及

> 小A发现当前分支test上的最后N个commit有问题，想丢弃它们，于是执行了`git reset --hard HEAD~N`. 过了一段时间，小A突然想起之前丢弃的N个commit里面还有一些重要的修改！那么小A有什么办法找回丢弃的那N个commit呢？

这时可以求助于git提供的`ORIG_HEAD`这个变量。在进行`pull`、`merge`或者`reset`等操作后，`HEAD`一般都会移动好几个commit。而`ORIG_HEAD`就是指向这些操作之前`HEAD`所指向的那个commit。可以认为`ORIG_HEAD`是git提供的一剂后悔药。小A要找回之前丢弃的N个commit，可以这么做：

	git checkout -b temp ORIG_HEAD
	git checkout test
	git merge temp

首先创建一个分支temp指向`ORIG_HEAD`指向的commit。然后切回test分支，merge一下temp，就可以让那N个commit重新回到test分支。

但是，如果小A在丢弃了那几个commit之后又做了`pull`、`merge`或者`reset`等操作，改变了`ORIG_HEAD`的值，那么上面这个方法就无效了。不过办法总比困难多，这时使用`git reflog`可以解决这个问题。git会记录`HEAD`的改变历史，而`git reflog`就是把这个历史记录输出。这个命令的输出如下：

	87fe8e2 HEAD@{0}: reset: moving to HEAD~1
	d42533e HEAD@{1}: checkout: moving from master to test
	d42533e HEAD@{2}: commit: another

小A只需找到执行reset命令的那一行，那么它下面一行最前面的commit hash就是“解药”了。找到了这个commit，之后再仿照上面的步骤操作就可以了。
 
### 场景五、挑三拣四1.0

> 小A对一个文件B做了多处修改。在提交的时候发现有一些修改是需要排除在这次提交之外的。
那么怎么做可以对同一个文件的多处修改进行选择性提交?

这时可以使用`git add -p`命令。输入此命令（后面可以指定文件名）后，git会对每一个修改区块（一般来讲一段连续的修改算是一个区块）询问需要执行的指令：

	Stage this hunk [y,n,q,a,d,/,j,J,g,e,?]?

其中各指令的含义如下：

	y - 将此区块加入index
	n - 不加入index，并跳到下一个区块
	q - 不加入index，同时结束操作
	a - 将此区块以及当前文件剩余区块都加入index，跳到下一个文件
	d - 与a相反
	g - 选择具体的区块进行操作
	/ - 搜索能够匹配给定正则表达式的区块
	j - 将此区块标记成“暂定”，跳到下一个“暂定”的区块
	J - 将此区块标记成“暂定”，跳到下一个区块
	k - 将此区块标记成“暂定”，跳到前一个“暂定”的区块
	K - 将此区块标记成“暂定”，跳到前一个区块
	s - 将当前区块划分成更小的区块
	e - 进入编辑界面进行更细的选择
	? - 打印帮助

如果选择执行指令`e`则会进入一个新的vim编辑页面，如下所示：

	# Manual hunk edit mode -- see bottom for a quick guide
	@@ -1,3 +1,3 @@
	 origin
	-1
	-2
	+abc
	+xyz
	# ---
	# To remove '-' lines, make them ' ' lines (context).
	# To remove '+' lines, delete them.
	# Lines starting with # will be removed.
	#
	# If the patch applies cleanly, the edited hunk will immediately be
	# marked for staging. If it does not apply cleanly, you will be given
	# an opportunity to edit again. If all lines of the hunk are removed,
	# then the edit is aborted and the hunk is left unchanged.

在这个界面可以进行更细致的操作：将某一行开头的`-`号换成空格，该行的删除修改则不会加入index，但这个删除的修改仍然保留；将以`＋`号开头的某一行删除，该行则不会加入index，但添加的这一行仍然保留在文件中。

### 场景六、挑三拣四2.0

> 如果上面那个B文件是一个Untracked file，那又该如何进行选择性提交呢？

可以先执行`git add -N B`命令。现在B里面的内容都变成了unstaged的修改。之后再用上面的方法进行选择性提交。

### 场景七、万源归一

> 小A开发出了一款可以自动编程并且提交代码的AI工具。不过有一天小A发现该工具的一个bug引发了一个问题：
它每修改一处就会进行一次commit，这样导致一次功能修改就提交了N多个commit。有什么办法可以合并这些commit变成一个commit？

可以再一次使用`git rebase -i`. 回顾场景三可以看到vim编辑界面的提示中有两条指令：

	# s, squash = use commit, but meld into previous commit
	# f, fixup = like "squash", but discard this commit's log message

这两条指令都可以把当前的commit归并到前一个commit中，区别就在于是否保留当前commit的提交信息。


