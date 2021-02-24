# queueWatcher 异步更新队列

接下来我们就聊一聊 `Vue` 中的异步更新队列。在上一节中我们讲解了触发依赖的过程，举个例子如下：

```html
<div id="app">
  <p>{{name}}</p>
</div>

<script>
  new Vue({
    el: '#app',
    data: {
      name: ''
    },
    mounted () {
      this.name = 'hcy'
    }
  })
</script>
```

如上代码所示，我们在模板中使用了数据对象的 `name` 属性，这意味着 `name` 属性将会收集渲染函数的观察者作为依赖，接着我们在 `mounted` 钩子中修改了 `name` 属性的值，这样就会触发响应：**渲染函数的观察者会重新求值，完成重渲染**。从属性值的变化到完成重新渲染，这是一个同步更新的过程，大家思考一下“同步更新”会导致什么问题？很显然这会导致每次属性值的变化都会引发一次重新渲染，假设我们要修改两个属性的值，那么同步更新将导致两次的重渲染，有时候这是致命的缺陷，想象一下复杂业务场景，你可能会同时修改很多属性的值，如果每次属性值的变化都要重新渲染，就会导致严重的性能问题，而异步更新队列就是用来解决这个问题的。

接下来我们就从具体代码入手，看一看其具体实现，我们知道当修改一个属性的值时，会通过执行该属性所收集的所有观察者对象的 `update` 方法进行更新，那么我们就找到观察者对象的 `update` 方法，如下：

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

如上代码所示，如果没有指定这个观察者是同步更新(`this.sync` 为真)，那么这个观察者的更新机制就是异步的，这时当调用观察者对象的 `update` 方法时，在 `update` 方法内部会调用 `queueWatcher` 函数，并将当前观察者对象作为参数传递，`queueWatcher` 函数的作用就是我们前面讲到过的，它将观察者放到一个队列中等待所有突变完成之后统一执行更新。

`queueWatcher` 函数来自 `src/core/observer/scheduler.js` 文件，如下是 `queueWatcher` 函数的全部代码：

```js
const queue: Array<Watcher> = []
let has: { [key: number]: ?true } = {}
let waiting = false
let flushing = false

export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
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

`queueWatcher` 函数接收观察者对象作为参数，首先定义了 `id` 常量，它的值是观察者对象的唯一 `id`，然后执行 `if` 判断语句，如下是简化的代码：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 省略...
  }
}
```

其中变量 `has` 定义在 `scheduler.js` 文件头部，它是一个空对象：

```js
let has: { [key: number]: ?true } = {}
```

当 `queueWatcher` 函数被调用之后，会尝试将该观察者放入队列中，并将该观察者的 `id` 值登记到 `has` 对象上作为 `has` 对象的属性同时将该属性值设置为 `true`。该 `if` 语句以及变量 `has` 的作用就是用来避免将相同的观察者重复入队的。在该 `if` 语句块内执行了真正的入队操作：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // 省略...
    }
    // 省略...
  }
}
```

其中 `queue` 常量也定义在 `scheduler.js` 文件的头部：

```js
const queue: Array<Watcher> = []
```

`queue` 常量是一个数组，入队就是调用该数组的 `push` 方法将观察者添加到数组的尾部。在入队之前有一个对变量 `flushing` 的判断，`flushing` 变量也定义在 `scheduler.js` 文件的头部，它的初始值是 `false`：

```js
let flushing = false
```

`flushing` 变量是一个标志，我们知道放入队列 `queue` 中的所有观察者将会在突变完成之后统一执行更新，当更新开始时会将 `flushing` 变量的值设置为 `true`，代表着此时正在执行更新，所以根据判断条件 `if (!flushing)` 可知只有当队列没有执行更新时才会简单地将观察者追加到队列的尾部，有的同学可能会问：“难道在队列执行更新的过程中还会有观察者入队的操作吗？”，实际上是会的，典型的例子就是计算属性，比如队列执行更新时经常会执行渲染函数观察者的更新，渲染函数中很可能有计算属性的存在，由于计算属性在实现方式上与普通响应式属性有所不同，所以当触发计算属性的 `get` 拦截器函数时会有观察者入队的行为，这个时候我们需要特殊处理，也就是 `else` 分支的代码，如下：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 省略...
  }
}
```

当变量 `flushing` 为真时，说明队列正在执行更新，这时如果有观察者入队则会执行 `else` 分支中的代码，这段代码的作用是为了保证观察者的执行顺序，现在大家只需要知道观察者会被放入 `queue` 队列中即可，我们后面会详细讨论。

接着我们再来看如下代码：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
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

大家观察如上代码中有这样一段 `if` 条件语句：

```js
if (process.env.NODE_ENV !== 'production' && !config.async) {
  flushSchedulerQueue()
  return
}
```

在接下来的讲解中我们将会忽略这段代码，并后面章节中补充讲解，

我们回到那段代码，这段代码是一个 `if` 语句块，其中变量 `waiting` 同样是一个标志，它也定义在 `scheduler.js` 文件头部，初始值为 `false`：

```js
let waiting = false
```

为什么需要这个标志呢？我们看 `if` 语句块内的代码就知道了，在 `if` 语句块内先将 `waiting` 的值设置为 `true`，这意味着无论调用多少次 `queueWatcher` 函数，该 `if` 语句块的代码只会执行一次。接着调用 `nextTick` 并以 `flushSchedulerQueue` 函数作为参数，其中 `flushSchedulerQueue` 函数的作用之一就是用来将队列中的观察者统一执行更新的。

接下来我们来看 flushSchedulerQueue 的实现，它的定义在 src/core/observer/scheduler.js 中。

```js
let flushing = false
let index = 0

function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

这里有几个重要的逻辑要梳理一下，对于一些分支逻辑如 keep-alive 组件相关和之前提到过的 updated 钩子函数的执行会略过。

* 队列排序

`queue.sort((a, b) => a.id - b.id)` 对队列做了从小到大的排序，这么做主要有以下要确保以下几点：

1. 组件的更新由父到子；因为父组件的创建过程是先于子的，所以 watcher 的创建也是先父后子，执行顺序也应该保持先父后子。

2. 用户的自定义 watcher 要优先于渲染 watcher 执行；因为用户自定义 watcher 是在渲染 watcher 之前创建的。

3. 如果一个组件在父组件的 watcher 执行期间被销毁，那么它对应的 watcher 执行都可以被跳过，所以父组件的 watcher 应该先执行。

* 队列遍历

在对 queue 排序后，接着就是要对它做遍历，拿到对应的 watcher，执行 `watcher.run()`。这里需要注意一个细节，在遍历的时候每次都会对 `queue.length` 求值，因为在 `watcher.run()` 的时候，很可能用户会再次添加新的 watcher，这样会再次执行到 queueWatcher，如下：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // ...
  }
}
```

可以看到，这时候 flushing 为 true，就会执行到 else 的逻辑，然后就会从后往前找，找到第一个待插入 watcher 的 id 比当前队列中 watcher 的 id 大的位置。把 watcher 按照 id的插入到队列中，因此 queue 的长度发生了变化。

* 状态恢复

这个过程就是执行 resetSchedulerState 函数，它的定义在 src/core/observer/scheduler.js 中。

```js
const queue: Array<Watcher> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0
/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

逻辑非常简单，就是把这些控制流程状态的一些变量恢复到初始值，把 watcher 队列清空。

对于 `nextTick` 相信大家已经很熟悉了，其实最好理解的方式就是把 `nextTick` 看做 `setTimeout(fn, 0)`，如下：

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    // 省略...
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      setTimeout(flushSchedulerQueue, 0)
    }
  }
}
```

我们完全可以使用 `setTimeout` 替换 `nextTick`，我们只需要执行一次 `setTimeout` 语句即可，`waiting` 变量就保证了 `setTimeout` 语句只会执行一次，这样 `flushSchedulerQueue` 函数将会在下一次事件循环开始时立即调用，但是既然可以使用 `setTimeout` 替换 `nextTick` 那么为什么不用 `setTimeout` 呢？原因就在于 `setTimeout` 并不是最优的选择，`nextTick` 的意义就是它会选择一条最优的解决方案，接下来我们就讨论一下 `nextTick` 是如何实现的。
