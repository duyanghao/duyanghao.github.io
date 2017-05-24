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

执行`prepareSubmitEnvironment`准备运行环境：

```scalastyle
/**
   * Prepare the environment for submitting an application.
   * This returns a 4-tuple:
   *   (1) the arguments for the child process,
   *   (2) a list of classpath entries for the child,
   *   (3) a map of system properties, and
   *   (4) the main class for the child
   * Exposed for testing.
   */
  private[deploy] def prepareSubmitEnvironment(args: SparkSubmitArguments)
      : (Seq[String], Seq[String], Map[String, String], String) = {
    // Return values
    val childArgs = new ArrayBuffer[String]()
    val childClasspath = new ArrayBuffer[String]()
    val sysProps = new HashMap[String, String]()
    var childMainClass = ""

    // Set the cluster manager
    val clusterManager: Int = args.master match {
      case "yarn" => YARN
      case "yarn-client" | "yarn-cluster" =>
        printWarning(s"Master ${args.master} is deprecated since 2.0." +
          " Please use master \"yarn\" with specified deploy mode instead.")
        YARN
      case m if m.startsWith("spark") => STANDALONE
      case m if m.startsWith("mesos") => MESOS
      case m if m.startsWith("k8s") => KUBERNETES
      case m if m.startsWith("local") => LOCAL
      case _ =>
        printErrorAndExit("Master must either be yarn or start with spark, mesos, k8s, or local")
        -1
    }

    // Set the deploy mode; default is client mode
    var deployMode: Int = args.deployMode match {
      case "client" | null => CLIENT
      case "cluster" => CLUSTER
      case _ => printErrorAndExit("Deploy mode must be either client or cluster"); -1
    }

    // Because the deprecated way of specifying "yarn-cluster" and "yarn-client" encapsulate both
    // the master and deploy mode, we have some logic to infer the master and deploy mode
    // from each other if only one is specified, or exit early if they are at odds.
    if (clusterManager == YARN) {
      (args.master, args.deployMode) match {
        case ("yarn-cluster", null) =>
          deployMode = CLUSTER
          args.master = "yarn"
        case ("yarn-cluster", "client") =>
          printErrorAndExit("Client deploy mode is not compatible with master \"yarn-cluster\"")
        case ("yarn-client", "cluster") =>
          printErrorAndExit("Cluster deploy mode is not compatible with master \"yarn-client\"")
        case (_, mode) =>
          args.master = "yarn"
      }

      // Make sure YARN is included in our build if we're trying to use it
      if (!Utils.classIsLoadable("org.apache.spark.deploy.yarn.Client") && !Utils.isTesting) {
        printErrorAndExit(
          "Could not load YARN classes. " +
          "This copy of Spark may not have been compiled with YARN support.")
      }
    }

    // Update args.deployMode if it is null. It will be passed down as a Spark property later.
    (args.deployMode, deployMode) match {
      case (null, CLIENT) => args.deployMode = "client"
      case (null, CLUSTER) => args.deployMode = "cluster"
      case _ =>
    }
    val isYarnCluster = clusterManager == YARN && deployMode == CLUSTER
    val isMesosCluster = clusterManager == MESOS && deployMode == CLUSTER
    val isKubernetesCluster = clusterManager == KUBERNETES && deployMode == CLUSTER

    // Resolve maven dependencies if there are any and add classpath to jars. Add them to py-files
    // too for packages that include Python code
    val exclusions: Seq[String] =
      if (!StringUtils.isBlank(args.packagesExclusions)) {
        args.packagesExclusions.split(",")
      } else {
        Nil
      }
    val resolvedMavenCoordinates = SparkSubmitUtils.resolveMavenCoordinates(args.packages,
      Option(args.repositories), Option(args.ivyRepoPath), exclusions = exclusions)
    if (!StringUtils.isBlank(resolvedMavenCoordinates)) {
      args.jars = mergeFileLists(args.jars, resolvedMavenCoordinates)
      if (args.isPython) {
        args.pyFiles = mergeFileLists(args.pyFiles, resolvedMavenCoordinates)
      }
    }

    // install any R packages that may have been passed through --jars or --packages.
    // Spark Packages may contain R source code inside the jar.
    if (args.isR && !StringUtils.isBlank(args.jars)) {
      RPackageUtils.checkAndBuildRPackage(args.jars, printStream, args.verbose)
    }

    // Require all python files to be local, so we can add them to the PYTHONPATH
    // In YARN cluster mode, python files are distributed as regular files, which can be non-local.
    // In Mesos cluster mode, non-local python files are automatically downloaded by Mesos.
    if (args.isPython && !isYarnCluster && !isMesosCluster) {
      if (Utils.nonLocalPaths(args.primaryResource).nonEmpty) {
        printErrorAndExit(s"Only local python files are supported: ${args.primaryResource}")
      }
      val nonLocalPyFiles = Utils.nonLocalPaths(args.pyFiles).mkString(",")
      if (nonLocalPyFiles.nonEmpty) {
        printErrorAndExit(s"Only local additional python files are supported: $nonLocalPyFiles")
      }
    }

    // Require all R files to be local
    if (args.isR && !isYarnCluster && !isMesosCluster) {
      if (Utils.nonLocalPaths(args.primaryResource).nonEmpty) {
        printErrorAndExit(s"Only local R files are supported: ${args.primaryResource}")
      }
    }

    // The following modes are not supported or applicable
    (clusterManager, deployMode) match {
      case (KUBERNETES, CLIENT) =>
        printErrorAndExit("Client mode is currently not supported for Kubernetes.")
      case (KUBERNETES, CLUSTER) if args.isPython || args.isR =>
        printErrorAndExit("Kubernetes does not currently support python or R applications.")
      case (STANDALONE, CLUSTER) if args.isPython =>
        printErrorAndExit("Cluster deploy mode is currently not supported for python " +
          "applications on standalone clusters.")
      case (STANDALONE, CLUSTER) if args.isR =>
        printErrorAndExit("Cluster deploy mode is currently not supported for R " +
          "applications on standalone clusters.")
      case (LOCAL, CLUSTER) =>
        printErrorAndExit("Cluster deploy mode is not compatible with master \"local\"")
      case (_, CLUSTER) if isShell(args.primaryResource) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark shells.")
      case (_, CLUSTER) if isSqlShell(args.mainClass) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark SQL shell.")
      case (_, CLUSTER) if isThriftServer(args.mainClass) =>
        printErrorAndExit("Cluster deploy mode is not applicable to Spark Thrift server.")
      case _ =>
    }

    // If we're running a python app, set the main class to our specific python runner
    if (args.isPython && deployMode == CLIENT) {
      if (args.primaryResource == PYSPARK_SHELL) {
        args.mainClass = "org.apache.spark.api.python.PythonGatewayServer"
      } else {
        // If a python file is provided, add it to the child arguments and list of files to deploy.
        // Usage: PythonAppRunner <main python file> <extra python files> [app arguments]
        args.mainClass = "org.apache.spark.deploy.PythonRunner"
        args.childArgs = ArrayBuffer(args.primaryResource, args.pyFiles) ++ args.childArgs
        if (clusterManager != YARN) {
          // The YARN backend distributes the primary file differently, so don't merge it.
          args.files = mergeFileLists(args.files, args.primaryResource)
        }
      }
      if (clusterManager != YARN) {
        // The YARN backend handles python files differently, so don't merge the lists.
        args.files = mergeFileLists(args.files, args.pyFiles)
      }
      if (args.pyFiles != null) {
        sysProps("spark.submit.pyFiles") = args.pyFiles
      }
    }

    // In YARN mode for an R app, add the SparkR package archive and the R package
    // archive containing all of the built R libraries to archives so that they can
    // be distributed with the job
    if (args.isR && clusterManager == YARN) {
      val sparkRPackagePath = RUtils.localSparkRPackagePath
      if (sparkRPackagePath.isEmpty) {
        printErrorAndExit("SPARK_HOME does not exist for R application in YARN mode.")
      }
      val sparkRPackageFile = new File(sparkRPackagePath.get, SPARKR_PACKAGE_ARCHIVE)
      if (!sparkRPackageFile.exists()) {
        printErrorAndExit(s"$SPARKR_PACKAGE_ARCHIVE does not exist for R application in YARN mode.")
      }
      val sparkRPackageURI = Utils.resolveURI(sparkRPackageFile.getAbsolutePath).toString

      // Distribute the SparkR package.
      // Assigns a symbol link name "sparkr" to the shipped package.
      args.archives = mergeFileLists(args.archives, sparkRPackageURI + "#sparkr")

      // Distribute the R package archive containing all the built R packages.
      if (!RUtils.rPackages.isEmpty) {
        val rPackageFile =
          RPackageUtils.zipRLibraries(new File(RUtils.rPackages.get), R_PACKAGE_ARCHIVE)
        if (!rPackageFile.exists()) {
          printErrorAndExit("Failed to zip all the built R packages.")
        }

        val rPackageURI = Utils.resolveURI(rPackageFile.getAbsolutePath).toString
        // Assigns a symbol link name "rpkg" to the shipped package.
        args.archives = mergeFileLists(args.archives, rPackageURI + "#rpkg")
      }
    }

    // TODO: Support distributing R packages with standalone cluster
    if (args.isR && clusterManager == STANDALONE && !RUtils.rPackages.isEmpty) {
      printErrorAndExit("Distributing R packages with standalone cluster is not supported.")
    }

    // TODO: Support distributing R packages with mesos cluster
    if (args.isR && clusterManager == MESOS && !RUtils.rPackages.isEmpty) {
      printErrorAndExit("Distributing R packages with mesos cluster is not supported.")
    }

    // If we're running an R app, set the main class to our specific R runner
    if (args.isR && deployMode == CLIENT) {
      if (args.primaryResource == SPARKR_SHELL) {
        args.mainClass = "org.apache.spark.api.r.RBackend"
      } else {
        // If an R file is provided, add it to the child arguments and list of files to deploy.
        // Usage: RRunner <main R file> [app arguments]
        args.mainClass = "org.apache.spark.deploy.RRunner"
        args.childArgs = ArrayBuffer(args.primaryResource) ++ args.childArgs
        args.files = mergeFileLists(args.files, args.primaryResource)
      }
    }

    if (isYarnCluster && args.isR) {
      // In yarn-cluster mode for an R app, add primary resource to files
      // that can be distributed with the job
      args.files = mergeFileLists(args.files, args.primaryResource)
    }

    // Special flag to avoid deprecation warnings at the client
    sysProps("SPARK_SUBMIT") = "true"

    // A list of rules to map each argument to system properties or command-line options in
    // each deploy mode; we iterate through these below
    val options = List[OptionAssigner](

      // All cluster managers
      OptionAssigner(args.master, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, sysProp = "spark.master"),
      OptionAssigner(args.deployMode, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        sysProp = "spark.submit.deployMode"),
      OptionAssigner(args.name, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, sysProp = "spark.app.name"),
      OptionAssigner(args.ivyRepoPath, ALL_CLUSTER_MGRS, CLIENT, sysProp = "spark.jars.ivy"),
      OptionAssigner(args.driverMemory, ALL_CLUSTER_MGRS, CLIENT,
        sysProp = "spark.driver.memory"),
      OptionAssigner(args.driverExtraClassPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        sysProp = "spark.driver.extraClassPath"),
      OptionAssigner(args.driverExtraJavaOptions, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        sysProp = "spark.driver.extraJavaOptions"),
      OptionAssigner(args.driverExtraLibraryPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
        sysProp = "spark.driver.extraLibraryPath"),

      // Yarn only
      OptionAssigner(args.queue, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.queue"),
      OptionAssigner(args.numExecutors, YARN, ALL_DEPLOY_MODES,
        sysProp = "spark.executor.instances"),
      OptionAssigner(args.jars, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.dist.jars"),
      OptionAssigner(args.files, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.dist.files"),
      OptionAssigner(args.archives, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.dist.archives"),
      OptionAssigner(args.principal, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.principal"),
      OptionAssigner(args.keytab, YARN, ALL_DEPLOY_MODES, sysProp = "spark.yarn.keytab"),

      OptionAssigner(args.kubernetesNamespace, KUBERNETES, ALL_DEPLOY_MODES,
        sysProp = "spark.kubernetes.namespace"),

        // Other options
      OptionAssigner(args.executorCores, STANDALONE | YARN, ALL_DEPLOY_MODES,
        sysProp = "spark.executor.cores"),
      OptionAssigner(args.executorMemory, STANDALONE | MESOS | YARN, ALL_DEPLOY_MODES,
        sysProp = "spark.executor.memory"),
      OptionAssigner(args.totalExecutorCores, STANDALONE | MESOS, ALL_DEPLOY_MODES,
        sysProp = "spark.cores.max"),
      OptionAssigner(args.files, LOCAL | STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
        sysProp = "spark.files"),
      OptionAssigner(args.jars, LOCAL, CLIENT, sysProp = "spark.jars"),
      OptionAssigner(args.jars, STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
        sysProp = "spark.jars"),
      OptionAssigner(args.driverMemory, STANDALONE | MESOS | YARN, CLUSTER,
        sysProp = "spark.driver.memory"),
      OptionAssigner(args.driverCores, STANDALONE | MESOS | YARN, CLUSTER,
        sysProp = "spark.driver.cores"),
      OptionAssigner(args.supervise.toString, STANDALONE | MESOS, CLUSTER,
        sysProp = "spark.driver.supervise"),
      OptionAssigner(args.ivyRepoPath, STANDALONE, CLUSTER, sysProp = "spark.jars.ivy")
    )

    // In client mode, launch the application main class directly
    // In addition, add the main application jar and any added jars (if any) to the classpath
    if (deployMode == CLIENT) {
      childMainClass = args.mainClass
      if (isUserJar(args.primaryResource)) {
        childClasspath += args.primaryResource
      }
      if (args.jars != null) { childClasspath ++= args.jars.split(",") }
      if (args.childArgs != null) { childArgs ++= args.childArgs }
    }

    // Map all arguments to command-line options or system properties for our chosen mode
    for (opt <- options) {
      if (opt.value != null &&
          (deployMode & opt.deployMode) != 0 &&
          (clusterManager & opt.clusterManager) != 0) {
        if (opt.clOption != null) { childArgs += (opt.clOption, opt.value) }
        if (opt.sysProp != null) { sysProps.put(opt.sysProp, opt.value) }
      }
    }

    // Add the application jar automatically so the user doesn't have to call sc.addJar
    // For YARN cluster mode, the jar is already distributed on each node as "app.jar"
    // In Kubernetes cluster mode, the jar will be uploaded by the client separately.
    // For python and R files, the primary resource is already distributed as a regular file
    if (!isYarnCluster && !isKubernetesCluster && !args.isPython && !args.isR) {
      var jars = sysProps.get("spark.jars").map(x => x.split(",").toSeq).getOrElse(Seq.empty)
      if (isUserJar(args.primaryResource)) {
        jars = jars ++ Seq(args.primaryResource)
      }
      sysProps.put("spark.jars", jars.mkString(","))
    }

    // In standalone cluster mode, use the REST client to submit the application (Spark 1.3+).
    // All Spark parameters are expected to be passed to the client through system properties.
    if (args.isStandaloneCluster) {
      if (args.useRest) {
        childMainClass = "org.apache.spark.deploy.rest.RestSubmissionClient"
        childArgs += (args.primaryResource, args.mainClass)
      } else {
        // In legacy standalone cluster mode, use Client as a wrapper around the user class
        childMainClass = "org.apache.spark.deploy.Client"
        if (args.supervise) { childArgs += "--supervise" }
        Option(args.driverMemory).foreach { m => childArgs += ("--memory", m) }
        Option(args.driverCores).foreach { c => childArgs += ("--cores", c) }
        childArgs += "launch"
        childArgs += (args.master, args.primaryResource, args.mainClass)
      }
      if (args.childArgs != null) {
        childArgs ++= args.childArgs
      }
    }

    // Let YARN know it's a pyspark app, so it distributes needed libraries.
    if (clusterManager == YARN) {
      if (args.isPython) {
        sysProps.put("spark.yarn.isPython", "true")
      }

      if (args.pyFiles != null) {
        sysProps("spark.submit.pyFiles") = args.pyFiles
      }
    }

    // assure a keytab is available from any place in a JVM
    if (clusterManager == YARN || clusterManager == LOCAL) {
      if (args.principal != null) {
        require(args.keytab != null, "Keytab must be specified when principal is specified")
        if (!new File(args.keytab).exists()) {
          throw new SparkException(s"Keytab file: ${args.keytab} does not exist")
        } else {
          // Add keytab and principal configurations in sysProps to make them available
          // for later use; e.g. in spark sql, the isolated class loader used to talk
          // to HiveMetastore will use these settings. They will be set as Java system
          // properties and then loaded by SparkConf
          sysProps.put("spark.yarn.keytab", args.keytab)
          sysProps.put("spark.yarn.principal", args.principal)

          UserGroupInformation.loginUserFromKeytab(args.principal, args.keytab)
        }
      }
    }

    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    if (isYarnCluster) {
      childMainClass = "org.apache.spark.deploy.yarn.Client"
      if (args.isPython) {
        childArgs += ("--primary-py-file", args.primaryResource)
        childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
      } else if (args.isR) {
        val mainFile = new Path(args.primaryResource).getName
        childArgs += ("--primary-r-file", mainFile)
        childArgs += ("--class", "org.apache.spark.deploy.RRunner")
      } else {
        if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
          childArgs += ("--jar", args.primaryResource)
        }
        childArgs += ("--class", args.mainClass)
      }
      if (args.childArgs != null) {
        args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
      }
    }

    if (isMesosCluster) {
      assert(args.useRest, "Mesos cluster mode is only supported through the REST submission API")
      childMainClass = "org.apache.spark.deploy.rest.RestSubmissionClient"
      if (args.isPython) {
        // Second argument is main class
        childArgs += (args.primaryResource, "")
        if (args.pyFiles != null) {
          sysProps("spark.submit.pyFiles") = args.pyFiles
        }
      } else if (args.isR) {
        // Second argument is main class
        childArgs += (args.primaryResource, "")
      } else {
        childArgs += (args.primaryResource, args.mainClass)
      }
      if (args.childArgs != null) {
        childArgs ++= args.childArgs
      }
    }

    if (isKubernetesCluster) {
      childMainClass = "org.apache.spark.deploy.kubernetes.Client"
      childArgs += args.primaryResource
      childArgs += args.mainClass
      childArgs ++= args.childArgs
    }

    // Load any properties specified through --conf and the default properties file
    for ((k, v) <- args.sparkProperties) {
      sysProps.getOrElseUpdate(k, v)
    }

    // Ignore invalid spark.driver.host in cluster modes.
    if (deployMode == CLUSTER) {
      sysProps -= "spark.driver.host"
    }

    // Resolve paths in certain spark properties
    val pathConfigs = Seq(
      "spark.jars",
      "spark.files",
      "spark.yarn.dist.files",
      "spark.yarn.dist.archives",
      "spark.yarn.dist.jars")
    pathConfigs.foreach { config =>
      // Replace old URIs with resolved URIs, if they exist
      sysProps.get(config).foreach { oldValue =>
        sysProps(config) = Utils.resolveURIs(oldValue)
      }
    }

    // Resolve and format python file paths properly before adding them to the PYTHONPATH.
    // The resolving part is redundant in the case of --py-files, but necessary if the user
    // explicitly sets `spark.submit.pyFiles` in his/her default properties file.
    sysProps.get("spark.submit.pyFiles").foreach { pyFiles =>
      val resolvedPyFiles = Utils.resolveURIs(pyFiles)
      val formattedPyFiles = if (!isYarnCluster && !isMesosCluster) {
        PythonRunner.formatPaths(resolvedPyFiles).mkString(",")
      } else {
        // Ignoring formatting python path in yarn and mesos cluster mode, these two modes
        // support dealing with remote python files, they could distribute and add python files
        // locally.
        resolvedPyFiles
      }
      sysProps("spark.submit.pyFiles") = formattedPyFiles
    }

    (childArgs, childClasspath, sysProps, childMainClass)
  }
```

执行`doRunMain` running the child main class based on the cluster manager and the deploy mode:

```scalastyle
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
```

执行`runMain`,Run the main method of the child class using the provided launch environment.

```scalastyle
  /**
   * Run the main method of the child class using the provided launch environment.
   *
   * Note that this main class will not be the one provided by the user if we're
   * running cluster deploy mode or python applications.
   */
  private def runMain(
      childArgs: Seq[String],
      childClasspath: Seq[String],
      sysProps: Map[String, String],
      childMainClass: String,
      verbose: Boolean): Unit = {
    // scalastyle:off println
    if (verbose) {
      printStream.println(s"Main class:\n$childMainClass")
      printStream.println(s"Arguments:\n${childArgs.mkString("\n")}")
      printStream.println(s"System properties:\n${sysProps.mkString("\n")}")
      printStream.println(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      printStream.println("\n")
    }
    // scalastyle:on println

    val loader =
      if (sysProps.getOrElse("spark.driver.userClassPathFirst", "false").toBoolean) {
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    Thread.currentThread.setContextClassLoader(loader)

    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    for ((key, value) <- sysProps) {
      System.setProperty(key, value)
    }

    var mainClass: Class[_] = null

    try {
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        e.printStackTrace(printStream)
        if (childMainClass.contains("thriftserver")) {
          // scalastyle:off println
          printStream.println(s"Failed to load main class $childMainClass.")
          printStream.println("You need to build Spark with -Phive and -Phive-thriftserver.")
          // scalastyle:on println
        }
        System.exit(CLASS_NOT_FOUND_EXIT_STATUS)
      case e: NoClassDefFoundError =>
        e.printStackTrace(printStream)
        if (e.getMessage.contains("org/apache/hadoop/hive")) {
          // scalastyle:off println
          printStream.println(s"Failed to load hive class.")
          printStream.println("You need to build Spark with -Phive and -Phive-thriftserver.")
          // scalastyle:on println
        }
        System.exit(CLASS_NOT_FOUND_EXIT_STATUS)
    }

    // SPARK-4170
    if (classOf[scala.App].isAssignableFrom(mainClass)) {
      printWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
    }

    val mainMethod = mainClass.getMethod("main", new Array[String](0).getClass)
    if (!Modifier.isStatic(mainMethod.getModifiers)) {
      throw new IllegalStateException("The main method in the given main class must be static")
    }

    @tailrec
    def findCause(t: Throwable): Throwable = t match {
      case e: UndeclaredThrowableException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: InvocationTargetException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: Throwable =>
        e
    }

    try {
      mainMethod.invoke(null, childArgs.toArray)
    } catch {
      case t: Throwable =>
        findCause(t) match {
          case SparkUserAppException(exitCode) =>
            System.exit(exitCode)

          case t: Throwable =>
            throw t
        }
    }
  }
```

```scalastyle
private[spark] object Client extends Logging {

  private[spark] val SECURE_RANDOM = new SecureRandom()

  def main(args: Array[String]): Unit = {
    require(args.length >= 2, s"Too few arguments. Usage: ${getClass.getName} <mainAppResource>" +
      s" <mainClass> [<application arguments>]")
    val mainAppResource = args(0)
    val mainClass = args(1)
    val appArgs = args.drop(2)
    val sparkConf = new SparkConf(true)
    new Client(
      mainAppResource = mainAppResource,
      mainClass = mainClass,
      sparkConf = sparkConf,
      appArgs = appArgs).run()
  }

  def resolveK8sMaster(rawMasterString: String): String = {
    if (!rawMasterString.startsWith("k8s://")) {
      throw new IllegalArgumentException("Master URL should start with k8s:// in Kubernetes mode.")
    }
    val masterWithoutK8sPrefix = rawMasterString.replaceFirst("k8s://", "")
    if (masterWithoutK8sPrefix.startsWith("http://")
        || masterWithoutK8sPrefix.startsWith("https://")) {
      masterWithoutK8sPrefix
    } else {
      val resolvedURL = s"https://$masterWithoutK8sPrefix"
      logDebug(s"No scheme specified for kubernetes master URL, so defaulting to https. Resolved" +
        s" URL is $resolvedURL")
      resolvedURL
    }
  }
}
```

```scalastyle
private[spark] class Client(
    sparkConf: SparkConf,
    mainClass: String,
    mainAppResource: String,
    appArgs: Array[String]) extends Logging {
  import Client._

  private val namespace = sparkConf.get(KUBERNETES_NAMESPACE)
  private val master = resolveK8sMaster(sparkConf.get("spark.master"))

  private val launchTime = System.currentTimeMillis
  private val appName = sparkConf.getOption("spark.app.name")
    .getOrElse("spark")
  private val kubernetesAppId = sparkConf.getOption("spark.kubernetes.driver.pod.name")
    .getOrElse(s"$appName-$launchTime".toLowerCase.replaceAll("\\.", "-"))
  private val secretName = s"$SUBMISSION_APP_SECRET_PREFIX-$kubernetesAppId"
  private val secretDirectory = s"$DRIVER_CONTAINER_SUBMISSION_SECRETS_BASE_DIR/$kubernetesAppId"
  private val driverDockerImage = sparkConf.get(DRIVER_DOCKER_IMAGE)
  private val uiPort = sparkConf.getInt("spark.ui.port", DEFAULT_UI_PORT)
  private val driverSubmitTimeoutSecs = sparkConf.get(KUBERNETES_DRIVER_SUBMIT_TIMEOUT)
  private val driverServiceManagerType = sparkConf.get(DRIVER_SERVICE_MANAGER_TYPE)
  private val sparkFiles = sparkConf.getOption("spark.files")
    .map(_.split(","))
    .getOrElse(Array.empty[String])
  private val sparkJars = sparkConf.getOption("spark.jars")
    .map(_.split(","))
    .getOrElse(Array.empty[String])

  // CPU settings
  private val driverCpuCores = sparkConf.getOption("spark.driver.cores").getOrElse("1")

  // Memory settings
  private val driverMemoryMb = sparkConf.get(org.apache.spark.internal.config.DRIVER_MEMORY)
  private val driverSubmitServerMemoryMb = sparkConf.get(KUBERNETES_DRIVER_SUBMIT_SERVER_MEMORY)
  private val driverSubmitServerMemoryString = sparkConf.get(
    KUBERNETES_DRIVER_SUBMIT_SERVER_MEMORY.key,
    KUBERNETES_DRIVER_SUBMIT_SERVER_MEMORY.defaultValueString)
  private val driverContainerMemoryMb = driverMemoryMb + driverSubmitServerMemoryMb
  private val memoryOverheadMb = sparkConf
    .get(KUBERNETES_DRIVER_MEMORY_OVERHEAD)
    .getOrElse(math.max((MEMORY_OVERHEAD_FACTOR * driverContainerMemoryMb).toInt,
      MEMORY_OVERHEAD_MIN))
  private val driverContainerMemoryWithOverhead = driverContainerMemoryMb + memoryOverheadMb

  private val waitForAppCompletion: Boolean = sparkConf.get(WAIT_FOR_APP_COMPLETION)

  private val secretBase64String = {
    val secretBytes = new Array[Byte](128)
    SECURE_RANDOM.nextBytes(secretBytes)
    Base64.encodeBase64String(secretBytes)
  }

  private val serviceAccount = sparkConf.get(KUBERNETES_SERVICE_ACCOUNT_NAME)
  private val customLabels = sparkConf.get(KUBERNETES_DRIVER_LABELS)
  private val customAnnotations = sparkConf.get(KUBERNETES_DRIVER_ANNOTATIONS)

  private val kubernetesResourceCleaner = new KubernetesResourceCleaner

  def run(): Unit = {
    logInfo(s"Starting application $kubernetesAppId in Kubernetes...")
    val submitterLocalFiles = KubernetesFileUtils.getOnlySubmitterLocalFiles(sparkFiles)
    val submitterLocalJars = KubernetesFileUtils.getOnlySubmitterLocalFiles(sparkJars)
    (submitterLocalFiles ++ submitterLocalJars).foreach { file =>
      if (!new File(Utils.resolveURI(file).getPath).isFile) {
        throw new SparkException(s"File $file does not exist or is a directory.")
      }
    }
    if (KubernetesFileUtils.isUriLocalFile(mainAppResource) &&
        !new File(Utils.resolveURI(mainAppResource).getPath).isFile) {
      throw new SparkException(s"Main app resource file $mainAppResource is not a file or" +
        s" is a directory.")
    }
    val driverServiceManager = getDriverServiceManager
    val parsedCustomLabels = parseKeyValuePairs(customLabels, KUBERNETES_DRIVER_LABELS.key,
      "labels")
    parsedCustomLabels.keys.foreach { key =>
      require(key != SPARK_APP_ID_LABEL, "Label with key" +
        s" $SPARK_APP_ID_LABEL cannot be used in" +
        " spark.kubernetes.driver.labels, as it is reserved for Spark's" +
        " internal configuration.")
    }
    val parsedCustomAnnotations = parseKeyValuePairs(
      customAnnotations,
      KUBERNETES_DRIVER_ANNOTATIONS.key,
      "annotations")
    val driverPodKubernetesCredentials = new DriverPodKubernetesCredentialsProvider(sparkConf).get()
    var k8ConfBuilder = new K8SConfigBuilder()
      .withApiVersion("v1")
      .withMasterUrl(master)
      .withNamespace(namespace)
    sparkConf.get(KUBERNETES_SUBMIT_CA_CERT_FILE).foreach {
      f => k8ConfBuilder = k8ConfBuilder.withCaCertFile(f)
    }
    sparkConf.get(KUBERNETES_SUBMIT_CLIENT_KEY_FILE).foreach {
      f => k8ConfBuilder = k8ConfBuilder.withClientKeyFile(f)
    }
    sparkConf.get(KUBERNETES_SUBMIT_CLIENT_CERT_FILE).foreach {
      f => k8ConfBuilder = k8ConfBuilder.withClientCertFile(f)
    }
    sparkConf.get(KUBERNETES_SUBMIT_OAUTH_TOKEN).foreach { token =>
      k8ConfBuilder = k8ConfBuilder.withOauthToken(token)
    }

    val k8ClientConfig = k8ConfBuilder.build
    Utils.tryWithResource(new DefaultKubernetesClient(k8ClientConfig)) { kubernetesClient =>
      driverServiceManager.start(kubernetesClient, kubernetesAppId, sparkConf)
      // start outer watch for status logging of driver pod
      // only enable interval logging if in waitForAppCompletion mode
      val loggingInterval = if (waitForAppCompletion) sparkConf.get(REPORT_INTERVAL) else 0
      val driverPodCompletedLatch = new CountDownLatch(1)
      val loggingWatch = new LoggingPodStatusWatcher(driverPodCompletedLatch, kubernetesAppId,
        loggingInterval)
      Utils.tryWithResource(kubernetesClient
          .pods()
          .withName(kubernetesAppId)
          .watch(loggingWatch)) { _ =>
        val resourceCleanShutdownHook = ShutdownHookManager.addShutdownHook(() =>
          kubernetesResourceCleaner.deleteAllRegisteredResourcesFromKubernetes(kubernetesClient))
        val cleanupServiceManagerHook = ShutdownHookManager.addShutdownHook(
          ShutdownHookManager.DEFAULT_SHUTDOWN_PRIORITY)(
          () => driverServiceManager.stop())
        // Place the error hook at a higher priority in order for the error hook to run before
        // the stop hook.
        val serviceManagerErrorHook = ShutdownHookManager.addShutdownHook(
          ShutdownHookManager.DEFAULT_SHUTDOWN_PRIORITY + 1)(() =>
          driverServiceManager.handleSubmissionError(
            new SparkException("Submission shutting down early...")))
        try {
          val sslConfigurationProvider = new DriverSubmitSslConfigurationProvider(
            sparkConf, kubernetesAppId, kubernetesClient, kubernetesResourceCleaner)
          val submitServerSecret = kubernetesClient.secrets().createNew()
            .withNewMetadata()
            .withName(secretName)
            .endMetadata()
            .withData(Map((SUBMISSION_APP_SECRET_NAME, secretBase64String)).asJava)
            .withType("Opaque")
            .done()
          kubernetesResourceCleaner.registerOrUpdateResource(submitServerSecret)
          val sslConfiguration = sslConfigurationProvider.getSslConfiguration()
          val (driverPod, driverService) = launchDriverKubernetesComponents(
            kubernetesClient,
            driverServiceManager,
            parsedCustomLabels,
            parsedCustomAnnotations,
            submitServerSecret,
            sslConfiguration)
          configureOwnerReferences(
            kubernetesClient,
            submitServerSecret,
            sslConfiguration.sslSecret,
            driverPod,
            driverService)
          submitApplicationToDriverServer(
            kubernetesClient,
            driverServiceManager,
            sslConfiguration,
            driverService,
            submitterLocalFiles,
            submitterLocalJars,
            driverPodKubernetesCredentials)
          // Now that the application has started, persist the components that were created beyond
          // the shutdown hook. We still want to purge the one-time secrets, so do not unregister
          // those.
          kubernetesResourceCleaner.unregisterResource(driverPod)
          kubernetesResourceCleaner.unregisterResource(driverService)
        } catch {
          case e: Throwable =>
            driverServiceManager.handleSubmissionError(e)
            throw e
        } finally {
          Utils.tryLogNonFatalError {
            kubernetesResourceCleaner.deleteAllRegisteredResourcesFromKubernetes(kubernetesClient)
          }
          Utils.tryLogNonFatalError {
            driverServiceManager.stop()
          }
          // Remove the shutdown hooks that would be redundant
          Utils.tryLogNonFatalError {
            ShutdownHookManager.removeShutdownHook(resourceCleanShutdownHook)
          }
          Utils.tryLogNonFatalError {
            ShutdownHookManager.removeShutdownHook(cleanupServiceManagerHook)
          }
          Utils.tryLogNonFatalError {
            ShutdownHookManager.removeShutdownHook(serviceManagerErrorHook)
          }
        }
        // wait if configured to do so
        if (waitForAppCompletion) {
          logInfo(s"Waiting for application $kubernetesAppId to finish...")
          driverPodCompletedLatch.await()
          logInfo(s"Application $kubernetesAppId finished.")
        } else {
          logInfo(s"Application $kubernetesAppId successfully launched.")
        }
      }
    }
  }

  private def submitApplicationToDriverServer(
      kubernetesClient: KubernetesClient,
      driverServiceManager: DriverServiceManager,
      sslConfiguration: DriverSubmitSslConfiguration,
      driverService: Service,
      submitterLocalFiles: Iterable[String],
      submitterLocalJars: Iterable[String],
      driverPodKubernetesCredentials: KubernetesCredentials): Unit = {
    sparkConf.getOption("spark.app.id").foreach { id =>
      logWarning(s"Warning: Provided app id in spark.app.id as $id will be" +
        s" overridden as $kubernetesAppId")
    }
    sparkConf.set(KUBERNETES_DRIVER_POD_NAME, kubernetesAppId)
    sparkConf.set(KUBERNETES_DRIVER_SERVICE_NAME, driverService.getMetadata.getName)
    sparkConf.set("spark.app.id", kubernetesAppId)
    sparkConf.setIfMissing("spark.app.name", appName)
    sparkConf.setIfMissing("spark.driver.port", DEFAULT_DRIVER_PORT.toString)
    sparkConf.setIfMissing("spark.blockmanager.port",
      DEFAULT_BLOCKMANAGER_PORT.toString)
    sparkConf.get(KUBERNETES_SUBMIT_OAUTH_TOKEN).foreach { _ =>
      sparkConf.set(KUBERNETES_SUBMIT_OAUTH_TOKEN, "<present_but_redacted>")
    }
    sparkConf.get(KUBERNETES_DRIVER_OAUTH_TOKEN).foreach { _ =>
      sparkConf.set(KUBERNETES_DRIVER_OAUTH_TOKEN, "<present_but_redacted>")
    }
    val driverSubmitter = buildDriverSubmissionClient(
      kubernetesClient,
      driverServiceManager,
      driverService,
      sslConfiguration)
    // Sanity check to see if the driver submitter is even reachable.
    driverSubmitter.ping()
    logInfo(s"Submitting local resources to driver pod for application " +
      s"$kubernetesAppId ...")
    val submitRequest = buildSubmissionRequest(
      submitterLocalFiles,
      submitterLocalJars,
      driverPodKubernetesCredentials)
    driverSubmitter.submitApplication(submitRequest)
    logInfo("Successfully submitted local resources and driver configuration to" +
      " driver pod.")
    // After submitting, adjust the service to only expose the Spark UI
    val uiServiceType = if (sparkConf.get(EXPOSE_KUBERNETES_DRIVER_SERVICE_UI_PORT)) "NodePort"
      else "ClusterIP"
    val uiServicePort = new ServicePortBuilder()
      .withName(UI_PORT_NAME)
      .withPort(uiPort)
      .withNewTargetPort(uiPort)
      .build()
    val resolvedService = kubernetesClient.services().withName(kubernetesAppId).edit()
      .editSpec()
        .withType(uiServiceType)
        .withPorts(uiServicePort)
        .endSpec()
      .done()
    kubernetesResourceCleaner.registerOrUpdateResource(resolvedService)
    logInfo("Finished submitting application to Kubernetes.")
  }

  private def launchDriverKubernetesComponents(
      kubernetesClient: KubernetesClient,
      driverServiceManager: DriverServiceManager,
      customLabels: Map[String, String],
      customAnnotations: Map[String, String],
      submitServerSecret: Secret,
      sslConfiguration: DriverSubmitSslConfiguration): (Pod, Service) = {
    val driverKubernetesSelectors = (Map(
      SPARK_DRIVER_LABEL -> kubernetesAppId,
      SPARK_APP_ID_LABEL -> kubernetesAppId,
      SPARK_APP_NAME_LABEL -> appName)
      ++ customLabels)
    val endpointsReadyFuture = SettableFuture.create[Endpoints]
    val endpointsReadyWatcher = new DriverEndpointsReadyWatcher(endpointsReadyFuture)
    val serviceReadyFuture = SettableFuture.create[Service]
    val serviceReadyWatcher = new DriverServiceReadyWatcher(serviceReadyFuture)
    val podReadyFuture = SettableFuture.create[Pod]
    val podWatcher = new DriverPodReadyWatcher(podReadyFuture)
    Utils.tryWithResource(kubernetesClient
        .pods()
        .withName(kubernetesAppId)
        .watch(podWatcher)) { _ =>
      Utils.tryWithResource(kubernetesClient
          .services()
          .withName(kubernetesAppId)
          .watch(serviceReadyWatcher)) { _ =>
        Utils.tryWithResource(kubernetesClient
            .endpoints()
            .withName(kubernetesAppId)
            .watch(endpointsReadyWatcher)) { _ =>
          val serviceTemplate = createDriverServiceTemplate(driverKubernetesSelectors)
          val driverService = kubernetesClient.services().create(
            driverServiceManager.customizeDriverService(serviceTemplate).build())
          kubernetesResourceCleaner.registerOrUpdateResource(driverService)
          val driverPod = createDriverPod(
            kubernetesClient,
            driverKubernetesSelectors,
            customAnnotations,
            submitServerSecret,
            sslConfiguration)
          waitForReadyKubernetesComponents(kubernetesClient, endpointsReadyFuture,
            serviceReadyFuture, podReadyFuture)
          (driverPod, driverService)
        }
      }
    }
  }

  /**
   * Sets the owner reference for all the kubernetes components to link to the driver pod.
   *
   * @return The driver service after it has been adjusted to reflect the new owner
   * reference.
   */
  private def configureOwnerReferences(
      kubernetesClient: KubernetesClient,
      submitServerSecret: Secret,
      sslSecret: Option[Secret],
      driverPod: Pod,
      driverService: Service): Service = {
    val driverPodOwnerRef = new OwnerReferenceBuilder()
      .withName(driverPod.getMetadata.getName)
      .withUid(driverPod.getMetadata.getUid)
      .withApiVersion(driverPod.getApiVersion)
      .withKind(driverPod.getKind)
      .withController(true)
      .build()
    sslSecret.foreach(secret => {
      val updatedSecret = kubernetesClient.secrets().withName(secret.getMetadata.getName).edit()
        .editMetadata()
        .addToOwnerReferences(driverPodOwnerRef)
        .endMetadata()
        .done()
      kubernetesResourceCleaner.registerOrUpdateResource(updatedSecret)
    })
    val updatedSubmitServerSecret = kubernetesClient
      .secrets()
      .withName(submitServerSecret.getMetadata.getName)
      .edit()
        .editMetadata()
          .addToOwnerReferences(driverPodOwnerRef)
          .endMetadata()
        .done()
    kubernetesResourceCleaner.registerOrUpdateResource(updatedSubmitServerSecret)
    val updatedService = kubernetesClient
      .services()
      .withName(driverService.getMetadata.getName)
      .edit()
        .editMetadata()
          .addToOwnerReferences(driverPodOwnerRef)
          .endMetadata()
        .done()
    kubernetesResourceCleaner.registerOrUpdateResource(updatedService)
    updatedService
  }

  private def waitForReadyKubernetesComponents(
      kubernetesClient: KubernetesClient,
      endpointsReadyFuture: SettableFuture[Endpoints],
      serviceReadyFuture: SettableFuture[Service],
      podReadyFuture: SettableFuture[Pod]) = {
    try {
      podReadyFuture.get(driverSubmitTimeoutSecs, TimeUnit.SECONDS)
      logInfo("Driver pod successfully created in Kubernetes cluster.")
    } catch {
      case e: Throwable =>
        val finalErrorMessage: String = buildSubmitFailedErrorMessage(kubernetesClient, e)
        logError(finalErrorMessage, e)
        throw new SparkException(finalErrorMessage, e)
    }
    try {
      serviceReadyFuture.get(driverSubmitTimeoutSecs, TimeUnit.SECONDS)
      logInfo("Driver service created successfully in Kubernetes.")
    } catch {
      case e: Throwable =>
        throw new SparkException(s"The driver service was not ready" +
          s" in $driverSubmitTimeoutSecs seconds.", e)
    }
    try {
      endpointsReadyFuture.get(driverSubmitTimeoutSecs, TimeUnit.SECONDS)
      logInfo("Driver endpoints ready to receive application submission")
    } catch {
      case e: Throwable =>
        throw new SparkException(s"The driver service endpoint was not ready" +
          s" in $driverSubmitTimeoutSecs seconds.", e)
    }
  }

  private def createDriverPod(
      kubernetesClient: KubernetesClient,
      driverKubernetesSelectors: Map[String, String],
      customAnnotations: Map[String, String],
      submitServerSecret: Secret,
      sslConfiguration: DriverSubmitSslConfiguration): Pod = {
    val containerPorts = buildContainerPorts()
    val probePingHttpGet = new HTTPGetActionBuilder()
      .withScheme(if (sslConfiguration.enabled) "HTTPS" else "HTTP")
      .withPath("/v1/submissions/ping")
      .withNewPort(SUBMISSION_SERVER_PORT_NAME)
      .build()
    val driverCpuQuantity = new QuantityBuilder(false)
      .withAmount(driverCpuCores)
      .build()
    val driverMemoryQuantity = new QuantityBuilder(false)
      .withAmount(s"${driverContainerMemoryMb}M")
      .build()
    val driverMemoryLimitQuantity = new QuantityBuilder(false)
      .withAmount(s"${driverContainerMemoryWithOverhead}M")
      .build()
    val driverPod = kubernetesClient.pods().createNew()
      .withNewMetadata()
        .withName(kubernetesAppId)
        .withLabels(driverKubernetesSelectors.asJava)
        .withAnnotations(customAnnotations.asJava)
        .endMetadata()
      .withNewSpec()
        .withRestartPolicy("Never")
        .addNewVolume()
          .withName(SUBMISSION_APP_SECRET_VOLUME_NAME)
          .withNewSecret()
            .withSecretName(submitServerSecret.getMetadata.getName)
            .endSecret()
          .endVolume()
        .addToVolumes(sslConfiguration.sslPodVolume.toSeq: _*)
        .withServiceAccount(serviceAccount.getOrElse("default"))
        .addNewContainer()
          .withName(DRIVER_CONTAINER_NAME)
          .withImage(driverDockerImage)
          .withImagePullPolicy("IfNotPresent")
          .addNewVolumeMount()
            .withName(SUBMISSION_APP_SECRET_VOLUME_NAME)
            .withMountPath(secretDirectory)
            .withReadOnly(true)
            .endVolumeMount()
          .addToVolumeMounts(sslConfiguration.sslPodVolumeMount.toSeq: _*)
          .addNewEnv()
            .withName(ENV_SUBMISSION_SECRET_LOCATION)
            .withValue(s"$secretDirectory/$SUBMISSION_APP_SECRET_NAME")
            .endEnv()
          .addNewEnv()
            .withName(ENV_SUBMISSION_SERVER_PORT)
            .withValue(SUBMISSION_SERVER_PORT.toString)
            .endEnv()
          // Note that SPARK_DRIVER_MEMORY only affects the REST server via spark-class.
          .addNewEnv()
            .withName(ENV_DRIVER_MEMORY)
            .withValue(driverSubmitServerMemoryString)
            .endEnv()
          .addToEnv(sslConfiguration.sslPodEnvVars: _*)
          .withNewResources()
            .addToRequests("cpu", driverCpuQuantity)
            .addToLimits("cpu", driverCpuQuantity)
            .addToRequests("memory", driverMemoryQuantity)
            .addToLimits("memory", driverMemoryLimitQuantity)
            .endResources()
          .withPorts(containerPorts.asJava)
          .withNewReadinessProbe().withHttpGet(probePingHttpGet).endReadinessProbe()
          .endContainer()
        .endSpec()
      .done()
    kubernetesResourceCleaner.registerOrUpdateResource(driverPod)
    driverPod
  }

   private def createDriverServiceTemplate(driverKubernetesSelectors: Map[String, String])
      : ServiceBuilder = {
    val driverSubmissionServicePort = new ServicePortBuilder()
      .withName(SUBMISSION_SERVER_PORT_NAME)
      .withPort(SUBMISSION_SERVER_PORT)
      .withNewTargetPort(SUBMISSION_SERVER_PORT)
      .build()
    new ServiceBuilder()
      .withNewMetadata()
        .withName(kubernetesAppId)
        .withLabels(driverKubernetesSelectors.asJava)
        .endMetadata()
      .withNewSpec()
        .withSelector(driverKubernetesSelectors.asJava)
        .withPorts(driverSubmissionServicePort)
        .endSpec()
  }

  private class DriverPodReadyWatcher(resolvedDriverPod: SettableFuture[Pod]) extends Watcher[Pod] {
    override def eventReceived(action: Action, pod: Pod): Unit = {
      if ((action == Action.ADDED || action == Action.MODIFIED)
          && pod.getStatus.getPhase == "Running"
          && !resolvedDriverPod.isDone) {
        pod.getStatus
          .getContainerStatuses
          .asScala
          .find(status =>
            status.getName == DRIVER_CONTAINER_NAME && status.getReady)
          .foreach { _ => resolvedDriverPod.set(pod) }
      }
    }

    override def onClose(cause: KubernetesClientException): Unit = {
      logDebug("Driver pod readiness watch closed.", cause)
    }
  }

  private class DriverEndpointsReadyWatcher(resolvedDriverEndpoints: SettableFuture[Endpoints])
      extends Watcher[Endpoints] {
    override def eventReceived(action: Action, endpoints: Endpoints): Unit = {
      if ((action == Action.ADDED) || (action == Action.MODIFIED)
          && endpoints.getSubsets.asScala.nonEmpty
          && endpoints.getSubsets.asScala.exists(_.getAddresses.asScala.nonEmpty)
          && !resolvedDriverEndpoints.isDone) {
        resolvedDriverEndpoints.set(endpoints)
      }
    }

    override def onClose(cause: KubernetesClientException): Unit = {
      logDebug("Driver endpoints readiness watch closed.", cause)
    }
  }

  private class DriverServiceReadyWatcher(resolvedDriverService: SettableFuture[Service])
      extends Watcher[Service] {
    override def eventReceived(action: Action, service: Service): Unit = {
      if ((action == Action.ADDED) || (action == Action.MODIFIED)
          && !resolvedDriverService.isDone) {
        resolvedDriverService.set(service)
      }
    }

    override def onClose(cause: KubernetesClientException): Unit = {
      logDebug("Driver service readiness watch closed.", cause)
    }
  }

  private def buildSubmitFailedErrorMessage(
      kubernetesClient: KubernetesClient,
      e: Throwable): String = {
    val driverPod = try {
      kubernetesClient.pods().withName(kubernetesAppId).get()
    } catch {
      case throwable: Throwable =>
        logError(s"Timed out while waiting $driverSubmitTimeoutSecs seconds for the" +
          " driver pod to start, but an error occurred while fetching the driver" +
          " pod's details.", throwable)
        throw new SparkException(s"Timed out while waiting $driverSubmitTimeoutSecs" +
          " seconds for the driver pod to start. Unfortunately, in attempting to fetch" +
          " the latest state of the pod, another error was thrown. Check the logs for" +
          " the error that was thrown in looking up the driver pod.", e)
    }
    val topLevelMessage = s"The driver pod with name ${driverPod.getMetadata.getName}" +
      s" in namespace ${driverPod.getMetadata.getNamespace} was not ready in" +
      s" $driverSubmitTimeoutSecs seconds."
    val podStatusPhase = if (driverPod.getStatus.getPhase != null) {
      s"Latest phase from the pod is: ${driverPod.getStatus.getPhase}"
    } else {
      "The pod had no final phase."
    }
    val podStatusMessage = if (driverPod.getStatus.getMessage != null) {
      s"Latest message from the pod is: ${driverPod.getStatus.getMessage}"
    } else {
      "The pod had no final message."
    }
    val failedDriverContainerStatusString = driverPod.getStatus
      .getContainerStatuses
      .asScala
      .find(_.getName == DRIVER_CONTAINER_NAME)
      .map(status => {
        val lastState = status.getState
        if (lastState.getRunning != null) {
          "Driver container last state: Running\n" +
            s"Driver container started at: ${lastState.getRunning.getStartedAt}"
        } else if (lastState.getWaiting != null) {
          "Driver container last state: Waiting\n" +
            s"Driver container wait reason: ${lastState.getWaiting.getReason}\n" +
            s"Driver container message: ${lastState.getWaiting.getMessage}\n"
        } else if (lastState.getTerminated != null) {
          "Driver container last state: Terminated\n" +
            s"Driver container started at: ${lastState.getTerminated.getStartedAt}\n" +
            s"Driver container finished at: ${lastState.getTerminated.getFinishedAt}\n" +
            s"Driver container exit reason: ${lastState.getTerminated.getReason}\n" +
            s"Driver container exit code: ${lastState.getTerminated.getExitCode}\n" +
            s"Driver container message: ${lastState.getTerminated.getMessage}"
        } else {
          "Driver container last state: Unknown"
        }
      }).getOrElse("The driver container wasn't found in the pod; expected to find" +
      s" container with name $DRIVER_CONTAINER_NAME")
    s"$topLevelMessage\n" +
      s"$podStatusPhase\n" +
      s"$podStatusMessage\n\n$failedDriverContainerStatusString"
  }

  private def buildContainerPorts(): Seq[ContainerPort] = {
    Seq((DRIVER_PORT_NAME, sparkConf.getInt("spark.driver.port", DEFAULT_DRIVER_PORT)),
      (BLOCK_MANAGER_PORT_NAME,
        sparkConf.getInt("spark.blockManager.port", DEFAULT_BLOCKMANAGER_PORT)),
      (SUBMISSION_SERVER_PORT_NAME, SUBMISSION_SERVER_PORT),
      (UI_PORT_NAME, uiPort)).map(port => new ContainerPortBuilder()
        .withName(port._1)
        .withContainerPort(port._2)
        .build())
  }

  private def buildSubmissionRequest(
      submitterLocalFiles: Iterable[String],
      submitterLocalJars: Iterable[String],
      driverPodKubernetesCredentials: KubernetesCredentials): KubernetesCreateSubmissionRequest = {
    val mainResourceUri = Utils.resolveURI(mainAppResource)
    val resolvedAppResource: AppResource = Option(mainResourceUri.getScheme)
        .getOrElse("file") match {
      case "file" =>
        val appFile = new File(mainResourceUri.getPath)
        val fileBytes = Files.toByteArray(appFile)
        val fileBase64 = Base64.encodeBase64String(fileBytes)
        UploadedAppResource(resourceBase64Contents = fileBase64, name = appFile.getName)
      case "local" => ContainerAppResource(mainAppResource)
      case other => RemoteAppResource(other)
    }
    val uploadFilesBase64Contents = CompressionUtils.createTarGzip(submitterLocalFiles.map(
      Utils.resolveURI(_).getPath))
    val uploadJarsBase64Contents = CompressionUtils.createTarGzip(submitterLocalJars.map(
      Utils.resolveURI(_).getPath))
    KubernetesCreateSubmissionRequest(
      appResource = resolvedAppResource,
      mainClass = mainClass,
      appArgs = appArgs,
      secret = secretBase64String,
      sparkProperties = sparkConf.getAll.toMap,
      uploadedJarsBase64Contents = uploadJarsBase64Contents,
      uploadedFilesBase64Contents = uploadFilesBase64Contents,
      driverPodKubernetesCredentials = driverPodKubernetesCredentials)
  }

  private def buildDriverSubmissionClient(
      kubernetesClient: KubernetesClient,
      driverServiceManager: DriverServiceManager,
      service: Service,
      sslConfiguration: DriverSubmitSslConfiguration): KubernetesSparkRestApi = {
    val serviceUris = driverServiceManager.getDriverServiceSubmissionServerUris(service)
    require(serviceUris.nonEmpty, "No uris found to contact the driver!")
    HttpClientUtil.createClient[KubernetesSparkRestApi](
      uris = serviceUris,
      maxRetriesPerServer = 10,
      sslSocketFactory = sslConfiguration
        .driverSubmitClientSslContext
        .getSocketFactory,
      trustContext = sslConfiguration
        .driverSubmitClientTrustManager
        .orNull,
      connectTimeoutMillis = 5000)
  }

  private def parseKeyValuePairs(
      maybeKeyValues: Option[String],
      configKey: String,
      keyValueType: String): Map[String, String] = {
    maybeKeyValues.map(keyValues => {
      keyValues.split(",").map(_.trim).filterNot(_.isEmpty).map(keyValue => {
        keyValue.split("=", 2).toSeq match {
          case Seq(k, v) =>
            (k, v)
          case _ =>
            throw new SparkException(s"Custom $keyValueType set by $configKey must be a" +
              s" comma-separated list of key-value pairs, with format <key>=<value>." +
              s" Got value: $keyValue. All values: $keyValues")
        }
      }).toMap
    }).getOrElse(Map.empty[String, String])
  }

  private def getDriverServiceManager: DriverServiceManager = {
    val driverServiceManagerLoader = ServiceLoader.load(classOf[DriverServiceManager])
    val matchingServiceManagers = driverServiceManagerLoader
      .iterator()
      .asScala
      .filter(_.getServiceManagerType == driverServiceManagerType)
      .toList
    require(matchingServiceManagers.nonEmpty,
      s"No driver service manager found matching type $driverServiceManagerType")
    require(matchingServiceManagers.size == 1, "Multiple service managers found" +
      s" matching type $driverServiceManagerType, got: " +
      matchingServiceManagers.map(_.getClass).toList.mkString(","))
    matchingServiceManagers.head
  }
}
```