任务调度时说到，`performUnitOfWork`会调用`beginWork`方法进行每一个节点的更新。

`beginWork`这个方法存在于`ReactFiberBeginWork.js`这个文件内，它接收`current`、`workInProgress`和`renderExpirationTime`三个参数。

* `current`表示当前的`fiber`对象，如果是第一次进入方法，那么就对应`RootFiber`。
* `workInProgress`表示当前操作的节点对象的副本。
* `renderExpirationTime`表示本次更新的最大优先级。

`beginWork`方法的作用就是判断是否需要更新，并返回下一个需要更新的节点或者`null`。

先后两次的`props`是一样的，并且本次没有更新或者更新的优先级不高这次更新不需要执行，那么就进入本次不需要更新的判断条件。虽然传入的节点不需要更新，但还需要继续判断它的子节点是否需要更新。如果需要更新，则返回需要更新的子节点。

```
const oldProps = current.memoizedProps;
const newProps = workInProgress.pendingProps;
if (
    oldProps === newProps &&
    !hasLegacyContextChanged() &&
    (updateExpirationTime === NoWork ||
        updateExpirationTime > renderExpirationTime)
) {
    // 先后两次props是一样的 && 本次没有更新或者更新的优先级不高这次更新不需要执行
    switch (workInProgress.tag) {
         ...
    }

    // 本次不需要更新，判断是否有需要更新的子树
    // 如果节点不更新但有需要更新的子树，将子树return出去
    return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderExpirationTime,
    );
}
```

判断是否需要更新子节点是在`bailoutOnAlreadyFinishedWork`中进行判断的。

```
const childExpirationTime = workInProgress.childExpirationTime;
if (
    childExpirationTime === NoWork ||
    childExpirationTime > renderExpirationTime
) {
    // 代表子树也没有本次更新需要执行的更新
    // 节点和子节点都不需要更新，相当于结束本次更新，直接return null
    return null;
} else {
    // 子树需要更新，复制子树，返回子树
    cloneChildFibers(current, workInProgress);
    return workInProgress.child;
}
```

继续回到`beginWork`方法。

如果本次需要更新，那么就根据`workInProgress.tag`来判断节点类型进行更新。具体的更新之后的文章再详细说明。


