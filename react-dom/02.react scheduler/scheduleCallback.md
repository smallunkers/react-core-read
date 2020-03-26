### scheduleCallback && scheduleSyncCallback

在之前的时候，我们讲到过在异步的情况下，会调用`scheduleCallback`这个函数，在同步的情况下，会调用`scheduleSyncCallback`这个函数,我们分别来看下。

先看下`scheduleCallback`，让我们看下这个函数时如何工作的,这个函数实际上调用了`Scheduler_scheduleCallback`。

然后看下`scheduleSyncCallback`，如果队列不为空就把callback加入队列，如果为空就立即推入任务调度队列。同样的也调用`Scheduler_scheduleCallback`

这个函数其实就是是`unstable_scheduleCallback`，其实就是直接将异步任务推入调度队列

```javascript
// packages/react-reconciler/src/SchedulerWithReactIntegration.js
export function scheduleCallback(
  reactPriorityLevel: ReactPriorityLevel,
  callback: SchedulerCallback,
  options: SchedulerCallbackOptions | void | null,
) {
  const priorityLevel = reactPriorityToSchedulerPriority(reactPriorityLevel);
  // 异步任务推入调度队列
  return Scheduler_scheduleCallback(priorityLevel, callback, options);
}
// packages/react-reconciler/src/SchedulerWithReactIntegration.js
export function scheduleSyncCallback(callback: SchedulerCallback) {
  // Push this callback into an internal queue. We'll flush these either in
  // the next tick, or earlier if something calls `flushSyncCallbackQueue`.
  if (syncQueue === null) {
    // 该队列为空，那么赋值
    syncQueue = [callback];
    // Flush the queue in the next tick, at the earliest.
    // 如果为空就立即推入任务调度队列
    immediateQueueCallbackNode = Scheduler_scheduleCallback(
      Scheduler_ImmediatePriority,
      flushSyncCallbackQueueImpl,
    );
  } else {
    // Push onto existing queue. Don't need to schedule a callback because
    // we already scheduled one when we created the queue.
    syncQueue.push(callback);
  }
  return fakeCallbackNode;
}

// packages/scheduler/src/Scheduler.js
function unstable_scheduleCallback(priorityLevel, callback, options) {
  // 当前时间减去页面加载记录的时间
  var currentTime = getCurrentTime();

  var startTime;
  var timeout;
  if (typeof options === 'object' && options !== null) {
    var delay = options.delay;
    if (typeof delay === 'number' && delay > 0) {
      startTime = currentTime + delay;
    } else {
      startTime = currentTime;
    }
    // 如果没有timeout就是使用优先级计算出来的
    timeout =
      typeof options.timeout === 'number'
        ? options.timeout
        : timeoutForPriorityLevel(priorityLevel);
  } else {
    // 针对不同的优先级算出不同的过期时间
    timeout = timeoutForPriorityLevel(priorityLevel);
    startTime = currentTime;
  }
	// 定义新的过期时间
  var expirationTime = startTime + timeout;
	// 创建一个新的任务
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

  if (startTime > currentTime) {
    // This is a delayed task.
    // 这是一个delay 的任务
    newTask.sortIndex = startTime; 
    push(timerQueue, newTask); // 放入一个堆中，并且执行最大堆操作
    // 如果任务队列为空且新增加的任务是优先级最高
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      // All tasks are delayed, and this is the task with the earliest delay.
      if (isHostTimeoutScheduled) {
        // Cancel an existing timeout.
        cancelHostTimeout();
      } else {
        isHostTimeoutScheduled = true;
      }
      // Schedule a timeout.
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 将新的任务推入任务队列
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    if (enableProfiling) {
      markTaskStart(newTask, currentTime);
      newTask.isQueued = true;
    }
    // Schedule a host callback, if needed. If we're already performing work,
    // wait until the next time we yield.
    // 执行回调方法，如果已经再工作需要等待一次回调的完成
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }

  return newTask;
}
```

目前来说，好像还没有什么地方看到是delay 的task, 那么实际上，我们在这个方法里执行的就是`requestHostCallback(flushWork)`这个函数。