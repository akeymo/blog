
`packages/react-dom/src/client/ReactDOM.js`里创建ReactDOM对象，`render`里调用`legacyRenderSubtreeIntoContainer`方法，这个方法会判断传入的`container`是否存在`_reactRootContainer`属性。
第一次渲染肯定不存在该属性。
调用`legacyCreateRootFromDOMContainer`方法创建`reactRoot`；并且调用`unbatchedUpdates`方法，不进行批量更新，因为是初次渲染需要尽快完成，该方法里调用`root.render`方法。

```
function legacyRenderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  ...
  
  let root: Root = (container._reactRootContainer: any);
  // 首次渲染container对象上肯定没有_reactRootContainer属性
  if (!root) {
    // Initial mount
    // 创建一个reactRoot
    root = container._reactRootContainer = legacyCreateRootFromDOMContainer(
      container,
      forceHydrate,
    );
    if (typeof callback === 'function') {
      ...
    }
    // Initial mount should not be batched.
    DOMRenderer.unbatchedUpdates(() => {
      if (parentComponent != null) {
        root.legacy_renderSubtreeIntoContainer(
          parentComponent,
          children,
          callback,
        );
      } else {
        root.render(children, callback);
      }
    });
  } else {
    ...
  }
  return DOMRenderer.getPublicRootInstance(root._internalRoot);
}
```

`legacyCreateRootFromDOMContainer`负责创建`reactRoot`，在方法内`new`一个`ReactRoot`对象。
```
function legacyCreateRootFromDOMContainer(
  container: DOMContainer,
  forceHydrate: boolean,
): Root {
  const shouldHydrate =
    forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
  // First clear any existing content.
  if (!shouldHydrate) {
    let warned = false;
    let rootSibling;
    // 遍历移除子节点
    while ((rootSibling = container.lastChild)) {
      if (__DEV__) {
        ...
      }
      container.removeChild(rootSibling);
    }
  }
  
  ...
  
  // Legacy roots are not async by default.
  const isConcurrent = false;
  return new ReactRoot(container, isConcurrent, shouldHydrate);
}
```

`ReactRoot`方法内调用`DOMRenderer.createContainer`创建`fierRoot`对象。这个方法存放在`packages/react-reconciler/src/ReactFiberReconciler.js`内，利用`createFiberRoot`创建一个`fiberRoot`对象。

__fiberRoot对象是：__
1. 整个应用的起点。
2. 包含应用挂载的目标节点。
3. 记录整个应用更新过程的各种信息。

__对应的数据结构：__
```
const fiberRoot = {
  // root节点，应用挂载的DOM节点
  containerInfo: any,
  // 只有在持久更新中会用到，也就是不支持增量更新的平台，react-dom不会用到
  pendingChildren: any,
  // 当前应用对应的fiber对象,是整个fiber树的顶点
  current: Fiber,

  // 以下的优先级是用来区分
  // 1) 没有提交(committed)的任务
  // 2) 没有提交的挂起任务
  // 3) 没有提交的可能被挂起的任务
  // 我们选择不追踪每个单独的阻塞登记，为了兼顾性能
  // 最老和新的在提交的时候被挂起的任务
  earliestSuspendedTime: ExpirationTime,
  latestSuspendedTime: ExpirationTime,
  // 最老和最新的不确定是否会挂起的优先级（所有任务进来一开始都是这个状态）
  earliestPendingTime: ExpirationTime,
  latestPendingTime: ExpirationTime,
  // 最新的通过一个promise被reslove并且可以重新尝试的优先级
  latestPingedTime: ExpirationTime,

  // 如果有错误被抛出并且没有更多的更新存在，我们尝试在处理错误前同步重新从头渲染
  // 在`renderRoot`出现无法处理的错误时会被设置为`true`
  didError: boolean,

  // 正在等待提交的任务的`expirationTime`
  pendingCommitExpirationTime: ExpirationTime,
  // 已经完成的任务的FiberRoot对象，如果你只有一个Root，那他永远只可能是这个Root对应的Fiber，或者是null
  // 在commit阶段只会处理这个值对应的任务
  finishedWork: Fiber | null,
  // 在任务被挂起的时候通过setTimeout设置的返回内容，用来下一次如果有新的任务挂起时清理还没触发的timeout
  timeoutHandle: TimeoutHandle | NoTimeout,
  // 顶层context对象，只有主动调用`renderSubtreeIntoContainer`时才会有用
  context: Object | null,
  pendingContext: Object | null,
  // 用来确定第一次渲染的时候是否需要融合
  +hydrate: boolean,
  // 当前root上剩余的过期时间
  // TODO: 提到renderer里面区处理
  nextExpirationTimeToWorkOn: ExpirationTime,
  // 当前更新对应的过期时间
  expirationTime: ExpirationTime,
  // 顶层批次（批处理任务？）这个变量指明一个commit是否应该被推迟
  // 同时包括完成之后的回调
  // 貌似用在测试的时候？
  firstBatch: Batch | null,
  // root之间关联的链表结构
  nextScheduledRoot: FiberRoot | null,
}
```

创建`fiberRoot`对象的同时，还会创建一个`fiber`对象。并且这个`fiber`对象会被赋到`fiberRoot`的`current`属性上。同时，`fiber`对象的`stateNode`也会指向`fiberRoot`。

__fiber对象是：__
1. 每一个`ReactElement`对应一个`fiber`对象。
2. 记录节点的各种状态。每个`class component`的`state`以及`props`都是记录在`fiber`对象上，更新节点之后才放到`this`上。实现`Hooks`的基础。
3. 串联整个应用形成树结构。

__对应的数据结构：__
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
    // 一个列表，存放这个Fiber依赖的context
    firstContextDependency: ContextDependency<mixed> | null,
    
    // 用来描述当前Fiber和他子树的`Bitfield`
	// 共存的模式表示这个子树是否默认是异步渲染的
    // Fiber被创建的时候他会继承父Fiber
    // 其他的标识也可以在创建的时候被设置
    // 但是在创建之后不应该再被修改，特别是他的子Fiber创建之前
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

`ReactRoot`对象的`render`方法调用了`DOMRenderer.updateContainer`并且把创建的`fiberRoot`传入。
它计算出一个`expirationTime`，调用了`updateContainerAtExpirationTime`方法。
`updateContainerAtExpirationTime`调用了`scheduleRootUpdate`。

```
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  ...
  
  const update = createUpdate(expirationTime);
  update.payload = {element};

  ...
  
  // 将update加入对应fiber对象的update队列
  enqueueUpdate(current, update);

  // 开始进行任务调度
  scheduleWork(current, expirationTime);
  return expirationTime;
}
```

利用`expirationTime`创建了一个`update`对象。

__update对象：__
1. 用于记录组件状态的改变。
2. 存放于`fiber`对象的`updateQueue`中。一次更新中，一个`queue`里可能存在多个`update`对象。
3. 多个`update`可以同时存在。在一个事件里可能存在多次调用`setState`，这些`setState`不会分多次执行。每次`setState`都会创建一个`update`对象，这些`update`会放在一个`queue`里，再一次性执行更新。

__对应数据结构：__
```
const update = {
  // 更新的过期时间
  expirationTime: ExpirationTime,

  // UpdateState = 0; 更新状态
  // ReplaceState = 1; 替换更新
  // ForceUpdate = 2; 强制更新
  // CaptureUpdate = 3; 渲染中如果捕获到错误会产生的更新
  // 指定更新的类型，值为以上几种
  tag: 0 | 1 | 2 | 3,
  // 更新内容，比如`setState`接收的第一个参数
  payload: any,
  // 对应的回调，`setState`，`render`都有
  callback: (() => mixed) | null,

  // 指向下一个更新
  next: Update<State> | null,
  // 指向下一个`side effect`
  nextEffect: Update<State> | null,
}
```

`enqueueUpdate`方法里，如果`fiber`对象的`alternate`为空，说明没有进行过更新是第一次渲染，会调用`createUpdateQueue`创建一个`updateQueue`对象。如果不是第一次渲染，那么根据条件判断去更新`updateQueue`。

__updateQueue对象的数据结构：__
```
const UpdateQueue = {
  // 每次操作完更新之后的`state`
  baseState: State,

  // 队列中的第一个`Update`
  firstUpdate: Update<State> | null,
  // 队列中的最后一个`Update`
  lastUpdate: Update<State> | null,

  // 第一个捕获类型的`Update`
  firstCapturedUpdate: Update<State> | null,
  // 最后一个捕获类型的`Update`
  lastCapturedUpdate: Update<State> | null,

  // 第一个`side effect`
  firstEffect: Update<State> | null,
  // 最后一个`side effect`
  lastEffect: Update<State> | null,

  // 第一个和最后一个捕获产生的`side effect`
  firstCapturedEffect: Update<State> | null,
  lastCapturedEffect: Update<State> | null,
};
```