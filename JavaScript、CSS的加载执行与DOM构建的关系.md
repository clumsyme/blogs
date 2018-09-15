上一篇文章中，我们了解到了，页面首次渲染需要DOM和CSSOM都构建完成才能实现，也就是CSS会阻塞页面的首次渲染。那么外部JS、CSS文件的加载和DOM构建之间是如何相互影响的呢，我们将对此一探究竟。

## 纯HTML文本

![纯html](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/raw-html.png)

可以看到，纯HTML文本会在（6ms）接收到返回后8ms（也就是14ms时）完成`DOMContentLoad`事件。

## 加载JS

```html
<script src="delay-1.js"></script>
```

![js-delay.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/js-delay.png)

JavaScript文件会延迟100ms后返回，在（6ms）HTML文件返回后，一直要等到JavaScript文件完全加载完成（124ms），`DOMContentLoad`事件才会触发，**JavaScript会阻塞DOM的构建**。

## 加载CSS

```html
<link rel="stylesheet" href="delay-1.css" />
```

![css-delay.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/css-delay.png)

CSS文件会延迟100ms返回，不过我们可以看到，`DOMContentLoad`事件在15ms时就触发了，虽然`onload`事件要等到123ms才触发，也就是**CSS不会阻塞DOM的构建**。

## CSS 比 JavaScript 先加载

```html
<link rel="stylesheet" href="delay-1.css" />
<script src="delay-2.js"></script>
```

![css-faster.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/css-faster.png)

正如我们所见，由于**JavaScript会阻塞DOM的构建**，`DOMContentLoad`事件在要等到JavaScript加载完成才能触发（226ms）。而同时由于**CSS不会阻塞DOM的构建**，那么我们是否可以推论：

> DOM构建时间取决于JavaScript的加载时间？

## JavaScript 比 CSS 先加载

```html
<link rel="stylesheet" href="delay-2.css" />
<script src="delay-1.js"></script>
```

![js-faster.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/js-faster.png)

在这种情况下，JavaScript文件在125ms时即加载完成，而`DOMContentLoad`事件仍然要等到226ms、CSS文件加载之后才出发。因此**上述推论是错误的**，DOM构建时间显然受到了CSS文件的加载时间影响。这是因为JavaScript存在修改CSSOM的可能性，因此JavaScript的解析、执行必须等到CSSOM构建完成之后才能执行，也就是说**CSS文件的加载本身并未阻塞DOM的构建，但是CSS文件的加载阻塞了JavaScript的解析与执行，而JavaScript是会阻塞DOM构建的，因此CSS文件的加载就引起了DOM的阻塞**。

## 将JavaScript设置为defer

```html
<link rel="stylesheet" href="delay-2.css" />
<script src="delay-1.js" defer></script>
```

![defer.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/defer.png)

带有`defer`属性的JavaScript将会在DOM构建完成、`DOMContentLoad`事件触发前执行，其加载和执行都不会阻塞DOM构建。如上图所示，`DOMContentLoad`事件在JavaScript加载执行完成之后（126ms）触发。那么我们如何确定DOM已经在JavaScript执行之前构建完成呢。打开Chrome的Performance面板，进行分析，可以看到脚本执行之后到`DOMContentLoad`，并没有ParseHTML活动。

![defer-p.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/defer-p.png)

作为对比，下图是不带defer属性的脚本阻塞DOM构建的例子，可看到脚本执行后还伴随着ParseHTML活动，之后才是`DOMContentLoad`。

![normal-p.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/normal-p.png)

`defer`脚本的加载执行模式如下图所示（via MDN）。

![defer-modal.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/defer-modal.png)

## 将JavaScript设置为async

![async.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/async.png)

带有`async`属性的JavaScript的加载不会阻塞DOM的构建，由上图可见，`DOMContentLoad`时间在14ms即触发，而JavaScript文件要等到120ms才能加载完成。`async`脚本的执行会在`onload`事件触发前，值得注意的是，如果`async`脚本如果加载很快，加载完成时DOM正在构建，则其会马上执行，从而阻塞DOM构建。
`async`脚本的加载执行模式如下图所示（via MDN）：

![async-modal.png](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/page-load/async-modal.png)

## 结论

- JavaScript的加载、执行会阻塞DOM构建
- CSS自身不会阻塞DOM构建
- CSS会阻塞JS执行，从而阻塞DOM构建
- `defer`属性的JavaScript加载、执行不会阻塞DOM构建，但是会阻塞`DOMContentLoad`事件触发
- `async`属性的JavaScript加载不会阻塞DOM构建，也不会阻塞`DOMContentLoad`事件触发，但是其执行可能会阻塞DOM构建（如果其加载够快）。