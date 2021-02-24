# Vue 初始化之 initRender

在 initEvents 的下面，执行的是 initRender 函数，该函数来自于 core/instance/render.js 文件，我们打开这个文件找到 initRender 函数，如下：

```js
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```

上面是 initRender 函数的全部代码，我们慢慢来看，首先在 Vue 实例对象上添加两个实例属性，即 _vnode 和 _staticTrees：

```js
vm._vnode = null // the root of the child tree
vm._staticTrees = null // v-once cached trees
```

并且这两个属性都被初始化为 null，它们会在合适的地方被赋值并使用，到时候我们再讲其作用，现在我们暂且不介绍这两个属性的作用，你只要知道这两句话仅仅是在当前实例对象上添加了两个属性就行了。

接着是这样一段代码：

```js
const options = vm.$options
const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
const renderContext = parentVnode && parentVnode.context
vm.$slots = resolveSlots(options._renderChildren, renderContext)
vm.$scopedSlots = emptyObject
```

上面这段代码从表面上看很复杂，可以明确地告诉大家，如果你看懂了上面这段代码就意味着你已经知道了 Vue 是如何解析并处理 slot 的了。由于上面这段代码涉及内部选项比较多如：options._parentVnode、options._renderChildren 甚至 parentVnode.context，这些内容牵扯的东西比较多，现在大家对 Vue 的储备还不够，所以我们会在本节的最后阶段补讲，那个时候相信大家理解起来要容易多了。

不讲归不讲，但是有一些事儿还是要讲清楚的，比如上面这段代码无论它处理的是什么内容，其结果都是在 Vue 当前实例对象上添加了三个实例属性：

```js
vm.$vnode
vm.$slots
vm.$scopedSlots
```

我们把这些属性都整理到 [Vue实例的设计](../附录/Vue%20实例的设计.md) 中。

再往下是这段代码：

```js
// bind the createElement fn to this instance
// so that we get proper render context inside it.
// args order: tag, data, children, normalizationType, alwaysNormalize
// internal version is used by render functions compiled from templates
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
// normalization is always applied for the public version, used in
// user-written render functions.
vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
```

这段代码在 Vue 实例对象上添加了两个方法：vm._c 和 `vm.$createElement`，这两个方法实际上是对内部函数 createElement 的包装。其中 `vm.$createElement` 相信手写过渲染函数的同学都比较熟悉，如下代码：

```js
render: function (createElement) {
  return createElement('h2', 'Title')
}
```

我们知道，渲染函数的第一个参数是 createElement 函数，该函数用来创建虚拟节点，实际上你也完全可以这么做：

```js
render: function () {
  return this.$createElement('h2', 'Title')
}
```

上面两段代码是完全等价的。而对于 `vm._c` 方法，则用于编译器根据模板字符串生成的渲染函数的。`vm._c` 和 `vm.$createElement` 的不同之处就在于调用 createElement 函数时传递的第六个参数不同，至于这么做的原因，我们放到后面讲解。有一点需要注意，即 `$createElement` 看上去像对外暴露的接口，但其实文档上并没有体现。

再往下，就是 initRender 函数的最后一段代码了：

```js
// $attrs & $listeners are exposed for easier HOC creation.
// they need to be reactive so that HOCs using them are always updated
const parentData = parentVnode && parentVnode.data

/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
  defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
    !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
  }, true)
  defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
    !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
  }, true)
} else {
  defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
  defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
}
```

上面的代码主要作用就是在 Vue 实例对象上定义两个属性：`vm.$attrs` 以及 `vm.$listeners`。这两个属性在 Vue 的文档中是有说明的，由于这两个属性的存在使得在 Vue 中创建高阶组件变得更容易，感兴趣的同学可以阅读 [探索Vue高阶组件](http://hcysun.me/2018/01/05/%E6%8E%A2%E7%B4%A2Vue%E9%AB%98%E9%98%B6%E7%BB%84%E4%BB%B6/)。

我们注意到，在为实例对象定义 `$attrs` 属性和 `$listeners` 属性时，使用了 defineReactive 函数，该函数的作用就是为一个对象定义响应式的属性，所以 `$attrs` 和 `$listeners` 这两个属性是响应式的，至于 defineReactive 函数的讲解，我们会放到 Vue 的响应系统中讲解。

另外，上面的代码中有一个对环境的判断，在非生产环境中调用 defineReactive 函数时传递的第四个参数是一个函数，实际上这个函数是一个自定义的 setter，这个 setter 会在你设置 `$attrs` 或 `$listeners` 属性时触发并执行。以 `$attrs` 属性为例，当你试图设置该属性时，会执行该函数：

```js
() => {
  !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
}
```

可以看到，当 `!isUpdatingChildComponent` 成立时，会提示你 `$attrs` 是只读属性，你不应该手动设置它的值。同样的，对于 `$listeners` 属性也做了这样的处理。

这里使用到了 isUpdatingChildComponent 变量，根据引用关系，该变量来自于 lifecycle.js 文件，打开 lifecycle.js 文件，可以发现有三个地方使用了这个变量：

```js
// 定义 isUpdatingChildComponent，并初始化为 false
export let isUpdatingChildComponent: boolean = false

// 省略中间代码 ...

export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = true
  }

  // 省略中间代码 ...

  // update $attrs and $listeners hash
  // these are also reactive so they may trigger child update if the child
  // used them during render
  vm.$attrs = parentVnode.data.attrs || emptyObject
  vm.$listeners = listeners || emptyObject

  // 省略中间代码 ...

  if (process.env.NODE_ENV !== 'production') {
    isUpdatingChildComponent = false
  }
}
```

上面代码是简化后的，可以发现 isUpdatingChildComponent 初始值为 false，只有当 updateChildComponent 函数开始执行的时候会被更新为 true，当 updateChildComponent 执行结束时又将 isUpdatingChildComponent 的值还原为 false，这是因为 updateChildComponent 函数需要更新实例对象的 `$attrs` 和 `$listeners` 属性，所以此时是不需要提示 `$attrs` 和 `$listeners` 是只读属性的。

最后，对于大家来讲，现在了解这些知识就足够了，至于 `$attrs` 和 `$listeners` 这两个属性的值到底是什么，等我们讲解虚拟DOM的时候再回来说明，这样大家更容易理解。
