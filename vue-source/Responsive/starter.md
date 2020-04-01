# Vue 数据响应系统开篇

前面章节介绍的都是 Vue 怎么实现数据渲染和组件化的，主要讲的是初始化的过程，把原始的数据最终映射到 DOM 中，但并没有涉及到数据变化到 DOM 变化的部分。而 Vue 的数据驱动除了数据渲染 DOM 之外，还有一个很重要的体现就是数据的变更会触发 DOM 的变化。

其实前端开发最重要的 2 个工作，一个是把数据渲染到页面，另一个是处理用户交互。Vue 把数据渲染到页面的能力我们已经通过源码分析出其中的原理了，但是由于一些用户交互或者是其它方面导致数据发生变化重新对页面渲染的原理我们还未分析。

可能很多小伙伴之前都了解过 Vue.js 实现响应式的核心是利用了 ES5 的 `Object.defineProperty`，这也是为什么 Vue.js 不能兼容 IE8 及以下浏览器的原因，我们先来对它有个直观的认识。

## Object.defineProperty

`Object.defineProperty` 方法会直接在一个对象上定义一个新属性，或者修改一个对象的现有属性， 并返回这个对象，先来看一下它的语法：

```js
Object.defineProperty(obj, prop, descriptor)
```

obj 是要在其上定义属性的对象；prop 是要定义或修改的属性的名称；descriptor 是将被定义或修改的属性描述符。

比较核心的是 descriptor，它有很多可选键值，具体的可以去参阅它的[文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。这里我们最关心的是 get 和 set，get 是一个给属性提供的 getter 方法，当我们访问了该属性的时候会触发 getter 方法；set 是一个给属性提供的 setter 方法，当我们对该属性做修改的时候会触发 setter 方法。

一旦对象拥有了 getter 和 setter，我们可以简单地把这个对象称为响应式对象。那么 Vue.js 把哪些对象变成了响应式对象了呢，接下来我们从源码层面分析。

## initState

在 Vue 的初始化阶段，`_init` 方法执行的时候，会执行 initState(vm) 方法，它的定义在 src/core/instance/state.js 中。

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

我们知道 initState 函数是很多选项初始化的汇总，在 initState 函数内部使用 initProps 函数初始化 props 属性；使用 initMethods 函数初始化 methods 属性；使用 initData 函数初始化 data 选项；使用 initComputed 函数和 initWatch 函数初始化 computed 和 watch 选项。那么我们从哪里开始讲起呢？这里我们决定以 initData 为切入点为大家讲解 Vue 的响应系统，因为 initData 几乎涉及了全部的数据响应相关的内容，这样将会让大家在理解 props、computed、watch 等选项时不费吹灰之力，且会有一种水到渠成的感觉。

话不多说，如下是 initState 函数中用于初始化 data 选项的代码：

```js
if (opts.data) {
  initData(vm)
} else {
  observe(vm._data = {}, true /* asRootData */)
}
```

首先判断 `opts.data` 是否存在，即 data 选项是否存在，如果存在则调用 `initData(vm)` 函数初始化 data 选项，否则通过 observe 函数观测一个空的对象，并且 `vm._data` 引用了该空对象。其中 observe 函数是将 data 转换成响应式数据的核心入口。

下面我们就从 `initData(vm)` 开始开启数据响应系统的探索之旅。
