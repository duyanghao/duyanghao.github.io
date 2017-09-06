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

### Kubernetes DNS 服务发现总结

推荐使用 kubeDNS 的模式来部署，因为它有着以下的好处：

* 不需要额外的存储，省去了额外的维护和数据保存的工作
* 更好的性能。通过 dnsmasq 缓存和直接把 DNS 记录保存在内存中，来提高 DNS 解析的速度

## 几种服务发现解决方案对比

### <font color="#8B0000">1、Manual configuration</font>

Most of the services are still managed manually. We decide in advance where to deploy the service, what is its configuration and hope beyond reason that it will continue working properly until the end of days. Such approach is not easily scalable. Deploying a second instance of the service means that we need to start the manual process all over. We need to bring up a new server or find out which one has low utilization of resources, create a new set of configurations and deploy it. The situation is even more complicated in case of, let’s say, a hardware failure since the reaction time is usually slow when things are managed manually. Visibility is another painful point. We know what the static configuration is. After all, we prepared it in advance. However, most of the services have a lot of information generated dynamically. That information is not easily visible. There is no single location we can consult when we are in need of that data.

Reaction time is inevitably slow, failure resilience questionable at best and monitoring difficult to manage due to a lot of manually handled moving parts.

While there was excuse to do this job manually in the past or when the number of services and/or servers is low, with emergence of service discovery tools, this excuse quickly evaporates.

### <font color="#8B0000">2、Zookeeper</font>

[ZooKeeper](http://zookeeper.apache.org/) is one of the oldest projects of this type. It originated out of the Hadoop world, where it was built to help in the maintenance of the various components in a Hadoop cluster. It is mature, reliable and used by many big companies (YouTube, eBay, Yahoo, and so on). The format of the data it stores is similar to the organization of the file system. If run on a server cluster, Zookeper will share the state of the configuration across all of nodes. Each cluster elects a leader and clients can connect to any of the servers to retrieve data.

The main advantages Zookeeper brings to the table is its maturity, robustness and feature richness. However, it comes with its own set of disadvantages as well, with Java and complexity being main culprits. While Java is great for many use cases, it is very heavy for this type of work. Zookeeper’s usage of Java together with a considerable number of dependencies makes it much more resource hungry that its competition. On top of those problems, Zookeeper is complex. Maintaining it requires considerably more knowledge than we should expect from an application of this type. This is the part where feature richness converts itself from an advantage to a liability. The more features an application has, the bigger the chances that we won’t need all of them. Thus, we end up paying the price in form of complexity for something we do not fully need.

Zookeeper paved the way that others followed with considerable improvements. “Big players” are using it because there were no better alternatives at the time. Today, Zookeeper it shows its age and we are better off with alternatives.

### <font color="#8B0000">3、etcd</font>

[etcd](https://github.com/coreos/etcd) is a key/value store accessible through HTTP. It is distributed and features hierarchical configuration system that can be used to build service discovery. It is very easy to deploy, setup and use, provides reliable data persistence, it’s secure and with a very good documentation.

etcd is a better option than Zookeeper due to its simplicity. However, it needs to be combined with few third-party tools before it can serve service discovery objectives.

![](/public/img/k8s/etcd1.png)

Now that we have a place to store the information related to our services, we need a tool that will send that information to etcd automatically. After all, why would we put data to etcd manually if that can be done automatically. Even if we would want to manually put the information to etcd, we often don’t know what that information is. Remember, services might be deployed to a server with least containers running and it might have a random port assigned. Ideally, that tool should monitor Docker on all nodes and update etcd whenever a new container is run or an existing one is stopped. One of the tools that can help us with this goal is Registrator.

#### Registrator

[Registrator](https://github.com/gliderlabs/registrator) automatically registers and deregisters services by inspecting containers as they are brought online or stopped. It currently supports `etcd`, `Consul` and `SkyDNS 2`.

**Registrator** combined with **etcd** is a powerful, yet simple combination that allows us to practice many advanced techniques. Whenever we bring up a container, all the data will be stored in etcd and propagated to all nodes in the cluster. What we’ll do with that information is up to us.

![](/public/img/k8s/etcd-registrator2.png)

There is one more piece of the puzzle missing. We need a way to create configuration files with data stored in **etcd** as well as run some commands when those files are created.

#### confd

[confd](https://github.com/kelseyhightower/confd) is a lightweight configuration management tool. Common usages are keeping configuration files up-to-date using data stored in **etcd**, **consul** and few other data registries. It can also be used to reload applications when configuration files change. In other words, we can use it as a way to reconfigure all the services with the information stored in etcd (or many other registries).

![](/public/img/k8s/etcd-registrator-confd2.png)

#### Final thoughts on etcd, Registrator and confd combination

When etcd, Registrator and confd are combined we get a simple yet powerful way to automate all our service discovery and configuration needs. This combination also demonstrates effectiveness of having the right combination of “small” tools. Those three do exactly what we need them to do. Less than this and we would not be able to accomplish the goals set in front of us. If, on the other hand, they were designed with bigger scope in mind, we would introduce unnecessary complexity and overhead on server resources.

Before we make the final verdict, let’s take a look at another combination of tools with similar goals. After all, we should never settle for some solution without investigating alternatives.

### <font color="#8B0000">4、Consul</font>

[Consul](https://www.consul.io/) is strongly consistent datastore that uses gossip to form dynamic clusters. It features hierarchical key/value store that can be used not only to store data but also to register watches that can be used for a variety of tasks from sending notifications about data changes to running health checks and custom commands depending on their output.

Unlike Zookeeper and etcd, Consul implements service discovery system embedded so there is no need to build your own or use a third-party one. This discovery includes, among other things, health checks of nodes and services running on top of them.

ZooKeeper and etcd provide only a primitive K/V store and require that application developers build their own system to provide service discovery. Consul, on the other hand, provides a built in framework for service discovery. Clients only need to register services and perform discovery using the DNS or HTTP interface. The other two tools require either a hand-made solution or the usage of third-party tools.

Consul offers out of the box native support for multiple datacenters and the gossip system that works not only among nodes in the same cluster but across datacenters as well.

![](/public/img/k8s/consul2.png)

Consul has another nice feature that distinguishes it from the others. Not only that it can be used to discover information about deployed services and nodes they reside on, but it also provides easy to extend health checks through HTTP requests, TTLs (time-to-live) and custom commands.

#### Registrator

[Registrator](https://github.com/gliderlabs/registrator) has two Consul protocols. The **consulkv** protocol produces similar results as those obtained with the etcd protocol.

Besides the IP and the port that is normally stored with **etcd** or **consulkv** protocols, Registrator’s **consul** protocol stored more information. We get the information about the node the service is running on as well as service ID and name. With few additional environment variables we can also store additional information in form of tags

![](/public/img/k8s/consul-registrator2.png)

#### consul-template

confd can be used with Consul in the same way as with etcd. However, Consul has its own templating service with features more in line with what Consul offers.

The [consul-template](https://github.com/hashicorp/consul-template) is a very convenient way to create files with values obtained from Consul. As an added bonus it can also run arbitrary commands after the files have been updated. Just as confd, consul-template also uses [Go Template](http://golang.org/pkg/text/template/) format.

![](/public/img/k8s/consul-registrator-consul-template2.png)

#### Consul health checks, Web UI and datacenters

Monitoring health of cluster nodes and services is as important as testing and deployment itself. While we should aim towards having stable environments that never fail, we should also acknowledge that unexpected failures happen and be prepared to act accordingly. We can, for example, monitor memory usage and if it reaches certain threshold move some services to a different node in the cluster. That would be an example of preventive actions performed before the “disaster” would happen. On the other hand, not all potential failures can be detected on time for us to act on time. A single service can fail. A whole node can stop working due to a hardware failure. In such cases we should be prepared to act as fast as possible by, for example, replacing a node with a new one and moving failed services. Consul has a simple, elegant and, yet powerful way to perform health checks and help us to define what actions should be performed when health thresholds are reached.

If you googled “etcd ui” or “etcd dashboard” you probably saw that there are a few solutions available and might be asking why we haven’t presented them. The reason is simple; etcd is a key/value store and not much more. Having an UI to present data is not of much use since we can easily obtain it through the etcdctl. That does not mean that etcd UI is of no use but that it does not make much difference due to its limited scope.

Consul is much more than a simple key/value store. As we’ve already seen, besides storing simple key/value pairs, it has a notion of a service together with data that belongs to it. It can also perform health checks thus becoming a good candidate for a dashboard that can be used to see the status of our nodes and services running on top of them. Finally, it understands the concept of multiple datacenters. All those features combined let us see the need for a dashboard in a different light.

With the Consul Web UI we can view all services and nodes, monitor health checks and their statuses, read and set key/value data as well as switch from one datacenter to another.

![](/public/img/k8s/consul-nodes.png)

#### Final thoughts on Consul, Registrator, Template, health checks and Web UI

Consul together with the tools we explored is in many cases a better solution than what etcd offers. It was designed with services architecture and discovery in mind. It is simple, yet powerful. It provides a complete solution without sacrificing simplicity and, in many cases, it is the best tool for service discovery and health checking needs.

### Conclusion

All of the tools are based on similar principles and architecture. They run on nodes, require quorum to operate and are strongly consistent. They all provide some form of key/value storage.

**Zookeeper** is the oldest of the three and the age shows in its complexity, utilization of resources and goals it’s trying to accomplish. It was designed in a different age than the rest of the tools we evaluated (even though it’s not much older).

**etcd** with **Registrator** and **confd** is a very simple yet very powerful combination that can solve most, if not all, of our service discovery needs. It also showcases the power we can obtain when we combine simple and very specific tools. Each of them performs a very specific task, communicates through well established API and is capable of working with relative autonomy. They themselves are **microservices** both in their architectural as well as functional approach.

What distinguishes Consul is the support for multiple datacenters and health checking without the usage of third-party tools. That does not mean that the usage of third-party tools is bad. Actually, throughout this blog we are trying to combine different tools by choosing those that are performing better than others without introducing unnecessary features overhead. The best results are obtained when we use right tools for the job. If the tool does more than the job we require, its efficiency drops. On the other hand, tool that doesn’t do what we need it to do is useless. Consul strikes the right balance. It does very few things and it does them well.

The way Consul propagates knowledge about the cluster using gossip makes it easier to set up than etcd especially in case of a big datacenter. The ability to store data as a service makes it more complete and useful than key/value storage featured in etcd (even though Consul has that option as well). While we could accomplish the same by inserting multiple keys in etcd, Consul’s service accomplishes a more compact result that often requires a single query to retrieve all the data related to the service. On top of that, Registrator has quite good implementation of the consul protocol making the two an excellent combination, especially when consul-template is added to this picture. Consul’s Web UI is like a cherry on top of a cake and provides a good way to visualize your services and their health.

I can’t say that Consul is a clear winner. Instead, it has a slight edge when compared with etcd. Service discovery as a concept as well as the tools we can use are so new that we can expect a lot of changes in this field. Have an open mind and try to take advises from this article with a grain of salt. Try different tools and make your own conclusions.

## Refs

* [kubernetes 简介：kube-dns 和服务发现](http://cizixs.com/2017/04/11/kubernetes-intro-kube-dns)
* [Service Discovery: Zookeeper vs etcd vs Consul](https://technologyconversations.com/2015/09/08/service-discovery-zookeeper-vs-etcd-vs-consul/)
* [service-discovery-using-etcd-consul-and-kubernetes](https://www.slideshare.net/SreenivasMakam/service-discovery-using-etcd-consul-and-kubernetes)
