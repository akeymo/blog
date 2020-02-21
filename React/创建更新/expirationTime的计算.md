在讲`ReactDOM.render`时候提到的`updateContainer`这个方法中，计算了一个重要的值`expirationTime`，在这篇文章中，就详细讲一下`expirationTime`的计算。

在计算`expirationTime`之前，首先要利用`requestCurrentTime`计算一下JS加载完毕到当前时间的时间差。这中间涉及到`currentRendererTime`和`currentSchedulerTime`两个值。这两个值是为了减少消耗性能，避免频繁调用获取当前时间，而用来在记录值的，便于在不需要重新计算的场景中直接使用。

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