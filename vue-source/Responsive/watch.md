# watch 侦听属性的实现

侦听属性的初始化发生在 Vue 的实例初始化阶段的 initState 函数中，在 computed 初始化之后，执行了：

```js
if (opts.watch && opts.watch !== nativeWatch) {
  initWatch(vm, opts.watch)
}
```

来看一下 initWatch 的实现，它的定义在 src/core/instance/state.js 中

```js
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

这里就是对 watch 对象做遍历，拿到每一个 handler，因为 Vue 是支持 watch 的同一个 key 对应多个 handler，如下：

```js
watch: {
  name: [
    function () {
      console.log('name 改变了1')
    },
    function () {
      console.log('name 改变了2')
    }
  ]
}
```

所以如果 handler 是一个数组，则遍历这个数组，调用 createWatcher 方法，否则直接调用 createWatcher：

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

这里的逻辑也很简单，首先对 hanlder 的类型做判断，拿到它最终的回调函数，最后调用 `vm.$watch(keyOrFn, handler, options)` 函数，`$watch` 是 Vue 原型上的方法，它是在执行 stateMixin 的时候定义的, stateMixin 函数同样定义在 src/core/instance/state.js 文件中：

```js
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

`$watch` 方法允许我们观察数据对象的某个属性，当属性变化时执行回调。所以 `$watch` 方法至少接收两个参数，一个要观察的属性，以及一个回调函数。通过上面的代码我们发现，`$watch` 方法接收三个参数，除了前面介绍的两个参数之后还接收第三个参数，它是一个选项参数，比如是否立即执行回调或者是否深度观测等。我们可以发现这三个参数与 `Watcher` 类的构造函数中的三个参数相匹配，如下：

```js
export default class Watcher {
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // 省略...
  }
}
```

其实这很好理解，因为 `$watch` 方法的实现本质就是创建了一个 `Watcher` 实例对象。另外通过[官方文档](https://cn.vuejs.org/v2/api/#vm-watch)的介绍可知 `$watch` 方法的第二个参数既可以是一个回调函数，也可以是一个纯对象，这个对象中可以包含 `handler` 属性，该属性的值将作为回调函数，同时该对象还可以包含其他属性作为选项参数，如 `immediate` 或 `deep`。

现在我们假设传递给 `$watch` 方法的第二个参数是一个函数，看看它是怎么实现的，在 `$watch` 方法内部首先执行的是如下代码：

```js
const vm: Component = this
if (isPlainObject(cb)) {
  return createWatcher(vm, expOrFn, cb, options)
}
```

定义了 `vm` 常量，它是当前组件实例对象，接着检测传递给 `$watch` 的第二个参数是否是纯对象，由于我们现在假设参数 `cb` 是一个函数，所以这段 `if` 语句块内的代码不会执行。再往下是这段代码：

```js
options = options || {}
options.user = true
const watcher = new Watcher(vm, expOrFn, cb, options)
```

首先如果没有传递 `options` 选项参数，那么会给其一个默认的空对象，接着将 `options.user` 的值设置为 `true`，我们前面讲到过这代表该观察者实例是用户创建的，然后就到了关键的一步，即创建 `Watcher` 实例对象，多么简单的实现。

再往下是一段 `if` 语句块：

```js
if (options.immediate) {
  try {
    cb.call(vm, watcher.value)
  } catch (error) {
    handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
  }
}
```

我们知道 `immediate` 选项用来在属性或函数被侦听后立即执行回调，如上代码就是其实现原理，如果发现 `options.immediate` 选项为真，那么会执行回调函数，不过此时回调函数的参数只有新值没有旧值。同时取值的方式是通过前面创建的观察者实例对象的 `watcher.value` 属性。我们知道观察者实例对象的 `value` 属性，保存着被观察属性的值。

最后 `$watch` 方法还有一个返回值，如下：

```js
return function unwatchFn () {
  watcher.teardown()
}
```

`$watch` 函数返回一个函数，这个函数的执行会解除当前观察者对属性的观察。它的原理是通过调用观察者实例对象的 `watcher.teardown` 函数实现的。我们可以看一下 `watcher.teardown` 函数是如何解除观察者与属性之间的关系的，它定义在 src/core/observer/watcher.js 中，如下是 `teardown` 函数的代码：

```js
export default class Watcher {
  // 省略...
  /**
   * Remove self from all dependencies' subscriber list.
   */
  teardown () {
    if (this.active) {
      // remove self from vm's watcher list
      // this is a somewhat expensive operation so we skip it
      // if the vm is being destroyed.
      if (!this.vm._isBeingDestroyed) {
        remove(this.vm._watchers, this)
      }
      let i = this.deps.length
      while (i--) {
        this.deps[i].removeSub(this)
      }
      this.active = false
    }
  }
}
```

首先检查 `this.active` 属性是否为真，如果为假则说明该观察者已经不处于激活状态，什么都不需要做，如果为真则会执行 `if` 语句块内的代码，在 `if` 语句块内首先执行的这段代码：

```js
if (!this.vm._isBeingDestroyed) {
  remove(this.vm._watchers, this)
}
```

首先说明一点，每个组件实例都有一个 `vm._isBeingDestroyed` 属性，它是一个标识，为真说明该组件实例已经被销毁了，为假说明该组件还没有被销毁，所以以上代码的意思是如果组件没有被销毁，那么将当前观察者实例从组件实例对象的 `vm._watchers` 数组中移除，我们知道 `vm._watchers` 数组中包含了该组件所有的观察者实例对象，所以将当前观察者实例对象从 `vm._watchers` 数组中移除是解除属性与观察者实例对象之间关系的第一步。由于这个操作的性能开销比较大，所以仅在组件没有被销毁的情况下才会执行此操作。

将观察者实例对象从 `vm._watchers` 数组中移除之后，会执行如下这段代码：

```js
let i = this.deps.length
while (i--) {
  this.deps[i].removeSub(this)
}
```

我们知道当一个属性与一个观察者建立联系之后，属性的 `Dep` 实例对象会收集到该观察者对象，同时观察者对象也会将该 `Dep` 实例对象收集，这是一个双向的过程，并且一个观察者可以同时观察多个属性，这些属性的 `Dep` 实例对象都会被收集到该观察者实例对象的 `this.deps` 数组中，所以解除属性与观察者之间关系的第二步就是将当前观察者实例对象从所有的 `Dep` 实例对象中移除，实现方法就如上代码所示。

最后会将当前观察者实例对象的 `active` 属性设置为 `false`，代表该观察者对象已经处于非激活状态了：

```js
this.active = false
```

以上就是 `$watch` 方法的实现，以及如何解除观察的实现。不过不要忘了我们前面所讲的这些内容是假设传递给 `$watch` 方法的第二个参数是一个函数，如果不是函数呢？比如是一个纯对象，这时如下高亮的代码就会被执行：

```js
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  // 省略...
}
```

当参数 `cb` 不是函数，而是一个纯对象，则会调用 `createWatcher` 函数，并将参数透传，注意还多传递给 `createWatcher` 函数一个参数，即组件实例对象 `vm`，那么 `createWatcher` 函数做了什么呢？`createWatcher` 函数也定义在 `src/core/instance/state.js` 文件中，如下是 `createWatcher` 函数的代码：

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

其实 `createWatcher` 函数的作用就是将纯对象形式的参数规范化一下，然后再通过 `$watch` 方法创建观察者。可以看到 `createWatcher` 函数的最后一句代码就是通过调用 `$watch` 函数并将其返回。来看 `createWatcher` 函数的第一段代码：

```js
if (isPlainObject(handler)) {
  options = handler
  handler = handler.handler
}
```

检测参数 `handler` 是否是纯对象，有的同学可能会问：“在 `$watch` 方法中已经检测过参数 `cb` 是否是纯对象了，这里又检测了一次是否多此一举？”，其实这么做并不是多余的，因为 `createWatcher` 函数除了在 `$watch` 方法中使用之外，还会用于 `watch` 选项，而这时就需要对 `handler` 进行检测。总之如果 `handler` 是一个纯对象，那么就将变量 `handler` 的值赋给 `options` 变量，然后用 `handler.handler` 的值重写 `handler` 变量的值。举个例子，如下代码所示：

```js
vm.$watch('name', {
  handler () {
    console.log('change')
  },
  immediate: true
})
```

如果你像如上代码那样使用 `$watch` 方法，那么对于 `createWatcher` 函数来讲，其 `handler` 参数为：

```js
handler = {
  handler () {
    console.log('change')
  },
  immediate: true
}
```

所以如下这段代码：

```js
if (isPlainObject(handler)) {
  options = handler
  handler = handler.handler
}
```

等价于：

```js
if (isPlainObject(handler)) {
  options = {
    handler () {
      console.log('change')
    },
    immediate: true
  }
  handler = handler () {
    console.log('change')
  }
}
```

这样就可正常通过 `$watch` 方法创建观察者了。另外我们注意 `createWatcher` 函数中接下来这段代码：

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

这段代码说明 `handler` 除了可以是一个纯对象还可以是一个字符串，当 `handler` 是一个字符串时，会读取组件实例对象的 `handler` 属性的值并用该值重写 `handler` 的值。然后再通过调用 `$watch` 方法创建观察者，这段代码实现的目的是什么呢？看如下例子就明白了：

```js
watch: {
  name: 'handleNameChange'
},
methods: {
  handleNameChange () {
    console.log('name change')
  }
}
```

上面的代码中我们在 `watch` 选项中观察了 `name` 属性，但是我们没有指定回调函数，而是指定了一个字符串 `handleNameChange`，这等价于指定了 `methods` 选项中同名函数作为回调函数。这就是如上 `createWatcher` 函数中那段代码的目的。
