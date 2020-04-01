# Watcher 构造函数

`Watcher` 类定义在 `src/core/observer/watcher.js` 文件中，如下是 `Watcher` 类的全部内容：

```js
export default class Watcher {
  vm: Component;
  expression: string;
  cb: Function;
  id: number;
  deep: boolean;
  user: boolean;
  lazy: boolean;
  sync: boolean;
  dirty: boolean;
  active: boolean;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  before: ?Function;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // options
    if (options) {
      this.deep = !!options.deep
      this.user = !!options.user
      this.lazy = !!options.lazy
      this.sync = !!options.sync
      this.before = options.before
    } else {
      this.deep = this.user = this.lazy = this.sync = false
    }
    this.cb = cb
    this.id = ++uid // uid for batching
    this.active = true
    this.dirty = this.lazy // for lazy watchers
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.expression = process.env.NODE_ENV !== 'production'
      ? expOrFn.toString()
      : ''
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
      if (!this.getter) {
        this.getter = noop
        process.env.NODE_ENV !== 'production' && warn(
          `Failed watching path: "${expOrFn}" ` +
          'Watcher only accepts simple dot-delimited paths. ' +
          'For full control, use a function instead.',
          vm
        )
      }
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
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

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      const value = this.get()
      if (
        value !== this.value ||
        // Deep watchers and watchers on Object/Arrays should fire even
        // when the value is the same, because the value may
        // have mutated.
        isObject(value) ||
        this.deep
      ) {
        // set new value
        const oldValue = this.value
        this.value = value
        if (this.user) {
          try {
            this.cb.call(this.vm, value, oldValue)
          } catch (e) {
            handleError(e, this.vm, `callback for watcher "${this.expression}"`)
          }
        } else {
          this.cb.call(this.vm, value, oldValue)
        }
      }
    }
  }

  /**
   * Evaluate the value of the watcher.
   * This only gets called for lazy watchers.
   */
  evaluate () {
    this.value = this.get()
    this.dirty = false
  }

  /**
   * Depend on all deps collected by this watcher.
   */
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }

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

通过 `Watcher` 类的 `constructor` 方法可以知道在创建 `Watcher` 实例时可以传递五个参数，分别是：组件实例对象 `vm`、要观察的表达式 `expOrFn`、当被观察的表达式的值变化时的回调函数 `cb`、一些传递给当前观察者对象的选项 `options` 以及一个布尔值 `isRenderWatcher` 用来标识该观察者实例是否是渲染函数的观察者。

之前我们介绍当对数据对象的访问会触发他们的 getter 方法，那么这些对象什么时候被访问呢？还记得之前我们介绍过 Vue 的 mount 过程是通过 mountComponent 函数，其中有一段比较重要的逻辑，大致如下：

```js
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}

new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

可以看到在创建渲染函数观察者实例对象时传递了全部的五个参数，第一个参数 `vm` 很显然就是当前组件实例对象；第二个参数 `updateComponent` 就是被观察的目标，它是一个函数；第三个参数 `noop` 是一个空函数；第四个参数是一个包含 `before` 函数的对象，这个对象将作为传递给该观察者的选项；第五个参数为 `true`，我们知道这个参数标识着该观察者实例对象是否是渲染函数的观察者，很显然上面的代码是在为渲染函数创建观察者对象，所以第五个参数自然为 `true`。

这里有几个问题需要注意，首先被观察的表达式是一个函数，即 `updateComponent` 函数，我们知道 `Watcher` 的原理是通过对“被观测目标”的求值，触发数据属性的 `get` 拦截器函数从而收集依赖，至于“被观测目标”到底是表达式还是函数或者是其他形式的内容都不重要，重要的是“被观测目标”能否触发数据属性的 `get` 拦截器函数，很显然函数是具备这个能力的。另外一个我们需要注意的是传递给 `Watcher` 构造函数的第三个参数 `noop` 是一个空函数，它什么事情都不会做，有的同学可能会有疑问：“不是说好了当数据变化时重新渲染吗，现在怎么什么都不做了？”，实际上数据的变化不仅仅会执行回调，还会重新对“被观察目标”求值，也就是说 `updateComponent` 也会被调用，所以不需要通过执行回调去重新渲染。说到这里大家或许又产生了一个疑问：“再次执行 `updateComponent` 函数难道不会导致再次触发数据属性的 `get` 拦截器函数导致重复收集依赖吗？”，这是个好问题，不过不用担心，因为 `Vue` 已经实现了避免收集重复依赖的处理，我们后面会讲到的。

接下来我们就从 `constructor` 函数开始，看一下创建渲染函数观察者实例对象的过程，进一步了解一个观察者，如下是 `constructor` 函数开头的一段代码：

```js
this.vm = vm
if (isRenderWatcher) {
  vm._watcher = this
}
vm._watchers.push(this)
```

首先将当前组件实例对象 `vm` 赋值给该观察者实例的 `this.vm` 属性，也就是说每一个观察者实例对象都有一个 `vm` 实例属性，该属性指明了这个观察者是属于哪一个组件的。接着使用 `if` 条件语句判断 `isRenderWatcher` 是否为真，前面说过 `isRenderWatcher` 标识着是否是渲染函数的观察者，只有在 `mountComponent` 函数中创建渲染函数观察者时这个参数为真，如果 `isRenderWatcher` 为真那么则会将当前观察者实例赋值给 `vm._watcher` 属性，也就是说组件实例的 `_watcher` 属性的值引用着该组件的渲染函数观察者。大家还记得 `_watcher` 属性是在哪里初始化的吗？是在 `initLifecycle` 函数中被初始化的，其初始值为 `null`。在 `if` 语句块的后面将当前观察者实例对象 `push` 到 `vm._watchers` 数组中，也就是说属于该组件实例的观察者都会被添加到该组件实例对象的 `vm._watchers` 数组中，包括渲染函数的观察者和非渲染函数的观察者。另外组件实例的
 `vm._watchers` 属性是在 `initState` 函数中初始化的，其初始值是一个空数组。

再往下是这样一段代码：

```js
if (options) {
  this.deep = !!options.deep
  this.user = !!options.user
  this.lazy = !!options.lazy
  this.sync = !!options.sync
  this.before = options.before
} else {
  this.deep = this.user = this.lazy = this.sync = false
}
```

这是一个 `if...else` 语句块，判断是否传递了 `options` 参数，如果没有传递则 `else` 语句块的代码将被执行，可以看到在 `else` 语句块内将当前观察者实例对象的四个属性 `this.deep`、`this.user`、`this.lazy` 以及 `this.sync` 全部初始化为 `false`。如果传递了 `options` 参数，那么这四个属性的值则会使用 `options` 对象中同名属性值的真假来初始化。通过 `if` 语句块内的代码我们可以知道在创建一个观察者对象时，可以传递五个选项，分别是：

* `options.deep`，用来告诉当前观察者实例对象是否是深度观测

我们平时在使用 `Vue` 的 `watch` 选项或者 `vm.$watch` 函数去观测某个数据时，可以通过设置 `deep` 选项的值为 `true` 来深度观测该数据。

* `options.user`，用来标识当前观察者实例对象是 **开发者定义的** 还是 **内部定义的**

实际上无论是 `Vue` 的 `watch` 选项还是 `vm.$watch` 函数，他们的实现都是通过实例化 `Watcher` 类完成的，等到我们讲解 `Vue` 的 `watch` 选项和 `vm.$watch` 的具体实现时大家会看到，除了内部定义的观察者(如：渲染函数的观察者、计算属性的观察者等)之外，所有观察者都被认为是开发者定义的，这时 `options.user` 会自动被设置为 `true`。

* `options.computed`，用来标识当前观察者实例对象是否是计算属性的观察者

这里需要明确的是，计算属性的观察者并不是指一个观察某个计算属性变化的观察者，而是指 `Vue` 内部在实现计算属性这个功能时为计算属性创建的观察者。等到我们讲解计算属性的实现时再详细说明。

* `options.sync`，用来告诉观察者当数据变化时是否同步求值并执行回调

默认情况下当数据变化时不会同步求值并执行回调，而是将需要重新求值并执行回调的观察者放到一个异步队列中，当所有数据的变化结束之后统一求值并执行回调，这么做的好处有很多，我们后面会详细讲解。

* `options.before`，可以理解为 `Watcher` 实例的钩子，当数据变化之后，触发更新之前，调用在创建渲染函数的观察者实例对象时传递的 `before` 选项。

如下代码：

```js
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```

可以看到当数据变化之后，触发更新之前，如果 `vm._isMounted` 属性的值为真，则会调用 `beforeUpdate` 生命周期钩子。

再往下又定义了一些实例属性，如下：

```js
this.cb = cb
this.id = ++uid // uid for batching
this.active = true
this.dirty = this.computed // for computed watchers
```

如上代码所示，定义了 `this.cb` 属性，它的值为 `cb` 回调函数。定义了 `this.id` 属性，它是观察者实例对象的唯一标识。定义了 `this.active` 属性，它标识着该观察者实例对象是否是激活状态，默认值为 `true` 代表激活。定义了 `this.dirty` 属性，该属性的值与 `this.computed` 属性的值相同，也就是说只有计算属性的观察者实例对象的 `this.dirty` 属性的值才会为真，因为计算属性是惰性求值。

接着往下看代码，如下：

```js
this.deps = []
this.newDeps = []
this.depIds = new Set()
this.newDepIds = new Set()
```

这四个属性两两一组，`this.deps` 与 `this.depIds` 为一组，`this.newDeps` 与 `this.newDepIds` 为一组。那么这两组属性的作用是什么呢？其实它们就是传说中用来实现避免收集重复依赖，且移除无用依赖的功能也依赖于它们，后面我们会详细讲解，现在大家注意一下这四个属性的数据结构，其中 `this.deps` 与 `this.newDeps` 被初始化为空数组，而 `this.depIds` 与 `this.newDepIds` 被初始化为 `Set` 实例对象。

再往下是这句代码：

```js
this.expression = process.env.NODE_ENV !== 'production'
  ? expOrFn.toString()
  : ''
```

定义了 `this.expression` 属性，在非生产环境下该属性的值为表达式(`expOrFn`)的字符串表示，在生产环境下其值为空字符串。所以可想而知 `this.expression` 属性肯定是在非生产环境下使用的，后面我们遇到了再说。

再往下，来到一段 `if...else` 语句块：

```js
if (typeof expOrFn === 'function') {
  this.getter = expOrFn
} else {
  this.getter = parsePath(expOrFn)
  if (!this.getter) {
    this.getter = function () {}
    process.env.NODE_ENV !== 'production' && warn(
      `Failed watching path: "${expOrFn}" ` +
      'Watcher only accepts simple dot-delimited paths. ' +
      'For full control, use a function instead.',
      vm
    )
  }
}
```

这段代码检测了 `expOrFn` 的类型，如果 `expOrFn` 是函数，那么直接使用 `expOrFn` 作为 `this.getter` 属性的值。如果 `expOrFn` 不是函数，那么将 `expOrFn` 透传给 `parsePath` 函数，并以 `parsePath` 函数的返回值作为 `this.getter` 属性的值。那么 `parsePath` 函数做了什么呢？`parsePath` 函数定义在 `src/core/util/lang.js` 文件，源码如下：

```js
const bailRE = /[^\w.$]/
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

首先我们需要知道 `parsePath` 函数接收的参数是什么，如下是平时我们使用 `$watch` 函数的例子：

```js
// 函数
const expOrFn = function () {
  return this.obj.a
}
this.$watch(expOrFn, function () { /* 回调 */ })

// 表达式
const expOrFn = 'obj.a'
this.$watch(expOrFn, function () { /* 回调 */ })
```

以上两种用法实际上是等价的，当 `expOrFn` 不是函数时，比如上例中的 `'obj.a'` 是一个字符串，这时便会将该字符串传递给 `parsePath` 函数，其实我们可以看到 `parsePath` 函数的返回值是另一个函数，那么返回的新函数的作用是什么呢？很显然其作用是触发 `'obj.a'` 的 `get` 拦截器函数，同时新函数会将 `'obj.a'` 的值返回。

接下来我们具体看一下 `parsePath` 函数的具体实现，首先来看一下在 `parsePath` 函数之前定义的 `bailRE` 正则：

```js
const bailRE = /[^\w.$]/
```

同时在 `parsePath` 函数开头有一段 `if` 语句，使用该正则来匹配传递给 `parsePath` 的参数 `path`，如果匹配则直接返回(`return`)，且返回值是 `undefined`，也就是说如果 `path` 匹配正则 `bailRE` 那么最终 `this.getter` 将不是一个函数而是 `undefined`。那么这个正则是什么含义呢？这个正则将匹配一个位置，该位置满足三个条件：

* 不是 `\w`，也就是说这个位置不能是 `字母` 或 `数字` 或 `下划线`
* 不是字符 `.`
* 不是字符 `$`

举几个例子如 `obj~a`、`obj/a`、`obj*a`、`obj+a` 等，这些字符串中的 `~`、`/`、`*` 以及 `+` 字符都能成功匹配正则 `bailRE`，这时 `parsePath` 函数将返回 `undefined`，也就是解析失败。实际上这些字符串在 `javascript` 中不是一个合法的访问对象属性的语法，按照 `bailRE` 正则只有如下这几种形式的字符串才能解析成功：`obj.a`、`this.$watch` 等，看到这里你也应该知道为什么 `bailRE` 正则中包含字符 `.` 和 `$`。

回过头来，如果参数 `path` 不满足正则 `bailRE`，那么如下代码将被执行：

```js
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

首先定义 `segments` 常量，它的值是通过字符 `.` 分割 `path` 字符串产生的数组，随后 `parsePath` 函数将返回值一个函数，该函数的作用是遍历 `segments` 数组循环访问 `path` 指定的属性值。这样就触发了数据属性的 `get` 拦截器函数。但要注意 `parsePath` 返回的新函数将作为 `this.getter` 的值，只有当 `this.getter` 被调用的时候，这个函数才会执行。

看完了 `parsePath` 函数，我们再回到如下这段代码中：

```js
if (typeof expOrFn === 'function') {
  this.getter = expOrFn
} else {
  this.getter = parsePath(expOrFn)
  if (!this.getter) {
    this.getter = function () {}
    process.env.NODE_ENV !== 'production' && warn(
      `Failed watching path: "${expOrFn}" ` +
      'Watcher only accepts simple dot-delimited paths. ' +
      'For full control, use a function instead.',
      vm
    )
  }
}
```

现在我们明白了观察者实例对象的 `this.getter` 函数终将会是一个函数，如果不是函数，如上代码所示。此时只有一种可能，那就是 `parsePath` 函数在解析表达式的时候失败了，那么这时在非生产环境会打印警告信息，告诉开发者：**`Watcher` 只接受简单的点(`.`)分隔路径，如果你要用全部的 `js` 语法特性直接观察一个函数即可**。

再往下我们来到了 `constructor` 函数的最后一段代码：

```js
this.value = this.lazy
  ? undefined
  : this.get()
```

通过这段代码我们可以发现，计算属性的观察者和其他观察者实例对象的处理方式是不同的，对于计算属性的观察者我们会在讲解计算属性时详细说明。除计算属性的观察者之外的所有观察者实例对象都将执行如上代码的 `else` 分支语句，即调用 `this.get()` 方法。
