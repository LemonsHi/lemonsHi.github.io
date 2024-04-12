---
title: （wip）React-hook（3）
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

### 1.5 回调处理

完成 `fiber` 树构造后，逻辑会进入 **渲染阶段**。在 `commitRootImpl` 函数中，整个渲染过程被 3 个函数分布实现：

1. `commitBeforeMutationEffects`
2. `commitMutationEffects`
3. `commitLayoutEffects`

这 3 个函数会处理 `fiber.flags`，也会根据情况处理 `fiber.updateQueue.lastEffect`

#### 1.5.1 commitBeforeMutationEffects

```js
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    // ...省略无关代码, 只保留Hook相关

    // 处理`Passive`标记
    const flags = nextEffect.flags
    if ((flags & Passive) !== NoFlags) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects()
          return null
        })
      }
    }
    nextEffect = nextEffect.nextEffect
  }
}
```

> **注意**：由于 `flushPassiveEffects` 被包裹在 `scheduleCallback` 回调中，由调度中心来处理，且参数是 `NormalSchedulerPriority`，故这是一个异步回调

#### 1.5.2 commitMutationEffects

```js
function commitMutationEffects(
  root: FiberRoot,
  renderPriorityLevel: ReactPriorityLevel
) {
  // ...省略无关代码, 只保留Hook相关
  while (nextEffect !== null) {
    const flags = nextEffect.flags
    const primaryFlags = flags & (Placement | Update | Deletion | Hydrating)
    switch (primaryFlags) {
      case Update: {
        // useEffect, useLayoutEffect 都会设置 Update 标记
        // 更新节点
        const current = nextEffect.alternate
        commitWork(current, nextEffect)
        break
      }
    }
    nextEffect = nextEffect.nextEffect
  }
}

function commitWork(current: Fiber | null, finishedWork: Fiber): void {
  // ...省略无关代码, 只保留Hook相关
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent:
    case Block: {
      // 在突变阶段调用销毁函数, 保证所有的 effect.destroy 函数都会在 effect.create 之前执行
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork)
      return
    }
  }
}

// 依次执行: effect.destroy
function commitHookEffectListUnmount(tag: number, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    (finishedWork.updateQueue: any)
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next
    let effect = firstEffect
    do {
      if ((effect.tag & tag) === tag) {
        // 根据传入的 tag 过滤 effect 链表.
        const destroy = effect.destroy
        effect.destroy = undefined
        if (destroy !== undefined) {
          destroy()
        }
      }
      effect = effect.next
    } while (effect !== firstEffect)
  }
}
```

> 调用关系：`commitMutationEffects` -> `commitWork` -> `commitHookEffectListUnmount`
>
> - 注意在调用 `commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork)` 时，参数是 `HookLayout | HookHasEffect`
> - 而 `HookLayout | HookHasEffect` 是通过 `useLayoutEffect` 创建的 `effect`。所以 `commitHookEffectListUnmount` 函数只能处理由 `useLayoutEffect()` 创建的 `effect`
> - 同步调用 `effect.destroy()`
