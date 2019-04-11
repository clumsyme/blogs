如果我们在一个组件内使用多个``useState` Hooks, React 如何知道哪个 Hooks 对应哪个 state？

为什么[不要在 `if` 内使用 Hooks](https://reactjs.org/docs/hooks-rules.html#only-call-hooks-at-the-top-level)？

我们以 `React-Dom/server` 内的 React Hooks 实现来进行探究<sup>[[1]](#ps)</sup>：

React Hooks的结构是一个链表型的数据结构：
每一个 Hooks 对象结构为：

```js
{
    memoizedState: null,
    queue: null,
    next: null,
}
```

有一个链表的头部，在 React 内部称为 `firstWorkInProgressHook`，保存着组件内所有的 Hooks。

还有一个链表，`workInProgressHook`，对应的是上述链表的一个节点，在组件内部就是调用`useState()` 的时候的当前的 Hooks。每一次我们调用`useState()`，React 内部会调用一个方法来生成一个 `workInProgressHook`，其[源代码为](https://github.com/facebook/react/blob/c05b4b81f91c0b43a02e101d6a37b3de768f017b/packages/react-dom/src/server/ReactPartialRendererHooks.js#L132)：

```js
function createWorkInProgressHook(): Hook {
  if (workInProgressHook === null) {
    // This is the first hook in the list
    if (firstWorkInProgressHook === null) {
      isReRender = false;
      firstWorkInProgressHook = workInProgressHook = createHook();
    } else {
      // There's already a work-in-progress. Reuse it.
      isReRender = true;
      workInProgressHook = firstWorkInProgressHook;
    }
  } else {
    if (workInProgressHook.next === null) {
      isReRender = false;
      // Append to the end of the list
      workInProgressHook = workInProgressHook.next = createHook();
    } else {
      // There's already a work-in-progress. Reuse it.
      isReRender = true;
      workInProgressHook = workInProgressHook.next;
    }
  }
  return workInProgressHook;
}
```

我们以组件的两次 render 来介绍该链表是如何初始化以及如何应对更新的：

- 初始条件下（组件还未渲染），`firstWorkInProgressHook` 和 `workInProgressHook` 都是 `null` 0️⃣
- 初次渲染：
    - 第一个 Hooks：`firstWorkInProgressHook = workInProgressHook = createHook()` 1️⃣
    - 第二个 Hooks：`workInProgressHook = workInProgressHook.next = createHook()` 2️⃣
    - ...
    - 第 n 个 Hooks: `workInProgressHook = workInProgressHook.next = createHook()` 3️⃣
    - 渲染结束，React 调用 `finishHooks`，重置 `workInProgressHook`为 `null` 4️⃣

```js
// 0️：
// firstWorkInProgressHook = workInProgressHook = null
firstWorkInProgressHook
                        ↘
                          null
                        ↗
     workInProgressHook

// 1:
// workInProgressHook = workInProgressHook.next = createHook()
          firstWorkInProgressHook
                                 ↘
                          null    Hook1
                                 ↗
              workInProgressHook

// 2:
// workInProgressHook = workInProgressHook.next = createHook()
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2
                                           ↗
                         workInProgressHook
// 3:
// workInProgressHook = workInProgressHook.next = createHook()
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                                                          ↗
                                        workInProgressHook
// 4:
// workInProgressHook = null
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                          ↗
        workInProgressHook
```

下一次 render：

- 开始前，链表状态和上次 render 结束一致 0️⃣
- 第一个 Hooks：`workInProgressHook = firstWorkInProgressHook`1️⃣
- 第二个 Hooks: `workInProgressHook = workInProgressHook.next`2️⃣
- ...
- 第 n 个 Hooks: `workInProgressHook = workInProgressHook.next`3️⃣
- 渲染结束，React 调用 `finishHooks`，重置 `workInProgressHook`为 `null` 4️⃣

```js
// 0:
// firstWorkInProgressHook = Hook1
// workInProgressHook = null
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                          ↗
        workInProgressHook

// 1:
// workInProgressHook = firstWorkInProgressHook(Hook1)
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                                 ↗
               workInProgressHook

// 2:
// workInProgressHook = workInProgressHook.next(Hook2)
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                                           ↗
                         workInProgressHook
// 3:
// workInProgressHook = workInProgressHook.next(HookN)
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                                                          ↗
                                        workInProgressHook
// 4:
// workInProgressHook = null
          firstWorkInProgressHook
                                 ↘
                           null   Hook1 ⟶️ Hook2 ...⟶️... HookN
                          ↗
        workInProgressHook
```

所以：

- 初次渲染的时候，Hooks 链表为空，每次 `useState()` 的时候都会新建一个 Hooks 作为当前的 Hooks（workInProgressHook）
- 再次渲染的时候，按照调用顺序，依次取上次生成的 Hooks 链表各个节点（每个节点就是一个 Hooks）

这也是为什么我们需要保证在每次渲染的时候各个 Hooks 以相同的顺序被调用，也是为什么不要在 `if` 内使用 Hooks 的原因：这会导致 Hooks 调用顺序不同。

### ps

React Hooks 有多个实现。实际上它的代码并不是在 React 内。

如果我们查看 ReactHooks.js 导出的 `useState`，它的[代码为](https://github.com/facebook/react/blob/c05b4b81f91c0b43a02e101d6a37b3de768f017b/packages/react/src/ReactHooks.js#L72)：

```js
export function useState<S>(initialState: (() => S) | S) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

而 `resolveDispatcher` 的代码为：

```js
function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;
  return dispatcher;
}
```

我们再看 `ReactCurrentDispatcher`:

```js
const ReactCurrentDispatcher = {
  current: (null: null | Dispatcher),
}
```

ReactCurrentDispatcher 只是一个普通 JavaScript 对象，甚至它的 `current` 属性是 `null`。

这是一个典型的[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)，React 本身并没有实现各个 Hooks，而是交由不同的模块来进行不同的实现。就像 `setState` 一样，React 组件可以调用 `setState`，React-Native 组件也可以调用这个方法，React 会自己决定对 Dom 还是 Native 的更新。

同样的，Hooks 在开发环境、生产环境、server 环境、React-Native 内，甚至第一次渲染、更新渲染时都有不同的实现，React 所做的，就是在不同的环境下使用不同的 `ReactCurrentDispatcher`来调用正确的 Hooks 实现。

本文分析采用了 `React-Dom/server` 的实现，虽然它和我们生产中通常调用的版本不同，但是其内部原理是统一的。