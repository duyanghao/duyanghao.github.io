---
layout: post
title: Service Discovery Using Kubernetes
date: 2017-9-5 23:10:31
category: 技术
tags: Kubernetes Docker Microservices
excerpt: Kubernetes服务发现……
---

## 服务发现

来自[Wikipedia](https://en.wikipedia.org/wiki/Service_discovery)的定义：

>>Service discovery is the automatic detection of devices and services offered by these devices on a computer network. A service discovery protocol (SDP) is a network protocol that helps accomplish service discovery.Service discovery requires a common language to allow software agents to make use of one another's services without the need for continuous user intervention.

来自[Encyclopedia](https://www.pcmag.com/encyclopedia/term/51181/service-discovery)定义：

>>The capability of automatically identifying a hardware or software service in a network such as a scanner, printer, Web server or shared file. Discovery systems either use central depositories where services are registered, or they provide a method for querying every device on the network. See Zeroconf, UPnP, UDDI, SLP and Web services.

服务发现组件记录了（大规模）分布式系统中所有服务的信息，人们或者其它服务可以据此找到这些服务。 DNS 就是一个简单的例子。当然，复杂系统的服务发现组件要提供更多的功能，例如，服务元数据存储、健康监控、多种查询和实时更新等

不同的使用情境，服务发现的含义也不同。例如，网络设备发现、零配置网络（ rendezvous ）发现和 SOA 发现等。无论是哪一种使用情境，服务发现提供了一种协调机制，方便服务的发布和查找

## 几种常用的服务发现机制

* 人们已经使用 DNS 很长时间了， DNS 可能是现有的最大的服务发现系统。小规模系统可以先使用 DNS 作为服务发现手段。一旦服务节点的启动和销毁变得更加动态， DNS 就有问题了，因为 DNS 记录传播的速度可能跟不上服务节点变化的速度
* ZooKeeper 大概是最成熟的配置存储方案，它的历史比较长，提供了包括配置管理、领导人选举和分布式锁在内的完整解决方案。因此， ZooKeeper 是非常有竞争力的通用的服务发现解决方案，当然，它也显得过于复杂
* etcd 和 doozerd 是新近出现的服务发现解决方案，它们与 ZooKeeper 具有相似的架构和功能，因此可与 ZooKeeper 互换使用
* Consul 是一种更新的服务发现解决方案。除了服务发现，它还提供了配置管理和一种键值存储。 Consul 提供了服务节点的健康检查功能，支持使用 DNS SRV 查找服务，这大大增强了它与其它系统的互操作性。 Consul 与 ZooKeeper 的主要区别是： Consul 提供了 DNS 和 HTTP 两种 API ，而 ZooKeeper 只支持专门客户端的访问

本文围绕`Kubernetes`展开讲述几种常用的服务发现机制

## Kubernetes服务发现

kubernetes 提供了 service 的概念可以通过 VIP 访问 pod 提供的服务，但是在使用的时候还有一个问题：怎么知道某个应用的 VIP？比如我们有两个应用，一个 app，一个 是 db，每个应用使用 rc 进行管理，并通过 service 暴露出端口提供服务。app 需要连接到 db 应用，我们只知道 db 应用的名称，但是并不知道它的 VIP 地址

最简单的办法是从 kubernetes 提供的 API 查询。但这是一个糟糕的做法，首先每个应用都要在启动的时候编写查询依赖服务的逻辑，这本身就是重复和增加应用的复杂度；其次这也导致应用需要依赖 kubernetes，不能够单独部署和运行（当然如果通过增加配置选项也是可以做到的，但这又是增加复杂度）

开始的时候，kubernetes 采用了 docker 使用过的方法——环境变量。每个 pod 启动时候，会把通过环境变量设置所有服务的 IP 和 port 信息，这样 pod 中的应用可以通过读取环境变量来获取依赖服务的地址信息。这种方式服务和环境变量的匹配关系有一定的规范，使用起来也相对简单，但是有个很大的问题：依赖的服务必须在 pod 启动之前就存在，不然是不会出现在环境变量中的

更理想的方案是：应用能够直接使用服务的名字，不需要关心它实际的 ip 地址，中间的转换能够自动完成。名字和 ip 之间的转换就是 DNS 系统的功能，因此 kubernetes 也提供了 DNS 方法来解决这个问题

### 部署 DNS 服务

DNS 服务不是独立的系统服务，而是一种 [addon](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns) ，作为插件来安装的，不是 kubernetes 集群必须的（但是非常推荐安装）。可以把它看做运行在集群上的应用，只不过这个应用比较特殊而已

DNS 有两种配置方式，在 1.3 之前使用 `etcd` + `kube2sky` + `skydns` 的方式，在 1.3 之后可以使用 `kubedns` + `dnsmasq` 的方式

不管以什么方式启动，对外的效果是一样的。要想使用 DNS 功能，还需要修改 kubelet 的启动配置项，告诉 kubelet，给每个启动的 pod 设置对应的 DNS 信息，一共有两个参数：--cluster_dns=10.10.10.10 --cluster_domain=cluster.local，分别是 DNS Service在集群中的 vip 和域名后缀(要和 DNS rc 中保持一致)

#### <font color="#8B0000">etcd + kube2sky + skydns部署方式</font>

下面是这种方式的部署配置文件：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
  name: kube-dns
  namespace: kube-system
spec:
  replicas: 1
  selector:
    k8s-app: kube-dns
  template:
    metadata:
      labels:
        k8s-app: kube-dns
        kubernetes.io/cluster-service: "true"
    spec:
      containers:
        - name: etcd
          command:
            - /usr/local/bin/etcd
            - "-listen-client-urls"
            - "http://127.0.0.1:2379,http://127.0.0.1:4001"
            - "-advertise-client-urls"
            - "http://127.0.0.1:2379,http://127.0.0.1:4001"
            - "-initial-cluster-token"
            - skydns-etcd
          image: "gcr.io/google_containers/etcd:2.0.9"
          resources:
            limits:
              cpu: 100m
              memory: 50Mi
        - name: kube2sky
          args:
            - "-domain=cluster.local"
            - "-kube_master_url=http://10.7.114.81:8080"
          image: "gcr.io/google_containers/kube2sky:1.11"
          resources:
            limits:
              cpu: 100m
              memory: 50Mi
        - name: skydns
          args:
            - "-machines=http://localhost:4001"
            - "-addr=0.0.0.0:53"
            - "-domain=cluster.local"
          image: "gcr.io/google_containers/skydns:2015-03-11-001"
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - "-c"
                - "nslookup kubernetes.default.svc.cluster.local localhost >/dev/null"
            initialDelaySeconds: 30
            timeoutSeconds: 5
          ports:
            - containerPort: 53
              name: dns
              protocol: UDP
            - containerPort: 53
              name: dns-tcp
              protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 50Mi
      dnsPolicy: Default
```

这里有两个需要根据实际情况配置的地方：

* `kube_master_url`： `kube2sky` 会用到 `kubernetes master API`，它会读取里面的 `service` 信息
* `domain`：域名后缀，默认是 `cluster.local`，你可以根据实际需要修改成任何合法的值

然后是 Service 的配置文件：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.10.10.10
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
```

这里需要注意的是 `clusterIP: 10.10.10.10` 这一行手动指定了 DNS service 的 IP 地址，这个地址必须在预留的 vip 网段。手动指定的原因是为了固定这个 ip，这样启动 kubelet 的时候配置 `--cluster_dns=10.10.10.10` 比较方便，不需要再动态获取 DNS 的 vip 地址

有了这两个文件，直接创建对象就行：

```bash
$ kubectl create -f ./skydns-rc.yml
$ kubectl create -f ./skydns-svc.yml
[root@localhost ~]# kubectl get svc,rc,pod --namespace=kube-system
NAME                     CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
svc/kube-dns             10.10.10.10    <none>        53/UDP                          1d

NAME          DESIRED   CURRENT   READY     AGE
rc/kube-dns   1         1         1         41m

NAME                READY     STATUS    RESTARTS   AGE
po/kube-dns-twl0q   3/3       Running   0          41m
```

#### <font color="#8B0000">kubeDNS + dnsmasq部署方式</font>

在 kubernetes 1.3 版本之后，kubernetes 改变了 DNS 的部署方式，变成了 `kubeDNS` + `dnsmasq`，没有了 `etcd` 。在这种模式下，`kubeDNS` 是原来 `kube2sky` + `skyDNS` + `etcd`，只不过它把数据都保存到自己的内存，而不是 kv store 中；`dnsmasq` 的引进是为了提高解析的速度，因为它可以配置 DNS 缓存

这种部署方式的完整配置文件参考[这里](https://gist.github.com/cizixs/1adc2ce56b8cf3c341a55bd502ccd9cd)

### 测试 DNS 可用性

不管那种部署很是，kubernetes 对外提供的 DNS 服务是一致的。每个 service 都会有对应的 DNS 记录，kubernetes 保存 DNS 记录的格式如下：

```
<service_name>.<namespace>.svc.<domain> 
```

每个部分的字段意思：

* service_name: 服务名称，就是定义 service 的时候取的名字
* namespace：service 所在 namespace 的名字
* domain：提供的域名后缀，比如默认的 `cluster.local`

在 pod 中可以通过 `service_name.namespace.svc.domain` 来访问任何的服务，也可以使用缩写 `service_name.namespace`，如果 pod 和 service 在同一个 namespace，甚至可以直接使用 `service_name`

**NOTE**：正常的 service 域名会被解析成 service vip，而 headless service 域名会被直接解析成背后的 pods ip

虽然不会经常用到，但是 pod 也会有对应的 DNS 记录，格式是 `pod-ip-address.<namespace>.pod.<domain>`，其中 `pod-ip-address` 为 `pod ip`地址用`-`符号隔开的格式，比如 pod ip 地址是 `1.2.3.4` ，那么对应的域名就是 `1-2-3-4.default.pod.cluster.local`

运行一个 busybox 来验证 DNS 服务能够正常工作：

```bash
/ # nslookup whoami
Server:    10.10.10.10
Address 1: 10.10.10.10

Name:      whoami
Address 1: 10.10.10.175

/ # nslookup whoami.default
Server:    10.10.10.10
Address 1: 10.10.10.10

Name:      whoami.default 
Address 1: 10.10.10.175

/ # nslookup whoami.default.svc
Server:    10.10.10.10
Address 1: 10.10.10.10

Name:      whoami.default.svc
Address 1: 10.10.10.175

/ # nslookup whoami.default.svc.cluster.local
Server:    10.10.10.10
Address 1: 10.10.10.10

Name:      whoami.default.svc.cluster.local
Address 1: 10.10.10.175
```

可以看出，如果我们在默认的 namespace default 创建了名为 whoami 的服务，以下所有域名都能被正确解析：

```
whoami
whoami.default
whoami.default.svc
whoami.default.svc.cluster.local
```

每个 pod 的 DNS 配置文件如下，可以看到 DNS vip 地址以及搜索的 domain 列表：

```bash
/ # cat /etc/resolv.conf
search default.pod.cluster.local default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.10.10.10
options ndots:5
options ndots:5
```

### kubernetes DNS 原理解析

我们前面介绍了两种不同 DNS 部署方式，这部分讲讲它们内部的原理：

#### <font color="#8B0000">etcd + kube2sky + skydns部署方式</font>

![](/public/img/k8s/kube2sky-arch.jpg)

这种模式下主要有三个容器在运行：

```bash
[root@localhost ~]# docker ps
CONTAINER ID        IMAGE                                              COMMAND                  CREATED             STATUS              PORTS                                          NAMES
919cbc006da2        172.16.1.41:5000/google_containers/kube2sky:1.12   "/kube2sky /kube2sky "   About an hour ago   Up About an hour                                                   k8s_kube2sky.80a41edc_kube-dns-twl0q_kube-system_ea1f5f4d-15cf-11e7-bece-080027c09e5b_1bd3fdb4
73dd11cac057        172.16.1.41:5000/jenkins/etcd:live                 "etcd -data-dir=/var/"   About an hour ago   Up About an hour                                                   k8s_etcd.4040370_kube-dns-twl0q_kube-system_ea1f5f4d-15cf-11e7-bece-080027c09e5b_b0e5a99f
0b10ae639989        172.16.1.41:5000/jenkins/skydns:20150703-113305    "bootstrap.sh"           About an hour ago   Up About an hour                                                   k8s_skydns.73baf3b1_kube-dns-twl0q_kube-system_ea1f5f4d-15cf-11e7-bece-080027c09e5b_2860aa6d
```

这三个容器的作用分别是：

* [etcd](https://github.com/coreos/etcd)：保存所有的 DNS 数据
* kube2sky： 通过 kubernetes API 监听 Service 的变化，然后同步到 etcd
* [skyDNS](https://github.com/skynetservices/skydns)：根据 etcd 中的数据，对外提供 DNS 查询服务

#### <font color="#8B0000">kubeDNS + dnsmasq部署方式</font>

![](/public/img/k8s/kubedns-arch.jpg)

这种模式下，`kubeDNS` 容器替代了原来的三个容器的功能，它会监听 apiserver 并把所有 service 和 endpoints 的结果在内存中用合适的数据结构保存起来，并对外提供 DNS 查询服务

* [kubeDNS](https://github.com/kubernetes/dns)：提供了原来 kube2sky + etcd + skyDNS 的功能，可以单独对外提供 DNS 查询服务
* [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html)： 一个轻量级的 DNS 服务软件，可以提供 DNS 缓存功能。kubeDNS 模式下，dnsmasq 在内存中预留一块大小（默认是 1G）的地方，保存当前最常用的 DNS 查询记录，如果缓存中没有要查找的记录，它会到 kubeDNS 中查询，并把结果缓存起来

#### <font color="#8B0000">health check 功能</font>

每种模式都可以运行额外的 [`exec-healthz` 容器](https://github.com/kubernetes/contrib/tree/master/exec-healthz)对外提供 `health check` 功能，证明当前 DNS 服务是正常的

### 总结

推荐使用 kubeDNS 的模式来部署，因为它有着以下的好处：

* 不需要额外的存储，省去了额外的维护和数据保存的工作
* 更好的性能。通过 dnsmasq 缓存和直接把 DNS 记录保存在内存中，来提高 DNS 解析的速度

## Refs

* [kubernetes 简介：kube-dns 和服务发现](http://cizixs.com/2017/04/11/kubernetes-intro-kube-dns)
