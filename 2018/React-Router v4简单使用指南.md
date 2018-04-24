---
title: React-Router v4简单使用指南
category: 前端
cdn: 'header-on'
tags: [react,react-router]
---

本文适合学习React-Router的新手以及使用过并想深入react-router底层原理的读者阅读，如有错误，还望读者指正。
> React Router v4是目前流行的React package。 以前版本的React Router使用配置伪装成伪组件，可能难以理解。 随着v4版本的到来，一切都是纯组件。

## 1. Histroty
理解与使用react-router之前，需要读者理解history。history包提供了react-router的核心功能，能搞轻松地在客户端添加位置导航。
主要有三种类型:
1. browser
2. hash
3. memory 
```js
import {
  createBrowserHistory,
  createHashHistory,
  createMemoryHistory
} from 'history'
```

### Location
无论何种类型，最终都会产生一个具有相同属性跟方法的history对象。
其中最重要的属性是**Location**，该属性反映了当前所在的位置，包含了pathname,search,hash等属性。此外每个Location属性都有一个unique key,用来识别和存储特定于某个位置的的数据。state则表示与之相关的状态，这提供了一种附加数据来表示location在URL中是否存在的状态。如下所示：
```js
{
  pathname: '/here',
  search: '?key=value',
  hash: '#extra-information',
  state: { modal: true },
  key: 'abc123'
}
```
### Navigation
Navigation属性可以更改位置。
1. push
push方法能搞让用户跳转到新的location。当用户点击```<Link>```时，其实就是使用```history.push```。

2. replace
replace与push类似，但是它并不是跳转到新的location，而是取代当前的页面。比如```<Redirect>```。

3. 其他
还有其他三个比较相似的方法:goBack,goForward,go。

### observer pattern
History对象采用观察者模式去监听location的变化。每一个history对象都有一个listen方法。React路由器的路由组件将订阅history对象，以便它可以在位置变化时重新渲染。

```javascript
const youAreHere = document.getElementById('youAreHere')
history.listen(function(location) {
  youAreHere.textContent = location.pathname
})
```

每个histroy对象都有一个createHref方法，可以每获取一个location对象而输出一个URL。

### 使用场景
1. 在浏览器中

browser跟hash对象都能搞用在浏览器中，最大的不同在于他们如何从URL中创建location对象。browser使用完整的URL，而hash仅使用位于第一个散列符号之后的URL。
```js
// Given the following URL
url = 'http://www.example.com/this/is/the/path?key=value#hash'
// a browser history creates the location object:
{
  pathname: '/this/is/the/path',
  search: '?key=value',
  hash: '#hash'
}
// a hash history creates the location object:
{
  pathname: 'hash',
  search: '',
  hash: ''
}
```
2. hash history使用场景

一般而言，对于动态服务器，请求的文件实际上是不存在的，此时，服务器将检查请求的URL并决定要响应的HTML。而对于静态文件服务器来说，只能返回磁盘上的文件。为了解决静态文件服务器的限制，最简单的方案是在服务器上只有一个真实的location来获取HTML。这也就意味着应用程序只有一个URL，这样就破坏了history的目的。为了解决这个问题，hash history能够使用URL的hash部分来设置和读取位置。比如:

```js
// If example.com uses a static file server, these URLs would
// both fetch html from /my-site/index.html
http://www.example.com/my-site#/one
http://www.example.com/my-site#/two
// However, with a hash history, an application's location will be
// different for each URL because the location is derived from
// the hash section of the URL
{ pathname: '/one' }
{ pathname: '/two' }
```

虽然hash history运行良好，但缺点是它依赖于将所有路径信息存储在URL的哈希表中。 因此，只有在您的网站没有动态服务器来处理您的HTML时才应该考虑它。

3. Memory：获取所有的history对象

memory history可以在任何运行javascript的地方使用它。一个最简单的例子在于你可以通过node运行的单元测试中使用它。这使您可以测试依赖于history对象的代码，而无需实际使用浏览器进行测试。更重要的是，你可以使用memory history在移动端app中，react-router-native就是使用memory history的。memory history与browser跟hash最大的不同在于它保持了自己的内存阵列，在创建history记录时，用户可以传递信息来设置状态，这个状态可以是一个location数组或者在数组中表示当前页面的索引。而后者主要依赖于浏览器已经存储的位置数组。

```js
const history = createMemoryHistory({
  initialEntries: ['/', '/next', '/last'],
  initialIndex: 0
})
```

## 2. 渲染<Router>
路由组件只接受一个子元素，因此将```<App />```与路由分开是十分有用的。同时，对于服务端渲染来说，也是十分有用。

```js
import { BrowserRouter } from 'react-router-dom'
ReactDOM.render((
  <BrowserRouter>
    <App />
  </BrowserRouter>
), document.getElementById('root'))
```

这里的BrowserRouter与上文将的browser一致，也是三种形式。分别为<BrowserRouter>,<HashRouter>,<MemoryRouter>。

## 3. Routes

Route组件是React-Router的主要构建的模块。只要用户想根据位置的路径名呈现内容，都应该使用Route元素。

React Router使用[path-to-regexp](https://github.com/pillarjs/path-to-regexp/tree/v1.7.0)依赖包来确定路径元素的路径prop是否与当前位置匹配。它将路径字符串编译为一个正则表达式，该正则表达式将与该位置的路径名相匹配。
 - url:当前位置路径名的匹配部分
 - path:路由的path
 - isExact:path === pathname
 - params:包含path-to-regexp捕获的路径名的对象
 
 ### 创建路由

```<Route>```可以在路由器的任何位置进行创建，但只有将它们渲染在同一位置才有意义。用户可以使用```<Switch>```组件去给```<Route>```分组， ```<Switch>```将迭代其子元素（路由），并仅渲染与当前路径名匹配的第一个元素。
比如，我们想匹配如下规则：
 1. ```/``` -主页
 2. ```/roster```- 队员花名册
 3. ```/roster/:number```具体某一个队员
 4. ```/schedule```球队的比赛时间表
 
为了在应用程序中匹配一个路径，我们所要做的就是创建一个<Route>元素和我们想要匹配的路径。
```
<Switch>
  <Route exact path='/' component={Home}/>
  {/* both /roster and /roster/:number begin with /roster */}
  <Route path='/roster' component={Roster}/>
  <Route path='/schedule' component={Schedule}/>
</Switch>
```

### ```<Route>```的渲染
Routes有三个属性：
1. component：React组件，当具有组件Prop的路由匹配时，路由将返回一个新元素，即为React组件（本质上是使用React.createElement创建）。

2. render：一个返回React元素的函数。 它将在路径匹配时被调用。 这与组件类似，但对于内联渲染和将更多props给元素很有用。
> 这本质上就是一个无状态的功能组件。 component与render最大的不同在于component使用React.createElement创建该元素，而render则将该组件作为函数调用。 如果您要定义一个内联函数并将其传递给component 中，它将比render属性慢得多。

3. children：一个返回React元素的函数，与前两个不同的是，无论path的路径是否匹配当前的location，该函数都会被渲染。

```js
<Route path='/page' component={Page} />
const extraProps = { color: 'red' }
<Route path='/page' render={(props) => (
  <Page {...props} data={extraProps}/>
)}/>
<Route path='/page' children={(props) => (
  props.match
    ? <Page {...props}/>
    : <EmptyPage {...props}/>
)}/>
```

一般而言，我们都使用component或者render，children偶尔会使用到，但通常最好在路径不匹配时不渲染任何东西。如果没有额外的属性传递，就推荐使用component属性。

## 4. Links
我们的应用一般需要一种在页面间导航的方法，如果我们使用锚点元素创建链接会导致整个页面重新加载。React-Router提供了```<Link>```组件来防止这种事情发生。点击```<Link>```时，URL更新，内容变化，但页面不重新加载。

```js
import { Link } from 'react-router-dom'
const Header = () => (
  <header>
    <nav>
      <ul>
        <li><Link to='/'>Home</Link></li>
        <li><Link to='/roster'>Roster</Link></li>
        <li><Link to='/schedule'>Schedule</Link></li>
      </ul>
    </nav>
  </header>
)
```

```<Link>```使用to属性来描述我们想要导航的路径，可以是字符串也可以是location对象（包含pathname,search,hash,sstate等），当它为字符串时，它将被转换为location对象。



## 参考文献
[A little bit of history](https://medium.com/@pshrmn/a-little-bit-of-history-f245306f48dd)

[A Simple React Router v4 Tutorial](https://medium.com/@pshrmn/a-simple-react-router-v4-tutorial-7f23ff27adf)

[react router教程](https://reacttraining.com/react-router/web/guides/quick-start)