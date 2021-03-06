# 数据响应系统的基本思路

接下来我们将重点讲解数据响应系统的实现，在具体到源码之前我们有必要了解一下数据响应系统实现的基本思路，这有助于我们更好地理解源码的目的，毕竟每一行代码都有它存在的意义。

在 Vue 中，我们可以使用 `$watch` 观测一个字段，当字段的值发生变化的时候执行指定的观察者，如下：

```js
const ins = new Vue({
  data: {
    a: 1
  }
})

ins.$watch('a', () => {
  console.log('修改了 a')
})
```

这样当我们试图修改 a 的值时：`ins.a = 2`，在控制台将会打印 '修改了 a'。现在我们将这个问题抽象一下，假设我们有数据对象 data，如下：

```js
const data = {
  a: 1
}
```

我们还有一个叫做 `$watch` 的函数：

```js
function $watch () {...}
```

`$watch` 函数接收两个参数，第一个参数是要观测的字段，第二个参数是当该字段的值发生变化后要执行的函数，如下：

```js
$watch('a', () => {
  console.log('修改了 a')
})
```

要实现这个功能，说复杂也复杂说简单也简单，复杂在于我们需要考虑的内容比较多，比如如何避免收集重复的依赖，如何深度观测，如何处理数组以及其他边界条件等等。简单在于如果不考虑那么多边界条件的话，要实现这样一个功能还是很容易的，这一小节我们就从简入手，致力于让大家思路清晰，至于各种复杂情况的处理我们会在真正讲解源码的部分一一为大家解答。

要实现上文的功能，我们面临的第一个问题是，如何才能知道属性被修改了(或被设置了)。这时候我们就要依赖 `Object.defineProperty` 函数，通过该函数为对象的每个属性设置一对 getter/setter 从而得知属性被读取和被设置，如下：

```js
Object.defineProperty(data, 'a', {
  set () {
    console.log('设置了属性 a')
  },
  get () {
    console.log('读取了属性 a')
  }
})
```

这样我们就实现了对属性 a 的设置和获取操作的拦截，有了它我们就可以大胆地思考一些事情，比如： 能不能在获取属性 a 的时候收集依赖，然后在设置属性 a 的时候触发之前收集的依赖呢？ 嗯，这是一个好思路，不过既然要收集依赖，我们起码需要一个“筐“，然后将所有收集到的依赖通通放到这个”筐”里，当属性被设置的时候将“筐”里所有的依赖都拿出来执行就可以了，落实到代码如下：

```js
// dep 数组就是我们所谓的“筐”
const dep = []
Object.defineProperty(data, 'a', {
  set () {
    // 当属性被设置的时候，将“筐”里的依赖都执行一次
    dep.forEach(fn => fn())
  },
  get () {
    // 当属性被获取的时候，把依赖放到“筐”里
    dep.push(fn)
  }
})
```

如上代码所示，我们定义了常量 dep，它是一个数组，这个数组就是我们所说的“筐”，当获取属性 a 的值时将触发 get 函数，在 get 函数中，我们将收集到的依赖放入“筐”内，当设置属性 a 的值时将触发 set 函数，在 set 函数内我们将“筐”里的依赖全部拿出来执行。

但是新的问题出现了，上面的代码中我们假设 fn 函数就是我们需要收集的依赖(观察者)，但 fn 从何而来呢？ 也就是说如何在获取属性 a 的值时收集依赖呢？ 为了解决这个问题我们需要思考一下我们现在都掌握了哪些条件，这个时候我们就需要在 `$watch` 函数中做文章了，我们知道 `$watch` 函数接收两个参数，第一个参数是一个字符串，即数据字段名,比如 'a'，第二个参数是依赖该字段的函数：

```js
$watch('a', () => {
  console.log('设置了 a')
})
```

重点在于 `$watch` 函数是知道当前正在观测的是哪一个字段的，所以一个思路是我们在 `$watch` 函数中读取该字段的值，从而触发字段的 get 函数，同时将依赖收集，如下代码：

```js
const data = {
  a: 1
}

const dep = []
Object.defineProperty(data, 'a', {
  set () {
    dep.forEach(fn => fn())
  },
  get () {
    // 此时 Target 变量中保存的就是依赖函数
    dep.push(Target)
  }
})

// Target 是全局变量
let Target = null
function $watch (exp, fn) {
  // 将 Target 的值设置为 fn
  Target = fn
  // 读取字段值，触发 get 函数
  data[exp]
}
```

上面的代码中，首先我们定义了全局变量 Target，然后在 `$watch` 中将 Target 的值设置为 fn 也就是依赖，接着读取字段的值 `data[exp]` 从而触发被设置的属性的 get 函数，在 get 函数中，由于此时 Target 变量就是我们要收集的依赖，所以将 Target 添加到 dep 数组。现在我们添加如下测试代码：

```js
$watch('a', () => {
  console.log('第一个依赖')
})
$watch('a', () => {
  console.log('第二个依赖')
})
```

此时当你尝试设置 `data.a = 3` 时，在控制台将分别打印字符串 '第一个依赖' 和 '第二个依赖'。我们仅仅用十几行代码就实现了这样一个最基本的功能，但其实现在的实现存在很多缺陷，比如目前的代码仅仅能够实现对字段 a 的观测，如果添加一个字段 b 呢？所以最起码我们应该使用一个循环将定义访问器属性的代码包裹起来，如下：

```js
const data = {
  a: 1,
  b: 1
}

for (const key in data) {
  const dep = []
  Object.defineProperty(data, key, {
    set () {
      dep.forEach(fn => fn())
    },
    get () {
      dep.push(Target)
    }
  })
}
```

这样我们就可以使用 `$watch` 函数观测任意一个 data 对象下的字段了，但是细心的同学可能早已发现上面代码的坑，即：

```js
console.log(data.a) // undefined
```

直接在控制台打印 data.a 输出的值为 undefined，这是因为 get 函数没有任何返回值，所以获取任何属性的值都将是 undefined，其实这个问题很好解决，如下：

```js
for (let key in data) {
  const dep = []
  let val = data[key] // 缓存字段原有的值
  Object.defineProperty(data, key, {
    set (newVal) {
      // 如果值没有变什么都不做
      if (newVal === val) return
      // 使用新值替换旧值
      val = newVal
      dep.forEach(fn => fn())
    },
    get () {
      dep.push(Target)
      return val  // 将该值返回
    }
  })
}
```

只需要在使用 `Object.defineProperty` 函数定义访问器属性之前缓存一下原来的值即 val，然后在 get 函数中将 val 返回即可，除此之外还要记得在 set 函数中使用新值(newVal)重写旧值(val)。

但这样就完美了吗？当然没有，这距离完美可以说还相差十万八千里，比如当数据 data 是嵌套的对象时，我们的程序只能检测到第一层对象的属性，如果数据对象如下：

```js
const data = {
  a: {
    b: 1
  }
}
```

对于以上对象结构，我们的程序只能把 `data.a` 字段转换成响应式属性，而 `data.a.b` 依然不是响应式属性，但是这个问题还是比较容易解决的，只需要递归定义即可：

```js
function walk (data) {
  for (let key in data) {
    const dep = []
    let val = data[key]
    // 如果 val 是对象，递归调用 walk 函数将其转为访问器属性
    const nativeString = Object.prototype.toString.call(val)
    if (nativeString === '[object Object]') {
      walk(val)
    }
    Object.defineProperty(data, key, {
      set (newVal) {
        if (newVal === val) return
        val = newVal
        dep.forEach(fn => fn())
      },
      get () {
        dep.push(Target)
        return val
      }
    })
  }
}

walk(data)
```

如上代码我们将定义访问器属性的逻辑放到了函数 walk 中，并增加了一段判断逻辑如果某个属性的值仍然是对象，则递归调用 walk 函数。这样我们就实现了深度定义访问器属性。

但是虽然经过上面的改造 `data.a.b` 已经是访问器属性了，但是如下代码依然不能正确执行：

```js
$watch('a.b', () => {
  console.log('修改了字段 a.b')
})
```

来看看目前 `$watch` 函数的代码：

```js
function $watch (exp, fn) {
  Target = fn
  // 读取字段值，触发 get 函数
  data[exp]
}
```

读取字段值的时候我们直接使用 `data[exp]`，如果按照 `$watch('a.b', fn)` 这样调用 `$watch` 函数，那么 `data[exp]` 等价于 `data['a.b']`，这显然是不正确的，正确的读取字段值的方式应该是 `data['a']['b']`。所以我们需要稍微做一点小小的改造：

```js
const data = {
  a: {
    b: 1
  }
}

function $watch (exp, fn) {
  Target = fn
  let pathArr,
      obj = data
  // 检查 exp 中是否包含 .
  if (/\./.test(exp)) {
    // 将字符串转为数组，例：'a.b' => ['a', 'b']
    pathArr = exp.split('.')
    // 使用循环读取到 data.a.b
    pathArr.forEach(p => {
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```

我们对 `$watch` 函数做了一些改造，首先检查要读取的字段是否包含 `.`，如果包含 `.` 说明读取嵌套对象的字段，这时候我们使用字符串的 `split('.')` 函数将字符串转为数组，所以如果访问的路径是 `a.b` 那么转换后的数组就是 `['a', 'b']`，然后使用一个循环从而读取到嵌套对象的属性值，不过需要注意的是读取到嵌套对象的属性值之后应该立即 return，不需要再执行后面的代码。

下面我们再进一步，我们思考一下 `$watch` 函数的原理是什么？其实 `$watch` 函数所做的事情就是想方设法地访问到你要观测的字段，从而触发该字段的 get 函数，进而收集依赖(观察者)。现在我们传递给 `$watch` 函数的第一个参数是一个字符串，代表要访问数据的哪一个字段属性，那么除了字符串之外可不可以是一个函数呢？假设我们有一个函数叫做 render，如下

```js
const data = {
  name: '霍春阳',
  age: 24
}

function render () {
  return document.write(`姓名：${data.name}; 年龄：${data.age}`)
}
```

可以看到 render 函数依赖了数据对象 data，那么 render 函数的执行是不是会触发 `data.name` 和 `data.age` 这两个字段的 get 拦截器呢？答案是肯定的，当然会！所以我们可以将 render 函数作为 `$watch` 函数的第一个参数：

```js
$watch(render, render)
```

为了能够保证 `$watch` 函数正常执行，我们需要对 `$watch` 函数做如下修改：

```js
function $watch (exp, fn) {
  Target = fn
  let pathArr,
      obj = data
  // 如果 exp 是函数，直接执行该函数
  if (typeof exp === 'function') {
    exp()
    return
  }
  if (/\./.test(exp)) {
    pathArr = exp.split('.')
    pathArr.forEach(p => {
      obj = obj[p]
    })
    return
  }
  data[exp]
}
```

在上面的代码中，我们检测了 exp 的类型，如果是函数则直接执行之，由于 render 函数的执行会触发数据字段的 get 拦截器，所以依赖会被收集。同时我们要注意传递给 `$watch` 函数的第二个参数：

```js
$watch(render, render)
```

第二个参数依然是 render 函数，也就是说当依赖发生变化时，会重新执行 render 函数，这样我们就实现了数据变化，并将变化自动应用到 DOM。其实这大概就是 Vue 的原理，但我们做的还远远不够，比如上面这句代码，第一个参数中 render 函数的执行使得我们能够收集依赖，当依赖变化时会重新执行第二个参数中的 render 函数，但不要忘了这又会触发一次数据字段的 get 拦截器，所以此时已经收集了两遍重复的依赖，那么我们是不是要想办法避免收集冗余的依赖呢？除此之外我们也没有对数组做处理，我们将这些问题留到后面，看看在 Vue 中它是如何处理的。

现在我们这个不严谨的实现暂时就到这里，意图在于让大家明白数据响应系统的整体思路，为接下来真正进入 Vue 源码做必要的铺垫。
