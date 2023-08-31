# Vue设计与实现

## [1. 权衡的艺术](../Vue%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.xmind)

### 1.1 命令式与声明式

- 命令式：关注过程
（代码将每一步过程描述出来了）
- 声明式：关注结果
（实现过程由框架完成，开发者不需要关心）

### 1.2 性能和可维护性
（声明式代码性能不优于命令式的）

- 命令式：直接修改必要的地方

- 声明式：比对差异 + 修改

	- 声明式代码优化：差异性能消耗最小化

### 1.3 虚拟DOM

- 需要考虑：兼顾性能和可维护性

	- 创建页面

		- innerHTML性能：
HTML字符串拼接计算量（JS层面） + innerHTML的DOM计算量（DOM层面）
		- 虚拟DOM性能：
创建JS对象计算量（JS层面） + 创建真实的DOM计算量（DOM层面）

	- 更新页面

		- innerHTML：重新构建HTML字符串 + 重新设置DOM元素的innerHTML

			- 性能和页面大小有关

		- 虚拟DOM：重新创建JS对象（虚拟DOM树） + 比较差异 + 更新DOM

			- 性能和更新的元素有关

### 1.4 运行时和编译时

- 运行时：用户提供数据，手动实现渲染过程

  ```js
  // 假设VDOM格式如下：
  const obj = {
    tag: 'div',
    children: [...]
  }
  
  // Render：
  function Render (obj, root) {
    const el = document.createElement(obj.tag) // 创建DOM元素
  
    if (typeof obj.children === 'string') { // string类型为文本节点
      const text = document.createTextNode(obj.children)
      el.appendChild(text)
    } else if (obj.children) { // 否则,递归调用Render,将子节点添加到el上
      obj.children.forEach(child => Render(child, el))
    }
    root.appendChild(el) // 将元素添加到root上
  }
  
  Render(obj, root) // 调用Render接收VDOM对象,和父容器,渲染DOM元素
  ```

	- 无编译过程，无法分析用户提供的内容

- 运行时 + 编译时：实现部分自动编译过程

  ```js
  // Render：
  function Render (obj, root) {
    const el = document.createElement(obj.tag) // 创建DOM元素
  
    if (typeof obj.children === 'string') { // string类型为文本节点
      const text = document.createTextNode(obj.children)
      el.appendChild(text)
    } else if (obj.children) { // 否则，递归调用Render，将子节点添加到el上
      obj.children.forEach(child => Render(child, el))
    }
    root.appendChild(el) // 将元素添加到root上
  }
  
  const html = `<div>...</div>` // HTML字符串
  
  // Compiler()将html字符串转换为VDOM树格式
  const obj = Compiler(html)
  
  Render(obj, root) // 调用Render接收VDOM对象，和父容器，渲染DOM元素
  
  // 上述代码是运行时编译，即代码运行时才开始编译 -- 消耗性能
  
  // 优化：在构建时编译好必要的内容，运行时则无需编译了
  ```

	- 编译时可以分析内容，标记可能会更改的内容，做优化

- 编译时：用户提供数据，编译器直接将数据编译成命令式代码

  ```js
  <div>...</div>
  
  // 经编译器编译后：
  // ...得到结果：
  const div = document.createElement('div')
  // ...
  document.body.appendChild(div)
  ```
  

	- 编译时可以做优化，性能高，
  - 但是用户提供的内容必须编译后使用，不够灵活
