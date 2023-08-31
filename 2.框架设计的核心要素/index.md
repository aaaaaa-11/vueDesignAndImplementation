# Vue设计与实现

## [2. 框架设计的核心要素](../Vue%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.xmind)

### 2.1 提升用户的开发体验

- 友好的错误提示：指明原因，定位问题
- 直观的输出内容

### 2.2 控制框架代码体积

- 根据环境输出不同代码

  ```js
  // __DEV__用来表示生产环境（true）还是开发环境（false）
  if (__DEV__ && !res) {
    // 开发环境下不会执行下面的代码
    warn(
      `Failed to mount app: mount target selector "${container}" returned null.`
    )
  }
  ```

### 2.3 良好的Tree-Shaking

- 条件：ESM，因为Tree-Shaking依赖ESM的静态结构

  ```js
  // rollup.js
  
  // 假设已经安装了rollup.js
  // input.js
  import { foo } from './utils.js'
  foo()
  
  // utils.js
  export function foo(obj) { obj && obj.foo }
  
  export function bar(obj) { obj && obj.foo }
  
  // 构建
  npx rollup input.js -f esm -o bundle.js // 入口：input.js，输出：bundle.js，模块为ESM
  
  // 输出的bundle.js
  function foo(obj) { obj && obj.foo }
  foo();
  
  // 结论
  // 由于Teee-Shaking，而没有在构建文件中包含bar函数
  ```

- 副作用：如果函数调用会产生副作用，则不能被移除
- 合理使用/*#__PURE__*/

  ```js
  // input.js
  import { foo } from './utils'
  
  /*#__PURE__*/ foo()
  
  // 以input.js为入口，输出的代码为空
  // 因为这里的foo()读取obj的值并没有什么意义，可以删掉
  ```

	- 通常产生副作用的代码都是模块内函数的顶级调用

    Vue3源码中，基本上都是在顶级调用函数上使用/*#___PURE__*/

	  ```js
	  foo() // 顶级调用，foo调用可能对外部产生影响

	  function bar () { // 只要bar不调用，内部的foo就不会调用
	    foo() // 函数内调用
	  }
	  ```

### 2.4 框架输出的构建产物

- 需求

	- \<script\>中直接引用

		- 输出IIFE格式的资源

	- 直接引入ESM格式的资源

		- 输出ESM格式的资源

	- 服务端渲染：node.js中用require引入

		- 输出cjs的资源

### 2.5 特性开关

- 机制：Tree-Shaking
- 优点：灵活性、打包体积优化

	- 开关特性不担心资源体积变大
	- 升级框架时，开关特性支持遗留API，是否使用由用户选择

- 实现：预定义插件

  ```js
  // rollup.js配置
  {
    ___FEATURE_OPTIONS_API__: isBundlerESMBuild ? `__VUE_OPTIONS_API__`: true // ___FEATURE_OPTIONS_API__类似__DEV__
  }
  
  // webpack中使用webpack.DefinePlugin
  new webpack.DefinePlugin({
    __VUE_OPTIONS_API__: JSON.stringify(true) // 开启特性
  })
  
  // __VUE_OPTIONS_API__开启后允许用户使用选项式API
  ```

### 2.6 错误处理

- 用户自行处理

	- 增加用户心智负担

- 框架统一处理错误，并为用户提供统一处理错误的接口

  ```js
  // 编写utils.js模块时处理错误：
  export default {
    foo(fn) {
      try {...} catch (e) {...}
    },
    bar(fn) {
      try { ... } catch(e) { ... }
    }
  }
  
  // 每个内部函数都用try..catch处理，所以可以封装成一个错误处理函数
  export default {
    foo(fn) {
      callWithErrorHandling(fn)
    },
    bar(fn) {
      callWithErrorHandling(fn)
    }
  }
  
  function callWithErrorHandling (fn) {
    try { ... } catch (e) { ... }
  }
  
  // 可以为用户提供统一处理错误的接口
  let handleError = null
  export default {
    foo (fn) {
      callWidthErrorHandling(fn)
    },
    registerErrorHandler(fn) { // 用户调用后注册统一错误处理函数
      handleError = fn
    }
  }
  
  function callWithErrorHandling (fn) {
    try { ... }
    catch (e) {
      handleError(e) // 可以把捕获到的错误传递给用户注册的错误处理程序
    }
  }
  ```

### 2.7 良好的TypeScript类型支持

- TS优点：代码即文档、编辑器自动提示、避免某些bug、提高可维护性、……
- 框架需要做大量的类型推导
- 对TSX支持
