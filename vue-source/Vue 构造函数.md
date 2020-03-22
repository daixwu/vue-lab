# Vue 构造函数

Vue 实际上就是一个用 Function 实现的类，来看一下源码，在 src/core/instance/index.js 中。

```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  console.log('options: ', options);
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```

## 原型上挂载的属性和方法

core/instance/index.js 为Vue 的出生文件，主要作用是定义 Vue 构造函数，并对其原型添加属性和方法，即实例属性和实例方法。这里是对 Vue 构造函数原型的整理，便于看源码时查看方法的对应位置。

### initMixin(Vue)

**Path**：src/core/instance/init.js

```js
Vue.prototype._init = function (options?: Object) {}
```

### stateMixin(Vue)

**Path**：src/core/instance/state.js

```js
Vue.prototype.$data
Vue.prototype.$props
Vue.prototype.$set = set
Vue.prototype.$delete = del
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {}
```

### eventsMixin(Vue)

**Path**：src/core/instance/events.js

```js
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {}
Vue.prototype.$once = function (event: string, fn: Function): Component {}
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {}
Vue.prototype.$emit = function (event: string): Component {}
```

### lifecycleMixin(Vue)

**Path**：src/core/instance/lifecycle.js

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {}
Vue.prototype.$forceUpdate = function () {}
Vue.prototype.$destroy = function () {}
```

### renderMixin(Vue)

**Path**：src/core/instance/render.js

```js
// installRenderHelpers 函数中
Vue.prototype._o = markOnce
Vue.prototype._n = toNumber
Vue.prototype._s = toString
Vue.prototype._l = renderList
Vue.prototype._t = renderSlot
Vue.prototype._q = looseEqual
Vue.prototype._i = looseIndexOf
Vue.prototype._m = renderStatic
Vue.prototype._f = resolveFilter
Vue.prototype._k = checkKeyCodes
Vue.prototype._b = bindObjectProps
Vue.prototype._v = createTextVNode
Vue.prototype._e = createEmptyVNode
Vue.prototype._u = resolveScopedSlots
Vue.prototype._g = bindObjectListeners
Vue.prototype._d = bindDynamicKeys
Vue.prototype._p = prependModifier

Vue.prototype.$nextTick = function (fn: Function) {}
Vue.prototype._render = function (): VNode {}
```

### 其他

**Path**：core/index.js

```js
Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})
```

**Path**：platforms/web/runtime/index.js

```js
Vue.prototype.__patch__ = inBrowser ? patch : noop
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

**Path**：在入口文件 entry-runtime-with-compiler.js 中重写了 Vue.prototype.$mount 方法

```js
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // ... 函数体
}
```

## 静态属性和方法（全局API）

core/index.js 文件的主要作用是，为 Vue 添加全局的API，也就是静态的方法和属性。这里是对 Vue 构造函数全局API(静态属性和方法)的整理，便于看源码时查看方法的对应位置。

### initGlobalAPI(Vue)

**Path**: core/index.js

```js
Vue.config
Vue.util = {
  warn,
  extend,
  mergeOptions,
  defineReactive
}
Vue.set = set
Vue.delete = del
Vue.nextTick = nextTick

// 2.6 explicit observable API
Vue.observable = <T>(obj: T): T => {
  observe(obj)
  return obj
}

Vue.options = {
  components: {
    // KeepAlive 是通过 extend 函数将 builtInComponents 的属性混合到 Vue.options.components 中，其中 builtInComponents 来自于 core/components/index.js 文件
    KeepAlive
    // Transition 和 TransitionGroup 组件在 platforms/web/runtime/index.js 文件中被添加
    // Transition,
    // TransitionGroup
  },
  directives: Object.create(null),
    // 在 platforms/web/runtime/index.js 文件中，为 directives 添加了两个平台化的指令 model 和 show
    // directives:{
    //  model,
    //  show
    // },
  filters: Object.create(null),
  _base: Vue
}
```

### initUse(Vue)

**Path**：global-api/use.js

```js
Vue.use = function (plugin: Function | Object) {}
```

### initMixin(Vue)

**Path**：global-api/mixin.js

```js
Vue.mixin = function (mixin: Object) {}
```

### initExtend(Vue)

**Path**：global-api/extend.js

```js
Vue.cid = 0
Vue.extend = function (extendOptions: Object): Function {}
```

### initAssetRegisters(Vue)

**Path**：global-api/assets.js

```js
Vue.component =
Vue.directive =
Vue.filter = function (
  id: string,
  definition: Function | Object
): Function | Object | void {}
```

## Vue 平台化的包装

前面 core/instance/index.js 文件以及 core/index.js 文件它们都在 core 目录下，而 core 目录存放的是与平台无关的代码，所以无论是 core/instance/index.js 文件还是 core/index.js 文件，它们都在包装核心的 Vue，且这些包装是与平台无关的。但是，Vue 是一个 Multi-platform 的项目（web和weex），不同平台可能会内置不同的组件、指令，或者一些平台特有的功能等等，那么这就需要对 Vue 根据不同的平台进行平台化地包装，这就是接下来我们要看的文件，这个文件也出现在我们寻找 Vue 构造函数的路线上，它就是：platforms/web/runtime/index.js 文件。该文件的作用是对 Vue 进行平台化地包装：

- 设置平台化的 Vue.config。
- 在 Vue.options 上混合了两个指令(directives)，分别是 model 和 show。
- 在 Vue.options 上混合了两个组件(components)，分别是 Transition 和 TransitionGroup。
- 在 Vue.prototype 上添加了两个方法：__patch__ 和 $mount。

在经过这个文件之后，Vue.options 以及 Vue.config 和 Vue.prototype 都有所变化。

```js
// Vue.options
Vue.options = {
  components: {
    KeepAlive,
    Transition,
    TransitionGroup
  },
  directives: {
    model,
    show
  },
  filters: Object.create(null),
  _base: Vue
}

// Vue.config
// install platform specific utils
Vue.config.mustUseProp = mustUseProp
Vue.config.isReservedTag = isReservedTag
Vue.config.isReservedAttr = isReservedAttr
Vue.config.getTagNamespace = getTagNamespace
Vue.config.isUnknownElement = isUnknownElement

// Vue.prototype
// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```

## with compiler

在看完 runtime/index.js 文件之后，其实 运行时 版本的 Vue 构造函数就已经“成型了”。我们可以打开 entry-runtime.js 这个入口文件，这个文件只有两行代码：

```js
import Vue from './runtime/index'

export default Vue
```

可以发现，运行时 版的入口文件，导出的 Vue 就到 ./runtime/index.js 文件为止。然而我们所选择的并不仅仅是运行时版，而是完整版的 Vue，入口文件是 entry-runtime-with-compiler.js，我们知道完整版和运行时版的区别就在于 compiler，所以其实在我们看这个文件的代码之前也能够知道这个文件的作用：就是在运行时版的基础上添加 compiler，对没错，这个文件就是干这个的，接下来我们就看看它是怎么做的，打开 entry-runtime-with-compiler.js 文件：

```js
// ... 其他 import 语句

// 导入 运行时 的 Vue
import Vue from './runtime/index'

// ... 其他 import 语句

// 从 ./compiler/index.js 文件导入 compileToFunctions
import { compileToFunctions } from './compiler/index'

// 根据 id 获取元素的 innerHTML
const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

// 使用 mount 变量缓存 Vue.prototype.$mount 方法
const mount = Vue.prototype.$mount

// 重写 Vue.prototype.$mount 方法
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)

  /* istanbul ignore if */
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) {
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  return mount.call(this, el, hydrating)
}

// 获取元素的 outerHTML
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}

// 在 Vue 上添加一个全局API `Vue.compile` 其值为上面导入进来的 compileToFunctions
Vue.compile = compileToFunctions

// 导出 Vue
export default Vue
```

上面代码是简化过的，但是保留了所有重要的部分，该文件的开始是一堆 import 语句，其中重要的两句 import 语句就是上面代码中出现的那两句，一句是导入运行时的 Vue，一句是从 ./compiler/index.js 文件导入 compileToFunctions，并且在倒数第二句代码将其添加到 Vue.compile 上。

然后定义了一个函数 idToTemplate，这个函数的作用是：获取拥有指定 id 属性的元素的 innerHTML。

之后缓存了运行时版 Vue 的 Vue.prototype.$mount 方法，并且进行了重写。

接下来又定义了 getOuterHTML 函数，用来获取一个元素的 outerHTML。

这个文件运行下来，对 Vue 的影响有两个，第一个影响是它重写了 `Vue.prototype.$mount` 方法；第二个影响是添加了 Vue.compile 全局API，目前我们只需要获取这些信息就足够了，我们把这些影响同样更新到 附录 对应的文件中，也都可以查看的到。
