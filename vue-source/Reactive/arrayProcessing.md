# 响应式数据之数组的处理

以上就是响应式数据对于纯对象的处理方式，接下来我们将会对数组展开详细的讨论。回到 Observer 类的 constructor 函数，找到如下代码：

```js
if (Array.isArray(value)) {
  if (hasProto) {
    protoAugment(value, arrayMethods)
  } else {
    copyAugment(value, arrayMethods, arrayKeys)
  }
  this.observeArray(value)
} else {
  this.walk(value)
}
```

在 if 条件语句中，使用 Array.isArray 函数检测被观测的值 value 是否是数组，如果是数组则会执行 if 语句块内的代码，从而实现对数组的观测。处理数组的方式与纯对象不同，我们知道数组是一个特殊的数据结构，它有很多实例方法，并且有些方法会改变数组自身的值，我们称其为变异方法，这些方法有：push、pop、shift、unshift、splice、sort 以及 reverse 等。这个时候我们就要考虑一件事，即当用户调用这些变异方法改变数组时需要触发依赖。换句话说我们需要知道开发者何时调用了这些变异方法，只有这样我们才有可能在这些方法被调用时做出反应。

## 拦截数组变异方法的思路

那么怎么样才能知道开发者何时调用了数组的变异方法呢？其实很简单，我们来思考这样一个问题，如下代码中 sayHello 函数用来打印字符串 'hello'：

```js
function sayHello () {
  console.log('hello')
}
```

但是我们有这样一个需求，在不改动 sayHello 函数源码的情况下，在打印字符串 'hello' 之前先输出字符串 'Hi'。这时候我们可以这样做：

```js
const originalSayHello = sayHello
sayHello = function () {
  console.log('Hi')
  originalSayHello()
}
```

看，这样就完美地实现了我们的需求，首先使用 originalSayHello 变量缓存原来的 sayHello 函数，然后重新定义 sayHello 函数，并在新定义的 sayHello 函数中调用缓存下来的 originalSayHello。这样我们就保证了在不改变 sayHello 函数行为的前提下对其进行了功能扩展。

这其实是一个很通用也很常见的技巧，而 Vue 正是通过这个技巧实现了对数据变异方法的拦截，即保持数组变异方法原有功能不变的前提下对其进行功能扩展。

数组本身也是一个对象，所以它实例的 `__proto__` 属性指向的就是数组构造函数的原型，即 `arr.__proto__ === Array.prototype` 为真。我们的一个思路是通过设置 `__proto__` 属性的值为一个新的对象，且该新对象的原型是数组构造函数原来的原型对象。

这样当通过数组实例调用变异方法时，首先执行的是 arrayMethods 上的同名函数，这样就能够实现对数组变异方法的拦截。用代码实现上图所示内容很简单，如下：

```js
// 要拦截的数组变异方法
const mutationMethods = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

const arrayMethods = Object.create(Array.prototype) // 实现 arrayMethods.__proto__ === Array.prototype
const arrayProto = Array.prototype  // 缓存 Array.prototype

mutationMethods.forEach(method => {
  arrayMethods[method] = function (...args) {
    const result = arrayProto[method].apply(this, args)

    console.log(`执行了代理原型的 ${method} 函数`)

    return result
  }
})
```

如上代码所示，我们通过 `Object.create(Array.prototype)` 创建了 arrayMethods 对象，这样就保证了 `arrayMethods.__proto__ === Array.prototype`。然后通过一个循环在 arrayMethods 对象上定义了与数组变异方法同名的函数，并在这些函数内调用了真正数组原型上的相应方法。我们可以测试一下，如下代码：

```js
const arr = []
arr.__proto__ = arrayMethods

arr.push(1)
```

可以发现控制台中打印了一句话：执行了代理原型的 push 函数。很完美，但是这实际上是存在问题的，因为 `__proto__` 属性是在 IE11+ 才开始支持，所以如果是低版本的 IE 怎么办？比如 IE9/10，所以出于兼容考虑，我们需要做能力检测，如果当前环境支持 `__proto__` 时我们就采用上述方式来实现对数组变异方法的拦截，如果当前环境不支持 `__proto__` 那我们就需要另想办法了，接下来我们就介绍一下兼容的处理方案。

实际上兼容的方案有很多，其中一个比较好的方案是直接在数组实例上定义与变异方法同名的函数，如下代码：

```js
const arr = []
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)

arrayKeys.forEach(method => {
  arr[method] = arrayMethods[method]
})
```

上面代码中，我们通过 `Object.getOwnPropertyNames` 函数获取所有属于 arrayMethods 对象自身的键，然后通过一个循环在数组实例上定义与变异方法同名的函数，这样当我们尝试调用 `arr.push()` 时，首先执行的是定义在数组实例上的 push 函数，也就是 arrayMethods.push 函数。这样我们就实现了兼容版本的拦截。不过细心的同学可能已经注意到了，上面这种直接在数组实例上定义的属性是可枚举的，所以更好的做法是使用 `Object.defineProperty`：

```js
arrayKeys.forEach(method => {
  Object.defineProperty(arr, method, {
    enumerable: false,
    writable: true,
    configurable: true,
    value: arrayMethods[method]
  })
})
```

这样就完美了。

## 拦截数组变异方法在 Vue 中的实现

我们已经了解了拦截数组变异方法的思路，接下来我们就可以具体的看一下 Vue 源码是如何实现的。在这个过程中我们会讲解数组是如何通过变异方法触发依赖(观察者)的。

我们回到 Observer 类的 constructor 函数：

```js
constructor (value: any) {
  this.value = value
  this.dep = new Dep()
  this.vmCount = 0
  def(value, '__ob__', this)
  if (Array.isArray(value)) {
    if (hasProto) {
      protoAugment(value, arrayMethods)
    } else {
      copyAugment(value, arrayMethods, arrayKeys)
    }
    this.observeArray(value)
  } else {
    this.walk(value)
  }
}
```

首先大家注意一点：无论是对象还是数组，都将通过 def 函数为其定义 `__ob__` 属性。接着我们来看一下 if 语句块的内容，如果被观测的值是一个数组，那么 if 语句块内的代码将被执行，即如下代码：

```js
if (hasProto) {
  protoAugment(value, arrayMethods)
} else {
  copyAugment(value, arrayMethods, arrayKeys)
}
this.observeArray(value)
```

如果 hasProto 为真则 执行 protoAugment 函数，否则则执行 copyAugment 函数。那么 hasProto 是什么呢？大家可以在附录 core/util 目录下的工具方法全解 中查看其讲解，其实 hasProto 是一个布尔值，它用来检测当前环境是否可以使用 `__proto__` 属性，如果 hasProto 为真则当前环境支持 `__proto__` 属性，否则意味着当前环境不能够使用 `__proto__` 属性。

如果当前环境支持使用 `__proto__` 属性，那么执行 protoAugment 函数，其中 protoAugment 就定义在 Observer 类的下方。源码如下：

```js
/**
 * Augment a target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```

那么 protoAugment 函数的作用是什么呢？相信大家已经猜到了，正如我们在讲解拦截数据变异方法的思路中所说的那样，可以通过设置数组实例的 `__proto__` 属性，让其指向一个代理原型，从而做到拦截。我们看一下 protoAugment 函数是如何被调用的：

```js
if (hasProto) {
  protoAugment(value, arrayMethods)
} else {
  copyAugment(value, arrayMethods, arrayKeys)
}
```

当 hasProto 为真时，则调用 protoAugment 函数，可以看到传递给 protoAugment 函数的参数有两个。第一个参数是 value，其实就是数组实例本身；第二个参数是 arrayMethods，这里的 arrayMethods 与我们在拦截数据变异方法的思路中所讲解的 arrayMethods 是一样的，它就是代理原型。

我们回到 protoAugment 函数，如下：

```js
/**
 * Augment a target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src: Object) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```

该函数的函数体只有一行代码：`target.__proto__ = src`。这行代码用来将数组实例的原型指向代理原型(arrayMethods)。下面我们具体看一下 arrayMethods 是如何实现的。打开 src/core/observer/array.js 文件：

```js
/*
 * not type checking this file because flow doesn't play well with
 * dynamically accessing methods on Array prototype
 */

import { def } from '../util/index'

const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```

如上是 src/core/observer/array.js 文件的全部代码，该文件只做了一件事情，那就是导出 arrayMethods 对象：

```js
const arrayProto = Array.prototype
export const arrayMethods = Object.create(arrayProto)
```

可以发现，arrayMethods 对象的原型是真正的数组构造函数的原型。接着定义了 methodsToPatch 常量：

```js
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
```

methodsToPatch 常量是一个数组，包含了所有需要拦截的数组变异方法的名字。再往下是一个 forEach 循环，用来遍历 methodsToPatch 数组。该循环的主要目的就是使用 def 函数在 arrayMethods 对象上定义与数组变异方法同名的函数，从而做到拦截的目的，如下是简化后的代码：

```js
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__

    // 省略中间部分...

    // notify change
    ob.dep.notify()
    return result
  })
})
```

上面的代码中，首先缓存了数组原本的变异方法：

```js
const original = arrayProto[method]
```

然后使用 def 函数在 arrayMethods 上定义与数组变异方法同名的函数，在函数体内优先调用了缓存下来的数组变异方法：

```js
const result = original.apply(this, args)
```

并将数组原本变异方法的返回值赋值给 result 常量，并且我们发现函数体的最后一行代码将 result 作为返回值返回。这就保证了拦截函数的功能与数组原本变异方法的功能是一致的。

关键要注意这两句代码：

```js
const ob = this.__ob__

// 省略中间部分...

// notify change
ob.dep.notify()
```

定义了 ob 常量，它是 `this.__ob__` 的引用，其中 this 其实就是数组实例本身，我们知道无论是数组还是对象，都将会被定义一个 `__ob__` 属性，并且 `__ob__.dep` 中收集了所有该对象(或数组)的依赖(观察者)。所以上面两句代码的目的其实很简单，当调用数组变异方法时，必然修改了数组，所以这个时候需要将该数组的所有依赖(观察者)全部拿出来执行，即：`ob.dep.notify()`。

注意上面的讲解中我们省略了中间部分，那么这部分代码的作用是什么呢？如下：

```js
def(arrayMethods, method, function mutator (...args) {
  // 省略...
  let inserted
  switch (method) {
    case 'push':
    case 'unshift':
      inserted = args
      break
    case 'splice':
      inserted = args.slice(2)
      break
  }
  if (inserted) ob.observeArray(inserted)
  // 省略...
})
```

首先我们需要思考一下数组变异方法对数组的影响是什么？无非是增加元素、删除元素以及变更元素顺序。有的同学可能会说还有替换元素，实际上替换可以理解为删除和增加的复合操作。那么在这些变更中，我们需要重点关注的是增加元素的操作，即 push、unshift 和 splice，这三个变异方法都可以为数组添加新的元素，那么为什么要重点关注呢？原因很简单，因为新增加的元素是非响应式的，所以我们需要获取到这些新元素，并将其变为响应式数据才行，而这就是上面代码的目的。下面我们看一下具体实现，首先定义了 inserted 变量，这个变量用来保存那些被新添加进来的数组元素：let inserted。接着是一个 switch 语句，在 switch 语句中，当遇到 push 和 unshift 操作时，那么新增的元素实际上就是传递给这两个方法的参数，所以可以直接将 inserted 的值设置为 args：`inserted = args`。当遇到 splice 操作时，我们知道 splice 函数从第三个参数开始到最后一个参数都是数组的新增元素，所以直接使用 `args.slice(2)` 作为 inserted 的值即可。最后 inserted 变量中所保存的就是新增的数组元素，我们只需要调用 observeArray 函数对其进行观测即可：

```js
if (inserted) ob.observeArray(inserted)
```

以上是在当前环境支持 `__proto__` 属性的情况，如果不支持则执行 copyAugment 函数，copyAugment 定义在 protoAugment 函数的下方：

```js
/**
 * Augment an target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

copyAugment 函数接收三个参数。在拦截数组变异方法的思路一节中我们讲解了在当前环境不支持 `__proto__` 属性的时候如何做兼容处理，实际上这就是 copyAugment 函数的作用。

我们知道 copyAugment 函数的第三个参数 keys 就是定义在 arrayMethods 对象上的所有函数的键，即所有要拦截的数组变异方法的名称。这样通过 for 循环对其进行遍历，并使用 def 函数在数组实例上定义与数组变异方法同名的且不可枚举的函数，这样就实现了拦截操作。

总之无论是 protoAugment 函数还是 copyAugment 函数，他们的目的只有一个：把数组实例与代理原型或与代理原型中定义的函数联系起来，从而拦截数组变异方法。下面我们再回到 Observer 类的 constructor 函数中，看如下代码：

```js
if (Array.isArray(value)) {
  if (hasProto) {
    protoAugment(value, arrayMethods)
  } else {
    copyAugment(value, arrayMethods, arrayKeys)
  }
  this.observeArray(value)
} else {
  this.walk(value)
}
```

可以发现后面还以该数组实例作为参数调用了 Observer 实例对象的 observeArray 方法：

```js
this.observeArray(value)
```

这句话的作用是什么呢？或者说 observeArray 方法的作用是什么呢？我们知道，当被观测的数据(value)是数组时，会执行 if 语句块的代码从而拦截数组的变异方法，这样当我们尝试通过这些变异方法修改数组时是会触发相应的依赖(观察者)的，比如下面的代码：

```js
const ins = new Vue({
  data: {
    arr: [1, 2]
  }
})

ins.arr.push(3) // 能够触发响应
```

但是如果数组中嵌套了其他的数组或对象，那么嵌套的数组或对象却不是响应的：

```js
const ins = new Vue({
  data: {
    arr: [
      [1, 2]
    ]
  }
})

ins.arr.push(1) // 能够触发响应
ins.arr[0].push(3) // 不能触发响应
```

上面的代码中，直接调用 arr 数组的 push 方法是能够触发响应的，但调用 arr 数组内嵌套数组的 push 方法是不能触发响应的。为了使嵌套的数组或对象同样是响应式数据，我们需要递归的观测那些类型为数组或对象的数组元素，而这就是 observeArray 方法的作用，如下是 observeArray 方法的全部代码：

```js
/**
  * Observe a list of Array items.
  */
observeArray (items: Array<any>) {
  for (let i = 0, l = items.length; i < l; i++) {
    observe(items[i])
  }
}
```

可以发现 observeArray 方法的实现很简单，只需要对数组进行遍历，并对数组元素逐个应用 observe 工厂函数即可，这样就会递归观测数组元素了。

## 数组的特殊性

本小节我们补讲 defineReactive 函数中的一段代码，如下：

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

在 get 函数中如何收集依赖 一节中我们已经讲解了关于依赖收集的内容，但是当时我们留下了如下三行代码没有讲：

```js
if (Array.isArray(value)) {
  dependArray(value)
}
```

现在我们就重点看一下这三句代码，为什么当被读取的属性是数组的时候需要调用 dependArray 函数？

为了弄清楚这个问题，假设我们有如下代码：

```js
<div id="demo">
  {{arr}}
</div>

const ins = new Vue({
  el: '#demo',
  data: {
    arr: [
      { a: 1 }
    ]
  }
})
```

首先我们观察一下数据对象：

```js
{
  arr: [
    { a: 1 }
  ]
}
```

数据对象中的 arr 属性是一个数组，并且数组的一个元素是另外一个对象。我们在 被观测后的数据对象的样子 一节中讲过了，上面的对象在经过观测后将变成如下这个样子：

```js
{
  arr: [
    { a: 1, __ob__ /* 我们将该 __ob__ 称为 ob2 */ },
    __ob__ /* 我们将该 __ob__ 称为 ob1 */
  ]
}
```

如上代码的注释所示，为了便于区别和讲解，我们分别称这两个 `__ob__` 属性为 ob1 和 ob2，然后我们再来观察一下模板：

```js
<div id="demo">
  {{arr}}
</div>
```

在模板里使用了数据 arr，这将会触发数据对象的 arr 属性的 get 函数，我们知道 arr 属性的 get 函数通过闭包引用了两个用来收集依赖的”筐“，一个是属于 arr 属性自身的 dep 对象，另一个是 `childOb.dep` 对象，其中 childOb 就是 ob1。这时依赖会被收集到这两个”筐“中，但大家要注意的是 `ob2.dep` 这个”筐“中，是没有收集到依赖的。有的同学会说：”模板中依赖的数据是 arr，并不是 arr 数组的第一个对象元素，所以 ob2 没有收集到依赖很正常啊“，这是一个错误的想法，因为依赖了数组 arr 就等价于依赖了数组内的所有元素，数组内所有元素的改变都可以看做是数组的改变。但由于 ob2 没有收集到依赖，所以现在就导致如下代码触发不了响应：

```js
ins.$set(ins.$data.arr[0], 'b', 2)
```

我们使用 `$set` 函数为 arr 数组的第一对象元素添加了一个属性 b，这是触发不了响应的。为了能够使得这段代码可以触发响应，就必须让 ob2 收集到依赖，而这就是 dependArray 函数的作用。如下是 dependArray 函数的代码：

```js
function dependArray (value: Array<any>) {
  for (let e, i = 0, l = value.length; i < l; i++) {
    e = value[i]
    e && e.__ob__ && e.__ob__.dep.depend()
    if (Array.isArray(e)) {
      dependArray(e)
    }
  }
}
```

当被读取的数据对象的属性值是数组时，会调用 dependArray 函数，该函数将通过 for 循环遍历数组，并取得数组每一个元素的值，如果该元素的值拥有 `__ob__` 对象和 `__ob__.dep` 对象，那说明该元素也是一个对象或数组，此时只需要手动执行 `__ob__.dep.depend()` 即可达到收集依赖的目的。同时如果发现数组的元素仍然是一个数组，那么需要递归调用 dependArray 继续收集依赖。

那么为什么数组需要这样处理，而纯对象不需要呢？那是因为数组的索引是非响应式的。现在我们已经知道了数据响应系统对纯对象和数组的处理方式是不同，对于纯对象只需要逐个将对象的属性重新定义为访问器属性，并且当属性的值同样为纯对象时进行递归定义即可，而对于数组的处理则是通过拦截数组变异方法的方式，也就是说如下代码是触发不了响应的：

```js
const ins = new Vue({
  data: {
    arr: [1, 2]
  }
})

ins.arr[0] = 3  // 不能触发响应
```

上面的代码中我们试图修改 arr 数组的第一个元素，但这么做是触发不了响应的，因为对于数组来讲，其索引并不是“访问器属性”。正是因为数组的索引不是”访问器属性“，所以当有观察者依赖数组的某一个元素时是触发不了这个元素的 get 函数的，当然也就收集不到依赖。这个时候就是 dependArray 函数发挥作用的时候了。
