# Observer 构造函数

其实真正将数据对象转换成响应式数据的是 Observer 函数，它是一个构造函数，同样定义在 core/observer/index.js 文件下，如下是简化后的代码：

```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    // 省略...
  }

  walk (obj: Object) {
    // 省略...
  }
  
  observeArray (items: Array<any>) {
    // 省略...
  }
}
```

可以清晰的看到 Observer 类的实例对象将拥有三个实例属性，分别是 value、dep 和 vmCount 以及两个实例方法 walk 和 observeArray。Observer 类的构造函数接收一个参数，即数据对象。下面我们就从 constructor 方法开始，研究实例化一个 Observer 类时都做了哪些事情。

## 数据对象的 `__ob__` 属性

如下是 constructor 方法的全部代码：

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

constructor 方法的参数就是在实例化 Observer 实例时传递的参数，即数据对象本身，可以发现，实例对象的 value 属性引用了数据对象：

```js
this.value = value
```

实例对象的 dep 属性，保存了一个新创建的 Dep 实例对象：

```js
this.dep = new Dep()
```

那么这里的 Dep 是什么呢？就像我们在了解数据响应系统基本思路 中所讲到的，它就是一个收集依赖的“筐”。但这个“筐”并不属于某一个字段，后面我们会发现，这个筐是属于某一个对象或数组的。

实例对象的 vmCount 属性被设置为 0：`this.vmCount = 0`。

初始化完成三个实例属性之后，使用 def 函数，为数据对象定义了一个 `__ob__` 属性，这个属性的值就是当前 Observer 实例对象。其中 def 函数其实就是 Object.defineProperty 函数的简单封装，之所以这里使用 def 函数定义 `__ob__` 属性是因为这样可以定义不可枚举的属性，这样后面遍历数据对象的时候就能够防止遍历到 `__ob__` 属性。

假设我们的数据对象如下：

```js
const data = {
  a: 1
}
```

那么经过 def 函数处理之后，data 对象应该变成如下这个样子：

```js
const data = {
  a: 1,
  // __ob__ 是不可枚举的属性
  __ob__: {
    value: data, // value 属性指向 data 数据对象本身，这是一个循环引用
    dep: dep实例对象, // new Dep()
    vmCount: 0
  }
}
```

## 响应式数据之纯对象的处理

接着进入一个 `if...else` 判断分支：

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

该判断用来区分数据对象到底是数组还是一个纯对象，因为对于数组和纯对象的处理方式是不同的，为了更好地理解我们先看数据对象是一个纯对象的情况，这个时候代码会走 else 分支，即执行 `this.walk(value)` 函数，我们知道这个函数实例对象方法，找到这个方法：

```js
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    defineReactive(obj, keys[i])
  }
}
```

walk 方法很简单，首先使用 `Object.keys(obj)` 获取对象所有可枚举的属性，然后使用 for 循环遍历这些属性，同时为每个属性调用了 defineReactive 函数。

## defineReactive 函数

那我们就看一看 defineReactive 函数都做了什么，该函数也定义在 core/observer/index.js 文件，内容如下：

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 省略...
    },
    set: function reactiveSetter (newVal) {
      // 省略...
    }
  })
}
```

defineReactive 函数的核心就是 将数据对象的数据属性转换为访问器属性，即为数据对象的属性设置一对 getter/setter，但其中做了很多处理边界条件的工作。defineReactive 接收五个参数，但是在 walk 方法中调用 defineReactive 函数时只传递了前两个参数，即数据对象和属性的键名。我们看一下 defineReactive 的函数体，首先定义了 dep 常量，它是一个 Dep 实例对象：

```js
const dep = new Dep()
```

我们在讲解 Observer 的 constructor 方法时看到过，在 constructor 方法中为数据对象定义了一个 `__ob__` 属性，该属性是一个 Observer 实例对象，且该对象包含一个 Dep 实例对象：

```js
const data = {
  a: 1,
  __ob__: {
    value: data,
    dep: dep实例对象, // new Dep() , 包含 Dep 实例对象
    vmCount: 0
  }
}
```

当时我们说过 `__ob__.dep` 这个 Dep 实例对象的作用与我们在讲解数据响应系统基本思路一节中所说的“筐”的作用不同。至于他的作用是什么我们后面会讲到。其实与我们前面所说过的“筐”的作用相同的 Dep 实例对象是在 defineReactive 函数一开始定义的 dep 常量，即：

```js
const dep = new Dep()
```

这个 dep 常量所引用的 Dep 实例对象才与我们前面讲过的“筐”的作用相同。细心的同学可能已经注意到了 dep 在访问器属性的 getter/setter 中被闭包引用，如下：

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  // 省略...

  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 这里闭包引用了上面的 dep 常量
        dep.depend()
        // 省略...
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      // 省略...

      // 这里闭包引用了上面的 dep 常量
      dep.notify()
    }
  })
}
```

如上面的代码中注释所写的那样，在访问器属性的 getter/setter 中，通过闭包引用了前面定义的“筐”，即 dep 常量。这里大家要明确一件事情，即每一个数据字段都通过闭包引用着属于自己的 dep 常量。因为在 walk 函数中通过循环遍历了所有数据对象的属性，并调用 defineReactive 函数，所以每次调用 defineReactive 定义访问器属性时，该属性的 setter/getter 都闭包引用了一个属于自己的“筐”。假设我们有如下数据字段：

```js
const data = {
  a: 1,
  b: 2
}
```

那么字段 `data.a` 和 `data.b` 都将通过闭包引用属于自己的 Dep 实例对象，每个字段的 Dep 对象都被用来收集那些属于对应字段的依赖。

在定义 dep 常量之后，是这样一段代码：

```js
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
  return
}
```

首先通过 `Object.getOwnPropertyDescriptor` 函数获取该字段可能已有的属性描述对象，并将该对象保存在 property 常量中，接着是一个 if 语句块，判断该字段是否是可配置的，如果不可配置(`property.configurable === false`)，那么直接 return ，即不会继续执行 defineReactive 函数。这么做也是合理的，因为一个不可配置的属性是不能使用也没必要使用 `Object.defineProperty` 改变其属性定义的。

再往下是这样一段代码：

```js
// cater for pre-defined getter/setters
const getter = property && property.get
const setter = property && property.set
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}

let childOb = !shallow && observe(val)
```

这段代码的前两句定义了 getter 和 setter 常量，分别保存了来自 property 对象的 get 和 set 函数，我们知道 property 对象是属性的描述对象，一个对象的属性很可能已经是一个访问器属性了，所以该属性很可能已经存在 get 或 set 方法。由于接下来会使用 `Object.defineProperty` 函数重新定义属性的 setter/getter，这会导致属性原有的 set 和 get 方法被覆盖，所以要将属性原有的 setter/getter 缓存，并在重新定义的 set 和 get 方法中调用缓存的函数，从而做到不影响属性的原有读写操作。

上面这段代码中比较难理解的是 if 条件语句：

```js
(!getter || setter) && arguments.length === 2
```

其中 `arguments.length === 2` 这个条件好理解，当只传递两个参数时，说明没有传递第三个参数 val，那么此时需要根据 key 主动去对象上获取相应的值，即执行 if 语句块内的代码：`val = obj[key]`。那么 (`!getter || setter`) 这个条件的意思是什么呢？要理解这个条件我们需要思考一些实际应用的场景，或者说边界条件，但是现在还不适合给大家讲解，我们等到讲解完整个 defineReactive 函数之后，再回头来说。

在 if 语句块的下面，是这句代码：

```js
let childOb = !shallow && observe(val)
```

定义了 childOb 变量，我们知道，在 if 语句块里面，获取到了对象属性的值 val，但是 val 本身有可能也是一个对象，那么此时应该继续调用 `observe(val)` 函数观测该对象从而深度观测数据对象。但前提是 defineReactive 函数的最后一个参数 shallow 应该是假，即 `!shallow` 为真时才会继续调用 observe 函数深度观测，由于在 walk 函数中调用 defineReactive 函数时没有传递 shallow 参数，所以该参数是 undefined，那么也就是说默认就是深度观测。其实非深度观测的场景我们早就遇到过了，即 initRender 函数中在 Vue 实例对象上定义 `$attrs` 属性和 `$listeners` 属性时就是非深度观测，如下：

```js
defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true) // 最后一个参数 shallow 为 true
defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
```

大家要注意一个问题，即使用 `observe(val)` 深度观测数据对象时，这里的 val 未必有值，因为必须在满足条件 `(!getter || setter) && arguments.length === 2` 时，才会触发取值的动作：`val = obj[key]`，所以一旦不满足条件即使属性是有值的但是由于没有触发取值的动作，所以 val 依然是 undefined。这就会导致深度观测无效。

## 被观测后的数据对象的样子

现在我们需要明确一件事情，那就是一个数据对象经过了 observe 函数处理之后变成了什么样子，假设我们有如下数据对象：

```js
const data = {
  a: {
    b: 1
  }
}

observe(data)
```

数据对象 data 拥有一个叫做 a 的属性，且属性 a 的值是另外一个对象，该对象拥有一个叫做 b 的属性。那么经过 observe 处理之后， data 和 data.a 这两个对象都被定义了 `__ob__` 属性，并且访问器属性 a 和 b 的 setter/getter 都通过闭包引用着属于自己的 Dep 实例对象和 childOb 对象：

```js
const data = {
  // 属性 a 通过 setter/getter 通过闭包引用着 dep 和 childOb
  a: {
    // 属性 b 通过 setter/getter 通过闭包引用着 dep 和 childOb
    b: 1
    __ob__: {a, dep, vmCount}
  }
  __ob__: {data, dep, vmCount}
}
```

需要注意的是，属性 a 闭包引用的 childOb 实际上就是 `data.a.__ob__`。而属性 b 闭包引用的 childOb 是 undefined，因为属性 b 是基本类型值，并不是对象也不是数组。

## 在 get 函数中如何收集依赖

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

上面的代码之所以能够触发依赖，就是因为 Vue.set 函数中触发了收集在 `data.a.__ob__.dep` 这个”筐“中的依赖：

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

## 在 set 函数中如何触发依赖

在 get 函数中收集了依赖之后，接下来我们就要看一下在 set 函数中是如何触发依赖的，即当属性被修改的时候如何触发依赖。set 函数如下：

```js
set: function reactiveSetter (newVal) {
  const value = getter ? getter.call(obj) : val
  /* eslint-disable no-self-compare */
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  /* eslint-enable no-self-compare */
  if (process.env.NODE_ENV !== 'production' && customSetter) {
    customSetter()
  }
  if (setter) {
    setter.call(obj, newVal)
  } else {
    val = newVal
  }
  childOb = !shallow && observe(newVal)
  dep.notify()
}
```

我们知道 get 函数主要完成了两部分重要的工作，一个是返回正确的属性值，另一个是收集依赖。与 get 函数类似， set 函数也要完成两个重要的事情，第一正确地为属性设置新值，第二是能够触发相应的依赖。

首先 set 函数接收一个参数 newVal，即该属性被设置的新值。在函数体内，先执行了这样一句话：

```js
const value = getter ? getter.call(obj) : val
```

这句话与 get 函数体的第一句话相同，即取得属性原有的值，为什么要取得属性原来的值呢？很简单，因为我们需要拿到原有的值与新的值作比较，并且只有在原有值与新设置的值不相等的情况下才需要触发依赖和重新设置属性值，否则意味着属性值并没有改变，当然不需要做额外的处理。如下代码：

```js
/* eslint-disable no-self-compare */
if (newVal === value || (newVal !== newVal && value !== value)) {
  return
}
```

这里就对比了新值和旧值：`newVal === value`。如果新旧值全等，那么函数直接 return，不做任何处理。但是除了对比新旧值之外，我们还注意到，另外一个条件：

```js
(newVal !== newVal && value !== value)
```

如果满足该条件，同样不做任何处理，那么这个条件什么意思呢？`newVal !== newVal` 说明新值与新值自身都不全等，同时旧值与旧值自身也不全等，大家想一下在 js 中什么时候会出现一个值与自身都不全等的？答案就是 `NaN`：

```js
NaN === NaN // false
```

所以我们现在重新分析一下这个条件，首先 `value !== value` 成立那说明该属性的原有值就是 `NaN`，同时 `newVal !== newVal` 说明为该属性设置的新值也是 `NaN`，所以这个时候新旧值都是 `NaN`，等价于属性的值没有变化，所以自然不需要做额外的处理了，set 函数直接 return 。

再往下又是一个 if 语句块：

```js
/* eslint-enable no-self-compare */
if (process.env.NODE_ENV !== 'production' && customSetter) {
  customSetter()
}
```

上面这段代码的作用是，如果 customSetter 函数存在，那么在非生产环境下执行 customSetter 函数。其中 customSetter 函数是 defineReactive 函数的第四个参数。那么 customSetter 函数的作用是什么呢？其实我们在讲解 initRender 函数的时候就讲解过 customSetter 的作用，如下是 initRender 函数中的一段代码：

```js
defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
  !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
}, true)
```

上面的代码中使用 defineReactive 在 Vue 实例对象 vm 上定义了 `$attrs` 属性，可以看到传递给 defineReactive 函数的第四个参数是一个箭头函数，这个函数就是 customSetter，这个箭头函数的作用是当你尝试修改 `vm.$attrs` 属性的值时，打印一段信息：`$attrs` 属性是只读的。这就是 customSetter 函数的作用，用来打印辅助信息，当然除此之外你可以将 customSetter 用在任何适合使用它的地方。

我们回到 set 函数，再往下是这样一段代码：

```js
if (setter) {
  setter.call(obj, newVal)
} else {
  val = newVal
}
```

上面这段代码的意图很明显，即正确地设置属性值，首先判断 setter 是否存在，我们知道 setter 常量存储的是属性原有的 set 函数。即如果属性原来拥有自身的 set 函数，那么应该继续使用该函数来设置属性的值，从而保证属性原有的设置操作不受影响。如果属性原本就没有 set 函数，那么就设置 val 的值：`val = newVal`。

接下来就是 set 函数的最后两句代码，如下：

```js
childOb = !shallow && observe(newVal)
dep.notify()
```

我们知道，由于属性被设置了新的值，那么假如我们为属性设置的新值是一个数组或者纯对象，那么该数组或纯对象是未被观测的，所以需要对新值进行观测，这就是第一句代码的作用，同时使用新的观测对象重写 childOb 的值。当然了，这些操作都是在 !shallow 为真的情况下，即需要深度观测的时候才会执行。最后是时候触发依赖了，我们知道 dep 是属性用来收集依赖的”筐“，现在我们需要把”筐“里的依赖都执行一下，而这就是 `dep.notify()` 的作用。

至此 set 函数我们就讲解完毕了。

## 保证定义响应式数据行为的一致性

本节我们主要讲解 defineReactive 函数中的一段代码，即：

```js
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}
```

在之前的讲解中，我们没有详细地讲解如上代码所示的这段 if 语句块。该 if 语句有两个条件：

- 第一：(`!getter || setter`)
- 第二：`arguments.length === 2`

并且这两个条件要同时满足才能会根据 key 去对象 obj 上取值：`val = obj[key]`，否则就不会触发取值的动作，触发不了取值的动作就意味着 val 的值为 undefined，这会导致 if 语句块后面的那句深度观测的代码无效，即不会深度观测：

```js
// val 是 undefined，不会深度观测
let childOb = !shallow && observe(val)
```

对于第二个条件，很好理解，当传递参数的数量为 2 时，说明没有传递第三个参数 val，那么当然需要通过执行 `val = obj[key]` 去获取属性值。比较难理解的是第一个条件，即 (`!getter || setter`)，要理解这个问题你需要知道 Vue 代码的变更，以及为什么变更。其实在最初并没有上面这段 if 语句块，在 walk 函数中是这样调用 defineReactive 函数的：

```js
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    // 这里传递了第三个参数
    defineReactive(obj, keys[i], obj[keys[i]])
  }
}
```

可以发现在调用 defineReactive 函数的时候传递了第三个参数，即属性值。这是最初的实现，后来变成了如下这样：

```js
walk (obj: Object) {
  const keys = Object.keys(obj)
  for (let i = 0; i < keys.length; i++) {
    // 在 walk 函数中调用 defineReactive 函数时暂时不获取属性值
    defineReactive(obj, keys[i])
  }
}

// ================= 分割线 =================

// 在 defineReactive 函数内获取属性值
if (!getter && arguments.length === 2) {
  val = obj[key]
}
```

在 walk 函数中调用 defineReactive 函数时去掉了第三个参数，而是在 defineReactive 函数体内增加了一段 if 分支语句，当发现调用 defineReactive 函数时传递了两个参数，同时只有在属性没有 get 函数的情况下才会通过 `val = obj[key]` 取值。

为什么要这么做呢？具体可以查看这个 issue。简单的说就是当属性原本存在 get 拦截器函数时，在初始化的时候不要触发 get 函数，只有当真正的获取该属性的值的时候，再通过调用缓存下来的属性原本的 getter 函数取值即可。所以看到这里我们能够发现，如果数据对象的某个属性原本就拥有自己的 get 函数，那么这个属性就不会被深度观测，因为当属性原本存在 getter 时，是不会触发取值动作的，即 `val = obj[key]` 不会执行，所以 val 是 undefined，这就导致在后面深度观测的语句中传递给 observe 函数的参数是 undefined。

举个例子，如下：

```js
const data = {
  getterProp: {
    a: 1
  }
}

new Vue({
  data,
  watch: {
    'getterProp.a': () => {
      console.log('这句话会输出')
    }
  }
})
```

上面的代码中，我们定义了数据 data，data 是一个嵌套的对象，在 watch 选项中观察了属性 `getterProp.a`，当我们修改 `getterProp.a` 的值时，以上代码是能够正常输出的，这也是预期行为。再看如下代码：

```js
const data = {}
Object.defineProperty(data, 'getterProp', {
  enumerable: true,
  configurable: true,
  get: () => {
    return {
      a: 1
    }
  }
})

const ins = new Vue({
  data,
  watch: {
    'getterProp.a': () => {
      console.log('这句话不会输出')
    }
  }
})
```

我们仅仅修改了定义数据对象 data 的方式，此时 `data.getterProp` 本身已经是一个访问器属性，且拥有 get 方法。此时当我们尝试修改 `getterProp.a` 的值时，在 watch 中观察 `getterProp.a` 的函数不会被执行。这是因为属性 getterProp 是一个拥有 get 拦截器函数的访问器属性，而当 Vue 发现该属性拥有原本的 getter 时，是不会深度观测的。

那么为什么当属性拥有自己的 getter 时就不会对其深度观测了呢？有两方面的原因，第一：由于当属性存在原本的 getter 时在深度观测之前不会取值，所以在深度观测语句执行之前取不到属性值从而无法深度观测。第二：之所以在深度观测之前不取值是因为属性原本的 getter 由用户定义，用户可能在 getter 中做任何意想不到的事情，这么做是出于避免引发不可预见行为的考虑。

我们回过头来再看这段 if 语句块：

```js
if (!getter && arguments.length === 2) {
  val = obj[key]
}
```

这么做难道不会有什么问题吗？当然有问题，我们知道当数据对象的某一个属性只拥有 get 拦截器函数而没有 set 拦截器函数时，此时该属性不会被深度观测。但是经过 defineReactive 函数的处理之后，该属性将被重新定义 getter 和 setter，此时该属性变成了既拥有 get 函数又拥有 set 函数。并且当我们尝试给该属性重新赋值时，那么新的值将会被观测。这时候矛盾就产生了：原本该属性不会被深度观测，但是重新赋值之后，新的值却被观测了。

这就是所谓的 定义响应式数据时行为的不一致，为了解决这个问题，采用的办法是当属性拥有原本的 setter 时，即使拥有 getter 也要获取属性值并观测之，这样代码就变成了最终这个样子：

```js
if ((!getter || setter) && arguments.length === 2) {
  val = obj[key]
}
```

## 响应式数据之数组的处理

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
