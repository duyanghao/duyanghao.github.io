---
layout: post
title: harbor备份还原方案
date: 2019-12-2 19:10:31
category: 技术
tags: harbor
excerpt: 本文介绍Harbor的备份还原方案……
---

## 前言

harbor是目前最流行的开源企业级镜像仓库解决方案，生产环境需要对harbor进行备份，以便发生问题时进行还原，目前有两种方式对harbor进行备份和还原，如下：

* 非屏蔽底层存储：针对harbor底层存储进行定制化的备份和还原，例如ceph、华为存储、Google Cloud等
* 屏蔽底层存储的备份：应用层的备份和还原，也即备份是将harbor中的所有镜像和charts拉取下来打成包，还原是将镜像和charts上传到harbor中

这两种方案各有优缺点，如下：

* 非屏蔽底层存储
    * 优点：
        * 1、备份和还原快
        * 2、harbor全备份
        * 3、成功率相对较高
    * 缺点：
        * 1、备份要停服，还原也需要停服
        * 2、另外需要一个一个存储定制方案，没有统一的方案

* 屏蔽底层存储
    * 优点：
        * 1、屏蔽底层存储细节，不用针对特定存储定制方案
        * 2、备份和还原不需要停服
    * 缺点：
        * 1、备份和还原速度较慢，无法做到增量备份
        * 2、有可能拉取某个镜像失败，备份和还原成功率相对没有第一种高，可以通过重试增加成功率
        * 3、另外，这种方案为非全备份——harbor只是备份了镜像和chart，其它日志等信息没有备份

## 非屏蔽底层存储

这里针对ceph存储举例说明：

ceph有三种存储服务：
* rbd——块设备：多读单写，I/O带宽高，读写延迟低；稳定性好
* cephfs——文件系统：多读多写，I/O带宽较低，读写延迟较高，性能较差；稳定性较差
* rgw——对象网关服务：多读多写，稳定性和性能均介于RBD与CephFS之间

下面针对ceph三种服务分别定制harbor备份还原方案

#### 1、rbd

由于pv对应的是块设备，所以需要先与系统内核进行映射，然后进行mount，最后进行备份操作（还原同理）：

![](/public/img/harbor_bur/harbor-bur-rbd-backup.png)

```bash
mapPath=`rbd map k8s/$rbdImage`
mount $mapPath $mountDir
rsync -avp --delete $mountDir/ $archiveDir/
```

#### 2、cephfs

由于pv对应的是文件系统，所以需要先找出对应路径，然后进行mount，最后进行备份操作（还原同理）：

![](/public/img/harbor_bur/harbor-bur-cephfs-backup.png)

```bash
mount -t ceph {device-string}:{path-to-mounted} {$mountDir} -o {key-value-args} {other-args}
rsync -avp --delete $mountDir/ $archiveDir/
```

#### 3、rgw

这里pv对应的对象存储，对于ceph rgw来说，实际落地就是bucket，我们需要找出对应数据的bucket，然后利用 s3cmd 进行备份操作（还原同理），如下：

![](/public/img/harbor_bur/harbor-bur-rgw-concurrent.png)

参考[registry-rgw-BUR-tools](https://github.com/duyanghao/registry-rgw-BUR-tools)实现

## 屏蔽底层存储

原理很简单，就是拉取harbor所有的镜像和charts，然后进行备份（还原同理）：

![](/public/img/harbor_bur/harbor-app-bur.png)

参考[backup_harbor.sh](https://github.com/duyanghao/Working-Tools/tree/master/harbor_tool)实现

## 补充

可以看到上述两种方案其实都有优缺点，理论上通过pv迁移的方式会更优雅一些，但是要解决**如何屏蔽底层存储的细节实现pv备份和还原**，目前社区项目[velero](https://github.com/vmware-tanzu/velero)就是做这个事情的，这个也是未来云原生应用备份还原的趋势：它解决了上层应用的多样性问题，不用考虑上层各个应用（例如MySQL、MongoDB、RabbitMQ、MariaDB以及Harbor等）的备份还原方案之间的差异，只需要专注pv的备份和还原就可以进行统一。后续会针对这种方案详细展开分析……

## Refs

* [registry-rgw-BUR-tools](https://github.com/duyanghao/registry-rgw-BUR-tools)
* [Working-Tools](https://github.com/duyanghao/Working-Tools/tree/master/harbor_tool)
* [Backup and migrate Kubernetes applications and their persistent volumes](https://github.com/vmware-tanzu/velero)