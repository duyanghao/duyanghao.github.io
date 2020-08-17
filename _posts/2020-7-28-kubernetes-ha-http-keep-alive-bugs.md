---
layout: post
title: Kubernetes Controller高可用诡异的15mins超时
date: 2020-7-28 19:10:31
category: 技术
tags: Kubernetes high-availability
excerpt: 本文详细描述了生产环境中Kubernetes Controller高可用诡异的15mins超时问题的排查，定位以及解决方案……
---

## 前言

节点宕机是生产环境无法规避的情况，生产环境必须适配这种情况，保障即便宕机后，服务依旧可用。而Kubernetes内部实现的[Leader Election Mechanism](https://github.com/kubernetes/client-go/tree/master/examples/leader-election)很好的解决了controller高可用的问题。但是某一次生产环境的测试发现controller在节点宕机后并没有马上(在规定的分布式锁释放时间后)实现切换，本文对这个问题进行了详尽的描述，分析，并在最后给出了解决方案

## 问题

在生产环境中，controller架构图部署如下：

![](/public/img/kubernetes-ha-http-keep-alives-bugs/controller-arch.png)

可以看到controller利用反亲和特性在不同的母机上部署有两个副本，通过Kubernetes service访问AA(Aggregated APIServer)

集群采用iptables的代理模式，假设目前两个母机的Controller经过iptables DNAT访问的都是母机10.0.0.3的AA；同时Controller采用[Leader Election](https://github.com/kubernetes/client-go/tree/master/examples/leader-election)机制实现高可用，母机10.0.0.3上的Controller是Leader，而母机10.0.0.2上的Controller是候选者

若此时母机10.0.0.3突然宕机，理论上母机10.0.0.2上的Controller会在分布式锁被释放后获取锁从而变成Leader，接管Controller任务，这也是Leader Election的原理

但是在手动关机母机10.0.0.3后(模拟宕机情景)，发现母机10.0.0.2上的Controller一直在报timeout错误，整个流程观察到现象如下：

Controller和AA部署详情如下：

```bash
$ kubectl get pods -o wide
controller-75f5547689-hjcmd               1/1     Running   0          47h     192.168.2.51    10.0.0.3   <none>           <none>
controller-75f5547689-m2l6g               1/1     Running   0          47h     192.168.1.108   10.0.0.2   <none>           <none>
aa-5ccf944d9f-9vnhx          1/1     Running   0          47h     192.168.2.52    10.0.0.3   <none>           <none>
aa-5ccf944d9f-zfldh          1/1     Running   0          47h     192.168.1.109   10.0.0.2   <none>           <none>
$ kubectl get svc 
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
aa         ClusterIP   192.168.255.220   <none>        443/TCP            51d
```

关闭母机10.0.0.3之前母机10.0.0.2上的Controller socket连接如下：

```bash
$ docker ps|grep controller-75f5547689-m2l6g
a10f0dcddddb ...
$ docker top a10f0dcddddb
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
root                16025               16007               0                   Aug01               ?                   00:03:18            controller xxx
$ nsenter -n -t 16025
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    keepalive (4.39/0/0)
```

母机10.0.0.2上的Controller日志如下：

```bash
2020-08-03 12:06:32.407 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:06:32.407 info    failed to acquire lease /controller
2020-08-03 12:06:35.344 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:06:35.344 info    failed to acquire lease /controller
...
```

母机10.0.0.2上的Controller(Candidate)每隔3s尝试获取分布式lock，由于已经被母机10.0.0.3上的Controller独占，所以会显示获取失败(正常逻辑)

另外，在母机10.0.0.2上抓包观察路由情况：

```bash
$ tcpdump -iany -nnvvXSs 0|grep 192.168.1.108|grep 443
...
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6c09 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x4f72 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6c09 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6e60 (correct), seq 1740053674:1740053764, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 90
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x4f72 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x51c9 (correct), seq 1740053764:1740054232, ack 3283065874, win 66, options [nop,nop,TS val 172345058 ecr 248885717], length 468
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xd003), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xcdac), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xcdac (correct), seq 3283065874, ack 1740053764, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xce2f), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xcbd8), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xcbd8 (correct), seq 3283065874, ack 1740054232, win 349, options [nop,nop,TS val 248885720 ecr 172345058], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x0bcf), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x0978), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x0978 (correct), seq 3283065874:3283065919, ack 1740054232, win 349, options [nop,nop,TS val 248889253 ecr 172345058], length 45
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x52e7 (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6dee (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x52e7 (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x553e (correct), seq 1740054232:1740054324, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 92
192.168.2.52.443 > 192.168.1.108.45220: Flags [P.], cksum 0x6dee (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.255.220.443 > 192.168.1.108.45220: Flags [P.], cksum 0x7045 (correct), seq 1740054324:1740054792, ack 3283065919, win 66, options [nop,nop,TS val 172348594 ecr 248889253], length 468
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xb205), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0x5bff (incorrect -> 0xafae), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.2.52.443: Flags [.], cksum 0xafae (correct), seq 3283065919, ack 1740054324, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
192.168.1.108.45220 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xb031), seq 3283065919, ack 1740054792, win 349, options [nop,nop,TS val 248889257 ecr 172348594], length 0
```

此时，关闭母机10.0.0.3电源(模拟宕机)。socket连接如下：

```bash
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0   1263 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    on (21.57/9/0)
```

母机10.0.0.2上的Controller日志如下：

```bash
...
2020-08-03 12:17:34.137 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:17:34.137 info    failed to acquire lease /controller
2020-08-03 12:17:37.674 info    lock is held by controller-75f5547689-hjcmd_23b81f76-884a-498a-8549-63371441bf18 and has not yet expired
2020-08-03 12:17:37.674 info    failed to acquire lease /controller
2020-08-03 12:17:52.040 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:17:52.040 info    failed to acquire lease /controller
2020-08-03 12:18:04.937 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:18:04.937 info    failed to acquire lease /controller
...
```

另外，在母机10.0.0.3宕机大约40s后，母机10.0.0.2上的iptables规则如下：

```bash
$ iptables-save |grep aa
-A KUBE-SERVICES -d 192.168.255.220/32 -p tcp -m comment --comment "default/aa: cluster IP" -m tcp --dport 443 -j KUBE-SVC-JIBCLJO3UBZXHPHV
$ iptables-save |grep KUBE-SVC-JIBCLJO3UBZXHPHV
:KUBE-SVC-JIBCLJO3UBZXHPHV - [0:0]
-A KUBE-SERVICES -d 192.168.255.220/32 -p tcp -m comment --comment "default/aa: cluster IP" -m tcp --dport 443 -j KUBE-SVC-JIBCLJO3UBZXHPHV
-A KUBE-SVC-JIBCLJO3UBZXHPHV -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3B7I6NOW767V7PGC
-A KUBE-SVC-JIBCLJO3UBZXHPHV -j KUBE-SEP-ATGG5YPR5S4AT627
$ iptables-save |grep KUBE-SEP-3B7I6NOW767V7PGC
:KUBE-SEP-3B7I6NOW767V7PGC - [0:0]
-A KUBE-SEP-3B7I6NOW767V7PGC -s 192.168.0.119/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-3B7I6NOW767V7PGC -p tcp -m tcp -j DNAT --to-destination 192.168.0.119:443
-A KUBE-SVC-JIBCLJO3UBZXHPHV -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-3B7I6NOW767V7PGC
$ iptables-save |grep KUBE-SEP-ATGG5YPR5S4AT627
:KUBE-SEP-ATGG5YPR5S4AT627 - [0:0]
-A KUBE-SEP-ATGG5YPR5S4AT627 -s 192.168.1.109/32 -j KUBE-MARK-MASQ
-A KUBE-SEP-ATGG5YPR5S4AT627 -p tcp -m tcp -j DNAT --to-destination 192.168.1.109:443
-A KUBE-SVC-JIBCLJO3UBZXHPHV -j KUBE-SEP-ATGG5YPR5S4AT627

$ kubectl get pods -o wide
aa-5ccf944d9f-9vnhx          1/1     Terminating   0          2d      192.168.2.52    10.0.0.3   <none>           <none>
aa-5ccf944d9f-zfldh          1/1     Running       0          47h     192.168.1.109   10.0.0.2   <none>           <none>
aa-5ccf944d9f-zzwr4          1/1     Running       0          3m3s    192.168.0.119   10.0.0.1   <none>           <none>
```

可以看到40s后iptables规则中已经剔除192.168.2.52(母机10.0.0.3上的AA)，但是socket一直存在，且一直访问的是192.168.2.52.443(母机10.0.0.3上的AA)，如下：

```bash
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0   5874 192.168.1.108:45220      192.168.255.220:443     ESTABLISHED 0          6334873    16025/controller    on (37.31/15/0)

...
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x1889), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x1632), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x1632 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
...
```

同时，母机10.0.0.2上的Controller一直在报timeout，如下：

```bash
...
2020-08-03 12:31:29.170 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:31:29.170 info    failed to acquire lease /controller
2020-08-03 12:31:42.856 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:31:42.856 info    failed to acquire lease /controller
...
```

最后，在15 mins的超时报错后，日志显示正常，如下：

```bash
2020-08-03 12:32:47.936 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 12:32:47.936 info    failed to acquire lease /controller
2020-08-03 12:33:01.005 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded
2020-08-03 12:33:01.005 info    failed to acquire lease /controller
2020-08-03 12:33:13.185 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: read tcp 192.168.1.108:45220->192.168.255.220:443: read: connection timed out
2020-08-03 12:33:13.185 info    failed to acquire lease /controller
2020-08-03 12:33:17.003 info    lock is held by controller-75f5547689-br8n8_c920cdc5-4ec1-431a-8278-0d1038340241 and has not yet expired
2020-08-03 12:33:17.003 info    failed to acquire lease /controller
2020-08-03 12:33:19.396 info    lock is held by controller-75f5547689-br8n8_c920cdc5-4ec1-431a-8278-0d1038340241 and has not yet expired
2020-08-03 12:33:19.396 info    failed to acquire lease /controller
```

同时，192.168.1.108.45220 > 192.168.255.220.443 socket消失，如下：

```bash
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.108:44272      192.168.255.220:443     ESTABLISHED 0          17580619   16025/controller    keepalive (14.59/0/0)
```

而抓包显示此时没有192.168.1.108:45220 > 192.168.255.220.443连接的包；转换到192.168.1.108.44272 > 192.168.1.109.443，如下：

```bash
...
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x1889), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x1632), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x1632 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249584128 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.255.220.443: Flags [P.], cksum 0x59d5 (incorrect -> 0x4287), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x5c2c (incorrect -> 0x4030), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45
192.168.1.108.45220 > 192.168.2.52.443: Flags [P.], cksum 0x4030 (correct), seq 3283065919:3283065964, ack 1740054792, win 349, options [nop,nop,TS val 249704448 ecr 172348594], length 45

192.168.1.108.44272 > 192.168.255.220.443: Flags [S], cksum 0x59b0 (incorrect -> 0x42f9), seq 479406945, win 28200, options [mss 1410,sackOK,TS val 249828577 ecr 0,nop,wscale 9], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [S], cksum 0x5b40 (incorrect -> 0x4169), seq 479406945, win 28200, options [mss 1410,sackOK,TS val 249828577 ecr 0,nop,wscale 9], length 0
192.168.1.109.443 > 192.168.1.108.44272: Flags [S.], cksum 0x5b40 (incorrect -> 0x107c), seq 847109001, ack 479406946, win 27960, options [mss 1410,sackOK,TS val 249828577 ecr 249828577,nop,wscale 9], length 0
192.168.255.220.443 > 192.168.1.108.44272: Flags [S.], cksum 0x59b0 (incorrect -> 0x120c), seq 847109001, ack 479406946, win 27960, options [mss 1410,sackOK,TS val 249828577 ecr 249828577,nop,wscale 9], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xada8), seq 479406946, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [.], cksum 0x5b38 (incorrect -> 0xac18), seq 479406946, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [P.], cksum 0x5a92 (incorrect -> 0xdc04), seq 479406946:479407180, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 234
192.168.1.108.44272 > 192.168.1.109.443: Flags [P.], cksum 0x5c22 (incorrect -> 0xda74), seq 479406946:479407180, ack 847109002, win 56, options [nop,nop,TS val 249828577 ecr 249828577], length 234
192.168.1.109.443 > 192.168.1.108.44272: Flags [.], cksum 0x5b38 (incorrect -> 0xab2d), seq 847109002, ack 479407180, win 57, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.255.220.443 > 192.168.1.108.44272: Flags [.], cksum 0x59a8 (incorrect -> 0xacbd), seq 847109002, ack 479407180, win 57, options [nop,nop,TS val 249828577 ecr 249828577], length 0
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x631b (incorrect -> 0xb6d9), seq 847109002:847111021, ack 479407180, win 57, options [nop,nop,TS val 249828580 ecr 249828577], length 2019
192.168.255.220.443 > 192.168.1.108.44272: Flags [P.], cksum 0x618b (incorrect -> 0xb869), seq 847109002:847111021, ack 479407180, win 57, options [nop,nop,TS val 249828580 ecr 249828577], length 2019
192.168.1.108.44272 > 192.168.255.220.443: Flags [.], cksum 0x59a8 (incorrect -> 0xa4ce), seq 479407180, ack 847111021, win 63, options [nop,nop,TS val 249828580 ecr 249828580], length 0
192.168.1.108.44272 > 192.168.1.109.443: Flags [.], cksum 0x5b38 (incorrect -> 0xa33e), seq 479407180, ack 847111021, win 63, options [nop,nop,TS val 249828580 ecr 249828580], length 0
192.168.1.108.44272 > 192.168.255.220.443: Flags [P.], cksum 0x5e41 (incorrect -> 0x7349), seq 479407180:479408357, ack 847111021, win 63, options [nop,nop,TS val 249828582 ecr 249828580], length 1177
192.168.1.108.44272 > 192.168.1.109.443: Flags [P.], cksum 0x5fd1 (incorrect -> 0x71b9), seq 479407180:479408357, ack 847111021, win 63, options [nop,nop,TS val 249828582 ecr 249828580], length 1177
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x5b6b (incorrect -> 0x3ee4), seq 847111021:847111072, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 51
192.168.255.220.443 > 192.168.1.108.44272: Flags [P.], cksum 0x59db (incorrect -> 0x4074), seq 847111021:847111072, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 51
192.168.1.109.443 > 192.168.1.108.44272: Flags [P.], cksum 0x5b76 (incorrect -> 0x3896), seq 847111072:847111134, ack 479408357, win 63, options [nop,nop,TS val 249828583 ecr 249828582], length 62
...
```

## 分析 - 禁用HTTP/2

从上面观察到的现象，我们开始分析问题出来哪里。一般分析问题的流程是自上而下，这里也不例外，我们从应用层开始进行分析

这里Controller每隔3s尝试获取一次分布式锁，超时时间设置为10s，如下为精简后的关键代码段(k8s.io/client-go/rest/config.go)：

```go
...
// RESTClientFor returns a RESTClient that satisfies the requested attributes on a client Config
// object. Note that a RESTClient may require fields that are optional when initializing a Client.
// A RESTClient created by this method is generic - it expects to operate on an API that follows
// the Kubernetes conventions, but may not be the Kubernetes API.
func RESTClientFor(config *Config) (*RESTClient, error) {
	if config.GroupVersion == nil {
		return nil, fmt.Errorf("GroupVersion is required when initializing a RESTClient")
	}
	if config.NegotiatedSerializer == nil {
		return nil, fmt.Errorf("NegotiatedSerializer is required when initializing a RESTClient")
	}
	qps := config.QPS
	if config.QPS == 0.0 {
		qps = DefaultQPS
	}
	burst := config.Burst
	if config.Burst == 0 {
		burst = DefaultBurst
	}

	baseURL, versionedAPIPath, err := defaultServerUrlFor(config)
	if err != nil {
		return nil, err
	}

	transport, err := TransportFor(config)
	if err != nil {
		return nil, err
	}

	var httpClient *http.Client
	if transport != http.DefaultTransport {
		httpClient = &http.Client{Transport: transport}
		if config.Timeout > 0 {
			httpClient.Timeout = config.Timeout
		}
	}

	return NewRESTClient(baseURL, versionedAPIPath, config.ContentConfig, qps, burst, config.RateLimiter, httpClient)
}

// TransportFor returns an http.RoundTripper that will provide the authentication
// or transport level security defined by the provided Config. Will return the
// default http.DefaultTransport if no special case behavior is needed.
func TransportFor(config *Config) (http.RoundTripper, error) {
	cfg, err := config.TransportConfig()
	if err != nil {
		return nil, err
	}
	return transport.New(cfg)
}

// New returns an http.RoundTripper that will provide the authentication
// or transport level security defined by the provided Config.
func New(config *Config) (http.RoundTripper, error) {
	// Set transport level security
	if config.Transport != nil && (config.HasCA() || config.HasCertAuth() || config.HasCertCallback() || config.TLS.Insecure) {
		return nil, fmt.Errorf("using a custom transport with TLS certificate options or the insecure flag is not allowed")
	}

	var (
		rt  http.RoundTripper
		err error
	)

	if config.Transport != nil {
		rt = config.Transport
	} else {
		rt, err = tlsCache.get(config)
		if err != nil {
			return nil, err
		}
	}

	return HTTPWrappersForConfig(config, rt)
}

...
func (c *tlsTransportCache) get(config *Config) (http.RoundTripper, error) {
	key, err := tlsConfigKey(config)
	if err != nil {
		return nil, err
	}

	// Ensure we only create a single transport for the given TLS options
	c.mu.Lock()
	defer c.mu.Unlock()

	// See if we already have a custom transport for this config
	if t, ok := c.transports[key]; ok {
		return t, nil
	}

	// Get the TLS options for this client config
	tlsConfig, err := TLSConfigFor(config)
	if err != nil {
		return nil, err
	}
	// The options didn't require a custom TLS config
	if tlsConfig == nil && config.Dial == nil {
		return http.DefaultTransport, nil
	}

	dial := config.Dial
	if dial == nil {
		dial = (&net.Dialer{
			Timeout:   30 * time.Second,
			KeepAlive: 30 * time.Second,
		}).DialContext
	}
	// Cache a single transport for these options
	c.transports[key] = utilnet.SetTransportDefaults(&http.Transport{
		Proxy:               http.ProxyFromEnvironment,
		TLSHandshakeTimeout: 10 * time.Second,
		TLSClientConfig:     tlsConfig,
		MaxIdleConnsPerHost: idleConnsPerHost,
		DialContext:         dial,
	})
	return c.transports[key], nil
}

// SetTransportDefaults applies the defaults from http.DefaultTransport
// for the Proxy, Dial, and TLSHandshakeTimeout fields if unset
func SetTransportDefaults(t *http.Transport) *http.Transport {
	t = SetOldTransportDefaults(t)
	// Allow clients to disable http2 if needed.
	if s := os.Getenv("DISABLE_HTTP2"); len(s) > 0 {
		klog.Infof("HTTP2 has been explicitly disabled")
	} else {
		if err := http2.ConfigureTransport(t); err != nil {
			klog.Warningf("Transport failed http2 configuration: %v", err)
		}
	}
	return t
}
...
```

可以看出这里Controller默认通过SetTransportDefaults使用HTTP/2协议，且设置了如下http Timeout参数：

* http.Client.Timeout：10s(http.Client.Timeout涵盖了HTTP请求从连接建立，请求发送，接收回应，重定向，以及读取http.Response.Body的整个生命周期的超时时间)
* net.Dialer.Timeout：30s(底层连接建立的超时时间)
* net.Dialer.KeepAlive：30s(设置TCP的keepalive参数(syscall.TCP_KEEPINTVL以及syscall.TCP_KEEPIDLE))

在设置了上述http timeout参数后，理论上访问分布式锁10s超时后，http client会将底层的tcp socket给关闭，但是这里一直没有关闭(一直处于ESTABLISHED状态)，是什么原因呢？

这里[分析源码](https://duyanghao.github.io/golang-http-timeout/)，可以看出HTTP/2在请求超时后并不会关闭底层的socket，而是继续使用；而HTTP/1则会在请求超时后将使用的tcp socket关闭

那么这里的现象就可以解释如下：

* 由于Controller使用了HTTP/2协议，在10s的请求超时后并没有关闭tcp socket，而是继续使用
* 同时由于每隔3s重试一次请求(获取分布式lock)，导致TCP keepalive没办法触发(30s+30s*9=300s, 也即5mins)
  ```bash
  tcp_keepalive_time = 30s
  tcp_keepalive_intvl = 30s
  tcp_keepalive_probes = 9
  ```

**但是上面的两点并没有解释为什么15分钟后tcp socket会消失，timeout现象消失**

于是抱着先解决问题后查找原因的想法尝试先关闭HTTP/2协议，看看问题是否解决？

尝试设置Controller deployment环境变量，禁止HTTP/2，切换HTTP/1：

```yaml
spec:
  containers:
  - args:
    ...
    env:
    - name: DISABLE_HTTP2
      value: "true"
```

重新部署后，再次测试，现象如下：

```bash
2020-08-03 16:00:52.817 info    failed to acquire lease /controller
2020-08-03 16:00:55.041 info    lock is held by controller-7bb867f56f-x94m4_366f0ee0-fc9c-4bff-bb4f-48dc00e995bb and has not yet expired
2020-08-03 16:00:55.041 info    failed to acquire lease /controller
2020-08-03 16:01:08.583 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:08.583 info    failed to acquire lease /controller
2020-08-03 16:01:22.665 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:22.665 info    failed to acquire lease /controller
2020-08-03 16:01:36.472 error   error retrieving resource lock /controller: Get https://aa:443/api/v1/namespaces/default/configmaps/controller?timeout=10s: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
2020-08-03 16:01:36.472 info    failed to acquire lease /controller
2020-08-03 16:01:39.980 info    lock is held by controller-7bb867f56f-x94m4_366f0ee0-fc9c-4bff-bb4f-48dc00e995bb and has not yet expired
2020-08-03 16:01:39.980 info    failed to acquire lease /controller
```

抓包如下：

```bash
192.168.2.148.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc4a2 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.2.148.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc4a2 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.255.220.443 > 192.168.1.204.50254: Flags [P.], cksum 0xc759 (correct), seq 136132035:136132691, ack 3350711668, win 319, options [nop,nop,TS val 645418 ecr 262286621], length 656
192.168.1.204.50254 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0x4745), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.2.148.443: Flags [.], cksum 0x5cbf (incorrect -> 0x448e), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.2.148.443: Flags [.], cksum 0x448e (correct), seq 3350711668, ack 136132691, win 210, options [nop,nop,TS val 262286624 ecr 645418], length 0
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x790a), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7653), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7653 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290166 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x783f), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7588), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7588 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290369 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x7774), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x74bd), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x74bd (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290572 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x75dd), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x7326), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x7326 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262290979 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x72b0), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x6ff9), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x6ff9 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262291792 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x6c54), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x699d), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x699d (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262293420 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x5fa0), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5dee (incorrect -> 0x5ce9), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.2.148.443: Flags [P.], cksum 0x5ce9 (correct), seq 3350711668:3350711971, ack 136132691, win 210, options [nop,nop,TS val 262296672 ecr 645418], length 303
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5a27 (incorrect -> 0x1466), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5cde (incorrect -> 0x11af), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x11af (correct), seq 3350711971:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262300166 ecr 645418], length 31
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5b56 (incorrect -> 0xa413), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5e0d (incorrect -> 0xa15c), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0xa15c (correct), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262303184 ecr 645418], length 334
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xcf6c), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xccb5), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xccb5 (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262304249 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xcb81), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xc8ca), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xc8ca (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262305252 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xc3ad), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xc0f6), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xc0f6 (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262307256 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xb405), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xb14e), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.51328 > 192.168.2.148.443: Flags [S], cksum 0xb14e (correct), seq 1911738311, win 28200, options [mss 1410,sackOK,TS val 262311264 ecr 0,nop,wscale 9], length 0
192.168.1.204.50254 > 192.168.255.220.443: Flags [FP.], cksum 0x5b56 (incorrect -> 0x7123), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x5e0d (incorrect -> 0x6e6c), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.50254 > 192.168.2.148.443: Flags [FP.], cksum 0x6e6c (correct), seq 3350711668:3350712002, ack 136132691, win 210, options [nop,nop,TS val 262316224 ecr 645418], length 334
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x1da9), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x1af2), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x1af2 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262318056 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x19bf), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x1708), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x1708 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262319058 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x11e9), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0x0f32), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x0f32 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262321064 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0x0241), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0x5cc7 (incorrect -> 0xff89), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51482 > 192.168.2.148.443: Flags [S], cksum 0xff89 (correct), seq 276343932, win 28200, options [mss 1410,sackOK,TS val 262325072 ecr 0,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [S], cksum 0x5a10 (incorrect -> 0xf7fc), seq 3929326320, win 28200, options [mss 1410,sackOK,TS val 262331555 ecr 0,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [S], cksum 0x5bfe (incorrect -> 0xf60e), seq 3929326320, win 28200, options [mss 1410,sackOK,TS val 262331555 ecr 0,nop,wscale 9], length 0
192.168.1.203.443 > 192.168.1.204.51674: Flags [S.], cksum 0x5bfe (incorrect -> 0x33cf), seq 1019320855, ack 3929326321, win 27960, options [mss 1410,sackOK,TS val 262331555 ecr 262331555,nop,wscale 9], length 0
192.168.255.220.443 > 192.168.1.204.51674: Flags [S.], cksum 0x5a10 (incorrect -> 0x35bd), seq 1019320855, ack 3929326321, win 27960, options [mss 1410,sackOK,TS val 262331555 ecr 262331555,nop,wscale 9], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xd159), seq 3929326321, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xcf6b), seq 3929326321, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5adc (incorrect -> 0x4fb3), seq 3929326321:3929326533, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 212
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x5cca (incorrect -> 0x4dc5), seq 3929326321:3929326533, ack 1019320856, win 56, options [nop,nop,TS val 262331555 ecr 262331555], length 212
192.168.1.203.443 > 192.168.1.204.51674: Flags [.], cksum 0x5bf6 (incorrect -> 0xce96), seq 1019320856, ack 3929326533, win 57, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.255.220.443 > 192.168.1.204.51674: Flags [.], cksum 0x5a08 (incorrect -> 0xd084), seq 1019320856, ack 3929326533, win 57, options [nop,nop,TS val 262331555 ecr 262331555], length 0
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x63d0 (incorrect -> 0x71e7), seq 1019320856:1019322866, ack 3929326533, win 57, options [nop,nop,TS val 262331557 ecr 262331555], length 2010
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x61e2 (incorrect -> 0x73d5), seq 1019320856:1019322866, ack 3929326533, win 57, options [nop,nop,TS val 262331557 ecr 262331555], length 2010
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xc8a0), seq 3929326533, ack 1019322866, win 63, options [nop,nop,TS val 262331557 ecr 262331557], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xc6b2), seq 3929326533, ack 1019322866, win 63, options [nop,nop,TS val 262331557 ecr 262331557], length 0
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5ea1 (incorrect -> 0xd7e0), seq 3929326533:3929327710, ack 1019322866, win 63, options [nop,nop,TS val 262331560 ecr 262331557], length 1177
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x608f (incorrect -> 0xd5f2), seq 3929326533:3929327710, ack 1019322866, win 63, options [nop,nop,TS val 262331560 ecr 262331557], length 1177
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5c29 (incorrect -> 0xdc6f), seq 1019322866:1019322917, ack 3929327710, win 63, options [nop,nop,TS val 262331560 ecr 262331560], length 51
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5a3b (incorrect -> 0xde5d), seq 1019322866:1019322917, ack 3929327710, win 63, options [nop,nop,TS val 262331560 ecr 262331560], length 51
192.168.1.204.51674 > 192.168.255.220.443: Flags [P.], cksum 0x5b37 (incorrect -> 0x9054), seq 3929327710:3929328013, ack 1019322917, win 63, options [nop,nop,TS val 262331561 ecr 262331560], length 303
192.168.1.204.51674 > 192.168.1.203.443: Flags [P.], cksum 0x5d25 (incorrect -> 0x8e66), seq 3929327710:3929328013, ack 1019322917, win 63, options [nop,nop,TS val 262331561 ecr 262331560], length 303
192.168.1.203.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5e86 (incorrect -> 0x4750), seq 1019322917:1019323573, ack 3929328013, win 67, options [nop,nop,TS val 262331562 ecr 262331561], length 656
192.168.255.220.443 > 192.168.1.204.51674: Flags [P.], cksum 0x5c98 (incorrect -> 0x493e), seq 1019322917:1019323573, ack 3929328013, win 67, options [nop,nop,TS val 262331562 ecr 262331561], length 656
192.168.1.204.51674 > 192.168.255.220.443: Flags [.], cksum 0x5a08 (incorrect -> 0xbfdd), seq 3929328013, ack 1019323573, win 69, options [nop,nop,TS val 262331602 ecr 262331562], length 0
192.168.1.204.51674 > 192.168.1.203.443: Flags [.], cksum 0x5bf6 (incorrect -> 0xbdef), seq 3929328013, ack 1019323573, win 69, options [nop,nop,TS val 262331602 ecr 262331562], length 0
```

可以看到这里面一共有三个timeout报错，依次对应抓包socket如下：

```bash
192.168.1.204.50254 > 192.168.2.148.443(宕机前的socket)
192.168.1.204.51328 > 192.168.2.148.443(TCP尝试三次握手Flags [S]，没有成功)
192.168.1.204.51482 > 192.168.2.148.443(TCP尝试三次握手Flags [S]，没有成功)

# TCP handshake
    _____                                                     _____
   |     |                                                   |     |
   |  A  |                                                   |  B  |
   |_____|                                                   |_____|
      ^                                                         ^
      |--->--->--->-------------- SYN -------------->--->--->---|
      |---<---<---<------------ SYN/ACK ------------<---<---<---|
      |--->--->--->-------------- ACK -------------->--->--->---|
```

后面日志正常，对应socket为：`192.168.1.204.51674 > 192.168.1.203.443`。其中`192.168.1.203.443`是其它非宕机母机上的AA

结合上述日志和抓包看，从16:00:53开始宕机，Controller在16:00:58开始尝试获取分布式锁，到16:01:08(10s间隔)超时，于是关闭socket(192.168.1.204.50254 > 192.168.2.148.443)

之后3s(16:01:11)开始第二轮获取，尝试创建TCP连接，这个时候由于没有[超过40s的service ep剔除时间](https://duyanghao.github.io/kubernetes-ha/)，192.168.2.148.443对应的DNAT链(KUBE-SEP-XXX)还存在iptables规则中，iptables轮询机制将192.168.255.220.443(Kubernetes aa service)'随机'转化成了192.168.2.148.443

但是由于192.168.2.148.443已经挂掉，所以TCP三次握手并没有成功，因此也看到了`Client.Timeout exceeded while awaiting headers`的错误(注意：**http.Client.Timeout包括TCP连接建立的时间，虽然设置为30s，但是这里http.Client.Timeout=10s会将其覆盖**)

之后3s(16:01:25)开始第三轮获取，原理类似，不再展开

直到第四次尝试(16:01:39)获取，这个时候已经触发endpoint controller剔除逻辑，于是iptables会将service转化为其它正常母机上的pod(192.168.1.203.443)。访问也就正常了

```bash
$ netstat -tnope 
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.1.204:51674      192.168.255.220:443     ESTABLISHED 0          18568190   26201/controller    keepalive (10.22/0/0)
```

从上面的分析可以看出似乎禁用HTTP/2可以解决问题

另外，社区提交了一个[PR](https://github.com/golang/net/commit/0ba52f642ac2f9371a88bfdde41f4b4e195a37c0)专门用于解决`HTTP/2存在的无法移除底层half-closed tcp连接的问题`，如下：

>> http2: perform connection health check
   
>   After the connection has been idle for a while, periodic pings are sent
   over the connection to check its health. Unhealthy connection is closed
   and removed from the connection pool.

目前最新版本的golang 1.4并没有合并该PR，下一个release(1.5)应该会有所合并

而Kubernetes社区也对应存在着类似的[client-go issue](https://github.com/kubernetes/client-go/issues/374)

## 分析 - 禁用HTTP/2不生效？？？

准备按照如上`禁用HTTP/2`的方法对集群中其它Controller进行测试，发现[cluster-coredns-controller](https://github.com/duyanghao/cluster-coredns-controller)在禁用HTTP/2之后依旧会出现15mins的不可用状态

如下是cluster-coredns-controller架构图：

![](/public/img/kubernetes-ha-http-keep-alives-bugs/cluster-coredns-controller.png)

如下是母机宕机前controller相关信息：

```bash
$ kubectl get pods -o wide
cluster-coredns-controller-j82s8           1/1     Running     0          33s     192.168.1.225   10.0.0.2   <none>           <none>
cluster-coredns-controller-nppjv           1/1     Running     0          44s     192.168.2.165   10.0.0.3   <none>           <none>
cluster-coredns-controller-xnxp8           1/1     Running     0          30s     192.168.0.122   10.0.0.1   <none>           <none>

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  keepalive (7.89/0/0)
tcp        0      0 192.168.0.122:55286      192.168.255.220:443     ESTABLISHED 0          583730245  726/./cluster-cored  keepalive (3.16/0/0)
```

日志如下：

```bash
I0803 09:26:07.795505       1 reflector.go:243] forcing resync
I0803 09:26:07.795602       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:26:07.799698       1 controller.go:227] Successfully synced 'global'
```

这里将10.0.0.3母机关机(模拟宕机)，cluster-coredns-controller现象如下：

```bash
$ log
I0803 09:26:07.795505       1 reflector.go:243] forcing resync
I0803 09:26:07.795602       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:26:07.799698       1 controller.go:227] Successfully synced 'global'

I0803 09:26:37.795700       1 reflector.go:243] forcing resync
I0803 09:26:37.795768       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:27:07.795908       1 reflector.go:243] forcing resync
I0803 09:27:07.795966       1 controller.go:133] Watch Cluster: global Updated ...

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0    261 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  on (81.21/9/0)
tcp        0      0 192.168.0.122:55286      192.168.255.220:443     ESTABLISHED 0          583730245  726/./cluster-cored  keepalive (19.00/0/4)

192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x9fc3), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x9cfe), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x9cfe (correct), seq 2494056844:2494057105, ack 2492532730, win 349, options [nop,nop,TS val 208032352 ecr 1530395], length 261
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x742d (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0xb99d (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x449f (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0xffbc (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0x344c (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x742d (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0x76f2 (correct), seq 2492532730:2492534128, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0xb99d (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0xbc62 (correct), seq 2492536855:2492538253, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [.], cksum 0x449f (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.255.220.443 > 192.168.0.122.46306: Flags [.], cksum 0x4764 (correct), seq 2492534128:2492535526, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1398
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0xffbc (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.255.220.443 > 192.168.0.122.46306: Flags [P.], cksum 0x0282 (correct), seq 2492538253:2492538869, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 616
192.168.2.162.443 > 192.168.0.122.46306: Flags [P.], cksum 0x344c (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.255.220.443 > 192.168.0.122.46306: Flags [P.], cksum 0x3711 (correct), seq 2492535526:2492536855, ack 2494057105, win 122, options [nop,nop,TS val 1560395 ecr 208032352], length 1329
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x93a4), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x90df), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x90df (correct), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xfec5), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xfc00), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xfc00 (correct), seq 2494057105, ack 2492534128, win 347, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xf951), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xf68c), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xf68c (correct), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538253}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58c2 (incorrect -> 0xf6e9), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b87 (incorrect -> 0xf424), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0xf424 (correct), seq 2494057105, ack 2492535526, win 345, options [nop,nop,TS val 208032356 ecr 1560395,nop,nop,sack 1 {2492536855:2492538869}], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x8127), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x7e62), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.46306 > 192.168.2.162.443: Flags [.], cksum 0x7e62 (correct), seq 2494057105, ack 2492538869, win 339, options [nop,nop,TS val 208032356 ecr 1560395], length 0
192.168.0.122.55286 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x7c4d), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x7988), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x7988 (correct), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208057856 ecr 1555815], length 0
192.168.2.162.443 > 192.168.0.122.55286: Flags [.], cksum 0x6580 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
192.168.2.162.443 > 192.168.0.122.55286: Flags [.], cksum 0x6580 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
192.168.255.220.443 > 192.168.0.122.55286: Flags [.], cksum 0x6845 (correct), seq 1455933629, ack 1519823273, win 204, options [nop,nop,TS val 1585896 ecr 207967512], length 0
...
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd5ec), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd327), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd327 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062352 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd521), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd25c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd25c (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062555 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd456), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xd191), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xd191 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208062758 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xd2bf), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xcffa), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xcffa (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063165 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xcf90), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xcccb), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xcccb (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208063980 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xc934), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xc66f), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xc66f (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208065608 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xbc7c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xb9b7), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xb9b7 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208068864 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0xa30c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0xa047), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0xa047 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208075376 ecr 1560395], length 261
192.168.0.122.55286 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x914b), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x5b7b (incorrect -> 0x8e86), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.55286 > 192.168.2.162.443: Flags [.], cksum 0x8e86 (correct), seq 1519823272, ack 1455933629, win 349, options [nop,nop,TS val 208087936 ecr 1585896], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x703c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x6d77), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x6d77 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208088384 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x0a7c), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x07b7), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x07b7 (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208114432 ecr 1560395], length 261
```

5分钟后出现如下日志：

```bash
...
I0803 09:31:07.797244       1 reflector.go:243] forcing resync
I0803 09:31:07.797318       1 controller.go:133] Watch Cluster: global Updated ...
W0803 09:31:34.099349       1 reflector.go:302] watch of *v1.Cluster ended with: an error on the server ("unable to decode an event from the watch stream: read tcp 192.168.0.122:55286->192.168.255.220:443: read: connection timed out") has prevented the request from succeeding
...
```

另外，Controller的socket如下：

```bash
$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0    261 192.168.0.122:46306      192.168.255.220:443     ESTABLISHED 0          583867749  726/./cluster-cored  on (98.33/14/0)
tcp        0      0 192.168.0.122:35258      192.168.255.220:443     ESTABLISHED 0          583982033  726/./cluster-cored  keepalive (10.97/0/0)
```

可以看到tcp socket：`192.168.0.122:55286->192.168.255.220:443`关闭了，并新产生了一个socket。但是Controller依旧异常

抓包如下：

```bash
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x6c75), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x6a70), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x6a70 (correct), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208844928 ecr 268182996], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xcaab (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xcaab (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0xccb0 (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268213076 ecr 208724707], length 0
192.168.0.122.46306 > 192.168.255.220.443: Flags [P.], cksum 0x59bb (incorrect -> 0x7970), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x5c80 (incorrect -> 0x76ab), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.46306 > 192.168.2.162.443: Flags [P.], cksum 0x76ab (correct), seq 2494057105:2494057366, ack 2492538869, win 349, options [nop,nop,TS val 208872448 ecr 1560395], length 261
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x8174), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x7f6f), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x7f6f (correct), seq 2189163102, ack 1656563798, win 106, options [nop,nop,TS val 208875008 ecr 268213076], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0x552b (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0x552b (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0x5730 (correct), seq 1656563798, ack 2189163103, win 76, options [nop,nop,TS val 268243156 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xe11f (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.1.226.443 > 192.168.0.122.35258: Flags [.], cksum 0xe11f (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.255.220.443 > 192.168.0.122.35258: Flags [.], cksum 0xe324 (correct), seq 1656563797, ack 2189163103, win 76, options [nop,nop,TS val 268272864 ecr 208724707], length 0
192.168.0.122.35258 > 192.168.255.220.443: Flags [.], cksum 0x58b6 (incorrect -> 0x97e7), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x5abb (incorrect -> 0x95e2), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length 0
192.168.0.122.35258 > 192.168.1.226.443: Flags [.], cksum 0x95e2 (correct), seq 2189163103, ack 1656563798, win 106, options [nop,nop,TS val 208904715 ecr 268243156], length 
```

可以看到新产生的socket访问正常，但是旧的socket`192.168.0.122.46306 > 192.168.255.220.443`依旧存在访问。现象和上面的Controller如出一撤，区别是这里禁用了HTTP/2

过了15mins后(17:26:31开始宕机，17:42:05恢复)，cluster-coredns-controller日志正常：

```bash
I0803 09:40:35.112293       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:41:05.112424       1 reflector.go:243] forcing resync
I0803 09:41:05.112484       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:41:35.112637       1 reflector.go:243] forcing resync
I0803 09:41:35.112751       1 controller.go:133] Watch Cluster: global Updated ...

I0803 09:42:05.112807       1 reflector.go:243] forcing resync
I0803 09:42:05.112908       1 controller.go:133] Watch Cluster: global Updated ...
I0803 09:42:08.221876       1 controller.go:227] Successfully synced 'global'

$ netstat -tnope
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name     Timer
tcp        0      0 192.168.0.122:35258      192.168.255.220:443     ESTABLISHED 0          583982033  726/./cluster-cored  keepalive (0.84/0/0)
tcp        0      0 192.168.0.122:54190      192.168.255.220:443     ESTABLISHED 0          584112800  726/./cluster-cored  keepalive (25.03/0/0)
```

对应的socket也被删除掉了，并新创建了一个新的socket：`192.168.0.122.54190 > 192.168.255.220.443`

查cluster-coredns-controller代码，发现是http.Client.Timeout没有设置，加上之后，controller force rsync会存在问题，每隔http.Client.Timeout时间会出现request Timeout(没有宕机情况下)，同时关闭底层的tcp socket(也即需要调整timeout参数)

分析到这里就会有如下疑问：

* 其中一个socket 5mins后超时关闭，这个5mins哪里触发的？(并没有设置Timeout)
* 另外一个socket 15mins后超时关闭，这个15mins哪里来的？

## 分析 - TCP ARQ&keepalive

从上面的分析可以看出来：似乎不同的应用上层有不同的Timeout设置(包括no timeout)，但是都会存在15mins的超时，看来从上层分析是看不出来问题了，必须追到底层

于是写了一个http request demo例子(为了脱离业务逻辑，追踪底层)模拟访问AA，希望可以复现15 mins的超时：

```go
// httptest.go
import (
        "crypto/tls"
        "fmt"
        "io/ioutil"
        "net"
        "net/http"
        "time"
)

func doRequest(client *http.Client, url string) {
        fmt.Printf("time: %v, start to get ...\n", time.Now())

        req, err := http.NewRequest("GET", url, nil)
        if err != nil {
                fmt.Printf("time: %v, http NewRequest fail: %v\n", time.Now(), err)
                return
        }
        rsp, err := client.Do(req)
        if err != nil {
                fmt.Printf("time: %v, http Client Do request fail: %v\n", time.Now(), err)
                return
        }
        fmt.Printf("time: %v, http Client Do request successfully\n", time.Now())

        buf, err := ioutil.ReadAll(rsp.Body)
        if err != nil {
                fmt.Printf("time: %v, read http response body fail: %v\n", time.Now(), err)
                return
        }
        defer rsp.Body.Close()
        fmt.Printf("\n end-time: %v \n%s -- \n\n\n", time.Now(), string(buf))
}

func main() {
        tr := &http.Transport{
                TLSClientConfig:   &tls.Config{InsecureSkipVerify: true},
                DialContext: (&net.Dialer{
                        Timeout:   10 * time.Second,
                        KeepAlive: 5 * time.Second,
                        DualStack: true,
                }).DialContext,
        }

        client := &http.Client{Transport: tr}

        for i := 0; i < 500; i++ {
                doRequest(client, "https://10.0.0.3:6443")
                time.Sleep(5 * time.Second)
        }
}
```

在Kubernetes集群外母机10.0.0.126上运行httptest访问10.0.0.3:6443，查看socket如下：

```bash
$ netstat -tnope|grep httptest
tcp        0    127 10.0.0.126:42174       10.0.0.3:6443       ESTABLISHED 0          783573965  16334/./httptest     keepalive (0.48/0/0)
```

关闭母机10.0.0.3，日志如下：

```bash
time: 2020-08-04 15:26:44.476931372 +0800 CST m=+20.008447810, start to get ...
time: 2020-08-04 15:26:44.477686648 +0800 CST m=+20.009203129, http Client Do request successfully

 end-time: 2020-08-04 15:26:44.477726859 +0800 CST m=+20.009243329 
{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User \"system:anonymous\" cannot get path \"/\"","reason":"Forbidden","details":{},"code":403}
 -- 


time: 2020-08-04 15:26:49.477844198 +0800 CST m=+25.009360665, start to get ...
```

抓包如下：

```bash
15:26:34.475194 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 3028317097:3028317224, ack 3343196009, win 292, options [nop,nop,TS val 431222239 ecr 2745903], length 127
15:26:34.475767 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 1:364, ack 127, win 59, options [nop,nop,TS val 2750904 ecr 431222239], length 363
15:26:34.475798 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 364, win 314, options [nop,nop,TS val 431222240 ecr 2750904], length 0
15:26:39.476197 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 127:254, ack 364, win 314, options [nop,nop,TS val 431227240 ecr 2750904], length 127
15:26:39.476718 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 364:727, ack 254, win 59, options [nop,nop,TS val 2755905 ecr 431227240], length 363
15:26:39.476750 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 727, win 335, options [nop,nop,TS val 431227241 ecr 2755905], length 0
15:26:44.477031 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 254:381, ack 727, win 335, options [nop,nop,TS val 431232241 ecr 2755905], length 127
15:26:44.477581 IP 10.0.0.3.sun-sr-https > 10.0.0.126.42174: Flags [P.], seq 727:1090, ack 381, win 59, options [nop,nop,TS val 2760906 ecr 431232241], length 363
15:26:44.477613 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [.], ack 1090, win 356, options [nop,nop,TS val 431232241 ecr 2760906], length 0


15:26:49.477981 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237242 ecr 2760906], length 127
15:26:49.678680 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237443 ecr 2760906], length 127
15:26:49.879674 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431237644 ecr 2760906], length 127
15:26:50.282678 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431238047 ecr 2760906], length 127
15:26:51.087680 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431238852 ecr 2760906], length 127
15:26:52.699685 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431240464 ecr 2760906], length 127
15:26:55.923668 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431243688 ecr 2760906], length 127
15:27:02.379682 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431250144 ecr 2760906], length 127
15:27:15.275684 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431263040 ecr 2760906], length 127
15:27:41.067690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431288832 ecr 2760906], length 127
15:28:32.651690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431340416 ecr 2760906], length 127
15:30:15.691692 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431443456 ecr 2760906], length 127

15:32:16.011697 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431563776 ecr 2760906], length 127

15:34:16.331690 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431684096 ecr 2760906], length 127

15:36:16.651678 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431804416 ecr 2760906], length 127

15:38:16.971688 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 431924736 ecr 2760906], length 127

15:40:17.291684 IP 10.0.0.126.42174 > 10.0.0.3.sun-sr-https: Flags [P.], seq 381:508, ack 1090, win 356, options [nop,nop,TS val 432045056 ecr 2760906], length 127
```

socket如下：

```bash
$ netstat -tnope|grep httptest
tcp        0    127 10.0.0.126:42174       10.0.0.3:6443       ESTABLISHED 0          783573965  16334/./httptest     on (58.06/12/0)
```

经过15mins，程序报错如下：

```bash
...
time: 2020-08-04 15:26:49.477844198 +0800 CST m=+25.009360665, start to get ...
time: 2020-08-04 15:42:27.612016356 +0800 CST m=+963.143532890, http Client Do request fail: Get https://10.0.0.3:6443: dial tcp 10.0.0.3:6443: i/o timeout
time: 2020-08-04 15:42:32.612151633 +0800 CST m=+968.143668107, start to get ...
time: 2020-08-04 15:42:42.612363196 +0800 CST m=+978.143879643, http Client Do request fail: Get https://10.0.0.3:6443: dial tcp 10.0.0.3:6443: i/o timeout
```

同时，tcp socket：`10.0.0.126.42174 > 10.0.0.3.6443` 消失

可以看到在宕机后(15mins内)，httptest依旧在使用tcp socket：`10.0.0.126.42174 > 10.0.0.3.6443`。并不断地间隔发包，那么这些包是什么？？？

是tcp keepalive的包吗？显然不是，因为keepalive的包长度应该为0且是按照固定间隔发送(tcp_keepalive_intvl)；但是这里的包长度为127，且前面间隔短，后面间隔长(最长两分钟)

观察上述包内容，看起来是tcp ARQ(自动重传请求)的包，于是开始研究tcp ARQ与15mins的关系

TCP使用滑动窗口和ARQ机制保障可靠传输：TCP每发送一个报文段，就会对此报文段设置一个超时重传计时器。此计时器设置的超时重传时间RTO（Retransmission Time－Out）应当略大于TCP报文段的平均往返时延RTT(Round Trip Time)，一般可取RTO＝2RTT。当超过了规定的超时重传时间还未收到对此TCP报文段的预期确认信息，则必须重新传输此TCP报文段

在参考了[RFC 6298](https://www.rfc-editor.org/pdfrfc/rfc6298.txt.pdf)后，了解到RTO按照双倍递增的算法进行计算。而Linux对RTO的设置为：RTO的最小值设为200ms（RFC建议1秒），最大值设置为120秒（RFC强制60秒以上）

再观察上面的抓包可以看出：从15:26:49.678680开始按照200ms间隔双倍递增，依次为15:26:49.879674(+0.2s)，15:26:50.282678(+0.4s)，一直到最后15:40:17.291684(+2mins)

tcp共计重试了15次(200ms为第一次)，总共重试时间为：924.6s，也即15mins 25s左右。可以跟上面httptest以及Controller恢复访问的时间对上，这也解释了15mins的来历

**最后验证发现：通过调整TCP ARQ重试次数 或者 设置http.Client.Timeout，发现httptest均会在短时间内关闭连接，服务重新建立连接**

## 解决

通过上面的分析，可以知道是TCP的ARQ机制导致了Controller在母机宕机后15mins内一直超时重试，超时重试失败后，tcp socket关闭，应用重新创建连接

这个问题本质上不是Kubernetes的问题，而是应用在复用tcp socket(长连接)时没有考虑设置超时，导致了母机宕机后，tcp socket没有及时关闭，服务依旧使用失效连接导致异常

要解决这个问题，可以从两方面考虑：

* 应用层使用超时设置或者健康检查机制，从上层保障连接的健康状态 - 作用于该应用
* 底层调整TCP ARQ设置(/proc/sys/net/ipv4/tcp_retries2)，缩小超时重试周期，作用于整个集群

由于应用层的超时或者健康检查机制无法使用统一的方案，这里只介绍如何采用系统配置的方式规避无效连接，如下：

```bash
# 0.2+0.4+0.8+1.6+3.2+6.4+12.8+25.6+51.2+102.4 = 222.6s
$ echo 9 > /proc/sys/net/ipv4/tcp_retries2
```

另外对于推送类的服务，比如Watch，在母机宕机后，可以通过tcp keepalive机制来关闭无效连接(这也是上面测试cluster-coredns-controller时其中一个连接5分钟(30+30*9=300s)断开的原因)：

```bash
# 30 + 30*5 = 180s
$ echo 30 >  /proc/sys/net/ipv4/tcp_keepalive_time
$ echo 30 > /proc/sys/net/ipv4/tcp_keepalive_intvl
$ echo 5 > /proc/sys/net/ipv4/tcp_keepalive_probes
```

注意：在Kubernetes环境中，容器不会直接继承母机tcp keepalive的配置(可以直接继承母机tcp超时重试的配置)，因此必须通过一定方式进行适配。这里介绍其中一种方式，添加initContainers使配置生效：

```yaml
- name: init-sysctl
  image: busybox
  command:
  - /bin/sh
  - -c
  - |
    sysctl -w net.ipv4.tcp_keepalive_time=30
    sysctl -w net.ipv4.tcp_keepalive_intvl=30
    sysctl -w net.ipv4.tcp_keepalive_probes=5
  securityContext:
    privileged: true
```

通过上述的TCP keepalive以及TCP ARQ配置，我们可以将无效连接断开时间缩短到4分钟以内，一定程度上解决了母机宕机导致的连接异常问题。不过最好的解决方案是在应用层设置超时或者健康检查机制及时关闭底层无效连接

## Refs

* [TCP的超时与重传](https://nieyong.github.io/wiki_ny/TCP%E7%9A%84%E8%B6%85%E6%97%B6%E4%B8%8E%E9%87%8D%E4%BC%A0-Timeout%20and%20Retransmission.html)
* [Golang http](https://github.com/duyanghao/kubernetes-reading-notes/blob/master/core/others/golang/http/README.md)
* [Linux Operating Systems - How to Determine if the keepalives are Enabled on a Socket](https://support.hpe.com/hpesc/public/docDisplay?docId=emr_na-c03474784)
* [RTO对tcp超时的影响](http://weakyon.com/2015/07/30/the-impact-fo-rto-to-tcp-timeout.html)
* [Client should expose a mechanism to close underlying TCP connections](https://github.com/kubernetes/client-go/issues/374)
* [Notes on TCP keepalive in Go](https://thenotexpert.com/golang-tcp-keepalive/)
* [Using TCP keepalive with Go](https://felixge.de/2014/08/26/tcp-keepalive-with-golang.html)
* [TCP keepalive overview](https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html)
* [Using TCP keepalive under Linux](https://tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html)

![](/public/img/duyanghao.png)