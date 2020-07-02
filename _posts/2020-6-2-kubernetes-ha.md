---
layout: post
title: Kubernetes高可用概述
date: 2020-7-1 19:10:31
category: 技术
tags: Kubernetes high-availability
excerpt: 本文对Kubernetes集群以及服务高可用过程中遇到的问题进行一个全面而系统的总结
---

## 前言

在企业生产环境，Kubernetes高可用是一个必不可少的特性，其中最通用的场景就是如何在Kubernetes集群宕机一个节点的情况下保障服务依旧可用

本文就针对这个场景下实现应用高可用遇到的各种问题进行一个总结

## 整体方案

本文描述的高可用方案对应架构如下：

* control plane node

  管理节点采用kubeadm搭建的3节点标准高可用方案([Stacked etcd topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology))：

  ![](/public/img/kubernetes_ha/kubeadm-ha-stacked-etcd.png)

  该方案中，所有管理节点都部署kube-apiserver，kube-controller-manager，kube-scheduler，以及etcd等组件；kube-apiserver均与本地的etcd进行通信，etcd在三个节点间同步数据；而kube-controller-manager和kube-scheduler也只与本地的kube-apiserver进行通信
  
  kube-apiserver前面顶一个LB；work节点kubelet以及kube-proxy组件对接LB
  
  在这种架构中，如果其中任意一个master节点宕机了，可以认为管理集群不受影响，相关组件依旧正常运行

* work node

  工作节点上部署应用，应用按照[反亲和](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#more-practical-use-cases)部署多个副本，使副本调度在不同的work node上，这样如果其中一个副本所在的母机宕机了，理论上通过Kubernetes service可以把请求切换到另外副本上

## 网络

针对上述架构，我们分析一下实际环境中节点宕机对网络层面带来的影响

* service backend剔除

正如上文所述，工作节点上应用会按照反亲和部署。这里我们讨论最简单的情况：例如一个nginx服务，部署了两个副本，分别落在work node1和work node2上

集群中还部署了依赖nginx服务的应用A，如果某个时刻work node1宕机了，此时应用A访问nginx service会有问题吗？

这里答案是会有问题。因为service不会马上剔除掉宕机上对应的nginx pod，同时由于service常用的[iptables和ipvs代理模式](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-iptables)都没有实现`retry with another backend`()特性，所以在一段时间内访问会出现间歇性问题(如果请求轮询到挂掉的nginx pod上)，这里会存在一个访问失败间隔期

![](/public/img/kubernetes_ha/kubernetes-iptables.png)

这个间隔期取决于service对应的endpoint什么时候踢掉宕机的pod。之后，kube-proxy会watch到endpoint变化，然后更新对应的iptables规则，使service访问后端列表恢复正常(也就是踢掉该pod)

要理解这个间隔，我们首先需要弄清楚Kubernetes节点心跳的概念：

每个node在kube-node-lease namespace下会对应一个Lease object，kubelet每隔node-status-update-frequency时间(默认10s)会更新对应node的Lease object

node-controller会每隔node-monitor-period时间(默认5s)检查Lease object是否更新，如果超过node-monitor-grace-period时间(默认40s)没有发生过更新，则认为这个node不健康，会更新NodeStatus(NotReady)。如果这样的状态在这之后持续了一段时间(默认5mins)，则会驱逐(evict)该node上的pod

对于母机宕机的场景，node controller在node-monitor-grace-period时间段没有观察到心跳的情况下，会更新NodeStatus(NotReady)，之后endpoint controller会从endpoint backend中踢掉该母机上的所有pod，最终服务访问正常，请求不会落在挂掉的pod上

也就是说在母机宕机后，在node-monitor-grace-period间隔内访问service会有间歇性问题，我们可以通过调整相关参数来缩小这个窗口期，如下：

```
(kubelet)node-status-update-frequency：default 10s
(kube-controller-manager)node-monitor-period：default 5s
(kube-controller-manager)node-monitor-grace-period：default 40s
- Amount of time which we allow running Node to be unresponsive before marking it unhealthy. Must be N times more than kubelet's nodeStatusUpdateFrequency, where N means number of retries allowed for kubelet to post node status.
- Currently nodeStatusUpdateRetry is constantly set to 5 in kubelet.go
(kube-apiserver)default-not-ready-toleration-seconds：default 300
(kube-apiserver)default-unreachable-toleration-seconds：default 300
```

* service长链接

对于使用长连接访问服务的应用来说，可能还会有其它超时问题……

## 存储

对于存储，可以分为Kubernetes系统存储和应用存储，系统存储专指etcd；而应用存储一般来说只考虑persistent volume

### 系统存储

etcd同步待研究……

### 应用存储

这里考虑存储是部署在集群外的情况(通常情况)。如果一个母机宕机了，由于没有对存储集群产生破坏，因此不会影响其它母机上应用访问pv存储

而对于Kubernetes存储本身组件功能(in-tree, flexVolume, external-storage以及csi)的影响实际上可以归纳为对Kubernetes应用的影响，这里以csi为例子进行说明：

![](/public/img/kubernetes_ha/kubernetes-csi-recommend-deploy.png)

后续在分析完应用层面的高可用方案后，我们再来看对csi组件的影响

## 应用

### Stateless Application

对于无状态的服务(通常部署为deployment工作负载)，我们可以直接通过设置反亲和+多副本来实现高可用，例如nginx服务：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
name: web-server
spec:
selector:
  matchLabels:
    app: web-store
replicas: 3
template:
  metadata:
    labels:
      app: web-store
  spec:
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web-store
          topologyKey: "kubernetes.io/hostname"
    containers:
    - name: web-app
      image: nginx:1.16-alpine
```

如果其中一个pod所在母机宕机了，则在endpoint controller踢掉该pod backend后，服务访问正常

这类服务通常依赖于其它有状态服务，例如：APIServer

### Stateful Application

对于有状态的服务，可以按照高可用类型分类如下：

* RWX Type

对于本身实现了基于存储多读写构建高可用的应用来说，我们可以直接利用反亲和+多副本进行部署(一般部署为deployment类型)，后接同一个存储(支持ReadWriteMany，例如：pv or 对象存储)，实现类似无状态服务的高可用模式

其中，docker distribution，helm chartmuseum以及harbor jobservice等都属于这种类型

* Special Type

对于特殊类型的应用，例如database，它们一般有自己定制的高可用方案，例如常用的主从模式

这类应用通常以statefulset的形式进行部署，每个副本对接一个pv，在母机宕机的情况下，由应用本身实现高可用(例如：master选举-主备切换)

其中，redis，postgres，以及各类db都基本是这种模式，如下是redis一主两从三哨模式：

![](/public/img/kubernetes_ha/redis-ha.png)

## Conclusion

……

## Refs

* [Recommended Mechanism for Deploying CSI Drivers on Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#recommended-mechanism-for-deploying-csi-drivers-on-kubernetes)
* [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec/blob/master/spec.md)
* [Client should expose a mechanism to close underlying TCP connections](https://github.com/kubernetes/client-go/issues/374)