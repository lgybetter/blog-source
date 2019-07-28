---
title: Vue源码解析之Watcher原理
date: 2019-07-28 21:11:31
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

对于用户定义的 Watche 属性官方描述如下:

> 一个对象，键是需要观察的表达式，值是对应回调函数。值也可以是方法名，或者包含选项的对象。Vue 实例将会在实例化时调用 $watch()，遍历 watch 对象的每一个属性。

接着看下文档对 $watch 的用法描述:

> 观察 Vue 实例变化的一个表达式或计算属性函数。回调函数得到的参数为新值和旧值。表达式只接受监督的键路径。对于更复杂的表达式，用一个函数取代。

## 初始化过程

挺久没更新文章了, 回顾一下 Vue 的 initState 函数

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

在 Vue 实例化执行 _init 的时候执行 initState, 这时候 initState 执行 initWatch 把 vm 实例和 opts 选项参数中的 watch 属性传递给 initWatch 方法。

```js
function initWatch (vm: Component, watch: Object) {
  for (const key in watch) {
    const handler = watch[key]
    if (Array.isArray(handler)) {
      for (let i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i])
      }
    } else {
      createWatcher(vm, key, handler)
    }
  }
}
```

initWatch 方法其实就是遍历用户定义的 watch 属性, 接着调用 createWatcher 进行 User Watcher 的创建

```js
function createWatcher (
  vm: Component,
  expOrFn: string | Function,
  handler: any,
  options?: Object
) {
  if (isPlainObject(handler)) {
    options = handler
    handler = handler.handler
  }
  if (typeof handler === 'string') {
    handler = vm[handler]
  }
  return vm.$watch(expOrFn, handler, options)
}
```

createWatcher 其实就是调用 vm 实例的 $watch 方法, $watch 方法我们刚才从官方文档的描述中可以知道, 就是观察 Vue 实例变化的一个表达式或计算属性函数。

$watch 是在执行 stateMixin 的时候定义在 Vue 的原型方法

```js
Vue.prototype.$watch = function (
  expOrFn: string | Function,
  cb: any,
  options?: Object
): Function {
  const vm: Component = this
  if (isPlainObject(cb)) {
    return createWatcher(vm, expOrFn, cb, options)
  }
  options = options || {}
  options.user = true
  const watcher = new Watcher(vm, expOrFn, cb, options)
  if (options.immediate) {
    try {
      cb.call(vm, watcher.value)
    } catch (error) {
      handleError(error, vm, `callback for immediate watcher "${watcher.expression}"`)
    }
  }
  return function unwatchFn () {
    watcher.teardown()
  }
}
```

从 $watch 的实现可以看出 

- options.user = true, 表明这是一个 user watcher, 本质上也是通过 new Watcher 来实例化得到一个 user watcher. 

- 当 options.immediate 为 true, 则会在实例化当前的 user watcher 后立即调用执行 cb 函数, 也就是用户定义的数据变化响应执行函数. 最后会返回一个取消观察函数，用来停止触发回调。

```
var unwatch = vm.$watch('a', cb)
// 之后取消观察
unwatch()
```

## 创建 User Watcher

new Watcher 的时候执行 Watcher 的构造函数:

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

这里 this.getter 为 parsePath 函数执行后返回的函数

```js
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```

返回的函数主要是用于对当前的 vm 上的数据属性进行获取, 从而触发对应监听数据响应式对象的 getter.


由于当前的 watcher 不是 computed watcher, 所以不需要延后计算, 于是 user watcher 主要调用 this.get 方法

```js
get () {
  debugger
  // 把当前的渲染watcher赋值给Dep.target
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
    debugger
    if (this.deep) {
      traverse(value)
    }
    popTarget()
    this.cleanupDeps()
  }
  return value
}
```

把当前的 user watcher 赋值给 Dep.target, 然后执行 this.getter, 触发定义监听的响应式数据依赖收集当前的 user watcher, 当 deep 为 true, 则会调用 traverse 对响应式数据进行深度遍历并触发对应的响应式数据的 getter, 从而达到深度监听子属性的功能. 

## 数据变化触发响应

当监听的响应式数据发生变化时, 对应的 setter 方法执行, 接着触发 被收集的 user watcher 的 update 方法

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

update 方法这里执行 queueWatcher 方法在下一个 tick 后对视图进行更新, 而在对视图进行更新过程中, flushSchedulerQueue 会触发 user watcher 的 run 方法, 

```js
run () {
  if (this.active) {
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      if (this.user) {
        try {
          this.cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        this.cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

最后执行了用户定义 watch 属性传入的 handler 函数, 这里为: this.cb.call(this.vm, value, oldValue).

到这里已经大概了解了整个 user watcher, 从定义, 初始化, 数据监听依赖收集, 更新触发响应函数执行的过程。
