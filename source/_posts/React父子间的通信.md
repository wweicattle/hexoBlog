---
title: React开篇介绍
date: 2020-01-01 00:00:00
category: "React"
tags: React
---

# 父子间的通信方法：

1.通过将父组件中的函数给子组件，之后在子组件中进行调用

```js
import React from "react"

class WWTest extends React.Component {
  constructor() {
    super()
  }

  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <ClassCom booleaNum={false} addSum={this.clickSum} />
        <div>this is content！</div>
      </div>
    )
  }
}

export default WWTest
```

2.通过在父组件中 获取子组件的 ref 在调用子组件中的方法

## Ref 版的表单组件

```js
import React from "react"
class WWTest extends React.Component {
  render() {
    return (
      <input
        value={this.state.name}
        onChange={(e) => {
          this.setState({
            name: e.target.value,
          })
        }}
      />
    )
  }
}
class WWTest extends React.Component {
  constructor() {
    super()
    this.curref = React.createRef()
  }

  render(prop) {
    return (
      <div className="contain">
        类组件页面：
        <div className="contain">
          <ClassCom ref={this.currefs} />
          <div
            onClick={() => {
              console.log(this.currefs)
            }}
          >
            change
          </div>
        </div>
      </div>
    )
  }
}

export default WWTest
```

通过上文我们可以看到就是在父组件上绑定一个 ref，直接可以获取 ref

# 非父子间的通信方法

（1）首先我们运用第一种方法就是使用发布订阅模式，当然后面还有更加成熟的 redux 进行管理我们的状态
先看下代码

```js
// 这是发布订阅简单函数，subscribe进行收集我们订阅者，publish进行派发更新
const emit = {
  list: [],
  subscribe(callback) {
    this.list.push(callback)
  },
  publish(param) {
    this.list.forEach((val) => val(param))
  },
}
export default emit

//在constructor中，进行注册，只要当值发生变化时候我们再publish
emit.subscribe((names) => {
  console.log(names)
  this.setState({
    names,
  })
})

// 典型在子组件中的文本框进行操作
   <input type="value" ref={this.curref} value={this.props.name} onChange={(e) => {
      emit.publish(e.target.value);
  }} />
```
（2）通过react拓展的context进行通信，消费者 comsumer  ，供应者 provider

```js
// 这是发布订阅简单函数，subscribe进行收集我们订阅者，publish进行派发更新
const emit = {
  list: [],
  subscribe(callback) {
    this.list.push(callback)
  },
  publish(param) {
    this.list.forEach((val) => val(param))
  },
}
export default emit

//在constructor中，进行注册，只要当值发生变化时候我们再publish
emit.subscribe((names) => {
  console.log(names)
  this.setState({
    names,
  })
})

// 典型在子组件中的文本框进行操作
   <input type="value" ref={this.curref} value={this.props.name} onChange={(e) => {
      emit.publish(e.target.value);
  }} />
``