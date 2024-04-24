---
title: React 源码调试（1）
date: 2024-04-24 11:25:30
tags: [React, Debug]
---

## 1. 基础环境准备

**源码下载**

```bash
git clone https://github.com/facebook/react.git
```

**依赖安装 & 选择版本**

```bash
cd react
git checkout v18.2.0
yarn install
```

## 2. 源码编译

查看 `package.json` 文件相关命令：

```json
{
  "scripts": {
    "build": "node ./scripts/rollup/build.js"
  }
}
```

## 2.1 build

### 2.2.1 编译出 sourcemap 文件

修改方法：

```js
function getRollupOutputOptions(
  outputPath,
  format,
  globals,
  globalName,
  bundleType
) {
  const isProduction = isProductionBundleType(bundleType)

  return {
    file: outputPath,
    format,
    globals,
    freeze: !isProduction,
    interop: false,
    name: globalName,
    sourcemap: true,
    esModule: false,
  }
}
```

修改上述代码中的 `sourcemap` 配置为 `true`。

### 2.2.2 注释部分插件

由于开启了 `sourcemap` 配置，在执行 `npm run build` 时，会出现 `Sourcemap is likely to be incorrect` 错误，只是由于[使用了一个或多个转换代码的插件，而没有生成转换所需的 `sourcemap`](https://cn.rollupjs.org/troubleshooting/#warning-sourcemap-is-likely-to-be-incorrect) 导致。

**注释掉 `remove 'use strict'` 插件**

```js
// Remove 'use strict' from individual source files.
return [
  {
    transform(source) {
      return source.replace(/['"]use strict["']/g, '')
    },
  },
]
```

该插件主要用于替换代码中的 `use strict` 标识

**注释掉 `top-level-definitions` 和 `license-and-signature-header` 插件**

```js
return [
  // License and haste headers, top-level `if` blocks.
  {
    renderChunk(source) {
      return Wrappers.wrapBundle(
        source,
        bundleType,
        globalName,
        filename,
        moduleType,
        bundle.wrapWithModuleBoundaries
      )
    },
  },
]
```

该插件是添加一些头部的代码的，比如 `Lisence`，顶层定义等

**注释 `closure` 和 `prettier` 插件**

```js
return [
  // Apply dead code elimination and/or minification.
  isProduction &&
    closure(
      Object.assign({}, closureOptions, {
        // Don't let it create global variables in the browser.
        // https://github.com/facebook/react/issues/10909
        assume_function_wrapper: !isUMDBundle,
        renaming: !shouldStayReadable,
      })
    ),
  // Add the whitespace back if necessary.
  shouldStayReadable &&
    prettier({
      parser: 'babel',
      singleQuote: false,
      trailingComma: 'none',
      bracketSpacing: true,
    }),
]
```

该插件主要是是负责，生产环境压缩代码和 `prettier` 格式化代码等

## 3. 创建软链

```bash
cd build/node_modules/react
yarn link

cd build/node_modules/react-dom
yarn link
```

## 4. 链接本地

```bash
yarn link react

yarn link react-dom
```

这步完成之后，就可以调试 `react` 源码
