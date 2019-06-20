---
title: Vue源码浅析之更新派发
date: 2019-06-20 22:39:45
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

getter 就是为了追踪依赖, 对依赖进行收集。而 setter 则是通知变化, 让视图进行更新。

之前已经探索了 vue 在 触发响应式对象的 getter 时候, 通过把组件对应的渲染 watcher 进行依赖收集, 这些 watcher 最终会被收集到 dep.subs 的数组中。

## 回顾 defineReactive 源码

那当数据发生变更，这时候会触发响应式对象的 setter, 重新看下 defineReactive 的源码实现:

```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()
  
  ...省略

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () { ...省略 },
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
  })
}
```

可以看到, 响应式对象的 setter 其实先获取当前的数据变更, 判断数据是否发生变更, 然后会对变更的数据进行 observe 响应式对象初始化, 当然前提是 newVal 是一个非 Vnode 类型的对象。接着这里调用当前响应式对象初始化过程中已经实例化的 dep 的 notify 方法。

而这时候 dep 的 subs 数组其实已经把当前订阅这个数据对象变更的渲染 watcher 收集起来了。

那我们看下 notify 的实现


## 更新派发过程

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
这里其实就是对 subs 的订阅者 watcher 进行遍历, 并调用 watcher 的 update 方法。 通俗一点来说, 就是通知 subs 里面的 watcher 去执行 update 方法, 从而进行视图更新。

我们看下 Watcher 实现的 update 方法:

```js
/**
  * Subscriber interface.
  * Will be called when a dependency changes.
  */
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

run 方法下面会详细分析。

这里比较关键的是先关注 queueWatcher 这个函数

```js
const queue: Array<Watcher> = []
const activatedChildren: Array<Component> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0

/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

这里通过全局的 has 对象来保证每次同一个 Watcher 只能被添加一次到 queue 这个队列。全局的 index 用于记录当前队列遍历的下标。一开始 flushing 和 waiting 都为 false, 所以执行 queue.push(watcher), 接着通过 nextTick 来异步执行 flushSchedulerQueue。至于 nextTick 怎么实现以后再单独探索, 反正就先认为是用来实现异步更新的机制。

当同步的代码执行完成了, 就会执行 nextTicke 的回调方法 flushSchedulerQueue

## flushSchedulerQueue 分析

这里我们重点分析一下这个 flushSchedulerQueue 方法

```js
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

### flashing 的标记为被置为 true

### 对全局保存 watcher 的这个 queue 队列进行排序

排序的注释翻译如下:

- 组件的创建从父到子进行, 所以组件的 watcher 也是先进行父级的创建, 再到子级的创建, 对应 watcher 的执行也需要保证从父到子
- 用户自定义的 watcher 优先于组件渲染 watcher 的创建
- 如果一个组件在父组件的 watcher 执行期间被 destroyed, 则这个子级对应的 watcher 会被跳过执行

### 执行在 new Watcher 时候传入的 before 函数进行 hook 的执行

```js
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted && !vm._isDestroyed) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```

### watcher.run 进行视图更新

```js
/**
  * Scheduler job interface.
  * Will be called by the scheduler.
  */
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
```

run 方法其实调用 this.get, 而 get 这里有个 pushTarget 会导致 Dep.target 变更为当前的 watcher, 然后执行 this.getter => updateComponet => _update => render 触发组件重新渲染。

### 用户自定义 watcher

run 方法执行同时会判断如果当前的 watcher 是属于用户自定义的 watcher, 则这时候会执行用户定义的回调方法, 而这个用户定义的回调方法很可能会再次进行数据更新修改, 从而再次触发 queueWatcher 的执行。

```js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      nextTick(flushSchedulerQueue)
    }
  }
}
```

而这时候由于 flashing 为true, 所以 queueWatcher 会进入 else 的逻辑, 然后从后往前找，找到第一个待插入 watcher 的 id 比当前队列中 watcher 的 id 大的位置。把 watcher 按照 id的插入到队列中，因此 queue 的长度发送了变化。


### 恢复状态

resetSchedulerState 方法是用于把一些逻辑流程执行的变量进行重置为初始值。

```js
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

## 小结一下

组件的数据更新触发视图更新，通过 getter 把订阅当前响应式对象数据的渲染 watcher 进行收集, 并保存到 dep 的 subs 中, 当数据进行更新触发 setter 的时候先是遍历 subs 执行每个 watcher 的 update 方法, 把当前的所有 watcher 保存到一个队列的结构中,  在同步的代码执行完成后, 通过 nextTick 来异步执行 flushSchedulerQueue 进行队列的遍历, 然后执行对应的 watcher 的 run 方法进行组件 patch 更新以及对应的响应对象的依赖重新收集。