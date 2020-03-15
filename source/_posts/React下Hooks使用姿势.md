---
title: React下Hooks使用姿势
date: 2020-02-16 20:52:09
tags:
  - React
categories:
  - 前端
---

![banner](/images/react-hooks.jpg)

> 赋予函数组件拥有类组件的能力

## State Hook 使用

用 Hooks 实现一个根据按钮点击计数的组件

```js
import React from 'react'
import { useState } from 'react'
import { Button } from 'antd'

const Counter = () => {
  const [count, setCount] = useState(0)
  const onButtonClick = () => {
    setCount(count => count + 1)
  }
  return (
    <Button onClick={onButtonClick}>{count}</Button>
  )
}

export default Counter
```

这里使用 useState 的 hook 实现在函数组件声明状态，当需要进行状态变更的时候，使用 useState 返回的 api 变更函数进行状态更新，一般回调函数的方式进行状态更新：

```js
setCount(count => count + 1)
```

在 setCount 传入一个回调函数，这个回调函数会接收到当前 count 的最新状态值，接着回调函数通过 return 一个新的值从而进行状态更新。 


## Reducer Hook 使用

useState 本身其实是基于 useReducer 进行实现的一个 hook。

我们看下 useReducer 的用法

```js
import React from 'react'
import { useReducer } from 'react'
import { Button } from 'antd'

const countReducer = (state, action) => {
  switch(action.type) {
    case 'add':
      return state + 1 
    case 'sub':
      return state - 1
    default:
      return state
  }
}

const Counter = () => {
  const [count, dispatchCount] = useReducer(countReducer, 0)
  const onButtonClick = (type) => {
    dispatchCount({ type })
  }
  return (
    <>
      <p>{count}</p>
      <Button onClick={() => onButtonClick('add')}>Add</Button>
      <Button onClick={() => onButtonClick('sub')}>Sub</Button>
    </>
  )
}

export default Counter
```

定义一个 countReducer 函数, 通过不同的 action 参数来进行状态行为的变更，然后使用 useReducer 进行声明获得对应状态值 count，和变更触发方法 dispatchCount。

## Effect Hook 使用

当要在函数组件实现类似类组件的 componentDidMount， componentWillReceiveProps 的功能的时候，我们需要借助 useEffect 的 hook 来实现

```js
import React, { useEffect, useState } from 'react'

const Counter = () => {
  const [autoCount, setAutoCount] = useState(0)

  useEffect(() => {
    const timer = setInterval(() => {
      setAutoCount(autoCount => autoCount + 1)
    }, 1000)
    return () => { clearInterval(timer) }
  }, [])

  return (
    <>
      <p>AutoCount: {autoCount}</p>
    </>
  )
}

export default Counter
```

这个例子主要是在函数渲染后，通过 useEffect 的 hook 来实现一个自动计数的功能，useEffect 接收一个回调函数，用来进行当前的 effect hook 的响应操作，第二个参数为一个依赖的变量数组，当传入的依赖数组变量为空，则起到了类似 componentDidMount 的作用，在 useEffect 的回调函数值可以 return 一个函数用于处理当前的 effect hook 需要销毁的尾处理。

我们再举一个例子：

```js
import React, { useEffect, useState } from 'react'
import { useReducer } from 'react'
import { Button } from 'antd'

const countReducer = (state, action) => {
  switch(action.type) {
    case 'add':
      return state + 1 
    case 'sub':
      return state - 1
    default:
      return state
  }
}

const Counter = () => {
  const [count, dispatchCount] = useReducer(countReducer, 0)
  const [full, setFull] = useState(false)
  const onButtonClick = (type) => {
    dispatchCount({ type })
  }

  useEffect(() => {
    count >= 10 ? setFull(true) : setFull(false)
  }, [count])
   
  return (
    <>
      <p>Full: {full ? 'Full' : 'Not Full'}</p>
      <p>{count}</p>
      <Button onClick={() => onButtonClick('add')}>Add</Button>
      <Button onClick={() => onButtonClick('sub')}>Sub</Button>
    </>
  )
}

export default Counter
```

这里例子主要是修改了上面 useReducer 所用到的例子，当 count 增加到 超过 10 的时候，useEffect 通过监听 count 的依赖变化，从而来判断并修改 full 的状态值，这里有点 vue 的 watch 的含义。

同样，我们也可以对 props 的参数传入到 useEffect 的依赖中，当 props 中的数据发生变化，可以触发 useEffect 的回调函数的执行，这样就起到了 componentWillReceiveProps 的作用。

## Context Hooks 使用

Context Hooks 目的是为了解决多层级组件的数据传递问题，通过 Context 的方式来中心化处理组件的数据更新，同时触发视图的渲染更新。

使用 Context Hooks 的方式如下：

Context 声明定义： context.js

```js
import React, {createContext} from 'react'

const Context = createContext('')

export default Context

```

定义父组件：page-context.js

```js
import React from 'react'
import Context from '../components/context'
import Counter from '../components/counter'

const App = () => {
  return (
    <>
      <Context.Provider value="This is Counter">
        <Counter></Counter>
      </Context.Provider>
    </>
  )
}

export default App
```

定义 Counter 子组件：counter.js

```js
import React, { useEffect, useState, useContext } from 'react'
import { useReducer } from 'react'
import { Button } from 'antd'
import Context from './context'

const countReducer = (state, action) => {
  switch(action.type) {
    case 'add':
      return state + 1 
    case 'sub':
      return state - 1
    default:
      return state
  }
}

const Counter = () => {
  const [count, dispatchCount] = useReducer(countReducer, 0)
  const [full, setFull] = useState(false)
  const context = useContext(Context)
  const onButtonClick = (type) => {
    dispatchCount({ type })
  }

  useEffect(() => {
    count >= 10 ? setFull(true) : setFull(false)
  }, [count])
   
  return (
    <>
      <p>Context: {context}</p>
      <p>Full: {full ? 'Full' : 'Not Full'}</p>
      <p>{count}</p>
      <Button onClick={() => onButtonClick('add')}>Add</Button>
      <Button onClick={() => onButtonClick('sub')}>Sub</Button>
    </>
  )
}

export default Counter
```

使用 context 的主要方式是使用 createContext 定义一个上下文对象 Context，接着使用 Context 对象的 Provider 提供者这个属性，提供给到需要当前上下文的子组件。
在子组件中，使用 useContext 把 Context 对象传递进去，获得到对应的消费者并使用该消费者进行视图渲染或数据计算。

## Ref Hook 使用

```js
import React, { useEffect, useRef } from 'react'
import { Button } from 'antd'

const Counter = () => {
  const ref = useRef()

  useEffect(() => {
    console.log(ref.current)
  }, [])

  return (
    <>
      <Button ref={ref} onClick={() => onButtonClick('add')}>Add</Button>
    </>
  )
}

export default Counter
```

使用 useRef 来声明获得当前的 ref 对象，赋值给对应的组件节点，ref.current 则表示为当前 ref 对应的组件的 dom 节点对象。

## Memo Hooks 和 Callback Hooks 使用


官网对着两个 Hook 的解释如下

### useCallback

```js
const memoizedCallback = useCallback(
  () => {
    doSomething(a, b);
  },
  [a, b],
);
```

在传入的依赖值 a, b, 不变的情况下, memoizedCallback 的引用保持不变，useCallback 的第一个入参函数会被缓存。

### useMemo

```js
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

在传入的依赖值 a, b, 不变的情况下, memoizedValue 的值保持不变, useMemo函数的第一个入参函数不会被执行。


useCallback(fn, deps) 等价于 useMemo(() => fn, deps)


### useCallback 使用

```js
import React, { useState, useCallback } from 'react'

const Counter = () => {
  const [count, setCount] = useState(0)
  const onCount = useCallback(() => {
    return setCount(count => count + 1)
  }, [])
  return (
    <>
      <p>{count}</p>
      <ChildComponent onCount={onCount}></ChildComponent>
    </>
  )
}

export default Counter
```

```js
import React, { memo } from 'react'
import { Button } from 'antd'

const ChildComponent = memo(({ onCount }) => {
  return <Button onClick={onCount}>Add</Button>
})
```

因为我们使用 useCallback 缓存了 onCount函数，使得当 count 发生变化时，Counter 重新渲染后 onCount 保持引用不变，传入 ChildComponent 借助 memo 方法使得 ChildComponent 组件避免了不必要的重新渲染。

### useMemo 使用

> useCallback 是根据传入的依赖,缓存第一个入参函数。useMemo 是根据传入的依赖，缓存第一个入参函数执行后的值。

useMemo 个人理解与 vue 的 computed 属性类似

```js
import React, { useState, useCallback, useMemo } from 'react'

const Counter = () => {
  const [count, setCount] = useState(0)
  const onCount = useCallback(() => {
    return setCount(count => count + 1)
  }, [])
  const couteComputed = useMemo(() => {
    return (count * 1000) / 1024 
  }, [count]) 
  return (
    <>
      <p>{count}</p>
      <p>{couteComputed}</p>
      <ChildComponent onCount={onCount}></ChildComponent>
    </>
  )
}

export default Counter
```

useMemo 的依赖就可以只在指定变量值更改时才执行计算，从而达到节约内存消耗。
