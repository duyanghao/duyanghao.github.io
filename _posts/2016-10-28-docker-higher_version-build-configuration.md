---
layout: post
title: docker build configuration分析
date: 2016-10-28 21:41:55
category: 技术
tags: Docker-registry Docker Docker-build
excerpt: 分析docker高版本build中镜像configuration文件如何生成
---

分析`docker build`中，镜像configuration文件如何生成，重点几个信息：`created`、`config`、`author`、`config.Image`等

configuration文件`example`：

```sh
{"architecture":"amd64","config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["sh"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"container":"7dfa08cb9cbf2962b2362b1845b6657895685576015a8121652872fea56a7509","container_config":{"Hostname":"xxxx","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","dd if=/dev/zero of=file bs=10M count=1"],"ArgsEscaped":true,"Image":"sha256:9e301a362a270bcb6900ebd1aad1b3a9553a9d055830bdf4cab5c2184187a2d1","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"created":"2016-08-18T06:13:28.269459769Z","docker_version":"1.11.0-dev","history":[{"created":"2015-09-21T20:15:47.433616227Z","created_by":"/bin/sh -c #(nop) ADD file:6cccb5f0a3b3947116a0c0f55d071980d94427ba0d6dad17bc68ead832cc0a8f in /"},{"created":"2015-09-21T20:15:47.866196515Z","created_by":"/bin/sh -c #(nop) CMD [\"sh\"]"},{"created":"2016-08-18T06:13:28.269459769Z","created_by":"/bin/sh -c dd if=/dev/zero of=file bs=10M count=1"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:ae2b342b32f9ee27f0196ba59e9952c00e016836a11921ebc8baaf783847686a","sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef","sha256:d13087c084482a01b15c755b55c5401e5514057f179a258b7b48a9f28fde7d06"]}}
```