# Watcher 触发依赖的过程

在上一小节中我们提到了，每次求值并收集完观察者之后，会将当次求值所收集到的观察者保存到另外一组属性中，即 `depIds` 和 `deps`，并将存有当次求值所收集到的观察者的属性清空，即清空 `newDepIds` 和 `newDeps`。我们当时也说过了，这么做的目的是为了对比当次求值与上一次求值所收集到的观察者的变化情况，并做出合理的矫正工作，比如移除那些已经没有关联关系的观察者等。本节我们将以数据属性的变化为切入点，讲解重新求值的过程。

假设我们有如下模板：

```html
<div id="demo">
  {{name}}
</div>
```

我们知道这段模板将会被编译成渲染函数，接着创建一个渲染函数的观察者，从而对渲染函数求值，在求值的过程中会触发数据对象 `name` 属性的 `get` 拦截器函数，进而将该观察者收集到 `name` 属性通过闭包引用的“筐”中，即收集到 `Dep` 实例对象中。这个 `Dep` 实例对象是属于 `name` 属性自身所拥有的，这样当我们尝试修改数据对象 `name` 属性的值时就会触发 `name` 属性的 `set` 拦截器函数，这样就有机会调用 `Dep` 实例对象的 `notify` 方法，从而触发了响应，如下代码截取自 `defineReactive` 函数中的 `set` 拦截器函数：

```js
set: function reactiveSetter (newVal) {
  // 省略...
  dep.notify()
}
```

由此，可以看到当属性值变化时确实通过 `set` 拦截器函数调用了 `Dep` 实例对象的 `notify` 方法，这个方法就是用来通知变化的，我们找到 `Dep` 类的 `notify` 方法，如下：

```js
export default class Dep {
  // 省略...

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 省略...

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
}
```

大家观察 `notify` 函数可以发现其中包含如下这段 `if` 条件语句块：

```js
if (process.env.NODE_ENV !== 'production' && !config.async) {
  // subs aren't sorted in scheduler if not running async
  // we need to sort them now to make sure they fire in correct
  // order
  subs.sort((a, b) => a.id - b.id)
}
```

对于这段代码的作用，我们之后会详细讲解，现在大家可以完全忽略，这并不影响我们对代码的理解。如果我们去掉如上这段代码，那么 `notify` 函数将变为：

```js
notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
```

`notify` 方法只做了一件事，就是遍历当前 `Dep` 实例对象的 `subs` 属性中所保存的所有观察者对象，并逐个调用观察者对象的 `update` 方法，这就是触发响应的实现机制，那么大家应该也猜到了，重新求值的操作应该是在 `update` 方法中进行的，那我们就找到观察者对象的 `update` 方法，看看它做了什么事情，如下：

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

在 `update` 方法中代码被拆分成了三部分，即 `if...else if...else` 语句块。首先 `if` 语句块的代码会在判断条件 `this.lazy` 为真的情况下执行。此时会继续判断 `else...if` 语句的条件 `this.sync` 是否为真，我们知道 `this.sync` 属性的值就是创建观察者实例对象时传递的第三个选项参数中的 `sync` 属性的值，这个值的真假代表了当变化发生时是否同步更新变化。对于渲染函数的观察者来讲，它并不是同步更新变化的，而是将变化放到一个异步更新队列中，也就是 `else` 语句块中代码所做的事情，即 `queueWatcher` 会将当前观察者对象放到一个异步更新队列，这个队列会在调用栈被清空之后按照一定的顺序执行。关于更多异步更新队列的内容我们会在后面单独讲解，这里大家只需要知道一件事情，那就是无论是同步更新变化还是将更新变化的操作放到异步更新队列，真正的更新变化操作都是通过调用观察者实例对象的 `run` 方法完成的。所以此时我们应该把目光转向 `run` 方法，如下：

```js
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
```

`run` 方法首先判断了当前观察者实例的 `this.active` 属性是否为真，其中 `this.active` 属性用来标识一个观察者是否处于激活状态，或者可用状态。if 语句中第一句代码就调用了 `this.get` 方法，这意味着重新求值，这也证明了我们在上一小节中的假设。对于渲染函数的观察者来讲，重新求值其实等价于重新执行渲染函数，最终结果就是重新生成了虚拟DOM并更新真实DOM，这样就完成了重新渲染的过程。在重新调用 `this.get` 方法之后是一个 `if` 语句块，实际上对于渲染函数的观察者来讲并不会执行这个 `if` 语句块，因为 `this.get` 方法的返回值其实就等价于 `updateComponent` 函数的返回值，这个值将永远都是 `undefined`。实际上 `if` 语句块内的代码是为非渲染函数类型的观察者准备的，它用来对比新旧两次求值的结果，当值不相等的时候会调用通过参数传递进来的回调。我们先看一下判断条件，如下：

```js
const value = this.get()
if (
  value !== this.value ||
  // Deep watchers and watchers on Object/Arrays should fire even
  // when the value is the same, because the value may
  // have mutated.
  isObject(value) ||
  this.deep
) {
  // 省略...
}
```

首先对比新值 `value` 和旧值 `this.value` 是否相等，只有在不相等的情况下才需要执行回调，但是两个值相等就一定不执行回调吗？未必，这个时候就需要检测第二个条件是否成立，即 `isObject(value)`，判断新值的类型是否是对象，如果是对象的话即使值不变也需要执行回调，注意这里的“不变”指的是引用不变，如下代码所示：

```js
const data = {
  obj: {
    a: 1
  }
}
const obj1 = data.obj
data.obj.a = 2
const obj2 = data.obj

console.log(obj1 === obj2) // true
```

上面的代码中由于 `obj1` 与 `obj2` 具有相同的引用，所以他们总是相等的，但其实数据已经变化了，这就是判断 `isObject(value)` 为真则执行回调的原因。

接下来我们就看一下 `if` 语句块内的代码：

```js
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
```

代码如果执行到了 `if` 语句块内，则说明应该执行观察者的回调函数了。首先定义了 `oldValue` 常量，它的值是旧值，紧接着使用新值更新了 `this.value` 的值。我们可以看到如上代码中是如何执行回调的：

```js
this.cb.call(this.vm, value, oldValue)
```

将回调函数的作用域修改为当前 `Vue` 组件对象，然后传递了两个参数，分别是新值和旧值。

除此之外，我们注意如下代码：

```js
if (this.user) {
  try {
    this.cb.call(this.vm, value, oldValue)
  } catch (e) {
    handleError(e, this.vm, `callback for watcher "${this.expression}"`)
  }
} else {
  this.cb.call(this.vm, value, oldValue)
}
```

在调用回调函数的时候，如果观察者对象的 `this.user` 为真意味着这个观察者是开发者定义的，所谓开发者定义的是指那些通过 `watch` 选项或 `$watch` 函数定义的观察者，这些观察者的特点是回调函数是由开发者编写的，所以这些回调函数在执行的过程中其行为是不可预知的，很可能出现错误，这时候将其放到一个 `try...catch` 语句块中，这样当错误发生时我们就能够给开发者一个友好的提示。并且我们注意到在提示信息中包含了 `this.expression` 属性，我们前面说过该属性是被观察目标(`expOrFn`)的字符串表示，这样开发者就能清楚的知道是哪里发生了错误。
