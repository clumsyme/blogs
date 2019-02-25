`cherry-pick` 是一个比较常用的 git 操作，可以将一个分支上的 commit “精选”到另一个分支上。然而在最近的开发过程中，却时不时的遇到 merge 冲突。在下文中，我将会详细的分析 `cherry-pick` 造成冲突的原因，以及 `cherry-pick` 可能造成的其他更严重问题。

我们以一个简单的例子来进行分析：

![the-cp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/cherry-merge/the_cp.png)

如上图：我们有一个 master 分支，以及一个 feature 分支。这个例子中我们只关注其中的一行代码，其初始值(commit A)为 `apple`。在此基础上我们进行了如下操作：

- 从 master 分支 checkout 出来了 feature 分支
- 在 feature 分支进行了 F1 提交，在 master 分支进行了 M1 提交，这些提交都与 `apple` 这行代码无关
- 在 feature 分支进行了 F2 提交，将 `apple` 修改为 `berry`
- 我们将 F2 提交 cherry-pick 到 master 分支（以虚线表示一次 cherry-pick）


现在 feature 和 master 分支上这一行代码都是 `berry`。

当最后我们将 feature 分支正式往 master 合并的时候，可能出现三种情况：

1. 这一行代码在两个分支上都没有再修改过，顺利合并
2. 这一行代码之后在两个分支上又有了修改，结果出现了冲突（⚠️解决冲突很麻烦）
3. 这一行代码之后在两个分支上又有了修改，没有冲突，顺利合并（❌但是可能导致更严重的错误）

### 1. 这一行代码在两个分支上都没有在修改过，顺利合并

这是最理想的情况，不做分析。

### 2. 这一行代码之后在两个分支上又有了修改，结果出现了冲突

![the-cp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/cherry-merge/conflict.png)

如上图所示，在 cherry-pick 之后，我们又进行了如下操作：

- 在 master 分支进行了 M3 提交，该提交并未修改 `berry` 这行代码
- 在 feature 分支进行了 F3 提交，将 `berry` 修改为了 `cherry`
- 尝试将 feature 分支往 master 合并：⚠️冲突发生

分析：合并时，M3 和 F3 的最近公共祖先（merge base）是 commit A，这行代码为 `apple`，然后对比发现，master 分支上这行代码由 `apple` 变成了 `berry`，feature 分支上这行代码由 `apple` 变成了 `cherry`，所以冲突出现了。

如果我们使用的是 merge 呢？那 M2 到 F2 将是一条实线，M3 和 F3的最近公共祖先（merge base）会是 F2，master 上：`berry -> berry`，feature 上 `berry -> cherry`，因为 master 分支相对 F2没有修改，所以就没有冲突。

这只是两个分支的简单情况，随着分支增多，冲突会越来越难以处理。

而且这是有冲突的情况，还有一种情况是没有冲突，却可能引发更严重的问题。

### 3. 这一行代码之后在两个分支上又有了修改，没有冲突，顺利合并

为了更直观，我们将这一行代码视为一个功能的开关配置：`apple` 代表这个功能是上线状态，`berry` 代表这个功能是下线状态。

![the-cp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/cherry-merge/error.png)

如上图，最开始（A），我们这一行代码是 `apple`，代表功能为上线状态。

然后我们发现了一些 bug，需要将该功能紧急下线，我们：

- 在 feature 分支上下线该功能（F2）：`apple -> berry`
- 然后将该操作 cherry-pick 到 master（M2），现在 master 上该功能也下线了
- 然后我们在 feature 分支上进行了 bug 修复，最终解决了 bug，我们在 feature 分支上将该功能上线（F3）：`berry -> apple`
- 然后我们决定将 bug 修复 merge 到 master
- merge 顺利完成，没有冲突。**但是：这行代码仍然是`berry`，下线状态**


merge 分析：M3(`berry`) 和 F3(`apple`) 的最近公共祖先是 A(`apple`)，因此 git 认为 feature 分支并未修改 `apple` 的值，结果合并后 master 分支上这行代码还是 `berry`，我们的功能在 master 上还是下线状态。

## 结论

在开发中，不要使用 cherry-pick 来进行不同分支之间代码的同步，这很可能会造成最终合并时出现冲突，而且可能产生比冲突更严重的问题：该有冲突却没有冲突。

### ps: 何时使用 cherry-pick

当我们需要在不同分支之间**移动**提交时，可以使用 cherry-pick。

本文主要参考了 [Raymond Chen](https://social.msdn.microsoft.com/profile/Raymond+Chen+-+MSFT) 的系列文章[Stop cherry-picking, start merging](https://blogs.msdn.microsoft.com/oldnewthing/20180323-01/?p=98325)。作者作为 Windows 开发团队的一员，因为项目需要有很多分支相互 merge，所以 cherry-pick 很容易造成问题。但是如果你的两个分支是两个单独的分支，**永远不会相互 merge**，那么就可以使用 cherry-pick 而不用担心上述问题。


参考：

- [Stop cherry-picking, start merging, Part 1: The merge conflict](https://blogs.msdn.microsoft.com/oldnewthing/20180312-00/?p=98215)
- [Stop cherry-picking, start merging, Part 2: The merge conflict that never happened (but should have)](https://blogs.msdn.microsoft.com/oldnewthing/20180313-00/?p=98225)
- [Stop merging if you need to cherry-pick: A response from the VSTS team.](https://blogs.msdn.microsoft.com/oldnewthing/20180709-00/?p=99195)