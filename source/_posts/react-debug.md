---
title: React 源码调试（1）
date: 2024-04-24 11:25:30
tags: [React, Debug]
---

## 1. 基础环境准备

**源码下载：**

```bash
git clone https://github.com/facebook/react.git
```

**依赖安装：**

```bash
cd react
yarn install
```

## 2. 源码编译

查看 `package.json` 文件相关命令：

```json
{
  "scripts": {
    "build": "node ./scripts/rollup/build-all-release-channels.js"
  }
}
```

### 2.1 build-all-release-channels

```js
function buildForChannel(channel, nodeTotal, nodeIndex) {
  const { status } = spawnSync(
    'node',
    ['./scripts/rollup/build.js', ...process.argv.slice(2)],
    {
      stdio: ['pipe', process.stdout, process.stderr],
      env: {
        ...process.env,
        RELEASE_CHANNEL: channel,
        CIRCLE_NODE_TOTAL: nodeTotal,
        CIRCLE_NODE_INDEX: nodeIndex,
      },
    }
  )

  if (status !== 0) {
    // Error of spawned process is already piped to this stderr
    process.exit(status)
  }
}
```

**上述代码的含义**：执行 `./scripts/rollup/build.js` 文件

> [`child_process.spawnSync(command[, args][, options])`](https://nodejs.cn/api/child_process.html#child_processspawnsynccommand-args-options)
>
> - `command` `<string>` 要运行的命令。
> - `args` `<string[]>` 字符串参数列表。
> - `options` `<Object>`
>   - `stdio` `<string> | <Array>` 子进程的标准输入输出配置。默认值：`pipe`。
>   - `env` `<Object>` 环境变量键值对。默认值：`process.env`。

## 2.2 build

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
    interop: getRollupInteropValue,
    name: globalName,
    sourcemap: true,
    esModule: false,
    exports: 'auto',
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
    name: "remove 'use strict'",
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
  {
    name: 'top-level-definitions',
    renderChunk(source) {
      return Wrappers.wrapWithTopLevelDefinitions(
        source,
        bundleType,
        globalName,
        filename,
        moduleType,
        bundle.wrapWithModuleBoundaries
      )
    },
  },
  {
    name: 'license-and-signature-header',
    renderChunk(source) {
      return Wrappers.wrapWithLicenseHeader(
        source,
        bundleType,
        globalName,
        filename,
        moduleType
      )
    },
  },
]
```

该插件是添加一些头部的代码的，比如 `Lisence`，顶层定义等

**注释 `closure`、`prettier` 和 `sizes` 插件**

```js
return [
   needsMinifiedByClosure &&
     closure({
       compilation_level: 'SIMPLE',
       language_in: 'ECMASCRIPT_2020',
       language_out:
         bundleType === NODE_ES2015
           ? 'ECMASCRIPT_2020'
           : bundleType === BROWSER_SCRIPT
           ? 'ECMASCRIPT5'
           : 'ECMASCRIPT5_STRICT',
       emit_use_strict:
         bundleType !== BROWSER_SCRIPT &&
         bundleType !== ESM_PROD &&
         bundleType !== ESM_DEV,
       env: 'CUSTOM',
       warning_level: 'QUIET',
       source_map_include_content: true,
       use_types_for_optimization: false,
       process_common_js_modules: false,
       rewrite_polyfills: false,
       inject_libraries: false,
       allow_dynamic_import: true,
       assume_function_wrapper: true,
       renaming: false,
     }),
   needsMinifiedByClosure &&
      // Add the whitespace back
     prettier({
       parser: 'flow',
       singleQuote: false,
       trailingComma: 'none',
       bracketSpacing: true,
     }),
   Record bundle size.
   sizes({
     getSize: (size, gzip) => {
       const currentSizes = Stats.currentBuildResults.bundleSizes;
       const recordIndex = currentSizes.findIndex(
         record =>
           record.filename === filename && record.bundleType === bundleType
       );
       const index = recordIndex !== -1 ? recordIndex : currentSizes.length;
       currentSizes[index] = {
         filename,
         bundleType,
         packageName,
         size,
         gzip,
       };
     },
   }),
]
```

该插件主要是是负责，生产环境压缩代码、prettier 格式化代码和打包后文件大小计算等

## 3. 创建软链

```bash
cd build/oss-stable/react
yarn link

cd build/oss-stable/react-dom
yarn link
```

完成软链之后，就可以正常本地调试了
