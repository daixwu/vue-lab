# vue-lab

Vue research laboratory

## Vue 构造函数（VueFunction）

[Vue 构造函数的定义](vue-source/VueFunction/starter.md)

[Vue 构造函数原型扩展](vue-source/VueFunction/prototype.md)

[Vue 构造函数全局API](vue-source/VueFunction/globalAPI.md)

[Vue 平台化的包装](vue-source/VueFunction/platform.md)

[With compiler](vue-source/VueFunction/compiler.md)

## Vue 初始化 (Init)

[Vue 初始化之开篇](vue-source/Init/starter.md)

[Vue 初始化之选项规范化](vue-source/Init/normalize.md)

[Vue 初始化之选项合并](vue-source/Init/merge.md)

[Vue 初始化之最终选项](vue-source/Init/options.md)

[Vue 初始化之渲染函数](vue-source/Init/initProxy.md)

[Vue 初始化之 initLifecycle](vue-source/Init/initLifecycle.md)

[Vue 初始化之 initEvents](vue-source/Init/initEvents.md)

[Vue 初始化之 initRender](vue-source/Init/initRender.md)

[Vue 生命周期钩子的实现](vue-source/Init/callHook.md)

[Vue 初始化之 initState](vue-source/Init/initState.md)

## Vue 数据驱动 （DataDriven）

[Vue 数据驱动](vue-source/DataDriven/starter.md)

[Vue $mount 挂载的开始](vue-source/DataDriven/mount.md)

[Vue mountComponent 执行组件挂载](vue-source/DataDriven/mountComponent.md)

[Vue _render 渲染虚拟 Node](vue-source/DataDriven/render.md)

[Vue createElement 创建 VNode](vue-source/DataDriven/createElement.md)

[Vue _update 渲染真实 DOM](vue-source/DataDriven/update.md)

## Vue 组件化（Component）

[Vue 组件化开篇](vue-source/Component/starter.md)

[Vue createComponent 创建组件 VNode](vue-source/Component/createComponent.md)

[Vue patch 渲染组件 VNode 为真实 DOM](vue-source/Component/patch.md)

[Vue 组件的合并配置](vue-source/Component/mergeOptions.md)

[Vue 生命周期钩子函数的执行](vue-source/Component/callHook.md)

[Vue 组件注册](vue-source/Component/register.md)

[Vue 异步组件](vue-source/Component/async.md)

## Vue 响应式原理（Reactive）

### 响应式对象

![原理图](vue-source/assets/img/reactive.png)

[Vue 数据响应系统开篇](vue-source/Reactive/starter.md)

[initData 初始化 data](vue-source/Reactive/initData.md)

[数据响应系统的基本思路](vue-source/Reactive/mentality.md)

[observe 工厂函数](vue-source/Reactive/observe.md)

[Observer 构造函数](vue-source/Reactive/Observer.md)

[defineReactive 添加 getter 和 setter](vue-source/Reactive/defineReactive.md)

[get 函数如何收集依赖](vue-source/Reactive/getter.md)

[set 函数如何触发依赖](vue-source/Reactive/setter.md)

[响应式数据之数组的处理](vue-source/Reactive/arrayProcessing.md)

[`set($set)` 和 `delete($delete)` 的实现](vue-source/Reactive/set&delete.md)

### 依赖收集

[Dep 管理 Watcher](vue-source/Reactive/dep.md)

[Watcher 构造函数](vue-source/Reactive/Watcher.md)

[依赖收集的过程](vue-source/Reactive/depDepend.md)

### 派发更新

[派发更新的过程](vue-source/Reactive/depNotify.md)

[queueWatcher 异步更新队列](vue-source/Reactive/queueWatcher.md)

[nextTick 的实现](vue-source/Reactive/nextTick.md)

### 计算属性 VS 侦听属性

> 计算属性本质上是 computed watcher，而侦听属性本质上是 user watcher。就应用场景而言，计算属性适合用在模板渲染中，某个值是依赖了其它的响应式对象甚至是计算属性计算而来；而侦听属性适用于观测某个值的变化去完成一段复杂的业务逻辑。

[computed 计算属性的初始化](vue-source/Reactive/computedInit.md)

[computed 计算属性的实现](vue-source/Reactive/computed.md)

[watch 侦听属性的实现](vue-source/Reactive/watch.md)

[Watcher 构造函数对 options 的处理](vue-source/Reactive/WatcherOptions.md)

### Vue 组件更新

[Vue 组件更新过程](vue-source/Reactive/componentUpdate.md)

### Vue Props

[Vue Props 实现原理](vue-source/Reactive/props.md)

## Vue 编译

[Vue 编译](vue-source/compile/starter.md)

[Vue 编译入口](vue-source/compile/entrance.md)

[Vue 编译之解析模板 生成 AST](vue-source/compile/parse.md)

[Vue 编译之优化 AST 树](vue-source/compile/optimize.md)

[Vue 编译之转换为可执行代码](vue-source/compile/codegen.md)

## 渲染器

[组件的本质](vue-source/ComponentEssence/essence.md)

[设计 VNode](vue-source/ComponentEssence/vnode.md)

[辅助创建 VNode 的 h 函数](vue-source/ComponentEssence/hFunction.md)

[渲染器之挂载](vue-source/ComponentEssence/render.md)

[渲染器之patch](vue-source/ComponentEssence/render-patch.md)

[渲染器的核心 Diff 算法](vue-source/ComponentEssence/render-diff.md)

[自定义渲染器和异步渲染](vue-source/ComponentEssence/render-advanced.md)

[Vue 知识点](vue-source/ComponentEssence/ExpandKnowledge.md)

## Vue 扩展

[Vue event 事件实现原理](vue-source/extend/event.md)

[Vue v-model 实现原理](vue-source/extend/v-model.md)

[Vue slot 实现原理](vue-source/extend/slot.md)

[Vue keep-alive 实现原理](vue-source/extend/keep-alive.md)

[Vue transition 实现原理](vue-source/extend/tansition.md)

[Vue transition-group 实现原理](vue-source/extend/tansition-group.md)

## 附录

[Vue 源码目录设计](vue-source/附录/Vue%20源码目录设计.md)

[Vue 构造函数整理-全局API](vue-source/附录/Vue%20构造函数整理-全局API.md)

[Vue 构造函数整理-原型](vue-source/附录/Vue%20构造函数整理-原型.md)\

[Vue 实例的设计](vue-source/附录/Vue%20实例的设计.md)

[core/util 目录下的工具方法全解](vue-source/附录/core-util%20目录下的工具方法全解.md)

[shared/util.js 文件工具方法全解](vue-source/附录/shared-util.js%20文件工具方法全解.md)

[platforms/web/util 目录下的工具方法全解](vue-source/附录/platforms-web-util%20目录下的工具方法全解.md)

## Vue-Router

[Vue-Router 路由注册](vue-source/vue-router/starter.md)

[VueRouter 对象](vue-source/vue-router/router.md)

[matcher 路径线路匹配](vue-source/vue-router/matcher.md)

[transitionTo 路径切换](vue-source/vue-router/transition-to.md)

## Vuex

[Vuex 核心思想](vue-source/vuex/starter.md)

[Vuex 初始化](vue-source/vuex/init.md)

[Vuex 操作store](vue-source/vuex/api.md)

[Vuex 插件](vue-source/vuex/plugin.md)