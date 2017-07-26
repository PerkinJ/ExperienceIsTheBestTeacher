使用react开发已经很久了，但之前很少去关注代码底层的东西，学得很肤浅。最近在深入了解react&redux技术栈，及相关源码。结合自己所学和所用，总结一些自己的看法，以便今后的学习。
> react-redux为我们提供了React与Redux之间的绑定，其中Provider是整个应用最外层的React组件，它接受一个store作为props。

其实provider源码的实现也非常简单，参考源码，我也自己实现了一个简易版的provider
首先为了包裹内层的组件，定义以下：
```js
export default class Provider extends Component{
    render () {
        return this.props.children
    }
}
```
然后加上getChildContext
```js
export default class Provider extends Component{
    constructor(props,context){
        super(props,context)
        
    }
    getChildContext(){
        return { store: this.props.store }
    }
    render () {
        return this.props.children
    }
}
Provider.childContextTypes = {
    store: React.PropsTypes.object
}
```
以上实现非常简单，在constructor中，拿到props中的store，并挂载在当前实例上。同时定义了getCHildContext方法，该方法定义了自动沿组件传递的特殊props。

> 每当使用getChildContext时，必须定义childContextTypes，这样才能是getChildContext起效果。如果没定义contextTypes，context将会是空对

下面重点再总结下context,getChildContext。
- [x] context:React的context和全局变量相似，场景包括：传递登录信息、当前语音以及主题信息。context可以跨级传递数据，但是会降低组件的复用性，应避免使用。
- [x] getChildContext:通过getChildContext方法定义了自动沿组件传递的特殊props。

以上就是简易实现createStore的内容，本文主要借鉴redux作者的demo例子以及《深入React技术栈》里面的相关分析。