# Vue 初始化之渲染函数

ok，现在我们已经足够了解 `vm.$options` 这个属性了，它才是用来做一系列初始化工作的最终选项，那么接下来我们就继续看 `_init` 方法中的代码，继续了解 Vue 的初始化工作。

`_init` 方法中，在经过 mergeOptions 合并处理选项之后，要执行的是下面这段代码：

```js
/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
    initProxy(vm)
} else {
    vm._renderProxy = vm
}
```

这段代码是一个判断分支，如果是非生产环境的话则执行 initProxy(vm) 函数，如果在生产环境则直接在实例上添加 `_renderProxy` 实例属性，该属性的值就是当前实例。

现在有一个问题需要大家思考一下，目前我们还没有看 initProxy 函数的具体内容，那么你能猜到 initProxy 函数的主要作用是什么吗？我可以直接告诉大家，这个函数的主要作用其实就是在实例对象 vm 上添加 `_renderProxy` 属性。为什么呢？因为生产环境和非生产环境下要保持功能一致。在上面的代码中生产环境下直接执行这句：

```js
vm._renderProxy = vm
```

那么可想而知，在非生产环境下也应该执行这句代码，但实际上却调用了 initProxy 函数，所以 initProxy 函数的作用之一必然也是在实例对象 vm 上添加 `_renderProxy` 属性，那么接下来我们就看看 initProxy 的内容，验证一下我们的判断，打开 core/instance/proxy.js 文件：

```js
/* not type checking this file because flow doesn't play well with Proxy */

import config from 'core/config'
import { warn, makeMap, isNative } from '../util/index'

// 声明 initProxy 变量
let initProxy

if (process.env.NODE_ENV !== 'production') {
  // ... 其他代码
  
  // 在这里初始化 initProxy
  initProxy = function initProxy (vm) {
    if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
        ? getHandler
        : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
    } else {
      vm._renderProxy = vm
    }
  }
}

// 导出
export { initProxy }
```

上面的代码是简化后的，可以发现在文件的开头声明了 initProxy 变量，但并未初始化，所以目前 initProxy 还是 undefined，随后，在文件的结尾将 initProxy 导出，那么 initProxy 到底是什么呢？实际上变量 initProxy 的赋值是在 if 语句块内进行的，这个 if 语句块进行环境判断，如果是非生产环境的话，那么才会对 initProxy 变量赋值，也就是说在生产环境下我们导出的 initProxy 实际上就是 undefined。只有在非生产环境下导出的 initProxy 才会有值，其值就是这个函数：

```js
initProxy = function initProxy (vm) {
  if (hasProxy) {
      // determine which proxy handler to use
      const options = vm.$options
      const handlers = options.render && options.render._withStripped
      ? getHandler
      : hasHandler
      vm._renderProxy = new Proxy(vm, handlers)
  } else {
      vm._renderProxy = vm
  }
}
```

这个函数接收一个参数，实际就是 Vue 实例对象，我们先从宏观角度来看一下这个函数的作用是什么，可以发现，这个函数由 `if...else` 语句块组成，但无论走 if 还是 else，其最终的效果都是在 vm 对象上添加了 `_renderProxy` 属性，这就验证了我们之前的猜想。如果 hasProxy 为真则走 if 分支，对于 hasProxy 顾名思义，这是用来判断宿主环境是否支持 js 原生的 Proxy 特性的，如果发现 Proxy 存在，则执行：

```js
vm._renderProxy = new Proxy(vm, handlers)
```

如果不存在，那么和生产环境一样，直接赋值就可以了：

```js
vm._renderProxy = vm
```

所以我们发现 initProxy 的作用实际上就是对实例对象 vm 的代理，通过原生的 [Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 实现。

另外 hasProxy 变量的定义也在当前文件中，代码如下：

```js
const hasProxy =
    typeof Proxy !== 'undefined' &&
    Proxy.toString().match(/native code/)
```

上面代码的作用是判断当前宿主环境是否支持原生 Proxy，相信大家都能看得懂，所以就不做过多解释，接下来我们就看看它是如何做代理的，并且有什么作用。

查看 initProxy 函数的 if 语句块，内容如下：

```js
initProxy = function initProxy (vm) {
    if (hasProxy) {
        // determine which proxy handler to use
        // options 就是 vm.$options 的引用
        const options = vm.$options
        // handlers 可能是 getHandler 也可能是 hasHandler
        const handlers = options.render && options.render._withStripped
            ? getHandler
            : hasHandler
        // 代理 vm 对象
        vm._renderProxy = new Proxy(vm, handlers)
    } else {
        // ...
    }
}
```

可以发现，如果 Proxy 存在，那么将会使用 Proxy 对 vm 做一层代理，代理对象赋值给 `vm._renderProxy`，所以今后对 `vm._renderProxy` 的访问，如果有代理那么就会被拦截。代理对象配置参数是 handlers，可以发现 handlers 既可能是 getHandler 又可能是 hasHandler，至于到底使用哪个，是由判断条件决定的：

```js
options.render && options.render._withStripped
```

如果上面的条件为真，则使用 getHandler，否则使用 hasHandler，判断条件要求 `options.render` 和 `options.render._withStripped` 必须都为真才行，我现在明确告诉大家 `options.render._withStripped` 这个属性只在测试代码中出现过，所以一般情况下这个条件都会为假，也就是使用 hasHandler 作为代理配置。

hasHandler 常量就定义在当前文件，如下：

```js
const hasHandler = {
  has (target, key) {
    // has 常量是真实经过 in 运算符得来的结果
    const has = key in target
    // 如果 key 在 allowedGlobals 之内，或者 key 是以下划线 _ 开头的字符串，则为真
    const isAllowed = allowedGlobals(key) ||
      (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data))
    // 如果 has 和 isAllowed 都为假，使用 warnNonPresent 函数打印错误
    if (!has && !isAllowed) {
      if (key in target.$data) warnReservedPrefix(target, key)
      else warnNonPresent(target, key)
    }
    return has || !isAllowed
  }
}
```

这里我假设大家都对 Proxy 的使用已经没有任何问题了，我们知道 has 可以拦截以下操作：

- 属性查询: `foo in proxy`
- 继承属性查询: `foo in Object.create(proxy)`
- with 检查: `with(proxy) { (foo); }`
- `Reflect.has()`

其中关键点就在 has 可以拦截 with 语句块里对变量的访问，后面我们会讲到。

has 函数内出现了两个函数，分别是 allowedGlobals 以及 warnNonPresent，这两个函数也是定义在当前文件中，首先我们看一下 allowedGlobals：

```js
const allowedGlobals = makeMap(
  'Infinity,undefined,NaN,isFinite,isNaN,' +
  'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
  'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
  'require' // for Webpack/Browserify
)
```

可以看到 allowedGlobals 实际上是通过 makeMap 生成的函数，所以 allowedGlobals 函数的作用是判断给定的 key 是否出现在上面字符串中定义的关键字中的。这些关键字都是在 js 中可以全局访问的。

warnNonPresent 函数如下：

```js
const warnNonPresent = (target, key) => {
  warn(
    `Property or method "${key}" is not defined on the instance but ` +
    'referenced during render. Make sure that this property is reactive, ' +
    'either in the data option, or for class-based components, by ' +
    'initializing the property. ' +
    'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
    target
  )
}
```

这个函数就是通过 warn 打印一段警告信息，警告信息提示你“在渲染的时候引用了 key，但是在实例对象上并没有定义 key 这个属性或方法”。其实我们很容易就可以看到这个信息，比如下面的代码：

```js
const vm = new Vue({
  el: '#app',
  template: '<div>{{a}}</div>',
  data: {
    message: 'Hello Vue!'
  }
})
```

大家注意，在模板中我们使用 a，但是在 data 属性中并没有定义这个属性，这个时候我们就能够得到以上报错信息。

大家可能比较疑惑的是为什么会这样，其实我们后面讲到渲染函数的时候你自然就知道了，不过现在大家可以先看一下，打开 core/instance/render.js 文件，找到 Vue.prototype._render 方法，里面有这样的代码：

```js
vnode = render.call(vm._renderProxy, vm.$createElement)
```

可以发现，调用 render 函数的时候，使用 call 方法指定了函数的执行环境为 `vm._renderProxy`，渲染函数长成什么样呢？还是以上面的例子为例，我们可以通过打印 `vm.$options.render` 查看，所以它长成这样：

```js
vm.$options.render = function () {
  // render 函数的 this 指向实例的 _renderProxy
  with(this){
    return _c('div',{attrs:{"id":"app"}},[_v(_s(message))])} // 在这里访问 message，相当于访问 vm._renderProxy.message
  })
}
```

从上面的代码可以发现，显然函数使用 with 语句块指定了内部代码的执行环境为 this，由于 render 函数调用的时候使用 call 指定了其 this 指向为 `vm._renderProxy`，所以 with 语句块内代码的执行环境就是 `vm._renderProxy`，所以在 with 语句块内访问 message 就相当于访问 `vm._renderProxy` 的 message 属性，前面我们提到过 with 语句块内访问变量将会被 Proxy 的 has 代理所拦截，所以自然就执行了 has 函数内的代码。最终通过 warnNonPresent 打印警告信息给我们，所以这个代理的作用就是为了在开发阶段给我们一个友好而准确的提示。

我们理解了 hasHandler，但是还有一个 getHandler，这个代理将会在判断条件：

```js
options.render && options.render._withStripped
```

为真的情况下被使用，那这个条件什么时候成立呢？其实 `_withStripped` 只在 test/unit/features/instance/render-proxy.spec.js 文件中出现过，该文件有这样一段代码：

```js
it('should warn missing property in render fns without `with`', () => {
    const render = function (h) {
        // 这里访问了 a
        return h('div', [this.a])
    }
    // 在这里将 render._withStripped 设置为 true
    render._withStripped = true
    new Vue({
        render
    }).$mount()
    // 应该得到警告
    expect(`Property or method "a" is not defined`).toHaveBeenWarned()
})
```

这个时候就会触发 getHandler 设置的 get 拦截，getHandler 代码如下：

```js
const getHandler = {
  get (target, key) {
    if (typeof key === 'string' && !(key in target)) {
        warnNonPresent(target, key)
    }
    return target[key]
  }
}
```

其最终实现的效果无非就是检测到访问的属性不存在就给你一个警告。但我们也提到了，只有当 render 函数的 `_withStripped` 为真的时候，才会给出警告，但是 `render._withStripped` 又只有写测试的时候出现过，也就是说需要我们手动设置其为 true 才会得到提示，否则是得不到的，比如：

```js
const render = function (h) {
  return h('div', [this.a])
}

var vm = new Vue({
  el: '#app',
  render,
  data: {
      test: 1
  }
})
```

上面的代码由于 render 函数是我们手动书写的，所以 render 函数并不会被包裹在 with 语句块内，当然也就触发不了 has 拦截，但是由于 `render._withStripped` 也未定义，所以也不会被 get 拦截，那这个时候我们虽然访问了不存在的 this.a，但是却得不到警告，想要得到警告我们需要手动设置 `render._withStripped` 为 true：

```js
const render = function (h) {
    return h('div', [this.a])
}
render._withStripped = true

var vm = new Vue({
    el: '#app',
    render,
    data: {
        test: 1
    }
})
```

为什么会这么设计呢？因为在使用 webpack 配合 vue-loader 的环境中， vue-loader 会借助 `vuejs@component-compiler-utils` 将 template 编译为不使用 with 语句包裹的遵循严格模式的 JavaScript，并为编译后的 render 方法设置 `render._withStripped = true`。在不使用 with 语句的 render 方法中，模板内的变量都是通过属性访问操作 `vm['a']` 或 `vm.a` 的形式访问的，从前文中我们了解到 Proxy 的 has 无法拦截属性访问操作，所以这里需要使用 Proxy 中可以拦截到属性访问的 get，同时也省去了 has 中的全局变量检查(全局变量的访问不会被 get 拦截)。

现在，我们基本知道了 initProxy 的目的，就是设置渲染函数的作用域代理，其目的是为我们提供更好的提示信息。但是我们忽略了一些细节没有讲清楚，回到下面这段代码：

```js
// has 变量是真实经过 in 运算符得来的结果
const has = key in target
// 如果 key 在 allowedGlobals 之内，或者 key 是以下划线 _ 开头的字符串，则为真
const isAllowed = allowedGlobals(key) || (typeof key === 'string' && key.charAt(0) === '_')
// 如果 has 和 isAllowed 都为假，使用 warnNonPresent 函数打印错误
if (!has && !isAllowed) {
    warnNonPresent(target, key)
}
```

上面这段代码中的 if 语句的判断条件是 (`!has` && `!isAllowed`)，其中 `!has` 我们可以理解为你访问了一个没有定义在实例对象上(或原型链上)的属性，所以这个时候提示错误信息是合理，但是即便 `!has` 成立也不一定要提示错误信息，因为必须要满足 `!isAllowed`，也就是说当你访问了一个虽然不在实例对象上(或原型链上)的属性，但如果你访问的是全局对象那么也是被允许的。这样我们就可以在模板中使用全局对象了，如：

```html
<template>
  {{Number(b) + 2}}
</template>
```

其中 Number 为全局对象，如果去掉 `!isAllowed` 这个判断条件，那么上面模板的写法将会得到警告信息。除了允许使用全局对象之外，还允许以 `_` 开头的属性，这么做是由于渲染函数中会包含很多以 `_` 开头的内部方法，如之前我们查看渲染函数时遇到的 `_c`、`_v` 等等。

最后对于 proxy.js 文件内的代码，还有一段是我们没有讲过的，就是下面这段：

```js
if (hasProxy) {
  // isBuiltInModifier 函数用来检测是否是内置的修饰符
  const isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta,exact')
  // 为 config.keyCodes 设置 set 代理，防止内置修饰符被覆盖
  config.keyCodes = new Proxy(config.keyCodes, {
    set (target, key, value) {
      if (isBuiltInModifier(key)) {
          warn(`Avoid overwriting built-in modifier in config.keyCodes: .${key}`)
          return false
      } else {
          target[key] = value
          return true
      }
    }
  })
}
```

上面的代码首先检测宿主环境是否支持 Proxy，如果支持的话才会执行里面的代码，内部的代码首先使用 makeMap 函数生成一个 isBuiltInModifier 函数，该函数用来检测给定的值是否是内置的事件修饰符，我们知道在 Vue 中我们可以使用事件修饰符很方便地做一些工作，比如阻止默认事件等。

然后为 `config.keyCodes` 设置了 set 代理，其目的是防止开发者在自定义键位别名的时候，覆盖了内置的修饰符，比如：

```js
Vue.config.keyCodes.shift = 16
```

由于 shift 是内置的修饰符，所以上面这句代码将会得到警告。
