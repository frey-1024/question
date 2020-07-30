# scheduler (16.10.0+) 任务调度器

### 背景

1. Improve queue performance by switching its internal data structure to a min binary heap. (https://github.com/facebook/react/pull/16214)
2. Use postMessage loop with short intervals instead of attempting to align to frame boundaries with requestAnimationFrame. (https://github.com/facebook/react/pull/16245)


### 说明

16.10.0 以前 是使用 `requestAnimationFrame` + `requestidlecallback的ployfill` 实现计算 下次渲染时间时，还剩多少时间，并按照优先级分配并执行任务。
新版将采用postMessage 形式通过派发的形式触发

```javascript
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;
```

### 整体介绍

#### unstable_scheduleCallback 方法

```javascript
// 任务队列， 缓存的数据格式相同
// 这个是没有delay 的任务队列
// 最后所有timerQueue经过handleTimeout 将进入到 taskQueue
var taskQueue = [];
// 这个是 需要延迟执行的任务队列
var timerQueue = [];

function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;

  // 根据权重 判断 执行时间
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }

  // 过期时间
  var expirationTime = startTime + timeout;

  // 新的任务数据
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  if (enableProfiling) {
    newTask.isQueued = false;
  }
  // 是否立即执行
  if (startTime > currentTime) {
    // This is a delayed task.
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    // 主任务已经执行完，并且当前的新任务是最优先级的
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      // setTimeout 延迟执行
      // handleTimeout 等到执行时，区分是继续延迟，还是加入到requestHostCallback中
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // 如果有任务正在执行，将先不执行
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      // 这里区分了服务端（nodejs）与客户端， 服务端情况下：setTimeout 0 执行， 也不是立刻执行，也是等Event loop 任务队列中 其他 微任务清理后，才执行
      // 客户端：port.postMessage(null); 直接执行
      // flushWork 是冲洗work, 主要是workLoop
      
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

##### 客户端与服务端requestHostCallback方法
```javascript
  // 客户端
  const channel = new MessageChannel();
  const port = channel.port2;
  channel.port1.onmessage = () => {
    if (scheduledHostCallback !== null) {
      const currentTime = getCurrentTime();
      // Yield after `yieldInterval` ms, regardless of where we are in the vsync
      // cycle. This means there's always time remaining at the beginning of
      // the message event.
      deadline = currentTime + yieldInterval;
      const hasTimeRemaining = true;
      try {
        const hasMoreWork = scheduledHostCallback(
          hasTimeRemaining,
          currentTime,
        );
        if (!hasMoreWork) {
          isMessageLoopRunning = false;
          scheduledHostCallback = null;
        } else {
          // If there's more work, schedule the next message event at the end
          // of the preceding one.
          port.postMessage(null);
        }
      } catch (error) {
        // If a scheduler task throws, exit the current browser task so the
        // error can be observed.
        port.postMessage(null);
        throw error;
      }
    } else {
      isMessageLoopRunning = false;
    }
    // Yielding to the browser will give it a chance to paint, so we can
    // reset this.
    needsPaint = false;
  };

  requestHostCallback = function(callback) {
    scheduledHostCallback = callback;
    if (!isMessageLoopRunning) {
      isMessageLoopRunning = true;
      port.postMessage(null);
    }
  };

  // 服务端
  requestHostCallback = function(cb) {
    if (_callback !== null) {
      // Protect against re-entrancy.
      setTimeout(requestHostCallback, 0, cb);
    } else {
      _callback = cb;
      setTimeout(_flushCallback, 0);
    }
  };
```

这里除了区分环境，其他没有区别， 都是在满足每个条件后，直接触发回调， 16.10.0 调整主要也是修改客户端，服务端基本不变


##### workLoop 方法

```javascript
function workLoop(hasTimeRemaining, initialTime) {
  let currentTime = initialTime;
  advanceTimers(currentTime);
  currentTask = peek(taskQueue);

  // 把所有能执行的taskQueue 都执行，并清除已经执行的数据
  while (
    currentTask !== null &&
    !(enableSchedulerDebugging && isSchedulerPaused)
  ) {
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      // This currentTask hasn't expired, and we've reached the deadline.
      break;
    }
    const callback = currentTask.callback;
    if (callback !== null) {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      markTaskRun(currentTask, currentTime);
      // 执行方法
      const continuationCallback = callback(didUserCallbackTimeout);
      currentTime = getCurrentTime();
      if (typeof continuationCallback === 'function') {
        currentTask.callback = continuationCallback;
        markTaskYield(currentTask, currentTime);
      } else {
        if (enableProfiling) {
          markTaskCompleted(currentTask, currentTime);
          currentTask.isQueued = false;
        }
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
      advanceTimers(currentTime);
    } else {
      pop(taskQueue);
    }
    currentTask = peek(taskQueue);
  }
  // Return whether there's additional work
  if (currentTask !== null) {
    return true;
  } else {
    let firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // timerQueue 没有清空，将从头再来
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}
```

##### 总结

scheduler (16.10.0+) 任务调度器 主要根据时间进行区分，是立刻执行还是等到 `startTime - currentTime` 再去执行， 

改良 通过实现优先级队列：二叉树最小堆， 方便存储、获取、删除任务队列。

requestHostCallback 区分了客户端和服务端， 客户端使用`MessageChannel`, 服务端使用 `setTimeout(cb, 0)`

