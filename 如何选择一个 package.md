软件开发离不开开源项目，自己单干无异闭门造车，所以我们需要别人的库/工具帮助我们完成工作。我们需要的，是尽可能轻松地集成到我们的项目内，并且能很好地解决我们的需求而不引起其他问题。我们以以下三个方面来考虑是否选择一个库：

- 功能性：可以解决我们的问题
- 可靠性：不会因为引入它而引起其他问题
- 复杂性：集成尽可能简单

功能性是我们的根本目的，但是不应该仅仅止于此。**不能简单地只是因为一个库能够解决我们的问题我们就选它**。

##  1. 我们可能不需要一个 package

![module_count](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/count.png)

这张图反映的是 npm 包含的库的数量对比 Java、Python 库的数量，可以看到 npm 库的数量非常之多。但是有多少是真是有用的呢。

这个库：[is-odd](https://www.npmjs.com/package/is-odd)，在目前每周有 849,241 次下载，在 GitHub 上有 622,528 个其他仓库依赖于它。但是它的作用，就是和它的名字一样，用来检查一个数字是不是奇数，它的源代码不到 20 行。

这是一个例子，告诉我们在实现很简单的情况下，我们完全可以自己实现，而不必去借助外部的库来帮我们实现。

GitHub 上有一些项目，专门用来总结“你可能不需要 XXX”，比如：

- [You-Dont-Need-jQuery](https://github.com/nefe/You-Dont-Need-jQuery)
- [You-Dont-Need-Momentjs](https://github.com/you-dont-need/You-Dont-Need-Momentjs)
- [You-Dont-Need-Lodash-Underscore](https://github.com/you-dont-need/You-Dont-Need-Lodash-Underscore)

我们再以 moment 为例，moment 是一个非常强大，同时也非常 API 友好的项目，它既能够方便地解决我们的问题。但是如果我们统计一下，就会发现，我们最常用到的 moment 功能，可能不超过 5 个。但是 moment 打包体积有 329KB，即使 GZip 压缩之后，仍然有 69.6KB。而那几个最常用的功能如果我们自己来实现的话，可能体积会缩小几十到上百倍，这时候更好的方案可能就是不引入 moment（当然这只是理想情况，比如我们用到了 antd，antd 的一些组件会依赖于 moment，那我们还是需要引入 moment）。

## 2. 当我们确实需要一个库时，我们应该考虑的因素

![module_react](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/react.png)


### 可靠性

一个常见且简单的方法是看它的 GitHub Star 数，但是我们会发现这个评价标准是非常片面的。我们应该从以下这些方面来评估其可靠性：

#### 更新日期（越近越好）

上图中，react 的 上次提交时间是 `Latest commit 45acbdc  7 hours ago`，说明它的维护很及时。而如果一个项目的最近更新日期是 `3 年前`，那选择它之前就要好好考虑一下了。

#### Issues / Pull Requests（越少，Open/Close 比越小越好）

Issue 数值越低，说明存在问题的可能越少。另外由于不同项目流行度的差异，看 Issues 的 Open/Close 比值更能说明问题。比值越小说明作者维护项目的意愿越大，出了问题更可能被解决。

Pull Requests 同理。

#### Contributors 贡献值数量（越多越好）

Contributors 越多，说明社区力量越强大，项目可维护性更好。

#### 作者（优先选择知名作者）

相比于 `someone/project`，我可能更倾向于选择 `Microsoft/project`。当然这个标准也不是绝对的。

#### Star / Watch / npm downloads（越多越好）

没错，这些指标还是有一定参考意义的。但是我们应该知道，一个项目比另一个更流行，可能仅仅是因为它是一个知名作者的项目，或者被媒体/知名作者推荐过。所以我们不能说一个 500 Star 的比 5000 Star 的差。

但是 5000 Star 的项目通常来说会比 50 Star 的项目要好。

#### 体积（越小越好）

对于 Web 应用来说，代码体积至关重要，它影响页面渲染时间，进而影响用户体验。所以要尽量选择较小的库。

### 复杂性

复杂性低的表现：

- 有全面的文档
- 有 `Get Started` 指引，帮助我们快速上手
- 即插即用，无需额外操作

复杂性高的表现：

- 没有文档或文档混乱
- 不知道怎样快速上手
- 接入十分复杂，需要改我们的很多代码，甚至其他第三方代码

一个例子是关于我们如何舍弃 [react-hot-loader](https://github.com/gaearon/react-hot-loader/releases) 的。该项目是著名的 Dan Abramov 开发的，现在由社区维护版本升级。直到 React 16.8 发布之后，它与 React Hooks 不兼容，以至于要想使其正常工作需要在开发时舍弃 React 官方的 `React-Dom` 而采用他们维护的 `React-Dom` 版本。即使如此，经过复杂的配置之后，我还是没能让它顺利正确地工作起来，所以最后只能舍弃了它。
