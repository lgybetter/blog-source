---
title: Vue源码浅析之nextTick
date: 2019-06-26 22:19:46
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

## nextTicke 的作用

官方文档描述如下:

> 在下次 DOM 更新循环结束之后执行延迟回调。在修改数据之后立即使用这个方法，获取更新后的 DOM。

从这个描述我们可以知道，nextTick 方法用于在视图渲染更新结束后进行回调方法执行, 目的是保证我们在回调方法中操作的 DOM 是已经完成视图更新渲染。

## 回顾响应式更新派发

我们已经知道当数据发送变更, 会触发响应式数据对象的 setter 方法, 接着订阅该数据更新的渲染 watcher 执行 queueWatcher, queueWatcher 的实现就是使用 nextTick(flushSchedulerQueue) 来实现视图的异步更新。

## nextTick 的实现

nextTick 的源码定义在 core/util/next-tick.js 中


我们先看一下这个模块定义的全局变量

```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

// 是否使用微任务标识
export let isUsingMicroTask = false
// 收集回调函数
const callbacks = []
// Promise 的判断标识
let pending = false

```
这里需要关注一下 callbacks 这个数组, 它的作用是为了收集每次调用 nextTick 方法对应的回调函数, 保证每个回调函数有序执行。 


我们先看下 nextTick 的定义
```js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
nextTick 实现很简单，如果是通过回调函数的形式来调用, 则把传入的回调函数 push 到 callbacks 这个数组中, 被 push 的回调函数会被封装为一个匿名函数, 这个匿名对每个回调函数进行 try/catch 的异常捕获, 目的是为了保证当某个回调函数发生异常时, 不影响 js 的进一步执行。如果是使用 Promise 的方式进行 nextTick 进行调用, 则会 new 一个 Promsie 对象并把 resolve 赋值给 _resolve 以便链式调用。

一开始 pending 为 false, 所以会执行 timerFunc。

这里我们看下 timerFunc 是怎么实现的

```js
let timerFunc

// The nextTick behavior leverages the microtask queue, which can be accessed
// via either native Promise.then or MutationObserver.
// MutationObserver has wider support, however it is seriously bugged in
// UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
// completely stops working after triggering a few times... so, if native
// Promise is available, we will use it:
/* istanbul ignore next, $flow-disable-line */
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    // In problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  // Use MutationObserver where native Promise is not available,
  // e.g. PhantomJS, iOS7, Android 4.4
  // (#6466 MutationObserver is unreliable in IE11)
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  // Fallback to setImmediate.
  // Techinically it leverages the (macro) task queue,
  // but it is still a better choice than setTimeout.
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

```

timerFunc 会根据不同浏览器的原生异步任务支持情况来选择对应的异步任务创建类型, 这里简单列举这里的异步任务类型:

- Promise 微任务
- MutationObserver 微任务
- setImmediate 宏任务
- setTimeout 宏任务

timerFunc 本质就是创建一个异步任务来执行 flushCallbacks 这个函数。目的是为了让传入 nextTick 的回调函数是异步执行的。

```js
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```
flushCallbacks 其实就是遍历 callbacks 数组里面的回调函数并依次执行。

对于用户调用 Vue.nextTick 暴露的 api 接口传入对应的回调函数, 其执行总会是在视图完成渲染更新后才进行, 因为当数据发生变更, 触发渲染 watcher 执行 queueWatcher, 这时候 vue 内部调用 nextTick 先保证 flushSchedulerQueue 这个方法是先被 push 到 callbacks 这个回调数组。

举个例子

```js
<template>
  <div>
    <p ref="dom">{{msg}}</p>
    <button @click="handleChange">change</button>
  </div>
</template>

<script>
export default {
  data () {
    return {
      msg: 'vue'
    }
  },
  methods: {
    handleChange () {
      this.$nextTick(() => {
        console.log(this.refs.dom.msg, 'pre')
      })
      this.msg = 'vue-nextTick'
      console.log(this.refs.dom.msg, 'common');
      this.$nextTick(() => {
        console.log(this.refs.dom.msg, 'after')
      })
    }
  }
}
</script>
```

这时候的输出结果如下：
```
vue common
vue pre
vue-nextTick after
```

这里我们简单来看下对应的函数执行顺序, 我们把第一个 nextTick 的回调函数取名 preTick, 第二个叫 afterTick

同步代码执行: 

callbacks.push(preTick)
callbacks.push(flushSchedulerQueue) // 修改 msg 的值所触发
console.log(this.refs.dom.msg, 'common') // 输出 vue common
callbacks.push(afterTick)

同步代码执行完成后, 执行 flushCallbacks 遍历执行回调函数:

preTick // 输出 vue pre
flushSchedulerQueue // 视图进行数据更新渲染, this.refs.dom.msg 为 vue-nextTick
afterTick // 输出 vue-nextTick after

到这里已经完成对 nextTick 的分析，其实 nextTick 的实现并不复杂, 如果已经深入了解浏览器 js 的执行机制可能会更容易理解。
