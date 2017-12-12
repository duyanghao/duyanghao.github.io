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

* 2、遍历`disconnected Pod`判断`knownExitReason(executorName,ExecutorExited)`中是否已经存在Pod`disconnected`的原因`ExecutorExited`：

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

* 3、若不存在对应Pod`disconnected`的原因，则执行：

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


## 改进方案测试

## 结论

## Refs

* [Changes to support executor recovery behavior during static allocation.](https://github.com/apache-spark-on-k8s/spark/pull/244)
* [Code enhancement: Replaced explicit synchronized access to a hashmap …](https://github.com/apache-spark-on-k8s/spark/commit/e5838c1d2bf7515ed00f56d437cbbb67c6aba9af)
* [Unit Tests for KubernetesClusterSchedulerBackend ](https://github.com/apache-spark-on-k8s/spark/pull/459/files)
* [Spark driver should exit and report a failure when all executors get killed/fail](https://github.com/apache-spark-on-k8s/spark/issues/134)
* [Spark behavior on k8s vs yarn on executor failures](https://docs.google.com/document/d/1GX__jsCbeCw4RrUpHLqtpAzHwV82NQrgjz1dCCqqRes/edit#)
* [Scala Runnable](https://twitter.github.io/scala_school/zh_cn/concurrency.html)

