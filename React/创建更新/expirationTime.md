所有的节点都会具有`expirationTime`和`childExpirationTime`两个属性。

### expirationTime

`expirationTime`的值有三种：
1. `NoWork`也就是0：初始值，表示没有更新。
2. `Sync`：同步模式，优先级最高，更新创建完成就要立马更新到真正的DOM里
3. 异步模式下计算出来的过期时间，该模式下任务可能被中断。

异步模式下，`expirationTime`的计算过程是这样的。

在讲`ReactDOM.render`时候提到的`updateContainer`这个方法中，计算了一个重要的值`expirationTime`。

首先，要利用`requestCurrentTime`计算一下JS加载完毕到当前时间的时间差。这中间涉及到`currentRendererTime`和`currentSchedulerTime`两个值。这两个值是为了减少消耗性能，避免频繁调用获取当前时间，而用来在记录值的，便于在不需要重新计算的场景中直接使用。

首先看：

```
if (isRendering) {
    // We're already rendering. Return the most recently read time.
    return currentSchedulerTime;
}
```

在一次渲染过程中，如果再次调用了`setState`或者任务挂起的时候，新发起了一次`requestCurrentTime`，直接返回了目前的`renderTime`。
这说明，`React`保证了在一次渲染过程中新产生的`update`的`currentTime`必须跟目前的`renderTime`保持一致，因此也保证了这个周期下所有产生的更新的过期时间都会保持一致。

之后的一段逻辑是：
```
if (
    nextFlushedExpirationTime === NoWork ||
    nextFlushedExpirationTime === Never
  ) {
    recomputeCurrentRendererTime();
    currentSchedulerTime = currentRendererTime;
    return currentSchedulerTime;
}
```

这段逻辑就是在第一次创建更新的时候计算`currentRendererTime`。

`currentTime`计算好之后，就是计算`expirationTime`了。计算这个过期时间根据条件不同会分别调用`computeInteractiveExpiration`和`computeAsyncExpiration`两个方法。两个方法都是会调用`computeExpirationBucket`，只是传入的`expirationInMs`和`bucketSizeMs`不同。
因为`computeInteractiveExpiration`涉及到交互，所以它的响应优先级会比较高。

```
const UNIT_SIZE = 10
const MAGIC_NUMBER_OFFSET = 2

export function msToExpirationTime(ms: number): ExpirationTime {
  return ((ms / UNIT_SIZE) | 0) + MAGIC_NUMBER_OFFSET
}

export function expirationTimeToMs(expirationTime: ExpirationTime): number {
  return (expirationTime - MAGIC_NUMBER_OFFSET) * UNIT_SIZE
}

function ceiling(num: number, precision: number): number {
  return (((num / precision) | 0) + 1) * precision
}

function computeExpirationBucket(
  currentTime,
  expirationInMs,
  bucketSizeMs,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET +
    ceiling(
      currentTime - MAGIC_NUMBER_OFFSET + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  )
}

export const LOW_PRIORITY_EXPIRATION = 5000
export const LOW_PRIORITY_BATCH_SIZE = 250

export function computeAsyncExpiration(
  currentTime: ExpirationTime,
): ExpirationTime {
  return computeExpirationBucket(
    currentTime,
    LOW_PRIORITY_EXPIRATION,
    LOW_PRIORITY_BATCH_SIZE,
  )
}

export const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150
export const HIGH_PRIORITY_BATCH_SIZE = 100

export function computeInteractiveExpiration(currentTime: ExpirationTime) {
  return computeExpirationBucket(
    currentTime,
    HIGH_PRIORITY_EXPIRATION,
    HIGH_PRIORITY_BATCH_SIZE,
  )
}
```

以`computeAsyncExpiration`举例，最终的计算公式是`((((currentTime - 2 + 5000 / 10) / 25) | 0) + 1) * 25`。
`| 0`的作用是用来取整。
因为`currentTime`是经过`msToExpirationTime`处理的，处理公式是`((now / 10) | 0) + 2`，所以`-2`可以忽略，而`(now / 10) | 0`是为了抹平10毫秒内的误差。
再回到最终计算公式上，`/ 25 | 0`就是为了保证在25为单位的时间内，得到的`expirationTime`是一致的。其实就是为了保证两次时间很接近的更新得到相同的`expirationTime`。


### childExpirationTime

每一次节点调用`setState`或者`forceUpdate`都会产生一个更新并且计算`expirationTime`。因为更新需要从`fiberRoot`开始，所以会执行一次向上遍历找到`fiberRoot`。这个过程中会对每一个该节点的父节点链上的节点设置`childExpirationTIme`。

这个值的作用就是，在向下更新整棵`fiber`树的时候，会根据节点本身的`expirationTime`和`childExpirationTime`来判断这个节点是否需要直接跳过不更新子树。

`expirationTime`代表他本身是否有更新，如果他本身有更新，那么他的更新可能会影响子树；`childExpirationTime`表示他的子树是否产生了更新；如果两个都没有，那么子树是不需要更新的。

在每个节点`completeWork`的时候会重置父链上的`childExpirationTime`，也就是说这个节点已经更新完了，那么他的`childExpirationTime`也就没有意义了。

`childExpirationTime`的特性：
1. 同一个节点产生的连续两次更新，最终在父节点上只会体现一次`childExpirationTime`。因为同一个节点在第一次更新还没有结束的情况下再次产生更新，那么不管他们优先级哪个高，最终都会按照高优先级那个过期时间把所有更新都更新掉了。
2. 不同子树产生的更新，最终体现在跟节点上的是优先级最高的那个更新。因为`React`在创建更新向上寻找`root`并设置`childExpirationTime`的时候，会对比之前设置过的和现在的，最终会等于非`NoWork`的最小的`childExpirationTime`，因为`expirationTime`越小优先级越高，`Sync`是最高优先级。