---
layout: post
title: Spark k8s analysis
date: 2017-5-24 11:04:31
category: 技术
tags: Spark kubernetes
excerpt: Spark k8s analysis……
---

## 前言

spark原生支持三种集群调度器：Standalone、Apache Mesos、Hadoop YARN，如下：

>>The system currently supports three cluster managers:

* (Standalone)[http://spark.apache.org/docs/latest/spark-standalone.html] – a simple cluster manager included with Spark that makes it easy to set up a cluster.
* (Apache Mesos)[http://spark.apache.org/docs/latest/running-on-mesos.html] – a general cluster manager that can also run Hadoop MapReduce and service applications.
* (Hadoop YARN)[http://spark.apache.org/docs/latest/running-on-yarn.html] – the resource manager in Hadoop 2.

参考[这里](http://spark.apache.org/docs/latest/cluster-overview.html#cluster-manager-types)

而kubernetes是目前最流行的容器调度和管理工具，所以使spark支持k8s是非常有必要的，而官方也有相关项目旨在完成这个目标：[apache-spark-on-k8s/spark](https://github.com/apache-spark-on-k8s/spark).

## 源码分析

本文主要目地是分析spark_k8s源码，如下：

```scalastyle
  /**
   * Submit the application using the provided parameters.
   *
   * This runs in two steps. First, we prepare the launch environment by setting up
   * the appropriate classpath, system properties, and application arguments for
   * running the child main class based on the cluster manager and the deploy mode.
   * Second, we use this launch environment to invoke the main method of the child
   * main class.
   */
  @tailrec
  private def submit(args: SparkSubmitArguments): Unit = {
    val (childArgs, childClasspath, sysProps, childMainClass) = prepareSubmitEnvironment(args)

    def doRunMain(): Unit = {
      if (args.proxyUser != null) {
        val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,
          UserGroupInformation.getCurrentUser())
        try {
          proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
            override def run(): Unit = {
              runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
            }
          })
        } catch {
          case e: Exception =>
            // Hadoop's AuthorizationException suppresses the exception's stack trace, which
            // makes the message printed to the output by the JVM not very helpful. Instead,
            // detect exceptions with empty stack traces here, and treat them differently.
            if (e.getStackTrace().length == 0) {
              // scalastyle:off println
              printStream.println(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
              // scalastyle:on println
              exitFn(1)
            } else {
              throw e
            }
        }
      } else {
        runMain(childArgs, childClasspath, sysProps, childMainClass, args.verbose)
      }
    }

     // In standalone cluster mode, there are two submission gateways:
     //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
     //   (2) The new REST-based gateway introduced in Spark 1.3
     // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
     // to use the legacy gateway if the master endpoint turns out to be not a REST server.
    if (args.isStandaloneCluster && args.useRest) {
      try {
        // scalastyle:off println
        printStream.println("Running Spark using the REST application submission protocol.")
        // scalastyle:on println
        doRunMain()
      } catch {
        // Fail over to use the legacy submission gateway
        case e: SubmitRestConnectionException =>
          printWarning(s"Master endpoint ${args.master} was not a REST server. " +
            "Falling back to legacy submission gateway instead.")
          args.useRest = false
          submit(args)
      }
    // In all other modes, just run the main class as prepared
    } else {
      doRunMain()
    }
  }
```

