---
title: React生命周期
date: 2022-12-01 10:00:02
category: "React"
tags: React
---

# 类组件生命钩子：

1. componentWillMount
   初始化状态使用；
   版本
   16.2 节点 fiber 优化（引入新的 UNSAFE_componentWillMount）
   16.8 节点 引入 hooks

2. render

   > 注意 ⚠️ 不要在 render 进行修改状态 state 中的属性，会有错误 （无限循环调用）

3. componentDidMount （像 better scroll 需要 dom 成型）

4. componentWillUpdate（执行频率多）

UNSAFE_componentWillUpdate
更新之前

> 注意 ⚠️ 当然你同时修改的多个 state 中的状态时，只会执行一次 componentWillUpdate

5. componentDidUpdate
   更新之后 形参是老的props 和state

如果修改 state 的值的话，会执行 componentWillUpdate->render->componentDidUpdate

> 注意 ⚠️ 不要在 render 进行修改状态 state 中的属性，会有错误 （无限循环调用）

6.shouldComponentUpdate 是否当 state 发生变化需要执行 render 和 update

在这个生命周期中 形参，参数得到的值时最新的props值
> 形参 a=>代表修改的值（注意是修改后的值） ，此时再根据判断是否要更新 render 渲染函数

```js
import React from "react"
class Box extends React.Component {
  shouldComponentUpdate(data) {
    console.log(data) //表示修改后的值
    console.log(this.props) //之前的props是还没发生变化的
    if (data.index === data.current) return true
    return false
  }
  render() {
    return (
      <div
        style={{
          border: "1px solid red",
          width: "200px",
          height: "200px",
          textAlign: "center",
          background: this.props.current === this.props.index ? "red" : "",
        }}
      >
        <span>{this.props.children}</span>
      </div>
    )
  }
}
class WWTest extends React.Component {
  constructor() {
    super()
    this.state = {
      books: [
        "一句顶万句",
        "草丛生睦邻",
        "独侠人",
        "一句顶万句1",
        "草丛生睦邻2",
        "独侠人3",
      ],
      item: {
        text: "<span>innerthml</span>",
      },
      current: -1,
    }
  }

  render() {
    return (
      <div>
        <input
          onChange={(event) => {
            this.setState({ current: event.target.value })
          }}
        ></input>
        {this.state.current}
        <div>
          <ul style={{ display: "flex" }}>
            {this.state.books.map((val, index) => {
              return (
                <Box
                  current={Number(this.state.current)}
                  index={index}
                  key={val}
                >
                  {val}
                </Box>
              )
            })}
          </ul>
        </div>
      </div>
    )
  }
}

export default WWTest
```


注意⚠️：当你修改响应state的值时，最终修改后的 值 只能可以在 render中和生命周期`componentDidUpdate`中取到

7.UNSAFE_componentWillUpdate 当父组件传过来的props时发生变化时触发。

8.UNSAFE_componentWillReceiveProps 当父组件把prop的值传过来的时候，就会立即执行一次，等到props 值发生变化就会再次执行

9.getDerivedStateFromProps 父组件中传入子组件中，一开始就会执行，等到state。

10.getSnapshotBeforeUpdate 该生命周期是render与update之中的执行的 ，可以在该声明周期中获取旧的dom

11.优化生命周期
shouldComponentUpdate和PureComponent （state如果一定变化就不用这个生命周期，shallowUpdate也要花时间）
12.