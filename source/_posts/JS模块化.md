---
title: JavaScript模块化
date: 2021-04-11 10:10:02
category:  
  - "JavaScript大全"
tags: JavaScript
---









作为一名程序员，你时常会感慨看的技术也很多，但就是一看就忘，为何不动动手把这些知识用文章的方式进行保存起来！让我们动起手来～


# 梳理一下你对你模块化的理解～



## 前言 

前端模块化已不是一个新的名词，早期js只是简单的表单交互，随着不断的发展，代码不断膨胀，模块化开发出现是必然的。

如果你对模块的规范化模拟两可，那么阅读起来吧！



## 1 .模块化的概念，进化历程！

首先我们必须理解模块的出现带来的优点。

1.代码复用性，

2.提高可维护

3.方便代码间依赖关系

4.减少变量污染




早期没有模块化的概念，那么开发者又是怎么实现类似模块化的方式呢！

### 函数封装

  定义全局函数，每个功能看作是一个模块，用时直接调用函数名即可。

 ```js
function module1(){
 console.log("module1");
}

function module2(){
  console.log("module2");
}
//直接调用
module1();
module2();
 ```

缺点 全局变量污染，模块间无依赖，

### namespace模式，对象封装

解决函数的缺点，保证减少了变量的污染，只暴露一个模块名，把所有模块成员封装在一个对象中。

```js
let module1 = {
  name: "moduleName",
  val: "moduleVal",
  fun() {
    this.val = "newVal";
    console.log("this is fun things!");
  },
};

module1.name = "newName"; //直接修改内部属性，好的模块应该具有独立性，局部作用域。
module1.fun(); //能够调用fun 修改 val属性。
```

缺点：虽然能保证减少了变量的污染。但最大的缺点 外部能够利用模块名进行修改内部属性，存在安全隐患。



### IIFE（匿名函数自定义）

数据是私有的，通过挂载在window上来暴露接口，外部不能直接操作内部中的属性。

```js
// module.js文件
(function (window) {
  // 定义内部变量
  let name = "https://wweicattle.github.io/";

  //操作数据的函数
  function fun1() {
    //用于暴露有函数
    console.log(`foo() ${name}`);
  }

  function fun2() {
    //内部私有的函数
    console.log("otherFun()");
  }

  //暴露行为
  window.moduele1 = { fun1, fun2 }; //ES6写法
})(window);

```

我们可以知道，只有暴露出来的属性方法才可操作。IIFE的解决了 对象封装的缺点。

### IIFE增强：

现代使用的模块实现，引入jQuery，通过行参传入，这样做的优点能够保证模块的独立性，以及模块之间的依赖关系变明显。

```js
(
  // module.js文件
  function (window, $) {
    function callMethods() {
      console.log("callMethods");
    }
    //操作数据的函数
    function fun1() {
      //调用callMthods方法
      callMethods();
    }

    function fun2() {
      //内部私有的函数
      console.log("otherFun()");
      $("html").css("color", "red");
    }

    //暴露行为
    window.moduele1 = { fun1, fun2 }; //ES6写法
  }
)(window, jQuery);
```



随着js代码 逻辑业务增多，我们不断会把一个js文件看作一个简单的模块，请看下面

```html
  <script src="./jquery.js"></script>
  <script>
    function moduleMethods() {
        $("html").css("color", "red");
        console.log("moduleMethods");
     }
  </script>
  <script>
      // 执行moduleMethods方法
      moduleMethods();
  </script>
```
（补充）当然script文件 还包含重要的async，defer属性，异步加载模块。

不加属性即同步加载，此时会更加消耗时间开销，同时阻塞html解析，渲染。

async：遇到该js文件，立马加载，加载完后就执行，其会阻塞html的解析。

deffer：遇到该js文件，立马加载，此时会阻塞html解析，但加载完后不立马执行，等到后面html解析渲染完后才执行。

请看下列图，非常清晰。

 ![image-20211214202207688](/Users/wuwei/Library/Application Support/typora-user-images/image-20211214202207688.png)

  模块作用于全局作用作用域下，我们必须时刻保证模块的先后引入，需要先引入jquery 才行，而且每个js文件都是暴露在全局的，造成变量冲突，并且请求数量增多。这些一系列 问题出现， 让我们后期维护造成成本增高。

当然这种简单的模块化实现思想是进步的。那我们要以什么方式解决的呢，那就是**模块化规范**。

## 2.了解模块化规范： 

简单来说以一种规则，一种模块编写、模块依赖和模块运行的方案。

常见的javascript模块规范有：CommonJS ，AMD，CMD，UMD，ES6Module。

### （1）CommonJS

特点，优点：

1. 每一个文件看作一个模块，其中的变量属性是私有的，不会污染全局。

2. 模块多次加载，加载一次便被缓存起来，再次加载从缓存中取出。要想让模块重新加载，需手动清除缓存。

3. 模块加载依照顺序，依次在代码中出现先后。

4. Node.js的模块采用该规范。 exports导出，require导入。

5. 模块加载 值拷贝，若是引用类型则是浅拷贝（与ES6不同，ES6采用引用）

6. 它是以同步的方式加载模块，模块加载完成后就开始执行。一般来说nodejs这种服务于服务器，模块文件一般都已经存在本地磁盘了，所以同步的方式就可以。


**在服务器使用：**

// module1.js

```js
let  name="hello,module!";
let moduleMethods=()=>{
  name="hello,ou!"
}
module.exports = { name,moduleMethods };
```


// main.js

```js
var params= require("./module1");
console.log(params.name); //hello,module!
params.moduleMethods();
console.log(params.name); //hello,module!
```

特别注意 commonjs加载的机制是导出的值是导入的值的拷贝，加载一次便缓存起来，所以第二次输出的name值直接从缓存中取出“hello,module!”；



**在浏览器使用：**

需要借助（Browserity）需要进行编译打包处理，转换成浏览器所能识别的代码

- 第一步： npm install browserify

  以node上面module1.js与main.js文件为例，直接将main.js文件编译成浏览器所识别的语法

- 第二步：npx browserify main.js -o bundle.js

  我们发现生成一个buundle.js文件 创建一个index.html文件引入打开浏览器即可：

### （2）AMD （ASynchromous Module Definition）异步加载方式

特点，优点：

在浏览器中同步加载模块方式让用户等待更多的时间，等到上一个模块加载且执行完成后才可向后继续执行。异步加载的方式出现，一般浏览器采用AMD的规范。它的出现也让我们的模块更有顺序化，加载完成后执行一个指定回调。

AMD是RequireJs在推广时候模块定义的规范的产出。

 // require.js用法

 使用require([module],callback)关键词加载模块

- [module] :是一个数组，加载的模块名
- callback :加载成功返回的回调函数
  使用define定义模块 
- define（fn）；fn则是定义的模块体。
- define（[module],fn）若定义的模块还依赖其他模块，则第一个参数是一个数组，引入的模块文件名。

​    具体使用:

// 创建一个index.html 引入require.js设置date-main入口文件mian.js


```html
<script data-main="main.js" src="https://cdn.bootcdn.net/ajax/libs/require.js/2.3.6/require.js"
></script> 
```

main.js 引入定义模块module2.js

```js
(function() {
    console.log("main");
    require(['module2'], function(alerter) {
      alerter.showMsg()
    })
  })()
```

module2.js文件引入module1.文件

```js
define(['module1'], function(dataService) {
    console.log("module2");
    let name = 'ww'
    function showMsg() {
      alert(dataService.getMsg() + ', ' + name)
    }
    // 暴露模块
    return { showMsg }
  })

```

module1.js文件

```js
 define(function() {
    console.log("module1");
    let msg = 'this is Module！，you are'
    function getMsg() {
      return msg.toUpperCase()
    }
    return { getMsg } // 暴露模块
  })


```



输出

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3eec36b23e364a289b6fcfd736f0dd73~tplv-k3u1fbpfcp-watermark.image?)

 我们可以发现其AMD模块，保证了模块间的依赖关系,其还是异步加载不会阻塞，当然也可以根据需要动态加载模块。







### （3）CMD（Common Module Definition） 通用模块定义

与AMD类似，只不过模块定义加载，解析时机有所不同。

CMD专门作用于浏览器端，模块的加载是异步的，使用时才会加载。它集合了CommonJs和AMD的规范的特点。

SeaJS使用CMD模块定义规范。

具体用法：

 在index.html文件，入口文件main.js

```html
  <script src="./sea.js"></script>
  <script type="text/javascript">
      seajs.use("./main.js");
  </script>

```


```js
// main.js文件
define(function (require) {
  var m2 = require("./module2.js");
  var m3 = require("./module3.js");
  console.log(m2);
  console.log(m3);
});

```


```js
// module1.js文件
define(function (require, exports, module) {
  //内部变量数据
  var data = 'this is module1'
  //内部函数
  function show() {
    console.log( data)
  }
  //向外暴露
  exports.show = show
})

```


```js
//module2.js
define(function (require, exports, module) {
  console.log("modeul2");
  module.exports = {
    msg: 'I am module2'
  }
})
```

```js
// module3.js文件
define(function(require, exports, module) {
  console.log("module3");
    const name = 'module3'
    //同步引入module4 模块
    require("module4")
    exports.name = name
  })
```

```js
// module4.js文件
define(function (require, exports, module) {
  console.log("module4");
  //引入依赖模块(异步)module1模块
  require.async("./module1", function (m1) {
    m1.show()
    console.log("异步引入依赖模块1 ");
  });
});

```

输出

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7658fdff7da40bebfd607537f03c804~tplv-k3u1fbpfcp-watermark.image?)





**AMD与CMD区别**：

1.AMD依赖前置，一开始就开始加载，而CMD是依赖就近，需要时才去

2.AMD提前执行，CMD是延迟执行。

 ```js
   define(['./a', './b'], function(a, b) {  // 在定义模块时 就要声明其依赖的模块
       a.doSomething()
       // ....
       b.doSomething()
       // ....
   })
   
   define(function(require, exports, module) {
      var a = require('./a')
      a.doSomething()
      // ... 
      
      var b = require('./b') // 可以在用到某个模块时 再去require
      b.doSomething()
      // ... 
   })
 ```



### （4）UMD通用模块定义 

AMD与CommonJs的综合产物

主要用作浏览器端，他能够统一CommonJS和AMD规范生态系统。使用时进行检测要使用的哪个模块系统。

并把所有的包装在一个立即执行函数中（IIFE）。目的是实现两个生态共存。

//例子

   ```js
  (function (window, factory) {
          //是否支持CommonJs
        if (typeof exports === 'object') {
            module.exports = factory();
        } else if (typeof define === 'function' && define.amd) {
          //是否支持Amd
          //dosomething();
        } else {
          //全局上
            window.eventUtil = factory();
        }
    })(this, function () {
        return {};
    }
   ```



### （5）ES6 Module（重要）

  在ES6出现模块化之前，根本没有模块化的规范。直到ES6 模块的出现，他的规范得到越来越广泛的支持。

 它集成了COMMONJS与AMD两种优点。他的出现可以完全替代ConmonJS和AMD规范，成为浏览器通用的模块解决方案。

主要特点：

它的设计是静态加载或者编译时加载思想，不用在模块运行加载时就能确定其依赖关系。而CommonJS，AMD只能在运行时才能确定这些东西。

- 单例模式，只加载一次，

- export，impotr 导入导出，

- 默认模块是严格模式，this指向undefined，且定义的变量不会添加到window上

- 异步加载执行
- 等等...

简单使用：

 // 定义模块

```js
//module1.js
let obj={
    name:"this is module1.js"
}
export default obj
```

```js
//module2.js
let moduleTwo={
    moduleMethod:()=>{
        console.log("this is module2 methods");
    }
}
export default obj
```


// 引用模块

```js
// main.js
import modeleOne from "./module1.js";
import { moduleTwo } from "./module2.js";
// 输出module1对象
console.log(modeleOne);
// 执行module2方法
moduleTwo.moduleMethod();
```

在浏览器中使用

- 主流浏览器中加载只需在script标签中添加type=“module” 属性即可加载ES6模块。
 ```html
     <script src="./moduel1.js" type="module"></script>
 ```

- 大部分浏览器还是不支持该ES6模块，因此需要先转为ES5代码，之后再使用Browserify编译打包。
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/daa6724b75654a84aad926128e308d55~tplv-k3u1fbpfcp-watermark.image?)

```js
//module1.js
let obj={
    name:"this is module1.js"
}
export default obj
```

```js
//module2.js
let moduleTwo={
    moduleMethod:()=>{
        console.log("this is module2 methods");
    }
}
export default obj
```

```js
// main.js
import modeleOne from "./module1.js";
import { moduleTwo } from "./module2.js";
// 输出module1对象
console.log(modeleOne);
// 执行module2方法
moduleTwo.moduleMethod();
```
- 第一步
 - npm install babel-cli browserify -D
 - npm install babel-preset-es2015 -D
 - reset 预设(将es6转换成es5的所有插件打包)


- 第二步定义.babelrc
```js
{ "presets": ["es2015"] }
```


- 第三步npx babel main -d budle；生成如图：
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8700c986cafd452988ec0a204b4ca992~tplv-k3u1fbpfcp-watermark.image?)


已经编译成require语句了，直接下只要将main.js利用browserify转化为浏览器所识别的语法。

- 第四步npx browserify main.js -o bundle.js

再index.html中引入即可。

```html
<script type="text/javascript" src="bundle.js"></script>
```

输出
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8aa8b6dcdd04fa39697e87e9065d4d5~tplv-k3u1fbpfcp-watermark.image?)
 
    



在服务器中







ES6与CommonJs的区别：





总结：







































