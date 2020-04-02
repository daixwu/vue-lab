# Watcher 构造函数对 options 的处理

Watcher 的构造函数对 options 做了处理，代码如下：

```js
if (options) {
  this.deep = !!options.deep
  this.user = !!options.user
  this.lazy = !!options.lazy
  this.sync = !!options.sync
  // ...
} else {
  this.deep = this.user = this.computed = this.sync = false
}
```

所以 watcher 总共有 4 种类型，我们来一一分析它们，看看不同的类型执行的逻辑有哪些差别。

## deep watcher

通常，如果我们想对一下对象做深度观测的时候，需要设置这个属性为 true，考虑到这种情况：

```js
var vm = new Vue({
  data() {
    a: {
      b: 1
    }
  },
  watch: {
    a: {
      handler(newVal) {
        console.log(newVal)
      }
    }
  }
})

vm.a.b = 2
```

这个时候是不会 log 任何数据的，因为我们是 watch 了 a 对象，只触发了 a 的 getter，并没有触发 a.b 的 getter，所以并没有订阅它的变化，导致我们对 `vm.a.b = 2` 赋值的时候，虽然触发了 setter，但没有可通知的对象，所以也并不会触发 watch 的回调函数了。

而我们只需要对代码做稍稍修改，就可以观测到这个变化了

```js
watch: {
  a: {
    deep: true,
    handler(newVal) {
      console.log(newVal)
    }
  }
}
```

这样就创建了一个 deep watcher 了，在 watcher 执行 get 求值的过程中有一段逻辑：

```js
get() {
  // ...
  if (this.deep) {
    traverse(value)
  }
  // ...
}
```

在对 watch 的表达式或者函数求值后，会调用 traverse 函数，它的定义在 src/core/observer/traverse.js 中：

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

上面的代码中定义了 `traverse` 函数，这个函数将接收被观察属性的值作为参数，拿到这个参数后在 `traverse` 函数内部会调用 `_traverse` 函数完成递归遍历。其中 `_traverse` 函数就定义在 `traverse` 函数的下方，如下：

```js
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

`_traverse` 函数接收两个参数，第一个参数是被观察属性的值，第二个参数是一个 `Set` 数据结构的实例，可以看到在 `traverse` 函数中调用 `_traverse` 函数时传递的第二个参数 `seenObjects` 就是一个 `Set` 数据结构的实例，它定义在文件头部：`const seenObjects = new Set()`。

接下来我们看一下 `_traverse` 函数是如何遍历访问数据对象的，我们先看下面一段简化代码：

```js
function _traverse (val: any, seen: SimpleSet) {
  let i, keys
  const isA = Array.isArray(val)
  if ((!isA && !isObject(val)) || Object.isFrozen(val) || val instanceof VNode) {
    return
  }
  // ...
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

来看一下 `_traverse` 函数的实现，在 `_traverse` 函数的开头声明了两个变量，分别是 `i` 和 `keys`，这两个变量在后面会使用到，接着检查参数 `val` 是不是数组，并将检查结果存储在常量 `isA` 中。再往下是一段 `if` 语句块：

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

现在 `_traverse` 函数的代码我们就解析完了，再来看下我们前面省略的代码，如下：

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

上面代码中我们定义了两个对象，分别是 `obj1` 和 `obj2`，并且 `obj1.data` 属性引用了 `obj2`，而 `obj2.data` 属性引用了 `obj1`，这是一个典型的循环引用，假如我们使用 `obj1` 或 `obj2` 这两个对象中的任意一个对象出现在 `Vue` 的响应式数据中，如果不做防循环引用的处理，将会导致死循环。

为了避免这种情况的发生，我们可以使用一个变量来存储那些已经被遍历过的对象，当再次遍历该对象时程序会发现该对象已经被遍历过了，这时会跳过遍历，从而避免死循环，如下代码：

```js
if (val.__ob__) {
  const depId = val.__ob__.dep.id
  if (seen.has(depId)) {
    return
  }
  seen.add(depId)
}
```

这里 `if` 语句块，用来判断 `val.__ob__` 是否有值，我们知道如果一个响应式数据是对象或数组，那么它会包含一个叫做 `__ob__` 的属性，这时我们读取 `val.__ob__.dep.id` 作为一个唯一的ID值，并将它放到 `seenObjects` 中：`seen.add(depId)`，这样即使 `val` 是一个拥有循环引用的对象，当下一次遇到该对象时，我们能够发现该对象已经遍历过了：`seen.has(depId)`，这样函数直接 `return` 即可。

以上就是深度观测的实现以及避免循环引用造成的死循环的解决方案。

那么在执行了 traverse 后，我们再对 watch 的对象内部任何一个值做修改，也会调用 watcher 的回调函数了。

对 deep watcher 的理解非常重要，今后工作中如果大家观测了一个复杂对象，并且会改变对象内部深层某个值的时候也希望触发回调，一定要设置 deep 为 true，但是因为设置了 deep 后会执行 traverse 函数，会有一定的性能开销，所以一定要根据应用场景权衡是否要开启这个配置。

## user watcher

前面我们分析过，通过 `vm.$watch` 创建的 watcher 是一个 user watcher，其实它的功能很简单，在对 watcher 求值以及在执行回调函数的时候，会处理一下错误，如下：

```js
get () {
  // ...
  if (this.user) {
    handleError(e, vm, `getter for watcher "${this.expression}"`)
  } else {
    throw e
  }
  // ...
},

run () {
  // ...
  if (this.user) {
    try {
      this.cb.call(this.vm, value, oldValue)
    } catch (e) {
      handleError(e, this.vm, `callback for watcher "${this.expression}"`)
    }
  } else {
    this.cb.call(this.vm, value, oldValue)
  }
  // ...
}
```

handleError 在 Vue 中是一个错误捕获并且暴露给用户的一个利器。

## lazy watcher

lazy watcher 几乎就是为计算属性量身定制的，我们刚才已经对它做了详细的分析，这里不再赘述了。

## sync watcher

在我们之前对 setter 的分析过程知道，当响应式数据发送变化后，触发了 `watcher.update()`，只是把这个 watcher 推送到一个队列中，在 nextTick 后才会真正执行 watcher 的回调函数。而一旦我们设置了 sync，就可以在当前 Tick 中同步执行 watcher 的回调函数。

```js
update () {
  if (this.computed) {
    // ...
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

只有当我们需要 watch 的值的变化到执行 watcher 的回调函数是一个同步过程的时候才会去设置该属性为 true。
