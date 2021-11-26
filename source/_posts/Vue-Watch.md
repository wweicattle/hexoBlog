---
title: Vue-Watch源码解析
date: 2020-01-01 00:00:00
category: "Vue源码"
tags: Vue
---

### 前言
----
>我们知道的Vue中的wathcer是一个侦听器，监听响应数据的变化，当你想在数据发生变化时，需要进行开销较大的操作时，比如异步请求等。这时你会使用到侦听器。

首先让我们看看初始化的watch的源码！

```js
function Vue(){
    //初始化属性,data,computed,props,watch等等
    initState(this)
}

function initState(vm) {
    //......
    if (opts.watch) {
        initWatch(this, vm.$options.watch);
    }
}



function initWatch(vm, watch) {   
//遍历watch，为 每一个 watch添加订阅者就是user-watcher
    for (var key in watch) {    
        var watchOpt = watch[key];
        createWatcher(vm, key, handler);
    }
}


function createWatcher(
    vm,
    expOrFn,
    handler,
    options
  ) {
    //不同的watch写法，得到的hander也是不一样的
    if (isPlainObject(handler)) {
      options = handler;
      handler = handler.handler;
    }
    if (typeof handler === 'string') {
      handler = vm[handler];
    }
    //expOrFn是key ，handler是该key的监听回调,下面会很多次说起（监听回调）
    return vm.$watch(expOrFn, handler, options)
  }
```

按步骤一步一步看下来 上面先遍历watch，后面给每个侦听属性添加一个`user-watcher`，其中传的参数`expOrFn`是key,`handler`是该key的监听回调,`handler`的取值可能会有几种情况，列出我们可能的watch形式。

```js
    watch:{
        name:{
         handler(){},
        },
        name(){},
        name:"getNameWatch"
    }
```
是一个**对象**时，就把该对象的`handler`赋值`handler`，**函数**的话就直接赋值`handler`，**字符串**就把`vm.getNameWatch`赋值给`handler`。


好了，之后我们看看`$watcher`发生了什么。
    
```js
  Vue.prototype.$watch = function (
      expOrFn,
      cb,
      options
    ) {
      //下面会说
      var watcher = new Watcher(vm, expOrFn, cb, options);
      //立即执行一次该侦听wather
      if (options.immediate) {
          cb.call(vm, watcher.value);
      }
    };
```
    
 让我们快速吧视线移到`new Watcher(),`就是给每个侦听属性添加的`user-watcher`，看下`Watcher`源码，我会针对该知识，列重要的源码，就不累赘别的代码了。

```js
var Watcher = function (vm, key, cb, opt) {  
    this.vm = vm;    
    this.deep = opt.deep;    
    this.cb = cb;  
    // 下面这段主要是设置该key的监听回调
     this.getter = parsePath(expOrFn);
    };    
    // this.get 作用就是执行 this.getter()
    this.value = this.get();
};
```



```js
 function parsePath(path) {
    var segments = path.split('.');
    return function (obj) {
      for (var i = 0; i < segments.length; i++) {
        if (!obj) {
          return
        }
        obj = obj[segments[i]];
      }
      return obj
    }
  }
```
`parsePath`主要返回了一个匿名函数，函数中的内容主要是会从`vm`上获取该侦听`key`的`value`，这样会发生依赖收集。
看看之后的执行吧，之后赋值给`this.getter`,之后执行`this.get()`就是执行`this.getter()`，执行了就会对该侦听属性进行依赖收集。说了这么多还是稍微多一个过程看吧。

我们已经知道parsePath会返回该侦听属性的监听回调。接下里继续 执行this.get()方法。
```js
Watcher.prototype.get = function get() {
      //把当前的user-watcher设置为Dep.target
     pushTarget(this);
     //把vm传进去，执行监听的回调，之后如果回调中有依赖的响应会触发它的依赖收集user-watcher。
     value = this.getter.call(vm, vm);
    var vm = this.vm;
      if (this.deep) {
        traverse(value);
      popTarget();
    }
    return value
  };

```

`value = this.getter.call(vm, vm);`
这段会依赖收集watcher中的 key的订阅者，此时Dep.target是 user-watcher所以会被收集到subs数组中。

之后返回`this.get()`返回到`this.value`,再退到上面就是`Vue.prototype.$watch`中`new Watcher()`返回的`watcher`，再判断`immediate`是否为`true`表示立即执行一次该`key`的`handler`回调啦 。接着initWatch就大功告成了。
#### 之后在响应数据发生变化的时候就会派发更新，执行`update`就会执行user-watcher的监听回调啦啦啦！



>感谢阅读，希望有错误的地方指点指点，一起交流，一起进步。





























