# Vue 生命周期钩子的实现

在 initRender 函数执行完毕后，是这样一段代码：

```js
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

可以发现，`initInjections(vm)`、`initState(vm)` 以及 `initProvide(vm)` 被包裹在两个 callHook 函数调用的语句中。那么 callHook 函数的作用是什么呢？正如它的名字一样，callHook 函数的作用是调用生命周期钩子函数。根据引用关系可知 callHook 函数来自于 lifecycle.js 文件，打开该文件找到 callHook 函数如下：

```js
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

以上是 callHook 函数的全部代码，它接收两个参数：实例对象和要调用的生命周期钩子的名称。接下来我们就看看 callHook 是如何实现的。

大家可能注意到了 callHook 函数体的代码以 `pushTarget()` 开头，并以 `popTarget()` 结尾，这里我们暂且不讲这么做的目的，这其实是为了避免在某些生命周期钩子中使用 props 数据导致收集冗余的依赖，我们在 Vue 响应系统的章节会回过头来仔细给大家讲解。下面我们开始分析 callHook 函数的代码的中间部分，首先获取要调用的生命周期钩子：

```js
const handlers = vm.$options[hook]
```

比如 callHook(vm, created)，那么上面的代码就相当于：

```js
const handlers = vm.$options.created
```

在 Vue选项的合并 一节中我们讲过，对于生命周期钩子选项最终会被合并处理成一个数组，所以得到的 handlers 就是对应生命周期钩子的数组。接着执行的是这段代码：

```js
if (handlers) {
  for (let i = 0, j = handlers.length; i < j; i++) {
    try {
      handlers[i].call(vm)
    } catch (e) {
      handleError(e, vm, `${hook} hook`)
    }
  }
}
```

由于开发者在编写组件时未必会写生命周期钩子，所以获取到的 handlers 可能不存在，所以使用 if 语句进行判断，只有当 handlers 存在的时候才对 handlers 进行遍历，handlers 数组的元素就是生命周期钩子函数，所以直接执行即可：

```js
handlers[i].call(vm)
```

为了保证生命周期钩子函数内可以通过 this 访问实例对象，所以使用 `.call(vm)` 执行这些函数。另外由于生命周期钩子函数的函数体是开发者编写的，为了捕获可能出现的错误，使用 `try...catch` 语句块，并在 catch 语句块内使用 handleError 处理错误信息。其中 handleError 来自于 core/util/error.js 文件，大家可以在附录 core/util 目录下的工具方法全解 中查看关于 handleError 的讲解。

所以我们发现，对于生命周期钩子的调用，其实就是通过 `this.$options` 访问处理过的对应的生命周期钩子函数数组，遍历并执行它们。原理还是很简单的。

我们回过头来再看一下这段代码：

```js
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

现在大家应该知道，beforeCreate 以及 created 这两个生命周期钩子的调用时机了。其中 initState 包括了：initProps、initMethods、initData、initComputed 以及 initWatch。所以当 beforeCreate 钩子被调用时，所有与 props、methods、data、computed 以及 watch 相关的内容都不能使用，当然了 inject/provide 也是不可用的。

作为对立面，created 生命周期钩子则恰恰是等待 initInjections、initState 以及 initProvide 执行完毕之后才被调用，所以在 created 钩子中，是完全能够使用以上提到的内容的。但由于此时还没有任何挂载的操作，所以在 created 中是不能访问DOM的，即不能访问 `$el`。

最后我们注意到 callHook 函数的最后有这样一段代码：

```js
if (vm._hasHookEvent) {
  vm.$emit('hook:' + hook)
}
```

其中 vm._hasHookEvent 是在 initEvents 函数中定义的，它的作用是判断是否存在生命周期钩子的事件侦听器，初始化值为 false 代表没有，当组件检测到存在生命周期钩子的事件侦听器时，会将 vm._hasHookEvent 设置为 true。那么问题来了，什么叫做生命周期钩子的事件侦听器呢？大家可能不知道，其实 Vue 是可以这么玩儿的：

```js
<child
  @hook:beforeCreate="handleChildBeforeCreate"
  @hook:created="handleChildCreated"
  @hook:mounted="handleChildMounted"
  @hook:生命周期钩子
 />
 ```

如上代码可以使用 hook: 加 生命周期钩子名称 的方式来监听组件相应的生命周期事件。这是 Vue 官方文档上没有体现的，但你确实可以这么用，不过除非你对 Vue 非常了解，否则不建议使用。

正是为了实现这个功能，才有了这段代码：

```js
if (vm._hasHookEvent) {
  vm.$emit('hook:' + hook)
}
```

另外大家可能会疑惑，vm._hasHookEvent 是在什么时候被设置为 true 的呢？或者换句话说，Vue 是如何检测是否存在生命周期事件侦听器的呢？对于这个问题等我们在讲解 Vue 事件系统时自然会知道。
