### ensureRootIsScheduled

当expirationTime !== Sync ，也就是异步调度的时候我们会执行这个函数，接下来看下它的源码：

```javascript
// packages/react-reconciler/src/ReactFiberWorkLoop.js
// 每一个root都有一个唯一的调度任务，如果已经存在，我们要确保到期时间与下一级别任务的相同，每一次更新都会调用这个方法
function ensureRootIsScheduled(root: FiberRoot) {
  const lastExpiredTime = root.lastExpiredTime;
  // 最近的优先级不为NoWork，设置为最高优先级，也就是上个任务已经过期
  if (lastExpiredTime !== NoWork) {
    // Special case: Expired work should flush synchronously.
    root.callbackExpirationTime = Sync;
    root.callbackPriority = ImmediatePriority;
    root.callbackNode = scheduleSyncCallback(
      performSyncWorkOnRoot.bind(null, root),
    );
    return;
  }
	// 获取下一个expirationTime
  const expirationTime = getNextRootExpirationTimeToWorkOn(root);
  const existingCallbackNode = root.callbackNode;
  // 现在没有work 调度
  if (expirationTime === NoWork) {
    // There's nothing to work on.
    if (existingCallbackNode !== null) {
      // 那么重置root 的属性
      root.callbackNode = null;
      root.callbackExpirationTime = NoWork;
      root.callbackPriority = NoPriority;
    }
    return;
  }

  // TODO: If this is an update, we already read the current time. Pass the
  // time as an argument.
  const currentTime = requestCurrentTimeForUpdate();
  const priorityLevel = inferPriorityFromExpirationTime(
    currentTime,
    expirationTime,
  );

  // If there's an existing render task, confirm it has the correct priority and
  // expiration time. Otherwise, we'll cancel it and schedule a new one.
  // 如果上一个callback 还存在
  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority;
    const existingCallbackExpirationTime = root.callbackExpirationTime;
    if (
      // Callback must have the exact same expiration time.
      existingCallbackExpirationTime === expirationTime &&
      // Callback must have greater or equal priority.
      existingCallbackPriority >= priorityLevel
    ) {
      // Existing callback is sufficient.
      return;
    }
    // Need to schedule a new task.
    // TODO: Instead of scheduling a new task, we should be able to change the
    // priority of the existing one.
    // 并且优先级低于当前，那么久取消callback
    cancelCallback(existingCallbackNode);
  }
	// 这里的情况就是当前优先级更高，所以覆盖老的root 属性
  root.callbackExpirationTime = expirationTime;
  root.callbackPriority = priorityLevel;

  let callbackNode;
  if (expirationTime === Sync) {
    // 同步更新，调用scheduleSyncCallback
    // Sync React callbacks are scheduled on a special internal queue
    callbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
  } else if (disableSchedulerTimeoutBasedOnReactExpirationTime) {
    callbackNode = scheduleCallback(
      priorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
    );
  } else {
    // 真正的异步
    callbackNode = scheduleCallback(
      priorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
      // Compute a task timeout based on the expiration time. This also affects
      // ordering because tasks are processed in timeout order.
      {timeout: expirationTimeToMs(expirationTime) - now()},
    );
  }
	// 赋值为最新的
  root.callbackNode = callbackNode;
}
```

看下其中的逻辑，首先获取了`root.lastExpiredTime`,也就是最近的一次更新的，如果它不为`NoWork`,那么说明该任务已经过期，所以同步执行，调用`scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root))`，并返回。如果当前没有任务调度，那么callback等属性会赋值到root 上，并返回。如果当前有任务调度并且上一个callback还存在，那么比较优先级，如果上一个callback 的优先级更高，那么return。如果当前优先级更高，那么就调用`cancelCallback(existingCallbackNode)`,取消callback。然后开启一个新的callback,并且将新的值赋值到root 上，在这个过程中，如果是同步的，那么调用`scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root))`,如果是异步的，那么调用的是`scheduleCallback`函数。

总结一下，`ensureRootIsScheduled()`这个函数会去判断上一次的callback，是否还存在，并且判断于当前的优先级，如果现在优先级更高，就会取消上一次的callback,然后产生新的callback,在同步的情况下，调用`scheduleSyncCallback`,异步的情况下调用`scheduleCallback`，另外需要注意在一个root 上，默认就只能有一个任务。