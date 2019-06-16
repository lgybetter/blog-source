---
title: Vue源码浅析之响应式对象初始化
date: 2019-06-17 01:48:11
tags:
  - Vue
categories:
  - 前端
---

![Vue](/images/vue-logo.png)

## 响应式对象简介

这里引用官方文档的解释：

> 当你把一个普通的 JavaScript 对象传给 Vue 实例的 data 选项，Vue 将遍历此对象所有的属性，并使用 Object.defineProperty 把这些属性全部转为 getter/setter。

根据上面的解释, 那这次我们来探索一下这个响应式对象的初始化过程, 也就是 Vue 如何把实例的 data 选项进行遍历并如何使用 Object.defineProperty 把属性转为 getter/setter。

## 初始化过程

示例：

```js
<template>
  <div>{{user.name}}</div>
</template>
<script>
export default {
  data () {
    return {
      user: {
        name: ''
      }
    }
  }
}
</script>
```

假设我们用这个简单的示例来定义一个Vue的单文件组件, 这个组件在初始化的过程中会进行组件的注册创建, 通过 Vue.extend 获取组件的构造器, 最后把定义好的 options 传参并实例化 vm 对象, 这时候构造器的执行就会调用 _init 方法, 在 _init 方法中执行了 initState 方法, initState 方法执行了 initProps, initMethods, initData, initComputed, initWatch 等等方法。

那这次我们通过 initData 来一起探索下 data 属性是怎么转化定义 getter/setter, 然后这个过程又做了什么事情。

## 响应式对象初始化 -- initData

我们来看下 initData 的源码:

```js
function initData (vm: Component) {
  let data = vm.$options.data
  // 把传入的 options 的 data 对象赋值到 vm._data
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}

  ...省略

  // proxy data on instance
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    // 判断props和data的属性命名是否冲突
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      // vue实例代理属性
      proxy(vm, `_data`, key)
    }
  }
  // observe data 监测数据变化
  observe(data, true /* asRootData */)
}
```

我们简单这看下, 一开始先判断传入的 options 的 data 属性是否为函数类型, 如果是则执行 getData 方法执行 data 函数获取对应的数据对象, 接着获取这个数据对象的 keys 并进行遍历, 这里遍历的目的有两个:

- 对 data 和 props 进行属性命名校验检测
- 代理 vm 实例的属性获取到 vm._data, 这也就是我们为什么可以通过 this.user.name 获取到我们定义的数据对象的属性值。

看完这里, 最后来到 observe 方法, 把 data 作为参数传入执行。


## 响应式对象初始化 -- observe

observe 主要是用来监测数据变化, 也就是初始化响应式对象

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  // 非Vnode的对象类型才执行监听器初始化
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    // 对于已经挂载监听器的value直接返回
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 把value传入Observer进行进行监听器实例化
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

这里可以很清楚看到, observer 通过判断当前的数据对象需要满足为非Vnode的对象类型才进行函数进一步执行, 接着这里通过判断当前的数据对象是否已经拥有 __ob__ 的属性, 如果有就直接获取返回 value.__ob__, 没有则实例化一个 Observer 并传入数据对象。

到这里我们可以大致看出, 这个函数最后返回的 ob 其实就是响应式对象。它使得传入的数据对象会增加 __ob__ 属性, 我们看下 Observer 是什么东东。

## 响应式对象初始化 -- Observer

```js
/**
 * Observer 用于给对象的属性添加getter和setter，用于进行依赖收集和更新下发
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    // 把实例对象添加到数据对象中的__ob__
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 数组类型遍历再次调用observe
      this.observeArray(value)
    } else {
      // 遍历对象调用defineReactive
      this.walk(value)
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

Observer 的够着函数通过 def 函数, 给传入的 value 这个数据对象定义 __ob__ 属性, 而对应这个属性的值就当前这个 Observer 的实例化对象。 def 函数的功能我们可以看下:

```js
/**
 * Define a property.
 */
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```
很简单, 就是调用 Object.defineProperty 来为 value 这个数据对象定义 __ob__ 属性, 这里需要注意的是为什么要通过 Object.defineProperty, 而不直接通过 value.__ob__ = this ? 先不揭晓答案, 我们只需要先注意这里 def 的调用中 !!enumerable 等同于 false, 也就是 __ob__ 属性在 value 数据对象中是不可枚举的。

继续回到构造函数, 先判断 当前 value 是否为数组, 如果为数组就调用 this.observeArray(value) 来遍历value, 并对遍历的每一项进行递归调用 observe, 目的是保证数组的每一项都能初始化响应式对象。

如果不为数组, 那也就是 value 为对象类型, 调用 walk 方法来遍历 value 的属性。 这时候, 我们会发现如果我们刚才通过 value.__ob__ = this 来定义 value 的 __ob__ 属性, 会导致 __ob__ 是可枚举, 这里遍历可以枚举出 __ob__, 但是 __ob__ 我们没必要对其进行响应式对象初始化, 所以也就是为什么要通过 Object.defineProperty 把 __ob__ 定义为不可枚举的属性。遍历过程其实就是把当前 value 的每个属性的值传参进行 defineReactive 调用。

## 响应式对象初始化 -- defineReactive

```js
/**
 * 定义一个响应式对象，动态添加getter和setter
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
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

我们刚才调用了 defineReactive 传入 value 和 keys[i], 这时候会执行 val = obj[key], 于是 val 这时候其实就是 value[key], 也就是数据对象的子属性, 这时候我们发现又会递归调用 observe 来对子属性进行响应式对象的初始化, 当然前提是子属性为非vnode的对象类型。 接着就对 value 的各个属性进行 getter/setter 的定义了, 具体 getter 和 setter 的实现我们这里不展开, 所以响应式对象的遍历初始化就到此结束了。 