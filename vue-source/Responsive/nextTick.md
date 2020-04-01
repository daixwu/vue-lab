# nextTick 的实现

nextTick 是 Vue 的一个核心实现，在介绍 Vue 的 nextTick 之前，为了方便大家理解，我先简单介绍一下 JS 的运行机制。

## JS 运行机制

JS 执行是单线程的，它是基于事件循环的。事件循环大致分为以下几个步骤：

（1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

（2）主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

（3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

（4）主线程不断重复上面的第三步。

![event-loop](../assets/img/event-loop.png)

主线程的执行过程就是一个 tick，而所有的异步结果都是通过 “任务队列” 来调度。 消息队列中存放的是一个个的任务（task）。 规范中规定 task 分为两大类，分别是 macro task 和 micro task，并且每个 macro task 结束后，都要清空所有的 micro task。

关于 macro task 和 micro task 的概念，这里不会细讲，简单通过一段代码演示他们的执行顺序：

```js
for (macroTask of macroTaskQueue) {
    // 1. Handle current MACRO-TASK
    handleMacroTask();

    // 2. Handle all MICRO-TASK
    for (microTask of microTaskQueue) {
        handleMicroTask(microTask);
    }
}
```

在浏览器环境中，常见的 macro task 有 setTimeout、MessageChannel、postMessage、setImmediate；常见的 micro task 有 MutationObsever 和 Promise.then。

## Vue 的实现

`nextTick` 函数来自于 `src/core/util/next-tick.js` 文件，对于 `nextTick` 函数相信大家都不陌生，我们常用的 `$nextTick` 方法实际上就是对 `nextTick` 函数的封装，如下：

```js
export function renderMixin (Vue: Class<Component>) {
  // 省略...
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }
  // 省略...
}
```

`$nextTick` 方法是在 `renderMixin` 函数中挂载到 `Vue` 原型上的，可以看到 `$nextTick` 函数体只有一句话即调用 `nextTick` 函数，这说明 `$nextTick` 确实是对 `nextTick` 函数的简单包装。

我们知道任务队列并非只有一个队列，在 `node` 中更为复杂，但总的来说我们可以将其分为 `micro task` 和 `macro task`，并且这两个队列的行为还要依据不同浏览器的具体实现去讨论，这里我们只讨论被广泛认同和接受的队列执行行为。当调用栈空闲后每次事件循环只会从 `macro task` 中读取一个任务并执行，而在同一次事件循环内会将 `micro task` 队列中所有的任务全部执行完毕，且要先于 `macro task`。

另外 `macro task` 中两个不同的任务之间可能穿插着UI的重渲染，那么我们只需要在 `micro task` 中把所有在UI重渲染之前需要更新的数据全部更新，这样只需要一次重渲染就能得到最新的DOM了。

恰好 `Vue` 是一个数据驱动的框架，如果能在UI重渲染之前更新所有数据状态，这对性能的提升是一个很大的帮助，所以要优先选用 `micro task` 去更新数据状态而不是 `macro task`，这就是为什么不使用 `setTimeout` 的原因，因为 `setTimeout` 会将回调放到 `macro task` 队列中而不是 `micro task` 队列，所以理论上最优的选择是使用 `Promise`，当浏览器不支持 `Promise` 时再降级为 `setTimeout`。如下是 `next-tick.js` 文件中的一段代码：

```js
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

上面的代码中首先检测当前宿主环境是否支持原生的 `Promise`，如果支持则优先使用 `Promise` 注册 `micro task`，做法很简单，首先定义常量 `p` 它的值是一个立即 `resolve` 的 `Promise` 实例对象，接着将变量 `microTimerFunc` 定义为一个函数，这个函数的执行将会把 `flushCallbacks` 函数注册为 `micro task`。另外大家注意这句代码：

```js
if (isIOS) setTimeout(noop)
```

注释已经写得很清楚了，这是一个解决怪异问题的变通方法，在一些 `UIWebViews` 中存在很奇怪的问题，即 `micro task` 没有被刷新，对于这个问题的解决方案就是让浏览做一些其他的事情比如注册一个 `macro task` 即使这个 `macro task` 什么都不做，这样就能够间接触发 `micro task` 的刷新。

使用 `Promise` 是最理想的方案，但是如果宿主环境不支持 `Promise`，我们就需要降级处理，即注册 `macro task`，这就是 `else` 语句块内代码所做的事情：

```js
let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

将一个回调函数注册为 `macro task` 的方式有很多，如 `setTimeout`、`setInterval` 以及 `setImmediate` 等等，但不同的方案之间是有区别的，通过上面的代码我们可以看到 `setTimeout` 被作为最后的备选方案。

如果宿主环境支持`MutationObserver`则使用`MutationObserver`,如果宿主环境支持原生 `setImmediate` 函数，则使用 `setImmediate` 注册 `macro task`，`setTimeout` 放在最后，为什么首选 `setImmediate` 呢？这是有原因的，因为 `setImmediate` 拥有比 `setTimeout` 更好的性能，这个问题很好理解，`setTimeout` 在将回调注册为 `macro task` 之前要不停的做超时检测，而 `setImmediate` 则不需要，这就是优先选用 `setImmediate` 的原因。但是 `setImmediate` 的缺陷也很明显，就是它的兼容性问题，到目前为止只有IE浏览器实现了它，所以为了兼容非IE浏览器我们还需要做兼容处理。

现在是时候仔细看一下 `nextTick` 函数都做了什么事情了，不过为了更融入理解 `nextTick` 函数的代码，我们需要从 `$nextTick` 方法入手，如下：

```js
export function renderMixin (Vue: Class<Component>) {
  // 省略...
  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }
  // 省略...
}
```

`$nextTick` 方法只接收一个回调函数作为参数，但在内部调用 `nextTick` 函数时，除了把回调函数 `fn` 透传之外，第二个参数是当前组件实例对象 `this`。我们知道在使用 `$nextTick` 方法时是可以省略回调函数这个参数的，这时 `$nextTick` 方法会返回一个 `promise` 实例对象。这些功能实际上都是由 `nextTick` 函数提供的，如下是 `nextTick` 函数的签名：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  // 省略...
}
```

`nextTick` 函数接收两个参数，第一个参数是一个回调函数，第二个参数指定一个作用域。下面我们逐个分析传递回调函数与不传递回调函数这两种使用场景功能的实现，首先我们来看传递回调函数的情况，那么此时参数 `cb` 就是回调函数，来看如下代码：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 省略
}
```

`nextTick` 函数会在 `callbacks` 数组中添加一个新的函数，`callbacks` 数组定义在文件头部：`const callbacks = []`。注意并不是将 `cb` 回调函数直接添加到 `callbacks` 数组中，但这个被添加到 `callbacks` 数组中的函数的执行会间接调用 `cb` 回调函数，并且可以看到在调用 `cb` 函数时使用 `.call` 方法将函数 `cb` 的作用域设置为 `ctx`，也就是 `nextTick` 函数的第二个参数。

所以对于 `$nextTick` 方法来讲，传递给 `$nextTick` 方法的回调函数的作用域就是当前组件实例对象，当然了前提是回调函数不能是箭头函数，其实在平时的使用中，回调函数使用箭头函数也没关系，只要你能够达到你的目的即可。另外我们再次强调一遍，此时回调函数并没有被执行，当你调用 `$nextTick` 方法并传递回调函数时，会使用一个新的函数包裹回调函数并将新函数添加到 `callbacks` 数组中。

我们继续看 `nextTick` 函数的代码，如下：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  // 省略...
  if (!pending) {
    pending = true
    timerFunc()
  }
  // 省略...
}
```

在将回调函数添加到 `callbacks` 数组之后，会进行一个 `if` 条件判断，判断变量 `pending` 的真假，`pending` 变量也定义在文件头部：`let pending = false`，它是一个标识，它的真假代表回调队列是否处于等待刷新的状态，初始值是 `false` 代表回调队列为空不需要等待刷新。假如此时在某个地方调用了 `$nextTick` 方法，那么 `if` 语句块内的代码将会被执行，在 `if` 语句块内优先将变量 `pending` 的值设置为 `true`，代表着此时回调队列不为空，正在等待刷新。既然等待刷新，那么当然要刷新回调队列啊，怎么刷新呢？这时就用到了我们前面讲过的 `timerFunc` 函数，我们知道这个函数的作用是将 `flushCallbacks` 函数分别注册为 `micro task` 和 `macro task`。但是无论哪种任务类型，它们都将会等待调用栈清空之后才执行。如下：

```js
created () {
  this.$nextTick(() => { console.log(1) })
  this.$nextTick(() => { console.log(2) })
  this.$nextTick(() => { console.log(3) })
}
```

上面的代码中我们在 `created` 钩子中连续调用三次 `$nextTick` 方法，但只有第一次调用 `$nextTick` 方法时才会执行 `timerFunc` 函数将 `flushCallbacks` 注册为 `micro task`，但此时 `flushCallbacks` 函数并不会执行，因为它要等待接下来的两次 `$nextTick` 方法的调用语句执行完后才会执行，或者准确的说等待调用栈被清空之后才会执行。也就是说当 `flushCallbacks` 函数执行的时候，`callbacks` 回调队列中将包含本次事件循环所收集的所有通过 `$nextTick` 方法注册的回调，而接下来的任务就是在 `flushCallbacks` 函数内将这些回调全部执行并清空。如下是 `flushCallbacks` 函数的源码：

```js
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```

很好理解，首先将变量 `pending` 重置为 `false`，接着开始执行回调，但需要注意的是在执行 `callbacks` 队列中的回调函数时并没有直接遍历 `callbacks` 数组，而是使用 `copies` 常量保存一份 `callbacks` 的复制，然后遍历 `copies` 数组，并且在遍历 `copies` 数组之前将 `callbacks` 数组清空：`callbacks.length = 0`。为什么要这么做呢？这么做肯定是有原因的，我们模拟一下整个异步更新的流程就明白了，如下代码：

```js
created () {
  this.name = 'tiger'
  this.$nextTick(() => {
    this.name = 'captain'
    this.$nextTick(() => { console.log('第二个 $nextTick') })
  })
}
```

上面代码中我们在外层 `$nextTick` 方法的回调函数中再次调用了 `$nextTick` 方法，理论上外层 `$nextTick` 方法的回调函数不应该与内层 `$nextTick` 方法的回调函数在同一个 `micro task` 任务中被执行，而是两个不同的 `micro task` 任务，虽然在结果上看或许没什么差别，但从设计角度就应该这么做。

我们注意上面代码中我们修改了两次 `name` 属性的值(假设它是响应式数据)，首先我们将 `name` 属性的值修改为字符串 `tiger`，我们前面讲过这会导致依赖于 `name` 属性的渲染函数观察者被添加到 `queue` 队列中，这个过程是通过调用 `src/core/observer/scheduler.js` 文件中的 `queueWatcher` 函数完成的。同时在 `queueWatcher` 函数内会使用 `nextTick` 将 `flushSchedulerQueue` 添加到 `callbacks` 数组中，所以此时 `callbacks` 数组如下：

```js
callbacks = [
  flushSchedulerQueue // queue = [renderWatcher]
]
```

同时会将 `flushCallbacks` 函数注册为 `micro task`，所以此时 `micro task` 队列如下：

```js
// microtask 队列
[
  flushCallbacks
]
```

接着调用了第一个 `$nextTick` 方法，`$nextTick` 方法会将其回调函数添加到 `callbacks` 数组中，那么此时的 `callbacks` 数组如下：

```js
callbacks = [
  flushSchedulerQueue, // queue = [renderWatcher]
  () => {
    this.name = 'captain'
    this.$nextTick(() => { console.log('第二个 $nextTick') })
  }
]
```

接下来主线程处于空闲状态(调用栈清空)，开始执行 `micro task` 队列中的任务，即执行 `flushCallbacks` 函数，`flushCallbacks` 函数会按照顺序执行 `callbacks` 数组中的函数，首先会执行 `flushSchedulerQueue` 函数，这个函数会遍历 `queue` 中的所有观察者并重新求值，完成重新渲染(`re-render`)，在完成渲染之后，本次更新队列已经清空，`queue` 会被重置为空数组，一切状态还原。接着会执行如下函数：

```js
() => {
  this.name = 'captain'
  this.$nextTick(() => { console.log('第二个 $nextTick') })
}
```

这个函数是第一个 `$nextTick` 方法的回调函数，由于在执行该回调函数之前已经完成了重新渲染，所以该回调函数内的代码是能够访问更新后的DOM的，到目前为止一切都很正常，我们继续往下看，在该回调函数内再次修改了 `name` 属性的值为字符串 `captain`，这会再次触发响应，同样的会调用 `nextTick` 函数将 `flushSchedulerQueue` 添加到 `callbacks` 数组中，但是由于在执行 `flushCallbacks` 函数时优先将 `pending` 的重置为 `false`，所以 `nextTick` 函数会将 `flushCallbacks` 函数注册为一个新的 `micro task`，此时 `micro task` 队列将包含两个 `flushCallbacks` 函数：

```js
// microtask 队列
[
  flushCallbacks, // 第一个 flushCallbacks
  flushCallbacks  // 第二个 flushCallbacks
]
```

怎么样？我们的目的达到了，现在有两个 `micro task` 任务。

而另外除了将变量 `pending` 的值重置为 `false` 之外，我们要知道第一个 `flushCallbacks` 函数遍历的并不是 `callbacks` 本身，而是它的复制品 `copies` 数组，并且在第一个 `flushCallbacks` 函数的一开头就清空了 `callbacks` 数组本身。所以第二个 `flushCallbacks` 函数的一切流程与第一个 `flushCallbacks` 是完全相同。

最后我们再来讲一下，当调用 `$nextTick` 方法时不传递回调函数时，是如何实现返回 `Promise` 实例对象的，实现很简单我们来看一下 `nextTick` 函数的代码，如下：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 省略...
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

如上代码所示，当 `nextTick` 函数没有接收到 `cb` 参数时，会检测当前宿主环境是否支持 `Promise`，如果支持则直接返回一个 `Promise` 实例对象，并且将 `resolve` 函数赋值给 `_resolve` 变量，`_resolve` 变量声明在 `nextTick` 函数的顶部。同时再来看如下代码：

```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 省略...
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

当 `flushCallbacks` 函数开始执行 `callbacks` 数组中的函数时，如果没有传递 `cb` 参数，则直接调用 `_resolve` 函数，我们知道这个函数就是返回的 `Promise` 实例对象的 `resolve` 函数。这样就实现了 `Promise` 方式的 `$nextTick` 方法。

通过这一节对 nextTick 的分析，并结合上一节的 setter 分析，我们了解到数据的变化到 DOM 的重新渲染是一个异步过程，发生在下一个 tick。这就是我们平时在开发的过程中，比如从服务端接口去获取数据的时候，数据做了修改，如果我们的某些方法去依赖了数据修改后的 DOM 变化，我们就必须在 nextTick 后执行。比如下面的伪代码：

```js
getData(res).then(()=>{
  this.xxx = res.data
  this.$nextTick(() => {
    // 这里我们可以获取变化后的 DOM
  })
})
```

Vue.js 提供了 2 种调用 nextTick 的方式，一种是全局 API `Vue.nextTick`，一种是实例上的方法 `vm.$nextTick`，无论我们使用哪一种，最后都是调用 next-tick.js 中实现的 `nextTick` 方法。
