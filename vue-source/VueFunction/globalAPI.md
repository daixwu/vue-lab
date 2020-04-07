# Vue 构造函数全局API

core/index.js 文件的主要作用是，为 Vue 添加全局的API，也就是静态的方法和属性。这里是对 Vue 构造函数全局API(静态属性和方法)的整理，便于看源码时查看方法的对应位置。

## initGlobalAPI(Vue)

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

## initUse(Vue)

**Path**：global-api/use.js

```js
Vue.use = function (plugin: Function | Object) {}
```

## initMixin(Vue)

**Path**：global-api/mixin.js

```js
Vue.mixin = function (mixin: Object) {}
```

## initExtend(Vue)

**Path**：global-api/extend.js

```js
Vue.cid = 0
Vue.extend = function (extendOptions: Object): Function {}
```

## initAssetRegisters(Vue)

**Path**：global-api/assets.js

```js
Vue.component =
Vue.directive =
Vue.filter = function (
  id: string,
  definition: Function | Object
): Function | Object | void {}
```

另见 附录 [Vue 构造函数整理-全局API](../附录/Vue%20构造函数整理-全局API.md)
