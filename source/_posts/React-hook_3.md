---
title: React-hook（3）
date: 2024-04-11 17:36:37
tags: React
---

## 1. Fiber mount 阶段

`useEffect` 对应源码 `mountEffect`，`useLayoutEffect` 对应源码 `mountLayoutEffect`。

### 1.1 mountEffect

```js
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  return mountEffectImpl(
    UpdateEffect | PassiveEffect, // fiberFlags
    HookPassive, // hookFlags
    create,
    deps
  )
}
```

### 1.2 mountLayoutEffect

```js
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<mixed> | void | null
): void {
  return mountEffectImpl(
    UpdateEffect, // fiberFlags
    HookLayout, // hookFlags
    create,
    deps
  )
}
```

### 1.3 mountEffectImpl

```js
function mountEffectImpl(fiberFlags, hookFlags, create, deps): void {
  // step1: 创建 hook
  const hook = mountWorkInProgressHook()
  const nextDeps = deps === undefined ? null : deps
  // step2: 设置 workInProgress 的副作用标记
  currentlyRenderingFiber.flags |= fiberFlags // fiberFlags 被标记到 workInProgress
  // step3: 创建 Effect, 挂载到 hook.memoizedState上
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags, // hookFlags 用于创建 effect
    create,
    undefined,
    nextDeps
  )
}
```

逻辑如下：

1. 创建 hook
2. 设置 `workInProgress` 的副作用标记：`flags |= fiberFlags`
3. 创建 `effect`（在 `pushEffect` 中）， 挂载到 `hook.memoizedState` 上, 即 `hook.memoizedState = effect`
   - **注意：** 状态 Hook 中 `hook.memoizedState = state`

### 1.4 pushEffect

```js
function pushEffect(tag, create, destroy, deps) {
  // step1: 创建 effect 对象
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  }
  // step2: 把 effect 对象添加到环形链表末尾
  let componentUpdateQueue: null | FunctionComponentUpdateQueue =
    (currentlyRenderingFiber.updateQueue: any)
  if (componentUpdateQueue === null) {
    // 新建 workInProgress.updateQueue 用于挂载 effect 对象
    componentUpdateQueue = createFunctionComponentUpdateQueue()
    currentlyRenderingFiber.updateQueue = (componentUpdateQueue: any)
    // updateQueue.lastEffect 是一个环形链表
    componentUpdateQueue.lastEffect = effect.next = effect
  } else {
    const lastEffect = componentUpdateQueue.lastEffect
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect
    } else {
      const firstEffect = lastEffect.next
      lastEffect.next = effect
      effect.next = firstEffect
      componentUpdateQueue.lastEffect = effect
    }
  }
  // step3: 返回 effect
  return effect
}
```

逻辑如下：

1. 创建 `effect`。
2. 把 `effect` 对象添加到环形链表末尾。
3. 返回 `effect`。

```ts
export type Effect = {
  tag: HookFlags
  create: () => (() => void) | void
  destroy: (() => void) | void
  deps: Array<mixed> | null
  next: Effect
}
```

结果说明：

1. **effect.tag**：使用位掩码形式，代表 `effect` 的类型
   ```js
   export const NoFlags = /*   */ 0b000
   export const HasEffect = /* */ 0b001 // 有副作用, 可以被触发
   export const Layout = /*    */ 0b010 // Layout, dom 突变后同步触发
   export const Passive = /*   */ 0b100 // Passive, dom 突变前异步触发
   ```
2. **effect.create**：实际上就是通过 `useEffect()` 所传入的函数
3. **effect.deps**：依赖项，如果依赖项变动，会创建新的 `effect`

`renderWithHooks` 执行完成后，我们可以画出 `fiber`，`hook`，`effect` 三者的引用关系，如下：

![](../images/react-hook-15.png)

`workInProgress.flags` 被打上了标记，最后会在 fiber 树渲染阶段的 `commitRoot` 函数中处理
