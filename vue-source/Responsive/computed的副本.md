# computed 计算属性的实现

上一节关于计算属性相关初始化工作已经完成了，初始化计算属性的过程中主要创建了计算属性观察者以及将计算属性定义到组件实例对象上，接下来我们将通过一些例子来分析计算属性是如何实现的，假设我们有如下代码：

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

也就是说当 `compA` 属性被读取时，`computedGetter` 函数将会执行，在 `computedGetter` 函数内部，首先定义了 `watcher` 常量，它的值为计算属性 `compA` 的观察者对象，紧接着是两个 if 判断，我们知道计算属性的观察者是惰性求值，所以在创建计算属性观察者时除了 `watcher.lazy` 属性为 `true` 之外，`watcher.dirty` 属性的值也为 `true`，所以此处将会调用 `evaluate` 方法。

我们找到 `Watcher` 类的 `evaluate` 方法看看它做了哪些事情，如下：

```js
evaluate () {
  if (this.dirty) {
    this.value = this.get()
    this.dirty = false
  }
  return this.value
}

evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

`evaluate` 方法的作用就是用来手动求值的。可以看到在 `evaluate` 方法内部首先是调用观察者对象的 `this.get` 方法求值，并将其赋给 `this.value`,同时将`this.dirty` 属性重置为 `false`。

这段代码的关键在于求值的这句代码，如下所示：

```js
this.value = this.get()
```

我们在计算属性的初始化一节中讲过了，在创建计算属性观察者对象时传递给 `Watcher` 类的第二个参数为 `getter` 常量，它的值就是开发者在定义计算属性时的函数(或 `userDef.get`)，如下高亮代码所示：

```js
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

现在思路大概明朗了，如果计算属性 `compA` 依赖了数据对象的 `a` 属性，那么属性 `a` 将收集计算属性 `compA` 的 **计算属性观察者对象**，而 **计算属性观察者对象** 将收集 **渲染函数观察者对象**。

假如此时我们修改响应式属性 `a` 的值，那么将触发属性 `a` 所收集的所有依赖，这其中包括计算属性的观察者。我们知道触发某个响应式属性的依赖实际上就是执行该属性所收集到的所有观察者的 `update` 方法，现在我们就找到 `Watcher` 类的 `update` 方法，如下：

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

如上高亮代码所示，由于响应式数据收集到了计算属性观察者对象，所以当计算属性观察者对象的 `update` 方法被执行时，如上 `if` 语句块的代码将被执行，因为 `this.lazy` 属性为真，此处便会将 `this.dirty` 属性重置为 `true`。
