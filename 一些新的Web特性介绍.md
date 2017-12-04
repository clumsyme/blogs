本文主要介绍一些新的Web特性。所谓新，要么是以前很难实现或者实现不了，现在可以了；或者是以前的方法太过于复杂，或者影响性能。

## Intersection Observer API

一个常见的需求是滚动到底部自动加载，目前一个常见的解决方案是对文档创建`scroll`事件监听，并计算`scrollTop`等值来判断页面滚动到了底部，从而触发事件。这样实现的缺点是：`scroll`事件触发非常频繁，在主线程上运行很影响性能。

`Intersection Observer`通过注册一个回调，每当被监控的元素进入或者退出另一个元素、或者只是两元素相交部分大小变化时，回调就会执行。如此就避免了在主线程内监听元素交集变化。

#### Intersection Observer Demo（尝试滚动Demo到底部查看效果）

`Chrome >= 55, FireFox >= 55`

<p data-height="400" data-theme-id="dark" data-slug-hash="EwXgjq" data-default-tab="result" data-user="liyan" data-embed-version="2" data-pen-title="IntersectionObserver scroll" class="codepen">See the Pen <a href="https://codepen.io/liyan/pen/EwXgjq/">IntersectionObserver scroll</a> by LiYan (<a href="https://codepen.io/liyan">@liyan</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

## backdrop-filter

如何在网页上实现毛玻璃效果？

[Filter Effects Module Level 1](https://drafts.fxtf.org/filter-effects/#FilterProperty)定义了`CSS filter`属性，可以对文档元素施加各种视觉效果。`blur`函数可以对元素实现高斯模糊效果，但是该函数只对它作用于的元素施加效果，该元素后边的元素是没有模糊效果的。可以看下边的演示：

<p data-height="400" data-theme-id="dark" data-slug-hash="jGwMRo" data-default-tab="result" data-user="liyan" data-embed-version="2" data-pen-title="filter" class="codepen">See the Pen <a href="https://codepen.io/liyan/pen/jGwMRo/">filter</a> by LiYan (<a href="https://codepen.io/liyan">@liyan</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

`backdrop-filter`则可以实现将指定元素后边的元素都应用模糊效果，实现毛玻璃效果。只需将上述CSS代码的`filter`改为`backdrop-filter`。效果如下（目前在Chrome中需在`chrome://flags/`找到`Experimental Web Platform Features`并启用来查看效果）：

<p data-height="400" data-theme-id="dark" data-slug-hash="gGRLPB" data-default-tab="result" data-user="liyan" data-embed-version="2" data-pen-title="backdrop-filter" class="codepen">See the Pen <a href="https://codepen.io/liyan/pen/gGRLPB/">backdrop-filter</a> by LiYan (<a href="https://codepen.io/liyan">@liyan</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>

## dynanmic import

如何按需加载js文件？

一个方法是动态创建`<script>`标签，或者使用类似[requirejs](http://requirejs.org/)这样的库。但是ecmascript标准是有`import`关键字的，正常是不应该绕这么多路的。好在现在一些浏览器(如Chrome)已经实现了[`dynamic import`](https://github.com/tc39/proposal-dynamic-import)（目前处于ecmascript proposal stage 3，也就是即将成为正式标准）。

### 示例

    :::html
    <button id="dynamicImport">点击我动态加载</button>

    <script>
        document.getElementById('dynamicImport').addEventListener('click', async (event) => {
            event.preventDefault()
            const module = await import(`https://raw.githubusercontent.com/clumsyme/learn/d6f1db41122ca3b36068ba08ab8ed0e8686355b3/js/dynamicImport.js`);
            module.introduce()
        })
    </script>

点击下面按钮：
<button id="dynamicImport">点击我动态加载</button>

<script>
    document.getElementById('dynamicImport').addEventListener('click', async (event) => {
        event.preventDefault()
        const module = await import(`https://raw.githubusercontent.com/clumsyme/learn/d6f1db41122ca3b36068ba08ab8ed0e8686355b3/js/dynamicImport.js`);
        module.introduce()
    })
</script>