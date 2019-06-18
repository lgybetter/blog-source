---
title: Vue源码浅析之依赖收集
date: 2019-06-18 23:52:27
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

## 响应式对象Getter/Setter

前一篇文章讲了响应式对象的初始化，其中我们了解到 defineReactive 最终就是通过  Object.defineProperty 为数据对象定义 getter/setter, 我们看下官方文档对 getter/setter 的解释:

> getter/setter 对用户来说是不可见的，但是在内部它们让 Vue 追踪依赖，在属性被访问和修改时通知变化

我们可以知道 getter 其实就是为了追踪依赖, 对依赖进行收集。而 setter 则是通知变化, 让视图进行更新.

这次我们先了解一下 getter 如何进行依赖收集。

## 回顾 defineReactive 源码

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 实例化一个 Dep 对象
  const dep = new Dep()
  // 获取对象的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 收集当前的渲染watcher，作为订阅者，方便set的时候进行触发更新
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      ...省略
    }
  })
}
```

defineReactive 在函数开始执行时实例化了一个 Dep 对象, 然后在 getter 函数中判断 Dep.target 是否存在, 如果存在则会执行 dep.depend()。

这时候我们看到这里会有疑问, 首先 Dep 是什么? Dep.target 又是什么? dep.depend() 发挥了什么作用?

那接下来我们一个个来探索。

## Dep

我们先看下官方文档:

> 每个组件实例都有相应的 watcher 实例对象，它会在组件渲染的过程中把属性记录为依赖，之后当依赖项的 setter 被调用时，会通知 watcher 重新计算，从而致使它关联的组件得以更新。

其实 Dep 就是用来收集当前组件的渲染 watcher, 并记录为依赖。


```js
/**
 * Dep的作用是建立起数据和watcher之间的桥梁
 */
export default class Dep {
  // 静态属性
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = [] // 订阅当前数据变化的watcher
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      // 调用watcher.addDep
      Dep.target.addDep(this)
    }
  }

  notify () {
    ...省略
  }
}

// 全局的watcher，同一时间只能有一个watcher被计算
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  // 把父级的watcher保存到栈结构中
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  // 恢复父级的watcher
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}

```

在 src/core/observer/dep 中的代码定义了 Dep 这个类, 其构造函数初始化了一个 uid 和 subs 的空数组。接着我们看下 depend 这个函数, 其实现非常简单, 就是判断 Dep.target 是否存在, 存在则执行 Dep.target.addDep(this), 那这时候我们得要先知道 Dep.target 是什么东东。

Dep.target 是一个全局的渲染 watcher, 而且其保证同一时间只能有一个 watcher 被计算, 所以在 pushTarget 就是对把传入的渲染 watcher 赋值给 Dep.target, 并通过一个数组实现栈结构来保存当前的渲染 watcher, 为啥需要通过栈结构来保存这些渲染 watcher, 因为 Vue 在 mountComponent 每个组件初始化过程中会实例化一个渲染 Watcher, 组件的初始化是先子后父递归执行, 所以这里是为了保证父级的渲染 watcher 能够等子组件初始化完成后进行恢复, 恢复方法对应 popTarget。

那我们知道了

```js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {

  ...省略

  let updateComponent
  /* istanbul ignore if */
  ... 省略

  updateComponent = () => {
    vm._update(vm._render(), hydrating)
  }

  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  return vm
}
```

看到这里，可能会有疑问, Dep.target 是什么时候进行赋值为当前的渲染 watcher 的呢?


## Watcher

mountComponent 实例化了渲染 watcher, 我们看下  Watcher 的实现:

```js
export default class Watcher {
  ...省略
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    
    ...省略

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
      ...省略
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
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
      if (this.deep) {
        traverse(value)
      }
      popTarget()
      this.cleanupDeps()
    }
    return value
  }
  ...省略
}
```

Watcher 的构造函数初始化数据后，把传入的 expOrFn 回调函数进行函数转换, 并赋值给当前实例的 getter。最后执行了 this.get 函数, get 函数里面调用了 pushTarget(this) 把当前的渲染 watcher 实例赋值了 Dep.target, 看到这里就觉得 Vue 的设计模式是真的很巧妙。

depend 函数其实就是把当前的 Dep 实例传参执行当前渲染 watcher 的 addDep 方法。

```js
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      // 把当前的watcher添加到dep的subs中，也就是当前的watcher作为订阅者
      dep.addSub(this)
    }
  }
}
```

其功能就是把当前传入的 dep 实例 和 id 分布记录到 newDeps 和 newDepIds 的数组中, 然后再调用 dep 实例自身的 addSub, 把当前的渲染 watcher 实例添加到 dep 实例中的 subs 数组, 其实就是把当前的渲染 watcher 订阅到这个组件的数据对象持有的 dep 的 subs 中, 目的是为后续数据变化时候能通知到哪些 subs 做准备。

现在我们已经大概了解整个依赖收集的过程了, 这里还有一个问题, 就是我们一开始说的 getter 什么时候能够触发执行呢?

我们刚才已经看了 mountComponent 的源码, 其中 updateComponent 执行了 vm._render() 方法, 而 _render 会执行传入的 render 方法, 这时候会解析传入的数据, 从而触发对应数据对象的 getter 方法。 