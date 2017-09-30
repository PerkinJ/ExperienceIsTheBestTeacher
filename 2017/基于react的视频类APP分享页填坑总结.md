> 随着内容型产品的逐渐增多，视频类的产品井喷式增长。腾讯系，阿里系，今日头条等视频产品越来越壮大，不少创业团队也纷纷加入，希望能在市场分一份羮。本人目前所在的创业团队也在负责视频类的app,作为前端新人的我，有幸搭建了分享页项目，现把所遇到的坑总结一下，欢迎共同讨论学习。

### 本项目是基于react搭建的，为什么要使用react?

主要考虑到以下几点:
1. 团队的主要技术栈是react,方便codeReview
2. 基于react的组件化开发，组件复用起来很方便，搭配ES6+函数式编程提高了开发速度
3. react的虚拟DOM特性，页面性能得到了极大的提升
4. 作为一个UI框架，项目打包之后，体积较小
5. npm下有太多的基于react的组件

本科上数字视频技术的时候，就听老师说，视频是个复杂的学科，包括现在的人脸识别，视频编解码，都离不开视频。甚至在客户端这样的应用层层面，视频的坑也依然有很多。

### 主要遇到的坑
1. 移动端适配的问题(本人项目还有PC版的，故需要考虑浏览器的兼容性)
2. h5的video标签在不同移动端浏览器(主要是iOS与Android)的不同表现
3. plyr.js在浏览器的表现(同事的chrome浏览器出现原始html5标签的样式在顶部)
4. 微信分享的认证(其他平台如QQ，微博也一样)
5. 页面无刷新跳转


#### 1. 移动端适配的问题
关于移动端适配问题，业界的主流的做法有
- 通过媒体查询的方式
- flex弹性布局
- rem+viewport缩放
- rem方式

>几种方法的比较请点击
- [移动端前端适配方案对比](http://www.jianshu.com/p/e5ca5b78e03e)
- [使用Flexible实现手淘H5页面的终端适配](https://www.w3cplus.com/mobile/lib-flexible-for-html5-layout.html)

由于本人项目有PC端，并且部分组件样式相同(主要还是自己太年轻，没有经验)。采用了media queries来做适配。
优点就是能够维护同一套代码，调整屏幕宽度的时候不用刷新页面即可响应式展开。但缺点也毋庸置疑，代码量比较大，维护不方便(目前项目的代码架构就不是很清晰，因为移动端跟PC端共用一套代码，后人来修改，容易造成错误)。

但不得不说，媒体查询做适配搭配react的组件化方案产生了不一样的化学反应。对于具体的代码实现，此处不做具体介绍了。


#### 2.h5的video标签在不同移动端浏览器(主要是iOS与Android)的不同表现
主要有以下问题：
- 1. 全屏播放
- 2. 封面图问题
- 3. 切换视频
- 4. 自定义控件

##### 全屏问题
对于Android手机不支持全屏、iOS只支持竖全屏。根据不同浏览器，例如UC浏览器是支持横竖全屏的，但是原生Safari则不支持。一般来说，禁止全屏是正常需求。

- 对于ios来说，利用webkit-playsinline这个好东西，只需要在video标签用上这个属性就能实现非全屏下播放视频。在react下使用
    ```js
    playsInline
    webkitPlaysInline
    ```
    加上这条属性之后，只是在ios微信上，能解决全屏问题，但safari不支持。这是需要引入[iphone-inline-video](https://github.com/bfred-it/iphone-inline-video)插件
    
    ```js
    const makeVideoPlayableInline = require('iphone-inline-video');
    
    var videoContainer = this.refs.videoContainer
    makeVideoPlayableInline(videoContainer)
    ```
    并且在上面github上，还讲了去掉video默认居中的播放按钮
    
    ```css
    video::-webkit-media-controls-start-playback-button {
      display: none;
    }
    .IIV::-webkit-media-controls-play-button,
    .IIV::-webkit-media-controls-start-playback-button {
      opacity: 0;
      pointer-events: none;
      width: 5px;
    }
    ```
    
    
    禁止掉默认的样式，就能自定义播放按钮，以及播放动画，加载动画等

- 对于android来说，android在微信/QQ浏览器(X5内核)中是不支持禁止全屏播放的（腾讯产品的策略，[腾讯浏览器论坛](http://bbs.mb.qq.com/thread-286111-1-1.html)早就炸开了锅），所以目前还没有解决方案。网上面讲的其他禁止全屏的案例，此全屏非彼全屏，x5-video-player-type="h5"属性，腾讯x5内核系的android微信和手Q内置浏览器用的浏览器webview的内核，使用这个属性在微信中视频会有不同的表现，会呈现全屏状态，貌似播放控件剥去了，至少加了这个属性后视频上层可以有其他dom元素出现了（非腾讯白名单机制的一种处理措施）。

##### 封面图问题
因为poster在android兼容的并不好，多数机型是不显示视频画面的，要不就是显示一个黑色的。在这里，做的处理就是先用一个img代替video，将视频设为width:1px;并且隐藏
```html
{/*用于解决安卓下poster问题，用图片来代替*/}
<div style={this.state.isPC?style.hide:this.state.imgToVideo?style.hide:style.show} className="video-tmp" ref="videoTmp">
        <img className="img-tmp" style={videoInfo.height?{height:videoInfo.height/videoInfo.width*this.state.screenWidth}:{}} src={this.props.videoInfo.coverUrl}/>
</div>
```
对于video
```js
const style = {
    showVideo:{width:'100%',position:'relative'},
    hideVideo:{width:'1px',position:'absolute',opacity:0},
}

<video
    className="video-UGC-plyr"
    style={this.state.isPC?style.showVideo:this.state.imgToVideo?style.showVideo:style.hideVideo}
    ref="videoContainer"
    src={this.props.videoInfo.url}        
    poster={this.props.videoInfo.coverUrl}
    controls
    loop
    preload="auto"
    playsInline 
    height={this.props.videoInfo.height/this.props.videoInfo.width*476.5}>
```
此代码可理解为点击封面图，出现this.state.imgToVideo变为true,从而使video出现，并且播放
```js
var videoTmp = this.refs.videoTmp
videoTmp.addEventListener('click',()=>{
            this.setState({pause:false,loading:true})
            this.setState({
                imgToVideo:true,
            })
            this.refs.videoContainer.play()
        },false)
```

##### 切换视频
在内容列表区域，会有很多推荐的相关视频或者热门视频。本项目切换视频采用前端路由策略，使用react-router的hash路由。

1. 首先在react的componentDidMount钩子函数中，通过react的location属性获取url的videoId，然后根据获得的videoId，异步请求后台数据，获得该video的数据，从而渲染整个页面。若url无videoId,则设置一个默认videoId

```js
//加载数据
 var videoId = this.props.location.query.videoId
    if(videoId === '' || videoId == null){
     //非线上环境 videoId不同
    videoId ='129b93d3f3029062d223c35e58a337ec'
    }
  this.props.getVideoData(videoId);     
```
2. 点击每个推荐视频的时候，获得videoId,再调用getVideoData(),从而渲染数据，由于页面使用的是react的hash路由，采用的是Link

```js
<Link to={"/?videoId="+videoId}>
    <div className="video-item" id={"videoImg-"+index} onClick={onClick}>
        <div className="img-box-player"></div>
          <div className="img-box">
            <img src={poster} className="video-img"
                    style={width>height?{height:'100%',width:'auto'}:{width:'100%',height:'auto'}}
                />
          </div>
        </div>
    </Link>
```
这样每次点击一次，就会改变url,然后根据url的videoID值，重新渲染数据。

3. 
```html
<body ontouchstart="" onmouseover=""></body>
```
在body上加上这两条属性，可以去掉ios上点击a标签出现active的问题

...未完待续