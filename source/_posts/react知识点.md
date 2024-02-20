---
title: React知识简单梳理
date: 2022-01-12 08:30:10
category: "React"
tags: React
---
## 概述

本文会把react使用过程中的知识点做一下梳理。

## 原理

参考[图解React原理](https://link.juejin.cn?target=https%3A%2F%2F7kms.github.io%2Freact-illustration-series%2F "https://7kms.github.io/react-illustration-series/")。

### 几个包

在react使用过程中比较重要的包

-   react
-   react-dom
-   react-reconciler
-   scheduler

react-dom提供一个api启动项目，react提供了其他绝大部分api来具体操作react。  
react-dom通过schedulerUpdateOnFiber发起更新请求。  
reconciler处理并将fiber树创建的逻辑封装成回调交给scheduler调度处理，返回新的fiber树，提交给react-dom渲染到屏幕上。

### 常见对象

ReactElement, Fiber, DOM 三者的关系： reactElement，所有jsx的节点都会被babal转化React.createElement，创建一个ReactElement对象.每个reactElement生成一个内存中的fiber节点，fiber节点生成fiber树，fiber树对应dom树。

[ReactElement](https://link.juejin.cn?target=https%3A%2F%2F7kms.github.io%2Freact-illustration-series%2Fmain%2Fobject-structure%23reactelement-%25E5%25AF%25B9%25E8%25B1%25A1 "https://7kms.github.io/react-illustration-series/main/object-structure#reactelement-%E5%AF%B9%E8%B1%A1")对象属性包括

-   type属性表示节点的种类，比如字符串，即span等dom节点，函数，比如function和class组件，还有react节点类型，比如portal,context,fragment，会在reconciler等以不同方式处理。
-   key属性，用于diff，默认null
-   props属性

[Fiber](https://link.juejin.cn?target=https%3A%2F%2F7kms.github.io%2Freact-illustration-series%2Fmain%2Fobject-structure%23fiber-%25E5%25AF%25B9%25E8%25B1%25A1 "https://7kms.github.io/react-illustration-series/main/object-structure#fiber-%E5%AF%B9%E8%B1%A1")用来在内存中表示dom节点，除了首次渲染，内存中维护两个fiber树，属性包括

-   stateNode 对应的dom节点
-   updatequeue，一个链表存储的更新队列，每一次发起更新, 都需要在该队列上创建一个update对象
-   memoizedState 上一次生成子节点之后保持在内存中的局部状态，在函数组件中指Hook队列

### react运行过程

包括首次启动和二次渲染，两者的触发和render阶段（目前的这个阶段不可中断，同步模式）不同，commit阶段相同。

#### 启动的触发

17.2中，有三种启动方式，后两种稳定版本不提供

-   legacy 当前的方式，不支持concurrency
-   block 过渡版本
-   currency 全部功能

启动过程中初始化项目，主要创建了三个对象

-   ReactDOM(Blocking)Root，包含render等方法，引导react启动
-   fiberRoot 包含fiber构建过程中的状态，通过前一个的_internalRoot属性访问
-   HostRootFiber 第一个fiber对象，通过前一个的current属性访问，它的子节点是App reactelement对应的fiber节点

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/054da90110bb4c70b799c82202f89cd8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1023&h=368&s=13688&e=webp&b=fefefe) 启动完了更新容器updateContainer

```js
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: Container,
  forceHydrate: boolean,
  callback: ?Function,
) {
  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // 初次调用, root还未初始化, 会进入此分支
    //1. 创建ReactDOMRoot对象, 初始化react应用环境
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        // instance最终指向 children(入参: 如<App/>)生成的dom节点
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 2. 更新容器
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    // root已经初始化, 二次调用render会进入
    // 1. 获取FiberRoot对象
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // 2. 调用更新
    updateContainer(children, fiberRoot, parentComponent, callback);
  }
  return getPublicRootInstance(fiberRoot);
}
```

创建update加入队列并进一步调用scheduleUpdateOnFiber进入reconciler阶段

```js
// ... 省略了部分代码
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): Lane {
  // 获取当前时间戳
  const current = container.current;
  const eventTime = requestEventTime();
  // 1. 创建一个优先级变量(车道模型)
  const lane = requestUpdateLane(current);

  // 2. 根据车道优先级, 创建update对象, 并加入fiber.updateQueue.pending队列
  const update = createUpdate(eventTime, lane);
  update.payload = { element };
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    update.callback = callback;
  }
  enqueueUpdate(current, update);

  // 3. 进入reconcier运作流程中的`输入`环节
  scheduleUpdateOnFiber(current, lane, eventTime);
  return lane;
}
```

#### 二次渲染的触发

setState和dispatchAction都会创建update对象，并添加到Fiber节点的updateQueue队列，然后调用scheduleUpdateOnFiber，进入reconciler中。

在shedule相关逻辑中，具体的任务由ensureRootIsScheduled组织，保证每个root只有一个任务，该任务会作为一个回调作为另一个宏任务（由[MessageChannel](https://link.juejin.cn?target=https%3A%2F%2Fsegmentfault.com%2Fa%2F1190000022942008 "https://segmentfault.com/a/1190000022942008")实现）在当前同步代码执行结束执行。 当当前同步代码多次调用setState时，如果新旧回调优先级一样，则会复用之前的任务，多次更新一起处理，否则取消之前的任务新建新的任务。

只要祖先组件重新渲染，子组件就默认重渲染,参考[Optimizing React performance by preventing unnecessary re-renders](https://link.juejin.cn?target=https%3A%2F%2Fwww.debugbear.com%2Fblog%2Freact-rerenders "https://www.debugbear.com/blog/react-rerenders")

##### 批量更新

前面提的当调用多个状态更新时节流到一次重渲染，这种节流在v18之前，只适用于react事件处理器，其他比如在promise中、定时器中或原生事件处理器中时，不会批量处理。在v18以后，这些都会批零处理，使用`flushSync`可以显式同步更新。

#### render阶段

入口为scheduleUpdateOnFiber，如果优先级为同步，则直接构建fiber树，否则注册一个任务按调度执行。

render阶段构建fiber树，这个阶段分beginWork和completeWork两步，前一步从hostrootfiber开始进行深度优先遍历为每个fiber节点生成表示各种副作用的flags，后一步从叶子节点返回，也会生成update等flags，并将所有flags连成一个effectlist供commit阶段处理。

当首次渲染时内存中没有可对比的fiber树，beginwork阶段根据ReactElement创建对应的fiber节点，二次渲染时，要通过reactElement和current树的对比来决定怎么生成新的fiber节点以及添加flags。

这里的对比过程具体为: 比较current tree上的fiber节点和新的reactELement，对于单个节点，如果是新增的直接新建，否则比较type决定是否复用，对于子节点，分别对比type和key（默认为null）决定复用规则，打上flags。

#### commit

fibertree构建完成调用commit tree，这个过程主要是分为三批处理带有各种标记的副作用，其中第二批将新生成的fiber树对应的dom树渲染到屏幕上。

### hook

hook主要用于处理函数组件的状态和副作用，每个fiber节点在memoizedState上挂载了一个hook链表，该链表根据在组件出现的顺序排列，当进行新的fiber树构建时，直接复制到新的树上。

### 事件

在根dom元素上监听除了scroll等事件外的子元素事件

## 组件

组件有两种写法，模板和jsx（这里等效于相应render函数），其中模板的语法固定，能够对性能深度优化，jsx就是js，写法很灵活。

svelte为代表的library是用模板，react为代表的是用jax，vue都支持，但性能原因推荐模板。

拆分组件时可以像拆分函数一样，使用单一职责原则。

组件的写法在react中包括class组件和函数组件。

## 函数组件

### props

是使用该组件时的参数，另外props.children表示组件使用时两个tag中间包含的内容。 props是只读的。

当一个函数直接传入（而不是作为值传入）时，每次渲染都是[新的函数](https://link.juejin.cn?target=https%3A%2F%2Freactjs.org%2Fdocs%2Ffaq-functions.html "https://reactjs.org/docs/faq-functions.html").

### ref

ref表示一个到dom元素或react元素的引用，可以在父组件创建然后使用React.forwardRef传给子组件，然后指定对应被引用对象。

#### 基本使用

1.  创建ref，这里推荐使用const refContainer= useRef(initialValue)

其是一个可变的对象，可以通过.current获取保存的值，在这里用来保存一个dom元素，当然也可以保存其他想保存的数据，ref变化不会引起组件重新渲染。

另外可以使用等效的React.createRef()

2.  添加ref,当用于dom元素时直接将ref添加到其ref属性上即可，用于react元素时由于没有实例，因此不能直接用,需要借助useImperativeHandle(ref, createHandle, [deps]) 其中createHandle是一个函数，其返回值赋值给ref，提供给父组件调用，比如

```js
function FancyInput(props, ref) {
  const inputRef = useRef();
  useImperativeHandle(ref, () => ({
    focus: () => {
      inputRef.current.focus();
    }
  }));
  return <input ref={inputRef} ... />;
}
FancyInput = forwardRef(FancyInput);
```

在当前组件创建一个ref指定dom元素，从而在父组件间接操作子组件内的dom元素。

```js
复制代码
<FancyInput ref={inputRef} /> 

inputRef.current.focus()
```

#### 当作数据容器

组件的单次渲染中，state和props是一个固定值，如果我们想获取其他渲染时期的数据，可以使用ref。

比如获取上次渲染的，其中useEffect会在state和衍生的ui更新完以后执行，因此当页面渲染时，ref中保存的还是上次渲染后赋值的。  
等useEffect执行后ref被修改，但不会触发渲染，因此ui上还是上次渲染的值。

```js
复制代码
function Counter() {
  const [count, setCount] = useState(0);

  const prevCountRef = useRef();
  useEffect(() => {
    prevCountRef.current = count;
  });
  const prevCount = prevCountRef.current;

  return <h1>Now: {count}, before: {prevCount}</h1>;
}
```

比如读取未来渲染时期的

```js
function Example() {
  const [count, setCount] = useState(0);
  const latestCount = useRef(count);

  useEffect(() => {
    // Set the mutable latest value
    latestCount.current = count;
    setTimeout(() => {
      // Read the mutable latest value
      console.log(`You clicked ${latestCount.current} times`);
    }, 3000);
  });
```

#### callback ref

当将ref指定为函数时，这被称为callback ref

```js
复制代码
  <input ref={props.inputRef} />
```

会在绑定和解除绑定时调用,参数是对应dom元素和null，在重新渲染时不会调用，如果想监控dom元素的变化，可以使用ResizeObserver

### State

组件包含内部状态，状态是组件某一时刻的数据快照，状态改变会引起组件重新渲染。

函数组件的状态包括三个来源

-   自身
-   props
-   context

函数组件的状态通常使用hook处理。

#### useState

使用方法如

```js
const [state, setState] = useState(initialState);
```

其中参数是该state初始值，可以是一个值或返回值得函数，返回一个数组，分别表示当前变量和修改当前变量的函数。

当修改当前变量时可以直接将新值传给setState，也可以传一个函数，其中参数是上一个值.

#### useReducer

```js
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

用于依赖多个值或依赖之前的值的场景，也可以将dispatch作为props传给子组件。

其中第一个参数是reducer，即(state, action) => newState，第二个参数是初始值，如果要懒计算，初始值是init(initialArg)

action包括type和其他自定义字段。

返回，state和dispatch.

useState是useReducer的一种特殊情况.

#### context

用于为多层的多个组件提供state,使用context时使用不当会造成不必要的渲染，可以将context细粒化，或直接使用redux,可参考[如何有效减少使用useContext导致的不必要渲染](https://link.juejin.cn?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F4a41d04c8e48 "https://www.jianshu.com/p/4a41d04c8e48")

1.创建context

```js
const MyContext = React.createContext(defaultValue);
```

默认值会在没有匹配的provider时使用

2.  用来挂载context

```js
<MyContext.Provider value={/* some value */}>
```

提供值给子组件消费，provider可以嵌套并进行覆盖。

3.订阅context 可以使用

```js
<MyContext.Consumer>
  {value => /* render something based on the context value */}
</MyContext.Consumer>
```

比如

```js
import {ThemeContext} from './theme-context';

function ThemeTogglerButton() {
  // The Theme Toggler Button receives not only the theme
  // but also a toggleTheme function from the context
  return (
    <ThemeContext.Consumer>
      {({theme, toggleTheme}) => (
        <button
          onClick={toggleTheme}
          style={{backgroundColor: theme.background}}>
          Toggle Theme
        </button>
      )}
    </ThemeContext.Consumer>
  );
}
```

也可以使用hook

```js
useContext(MyContext);
```

### effect

#### useEffect

```js
useEffect(didUpdate,[dependences]);
```

用来当state更新后执行副作用。 在react中，每次渲染都有它自己的数据，包括state和函数，每次渲染都可以看成是一次函数的分别调用。

disUpdate是个函数，函数体是执行的副作用，会在页面渲染后执行，如果有返回值，会执行一些清理作用，会在下次渲染结束执行然后执行新的effect。

这里整理一下完整的执行步骤

1.  状态变化
1.  函数组件执行，组件中涉及到状态的地方会被换成当前state
1.  更新ui
1.  清理上一次effect
1.  执行新的effect

第二个参数是个依赖项，只有依赖项有变化时才会引起effect调用。

#### useLayoutEffect

会在元素更新完，但还没渲染时同步调用，从而阻塞渲染

#### 父子组件执行顺序

```
javascript
复制代码
import { useState, useEffect, useLayoutEffect } from 'react'

export default function RenderOrder() {
  const [state, setState] = useState(0)
  useEffect(() => {
    console.log('parents effect' + state)
  }, [state])
  useLayoutEffect(() => {
    console.log('parents layout effect' + state)
  }, [state])
  console.log('parent exec' + state)
  return (
    <div>
      <button onClick={() => setState((c) => c + 1)}>change</button>
      <Children state={state} />
      {state}
    </div>
  )
}

function Children({ state }: { state: number }) {
  useEffect(() => {
    console.log('child effect' + state)
  }, [state])
  useLayoutEffect(() => {
    console.log('child layout effect' + state)
  }, [state])
  console.log('child exec' + state)
  return <div>children{state}</div>
}
```

执行顺序是函数组件执行时先父后子，副作用先子后父 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78fed5ca5f3448a7814a58470ca983d4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=531&h=331&s=14624&e=webp&b=fefcfc)

### 缓存

#### useCallback

```js
const memoizedCallback = useCallback(
  fn,
  [a, b],
);
```

将一个函数缓存

#### useMemo

将一个值缓存

```
scss
复制代码
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`,前者是后者的特殊情况。

#### React.memo

作用类似于shouldComponentUpdate()

### 其他用法

#### Portals

这里提供一个在挂载dom元素以外渲染节点的方式

```
scss
复制代码
ReactDOM.createPortal(child, container)
```

其中第一个参数表示可以作为children的参数，比如string，元素或fragment。

除了渲染位置，其他和正常写法一致。

#### Transitions

用来区分紧急的和非紧急的需求。  
紧急的交互比如输入、点击等，非紧急的比如切换ui。  
在`startTransition`中的处理可以被紧急的更新打断，另外还提供了一个hook`useTransition`

```
javascript
复制代码
function App() {
  const [isPending, startTransition] = useTransition();
  const [count, setCount] = useState(0);
  
  function handleClick() {
    startTransition(() => {
      setCount(c => c + 1);
    })
  }

  return (
    <div>
      {isPending && <Spinner />}
      <button onClick={handleClick}>{count}</button>
    </div>
  );
}
```

#### Suspense

目前和`React.lazy`一起使用用来提示loading

```
javascript
复制代码
// This component is loaded dynamically
const OtherComponent = React.lazy(() => import('./OtherComponent'));

function MyComponent() {
  return (
    // Displays <Spinner> until OtherComponent loads
    <React.Suspense fallback={<Spinner />}>
      <div>
        <OtherComponent />
      </div>
    </React.Suspense>
  );
}
```

当同时有Transitions时，loading时显示旧界面。

#### useId

用来生成id，避免hydration时的错误

#### useDeferredValue

用于复制当前一个状态，在紧急更新后更新。

#### Error Boundaries

用来捕获子组件的发生的错误，从而执行一些副作用或者展示一个回退ui。  
注意有四个场景不会捕获

1.  事件处理
1.  异步代码
1.  ssr
1.  本身

#### Higher-Order Components

高阶组件就是一个函数，输入一个组件，输出另一个组件

#### Render Props

通过一个值为函数的prop来共享代码

#### 自定义hook

用来将组件逻辑提取到一个可复用的函数，命名以use开头，使用时像使用其他hook一样。

对于react而言，使用自定义hook跟直接在组件中执行自定义hook中的代码一样

这里想强调的是自定义hook的参数没限制，且当参数变化时可以在自定义hook中使用useEffect监听。

```js
function Counter() {
  const [count, setCount] = useState(0);
  const prevCount = usePrevious(count);
  return <h1>Now: {count}, before: {prevCount}</h1>;
}

function usePrevious(value) {
  const ref = useRef();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}
```

### hook中的事件监听

-   [Master Your React Skills With Event Listeners](https://link.juejin.cn?target=https%3A%2F%2Fbetterprogramming.pub%2Fmaster-your-react-skills-with-event-listeners-ebc01dde4fad "https://betterprogramming.pub/master-your-react-skills-with-event-listeners-ebc01dde4fad")
-   [React Hook Optimised Event Listener](https://link.juejin.cn?target=https%3A%2F%2Fgeoffroymounier.medium.com%2Freact-hook-optimised-scrollevent-listener-13513649a64d "https://geoffroymounier.medium.com/react-hook-optimised-scrollevent-listener-13513649a64d")

## class组件

class组件是一个继承了React.Component或React.PureComponent的class，这两个父类的区别是后者使用state和props的浅对比实现了shouldComponentUpdate()

新建的class必须实现render方法，用来返回待渲染的react元素。

### 生命周期

整体生命周期是[这样的](https://link.juejin.cn?target=https%3A%2F%2Fprojects.wojtekmaj.pl%2Freact-lifecycle-methods-diagram%2F "https://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/")。  
当首次渲染时依次执行

-   [**`constructor()`** ](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23constructor "https://zh-hans.reactjs.org/docs/react-component.html#constructor")构造函数
-   [`static getDerivedStateFromProps()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23static-getderivedstatefromprops "https://zh-hans.reactjs.org/docs/react-component.html#static-getderivedstatefromprops") 从props上获取state，会在每次render之前执行，静态方法
-   [**`render()`**](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23render "https://zh-hans.reactjs.org/docs/react-component.html#render")
-   [**`componentDidMount()`**](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23componentdidmount "https://zh-hans.reactjs.org/docs/react-component.html#componentdidmount")

更新时依次执行

-   [`static getDerivedStateFromProps()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23static-getderivedstatefromprops "https://zh-hans.reactjs.org/docs/react-component.html#static-getderivedstatefromprops")
-   [`shouldComponentUpdate()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23shouldcomponentupdate "https://zh-hans.reactjs.org/docs/react-component.html#shouldcomponentupdate")
-   [**`render()`**](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23render "https://zh-hans.reactjs.org/docs/react-component.html#render")
-   [`getSnapshotBeforeUpdate()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23getsnapshotbeforeupdate "https://zh-hans.reactjs.org/docs/react-component.html#getsnapshotbeforeupdate") 在最近一次渲染输出（提交到 DOM 节点）之前调用。它使得组件能在发生更改之前从 DOM 中捕获一些信息（例如，滚动位置）。此生命周期方法的任何返回值将作为参数传递给 `componentDidUpdate()`。
-   [**`componentDidUpdate()`**](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23componentdidupdate "https://zh-hans.reactjs.org/docs/react-component.html#componentdidupdate")

卸载时执行

-   [**`componentWillUnmount()`**](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23componentwillunmount "https://zh-hans.reactjs.org/docs/react-component.html#componentwillunmount")

发生错误时执行

-   [`static getDerivedStateFromError()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23static-getderivedstatefromerror "https://zh-hans.reactjs.org/docs/react-component.html#static-getderivedstatefromerror")
-   [`componentDidCatch()`](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23componentdidcatch "https://zh-hans.reactjs.org/docs/react-component.html#componentdidcatch")

### [](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-component.html%23other-apis "https://zh-hans.reactjs.org/docs/react-component.html#other-apis")

### 函数组件和class组件区别

函数使用闭包捕获了每次渲染的变量，参考[# How Are Function Components Different from Classes?](https://link.juejin.cn?target=https%3A%2F%2Foverreacted.io%2Fhow-are-function-components-different-from-classes%2F "https://overreacted.io/how-are-function-components-different-from-classes/")

### 代码复用

class组件最开始是用[mixin](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F20361937 "https://zhuanlan.zhihu.com/p/20361937")，由于[它的一些问题](https://link.juejin.cn?target=https%3A%2F%2Freactjs.org%2Fblog%2F2016%2F07%2F13%2Fmixins-considered-harmful.html "https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html")，现在推荐使用[HOC](https://link.juejin.cn?target=https%3A%2F%2Freactjs.org%2Fdocs%2Fhigher-order-components.html "https://reactjs.org/docs/higher-order-components.html")和[Render Props](https://link.juejin.cn?target=https%3A%2F%2Freactjs.org%2Fdocs%2Frender-props.html "https://reactjs.org/docs/render-props.html")。

## 不同场景下的diff方法

-   shouldComponentUpdate [用浅比较之前和之后的props和state](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-api.html%23reactpurecomponent "https://zh-hans.reactjs.org/docs/react-api.html#reactpurecomponent")
-   react.memo [用浅对比对比props](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Freact-api.html%23reactmemo "https://zh-hans.reactjs.org/docs/react-api.html#reactmemo")
-   useState和usereducer和useEffect 用[object.is](https://link.juejin.cn?target=https%3A%2F%2Fzh-hans.reactjs.org%2Fdocs%2Fhooks-reference.html%23bailing-out-of-a-state-update "https://zh-hans.reactjs.org/docs/hooks-reference.html#bailing-out-of-a-state-update")
-   useContext [使用object.is](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Freact%2Fblob%2F12adaffef7105e2714f82651ea51936c563fe15c%2Fpackages%2Freact-reconciler%2Fsrc%2FReactFiberNewContext.old.js%23L128 "https://github.com/facebook/react/blob/12adaffef7105e2714f82651ea51936c563fe15c/packages/react-reconciler/src/ReactFiberNewContext.old.js#L128"),不会在意react.memo和shouldComponentUpdate()
-   redux [使用===](https://link.juejin.cn?target=https%3A%2F%2Freact-redux.js.org%2Fapi%2Fhooks%23equality-comparisons-and-updates "https://react-redux.js.org/api/hooks#equality-comparisons-and-updates")

### 浅对比

react中经常遇到浅对比这个概念，大概步骤包括

1.  如果Obejct.is返回true则返回true
1.  如果其中一个不是对象或是null返回false
1.  如果两者自身可枚举属性数量不同，返回false
1.  对每个属性使用Object.is判断是否相等

详见[源码](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ffacebook%2Freact%2Fblob%2F0e100ed00fb52cfd107db1d1081ef18fe4b9167f%2Fpackages%2Fshared%2FshallowEqual.js%23L18 "https://github.com/facebook/react/blob/0e100ed00fb52cfd107db1d1081ef18fe4b9167f/packages/shared/shallowEqual.js#L18")

  