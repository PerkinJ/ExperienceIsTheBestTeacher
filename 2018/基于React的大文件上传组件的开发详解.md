---
title: 基于React的大文件上传组件的开发详解
category: 前端
tags: [react,FileReader,文件上传,md5]
---

> 以前实习的时候有做过大文件上传的需求，当时我们团队用的是网宿科技的存储服务，自然而然用的也是他们上传的js-sdk，不管是网宿科技还是七牛等提供存储服务的公司，他们的文件上传底层使用的基本上都是[plupload](http://www.plupload.com/)库。除了这个，百度FEX团队开源的[webuploader](http://fex.baidu.com/webuploader/)也是鼎鼎大名的，当然，对于文件操作的库有许多许多，本文不做过多介绍。
    
对于一个中小型企业的小项目或者个人项目来说，使用第三方的存储服务也许昂贵了点，且如果上传的文件涉及到隐私的话也是不安全的（各种方案都是因项目而异的）。本文主要讲解在不使用webuploader,plupload等库的情况下，使用html5的File API来解决大文件上传的问题(本文主要指前端部分)。当然，由于是对内的项目，本文并没有过多考虑浏览器兼容性的问题，毕竟对于IE低版本浏览器来说，Flash可能是最适合的。

本文主要使用了[antd](https://ant.design/docs/react/introduce-cn)为UI组件，搭建了如下系统。

### Demo演示

下图为文件预加载时的动图，考虑到gif时间的限制，拿了个30多M文件测试。

![image](http://oum6ifofe.bkt.clouddn.com/image/uploadPre.gif)


下图为上传中的过程

![image](http://oum6ifofe.bkt.clouddn.com/image/uploaded.gif)

### 前后端联调步骤
其实之所以不使用WebUploader等库来实现，也是因为后端的需求跟一般的大文件上传有一点不同，所以前端干脆不使用库来写。

前后端重点考虑的点，是使用**分片上传**，且每个分片都需要生成md5值，以便后端去校验。因此，每一次分片上传，都需要上传该片段的file,以及chunkMd5，和整个文件的fileMd5。同时，前后端采用arrayBuffer的blob格式来进行文件传输。

如下为前后端联调的步骤
#### 第一步：用户选择文件，进行预处理
- 1. 计算总文件的md5值,即fileMd5
- 2. 按照固定的分片大小（比如5M，该值为用户自定义），进行切分
- 3. 计算每个分片的md5值，chunkMd5,start,end,size等

#### 第二步：用户点击上传
- 1. 发送第一步生成的json数据到requestUrl
- 2. requestUrl接口返回响应，来验证该文件是否已经上传，或者已上传了哪些chunk。（返回的response应该包括每个chunk的状态，即pending or uploaded，第一次上传所有chunk状态都为pending）
- 3. 前端过滤掉已经上传的chunks后，对pending状态的chunks构成一个待上传队列进行上传。
- 4. 每一个chunk上传到partUpload接口，都应该包括，chunkMd5,start,end以及该分片的arrayBuffer数据。

#### 第三步：上传结果反馈
- 1. partUpload接口会返回该分片上传的基本情况，每一次上传成功，上传队列的个数即减一，这样也可以自定义上传的progress。
- 2. 当上传队列个数为0时，此时调用checkUrl，检查整个文件是否上传成功，与前端进行一个同步校验。

### 代码拆分

#### 总体架构
本文Demo主要是对UI组件进行描述，所以没有考虑数据层，读者可以自己配合dva或者redux。下文为主要的代码结构

```js
import React, { Component } from 'react'
import PropTypes from 'prop-types'

import { Upload, Icon, Button, Progress,Checkbox, Modal, Spin, Radio, message } from 'antd'

import request from 'superagent'
import SparkMD5 from 'spark-md5'

const confirm = Modal.confirm
const Dragger = Upload.Dragger

class FileUpload extends Component {
  constructor(props) {
    super(props)
    this.state = {
      preUploading:false,   //预处理
      chunksSize:0,   // 上传文件分块的总个数
      currentChunks:0,  // 当前上传的队列个数 当前还剩下多少个分片没上传
      uploadPercent:-1,  // 上传率
      preUploadPercent:-1, // 预处理率  
      uploadRequest:false, // 上传请求，即进行第一个过程中
      uploaded:false, // 表示文件是否上传成功
      uploading:false, // 上传中状态
    }
  }
  showConfirm = () => {
    const _this = this
    confirm({
      title: '是否提交上传?',
      content: '点击确认进行提交',
      onOk() {
        _this.preUpload()
      },
      onCancel() { },
    })
  }
  
 
  preUpload = ()=>{
   // requestUrl,返回可以上传的分片队列
   //...
  }
 
  handlePartUpload = (uploadList)=>{
   // 分片上传
   // ...
  }
  render() {
    const {preUploading,uploadPercent,preUploadPercent,uploadRequest,uploaded,uploading} = this.state
    const _this = this
    const uploadProp = {
      onRemove: (file) => {
      // ...
      },
      beforeUpload: (file) => {
        // ...对文件的预处理

      },
      fileList: this.state.fileList,
    }

    return (
      <div className="content-inner">
        <Spin tip={
              <div >
                <h3 style={{margin:'10px auto',color:'#1890ff'}}>文件预处理中...</h3>
                <Progress width={80} percent={preUploadPercent} type="circle" status="active" />
              </div>
              } 
              spinning={preUploading} 
              style={{ height: 350 }}>
          <div style={{ marginTop: 16, height: 250 }}>
            <Dragger {...uploadProp}>
              <p className="ant-upload-drag-icon">
                <Icon type="inbox" />
              </p>
              <p className="ant-upload-text">点击或者拖拽文件进行上传</p>
              <p className="ant-upload-hint">Support for a single or bulk upload. Strictly prohibit from uploading company data or other band files</p>
            </Dragger>
            {uploadPercent>=0&&!!uploading&&<div style={{marginTop:20,width:'95%'}}>
              <Progress percent={uploadPercent} status="active" />
              <h4>文件上传中，请勿关闭窗口</h4>
            </div>}
            {!!uploadRequest&&<h4 style={{color:'#1890ff'}}>上传请求中...</h4>}
            {!!uploaded&&<h4 style={{color:'#52c41a'}}>文件上传成功</h4>}
            <Button type="primary" onClick={this.showConfirm} disabled={!!(this.state.preUploadPercent <100)}>
                <Icon type="upload" />提交上传
             </Button>
          </div>
        </Spin>
      </div>
    )
  }
}

FileUpload.propTypes = {
  //...
}

export default FileUpload
```

#### 文件分片
> 使用Html5 的File API是现在主流的处理文件上传的方案。在使用FileReader API之前，应该了解一下[Blob](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob)对象，Blob对象表示不可变的类似文件对象的原始数据。File接口就是基于Blob，继承了blob的功能并将其扩展使其支持用户系统上的文件。

- 本文前后端约束采用二进制的ArrayBuffer 对象格式来传输文件,类型话数组(ArrayBuffer)可以直接操作内存，接口之间完全可以用二进制数据通信。

- 使用FileReader来读取文件，主要有5个方法：

| 方法名 | 参数 | 描述 |
|---|---|---|
| abort | none | 中断读取 | 
| readAsBinaryString | file | 将文件读取为二进制码 |
| readAsDataURL | file | 将文件读取为DataURL |
| readAsText | file,[encoding] | 将文件读取为文本 |
| readAsArrayBuffer | file | 将文件读取为ArrayBuffer | 
 
- 使用Antd的Drag(Uploader)组件，我们可以在props的beforeUpload属性中操作file，也可以通过onChange监听file。当然，使用beforeUpload更加方便。关键代码如下：

```js
const uploadProp = {
      onRemove: (file) => {
        this.setState(({ fileList }) => {
          const index = fileList.indexOf(file)
          const newFileList = fileList.slice()
          newFileList.splice(index, 1)
          return {
            fileList: newFileList,
          }
        })
      },
      beforeUpload: (file) => {
        // 首先清除一下各种上传的状态
        this.setState({
          uploaded:false,   // 上传成功
          uploading:false,  // 上传中
          uploadRequest:false   // 上传预处理
        })
        // 兼容性的处理
        let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice,
          chunkSize = 1024*1024*5,                             // 切片每次5M
          chunks = Math.ceil(file.size / chunkSize),
          currentChunk = 0, // 当前上传的chunk
          spark = new SparkMD5.ArrayBuffer(),
          // 对arrayBuffer数据进行md5加密，产生一个md5字符串
          chunkFileReader = new FileReader(),  // 用于计算出每个chunkMd5
          totalFileReader = new FileReader()  // 用于计算出总文件的fileMd5
          
        let params = {chunks: [], file: {}},   // 用于上传所有分片的md5信息
            arrayBufferData = []              // 用于存储每个chunk的arrayBuffer对象,用于分片上传使用
        params.file.fileName = file.name
        params.file.fileSize = file.size

        totalFileReader.readAsArrayBuffer(file)
        totalFileReader.onload = function(e){
            // 对整个totalFile生成md5
            spark.append(e.target.result)
            params.file.fileMd5 = spark.end() // 计算整个文件的fileMd5
          }

        chunkFileReader.onload = function (e) {
          // 对每一片分片进行md5加密
          spark.append(e.target.result)
          // 每一个分片需要包含的信息
          let obj = {
            chunk:currentChunk + 1,
            start:currentChunk * chunkSize, // 计算分片的起始位置
            end:((currentChunk * chunkSize + chunkSize) >= file.size) ? file.size : currentChunk * chunkSize + chunkSize, // 计算分片的结束位置
            chunkMd5:spark.end(),
            chunks
          }
          // 每一次分片onload,currentChunk都需要增加，以便来计算分片的次数
          currentChunk++;          
          params.chunks.push(obj)
          
          // 将每一块分片的arrayBuffer存储起来，用来partUpload
          let tmp = {
            chunk:obj.chunk,
            currentBuffer:e.target.result
          }
          arrayBufferData.push(tmp)
          
          if (currentChunk < chunks) {
            // 当前切片总数没有达到总数时
            loadNext()
            
            // 计算预处理进度
            _this.setState({
              preUploading:true,
              preUploadPercent:Number((currentChunk / chunks * 100).toFixed(2))
            })
          } else {
            //记录所有chunks的长度
            params.file.fileChunks = params.chunks.length  
            // 表示预处理结束，将上传的参数，arrayBuffer的数据存储起来
            _this.setState({
              preUploading:false,
              uploadParams:params,
              arrayBufferData,
              chunksSize:chunks,
              preUploadPercent:100              
            })
          }
        }

        fileReader.onerror = function () {
          console.warn('oops, something went wrong.');
        };
        
        function loadNext() {
          var start = currentChunk * chunkSize,
            end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;
          fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
        }

        loadNext()

        // 只允许一份文件上传
        this.setState({
          fileList: [file],
          file: file
        })
        return false
      },
      fileList: this.state.fileList,
    }

```
#### 分片上传
- 在预处理过程中会拿到uploadParams的json数据，如下所示
```js
{
    file:{
     fileChunks:119,
     fileMd5:"f5aeec69076483585f4f112223265c0c",
     fileName:"xxxx.test",
     fileSize:6205952600
    },
    chunks:[{
        chunk:1,
        chunkMd5:"8770f43dc59effdc8b995e4aacc8a26c",
        chunks:119,
        end:5242880,
        start:0
    },
    ...
    ]
}
```
将以上数据post到RequestUrl接口中，会得到如下json数据：
```js
{
    Chunks:[
        {
            chunk: 1, 
            chunkMd5:"8770f43dc59effdc8b995e4aacc8a26c", 
            fileMd5:"f5aeec69076483585f4f672223265c0c",
            end: 5242880,
            start:0,
            status:"pending"
        },
        …
    ],
    Code:200,
    FileMd5:"f5aeec69076483585f4f672223265c0c"
    MaxThreads:1,
    Message:"OK",
    Total:119,
    Uploaded:0
}

```
- 拿到json数据，会先对得到的Chunks进行一次过滤，将status为pengding的过滤出来。
```js
      let uploadList = res.body.Chunks.filter((value)=>{
        return value.status === 'Pending'
      })

      // 从返回结果中获取当前还有多少个分片没传
      let currentChunks = res.body.Total - res.body.Uploaded

      // 获得上传进度
      let uploadPercent = Number(((this.state.chunksSize - currentChunks) /this.state.chunksSize * 100).toFixed(2))      
      // 上传之前，先判断文件是否已经上传成功
      if(uploadPercent === 100){
        message.success('上传成功')
        this.setState({
          uploaded:true,    // 让进度条消失
          uploading:false
        })
      }else{
        this.setState({
          uploaded:false,
          uploading:true    
        })
      }

      this.setState({
        uploadRequest:false,    // 上传请求成功
        currentChunks,
        uploadPercent
      })
      //进行分片上传
      this.handlePartUpload(uploadList)

```
- 遍历uploadList的数据，分别将数据传入到uploadUrl接口中。**此过程最关键的，就是如何将分片的arrayBuffer数据如何添加到Blob对象中。**
为了减轻服务器的压力，这里可以采用分治的思想去处理每个分片。对于如何实现分治的思想，请参考本人之前写的博客[由requestAnimationFrame谈浏览器渲染优化](https://github.com/PerkinJ/ExperienceIsTheBestTeacher/blob/master/2017/%E7%94%B1requestAnimationFrame%E8%B0%88%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B8%B2%E6%9F%93%E4%BC%98%E5%8C%96.md)。

```js
handlePartUpload = (uploadList)=>{
    let blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
    const _this = this
    const batchSize = 4,    // 采用分治思想，每批上传的片数，越大越卡
          total = uploadList.length,   //获得分片的总数
          batchCount = total / batchSize    // 需要批量处理多少次
    let batchDone = 0     //已经完成的批处理个数
    doBatchAppend()
    function doBatchAppend() {
      if (batchDone < batchCount) {
          let list = uploadList.slice(batchSize*batchDone,batchSize*(batchDone+1))
          setTimeout(()=>silcePart(list),2000);
      }
    }
    
    function silcePart(list){
        batchDone += 1;
        doBatchAppend();
        list.forEach((value)=>{
          let {fileMd5,chunkMd5,chunk,start,end} = value
          let formData = new FormData(),
              blob = new Blob([_this.state.arrayBufferData[chunk-1].currentBuffer],{type: 'application/octet-stream'}),
              params = `fileMd5=${fileMd5}&chunkMd5=${chunkMd5}&chunk=${chunk}&start=${start}&end=${end}&chunks=${_this.state.arrayBufferData.length}`
                
          formData.append('chunk', blob, chunkMd5)
          request
            .post(`http://x.x.x.x/api/contest/upload_file_part?${params}`)
            .send(formData)
            .withCredentials()
            .end((err,res)=>{
              if(res.body.Code === 200){
                let currentChunks = this.state.currentChunks
                --currentChunks
                // 计算上传进度
                let uploadPercent = Number(((this.state.chunksSize - currentChunks) /this.state.chunksSize * 100).toFixed(2))
                this.setState({
                  currentChunks,  // 同步当前所需上传的chunks
                  uploadPercent,
                  uploading:true
                })
                if(currentChunks ===0){
                  // 调用验证api
                  this.checkUploadStatus(this.state.fileMd5)
                }
              }
            })
      })
    }
  }
```

### 总结与展望
以上就是一个简单的基于react的大文件上传组件，主要的知识点包括：**分片上传技术，FileReader API，ArrayBuffer数据结构，md5加密技术，Blob对象的应用**等  知识点。读者可以自行扩展该React组件，可以跟Dva/Redux结合扩展Model层或者集中的状态管理等。同时，对于该组件中出现的异步流程是很简单粗暴的，如何建立合理的异步流程控制，也是需要去思考的。当然，对于大文件来说，文件压缩也是一个需要去考虑的点，比如使用snappy.js等工具库。

---

### 参考文献

- [1. 踩坑Webuploader视频上传](https://juejin.im/post/5a263b79f265da431a430b4d)
- [2.Filereader API](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader)
- [3.ArrayBuffer：类型化数组](http://blog.csdn.net/lichwei1983/article/details/43893025)
- [文件和二进制数据的操作](http://javascript.ruanyifeng.com/htmlapi/file.html)