# 初始化 data 做的事情

从`src/core/instance/init.js`中我们可以看到在初始化的过程中有执行`initState()`函数，此函数定义在 `src/core/instance/state.js` 中

```js
function initState (vm: Component) {
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

这里会根据用户的 options 中如果有定义 props、methods、data、 computed、watcher 则对其进行初始化，这里我们重点分析 `initData()` 的执行过程。

```js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  observe(data, true /* asRootData */)
}
```

```js
function getData (data: Function, vm: Component): any {
  // #7573 disable dep collection when invoking data getters
  pushTarget()
  try {
    return data.call(vm, vm)
  } catch (e) {
    handleError(e, vm, `data()`)
    return {}
  } finally {
    popTarget()
  }
}
```

这里通过 `getData()` 方法拿到 options 里的 data 数据，并将其赋值给了 `vm._data`,这个会在之后的 proxy 方法中会用到。（注：vue中不允许 data、props、methods 中存在相同的 key，因为它们最终都会挂载到 vue 实例上）

```js
const sharedPropertyDefinition = {
  enumerable: true,
  configurable: true,
  get: noop,
  set: noop
}

function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

通过 proxy 做了一层代理，通过 [Object.defineProperty](https://blog.csdn.net/weixin_33877092/article/details/91455994) 方法实现 用户在调用 vm.key 的时候实质上是在调用 vm._data.key

