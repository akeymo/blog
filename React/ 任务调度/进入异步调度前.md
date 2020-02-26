创建更新后，就会调用`scheduleWork`进入任务调度的阶段。

### scheduleWork
在`scheduleWork`方法中，首先会调用`scheduleWorkToRoot`方法。这个方法会根据传入的`fiber`寻找对应的`fiberRoot`；同时，根据优先级判断是否需要更新`fiber`对象及其`alternate`的`expirationTime`；另外，也会遍历`fiber`的父节点链，判断是否要更新父节点的`childExpirationTime`。

```
function scheduleWorkToRoot(fiber: Fiber, expirationTime): FiberRoot | null {
  recordScheduleUpdate();

  if (__DEV__) {
    if (fiber.tag === ClassComponent) {
      const instance = fiber.stateNode;
      warnAboutInvalidUpdates(instance);
    }
  }

  // Update the source fiber's expiration time
  if (
    fiber.expirationTime === NoWork ||
    fiber.expirationTime > expirationTime // 之前产生过更新还没完成；之前产生的更新的优先级比较低
  ) {
    fiber.expirationTime = expirationTime;
  }
  let alternate = fiber.alternate;
  if (
    alternate !== null &&
    (alternate.expirationTime === NoWork ||
      alternate.expirationTime > expirationTime)
  ) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  let node = fiber.return;
  let root = null;
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode;
  } else {
    while (node !== null) {
      alternate = node.alternate;
      // node的childExpirationTime是它的子树中优先级最高的expirationTime
      if (
        node.childExpirationTime === NoWork ||
        node.childExpirationTime > expirationTime
      ) {
        node.childExpirationTime = expirationTime;
        if (
          alternate !== null &&
          (alternate.childExpirationTime === NoWork ||
            alternate.childExpirationTime > expirationTime)
        ) {
          alternate.childExpirationTime = expirationTime;
        }
      } else if (
        alternate !== null &&
        (alternate.childExpirationTime === NoWork ||
          alternate.childExpirationTime > expirationTime)
      ) {
        alternate.childExpirationTime = expirationTime;
      }
      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode;
        break;
      }
      node = node.return;
    }
  }

  if (root === null) {
    if (__DEV__ && fiber.tag === ClassComponent) {
      warnAboutUpdateOnUnmounted(fiber);
    }
    return null;
  }

  ...

  return root;
}
```

继续回到`scheduleWork`中。

```
if (
    !isWorking &&
    nextRenderExpirationTime !== NoWork &&
    expirationTime < nextRenderExpirationTime
  ) {
    // This is an interruption. (Used for performance tracking.)
    // 高优先级任务打断低优先级任务
    interruptedBy = fiber;
    resetStack();
  }
```

`working`阶段包括`render`阶段和`committing`阶段。`nextRenderExpirationTime`会在新的`renderRoot`时被设置成当前的`expirationTime`，一旦被设置，只有当下次任务是`NoWork`的时候才会被设置成`NoWork`。

那么，上面这段判断就表示当前没有任务在执行，并且之前有执行过任务还没有执行完，新的`expirationTime`优先级又更高，因此上一个任务需要被打断，并且清空之前任务的`stack`，因为`React`只有一个`stack`。

```
if (
    // If we're in the render phase, we don't need to schedule this root
    // for an update, because we'll do it before we exit...
    // Working：包含render阶段和committing阶段
    // Committing：第二阶段提交阶段，fiber树整体渲染完成后更新到DOM上的过程。不可打断
    !isWorking ||
    isCommitting ||
    // ...unless this is a different root than the one we're rendering.
    nextRoot !== root
) {
    const rootExpirationTime = root.expirationTime;
    requestWork(root, rootExpirationTime);
}
if (nestedUpdateCount > NESTED_UPDATE_LIMIT) {
    // 防止进入无限更新循环
    // Reset this back to zero so subsequent updates don't throw.
    nestedUpdateCount = 0;
    invariant(
        false,
        'Maximum update depth exceeded. This can happen when a ' +
        'component repeatedly calls setState inside ' +
        'componentWillUpdate or componentDidUpdate. React limits ' +
        'the number of nested updates to prevent infinite loops.',
    );
}
```

最后，根据条件判断如果不是在`working`阶段 | 在`committing`阶段 | 当前`root`不是正在`render`的`root`，就进入`requestWork`。

### requestWork
在`requestWork`方法中，首先是调用`addRootToSchedule`。这个方法是将`fiberRoot`添加进调度队列中。

在添加之前，会先判断`fiberRoot`是否已经被调度。如果`fiberRoot`的`nextScheduledRoot`等于`null`，说明还未调度。通过`lastScheduledRoot`是否等于`null`来判断调度队列中是否有已经存在的任务。

调度队列是个闭环，`lastScheduledRoot.nextScheduledRoot`会指向`firstScheduledRoot`。如果是已经调度的情况下，判断是否更新`fiberRoot`的`expirationTime`。

```
function addRootToSchedule(root: FiberRoot, expirationTime: ExpirationTime) {
  // Add the root to the schedule.
  // Check if this root is already part of the schedule.
  if (root.nextScheduledRoot === null) {
    // This root is not already scheduled. Add it.
    root.expirationTime = expirationTime;
    if (lastScheduledRoot === null) {
      // 调度队列中还没有任务
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else {
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else {
    // This root is already scheduled, but its priority may have increased.
    const remainingExpirationTime = root.expirationTime;
    if (
      remainingExpirationTime === NoWork ||
      expirationTime < remainingExpirationTime
    ) {
      // Update the priority.
      root.expirationTime = expirationTime;
    }
  }
}
```

在`requestWork`方法的最后，根据`expirationTime`来分别调用`performSyncWork`和`scheduleCallbackWithExpirationTime`。

同步的情况下基本等于直接调用`performWork`，而在`scheduleCallbackWithExpirationTime`中，是根据时间片来执行任务的，会涉及到`requestIdleCallback`，但最终也会回到`performWork`。

```
if (expirationTime === Sync) {
    performSyncWork();
} else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
}
```

非`Sync`情况下，接下去就是进入异步调度阶段了。