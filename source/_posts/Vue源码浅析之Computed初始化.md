---
title: Vue源码浅析之Computed初始化
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

## Computed 初始化 Get 过程

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


这时候我们看下 new Watcher 的实现。


### 实例化 computed watcher

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

此时 this.lazy 为 true, 然后会把用户定义的 getter 传入的并保存在当前 computed watcher 的 this.getter中。


### defineComputed 实现

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


defineComputed 的实现如下:

```js
export function defineComputed (
  target: any, // Sub原型
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
defineComputed 其实通过 Object.defineProperty 为当前的 Sub 原型定义对应计算属性的 getter/setter, 在原型上定义主要是为了给多个组件能够共享调用 createComputedGetter 这个 getter 的优化点。

### createComputedGetter 实现

接下来主要看下 createComputedGetter 的实现：

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

```js
evaluate () {
  this.value = this.get()
  this.dirty = false
}
```

createComputedGetter 返回一个函数, 这个函数 computedGetter 就是每个 computed 属性所对应的 getter 函数, 当 computed 的属性被访问时, 比如在渲染过程中属性被访问, 会触发 computedGetter 的执行, 该函数其实就是触发当前 computed watcher 的 evaluate 或者 depend 方法执行, 一开始 watcher.dirty 为 true, 原因是 dirty 的初始值其实就是我们传入的 computedWatcherOptions = { lazy: true } 的 lazy。于是执行 watcher.evaluate() 这时候会调用 this.get() 进行求值。

```js
get () {
  // 把当前的watcher, 渲染watcher 或者 computed watcher 赋值给Dep.target
  pushTarget(this)
  let value
  const vm = this.vm
  try {
    // 执行在mountComponet中传入的updateComponent
    value = this.getter.call(vm, vm)
  } catch (e) {
    if (this.user) {
      handleError(e, vm, `getter for watcher "${this.expression}"`)
    } else {
      throw e
    }
  } finally {
    // "touch" every property so they are all tracked as
    // dependencies for deep watching
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

我们之前已经了解，pushTarget(this) 其实就是把当前的 computed watcher 赋值给 Dep.target, 接着执行 this.getter 计算 computed 属性对应的数值。

这里需要特别注意一点, this.getter 为用户给 computed 属性定义的 getter 方法, 此方法执行会触发这个方法所依赖的响应式数据的 getter 的执行。

这时候需要特别关注，当前的 Dep.target 为此时 computed 属性对应的 computed watcher, 而触发响应式数据的 getter 的执行则会使得响应式数据对象对应的 dep 收集 Dep.target, 也就是此时的 computed watcher。当前的 computed watcher 会被 push 到响应式对象的 dep.subs 的数组中, 这里其实就是当前的 computed watcher 订阅了所依赖的响应式数据的变化。

最后执行完成后调用 popTarget 把原始的 watcher, 比如渲染 watcher 恢复重新赋值给 Dep.target。

this.get() 执行完成后则把 this.dirty 置为 false。

回到 computedGetter, 如果处于渲染过程中, 也就是 Dep.target 不为空, 则继续执行了 watcher.depend 

```js
depend () {
  let i = this.deps.length
  while (i--) {
    this.deps[i].depend()
  }
}
```
当前的 computed watcher 已经订阅了所依赖的响应式数据的变化, 于是 computed watcher 的 deps 为所有依赖的响应式数据对应的 Dep。于是执行 dep.depend, 把当前的 Dep.target 也就是当前的渲染 watcher, push 到每个响应式数据对象所对应的 dep.subs 中。当前的渲染 watcher 订阅了 computed watcher 所依赖的数据的变化, 用于依赖数据更新触发视图 update 渲染。完成后, 然后返回通过 evaluate 计算得到的值。


这里整个 computed 的 get 求值就已经完成了。

## Computed 变更 Set 过程

当 computed 属性所依赖的响应式数据发生变更后, 则响应式数据会触发其对应的 setter 执行

```js
set: function reactiveSetter (newVal) {
  const value = getter ? getter.call(obj) : val
  /* eslint-disable no-self-compare */
  if (newVal === value || (newVal !== newVal && value !== value)) {
    return
  }
  /* eslint-enable no-self-compare */
  if (process.env.NODE_ENV !== 'production' && customSetter) {
    customSetter()
  }
  // #7981: for accessor properties without setter
  if (getter && !setter) return
  if (setter) {
    setter.call(obj, newVal)
  } else {
    val = newVal
  }
  childOb = !shallow && observe(newVal)
  dep.notify()
}
```
set 的执行我们之前有了解过, 这里主要看两个点
- newVal === value || (newVal !== newVal && value !== value)
- dep.notify()

第一个是 set 的触发会对比当前更新的值是否发生变更, 如果没有变更则直接 return 不往下执行触发视图更新。

第二个则是执行了当前响应式数据对象对应的 dep 的 notify 方法

```js
notify () {
  // stabilize the subscriber list first
  const subs = this.subs.slice()
  if (process.env.NODE_ENV !== 'production' && !config.async) {
    // subs aren't sorted in scheduler if not running async
    // we need to sort them now to make sure they fire in correct
    // order
    subs.sort((a, b) => a.id - b.id)
  }
  for (let i = 0, l = subs.length; i < l; i++) {
    subs[i].update()
  }
}
```

因为在 get 过程中, 计算属性对应的 computed watcher 订阅了响应式数据的 dep, 这时候会通知 computed watcher 进行 update, 对应的渲染 watcher 也订阅了响应式数据的 dep, 也会执行 update。

computed watcher 的 update 会先执行, 然后执行渲染 watcher 的 update

```js
update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```

computed watcher 执行 update 把其对应的 dirty 属性置为 true, 接着执行渲染 watcher 的update, 渲染 watcher 的 update 执行了 queueWatcher 让视图在下一个 tick 进行更新。

于是视图进行 flushSchedulerQueue 更新, 渲染 watcher 的更新会触发 computed 属性的 getter, 所以就再次执行 computedGetter 的执行, computed watcher 的 evaluate 和 depend 再次执行, 重新依赖收集所依赖的响应式数据, 接着返回数据更新后的新的 computed 值, 接着视图触发渲染更新。


这里已经完成了整个 computed 属性的初始化的解析以及当 computed 属性所依赖的数据发生变更后的重新计算依赖和数值的过程。