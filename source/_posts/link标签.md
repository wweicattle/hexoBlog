---
title: JavaScript-《link标签》
date: 2023-06-12 13:30:03
category:  
  - "JavaScript大全"
tags: JavaScript
---

## 一.发生了什么? 

打开控制面板,意外发现Elements,点击菜单时会往head标签中插入一个link标签. 这是用vite打包的Vue3代码,在项目中做到了按需加载组件

`我懂,每次点击一个导航栏菜单,如果没存在该link对应的js都会添加一个link标签`
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2c939f7199a461f95e692561bb1ce33~tplv-k3u1fbpfcp-watermark.image?)

很简单嘛,只是一个简单的预加载,用到该组件会往head标签中添加一个link标签进行预加载
  <link rel="prefetch" as="script" crossorigin href="./index.js">

查看network会发起请求
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/12c57eb0f3124a78abf07dea35c4760c~tplv-k3u1fbpfcp-watermark.image?)
## 二.思考第一个问题

`发现问题了,不是link发起请求了,那我是不是直接在其他scipt标签中就能使用该组件中的全局的变量了?`

平常你script标签引入第三方的资源时,如果往全局注入属性方法,那么在后面的script标签中都是可以直接使用的.

好奇心驱使我搭建了一个简单的模板进行测试一下.

```html
<html lang="en">
  <head>
    <link rel="preload" as="script" crossorigin href="./index.js" />
  </head>
  <body>
    <div id="content">this is test</div>
    <script>
      console.log(demoNum)
    </script>
  </body>
</html>
        
```

index.js中:

```js
const demoTitle = 'this is test!'
window.demoNum = 2323
export { demoTitle }
```


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df762c14ef7045baaa9bb602a1489b45~tplv-k3u1fbpfcp-watermark.image?)
显示无该变量undefined,今天一定要给它输出来,我哪里做写错了! ok检查一下

--------
看了半天,这只是一个***link发起请求,给你加载了,没有执行解析js文件啊***,何来变量!(注意:如果是一个css文件是有去加载的,打开network会有请求)

那么问题来了,vite打包后的对应的每一个组件的js,怎么执行,肯定要执行啊,组件按需加载对应的页面dom结构也会发生改变.
现在的问题是怎么加载解析该`index.js`文件呢

脑海里跳出ES6的import()按需加载与动态添加scipt标签引入

1. import()引入可以加载执行,ES6模块
2. 动态添加script加载,不合适吧,有export的导出

最终使用ES6的import()按需加载.不懂的同学自行查看阮一峰的
[Module 的语法 - ECMAScript 6入门 (ruanyifeng.com)](https://es6.ruanyifeng.com/#docs/module#import)


```js
     <script>
       import('./index.js').then(res => {
        console.log(res.demoTitle)
        console.log(demoNum)
      })
    </script>
```

查看结果:
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0db4705dadf141159d53a28dcfd286fb~tplv-k3u1fbpfcp-watermark.image?)
果真导出的属性方法,全局的变量都输出来了,确定是可以了

> 初步总结:如果该link标签上有rel="preload"（或rel="modulepreload"），则表示预加载请求，请求但不运行脚本。

## 三.思考第二个问题
`又有问题来了,那么vite打包过后,按需加载组件,那要怎么去加载该组件对应的js文件呢~
事实证明确实在Vite底层打包时也是这样加载对应的js文件。`

看下图打包的源码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa8fb65f1fe74b61accd1aa75d1eeb4c~tplv-k3u1fbpfcp-watermark.image?)

只是截了一小段图,我们只需要查看关键代码.从图中可以看到打包出的代码,按需加载组件js也是利用import()来实现.

## 四.思考第三个问题
`脑海又闪过一个异步组件的概念,我们知道在Vue3中知道“defineAsyncComponent“ 也是可以异步加载组件,并且在打包时,会单独打包成一个js文件.其原理又是怎样的呢!`


直接模板动手实践下,快速创建一个vite+Vue3模板,其在app.vue中:
```js
<script setup lang="ts">
import { defineAsyncComponent } from "vue"
const testCom = defineAsyncComponent(() => import("./components/testCom.vue"))
</script>
<template>
  <test-Com />
</template>
```
yarn build后,查看dist源码目录:
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/611cc08908b44842bbcbdd123cdb5e40~tplv-k3u1fbpfcp-watermark.image?)

我们可以看到确实是打包成单独的testCom.js文件

在主入口index.js看看是咋样引入的
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93a72fa9f83341c0a00a3351333ce7b9~tplv-k3u1fbpfcp-watermark.image?)

可以看到也是利用import()去加载的.

## 总结
> 所以总结Vite打包后,异步组件,单独打包成一个js文件,之后动态添加link标签预加载,当有需要时使用import()导入模块,运行解析

















