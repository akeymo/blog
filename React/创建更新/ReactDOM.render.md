
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

### 创建fiberRoot和fiber
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
`createFiberRoot`方法位于同级目录下的`ReactFiberRoot.js`文件，作用是创建了一个`fiberRoot`，同时创建一个`fiber`对象，这个`fiber`对象挂到`fiberRoot`的`current`上，同时`fiber`对象的`stateNode`又指向`fiberRoot`。
```
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
): FiberRoot {
  // 创建一个FiberRoot
  const root: FiberRoot = (new FiberRootNode(containerInfo, tag, hydrate): any);
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  // 创建一个Fiber对象
  const uninitializedFiber = createHostRootFiber(tag);
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;

  initializeUpdateQueue(uninitializedFiber);

  return root;
}
```

`fiberRoot`的概念：
1. 整个应用的起点。
2. 包含应用挂载的目标节点。
3. 记录整个应用更新过程的各种信息。

`fiberRoot`对应的数据结构：
```
const fiberRoot = {
    tag: tag,
    // 当前应用对应的fiber对象,是整个fiber树的顶点
    current: Fiber,
    // root节点，应用挂载的DOM节点
    containerInfo: any,
    // 只有持久化更新会用到，react-dom里不会用到
    pendingChildren: any,

    pingCache: null,
    finishedExpirationTime: NoWork,
    // 记录一次更新渲染过程中完成了的更新任务
    finishedWork: Fiber | null,
    // 在任务被挂起的时候通过setTimeout设置的返回内容，用来下一次如果有新任务挂起时清理还没有触发的timeout
    timeoutHandle: timeoutHandle | noTimeout,
    // 顶层context对象，只有主动调用renderSubtreeIntoContainer时才会有用
    context: Object | null,
    pendingContext: Object | null,
    // 用来确定第一次渲染的时候是否需要融合
    hydrate: boolean,
    callbackNode: null,
    callbackPriority: NoPriority,
    // 最老和最新的在提交时被挂起的任务
    firstSuspendedTime: ExpirationTime,
    lastSuspendedTime: ExpirationTime,
    // 最老的不确定是否会挂起的优先级（所有任务进来一开始都是这个状态）
    firstPendingTime: ExpirationTime,
    // 最新的通过一个promise被resolve并且可以重新尝试的优先级
    lastPingedTime: ExpirationTime,

    nextKnownPendingLevel: NoWork,
    lastExpiredTime: ExpirationTime,
}
```

`fiber`又是什么呢？它是：
1. 每一个`ReactElement`对应一个`fiber`对象。
2. 记录节点的各种状态。每个`class component`的`state`以及`props`都是记录在`fiber`对象上，更新节点之后才放到`this`上。实现`Hooks`的基础。
3. 串联整个应用形成树结构。

`fiber`对应的数据结构：
```
const fiber = {
    // Instance
    // 标记不同的组件类型
    tag: tag,
    // ReactElament里面的key
    key: null | string,
    // ReactElement.type，也就是我们调用createElement的第一个参数
    elementType: any,
    // 异步组件resolved之后返回的内容，一般是function或者class
    type: any,
    // 对应节点实际的一个实例
    // 例如一个class component对应的就是class的一个实例，function component没有实例就是没有stateNode
    stateNode: any,

    // Fiber
    // 指向它在fiber树中的parent，用来在处理完这个节点后向上返回
    return: Fiber | null,
    // 指向自己的第一个子节点
    child: Fiber | null,
    // 指向自己的兄弟节点
    sibling: Fiber | null,
    index: Number,

    ref: null,

    // 新的变动带来的新的props
    pendingProps: any,
    // 上一次渲染完成之后的props
    memoizedProps: any,
    // 该fiber对应的组件产生的update会存放在这个队列里面
    updateQueue: UpdateQueue<any> | null,
    // 上一次渲染的时候的state
    memoizedState: any,
    dependencies: null,
    // 
    mode: TypeOfMode,

    // Effects
    // 用来记录Side Effect
    effectTag: SideEffectTag,
    // 单链表用来快速查找下一个side effect
    nextEffect: Fiber | null,

    // 子树中第一个side effect
    firstEffect: Fiber | null,
    // 子树中最后一个side effect
    lastEffect: Fiber | null,

    // 代表任务在未来的哪个时间应该被完成
    // 不包括它的子树产生的任务
    expirationTime: ExpirationTime,
    // 快速确定子树中是否有不在等待的变化
    childExpirationTime: ExpirationTime,

    // 在Fiber树更新过程中，每个Fiber都会有一个跟其对应的Fiber
    // 我们称它为 current(当前的) <===> workInProgress(要更新的)
    // 在渲染完成之后它们会交换位置
    alternate: Fiber | null,
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

