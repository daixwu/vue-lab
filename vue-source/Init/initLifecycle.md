# Vue 初始化之 initLifecycle

`_init` 函数在执行完 initProxy 之后，执行的就是 initLifecycle 函数：

```js
vm._self = vm
initLifecycle(vm)
```

在 initLifecycle 函数执行之前，执行了 `vm._self = vm` 语句，这句话在 Vue 实例对象 vm 上添加了 `_self` 属性，指向真实的实例本身。注意 `vm._self` 和 `vm._renderProxy` 不同，首先在用途上来说寓意是不同的，另外 `vm._renderProxy` 有可能是一个代理对象，即 Proxy 实例。

接下来执行的才是 initLifecycle 函数，同时将当前 Vue 实例 vm 作为参数传递。打开 core/instance/lifecycle.js 文件找到 initLifecycle 函数，如下：

```js
export function initLifecycle (vm: Component) {
  // 定义 options，它是 vm.$options 的引用，后面的代码使用的都是 options 常量
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```

上面代码是 initLifecycle 函数的全部内容，首先定义 options 常量，它是 `vm.$options` 的引用。接着将执行下面这段代码：

```js
// locate first non-abstract parent (查找第一个非抽象的父组件)
// 定义 parent，它引用当前实例的父实例
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}

// 设置当前实例的 $parent 属性，指向父级
vm.$parent = parent
// 设置 $root 属性，有父级就是用父级的 $root，否则 $root 指向自身
vm.$root = parent ? parent.$root : vm

```

上面代码的作用可以用一句话总结：“将当前实例添加到父实例的 `$children` 属性里，并设置当前实例的 `$parent` 指向父实例”。那么要实现这个目标首先要寻找到父级才行，那么父级的来源是哪里呢？就是这句话：

```js
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
```

通过读取 options.parent 获取父实例，但是问题来了，我们知道 options 是 `vm.$options` 的引用，所以这里的 options.parent 相当于 `vm.$options.parent`，那么 `vm.$options.parent` 从哪里来？比如下面的例子：

```js
// 子组件本身并没有指定 parent 选项
var ChildComponent = {
  created () {
    // 但是在子组件中访问父实例，能够找到正确的父实例引用
    console.log(this.$options.parent)
  }
}

var vm = new Vue({
    el: '#app',
    components: {
      // 注册组件
      ChildComponent
    },
    data: {
        test: 1
    }
})
```

我们知道 Vue 给我们提供了 parent 选项，使得我们可以手动指定一个组件的父实例，但在上面的例子中，我们并没有手动指定 parent 选项，但是子组件依然能够正确地找到它的父实例，这说明 Vue 在寻找父实例的时候是自动检测的。那它是怎么做的呢？目前不准备给大家介绍，因为时机还不够成熟，现在讲大家很容易懵，不过可以给大家看一段代码，打开 core/vdom/create-component.js 文件，里面有一个函数叫做 createComponentInstanceForVnode，如下：

```js
export function createComponentInstanceForVnode (
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

这个函数是干什么的呢？我们知道当我们注册一个组件的时候，还是拿上面的例子，如下：

```js
// 子组件
var ChildComponent = {
  created () {
    console.log(this.$options.parent)
  }
}

var vm = new Vue({
    el: '#app',
    components: {
      // 注册组件
      ChildComponent
    },
    data: {
        test: 1
    }
})
```

上面的代码中，我们的子组件 ChildComponent 说白了就是一个 json 对象，或者叫做组件选项对象，在父组件的 components 选项中把这个子组件选项对象注册了进去，实际上在 Vue 内部，会首先以子组件选项对象作为参数通过 Vue.extend 函数创建一个子类出来，然后再通过实例化子类来创建子组件，而 createComponentInstanceForVnode 函数的作用，在这里大家就可以简单理解为实例化子组件，只不过这个过程是在虚拟DOM的 patch 算法中进行的，我们后边会详细去讲。我们看 createComponentInstanceForVnode 函数内部有这样一段代码：

```js
const options: InternalComponentOptions = {
  _isComponent: true,
  _parentVnode: vnode,
  parent
}
```

这是实例化子组件时的组件选项，我们发现，第三个值就是 parent，那么这个 parent 是谁呢？它是 createComponentInstanceForVnode 函数的形参，所以我们需要找到 createComponentInstanceForVnode 函数是在哪里调用的，它的调用位置就在 core/vdom/create-component.js 文件内的 componentVNodeHooks 钩子对象的 init 钩子函数内，如下：

```js
// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    ...
  },

  insert (vnode: MountedComponentVNode) {
    ...
  },

  destroy (vnode: MountedComponentVNode) {
    ...
  }
}
```

在 init 函数内有这样一段代码：

```js
const child = vnode.componentInstance = createComponentInstanceForVnode(
  vnode,
  activeInstance
)
```

第二个参数 activeInstance 就是我们要找的 parent，那么 activeInstance 是什么呢？根据文件顶部的 import 语句可知，activeInstance 来自于 core/instance/lifecycle.js 文件，也就是我们正在看的 initLifecycle 函数的上面，如下：

```js
export let activeInstance: any = null
```

这个变量将总是保存着当前正在渲染的实例的引用，所以它就是当前实例 components 下注册的子组件的父实例，所以 Vue 实际上就是这样做到自动侦测父级的。

这里大家尽量去理解一下，不过如果还是有点懵也没关系，随着我们对 Vue 的深入，慢慢的都会很好消化。上面我们解释了这么多，其实就是想说明白一件事，即 initLifecycle 函数内的代码中的 `options.parent` 的来历，它有值的原因。

所以现在我们初步知道了 `options.parent` 值的来历，且知道了它的值指向父实例，那么接下来我们继续看代码，还是这段代码：

```js
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}
```

拿到父实例 parent 之后，进入一个判断分支，条件是：`parent && !options.abstract`，即父实例存在，且当前实例不是抽象的，这里大家可能会有疑问：什么是抽象的实例？实际上 Vue 内部有一些选项是没有暴露给我们的，就比如这里的 abstract，通过设置这个选项为 true，可以指定该组件是抽象的，那么通过该组件创建的实例也都是抽象的，比如：

```js
AbsComponents = {
  abstract: true,
  created () {
    console.log('我是一个抽象的组件')
  }
}
```

抽象的组件有什么特点呢？一个最显著的特点就是它们一般不渲染真实DOM，这么说大家可能不理解，我举个例子大家就明白了，我们知道 Vue 内置了一些全局组件比如 `keep-alive` 或者 `transition`，我们知道这两个组件它是不会渲染DOM至页面的，但他们依然给我提供了很有用的功能。所以他们就是抽象的组件，我们可以查看一下它的源码，打开 core/components/keep-alive.js 文件，你能看到这样的代码：

```js
export default {
  name: 'keep-alive',
  abstract: true,
  ...
}
```

可以发现，它使用 abstract 选项来声明这是一个抽象组件。除了不渲染真实DOM，抽象组件还有一个特点，就是它们不会出现在父子关系的路径上。这么设计也是合理的，这是由它们的性质所决定的。

所以现在大家再回看这段代码：

```js
// locate first non-abstract parent (查找第一个非抽象的父组件)
// 定义 parent，它引用当前实例的父组件
let parent = options.parent
// 如果当前实例有父组件，且当前实例不是抽象的
if (parent && !options.abstract) {
  // 使用 while 循环查找第一个非抽象的父组件
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  // 经过上面的 while 循环后，parent 应该是一个非抽象的组件，将它作为当前实例的父级，所以将当前实例 vm 添加到父级的 $children 属性里
  parent.$children.push(vm)
}

// 设置当前实例的 $parent 属性，指向父级
vm.$parent = parent
// 设置 $root 属性，有父级就是用父级的 $root，否则 $root 指向自身
vm.$root = parent ? parent.$root : vm
```

如果 `options.abstract` 为真，那说明当前实例是抽象的，所以并不会走 if 分支的代码，所以会跳过 if 语句块直接设置 `vm.$parent` 和 `vm.$root` 的值。跳过 if 语句块的结果将导致该抽象实例不会被添加到父实例的 `$children` 中。如果 `options.abstract` 为假，那说明当前实例不是抽象的，是一个普通的组件实例，这个时候就会走 while 循环，那么这个 while 循环是干嘛的呢？我们前面说过，抽象的组件是不能够也不应该作为父级的，所以 while 循环的目的就是沿着父实例链逐层向上寻找到第一个不抽象的实例作为 parent（父级）。并且在找到父级之后将当前实例添加到父实例的 `$children` 属性中，这样最终的目的就达成了。

在上面这段代码执行完毕之后，initLifecycle 函数还负责在当前实例上添加一些属性，即后面要执行的代码：

```js
vm.$children = []
vm.$refs = {}

vm._watcher = null
vm._inactive = null
vm._directInactive = false
vm._isMounted = false
vm._isDestroyed = false
vm._isBeingDestroyed = false
```

其中 `$children` 和 `$refs` 都是我们熟悉的实例属性，他们都在 initLifecycle 函数中被初始化，其中 `$children` 被初始化为一个数组，`$refs` 被初始化为一个空 json 对象，除此之外，还定义了一些内部使用的属性，大家先混个脸熟，在后面的分析中自然会知道他们的用途，但是不要忘了，既然这些属性是在 initLifecycle 函数中定义的，那么自然会与生命周期有关。这样 initLifecycle 函数我们就分析完毕了，我们回到 _init 函数，看看接下来要做的初始化工作是什么。
