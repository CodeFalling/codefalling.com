title: learn-redux-from-zero
author: codefalling
tags:
  - frontend
  - javascript
  - redux
  - react
categories:
  - 编程
date: 2016-10-07 00:00:00
---
前端生态日新月异, flux 已经过时（恩然而我都还没有来得及学），redux 成了状态管理的标配，每一个前端开发者都应该学习。然而因为 redux 的多种概念着实让新手费解，很多人沉浸在 react 全家桶的配置无法自拔。所以我试着丢开 react，理清楚 redux 本身的几个主要概念，并争取循序渐进的讲解 redux 各种概念的用法和意义。

<!-- more -->


## 从零开始

一次性引入过多的新东西会让人不知所措，所以我们尽可能的减轻依赖。同时为了避免新人掉进环境配置(npm install)的大坑，我们直接采用 https://jsfiddle.net/ 来实践。

新建一个 jsfiddle，将语言设为 Babel，在左侧的 External Resources 直接引入 redux: https://cdnjs.cloudflare.com/ajax/libs/redux/3.6.0/redux.min.js 。记得打开开发者工具，方便看到报错和后面的输出。

目前为止这么多足以，不需要引入 react 或者任何其他东西。

## 最简单的例子

很多教程喜欢一开始就把 redux 里的几个概念拆开讲，每个概念又贴上大段大段的代码，弄得人一头雾水。而我们因为暂时丢掉了 react，得以用一个足够简单却完整的例子讲解 redux 的工作流程。

```js
// 这里是因为直接引入了 js 才用这种写法，当本地使用 webpack 时我们应该使用 
// import { createStore, applyMiddleware  } from 'redux'
const { createStore, applyMiddleware } = Redux

const reducer = (state, action) => {
  let result = state
  switch (action.type) {
    case "INC":
      result += action.payload
      break;
    case "DEC":
      result -= action.payload
      break;
  }
  return result
}

const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}

const store = createStore(reducer, 0, applyMiddleware(logger))

store.dispatch({type: 'INC', payload: 1})
store.dispatch({type: 'INC', payload: 2})
store.dispatch({type: 'DEC', payload: 5})
```

观察控制台，我们得到的输出是：

```
dispatching Object {type: "INC", payload: 1}
next state 1
dispatching Object {type: "INC", payload: 2}
next state 3
dispatching Object {type: "DEC", payload: 5}
next state -2
```

这个短小的例子已经用到了我们要讲的几个主要概念，reducer、store、middleware、action。

store 负责状态的存储，需要注意的是，在 redux 中只有一个 store，而如何在一个 store 中维护众多的状态，我们后面会提到。

action 则类似触发事件，使用 `store.dispatch` 发送出去，必须要有一个 `type` 属性。同时 action 也可以附带其他数据，例如上面代码里的 `payload`。

reducer 在每次有新的 action 时被触发，则只做一件事情，就是收到当前的 state 和 action，并且返回一个全新的 state。

middleware 则发生在每次 action dispatch 后，reducer 触发前，在 middleware 中调用 `next(action)` 才会将 action 传递给 reducer 并且返回新的结果。这种设计可以让我们很灵活的实现一些针对所有 action 的功能，例如 log 或者针对特定类型的 action 做一些处理等。

这样，整个 redux 就相当于一个状态机，新的 action 被触发，经过 middleware，由 reducer 产生新的状态。

![redux-flow](\images\redux_flow.png)
我特意没有把他们画成一个循环，因为 state 是不应该改变的，只是由 reducer 返回新的 state。也正以为这个原因，我们在使用 redux 开发应用时，可以很轻松地跟踪到状态的变化，将状态直接 revert 到某个点，撤销这一类的功能非常容易实现。

## immutable

上面我们提到，state 不会改变，而是由 reducer 返回新的 state。这样我们称 state 是 immutable 的。但是 JavaScript 本身没有提供这个特性，我们需要使用一些和平时不同的写法来达到这个目的。在更详细地介绍 redux 的各个概念前，我们先来了解一下 immutable。

immutable 和 const 不是一个含义，const 指的是标识符对应的值不可被改变，例如

```js
const a = 3
a = 4
```

会抛出一个异常，表示 a 不能被修改。

而 immutable 则指的是值本身不能被修改，例如在 JavaScript 中的 string 和 number 等基本类型都是不能被修改的，但 object 和 array 则不是。

```js
a = 'I am string'
b = a
b = 'Oh yes'

c = {name: "Li"}
d = c
d.name = 'K'

console.log(a) // "I am string"
console.log(b) // "Oh yes"
console.log(c) // {name: "K"}
console.log(d) // {name: "K"}
```

可以看到对象的内容被改变了。所以在处理对象和数组的时候，我们需要注意一些，只使用没有副作用（side effect）的操作：

```js
c = {name: "Li", age: 17}
d = {...c, name: "K"}
e = [2, 4, 6]
f = e.slice(0,1).concat(e.slice(2))

console.log(c) // {name: "Li", age: 17}
console.log(d) // {name: "K", age: 17}
console.log(e) // [2, 4, 6]
console.log(f) // [2, 6]
```

当然，我们也可以使用 immutable.js 等库来解决这个问题。

## 一步步改进

了解了 immutable 后我们可以学习更加复杂的 reducer。上面的例子中整个 state 只是一个数字，而这次我们存储更加复杂的数据。以 github user API 为例( https://api.github.com/users/codefalling )，我们来写一段代码来获取指定用户头像。

### 初始化 store
我们将初始化 store 的地方改为：

```js
const store = createStore(reducer, {
  username: null,

  info: {
    fetching: null,
   fetched: null,
  }
},
applyMiddleware(logger))
```

### 异步 action

看起来我们只需要一个获取头像的 action，但其实我们应该还需要一个获取成功的 action，因为获取头像不会马上得到数据，而请求返回后应该更新 state。

在这里我们使用 redux-thunk, (因为 cdn 上找不到 redux-promise 之类的。。)。

同样的，在左侧的 External Resources 中引入 https://cdnjs.cloudflare.com/ajax/libs/redux-thunk/2.1.0/redux-thunk.min.js 。

然后只要将 `applyMiddleware(logger)` 改为 `applyMiddleware(logger, ReduxThunk.default)`。

然后我们就能写出类似

```js
store.dispatch(dispatch => {
  dispatch({ type: 'FETCH_START' })
  fetch('https://api.github.com/users/codefalling').then(res => {
    return res.text()
  })
  .then(data => {
    const obj = JSON.parse(data)
    const avatarUrl = obj['avatar_url']
    const blog = obj.blog
    dispatch({ type: 'FETCH_END', payload: { avatarUrl, blog } })
  })
})
```

然后会先触发 FETCH_START，完成后带着数据触发 FETCH_END。

很容易可以写出对应的 reducer，FETCH_START 时把 fetching 设为 true，FETCH_END 时设置 avatarUrl 并且将 fetched 设为 true，fetching 设为 false.

```js
const reducer = (state, action) => {
  switch (action.type) {
    case "FETCH_START":
      return {
        fetching: true,
      }
    case "FETCH_END":
      return {
        fetching: false,
        fetched: true,
        info: {
          avatarUrl: action.payload.avatarUrl,
          blog: action.payload.blog,
        },
      }
  }
}
```

## 拆分 reducer

前面曾经提到，redux 只有一个 store，当应用变得越来越复杂时，reducer 和 store 中的数据都越来越多。而 reducer 可能并不关心和自己无关的 state，把 reducer 全部写在一个大 switch case 中也不是个好主意，所以我们要将 reducer 拆分开。redux 提供一个函数，combineReducers，用于合并多个 reducer。

以上面的代码为例，假设我们有一个和其没有关系的其他 reducer，我们可以写成：

```js
const { createStore, applyMiddleware, combineReducers } = Redux

const fetchReducer = (state = {}, action) => {
  switch (action.type) {
    case "FETCH_START":
      return {
        ...state,
        fetching: true,
      }
    case "FETCH_END":
      return {
        ...state,
        fetching: false,
        fetched: true,
        info: {
          avatarUrl: action.payload.avatarUrl,
          blog: action.payload.blog,
        },
      }
  }
  return state
}

const helloReducer = (state = {}, action) => {
  if (action.type === 'HELLO') {
  	state = action.payload
  }
  return state
}

const reducer = combineReducers({
  fetch: fetchReducer,
  hello: helloReducer,
})

const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}

const store = createStore(reducer, {}, applyMiddleware(logger, ReduxThunk.default))

store.dispatch(dispatch => {
  dispatch({ type: 'FETCH_START' })
  fetch('https://api.github.com/users/codefalling').then(res => {
    return res.text()
  })
  .then(data => {
    const avatarUrl = JSON.parse(data)['avatar_url']
        dispatch({ type: 'FETCH_END', payload: avatarUrl })
  })
})

```

初始化不再在 store 中完成，而是直接写在缺省参数里。拆分开的 reducer 不能返回 undefined，如果什么都没发生，就原样返回即可。

我们可以看到，多个 reducer 被拆分开，各自独立，state 也不会互相影响。

## 更多
到这里介绍完了 redux 中比较重要的概念，本文为了简单和易于理解没有引入 react，先搞清楚 redux 本身的工作方式，避免陷入茫茫多的细节和概念。将会在未来的文章中介绍 react 和 redux 共同工作的方式。
