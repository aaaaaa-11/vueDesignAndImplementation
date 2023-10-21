# Vue设计与实现

## [3. Vue.js 3的设计思路](../Vue%E8%AE%BE%E8%AE%A1%E4%B8%8E%E5%AE%9E%E7%8E%B0.xmind)

### 3.1 声明式地描述UI

Vue3是一个声明式UI框架，设计思路：

1. 涉及内容：
   - DOM元素
   - 属性
   - 事件
   - 层级结构
2. 声明式地描述
   - DOM元素：与HTML标签一致
   - 属性：与HTML标签属性一致
   - 绑定动态属性：v-bind / :
   - 绑定事件：v-on / @
   - 层级结构：与HTML标签嵌套一致

  （用户不用写命令式的代码）

声明式描述UI：

1. **模版**方式
   ```js
   <h1 onClick="handler">...</h1>
   ```
2. 对象描述
   ```js
   const title = {
    tag: 'h1', // 标签
    props: { // 标签属性
      onClick: handler
    },
    children: [...] // 子节点
   }
   ```


特点：对象描述更灵活

```js
// 对象描述
let level = 3
const title = {
  tag: `h${level}` // 动态描述标签
}

// 模版描述，需要穷举
<h1 v-if="level===1"></h1>
<h2 v-else-if="level===2"></h2>
<h3 v-else-if="level===3"></h3>
<h4 v-else-if="level===4"></h4>
...
```

**虚拟DOM**

虚拟DOM就是用JS对象的方式来描述UI（真实的DOM）

```js
// vue提供的渲染函数就是虚拟DOM来描述UI的
import { h } from 'vue'

export default {
  render () {
    return h('h1', {
      onClick: handler
    })
  }
}

// 等价于：
export default {
  render () {
    return {
      tag: 'h1',
      props: {
        onClick: handler
      }
    }
  }
}
```

注：渲染函数用来描述组件的渲染内容，vue组件的render函数返回值就是该组件的虚拟DOM

### 渲染器

**渲染器的作用：将虚拟DOM渲染为真实的DOM**

一个渲染器demo：
```js
/*
vnode格式：
{
  tag: string,
  props: {...},
  children: string || vnode[]
}
 */
function renderer (vnode, container) {
  const el = document.createElement(vnode.tag); // 根据虚拟DOM的tag创建标签

  // 遍历props处理属性和事件
  for (const key in vnode.props) {
    if (/^on/.test(key)) { // 以on开头的key表示事件
      el.addEventListener(
        key.substr(2).toLowerCase(),
        vnode.props[key]
      )
    }
  }

  // 处理子节点
  if (typeof vnode.children === 'string') { // 文本节点
    el.appendChild(document.createTextNode(vnode.children))
  } else if (Array.isArray(vnode.children)) {
    // 递归调用renderer渲染子节点，并挂到创建的el下
    vnode.children.forEach(child => renderer(child, el))
  }

  // 将创建的标签挂到容器上
  container.appendChild(el);
}
```

渲染器实现：
1. 创建元素
2. 为元素添加属性，绑定事件
3. 处理子节点：字符串则直接创建文本节点挂到元素上；数组则递归调用渲染函数渲染子节点内容，并挂到元素上
4. 将创建的元素挂到挂载点下


更新元素时，渲染器只要精准地找到vnode的变更点并且只更新变更的地方。

### 组件的本质

**组件就是一组DOM元素的封装**，这组DOM元素就是组件要渲染的内容

1. 可以用函数的方式描述组件，函数返回值就是要渲染的虚拟DOM：
```js
const MyComponent = function () {
  return {
    tag: 'div',
    props: {},
    children: 'click'
  }
}
```

用虚拟DOM来描述组件，并传给renderer渲染：
```js
const vnode = {
  tag: MyComponent
}

function renderer(vnode, container) {
  if (typeof vnode.tag === 'string') { // 标签是字符串，表示HTML元素
    mountElement(vnode, container)
  } else if (typeof vnode.tag === 'function') { // 用函数描述的组件
    mountComponent(vnode, container)
  }
}

// 渲染组件
function mountComponent (vnode, container) {
  // 执行组件函数，就可以返回要渲染的虚拟DOM
  const subtree = vnode.tag()
  // 将拿到的虚拟DOM传给renderer渲染出真实的DOM
  renderer(subtree, container)
}
```

2. 用JS对象的方式描述组件

```js
const MyComponent = {
  render () {
    return {
      tag: 'div',
      props: {},
      children: []
    }
  }
}

// 改写renderer
function renderer (vnode, container) {
  if (typeof vnode.tag === 'string') {
    // ...
  } else if (typeof vnode.tag === 'object') { // vnode.tag此时是一个对象
    mountComponent(vnode, container)
  }
}

// 改写mountComponent
function mountComponent (vnode, container) {
  // 执行tag.render()函数拿到要渲染的虚拟DOM
  const subtree = vnode.tag.render()
  renderer(subtree, container)
}
```

### 模版的工作原理

**编译器**：将模板编译为渲染函数

```js
<div @click="handler">click me</div>
```
经过编译器编译后：
```js
render () {
  return h('div', { onClick: handler }, 'click me')
}
```

模板的工作原理：

如果使用模板，则经过编译器编译为渲染函数，用户也可以手写渲染函数，组件要渲染的内容最终都是通过渲染函数产生的，然后渲染器将渲染函数返回的虚拟DOM渲染为真实的DOM

### Vue.js是由各个模块组成的有机整体

Vue中组件的实现依赖渲染器，模板的编译依赖编译器，编译后的代码由渲染器和虚拟DOM决定。

例如：
```js
// 模板
<div id="foo" :class="cls"></div>


// 编译后
render () {
  // return h('div', { id: 'foo', class: cls });
  return {
    tag: 'div',
    props: {
      id: 'foo', // 字符串
      class: cls, // 变量
    }
  }
}
```

`class: cls`表示class绑定了一个动态值，内容可能发生变化。

对于渲染器来说，更新内容时，“寻找”变更点需要消耗一定的性能，而编译器可以告知哪些内容可能会改变，哪些不会变：

```js
render () {
  return {
    tag: 'div',
    props: {
      id: 'foo',
      class: cls
    },
    patchFlags: 1 // 标记动态内容，1表示class是动态的
  }
}
```

编译器可以在编译过程中，给动态内容添加patchFlags，渲染器可以根据这个属性找到变更点，减少工作量，提升性能。

### 总结
Vue.js是一个声明式的UI框架，需要设计DOM元素、属性、事件、子节点的描述方式，它直接描述结果，用户不需要关注过程。

Vue支持两种方式描述UI：
- 模版：更直观
- 虚拟DOM：更灵活

渲染器：用来将虚拟DOM渲染成真实的DOM，通过递归遍历虚拟DOM对象，调用原生DOM API来实现渲染，另外还可以通过Diff算法提升性能。

组件就是一组DOM元素的封装，可以是一个函数，返回要渲染的虚拟DOM，也可以是一个虚拟DOM，通过render返回要渲染的内容。渲染器可以通过调用函数拿到要渲染的虚拟DOM，然后递归调用renderer渲染。

Vue模板通过编译器编译为渲染函数，编译器、渲染函数等相互配合，可提升框架性能。

