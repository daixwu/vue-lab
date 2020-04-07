# 依赖收集的过程

`this.get()` 是我们遇到的第一个观察者对象的实例方法，它的作用可以用两个字描述：**求值**。求值的目的有两个，第一个是能够触发访问器属性的 `get` 拦截器函数，第二个是能够获得被观察目标的值。而且能够触发访问器属性的 `get` 拦截器函数是依赖被收集的关键，下面我们具体查看一下 `this.get()` 方法的内容：

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

如上是 `this.get()` 方法的全部代码，一上来调用了 `pushTarget(this)` 函数，并将当前观察者实例对象作为参数传递，这里的 `pushTarget` 函数来自于 `src/core/observer/dep.js` 文件，如下代码所示：

```js
export default class Dep {
  // 省略...
}

Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

在 `src/core/observer/dep.js` 文件中定义了 `Dep` 类，我们之前有遇到过 `Dep` 类，当时我们说每个响应式数据的属性都通过闭包引用着一个用来收集属于自身依赖的“筐”，实际上那个“筐”就是 `Dep` 类的实例对象。更多关于 `Dep` 类的内容我们会在合适的地方讲解，现在我们的主要目的是搞清楚 `pushTarget` 函数是做什么的。在上面这段代码中我们可以看到 `Dep` 类拥有一个静态属性，即 `Dep.target` 属性，该属性的初始值为 `null`，其实 `pushTarget` 函数的作用就是用来为 `Dep.target` 属性赋值的，`pushTarget` 函数会将接收到的参数赋值给 `Dep.target` 属性，我们知道传递给 `pushTarget` 函数的参数就是调用该函数的观察者对象，所以 `Dep.target` 保存着一个观察者对象，其实这个观察者对象就是即将要收集的目标。

我们再回到 `this.get()` 方法中，如下是简化后的代码：

```js
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    // 省略...
  } finally {
    // 省略...
  }
  return value
}
```

在调用 `pushTarget` 函数之后，定义了 `value` 变量，该变量的值为 `this.getter` 函数的返回值，我们知道观察者对象的 `this.getter` 属性是一个函数，这个函数的执行就意味着对被观察目标的求值，并将得到的值赋值给 `value` 变量，而且我们可以看到 `this.get` 方法的最后将 `value` 返回，为什么要强调这一点呢？如下代码所示：

```js
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: ?Object,
  isRenderWatcher?: boolean
) {
  // 省略...
  this.value = this.lazy
    ? undefined
    : this.get()
}
```

这段代码将 `this.get()` 方法的返回值赋值给了观察者实例对象的 `this.value` 属性。也就是说 `this.value` 属性保存着被观察目标的值。

`this.get()` 方法除了对被观察目标求值之外，大家别忘了正是因为对被观察目标的求值才得以触发数据属性的 `get` 拦截器函数，还是以渲染函数的观察者为例，假设我们有如下模板：

```html
<div id="demo">
  <p>{{name}}</p>
</div>
```

这段模板被编译将生成如下渲染函数：

```js
// 编译生成的渲染函数是一个匿名函数
function anonymous () {
  with (this) {
    return _c('div',
      { attrs:{ "id": "demo" } },
      [_v("\n      "+_s(name)+"\n    ")]
    )
  }
}
```

大家看不懂渲染函数没关系，关于模板到渲染函数的编译过程我们会在编译器相关章节为大家讲解，现在大家只需要注意到渲染函数的执行会读取数据属性 `name` 的值，这将会触发 `name` 属性的 `get` 拦截器函数，如下代码截取自 `defineReactive` 函数：

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

这段代码我们已经很熟悉了，它是数据属性的 `get` 拦截器函数，由于渲染函数读取了 `name` 属性的值，所以 `name` 属性的 `get` 拦截器函数将被执行，大家注意以上代码，首先判断了 `Dep.target` 是否存在，如果存在则调用 `dep.depend` 方法收集依赖。那么 `Dep.target` 是否存在呢？答案是存在，这就是为什么 `pushTarget` 函数要在调用 `this.getter` 函数之前被调用的原因。既然 `dep.depend` 方法被执行，那么我们就找到 `dep.depend` 方法，如下：

```js
depend () {
  if (Dep.target) {
    Dep.target.addDep(this)
  }
}
```

在 `dep.depend` 方法内部又判断了一次 `Dep.target` 是否有值，有的同学可能会有疑问，这不是多此一举吗？其实这么做并不多余，因为 `dep.depend` 方法除了在属性的 `get` 拦截器函数内被调用之外还在其他地方被调用了，这时候就需要对 `Dep.target` 做判断，至于在哪里调用的我们后面会讲到。另外我们发现在 `depend` 方法内部其实并没有真正的执行收集依赖的动作，而是调用了观察者实例对象的 `addDep` 方法：`Dep.target.addDep(this)`，并以当前 `Dep` 实例对象作为参数。为了搞清楚这么做的目的，我们找到观察者实例对象的 `addDep` 方法，如下：

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

可以看到 `addDep` 方法接收一个参数，这个参数是一个 `Dep` 对象，在 `addDep` 方法内部首先定义了常量 `id`，它的值是 `Dep` 实例对象的唯一 `id` 值。接着是一段 `if` 语句块，该 `if` 语句块的代码很关键，因为它的作用就是用来 **避免收集重复依赖** 的，既然是用来避免收集重复的依赖，那么就不得不用到我们前面提到过的两组属性，即 `newDepIds`、`newDeps` 以及 `depIds`、`deps`。为了让大家更好地理解，我们思考一下可不可以把 `addDep` 方法修改成如下这样：

```js
addDep (dep: Dep) {
  dep.addSub(this)
}
```

首先解释一下 `dep.addSub` 方法，它的源码如下：

```js
addSub (sub: Watcher) {
  this.subs.push(sub)
}
```

`addSub` 方法接收观察者对象作为参数，并将接收到的观察者添加到 `Dep` 实例对象的 `subs` 数组中，其实 `addSub` 方法才是真正用来收集观察者的方法，并且收集到的观察者都会被添加到 `subs` 数组中存起来。

了解了 `addSub` 方法之后，我们再回到如下这段代码：

```js
addDep (dep: Dep) {
  dep.addSub(this)
}
```

我们修改了 `addDep` 方法，直接在 `addDep` 方法内调用 `dep.addSub` 方法，并将当前观察者对象作为参数传递。这不是很好吗？难道有什么问题吗？当然有问题，假如我们有如下模板：

```html
<div id="demo">
  {{name}}{{name}}
</div>
```

这段模板的不同之处在于我们使用了两次 `name` 数据，那么相应的渲染函数也将变为如下这样：

```js {5}
function anonymous () {
  with (this) {
    return _c('div',
      { attrs:{ "id": "demo" } },
      [_v("\n      "+_s(name)+_s(name)+"\n    ")]
    )
  }
}
```

可以看到，渲染函数的执行将读取两次数据对象 `name` 属性的值，这必然会触发两次 `name` 属性的 `get` 拦截器函数，同样的道理，`dep.depend` 也将被触发两次，最后导致 `dep.addSub` 方法被执行了两次，且参数一模一样，这样就产生了同一个观察者被收集多次的问题。所以我们不能像如上那样修改 `addDep` 函数的代码，那么此时我相信大家也应该知道如下代码的含义了：

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

在 `addDep` 内部并不是直接调用 `dep.addSub` 收集观察者，而是先根据 `dep.id` 属性检测该 `Dep` 实例对象是否已经存在于 `newDepIds` 中，如果存在那么说明已经收集过依赖了，什么都不会做。如果不存在才会继续执行 `if` 语句块的代码，同时将 `dep.id` 属性和 `Dep` 实例对象本身分别添加到 `newDepIds` 和 `newDeps` 属性中，这样无论一个数据属性被读取了多少次，对于同一个观察者它只会收集一次。

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

另外这里的判断条件 `!this.depIds.has(id)` 是什么意思呢？我们知道 `newDepIds` 属性用来避免在 **一次求值** 的过程中收集重复的依赖，其实 `depIds` 属性是用来在 **多次求值** 中避免收集重复依赖的。什么是多次求值，其实所谓多次求值是指当数据变化时重新求值的过程。大家可能会疑惑，难道重新求值的时候不能用 `newDepIds` 属性来避免收集重复的依赖吗？不能，原因在于每一次求值之后 `newDepIds` 属性都会被清空，也就是说每次重新求值的时候对于观察者实例对象来讲 `newDepIds` 属性始终是全新的。虽然每次求值之后会清空 `newDepIds` 属性的值，但在清空之前会把 `newDepIds` 属性的值以及 `newDeps` 属性的值赋值给 `depIds` 属性和 `deps` 属性，这样重新求值的时候 `depIds` 属性和 `deps` 属性将会保存着上一次求值中 `newDepIds` 属性以及 `newDeps` 属性的值。为了证明这一点，我们来看一下观察者对象的求值方法，即 `get()` 方法：

```js
get () {
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    value = this.getter.call(vm, vm)
  } catch (e) {
    // 省略...
  } finally {
    // 省略...
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

可以看到在 `finally` 语句块内调用了观察者对象的 `cleanupDeps` 方法，这个方法的作用正如我们前面所说的那样，每次求值完毕后都会使用 `depIds` 属性和 `deps` 属性保存 `newDepIds` 属性和 `newDeps` 属性的值，然后再清空 `newDepIds` 属性和 `newDeps` 属性的值，如下是 `cleanupDeps` 方法的源码：

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

在 `cleanupDeps` 方法内部，首先是一个 `while` 循环，我们暂且不关心这个循环的作用，我们看循环下面的代码，即高亮的部分，这段代码是典型的引用类型变量交换值的过程，最终的结果就是 `newDepIds` 属性和 `newDeps` 属性被清空，并且在被清空之前把值分别赋给了 `depIds` 属性和 `deps` 属性，这两个属性将会用在下一次求值时避免依赖的重复收集。

现在我们可以做几点总结：

* 1、`newDepIds` 属性用来在一次求值中避免收集重复的观察者
* 2、每次求值并收集观察者完成之后会清空 `newDepIds` 和 `newDeps` 这两个属性的值，并且在被清空之前把值分别赋给了 `depIds` 属性和 `deps` 属性
* 3、`depIds` 属性用来避免重复求值时收集重复的观察者

通过以上三点内容我们可以总结出一个结论，即 `newDepIds` 和 `newDeps` 这两个属性的值所存储的总是当次求值所收集到的 `Dep` 实例对象，而 `depIds` 和 `deps` 这两个属性的值所存储的总是上一次求值过程中所收集到的 `Dep` 实例对象。

除了以上三点之外，其实 `deps` 属性还能够用来移除废弃的观察者，`cleanupDeps` 方法中开头的那段 `while` 循环就是用来实现这个功能的，如下代码所示：

```js
cleanupDeps () {
  let i = this.deps.length
  while (i--) {
    const dep = this.deps[i]
    if (!this.newDepIds.has(dep.id)) {
      dep.removeSub(this)
    }
  }
  // 省略...
}
```

这段 `while` 循环就是对 `deps` 数组进行遍历，也就是对上一次求值所收集到的 `Dep` 对象进行遍历，然后在循环内部检查上一次求值所收集到的 `Dep` 实例对象是否存在于当前这次求值所收集到的 `Dep` 实例对象中，如果不存在则说明该 `Dep` 实例对象已经和该观察者不存在依赖关系了，这时就会调用 `dep.removeSub(this)` 方法并以该观察者实例对象作为参数传递，从而将该观察者对象从 `Dep` 实例对象中移除。

我们可以找到 `Dep` 类的 `removeSub` 实例方法，如下：

```js
removeSub (sub: Watcher) {
  remove(this.subs, sub)
}
```

它的内容很简单，接收一个要被移除的观察者作为参数，然后使用 `remove` 工具函数，将该观察者从 `this.subs` 数组中移除。其中 `remove` 工具函数来自 `src/shared/util.js` 文件，可以在 shared/util.js 文件工具方法全解 中查看。
