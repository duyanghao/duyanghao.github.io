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

对于在生产环境搭建的Kubernetes集群，我们可以利用[kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)搭建高可用集群，一定程度实现容灾：

![](/public/img/kubernetes_bur/kubernetes-ha.png)

但是即便实现集群的高可用，我们依旧会需要备份还原功能，主要原因如下：

* 误删除：运维人员不小心删除了某个namespace，某个pv
* 服务器死机：因为物理原因服务器损坏，或者需要重装系统
* 集群迁移：需要将一个集群的数据迁移到另一个集群，用于测试或者其它目的

而对于Kubernetes的备份和还原，社区有一个16年创建的[issue](https://github.com/kubernetes/kubernetes/issues/24229)，从这个issue中我们可以看出Kubernetes官方并不打算提供Kubernetes备份还原方案以及工具，主要原因有两点：

* 官方认为Kubernetes只提供平台，管理和运行应用(类似于操作系统)。不负责应用层面的备份还原
* 基于Kubernetes集群的应用各不相同，无法(or 不太好)抽象统一备份方案

为了解决这个问题，社区中也有一些项目产生，例如：[reshifter](https://github.com/mhausenblas/reshifter)，[kube-backup](https://github.com/pieterlange/kube-backup)以及[velero](https://github.com/vmware-tanzu/velero)。
这其中最流行的备份还原工具是Velero，基于这个工具的研究可以参考[这篇文章](https://duyanghao.github.io/velero/)

虽然社区和Google上会有一些关于Kubernetes备份还原的工具和方案，但是都杂而不全，本文就针对Kubernetes云原生备份还原这个专题进行一个全面而系统的方案介绍

# 备份什么？

对于Kubernetes搭建的集群，我们需要备份：

* 应用版本信息(Application Version)
  * 应用对应的工作负载，例如：deploymenet，statefulset以及daemonset等等
  * 应用需要使用的配置，例如：configmap，secret等等
* 应用状态信息(Application State)
  * 应用可以分为有状态应用和无状态应用
  * 有状态应用需要备份状态数据，例如：database
  
# 备份方案

针对上述备份数据，我们可以制定如下几种不同的备份还原方案（注意是逐项演进的）

## 一、ETCD+应用状态

方案要点如下：

* 应用版本备份：Kubernetes集群的所有应用版本信息都存放于ETCD中，我们可以直接备份ETCD来达到备份版本的目的
* 应用状态备份：可以从应用层或者文件系统层备份应用状态，例如MariaDB采用`mysqldump`，MongoDB采用`mongodump`等

备份流程大致如下：

* step1：备份ETCD
  ```bash
  # backup kubernetes pki
  $ cp -r /etc/kubernetes/pki backup/
  # backup kubeadm-config
  $ cp kubeadm-config.yaml backup/
  # backup etcd snapshot
  $ cat <<EOF > etcd-backup-job.yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: etcd-backup
  spec:
    template:
      spec:
        containers:
        - name: etcd-backup
          # Same image as in /etc/kubernetes/manifests/etcd.yaml
          image: k8s.gcr.io/etcd:3.2.24
          env:
          - name: ETCDCTL_API
            value: "3"
          command: ["/bin/sh"]
          args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-snapshot.db"]
          volumeMounts:
          - mountPath: /etc/kubernetes/pki/etcd
            name: etcd-certs
            readOnly: true
          - mountPath: /backup
            name: backup
        restartPolicy: OnFailure
        hostNetwork: true
        volumes:
        - name: etcd-certs
          hostPath:
            path: /etc/kubernetes/pki/etcd
            type: DirectoryOrCreate
        - name: backup
          hostPath:
            path: /tmp/etcd-backup
            type: DirectoryOrCreate
  EOF
  $ kubectl apply -f etcd-backup-job.yaml 
  $ cp /tmp/etcd-backup/etcd-snapshot.db backup/ 
  ```

* step2：应用层备份(eg: MariaDB)
  ```bash
  # backup
  $ mysqldump -uxxx -pxxx db_name > backup-file.sql
  ```

还原流程大致如下：

* step1：还原ETCD
  ```bash
  # restore kubernetes pki
  $ cp -r backup/pki /etc/kubernetes/
  # restore kubeadm-config
  $ cp backup/kubeadm-config.yaml ./
  # initialize the first control plane node with backup kubernetes pki and kubeadm-config
  $ kubeadm init --config ./kubeadm-config.yaml
  # join other control plane nodes
  $ kubeadm join ...
  # restore etcd with snapshot(Refs: https://duyanghao.github.io/etcd-bur/)
  ```
  
* step2：清空数据
  ```bash
  # restore-phase-1(drop db)
  $ mysqladmin -uxxx -pxxx -f drop db_name
  Database "xxx" dropped
  
  # restore-phase-2(recreate db)
  $ mysqladmin -uxxx -pxxx create db_name  
  ```

* step3：还原数据
  ```bash
  # restore-phase-3(restore db)
  $ mysql -uxxx -pxxx db_name < backup-file.sql
  ```

## 应用版本+应用状态



## VC+应用状态

# 集群分类

我们按照集群的作用可以将集群分类如下：

* 管理集群：负责管理客户集群，主要部署自研的管理组件
  * 特点：组件可能经常升级，内部可以管控部署组件；应用数据量小
* 客户集群：客户实际部署应用的集群
  * 特点：无法管控客户部署的应用；应用数据量大

针对这两种集群类型，我们分别设计备份还原方案

# 客户集群备份还原方案

由于我们无法管控客户集群中部署的应用类型，所以最好的

# 管理集群备份还原方案

# 结论

# Refs

* [The Ultimate Guide to Disaster Recovery for Your Kubernetes Clusters](https://medium.com/velotio-perspectives/the-ultimate-guide-to-disaster-recovery-for-your-kubernetes-clusters-94143fcc8c1e)
* [Backup Kubernetes – how and why](https://elastisys.com/2018/12/10/backup-kubernetes-how-and-why/)
* [Backup and Restore a Kubernetes Master with Kubeadm](https://labs.consol.de/kubernetes/2018/05/25/kubeadm-backup.html)
* [Backup/migrate cluster?](https://github.com/kubernetes/kubernetes/issues/24229)
* [Creating Highly Available clusters with kubeadm](https://v1-15.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
* [Use an HTTP Proxy to Access the Kubernetes API](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/)
* [Kubernetes Component Status API](https://supereagle.github.io/2019/05/26/k8s-health-monitoring/)
