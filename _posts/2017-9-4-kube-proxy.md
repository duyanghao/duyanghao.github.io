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

`kube-proxy`的作用主要是负责service的实现，`kube-proxy`运行在每个节点上，监听 API Server 中服务对象的变化，通过管理 iptables 来实现网络的转发。目前有`userspace`和`iptables`两种实现方式：

* `userspace` 是在用户空间监听一个端口，所有的 service 都转发到这个端口，然后 `kube-proxy` 在内部应用层对其进行转发。因为是在用户空间进行转发，所以效率也不高

*  `iptables` 完全用 `iptables` 来实现 service，是目前默认的方式，也是推荐的方式，效率很高（只有内核中 `netfilter` 一些损耗）

这篇文章通过 `iptables` 模式运行 `kube-proxy`，后面的分析也是针对这个模式的，`userspace` 只是旧版本支持的模式，以后可能会放弃维护和支持

`iptables`的方式是利用了linux的`iptables`的`nat转发`进行实现。在本例中，创建了名为`mysql-service`的service:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: mysql
    role: service
  name: mysql-service
spec:
  ports:
    - port: 3306
      targetPort: 3306
      nodePort: 30964
  type: NodePort
  selector:
    mysql-service: "true"
```

mysql-service对应的nodePort暴露出来的端口为30964，对应的cluster IP(10.254.162.44)的端口为3306，进一步对应于后端的pod的端口为3306

mysql-service后端代理了两个pod，ip分别是192.168.125.129和192.168.125.131。先来看一下iptables

```bash
[root@localhost ~]# iptables -S -t nat
...
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/mysql-service:" -m tcp --dport 30964 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/mysql-service:" -m tcp --dport 30964 -j KUBE-SVC-67RL4FN6JRUPOJYM
-A KUBE-SEP-ID6YWIT3F6WNZ47P -s 192.168.125.129/32 -m comment --comment "default/mysql-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ID6YWIT3F6WNZ47P -p tcp -m comment --comment "default/mysql-service:" -m tcp -j DNAT --to-destination 192.168.125.129:3306
-A KUBE-SEP-IN2YML2VIFH5RO2T -s 192.168.125.131/32 -m comment --comment "default/mysql-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-IN2YML2VIFH5RO2T -p tcp -m comment --comment "default/mysql-service:" -m tcp -j DNAT --to-destination 192.168.125.131:3306
-A KUBE-SERVICES -d 10.254.162.44/32 -p tcp -m comment --comment "default/mysql-service: cluster IP" -m tcp --dport 3306 -j KUBE-SVC-67RL4FN6JRUPOJYM
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
-A KUBE-SVC-67RL4FN6JRUPOJYM -m comment --comment "default/mysql-service:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ID6YWIT3F6WNZ47P
-A KUBE-SVC-67RL4FN6JRUPOJYM -m comment --comment "default/mysql-service:" -j KUBE-SEP-IN2YML2VIFH5RO2T
```

下面来逐条分析

首先如果是通过node的30964端口访问，则会进入到以下链:

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/mysql-service:" -m tcp --dport 30964 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/mysql-service:" -m tcp --dport 30964 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

然后进一步跳转到`KUBE-SVC-67RL4FN6JRUPOJYM`的链:

```
-A KUBE-SVC-67RL4FN6JRUPOJYM -m comment --comment "default/mysql-service:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-ID6YWIT3F6WNZ47P
-A KUBE-SVC-67RL4FN6JRUPOJYM -m comment --comment "default/mysql-service:" -j KUBE-SEP-IN2YML2VIFH5RO2T
```

这里利用了iptables的`--probability`的特性，使连接有50%的概率进入到`KUBE-SEP-ID6YWIT3F6WNZ47P`链，50%的概率进入到`KUBE-SEP-IN2YML2VIFH5RO2T`链

`KUBE-SEP-ID6YWIT3F6WNZ47P`的链的具体作用就是将请求通过DNAT发送到192.168.125.129的3306端口:

```
-A KUBE-SEP-ID6YWIT3F6WNZ47P -s 192.168.125.129/32 -m comment --comment "default/mysql-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-ID6YWIT3F6WNZ47P -p tcp -m comment --comment "default/mysql-service:" -m tcp -j DNAT --to-destination 192.168.125.129:3306
```

同理`KUBE-SEP-IN2YML2VIFH5RO2T`的作用是通过DNAT发送到192.168.125.131的3306端口:

```
-A KUBE-SEP-IN2YML2VIFH5RO2T -s 192.168.125.131/32 -m comment --comment "default/mysql-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-IN2YML2VIFH5RO2T -p tcp -m comment --comment "default/mysql-service:" -m tcp -j DNAT --to-destination 192.168.125.131:3306
```

分析完nodePort的工作方式，接下里说一下clusterIP的访问方式。 对于直接访问cluster IP(10.254.162.44)的3306端口会直接跳转到`KUBE-SVC-67RL4FN6JRUPOJYM`:

```
-A KUBE-SERVICES -d 10.254.162.44/32 -p tcp -m comment --comment "default/mysql-service: cluster IP" -m tcp --dport 3306 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

接下来的跳转方式同上文，这里就不再赘述了

## cni

