# computed 计算属性的初始化

计算属性本质上是一个惰性求值的观察者。它的初始化是发生在 Vue 实例初始化阶段的 initState 函数中：

```js
export function initState (vm: Component) {
  // ...
  if (opts.computed) initComputed(vm, opts.computed)
  // ...
}
```

这句代码首先检查开发者是否传递了 computed 选项，只有传递了该选项的情况下才会调用 initComputed 函数进行初始化，initComputed 的定义在 src/core/instance/state.js 中：

```js
const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {
  // $flow-disable-line
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
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
const computedWatcherOptions = { lazy: true }
```

我们知道传递给 `Watcher` 类的第四个参数是观察者的选项参数，选项参数对象可以包含如 `deep`、`sync` 等选项，当然了其中也包括 `lazy` 选项，通过如上这句代码可知在创建计算属性观察者对象时 `lazy` 选项为 `true`，它的作用就是用来标识一个观察者对象是计算属性的观察者，计算属性的观察者与非计算属性的观察者的行为是不一样的。

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

```js
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

export function defineComputed (
  target: any,
  key: string,
  userDef: Object | Function
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
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
    : createGetterInvoker(userDef)
  sharedPropertyDefinition.set = noop
} else {
  sharedPropertyDefinition.get = userDef.get
    ? shouldCache && userDef.cache !== false
      ? createComputedGetter(key)
      : createGetterInvoker(userDef.get)
    : noop
  sharedPropertyDefinition.set = userDef.set || noop
}
```

这段代码无论 `userDef` 是函数还是对象，在非服务端渲染的情况下，配置对象 `sharedPropertyDefinition` 最终将变成如下这样：

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
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
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
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
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
