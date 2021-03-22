---
layout:     post
title:      "Show modal after deselecting specified items for ant design select component"
subtitle:   ""
date:       2021-03-22 19:00:00
author:     "Echo Yuan"
tags:
    - ant-design-select-component
---

正在做的项目里，有一个东西(记为label)会被A、B、C三个实体分别引用，A、B、C从前往后是包含与被包含的关系，且B和C引用的label是A所引用的label的子集。

现在要求，解除A和某个label的引用时，要遍历一下B和C看是否也同时引用了这个label，如果引用了就必须先解除它们之间的引用后才能再解除A和此label的引用。

前端用的是[ant-design-components-select-cn](https://ant.design/components/select-cn/)的组件来渲染的UI，说这个功能不好做，最主要还是没时间。于是，我这个不会写前端的小白就想担起这个研究的小任务。

最开始是在google上搜索了一番，还真找到了一个解决方案：[https://stackblitz.com/edit/react-b4v6uw](https://stackblitz.com/edit/react-b4v6uw) 。

闲暇之余，又翻了下这个select组件的例子，在 [隐藏已选择选项](https://ant.design/components/select-cn/#components-select-demo-hide-selected) 这个例子里受到了点儿启发，觉得用类似这样的方式会更合适。

于是动起手来，最终的主要代码如下：

```javascript
import React from "react";
import ReactDOM from "react-dom";
import "antd/dist/antd.css";
import "./index.css";
import { Select } from "antd";
import { Modal, Button, Space } from "antd";

const OPTIONS = [
  "Apples",
  "Nails",
  "Bananas",
  "Helicopters",
  "Let's warn",
  "Issue warning"
];

const WARNING_OPTIONS = ["Let's warn", "Issue warning"];

function warning(value) {
  Modal.warning({
    title: "Warning!",
    content: (
      <div>
        <p>The following reference "{value}", please unbind them first: </p>
        <p>Picture 1</p>
        <p>Picture 1 - HelloLabel</p>
        <p>Picture 1 - WorldLabel</p>
        <p>Picture 3</p>
      </div>
    ),
    onOk() {}
  });
}

class WarningIfNeedBeforeDeselectingOptions extends React.Component {
  state = {
    selectedItems: []
  };

  handleChange = selectedItems => {
    this.setState({ selectedItems });
  };
  handleDeselect = value => {
    const that = this;
    if (WARNING_OPTIONS.includes(value)) {
      console.log(that.state.selectedItems);
      this.setState(prevState => ({
        selectedItems: that.state.selectedItems
      }));
      warning(value);
    }
  };

  render() {
    const { selectedItems } = this.state;
    const filteredOptions = OPTIONS.filter(o => !selectedItems.includes(o));
    return (
      <Select
        mode="multiple"
        value={selectedItems}
        onChange={this.handleChange}
        onDeselect={this.handleDeselect}
        style={{ width: "100%" }}
      >
        {filteredOptions.map(item => (
          <Select.Option key={item} value={item}>
            {item}
          </Select.Option>
        ))}
      </Select>
    );
  }
}

ReactDOM.render(
  <WarningIfNeedBeforeDeselectingOptions />,
  document.getElementById("container")
);
```

效果如图：

![warning-when-deselect](/img/in-post/ant-design-components/deselect-warning.png)

在线体验请点击[这里](https://react-r3hajh-rubbvr.stackblitz.io/) ，但不保证一直有效。