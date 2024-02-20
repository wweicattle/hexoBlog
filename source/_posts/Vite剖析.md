---
title: Vite剖析
date: 2024-01-08 16:20:01
category:  
  - "工程化"
tags: 工程化
---

> 前言: 小弟不才,大胆猜测 以后前端项目构建工具**Vite**是未来.所以多学学它的核心底层思想.这一点也不过分.

先说说早期:

*   构建大型的项目,数千个模块非常普遍,开发时构建工具,也会受到**性能瓶颈**
*   开发过程时,启动项目需要很长时间,HMR也需要等待几秒钟才可以,如此反复极大影响开发者的**开发效率**

**Vite的出现**,实现开发阶段无打包或者说是让浏览器去执行,多亏EsModule模块规范

*   每个特有的模块都会发起一个请求,内部起一个服务,针对不同的模块做不同的处理.
*   当然这也会造成请求过多,当然Vite内部实现了**预构建**,将一些依赖中的子依赖,最终只需请求一次即可.
*   使用**ESbuild**进行预构建处理,并且在内部中 也存在一些ts jsx等类型,也会使用**ESbuild**进行转换
*   提供更快的**HMR**,允许一个模块进行替换,不会影响页面其他部分,
*   当然最终打包还是使用的是**Rollup** 更加灵活,生态更加成熟,兼容生态,毕竟Rollup的npm包下载量过亿,生态也经营7年左右
### 接下来我们重点围绕3部分,来让我们更懂**Vite**这个东西

### **1.开发阶段:**

首先我们简单创建一个Vite项目

执行 :
`yarn create Vite`快速建立一个Vite+vue3模版

` yarn install` 后 `yarn serve`

启动项目:

> Vite内部帮我们起了一个服务,我们大概来看下具体做了哪些事情.
>
> 我们发现启动项目之后非常快,项目就启动完毕了
> 我们截图看下项目的network! ! !

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15554b35928c40b9a75020b0ce0be347~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1538\&h=668\&s=210601\&e=png\&b=fefefe)

**非常多的请求!** 🤫 🤫 🤫

果然把项目中的模块都转换成ESM模块,这样每次每个模块请求都会经过服务器拦截.然后经过不同的中间件处理

**那你可能会有一个疑问,有些第三方库根本不是符合ESM的模块啊,那你咋办?**

> 答案:当然尤大也知道,那肯定需要做一层转换啊,将UMD模块,CMD模块转换成统一的ESM模块,在**预构建中**处理

**这么多请求,这对带宽的压力也有点大啊?**

> 答案:当然尤大也知道,那就是在一个库中,如果存在多个引入模块,都处理成一个模块中,这样只需请求一次啦!也是在**预构建中**处理

okok我们知道,当你需要什么模块,就去请求,此时返回给浏览器,完全不需要开发打包,实现 **no bundle** 概念,所以这样就快多了🐮🐮

注意⚠️ (此时的ts,vue等每个请求都以 application/javascript处理,都统一以js返回形式处理)**浏览器只能识别js**

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/74a65284b5164dee8fb78e1df8666d26~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=792\&h=468\&s=100132\&e=png\&b=fcfcfc)

**Vite内部,使用connnet包创建一个服务,以中间件的形式处理不同的请求**

![vite setve.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/824adfa865404a8c89892ab4aae222a5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=318\&h=361\&s=20911\&e=png\&a=1\&b=fefefe)

从入口文件 index.html开始,当处理index.html时,发现内部有其他的资源的请求!

比如发现`main.ts`
此时请求main.ts,会有专门的中间件处理,并返回给浏览器执行

返回内容:

![carbon (5).png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db8b67f4fdf74b88b23207338e0940ba~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1464\&h=558\&s=130324\&e=png\&b=151718)

我们发现引入的Vue模块,路径会被内部解析到.vite下.(***这是访问预构建的路径,我们后面了解一下***)
此时会去请求`Vue`,`style.css`,`App.vue`资源

此时运用到中间件,不同的资源,不同的中间件处理

我们重点看下遇到**vue组件**是怎么处理的呢?

首先我们应该知道,在创建一个新的模版中,会有一个vite.config.ts来对项目的开发与构建中做配置.

我们可以发现有一个`@vitejs/plugin-vue`插件

![carbon (6).png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d93995a015040a5930faf9363104487~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=908\&h=670\&s=126015\&e=png\&b=161819)
这个插件是用来处理Vue组件的,用来扩展对Vue的支持,同时还会注入热更新的代码.
简单看下

**有同志就不懂了,这插件跟处理Vue组件有啥关系?**

> 答案: 其实呢,插件机制更够让Vite内部处理更多的资源,Vite内部本身是支持像js文件处理的,那不能处理Vue组件代码,scss等资源吧,都写在Vite内部,代码只会变得膨大臃肿,使用插件这种机制,处理不同的资源,对于日后的维护性,以及扩展性是非常友好的.

重点我们说下遇到Vue组件,会怎么处理转换,遇到Sfc组件,会使用@vue/compier-sfc转化,分为三部分,`template` `script` `css`

我们需要处理不同的资源.

**最终返回给浏览器的,查看下图**

![carbon (14).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f91337d25ef4c3884a6a9d7c8d99aa7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1906\&h=1570\&s=368531\&e=png\&b=151718)

图中第一部分: **是组件的script部分**

图中第二部分: **是组件的template部分,生成render函数**

图中第三部分: **是组件的css部分**

我们需要知道引入一个组件,会导出组件实例,实例包含组件的render渲染函数[渲染选项 | Vue.js (vuejs.org)](https://cn.vuejs.org/api/options-rendering.html#render)

我们可以通过Vue文档了解一下渲染的机制[渲染机制 | Vue.js (vuejs.org)](https://cn.vuejs.org/guide/extras/rendering-mechanism.html)

最终得到了一个组件开发的最终结果.

*特别说明*: Vue中的样式,最终处理成一个ESM,import快速导入.方便维护扩展.

`import "/src/App.vue?vue&type=style&index=0&scoped=7a7a37b1&lang.css";`

### **2.预构建**

预构建做了什么事?
我们知道开发阶段,需要根据不同请求编辑返回.

那如果这么多请求需要怎么处理?

过多的请求会造成带宽的压力,造成请求的加载等待,严重影响用户的体验💣💣

**预构建因此而生:**

将过多的请求,处理成只有一个请求,这样只需请求一次,例如，lodash-es 有超过 600 个内置模块！当我们执行 import { debounce } from 'lodash-es' 时，浏览器同时发出 600 多个 HTTP 请求！即使服务器能够轻松处理它们，但大量请求会导致浏览器端的网络拥塞，使页面加载变得明显缓慢。通过将 lodash-es 预构建成单个模块，现在我们只需要一个HTTP请求！详情Vite官方文档(<https://cn.vitejs.dev/guide/dep-pre-bundling.html>)

并且将一些模块.将CommonJs或者UMD的依赖转为Es模块, Vite开发的服务器 将所有的模块都视为ES模块处理.

什么条件模块下,才会预构建.

> 答:只有一些裸模块,也就是第三方库,没有相对路径,像一些`antd,dayjs,vue` 等.....

预构建,将处理的模块放在node\_modules/.vite,也就是每次引入这些模块都会去这些已经预处理的模块中去拿取.

并且 请求时设置 HTTP 头 `max-age=31536000, immutable` 进行强缓存,不用每次都去处理,不会再经过 Vite Dev Server，直接用缓存结果.

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09f837974d8647e5bafd47bff4214d31~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=728\&h=175\&s=61737\&e=png\&b=fefefe)

预构建分为**手动开启与自动开启**.

手动: 启动开发服务器时指定 `--force` 选项，或手动删除 `node_modules/.vite` 缓存目录

自动: 首次开启项目,我们会发现

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4305f77107e1467886f206a3aeab4c49~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1293\&h=127\&s=57823\&e=png\&b=1e1e1e)

`new dependencies optimized` 新的依赖正在预构建.

这样会把一些缓存到,**.vite/** 文件夹下:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f88bc33837a949bca22e563d2f1ba206~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=481\&h=417\&s=66157\&e=png\&b=262627)

> 注意: 依赖预构建仅适用于开发模式，并使用 `esbuild` 将依赖项转换为 ES 模块。在生产构建中，将使用 \`@rollup/plugin-commonjs([依赖预构建 | Vite 官方中文文档 (vitejs.dev)](https://cn.vitejs.dev/guide/dep-pre-bundling#file-system-cache))

会有一个问题,那就是动态引入模块的时候,根本不知道你要引入什么模块,只能在文件被浏览器请求并转换后才能发现.也就是执行了才知道引入什么模块!

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b523b79944354d8b9afb1d1bdb0f7f59~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=533\&h=38\&s=14447\&e=png\&b=1f1f1f)

正如我们上面看到的,意思发现了新的模块lodash-es,这时需要重新二次预构建,而且在有必要时候,会刷新页面.
这也就是为什么,我们常常在开发阶段,点击新的组件菜单的时候,发现莫名刷新一下,其实此时再预构建,等到第二次你在点击该页面就不会刷新了.

那我们有什么办法可以防止这种不好的体验呢?

> 答:有的,Vite支持使用`optimizeDeps.include` 或 `optimizeDeps.exclude`

####  include: 使用此选项可强制预构建链接的包。

![carbon (7).png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54688cf76c464ea8ab7fe0fb60409ce5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=908\&h=596\&s=104005\&e=png\&b=151718)

> 我们可以发现一开始就会预构建了这些include的依赖,查看.metadata.json文件

```js
   "optimized": {
    "jszip": {
      "src": "../../jszip/dist/jszip.min.js",
      "file": "jszip.js",
      "fileHash": "0e9ea30c",
      "needsInterop": true
    },
    "lodash-es": {
      "src": "../../lodash-es/lodash.js",
      "file": "lodash-es.js",
      "fileHash": "86bcc851",
      "needsInterop": false
    }
   }
```


**后面打开该引入的模块,不会强制刷新了,体验杠杠的🎉🎉**

#### exclude: 在预构建中强制排除的依赖项。


![carbon (2).png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f37acbdf4d6040219e31688762ea5f50~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=760&h=566&s=94764&e=png&b=2b2b2b)


我们将jsZip模块进行过滤,不构建,也就是不处理.这样会造成什么问题呢?

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec27ef456db543deb1c22e7180e45782~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=710\&h=53\&s=34875\&e=png\&b=fbedec)

> 直接给你报错了,**jsZip**根本不是~~ES~~模块,我们知道在开发阶段,是只能处理ES模块的.你排除了对jsZip的预构建.💥💥💥

### **3.插件系统**

    开发: Vite内部都是在Rollup的插件机制上,去实现一套类似的插件机制,大部分可以兼容rollup钩子.

    生产: 使用Rollup做生产的构建.直接使用Rollup成熟的打包能力进行扩展和优化

也就是说其实大部分的rollup插件,可以直接在Vite插件中使用,详细\[[插件 API | Vite 官方中文文档 (vitejs.dev)](https://cn.vitejs.dev/guide/api-plugin.html#rollup-plugin-compatibility)]

**插件钩子:**
以下为通用钩子

以下钩子在服务器启动时被调用：

*   [`options`](https://rollupjs.org/plugin-development/#options)
*   [`buildStart`](https://rollupjs.org/plugin-development/#buildstart)

以下钩子会在每个传入模块请求时被调用：

*   [`resolveId`](https://rollupjs.org/plugin-development/#resolveid)
*   [`load`](https://rollupjs.org/plugin-development/#load)
*   [`transform`](https://rollupjs.org/plugin-development/#transform)

以下钩子在服务器关闭时被调用：

*   [`buildEnd`](https://rollupjs.org/plugin-development/#buildend)
*   [`closeBundle`](https://rollupjs.org/plugin-development/#closebundle)

开发阶段实现了一套自己的插件容器来调度Roolup钩子.

以下为Vite特有的钩子

\[`config`]\([插件 API | Vite 官方中文文档 (vitejs.dev)](https://cn.vitejs.dev/guide/api-plugin.html#config))

\[`configResolved`]\([插件 API | Vite 官方中文文档 (vitejs.dev)](https://cn.vitejs.dev/guide/api-plugin.html#configresolved))

....具体查看\[Vite独有]\([插件 API | Vite 官方中文文档 (vitejs.dev)](https://cn.vitejs.dev/guide/api-plugin.html#vite-specific-hooks))

我们直接快速导入我们的配置文件中.
我们重点看下,`transform` `load` `resolveId`,这三个钩子,大部分的插件都使用到,能运用到80%的场景中

*   **transform:** 简单来说 转换模块的哪痛,进行自定义转换,比如babel转换,或者有一个需求我们可能需要将当引入的svg文件.转化为一个Vue组件

*   **load:** 加载模块的内容

*   **resolveId:** 解析文件的路径

插件执行的顺序,[不懂就戳](https://cn.vitejs.dev/guide/api-plugin.html#plugin-ordering)

*   Alias
*   带有 `enforce: 'pre'` 的用户插件
*   Vite 核心插件
*   没有 enforce 值的用户插件
*   Vite 构建用的插件
*   带有 `enforce: 'post'` 的用户插件
*   Vite 后置构建插件（最小化，manifest，报告）

我们可以发现可以利用 `enforce = ’pre‘` 与 `’post‘`来进行对插件的顺序调整

使用`apply` 属性指明它们仅在 `'build'` 或 `'serve'` 模式时调用,也就是在**预览**或**构建期**间有条件地应用

列举一些插件,同志们可以去看看

[图片优化,压缩](https://github.com/FatehAK/vite-plugin-image-optimizer/blob/main/src/index.ts)


[Commonjs转Esm](https://github.com/originjs/vite-plugins/blob/main/packages/vite-plugin-commonjs/src/index.ts)

更多插件,[自行翻阅](https://github.com/vitejs/awesome-vite#plugins)

***一瓶水 一个Debugger 解决无数问题🤪***




[附上最近做的一个小图标库](https://www.wwcattle.site/),有需要的伙伴们用起来,后面图标还会更新!

> 总结: 项目中越来越多使用Vite进行构建,学习构建要同时学习**Esbuild**与**Rollup**打包工具,当然在打包过程中使用Rollup进行打包构建,还是多多少少会有性能的差异,毕竟还是使用js来.尤大 正在开发一种**RollDown**底层使用Rust来进行构建(包含预构建与构建过程,二者合一),狠狠期待了!
> [速度应该会有飞速的提升](https://www.youtube.com/watch?v=hrdwQHoAp0M\&feature=youtu.be)
