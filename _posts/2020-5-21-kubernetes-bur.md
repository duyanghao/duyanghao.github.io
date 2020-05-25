---
layout: post
title: Kubernetes备份还原方案
date: 2020-5-21 19:10:31
category: 技术
tags: Kubernetes bur
excerpt: 本文介绍了三种Kubernetes集群的备份还原方案，并列举了各方案的优缺点。最后根据集群类型给出了方案选型建议。总的来说：方案一限制太多，ETCD的备份和还原存在太多不确定性，所以不建议使用；方案二采用Velero开源工具，通用性强，适用于大多数集群的备份还原；方案三灵活性高，适合高度定制的集群
---

# Contents

- [Overview](#Overview)
- [备份什么？](#备份什么？)
- [备份方案](#备份方案)
- [集群分类](#集群分类)
- [结论](#结论)
- [Refs](#Refs)

# Overview

对于在生产环境搭建的Kubernetes集群，我们可以利用[kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)搭建高可用集群，一定程度实现容灾：

![](/public/img/kubernetes_bur/kubernetes-ha.svg)

但是即便实现集群的高可用，我们依旧会需要备份还原功能，主要原因如下：

* 误删除：运维人员不小心删除了某个namespace，某个pv
* 服务器死机：因为物理原因服务器损坏，或者需要重装系统
* 集群迁移：需要将一个集群的数据迁移到另一个集群，用于测试或者其它目的

而对于Kubernetes的备份和还原，社区有一个16年创建的[issue](https://github.com/kubernetes/kubernetes/issues/24229)，从这个issue中我们可以看出Kubernetes官方并不打算提供Kubernetes备份还原方案以及工具，主要原因有两点：

* 官方认为Kubernetes只提供平台，管理和运行应用(类似于操作系统)。不负责应用层面的备份还原
* 基于Kubernetes集群的应用各不相同，无法(or 不太好)抽象统一备份方案

为了解决这个问题，社区中也有一些项目产生，例如：[reshifter](https://github.com/mhausenblas/reshifter)，[kube-backup](https://github.com/pieterlange/kube-backup)以及[velero](https://github.com/vmware-tanzu/velero)。
这其中最流行的备份还原工具是Velero，基于这个工具的研究可以参考[这篇文章](https://duyanghao.github.io/velero/)

虽然社区和Google上会有一些关于Kubernetes备份还原的工具和方案，但是都杂而不全，本文就针对Kubernetes备份还原这个专题进行一个全面而系统的方案介绍

# 备份什么？

对于Kubernetes搭建的集群，我们需要备份：

* 应用版本信息(Application Version)
  * 应用对应的工作负载，例如：deploymenet，statefulset以及daemonset等等
  * 应用需要使用的配置，例如：configmap，secret等等
* 应用状态信息(Application State)
  * 应用可以分为有状态应用和无状态应用
  * 有状态应用需要备份状态数据，例如：database
  
# 备份方案

针对上述备份数据，我们可以制定如下几种不同的备份还原方案（注意是逐项演进的）

## <font color="#dd0000">ETCD+应用状态</font><br />

方案要点如下：

* 应用版本备份：Kubernetes集群的所有应用版本信息都存放于ETCD中，我们可以直接备份ETCD来达到备份版本的目的
* 应用状态备份：可以从应用层或者文件系统层备份应用状态，例如MariaDB采用`mysqldump`，MongoDB采用`mongodump`等

备份流程大致如下：

* step1：备份ETCD
  ```bash
  # backup kubernetes pki
  $ cp -r /etc/kubernetes/pki backup/
  # backup kubeadm-config
  $ cp kubeadm-config.yaml backup/
  # backup etcd snapshot
  $ cat <<EOF > etcd-backup-job.yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: etcd-backup
  spec:
    template:
      spec:
        containers:
        - name: etcd-backup
          # Same image as in /etc/kubernetes/manifests/etcd.yaml
          image: k8s.gcr.io/etcd:3.2.24
          env:
          - name: ETCDCTL_API
            value: "3"
          command: ["/bin/sh"]
          args: ["-c", "etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /backup/etcd-snapshot.db"]
          volumeMounts:
          - mountPath: /etc/kubernetes/pki/etcd
            name: etcd-certs
            readOnly: true
          - mountPath: /backup
            name: backup
        restartPolicy: OnFailure
        hostNetwork: true
        volumes:
        - name: etcd-certs
          hostPath:
            path: /etc/kubernetes/pki/etcd
            type: DirectoryOrCreate
        - name: backup
          hostPath:
            path: /tmp/etcd-backup
            type: DirectoryOrCreate
  EOF
  $ kubectl apply -f etcd-backup-job.yaml 
  $ cp /tmp/etcd-backup/etcd-snapshot.db backup/ 
  ```

* step2：应用层备份(eg: MariaDB)
  ```bash
  # backup
  $ mysqldump -uxxx -pxxx db_name > backup-file.sql
  ```

还原流程大致如下：

* step1：还原ETCD
  ```bash
  # restore kubernetes pki
  $ cp -r backup/pki /etc/kubernetes/
  # restore kubeadm-config
  $ cp backup/kubeadm-config.yaml ./
  # initialize the first control plane node with backup kubernetes pki and kubeadm-config
  $ kubeadm init --config ./kubeadm-config.yaml
  # join other control plane nodes
  $ kubeadm join ...
  # restore etcd with snapshot(Refs: https://duyanghao.github.io/etcd-bur/)
  ```
  
* step2：清空数据
  ```bash
  # restore-phase-1(drop db)
  $ mysqladmin -uxxx -pxxx -f drop db_name
  Database "xxx" dropped
  
  # restore-phase-2(recreate db)
  $ mysqladmin -uxxx -pxxx create db_name  
  ```

* step3：还原数据
  ```bash
  # restore-phase-3(restore db)
  $ mysql -uxxx -pxxx db_name < backup-file.sql
  ```

这种方案的优缺点如下：

* Pros
  * 原理相对直接，简单
* Cons
  * 由于采用ETCD备份应用版本，保留了工作负载的母机node和pod ip等信息，只支持原地还原
  * ETCD还原粒度较大，存在很多的不确定性
  * 需要针对应用层分别制定备份方案，例如各种数据库使用不同的dump工具和命令。维护成本高
  * 全量备份，不支持增量备份

## <font color="#dd0000">应用版本+应用状态</font><br />

本方案主要使用社区最流行的备份还原工具[Velero](https://github.com/vmware-tanzu/velero)，要点如下：

* 应用版本备份：只备份Kubernetes指定namespace下的资源
* 应用状态备份：备份相应资源对应的状态数据

这里我们介绍最通用的[Velero Restic Integration](https://velero.io/docs/v1.3.2/restic/)方案，该方案集成了[Restic](https://github.com/restic/restic)工具在文件系统层面对应用状态数据进行备份，可以达到屏蔽底层存储细节的目的：

![](/public/img/kubernetes_bur/velero-bur.png)

备份流程如下：

* step1：安装velero
  ```bash
  $ velero install \
        --provider aws \
        --plugins velero/velero-plugin-for-aws:v1.0.0 \
        --bucket velero \
        --secret-file ./credentials-velero \
        --use-restic \
        --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000 \
        --snapshot-location-config region=minio
  
  [...truncate...]
  Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.
  
  $ kubectl get pods -nvelero
  NAME                     READY   STATUS      RESTARTS   AGE
  minio-d787f4bf7-ljqz9    1/1     Running     0          36s
  minio-setup-6ztpp        0/1     Completed   2          36s
  restic-xstrc             1/1     Running     0          11s
  velero-fd75fbd6c-2j2lb   1/1     Running     0          11s
  ```
  
* step2：安装[velero-volume-controller](https://github.com/duyanghao/velero-volume-controller)
  ```bash
  # Generated image
  $ make dockerfiles.build
  # Retag and push to your docker registry
  $ docker tag duyanghao/velero-volume-controller:v2.0 xxx/duyanghao/velero-volume-controller:v2.0
  $ docker push xxx/duyanghao/velero-volume-controller:v2.0
  # Update the deployment 'Image' field with the built image name
  $ sed -i 's|REPLACE_IMAGE|xxx/duyanghao/velero-volume-controller:v2.0|g' examples/deployment/velero-volume-controller.yaml
  # Create ClusterRole and ClusterRoleBinding
  $ kubectl apply -f examples/deployment/cluster-role.yaml
  $ kubectl apply -f examples/deployment/cluster-role-binding.yaml
  # Create ConfigMap
  $ kubectl apply -f examples/deployment/configmap.yaml
  # Create velero-volume-controller deployment
  $ kubectl apply -f examples/deployment/velero-volume-controller.yaml
  ```
  
* step3：创建velero restic backup
  ```bash
  $ velero backup create nginx-backup-with-pv --include-namespaces nginx-example
  Backup request "nginx-backup-with-pv" submitted successfully.
  
  $ velero backup get 
  NAME                   STATUS      CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
  nginx-backup-with-pv   Completed   2020-03-20 09:46:36 +0800 CST   29d       default            <none>
  ```  

还原流程如下：

* step1：清除资源
  ```bash
  $ kubectl delete namespace nginx-example
  namespace "nginx-example" deleted
  ```
  
* step2：创建velero restic restore
  ```bash
  $ velero restore create --from-backup nginx-backup-with-pv
  Restore request "nginx-backup-with-pv-20200320094935" submitted successfully.
  
  $ velero restore get
  NAME                                  BACKUP                 STATUS      WARNINGS   ERRORS   CREATED                         SELECTOR
  nginx-backup-with-pv-20200320094935   nginx-backup-with-pv   Completed   0          0        2020-03-20 09:49:35 +0800 CST   <none>
  ```
 
由于该方案的最佳实践工具是Velero，所以如果你想了解更多这个工具，可以参考[这篇文章](https://duyanghao.github.io/velero/)

该方案优缺点如下：

* Pros
  * 屏蔽底层存储，无需关心底层存储
  * 支持增量备份
  * 用户可以选择要备份的资源
  * 支持跨集群备份还原(数据迁移)
  * 使用简单方便，仅一条命令
* Cons
  * Velero Restic目前并不支持并发备份，第一次备份可能较慢
  * 需要维护一个对象存储，保存备份数据
  * Velero Restic目前不支持高可用
  * 不够灵活。虽然可以选择备份的版本信息，但是对应应用的状态信息却是全部进行备份的；另外应用版本和状态是绑定关系，无法分离
  
## <font color="#dd0000">VC+应用状态</font><br />

为了解决Velero备份不够灵活的问题。设计第三种备份方案，要点如下：

* 应用版本备份：只备份Kubernetes指定(最小集合)应用版本
* 应用状态备份：备份指定应用的最小状态集

可以看出相比第二种方案，本方案在备份粒度上更加细化了，应用版本和状态信息全部都缩减到最小集，也即只备份足够还原出集群的最小数据集(**数据量越多，不确定性越大**)

一般来说为了实现该方案，具体实施参考如下：

* 利用git管理Kubernetes集群中应用的版本信息
* 对各应用状态在上层定制备份还原方案，备份最小数据集

该方案优缺点如下：

* Pros
  * 备份最小数据集
  * 支持跨集群备份还原(数据迁移)
  * 灵活性高，可以在应用上层定制备份还原方案；另外，应用的版本和状态信息分离，可以实现升级还原
* Cons
  * 高度定制备份还原方案，所以维护成本高
  * 由于采用VC来备份应用版本信息，对版本管理有比较高的要求
  * 全量备份
  
# 集群分类

我们按照集群的作用可以将集群分类如下：

* 管理集群：负责管理客户集群，主要部署自研的管理组件
  * 特点：组件可能经常升级，内部可以管控部署组件；应用数据量小
* 客户集群：客户实际部署应用的集群
  * 特点：无法管控客户部署的应用；应用数据量大

针对这两种集群类型，对备份还原方案选型建议如下：

* 管理集群：上述方案均可，但由于方案一限制较多，故建议优先选择方案二和方案三
* 客户集群：由于不能管控该类集群上部署的应用类型。方案一和方案三都大概率需要定制应用上层备份还原方案，故建议方案二

# 结论

本文介绍了三种Kubernetes集群的备份还原方案，并列举了各方案的优缺点。最后根据集群类型给出了方案选型建议。总的来说：方案一限制太多，ETCD的备份和还原存在太多不确定性，所以不建议使用；方案二采用Velero开源工具，通用性强，适用于大多数集群的备份还原；方案三灵活性高，适合高度定制的集群

# Refs

* [The Ultimate Guide to Disaster Recovery for Your Kubernetes Clusters](https://medium.com/velotio-perspectives/the-ultimate-guide-to-disaster-recovery-for-your-kubernetes-clusters-94143fcc8c1e)
* [Backup Kubernetes – how and why](https://elastisys.com/2018/12/10/backup-kubernetes-how-and-why/)
* [Backup and Restore a Kubernetes Master with Kubeadm](https://labs.consol.de/kubernetes/2018/05/25/kubeadm-backup.html)
* [Backup/migrate cluster?](https://github.com/kubernetes/kubernetes/issues/24229)
* [Creating Highly Available clusters with kubeadm](https://v1-15.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
* [Use an HTTP Proxy to Access the Kubernetes API](https://kubernetes.io/docs/tasks/access-kubernetes-api/http-proxy-access-api/)
* [Kubernetes Component Status API](https://supereagle.github.io/2019/05/26/k8s-health-monitoring/)
