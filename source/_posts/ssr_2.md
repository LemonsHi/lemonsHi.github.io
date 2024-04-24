---
title: （wip）SSR 相关（2）
date: 2024-04-23 14:55:24
tags: [React, SSR]
---

## 1. 一个例子

组件如下：

```tsx
import React, { useState } from 'react'
import classNames from 'classnames'

import * as styles from './App.module.css'

export const App: React.FC = () => {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p style={{ color: 'red' }}>count: {count}</p>
      <div className={styles.btnWrapper}>
        <Button
          text="Increment"
          className={styles.btn}
          onClick={() => setCount((prev) => prev + 1)}
        />
        <Button
          text="Decrement"
          className={styles.btn}
          onClick={() => setCount((prev) => prev - 1)}
        />
        <Button
          text="Reset"
          className={styles.btn}
          onClick={() => setCount(0)}
        />
      </div>
    </div>
  )
}

export const Button: React.FC<IButtonProps> = ({
  text,
  className,
  onClick,
}) => {
  return (
    <button onClick={onClick} className={classNames(styles.btn, className)}>
      {text}
    </button>
  )
}
```

服务端渲染如下：

```tsx
import fs from 'fs'
import path from 'path'
import express from 'express'
import React from 'react'
import ReactDOMServer from 'react-dom/server'
import { App } from './App'

const clientDistDir = path.resolve(__dirname, '../dist/client')
const htmlPath = path.resolve(clientDistDir, 'index.html')

const app = express()

app.get('/', (req, res) => {
  // 读取 dist/client/index.html 文件
  const html = fs.readFileSync(htmlPath, 'utf-8')
  const app = ReactDOMServer.renderToString(<App />)
  // 将渲染后的 React HTML 插入到 div#root 中
  const finalHtml = html.replace(
    '<div id="root"></div>',
    `<div id="root">${app}</div>`
  )

  res.send(finalHtml)
})

// 提供静态资源访问服务
app.use(express.static(clientDistDir))

const PORT = process.env.PORT || 3007
app.listen(PORT, () => {
  console.log(`Server is listening on port ${PORT}`)
  console.log(`http://localhost:${PORT}`)
  console.log(`http://127.0.0.1:${PORT}`)
})
```

## 2. 服务渲染流程

### 2.1 服务返回

服务端返回的 `HTML`，如下：

```html
<!DOCTYPE html>
<html lang="zh-cn">
  <head>
    <meta charset="utf-8" />
    <meta
      name="viewport"
      content="width=device-width,initial-scale=1,user-scalable=no,minimum-scale=1,maximum-scale=1,minimal-ui,viewport-fit=cover"
    />
    <title></title>
    <script defer="defer" src="main.c0c45135.js"></script>
    <link href="assets/css/c0c45135.css" rel="stylesheet" />
  </head>
  <body>
    <div id="root">
      <div>
        <p style="color:red">
          count:
          <!-- -->
          0
        </p>
        <div class="cVChZBXm__xXjk7faER8">
          <button class="wEy8fdhueJs2LXJ5e4Td Ntm6Y_jZKK1UZ15YcDml">
            Increment
          </button>
          <button class="wEy8fdhueJs2LXJ5e4Td Ntm6Y_jZKK1UZ15YcDml">
            Decrement
          </button>
          <button class="wEy8fdhueJs2LXJ5e4Td Ntm6Y_jZKK1UZ15YcDml">
            Reset
          </button>
        </div>
      </div>
    </div>
  </body>
</html>
```

### 2.2 浏览器再次渲染

```tsx
import React from 'react'
import ReactDOM from 'react-dom'
import { App } from './App'

ReactDOM.hydrate(<App />, document.getElementById('root'))
```

> **注意**：这里执行的不是 `render` 的 `api`，而是 `hydrate` 的 `api`。
>
> 因为浏览器接收到 `HTML` 就会把它渲染出来，这时候已经有标签了，只需要把它和组件关联之后，就可以更新和绑定事件了（直接关联已有 `DOM` 标签，避免没必要的渲染）。

### 2.3 总结

整个 `SSR` 的流程，如下：

服务端渲染组件为 `string`，拼接成 `HTML` 返回，浏览器渲染出返回的 `HTML`，然后执行 `hydrate`，把渲染和已有的 `HTML` 标签关联。

## 3. renderToString

```js
function renderToString(
  children: ReactNodeList,
  options?: ServerOptions
): string {
  return renderToStringImpl(
    children,
    options,
    false
    // ...
  )
}
```

本质上调用的是 `renderToStringImpl` 方法

### 3.1 renderToStringImpl

```js
function renderToStringImpl(
  children: ReactNodeList,
  options: void | ServerOptions,
  generateStaticMarkup: boolean,
  abortReason: string
): string {
  let didFatal = false
  let fatalError = null
  let result = ''
  const destination = {
    // $FlowFixMe[missing-local-annot]
    push(chunk) {
      if (chunk !== null) {
        result += chunk
      }
      return true
    },
    // $FlowFixMe[missing-local-annot]
    destroy(error) {
      didFatal = true
      fatalError = error
    },
  }

  let readyToStream = false
  function onShellReady() {
    readyToStream = true
  }
  const resumableState = createResumableState(
    options ? options.identifierPrefix : undefined,
    undefined
  )
  const request = createRequest(
    children,
    resumableState,
    createRenderState(resumableState, generateStaticMarkup),
    createRootFormatContext(),
    Infinity,
    onError,
    undefined,
    onShellReady,
    undefined,
    undefined,
    undefined
  )
  startWork(request)
  // If anything suspended and is still pending, we'll abort it before writing.
  // That way we write only client-rendered boundaries from the start.
  abort(request, abortReason)
  startFlowing(request, destination)
  if (didFatal && fatalError !== abortReason) {
    throw fatalError
  }

  if (!readyToStream) {
    // Note: This error message is the one we use on the client. It doesn't
    // really make sense here. But this is the legacy server renderer, anyway.
    // We're going to delete it soon.
    throw new Error(
      'A component suspended while responding to synchronous input. This ' +
        'will cause the UI to be replaced with a loading indicator. To fix, ' +
        'updates that suspend should be wrapped with startTransition.'
    )
  }

  return result
}
```
