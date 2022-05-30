# vite 是什么

用作者在微博上的原话：

> Vite，一个基于浏览器原生 ES imports 的开发服务器。利用浏览器去解析 imports，在服务器端按需编译返回，完全跳过了打包这个概念，服务器随起随用。同时不仅有 Vue 文件支持，还搞定了热更新，而且热更新的速度不会随着模块增多而变慢。针对生产环境则可以把同一份代码用 rollup 打。虽然现在还比较粗糙，但这个方向我觉得是有潜力的，做得好可以彻底解决改一行代码等半天热更新的问题。

## 一、[搭建第一个 Vite 项目](https://cn.vitejs.dev/guide/#scaffolding-your-first-vite-project)

## 二、诞生的背景

### 1. webpack 慢

#### 服务器启动慢，启动服务之前，需要将所有的模块先打包好

![webpack-bundler](https://cn.vitejs.dev/assets/bundler.37740380.png)

#### 热更新的速度会随着模块增多而变慢

当改动了一个模块后，把该模块作为依赖的宿主模块也会重新打包编译。

例如：c 模块依赖 a 和 b 两个模块，改动 a，会导致用到 a 的 c 也会重新编译，c 里面的 b 虽然没改也要重新编译，如果 c 依赖越多就会越来越慢

> 改善方案：dll 预构建、路由增量编译，配置持久缓存，`cache:{type:'filesystem'}`

### 2. [浏览器原生支持 esm](https://www.zhangxinxu.com/wordpress/2018/08/browser-native-es6-export-import-module/)

## 三、丝滑的开发体验

### 1. Vite 启动快，热更新快

![img](https://cn.vitejs.dev/assets/esm.3070012d.png)

Vite 通过在一开始将应用中的模块区分为 依赖 和 源码 两类，改进了开发服务器启动时间。

- esbuild 预构建第三方依赖，[为什么还需要预构建](https://cn.vitejs.dev/guide/dep-pre-bundling.html)
- 源码模块按需编译，返回也是 esm 格式代码，没有打包过程
- 当改动了一个模块后，大多数情况只需重新编译自身，所以热更新的速度不会随着模块增多而变慢。

### 2. [esbuild 使用](https://juejin.cn/post/7043777969051058183)

### 3. [访问 `localhost:3000/`，Vite 做了什么](https://juejin.cn/post/7045206518450356231#heading-8)

### 4. [热更新原理](https://juejin.cn/post/7047378914108243982#heading-8)

## 四、不够友好的生产环境

### 1. [Rollup 简介](https://www.rollupjs.com)

### 2. [为啥在生产又使用另外一个打包器 Rollup?](https://cn.vitejs.dev/guide/why.html#why-not-bundle-with-esbuild)

### 3. 浏览器兼容性如何解决？

- 默认情况下，Vite 只能支持 esm 的浏览器，只处理语法转译，且 默认不包含任何 api polyfill。[见说明](https://cn.vitejs.dev/guide/build.html#config-file-resolving)
- 设置`build:{target:'chrome58'}`，只是将 js 语法转换为 chrome58 兼容，比如 a?.b 这种语法，
  但是其中用到新的 api 并不会处理 polifill 垫片，比如`Object.fromEntries`方法
- 官方推荐使用 @vitejs/plugin-legacy，支持 api 是 和 esm chunk 的 polifill
- .browserslistrc/babelrc 等配置文件，对 build.target/legacy.target 不生效，需要手动填入、[见 issues](https://github.com/vitejs/vite/issues/2476)，但是 css 的 polifill 此文件生效，比如 postcss 里的`autoprefixer`

## 五、Vite 在实单求购移动端项目的实践

> pm-mobile 分支：chore/vite

### [改造 dev](http://git.zhaogangren.com/Front/pm-mobile/commit/a647852361fc73c77525fc212ae0177a9039f0c7?view=parallel)

### [改造 prod](http://git.zhaogangren.com/Front/pm-mobile/commit/c62fc363b31b0e590548486baff3cdce076a36e4?view=parallel)

```ts
//vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { resolve } from 'node:path'
import { createSvgIconsPlugin } from 'vite-plugin-svg-icons'
import legacy from '@vitejs/plugin-legacy'

// https://vitejs.dev/config/
export default defineConfig({
  server: {
    port: 9000
  },
  plugins: [
    react(),
    legacy({
      targets: ['> 1%', 'iOS >= 10', 'Android >= 6.0', 'last 2 versions'], // 必须设置否则会使用‘defaults’,这个插件不读.browserslistrc，见issue：https://github.com/vitejs/vite/issues/2476
      polyfills: [] // 假如第三方依赖有版本存在api兼容问题，可以加入缺少的polifill，或者想办法编译下第三方依赖
    }),
    createSvgIconsPlugin({
      iconDirs: [resolve('./src/assets/svg')],
      symbolId: '[name]',
      inject: 'body-first',
      customDomId: '__svg__icons__dom__'
    })
  ],
  resolve: {
    alias: {
      '@': resolve('./src'),
      'antd-mobile': 'antd-mobile/2x'
    }
  },
  css: {
    preprocessorOptions: {
      less: {
        modifyVars: {
          hack: `true; @import (reference) "${resolve('./src/assets/style/global.less')}";`
        },
        javascriptEnabled: true
      }
    }
  },
  build: {
    minify: 'terser' // 使用esbuild压缩出现问题 {'-512’:'xxxx'} => {-512:'xxxx'}，报错问题暂未解决
  }
})

/**
 * TODO:
 * production:
 * 1、资源设置publicPath、上传cdn等
 */
```
