今天看代码，看到一个异步actionCreater，返回了一个函数而不是一个带有`type`属性的action object。大致如下：

```js
const asyncActionCreater = (action, url) => (query) => (dispatch, getState) => {
  // dosth()
  // dispatch(action.xyz)
}
```

可以通过调用获得一个异步action creater

```js
const asyncAction =  asyncActionCreater(someAction, someUrl)
```

如此，调用`asycnAction(someQuery)`，将返回一个函数对象，接受`dispatch`和`getState`为参数。这样在后续使用`mapDispatchToProps`时：

```js
const mapDispatchToProps = (dispatch) => ({
  ...bindActionCreaters({ asyncAction }, dispatch)
  // 等于
  // asyncAction: (query) => {dispatch(asyncAction(query))}
})
```

可以看到`dispatch`了一个函数对象。

查看了redux的API文档，`dispatch`的参数应该是action object，google了之后，明白了是因为使用了`redux-thunk`中间件。

## redux-thunk 中间件

`redux-thunk`中间件的作用就是可以让`dispatch`接受函数作为参数，这样就可以控制何时触发一个action，从而可以触发异步或其他条件action。

例1：只在偶数次调用

```js
const increEven = () => (dispatch, getState) => {
  if (getState().count & 1) {
    return
  }
  dispatch(incre())
}

const mapDispatchToProps = (dispatch) => ({
  increEven: () => dispatch(increEven())
})

<button onClick={this.props.IncreEven}>IncreEven</button>
```

如此，当我们点击`button`，dispatch了一个函数，会调用这个函数，检查store内count是否为偶数，是的话调用`incre()` action，否则什么都不做。

例2：异步网络请求

```js
const getDataActions = {
    request: 'FETCH_DATA_REQUEST',
    success: 'FETCH_DATA_SUCCESS',
    failure: 'FETCH_DATA_FAILURE'
}
const url = 'example.com/xyz'
const asyncActionCreater = (action, url) => (query) => (dispatch, getState) => {
  dispatch({type: action.request})
  fetch(url, query)
  .then(data => {
    // ............
    dispatch({type: action.success, ...payload})
  })
  .catch(error => {
    // ...........
    dispatch({type: action.failure, ...payload})
  })
}
const getData = asyncActionCreater(getDataActions, url)

const mapDispatchToProps = (dispatch) => ({
  getData: (query) => dispatch(getData(query))
})
```

如此，我们在调用`getData(query)`时，就会依次触发最内部的函数语句，`request -> success/failure`。

## 参考

- [redux-thunk](https://github.com/gaearon/redux-thunk)
- [异步 Action](http://cn.redux.js.org/docs/advanced/AsyncActions.html) 