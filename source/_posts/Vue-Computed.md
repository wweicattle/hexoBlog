---
title: Vue-Computed 源码解析
date: 2020-01-01 00:00:00
category: "Vue源码"
tags: Vue
---


# 前言
---
今天来研究下Vue中的 Computed源码过程，学习computed我们必须理解以下几点：

> 1. computed内部是怎么缓存的？
> 2. computed中的依赖响应数据如何依赖收集？


---
我们知道vue调用实例后，Vue内部initState时候，会处理很多不同的数据，包括有`data、props、methods....`

处理computed的方法就是initComputed。
看看initComputed做了些啥！

```js
function initComputed(vm, computed) {
    var watchers = vm._computedWatchers = Object.create(null);
    // 是否服务器渲染 
    var isSSR = isServerRendering();
    for (var key in computed) {
      var userDef = computed[key];
      var getter = typeof userDef === 'function' ? userDef : userDef.get;
      if (getter == null) {
        warn(
          ("Getter is missing for computed property \"" + key + "\"."),
          vm
        );
      }
      if (!isSSR) {
        // create internal watcher for the computed property.
        watchers[key] = new Watcher(
          vm,
          getter || noop,
          noop,
          computedWatcherOptions
        );
      }
      if (!(key in vm)) {
        defineComputed(vm, key, userDef);
      }
    }
  }
```
上面做了几件事1.为每个computed中的属性添加 watcher。2.defineComputed的处理

## 1.new Watcher生成一个computed-Watcher

#### 先来看下new Watcher()做了些啥，先列出重点的源码
```js
function Watcher(vm, expOrFn, options) {    
    this.dirty = this.lazy = options.lazy;    
    this.getter = expOrFn;   
    this.value = this.lazy ? undefined: this.get();
};
```
  

```js
// getter 就是 watcher 回调
Watcher.prototype.get = function() {  
    var value = this.getter.call(vm, vm);    
    return value
};

```
   
一开始`new Watcher(vm, getter, { lazy: true })`的`lazy`是设为true，这样在执行computed中属性初始化的时候，不用立马求值，算是一个小优化。只有等到后面再次请求computed中的值才会进行`this.get();`



## 2.再看看defineComputed做了一些什么鬼！

```js
function defineComputed(
    target, key, userDef
) {    
    // 设置 set 为默认值，避免 computed 并没有设置 set
    var set = function(){}      
    
    //  如果用户设置了set，就使用用户的set
    Object.defineProperty(target, key, {        
        // 包装get 函数，主要用于判断计算缓存结果是否有效
        get:createComputedGetter(key),        
        set:set
    });
}
```

其实就是给computed中的每一个属性都进行依赖拦截处理，`看下get中重点createComputedGetter。`

```js
function createComputedGetter(key) {    
    return function() {        
        // 获取到相应 key 的 computed-watcher
        var watcher = this._computedWatchers[key];        
        // 如果 computed 依赖的数据变化，dirty 会变成true，从而重新计算，然后更新缓存值 watcher.value
        if (watcher.dirty) {
            watcher.evaluate();
        }        
        // 这里后面重点讲，主要是让页面watcher再收集依赖一次
        if (Dep.target) {
            watcher.depend();
        }        
        return watcher.value
}


```
看到上面的`watcher.dirty`了吗，就是这个，当你dirty为true时才会执行内部的`watcher.evaluate()`,执行之后又会进行`this.dirty`进行设为false，这样就不用每一次都进行重新求值，起到了真正缓存的作用，那什么时候才要进行设为dirty为true呢，就是当computed依赖的响应数据进行变化的时候，会触发它依赖收集中的`computed-watcher`，之后进行update。看下面简单源码

```js
Watcher.prototype.update = function() {    
    if (this.lazy)  this.dirty = true;
    ....还有其他无关操作，已被省略
};
```


## 3.computed中的响应数据是如何收集依赖
Vue内部第一次的依赖收集是在`render`遍历生成vnode的时候 ，此时render像下面这样

```js
with(this)
 {
 return _c('div',{staticClass:"app"},_v(_s(computedval)+"\n ")，
 _c('button',{on:{"click":btn}},[_v("buttonn")])])
 }
```

```js
Watcher.prototype.evaluate = function() {    
    this.value = this.get();    
    this.dirty = false;
};
```
比如上面的`computedval`是一个计算属性，获取计算属性值会去触发该属性的get，因为上面说的dirty是`true`（不懂上面看一下！）之后会执行 `watcher.evaluate()`;重点看下evaluate中执行的`this.get()`方法，先重要几行源码。

```js
Watcher.prototype.get = function get() {
    //   将当前的computed-watcher设为Dep.target
    pushTarget(this);
    var value;
    var vm = this.vm;
    // 会执行响应数据的依赖，此时的Dep.target是computed-watcher
    value = this.getter.call(vm, vm);
    popTarget();
    return value
  };
```
接下来我会**将上面这段get方法按过程说明一下，小伙伴最好把这段代码印在脑子中哦！**
我们首先来pushTarget究竟发生了是什么。

```js
 Dep.target = null;
 //保存 watcher
 var targetStack = [];
  function pushTarget(target) {
    targetStack.push(target);
    console.log(targetStack.slice())
    Dep.target = target;
  }
  
  
```
把`computed-watcher`传进来，放进到`targetStack`数组中，此时该数组中有[页面watcher，computed-watcher],诶  ！小伙伴又会问为什么不是[computed-watcher]呢，是的会有一个**页面渲染watcher**。

我们得回来**Vue的初始化**`$mount`，之后会执行`mountComponent()`;不信看源码

```js
  Vue.prototype.$mount = function (
    el,
    hydrating
  ) {
    el = el && inBrowser ? query(el) : undefined;
    return mountComponent(this, el, hydrating)
  };

```
之后

```js
function mountComponent(
    vm,
    el,
    hydrating
  ) {
    var updateComponent;
      updateComponent = function () {
        console.log();
        debugger;
        vm._update(vm._render(), hydrating);
    // 已经会执行new watcher一次
    new Watcher(vm, updateComponent, noop, {
      before: function before() {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate');
        }
    }
  }
```
重点看上面的new Watcher()，看下构造函数主要的Watcher源码，


```js
 var Watcher = function Watcher(){
     this.value = this.lazy ?
      undefined :
      this.get();
 }
```
之后执行this.get(),因为new watcher没传lazy，默认就是空的。接着是不是就又会执行pushTarget(this),之后页面watcher会被保存在targetStack中，之后就会执行updatecompoennt了，之后会遍历render生成vnode，**回到3问题的开头了**。

我们已经知道pushtarget中的过程了，接下来接着执行 value = this.getter.call(vm, vm);此时会发生响应数据的get

```js
  Object.defineProperty(obj, key, {
      get: function reactiveGetter() {
        //此时Dep.target是computed-watchers
        if (Dep.target) {
         //依赖收集
          dep.depend();
        }
        return value
      },
```
此时依赖的响应数据就已经收集到`computed-watcher`，此时`value`已经有值。
接下来我们看看会执行popTarget()

```js
  function popTarget() {
    targetStack.pop();
    Dep.target = targetStack[targetStack.length - 1];
  }
```
popTarget方法主要做了把`computed-watcher`移除，此时Dep.target是页面的watcher，此时可以说把get执行已经执行完，就是evaluate执行完了。
```js
function createComputedGetter(key) {    
    return function() {        
        // 获取到相应 key 的 computed-watcher
        var watcher = this._computedWatchers[key];        
        // 如果 computed 依赖的数据变化，dirty 会变成true，从而重新计算，然后更新缓存值 watcher.value
        if (watcher.dirty) {
            watcher.evaluate();
        }        
        // 这里后面重点讲，主要是让页面watcher再收集依赖一次
        if (Dep.target) {
            watcher.depend();
        }        
        return watcher.value
}

```

看到了吗就是上满的`watcher，evaluate()`执行，之后执行`this.get()`我们才balabala上面说了一大堆，说明这个`evaluate`不简单，里面还有一些方法都没说，只是把我们主要的说一下。

接着继续看又会执行

```js
 if (Dep.target) {
      watcher.depend();
    }   
```
这一段也是重要哦，后面有时间在扩展吧，主要是让响应数据的dep中的subs收集到页面的watcher。

这样上面computed中的属性所依赖的响应的数据就会把computed-watcher与页面的watcher全部收集到了。

这时当你响应的数据发生变化，就会遍历subs订阅者[computed-wathcer,页面wathcer],执行update，最终达到更新视图。






>感谢小伙伴的阅读，希望文章有错的，可以交流一下。共同学习




























