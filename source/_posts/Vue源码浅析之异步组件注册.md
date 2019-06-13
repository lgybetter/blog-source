---
title: Vue源码浅析之异步组件注册
date: 2019-06-13 22:44:09
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

## Vue的异步组件注册

Vue官方文档提供注册异步组件的方式有三种:

1. 工厂函数执行 resolve 回调
2. 工厂函数中返回Promise
3. 工厂函数返回一个配置化组件对象

## 工厂函数执行 resolve 回调

我们看下 Vue 官方文档提供的示例:

```js
Vue.component('async-webpack-example', function (resolve) {
  // 这个特殊的 `require` 语法将会告诉 webpack
  // 自动将你的构建代码切割成多个包, 这些包
  // 会通过 Ajax 请求加载
  require(['./my-async-component'], resolve)
})
```

简单说明一下, 这个示例调用 Vue 的静态方法 component 实现组件注册, 需要了解下 Vue.component 的大致实现

```js
// 此时type为component
Vue[type] = function (
  id: string,
  definition: Function | Object
): Function | Object | void {
  if (!definition) {
    return this.options[type + 's'][id]
  } else {
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && type === 'component') {
      validateComponentName(id)
    }
    // 是否为对象
    if (type === 'component' && isPlainObject(definition)) {
      definition.name = definition.name || id
      definition = this.options._base.extend(definition)
    }
    if (type === 'directive' && typeof definition === 'function') {
      definition = { bind: definition, update: definition }
    }
    // 记录当前Vue的全局components, filters, directives对应的声明映射
    this.options[type + 's'][id] = definition
    return definition
  }
}
```
先判断传入的 definition 也就是我们的工厂函数, 是否为对象, 都说是工厂函数了, 那肯定不为对象, 于是这里不调用 this.options._base.extend(definition) 来获取组件的构造函数, 而是直接把当前的 definition(工厂函数) 保存到 this.options.components 的 'async-webpack-example' 属性值中, 并返回definition。

接下来发生什么事情呢？
其实刚才我们只是调用了 Vue.component 注册一个异步组件, 但是我们最终是通过 new Vue 实例来实现页面的渲染。这里大致浏览一下渲染的过程：

Run:
- new Vue执行构造函数 
- 构造函数 执行 this._init, 在 initMixin 执行的时候定义 Vue.prototype._init
- $mount执行, 在 web/runtime/index.js 中已经进行定义 Vue.prototype.$mount
- 执行 core/instance/lifecycle.js 中的 mountComponent
- 实例化渲染Watcher, 并传入 updateComponent(通过 Watcher 实例对象的 getter 触发vm._update, 而至于怎么触发先忽略, 会另外讲解)
- vm._update 触发 vm._render(renderMixin 时定义在 Vue.prototype._render) 执行
- 在 vm.$options 中获取 render 函数并执行, 使得传入的 vm.$createElement(在 initRender 中定义在vm中)执行, vm.$createElement也就是平时书写的 h => h(App)这个h函数。
- vm.$createElement = createElement
- createComponent 通过 resolveAsset 查询当前组件是否正常注册

所以我们现在以及进入到 createComponent 这个函数了, 看下这里异步组件具体的实现逻辑：

```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component, // vm实例
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {

  // 在init初始化的时候赋值Vue
  const baseCtor = context.$options._base

  // Ctor当前为异步组件的工厂函数, 所以此步骤不执行
  if (isObject(Ctor)) {
    // 获取构造器, 对于非全局注册的组件使用
    Ctor = baseCtor.extend(Ctor)
  }

  // async component
  let asyncFactory
  // 如果Ctro.cid为undefined, 则说明h会是异步组件注册
  // 原因是没有调用过 Vue.extend 进行组件构造函数转换获取
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    // 解析异步组件
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    // Ctor为undefined则直接创建并返回异步组件的占位符组件Vnode
    if (Ctor === undefined) {
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  ...此处省略不分析的代码

  // 安装组件的钩子
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children }, // 组件对象componentOptions
    asyncFactory
  )

  return vnode
}
```

从源码我们可以看出, 异步组件不执行组件构造器的转换获取, 而是执行 resolveAsyncComponent 来获取返回的组件对象。由于改过程是异步请求组件, 所以我们看下 resolveAsyncComponent 的实现  

```js
// 定义在render.js中的全局变量, 用于记录当前正在渲染的vm实例
import { currentRenderingInstance } from 'core/instance/render'

export function resolveAsyncComponent (
  factory: Function,
  baseCtor: Class<Component>
): Class<Component> | void {
  // 高级异步组件使用
  if (isTrue(factory.error) && isDef(factory.errorComp)) {...先省略}

  if (isDef(factory.resolved)) {
    return factory.resolved
  }
  // 获取当前正在渲染的vm实例
  const owner = currentRenderingInstance
  if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
    // already pending
    factory.owners.push(owner)
  }

  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {...省略}

  // 执行该逻辑
  if (owner && !isDef(factory.owners)) {
    const owners = factory.owners = [owner]
    // 用于标记是否
    let sync = true

    ...省略
    const forceRender = (renderCompleted: boolean) => { ...省略 }
    
    // once让被once包装的任何函数的其中一个只执行一次
    const resolve = once((res: Object | Class<Component>) => {
      // cache resolved
      factory.resolved = ensureCtor(res, baseCtor)
      // invoke callbacks only if this is not a synchronous resolve
      // (async resolves are shimmed as synchronous during SSR)
      if (!sync) {
        forceRender(true)
      } else {
        owners.length = 0
      }
    })

    const reject = once(reason => { ...省略 })

    // 执行工厂函数, 比如webpack获取异步组件资源
    const res = factory(resolve, reject)
    
    ...省略
    
    sync = false
    // return in case resolved synchronously
    return factory.loading
      ? factory.loadingComp
      : factory.resolved
  }
}
```

resolveAsyncComponent 传入异步组件工厂函数和 baseCtor(也就是Vue.extend), 先获取当前渲染的vm实例接着标记sync为true, 表示当前为执行同步代码阶段, 定义 resolve 和 reject 函数(忽略不分析), 此时我们可以发现 resolve 和 reject 都被 once 函数所封装, 目的是让被 once 包装的任何函数的其中一个只执行一次, 保证 resolve 和 reject 两者只能择一并只执行一次。OK, 接着来到 factory 的执行, 其实就是执行官方示例中传入的工厂函数, 这时候发起异步组件的请求。同步代码继续执行, sync置位false, 表示当前的同步代码执行完毕, 然后返回undefined

这里可能会问怎么会返回undefined, 因为我们传入的工厂函数没有loading属性, 然后当前的 factory 也没有 resolved 属性。

接着回到 createComponent 的代码中:

```js
if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    // 解析异步组件
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor)
    // Ctor为undefined则直接创建并返回异步组件的占位符组件Vnode
    if (Ctor === undefined) {
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
```
因为刚才说 resolveAsyncComponent 执行返回了undefined, 所以执行 createAsyncPlaceholder 创建注释vnode

这里可能还会问为什么要创建一个注释vnode, 提前揭晓答案: 

因为先要返回一个占位的 vnode, 等待异步请求加载后执行 forceUpdate 重新渲染, 然后这个节点会被更新渲染成组件的节点。

那继续,  刚才答案说了, 当异步组件请求完成后, 则执行 resolve 并传入对应的异步组件, 这时候 factory.resolved 被赋值为 ensureCtor 执行的返回结果, 就是一个组件构造器, 然后这时候 sync 为 false, 所以执行 forceRender, 而 forceRender 其实就是调用 vm.$forceUpdate 实现如下:

```js
Vue.prototype.$forceUpdate = function () {
  const vm: Component = this
  if (vm._watcher) {
    vm._watcher.update()
  }
}
```

$forceUpdate 执行渲染 watcher 的 update 方法，于是我们又会执行 createComponent 的方法, 执行 resolveAsyncComponent, 这时候 factory.resolved 已经定义过了，于是直接返回 factory.resolved 的组件构造器。 于是就执行 createComponent 的后续组件的渲染和 patch 逻辑了。组件渲染和 patch 这里先不展开。

于是整个异步组件的流程就结束了。

## 工厂函数中返回Promise

啊!!! 凌晨2.42了，先睡觉，明天继续补充。