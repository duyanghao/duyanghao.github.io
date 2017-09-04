---
layout: post
title: kube proxy
date: 2017-9-4 18:10:31
category: 技术
tags: Linux Network Kubernetes
excerpt: kubernetes网络架构……
---

## Kubernetes网络

在 kubernetes 集群中，网络是非常基础也非常重要的一部分。对于大规模的节点和容器来说，要保证网络的连通性、网络转发的高效，同时能做的 ip 和 port 自动化分配和管理，并让用户用直观简单的方式来访问需要的应用，这是需要复杂且细致设计的

kubernetes 在这方面下了很大的功夫，它通过 service、dns、ingress 等概念，解决了服务发现、负载均衡的问题，也大大简化了用户的使用和配置

这篇文章就讲解如何配置 kubernetes 的网络。一直以来，kubernetes 并没有专门的网络模块负责网络配置，它需要用户在主机上已经配置好网络。kubernetes 对网络的要求是：容器之间（包括同一台主机上的容器，和不同主机的容器）可以互相通信，容器和集群中所有的节点也能直接通信

至于具体的网络方案，用户可以自己选择，目前使用比较多的是 flannel，因为它比较简单，而且刚好满足 kubernetes 对网络的要求。以后 kubernetes 网络的发展方向是希望通过插件的方式来集成不同的网络方案， CNI 就是这一努力的结果，flannel 也能够通过 CNI 插件的形式使用

Kubernetes采用扁平化的网络模型，每个Pod都有一个全局唯一的IP(IP-per-pod),Pod之间可以跨主机通信，相比于Docker原生的NAT方式来说，这样使得容器在网络层面更像虚拟机或者物理机，复杂度整体降低，更加容易实现服务发现，迁移，负载均衡等功能。为了实现这个目标，Kubernetes中需要解决4个问题：

### 容器间通信(Netowrk Container)

Pod是容器的集合，Pod包含的容器都运行在同一个Host上，并且拥有同样的网络空间。现在创建一个Pod,包含2种Container：业务容器，网络容器。

举例如下：

```
#web-pod.yaml
apiVersion: v1
kind: Pod
metadata:
name: webpod
labels:
name: webpod
spec:
containers:
- name: webpod80
  image: jonlangemak/docker:web_container_80
  ports:
    - containerPort: 80
      hostPort: 80
- name: webpod8080
  image: jonlangemak/docker:web_container_8080
  ports:
    - containerPort: 8080
      hostPort: 8080
```

Pod运行成功后，在其所在的Node上查询容器：

```bash
$ docker ps
CONTAINER ID        IMAGE                                      PORTS                                         
63dc7e032ab6        jonlangemak/docker:web_container_8080                                                  
4ac1a5156a04        jonlangemak/docker:web_container_80                                                    
b77896498f8f        gcr.io/google_containers/pause:0.8.0       0.0.0.0:80->80/tcp, 0.0.0.0:8080->8080/tcp
```

可以看到运行了3个容器，其中2个这是Pod定义好的，第3个运行的容器镜像是`gcr.io/google_containers/pause`，它是`Netowrk Container`，它不做任何事情，只是用来接管Pod的网络

通过`docker inspect`查看着几个容器的信息，可以看出Pod中定义的容器的网络设置都集中配置在了`Netowrk Container`上，然后再加入`Netowrk Container`的网络中。这样的好处是避免服务容器之间产生依赖，用一个简单的容器来统一管理网络

### Pod间通信(Flannel)

Pod间通信是使用一个内部IP,这个IP即使Netowrk Container的IP

以web-pod为例:

```bash
$ kubectl describe pod webpod
Name:      webpod
Namespace:     default
IP:    10.1.14.51
...

$ docker inspect b77896498f8f  |grep IPAddress
    "IPAddress": "10.1.14.51",
```

对应用来说，这个IP是应用能看到，并且是可以对外宣称的（服务注册，服务发现）；而NAT方式，应用能看到的IP是不能对外宣称的（必须要使用主机IP+port方式），端口本身就是稀缺资源，并且ip+port的方式，无疑增加了复杂度，这就是IP-per-pod的优势所在

那么第一个问题就是，如何保证Pod的IP是全局唯一的。其实做法也很简单，因为Pod的IP是docker bridge分配的，不同Node之间docker bridge配置成不同的网段:

```
- node1: docker -d --bip=10.1.79.1/24 ...
- node2: docker -d --bip=10.1.14.1/24 ...
- node3: docker -d --bip=10.1.58.1/24 ...
```

同一个Node上的Pod原生能通信，但是不同Node之间的Pod如何通信的，这就需要对Docker进行增强，现有的方案有Flannel,OpenVSwitch,Weave等。本文Kubernetes环境是采用Flannel，下面进行介绍：

#### [Flannel介绍](http://dockone.io/article/618)

Flannel是CoreOS团队针对Kubernetes设计的一个网络规划服务，简单来说，它的功能是让集群中的不同节点主机创建的Docker容器都具有全集群唯一的虚拟IP地址

在Kubernetes的网络模型中，假设了每个物理节点应该具备一段“属于同一个内网IP段内”的“专用的子网IP”。例如：

```
节点A：10.0.1.0/24
节点B：10.0.2.0/24
节点C：10.0.3.0/24
```

但在默认的Docker配置中，每个节点上的Docker服务会分别负责所在节点容器的IP分配。这样导致的一个问题是，不同节点上容器可能获得相同的内外IP地址。并使这些容器之间能够之间通过IP地址相互找到，也就是相互ping通

Flannel的设计目的就是为集群中的所有节点重新规划IP地址的使用规则，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信

#### Flannel的工作原理

Flannel实质上是一种“覆盖网络(overlay network)”，也就是将TCP数据包装在另一种网络包里面进行路由转发和通信，目前已经支持UDP、VxLAN、AWS VPC和GCE路由等数据转发方式

默认的节点间数据通信方式是UDP转发，在Flannel的GitHub页面有如下的一张原理图：

![](/public/img/k8s/flannel.png)

这张图的信息量很全，下面简单的解读一下

数据从源容器中发出后，经由所在主机的docker0虚拟网卡转发到flannel0虚拟网卡，这是个P2P的虚拟网卡，flanneld服务监听在网卡的另外一端

Flannel通过Etcd服务维护了一张节点间的路由表，在稍后的配置部分我们会介绍其中的内容

源主机的flanneld服务将原本的数据内容UDP封装后根据自己的路由表投递给目的节点的flanneld服务，数据到达以后被解包，然后直接进入目的节点的flannel0虚拟网卡，然后被转发到目的主机的docker0虚拟网卡，最后就像本机容器通信一样的由docker0路由到达目标容器

这样整个数据包的传递就完成了，这里需要解释三个问题：

**第一个问题，UDP封装是怎么一回事？**

我们来看下面这个图，这是在其中一个通信节点上抓取到的ping命令通信数据包。可以看到在UDP的数据内容部分其实是另一个ICMP（也就是ping命令）的数据包：

![](/public/img/k8s/flannel_ping.png)

原始数据是在起始节点的Flannel服务上进行UDP封装的，投递到目的节点后就被另一端的Flannel服务还原成了原始的数据包，两边的Docker服务都感觉不到这个过程的存在

**第二个问题，为什么每个节点上的Docker会使用不同的IP地址段？**

这个事情看起来很诡异，但真相十分简单。其实只是单纯的因为Flannel通过Etcd分配了每个节点可用的IP地址段后，偷偷的修改了Docker的启动参数，见下图:

![](/public/img/k8s/flannel_docker.png)

这个是在运行了Flannel服务的节点上查看到的Docker服务进程运行参数

注意其中的“--bip=172.17.18.1/24”这个参数，它限制了所在节点容器获得的IP范围

这个IP范围是由Flannel自动分配的，由Flannel通过保存在Etcd服务中的记录确保它们不会重复

**第三个问题，为什么在发送节点上的数据会从docker0路由到flannel0虚拟网卡，在目的节点会从flannel0路由到docker0虚拟网卡？**

我们来看一眼安装了Flannel的节点上的路由表。下面是数据发送节点的路由表：

![](/public/img/k8s/flannel_send_route.png)

这个是数据接收节点的路由表：

![](/public/img/k8s/flannel_receive_route.png)

例如现在有一个数据包要从IP为172.17.18.2的容器发到IP为172.17.46.2的容器。根据数据发送节点的路由表，它只与172.17.0.0/16匹配这条记录匹配，因此数据从docker0出来以后就被投递到了flannel0。同理在目标节点，由于投递的地址是一个容器，因此目的地址一定会落在docker0对于的172.17.46.0/24这个记录上，自然的被投递到了docker0网卡
 
### Pod到Service通信(Kube-proxy)
 
Pod本身是变化的，比如当Pod发生迁移，那么Pod的IP是变化的, 那么Service的就是在Pod之间起到中转和代理的作用，Service会生成一个虚拟IP, 这个虚拟IP负载均衡到后端的Pod的IP，先给出`kubernetes`服务架构，如下：

![](/public/img/k8s/k8s服务架构图.png)

关于`kubernetes`服务可以参考如下[文章](http://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy)，本文主要介绍kubernetes网络架构，也即图中service的kube-proxy部分(Pod到Service通信)：

`kube-proxy`的作用主要是负责service的实现，`kube-proxy`运行在每个节点上，监听 API Server 中服务对象的变化，通过管理 iptables 来实现网络的转发。目前有`userspace`和`iptables`两种实现方式：

* `userspace` 是在用户空间监听一个端口，所有的 service 都转发到这个端口，然后 `kube-proxy` 在内部应用层对其进行转发。因为是在用户空间进行转发，所以效率也不高
* `iptables` 完全用 `iptables` 来实现 service，是目前默认的方式，也是推荐的方式，效率很高（只有内核中 `netfilter` 一些损耗）

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

mysql-service后端代理了两个pod，ip分别是192.168.125.129和192.168.125.131。先来看一下iptables:

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

如果转发的 pod 不能正常提供服务，它不会自动尝试另一个 pod，当然这个可以通过 readiness probes 来解决。每个 pod 都有一个健康检查的机制，当有 pod 健康状况有问题时，kube-proxy 会删除对应的转发规则

还需要注意的是，当前Kube-Proxy只是3层"(TCP/UDP over IP) 转发，当后端有多个Pod的时候，Kube-Proxy默认采用轮询方式进行选择，也可以设置成基于CLientIP的会话保持

### 外网到内网的通信

上面创建的服务只能在集群内部访问，这在生产环境中还不能直接使用。如果希望有一个能直接对外使用的服务，可以使用 NodePort 或者 LoadBalancer 类型的 Service。我们先说说 NodePort ，它的意思是在所有 worker 节点上暴露一个端口，这样外部可以直接通过访问 nodeIP:Port 来访问应用

`whoami-svc.yml`如下：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: whoami
  name: whoami
spec:
  ports:
    - port: 3000
      protocol: TCP
      # nodePort: 31000
  selector:
    app: whoami
  type: NodePort
```

因为我们的应用比较简单，只有一个端口。如果 pod 有多个端口，也可以在 spec.ports中继续添加，只有保证多个 port 之间不冲突就行

`whoami-rc.yml`如下：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: whoami
spec:
  replicas: 3
  selector:
    app: whoami
  template:
    metadata:
      name: whoami
      labels:
        app: whoami
        env: dev
    spec:
      containers:
      - name: whoami
        image: cizixs/whoami:v0.5
        ports:
        - containerPort: 3000
        env:
          - name: MESSAGE
            value: viola
```

创建 `rc` 和 `svc`：

```bash
[root@localhost ~]# kubectl create -f ./whoami-svc.yml
service "whoami" created
[root@localhost ~]# kubectl get rc,pods,svc
NAME        DESIRED   CURRENT   READY     AGE
rc/whoami   3         3         3         10s

NAME              READY     STATUS    RESTARTS   AGE
po/whoami-8zc3d   1/1       Running   0          10s
po/whoami-mc2fg   1/1       Running   0          10s
po/whoami-z6skj   1/1       Running   0          10s

NAME             CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
svc/kubernetes   10.10.10.1     <none>        443/TCP          14h
svc/whoami       10.10.10.163   <nodes>       3000:31647/TCP   7s
```

需要注意的是，因为我们没有指定 nodePort 的值，kubernetes 会自动给我们分配一个，比如这里的 31647（默认的取值范围是 30000-32767）。当然我们也可以删除配置中 `# nodePort: 31000` 的注释，这样会使用 31000 端口

nodePort 类型的服务会在所有的 worker 节点（运行了 kube-proxy）上统一暴露出端口对外提供服务，也就是说外部可以任意选择一个节点进行访问。比如我本地集群有三个节点：172.17.8.100、172.17.8.101 和 172.17.8.102：

```bash
[root@localhost ~]# curl http://172.17.8.100:31647
viola from whoami-mc2fg
[root@localhost ~]# curl http://172.17.8.101:31647
viola from whoami-8zc3d
[root@localhost ~]# curl http://172.17.8.102:31647
viola from whoami-z6skj
```

有了 nodePort，用户可以通过外部的 Load Balance 或者路由器把流量转发到任意的节点，对外提供服务的同时，也可以做到负载均衡的效果

nodePort 类型的服务并不影响原来虚拟 IP 的访问方式，内部节点依然可以通过 `vip:port` 的方式进行访问

## Refs

* [kubernetes入门之kube-proxy实现原理](https://xuxinkun.github.io/2016/07/22/kubernetes-proxy/)
* [kubernetes 简介：service 和 kube-proxy 原理](http://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy)
* [Kubernetes技术分析之网络](http://dockone.io/article/545)
* [一篇文章带你了解Flannel](http://dockone.io/article/618)
* [Network Plugins](https://kubernetes.io/docs/concepts/cluster-administration/network-plugins/)