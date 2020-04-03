# vue-lab

Vue(2.6.10) research laboratory

## Vue 构造函数

[Vue 构造函数的定义](vue-source/Vue%20构造函数/Vue%20构造函数的定义.md)

[Vue 构造函数原型扩展](vue-source/Vue%20构造函数/Vue%20构造函数原型扩展.md)

[Vue 构造函数全局API](vue-source/Vue%20构造函数/Vue%20构造函数全局API.md)

[Vue 平台化的包装](vue-source/Vue%20构造函数/Vue%20平台化的包装.md)

[With compiler](vue-source/Vue%20构造函数/With%20compiler.md)

## Vue 初始化

[Vue 初始化之开篇](vue-source/Vue%20初始化/Vue%20初始化之开篇.md)

[Vue 初始化之选项规范化](vue-source/Vue%20初始化/Vue%20初始化之选项规范化.md)

[Vue 初始化之选项合并](vue-source/Vue%20初始化/Vue%20初始化之选项合并.md)

[Vue 初始化之最终选项](vue-source/Vue%20初始化/Vue%20初始化之最终选项.md)

[Vue 初始化之渲染函数](vue-source/Vue%20初始化/Vue%20初始化之渲染函数.md)

[Vue 初始化之 initLifecycle](vue-source/Vue%20初始化/Vue%20初始化之initLifecycle.md)

[Vue 初始化之 initEvents](vue-source/Vue%20初始化/Vue%20初始化之%20initEvents.md)

[Vue 初始化之 initRender](vue-source/Vue%20初始化/Vue%20初始化之%20initRender.md)

[Vue 生命周期钩子的实现](vue-source/Vue%20初始化/Vue%20生命周期钩子的实现.md)

[Vue 初始化之 initState](vue-source/Vue%20初始化/Vue%20初始化之%20initState.md)

## Vue 响应式原理（Responsive）

### 响应式对象

[Vue 数据响应系统开篇](vue-source/Responsive/starter.md)

[initData 初始化 data](vue-source/Responsive/initData.md)

[数据响应系统的基本思路](vue-source/Responsive/mentality.md)

[observe 工厂函数](vue-source/Responsive/observe.md)

[Observer 构造函数](vue-source/Responsive/Observer.md)

[defineReactive 添加 getter 和 setter](vue-source/Responsive/defineReactive.md)

[get 函数如何收集依赖](vue-source/Responsive/getter.md)

[set 函数如何触发依赖](vue-source/Responsive/setter.md)

[响应式数据之数组的处理](vue-source/Responsive/arrayProcessing.md)

[`set($set)` 和 `delete($delete)` 的实现](vue-source/Responsive/set&delete.md)

### 依赖收集

[Dep 管理 Watcher](vue-source/Responsive/dep.md)

[Watcher 构造函数](vue-source/Responsive/Watcher.md)

[依赖收集的过程](vue-source/Responsive/depDepend.md)

### 派发更新

[派发更新的过程](vue-source/Responsive/depNotify.md)

[queueWatcher 异步更新队列](vue-source/Responsive/queueWatcher.md)

[nextTick 的实现](vue-source/Responsive/nextTick.md)

## 计算属性 VS 侦听属性

> 计算属性本质上是 computed watcher，而侦听属性本质上是 user watcher。就应用场景而言，计算属性适合用在模板渲染中，某个值是依赖了其它的响应式对象甚至是计算属性计算而来；而侦听属性适用于观测某个值的变化去完成一段复杂的业务逻辑。

[computed 计算属性的初始化](vue-source/Responsive/computedInit.md)

[computed 计算属性的实现](vue-source/Responsive/computed.md)

[watch 侦听属性的实现](vue-source/Responsive/watch.md)

[Watcher 构造函数对 options 的处理](vue-source/Responsive/WatcherOptions.md)

## Vue 组件更新
