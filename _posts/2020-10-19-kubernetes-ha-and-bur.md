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

首先我们介绍一下母机宕机下的网络影响。如下给出了Kubernetes service的两种代理模式：

![](/public/img/share_review/service_proxy.png)



#### 问题1：service backend剔除

#### 解决方案

#### 问题2: TCP长连接超时

#### 解决方案

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
