---
layout: post
title: Static Allocation Recover in Spark on k8s
date: 2017-10-26 11:54:31
category: 技术
tags: Kubernetes Docker Spark
excerpt: Spark on k8s executors静态恢复机制……
---

本文分析[Spark on k8s项目](https://github.com/apache-spark-on-k8s/spark)静态恢复机制也即SAR（相关源码可能不是最新，仅供参考）

## 一、前言

`spark on k8s`项目SAR并不健壮和完善，主要体现在如下方面（参考[这里](https://docs.google.com/document/d/1GX__jsCbeCw4RrUpHLqtpAzHwV82NQrgjz1dCCqqRes/edit#))：

* 1、如果`executor`挂掉，则driver不会产生新的 替换`executor`
* 2、如果`executor`全部挂掉，driver依然运行——bug（原因待查……）
* 3、`executor`产生过程存在问题——如果某些`executor`因为某些原因始终无法起来，则`driver`不会继续产生新的`executor`

最终想要的结果是在spark应用程序正确的情况下，始终产生并维持用户指定数目的`executor`数目，并采用某种机制保障（检测）运行的`executor`是`"健康状态"`，最终保证应用程序的成功运行

## 现有方案解析

### pod error(do not delete)


### pod disconnected(delete)

* `log`：

```
2017-09-22 08:20:20 ERROR KubernetesTaskSchedulerImpl: Lost executor 59 on 192.168.23.56: Executor heartbeat timed out after 162451 ms
2017-09-22 08:20:53 ERROR KubernetesTaskSchedulerImpl: Lost executor 59 on 192.168.23.56: Remote RPC client disassociated. Likely due to containers exceeding thresholds, or network issues. Check driver logs for WARN messages.
```

* `分析`：

`HeartbeatReceiver.scala`:

```scala
  override def onStart(): Unit = {
    timeoutCheckingTask = eventLoopThread.scheduleAtFixedRate(new Runnable {
      override def run(): Unit = Utils.tryLogNonFatalError {
        Option(self).foreach(_.ask[Boolean](ExpireDeadHosts))
      }
    }, 0, checkTimeoutIntervalMs, TimeUnit.MILLISECONDS)
  }
  private def expireDeadHosts(): Unit = {
    logTrace("Checking for hosts with no recent heartbeats in HeartbeatReceiver.")
    val now = clock.getTimeMillis()
    for ((executorId, lastSeenMs) <- executorLastSeen) {
      if (now - lastSeenMs > executorTimeoutMs) {
        logWarning(s"Removing executor $executorId with no recent heartbeats: " +
          s"${now - lastSeenMs} ms exceeds timeout $executorTimeoutMs ms")
        scheduler.executorLost(executorId, SlaveLost("Executor heartbeat " +
          s"timed out after ${now - lastSeenMs} ms"))
          // Asynchronously kill the executor to avoid blocking the current thread
        killExecutorThread.submit(new Runnable {
          override def run(): Unit = Utils.tryLogNonFatalError {
            // Note: we want to get an executor back after expiring this one,
            // so do not simply call `sc.killExecutor` here (SPARK-8119)
            sc.killAndReplaceExecutor(executorId)
          }
        })
        executorLastSeen.remove(executorId)
      }
    }
  }
  // "spark.network.timeout" uses "seconds", while `spark.storage.blockManagerSlaveTimeoutMs` uses
  // "milliseconds"
  private val slaveTimeoutMs =
    sc.conf.getTimeAsMs("spark.storage.blockManagerSlaveTimeoutMs", "120s")
  private val executorTimeoutMs =
    sc.conf.getTimeAsSeconds("spark.network.timeout", s"${slaveTimeoutMs}ms") * 1000
```

`SparkContext.scala`：

```scala
  /**
   * Request that the cluster manager kill the specified executor without adjusting the
   * application resource requirements.
   *
   * The effect is that a new executor will be launched in place of the one killed by
   * this request. This assumes the cluster manager will automatically and eventually
   * fulfill all missing application resource requests.
   *
   * @note The replace is by no means guaranteed; another application on the same cluster
   * can steal the window of opportunity and acquire this application's resources in the
   * mean time.
   *
   * @return whether the request is received.
   */
  private[spark] def killAndReplaceExecutor(executorId: String): Boolean = {
    schedulerBackend match {
      case b: CoarseGrainedSchedulerBackend =>
        b.killExecutors(Seq(executorId), replace = true, force = true).nonEmpty
      case _ =>
        logWarning("Killing executors is only supported in coarse-grained mode")
        false
    }
  }
```

`CoarseGrainedSchedulerBackend.scala`：

```scala
  /**
   * Request that the cluster manager kill the specified executors.
   *
   * When asking the executor to be replaced, the executor loss is considered a failure, and
   * killed tasks that are running on the executor will count towards the failure limits. If no
   * replacement is being requested, then the tasks will not count towards the limit.
   *
   * @param executorIds identifiers of executors to kill
   * @param replace whether to replace the killed executors with new ones
   * @param force whether to force kill busy executors
   * @return whether the kill request is acknowledged. If list to kill is empty, it will return
   *         false.
   */
  final def killExecutors(
      executorIds: Seq[String],
      replace: Boolean,
      force: Boolean): Seq[String] = {
    logInfo(s"Requesting to kill executor(s) ${executorIds.mkString(", ")}")
    val response = synchronized {
      val (knownExecutors, unknownExecutors) = executorIds.partition(executorDataMap.contains)
      unknownExecutors.foreach { id =>
        logWarning(s"Executor to kill $id does not exist!")
      }
      // If an executor is already pending to be removed, do not kill it again (SPARK-9795)
      // If this executor is busy, do not kill it unless we are told to force kill it (SPARK-9552)
      val executorsToKill = knownExecutors
        .filter { id => !executorsPendingToRemove.contains(id) }
        .filter { id => force || !scheduler.isExecutorBusy(id) }
      executorsToKill.foreach { id => executorsPendingToRemove(id) = !replace }
      logInfo(s"Actual list of executor(s) to be killed is ${executorsToKill.mkString(", ")}")
      // If we do not wish to replace the executors we kill, sync the target number of executors
      // with the cluster manager to avoid allocating new ones. When computing the new target,
      // take into account executors that are pending to be added or removed.
      val adjustTotalExecutors =
        if (!replace) {
          doRequestTotalExecutors(
            numExistingExecutors + numPendingExecutors - executorsPendingToRemove.size)
        } else {
          numPendingExecutors += knownExecutors.size
          Future.successful(true)
        }
      val killExecutors: Boolean => Future[Boolean] =
        if (!executorsToKill.isEmpty) {
          _ => doKillExecutors(executorsToKill)
        } else {
          _ => Future.successful(false)
        }
      val killResponse = adjustTotalExecutors.flatMap(killExecutors)(ThreadUtils.sameThread)
      killResponse.flatMap(killSuccessful =>
        Future.successful (if (killSuccessful) executorsToKill else Seq.empty[String])
      )(ThreadUtils.sameThread)
    }
    defaultAskTimeout.awaitResult(response)
  }
  override def doRequestTotalExecutors(requestedTotal: Int): Future[Boolean] = Future[Boolean] {
    totalExpectedExecutors.set(requestedTotal)
    true
  }
  override def doKillExecutors(executorIds: Seq[String]): Future[Boolean] = Future[Boolean] {
    RUNNING_EXECUTOR_PODS_LOCK.synchronized {
      for (executor <- executorIds) {
        runningExecutorPods.remove(executor) match {
          case Some(pod) => kubernetesClient.pods().delete(pod)
          case None => logWarning(s"Unable to remove pod for unknown executor $executor")
        }
      }
    }
    true
  }
  // Executors we have requested the cluster manager to kill that have not died yet; maps
  // the executor ID to whether it was explicitly killed by the driver (and thus shouldn't
  // be considered an app-related failure).
  @GuardedBy("CoarseGrainedSchedulerBackend.this")
  private val executorsPendingToRemove = new HashMap[String, Boolean]
  // Number of executors requested from the cluster manager that have not registered yet
  @GuardedBy("CoarseGrainedSchedulerBackend.this")
  private var numPendingExecutors = 0
```

## 解决方案调研

官方有开源解决方案[Changes to support executor recovery behavior during static allocation.](https://github.com/apache-spark-on-k8s/spark/pull/244)，基本是参考`spark on yarn`的静态恢复方案

## 改进方案解析

>>
What changes were proposed in this pull request?
>>
Added initial support for driver to ask for more executors in case of framework faults.
>>
Reviewer notes:
This is WIP and currently being tested. Seems to work for simple smoke-tests. Looking for feedback on
>>
* Any major blindspots in logic or functionality
* General flow. Potential issues with control/data flows.
* Are style guidelines followed.
>>
Potential issues/Todos:
>>
* Verify that no deadlocks are possible.
* May be explore message passing between threads instead of using synchronization
* Any uncovered issues in further testing
>>
Reviewer notes
>>
Main business logic is in
removeFailedAndRequestNewExecutors()
>>
Overall executor recovery logic at a high-level:
>>
* On executor disconnect, we immediately disable the executor.
* Delete/Error Watcher actions will trigger a capture of executor loss reasons. This happens on a separate thread.
* There is another dedicated recovery thread, which looks at all previously disconnected executors and their loss reasons to remove those executors with the right loss reasons or keep trying till the loss reasons' are discovered. If the loss reason of a lost executors is not discovered within a sufficient time window, then we give up and still remove the executor. For all removed executors, we request new executors on this recovery thread.
>>
How was this patch tested?
>>
Manually tested that on deleting a pod, new pods were being requested.

参考如下三个Commits（`SAR`）：

* [Changes to support executor recovery behavior during static allocation.](https://github.com/apache-spark-on-k8s/spark/pull/244/files)
* [Unit Tests for KubernetesClusterSchedulerBackend](https://github.com/apache-spark-on-k8s/spark/pull/459/files)
* [Code enhancement: Replaced explicit synchronized access to a hashmap with a concurrent map.](https://github.com/apache-spark-on-k8s/spark/pull/392/files)

下面从源码分析`SAR`解决方案：

数据结构：

```scala
  private val RUNNING_EXECUTOR_PODS_LOCK = new Object

  // Indexed by executor IDs and guarded by RUNNING_EXECUTOR_PODS_LOCK.
  private val runningExecutorsToPods = new mutable.HashMap[String, Pod]
  // Indexed by executor pod names and guarded by RUNNING_EXECUTOR_PODS_LOCK.
  private val runningPodsToExecutors = new mutable.HashMap[String, String] 
  // TODO(varun): Get rid of this lock object by my making the underlying map a concurrent hash map.
  private val EXECUTOR_PODS_BY_IPS_LOCK = new Object
  // Indexed by executor IP addrs and guarded by EXECUTOR_PODS_BY_IPS_LOCK
  private val executorPodsByIPs = new mutable.HashMap[String, Pod]
  private val podsWithKnownExitReasons: concurrent.Map[String, ExecutorExited] =
    new ConcurrentHashMap[String, ExecutorExited]().asScala
  private val disconnectedPodsByExecutorIdPendingRemoval =
    new ConcurrentHashMap[String, Pod]().asScala
```

`allocatorRunnable`线程执行具体分配`executor`逻辑（核心函数）：

```scala
override def start(): Unit = {
  super.start()
  executorWatchResource.set(kubernetesClient.pods().withLabel(SPARK_APP_ID_LABEL, applicationId())
    .watch(new ExecutorPodsWatcher()))

  allocator.scheduleWithFixedDelay(
    allocatorRunnable, 0, podAllocationInterval, TimeUnit.SECONDS)

  if (!Utils.isDynamicAllocationEnabled(sc.conf)) {
    doRequestTotalExecutors(initialExecutors)
  } else {
    shufflePodCache = shuffleServiceConfig
      .map { config => new ShufflePodCache(
        kubernetesClient, config.shuffleNamespace, config.shuffleLabels) }
    shufflePodCache.foreach(_.start())
    kubernetesExternalShuffleClient.foreach(_.init(applicationId()))
  }
}

private val allocatorRunnable: Runnable = new Runnable {

  // Maintains a map of executor id to count of checks performed to learn the loss reason
  // for an executor.
  private val executorReasonCheckAttemptCounts = new mutable.HashMap[String, Int]

  override def run(): Unit = {
    handleDisconnectedExecutors()
    RUNNING_EXECUTOR_PODS_LOCK.synchronized {
      if (totalRegisteredExecutors.get() < runningExecutorsToPods.size) {
        logDebug("Waiting for pending executors before scaling")
      } else if (totalExpectedExecutors.get() <= runningExecutorsToPods.size) {
        logDebug("Maximum allowed executor limit reached. Not scaling up further.")
      } else {
        for (i <- 0 until math.min(
          totalExpectedExecutors.get - runningExecutorsToPods.size, podAllocationSize)) {
          val (executorId, pod) = allocateNewExecutorPod()
          runningExecutorsToPods.put(executorId, pod)
          runningPodsToExecutors.put(pod.getMetadata.getName, executorId)
          logInfo(
            s"Requesting a new executor, total executors is now ${runningExecutorsToPods.size}")
        }
      }
    }
  }
  
  def handleDisconnectedExecutors(): Unit = {
    // For each disconnected executor, synchronize with the loss reasons that may have been found
    // by the executor pod watcher. If the loss reason was discovered by the watcher,
    // inform the parent class with removeExecutor.
    val disconnectedPodsByExecutorIdPendingRemovalCopy =
        Map.empty ++ disconnectedPodsByExecutorIdPendingRemoval
    disconnectedPodsByExecutorIdPendingRemovalCopy.foreach { case (executorId, executorPod) =>
      val knownExitReason = podsWithKnownExitReasons.remove(executorPod.getMetadata.getName)
      knownExitReason.fold {
        removeExecutorOrIncrementLossReasonCheckCount(executorId)
      } { executorExited =>
        logDebug(s"Removing executor $executorId with loss reason " + executorExited.message)
        removeExecutor(executorId, executorExited)
        // We keep around executors that have exit conditions caused by the application. This
        // allows them to be debugged later on. Otherwise, mark them as to be deleted from the
        // the API server.
        if (!executorExited.exitCausedByApp) {
          deleteExecutorFromClusterAndDataStructures(executorId)
        }
      }
    }
  }

  def removeExecutorOrIncrementLossReasonCheckCount(executorId: String): Unit = {
    val reasonCheckCount = executorReasonCheckAttemptCounts.getOrElse(executorId, 0)
    if (reasonCheckCount >= MAX_EXECUTOR_LOST_REASON_CHECKS) {
      removeExecutor(executorId, SlaveLost("Executor lost for unknown reasons."))
      deleteExecutorFromClusterAndDataStructures(executorId)
    } else {
      executorReasonCheckAttemptCounts.put(executorId, reasonCheckCount + 1)
    }
  }

  def deleteExecutorFromClusterAndDataStructures(executorId: String): Unit = {
    disconnectedPodsByExecutorIdPendingRemoval -= executorId
    executorReasonCheckAttemptCounts -= executorId
    RUNNING_EXECUTOR_PODS_LOCK.synchronized {
      runningExecutorsToPods.remove(executorId).map { pod =>
        kubernetesClient.pods().delete(pod)
        runningPodsToExecutors.remove(pod.getMetadata.getName)
      }.getOrElse(logWarning(s"Unable to remove pod for unknown executor $executorId"))
    }
  }
}
```

`allocatorRunnable`线程首先创建`executorReasonCheckAttemptCounts(executorId,executorCountCheckPerformed)`，如下：

```scala
// Maintains a map of executor id to count of checks performed to learn the loss reason
// for an executor.
private val executorReasonCheckAttemptCounts = new mutable.HashMap[String, Int]
...
```

根据注释可以知道该结构体保存了`executor`已经被执行`removeExecutorOrIncrementLossReasonCheckCount`的次数 

`run`函数首先执行`handleDisconnectedExecutors`如下：

```scala
def handleDisconnectedExecutors(): Unit = {
  // For each disconnected executor, synchronize with the loss reasons that may have been found
  // by the executor pod watcher. If the loss reason was discovered by the watcher,
  // inform the parent class with removeExecutor.
  val disconnectedPodsByExecutorIdPendingRemovalCopy =
      Map.empty ++ disconnectedPodsByExecutorIdPendingRemoval
  disconnectedPodsByExecutorIdPendingRemovalCopy.foreach { case (executorId, executorPod) =>
    val knownExitReason = podsWithKnownExitReasons.remove(executorPod.getMetadata.getName)
    knownExitReason.fold {
      removeExecutorOrIncrementLossReasonCheckCount(executorId)
    } { executorExited =>
      logDebug(s"Removing executor $executorId with loss reason " + executorExited.message)
      removeExecutor(executorId, executorExited)
      // We keep around executors that have exit conditions caused by the application. This
      // allows them to be debugged later on. Otherwise, mark them as to be deleted from the
      // the API server.
      if (!executorExited.exitCausedByApp) {
        deleteExecutorFromClusterAndDataStructures(executorId)
      }
    }
  }
}  
```

`handleDisconnectedExecutors`执行逻辑如下：

* 1、从`disconnectedPodsByExecutorIdPendingRemoval(executorId,executorPod)`获取`disconnected Pod`，如下：

```scala
private val disconnectedPodsByExecutorIdPendingRemoval =
  new ConcurrentHashMap[String, Pod]().asScala

def handleDisconnectedExecutors(): Unit = {
  // For each disconnected executor, synchronize with the loss reasons that may have been found
  // by the executor pod watcher. If the loss reason was discovered by the watcher,
  // inform the parent class with removeExecutor.
  val disconnectedPodsByExecutorIdPendingRemovalCopy =
      Map.empty ++ disconnectedPodsByExecutorIdPendingRemoval
  disconnectedPodsByExecutorIdPendingRemovalCopy.foreach { case (executorId, executorPod) =>
    val knownExitReason = podsWithKnownExitReasons.remove(executorPod.getMetadata.getName)
    knownExitReason.fold {
      removeExecutorOrIncrementLossReasonCheckCount(executorId)
    } { executorExited =>
      logDebug(s"Removing executor $executorId with loss reason " + executorExited.message)
      removeExecutor(executorId, executorExited)
      // We keep around executors that have exit conditions caused by the application. This
      // allows them to be debugged later on. Otherwise, mark them as to be deleted from the
      // the API server.
      if (!executorExited.exitCausedByApp) {
        deleteExecutorFromClusterAndDataStructures(executorId)
      }
    }
  }
}
```

* 2、遍历`disconnected Pod`判断`knownExitReason(executorName,ExecutorExited)`中是否已经存在Pod `disconnected`的原因`ExecutorExited`：

```scala
private val podsWithKnownExitReasons: concurrent.Map[String, ExecutorExited] =
  new ConcurrentHashMap[String, ExecutorExited]().asScala

def handleDisconnectedExecutors(): Unit = {
  // For each disconnected executor, synchronize with the loss reasons that may have been found
  // by the executor pod watcher. If the loss reason was discovered by the watcher,
  // inform the parent class with removeExecutor.
  val disconnectedPodsByExecutorIdPendingRemovalCopy =
      Map.empty ++ disconnectedPodsByExecutorIdPendingRemoval
  disconnectedPodsByExecutorIdPendingRemovalCopy.foreach { case (executorId, executorPod) =>
    val knownExitReason = podsWithKnownExitReasons.remove(executorPod.getMetadata.getName)
    knownExitReason.fold {
      removeExecutorOrIncrementLossReasonCheckCount(executorId)
    } { executorExited =>
      logDebug(s"Removing executor $executorId with loss reason " + executorExited.message)
      removeExecutor(executorId, executorExited)
      // We keep around executors that have exit conditions caused by the application. This
      // allows them to be debugged later on. Otherwise, mark them as to be deleted from the
      // the API server.
      if (!executorExited.exitCausedByApp) {
        deleteExecutorFromClusterAndDataStructures(executorId)
      }
    }
  }
}
```

* 3、若不存在对应Pod `disconnected`的原因，则执行：

```scala
removeExecutorOrIncrementLossReasonCheckCount(executorId)
```

看`removeExecutorOrIncrementLossReasonCheckCount(executorId)`函数，如下：

```scala
def removeExecutorOrIncrementLossReasonCheckCount(executorId: String): Unit = {
  val reasonCheckCount = executorReasonCheckAttemptCounts.getOrElse(executorId, 0)
  if (reasonCheckCount >= MAX_EXECUTOR_LOST_REASON_CHECKS) {
    removeExecutor(executorId, SlaveLost("Executor lost for unknown reasons."))
    deleteExecutorFromClusterAndDataStructures(executorId)
  } else {
    executorReasonCheckAttemptCounts.put(executorId, reasonCheckCount + 1)
  }
}
```

该函数执行逻辑是：

* 根据`executorId`从`executorReasonCheckAttemptCounts`中获取（默认为0）count of checks performed（已经执行`removeExecutorOrIncrementLossReasonCheckCount`的次数）
* 如果`reasonCheckCount`还没有达到最大check上限`MAX_EXECUTOR_LOST_REASON_CHECKS`，则添加对应`executorId`次数
* 如果`reasonCheckCount`达到最大check上限`MAX_EXECUTOR_LOST_REASON_CHECKS`，则执行`removeExecutor`，如下：

```scala
...
removeExecutor(executorId, SlaveLost("Executor lost for unknown reasons."))
...
/**
  * Called by subclasses when notified of a lost worker. It just fires the message and returns
  * at once.
  */
protected def removeExecutor(executorId: String, reason: ExecutorLossReason): Unit = {
  // Only log the failure since we don't care about the result.
  driverEndpoint.ask[Boolean](RemoveExecutor(executorId, reason)).onFailure { case t =>
    logError(t.getMessage, t)
  }(ThreadUtils.sameThread)
}

...

case class RemoveExecutor(executorId: String, reason: ExecutorLossReason)
  extends CoarseGrainedClusterMessage

...

override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {

  case RegisterExecutor(executorId, executorRef, hostname, cores, logUrls) =>
    if (executorDataMap.contains(executorId)) {
      executorRef.send(RegisterExecutorFailed("Duplicate executor ID: " + executorId))
      context.reply(true)
    } else {
      // If the executor's rpc env is not listening for incoming connections, `hostPort`
      // will be null, and the client connection should be used to contact the executor.
      val executorAddress = if (executorRef.address != null) {
          executorRef.address
        } else {
          context.senderAddress
        }
      logInfo(s"Registered executor $executorRef ($executorAddress) with ID $executorId")
      addressToExecutorId(executorAddress) = executorId
      totalCoreCount.addAndGet(cores)
      totalRegisteredExecutors.addAndGet(1)
      val data = new ExecutorData(executorRef, executorRef.address, hostname,
        cores, cores, logUrls)
      // This must be synchronized because variables mutated
      // in this block are read when requesting executors
      CoarseGrainedSchedulerBackend.this.synchronized {
        executorDataMap.put(executorId, data)
        if (currentExecutorIdCounter < executorId.toInt) {
          currentExecutorIdCounter = executorId.toInt
        }
        if (numPendingExecutors > 0) {
          numPendingExecutors -= 1
          logDebug(s"Decremented number of pending executors ($numPendingExecutors left)")
        }
      }
      executorRef.send(RegisteredExecutor)
      // Note: some tests expect the reply to come after we put the executor in the map
      context.reply(true)
      listenerBus.post(
        SparkListenerExecutorAdded(System.currentTimeMillis(), executorId, data))
      makeOffers()
    }

  case StopDriver =>
    context.reply(true)
    stop()

  case StopExecutors =>
    logInfo("Asking each executor to shut down")
    for ((_, executorData) <- executorDataMap) {
      executorData.executorEndpoint.send(StopExecutor)
    }
    context.reply(true)

  case RemoveExecutor(executorId, reason) =>
    // We will remove the executor's state and cannot restore it. However, the connection
    // between the driver and the executor may be still alive so that the executor won't exit
    // automatically, so try to tell the executor to stop itself. See SPARK-13519.
    executorDataMap.get(executorId).foreach(_.executorEndpoint.send(StopExecutor))
    removeExecutor(executorId, reason)
    context.reply(true)

  case RetrieveSparkAppConfig(executorId) =>
    val reply = SparkAppConfig(sparkProperties,
      SparkEnv.get.securityManager.getIOEncryptionKey())
    context.reply(reply)
}

...

// Remove a disconnected slave from the cluster
private def removeExecutor(executorId: String, reason: ExecutorLossReason): Unit = {
  logDebug(s"Asked to remove executor $executorId with reason $reason")
  executorDataMap.get(executorId) match {
    case Some(executorInfo) =>
      // This must be synchronized because variables mutated
      // in this block are read when requesting executors
      val killed = CoarseGrainedSchedulerBackend.this.synchronized {
        addressToExecutorId -= executorInfo.executorAddress
        executorDataMap -= executorId
        executorsPendingLossReason -= executorId
        executorsPendingToRemove.remove(executorId).getOrElse(false)
      }
      totalCoreCount.addAndGet(-executorInfo.totalCores)
      totalRegisteredExecutors.addAndGet(-1)
      scheduler.executorLost(executorId, if (killed) ExecutorKilled else reason)
      listenerBus.post(
        SparkListenerExecutorRemoved(System.currentTimeMillis(), executorId, reason.toString))
    case None =>
      // SPARK-15262: If an executor is still alive even after the scheduler has removed
      // its metadata, we may receive a heartbeat from that executor and tell its block
      // manager to reregister itself. If that happens, the block manager master will know
      // about the executor, but the scheduler will not. Therefore, we should remove the
      // executor from the block manager when we hit this case.
      scheduler.sc.env.blockManager.master.removeExecutorAsync(executorId)
      logInfo(s"Asked to remove non-existent executor $executorId")
  }
}
```

`removeExecutor`执行如下：

```scala
// Remove a disconnected slave from the cluster
private def removeExecutor(executorId: String, reason: ExecutorLossReason): Unit = {
  logDebug(s"Asked to remove executor $executorId with reason $reason")
  executorDataMap.get(executorId) match {
    case Some(executorInfo) =>
      // This must be synchronized because variables mutated
      // in this block are read when requesting executors
      val killed = CoarseGrainedSchedulerBackend.this.synchronized {
        addressToExecutorId -= executorInfo.executorAddress
        executorDataMap -= executorId
        executorsPendingLossReason -= executorId
        executorsPendingToRemove.remove(executorId).getOrElse(false)
      }
      totalCoreCount.addAndGet(-executorInfo.totalCores)
      totalRegisteredExecutors.addAndGet(-1)
      scheduler.executorLost(executorId, if (killed) ExecutorKilled else reason)
      listenerBus.post(
        SparkListenerExecutorRemoved(System.currentTimeMillis(), executorId, reason.toString))
    case None =>
      // SPARK-15262: If an executor is still alive even after the scheduler has removed
      // its metadata, we may receive a heartbeat from that executor and tell its block
      // manager to reregister itself. If that happens, the block manager master will know
      // about the executor, but the scheduler will not. Therefore, we should remove the
      // executor from the block manager when we hit this case.
      scheduler.sc.env.blockManager.master.removeExecutorAsync(executorId)
      logInfo(s"Asked to remove non-existent executor $executorId")
  }
}

// Accessing `executorDataMap` in `DriverEndpoint.receive/receiveAndReply` doesn't need any
// protection. But accessing `executorDataMap` out of `DriverEndpoint.receive/receiveAndReply`
// must be protected by `CoarseGrainedSchedulerBackend.this`. Besides, `executorDataMap` should
// only be modified in `DriverEndpoint.receive/receiveAndReply` with protection by
// `CoarseGrainedSchedulerBackend.this`.
private val executorDataMap = new HashMap[String, ExecutorData]

/**
 * Grouping of data for an executor used by CoarseGrainedSchedulerBackend.
 *
 * @param executorEndpoint The RpcEndpointRef representing this executor
 * @param executorAddress The network address of this executor
 * @param executorHost The hostname that this executor is running on
 * @param freeCores  The current number of cores available for work on the executor
 * @param totalCores The total number of cores available to the executor
 */
private[cluster] class ExecutorData(
   val executorEndpoint: RpcEndpointRef,
   val executorAddress: RpcAddress,
   override val executorHost: String,
   var freeCores: Int,
   override val totalCores: Int,
   override val logUrlMap: Map[String, String]
) extends ExecutorInfo(executorHost, totalCores, logUrlMap)

/**
 * :: DeveloperApi ::
 * Stores information about an executor to pass from the scheduler to SparkListeners.
 */
@DeveloperApi
class ExecutorInfo(
   val executorHost: String,
   val totalCores: Int,
   val logUrlMap: Map[String, String]) {

  def canEqual(other: Any): Boolean = other.isInstanceOf[ExecutorInfo]

  override def equals(other: Any): Boolean = other match {
    case that: ExecutorInfo =>
      (that canEqual this) &&
        executorHost == that.executorHost &&
        totalCores == that.totalCores &&
        logUrlMap == that.logUrlMap
    case _ => false
  }

  override def hashCode(): Int = {
    val state = Seq(executorHost, totalCores, logUrlMap)
    state.map(_.hashCode()).foldLeft(0)((a, b) => 31 * a + b)
  }
}
```

转到`removeExecutorOrIncrementLossReasonCheckCount`函数，执行完`removeExecutor`后，执行`deleteExecutorFromClusterAndDataStructures`如下：

```scala
def deleteExecutorFromClusterAndDataStructures(executorId: String): Unit = {
  disconnectedPodsByExecutorIdPendingRemoval -= executorId
  executorReasonCheckAttemptCounts -= executorId
  RUNNING_EXECUTOR_PODS_LOCK.synchronized {
    runningExecutorsToPods.remove(executorId).map { pod =>
      kubernetesClient.pods().delete(pod)
      runningPodsToExecutors.remove(pod.getMetadata.getName)
    }.getOrElse(logWarning(s"Unable to remove pod for unknown executor $executorId"))
  }
}
```

* 将`executorId`从`disconnectedPodsByExecutorIdPendingRemoval(executorId,executorPod)`中剔除: `disconnectedPodsByExecutorIdPendingRemoval -= executorId`
* 将`executorId`从`executorReasonCheckAttemptCounts(executorId,executorCountCheckPerformed)`中剔除：`executorReasonCheckAttemptCounts -= executorId`
* 将`executorId`从`runningExecutorsToPods(executorId,executorPod)`中剔除：`runningExecutorsToPods.remove(executorId)`
* 将`executorPod`从集群中物理上删除：`kubernetesClient.pods().delete(pod)`
* 将`executorName`从`runningPodsToExecutors(executorName,executorId)`中剔除：`runningPodsToExecutors.remove(pod.getMetadata.getName)`

总结`removeExecutorOrIncrementLossReasonCheckCount`函数逻辑也即：若`executorId`对应的check次数没有到达阈值：`MAX_EXECUTOR_LOST_REASON_CHECKS`，则增加check次数；否则删除集群中`executorId`对应的Pod以及相应的结构

* 4、若存在对应Pod `disconnected`的原因，执行如下：

```scala
def handleDisconnectedExecutors(): Unit = {
  // For each disconnected executor, synchronize with the loss reasons that may have been found
  // by the executor pod watcher. If the loss reason was discovered by the watcher,
  // inform the parent class with removeExecutor.
  val disconnectedPodsByExecutorIdPendingRemovalCopy =
      Map.empty ++ disconnectedPodsByExecutorIdPendingRemoval
  disconnectedPodsByExecutorIdPendingRemovalCopy.foreach { case (executorId, executorPod) =>
    val knownExitReason = podsWithKnownExitReasons.remove(executorPod.getMetadata.getName)
    knownExitReason.fold {
      removeExecutorOrIncrementLossReasonCheckCount(executorId)
    } { executorExited =>
      logDebug(s"Removing executor $executorId with loss reason " + executorExited.message)
      removeExecutor(executorId, executorExited)
      // We keep around executors that have exit conditions caused by the application. This
      // allows them to be debugged later on. Otherwise, mark them as to be deleted from the
      // the API server.
      if (!executorExited.exitCausedByApp) {
        deleteExecutorFromClusterAndDataStructures(executorId)
      }
    }
  }
}
```

* 执行`removeExecutor`将`executorId`从`scheduler`和`block manager`中删除
* 若该`executorId`对应的Pod `disconnected`不是由`spark`内部原因造成的，而是由外部原因造成的(比如从k8s master执行`kubectl delete pods/xxx`等外部命令)，则执行`deleteExecutorFromClusterAndDataStructures`将该`executorId`对应的Pod从集群中删除（也即内部原因造成的`disconnected`对应的Pod在集群中保留，以便后续debug；否则从集群中删除，不保留）

<span style="color:red">这里保留一个疑问：`executorExited.exitCausedByApp`具体可能是哪些？同时`!executorExited.exitCausedByApp`具体可能又是哪些？</span>

回到`allocatorRunnable`线程的`run`函数：

```scala
private val allocatorRunnable: Runnable = new Runnable {

  // Maintains a map of executor id to count of checks performed to learn the loss reason
  // for an executor.
  private val executorReasonCheckAttemptCounts = new mutable.HashMap[String, Int]

  override def run(): Unit = {
    handleDisconnectedExecutors()
    RUNNING_EXECUTOR_PODS_LOCK.synchronized {
      if (totalRegisteredExecutors.get() < runningExecutorsToPods.size) {
        logDebug("Waiting for pending executors before scaling")
      } else if (totalExpectedExecutors.get() <= runningExecutorsToPods.size) {
        logDebug("Maximum allowed executor limit reached. Not scaling up further.")
      } else {
        for (i <- 0 until math.min(
          totalExpectedExecutors.get - runningExecutorsToPods.size, podAllocationSize)) {
          val (executorId, pod) = allocateNewExecutorPod()
          runningExecutorsToPods.put(executorId, pod)
          runningPodsToExecutors.put(pod.getMetadata.getName, executorId)
          logInfo(
            s"Requesting a new executor, total executors is now ${runningExecutorsToPods.size}")
        }
      }
    }
  }
}
```

修改前`allocatorRunnable`线程如下：

```scala
private val allocatorRunnable: Runnable = new Runnable {
  override def run(): Unit = {
    if (totalRegisteredExecutors.get() < runningExecutorPods.size) {
      logDebug("Waiting for pending executors before scaling")
    } else if (totalExpectedExecutors.get() <= runningExecutorPods.size) {
      logDebug("Maximum allowed executor limit reached. Not scaling up further.")
    } else {
      RUNNING_EXECUTOR_PODS_LOCK.synchronized {
        for (i <- 0 until math.min(
          totalExpectedExecutors.get - runningExecutorPods.size, podAllocationSize)) {
          runningExecutorPods += allocateNewExecutorPod()
          logInfo(
            s"Requesting a new executor, total executors is now ${runningExecutorPods.size}")
        }
      }
    }
  }
}
```

<span style="color:red">这是创建`executor`的主要函数，逻辑很清晰：</span>

* 1、若已经成功创建（`executor`注册了自己(register itself)，则视为成功创建）的`executor` pod数量（`totalRegisteredExecutors`） < 已经发出创建请求的数量(`runningExecutorsToPods`)，则等待`k8s`创建`executor` pod(或者等待`executor` register itself)，直到两者相等为止
* 2、若需要创建的`executor` pod数量（`totalExpectedExecutors`）= 已经发出创建请求的数量(`runningExecutorsToPods`)，则不再发出新的创建请求
* 3、否则，按照策略：`math.min(totalExpectedExecutors.get - runningExecutorsToPods.size, podAllocationSize)`批量发出`executor` pod 创建请求`allocateNewExecutorPod`，并同时增加`runningExecutorsToPods(executorId,executorPod)`和`runningPodsToExecutors(executorName,executorId)`数值



## 改进方案测试

## 结论

## Refs

* [Changes to support executor recovery behavior during static allocation.](https://github.com/apache-spark-on-k8s/spark/pull/244)
* [Code enhancement: Replaced explicit synchronized access to a hashmap …](https://github.com/apache-spark-on-k8s/spark/commit/e5838c1d2bf7515ed00f56d437cbbb67c6aba9af)
* [Unit Tests for KubernetesClusterSchedulerBackend ](https://github.com/apache-spark-on-k8s/spark/pull/459/files)
* [Spark driver should exit and report a failure when all executors get killed/fail](https://github.com/apache-spark-on-k8s/spark/issues/134)
* [Spark behavior on k8s vs yarn on executor failures](https://docs.google.com/document/d/1GX__jsCbeCw4RrUpHLqtpAzHwV82NQrgjz1dCCqqRes/edit#)
* [Scala Runnable](https://twitter.github.io/scala_school/zh_cn/concurrency.html)
* [Scala - for Loops](https://www.tutorialspoint.com/scala/scala_for_loop.htm)


