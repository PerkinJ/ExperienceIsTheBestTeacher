使用react开发已经很久了，但之前很少去关注代码底层的东西，学得很肤浅。最近在深入了解react&redux技术栈，及相关源码。结合自己所学和所用，总结一些自己的看法，以便今后的学习。
> createStore可以说是redux的精髓，基本上，redux核心功能已经全部被囊括在createStore以及createStore方法最终生成的store中。

createStore的函数签名如下：
```js
export default function createStore(reducer, initialState, enhancer){
    // ...
}
```
本文主要是写一个简易版的createStore，关注的仅仅是第一个参数也是用的最多的reducer参数。

#### 1. 建立简单的demo模型
首先我们先建立简单的redux的demo
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width">
  <title>Redux&createStore</title>
</head>
<body>
</body>
</html>
```
```JS
const counter = (state = 0, 
action) => {
  switch (action.type){
    case 'INCREMENT':
      return state + 1
    case 'DECREMENT':
      return state - 1
    default:
      return state
  }
}
//此处需要定义createStore
// ...
const store = createStore(counter)

const render = () => {
  document.body.innerText = store.getState()
}

store.subscribe(render)
render()

document.addEventListener('click', () => {
  store.dispatch({ type: 'INCREMENT' })
})
```

#### 2. 确定返回值
我们知道，store主要的参数是dispatch,getState以及subscribe。
1. 其中dispatch是store非常核心的一个方法，也是我们在应用中经常直接(store.dispatch({type:'INCREMENT'})或间接(使用connect将action creator与dispatch关联)使用方法--dispatch
2. getState方法主要用于返回当前状态
3. subscribe主要用于全局监听变化，从而做出相应的改变。事实上，react-redux中的connect方法隐式地帮我们完成了这个工作。


```js
const createStore = (reduccer) => {
    let state, listener
    const getState = () => { state }

    const dispatch = ( action ) => {
        
    }
    
    const subscribe = ( listener ) => {
        
    }
    return {getState, dispatch, subscribe}
}

```

#### 3. 完善createStore
```js
const createStore = (reducer) => {
    let state
    //当前监听store变化的监听器
    let listeners = []
    //getState返回当前状态
    const getState = () => state 

    const dispatch = ( action ) => {
        //更新state
        state = reducer(state, action)
       //在得到新的状态以后，依次调用所有的监听器，通知状态的变更。
       listeners.forEach(listener => listener())
    }
    
    const subscribe = ( listener ) => {
        listeners.push(listener)
        //此处主要起到unsubscribe的作用
        return () =>{
            listeners = listeners.filter(l = l !== listener)
        }
    }
    //最终，处理之后的action会被dispatch方法返回，此处假设传了一个空的object
    dispatch({})
    return {getState, dispatch, subscribe}
}
```
[查看结果](http://jsbin.com/mikutip/edit?html,js,console,output)

当然以上代码是最简易的createStore，真正的源码还加上了很多参数校验，判断条件等。

>replaceReducer，这个方法主要用于reducer的热替换，在开发过程中，我们一般不会直接使用这个API

以上就是简易实现createStore的内容，本文主要借鉴redux作者的demo例子以及《深入React技术栈》里面的相关分析。