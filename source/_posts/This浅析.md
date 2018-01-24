---
title: This浅析
date: 2017-11-14 00:02:26
tags:
  - Javascript
categories:
  - 前端
---

![你不知道的js](/images/you-don-kown.jpg)

## 概念

- this的绑定和函数的声明位置没有任何关系，只取决于函数的调用方式。当一个函数被调用时，会创建一个活动记录（有时候也称为执行上下文）。这个记录会包含函数在哪里被调用（调用栈），函数的调用方式，传入的参数等信息。this就是这个记录的一个属性，会在函数执行的过程中用到。

## 调用位置

- 调用位置就是函数在代码中被调用的位置（而不是声明位置）

## 绑定规则

- 默认绑定

```js
function foo () {
  console.log(this.a)
}
var a = 2
foo() // 2
```
当函数调用直接使用不带任何修饰的函数引用进行调用时，在非严格模式下，这时this会绑定到全局对象window/global，而在严格模式下，就会绑定到undefined

- 隐式绑定

```js
function foo () {
  console.log(this.a)
}
var obj = {
  a: 2,
  foo: foo
}

obj.foo() // 2
```

当函数引用拥有上下文对象时，隐式绑定规则会把函数调用中的this绑定到这个上下文对象。因为调用foo()时this被绑定到obj，因此this.a和obj.a是一样的。

  - 隐式丢失

```js
function foo () {
  console.log(this.a)
}
var obj = {
  a: 2,
  foo: foo
}
var bar = obj.foo // 函数别名
var a = 'lgybetter'
bar() // 'lgybetter'
```

虽然bar是obj.foo的一个引用，但是它引用的是foo函数本身，因此此时的bar()其实是一个不带任何修饰的函数调用，使用了默认绑定(非严格模式)

  - 参数传递引起的隐式绑定

```js
function foo () {
  console.log(this.a)
}
var obj = {
  a: 2,
  foo: foo
}
var a = 'lgybetter'
setTimeout(obj.foo, 100) // 'lgybetter'
```

JavaScript环境内置的setTimeout()函数实现和以下伪代码类似：

```js
function setTimeout (fn, delay) {
  fn() // <--- 调用位置
}
```

- 显示绑定：

  - call 和 apply：

```js
function foo () {
  console.log(this.a)
}
var obj = {
  a: 2
}
foo.call(obj) // 2
foo.apply(obj) // 2
```

简单使用call和apply显示绑定无法解决丢失绑定问题

- 硬绑定

  - 手动创建函数强制绑定

```js
function foo () {
  console.log(this.a)
}
var obj = {
  a: 2
}
var bar = function () {
  foo.call(obj)
}

bar() // 2
setTimeout(bar, 100) // 2

// 硬绑定的bar不可能再修改它的this
bar.call(window) // 2
```

通过创建bar()，并在它的内部手动调用foo.call(obj)，因此强制把foo的this绑定到了obj

  - 创建一个可以重复使用的函数

```js
function foo (something) {
  console.log(this.a, something)
  return this.a + something
}

function bind (fn, obj) {
  return function () {
    return fn.apply(obj, arguments)
  }
}

var obj = {
  a: 2
}

var bar = bind(foo, obj)
var b = bar(3) // 2 3
console.log(b) // 5
```

由于硬绑定式一种非常常用的模式，于是ES5提供了内置方法Function.prototype.bind，bind会返回一个硬编码的新函数，它会把你指定的参数设置为this的上下文并调用原始函数。

- API调用的"上下文"：

```js
function foo (el) {
  console.log(el, this.id)
}
var obj = {
  id: 'lgybetter'
}
[1,2,3].forEach(foo, obj)
// 1 lgybetter 2 lgybetter 3 lgybetter
```

- new绑定:

JavaScript中构造函数只是一些使用new操作符时被调用的函数。它们并不会属于某个类，也不会实例化一个类。它们就是被new操作符调用的普通函数而已。
实际上，不存在所谓的构造函数，只有对函数的“构造调用”。

new调用函数操作：
1. 创建一个全新的对象
2. 新对象会被执行[[Prototype]]连接
3. 新对象会绑定到函数调用的this
4. 如果函数没有返回其他对象，那么new表达式中的函数调用会自动返回这个新对象。

```js
function Foo (a) {
  this.a = a
}
var bar = new Foo(2)
console.log(bar.a) // 2
```

使用new来调用Foo(...)， 会构造一个新对象并把它绑定到Foo(...)调用中的this上。









