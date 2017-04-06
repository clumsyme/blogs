初学React时，使用组件自身管理状态，这样当业务逻辑比较简单时还比较有用，一旦业务逻辑复杂起来，各组件之间要相互通信，情况就复杂起来了。可以使用状态提升，将各子组件状态提升到父组件内，这样又要经历事件向上传递 --》 属性向下传递的过程，如果组件嵌套较深，回调传递也是十分复杂。好在有了Redux，可以将整个项目的状态存储到一个对象内，各组件分别与该对象进行交互，逻辑就清晰多了。但是有了Redux，是否就可以抛去`setState`方法了？

## 表单

一个例子是表单。如果我们使用受控组件，那么每一个表单项`onChange`事件都应该触发Redux事件，改变状态，再作为属性传递给表单，和状态提升思路差不多。但是可以发现，其他组件（**一般**）是不需要访问我们表单的状态的，将表单状态传递到全局Redux，最后还是传递回表单。对比一下两者需要的工作：

### 使用Redux

- 编写action、reducer、store
- 编写state、dispatch与props的映射函数
- connect组件

### 使用`setState`

    this.setState({
      //***
    })

这种情况下，明显由组件自身管理状态更为方便。

## Redux VS setState

> Use React for ephemeral state that doesn’t matter to the app globally and doesn’t mutate in complex ways. For example, a toggle in some UI element, a form input state. Use Redux for state that matters globally or is mutated in complex ways. For example, cached users, or a post draft.<br>
> —— Dan Abramov

简而言之，在无关全局状态的组件内部，应该使用React的`setState`来自管理状态，而关乎全局状态的组件应该使用Redux来进行管理。