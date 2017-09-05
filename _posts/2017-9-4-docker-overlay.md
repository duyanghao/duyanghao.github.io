---
layout: post
title: docker overlay
date: 2017-9-4 19:10:31
category: 技术
tags: Linux Network Docker
excerpt: Docker overlay简介……
---

##　Docker网络模型前言

Docker在1.9版本中引入了一整套的自定义网络命令和跨主机网络支持。这是libnetwork项目从Docker的主仓库抽离之后的一次重大变化。

libnetwork项目从lincontainer和Docker代码的分离早在Docker 1.7版本就已经完成了（从Docker 1.6版本的网络代码中抽离）。在此之后，容器的网络接口就成为了一个个可替换的插件模块。由于这次变化进行的十分平顺，作为Docker的使用者几乎不会感觉到其中的差异，然而这个改变为接下来的一系列扩展埋下了很好的伏笔

概括来说，libnetwork所做的最核心事情是定义了一组标准的容器网络模型（Container Network Model，简称CNM），只要符合这个模型的网络接口就能被用于容器之间通信，而通信的过程和细节可以完全由网络接口来实现

Docker的容器网络模型最初是由思科公司员工Erik提出的设想，比较有趣的是Erik本人并不是Docker和libnetwork代码的直接贡献者。最初Erik只是为了扩展Docker网络方面的能力，设计了一个Docker网桥的扩展原型，并将这个思路反馈给了Docker社区。然而他的大胆设想得到了Docker团队的认同，并在与Docker的其他合作伙伴广泛讨论之后，逐渐形成了libnetwork的雏形

在这个网络模型中定义了三个的术语：Sandbox、Endpoint和Network：

![](/public/img/k8s/dockernetwork.jpg)

如上图所示，它们分别是容器通信中『容器网络环境』、『容器虚拟网卡』和『主机虚拟网卡/网桥』的抽象

* Sandbox：对应一个容器中的网络环境，包括相应的网卡配置、路由表、DNS配置等。CNM很形象的将它表示为网络的『沙盒』，因为这样的网络环境是随着容器的创建而创建，又随着容器销毁而不复存在的；
* Endpoint：实际上就是一个容器中的虚拟网卡，在容器中会显示为eth0、eth1依次类推；
* Network：指的是一个能够相互通信的容器网络，加入了同一个网络的容器直接可以直接通过对方的名字相互连接。它的实体本质上是主机上的虚拟网卡或网桥

这种抽象为Docker的1.7版本带来了十分平滑的过渡，除了文档中的三种经典『网络模式』被换成了『网络插件』，用户几乎感觉不到使用起来的差异

直到1.9版本的到来，Docker终于将网络的控制能力完全开放给了终端用户，并因此改变了连接两个容器通信的操作方式（当然，Docker为兼容性做足了功夫，所以即便你不知道下面所有的这些差异点，仍然丝毫不会影响继续用过去的方式使用Docker）

## docker network命令

Docker 1.9中网络相关的变化集中体现在新的『docker network』命令上：

```bash
$ docker network –help
Usage: docker network [OPTIONS]COMMAND [OPTIONS]
Commands:
ls List all networks
rm Remove a network
create Create a network
connect Connect container to anetwork
disconnect Disconnect container from anetwork
inspect Display detailed networkinformation
```

简单介绍一下这些命令的作用：

### 1、docker network ls

这个命令用于列出所有当前主机上或Swarm集群上的网络：

```bash
$ docker network ls
NETWORK ID NAME DRIVER
6e6edc3eee42 bridge bridge
1caa9a605df7 none null
d34a6dec29d3 host host
```

在默认情况下会看到三个网络，它们是Docker Deamon进程创建的。它们实际上分别对应了Docker过去的三种『网络模式』：

* bridge：容器使用独立网络Namespace，并连接到docker0虚拟网卡（默认模式）
* none：容器没有任何网卡，适合不需要与外部通过网络通信的容器
* host：容器与主机共享网络Namespace，拥有与主机相同的网络设备

在引入libnetwork后，它们不再是固定的『网络模式』了，而只是三种不同『网络插件』的实体。说它们是实体，是因为现在用户可以利用Docker的网络命令创建更多与默认网络相似的网络，每一个都是特定类型网络插件的实体

### 2、docker network create / docker network rm

这两个命令用于新建或删除一个容器网络，创建时可以用『–driver』参数使用的网络插件，例如：

```bash
$ docker network create –driver=bridge br0
b6942f95d04ac2f0ba7c80016eabdbce3739e4dc4abd6d3824a47348c4ef9e54
```

现在这个主机上有了一个新的bridge类型的Docker网络：

```bash
$ docker network ls
NETWORK ID NAME DRIVER
b6942f95d04a br0 bridge
...
```

Docker容器可以在创建时通过『–net』参数指定所使用的网络，连接到同一个网络的容器可以直接相互通信

当一个容器网络不再需要时，可以将它删除：

```bash
$ docker network rm br0
```

### 3、docker network connect / docker network disconnect

这两个命令用于动态的将容器添加进一个已有网络，或将容器从网络中移除。为了比较清楚的说明这一点，我们来看一个例子：

参照前面的libnetwork容器网络模型示意图中的情形创建两个网络：

```bash
$ docker network create –driver=bridge frontend
$ docker network create –driver=bridge backend
```

然后运行三个容器，让第一个容器接入frontend网络，第二个容器同时接入两个网络，三个容器只接入backend网络。首先用『–net』参数可以很容易创建出第一和第三个容器：

```bash
$ docker run -td –name ins01 –net frontend index.alauda.cn/library/busybox
$ docker run -td –name ins03 –net backend index.alauda.cn/library/busybox
```

如何创建一个同时加入两个网络的容器呢？由于创建容器时的『–net』参数只能指定一个网络名称，因此需要在创建过后再用docker network connect命令添加另一个网络：

```bash
$ docker run -td –name ins02 –net frontend index.alauda.cn/library/busybox
$ docker network connect backend ins02
```

现在通过ping命令测试一下这几个容器之间的连通性：

```bash
docker exec -it ins01 ping ins02
```

可以连通

```bash
$ docker exec -it ins01 ping ins03
```

找不到名称为ins03的容器

```bash
$ docker exec -it ins02 ping ins01
```

可以连通

```bash
$ docker exec -it ins02 ping ins03
```

可以连通

```bash
$ docker exec -it ins03 ping ins01
```

找不到名称为ins01的容器

```bash
$ docker exec -it ins03 ping ins02
```

可以连通

这个结果也证实了在相同网络中的两个容器可以直接使用名称相互找到对方，而在不同网络中的容器直接是不能够直接通信的。此时还可以通过docker networkdisconnect动态的将指定容器从指定的网络中移除：

```bash
$ docker network disconnect backend ins02
$ docker exec -it ins02 ping ins03
```

找不到名称为ins03的容器

可见，将ins02容器实例从backend网络中移除后，它就不能直接连通ins03容器实例了

### 4、docker network inspect

最后这个命令可以用来显示指定容器网络的信息，以及所有连接到这个网络中的容器列表：

```bash
$ docker network inspect bridge
[{
“Name”:”bridge”,
“Id”: “6e6edc3eee42722df8f1811cfd76d7521141915b34303aa735a66a6dc2c853a3”,
“Scope”: “local”,
“Driver”:”bridge”,
“IPAM”: {
“Driver”:”default”,
“Config”: [{“Subnet”: “172.17.0.0/16”}]
},
“Containers”: {
“3d77201aa050af6ec8c138d31af6fc6ed05964c71950f274515ceca633a80773”:{
“EndpointID”:”0751ceac4cce72cc11edfc1ed411b9e910a8b52fd2764d60678c05eb534184a4″,
“MacAddress”:”02:42:ac:11:00:02″,
“IPv4Address”: “172.17.0.2/16”,
“IPv6Address”:””
}
},
…
```

值得指出的是，同一主机上的每个不同网络分别拥有不同的网络地址段，因此同时属于多个网络的容器会有多个虚拟网卡和多个IP地址

由此可以看出，libnetwork带来的最直观变化实际上是：docker0不再是唯一的容器网络了，用户可以创建任意多个与docker0相似的网络来隔离容器之间的通信。然而，要仔细来说，用户自定义的网络和默认网络还是有不一样的地方：

* 默认的三个网络是不能被删除的，而用户自定义的网络可以用『docker networkrm』命令删掉；
* 连接到默认的bridge网络连接的容器需要明确的在启动时使用『–link』参数相互指定，才能在容器里使用容器名称连接到对方。而连接到自定义网络的容器，不需要任何配置就可以直接使用容器名连接到任何一个属于同一网络中的容器。这样的设计即方便了容器之间进行通信，又能够有效限制通信范围，增加网络安全性；
* 在Docker 1.9文档中已经明确指出，不再推荐容器使用默认的bridge网卡，它的存在仅仅是为了兼容早期设计。而容器间的『–link』通信方式也已经被标记为『过时的』功能，并可能会在将来的某个版本中被彻底移除

## DOCKER的内置OVERLAY网络

内置跨主机的网络通信一直是Docker备受期待的功能，在1.9版本之前，社区中就已经有许多第三方的工具或方法尝试解决这个问题，例如Macvlan、Pipework、Flannel、Weave等。虽然这些方案在实现细节上存在很多差异，但其思路无非分为两种：二层VLAN网络和Overlay网络

简单来说，二层VLAN网络的解决跨主机通信的思路是把原先的网络架构改造为互通的大二层网络，通过特定网络设备直接路由，实现容器点到点的之间通信。这种方案在传输效率上比Overlay网络占优，然而它也存在一些固有的问题

* 这种方法需要二层网络设备支持，通用性和灵活性不如后者；
* 由于通常交换机可用的VLAN数量都在4000个左右，这会对容器集群规模造成限制，远远不能满足公有云或大型私有云的部署需求；
* 大型数据中心部署VLAN，会导致任何一个VLAN的广播数据会在整个数据中心内泛滥，大量消耗网络带宽，带来维护的困难;

相比之下，Overlay网络是指在不改变现有网络基础设施的前提下，通过某种约定通信协议，把二层报文封装在IP报文之上的新的数据格式。这样不但能够充分利用成熟的IP路由协议进程数据分发，而且在Overlay技术中采用扩展的隔离标识位数，能够突破VLAN的4000数量限制，支持高达16M的用户，并在必要时可将广播流量转化为组播流量，避免广播数据泛滥。因此，Overlay网络实际上是目前最主流的容器跨节点数据传输和路由方案

在Docker的1.9中版本中正式加入了官方支持的跨节点通信解决方案，而这种内置的跨节点通信技术正是使用了Overlay网络的方法

说到Overlay网络，许多人的第一反应便是：低效，这种认识其实是带有偏见的。Overlay网络的实现方式可以有许多种，其中IETF（国际互联网工程任务组）制定了三种Overlay的实现标准，分别是：虚拟可扩展LAN（VXLAN）、采用通用路由封装的网络虚拟化（NVGRE）和无状态传输协议（SST），其中以VXLAN的支持厂商最为雄厚，可以说是Overlay网络的事实标准

而在这三种标准以外还有许多不成标准的Overlay通信协议，例如Weave、Flannel、Calico等工具都包含了一套自定义的Overlay网络协议（Flannel也支持VXLAN模式），这些自定义的网络协议的通信效率远远低于IETF的标准协议[5]，但由于他们使用起来十分方便，一直被广泛的采用而造成了大家普遍认为Overlay网络效率低下的印象。然而，根据网上的一些测试数据来看，采用VXLAN的网络的传输速率与二层VLAN网络是基本相当的

解除了这些顾虑后，一个好消息是，Docker内置的Overlay网络是采用IETF标准的VXLAN方式，并且是VXLAN中普遍认为最适合大规模的云计算虚拟化环境的SDN Controller模式

## Difference of VXLAN L3MISS between flannel and docker overlay implementation

关于`flannel`与`docker overlay`之间实现的区别可以参考这篇[文章](http://hustcat.github.io/vxlan-l3miss-problem/)

## 补充

### SDK(Software-Defined Networking (SDN) Definition)

>>What is SDN? The physical separation of the network control plane from the forwarding plane, and where a control plane controls several devices.

>>Software-Defined Networking (SDN) is an emerging architecture that is dynamic, manageable, cost-effective, and adaptable, making it ideal for the high-bandwidth, dynamic nature of today’s applications. This architecture decouples the network control and forwarding functions

>>enabling the network control to become directly programmable and the underlying infrastructure to be abstracted for applications and network services. The OpenFlow® protocol is a foundational element for building SDN solutions.

`OpenFlow`与`VXLAN`区别参考[如下](http://www.networkcomputing.com/networking/understanding-openflow-vxlan-and-ciscos-aci/551182896)

### Overlay 网络技术

#### 云计算虚拟化网络的挑战与革新

在云中，虚拟计算负载的高密度增长及灵活性迁移在一定程度上对网络产生了压力，然而当前虚拟机的规模与可迁移性受物理网络能力约束，云中的业务负载不能与物理网络脱离：

* 虚拟机迁移范围受到网络架构限制

由于虚拟机迁移的网络属性要求，其从一个物理机上迁移到另一个物理机上，要求虚拟机不间断业务，则需要其IP地址、MAC地址等参数维保持不变，如此则要求业务网络是一个二层网络，且要求网络本身具备多路径多链路的冗余和可靠性。传统的网络生成树(STPSpaning Tree Protocol)技术不仅部署繁琐，且协议复杂，网络规模不宜过大，限制了虚拟化的网络扩展性。基于各厂家私有的的IRF/vPC等设备级的（网络N:1）虚拟化技术，虽然可以简化拓扑简化、具备高可靠性的能力，但是对于网络有强制的拓扑形状限制，在网络的规模和灵活性上有所欠缺，只适合小规模网络构建，且一般适用于数据中心内部网络。而为了大规模网络扩展的TRILL/SPB/FabricPath/VPLS等技术，虽然解决了上述技术的不足，但对网络有特殊要求，即网络中的设备均要软硬件升级而支持此类新技术，带来部署成本的上升

* 虚拟机规模受网络规格限制

在大二层网络环境下，数据流均需要通过明确的网络寻址以保证准确到达目的地，因此网络设备的二层地址表项大小（(即MAC地址表)，成为决定了云计算环境下虚拟机的规模的上限，并且因为表项并非百分之百的有效性，使得可用的虚机数量进一步降低，特别是对于低成本的接入设备而言，因其表项一般规格较小，限制了整个云计算数据中心的虚拟机数量，但如果其地址表项设计为与核心或网关设备在同一档次，则会提升网络建设成本。虽然核心或网关设备的MAC与ARP规格会随着虚拟机增长也面临挑战，但对于此层次设备能力而言，大规格是不可避免的业务支撑要求。减小接入设备规格压力的做法可以是分离网关能力，如采用多个网关来分担虚机的终结和承载，但如此也会带来成本的上升

* 网络隔离/分离能力限制

当前的主流网络隔离技术为VLAN（或VPN），在大规模虚拟化环境部署会有两大限制：一是VLAN数量在标准定义中只有12个比特单位，即可用的数量为4000个左右，这样的数量级对于公有云或大型虚拟化云计算应用而言微不足道，其网络隔离与分离要求轻而易举会突破4000；二是VLAN技术当前为静态配置型技术（只有EVB/VEPA的802.1Qbg技术可以在接入层动态部署VLAN，但也主要是在交换机接主机的端口为常规部署，上行口依然为所有VLAN配置通过)，这样使得整个数据中心的网络几乎为所有VLAN被允许通过(核心设备更是如此)，导致任何一个VLAN的未知目的广播数据会在整网泛滥，无节制消耗网络交换能力与带宽

#### Overlay技术形态

Overlay在网络技术领域，指的是一种网络架构上叠加的虚拟化技术模式，其大体框架是对基础网络不进行大规模修改的条件下，实现应用在网络上的承载，并能与其它网络业务分离，并且以基于IP的基础网络技术为主：

![](/public/img/k8s/overlay_network.jpg)

#### Overlay如何解决当前的主要问题

针对前文提出的三大技术挑战，Overlay在很大程度上提供了全新的解决方式：

* 针对虚机迁移范围受到网络架构限制的解决方式

Overlay是一种封装在IP报文之上的新的数据格式，因此，这种数据可以通过路由的方式在网络中分发，而路由网络本身并无特殊网络结构限制，具备良性大规模扩展能力，并且对设备本身无特殊要求，以高性能路由转发为佳，且路由网络本身具备很强的的故障自愈能力、负载均衡能力。采用Overlay技术后，企业部署的现有网络便可用于支撑新的云计算业务，改造难度极低(除性能可能是考量因素外，技术上对于承载网络并无新的要求)

* 针对虚机规模受网络规格限制的解决方式

虚拟机数据封装在IP数据包中后，对网络只表现为封装后的的网络参数，即隧道端点的地址，因此，对于承载网络（特别是接入交换机），MAC地址规格需求极大降低，最低规格也就是几十个（每个端口一台物理服务器的隧道端点MAC）。当然，对于核心/网关处的设备表项(MAC/ARP)要求依然极高，当前的解决方案仍然是采用分散方式，通过多个核心/网关设备来分散表项的处理压力

* 针对网络隔离/分离能力限制的解决方式

针对VLAN数量4000以内的限制，在Overlay技术中引入了类似12比特VLAN ID的用户标识，支持千万级以上的用户标识，并且在Overlay中沿袭了云计算“租户”的概念，称之为Tenant ID(租户标识)，用24或64比特表示。针对VLAN技术下网络的TRUANK ALL(VLAN穿透所有设备)的问题，Overlay对网络的VLAN配置无要求，可以避免网络本身的无效流量带宽浪费，同时Overlay的二层连通基于虚机业务需求创建，在云的环境中全局可控

#### Overlay主要技术标准及比较

目前，IETF在Overlay技术领域有如下三大技术路线正在讨论，为简单起见，本文只讨论基于IPv4的Overlay相关内容：

* VXLAN：VXLAN是将以太网报文封装在UDP传输层上的一种隧道转发模式，目的UDP端口号为4798；为了使VXLAN充分利用承载网络路由的均衡性，VXLAN通过将原始以太网数据头(MAC、IP、四层端口号等)的HASH值作为UDP的号；采用24比特标识二层网络分段，称为VNI(VXLAN Network Identifier)，类似于VLAN ID作用；未知目的、广播、组播等网络流量均被封装为组播转发，物理网络要求支持任意源组播(ASM)
* NVGRE：NVGRE是将以太网报文封装在GRE内的一种隧道转发模式；采用24比特标识二层网络分段，称为VSI(Virtual Subnet Identifier)，类似于VLAN ID作用；为了使NVGRE利用承载网络路由的均衡性，NVGRE在GRE扩展字段flow ID，这就要求物理网络能够识别到GRE隧道的扩展信息，并以flow ID进行流量分担；未知目的、广播、组播等网络流量均被封装为组播转发
* STT：STT利用了TCP的数据封装形式，但改造了TCP的传输机制，数据传输不遵循TCP状态机，而是全新定义的无状态机制，将TCP各字段意义重新定义，无需三次握手建立TCP连接，因此称为无状态TCP；以太网数据封装在无状态TCP；采用64比特Context ID标识二层网络分段；为了使STT充分利用承载网络路由的均衡性，通过将原始以太网数据头(MAC、IP、四层端口号等)的HASH值作为无状态TCP的源端口号；未知目的、广播、组播等网络流量均被封装为组播转发

这三种二层Overlay技术，大体思路均是将以太网报文承载到某种隧道层面，差异性在于选择和构造隧道的不同，而底层均是IP转发。如表1所示为这三种技术关键特性的比较：VXLAN和STT对于现网设备对流量均衡要求较低，即负载链路负载分担适应性好，一般的网络设备都能对L2-L4的数据内容参数进行链路聚合或等价路由的流量均衡，而NVGRE则需要网络设备对GRE扩展头感知并对flow ID进行HASH，需要硬件升级；STT对于TCP有较大修改，隧道模式接近UDP性质，隧道构造技术属于革新性，且复杂度较高，而VXLAN利用了现有通用的UDP传输，成熟性极高。总体比较，VLXAN技术相对具有优势:

![](/public/img/k8s/overlay_compare.jpg)

### Open vSwitch

##### What is Open vSwitch?

>>Open vSwitch is a production quality, multilayer virtual switch licensed under the open source Apache 2.0 license.  It is designed to enable massive network automation through programmatic extension, while still supporting standard management interfaces and protocols (e.g. NetFlow, sFlow, IPFIX, RSPAN, CLI, LACP, 802.1ag).  In addition, it is designed to support distribution across multiple physical servers similar to VMware's vNetwork distributed vswitch or Cisco's Nexus 1000V. See full feature list [here](http://openvswitch.org/features/)

![](/public/img/k8s/OpenvSwitch.jpg)

### Docker overlayfs

是一种docker storage drivers。在内核3.18中，overlayfs终于正式进入主线。相比AUFS，overlayfs设计简单，代码也很少。而且可以实现pagecache共享。似乎是一个非常好的选择。于是，在这之后，docker社区开始转向将overlayfs作为第一选择。详细参考这篇[文章](http://hustcat.github.io/overlayfs-intro/)

## Refs

* [Docker container networking](https://docs.docker.com/engine/userguide/networking/)
* [Docker 1.9的新网络特性，以及Overlay详解](https://community.qingcloud.com/topic/328/docker-1-9%E7%9A%84%E6%96%B0%E7%BD%91%E7%BB%9C%E7%89%B9%E6%80%A7-%E4%BB%A5%E5%8F%8Aoverlay%E8%AF%A6%E8%A7%A3)
* [Docker native overlay network practice](http://hustcat.github.io/docker-overlay-network-practice/)
* [Docker storage driver history and overlayfs](http://hustcat.github.io/overlayfs-intro/)
* [基于多租户的云计算Overlay网络](http://www.h3c.com/cn/About_H3C/Company_Publication/IP_Lh/2013/04/Home/Catalog/201309/796466_30008_0.htm)
* [Overlay 网络技术，最想解决什么问题？](https://www.zhihu.com/question/24393680)





