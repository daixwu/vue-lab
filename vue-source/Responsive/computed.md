# computed 计算属性的实现

上一节关于计算属性相关初始化工作已经完成了，初始化计算属性的过程中主要创建了计算属性观察者以及将计算属性定义到组件实例对象上，接下来我们将通过一些例子来分析计算属性是如何实现的，假设我们有如下代码：

```js
<div>{{ fullName }}</div>

data: {
  firstName: 'Dai',
  lastName: 'Xwu'
},
computed: {
  fullName: function () {
    return this.firstName + ' ' + this.lastName
  }
}
```

当初始化这个 computed watcher 实例的时候，构造函数部分逻辑稍有不同：

```js
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: ?Object,
  isRenderWatcher?: boolean
) {
  // ...
  this.value = this.lazy
    ? undefined
    : this.get()
}
```

可以发现 computed watcher 会并不会立刻求值。

然后当我们的 render 函数执行访问到 `this.fullName` 的时候，就触发了计算属性的 getter，就是我们前面分析的 `sharedPropertyDefinition.get` 函数，我们知道在非服务端渲染的情况下，这个函数为：

```js
sharedPropertyDefinition.get = function computedGetter () {
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
```

它会拿到计算属性对应的 watcher，然后会首先执行 `watcher.evaluate()`，来看一下它的定义：

我们找到 `Watcher` 类的 `evaluate` 方法看看它做了哪些事情，如下：

```js
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

`evaluate` 的逻辑非常简单，首先是通过 `this.get()` 求值，然后把 `this.dirty` 设置为 `false`。

再看一下 get 方法：

```js
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

注意每次 get 都会先执行 pushTarget 方法，所以现在 `Dep.target` 是当前这个 `computed Watcher`。然后它会执行 `this.getter.call(vm, vm)` 这里就相当于调用了计算属性函数做求值由于我们知道计算属性往往依赖了 data、props 甚至是其它计算属性的数据所以这个时候会触发这些依赖的 getter 会把这些依赖收集到当前这个 `computed Watcher` 中。在这个收集过程中会触发 Dep 中的 depend 方法

```js
depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}
```

其实就是执行 computed watcher 中的 addDep 方法

```js
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
```

新的依赖会添加到 newDeps 中。

再回到 get 方法执行完 `this.getter.call(vm, vm)` 后会执行 pop Target这个时候 `Dep.target` 又回到了 render watcher 了接着又执行 cleanupDeps 方法：

```js
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
```

这个过程中会把新收集的依赖更新到 `this.deps` 中，所以 `computed watcher` 的 deps 就是在执行计算属性函数求值过程中的依赖。

至此计算属性的求值完成依赖收集过程完成。接下来看第二个步骤

```js
if (Dep.target) {
  watcher.depend()
}
```

把计算属性中的依赖收集到当前渲染 watcher 中。那么为啥要执行这个步骤我们要想一个问题假设我们的模板渲染过程中只访问了计算属性值触发了计算属性的 Getter 执行了我们上述一系列操作那么当计算属性的依赖发生变化也只会触发 computed watcher 的 update

```js
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
```

update 也仅仅是把 `this.dirty` 设置为 false 并不会触发计算属性的重新计算也不会让页面重新渲染。所以我们需要把计算属性中的依赖收集到当前渲染 watcher 中。这样一旦计算属性的依赖发生变化就会触发 render watcher 的 update 就会触发重新渲染在重新渲染的过程中会再次访问到计算属性的 getter。然后又回到最初:

```js
function computedGetter () {
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
```

因为计算属性的依赖更新触发了 computed watcher 的 update，dirty 为 true所以又触发了计算属性的重新计算。

这就是为什么当计算属性的依赖不改变计算属性就会用缓存的值只有它依赖发生了变化它才会重新计算。
