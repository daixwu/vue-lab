# Vue 构造函数原型扩展

core/instance/index.js 为Vue 的出生文件，主要作用是定义 Vue 构造函数，并对其原型添加属性和方法，即实例属性和实例方法。这里是对 Vue 构造函数原型的整理，便于看源码时查看方法的对应位置。

## initMixin(Vue)

**Path**：src/core/instance/init.js

```js
Vue.prototype._init = function (options?: Object) {}
```

## stateMixin(Vue)

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

## eventsMixin(Vue)

**Path**：src/core/instance/events.js

```js
Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {}
Vue.prototype.$once = function (event: string, fn: Function): Component {}
Vue.prototype.$off = function (event?: string | Array<string>, fn?: Function): Component {}
Vue.prototype.$emit = function (event: string): Component {}
```

## lifecycleMixin(Vue)

**Path**：src/core/instance/lifecycle.js

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {}
Vue.prototype.$forceUpdate = function () {}
Vue.prototype.$destroy = function () {}
```

## renderMixin(Vue)

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

## 其他

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

另见 附录 [Vue 构造函数整理-原型](../附录/Vue%20构造函数整理-原型.md)
