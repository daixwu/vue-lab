# get 函数如何收集依赖

我们回过头来继续查看 defineReactive 函数的代码，接下来是 defineReactive 函数的关键代码，即使用 `Object.defineProperty` 函数定义访问器属性：

```js
Object.defineProperty(obj, key, {
  enumerable: true,
  configurable: true,
  get: function reactiveGetter () {
    // 省略...
  },
  set: function reactiveSetter (newVal) {
    // 省略...
})
```

当执行完以上代码实际上 defineReactive 函数就执行完毕了，对于访问器属性的 get 和 set 函数是不会执行的，因为此时没有触发属性的读取和设置操作。不过这不妨碍我们研究一下在 get 和 set 函数中都做了哪些事情，这里面就包含了我们在前面埋下伏笔的 if 条件语句的答案。我们先从 get 函数开始，看一看当属性被读取的时候都做了哪些事情，get 函数如下：

```js
get: function reactiveGetter () {
  const value = getter ? getter.call(obj) : val
  if (Dep.target) {
    dep.depend()
    if (childOb) {
      childOb.dep.depend()
      if (Array.isArray(value)) {
        dependArray(value)
      }
    }
  }
  return value
}
```

既然是 getter，那么当然要能够正确地返回属性的值才行，我们知道依赖的收集时机就是属性被读取的时候，所以 get 函数做了两件事：正确地返回属性值以及收集依赖，我们具体看一下代码，get 函数的第一句代码如下：

```js
const value = getter ? getter.call(obj) : val
```

首先判断是否存在 getter，我们知道 getter 常量中保存的是属性原有的 get 函数，如果 getter 存在那么直接调用该函数，并以该函数的返回值作为属性的值，保证属性的原有读取操作正常运作。如果 getter 不存在则使用 val 作为属性的值。可以发现 get 函数的最后一句将 value 常量返回，这样 get 函数需要做的第一件事就完成了，即正确地返回属性值。

除了正确地返回属性值，还要收集依赖，而处于 get 函数第一行和最后一行代码中间的所有代码都是用来完成收集依赖这件事儿的，下面我们就看一下它是如何收集依赖的，由于我们还没有讲解过 Dep 这个类，所以现在大家可以简单的认为 `dep.depend()` 这句代码的执行就意味着依赖被收集了。接下来我们仔细看一下代码：

```js
if (Dep.target) {
  dep.depend()
  if (childOb) {
    childOb.dep.depend()
    if (Array.isArray(value)) {
      dependArray(value)
    }
  }
}
```

首先判断 `Dep.target` 是否存在，那么 `Dep.target` 是什么呢？其实 `Dep.target` 与我们在 数据响应系统基本思路 一节中所讲的 Target 作用相同，所以 `Dep.target` 中保存的值就是要被收集的依赖(观察者)。所以如果 `Dep.target` 存在的话说明有依赖需要被收集，这个时候才需要执行 if 语句块内的代码，如果 `Dep.target` 不存在就意味着没有需要被收集的依赖，所以当然就不需要执行 if 语句块内的代码了。

在 if 语句块内第一句执行的代码就是：`dep.depend()`，执行 dep 对象的 depend 方法将依赖收集到 dep 这个“筐”中，这里的 dep 对象就是属性的 getter/setter 通过闭包引用的“筐”。

接着又判断了 childOb 是否存在，如果存在那么就执行 `childOb.dep.depend()`，这段代码是什么意思呢？要想搞清楚这段代码的作用，你需要知道 childOb 是什么，前面我们分析过，假设有如下数据对象：

```js
const data = {
  a: {
    b: 1
  }
}
```

该数据对象经过观测处理之后，将被添加 `__ob__` 属性，如下：

```js
const data = {
  a: {
    b: 1,
    __ob__: {value, dep, vmCount}
  },
  __ob__: {value, dep, vmCount}
}
```

对于属性 a 来讲，访问器属性 a 的 setter/getter 通过闭包引用了一个 Dep 实例对象，即属性 a 用来收集依赖的“筐”。除此之外访问器属性 a 的 setter/getter 还通过闭包引用着 childOb，且 `childOb === data.a.__ob__` 所以 `childOb.dep === data.a.__ob__.dep`。也就是说 `childOb.dep.depend()` 这句话的执行说明除了要将依赖收集到属性 a 自己的“筐”里之外，还要将同样的依赖收集到 `data.a.__ob__.dep` 这里”筐“里，为什么要将同样的依赖分别收集到这两个不同的”筐“里呢？其实答案就在于这两个”筐“里收集的依赖的触发时机是不同的，即作用不同，两个”筐“如下：

- 第一个”筐“是 dep
- 第二个”筐“是 `childOb.dep`

第一个”筐“里收集的依赖的触发时机是当属性值被修改时触发，即在 set 函数中触发：`dep.notify()`。而第二个”筐“里收集的依赖的触发时机是在使用 `$set` 或 `Vue.set` 给数据对象添加新属性时触发，我们知道由于 js 语言的限制，在没有 Proxy 之前 Vue 没办法拦截到给对象添加属性的操作。所以 Vue 才提供了 `$set` 和 `Vue.set` 等方法让我们有能力给对象添加新属性的同时触发依赖，那么触发依赖是怎么做到的呢？就是通过数据对象的 `__ob__` 属性做到的。因为 `__ob__.dep` 这个”筐“里收集了与 dep 这个”筐“同样的依赖。假设 `Vue.set` 函数代码如下：

```js
Vue.set = function (obj, key, val) {
  defineReactive(obj, key, val)
  obj.__ob__.dep.notify()
}
```

如上代码所示，当我们使用上面的代码给 `data.a` 对象添加新的属性：

```js
Vue.set(data.a, 'c', 1)
```

上面的代码之所以能够触发依赖，就是因为 `Vue.set` 函数中触发了收集在 `data.a.__ob__.dep` 这个”筐“中的依赖：

```js
Vue.set = function (obj, key, val) {
  defineReactive(obj, key, val)
  obj.__ob__.dep.notify() // 相当于 data.a.__ob__.dep.notify()
}

Vue.set(data.a, 'c', 1)
```

所以 `__ob__` 属性以及 `__ob__.dep` 的主要作用是为了添加、删除属性时有能力触发依赖，而这就是 `Vue.set` 或 `Vue.delete` 的原理。

在 `childOb.dep.depend()` 这句话的下面还有一个 if 条件语句，如下：

```js
if (Array.isArray(value)) {
  dependArray(value)
}
```

如果读取的属性值是数组，那么需要调用 dependArray 函数逐个触发数组每个元素的依赖收集，为什么这么做呢？那是因为 Observer 类在定义响应式属性时对于纯对象和数组的处理方式是不同，对于上面这段 if 语句的目的等到我们讲解到对于数组的处理时，会详细说明。
