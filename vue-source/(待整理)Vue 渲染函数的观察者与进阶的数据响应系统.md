# 渲染函数的观察者与进阶的数据响应系统


## $watch和watch选项的实现

前面我们已经讲了足够多关于 `Watcher` 类的内容，接下来是时候看一下 `$watch` 方法以及 `watch` 选项的实现了。实际上无论是 `$watch` 方法还是 `watch` 选项，他们的实现都是基于 `Watcher` 的封装。首先我们来看一下 `$watch` 方法，它定义在 `src/core/instance/state.js` 文件的 `stateMixin` 函数中，如下：

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
    cb.call(vm, watcher.value)
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

`$watch` 方法允许我们观察数据对象的某个属性，当属性变化时执行回调。所以 `$watch` 方法至少接收两个参数，一个要观察的属性，以及一个回调函数。通过上面的代码我们发现，`$watch` 方法接收三个参数，除了前面介绍的两个参数之后还接收第三个参数，它是一个选项参数，比如是否立即执行回调或者是否深度观测等。我们可以发现这三个参数与 `Watcher` 类的构造函数中的三个参数相匹配，如下：

```js {4-6}
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

其实这很好理解，因为 `$watch` 方法的实现本质就是创建了一个 `Watcher` 实例对象。另外通过官方文档的介绍可知 `$watch` 方法的第二个参数既可以是一个回调函数，也可以是一个纯对象，这个对象中可以包含 `handler` 属性，该属性的值将作为回调函数，同时该对象还可以包含其他属性作为选项参数，如 `immediate` 或 `deep`。

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
  cb.call(vm, watcher.value)
}
```

我们知道 `immediate` 选项用来在属性或函数被侦听后立即执行回调，如上代码就是其实现原理，如果发现 `options.immediate` 选项为真，那么会执行回调函数，不过此时回调函数的参数只有新值没有旧值。同时取值的方式是通过前面创建的观察者实例对象的 `watcher.value` 属性。我们知道观察者实例对象的 `value` 属性，保存着被观察属性的值。

最后 `$watch` 方法还有一个返回值，如下：

```js
return function unwatchFn () {
  watcher.teardown()
}
```

`$watch` 函数返回一个函数，这个函数的执行会解除当前观察者对属性的观察。它的原理是通过调用观察者实例对象的 `watcher.teardown` 函数实现的。我们可以看一下 `watcher.teardown` 函数是如何解除观察者与属性之间的关系的，如下是 `teardown` 函数的代码：

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

```js {7-9}
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

这样就可正常通过 `$watch` 方法创建观察者了。另外我们注意 `createWatcher` 函数中如下这段高亮代码：

```js {11-13}
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

上面的代码中我们在 `watch` 选项中观察了 `name` 属性，但是我们没有指定回调函数，而是指定了一个字符串 `handleNameChange`，这等价于指定了 `methods` 选项中同名函数作为回调函数。这就是如上 `createWatcher` 函数中那段高亮代码的目的。

上例中我们使用了 `watch` 选项，接下来我们就顺便来看一下 `watch` 选项是如何初始化的，找到 `initState` 函数，如下：

```js {12-14}
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

如上高亮代码所示，在这个 `if` 条件语句块中，调用 `initWatch` 函数，这个函数用来初始化 `watch` 选项，至于判断条件我们就不多讲了，前面的讲解中我们已经讲解过类似的判断条件。至于 `initWatch` 函数，它就定义在 `createWatcher` 函数的上方，如下是其全部代码：

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

可以看到 `initWatch` 函数就是通过对 `watch` 选项遍历，然后通过 `createWatcher` 函数创建观察者对象的，需要注意的是上面代码中有一个判断条件，如下高亮代码所示：

```js {4}
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

通过这个条件我们可以发现 `handler` 常量可以是一个数组，`handler` 常量是什么呢？它的值是 `watch[key]`，也就是说我们在使用 `watch` 选项时可以通过传递数组来实现创建多个观察者，如下：

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

总的来说，在 `Watcher` 类的基础上，无论是实现 `$watch` 方法还是实现 `watch` 选项，都变得非常容易，这得益于一个良好的设计。

## 深度观测的实现

接下来我们将会讨论深度观测的实现，在这之前我们需要回顾一下数据响应的原理，我们知道响应式数据的关键在于数据的属性是访问器属性，这使得我们能够拦截对该属性的读写操作，从而有机会收集依赖并触发响应。思考如下代码：

```js
watch: {
  a () {
    console.log('a 改变了')
  }
}
```

这段代码使用 `watch` 选项观测了数据对象的 `a` 属性，我们知道 `watch` 方法内部是通过创建 `Watcher` 实例对象来实现观测的，在创建 `Watcher` 实例对象时会读取 `a` 的值从而触发属性 `a` 的 `get` 拦截器函数，最终将依赖收集。但问题是如果属性 `a` 的值是一个对象，如下：

```js {3-5}
data () {
  return {
    a: {
      b: 1
    }
  }
},
watch: {
  a () {
    console.log('a 改变了')
  }
}
```

如上高亮代码所示，数据对象 `data` 的属性 `a` 是一个对象，当实例化 `Watcher` 对象并观察属性 `a` 时，会读取属性 `a` 的值，这样的确能够触发属性 `a` 的 `get` 拦截器函数，但由于没有读取 `a.b` 属性的值，所以对于 `b` 来讲是没有收集到任何观察者的。这就是我们常说的浅观察，直接修改属性 `a` 的值能够触发响应，而修改 `a.b` 的值是触发不了响应的。

深度观测就是用来解决这个问题的，深度观测的原理很简单，既然属性 `a.b` 中没有收集到观察者，那么我们就主动读取一下 `a.b` 的值，这样不就能够触发属性 `a.b` 的 `get` 拦截器函数从而收集到观察者了吗，其实 `Vue` 就是这么做的，只不过你需要将 `deep` 选项参数设置为 `true`，主动告诉 `Watcher` 实例对象你现在需要的是深度观测。我们找到 `Watcher` 类的 `get` 方法，如下：

```js {6, 16-18}
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

如上高亮代码所示，我们知道 `Watcher` 类的 `get` 方法用来求值，在 `get` 方法内部通过调用 `this.getter` 函数对被观察的属性求值，并将求得的值赋值给变量 `value`，同时我们可以看到在 `finally` 语句块内，如果 `this.deep` 属性的值为真说明是深度观测，此时会将被观测属性的值 `value` 作为参数传递给 `traverse` 函数，其中 `traverse` 函数的作用就是递归地读取被观察属性的所有子属性的值，这样被观察属性的所有子属性都将会收集到观察者，从而达到深度观测的目的。

`traverse` 函数来自 `src/core/observer/traverse.js` 文件，如下：

```js
const seenObjects = new Set()

/**
 * Recursively traverse an object to evoke all converted
 * getters, so that every nested property inside the object
 * is collected as a "deep" dependency.
 */
export function traverse (val: any) {
  _traverse(val, seenObjects)
  seenObjects.clear()
}
```

上面的代码中定义了 `traverse` 函数，这个函数将接收被观察属性的值作为参数，拿到这个参数后在 `traverse` 函数内部会调用 `_traverse` 函数完成递归遍历。其中 `_traverse` 函数就定义在 `traverse` 函数的下方，如下是 `_traverse` 函数的签名：

```js
function _traverse (val: any, seen: SimpleSet) {
  // 省略...
}
```

`_traverse` 函数接收两个参数，第一个参数是被观察属性的值，第二个参数是一个 `Set` 数据结构的实例，可以看到在 `traverse` 函数中调用 `_traverse` 函数时传递的第二个参数 `seenObjects` 就是一个 `Set` 数据结构的实例，它定义在文件头部：`const seenObjects = new Set()`。

接下来我们看一下 `_traverse` 函数是如何遍历访问数据对象的，如下是 `_traverse` 函数的全部代码：

```js {7-13}
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

注意上面代码中高亮的部分，现在我们把高亮的代码删除，那么 `_traverse` 函数将变成如下这个样子：

```js
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

之所以要删除这段代码是为了降低复杂度，现在我们就当做删除的那段代码不存在，来看一下 `_traverse` 函数的实现，在 `_traverse` 函数的开头声明了两个变量，分别是 `i` 和 `keys`，这两个变量在后面会使用到，接着检查参数 `val` 是不是数组，并将检查结果存储在常量 `isA` 中。再往下是一段 `if` 语句块：

```js
if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
  return
}
```

这段代码是对参数 `val` 的检查，后面我们统一称 `val` 为 **被观察属性的值**，我们知道既然是深度观测，所以被观察属性的值要么是一个对象要么是一个数组，并且该值不能是冻结的，同时也不应该是 `VNode` 实例(这是Vue单独做的限制)。只有当被观察属性的值满足这些条件时，才会对其进行深度观测，只要有一项不满足 `_traverse` 就会 `return` 结束执行。所以上面这段 `if` 语句可以理解为是在检测被观察属性的值能否进行深度观测，一旦能够深度观测将会继续执行之后的代码，如下：

```js
if (isA) {
  i = val.length
  while (i--) _traverse(val[i], seen)
} else {
  keys = Object.keys(val)
  i = keys.length
  while (i--) _traverse(val[keys[i]], seen)
}
```

这段代码将检测被观察属性的值是数组还是对象，无论是数组还是对象都会通过 `while` 循环对其进行遍历，并递归调用 `_traverse` 函数，这段代码的关键在于递归调用 `_traverse` 函数时所传递的第一个参数：`val[i]` 和 `val[keys[i]]`。这两个参数实际上是在读取子属性的值，这将触发子属性的 `get` 拦截器函数，保证子属性能够收集到观察者，仅此而已。

现在 `_traverse` 函数的代码我们就解析完了，但大家有没有想过目前 `_traverse` 函数存在什么问题？别忘了前面我们删除了一段代码，如下：

```js
if (val.__ob__) {
  const depId = val.__ob__.dep.id
  if (seen.has(depId)) {
    return
  }
  seen.add(depId)
}
```

这段代码的作用不容忽视，它解决了循环引用导致死循环的问题，为了更好地说明问题我们举个例子，如下：

```js
const obj1 = {}
const obj2 = {}

obj1.data = obj2
obj2.data = obj1
```

上面代码中我们定义了两个对象，分别是 `obj1` 和 `obj2`，并且 `obj1.data` 属性引用了 `obj2`，而 `obj2.data` 属性引用了 `obj1`，这是一个典型的循环引用，假如我们使用 `obj1` 或 `obj2` 这两个对象中的任意一个对象出现在 `Vue` 的响应式数据中，如果不做防循环引用的处理，将会导致死循环，如下代码：

```js
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

如果被观察属性的值 `val` 是一个循环引用的对象，那么上面的代码将导致死循环，为了避免这种情况的发生，我们可以使用一个变量来存储那些已经被遍历过的对象，当再次遍历该对象时程序会发现该对象已经被遍历过了，这时会跳过遍历，从而避免死循环，如下代码所示：

```js {7-13}
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  if (val.__ob__) {
    const depId = val.__ob__.dep.id
    if (seen.has(depId)) {
      return
    }
    seen.add(depId)
  }
  if (isA) {
    i = val.length
    while (i--) _traverse(val[i], seen)
  } else {
    keys = Object.keys(val)
    i = keys.length
    while (i--) _traverse(val[keys[i]], seen)
  }
}
```

如上高亮的代码所示，这是一个 `if` 语句块，用来判断 `val.__ob__` 是否有值，我们知道如果一个响应式数据是对象或数组，那么它会包含一个叫做 `__ob__` 的属性，这时我们读取 `val.__ob__.dep.id` 作为一个唯一的ID值，并将它放到 `seenObjects` 中：`seen.add(depId)`，这样即使 `val` 是一个拥有循环引用的对象，当下一次遇到该对象时，我们能够发现该对象已经遍历过了：`seen.has(depId)`，这样函数直接 `return` 即可。

以上就是深度观测的实现以及避免循环引用造成的死循环的解决方案。

## 计算属性的实现

### 计算属性的初始化

到目前为止，我们对响应系统的了解已经足够多了，是时候研究一下计算属性的实现了，实际上很多看上去神奇的东西在良好设计的系统中实现起来并没有想象的那么复杂，计算属性就是典型的案例，它本质上就是一个惰性求值的观察者。我们回到 `src/core/instance/state.js` 文件中的 `initState` 函数，因为计算属性是在这里被初始化的，如下高亮代码所示：

```js {11}
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

这句代码首先检查开发者是否传递了 `computed` 选项，只有传递了该选项的情况下才会调用 `initComputed` 函数进行初始化，找到 `initComputed` 函数，如下：

```js
function initComputed (vm: Component, computed: Object) {
  // 省略...
}
```

与其它初始化响应数据相关的函数一样，都接收两个参数，第一个参数是组件对象实例，第二个参数是对应的选项。在 `initComputed` 函数的开头定义了两个常量：

```js
// $flow-disable-line
const watchers = vm._computedWatchers = Object.create(null)
// computed properties are just getters during SSR
const isSSR = isServerRendering()
```

其中 `watchers` 常量与组件实例的 `vm._computedWatchers` 属性拥有相同的引用，且初始值都是通过 `Object.create(null)` 创建的空对象，`isSSR` 常量是用来判断是否是服务端渲染的布尔值。接着开启一个 `for...in` 循环，后续的所有代码都写在了这个 `for...in` 循环中：

```js
for (const key in computed) {
  // 省略...
}
```

这个 `for...in` 循环用来遍历 `computed` 选项对象，在循环的内部首先是这样一段代码：

```js
const userDef = computed[key]
const getter = typeof userDef === 'function' ? userDef : userDef.get
if (process.env.NODE_ENV !== 'production' && getter == null) {
  warn(
    `Getter is missing for computed property "${key}".`,
    vm
  )
}
```

定义了 `userDef` 常量，它的值是计算属性对象中相应的属性值，我们知道计算属性有两种写法，计算属性可以是一个函数，如下：

```js
computed: {
  someComputedProp () {
    return this.a + this.b
  }
}
```

如果你使用上面的写法，那么 `userDef` 的值就是一个函数：

```js
userDef = someComputedProp () {
  return this.a + this.b
}
```

另外计算属性也可以写成对象，如下：

```js
computed: {
  someComputedProp: {
    get: function () {
      return this.a + 1
    },
    set: function (v) {
      this.a = v - 1
    }
  }
}
```

如果你使用如上这种写法，那么 `userDef` 常量的值就是一个对象：

```js
userDef = {
  get: function () {
    return this.a + 1
  },
  set: function (v) {
    this.a = v - 1
  }
}
```

在 `userDef` 常量的下面定义了 `getter` 常量，它的值是根据 `userDef` 常量的值决定的：

```js
const getter = typeof userDef === 'function' ? userDef : userDef.get
```

如果计算属性使用函数的写法，那么 `getter` 常量的值就是 `userDef` 本身，即函数。如果计算属性使用的是对象写法，那么 `getter` 的值将会是 `userDef.get` 函数。总之 `getter` 常量总会是一个函数。

在 `getter` 常量的下面做了一个检测：

```js
if (process.env.NODE_ENV !== 'production' && getter == null) {
  warn(
    `Getter is missing for computed property "${key}".`,
    vm
  )
}
```

在非生产环境下如果发现 `getter` 不存在，则直接打印警告信息，提示你计算属性没有对应的 `getter`。也就是说计算属性的函数写法实际上是对象写法的简化，如下这两种写法是等价的：

```js
computed: {
  someComputedProp () {
    return this.a + this.b
  }
}

// 等价于

computed: {
  someComputedProp: {
    get () {
      return this.a + this.b
    }
  }
}
```

再往下，是一段 `if` 条件语句块，如下：

```js
if (!isSSR) {
  // create internal watcher for the computed property.
  watchers[key] = new Watcher(
    vm,
    getter || noop,
    noop,
    computedWatcherOptions
  )
}
```

只有在非服务端渲染时才会执行 `if` 语句块内的代码，因为服务端渲染中计算属性的实现本质上和使用 `methods` 选项差不多。这里我们着重讲解非服务端渲染的实现，这时 `if` 语句块内的代码会被执行，可以看到在 `if` 语句块内创建了一个观察者实例对象，我们称之为 **计算属性的观察者**，同时会把计算属性的观察者添加到 `watchers` 常量对象中，键值是对应计算属性的名字，注意由于 `watchers` 常量与 `vm._computedWatchers` 属性具有相同的引用，所以对 `watchers` 常量的修改相当于对 `vm._computedWatchers` 属性的修改，现在你应该知道了，`vm._computedWatchers` 对象是用来存储计算属性观察者的。

另外有几点需要注意，首先创建计算属性观察者时所传递的第二个参数是 `getter` 函数，也就是说计算属性观察者的求值对象是 `getter` 函数。传递的第四个参数是 `computedWatcherOptions` 常量，它是一个对象，定义在 `initComputed` 函数的上方：

```js
const computedWatcherOptions = { computed: true }
```

我们知道传递给 `Watcher` 类的第四个参数是观察者的选项参数，选项参数对象可以包含如 `deep`、`sync` 等选项，当然了其中也包括 `computed` 选项，通过如上这句代码可知在创建计算属性观察者对象时 `computed` 选项为 `true`，它的作用就是用来标识一个观察者对象是计算属性的观察者，计算属性的观察者与非计算属性的观察者的行为是不一样的。

再往下是 `for...in` 循环中的最后一段代码，如下：

```js
if (!(key in vm)) {
  defineComputed(vm, key, userDef)
} else if (process.env.NODE_ENV !== 'production') {
  if (key in vm.$data) {
    warn(`The computed property "${key}" is already defined in data.`, vm)
  } else if (vm.$options.props && key in vm.$options.props) {
    warn(`The computed property "${key}" is already defined as a prop.`, vm)
  }
}
```

这段代码首先检查计算属性的名字是否已经存在于组件实例对象中，我们知道在初始化计算属性之前已经初始化了 `props`、`methods` 和 `data` 选项，并且这些选项数据都会定义在组件实例对象上，由于计算属性也需要定义在组件实例对象上，所以需要使用计算属性的名字检查组件实例对象上是否已经有了同名的定义，如果该名字已经定义在组件实例对象上，那么有可能是 `data` 数据或 `props` 数据或 `methods` 数据之一，对于 `data` 和 `props` 来讲他们是不允许被 `computed` 选项中的同名属性覆盖的，所以在非生产环境中还要检查计算属性中是否存在与 `data` 和 `props` 选项同名的属性，如果有则会打印警告信息。如果没有则调用 `defineComputed` 定义计算属性。

`defineComputed` 函数就定义在 `initComputed` 函数的下方，如下是 `defineComputed` 函数的签名及最后一句代码：

```js {7}
export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  // 省略...
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

根据 `defineComputed` 函数的最后一句代码可知，该函数的作用就是通过 `Object.defineProperty` 函数在组件实例对象上定义与计算属性同名的组件实例属性，而且是一个访问器属性，属性的配置参数是 `sharedPropertyDefinition` 对象，`defineComputed` 函数中除最后一句代码之外的所有代码都是用来完善 `sharedPropertyDefinition` 对象的。

`sharedPropertyDefinition` 对象定义在当前文件头部，如下：

```js
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}
```

接下来我们就看一下 `defineComputed` 函数是如何完善这个对象的，在 `defineComputed` 函数开头定义了 `shouldCache` 常量，它的值与 `initComputed` 函数中定义的 `isSSR` 常量的值是取反的关系，也是一个布尔值，用来标识是否应该缓存值，也就是说只有在非服务端渲染的情况下计算属性才会缓存值。

紧接着是一段 `if...else` 语句块：

```js
if (typeof userDef === 'function') {
  sharedPropertyDefinition.get = shouldCache
    ? createComputedGetter(key)
    : userDef
  sharedPropertyDefinition.set = noop
} else {
  sharedPropertyDefinition.get = userDef.get
    ? shouldCache && userDef.cache !== false
      ? createComputedGetter(key)
      : userDef.get
    : noop
  sharedPropertyDefinition.set = userDef.set
    ? userDef.set
    : noop
}
```

这段 `if...else` 语句块的作用是为 `sharedPropertyDefinition.get` 和 `sharedPropertyDefinition.set` 赋予合适的值。首先检查 `userDef` 是否是函数，如果是函数则执行 `if` 语句块内的代码，如果不是函数则说明 `userDef` 是对象，此时会执行 `else` 分支的代码。假如 `userDef` 是函数，在 `if` 语句块内首先会使用三元运算符检查 `shouldCache` 是否为真，如果为真说明不是服务端渲染，此时会调用 `createComputedGetter` 函数并将其返回值作为 `sharedPropertyDefinition.get` 的值。如果 `shouldCache` 为假说明是服务端渲染，由于服务端渲染不需要缓存值，所以直接使用 `userDef` 函数作为 `sharedPropertyDefinition.get` 的值。另外由于 `userDef` 是函数，这说明该计算属性并没有指定 `set` 拦截器函数，所以直接将其设置为空函数 `noop`：`sharedPropertyDefinition.set = noop`。

如果代码走到了 `else` 分支，那说明 `userDef` 是一个对象，如果 `userDef.get` 存在并且是在非服务端渲染的环境下，同时没有指定选项 `userDef.cache` 为假，则此时会调用 `createComputedGetter` 函数并将其返回值作为 `sharedPropertyDefinition.get` 的值，否则 `sharedPropertyDefinition.get` 的值为 `userDef.get` 函数。同样的如果 `userDef.set` 函数存在，则使用 `userDef.set` 函数作为 `sharedPropertyDefinition.set` 的值，否则使用空函数 `noop` 作为其值。

总之，无论 `userDef` 是函数还是对象，在非服务端渲染的情况下，配置对象 `sharedPropertyDefinition` 最终将变成如下这样：

```js
sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: createComputedGetter(key),
  set: userDef.set // 或 noop
}
```

举个例子，假如我们像如下这样定义计算属性：

```js
computed: {
  someComputedProp () {
    return this.a + this.b
  }
}
```

那么定义 `someComputedProp` 访问器属性时的配置对象为：

```js
sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: createComputedGetter(key),
  set: noop // 没有指定 userDef.set 所以是空函数
}
```

对于 `createComputedGetter` 函数，它的返回值很显然的应该也是一个函数才对，`createComputedGetter` 函数定义在 `defineComputed` 函数的下方，如下：

```js
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      watcher.depend()
      return watcher.evaluate()
    }
  }
}
```

可以看到 `createComputedGetter` 函数只是返回一个叫做 `computedGetter` 的函数，并没有做任何其他事情。也就是说计算属性真正的 `get` 拦截器函数就是 `computedGetter` 函数，如下：

```js
sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      watcher.depend()
      return watcher.evaluate()
    }
  },
  set: noop // 没有指定 userDef.set 所以是空函数
}
```

最后在 `defineComputed` 函数中还有一段代码我们没有讲到，如下：

```js
if (process.env.NODE_ENV !== 'production' &&
    sharedPropertyDefinition.set === noop) {
  sharedPropertyDefinition.set = function () {
    warn(
      `Computed property "${key}" was assigned to but it has no setter.`,
      this
    )
  }
}
```

这是一段 `if` 条件语句块，在非生产环境下如果发现 `sharedPropertyDefinition.set` 的值是一个空函数，那么说明开发者并没有为计算属性定义相应的 `set` 拦截器函数，这时会重写 `sharedPropertyDefinition.set` 函数，这样当你在代码中尝试修改一个没有指定 `set` 拦截器函数的计算属性的值时，就会得到一个警告信息。

### 计算属性的实现

以上关于计算属性相关初始化工作已经完成了，初始化计算属性的过程中主要创建了计算属性观察者以及将计算属性定义到组件实例对象上，接下来我们将通过一些例子来分析计算属性是如何实现的，假设我们有如下代码：

```js
data () {
  return {
    a: 1
  }
},
computed: {
  compA () {
    return this.a + 1
  }
}
```

如上代码中，我们定义了本地数据 `data`，它拥有一个响应式的属性 `a`，我们还定义了计算属性 `compA`，它的值将依据 `a` 的值来计算求得。另外我们假设有如下模板：

```html
<div>{{compA}}</div>
```

模板中我们使用到了计算属性，我们知道模板会被编译成渲染函数，渲染函数的执行将触发计算属性 `compA` 的 `get` 拦截器函数，那么 `compA` 的拦截器函数是什么呢？就是我们前面分析的 `sharedPropertyDefinition.get` 函数，我们知道在非服务端渲染的情况下，这个函数为：

```js
sharedPropertyDefinition.get = function computedGetter () {
  const watcher = this._computedWatchers && this._computedWatchers[key]
  if (watcher) {
    watcher.depend()
    return watcher.evaluate()
  }
}
```

也就是说当 `compA` 属性被读取时，`computedGetter` 函数将会执行，在 `computedGetter` 函数内部，首先定义了 `watcher` 常量，它的值为计算属性 `compA` 的观察者对象，紧接着如果该观察者对象存在，则会分别执行观察者对象的 `depend` 方法和 `evaluate` 方法。

我们首先找到 `Watcher` 类的 `depend` 方法，如下：

```js
depend () {
  if (this.dep && Dep.target) {
    this.dep.depend()
  }
}
```

`depend` 方法的内容很简单，检查 `this.dep` 和 `Dep.target` 是否全部有值，如果都有值的情况下便会执行 `this.dep.depend` 方法。这里我们首先要知道 `this.dep` 属性是什么，实际上计算属性的观察者与其他观察者对象不同，不同之处首先会体现在创建观察者实例对象的时候，如下是 `Watcher` 类的 `constructor` 方法中的一段代码：

```js {9-11}
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: ?Object,
  isRenderWatcher?: boolean
) {
  // 省略...
  if (this.computed) {
    this.value = undefined
    this.dep = new Dep()
  } else {
    this.value = this.get()
  }
}
```

如上高亮代码所示，当创建计算属性观察者对象时，由于第四个选项参数中 `options.computed` 为真，所以计算属性观察者对象的 `this.computed` 属性的值也会为真，所以对于计算属性的观察者来讲，在创建时会执行 `if` 条件分支内的代码，而对于其他观察者对象则会执行 `else` 分支内的代码。同时我们能够看到在 `else` 分支内直接调用 `this.get()` 方法求值，而 `if` 分支内并没有调用 `this.get()` 方法求值，而是定义了 `this.dep` 属性，它的值是一个新创建的 `Dep` 实例对象。这说明计算属性的观察者是一个惰性求值的观察者。

现在我们再回到 `Watcher` 类的 `depend` 方法中：

```js {3}
depend () {
  if (this.dep && Dep.target) {
    this.dep.depend()
  }
}
```

此时我们已经知道了 `this.dep` 属性是一个 `Dep` 实例对象，所以 `this.dep.depend()` 这句代码的作用就是用来收集依赖。那么它收集到的东西是什么呢？这就要看 `Dep.target` 属性的值是什么了，我们回想一下整个过程：首先渲染函数的执行会读取计算属性 `compA` 的值，从而触发计算属性 `compA` 的 `get` 拦截器函数，最终调用了 `this.dep.depend()` 方法收集依赖。这个过程中的关键一步就是渲染函数的执行，我们知道在渲染函数执行之前 `Dep.target` 的值必然是 **渲染函数的观察者对象**。所以计算属性观察者对象的 `this.dep` 属性中所收集的就是渲染函数的观察者对象。

记得此时计算属性观察者对象的 `this.dep` 中所收集的是渲染函数观察者对象，假设我们把渲染函数观察者对象称为 `renderWatcher`，那么：

```js
this.dep.subs = [renderWatcher]
```

这样 `computedGetter` 函数中的 `watcher.depend()` 语句我们就讲解完了，但 `computedGetter` 函数还没执行完，接下来要执行的是 `watcher.evaluate()` 语句：

```js {5}
sharedPropertyDefinition.get = function computedGetter () {
  const watcher = this._computedWatchers && this._computedWatchers[key]
  if (watcher) {
    watcher.depend()
    return watcher.evaluate()
  }
}
```

我们找到 `Watcher` 类的 `evaluate` 方法看看它做了哪些事情，如下：

```js
evaluate () {
  if (this.dirty) {
    this.value = this.get()
    this.dirty = false
  }
  return this.value
}
```

我们知道计算属性的观察者是惰性求值，所以在创建计算属性观察者时除了 `watcher.computed` 属性为 `true` 之外，`watcher.dirty` 属性的值也为 `true`，代表着当前观察者对象没有被求值，而 `evaluate` 方法的作用就是用来手动求值的。可以看到在 `evaluate` 方法内部对 `this.dirty` 属性做了真假判断，如果为真则调用观察者对象的 `this.get` 方法求值，同时将`this.dirty` 属性重置为 `false`。最后将求得的值返回：`return this.value`。

这段代码的关键在于求值的这句代码，如下高亮部分所示：

```js {3}
evaluate () {
  if (this.dirty) {
    this.value = this.get()
    this.dirty = false
  }
  return this.value
}
```

我们在计算属性的初始化一节中讲过了，在创建计算属性观察者对象时传递给 `Watcher` 类的第二个参数为 `getter` 常量，它的值就是开发者在定义计算属性时的函数(或 `userDef.get`)，如下高亮代码所示：

```js {5,12}
function initComputed (vm: Component, computed: Object) {
  // 省略...
  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    // 省略...

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // 省略...
  }
}
```

所以在 `evaluate` 方法中求值的那句代码最终所执行的求值函数就是用户定义的计算属性的 `get` 函数。举个例子，假设我们这样定义计算属性：

```js
computed: {
  compA () {
    return this.a +1
  }
}
```

那么对于计算属性 `compA` 来讲，执行其计算属性观察者对象的 `wather.evaluate` 方法求值时，本质上就是执行如下函数进行求值：

```js
compA () {
  return this.a +1
}
```

大家想一想这个函数的执行会发生什么事情？我们知道数据对象的 `a` 属性是响应式的，所以如上函数的执行将会触发属性 `a` 的 `get` 拦截器函数。所以这会导致属性 `a` 将会收集到一个依赖，这个依赖实际上就是计算属性的观察者对象。

现在思路大概明朗了，如果计算属性 `compA` 依赖了数据对象的 `a` 属性，那么属性 `a` 将收集计算属性 `compA` 的 **计算属性观察者对象**，而 **计算属性观察者对象** 将收集 **渲染函数观察者对象**，整个路线是这样的：

![](http://7xlolm.com1.z0.glb.clouddn.com/2018-06-10-074626.png)

假如此时我们修改响应式属性 `a` 的值，那么将触发属性 `a` 所收集的所有依赖，这其中包括计算属性的观察者。我们知道触发某个响应式属性的依赖实际上就是执行该属性所收集到的所有观察者的 `update` 方法，现在我们就找到 `Watcher` 类的 `update` 方法，如下：

```js {3,18}
update () {
  /* istanbul ignore else */
  if (this.computed) {
    // A computed property watcher has two modes: lazy and activated.
    // It initializes as lazy by default, and only becomes activated when
    // it is depended on by at least one subscriber, which is typically
    // another computed property or a component's render function.
    if (this.dep.subs.length === 0) {
      // In lazy mode, we don't want to perform computations until necessary,
      // so we simply mark the watcher as dirty. The actual computation is
      // performed just-in-time in this.evaluate() when the computed property
      // is accessed.
      this.dirty = true
    } else {
      // In activated mode, we want to proactively perform the computation
      // but only notify our subscribers when the value has indeed changed.
      this.getAndInvoke(() => {
        this.dep.notify()
      })
    }
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

如上高亮代码所示，由于响应式数据收集到了计算属性观察者对象，所以当计算属性观察者对象的 `update` 方法被执行时，如上 `if` 语句块的代码将被执行，因为 `this.computed` 属性为真。接着检查了 `this.dep.subs.length === 0` 的真假，我们知道既然是计算属性的观察者，那么 `this.dep` 中将收集渲染函数作为依赖(或其他观察该计算属性变化的观察者对象作为依赖)，所以当依赖的数量不为 `0` 时，在 `else` 语句块内会调用 `this.dep.notify()` 方法继续触发响应，这会导致 `this.dep.subs` 属性中收集到的所有观察者对象的更新，如果此时 `this.dep.subs` 中包含渲染函数的观察者，那么这就会导致重新渲染，最终完成视图的更新。

以上就是计算属性的实现思路，本质上计算属性观察者对象就是一个桥梁，它搭建在响应式数据与渲染函数观察者中间，另外大家注意上面的代码中并非直接调用 `this.dep.notify()` 方法触发响应，而是将这个方法作为 `this.getAndInvoke` 方法的回调去执行的，为什么这么做呢？那是因为 `this.getAndInvoke` 方法会重新求值并对比新旧值是否相同，如果满足相同条件则不会触发响应，只有当值确实变化时才会触发响应，这就是文档中的描述，现在你明白了吧：

![](http://7xlolm.com1.z0.glb.clouddn.com/2018-06-10-080745.png)

## 同步执行观察者

通常情况下当数据状态发生改变时，所有 `Watcher` 都为异步执行，这么做的目的是出于对性能的考虑。但在某些场景下我们仍需要同步执行的观察者，我们可以使用 `sync` 选项定义同步执行的观察者，如下：

```js
new Vue({
  watch: {
    someWatch: {
      handler () {/* ... */},
      sync: true
    }
  }
})
```

如上代码所示，我们在定义一个观察者时使用 `sync` 选项，并将其设置为 `true`，此时当数据状态发生变化时该观察者将以同步的方式执行。这么做当然没有问题，因为我们仅仅定义了一个观察者而已。

`Vue` 官方推出了 [vue-test-utils](https://github.com/vuejs/vue-test-utils) 测试工具库，这个库的一个特点是，当你使用它去辅助测试 `Vue` 单文件组件时，数据变更将会以同步的方式触发组件变更，这对于测试而言会提供很大帮助。大家思考一下 [vue-test-utils](https://github.com/vuejs/vue-test-utils) 库是如何实现这个功能的？我们知道开发者在开发组件的时候基本不太可能手动地指定一个观察者为同步的，所以 [vue-test-utils](https://github.com/vuejs/vue-test-utils) 库需要有能力拿到组件的定义并人为地把组件中定义的所有观察者都转换为同步的，这是一个繁琐并容易引起 `bug` 的工作，为了解决这个问题，`Vue` 提供了 `Vue.config.async` 全局配置，它的默认值为 `true`，我们可以在 `src/core/config.js` 文件中看到这样一句代码，如下：

```js {8}
export default ({
  // 省略...

  /**
   * Perform updates asynchronously. Intended to be used by Vue Test Utils
   * This will significantly reduce performance if set to false.
   */
  async: true,

  // 省略...
}: Config)
```

这个全局配置将决定 `Vue` 中的观察者以何种方式执行，默认是异步执行的，当我们将其修改为 `Vue.config.async = false` 时，所有观察者都将会同步执行。其实现方式很简单，我们打开 `src/core/observer/scheduler.js` 文件，找到 `queueWatcher` 函数：

```js {9-12}
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    // 省略...
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

如上高亮代码所示，在非生产环境下如果 `!config.async` 为真，则说明开发者配置了 `Vue.config.async = false`，这意味着所有观察者需要同步执行，所以只需要把原本通过 `nextTick` 包装的 `flushSchedulerQueue` 函数单独拿出来执行即可。另外通过如上高亮的代码我们也能够明白一件事儿，那就是 `Vue.config.async` 这个配置项只会在非生产环境生效。

为了实现同步执行的观察者，除了把 `flushSchedulerQueue` 函数从 `nextTick` 中提取出来之外，还需要做一件事儿，我们打开 `src/core/observer/dep.js` 文件，找到 `notify` 方法，如下：

```js {4-9}
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  if (process.env.NODE_ENV !== 'production' && !config.async) {
    // subs aren't sorted in scheduler if not running async
    // we need to sort them now to make sure they fire in correct
    // order
    subs.sort((a, b) => a.id - b.id)
  }
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

在异步执行观察者的时候，当数据状态方式改变时，会通过如上 `notify` 函数通知变化，从而执行所有观察者的 `update` 方法，在 `update` 方法内会将所有即将被执行的观察者都添加到观察者队列中，并在 `flushSchedulerQueue` 函数内对观察者回调的执行顺序进行排序。但是当同步执行的观察者时，由于 `flushSchedulerQueue` 函数是立即执行的，它不会等待所有观察者入队之后再去执行，这就没有办法保证观察者回调的正确更新顺序，这时就需要如上高亮的代码，其实现方式是在执行观察者对象的 `update` 更新方法之前就对观察者进行排序，从而保证正确的更新顺序。
