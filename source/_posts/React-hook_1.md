---
title: React Hook（1）
date: 2024-04-01 20:58:30
tags: React
---

## 1. 背景

### 1.1 什么需要 Hook？

- **组件间的状态难以复用：** 在 hook 出现之前的状态复用有 Mixins、props 和 HOC（高阶组件）。其中，props、HOC（高阶组件）虽然能解决状态复用问题，但是组件会形成“嵌套地狱”。而 Minxins 存在以下问题：可能会相互依赖，相互耦合，不利于代码维护；不同的 Mixins 中的方法可能会相互冲突；Mixins 非常多时，组件是可以感知到的，甚至还要为其做相关处理，这样会给代码造成滚雪球的复杂性。
- **复杂组件难以理解：** classComponent 的生命周期中充斥各种状态逻辑和副作用，变成了“面向生命周期变成”。
- **难以理解的 class：** 除了代码复用和代码管理会遇到困难外，class 是学习 React 的一大屏障，必须去理解 JavaScript 中  this  的工作方式，这与其他语言存在巨大差异。还不能忘记绑定事件处理器。如果不使用  ES2022 public class fields，这些代码非常冗余。此外，class 也给目前的工具带来了一些问题，例如，组件预编译技术会在 class 中遇到优化失效的 case；class 不能很好的压缩；class 在热重载会出现不稳定的情况。

### 1.2 hook 是什么？

React 的特征是 UI=f(data)，一切皆组件、声明式编程。那么，既然是 UI=f(data)，data（数据）通过 function 来驱动 UI 视图变化。在业务中，你不能简单只展示，也需交互，交互就会更新状态，React 是通过 setState 来改变状态。但这仅限于类组件，所以在 Hooks 出现之前，函数式组件用来渲染组件（也称它为木偶组件），类组件用来控制状态。

React Hooks 是 React 16.8 推出的新特性。它可以让你在不编写 class 的情况下使用 state 以及其他的 React 特性

Hook 主要是为了解决以下问题：

- 无 class 的复杂性；
- 无生命周期的困扰；
- 优雅复用；
- 对齐 class 组件已经具备的能力。

> [Hook 会因为在渲染时创建函数而变慢吗？](https://zh-hans.legacy.reactjs.org/docs/hooks-faq.html#are-hooks-slow-because-of-creating-functions-in-render)
> 不会。在现代浏览器中，闭包和类的原始性能只有在极端场景下才会有明显的差别。
> 除此之外，可以认为 Hook 的设计在某些方面更加高效：
>
> - Hook 避免了 class 需要的额外开支，像是创建类实例和在构造函数中绑定事件处理器的成本。
> - 符合语言习惯的代码在使用 Hook 时不需要很深的组件树嵌套。这个现象在使用高阶组件、render props、和 context 的代码库中非常普遍。组件树小了，React 的工作量也随之减少。
>   传统上认为，在 React 中使用内联函数对性能的影响，与每次渲染都传递新的回调会如何破坏子组件的 shouldComponentUpdate 优化有关。Hook 从三个方面解决了这个问题。
> - useCallback Hook 允许你在重新渲染之间保持对相同的回调引用以使得 shouldComponentUpdate 继续工作：
> - useMemo Hook 使得控制具体子节点何时更新变得更容易，减少了对纯组件的需要。
> - 最后，useReducer Hook 减少了对深层传递回调的依赖，正如下面解释的那样。
