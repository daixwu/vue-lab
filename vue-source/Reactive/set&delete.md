# `set($set)` 和 `delete($delete)` 的实现

现在我们是时候讲解一下 `Vue.set` 和 `Vue.delete` 函数的实现了，我们知道 Vue 数据响应系统的原理的核心是通过 `Object.defineProperty` 函数将数据对象的属性转换为访问器属性，从而使得我们能够拦截到属性的读取和设置，但正如官方文档中介绍的那样，Vue 是没有能力拦截到为一个对象(或数组)添加属性(或元素)的，而 `Vue.set` 和 `Vue.delete` 就是为了解决这个问题而诞生的。

同时为了方便使用， Vue 还在实例对象上定义了 `$set` 和 `$delete` 方法，实际上 `$set` 和 `$delete` 方法仅仅是 `Vue.set` 和 `Vue.delete` 的别名，为了证明这点，我们首先来看看 `$set` 和 `$delete` 的实现，还记得 `$set` 和 `$delete` 方法定义在哪里吗？不记得也没关系，我们可以通过查看附录 Vue 构造函数整理-原型 找到 `$set` 和 `$delete` 方法的定义位置，我们发现 `$set` 和 `$delete` 定义在 src/core/instance/state.js 文件的 stateMixin 函数中，如下代码：

```js
export function stateMixin (Vue: Class<Component>) {
  // 省略...

  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: any,
    options?: Object
  ): Function {
    // 省略...
  }
}
```

可以看到 `$set` 和 `$delete` 的值分别是是 set 和 del，根据文件头部的引用关系可知 set 和 del 来自 src/core/observer/index.js 文件中定义的 set 函数和 del 函数。

接着我们再来看看 `Vue.set` 和 `Vue.delete` 函数的定义，如果你同样不记得这两个函数时在哪里定义的也没关系，可以查看附录 Vue 构造函数整理-全局API，我们发现这两个函数是在 initGlobalAPI 函数中定义的，打开 src/core/global-api/index.js 文件，找到 initGlobalAPI 函数如下：

```js
export function initGlobalAPI (Vue: GlobalAPI) {
  // 省略...

  Vue.set = set
  Vue.delete = del
  
  // 省略...
}
```

可以发现 `Vue.set` 函数和 `Vue.delete` 函数的值同样是来自 src/core/observer/index.js 文件中定义的 set 函数和 del 函数。现在我们可以坚信 `Vue.set` 其实就是 `$set`，而 `Vue.delete` 就是 `$delete`，所以现在我们只需要搞清楚定义在 src/core/observer/index.js 文件中的 set 函数和 del 函数是如何实现的就可以了。

## `Vue.set/$set`

首先我们来看一下 `Vue.set/$set` 函数，打开 src/core/observer/index.js 文件，找到 set 函数，它定义在 defineReactive 函数的下面，如下是 set 函数的定义：

```js
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```

set 函数接收三个参数，相信很多同学都有使用过 `Vue.set/$set` 函数的经验，那么大家对这三个参数应该不陌生。target 可能是数组或者是普通对象，key 代表的是数组的下标或者是对象的键值，val 代表添加的值。

下面我们一点点来看 set 函数的代码，首先是一个 if 语句块：

```js
if (process.env.NODE_ENV !== 'production' &&
  (isUndef(target) || isPrimitive(target))
) {
  warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
}
```

该 if 语句块的判断条件中包含两个函数，分别是 isUndef 和 isPrimitive，可以在附录 shared/util.js 文件工具方法全解 中找到关于这两个函数的讲解。isUndef 函数用来判断一个值是否是 undefined 或 null，如果是则返回 true，isPrimitive 函数用来判断一个值是否是原始类型值，如果是则返回 true。所以如上代码 if 语句块的作用是：如果 set 函数的第一个参数是 undefined 或 null 或者是原始类型值，那么在非生产环境下会打印警告信息。这么做是合理的，因为理论上只能为对象(或数组)添加属性(或元素)。

紧接着又是一段 if 语句块，如下：

```js
if (Array.isArray(target) && isValidArrayIndex(key)) {
  target.length = Math.max(target.length, key)
  target.splice(key, 1, val)
  return val
}
```

这段代码对 target 和 key 这两个参数做了校验，如果 target 是一个数组，并且 key 是一个有效的数组索引，那么就会执行 if 语句块的内容。在校验 key 是否是有效的数组索引时使用了 isValidArrayIndex 函数，可以在附录 shared/util.js 文件工具方法全解 中查看详细讲解。也就是说当我们尝试使用 `Vue.set/$set` 为数组设置某个元素值的时候就会执行 if 语句块的内容，如下例子：

```js
const ins = new Vue({
  data: {
    arr: [1, 2]
  }
})

ins.$data.arr[0] = 3 // 不能触发响应
ins.$set(ins.$data.arr, 0, 3) // 能够触发响应
```

上面的代码中我们直接修改 `arr[0]` 的值是不能够触发响应的，但是如果我们使用 `$set` 函数重新设置 arr 数组索引为 0 的元素的值，这样是能够触发响应的，我们看看 `$set` 函数是如何实现的，注意如下高亮代码：

```js
if (Array.isArray(target) && isValidArrayIndex(key)) {
  target.length = Math.max(target.length, key)
  target.splice(key, 1, val)
  return val
}
```

原理其实很简单，我们知道数组的 splice 变异方法能够完成数组元素的删除、添加、替换等操作。而 `target.splice(key, 1, val)` 就利用了替换元素的能力，将指定位置元素的值替换为新值，同时由于 splice 方法本身是能够触发响应的，所以一切看起来如此简单。

另外大家注意在调用 `target.splice` 函数之前，需要修改数组的长度：

```js
target.length = Math.max(target.length, key)
```

将数组的长度修改为 target.length 和 key 中的较大者，否则如果当要设置的元素的索引大于数组长度时 splice 无效。

再往下依然是一个 if 语句块，如下：

```js
if (key in target && !(key in Object.prototype)) {
  target[key] = val
  return val
}
```

如果 target 不是一个数组，那么必然就是纯对象了，当给一个纯对象设置属性的时候，假设该属性已经在对象上有定义了，那么只需要直接设置该属性的值即可，这将自动触发响应，因为已存在的属性是响应式的。但这里要注意的是 if 语句的两个条件：

- `key in target`
- `!(key in Object.prototype)`

这两个条件保证了 key 在 target 对象上，或在 target 的原型链上，同时必须不能在 `Object.prototype` 上。这里我们需要提一点，上面这段代码为什么不像如下代码这样做：

```js
if (hasOwn(target, key)) {
  target[key] = val
  return val
}
```

使用 hasOwn 检测 key 是不是属于 target 自身的属性不就好了？其实原本代码的确是这样写的，后来因为一个 issue 代码变成了现在这个样子，可以 点击这里查看 [issue](https://github.com/vuejs/vue/issues/6845)。

我们继续看代码，接下来是这样一段代码，这是 set 函数剩余的全部代码，如下：

```js
const ob = (target: any).__ob__
// if 语句省略
defineReactive(ob.value, key, val)
ob.dep.notify()
return val
```

如果代码运行到了这里，那说明正在给对象添加一个全新的属性，代码中首先定义了 ob 常量，它是数据对象 `__ob__` 属性的引用。然后使用 defineReactive 函数设置属性值，这是为了保证新添加的属性是响应式的。之后调用了 `__ob__.dep.notify()` 从而触发响应。这就是添加全新属性触发响应的原理。

再看如下if语句中的代码：

```js
if (!ob) {
  target[key] = val
  return val
}
```

高亮的部分是一个 if 语句块，我们知道 target 也许原本就是非响应的，这个时候 `target.__ob__` 是不存在的，所以当发现 `target.__ob__` 不存在时，就简单的赋值即可。

最后我们来看一下剩下的这段 if 语句块：

```js
const ob = (target: any).__ob__
if (target._isVue || (ob && ob.vmCount)) {
  process.env.NODE_ENV !== 'production' && warn(
    'Avoid adding reactive properties to a Vue instance or its root $data ' +
    'at runtime - declare it upfront in the data option.'
  )
  return val
}
```

这个 if 语句块有两个条件，只要有一个条件成立，就会执行 if 语句块内的代码。我们来看第一个条件 `target._isVue`，我们知道 Vue 实例对象拥有 `_isVue` 属性，所以当第一个条件成立时，那么说明你正在使用 `Vue.set/$set` 函数为 Vue 实例对象添加属性，为了避免属性覆盖的情况出现，`Vue.set/$set` 函数不允许这么做，在非生产环境下会打印警告信息。

第二个条件是：(`ob && ob.vmCount`)，我们知道 ob 就是 `target.__ob__` 那么 `ob.vmCount` 是什么呢？为了搞清这个问题，我们回到 observe 工厂函数中，如下代码：

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 省略...
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

observe 函数接收两个参数，第二个参数指示着被观测的数据对象是否是根数据对象，什么叫根数据对象呢？那就看 asRootData 什么时候为 true 即可，我们找到 initData 函数中，他在 src/core/instance/state.js 文件中，如下：

```js
function initData (vm: Component) {
  let data = vm.$options.data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  
  // 省略...

  // observe data
  observe(data, true /* asRootData */)
}
```

可以看到在调用 observe 观测 data 对象的时候 asRootData 参数为 true。而在后续的递归观测中调用 observe 的时候省略了 asRootData 参数。所以所谓的根数据对象就是 data 对象。这时候我们再来看如下代码：

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 省略...
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

可以发现，根数据对象将拥有一个特质，即 `target.__ob__.vmCount > 0`，这样条件 (`ob && ob.vmCount`) 是成立的，也就是说：当使用 `Vue.set/$set` 函数为根数据对象添加属性时，是不被允许的。

那么为什么不允许在根数据对象上添加属性呢？因为这样做是永远触发不了依赖的。原因就是根数据对象的 Observer 实例收集不到依赖(观察者)，如下：

```js
const data = {
  obj: {
    a: 1
    __ob__ // ob2
  },
  __ob__ // ob1
}
new Vue({
  data
})
```

如上代码所示，obj 就是属于根数据的 Observer 实例对象，如果想要在根数据上使用 `Vue.set/$set` 并触发响应：

```js
Vue.set(data, 'someProperty', 'someVal')
```

那么 data 字段必须是响应式数据才行，这样当 data 字段被依赖时，才能够收集依赖(观察者)到两个“筐”中(data属性自身的 dep以及`data.__ob__`)。这样在 `Vue.set/$set` 函数中才有机会触发根数据的响应。但 data 本身并不是响应的，这就是问题所在。

## `Vue.delete/$delete`

接下来我们继续看一下 Vue.delete/$delete 函数的实现，仍然是 src/core/observer/index.js 文件，找到 del 函数：

```js
export function del (target: Array<any> | Object, key: any) {
  // 省略...
}
```

del 函数接收两个参数，分别是将要被删除属性的目标对象 target 以及要删除属性的键名 key，与 set 函数相同，在函数体的开头是如下 if 语句块：

```js
if (process.env.NODE_ENV !== 'production' &&
  (isUndef(target) || isPrimitive(target))
) {
  warn(`Cannot delete reactive property on undefined, null, or primitive value: ${(target: any)}`)
}
```

检测 target 是否是 undefined 或 null 或者是原始类型值，如果是的话那么在非生产环境下会打印警告信息。

接着是如下这段 if 语句块：

```js
if (Array.isArray(target) && isValidArrayIndex(key)) {
  target.splice(key, 1)
  return
}
```

很显然，如果我们使用 `Vue.delete/$delete` 去删除一个数组的索引，如上这段代码将被执行，当然了前提是参数 key 需要是一个有效的数组索引。与为数组添加元素类似，移除数组元素同样使用了数组的 splice 方法，大家知道这样是能够触发响应的。

再往下是如下这段 if 语句块：

```js
const ob = (target: any).__ob__
if (target._isVue || (ob && ob.vmCount)) {
  process.env.NODE_ENV !== 'production' && warn(
    'Avoid deleting properties on a Vue instance or its root $data ' +
    '- just set it to null.'
  )
  return
}
```

与不能使用 `Vue.set/$set` 函数为根数据或 Vue 实例对象添加属性一样，同样不能使用 `Vue.delete/$delete` 删除 Vue 实例对象或根数据的属性。不允许删除 Vue 实例对象的属性，是出于安全因素的考虑。而不允许删除根数据对象的属性，是因为这样做也是触发不了响应的，关于触发不了响应的原因，我们在讲解 `Vue.set/$set` 时已经分析过了。

接下来是 `Vue.delete/$delete` 函数的最后一段代码，如下：

```js
if (!hasOwn(target, key)) {
  return
}
delete target[key]
if (!ob) {
  return
}
ob.dep.notify()
```

首先使用 hasOwn 函数检测 key 是否是 target 对象自身拥有的属性，如果不是那么直接返回(return)。很好理解，如果你将要删除的属性原本就不在该对象上，那么自然什么都不需要做。

如果 key 存在于 target 对象上，那么代码将继续运行，此时将使用 delete 语句从 target 上删除属性 key。最后判断 ob 对象是否存在，如果不存在说明 target 对象原本就不是响应的，所以直接返回(return)即可。如果 ob 对象存在，说明 target 对象是响应的，需要触发响应才行，即执行 `ob.dep.notify()`。
