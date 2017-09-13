---
layout: post
title: Spark Memory Management
date: 2017-9-8 17:10:31
category: 技术
tags: Java Spark
excerpt: Spark内存管理……
---

## Spark Memory Management

Starting Apache Spark version 1.6.0, memory management model has changed. The old memory management model is implemented by [StaticMemoryManager](https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/StaticMemoryManager.scala) class, and now it is called “legacy”. “Legacy” mode is disabled by default, which means that running the same code on Spark 1.5.x and 1.6.0 would result in different behavior, be careful with that. For compatibility, you can enable the “legacy” model with `spark.memory.useLegacyMode` parameter, which is turned off by default.

Previously I have described the “legacy” model of memory management in this [article about Spark Architecture](https://0x0fff.com/spark-architecture/) almost one year ago. Also I have written an article on [Spark Shuffle implementations](https://0x0fff.com/spark-architecture-shuffle/) that briefly touches memory management topic as well.

This article describes new memory management model used in Apache Spark starting version 1.6.0, which is implemented as [UnifiedMemoryManager](https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala).

Long story short, new memory management model looks like this:

![](/public/img/java/Spark-Memory-Management-1.6.0-768x808.png)
Apache Spark Unified Memory Manager introduced in v1.6.0+

You can see 3 main memory regions on the diagram:







