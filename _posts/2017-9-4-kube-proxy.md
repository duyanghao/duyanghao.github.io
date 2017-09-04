---
layout: post
title: kube proxy
date: 2017-9-4 18:10:31
category: 技术
tags: Linux Network Kubernetes
excerpt: kubernetes网络架构……
---

## kubernetes服务架构
 
先给出`kubernetes`服务架构，如下：

![](/public/img/k8s/k8s服务架构图.png)

关于`kubernetes`服务可以参考如下[文章](http://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy)，本文主要介绍kubernetes网络架构，也即图中service的iptables部分（cni插件化）

## kube-proxy

## cni

