---
title: React-base（1）
date: 2024-04-16 14:08:15
tags: React
---

## 1. 包说明

### 1.1 react

`react` 基础包，**只提供定义 `react` 组件（`ReactElement`）的必要函数**，一般来说需要和渲染器（`react-dom`，`react-native`）一同使用。在编写 `react` 应用的代码时，大部分都是调用此包的 API

### 1.2 react-dom

`react` 渲染器之一，**是 `react` 与 web 平台连接的桥梁（可以在浏览器和 `nodejs` 环境中使用），将 `react-reconciler` 中的运行结果输出到 web 界面上**。在编写 `react` 应用的代码时，大多数场景下能用到此包的就是一个入口函数 `ReactDOM.render(<App/>, document.getElementById('root'))`，其余使用的 api，基本是 `react` 包提供的

### 1.3 react-reconciler

`react` 得以运行的核心包（综合协调 `react-dom`，`react`，`scheduler` 各包之间的调用与配合）。**管理 `react` 应用状态的输入和结果的输出。将输入信号最终转换成输出信号传递给渲染器**

- 接受输入（`scheduleUpdateOnFiber`），将 `fiber` 树生成逻辑封装到一个回调函数中（涉及 `fiber` 树形结构，`fiber.updateQueue` 队列，调和算法等）
- 把此回调函数（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）送入 `scheduler` 进行调度
- `scheduler` 会控制回调函数执行的时机，回调函数执行完成后得到全新的 `fiber` 树
- 再调用渲染器（如 `react-dom`, `react-native` 等），将 `fiber` 树形结构最终反映到界面上

### 1.4 scheduler

调度机制的核心实现，**控制由 `react-reconciler` 送入的回调函数的执行时机，在 `concurrent` 模式下可以实现任务分片**。在编写 `react` 应用的代码时，同样几乎不会直接用到此包提供的 api

- 核心任务就是执行回调（回调函数由 `react-reconciler` 提供）
- 通过控制回调函数的执行时机，来达到任务分片的目的，实现可中断渲染（`concurrent` 模式下才有此特性）

## 2. 包结构分类

### 2.1 分类

可将 `react` 应用整体结构分为接口层（`api`）和内核层（`core`）2 个部分

1.  接口层（`api`）：`react` 包，平时在开发过程中使用的绝大部分 `api` 均来自此包（不是所有）。 在 `react` 启动之后，正常可以改变渲染的基本操作有 3 个

    - `class` 组件中使用 `setState()`
    - `function` 组件里面使用 `hook`，并发起 `dispatchAction` 去改变 `hook` 对象
    - 改变 `context`（其实也需要 `setState` 或 `dispatchAction` 的辅助才能改变）
    - 以上 `setState` 和 `dispatchAction` 都由 `react` 包直接暴露。所以要想 `react` 工作，基本上是调用 `react` 包的 `api` 去与其他包进行交互

2.  内核层（`core`）

    - 调度器（`scheduler`），核心职责只有 1 个，就是执行回调

      - 把 `react-reconciler` 提供的回调函数，包装到一个任务对象中
      - 在内部维护一个任务队列，优先级高的排在最前面
      - 循环消费任务队列，直到队列清空

    - 构造器（`react-reconciler`），有 3 个核心职责
      - 装载渲染器，渲染器必须实现 `HostConfig` 协议（如：`react-dom`），保证在需要的时候，能够正确调用渲染器的 `api`，生成实际节点（如：`dom` 节点）
      - 接收 `react-dom` 包（初次 `render`）和 `react` 包（后续更新 `setState`）发起的更新请求
      - 将 `fiber` 树的构造过程包装在一个回调函数中，并将此回调函数传入到 `scheduler` 包等待调度
    - 渲染器（`react-dom`），有 2 个核心职责
      - 引导 `react` 应用的启动（通过 `ReactDOM.render`）
      - 实现 `HostConfig` 协议，能够将 `react-reconciler` 包构造出来的 `fiber` 树表现出来，生成 `dom` 节点（`CSR`），生成字符串（`SSR`）

> [`HostConfig` 协议](https://github.com/facebook/react/blob/v17.0.2/packages/react-reconciler/README.md#practical-examples)：本质上是一个对象，该对象描述了如何在宿主环境中实现某些操作

```js
const HostConfig = {
  createInstance(type, props) {
    // e.g. DOM renderer returns a DOM node
  },
  // ...
  supportsMutation: true, // it works by mutating nodes
  appendChild(parent, child) {
    // e.g. DOM renderer would call .appendChild() here
  },
  // ...
}
```

### 2.2 关系

![](../images/react-base-1.png)

> - **红色**方块代表入口函数，**绿色**方块代表出口函数
> - `package` 之间的调用脉络就是通过板块间的入口和出口函数连接起来的
