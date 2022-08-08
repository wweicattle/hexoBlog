---
title: React开篇介绍
date: 2020-01-01 00:00:00
category: "React"
tags: React
---

# 前言

---

React 是前端三大框架之一，也是受开发人员比较喜欢的前端框架之一。
今天开始 React 之旅。
React 起源 2013.05 开源 Fackbook 因为对市场上的 js 框架不是很满意，就决定自己写一套

React 的特点：

> 1.声明式设计

> 2.高效 vdom 减少交互 dom 回流重绘

> 3.单向响应的数据流

> 4.声明式设计

---

> 理解 React 大致特点后，我们看下实际的操作吧。
> React 中支持`函数式组件（16.8.0 无状态使用hooks）`与`类组件class`

## class 组件

```js
import React from "react"

class WWTest extends React.Component {
  constructor() {
    super()
    this.state = {
      name: "render",
      age: 36,
      books: ["一句顶万句", "草丛生睦邻", "独侠人"],
      item: {
        text: "<span>innerthml</span>",
      },
    }
  }

  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <div>this is content！</div>
      </div>
    )
  }
}

export default WWTest
```

上面即是一个简单的 class 组件，注意类组件中定义状态用 `this.state` 设置。

修改状态值 `this.setState`

```js
import React from "react"

class WWTest extends React.Component {
  constructor() {
    super()
    this.state = {
      name: "render",
      age: 36,
      books: ["一句顶万句", "草丛生睦邻", "独侠人"],
      item: {
        text: "<span>innerthml</span>",
      },
    }
  }
  handlerCommit = function () {
    // 修改state中的name 与age自增
    this.setState({
      name: "xiaochen",
      age: this.state.age + 1,
    })
  }

  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <div>name:{this.state.name}</div>
        <div>age:{this.state.age}</div>
        <div>this is content！</div>
        <div onClick={this.handlerCommit.bind(this)}>change state</div>
      </div>
    )
  }
}

export default WWTest
```

通过上文可以发现 修改状态的 setState 修改即可，传入新的属性就可以把状态覆盖

> 注意：修改一次 setState 类组件中的 render 渲染函数会重新 执行 1 次，这样会重新进行生成 vnode 之后比较打补丁
> 注意，setState 在版本 18 之前，如何同时修改多个 setState 中的属性时，会有区别 1.如果多个 setState 是在同步中执行 2.如果多个 setState 是在异步像（setTimeout 中）

```js
import React from "react"
class WWTest extends React.Component {
  constructor() {
    super()
    this.state = {
      name: "render",
      age: 36,
      books: ["一句顶万句", "草丛生睦邻", "独侠人"],
      item: {
        text: "<span>innerthml</span>",
      },
    }
  }
  handlerCommitTwo = function () {
    this.setState({
      age: this.state.age + 1,
    })
    console.log(this.state.age)
    this.setState({
      age: this.state.age + 1,
    })
    console.log(this.state.age)
    this.setState({
      age: this.state.age + 1,
    })
    console.log(this.state.age)
  }
  handlerCommitOne = () => {
    setTimeout((val) => {
      this.setState({
        age: this.state.age + 1,
      })
      console.log(this.state.age)
      this.setState({
        age: this.state.age + 1,
      })
      console.log(this.state.age)
      this.setState({
        age: this.state.age + 1,
      })
      console.log(this.state.age)
    }, 0)
  }
  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <div>this is content！</div>
        <div onClick={() => this.handlerCommitOne()}>hanlderCommitOne</div>
        <div
          onClick={() => {
            this.handlerCommitTwo()
          }}
        >
          hanlderCommitTwo
        </div>
      </div>
    )
  }
}

export default WWTest
```

通过上文可以知道 执行 handlerCommitOne 输出：
`37 38 39`

执行 handlerCommitTwo 输出：
`37 37 37`

记住口诀：在同步任务中执行，setState 是异步任务 并且会合并所有 setState 操作 同样的 37
在异步任务中执行会是异步输出 37 38 39

```
重点说明：为了保证state中的值是正确的，后做一些操作我们应该放在setState的第二个形参handler中,看下图：

this.setState(({}),handler)

```

```js
this.setState(
  {
    age: this.state.age + 1,
  },
  () => {
    console.log("(--------+" + this.state.age)
  }
)
```

类组件定义事件

```js
import React from "react"

class WWTest extends React.Component {
  constructor() {
    super()
    ...
  }
  handlerCommit = function () {
    console.log("handlerCommit");
    console.log(this);
  }
  handlerCommitOne = () => {
    console.log("handlerCommitOne");
    console.log(this);

  }
  handlerCommitTwo = () => {
    console.log("handlerCommitTwo");
    console.log(this);
  }
  render(prop) {
    return (
      <div className="contain">
        类组件页面：PropProp
        <div>this is content！</div>
        <div onClick={this.handlerCommit.bind(this)}>hanlderCommit</div>
        <div onClick={() => this.handlerCommitOne()}>hanlderCommitOne</div>
        <div onClick={() => {
          this.handlerCommitTwo()
        }}>hanlderCommitTwo</div>
      </div>
    )
  }
}

export default WWTest

```

从上文可以看到定义事件中，最好使用箭头函数才能够保证 this 的指向，使用 function 函数 使用 bind 改变 this 的指向。

如何获取 dom 元素呢！像 input 文本框中的值

```js
import React from "react"

class WWTest extends React.Component {
  constructor() {
    super()
    this.curref = React.createRef()
  }

  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <input type="value" ref={this.curref} />
      </div>
    )
  }
}

export default WWTest
```

通过上文可以发现获取一个 dom 元素，`this.curref=React.createRef()` 再用于 dom 元素上

props 属性的验证：

```js
import React from "react"
import kerwinPropsTypes from "prop-types"

class WWTest extends React.Component {
  constructor() {
    super()
  }
  // 设置booleaNum必须为布尔值
  static propTypes = {
    booleaNum: kerwinPropsTypes.bool,
  }
  // 默认prop属性值
  static defaultProps = {
    wuwei: 1111,
  }
  render(prop) {
    return <div className="contain"></div>
  }
}

export default WWTest
```

> 通过上文我们可以发现，为了限制 prop 的类型的时候我们可以使用 `kerwinPropTypes` 进行校验

> 注意 ⚠️：一个小技巧 `this.state` 可以在子组件中进行{...this.state}

> 注意 类组件的 props 只能在 render 中进行访问到,需要在下一个宏，微任务中执行

`Prop与State的区别`：
Prop 是父组件传进来，可以设置默认值，不可以修改
State 是组件自身状态，不可以在外部修改，尽量多写一些无状态的组件，也是方便后续的复用

## 表单的受控与非受控

## 组件的受控与非受控

通过 props 来判断能不能修改，不能修改 受控组件

