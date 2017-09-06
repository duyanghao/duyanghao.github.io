---
layout: post
title: Kubernetes在微服务化游戏中的探索实践
date: 2017-9-5 20:10:31
category: 技术
tags: Kubernetes Docker Microservices
excerpt: Kubernetes在微服务化游戏中的探索实践……
---

## 首先来看下微服务化游戏容器化探索之路

随着docker技术在近几年的快速发展，国内外掀起了一股容器之风。而我们也在这时，开启了游戏容器化的探索之路。最开始在[docker](https://www.kubernetes.org.cn/tags/docker)容器的应用上，还是以VM的模式去部署，毕竟游戏是非常复杂的应用，没有统一的模式。除此之外，对于一项全新技术的应用，大家都很谨慎，一步一步地去实践。

而在近一两年，部分游戏的架构也逐渐往微服务化方向转变，以下是一款游戏的架构：

![](/public/img/k8s/game1_arch.jpg)

游戏的逻辑层按不同的服务划分为不同的模块，每个模块都是高内聚低耦合，之间的通信通过消息队列（或者API）来实现。模块的版本通过[CI/CD](https://www.kubernetes.org.cn/tags/cicd)，实现镜像标准交付，快速部署。在这些模块中，大部分是无状态服务，很容易实现弹性伸缩。

我们再来看另一款微服务化游戏的架构：

![](/public/img/k8s/game2_arch.jpg)

也是按功能模块划分不同的服务，前端通过HAProxy来代理用户请求，后端服务可以根据负载来实现扩缩容。在服务发现模块中，通过registrator来监视容器的启动和停止，根据容器暴露的端口和环境变量自动注册服务，后端存储使用了consul，结合consul-template来发现服务的变化时，可以更新业务配置，并重载。

对于这些微服务化的游戏，服务模块小且多。那么，怎样快速部署，怎样弹性伸缩，怎样实现服务发现等等，都是我们需要解决的问题。容器化这些服务是一个不错的方案，接下来就是容器调度、编排平台的建设了。在当前也有多种方案可选，[mesos](https://www.kubernetes.org.cn/tags/mesos)，swarm等，而我们沿用了kubernetes做来容器的整个调度管理平台，这也得利于之前VM模式下kubernetes的成功应用。不同的是我们选择了高版本的kubernetes，无论从功能的丰富上，性能的提升上，稳定性，可扩展性上来说，都有绝对的优势

## 以下会从几个维度来分析kubernetes在微服务化游戏上的实践

### 一、定制的网络与调度方案

#### 1、定制的网络方案

集群的网络方案，是最为复杂，也是最为基础的一项。结合业务各模块之间的访问关系，我们选定的方案如下：

![](/public/img/k8s/k8s_network_arch.jpg)

集群内各模块之间的通信：[overlay网络](https://www.kubernetes.org.cn/tags/%E7%BD%91%E7%BB%9C)

我们基于flannel来实现overlay网络，每个主机拥有一个完整的子网，在这个扁平化的网络里面，管理简单。当我们创建容器的时候，会为容器分配一个唯一的虚拟IP，容器与容器之间可以方便地通信。当然，在实现中，业务也并非单纯的用IP来访问，而是结合DNS服务，通过域名来访问，后面会讲到

以下是基于flannel实现的overlay网络的通信案例：

![](/public/img/k8s/flannel_overlay_example.jpg)

假设当sshd-2访问nginx-0：当packet{172.16.28.5:port => 172.16.78.9:port} 到达docker0时，根据node1上的路由规则，选对flannel.1作为出口，同时，根据iptables SNAT规则，将packet的源IP地址改为flannel.1的地址(172.16.28.0/12)。flannel.1是一个VXLAN设备，将packet进行隧道封包，然后发到node2。node2解包，然后根据node2上的路由规则，从接口docker0出发，再转给nginx-0。最终实现通信

##### <font color="#8B0000">公司内网到集群内模块的通信：sriov-cni</font>

sriov-cni是我们基于CNI定制的一套SRIOV网络方案，而CNI作为kubernetes network plugins，插件式接入，非常方便，目前已[开源](https://github.com/hustcat/sriov-cni)

以下是SRIOV网络拓扑图与实现细节：

![](/public/img/k8s/sriov.jpg)

* 1、母机上开启SRIOV功能，同时向公司申请子机IP资源，每个VF对应一个子机IP
* 2、Kubernetes在调度时，为每个pod分配一个VF与子机IP
* 3、在pod拿到VF与IP资源，进行绑定设置后，就可以像物理网卡一样使用

同时我们也做了一些优化：包括VF中断，CPU绑定，同时关闭物理机的irqbalance功能，容器内设置RPS，把容器内的中断分到各个CPU处理，来提升网络性能

此类容器除了SRIOV网络之外，还有一个overlay网络接口，也即是多重网络，可以把公司内网流量导入到overlay集群中，实现集群内外之间的通信。在实际应用中，我们会用此类容器来收集通往集群内的通信，例如我们用haproxy LB容器来提供服务

##### <font color="#8B0000">对接公网：采用公司的TGW方案</font>

TGW接入时，需要提供物理IP，所以对接TGW都会用到SRIOV网络的容器，例如上面提到的haproxy LB容器。这样公网通过TGW访问haproxy，再由haproxy转到集群内容器，从而打通访问的整个链路

##### <font color="#8B0000">集群内模块访问公司内网通信</font>

NAT方案

#### 2、定制的调度方案

在上述网络方案中，我们讲到SRIOV需要绑定物理IP，所以在容器调度中，除了kubernetes原生提供的CPU＼Memory＼GPU之外，我们还把网络（物理IP）也作为一个计算资源，同时结合kubernetes提供的extender scheduler接口，我们定制了符合我们需求的调度程序(cr-arbitrator)。其结构如下：

![](/public/img/k8s/cr-arbitrator.jpg)

cr-arbitrator做为extender scheduler，集成到kubernetes中，包括两部分内容：

* 预选算法：在完成kuernetes的predicates调度后，会进入到cr-arbitrator的预选调度算法，我们以网络资源为例，会根据创建的容器是否需要物理IP，从而计算符合条件的node（母机）
* 优选算法：在整个集群中，需要物理IP的容器与overlay网络的容器并未严格的划分，而是采用混合部署方式，所以在调度overlay网络的容器时，需要优化分配到没有开启sriov的node上，只有在资源紧张的情况下，才会分配到sriov的node上

除了cr-arbitrator实现的调度策略外，我们还实现了CPU核绑定。可以使容器在其生命周期内使用固定的CPU核，一方面是避免不同游戏业务CPU抢占问题；另一方面在稳定性、性能上（结合NUMA）得到保障及提升，同时在游戏业务资源核算方面会更加的清晰

### 二、域名服务与负载均衡

在网络一节，我们讲到kubernetes会为每个pod分配一个虚拟的IP，但这个IP是非固定的，例如pod发生故障迁移后，那么IP就会发生变化。所以在微服务化游戏架构下，业务模块之间的访问更多地采用域名方式进行访问。在kubernetes生态链中，提供了skydns作为DNS服务器，可以很好的解决域名访问问题

在kubernetes集群中，业务使用域名访问有两种方式：

* 一是通过创建service来关联一组pod ,这时service会拥有一个名字（域名），pod可以直接使用此名字进行访问
* 另一个是通过pod的hostname访问（例如redis.default.pod…）。原生功能不支持，主要是kube2sky组件在生成域名规则有缺陷。针对这个问题，我们进行了优化，把pod的hostname也记录到etcd中，实现skydns对pod的hostname进行域名解析

在负载均衡方面：通常情况下，游戏的一个模块可以通过deployment或者是replication controller来创建多个pod（即多组服务），同时这些容器又需要对外提供服务。如果给每个pod都配置一个公司内网IP，也是可以解决，但带来的问题就是物理IP资源浪费，同时无法做到负载均衡，以及弹性伸缩

因此，我们需要一个稳固、高效的Loadbalancer方案来代理这些服务，其中也评估了kubernetes的service方案，不够成熟，在业务上应用甚少。刚好kubernetes的第三方插件service-loadbalancer提供了这方面的功能，它主要是通过haproxy来提供代理服务，而且有其它在线游戏也用了haproxy，所以我们选择了service-loadbalancer

service-loadbalancer除了haproxy服务外，还有一个servicelb服务。servicelb通过kubernetes的master api来实时获取对应pod信息（IP和port)，然后设置haproxy的backends，再reload haproxy进程，从而保证haproxy提供正确的服务

### 三、监控与告警

[监控](https://www.kubernetes.org.cn/tags/%E7%9B%91%E6%8E%A7)、告警是整个游戏运营过程中最为核心的功能之一，在游戏运行过程中，对其性能进行收集、统计与分析，来发现游戏模块是否存在问题，负载是否过高，是否需要扩缩容之类等等。在监控这一块，我们在cadvisor基础上进行定制，其结构如下：

![](/public/img/k8s/docker-monitor.jpg)

每个母机部署cadvisor程序，用于收集母机上容器的性能数据，目前包括CPU使用情况，memory，网络流量，TCP连接数

在存储方面，目前直接写入到TenDis中，后续如果压力太大，还可以考虑在TenDis前加一层消息队列，例如kafka集群

Docker-monitor，是基于cadvisor收集的数据而实现的一套性能统计与告警程序。在性能统计方面，除了对每个容器的性能计算外，还可以对游戏的每个服务进行综合统计分析，一方面用于前端用户展示，另一方面可以以此来对服务进行智能扩缩容。告警方面，用户可以按业务需求，配置个性化的告警规则，docker-monitor会针对不同的告警规则进行告警

### 四、日志收集

Docker在容器[日志](https://www.kubernetes.org.cn/tags/%e6%97%a5%e5%bf%97)处理这一块，目前已很丰富，除了默认的json-file之外，还提供了gcplogs、awslogs、fluentd等log driver。而在我们的日志系统中，还是简单的使用json-file，一方面容器日志并非整个方案中的关键节点，不想因为日志上的问题而影响docker的正常服务；另一方面，把容器日志落地到母机上，接下来只需要把日志及时采集走即可，而采集这块方案可以根据情况灵活选择，可扩展性强。我们当前选择的方案是filebeat+kafka+logstash+elasticsearch，其结构如下：

![](/public/img/k8s/log_gather.jpg)

我们以DaemonSet方式部署filebeat到集群中，收集容器的日志，并上报到kafka，最后存储到Elasticsearch集群，整个过程还是比较简单。而这里有个关键点，在业务混合部署的集群中，通过filebeat收集日志时怎样去区分不同的业务？而这恰恰是做日志权限管理的前提条件，我们只希望用户只能查看自己业务的日志

以下是具体的处理方案与流程：

* 首先我们在docker日志中，除了记录业务程序的日志外，还会记录容器的name与namespace信息。
* 接着我们在filebeat的kafka输出配置中，把namespace作为topic进行上报，最终对应到Elasticsearch的index

在我们的平台中，一个namespace只属于一个业务，通过namespace，可以快速的搜索到业务对应的日志，通过容器的name，可以查看业务内每个模块的日志

### 五、基于image的发布扩容

在微服务化游戏中，模块与模块之间是高内聚低偶合，模块的版本内容一般都会通过持续集成来构建成一个个镜像（即image），然后以image来交付、部署。同时，游戏版本发布都有一个时间窗，整个发布流程都需要在这个时间窗里完成，否则就会影响用户体验。怎样做到版本的高效发布? 这里有两个关键点：一是基于kubernetes的发布有效性；一是image下发效率；

Kubernetes在容器image发布这一块的支持已比较稳定，对于无状态的服务，还可以考虑rolling-update方式进行，使游戏服务近乎无缝地平滑升级，即在不停止对外服务的前提下完成应用的更新

在提高image下发效率方面，我们基于Distribution打造了一个企业级镜像中心，主要涉及到以下几点：

* ceph集群提供稳定、强大的后端数据存储
* 性能优化：mirror方案与P2P方案，实现快速的下载镜像。同时对于时效性更高的用户需求，还可以实现image预部署方案
* 安全方面：不同类型用户不同的权限验证方案。例如公司内部用户会提供安全认证，其它用户提供用户名密码认证
* Notification Server实现pull\push日志记录，便于后续分析与审计

以上便是kubernetes在微服务化游戏中的一个解决方案。定制的网络与调度方案，为游戏容器的运行提供基础环境；域名服务与负载均衡，解决游戏高可用、弹性伸缩问题；通过性能数据、日志的收集、统计分析，及时发现程序问题与性能瓶颈，保证游戏容器稳定、可持续性运行；最后，基于image的发布扩容，使得游戏部署流程更加标准化以及高效

## 补充

### 服务发现

#### 1、什么是服务发现？

服务发现组件记录了（大规模）分布式系统中所有服务的信息，人们或者其它服务可以据此找到这些服务。 DNS 就是一个简单的例子。当然，复杂系统的服务发现组件要提供更多的功能，例如，服务元数据存储、健康监控、多种查询和实时更新等

不同的使用情境，服务发现的含义也不同。例如，网络设备发现、零配置网络（ rendezvous ）发现和 SOA 发现等。无论是哪一种使用情境，服务发现提供了一种协调机制，方便服务的发布和查找

#### 2、时至今日，哪一种服务发现解决方案是最可靠的？

目前，业界提供了很多种服务发现解决方案：

* 人们已经使用 DNS 很长时间了， DNS 可能是现有的最大的服务发现系统。小规模系统可以先使用 DNS 作为服务发现手段。一旦服务节点的启动和销毁变得更加动态， DNS 就有问题了，因为 DNS 记录传播的速度可能跟不上服务节点变化的速度
* ZooKeeper 大概是最成熟的配置存储方案，它的历史比较长，提供了包括配置管理、领导人选举和分布式锁在内的完整解决方案。因此， ZooKeeper 是非常有竞争力的通用的服务发现解决方案，当然，它也显得过于复杂
* etcd 和 doozerd 是新近出现的服务发现解决方案，它们与 ZooKeeper 具有相似的架构和功能，因此可与 ZooKeeper 互换使用
* Consul 是一种更新的服务发现解决方案。除了服务发现，它还提供了配置管理和一种键值存储。 Consul 提供了服务节点的健康检查功能，支持使用 DNS SRV 查找服务，这大大增强了它与其它系统的互操作性。 Consul 与 ZooKeeper 的主要区别是： Consul 提供了 DNS 和 HTTP 两种 API ，而 ZooKeeper 只支持专门客户端的访问

#### 3、实施服务发现面临的最大挑战是什么？

服务发现之复杂，远超一般人的想象。这种复杂性源自分布式系统的复杂性:

一开始，你可以用一个配置文件实现服务发现，这个文件中包含了所有服务的名字、 IP 地址和端口等信息。当系统变得更加动态后，你就要把服务发现从静态配置迁移到一个真正的解决方案。这个迁移过程，并不像一般人所想的那么容易。最大的一项挑战是无从知晓所选的服务发现系统的侵入性如何：一旦选定一个服务发现系统，就很难再改用其它的服务发现系统，因此，一开始就必须选择正确的解决方案

很多服务发现系统都实现了某种形式的分布式共识算法，保证即使有节点失效系统仍然能够正常运转。但是，这些算法是出名地难实现，关键是要识别出分布式系统中失效的节点，这很困难。如果不能正确地识别，就不会正确地实现分布式共识算法

## Refs

* [Kubernetes在微服务化游戏中的探索实践](https://www.kubernetes.org.cn/2165.html)
* [六个问题带你了解服务发现](http://dockone.io/article/509)
