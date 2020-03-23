# With compiler

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

之后缓存了运行时版 Vue 的 `Vue.prototype.$mount` 方法，并且进行了重写。

接下来又定义了 getOuterHTML 函数，用来获取一个元素的 outerHTML。

这个文件运行下来，对 Vue 的影响有两个，第一个影响是它重写了 `Vue.prototype.$mount` 方法；第二个影响是添加了 `Vue.compile` 全局API，目前我们只需要获取这些信息就足够了。
