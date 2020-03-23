# Vue 初始化之最终选项

在前面我们写了一个很简单的例子，这个例子如下：

```js
var vm = new Vue({
    el: '#app',
    data: {
      message: 'Hello Vue!'
    }
})
```

我们以这个例子为线索开始了对 Vue 代码的讲解，我们知道了在实例化 Vue 实例的时候，`Vue.prototype._init` 方法被第一个执行，这个方法定义在 src/core/instance/init.js 文件中，在分析 _init 方法的时候我们遇到了下面的代码：

```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

正是因为上面的代码，使得我们花了大篇章来讲解其内部实现和运作，也就是 Vue选项规范化 和 Vue选项合并 这两节所介绍的内容。现在我们已经知道了 mergeOptions 函数是如何对父子选项进行合并处理的，也知道了它的作用。

我们打开 core/util/options.js 文件，找到 mergeOptions 函数，看其最后一句代码：

```js
return options
```

这说明 mergeOptions 函数最终将合并处理后的选项返回，并以该返回值作为 `vm.$options` 的值。`vm.$options` 在 Vue 的官方文档中是可以找到的，它作为实例属性暴露给开发者，那么现在你应该知道 `vm.$options` 到底是什么了。并且看文档的时候你应该更能够理解其作用，比如官方文档是这样介绍 `$options` 实例属性的：

> 用于当前 Vue 实例的初始化选项。需要在选项中包含自定义属性时会有用处

并且给了一个例子，如下：

```js
new Vue({
  customOption: 'foo',
  created: function () {
    console.log(this.$options.customOption) // => 'foo'
  }
})
```

上面的例子中，在创建 Vue 实例的时候传递了一个自定义选项：customOption，在之后的代码中我们可以通过 `this.$options.customOption` 进行访问。那原理其实就是使用 mergeOptions 函数对自定义选项进行合并处理，由于没有指定 customOption 选项的合并策略，所以将会使用默认的策略函数 defaultStrat。最终效果就是你初始化的值是什么，得到的就是什么。

另外，Vue 也提供了 `Vue.config.optionMergeStrategies` 全局配置，大家也可以在官方文档中找到，我们知道这个对象其实就是选项合并中的策略对象，所以我们可以通过他指定某一个选项的合并策略，常用于指定自定义选项的合并策略，比如我们给 customOption 选项指定一个合并策略，只需要在 `Vue.config.optionMergeStrategies` 上添加与选项同名的策略函数即可：

```js
Vue.config.optionMergeStrategies.customOption = function (parentVal, childVal) {
  return parentVal ? (parentVal + childVal) : childVal
}
```

如上代码中，我们添加了自定义选项 customOption 的合并策略，其策略为：如果没有 parentVal 则直接返回 childVal，否则返回两者的和。

所以如下代码：

```js
// 创建子类
const Sub = Vue.extend({
  customOption: 1
})
// 以子类创建实例
const v = new Sub({
  customOption: 2,
  created () {
    console.log(this.$options.customOption) // 3
  }
})
```

最终，在实例的 created 方法中将打印数字 3。上面的例子很简单，没有什么实际作用，但这为我们提供了自定义选项的机会，这其实是非常有用的。

现在我们需要回到正题上了，还是拿我们的例子，如下：

```js
var vm = new Vue({
    el: '#app',
    data: {
      message: 'Hello Vue!'
    }
})
```

这个时候 mergeOptions 函数将会把 `Vue.options` 作为 父选项，把我们传递的实例选项作为子选项进行合并，合并的结果我们可以通过打印 `$options` 属性得知。其实我们前面已经分析过了，el 选项将使用默认合并策略合并，最终的值就是字符串 `'#app'`，而 data 选项将变成一个函数，且这个函数的执行结果就是合并后的数据，即： `{message: 'Hello Vue!'}`，除此之外我们还能看到其他合并后的选项，其中 components、directives、filters 以及 _base 是存在于 `Vue.options` 中的，这些是我们所知道的，至于 render 和 staticRenderFns 这两个选项是在将模板编译成渲染函数时添加上去的，我们后面会遇到。另外 `_parentElm` 和 `_refElm` 这两个选项是在为虚拟DOM创建组件实例时添加的，我们后面也会讲到，这里大家不需要关心，免得失去重点。
