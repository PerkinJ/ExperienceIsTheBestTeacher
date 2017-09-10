---
title: 基于React与Redux Saga的权限验证应用(一)
category: 前端
tags: [react,redux-saga,redux-form,JWT]
---

>本文是基于[React and Redux Sagas Authentication App Tutorial](https://start.jcolemorrison.com/react-and-redux-sagas-authentication-app-tutorial/)翻译整理，并总结而来，目的是学习redux-saga以及redux-form以及JWT形式的验证方式。

知识点
- Redux-Saga的理解以及使用
- Redux-Form的使用
- 登陆与权限验证,JWT的使用与原理

本文就以上问题来展开深入讨论，项目的code已经放在[github](https://github.com/PerkinJ/redux-saga-demo)上。

### 项目搭建
#### 1. 建立API
本文主要了解react方面的技术栈，所以对api相关的创建，这里不做介绍，后文统一使用
```
http://widgetizer.jcolemorrison.com
```
为接口，该API的作者会对注册的数据保存24小时。
>注意：如果您要运行本demo，可以使用.env环境的API,内容如下。一定要以REACT_APP
_*开头，因为create-react-app脚手架有对环境变量做限制。
```
REACT_APP_API_URL=http://widgetizer.jcolemorrison.com
```
#### 2. 建立脚手架
a) 确认安装了create-react-app 
```js
npm install -g create-react-app 
```

b) 创建目录，并进入
```bash 
mkdir app 
cd app
```
c) 运行
```js 
create-react-app .
```
d) 安装依赖
```js
 npm i redux react-redux redux-saga react-router redux-form --save
```
e) 启动项目
```js
npm start
```
f) 创建目录结构
```
src/  
  login/ # 不同的容器对应不同的路由
    sagas.js # 存储跟api调用有关的saga
    reducer.js # 管理所有跟容器有关的状态
    actions.js # 提供给容器层分发任务的动作(action)
    constants.js # 对reducers/actions存储常量
    index.js  # 容器型组件

  signup/
    sagas.js
    reducer.js
    actions.js
    constants.js
    index.js

  widgets/
    sagas.js
    reducer.js
    actions.js
    constants.js
    index.js

  client/
    reducer.js
    actions.js
    constants.js

  notifications/ # 消息与错误的通知
    Messages.js
    Errors.js

  lib/
    api-errors.js # 处理api调用错误
    check-auth.js # 当用户访问建立权限验证时应该被保护

    index-reducer.js #对reducer统一管理，方便compose
    index-sagas.js  #对saga统一管理
```

### 建立入口文件
---

#### 1. 建立路由结构

```javascript
//...
ReactDOM.render(  
  <Provider store={store}>
    <Router history={browserHistory}>
      <Route path="/" component={App} >
        <Route path="/login" component={Login} />
        <Route path="/signup" component={Signup} />
        <Route path="/widgets" component={Widgets} />
      </Route>
    </Router>
  </Provider>,
  document.getElementById('root'),
)

```
#### 2. 建立中间件

```javascript
// Import the index reducer and sagas
import IndexReducer from './index-reducer'  
import IndexSagas from './index-sagas
const sagaMiddleware = createSagaMiddleware()

```

#### 3. 引入Redux Devtools Chrome Extension
```javascript
const composeSetup = process.env.NODE_ENV !== 'production' && typeof window === 'object' &&  
  window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ ?
  window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ : compose
```

#### 4. 开始saga
```
const store = createStore(  
  IndexReducer,
  composeSetup(applyMiddleware(sagaMiddleware)), // 使用Redux Devtool 去跟踪saga
)

sagaMiddleware.run(IndexSagas)
```

### Redux Aside

---

当然，在这里redux有点老生常谈了，言简意赅一下：

a) 提供一个全局store，去存储应用的state，这里与内部的state不冲突，这里比如一个国家，有很多个省。

b) 提供reducers给tore。就好比是每个省的省长，当遇到一些要改变省内状态的事情时，他们来决定是否接受或拒绝。

c) 通过```<provider store={store}/>```传递，使得我们的应用能够拿到这个全局状态

d)app分发actions

e)对于reducers来说，如果action与他们有关，便会捕获，修正他们的状态(纯函数的方式)

f)当全局状态改变，被connected的app会接收这个改变，会再次渲染我们的react组件。
### 模块搭建

---
#### 1. The Client State
client模块主要是保存用户登录之后的token信息，此token为JWT(后文详细介绍)。


```javascript
// client存储的初始state
const initialState = {  
  id: null,
  token: null,
}
```
- 设置action
```javascript
import { CLIENT_SET, CLIENT_UNSET } from './constants'

export function setClient (token) {  
  return {
    type: CLIENT_SET,
    token,
  }
}

export function unsetClient () {  
  return {
    type: CLIENT_UNSET,
  }
}

```
- 设置reducer
```javascript
import { CLIENT_SET, CLIENT_UNSET } from './constants'

const initialSate = {  
  id: null,
  token: null,
}

const reducer = function clientReducer (state = initialSate, action) {  
  switch (action.type) {
    case CLIENT_SET:
      return {
        id: action.token.userId,
        token: action.token,
      }

    case CLIENT_UNSET:
      return {
        id: null,
        token: null,
      }

    default:
      return state
  }
}

export default reducer

```
#### 2. The Signup State
我们需要四种state来表示请求的过程，分别是
- requesting - 注册的请求
- successful - 请求返回成功
- errors - 请求返回失败
- messages - 展示用户信息的数组

接着，创建reducer
```javascript
import { SIGNUP_REQUESTING } from './constants'

const signupRequest = function signupRequest ({ email, password }) {  
  return {
    type: SIGNUP_REQUESTING,
    email,
    password,
  }
}

export default signupRequest  
```
为了保持我们的actions是纯(pure)的,我们将所有异步的操作交给saga来完成。（与thunk不同的是，在这里saga将ations抽成了promises chain来处理）


#### 3. The Signup View
在这之前，我们先花一点功夫介绍下redux-form。为什么要用redux-form呢？想象一下，如果用state来保存要提交的数据，用onChange来获取用户输入，然后改变state中相应数据荐，简直是梦魇一般。**如果使用Redux跟Redux-form,redux用来管理状态，redux-form来负责表单数据部分。**

a) redux-form主要有以下特点：
- formReducer (reducer) :表单的各种操作以 Redux action 的方式，通过此 reducer 来促使 Redux store 数据的变化。
- reduxForm() (HOC):属于react装饰器，此高阶组件用以整合 Redux action 绑定的用户交互与您的组件，并返回一个新的组件供以使用。
- <Field/>:一个Filed组件将用户输入与redux store相连接

b) 数据流:

![image](http://oum6ifofe.bkt.clouddn.com/image/reduxFormDiagram.png)
数据流大概是这个样子的:

1. 用户点击这个 ```<input/>``` 组件,
2. "Focus action" 被触发,
3. formReducer 更新了对应的状态,
4. 这个状态被传回 ```<input/>```

根据上面的数据流模型，我们搬出项目里的代码来分析：
第一步：Form reducer的添加
```javascript
//index-reducer.js
import { combineReducers } from 'redux'
import { reducer as form } from 'redux-form'

import signup from './signup/reducer'
 


const IndexReducer = combineReducers({
  signup,
})

export default IndexReducer
```
```js
// index.js
const store = createStore(
  IndexReducer,
  composeSetup(applyMiddleware(sagaMiddleware)), // allows redux devtools to watch sagas
)
```
store需要知道组件如何发送action,因此我们需要在store中注册 formReducer，这样可以服务于整个app中你定义的所有表单组件，因此只需要注册一次。

第二步:建立 Form component

为了使表单组件可以与store进行交互，我们需要使用高价函数 reduxForm() 来包裹组件。他可以在执行提交表单等操作的时候，以props的方式提供表单内的state。
```javascript
class Signup extends Component {
  static propTypes = {
    // ...
  }

  render () {
    const {
      handleSubmit,
      signup: {
        requesting,
        successful,
        messages,
        errors,
      },
    } = this.props

    return (
      <div className="signup">
        {/* Use the Submit handler with our own submit handler*/}
        <form className="widget-form" onSubmit={handleSubmit(this.submit)}>
          //...
        </form>
      </div>
    )
  }
}

const mapStateToProps = state => ({
  signup: state.signup,
})


const connected = connect(mapStateToProps, { signupRequest })(Signup)

const formed = reduxForm({
  form: 'signup',
})(connected)

// Export our well formed component!
export default formed
```
第三步：完善 Form <Field/> Components
<Field/> 组件可以连接所有input类型组件的数据到store中，基本用法如下:
```js
 <label htmlFor="email">Email</label>
 <Field
    name="email"
    type="text"
    id="email"
    className="email"
    label="Email"
    component="input"
 />
```
>它创建了一个text类型的```<input/>```组件，还提供了诸如 value onChange,onBlur等属性，用于跟踪和维护此组件的各种状态。

到第三步为止表单上的操作数据已经可以填充至store，并可以执行提交表单操作

第四步：增加sumbit
```js
  // Form fields `email` and `password` when the form is submitted
  // this will in turn call the action
  submit = (values) => {
    console.log(values)
    this.props.signupRequest(values)
  }
```
提交的数据以JSON对象的形式注入了此表单组件的 onSubmit方法里了，可以打印出来看

c) 表单value的生命周期:

![image](https://github.com/erikras/redux-form/raw/master/docs/valueLifecycle.png)

d) API

这里只介绍本文使用的api，详细的api请移步[官方文档](http://redux-form.com/6.8.0/docs/api/)

- reduxForm：可以通过配置一些参数创建表单修饰器，如配置表单验证，提交成功或者失败的回调，获取或者失去焦点的action发送，prop命名等。

    - 必要参数

    form : 用于命名您的表单，在store生成此命名的数据节点。

    - 可选参数

    onChange : 表单触发 onChange 事件后的回调。
    
    onSubmit : 表单提交配置，可以配置需要提交哪些参数，还有提交时触发的 dispatch等。
    
    onSubmitSuccess  & onSubmitFail :  提交表单成功和失败的回调。
    
    shouldValidate: 同步验证。
    
    shouldAsyncValidate: 异步验证。

- Field：所有需要与 store 数据连接的表单组件，都可以用 <Field/>。
    - name：可以是简单的字符串，如 userName、password，也可以是复杂的结构。
    - component：可以是一个组件、无状态组件或者DOM所支持的默认的标签(input、textarea、select)。
    - 其他所有属性会通过prop传递到元素生成器中。如 className

- reducer:表单的reducer用来安装Redux state 到表单中。
> 如果您使用 Immutablejs 来管理您的 Redux state，你必须这么从 redux-form/immutable 中导入 reducer 模块。
```javascript
import { combineReducers } from 'redux'
import { reducer as form } from 'redux-form'
 // Or with Immutablejs:
// import { reducer as form } from 'redux-form/immutable'


const IndexReducer = combineReducers({
  form,
})

export default IndexReducer
```
以上就是Redux-Form的基本介绍。

### 总结
1. 项目的基本搭建
2. redux的相关介绍
3. **Redux-Form的使用以及基本的原理**


由于篇幅原因，基于Redux-Form与Redux-Saga的权限验证应用的第一系列就到此结束。基于Redux-Form与Redux-Saga的权限验证应用的第二系列，我们将详细介绍Redux-Saga在该项目中的应用，以及相关的原理。

### 参考链接
---
[React and Redux Sagas Authentication App Tutorial](https://start.jcolemorrison.com/react-and-redux-sagas-authentication-app-tutorial/)

[浅谈redux-form在项目中的运用](https://juejin.im/entry/599174956fb9a03c5b04ce9e)

[React-Redux技术栈——之redux-form详解](http://react-china.org/t/react-redux-redux-form/13292)