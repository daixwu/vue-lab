# Vue mountComponent执行组件挂载

无论是完整版 Vue 的 `$mount` 函数还是运行时版 Vue 的 `$mount` 函数，他们最终都将通过 mountComponent 函数去真正的挂载组件，接下来我们就看一看在 mountComponent 函数中发生了什么，打开 src/core/instance/lifecycle.js 文件找到 mountComponent 如下：

```js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  // 省略...
}
```

mountComponent 函数接收三个参数，分别是组件实例 vm，挂载元素 el 以及透传过来的 hydrating 参数。mountComponent 函数的第一句代码如下：

```js
vm.$el = el
```

在组件实例对象上添加 `$el` 属性，其值为挂载元素 el。我们知道 `$el` 的值是组件模板根元素的引用，如下代码：

```js
<div id="foo"></div>

<script>
const new Vue({
  el: '#foo',
  template: '<div id="bar"></div>'
})
</script>
```

上面代码中，挂载元素是一个 id 为 foo 的 div 元素，而组件模板是一个 id 为 bar 的 div 元素。那么大家思考一个问题：`vm.$el` 的值应该是哪一个 div 元素的引用？答案是：`vm.$el` 是 id 为 bar 的 div 的引用。这是因为 `vm.$el` 始终是组件模板的根元素。由于我们传递了 template 选项指定了模板，那么 `vm.$el` 自然就是 id 为 bar 的 div 的引用。假设我们没有传递 template 选项，那么根据我们前面的分析，el 选项指定的挂载点将被作为组件模板，这个时候 `vm.$el` 则是 id 为 foo 的 div 元素的引用。

再结合 mountComponent 函数体的这句话：`vm.$el = el`，有的同学就会有疑问了，这里明明把 el 挂载元素赋值给了 `vm.$el`，那么 `vm.$el` 怎么可能引用的是 template 选项指定的模板的根元素呢？其实这里仅仅是暂时赋值而已，这是为了给虚拟DOM的 patch 算法使用的，实际上 `vm.$el` 会被 patch 算法的返回值重写，为了证明这一点我们可以打开 src/core/instance/lifecycle.js 文件找到 `Vue.prototype._update` 方法，如下代码所示：

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  // 省略...

  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  
  // 省略...
}
```

正如上面代码所示的那样，`vm.$el` 的值将被 `vm.__patch__` 函数的返回值重写。不过现在大家或许还不清楚 `Vue.prototype._update` 的作用是什么，这块内容我们将在后面的章节详细讲解。

我们继续查看 mountComponent 函数的代码，接下来是一段 if 语句块：

```js
if (!vm.$options.render) {
  vm.$options.render = createEmptyVNode
  if (process.env.NODE_ENV !== 'production') {
    /* istanbul ignore if */
    if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
      vm.$options.el || el) {
      warn(
        'You are using the runtime-only build of Vue where the template ' +
        'compiler is not available. Either pre-compile the templates into ' +
        'render functions, or use the compiler-included build.',
        vm
      )
    } else {
      warn(
        'Failed to mount component: template or render function not defined.',
        vm
      )
    }
  }
}
```

这段 if 条件语句块首先检查渲染函数是否存在，即 `vm.$options.render` 是否为真，如果不为真说明渲染函数不存在，这时将会执行 if 语句块内的代码，在 if 语句块内首先将 `vm.$options.render` 的值设置为 createEmptyVNode 函数，也就是说此时渲染函数的作用将仅仅渲染一个空的 `vnode` 对象，然后在非生产环境下会根据相应的情况打印警告信息。

在上面这段 if 语句块的下面，执行了 callHook 函数，触发 beforeMount 生命周期钩子：

```js
callHook(vm, 'beforeMount')
```

在触发 beforeMount 生命周期钩子之后，组件将开始挂载工作，首先是如下这段代码：

```js
let updateComponent
/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  updateComponent = () => {
    const name = vm._name
    const id = vm._uid
    const startTag = `vue-perf-start:${id}`
    const endTag = `vue-perf-end:${id}`

    mark(startTag)
    const vnode = vm._render()
    mark(endTag)
    measure(`vue ${name} render`, startTag, endTag)

    mark(startTag)
    vm._update(vnode, hydrating)
    mark(endTag)
    measure(`vue ${name} patch`, startTag, endTag)
  }
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}

new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

这段代码的作用只有一个，即定义并初始化 updateComponent 函数，这个函数将用作创建 Watcher 实例时传递给 Watcher 构造函数的第二个参数，这也将是我们第一次真正地接触 Watcher 构造函数，不过现在我们需要先把 updateComponent 函数搞清楚，在上面的代码中首先定义了 updateComponent 变量，虽然是一个 `if...else` 语句块，其中 if 语句块的条件我们已经遇到过很多次了，在满足该条件的情况下会做一些性能统计，可以看到在 if 语句块中分别统计了 `vm._render()` 函数以及 `vm._update()` 函数的运行性能。也就是说无论是执行 if 语句块还是执行 else 语句块，最终 updateComponent 函数的功能是不变的。

既然功能相同，我们就直接看 else 语句块的代码，因为它要简洁的多：

```js
let updateComponent
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  // 省略...
} else {
  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }
}
```

可以看到 updateComponent 是一个函数，该函数的作用是以 `vm._render()` 函数的返回值作为第一个参数调用 `vm._update()` 函数。由于我们还没有讲解 `vm._render` 函数和 `vm._update` 函数的作用，所以为了让大家更好理解，我们可以简单地认为：

- `vm._render` 函数的作用是调用 `vm.$options.render` 函数并返回生成的虚拟节点(vnode)
- `vm._update` 函数的作用是把 `vm._render` 函数生成的虚拟节点渲染成真正的 DOM

也就是说目前我们可以简单地认为 updateComponent 函数的作用就是：**把渲染函数生成的虚拟DOM渲染成真正的DOM**，其实在 `vm._update` 内部是通过虚拟DOM的补丁算法(patch)来完成的，这些我们放到后面的具体章节去讲。

再往下，我们将遇到创建观察者(Watcher)实例的代码：

```js
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

前面说过，这将是我们第一次真正意义上的遇到观察者构造函数 Watcher，我们在 Vue数据响应系统中有提到过，正是因为 watcher 对表达式的求值，触发了数据属性的 get 拦截器函数，从而收集到了依赖，当数据变化时能够触发响应。在上面的代码中 Watcher 观察者实例将对 updateComponent 函数求值，我们知道 updateComponent 函数的执行会间接触发渲染函数(`vm.$options.render`)的执行，而渲染函数的执行则会触发数据属性的 get 拦截器函数，从而将依赖(观察者)收集，当数据变化时将重新执行 updateComponent 函数，这就完成了重新渲染。同时我们把上面代码中实例化的观察者对象称为 **渲染函数的观察者**。

```js
// manually mounted instance, call mounted on self
// mounted is called for render-created child components in its inserted hook
if (vm.$vnode == null) {
  vm._isMounted = true
  callHook(vm, 'mounted')
}
return vm
```

函数最后判断为根节点的时候设置 `vm._isMounted` 为 true， 表示这个实例已经挂载了，同时执行 mounted 钩子函数。 这里注意 `vm.$vnode` 表示 Vue 实例的父虚拟 Node，所以它为 Null 则表示当前是根 Vue 的实例。
