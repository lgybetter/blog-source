---
title: ReactDOM.render 初始化过程浅析
date: 2021-07-19 23:16:24
tags:
  - React
categories:
  - 前端
---

![banner](/images/react.jpg)

## ReactDOM.render 用法

```tsx
ReactDOM.render(<Root />, document.getElementById('root'))
```

## ReactDOM 定义

```js
const ReactDOM: Object = {
  // 省略...
  hydrate(element: React$Node, container: DOMContainer, callback: ?Function) {
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      true,
      callback,
    );
  },
  render(
    element: React$Element<any>,
    container: DOMContainer,
    callback: ?Function,
  ) {
    return legacyRenderSubtreeIntoContainer(
      null,
      element,
      container,
      false,
      callback,
    );
  },
  // 省略...
}
```
hydrate 和 render 本质都是调用 legacyRenderSubtreeIntoContainer 方法，只是第四个参数 forceHydrate 传入有区别，render 我们很清楚知道就是进行渲染调用，那 hydrate 是用来做什么？

> hydrate 描述的是 ReactDOM 复用 ReactDOMServer 服务端渲染的内容时尽可能保留结构，并补充事件绑定等 Client 特有内容的过程。


## ReactDOM render 过程

1. render 调用 legacyRenderSubtreeIntoContainer

```js
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  // 省略...
  let root: Root = (container._reactRootContainer: any);
  if (!root) {
    // 创建 ReactRoot
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    if (typeof callback === 'function') {
      const originalCallback = callback;
      // 封装回调函数
      callback = function() {
        const instance = DOMRenderer.getPublicRootInstance(root._internalRoot);
        originalCallback.call(instance);
      };
    }
    // 执行渲染
    DOMRenderer.unbatchedUpdates(() => {
      if (parentComponent != null) {
        // 忽略
        root.legacy_renderSubtreeIntoContainer(
          parentComponent,
          children,
          callback,
        );
      } else {
        // 调用 ReactRoot 的 render 方法
        root.render(children, callback);
      }
    });
  } else {
    // 省略...
  }
  return DOMRenderer.getPublicRootInstance(root._internalRoot);
}
```
legacyRenderSubtreeIntoContainer 通过 legacyCreateRootFromDOMContainer 这个方法创建了一个 ReactRoot 实例，然后赋值给 root 和 container._reactRootContainer，完成后执行回调函数的封装，接着执行
unbatchedUpdates 的回调函数，会执行 root 的 render 方法，也就是 ReactRoot 的原型方法 render，最后传入 root._internalRoot，执行后返回 DOMRenderer.getPublicRootInstance 的结果;

2. legacyCreateRootFromDOMContainer 实现过程

```js
function legacyCreateRootFromDOMContainer(
  container: DOMContainer,
  forceHydrate: boolean,
): Root {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    while ((rootSibling = container.lastChild)) {
      container.removeChild(rootSibling);
    }
  }
  const isConcurrent = false;
  return new ReactRoot(container, isConcurrent, shouldHydrate);
}
```

判断当前的 container 是否需要进行 forceHydrate，forceHydrate 为 false，则移除 container 下的所有子节点，然后标记 isConcurrent 为 false，实例化 ReactRoot；

3. ReactRoot 的构造函数

```js
function ReactRoot(
  container: Container,
  isConcurrent: boolean,
  hydrate: boolean,
) {
  const root = DOMRenderer.createContainer(container, isConcurrent, hydrate);
  this._internalRoot = root;
}
```

```js
export function createContainer(
  containerInfo: Container,
  isConcurrent: boolean,
  hydrate: boolean,
): OpaqueRoot {
  return createFiberRoot(containerInfo, isConcurrent, hydrate);
}
```

调用 DOMRenderer.createContainer 创建一个 FiberRoot，并把当前的 FiberRoot 实例化对象记录到 root._internalRoot 中。


4. ReactRoot 实例化对象 render 调用触发更新

```js
ReactRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  // 创建一个 work
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    // 等待当前的 work 执行完成后执行回调函数
    work.then(callback);
  }
  // 触发更新
  DOMRenderer.updateContainer(children, root, null, work._onCommit);
  return work;
};
```

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  // container 为当前的 FiberRoot 实例化对象
  const current = container.current;
  // 用于计算任务调度优先级
  const currentTime = requestCurrentTime();
  const expirationTime = computeExpirationForFiber(currentTime, current);
  // 推送和触发更新调度任务
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
  );
}
```

ReactRoot 的原型方法 render 创建了一个 work，等待 work 执行完成后会执行 callback 的回调，而 work 的回调执行在于 wor._onCommit 的触发，render 方法最后调用了 DOMRenderer.updateContainer, updateContainer 这个方法一开进行 currentTime 和 expirationTime 来进行任务优先级的计算，然后执行 updateContainerAtExpirationTime 来进行渲染更新任务的触发。

5. updateContainerAtExpirationTime 实现

```js
export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  const current = container.current;
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }
  return scheduleRootUpdate(current, element, expirationTime, callback);
}
```

```js
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  // 创建一个 update 更新任务
  const update = createUpdate(expirationTime);
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    warningWithoutStack(
      typeof callback === 'function',
      'render(...): Expected the last optional `callback` argument to be a ' +
        'function. Instead received: %s.',
      callback,
    );
    update.callback = callback;
  }
  // 把当前的更新任务和当前的 FiberRoot 压入任务队列
  enqueueUpdate(current, update);
  // 触发任务队列进行任务执行
  scheduleWork(current, expirationTime);
  return expirationTime;
}
```
updateContainerAtExpirationTime 主要是执行 scheduleRootUpdate，scheduleRootUpdate 方法生成一个 update 更新任务，然后把当前的更新任务和当前的 FiberRoot 压入任务队列，接着触发任务队列进行任务执行。


到这里就完成了 ReactDOM render 的初始化，后续界面视图的更新渲染就依赖于 React 的 Fiber Schedule 进行更新调度。