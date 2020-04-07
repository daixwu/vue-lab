# Observer 构造函数

其实真正将数据对象转换成响应式数据的是 Observer，它是一个类，它的作用是给对象的属性添加 getter 和 setter，用于依赖收集和派发更新，同样定义在 core/observer/index.js 文件下，如下是简化后的代码：

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

初始化完成三个实例属性之后，使用 def 函数，为数据对象定义了一个 `__ob__` 属性，这个属性的值就是当前 Observer 实例对象。其中 def 函数其实就是 `Object.defineProperty` 函数的简单封装，之所以这里使用 def 函数定义 `__ob__` 属性是因为这样可以定义不可枚举的属性，这样后面遍历数据对象的时候就能够防止遍历到 `__ob__` 属性。

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
