---
layout: post
title: 云原生备份还原方案
date: 2020-5-21 19:10:31
category: 技术
tags: Kubernetes bur
excerpt: ​本文介绍基于Kubernetes搭建的云原生平台备份还原方案
---

# Contents

- [Overview](#Overview)
- [管理集群备份还原方案](#管理集群备份还原方案)
- [客户集群备份还原方案](#客户集群备份还原方案)
- [结论](#结论)
- [Refs](#Refs)

# Overview

对于在生产环境搭建的Kubernetes集群，我们可以利用[kubeadm](Creating Highly Available clusters with kubeadm)搭建高可用集群，一定程度实现容灾：

![](/public/img/kubernetes-ha.png)

但是即便实现集群的高可用，我们依旧会需要备份还原功能，主要原因如下：

* 误删除：运维人员不小心删除了某个namespace，某个pv
* 服务器死机：因为物理原因服务器损坏，或者需要重装系统
* 集群迁移：需要将一个集群的数据迁移到另一个集群，用于测试或者其它目的

而对于Kubernetes的备份和还原，社区有一个16年创建的[issue](https://github.com/kubernetes/kubernetes/issues/24229)，从这个issue中我们可以看出Kubernetes官方并不打算
提供Kubernetes备份还原方案以及工具，主要原因有两点：

* 社区认为Kubernetes只提供平台，管理和运行应用(类似于操作系统)。不负责应用层面的备份还原
* 基于Kubernetes集群的应用各不相同，无法(or 不太好)抽象统一备份方案

为了解决这个问题，社区中也有一些项目产生，例如：[reshifter](https://github.com/mhausenblas/reshifter)，[kube-backup](https://github.com/pieterlange/kube-backup)以及[velero](https://github.com/vmware-tanzu/velero)。
这其中最流行的备份还原工具是Velero，基于这个工具的研究可以参考[这篇文章](https://duyanghao.github.io/velero/)

虽然社区和Google上会有一些关于Kubernetes备份还原的工具和方案，但是都杂而不全，本文就针对Kubernetes云原生备份还原这个专题进行一个全面而系统的方案介绍

# 集群分类



# 管理集群备份还原方案


# 客户集群备份还原方案


# 结论

# Refs

* [The Ultimate Guide to Disaster Recovery for Your Kubernetes Clusters](https://medium.com/velotio-perspectives/the-ultimate-guide-to-disaster-recovery-for-your-kubernetes-clusters-94143fcc8c1e)
* [Backup Kubernetes – how and why](https://elastisys.com/2018/12/10/backup-kubernetes-how-and-why/)
* [Backup and Restore a Kubernetes Master with Kubeadm](https://labs.consol.de/kubernetes/2018/05/25/kubeadm-backup.html)
* [Backup/migrate cluster?](https://github.com/kubernetes/kubernetes/issues/24229)
* [Creating Highly Available clusters with kubeadm](https://v1-15.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
* [Use an HTTP Proxy to Access the Kubernetes API](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/)
* [Kubernetes Component Status API](https://supereagle.github.io/2019/05/26/k8s-health-monitoring/)