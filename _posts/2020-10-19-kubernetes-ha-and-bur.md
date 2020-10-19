---
layout: post
title: 腾讯云十年乘风破浪直播-Kubernetes集群高可用&备份还原概述 回顾
date: 2020-10-19 19:10:31
category: 技术
tags: Kubernetes high-availability bur
excerpt: 本文为我在腾讯云十年乘风破浪主题直播分享中的回顾
---

## 前言

大家好，我叫杜杨浩，今天很高兴在这里给大家分享一下Kubernetes集群高可用&备份还原方案。无论是高可用还是备份还原都是生产环境所必须具备的特性，本次分享将依次介绍我在实现这些特性过程中遇到的问题，以及相应的思考和解决方案。

## Kubernetes高可用实战

对于Kubernetes集群高可用，这里主要指母机宕机情况下的高可用方案。我将依次从高可用架构，母机宕机下的网络影响，存储影响，应用高可用以及Pod驱逐(服务恢复)等角度对集群高可用进行系统阐述，并在最后给出一个实战总结。

### 高可用架构

* control plane node

如下给出了Kubernetes社区采用kubeadm搭建的3节点高可用集群架构图([Stacked etcd topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/#stacked-etcd-topology))：

![](/public/img/kubernetes_ha/kubernetes-ha.svg)

该方案中，所有管理节点都部署kube-apiserver，kube-controller-manager，kube-scheduler，以及etcd等组件；kube-apiserver均与本地的etcd进行通信，etcd在三个节点间同步数据并构成高可用集群；而kube-controller-manager和kube-scheduler也只与本地的kube-apiserver进行通信(或者通过LB访问)

kube-apiserver前面顶一个LB；work节点kubelet以及kube-proxy组件对接LB访问apiserver

在这种架构中，如果其中任意一个master节点宕机了，由于kube-controller-manager以及kube-scheduler基于[Leader Election Mechanism](https://github.com/kubernetes/client-go/tree/master/examples/leader-election)实现了高可用，可以认为管理集群不受影响，相关组件依旧正常运行(在分布式锁释放后)

* work node

工作节点上部署应用，应用按照[反亲和](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#more-practical-use-cases)部署多个副本，使副本调度在不同的work node上，这样如果其中一个副本所在的母机宕机了，理论上通过Kubernetes service可以把请求切换到另外副本上，使服务依旧可用

### 母机宕机下的网络影响

#### 问题1：service backend剔除

首先我们介绍一下母机宕机下的网络影响。如下给出了Kubernetes service的[两种代理模式](https://kubernetes.io/docs/concepts/services-networking/service/#proxy-mode-iptables)：

![](/public/img/share_review/service_proxy.png)

这两种代理模式都采用了linux kernel netfilter hook function。区别在于iptables代理模式使用的是iptables的DNAT规则实现service的负载均衡；而IPVS底层使用了hash数据结构，通过调用netlink接口创建IPVS规则实现负载均衡。与iptables相比，IPVS并且具备更低的网络延时，更高的规则同步效率以及更丰富的负载均衡选项

我们知道Kubernetes是通过节点心跳来保证节点健康状态的，每个node在kube-node-lease namespace下会对应一个Lease object，kubelet每隔node-status-update-frequency时间(默认10s)会更新对应node的Lease object，超过一定时间没有更新，node-controller会将节点标记为ConditionUnknown状态，并将该母机上的pod从相应service的后端列表中给剔除掉

也就是说在node-monitor-grace-period(默认40s)时间内，iptables代理模式下的k8s service对应的ep不会剔除宕机母机上的pod，访问会出现间歇性问题(iptables轮询机制)；而IPVS模式具备健康检查能力，能够自动从IPVS规则中及时剔除故障的pod，具备更高的可用性

#### 解决方案

对于这个问题，我们可以尝试调整Kubernetes参数将访问异常间隔缩短：

```
(kubelet)node-status-update-frequency：default 10s
(kube-controller-manager)node-monitor-period：default 5s
(kube-controller-manager)node-monitor-grace-period：default 40s 
Amount of time which we allow running Node to be unresponsive before marking it unhealthy. Must be N times more than kubelet's nodeStatusUpdateFrequency, where N means number of retries allowed for kubelet to post node status. 
- Currently nodeStatusUpdateRetry is constantly set to 5 in kubelet.go
```

另外，可以将service代理模式从iptables切换成IPVS，同时给service设置健康检查机制及时剔除异常pod backend

#### 问题2: TCP长连接超时

接下来我们探讨一下母机宕机下的另外一个问题：TCP长连接超时。之前我写过一篇文章详细描述这个问题，感兴趣的读者可以深入看看[Kubernetes Controller高可用诡异的15mins超时](https://duyanghao.github.io/kubernetes-ha-http-keep-alive-bugs/)

![](/public/img/share_review/controller-tcp-timeout.png)

可以看到上图中两个母机分别部署了Controller和Aggregated APIServer服务，Controller通过service访问AA(Aggregated APIServer)，且两台母机上Controller都是访问的Node2的AA pod，假设在某一时刻Node2宕机了，按照正常的理解，Node1上的Controller会在分布式锁被释放后获取锁并启动正常逻辑

但是观察到的现象是Node1上的Controller会一直尝试获取锁，并且超时，整个过程持续15mins左右。

对于这个问题的原因可以归纳如下：

* 母机宕机后，无法及时发送RST包给请求对端
* HTTP/2在请求超时后并不会关闭底层TCP连接，而是会继续复用(HTTP/1在请求超时后关闭TCP连接)。TCP ARQ机制导致了上述的15mins超时
* 本质是应用没有对TCP Socket设置超时&健康检查

Kubernetes集群中几乎所有使用client-go的应用([kubelet](https://github.com/kubernetes/kubernetes/pull/63492)除外)都会出现上述15mins超时问题。虽然可以简单采用禁用HTTP/2切换HTTP/1同时设置请求超时的方法进行规避，但却无法解决推送类服务(Watch)的超时问题(如果设置了超时，正常情况下Watch会超时)

而Kubernetes社区也对应存在着类似的[client-go issue](https://github.com/kubernetes/client-go/issues/374)，至今依然处于Open的状态，问题一直没有解决

#### 解决方案

对于这个问题，我们可以从多个角度进行思考和解决：

* 代码角度

从代码角度来看，我们可以通过对应用层使用超时设置或者健康检查机制，从上层保障连接的健康状态 - 作用于该应用

举例来说，对于HTTP/1的应用设置请求超时；而对于HTTP/2的应用，为了解决HTTP/2无法及时移除异常连接的问题，我分别给[golang/net](https://github.com/golang/net/pull/84)以及[k8s.io/apimachinery](https://github.com/kubernetes/kubernetes/pull/94844)提交了PR，目前均等待Merged中

* 工程角度

从工程的角度，一方面我们可以调整TCP ARQ&keepalive参数来缩短异常连接关闭的时间：

```
# /etc/sysctl.conf
net.ipv4.tcp_keepalive_time=30
net.ipv4.tcp_keepalive_intvl=30 
net.ipv4.tcp_keepalive_probes=5 
net.ipv4.tcp_retries2=9
$ /sbin/sysctl -p 
```

在Kubernetes环境中，由于容器不会直接继承母机tcp keepalive的配置(可以直接继承母机tcp超时重试的配置)，因此必须通过一定方式进行适配。这里介绍其中一种方式，添加initContainers使配置生效：

```
- name: init-sysctl
  image: busybox
  command:
  - /bin/sh
  - -c
  - |
    sysctl -w net.ipv4.tcp_keepalive_time=30
    sysctl -w net.ipv4.tcp_keepalive_intvl=30
    sysctl -w net.ipv4.tcp_keepalive_probes=5
  securityContext:
    privileged: true
```

另一方面，我们可以采用调整服务部署架构规避该问题。这里介绍其中一种方案 - Kubernetes服务拓扑感知：

这里利用了Kubernetes服务拓扑感知规则[“kubernetes.io/hostname”，“*”]：优先使用同一节点上的端点，如果该节点上没有可用端点，则回退到任何可用端点上

通过如上的规则，我们将client和server进行捆绑安装，同一个母机上的client永远只访问本地的server，如下：

![](/public/img/share_review/service_topology.png)

这样即便存在母机宕机，对其它母机上的服务也没有任何影响

### 母机宕机下的存储影响

### 母机宕机下的应用高可用

### 母机宕机下的Pod驱逐

### 高可用总结

## Kubernetes备份还原实战

## 总结

## 参考

* [Kubernetes高可用实战干货](https://duyanghao.github.io/kubernetes-ha/)
* [Kubernetes Controller高可用诡异的15mins超时](https://duyanghao.github.io/kubernetes-ha-http-keep-alive-bugs/)
* [Kubernetes备份还原概述](https://duyanghao.github.io/kubernetes-bur/)
