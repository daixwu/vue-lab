# Vue 初始化之 initEvents

在 initLifecycle 函数之后，执行的就是 initEvents，它来自于 core/instance/events.js 文件，打开该文件找到 initEvents 方法，其内容很简短，如下：

```js
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```

首先在 vm 实例对象上添加两个实例属性 `_events` 和 `_hasHookEvent`，其中 `_events` 被初始化为一个空对象，`_hasHookEvent` 的初始值为 false。之后将执行这段代码：

```js
// init parent attached events
const listeners = vm.$options._parentListeners
if (listeners) {
  updateComponentListeners(vm, listeners)
}
```

其中 `_parentListeners` 在创建子组件实例的时候才会有这个参数选项，所以现在我们不做深入讨论，后面自然有机会。
