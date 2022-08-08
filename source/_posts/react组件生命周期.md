---
title: React生命周期
date: 2020-01-01 00:00:00
category: "React"
tags: React
---

# 类组件生命钩子：
1. componentWillMount

2. render

3. componentDidMount （像better scroll 需要dom成型）

>注意⚠️ 不要在render进行修改状态state中的属性，会有错误 （无限循环调用）


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

