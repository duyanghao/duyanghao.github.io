---
layout: post
title: SuperEdge分布式健康检查深度剖析——edge-health-daemon
date: 2021-3-1 19:10:31
category: 技术
tags: Kubernetes SuperEdge
excerpt: 本文首先介绍了SuperEdge分布式健康检查对于云边断连情况的处理与Kubernetes原生做法区别，之后概述了edge-health架构原理并从源码角度深度剖析了边端组件edge-health-daemon的核心实现，有助于读者对SuperEdge分布式健康检查的进一步理解和使用
---

## 前言

边缘计算场景下，边缘节点与云端的网络环境十分复杂，连接并不可靠，在原生Kubernetes集群中，会造成apiserver和节点连接的中断，节点状态的异常，最终导致pod的驱逐和endpoint的缺失，造成服务的中断和波动，具体来说原生Kubernetes处理如下：

* 失联的节点被置为ConditionUnknown状态，并被添加NoSchedule和NoExecute的taints
* 失联的节点上的pod被驱逐，并在其他节点上进行重建
* 失联的节点上的pod从Service的Endpoint列表中移除

因此，边缘计算场景仅仅依赖边端和apiserver的连接情况是不足以判断节点是否异常的，会因为网络的不可靠造成误判，影响正常服务。而相较于云端和边缘端的连接，显然边端节点之间的连接更为稳定，具有一定的参考价值，因此superedge提出了边缘分布式健康检查机制。该机制中节点状态判定除了要考虑apiserver的因素外，还引入了节点的评估因素，进而对节点进行更为全面的状态判断。通过这个功能，能够避免由于云边网络不可靠造成的大量的pod迁移和重建，保证服务的稳定

具体来说，主要通过如下三个层面增强节点状态判断的准确性：

* 每个节点定期探测其他节点健康状态
* 集群内所有节点定期投票决定各节点的状态
* 云端和边端节点共同决定节点状态

而分布式健康检查最终的判断处理如下：

![img](https://github.com/duyanghao/kubernetes-reading-notes/raw/master/superedge/edge-health/images/edge-health-effect.png)

## edge-health-daemon源码分析

下面将基于我对edge-health的重构PR [Refactor edge-health and admission webhook for a better maintainability and extendibility](https://github.com/superedge/superedge/pull/46) 进行分析，在深入源码之前先介绍一下分布式健康检查的实现原理，其架构图如下所示：

![img](https://github.com/duyanghao/kubernetes-reading-notes/raw/master/superedge/edge-health/images/edge-health-arch.png)

Kubernetes每个node在kube-node-lease namespace下会对应一个Lease object，kubelet每隔node-status-update-frequency时间(默认10s)会更新对应node的Lease object

node-controller会每隔node-monitor-period时间(默认5s)检查Lease object是否更新，如果超过node-monitor-grace-period时间(默认40s)没有发生过更新，则认为这个node不健康，会更新NodeStatus(ConditionUnknown)

而当节点心跳超时(ConditionUnknown)之后，node controller会给该node添加如下taints：

```yaml
spec:
  ...
  taints:
  - effect: NoSchedule
    key: node.kubernetes.io/unreachable
    timeAdded: "2020-07-02T03:50:47Z"
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    timeAdded: "2020-07-02T03:50:53Z"
```

同时，endpoint controller会从endpoint backend中踢掉该母机上的所有pod

对于打上NoSchedule taint的母机，Scheduler不会调度新的负载在该node上了；而对于打上NoExecute(node.kubernetes.io/unreachable) taint的母机，node controller会在节点心跳超时之后一段时间(默认5mins)驱逐该节点上的pod

分布式健康检查边端的edge-health-daemon组件会对同区域边缘节点执行分布式健康检查，并向apiserver发送健康状态投票结果(给node打annotation)

此外，为了实现在云边断连且分布式健康检查状态正常的情况下：

* 失联的节点上的pod不会从Service的Endpoint列表中移除
* 失联的节点上的pod不会被驱逐

还需要在云端运行edge-health-admission([Kubernetes mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook))，不断根据node edge-health annotation调整kube-controller-manager设置的node taint(去掉NoExecute taint)以及endpoints(将失联节点上的pods从endpoint subsets notReadyAddresses移到addresses中)，从而实现云端和边端共同决定节点状态

本章将主要介绍edge-health-daemon原理，如下为edge-health-daemon的相关数据结构：

```go
type EdgeHealthMetadata struct {
	*NodeMetadata
	*CheckMetadata
}

type NodeMetadata struct {
	NodeList []v1.Node
	sync.RWMutex
}

type CheckMetadata struct {
	CheckInfo            map[string]map[string]CheckDetail // Checker ip:{Checked ip:Check detail}
	CheckPluginScoreInfo map[string]map[string]float64     // Checked ip:{Plugin name:Check score}
	sync.RWMutex
}

type CheckDetail struct {
	Normal bool
	Time   time.Time
}

type CommunInfo struct {
	SourceIP    string                 // ClientIP，Checker ip
	CheckDetail map[string]CheckDetail // Checked ip:Check detail
	Hmac        string
}
```

含义如下：

* NodeMetadata：为了实现分区域分布式健康检查机制而维护的边缘节点cache，其中包含该区域内的所有边缘节点列表NodeList
* CheckMetadata：存放健康检查的结果，具体来说包括两个数据结构：
  * CheckPluginScoreInfo：为`Checked ip:{Plugin name:Check score}`组织形式。第一级key表示：被检查的ip；第二级key表示：检查插件的名称；value表示：检查分数
  * CheckInfo：为`Checker ip:{Checked ip:Check detail}`组织形式。第一级key表示：执行检查的ip；第二级key表示：被检查的ip；value表示检查结果CheckDetail
* CheckDetail：代表健康检查的结果
  * Normal：Normal为true表示检查结果正常；false表示异常
  * Time：表示得出该结果时的时间，用于结果有效性的判断(超过一段时间没有更新的结果将无效)
* CommunInfo：边缘节点向其它节点发送健康检查结果时使用的数据，其中包括：
  * SourceIP：表示执行检查的ip
  * CheckDetail：为`Checked ip:Check detail`组织形式，包含被检查的ip以及检查结果
  * Hmac：SourceIP以及CheckDetail进行hmac得到，用于边缘节点通信过程中判断传输数据的有效性(是否被篡改)

edge-health-daemon主体逻辑包括四部分功能：

* SyncNodeList：根据边缘节点所在的zone刷新node cache，同时更新CheckMetadata相关数据
* ExecuteCheck：对每个边缘节点执行若干种类的健康检查插件(ping，kubelet等)，并将各插件检查分数汇总，根据用户设置的基准线得出节点是否健康的结果
* Commun：将本节点对其它各节点健康检查的结果发送给其它节点
* Vote：对所有节点健康检查的结果分类，如果某个节点被大多数(>1/2)节点判定为正常，则对该节点添加`superedgehealth/node-health：true` annotation，表明该节点分布式健康检查结果为正常；否则，对该节点添加`superedgehealth/node-health：false` annotation，表明该节点分布式健康检查结果为异常

下面依次对上述功能进行源码分析：

1、SyncNodeList

SyncNodeList每隔HealthCheckPeriod秒(health-check-period选项)执行一次，会按照如下情况分类刷新node cache：

* 如果kube-system namespace下不存在名为edge-health-zone-config的configmap，则没有开启多地域探测，因此会获取所有边缘节点列表并刷新node cache
* 否则，如果edge-health-zone-config的configmap数据部分TaintZoneAdmission为false，则没有开启多地域探测，因此会获取所有边缘节点列表并刷新node cache
* 如果TaintZoneAdmission为true，且node有"superedgehealth/topology-zone"标签(标示区域)，则获取"superedgehealth/topology-zone" label value相同的节点列表并刷新node cache
* 如果node没有"superedgehealth/topology-zone" label，则只会将边缘节点本身添加到分布式健康检查节点列表中并刷新node cache(only itself)

```go
func (ehd *EdgeHealthDaemon) SyncNodeList() {
	// Only sync nodes when self-located found
	var host *v1.Node
	if host = ehd.metadata.GetNodeByName(ehd.cfg.Node.HostName); host == nil {
		klog.Errorf("Self-hostname %s not found", ehd.cfg.Node.HostName)
		return
	}

	// Filter cloud nodes and retain edge ones
	masterRequirement, err := labels.NewRequirement(common.MasterLabel, selection.DoesNotExist, []string{})
	if err != nil {
		klog.Errorf("New masterRequirement failed %+v", err)
		return
	}
	masterSelector := labels.NewSelector()
	masterSelector = masterSelector.Add(*masterRequirement)

	if mrc, err := ehd.cmLister.ConfigMaps(metav1.NamespaceSystem).Get(common.TaintZoneConfigMap); err != nil {
		if apierrors.IsNotFound(err) { // multi-region configmap not found
			if NodeList, err := ehd.nodeLister.List(masterSelector); err != nil {
				klog.Errorf("Multi-region configmap not found and get nodes err %+v", err)
				return
			} else {
				ehd.metadata.SetByNodeList(NodeList)
			}
		} else {
			klog.Errorf("Get multi-region configmap err %+v", err)
			return
		}
	} else { // multi-region configmap found
		mrcv := mrc.Data[common.TaintZoneConfigMapKey]
		klog.V(4).Infof("Multi-region value is %s", mrcv)
		if mrcv == "false" { // close multi-region check
			if NodeList, err := ehd.nodeLister.List(masterSelector); err != nil {
				klog.Errorf("Multi-region configmap exist but disabled and get nodes err %+v", err)
				return
			} else {
				ehd.metadata.SetByNodeList(NodeList)
			}
		} else { // open multi-region check
			if hostZone, existed := host.Labels[common.TopologyZone]; existed {
				klog.V(4).Infof("Host %s has HostZone %s", host.Name, hostZone)
				zoneRequirement, err := labels.NewRequirement(common.TopologyZone, selection.Equals, []string{hostZone})
				if err != nil {
					klog.Errorf("New masterZoneRequirement failed: %+v", err)
					return
				}
				masterZoneSelector := labels.NewSelector()
				masterZoneSelector = masterZoneSelector.Add(*masterRequirement, *zoneRequirement)
				if nodeList, err := ehd.nodeLister.List(masterZoneSelector); err != nil {
					klog.Errorf("TopologyZone label for hostname %s but get nodes err: %+v", host.Name, err)
					return
				} else {
					ehd.metadata.SetByNodeList(nodeList)
				}
			} else { // Only check itself if there is no TopologyZone label
				klog.V(4).Infof("Only check itself since there is no TopologyZone label for hostname %s", host.Name)
				ehd.metadata.SetByNodeList([]*v1.Node{host})
			}
		}
	}

	// Init check plugin score
	ipList := make(map[string]struct{})
	for _, node := range ehd.metadata.Copy() {
		for _, addr := range node.Status.Addresses {
			if addr.Type == v1.NodeInternalIP {
				ipList[addr.Address] = struct{}{}
				ehd.metadata.InitCheckPluginScore(addr.Address)
			}
		}
	}

	// Delete redundant check plugin score
	for _, checkedIp := range ehd.metadata.CopyCheckedIp() {
		if _, existed := ipList[checkedIp]; !existed {
			ehd.metadata.DeleteCheckPluginScore(checkedIp)
		}
	}

	// Delete redundant check info
	for checkerIp := range ehd.metadata.CopyAll() {
		if _, existed := ipList[checkerIp]; !existed {
			ehd.metadata.DeleteByIp(ehd.cfg.Node.LocalIp, checkerIp)
		}
	}

	klog.V(4).Infof("SyncNodeList check info %+v successfully", ehd.metadata)
}

...
func (cm *CheckMetadata) DeleteByIp(localIp, ip string) {
	cm.Lock()
	defer cm.Unlock()
	delete(cm.CheckInfo[localIp], ip)
	delete(cm.CheckInfo, ip)
}
```

在按照如上逻辑更新node cache之后，会初始化CheckMetadata.CheckPluginScoreInfo，将节点ip赋值给CheckPluginScoreInfo key(`Checked ip`：被检查的ip)

另外，会删除CheckMetadata.CheckPluginScoreInfo以及CheckMetadata.CheckInfo中多余的items(不属于该边缘节点检查范围)

2、ExecuteCheck

ExecuteCheck也是每隔HealthCheckPeriod秒(health-check-period选项)执行一次，会对每个边缘节点执行若干种类的健康检查插件(ping，kubelet等)，并将各插件检查分数汇总，根据用户设置的基准线HealthCheckScoreLine(health-check-scoreline选项)得出节点是否健康的结果

```go
func (ehd *EdgeHealthDaemon) ExecuteCheck() {
	util.ParallelizeUntil(context.TODO(), 16, len(ehd.checkPlugin.Plugins), func(index int) {
		ehd.checkPlugin.Plugins[index].CheckExecute(ehd.metadata.CheckMetadata)
	})
	klog.V(4).Infof("CheckPluginScoreInfo is %+v after health check", ehd.metadata.CheckPluginScoreInfo)

	for checkedIp, pluginScores := range ehd.metadata.CopyCheckPluginScore() {
		totalScore := 0.0
		for _, score := range pluginScores {
			totalScore += score
		}
		if totalScore >= ehd.cfg.Check.HealthCheckScoreLine {
			ehd.metadata.SetByCheckDetail(ehd.cfg.Node.LocalIp, checkedIp, metadata.CheckDetail{Normal: true})
		} else {
			ehd.metadata.SetByCheckDetail(ehd.cfg.Node.LocalIp, checkedIp, metadata.CheckDetail{Normal: false})
		}
	}
	klog.V(4).Infof("CheckInfo is %+v after health check", ehd.metadata.CheckInfo)
}
```

这里会调用ParallelizeUntil并发执行各检查插件，edge-health目前支持ping以及kubelet两种检查插件，在checkplugin目录(github.com/superedge/superedge/pkg/edge-health/checkplugin)，通过Register注册到PluginInfo单例(plugin列表)中，如下：

```go
// TODO: handle flag parse errors
func (pcp *PingCheckPlugin) Set(s string) error {
	var err error
	for _, para := range strings.Split(s, ",") {
		if len(para) == 0 {
			continue
		}
		arr := strings.Split(para, "=")
		trimKey := strings.TrimSpace(arr[0])
		switch trimKey {
		case "timeout":
			timeout, _ := strconv.Atoi(strings.TrimSpace(arr[1]))
			pcp.HealthCheckoutTimeOut = timeout
		case "retries":
			retries, _ := strconv.Atoi(strings.TrimSpace(arr[1]))
			pcp.HealthCheckRetries = retries
		case "weight":
			weight, _ := strconv.ParseFloat(strings.TrimSpace(arr[1]), 64)
			pcp.Weight = weight
		case "port":
			port, _ := strconv.Atoi(strings.TrimSpace(arr[1]))
			pcp.Port = port
		}
	}
	PluginInfo = NewPlugin()
	PluginInfo.Register(pcp)
	return err
}

func (p *Plugin) Register(plugin CheckPlugin) {
	p.Plugins = append(p.Plugins, plugin)
	klog.V(4).Info("Register check plugin: %+v", plugin)
}


...
var (
	PluginOnce sync.Once
	PluginInfo Plugin
)

type Plugin struct {
	Plugins []CheckPlugin
}

func NewPlugin() Plugin {
	PluginOnce.Do(func() {
		PluginInfo = Plugin{
			Plugins: []CheckPlugin{},
		}
	})
	return PluginInfo
}
```

每种插件具体执行健康检查的逻辑封装在CheckExecute中，这里以ping plugin为例：

```go
// github.com/superedge/superedge/pkg/edge-health/checkplugin/pingcheck.go
func (pcp *PingCheckPlugin) CheckExecute(checkMetadata *metadata.CheckMetadata) {
	copyCheckedIp := checkMetadata.CopyCheckedIp()
	util.ParallelizeUntil(context.TODO(), 16, len(copyCheckedIp), func(index int) {
		checkedIp := copyCheckedIp[index]
		var err error
		for i := 0; i < pcp.HealthCheckRetries; i++ {
			if _, err := net.DialTimeout("tcp", checkedIp+":"+strconv.Itoa(pcp.Port), time.Duration(pcp.HealthCheckoutTimeOut)*time.Second); err == nil {
				break
			}
		}
		if err == nil {
			klog.V(4).Infof("Edge ping health check plugin %s for ip %s succeed", pcp.Name(), checkedIp)
			checkMetadata.SetByPluginScore(checkedIp, pcp.Name(), pcp.GetWeight(), common.CheckScoreMax)
		} else {
			klog.Warning("Edge ping health check plugin %s for ip %s failed, possible reason %s", pcp.Name(), checkedIp, err.Error())
			checkMetadata.SetByPluginScore(checkedIp, pcp.Name(), pcp.GetWeight(), common.CheckScoreMin)
		}
	})
}

// CheckPluginScoreInfo relevant functions
func (cm *CheckMetadata) SetByPluginScore(checkedIp, pluginName string, weight float64, score int) {
	cm.Lock()
	defer cm.Unlock()

	if _, existed := cm.CheckPluginScoreInfo[checkedIp]; !existed {
		cm.CheckPluginScoreInfo[checkedIp] = make(map[string]float64)
	}
	cm.CheckPluginScoreInfo[checkedIp][pluginName] = float64(score) * weight
}
```

CheckExecute会对同区域每个节点执行ping探测(net.DialTimeout)，如果失败，则给该节点打CheckScoreMin分(0)；否则，打CheckScoreMax分(100)

每种检查插件会有一个Weight参数，表示了该检查插件分数的权重值，所有权重参数之和应该为1，对应基准分数线HealthCheckScoreLine范围0-100。因此这里在设置分数时，会乘以权重

回到ExecuteCheck函数，在调用各插件执行健康检查得出权重分数(CheckPluginScoreInfo)后，还需要将该分数与基准线HealthCheckScoreLine对比：如果高于(>=)分数线，则认为该节点本次检查正常；否则异常

```go
func (ehd *EdgeHealthDaemon) ExecuteCheck() {
	util.ParallelizeUntil(context.TODO(), 16, len(ehd.checkPlugin.Plugins), func(index int) {
		ehd.checkPlugin.Plugins[index].CheckExecute(ehd.metadata.CheckMetadata)
	})
	klog.V(4).Infof("CheckPluginScoreInfo is %+v after health check", ehd.metadata.CheckPluginScoreInfo)

	for checkedIp, pluginScores := range ehd.metadata.CopyCheckPluginScore() {
		totalScore := 0.0
		for _, score := range pluginScores {
			totalScore += score
		}
		if totalScore >= ehd.cfg.Check.HealthCheckScoreLine {
			ehd.metadata.SetByCheckDetail(ehd.cfg.Node.LocalIp, checkedIp, metadata.CheckDetail{Normal: true})
		} else {
			ehd.metadata.SetByCheckDetail(ehd.cfg.Node.LocalIp, checkedIp, metadata.CheckDetail{Normal: false})
		}
	}
	klog.V(4).Infof("CheckInfo is %+v after health check", ehd.metadata.CheckInfo)
}
```

3、Commun

在对同区域各边缘节点执行健康检查后，需要将检查的结果传递给其它各节点，这也就是commun模块负责的事情：

```go
func (ehd *EdgeHealthDaemon) Run(stopCh <-chan struct{}) {
	// Execute edge health prepare and check
	ehd.PrepareAndCheck(stopCh)

	// Execute vote
	vote := vote.NewVoteEdge(&ehd.cfg.Vote)
	go vote.Vote(ehd.metadata, ehd.cfg.Kubeclient, ehd.cfg.Node.LocalIp, stopCh)

	// Execute communication
	communEdge := commun.NewCommunEdge(&ehd.cfg.Commun)
	communEdge.Commun(ehd.metadata.CheckMetadata, ehd.cmLister, ehd.cfg.Node.LocalIp, stopCh)

	<-stopCh
}
```

既然是互相传递结果给其它节点，则必然会有接受和发送模块：

```go
func (c *CommunEdge) Commun(checkMetadata *metadata.CheckMetadata, cmLister corelisters.ConfigMapLister, localIp string, stopCh <-chan struct{}) {
	go c.communReceive(checkMetadata, cmLister, stopCh)
	wait.Until(func() {
		c.communSend(checkMetadata, cmLister, localIp)
	}, time.Duration(c.CommunPeriod)*time.Second, stopCh)
}
```

其中communSend负责向其它节点发送本节点对它们的检查结果；而communReceive负责接受其它边缘节点的检查结果。下面依次分析：

```go
func (c *CommunEdge) communSend(checkMetadata *metadata.CheckMetadata, cmLister corelisters.ConfigMapLister, localIp string) {
	copyLocalCheckDetail := checkMetadata.CopyLocal(localIp)
	var checkedIps []string
	for checkedIp := range copyLocalCheckDetail {
		checkedIps = append(checkedIps, checkedIp)
	}
	util.ParallelizeUntil(context.TODO(), 16, len(checkedIps), func(index int) {
		// Only send commun information to other edge nodes(excluding itself)
		dstIp := checkedIps[index]
		if dstIp == localIp {
			return
		}
		// Send commun information
		communInfo := metadata.CommunInfo{SourceIP: localIp, CheckDetail: copyLocalCheckDetail}
		if hmac, err := util.GenerateHmac(communInfo, cmLister); err != nil {
			log.Errorf("communSend: generateHmac err %+v", err)
			return
		} else {
			communInfo.Hmac = hmac
		}
		commonInfoBytes, err := json.Marshal(communInfo)
		if err != nil {
			log.Errorf("communSend: json.Marshal commun info err %+v", err)
			return
		}
		commonInfoReader := bytes.NewReader(commonInfoBytes)
		for i := 0; i < c.CommunRetries; i++ {
			req, err := http.NewRequest("PUT", "http://"+dstIp+":"+strconv.Itoa(c.CommunServerPort)+"/result", commonInfoReader)
			if err != nil {
				log.Errorf("communSend: NewRequest for remote edge node %s err %+v", dstIp, err)
				continue
			}
			if err = util.DoRequestAndDiscard(c.client, req); err != nil {
				log.Errorf("communSend: DoRequestAndDiscard for remote edge node %s err %+v", dstIp, err)
			} else {
				log.V(4).Infof("communSend: put commun info %+v to remote edge node %s successfully", communInfo, dstIp)
				break
			}
		}
	})
}
```

发送逻辑如下：

* 构建CommunInfo结构体，包括：
  * SourceIP：表示执行检查的ip
  * CheckDetail：为Checked ip:Check detail组织形式，包含被检查的ip以及检查结果
* 调用GenerateHmac构建Hmac：实际上是以kube-system下的hmac-config configmap hmackey字段为key，对SourceIP以及CheckDetail进行hmac得到，用于判断传输数据的有效性(是否被篡改)

```go
func GenerateHmac(communInfo metadata.CommunInfo, cmLister corelisters.ConfigMapLister) (string, error) {
	addrBytes, err := json.Marshal(communInfo.SourceIP)
	if err != nil {
		return "", err
	}
	detailBytes, _ := json.Marshal(communInfo.CheckDetail)
	if err != nil {
		return "", err
	}
	hmacBefore := string(addrBytes) + string(detailBytes)
	if hmacConf, err := cmLister.ConfigMaps(metav1.NamespaceSystem).Get(common.HmacConfig); err != nil {
		return "", err
	} else {
		return GetHmacCode(hmacBefore, hmacConf.Data[common.HmacKey])
	}
}

func GetHmacCode(s, key string) (string, error) {
	h := hmac.New(sha256.New, []byte(key))
	if _, err := io.WriteString(h, s); err != nil {
		return "", err
	}
	return fmt.Sprintf("%x", h.Sum(nil)), nil
}
```

* 发送上述构建的CommunInfo给其它边缘节点(DoRequestAndDiscard)

communReceive逻辑也很清晰：

```go
// TODO: support changeable server listen port
func (c *CommunEdge) communReceive(checkMetadata *metadata.CheckMetadata, cmLister corelisters.ConfigMapLister, stopCh <-chan struct{}) {
	svr := &http.Server{Addr: ":" + strconv.Itoa(c.CommunServerPort)}
	svr.ReadTimeout = time.Duration(c.CommunTimeout) * time.Second
	svr.WriteTimeout = time.Duration(c.CommunTimeout) * time.Second
	http.HandleFunc("/debug/flags/v", pkgutil.UpdateLogLevel)
	http.HandleFunc("/result", func(w http.ResponseWriter, r *http.Request) {
		var communInfo metadata.CommunInfo
		if r.Body == nil {
			http.Error(w, "Invalid commun information", http.StatusBadRequest)
			return
		}

		err := json.NewDecoder(r.Body).Decode(&communInfo)
		if err != nil {
			http.Error(w, fmt.Sprintf("Invalid commun information %+v", err), http.StatusBadRequest)
			return
		}
		log.V(4).Infof("Received common information from %s : %+v", communInfo.SourceIP, communInfo.CheckDetail)

		if _, err := io.WriteString(w, "Received!\n"); err != nil {
			log.Errorf("communReceive: send response err %+v", err)
			http.Error(w, fmt.Sprintf("Send response err %+v", err), http.StatusInternalServerError)
			return
		}
		if hmac, err := util.GenerateHmac(communInfo, cmLister); err != nil {
			log.Errorf("communReceive: server GenerateHmac err %+v", err)
			http.Error(w, fmt.Sprintf("GenerateHmac err %+v", err), http.StatusInternalServerError)
			return
		} else {
			if hmac != communInfo.Hmac {
				log.Errorf("communReceive: Hmac not equal, hmac is %s but received commun info hmac is %s", hmac, communInfo.Hmac)
				http.Error(w, "Hmac not match", http.StatusForbidden)
				return
			}
		}
		log.V(4).Infof("communReceive: Hmac match")

		checkMetadata.SetByCommunInfo(communInfo)
		log.V(4).Infof("After communicate, check info is %+v", checkMetadata.CheckInfo)
	})

	go func() {
		if err := svr.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("Server: exit with error %+v", err)
		}
	}()

	for {
		select {
		case <-stopCh:
			ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
			defer cancel()
			if err := svr.Shutdown(ctx); err != nil {
				log.Errorf("Server: program exit, server exit error %+v", err)
			}
			return
		default:
		}
	}
}
```

负责接受其它边缘节点的检查结果，并写入自身检查结果CheckInfo，流程如下：

* 通过`/result`路由接受请求，并将请求内容解析成CommunInfo
* 对CommunInfo执行GenerateHmac获取hmac值，并与CommunInfo.Hmac字段进行对比，检查接受数据的有效性
* 最后将CommunInfo检查结果写入CheckInfo，注意：CheckDetail.Time设置为写入时的时间
```go
// CheckInfo relevant functions
func (cm *CheckMetadata) SetByCommunInfo(c CommunInfo) {
	cm.Lock()
	defer cm.Unlock()

	if _, existed := cm.CheckInfo[c.SourceIP]; !existed {
		cm.CheckInfo[c.SourceIP] = make(map[string]CheckDetail)
	}
	for k, detail := range c.CheckDetail {
		// Update time to local timestamp since different machines have different ones
		detail.Time = time.Now()
		c.CheckDetail[k] = detail
	}
	cm.CheckInfo[c.SourceIP] = c.CheckDetail
}
```
* 最后在接受到stopCh信号时，通过svr.Shutdown平滑关闭服务

4、Vote

在接受到其它节点的健康检查结果后，vote模块会对结果进行统计得出最终判决，并向apiserver报告：

```go
func (v *VoteEdge) Vote(edgeHealthMetadata *metadata.EdgeHealthMetadata, kubeclient clientset.Interface,
	localIp string, stopCh <-chan struct{}) {
	go wait.Until(func() {
		v.vote(edgeHealthMetadata, kubeclient, localIp, stopCh)
	}, time.Duration(v.VotePeriod)*time.Second, stopCh)
}
```

首先根据检查结果统计出状态正常以及异常的节点列表：

```go
type votePair struct {
	pros int
	cons int
}

...
var (
    prosVoteIpList, consVoteIpList []string
    // Init votePair since cannot assign to struct field voteCountMap[checkedIp].pros in map
    vp votePair
)
voteCountMap := make(map[string]votePair) // {"127.0.0.1":{"pros":1,"cons":2}}
copyCheckInfo := edgeHealthMetadata.CopyAll()
// Note that voteThreshold should be calculated by checked instead of checker
// since checked represents the total valid edge health nodes while checker may contain partly ones.
voteThreshold := (edgeHealthMetadata.GetCheckedIpLen() + 1) / 2
for _, checkedDetails := range copyCheckInfo {
    for checkedIp, checkedDetail := range checkedDetails {
        if !time.Now().After(checkedDetail.Time.Add(time.Duration(v.VoteTimeout) * time.Second)) {
            if _, existed := voteCountMap[checkedIp]; !existed {
                voteCountMap[checkedIp] = votePair{0, 0}
            }
            vp = voteCountMap[checkedIp]
            if checkedDetail.Normal {
                vp.pros++
                if vp.pros >= voteThreshold {
                    prosVoteIpList = append(prosVoteIpList, checkedIp)
                }
            } else {
                vp.cons++
                if vp.cons >= voteThreshold {
                    consVoteIpList = append(consVoteIpList, checkedIp)
                }
            }
            voteCountMap[checkedIp] = vp
        }
    }
}
log.V(4).Infof("Vote: voteCountMap is %+v", voteCountMap)
...
```

其中状态判断的逻辑如下：

* 如果超过一半(>)的节点对该节点的检查结果为正常，则认为该节点状态正常(注意时间差在VoteTimeout内)
* 如果超过一半(>)的节点对该节点的检查结果为异常，则认为该节点状态异常(注意时间差在VoteTimeout内)
* 除开上述情况，认为节点状态判断无效，对这些节点不做任何处理(可能存在脑裂的情况)

对状态正常的节点做如下处理：

```go
...
// Handle prosVoteIpList
util.ParallelizeUntil(context.TODO(), 16, len(prosVoteIpList), func(index int) {
    if node := edgeHealthMetadata.GetNodeByAddr(prosVoteIpList[index]); node != nil {
        log.V(4).Infof("Vote: vote pros to edge node %s begin ...", node.Name)
        nodeCopy := node.DeepCopy()
        needUpdated := false
        if nodeCopy.Annotations == nil {
            nodeCopy.Annotations = map[string]string{
                common.NodeHealthAnnotation: common.NodeHealthAnnotationPros,
            }
            needUpdated = true
        } else {
            if healthy, existed := nodeCopy.Annotations[common.NodeHealthAnnotation]; existed {
                if healthy != common.NodeHealthAnnotationPros {
                    nodeCopy.Annotations[common.NodeHealthAnnotation] = common.NodeHealthAnnotationPros
                    needUpdated = true
                }
            } else {
                nodeCopy.Annotations[common.NodeHealthAnnotation] = common.NodeHealthAnnotationPros
                needUpdated = true
            }
        }
        if index, existed := admissionutil.TaintExistsPosition(nodeCopy.Spec.Taints, common.UnreachableNoExecuteTaint); existed {
            nodeCopy.Spec.Taints = append(nodeCopy.Spec.Taints[:index], nodeCopy.Spec.Taints[index+1:]...)
            needUpdated = true
        }
        if needUpdated {
            if _, err := kubeclient.CoreV1().Nodes().Update(context.TODO(), nodeCopy, metav1.UpdateOptions{}); err != nil {
                log.Errorf("Vote: update pros vote to edge node %s error %+v ", nodeCopy.Name, err)
            } else {
                log.V(2).Infof("Vote: update pros vote to edge node %s successfully", nodeCopy.Name)
            }
        }
    } else {
        log.Warningf("Vote: edge node addr %s not found", prosVoteIpList[index])
    }
})
...
```

* 添加或者更新"superedgehealth/node-health" annotation值为"true"，表明分布式健康检查判断该节点状态正常
* 如果node存在NoExecute(node.kubernetes.io/unreachable) taint，则将其去掉，并更新node

而对状态异常的节点会添加或者更新"superedgehealth/node-health" annotation值为"false"，表明分布式健康检查判断该节点状态异常：

```go
// Handle consVoteIpList
util.ParallelizeUntil(context.TODO(), 16, len(consVoteIpList), func(index int) {
    if node := edgeHealthMetadata.GetNodeByAddr(consVoteIpList[index]); node != nil {
        log.V(4).Infof("Vote: vote cons to edge node %s begin ...", node.Name)
        nodeCopy := node.DeepCopy()
        needUpdated := false
        if nodeCopy.Annotations == nil {
            nodeCopy.Annotations = map[string]string{
                common.NodeHealthAnnotation: common.NodeHealthAnnotationCons,
            }
            needUpdated = true
        } else {
            if healthy, existed := nodeCopy.Annotations[common.NodeHealthAnnotation]; existed {
                if healthy != common.NodeHealthAnnotationCons {
                    nodeCopy.Annotations[common.NodeHealthAnnotation] = common.NodeHealthAnnotationCons
                    needUpdated = true
                }
            } else {
                nodeCopy.Annotations[common.NodeHealthAnnotation] = common.NodeHealthAnnotationCons
                needUpdated = true
            }
        }
        if needUpdated {
            if _, err := kubeclient.CoreV1().Nodes().Update(context.TODO(), nodeCopy, metav1.UpdateOptions{}); err != nil {
                log.Errorf("Vote: update cons vote to edge node %s error %+v ", nodeCopy.Name, err)
            } else {
                log.V(2).Infof("Vote: update cons vote to edge node %s successfully", nodeCopy.Name)
            }
        }
    } else {
        log.Warningf("Vote: edge node addr %s not found", consVoteIpList[index])
    }
})
```

在边端edge-health-daemon向apiserver发送节点健康结果后，云端运行edge-health-admission([Kubernetes mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook))，会不断根据node edge-health annotation调整kube-controller-manager设置的node taint(去掉NoExecute taint)以及endpoints(将失联节点上的pods从endpoint subsets notReadyAddresses移到addresses中)，从而实现即便云边断连，但是分布式健康检查状态正常的情况下：

* 失联的节点上的pod不会从Service的Endpoint列表中移除
* 失联的节点上的pod不会被驱逐

## 总结

* 分布式健康检查对于云边断连情况的处理区别原生Kubernetes如下：
  * 原生Kubernetes：
    * 失联的节点被置为ConditionUnknown状态，并被添加NoSchedule和NoExecute的taints
    * 失联的节点上的pod被驱逐，并在其他节点上进行重建
    * 失联的节点上的pod从Service的Endpoint列表中移除
  * 分布式健康检查：
    ![img](https://github.com/duyanghao/kubernetes-reading-notes/raw/master/superedge/edge-health/images/edge-health-effect.png)
* 分布式健康检查主要通过如下三个层面增强节点状态判断的准确性：
  * 每个节点定期探测其他节点健康状态
  * 集群内所有节点定期投票决定各节点的状态
  * 云端和边端节点共同决定节点状态    
* 分布式健康检查功能由边端的edge-health-daemon以及云端的edge-health-admission组成，功能分别如下：
  * edge-health-daemon：对同区域边缘节点执行分布式健康检查，并向apiserver发送健康状态投票结果(给node打annotation)，主体逻辑包括四部分功能：
    * SyncNodeList：根据边缘节点所在的zone刷新node cache，同时更新CheckMetadata相关数据
    * ExecuteCheck：对每个边缘节点执行若干种类的健康检查插件(ping，kubelet等)，并将各插件检查分数汇总，根据用户设置的基准线得出节点是否健康的结果
    * Commun：将本节点对其它各节点健康检查的结果发送给其它节点
    * Vote：对所有节点健康检查的结果分类，如果某个节点被大多数(>1/2)节点判定为正常，则对该节点添加superedgehealth/node-health：true annotation，表明该节点分布式健康检查结果为正常；否则，对该节点添加superedgehealth/node-health：false annotation，表明该节点分布式健康检查结果为异常
  * edge-health-admission([Kubernetes mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook))：不断根据node edge-health annotation调整kube-controller-manager设置的node taint(去掉NoExecute taint)以及endpoints(将失联节点上的pods从endpoint subsets notReadyAddresses移到addresses中)，从而实现云端和边端共同决定节点状态

## Refs

* [duyanghao kubernetes-reading-notes](https://github.com/duyanghao/kubernetes-reading-notes/blob/master/superedge/edge-health/README.md)