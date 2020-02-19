
`packages/react-dom/src/client/ReactDOM.js`里创建ReactDOM对象，但实际的`render`方法存放在同级目录下的`ReactDOMLegacy.js`文件里。

`render`里调用`legacyRenderSubtreeIntoContainer`方法，这个方法会判断传入的`container`是否存在`_reactRootContainer`属性。
第一次渲染肯定不存在该属性。
调用`legacyCreateRootFromDOMContainer`方法创建`fiberRoot`；并且调用`unbatchedUpdates`方法，不进行批量更新，因为是初次渲染需要尽快完成，该方法内调用`updateContainer`方法。

```
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  if (__DEV__) {
    ...
  }

  let root: RootType = (container._reactRootContainer: any);
  let fiberRoot;
  if (!root) {
    // 初次渲染
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    fiberRoot = root._internalRoot;
    if (typeof callback === 'function') {
      const originalCallback = callback;
      callback = function() {
        const instance = getPublicRootInstance(fiberRoot);
        originalCallback.call(instance);
      };
    }
    // Initial mount should not be batched
    // 初次渲染不进行批量更新
    unbatchedUpdates(() => {
      updateContainer(children, fiberRoot, parentComponent, callback);
    });
  } else {
    ...
  }
  return getPublicRootInstance(fiberRoot);
}
```
由此开始分成两个任务。

### 创建fiberRoot
`legacyCreateRootFromDOMContainer`方法内调用`createLegacyRoot`方法，该方法位于同级目录下的`ReactDOMRoot.js`文件内。经过一系列的调用，最终有具体实现的方法是`createRootImpl`方法，方法内调用`createContainer`方法。
```
function createRootImpl(
  container: DOMContainer,
  tag: RootTag,
  options: void | RootOptions,
) {
  const hydrate = options != null && options.hydrate === true;
  const hydrationCallbacks =
    (options != null && options.hydrationOptions) || null;
    
  const root = createContainer(container, tag, hydrate, hydrationCallbacks);
  
  markContainerAsRoot(root.current, container);
  if (hydrate && tag !== LegacyRoot) {
    const doc =
      container.nodeType === DOCUMENT_NODE
        ? container
        : container.ownerDocument;
    eagerlyTrapReplayableEvents(doc);
  }
  return root;
}
```

`ceateContainer`方法位于`packages/react-reconciler/src/ReactFiberReconciler.js`文件内，具体实现调用`createFiberRoot`方法。
`createFiberRoot`方法位于同级目录下的`ReactFiberRoot.js`文件，作用是创建了`FiberRootNode`实例赋给`root`，并且创建了一个`FiberNode`实例挂载到`root`的`current`属性上，最终将`root`放到`FiberNode`的`stateNode`属性上，作为参数传入`initializeUpdateQueue`，该方法将创建好的对象加入到对应的`fiber`的`queue`上。
```
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  // 创建一个FiberRootNode实例，挂载一些属性
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // 创建一个FiberNode，挂载一些属性
  const uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  // 将创建好的对象加入对应的fiber的queue里
  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

### 生成expirationTime和开始任务调度
`updateContainer`同样位于`ReactFiberReconciler.js`文件里。它创建了一个`expirationTime`，并且利用这个`expirationTime`创建了一个`update`对象。利用`enqueueUpdate`方法将`update`对象添加到对应`fiber`对象的`update`队列。利用`scheduleWork`开始进行任务调度。

```
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  if (__DEV__) {
    onScheduleRoot(container, element);
  }
  const current = container.current;
  const currentTime = requestCurrentTimeForUpdate();
  if (__DEV__) {
    ...
  }
  const suspenseConfig = requestCurrentSuspenseConfig();
  // 生成expirationTime
  const expirationTime = computeExpirationForFiber(
    currentTime,
    current,
    suspenseConfig,
  );

  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  if (__DEV__) {
    ...
  }

  // 创建update对象
  const update = createUpdate(expirationTime, suspenseConfig);
  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    ...
  }

  // 将update加入对应fiber对象的update队列
  enqueueUpdate(current, update);
  // 开始进行任务调度
  scheduleWork(current, expirationTime);

  return expirationTime;
}
```

### 总结
`ReactDOM.render`做了两件事，一个是创建了`fiberRoor`，另一个就是生成了`expirationTime`和开始进行任务调度。
