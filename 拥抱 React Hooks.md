## 问题

如何在组件之间复用代码一直是困扰 React 开发者的一个问题，虽然通过 React 的 [Higher-Order Components(HOC)](https://reactjs.org/docs/higher-order-components.html)、[Render Props](https://reactjs.org/docs/render-props.html) 等技术可以解决  这些部分，但是同时也带来了很多其他问题：

-   多层不必要的嵌套组件(HOC、Render Props)
-   可读性差(Render Props)
-   需要重构现有组件(HOC)
-   复用状态性逻辑困难(Composition、HOC)

我们通过下边这个 Demo 来对其进行比较：

## 示例

比如我们有如下的组件，可用于试验[考拉兹猜想](https://zh.wikipedia.org/wiki/%E8%80%83%E6%8B%89%E5%85%B9%E7%8C%9C%E6%83%B3)。这个组件主要有两个处理逻辑：

- 初始构建时设置一个随机的值为 n 的 state
- 每次点击按钮计算下一个 n 的值并更新 state

代码如下：

    :::javascript
    class CollatzRandom extends Component {
        constructor(props) {
            super(props)
            let n = parseInt(Math.random() * 100, 10)
            this.state = { n }
        }
        calcNext = () => {
            let { n } = this.state
            if (n % 2) {
                n = 3 * n + 1
            } else {
                n = n / 2
            }
            this.setState({ n })
        }
        render() {
            return (
                <>
                    <h1>{this.state.n}</h1>
                    <button onClick={this.calcNext}>next</button>
                </>
            )
        }
    }

组件结构如下图：

![Comp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/react-hooks/Comp.png)

### 如果要复用 calcNext 逻辑呢

如果现在有另一个组件需要复用 calcNext 逻辑呢？

### 1. 使用 HOC

    :::js
    function withCalcNext(Comp, initN) {
        return class extends Component {
            constructor(props) {
                super(props)
                this.state = { n: initN }
            }
            calcNext = () => {
                let { n } = this.state
                if (n % 2) {
                    n = 3 * n + 1
                } else {
                    n = n / 2
                }
                this.setState({ n })
            }
            render() {
                return <Comp {...this.props} n={this.state.n} calcNext={this.calcNext} />
            }
        }
    }

通过高阶组件，我们的组件  就可以共用 n、calcNext 等逻辑，只需要在组件内 `n = this.props.n` 以及 `calcNext = this.props.calcNext`。

但是同时要对我们的 Collatz 组件进行重构：

    :::javascript
    export const CollatzHOC = withCalcNext(
        class Collatz extends Component {
            render() {
                return (
                    <>
                        <h1>{this.props.n}</h1>
                        <button onClick={this.props.calcNext}>next</button>
                    </>
                )
            }
        },
        parseInt(Math.random() * 100, 10),
    )

以及增加了新的一层 Wrapper 包装层。

![HOCComp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/react-hooks/HOCComp.png)

### 使用 Render Props

    :::javascript
    class CollatzWrapper extends Component {
        constructor(props) {
            super(props)
            this.state = { n: this.props.initN }
        }
        calcNext = () => {
            let { n } = this.state
            if (n % 2) {
                n = 3 * n + 1
            } else {
                n = n / 2
            }
            this.setState({ n })
        }
        render() {
            return this.props.children(this.state.n, this.calcNext)
        }
    }

使用 Render Props，我们的组件将使用 CollatzWrapper 的内部状态进行渲染，数据 `n` 和处理函数 `calcNext` 都由 CollatzWrapper 进行处理，但是我们的代码可读性就变得很差，特别是如果有很多 Render Props 组建的话。

    :::javascript
    export class CollatzRenderProps extends Component {
        constructor(props) {
            super(props)
            this.initN = parseInt(Math.random() * 100, 10)
        }
        render() {
            return (
                <CollatzWrapper initN={this.initN}>
                    {(n, calcNext, onNChange) => (
                        <>
                            <h1>{n}</h1>
                            <button onClick={calcNext}>next</button>
                        </>
                    )}
                </CollatzWrapper>
            )
        }
    }

而且，Render Props 也生成了一个 Wrapper 包装层。

![RPComp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/react-hooks/RPComp.png)

### 拥抱 Hooks

其实我们需要的只是 `n` 以及 `calcNext`，但是由于 state 与组件不能分离，要想共用这些 `stateful logic`，就只能把共用逻辑的 `state` 提升到 HOC，或者使用放到一个特殊组件内，使用 Render Props 进行渲染，但是这些方法在解决问题的带来了更多前述问题。**Hooks**则实现了 `stateful logic` 与组件的解耦，就像我们的组件和`state`是单独管理的一样，然后通过 `Hooks` 将它们 Hook(勾) 起来，如此，state 就不再依赖于单独的组件（虽然内部每个 Hooks 产生的内部状态依然作用于单个组件）。

采用 Hooks，我们的代码可以改写成下边的格式：

    :::javascript
    function useCalcNext(initN) {
        let [n, setN] = useState(initN)
        let calcNext = function() {
            if (n % 2) {
                n = 3 * n + 1
            } else {
                n = n / 2
            }
            setN(n)
        }
        return [n, calcNext]
    }

使用 Hooks，我们定义了`useCalcNext`之后，就可以在各个组件之间使用，同时不用使用额外的 Wrapper 对组件进行包装，代码可读性也明显更好。

    :::javascript
    export function CollatzHooks(props) {
        let [n, calcNext] = useCalcNext(parseInt(Math.random() * 100, 10))
        return (
            <>
                <h1>{n}</h1>
                <button onClick={calcNext}>next</button>
            </>
        )
    }

组件结构如下：
![HooksComp](https://raw.githubusercontent.com/clumsyme/blogs/master/imgs/react-hooks/HooksComp.png)

##  尝试实现一个简单的 `useState`

    :::javascript
    class Hooks {
        constructor() {
            this.cacheObj = null
            this.useState = this.useState.bind(this)
        }
        useState(initValue) {
            let cacheObj = this.cacheObj
            if (!cacheObj) {
                this.cacheObj = { value: initValue }
                cacheObj = this.cacheObj
            }
            function updater(newValue) {
                cacheObj.value = newValue
            }
            return [cacheObj.value, updater]
        }
    }

    let useState = new Hooks().useState

    function render() {
        let [count, setCount] = useState(0)
        console.group(`render`)
        console.log('count ', count)
        setCount(count + 1)
        console.groupEnd()
    }
    render()
    // render
    //   count 0
    render()
    // render
    //   count 1
    render()
    // render
    //   count 2
    render()
    // render
    //   count 3

*该实现只是简单试验下，对于一个组件内多个 useState，还需要更多改进* 

参考：

-   [Introducing Hooks](https://reactjs.org/docs/hooks-intro.html)
-   [How does React associate Hook calls with components](https://reactjs.org/docs/hooks-faq.html#how-does-react-associate-hook-calls-with-components)
-   [Render Props](https://reactjs.org/docs/render-props.html)
-   [Higher-Order Components](https://reactjs.org/docs/higher-order-components.html)
