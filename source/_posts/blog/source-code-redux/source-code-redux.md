---
title: Redux源码解读
author: 大大白
date: 2020-03-24 15:55:00
categories:
- WEB前端
- 源码解读
tags:
- Redux
---

Redux 是 JavaScript 状态容器，提供可预测化的状态管理。
可以让你构建一致化的应用，运行于不同的环境（客户端、服务器、原生应用），并且易于测试。不仅于此，它还提供 超爽的开发体验，比如有一个时间旅行调试器可以编辑后实时预览。
Redux 除了和 React 一起用外，还支持其它界面库。 它体小精悍（只有2kB，包括依赖）。
本文先简单介绍redux的用法，然后分析redux的原理，最后再根据自己的理解手写一个redux。读完本文你将对redux有一个更加深刻的认识。

<!-- more -->

## 使用Redux
首先简单介绍一下redux的用法。
### store
核心方法：`createStore(reducer, [initialState], enhancer)`

```js
// index.js
import React from 'react'
import { render } from 'react-dom'
import { createStore, applyMiddleware } from '@/common/redux'
import logger from '@/common/redux-logger'
import thunk from '@/common/redux-thunk'

import reducers from './reducers'
import Counter from './components/Counter'

const store = createStore(reducers, { counter: 0 }, applyMiddleware(thunk, logger))
const renderWithStore = () => render(
  <Counter
    data={store.getState()}
    dispatch={store.dispatch}
  />,
  document.getElementById('root')
)

renderWithStore()
store.subscribe(renderWithStore)
```
`store.getState()`获取sotre中的状态。
`store.dispatch`调用action。
`store.subscribe()`监听事件。每当dispatch触发时，所有注册到监听流的函数都会执行，这也是为什么状态改变时视图也会更新的关键。

store 订阅了所有的dispatch事件，只要触发了dispatch就会执行。可以理解成store里面是一个个key，而这个dispatch执行后会一个个去对比。
```js
store.subscribe(() => {
  console.info('test')
})
```

### reducers
reducer是纯函数，接受输入返回相应的值。
```js
// reducers/index.js
import { combineReducers } from '@/common/redux'
import counter from './counter'
export default combineReducers({
  counter
})

// reducers/counter.js
export default (state = 0, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    case 'INCREMENT_BY':
      return state + action.count
    default:
      return state
  }
}
```

### action

```js
// actions/counter.js
export function increment () {
  return {
    type: 'INCREMENT'
  }
}

export function decrement () {
  return {
    type: 'DECREMENT'
  }
}

export function incrementAsync () {
  return dispatch => {
    setTimeout(() => {
      dispatch(increment())
    }, 1000)
  }
}

export function incrementBy (count = 0) {
  return {
    type: 'INCREMENT_BY',
    count
  }
}
```

### smart & dump
在smart中处理state和function，然后连接到组件。
```js
// smart
// container/Counter.js
import { connect } from 'react-redux'
import CounterPage from 'components/counterPage'
import { increment, decrement } from 'actions/counter'

const mapStateToProps = state => {
  return {
    counter1: state.counter,
  }
}

const mapDispatchToProps = dispatch => {
  return {
    increment: () => {
      dispatch(increment())
    },
    decrement: args => {
      dispatch(decrement(args))
    },
  }
}

const HeaderSmart = connect(
  mapStateToProps,
  mapDispatchToProps,
)(HeaderDump)
export default HeaderSmart

// dump
// components/Counter.js
import React, { Component } from 'react'
import PropTypes from 'prop-types'
import { bindActionCreators } from '@/common/redux'
import { counterCreator } from '../actions'

export default class Counter extends Component {
  constructor (props) {
    super(props)
  }
  _incincrement = (args) =>{
    this.props.increment(args)
  }
  render () {
    const { data } = this.props

    return (
      <div>
        <p>
          Counter: {data.counter} times
          {' '}
          <button onClick={this._incincrement}>
            +
          </button>
          {' '}
          <button onClick={this._decrement}>
            -
          </button>
          {' '}
          <button onClick={this._incrementIfOdd}>
            Increment if odd
          </button>
          {' '}
          <button onClick={this._incrementAsync}>
            Increment async
          </button>
          {' '}
          <button onClick={this._incrementBy}>
            Increment by
          </button>
        </p>
      </div>
    )
  }
}
```
[内容来源参考](https://github.com/ansenhuang/ansenhuang.github.io/issues/29)

## 理解Redux
懂得如何使用Redux后，还有几个点需要掌握。

### reducer
我们经常说的reducer是什么？为什么要叫reducer？
从字面上看，它的意思是减少缩小，要把数组逐步减小缩小。
从代码上看，它在对数组逐步缩小的过程中，是这把数组从左到右依次拿出来遍历。第一次先拿出第1个和第2个，第二次拿出第3个，第三次拿出第4个，以此类推。
参数total，如果函数有return会把值放到total里面，给下一个用。
参数curr，从数组第1个元素开始算起（不是第0个，因为第0个默认赋值给total了。）

什么是reducer？`(total,curr)=>{console.log('total',total,'curr',curr)}`就是一个reducer。所以为什么说reducer是一个纯函数，这个代码应该可以很好的解释。

```js
[65, 44, 12, 4].reduce((total,curr)=>{console.log('total',total,'curr',curr)})
// total 65 curr 44
// total undefined curr 12
// total undefined curr 4
// 累加器
[65, 44, 12, 4].reduce((total,curr)=>{return total+curr})
// 累减器
[65, 44, 12, 4].reduce((total,curr)=>{return total-curr})
// 第一个元素加了3次
[65, 44, 12, 4].reduce((total,curr)=>{return total+1})
// 第一个元素减了3次
[65, 44, 12, 4].reduce((total,curr)=>{return total-1})
```

### applyMiddleware 
为什么applyMiddleware进来的中间件是从右到左执行的？
它的核心代码很简单就是把中间件增强到store里面。
```js
import { compose } from './utils'

export default function applyMiddleware (...middlewares) {
  return store => {
    const chains = middlewares.map(middleware => middleware(store))
    store.dispatch = compose(...chains)(store.dispatch)

    return store
  }
}
```
注意compose这个方法，是redux中一个重要的方法，它的作用是从右到左来组合多个函数。
`compose(funcA, funcB, funcC) 的结果是 compose(funcA(funcB(funcC())))`
```js
function compose (...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
// 合并function
// 依次遍历func，把后面的func作为前面的func的参数
[func1, func2, func3].reduce((a, b) => (...args) => a(b(...args)))
// (...args)=>func1(func2(func3(...args)))
```

### combineReducers

由于Redux是单一状态流管理的模式，因此如果有多个reducer，我们需要合并一下，这块的逻辑比较简单，直接上代码。
```js
export default function combineReducers (reducers) {
  const availableKeys = []
  const availableReducers = {}

  Object.keys(reducers).forEach(key => {
    if (typeof reducers[key] === 'function') {
      availableKeys.push(key)
      availableReducers[key] = reducers[key]
    }
  })

  return (state = {}, action) => {
    const nextState = {}
    let hasChanged = false

    availableKeys.forEach(key => {
      nextState[key] = availableReducers[key](state[key], action)

      if (!hasChanged) {
        hasChanged = state[key] !== nextState[key]
      }
    })

    return hasChanged ? nextState : state
  }
}
```

### bindActionCreators
这个方法就是将我们的action和dispatch连接起来。
```js
function bindActionCreator (actionCreator, dispatch) {
  return function () {
    dispatch(actionCreator.apply(this, arguments))
  }
}

export default function bindActionCreators (actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }

  const boundActionCreators = {}

  Object.keys(actionCreators).forEach(key => {
    let actionCreator = actionCreators[key]

    if (typeof actionCreator === 'function') {
      boundActionCreators[key] = bindActionCreator(actionCreator, dispatch)
    }
  })

  return boundActionCreators
}
```
它返回一个方法集合，直接调用来触发dispatch。


### 中间件
在自己动手编写中间件时，你一定会惊奇的发现，原来这么好用的中间件代码竟然只有寥寥数行，却可以实现这么强大的功能。
```js
// logger
function getFormatTime () {
  const date = new Date()
  return date.getHours() + ':' + date.getMinutes() + ':' + date.getSeconds() + ' ' + date.getMilliseconds()
}

export default function logger ({ getState }) {
  return next => action => {
    /* eslint-disable no-console */
    console.group(`%caction %c${action.type} %c${getFormatTime()}`, 'color: gray; font-weight: lighter;', 'inherit', 'color: gray; font-weight: lighter;')
    // console.time('time')
    console.log(`%cprev state`, 'color: #9E9E9E; font-weight: bold;', getState())
    console.log(`%caction    `, 'color: #03A9F4; font-weight: bold;', action)

    next(action)

    console.log(`%cnext state`, 'color: #4CAF50; font-weight: bold;', getState())
    // console.timeEnd('time')
    console.groupEnd()
  }
}
// thunk
export default function thunk ({ getState }) {
  return next => action => {
    if (typeof action === 'function') {
      action(next, getState)
    } else {
      next(action)
    }
  }
}
```
这里要注意的一点是，中间件是有执行顺序的。像在这里，第一个参数是thunk，然后才是logger，因为假如logger在前，那么这个时候action可能是一个包含异步操作的方法，不能正常输出action的信息。

[内容来源参考](https://zhuanlan.zhihu.com/p/38615602)







## 手写Redux 
到现在为止，我们应该是对redux的用法以及基本原理都比较清楚了。如果你上面的源码有些没看懂没关系，接下来我们将手写一个redux，你再回过头去看它的源码应该就能知道是怎么回事了。

### createStore
我们一直说redux是一个状态管理器。既然是状态管理，自然就要有get和set了。getState用来获取状态，dispatch用来设置状态。我们的dispatch传递的参数是action。所以代码可以写成下面这样。
```js
const createStore = ()=>{
  let currentState={}
  getState = () => {
    return currentState
  }
  dispatch =( action) => {
    switch(action.type){
      case 'plus':{
        return {
          ...currentState,
          count: currentState.count + action.payload
        }
      }
    }
  }
  return {
    getState,
    dispatch
  }
}
```
这样子get和set的方法就都有了，但是很明显这有一个问题，就是dispatch里面的内容会越来越臃肿。所以我们需要对这个dispatch改一下。从外面传进来。
```js
const initState = {
  count:0
}
const myReducer = (state=initState,action) => {
  const {type,payload} = action
  switch(type){
      case 'plus':{
        return {
          ...state,
          count: state.count + payload
        }
      }
    }
}
const createStore = (reducer)=>{
  let currentState
  getState = () => {
    return currentState
  }
  dispatch = (action) => {
      currentState = reducer(currentState, action)
  }
  return {
    getState,
    dispatch
  }
}
let store = createStore(myReducer)
console.log(store.getState())
store.dispatch({type:'plus',payload:11})
console.log(store.getState())
```
可以看到我们的store已经开始运作，但是仅仅能改变数据肯定是不够的，我们还需要能让组件自动更新这个数据。所以我们需要加入一个订阅的机制。
发布订阅模式和观察者模式有什么区别？他们最大的区别就是发布订阅模式有个事件调度中心，他们的联系是发布订阅模式是观察者模式的一种实现，发布订阅模式是广义上的观察者模式。
React Hooks的实现原理也是使用了数组。
```js
const initState = {
    count:0
}
const myReducer = (state=initState,action) => {
    const {type,payload} = action
    switch(type){
        case 'plus':{
          return {
            ...state,
            count: state.count + payload
          }
        }
      }
}
const createStore = (reducer)=>{
    let currentState
    let observer = []
    getState = () => {
      return currentState
    }
    dispatch = (action) => {
        currentState = reducer(currentState, action)
        observer.forEach(func => func())
    }
    subscribe = (func) => {
        observer.push(func)
    }
    return {
      getState,
      dispatch,
      subscribe
    }
  }
  let store = createStore(myReducer)
  console.log(store.getState())
  store.subscribe(()=>{console.log('订阅store状态变化！')})
  store.dispatch({type:'plus',payload:11})
  console.log(store.getState())
```
到这里store应该算是挺完整了getState,dispatch,subscribe都有了。

### applyMiddleware
接下来我们思考一个问题，我希望每次调用action后都能打印出log。还有我希望能知道修改状态所花的时间。
我们可以每次执行完dispatch之后getState
```js
let store = createStore(myReducer)
store.dispatch({type:'plus',payload:11})
console.log(store.getState())
```
但是我不能在所有的地方都这样做，这种做法太low了，有没有什么办法执行了dispatch就能自动getState。当然我们的第一个感觉是封装一下。
```js
let store = createStore(myReducer)
const myDispatch = (store,action) => {
  store.dispatch(action)
  console.log(store.getState())
}
myDispatch(store,{type:'plus',payload:12})
```
好了，现在只要调用myDispatch就能自动打印出log了。但是这样做有什么问题呢？是不是必须在需要的地方调用一下这个方法才可以，有没有什么办法是可以让log打印在全局范围内有效呢？
最直接的想法就是修改store的dispatch
```js
let store = createStore(myReducer)
let next = store.dispatch
store.dispatch = (action)=>{
  let result = next(action)
  console.log(store.getState())
  return result
}
```
但是这种做法也有问题，因为它破坏了store的方法。另外就是如果未来有更多的需求无法扩展，或者说代码会变得越来越臃肿。所以我们可以再封装一下。
```js
let store = createStore(myReducer)
const dispatchWithLog = (store) => {
    let next = store.dispatch
    store.dispatch = (action)=>{
        console.log(1,store.getState())
        let result = next(action)
        console.log(2,store.getState())
        return result
    }
}
const dispatchWithLog2 = (store) => {
    let next = store.dispatch
    store.dispatch = (action)=>{
        console.log(3,store.getState())
        let result = next(action)
        console.log(4,store.getState())
        return result
    }
}
dispatchWithLog(store)
// dispatchWithLog2(store)
store.dispatch({type:'plus',payload:123})
```
这样封装后中间件就变得可插拔，需要才加入，不需要就不加入。
现在看起来正在慢慢接近中间件的特性了，但是这种写法还是没有解决刚才的问题，就是他破坏了store的dispatch方法。所以我们可以改成不修改dispatch而是直接return
```js
let store = createStore(myReducer)
const dispatchWithLog = (store) => {
    let next = store.dispatch
    return (action)=>{
        console.log(1,store.getState())
        let result = next(action)
        console.log(2,store.getState())
        return result
    }
}
const dispatchWithLog2 = (store) => {
    let next = store.dispatch
    return (action)=>{
        console.log(3,store.getState())
        let result = next(action)
        console.log(4,store.getState())
        return result
    }
}
store.dispatch = dispatchWithLog(store)
// store.dispatch = dispatchWithLog2(store)
store.dispatch({type:'plus',payload:123})
```
可以看到store.dispatch被抽取了出来执行了两次，因此我们可以用一个方法来实现它。
```js
const applyMiddleware = (store,middlewares) => {
    middlewares.forEach(middleware => store.dispatch = middleware(store) )
}
let store = createStore(myReducer)
// store.dispatch = dispatchWithLog(store)
// store.dispatch = dispatchWithLog2(store)
applyMiddleware(store,[dispatchWithLog,dispatchWithLog2])
store.dispatch({type:'plus',payload:123})
```
这个就是applyMiddleware的雏形了，它的作用是增强store的效果，但是一直到现在还是没有解决刚才这个侵入了store的dispatch的问题。

我们先对中间件进行柯里化
`dispatchWithLog(store)`返回一个接收dispatch的函数
`dispatchWithLog(store)(dispatch)`返回一个接收action的函数，这个函数就是一个新的dispatch

然后修改applyMiddleware
`middlewares.map(middleware => middleware(store))`相当于把函数解开了，变成`dispatchWithLog(store)`这个表达式的意思是返回一个函数，函数接收的参数是一个dispatch。
compose的结果就是`(args)=>dispatchWithLog(store)(dispatchWithLog2(store)(args))`，`compose(...dispatchArray)(dispatch)`就是把dispatch作为参数传递给args结果就是`dispatchWithLog(store)(dispatchWithLog2(store)(dispatch)`
有点难懂，你可以这样理解
dispatch1 = dispatchWithLog(store) , dispatch2 = dispatchWithLog2(store)
他的作用其实是这样的dispatch1(dispatch2(dispatch))

然后把这些接收dispatch的函数进行compose，自己返回的函数作为前面一个函数的参数
```js
const dispatchWithLog = store => next => action => {
    console.log(1,store.getState())
    let result = next(action)
    console.log(2,store.getState())
    return result
}
const dispatchWithLog2 = store => next => action => {
    console.log(3,store.getState())
    let result = next(action)
    console.log(4,store.getState())
    return result
}

const applyMiddleware = (store, middlewares) => {
    let {dispatch} = store
    const dispatchArray = middlewares.map(middleware => middleware(store))
    dispatch = compose(...dispatchArray)(dispatch)
    return {...store, dispatch}
}
    
const compose = (...fns) => {
    if (fns.length === 0) return arg => arg    
    if (fns.length === 1) return fns[0]    
    return fns.reduce((res, cur) =>(...args) => res(cur(...args)))
}

let store = createStore(myReducer)
store.dispatch({type:'plus',payload:123})
store = applyMiddleware(store,[dispatchWithLog,dispatchWithLog2])
store.dispatch({type:'plus',payload:123})

```
现在我们离真正的applyMiddleware已经非常接近了，但是还需要做一点修改。（不看也可以前面基本把最重要的内容都说了。）只需要再把applyMiddleware进一步柯里化一下，同时避免用户修改dispatch，所以先把store的dispatch保存下来。
```js
const applyMiddleware = (...middlewares) => createStore => reducer => {    
    const store = createStore(reducer)    
    let { getState, dispatch } = store    
    const params = {      
        getState,      
        dispatch: (action) => dispatch(action)      
    }    

    const middlewareArr = middlewares.map(middleware => middleware(params)) 
   
    dispatch = compose(...middlewareArr)(dispatch)    
    return { ...store, dispatch }
}
```
### 其余的api
前面已经把redux最核心的createStore和applyMiddleware都讲完了剩下的几个比较简单。
`bindActionCreators`顾名思义用来绑定创建action，`combineReducers`用来合并reducer。源码都不难我就不过多讲解了。


## 总结
看完大佬写的代码给，我内心最深的感受就是一个大写的服！太精妙了！这中间里面包含了许多设计思想——观察者模式、装饰器模式、中间件机制、函数柯里化、函数式编程等。

### 三大原则
#### 单一数据源
整个应用的 state 被储存在一棵 object tree 中，并且这个 object tree 只存在于唯一一个 store 中。
这让同构应用开发变得非常容易。来自服务端的 state 可以在无需编写更多代码的情况下被序列化并注入到客户端中。由于是单一的 state tree ，调试也变得非常容易。在开发中，你可以把应用的 state 保存在本地，从而加快开发速度。此外，受益于单一的 state tree ，以前难以实现的如“撤销/重做”这类功能也变得轻而易举。

#### State 是只读的
唯一改变 state 的方法就是触发 action，action 是一个用于描述已发生事件的普通对象。
这样确保了视图和网络请求都不能直接修改 state，相反它们只能表达想要修改的意图。因为所有的修改都被集中化处理，且严格按照一个接一个的顺序执行，因此不用担心竞态条件（race condition）的出现。 Action 就是普通对象而已，因此它们可以被日志打印、序列化、储存、后期调试或测试时回放出来。

#### 使用纯函数来执行修改
为了描述 action 如何改变 state tree ，你需要编写 reducers。
Reducer 只是一些纯函数，它接收先前的 state 和 action，并返回新的 state。刚开始你可以只有一个 reducer，随着应用变大，你可以把它拆成多个小的 reducers，分别独立地操作 state tree 的不同部分，因为 reducer 只是函数，你可以控制它们被调用的顺序，传入附加数据，甚至编写可复用的 reducer 来处理一些通用任务，如分页器。

### Redux 与传统后端 MVC 的对照
| Redux                       |传统后端 MVC |
|-----------------------------|------------|
|store                        |数据库实例
|state	                      |数据库中存储的数据
|dispatch(action)             |  用户发起请求
|action: { type, payload }    | type 表示请求的 URL，payload 表示请求的数据
|reducer                      |路由 + 控制器（handler）
|reducer中的 switch-case 分支  | 路由，根据 action.type 路由到对应的控制器
|reducer内部对 state 的处理    | 控制器对数据库进行增删改操作
|reducer返回 nextState        | 将修改后的记录写回数据库
