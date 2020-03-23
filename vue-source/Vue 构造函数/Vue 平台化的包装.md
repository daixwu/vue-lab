# Vue 平台化的包装

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
