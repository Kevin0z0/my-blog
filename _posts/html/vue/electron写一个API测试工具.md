---
title: Electron写一个API测试工具
date: 2022-07-03 21:54:20
tags: 
  - electron
  - vue
  - nodejs
  - API
categories:
  - 编程
img: /images/article-banner/QQ截图20220703220722.jpg
typora-root-url: ..\..\..\
---

# ![主页面](/images/1ca67d7744cf4d5781191fe655fb0383tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp)

## 简介

这个项目是作为我的毕业设计来写的，粗略地实现了HTTP/HTTPS请求的功能，包括请求体、请求头、Get参数、Auth认证、CookieJar、代理、重定向、超时、自定义请求方式等，请求的模块由nodejs中的http、https模块封装，相当于自己封装了一个异步的request，其中除了markdown和代码编辑器，其余组件都是自己封装的，没用任何组件库。工程量还是蛮大的，期间也学到了不少东西。

项目地址：[Requester](https://github.com/Kevin0z0/Requester)

## 安装 && 运行

目前软件仅支持在Windows端运行，Mac m1 win11虚拟机中也可运行

```bash
npm install 
# 运行开发版
npm run dev
# 打包
npm run build
npm install`时发生错误可以尝试使用`npm install --force
```

运行时发生错误可以运行`.\install.ps1`安装对应版本的electron。

## HTTP请求模块开发

HTTP模块用Promise封装，使用异步的方式完成请求，为了省事直接照着Postman写了以下的功能

下图是目前开发的进度，蓝色为未完成的模块，其余功能均已实现 ![HTTP.png](/images/65d24e712d7d440cbdad231ab93616b9tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp)

### 基本流程

**由以下代码可以看出，请求流程大概为：**

进入重定向循环

​      ∨

 设置请求方法

​      ∨

  设置请求体

​      ∨

   设置认证

​      ∨

  设置请求头

​      ∨

  设置请求路径

​      ∨

  设置超时时间

​      ∨

   设置代理

​      ∨

​     发送

​      ∨

根据响应头判断是否重定向

```javascript
async _redirect(url, method, config){
   const isRedirect = _isRedirect(config.redirect)
   this.cookieJar = new CookieJar()
   let status = await this._handler(url, method, config)
   if(isRedirect) {
      let max = Math.max(0,this.config.redirect?.maxRedirect || 20)
      while(max--){
         this.cookieJar.setCookies(status?.headers["set-cookie"], status.raw.req.host)
         const code = status.status_code
         if(code > 300 && code < 400){
               ...
               status = await this._handler(
                  this.url,
                  this.config.redirect?.followMethod ? method : "GET",
                  this.config
               )
               ...
            }
         }else{
            break
         }
      }
      if(max === 0) {
         return new Promise((_,rej)=>{
            rej({
               data: null,
               timings: null,
               error: 'Exceeded maxRedirects. Probably stuck in a redirect loop ' + url
            })
         })
      }
   }
   decode(status)
   ...
}

_handler(url, method, config){
   ...
   this.url = url
   this.config = config
   this.info = this._urlParser()
   this.method = method ? method.toUpperCase() : 'GET'
   return this._setBody(this.config.type, this.config.data)
      .then(data => this._setAuth(data))
      .then(data => {
         this._setHeader()
         ...
         this.options = {
            ...
         }
         this._setPath()
         this._setTimeout()
         this._setProxy()
         return this._send(data)
      })
}
```

### 读取Windows系统代理

直接用命令行读取注册表，将字符串分割即可，再将提取出来的代理替换成正则可以解析的方式

```javascript
const { process, require } = window
const cp = require('child_process');

function win(done, failed){
   try{
      cp.exec('REG QUERY "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings"',(error, stdout, stderr)=>{
         if(error || stderr){
            cb({error: error || stderr})
         }
         const value = {}
         stdout.split('\r\n').forEach(v=>{
            const str = v.trim()
            for(let i of ['ProxyEnable', 'ProxyOverride', 'ProxyServer']) {
               if (str.startsWith(i)) {
                  const arr = str.split('    ')
                  value[arr[0]] = arr[2]
                  return
               }
            }
         })
         if(value['ProxyEnable'] === '0x0') return done(null)
         value['ProxyOverride'] = value['ProxyOverride'] //替换成正则可以解析的字符串
            .replace(/./g,"\.")
            .replace(/*/g,".*")
            .split(',').map(v=>v.trim())
         done(value)
      })
   }catch (e){
      failed(null)
   }
}
```

### Authorization 之 Digest Auth

这个模块是整个项目中用时最长的一个模块，为了实现`Auth-int`功能，我找了不少资料也翻了不少源码，也算粗略地实现了一遍。

#### 什么是Auth-int

![参考公式](/images/f15a7125f947414d87336ca0b243fe4dtplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp) 由维基百科所述，大体的意思就是为了传输更加安全，RFC2617增加了一个EntityBody哈希值校验的功能。

EntityBody我也找了不少资料，包括[rfc2616](https://www.w3.org/Protocols/rfc2616/rfc2616-sec7.html#sec7)中的描述，没怎么看懂，后来翻了Postman的源码，基本可以确定就是请求体了。图中的MD5(entityBody)也就是计算请求体的MD5。

#### 在请求时计算哈希值

在请求过程中，可能会在请求体中发送文件，此时无论用FormData还是ReadStream都会造成一个问题：先读取文件计算文件的哈希值，到第二次要发送数据时就读不出来了，所以在读取前需要先创建一个克隆的文件流。

**实现克隆文件流**

```javascript
const { createReadStream, ReadStream } = require("fs");
import {EventEmitter} from 'events'

//EventEmitter用实现一个事件监听对象，读取文件流
class CloneStream extends EventEmitter{
	writable = true
	write(chunk){
		this.emit('data',chunk)
	}
	end(){
		this.emit('end')
	}
}

//根据文件路径再次读取
class CreateReadStream extends ReadStream{
	constructor(path) {
		super(path);
	}
	getClonedStream(callback){
		const stream = new CloneStream()
		callback(stream)
		createReadStream(this.path).pipe(stream)
	}
}
```

**实现克隆FormData**

```javascript
class FormData extends Form{
   _clonedStream = []
   append(field, value, options) { //重写append
      ... //Form原本的代码，保留原有功能
      append(header);
      append(value);
      append(footer);
      this._clonedStream.push({ //把存入原来对象的值复制一份到新的数组
         key: field,
         value: value
      })
      ...
   }

   getClonedStream(callback){
      const form = new FormData() //新建一个FormData对象
      form.setBoundary(this.getBoundary())
      for(let i = 0; i < this._clonedStream.length; i++){ //遍历所有的值，如果是文件流，则使用文件路径重新读取一份
         const temp = this._clonedStream[i]
         let stream = temp.value
         if(stream instanceof ReadStream) {
            stream = createReadStream(temp.value.path)
         }
         form.append(temp.key, stream)
      }
      const cloneStream = new CloneStream()
      callback(cloneStream)
      form.pipe(cloneStream)
   }
   setBoundary(boundary){
      this["_boundary"] = boundary
   }
}
```

#### 实现完整加密流程

完整的代码在[这里](https://github.com/Kevin0z0/Requester/blob/main/src/utils/requester/authorization/digest.js)，参照Postman中的auth，实现了其中所有的哈希计算方式。

## 功能开发

### 实现拖拽树状列表

![拖拽动画演示](/images/034e8bc2b60441f8944466e3465a8b4dtplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp)

首先在最外面包一层div，主要解决溢出生成滚动条的问题，tree-item是真正的核心组件，在这里封装一层也可以更好地实现递归生成。

```html
<!-- Tree -->
<template>
    <div class="tree">
        <ul class="tree__list">
            <tree-item v-for="(i, index) in item"
                       :key="i.cid"
                       :father="item"
                       :index="index"
                       :item="i"
                       :empty="empty"/>
        </ul>
    </div>
</template>
<!-- Tree-Item -->
<template>
    <li class="t-item"
        :itemid="item.id"
        :folder="item.isFolder">
        <div class="t-item__title"
             :class="{'t-item__title_bg': symbol === shared.symbol}"
             draggable="true"
             @drop="drop"
             @dragend="dragEnd"
             @dragover="dragOver"
             @dragstart="dragStart"
             @dragleave="dragLeave"
             @contextmenu.prevent="contextMenu"
             :itemid="item.id"
             :fatherid="item.father_id"
             :folder="item.isFolder"
             ref="item"
        >
            <div class="t-item__arrow" @click="toggle"
                 :class="{'t-item__prevent': shared.status}">
                <span class="iconfont icon-chevron_forward"
                      :class="{'t-item__expand': item.expand}"
                      v-if="item.isFolder"
                ></span>
            </div>
            <p class="t-item__name" @click="expand"
               :class="{'t-item__prevent': shared.status}">
                <span class="iconfont icon-folder"
                      v-if="item.isFolder && item.father_id"></span>
                <editable-title :modelValue="item.name || item.url || itemPlaceholder"
                                @update:modelValue="setName"
                                :icon="false"
                                size="small"
                                ref="title"
                                @onBlur="saveName"/>
            </p>
        </div>
        <!--递归生成更下一级树状列表-->
        <ul v-show="item.expand" v-if="item.isFolder">
            <tree-item v-for="(child, index) in itemChildren"
                       :item="child"
                       :father="item"
                       :index="index"
                       :key="index"
                       :empty="empty"/>
            <li v-if="!itemChildren?.length" class="t-item__hint">
               {{ empty }}
            </li>
        </ul>
    </li>
</template>
```

拖拽功能用drop和drag实现，在拖拽的同时修改当前鼠标下item的样式，给用户视觉上的反馈

```javascript
export default {
    ...
    methods:{
        ...
        dragStart(e) {
            this.shared.status = true
            shared.element = e.target
            shared.element.style.opacity = "0.5"
        },
        dragOver(e) {
            pauseEvent(e)  //防止冒泡
            const target = e.target
            if(this.preventMove(target)) return
            target.style = "border: 2px solid var(--primary-color);"
        },
        dragLeave(e){
            const target = e.target
            if(target === shared.element) return
            target.style = ""
        },
        dragEnd() {
            this.shared.status = false
            shared.element.style.opacity = ""
            this.shared.element = null
        },
        drop(e){
            pauseEvent(e)
            const target = e.target
            target.style = ""
            //判断是否已存在于目标列表中，或移动的是最上级的列表
            if(this.preventMove(target) || target.getAttribute('itemid') === shared.element.getAttribute('fatherid')) return
            this.onDrop(target, shared.element)
        },
        ...
    }
    ...
}
```

### 本地数据持久化

数据库用来存储请求的数据和用户信息，为了和服务器端MySQL相匹配，我选择了better-sqlite3作为本地数据库，而且在存取性能上也有保证，通过预编译sql语句，尽量保证不被注入

```javascript
import {isDev} from "@/utils/functions";
const {require} = window
const {ipcRenderer} = require('electron')
const sqlite = require('better-sqlite3')
import path from 'path'

class DB{
   constructor() {
      this._base = ipcRenderer.invoke('read-user-data').then(v=>{ // IPC通信获取%appdata%路径
         if(isDev) return sqlite(
            path.join(v, "requester.sqlite"),
            { verbose: console.log }
         );
         return sqlite(path.join(v, "requester.sqlite"))
      })
   }
   get(statement, args){
      return this.prepare(statement).then(v=>{
         if(args) return v.get(args)
         return v.get()
      })
   }
   all(statement, args){
      return this.prepare(statement).then(v=>{
         if(args) return v.all(args)
         return v.all()
      })
   }
   ...
}
```

### VM虚拟机实现setInterval和setTimeout

VM虚拟机用在预执行脚本和测试脚本的功能中。

![测试脚本](/images/1d14f942e06c47d7a00c87dad0283872tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp) 考虑到安全性的问题（~~整个项目就没有很安全~~），我选择了vm2作为js虚拟机，由于虚拟机本身貌似是同步模式，所以只想到了封装一个Promise来实现（试过新建一个进程，但是node和webview之间进程间通信好像不好建立)不知道有没有更简单的方式。实现的原理就是重新封装setInterval和setTimeout，并设置一个EventEmitter监听调用，每调用一次计数器就加1，执行结束或清除定时则计数器减一，并检查是否还有正在执行的定时器，如果没有则向EventEmitter提交finish。

以下为实现setTimeout的代码，setInterval也同理

```javascript
let $_event_$ = new EventEmitter()
let $_count_$ = 0;
let o = setTimeout;

const $_setTimeout_$ = new Set();

function checkEvent() {
    $_count_$--;
    if (!$_count_$) {
        $_event_$.emit("finish");
    }
};
setTimeout = function (callback, time) {
    $_count_$++; //计数器+1
    const handler = o(function () {//执行原来的setTimeout
        callback();
        checkEvent(); //在执行完之后检查计数器状态
    }, time)
    $_setTimeout_$.add(handler); //将还未执行的计数器存入Set，在强制退出时可清除环境
    return handler;
};
let x = clearTimeout;
clearTimeout = function (handler) {
    x(handler);
    checkEvent();
    $_setTimeout_$.delete(handler);
};
module.exports = function(){
return new Promise((resolve, reject) => {
   try{
      "<your code>"
   }catch(e){
      error.error = e.toString()
      $_event_$.removeAllListeners()
      reject()
   }
   if($_count_$ === 0) { //当用户代码执行完毕，则直接返回resolve
      $_event_$.removeAllListeners()
      return resolve()
   }
   $_event_$.once("finish", ()=>{ //当定时器执行完毕，也返回resolve
      $_event_$.removeAllListeners()
      resolve()
   })
   $_event_$.once("abort", ()=>{ //当用户手动退出时，清除环境，同时退出Promise
      try{
         error.error = "abort"
         const timeout = $_setTimeout_$.entries()
         const interval = $_setInterval_$.entries()
         let timeoutNum = timeout.next()
         while(!timeoutNum.done){
            clearTimeout(timeoutNum.value[0]) //强制清除setTimeout
            timeoutNum = timeout.next()
         }
         let intervalNum = interval.next()
         while(!intervalNum.done){
            clearTimeout(intervalNum.value[0])  //强制清除setInterval
            intervalNum = interval.next()
         }
      }catch(e){
         error.error = e.toString()
      }
      $_event_$.removeAllListeners() 
      reject()
   })
})
}()
```

**正常执行** ![setTimeout](/images/c29916de8895428a9d79caed274c52betplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp)

**强制退出**

![强制退出](/images/9b85f84de0ac4c26a7c6f6cceb6532aftplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp)

## 插件开发

![插件](/images/90f908b2792f4006b88b4515b1c0f8e0tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.webp) 简单地开发了一个右键插件，实现了字符串替换的功能

**vuex初始化所有插件内容**

```javascript
import {isDev} from "@/utils/functions";

const {require} = window
const path = require('path')
const {ipcRenderer} = require('electron')

export default {
    namespaced: true,
    state: {
        textMenu: []
    },
    mutations: {
        init(state) {
            ipcRenderer.invoke('read-user-data').then(v => {
                const BASE_PATH = isDev ? path.join(__dirname, "../../../../../../plugins/") : path.join(v, 'plugins/') 
                //从plugins文件夹的package.json读出配置
                const pluginPackage = require(BASE_PATH + "package.json")
                const plugins = pluginPackage.plugins
                for (const i in plugins) { 
                    if (plugins[i].target === 'TextMenu') {
                        const pluginPath = BASE_PATH + plugins[i].path
                        state.textMenu.push({
                            name: i,
                            info: plugins[i],
                            main: require(pluginPath + require(pluginPath + "package.json").main)
                        })
                    }
                }
            })
        }
    }
}
```

**在加载右键菜单时调用的函数**

```javascript
import plugins from "@/store/global/plugins"; //保存了在启动软件时加载的插件信息

export function getTextMenuFunction(){
   return plugins.state.textMenu.reduce((prev, current) =>  //获取菜单项对应的函数
      prev[current.name] = current.main
      return prev
   }, {})
}

export function getTextMenuInfo(){
   return plugins.state.textMenu.reduce((prev, current) => { //获取菜单信息
      prev.push(current.info.menu)
      return prev
   }, [{type: 'separator'}])
}
```

**md5插件**

```javascript
const crypto = require("crypto")
function md5(){
   //在调用md5函数时call了一个当前选中的区域，所以用this
   this.selection.replace(crypto.createHash('md5').update(this.selection.text).digest('hex'))
}

module.exports = {
   md5
}
```

### electron打包

在vue.config.js中简单配置以下

```javascript
module.exports = defineConfig({
  ...
  pluginOptions: {
    electronBuilder: {
      customFileProtocol: 'requester://./', //自定义协议，没什么卵用
      externals: [ //这些不能被webpack打包，会报错
        "fs",
        "form-data",
        "better-sqlite3",
        "vm2",
        "jsonwebtoken"
      ],
      builderOptions: {
        appId: "com.Requester.0.1.0",
        productName: "Requester",
        copyright: "Copyright © 2022 Kevin0z0",
        win: { //暂时就打包了个windows平台
          icon: "build/icon.ico",
          target: [{
            target: "nsis",
            arch: [
              "x64"
            ]
          }],
          extraResources: ["plugins/**", "requester.sqlite"] 不打包这两个文件/文件夹
        },
        nsis: {
          oneClick: false,
          allowToChangeInstallationDirectory: true,
          installerIcon: "build/icon.ico",
          installerHeaderIcon: "build/icon.ico",
          deleteAppDataOnUninstall: true
        },
      }
    },
  }
})
```

## 开发时遇到的问题

### 升级到WebPack5之后fs报错问题

碰到了以下的报错

```bash
BREAKING CHANGE: webpack < 5 used to include polyfills for node.js core modules by default.
This is no longer the case. Verify if you need this module and configure a polyfill for it.

If you want to include a polyfill, you need to:
        - add a fallback 'resolve.fallback: { "stream": require.resolve("stream-browserify") }'
        - install 'stream-browserify'
If you don't want to include a polyfill, you can use an empty module like this:
        resolve.fallback: { "stream": false }
```

#### 解决方案

安装`node-polyfill-webpack-plugin`

```bash
npm install node-polyfill-webpack-plugin --save-dev
```

在`vue.config.js`中添加如下代码

```javascript
const NodePolyfillPlugin = require("node-polyfill-webpack-plugin")
module.exports = defineConfig({
  ...
  configureWebpack: {
    plugins: [
      new NodePolyfillPlugin()
    ]
  }
  ...
})
```

### Vue3引用monaco-editor时报错

众所周知，Vue在data中的对象都会转换成Proxy对象，在转换过程中可能修改了原来对象的一些属性，导致monaco不能正常使用，翻了许久文档找到了`markraw`函数

```js
import * as monaco from 'monaco-editor/esm/vs/editor/editor.main.js';

data(){
    editor: null
},
mounted(){
    this.editor = markRaw(monaco.editor.create(this.textarea,{
        ...
    }))
}
```

其实不在data里面预先设置属性，直接把editor挂到this上也可以，但没有上面一种写的明确

```js
data(){
    //editor: null
},
mounted(){
    this.editor = monaco.editor.create(this.textarea,{
        ...
    })
}
```

## 写在最后

留个坑，想起来了再写