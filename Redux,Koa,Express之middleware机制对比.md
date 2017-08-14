---
title: Redux,Koa,Express之middleware机制对比
category: BackEnd
tags: [middleware,Redux,Koa,Express,redux-thunk,函数式编程,compose,curring]
---
看到标题，也许您会觉得奇怪，redux跟Koa以及Express并不是同一类别的框架，干嘛要拿来做类比。尽管，例如Express以及koa等这类的middleware是**指可以被嵌入在框架接收请求到产生响应过程之中的代码**。而redux的middleware是**提供的是位于 action 被发起之后，到达 reducer 之前的扩展点**。我觉得，不管是什么框架，一种思想才是最重要的。相同的概念，可能实现的思路与方法不同，同样值得我们去深究，去学习。

面对多样的业务场景，前后端都需要一种插件机制，可以随意组合。middleware即是这种可以自由组合，自由插拔的插件机制。所以我们就来横向比较一下middleware在redux,koa以及express的运行机制，本文主要通过以下三点来进行比较:
1. 异步编程模式
2. middleware的使用
3. middleware的执行原理及实现

### 异步编程模式
理解异步编程方式式理解几种middleware执行原理的前提。


框架 | 异步方式
---|---
Express | callback
Koa1 | generator/yield+co
Koa2 | Async/Await
Redux | redux-thunk,redux-saga,redux-promise等

- express: 由于是在ES6之前出现的，所以中间件的基础原理还是callback方式
- koa: koa1得益于generator特性并通过co框架加入了自动流程管理。（co会把所有generator的返回封装成为Promise对象）koa2则使用了Async/Await的形式（仅仅知识genertator函数的语法糖）
- redux: 论redux的异步变成，更准确的应该是异步的action的方式。解决异步action的方式有多种，比如：redux-thunk,redux-saga,redux-promise等

> 本文并不会对js里的异步编程作详细，推荐阮一峰老师写的《深入掌握 ECMAScript 6 异步编程》系列文章。

### middleware的使用
众所周知，Koa是Express框架原班人马基于ES6新特性重新开发的敏捷开发框架。Express主要是基于Connect中间件框架，其自身封装了大量的功能，比如路由、请求等。而Koa是基于co(koa2基于async/await)中间件框架，框架自身并没集成太多功能，大部分功能需要用户自行require中间件去解决。我们首先来比较一下Koa与Express的写法：
```javascript
//Express
var express = require('express')
var app = express()

app.get('/',(req,res)=>{
    res.send('Hello Express!')
})
app.listen(3000)

```
```javascript
//Koa
var koa = require('koa')
var app = koa()
var route = require('koa-route')

app.use(route.get('/',async (ctx) => {
    ctx.body = 'Hello Koa'
}))

app.listen(3000)

```

redux使用包含自定义功能的middleware来扩展。Middleware可以让你包装store的dispatch方法来达到你想要的目的。同时，middleware还拥有"compose"这一关键特性。(这个下文会讲)多个middleware可以被组合到一起使用，形成middleware链。其中，每个middleware都不需要关心链中它前后的middleware 的任何信息。

例如：
```javascript
const logger = ()=>{
    // ...
}
const crashReporter = ()=>{
    // ...
}
const thunk = () =>{
    // ...
}

let store = createStore(
  App,
  applyMiddleware(
    crashReporter,
    thunk,
    logger
  )
)
```


### middleware的执行原理及实现
##### express的middleware执行原理及实现
Express更像是中间件顺序执行，称之为线性模型

```
↓
---------------
| middleware1 |
---------------
       ↓
---------------
| ... ... ... |
---------------
       ↓
---------------
| middlewareN |
---------------
       ↓
```

其实express middleware的原理很简单，express内部维护一个函数数组，这个函数数组表示在发出响应之前要执行的所有函数，也就是中间件数组，每一次use以后，传进来的中间件就会推入到数组中，执行完毕后调用next方法执行函数的下一个函数，如果没用调用，调用就会终止。
下面我们实现一个简单的Express中间件功能
```
function express() {
    var funcs = [] // 中间件存储的数组
    var app = function (req, res) {
        var i = 0  
        // 定义next()
        function next() {
            var task = funcs[i++]  // 取出中间件数组里的下一个中间件函数
            if (!task) {    // 如果中间件不存在,return
                return
            }
            task(req, res, next);   // 否则,执行下一个中间件
        }
        next()
    }
    // use方法则是将中间件函数推入到中间件数组中
    app.use = function (task) {
        funcs.push(task);
    }
    return app    // 返回实例
}
```

##### koa的middleware执行原理及实现
Koa会把多个中间件推入栈中，与express不同，koa的中间件是所谓的洋葱型模型。
![image](https://camo.githubusercontent.com/d80cf3b511ef4898bcde9a464de491fa15a50d06/68747470733a2f2f7261772e6769746875622e636f6d2f66656e676d6b322f6b6f612d67756964652f6d61737465722f6f6e696f6e2e706e67)

koa的中间件的实现主要依靠的是koa-compose。首先我们来看下koa-compose的使用,koa-compose模块可以将多个中间件合成为一个:
```javascript
const Koa = require('koa')
const compose = require('koa-compose')
const app = new Koa()

const logger = (ctx, next) => {
  console.log(`${Date.now()} ${ctx.request.method} ${ctx.request.url}`)
  next()
}

const main = ctx => {
  ctx.response.body = 'Hello Koa'
};

const middlewares = compose([logger, main])

app.use(middlewares)
app.listen(3000)
```
下面我们来分析一下koa-compose的源码
```javascript
module.exports = compose

/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */

function compose (middleware) {
  // 错误处理
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      // 当前执行第 i 个中间件
      index = i
      let fn = middleware[i]
      // 所有的中间件执行完毕
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        // 执行当前的中间件
        // 这里的fn也就是app.use(fn)中的fn
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}

```
Koa的中间件支持普通函数，返回一个Promise的函数，以及async函数。

##### redux的middleware执行原理及实现
> "It provides a third-party extension point between dispatching an action,and the moment it reaches the reducer"
这是redux作者的描述。

正因为redux单一数据源的特点，数据从顶层流动，middleware就好比管道去辅助这些数据的流向，不同的管道具有不同的特点与功能。

每个middleware函数接受Store的dispatch和getState函数作为命名参数，并返回一个函数。该函数会被传入被称为next的下一个middleware的 dispatch方法，并返回一个接收action的新函数，这个函数可以直接调用 next(action)，或者在其他需要的时刻调用，甚至根本不去调用它。调用链中最后一个middleware会接受真实的store的dispatch方法作为next参数，并借此结束调用链。

下面以redux-thunk为例，来介绍下如何写一个redux middleware，下面是redux-thunk的源码
> redux-thunk帮助你统一了异步和同步action的调用方式，把异步过程放在action级别解决

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```
一共只有11行，action本身是一个object，带有type和arguments。上述将dispatch和getState传入action，next()和action()是redux提供的方法。接着做判断，如果action是一个function，就返回action(dispatch, getState,extraArgument)，否则返回next(action)。

然后将他们引用到Redux Store中
```javascript
import { createStore, combineReducers, applyMiddleware } from 'redux'

let todoApp = combineReducers(reducers)
let store = createStore(
  todoApp,
  // applyMiddleware() 告诉 createStore() 如何处理中间件
  applyMiddleware(thunk)
)
```

首先我们从源码出发
```js
import compose from './compose'

export default function applyMiddleware(...middleware){
    return (next) => (reducer,initialState) => {
        let store = next(reducer, initialState)
        let disptach = store.dispatch
        let chain = []
        
        var middlewareAPI = {
            getState: store.getState,
            dispatch: (action) => dispatch(action)
        } 
        chain = middlewares.map(middlre => middleware(middlewareAPI))
        dispatch = compose(...chain)(store.dispatch)
        
        return {
            ...store,
            dispatch
        }
    }
}
```
下面我们从以下几个点介绍redux的middleware机制。
1. 函数式编程思想
redux middlreware的思想是使用匿名单参数函数来实现多参数函数的方法。

> ES6实现一个curring函数
> ```javascript
> function curring(fn){
>     return function curried(...args){
>         return args.length >= fn.length ? fn.call(this,...args):(...rest)=>{
>             return curried.call(this,...args,...rest)
>         }
>     }
> }
> ```
好处：
- 易串联
- 共享store
最后我们来总结一下三者的区别

2. 分发store

创建一个普通的store
```js
let newStore = applyMiddleware(mid1,mid2,...)(createStore)(reducer,null)
```
由于
```js
var middlewareAPI = {
            getState: store.getState,
            dispatch: (action) => dispatch(action)
        } 
chain = middlewares.map(middlre => middleware(middlewareAPI))
```
因为闭包，每个匿名函数都可以访问相同的store，即middlewareAPI。
3. compose
compose 的源码就是一个函数 compose :
```js
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }

  if (funcs.length === 1) {
    return funcs[0]
  }
  // 获取最后一个函数
    const last = funcs[funcs.length - 1];
    // 获取除最后一个以外的函数[0,length-1)
    const rest = funcs.slice(0, -1)
   // 通过函数 curry 化
  return (...args) => rest.reduceRight((composed, f) => f(composed), last(...args))
}

```
这里的compose跟上文中的koa-compose有些类似。属于函数式编程中的组合，它将chain中的所有匿名函数[f1,f2,f3,...,fn]组成一个新的函数，即新的dispatch,假设n = 3:
dispatch = f1(f2(f3(store.dispatch)))
这时调用新dispatch，每一个middleware就依次执行了。

4. 在middleware中调用dispatch

![image](http://oum6ifofe.bkt.clouddn.com/image/v2-e5b8f433fec45c09260759fb12e90bb6_b.png)

从上图中得出结论，middleware通过next(action)一层层处理和传递action直到redux原生的dispatch。而如果某个middleware使用store.dispatch(action)来分发action，就相当于重新来一遍。

**在middleware中使用dispatch的场景一般是接受一个定向action，这个action并不希望到达原生的分发action,往往用在一步请求的需求里，比如上面提到的redux-thunk，就是直接接受dispatch。**

下面我们来总结一下三者的区别

express 
 1. 中间件为一个方法，接受 req,res,next三个参数。
 2. 中间可以执行任何方法包括异步方法。
 3. 最后一定要通过res.end或者next来通知结束这个中间件方法。
 4. 如果没有执行res.end或者next访问会一直卡着不动直到超时。
 5. 并且在这之后的中间件也会没法执行到。

koa 
1. 中间件为一个方法或者其它，接受ctx,next两个参数。
2. 方法中可以执行任何同步方法。可以使用返回一个Promise来做异步。
3. 中间件通过方法结束时的返回来判断是否进入下一个中间件。
4. 返回一个Promise对象koa会等待异步通知完成。then中可以返回next()来跳转到下一个中间件。
5. 如果Promise没有异步通知也会卡住。

Redux 
1. 中间件为一个方法，接受store参数。
2. 中间可以执行任何方法包括异步方法。
3. 中间件通过组合串联middlware。
4. 通过next(action)处理和传递action直到redux原生的dispatch,或者使用store.dispatch(actio)来分发action。
5. 如果一只简单粗暴调用store.dispatch(action)，就会形成无限循环。



### 参考文档
---
- [你知道 koa 中间件执行原理吗?](https://juejin.im/post/591c8b4544d904006c90a2cb)
- [redux gitbook](http://cn.redux.js.org/docs/api/applyMiddleware.html)
- [redux-thunk of stackoverflow](https://stackoverflow.com/questions/35411423/how-to-dispatch-a-redux-action-with-a-timeout/35415559#35415559)
- [Koa2原理详解](http://syaning.com/2016/11/08/koa2/)
- 《深入React技术栈》