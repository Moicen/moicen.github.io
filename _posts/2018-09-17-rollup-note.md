---
layout: post
title: 使用rollup.js打包文件
abstract: 简单记录下对rollup.js的使用
category: technical
permalink: rollup-note
author: 木逸辰
tags: [tech, js, rollup, pack]
---

### {{ page.title }}


> [rollup.js ](https://www.rollupjs.com)是一个 JavaScript 模块打包器，可以将小块代码编译成大块复杂的代码，例如 library 或应用程序。Rollup 对代码模块使用新的标准化格式，这些标准都包含在 JavaScript 的 ES6 版本中，而不是以前的特殊解决方案，如 CommonJS 和 AMD。ES6 模块可以使你自由、无缝地使用你最喜爱的 library 中那些最有用独立函数，而你的项目不必携带其他未使用的代码。ES6 模块最终还是要由浏览器原生实现，但当前 Rollup 可以使你提前体验。  

如上，Rollup.js的优势在于，可以直接打包使用ES6语法(`export`/`import`)引入的模块，而且可以智能分析被引入但未被使用的代码，在打包时会自动将这些代码过滤掉，减少代码体积。

如果要使用命令行进行打包，可以全局安装：
```js
npm install --global rollup
```
在项目中使用，可以本地安装：
```js
npm install rollup --save-dev
```

`rollup.js`有许多相关的插件，可以帮助处理打包过程中的需要，比如`rollup-plugin-json`插件可以读取到`package.json`文件中的内容（项目名、版本号等），方便进行版本号的输出。
ES6的命名导入在直接使用时存在问题，需要使用`rollup-plugin-node-resolve`来处理，`rollup-plugin-commonjs`可以处理`commonjs`模式的引入。
前端常用的压缩工具`uglify`也有响应的`rollup-plugin-uglify`插件，用于压缩打包后的文件。

`rollup.js`支持四种输出格式：
	* `cjs`：输出commonjs模式
	* `umd` ：输出`umd`模式
	* `iife`：输出IIFE函数
	* `es`：输出ES6格式

针对项目一般本地安装比较好，使用配置文件`rollup.config.js`描述打包配置：
```js
// rollup.config.js
import json from 'rollup-plugin-json';
import {uglify} from 'rollup-plugin-uglify'
import resolve from 'rollup-plugin-node-resolve';
import commonjs from 'rollup-plugin-commonjs';

const opt = (format) => ({
    // 入口文件
    input: 'src/core.js',
    output: {
        // 对压缩文件添加sourcemap便于调试
        sourcemap: "inline",
        file: `dist/target.${format}.js`,
        format: format
    },
    plugins: [
        json(),
        resolve(),
        commonjs()
    ]
})
const min = (format) => ({
    input: 'src/core.js',
    output: {
        sourcemap: "inline",
        file: `dist/target.${format}.min.js`,
        format: format
    },
    plugins: [
        json(),
        resolve(),
        commonjs(),
        uglify()
    ]
})
// 输出多个配置，rollup会自动为每个配置分别生成一个文件
export default [
    opt("cjs"), opt("umd"), opt("iife"), opt("es"),
    min("cjs"), min("umd"), min("iife"), min("es"),
];
```
在`core.js`中对其他文件的引用：
```js
import logger from "./lib/logger"
import _ from "./lib/underscore"
import config from "./config.js"
import utils from "./lib/utils"
```
