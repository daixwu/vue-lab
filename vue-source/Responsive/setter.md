# set 函数如何触发依赖

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

为什么要这么做呢？具体可以查看这个 [issue](https://github.com/vuejs/vue/pull/7302)。简单的说就是当属性原本存在 get 拦截器函数时，在初始化的时候不要触发 get 函数，只有当真正的获取该属性的值的时候，再通过调用缓存下来的属性原本的 getter 函数取值即可。所以看到这里我们能够发现，如果数据对象的某个属性原本就拥有自己的 get 函数，那么这个属性就不会被深度观测，因为当属性原本存在 getter 时，是不会触发取值动作的，即 `val = obj[key]` 不会执行，所以 val 是 undefined，这就导致在后面深度观测的语句中传递给 observe 函数的参数是 undefined。

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
