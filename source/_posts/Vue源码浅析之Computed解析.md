---
title: Vue源码浅析之Computed解析
date: 2019-07-03 23:48:36
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

## 什么是计算属性

官方文档描述如下:

模板内的表达式非常便利，但是设计它们的初衷是用于简单运算的。在模板中放入太多的逻辑会让模板过重且难以维护。例如：

```html
<div id="example">
  {{ message.split('').reverse().join('') }}
</div>
```

在这个地方，模板不再是简单的声明式逻辑。你必须看一段时间才能意识到，这里是想要显示变量 message 的翻转字符串。当你想要在模板中多次引用此处的翻转字符串时，就会更加难以处理。

所以，对于任何复杂逻辑，你都应当使用计算属性。

个人理解如下:

computed 主要的用途是把一个或多个变量进行计算处理, 得到计算后的结果, 其能够监听当前所依赖的变量的变更, 并重新计算变更的结果, 进而触发视图进行渲染更新。

这次就来探索一下 computed 的原理实现。

## Computed 初始化

在实例化 Vue 的过程中, initState 函数实现如下:

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    // 响应式处理
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

这里判断我们的 opts 可选参数是否传入 computed 属性, 若有则执行 initComputed 进行初始化。


```js
const computedWatcherOptions = { lazy: true }

function initComputed (vm: Component, computed: Object) {
  const watchers = vm._computedWatchers = Object.create(null)
  // computed properties are just getters during SSR
  const isSSR = isServerRendering()

  for (const key in computed) {
    const userDef = computed[key]
    const getter = typeof userDef === 'function' ? userDef : userDef.get
    if (process.env.NODE_ENV !== 'production' && getter == null) {
      warn(
        `Getter is missing for computed property "${key}".`,
        vm
      )
    }

    if (!isSSR) {
      // create internal watcher for the computed property.
      watchers[key] = new Watcher(
        vm,
        getter || noop,
        noop,
        computedWatcherOptions
      )
    }

    // component-defined computed properties are already defined on the
    // component prototype. We only need to define computed properties defined
    // at instantiation here.
    if (!(key in vm)) {
      defineComputed(vm, key, userDef)
    } else if (process.env.NODE_ENV !== 'production') {
      if (key in vm.$data) {
        warn(`The computed property "${key}" is already defined in data.`, vm)
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(`The computed property "${key}" is already defined as a prop.`, vm)
      }
    }
  }
}
```

initComputed 先定义了一个 watchers 用于保存当前 computed 对象各个计算属性所以对应的 computed watcher。这个 computed watcher 用途我们后续会介绍。把用户定义的 computed 属性对应的 getter 函数进行获取。

接着, 就根据计算属性的 key 值实例化一个相对应的 computed watcher, 这个 watcher 为啥叫 computed watcher, 因为我们看到 Watcher 的构造函数传参有 computedWatcherOptions = { lazy: true }。表示这是一个 computed watcher, 用于计算属性监听所使用, 和之前说过的渲染 watcher 有所区别。computed watcher 的构造函数还会传入当前的 vm 实例, computed 属性中对应的 getter。

接下来我们需要关注一下英文注释：

component-defined computed properties are already defined on the
component prototype. We only need to define computed properties defined
at instantiation here.

这句话其实就是介绍, 对于子组件来说, 其声明的 computed 属性已经被定义在当前组件的原型中, 这时候 key in vm 其实会为 true。

为什么已经定义在子组件中呢?

我们可以了解到, 在对子组件在初始化的过程中, 会通过 Vue.extend 来获取组件的构造器

```js
// 继承组件定义
const Sub = function VueComponent (options) {
  // 执行Vue.prototype._init方法
  this._init(options)
}
// 继承父组件Vue的原型
Sub.prototype = Object.create(Super.prototype)
// 拦截重置构造函数
Sub.prototype.constructor = Sub

...省略

if (Sub.options.props) {
  initProps(Sub)
}
if (Sub.options.computed) {
  initComputed(Sub)
}
```

Vue.extend 的方法其实在进行 initState 的 initComputed 调用前, 已经在获取子组件的构造器的时候就调用了 initComputed 对 computed 进行挂载初始化了。


这时候我们回过头看下 new Watcher 的实现。


## 实例化 computed watcher

```js
constructor (
  vm: Component,
  expOrFn: string | Function,
  cb: Function,
  options?: ?Object,
  isRenderWatcher?: boolean
) {
  this.vm = vm
  if (isRenderWatcher) {
    vm._watcher = this
  }
  vm._watchers.push(this)
  // options
  if (options) {
    this.deep = !!options.deep
    this.user = !!options.user
    // lazy 为 true
    this.lazy = !!options.lazy
    this.sync = !!options.sync
    this.before = options.before
  } else {
    this.deep = this.user = this.lazy = this.sync = false
  }
  this.cb = cb
  this.id = ++uid // uid for batching
  this.active = true
  this.dirty = this.lazy // for lazy watchers
  this.deps = []
  this.newDeps = []
  this.depIds = new Set()
  this.newDepIds = new Set()
  this.expression = process.env.NODE_ENV !== 'production'
    ? expOrFn.toString()
    : ''
  // parse expression for getter
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn
  } else {
    this.getter = parsePath(expOrFn)
    if (!this.getter) {
      this.getter = noop
      process.env.NODE_ENV !== 'production' && warn(
        `Failed watching path: "${expOrFn}" ` +
        'Watcher only accepts simple dot-delimited paths. ' +
        'For full control, use a function instead.',
        vm
      )
    }
  }
  this.value = this.lazy
    ? undefined
    : this.get()
}
```



我们继续看下 defineComputed 的实现:

```js
export function defineComputed (
  target: any, // vm
  key: string,
  userDef: Object | Function // computed getter/setter
) {
  const shouldCache = !isServerRendering()
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = shouldCache
      ? createComputedGetter(key)
      : createGetterInvoker(userDef)
    sharedPropertyDefinition.set = noop
  } else {
    sharedPropertyDefinition.get = userDef.get
      ? shouldCache && userDef.cache !== false
        ? createComputedGetter(key)
        : createGetterInvoker(userDef.get)
      : noop
    sharedPropertyDefinition.set = userDef.set || noop
  }
  if (process.env.NODE_ENV !== 'production' &&
      sharedPropertyDefinition.set === noop) {
    sharedPropertyDefinition.set = function () {
      warn(
        `Computed property "${key}" was assigned to but it has no setter.`,
        this
      )
    }
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```

defineComputed 其实通过 Object.defineProperty 为当前的 vm 实例定义对应计算属性的 getter/setter。这里主要看下 createComputedGetter 的实现：

```js
function createComputedGetter (key) {
  return function computedGetter () {
    const watcher = this._computedWatchers && this._computedWatchers[key]
    if (watcher) {
      if (watcher.dirty) {
        watcher.evaluate()
      }
      if (Dep.target) {
        watcher.depend()
      }
      return watcher.value
    }
  }
}
```
createComputedGetter 返回一个函数, 这个函数就是每个 computed 属性所对应的 getter 函数, 当 computed 被访问时, 会触发改函数的执行, 该函数其实就是触发当前 computed watcher 的 evaluate 或者 depend 方法执行, 一开始 watcher.dirty 为 true, 原因是 dirty 的初始值其实就是我们传入的 computedWatcherOptions = { lazy: true } 的 lazy。于是执行 watcher.evaluate()

```js
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

处于渲染过程中, 也就是 Dep.target 不为空, 则该 computed watcher 会被当前的渲染 watcher 对应的 Dep 收集到subs中。也就是当前的渲染 watcher 订阅了该 computed 数据的变化。

evaluate 函数执行传入 computed watcher 的 getter 函数, 也就是用户定义的 computed getter 的函数。把计算好的值赋值给 computed watcher 实例的 value。

由于用户定义在 computed getter 函数中的数据也是响应式对象, 在计算过程中会触发其 getter 函数, 这时候会其对应的 dep 会被添加到当前的 computed watcher中作为依赖进行收集。

