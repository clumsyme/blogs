最近写了一个 React 水印组件，为了支持水印防隐藏，采用了[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)，主要思路如下：

-   一个 MutationObserver 用于监测水印父节点(`parentNode`)的`childList`，在水印被手动删除之后，重新插入新的水印
-   一个 MutationObserver 用于监测水印自身属性(`attributes`)，在水印属性(`display`、`opacity`、`visibility`等)变化后重置属性值，防止通过修改 `style` 隐藏水印

我们只讨论如何监测自身属性来防止属性修改，主要有 2 个思路：

## 1. 设定一个初始值，任何属性变化时都将 `style` 属性重置为初始值

```javascript
const callback = (mutationsList, observer) => {
    for (var mutation of mutationsList) {
        observer.disconnect()
        mutation.target.setAttribute('style', initStyle)
        observer.observe(node, options)
    }
}
```

### 演示 1

<p data-height="265" data-theme-id="0" data-slug-hash="LMWGKJ" data-default-tab="js,result" data-user="liyan" data-pen-title="React WaterMark Component" class="codepen">See the Pen <a href="https://codepen.io/liyan/pen/LMWGKJ/">React WaterMark Component</a> by LiYan (<a href="https://codepen.io/liyan">@liyan</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

## 2. 监测 `oldValue`，任何属性修改都将属性值重置为 `oldValue`

```javascript
const callback = (mutationsList, observer) => {
    for (var mutation of mutationsList) {
        observer.disconnect()
        let { oldValue, attributeName } = mutation
        mutation.target.setAttribute(attributeName, oldValue)
        observer.observe(node, options)
    }
}
```

### 演示 2

<p data-height="265" data-theme-id="0" data-slug-hash="vvmdVo" data-default-tab="js,result" data-user="liyan" data-pen-title="WaterMark use oldValue" class="codepen">See the Pen <a href="https://codepen.io/liyan/pen/vvmdVo/">WaterMark use oldValue</a> by LiYan (<a href="https://codepen.io/liyan">@liyan</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

方法 2 与方法 1 相比，可以重置任何属性的改变，而方法 1 只能重置我们指定的几个属性。看起来方法 2 是更好的选项，事实如此吗？

注意演示 2 的 `hide` 与 `show` 按钮，点击发现，可以改变水印的  `style` 属性实现水印隐藏，其中 `hide` 按钮的时间处理函数为：

```javascript
onClick = () => {
    let watermark = document.getElementById('watermark')
    watermark.style.display = 'none'
    watermark.style.display = 'block'
}
```

为什么这样会实现水印的隐藏呢？其实在于 JavaScript 的事件运行机制（任务队列）。

关于任务队列，此处不做详述，可以参看这篇文章[Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)。主要就是：

- JavsScript 基于事件循环，每个循环会先处理当前循环的宏任务，然后会处理当前循环的微任务
- 微任务处理完后会进入下一个事件循环
- 如此循环往复

比如 setTimeout 就会产生一个宏任务，在当前任务运行完成之后再运行，而一个 fullfilled 的 Promise 、我们讨论的 MutationObserver 则是微任务的代表，基于此，我们来解释为什么上述代码会实现水印的隐藏。

### 当我们点击 `hide` 按钮时，就会产生一个任务，运行 `onClick` 回调函数

### 运行第一次属性修改

```javascript
watermark.style.display = 'none'
```
`style.display` 由 `block` 改变为`none`，这里会在微任务队列内推入一个微任务（MutationObserver callback， 其中 oldValue 为 `block`）。

### click 任务继续运行

```javascript
watermark.style.display = 'block'
```

`style.display` 由 `none` 改变为`block`，这里会在微任务队列内推入 另一个微任务（MutationObserver callback， 其中 oldValue 为 `none`）。

### click 任务运行完成，开始运行微任务

- 微任务 1 开始运行，`style` 属性恢复到 `oldValue`，`display` 重置为 `block`；
- 微任务 2 开始运行，`style` 属性恢复到 `oldValue`，`display` 重置为 `none`；

由此就实现了水印的隐藏。

具体过程如下：

*`style` 初始值 `display: block`*

click:
- 任务开始
    - watermark.style.display = 'none'
        - `style` 由 `display: block` -> `display: none`，MutationObserver 入队 callback 1，其中 oldValue 为 `display: block`
    - watermark.style.display = 'block'
        - `style` 由 `display: none` -> `display: block`，MutationObserver  再次入队 callback 2，其中 oldValue 为 `display: none`
- 任务运行结束,开始微任务队列
    - callback 1
        - 恢复 oldValue：`display: block`
    - callback 2
        - 恢复 oldValue：`display: none`

所以我们的方法 1 虽然只能锁定几个指定的属性，但是在任何情况下属性都会重置为初始值，因此可以实现对属性的真正锁定。

## 参考

- [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)
- [Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)