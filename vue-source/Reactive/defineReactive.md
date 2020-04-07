# defineReactive 添加 getter 和 setter

defineReactive 的功能就是定义一个响应式对象，给对象动态添加 getter 和 setter，该函数也定义在 core/observer/index.js 文件，内容如下：

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

数据对象 data 拥有一个叫做 a 的属性，且属性 a 的值是另外一个对象，该对象拥有一个叫做 b 的属性。那么经过 observe 处理之后， data 和 `data.a` 这两个对象都被定义了 `__ob__` 属性，并且访问器属性 a 和 b 的 setter/getter 都通过闭包引用着属于自己的 Dep 实例对象和 childOb 对象：

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
