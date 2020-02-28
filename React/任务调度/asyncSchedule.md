异步调度的目的是，在浏览器每一帧的时间里，保证`React`执行更新的时间不超过一个特定的时限，保证留给浏览器足够的时间去刷新动画或响应用户输入反馈。

异步调度的作用：
1. 维护时间片
2. 模拟`requestIdleCallback`
3. 调度列表和超时判断

### scheduleCallbackWithExpirationTime
首先会判断是否存在上一次调用异步任务调度申请的一个`callback`的`expirationTime`，如果存在并且新的`expirationTime`的优先级比它高，取消掉旧的。

```
// callbackExpirationTime：上一次调用react schedule申请的一个callback的expirationTime
if (callbackExpirationTime !== NoWork) {
    // 表示已经有一个expirationTime在执行了
    // A callback is already scheduled. Check its expiration time (timeout).
    if (expirationTime > callbackExpirationTime) {
        // Existing callback has sufficient timeout. Exit.
        return;
    } else {
        if (callbackID !== null) {
            // Existing callback has insufficient timeout. Cancel and schedule a
            // new one.
            cancelDeferredCallback(callbackID);
        }
    }
    // The request callback timer is already running. Don't start a new one.
} else {
    startRequestCallbackTimer();
}
```

根据`currentMs`和`expirationTimeMs`去计算一个`timeout`，它表示从当前时间开始还有多久到过期时间。

最后调用`scheduleDeferredCallback`，将`performAsyncWork`作为回调函数`callback`、`timeout`作为参数传入。

```
callbackExpirationTime = expirationTime;
const currentMs = now() - originalStartTimeMs;
const expirationTimeMs = expirationTimeToMs(expirationTime);
const timeout = expirationTimeMs - currentMs;
callbackID = scheduleDeferredCallback(performAsyncWork, { timeout });
```

### scheduleDeferredCallback
`scheduleDeferredCallback`方法其实是`/packages/scheduler/src/Scheduler.js`里的`unstable_scheduleCallback`方法。

在我们讲述的这种情况下，方法内的`expirationTime`就是当前时间加上传入的`timeout`。

该方法会创建一个调度节点`newNode`，然后根据`expirationTime`的优先级将`newNode`添加进`callbackList`中。

然后调用`ensureHostCallbackIsScheduled`开始调度。

```
function unstable_scheduleCallback(callback, deprecated_options) {
    var startTime =
        currentEventStartTime !== -1 ? currentEventStartTime : getCurrentTime();

    var expirationTime;
    if (
        typeof deprecated_options === 'object' &&
        deprecated_options !== null &&
        typeof deprecated_options.timeout === 'number'
    ) {
        // FIXME: Remove this branch once we lift expiration times out of React.
        expirationTime = startTime + deprecated_options.timeout;
    } else {
        ...
    }

    var newNode = {
        callback,
        priorityLevel: currentPriorityLevel,
        expirationTime,
        next: null,
        previous: null,
    };

    // Insert the new callback into the list, ordered first by expiration, then
    // by insertion. So the new callback is inserted any other callback with
    // equal expiration.
    if (firstCallbackNode === null) {
        // This is the first callback in the list.
        firstCallbackNode = newNode.next = newNode.previous = newNode;
        ensureHostCallbackIsScheduled();
    } else {
        var next = null;
        var node = firstCallbackNode;
        do {
            if (node.expirationTime > expirationTime) {
                // The new callback expires before this one.
                next = node;
                break;
            }
            node = node.next;
        } while (node !== firstCallbackNode);

        if (next === null) {
            // No callback with a later expiration was found, which means the new
            // callback has the latest expiration in the list.
            // 没有任何一个任务的expirationTime大于当前的expirationTime
            next = firstCallbackNode;
        } else if (next === firstCallbackNode) {
            // The new callback has the earliest expiration in the entire list.
            firstCallbackNode = newNode;
            ensureHostCallbackIsScheduled();
        }

        var previous = next.previous;
        previous.next = next.previous = newNode;
        newNode.next = next;
        newNode.previous = previous;
    }

    return newNode;
}
```

### ensureHostCallbackIsScheduled
该方法会在正式开始调度前，先判断处理是否已经在调用回调的情况，或者是已经开始调度的情况。

已经在调用回调的情况下，直接退出。

如果是开始调度的情况下，则将正在调度的取消，因为此时顺序可能已经发生了改变。

```
function ensureHostCallbackIsScheduled() {
  if (isExecutingCallback) {
    // 已经在调用回调的情况，直接return
    // Don't schedule work yet; wait until the next time we yield.
    return;
  }
  // Schedule the host callback using the earliest expiration in the list.
  var expirationTime = firstCallbackNode.expirationTime;
  if (!isHostCallbackScheduled) {
    // isHostCallbackScheduled：false表示还没开始调度
    isHostCallbackScheduled = true;
  } else {
    // Cancel the existing host callback.
    // 已经开始调度，直接取消，因为顺序可能发生改变
    cancelHostCallback();
  }
  // 开始调度
  requestHostCallback(flushWork, expirationTime);
}
```

### requestHostCallback

`scheduledHostCallback`是全局变量，记录传入的`callback`也就是`flushWork`。

`timeoutTime`也是全局变量，记录对应的过期时间。

`isFlushingHostCallback`表示是否正在执行`callback`。

`isAnimationFrameScheduled`表示是否已经开始调用`requestAnimationFrame`。

```
requestHostCallback = function (callback, absoluteTimeout) {
    // 全局变量，用于记录回调函数。firstCallback里面对应的callback
    scheduledHostCallback = callback;
    // 全局变量，用于记录对应的过期时间。firstCallback对应的expirationTime
    timeoutTime = absoluteTimeout;
    if (isFlushingHostCallback || absoluteTimeout < 0) {
        // Don't wait for the next frame. Continue working ASAP, in a new event.
        window.postMessage(messageKey, '*');
    } else if (!isAnimationFrameScheduled) {
        isAnimationFrameScheduled = true;
        requestAnimationFrameWithTimeout(animationTick);
    }
};
```

### 模拟requestIdleCallback
`requestIdleCallback`方法将在浏览器的空闲时段内调用的函数排队。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间timeout，则有可能为了在超时前执行函数而打乱执行顺序。

但因为这个方法兼容性不高，因此，在异步调度过程中，`React`自己实现了一个`polyfill`。

`requestAnimationFrame`方法要求浏览器在下次重绘之前调用指定的回调函数更新动画，因此，利用`requestAnimationFrame`在浏览器渲染一帧之前做一些处理，通过`postMessage`在`macro task`中加入一个回调。因为接下去会进入浏览器的渲染阶段，这个阶段主线程是被block的，等到渲染完了再回来清空`macro task`。

### animationTick
作为`requestAnimationFrameWithTimeout`里调用的的`callback`，`animationTick`方法的作用，一个是只要`scheduledHostCallback`还在就继续调用`requestAnimationFrameWithTimeout`，因为可能这一帧渲染完了还有队列没有清空；一个是判断当前平台的刷新频率以计算出`frameDeadline`，这个值之后用来判断是否还有空余的帧时间；最后一个是如果没有触发`idleTick`则发送消息。

```
var animationTick = function (rafTime) {
    // 全局变量，用于记录回调函数
    if (scheduledHostCallback !== null) {
        // 一帧渲染完了但队列可能还没清空
        requestAnimationFrameWithTimeout(animationTick);
    } else {
        // No pending work. Exit.
        isAnimationFrameScheduled = false;
        return;
    }

    // 到下一帧可以执行这个方法的时间
    // activeFrameTime是一帧的完整时间，初始是33
    // frameDeadline初始是0
    var nextFrameTime = rafTime - frameDeadline + activeFrameTime;
    // 通过前后几次帧时间的判断来判断当前平台的刷新频率，更新activeFrameTime
    // 达到react减少自己运行时间的目的
    if (
        nextFrameTime < activeFrameTime &&
        previousFrameTime < activeFrameTime // previousFrameTime初始是33
    ) {
        if (nextFrameTime < 8) {
            // 目前不支持帧率大于120的情况
            nextFrameTime = 8;
        }

        activeFrameTime =
            nextFrameTime < previousFrameTime ? previousFrameTime : nextFrameTime;
    } else {
        previousFrameTime = nextFrameTime;
    }
    frameDeadline = rafTime + activeFrameTime;

    if (!isMessageEventScheduled) {
        isMessageEventScheduled = true;
        // postMessage的接收方会等到浏览器执行完自己的任务再执行
        // 此时一帧时间已经消耗掉一部分了
        // 所以frameDeadline的值会加activeFrameTime
        window.postMessage(messageKey, '*');
    }
};
```

### idleTick
`Reack`会设置一个监听，监听是否触发了`message`。

```
window.addEventListener('message', idleTick, false);
```

方法内，首先判断是否是自己的`postMessage`。

```
if (event.source !== window || event.data !== messageKey) {
    return;
}
```

清空`scheduledHostCallback`和`timeoutTime`。

```
var prevScheduledCallback = scheduledHostCallback;
var prevTimeoutTime = timeoutTime;
scheduledHostCallback = null;
timeoutTime = -1;
```

获取当前时间，和`frameDeadline`做对比。

如果已经消耗完一帧时间，判断是否到过期时间，如果已经到过期时间了，要强制输入，将`didTimeout`置为`true`；如果没有到过期时间，就重新调度。

```
if (frameDeadline - currentTime <= 0) {
    // 浏览器已经消耗掉所有的一帧时间
    if (prevTimeoutTime !== -1 && prevTimeoutTime <= currentTime) {
        // 任务已经过期了，要强制输出
        didTimeout = true;
    } else {
        // No timeout.
        if (!isAnimationFrameScheduled) {
            // Schedule another animation callback so we retry later.
            isAnimationFrameScheduled = true;
            requestAnimationFrameWithTimeout(animationTick);
        }
        // Exit without invoking the callback.
        scheduledHostCallback = prevScheduledCallback;
        timeoutTime = prevTimeoutTime;
        return;
    }
}
```

调用`callback`，`isFlushingHostCallback`置为`true`表示正在执行。调用`prevScheduledCallback`即传入的`flushWork`并传入`didTimeout`。

```
if (prevScheduledCallback !== null) {
    isFlushingHostCallback = true;
    try {
        // scheduledHostCallback即callback即flushWork
        prevScheduledCallback(didTimeout);
    } finally {
        isFlushingHostCallback = false;
    }
}
```

总而言之，`idleTick`的主要作用就是，计算`didTimeout`，根据`frameDeadline`去判断是否又剩余的一帧时间，如果没有了，根据是否过期去设置`didTimeout`和安排重新调度；然后调用`flushWork`传入`didTimeout`。完整代码如下：

```
var idleTick = function (event) {
    if (event.source !== window || event.data !== messageKey) {
        return;
    }

    isMessageEventScheduled = false;

    // 清空scheduledHostCallback和timeoutTime
    var prevScheduledCallback = scheduledHostCallback;
    var prevTimeoutTime = timeoutTime;
    scheduledHostCallback = null;
    timeoutTime = -1;

    var currentTime = getCurrentTime();

    var didTimeout = false;
    if (frameDeadline - currentTime <= 0) {
        // 浏览器已经消耗掉所有的一帧时间
        if (prevTimeoutTime !== -1 && prevTimeoutTime <= currentTime) {
            // 任务已经过期了，要强制输出
            didTimeout = true;
        } else {
            // No timeout.
            if (!isAnimationFrameScheduled) {
                // Schedule another animation callback so we retry later.
                isAnimationFrameScheduled = true;
                requestAnimationFrameWithTimeout(animationTick);
            }
            // Exit without invoking the callback.
            scheduledHostCallback = prevScheduledCallback;
            timeoutTime = prevTimeoutTime;
            return;
        }
    }

    if (prevScheduledCallback !== null) {
        isFlushingHostCallback = true;
        try {
            // scheduledHostCallback即callback即flushWork
            prevScheduledCallback(didTimeout);
        } finally {
            isFlushingHostCallback = false;
        }
    }
};
```

### flushWork

`flushWork`的作用，一个是在已经超时的情况下，把`callbackList`里过期的任务都执行了；另一个是在帧时间有空余的情况下，把执行`callbackList`里的任务；最后清理变量，如果`callbackList`未执行完继续进入调度，并且在退出前把`immedia`优先级的任务都调用一遍。

```
function flushWork(didTimeout) {
  isExecutingCallback = true;
  deadlineObject.didTimeout = didTimeout;
  try {
    if (didTimeout) {
      // Flush all the expired callbacks without yielding.
      while (firstCallbackNode !== null) {
        var currentTime = getCurrentTime();
        if (firstCallbackNode.expirationTime <= currentTime) {
          // 将callbackList里的过期任务都强制输出
          do {
            flushFirstCallback();
          } while (
            firstCallbackNode !== null &&
            firstCallbackNode.expirationTime <= currentTime
          );
          continue;
        }
        break;
      }
    } else {
      // Keep flushing callbacks until we run out of time in the frame.
      if (firstCallbackNode !== null) {
        // 帧时间有空余的情况下并且callbackList执行完之前，执行list里的任务
        do {
          flushFirstCallback();
        } while (
          firstCallbackNode !== null &&
          getFrameDeadline() - getCurrentTime() > 0
        );
      }
    }
  } finally {
    isExecutingCallback = false;
    if (firstCallbackNode !== null) {
      // There's still work remaining. Request another callback.
      ensureHostCallbackIsScheduled();
    } else {
      isHostCallbackScheduled = false;
    }
    // Before exiting, flush all the immediate work that was scheduled.
    flushImmediateWork();
  }
}
```

### flushFirstCallback

`flushFirstCallback`的作用，其实就是执行掉`firstCallbackNode`。

首先判断如果队列只剩一个任务，则清空队列。否则就进行移除第一个任务的处理。

```
if (firstCallbackNode === next) {
    // This is the last callback in the list.
    firstCallbackNode = null;
    next = null;
} else {
    // 移除第一个，将第二个变为第一个
    var lastCallbackNode = firstCallbackNode.previous;
    firstCallbackNode = lastCallbackNode.next = next;
    next.previous = lastCallbackNode;
}

// 清空指针
flushedNode.next = flushedNode.previous = null;
```

接下来就是调用`callback`，这个`callback`就是挂在`firstCallbackNode`上的`callback`，在异步调度的情况下就是`performAsyncWork`。

```
var callback = flushedNode.callback; // performAsyncWork
var expirationTime = flushedNode.expirationTime;
var priorityLevel = flushedNode.priorityLevel;
var previousPriorityLevel = currentPriorityLevel;
var previousExpirationTime = currentExpirationTime;
currentPriorityLevel = priorityLevel;
currentExpirationTime = expirationTime;
var continuationCallback;
try {
    // callback: performAsyncWork
    continuationCallback = callback(deadlineObject);
} finally {
    currentPriorityLevel = previousPriorityLevel;
    currentExpirationTime = previousExpirationTime;
}
```

最后，如果调用的回调有返回内容，就把返回的内容重新加入回调队列中，逻辑基本和`unstable_scheduleCallback`中将创建的`newNode`添加进回调队列中一样。

