# observe 工厂函数

了解了数据响应系统的基本思路，我们是时候回过头来深入研究 Vue 的数据响应系统是如何实现的了，我们回到 initData 函数的最后一句代码：

```js
// observe data
observe(data, true /* asRootData */)
```

调用了 observe 函数观测数据，observe 函数来自于 core/observer/index.js 文件，打开该文件找到 observe 函数：

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

如上是 observe 函数的全部代码， observe 函数接收两个参数，第一个参数是要观测的数据，第二个参数是一个布尔值，代表将要被观测的数据是否是根级数据。在 observe 函数的一开始是一段 if 判断语句：

```js
if (!isObject(value) || value instanceof VNode) {
  return
}
```

用来判断如果要观测的数据不是一个对象或者是 VNode 实例，则直接 return 。接着定义变量 ob，该变量用来保存 Observer 实例，可以发现 observe 函数的返回值就是 ob。紧接着又是一个 `if...else` 分支：

```js
if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
  ob = value.__ob__
} else if (
  shouldObserve &&
  !isServerRendering() &&
  (Array.isArray(value) || isPlainObject(value)) &&
  Object.isExtensible(value) &&
  !value._isVue
) {
  ob = new Observer(value)
}
```

我们先看 if 分支的判断条件，首先使用 hasOwn 函数检测数据对象 value 自身是否含有 `__ob__` 属性，并且 `__ob__` 属性应该是 Observer 的实例。如果为真则直接将数据对象自身的 `__ob__` 属性的值作为 ob 的值：`ob = value.__ob__`。那么 `__ob__` 是什么呢？其实当一个数据对象被观测之后将会在该对象上定义 `__ob__` 属性，所以 if 分支的作用是用来避免重复观测一个数据对象。

接着我们再来看看 `else...if` 分支，如果数据对象上没有定义 `__ob__` 属性，那么说明该对象没有被观测过，进而会判断 `else...if` 分支，如果 `else...if` 分支的条件为真，那么会执行 `ob = new Observer(value)` 对数据对象进行观测。也就是说只有当数据对象满足所有 `else...if` 分支的条件才会被观测，我们看看需要满足什么条件：

**第一个条件是 shouldObserve 必须为 true**
shouldObserve 变量也定义在 core/observer/index.js 文件内，如下：

```js
/**
 * In some cases we may want to disable observation inside a component's
 * update computation.
 */
export let shouldObserve: boolean = true

export function toggleObserving (value: boolean) {
  shouldObserve = value
}
```

该变量的初始值为 true，在 shouldObserve 变量的下面定义了 toggleObserving 函数，该函数接收一个布尔值参数，用来切换 shouldObserve 变量的真假值，我们可以把 shouldObserve 想象成一个开关，为 true 时说明打开了开关，此时可以对数据进行观测，为 false 时可以理解为关闭了开关，此时数据对象将不会被观测。为什么这么设计呢？原因是有一些场景下确实需要这个开关从而达到一些目的，后面我们遇到的时候再仔细来说。

**第二个条件是 `!isServerRendering()` 必须为真**
`isServerRendering()` 函数的返回值是一个布尔值，用来判断是否是服务端渲染。也就是说只有当不是服务端渲染的时候才会观测数据，关于这一点 Vue 的服务端渲染文档中有相关介绍，我们不做过多说明。

**第三个条件是 `(Array.isArray(value) || isPlainObject(value))` 必须为真**
这个条件很好理解，只有当数据对象是数组或纯对象的时候，才有必要对其进行观测。

**第四个条件是 `Object.isExtensible(value)` 必须为真**
也就是说要被观测的数据对象必须是可扩展的。一个普通的对象默认就是可扩展的，以下三个方法都可以使得一个对象变得不可扩展：Object.preventExtensions()、Object.freeze() 以及 Object.seal()。

**第五个条件是 !value._isVue 必须为真**
我们知道 Vue 实例对象拥有 _isVue 属性，所以这个条件用来避免 Vue 实例对象被观测。

当一个对象满足了以上五个条件时，就会执行 `else...if` 语句块的代码，即创建一个 Observer 实例：

```js
ob = new Observer(value)
```
