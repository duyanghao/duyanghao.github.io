---
layout: post
title: 容器安全——镜像加密
date: 2020-2-27 19:10:31
category: 技术
tags: Kubernetes container-runtime image-encryption
excerpt: ​在容器安全涉及的内容中，目前还不十分成熟的部分是镜像加密，镜像加密在数据保密性要求较强的领域中会显得十分重要，例如：金融、银行以及国企等，本文针对社区目前的工作对镜像加密的原理及使用进行分析和讲解
---

## Overview

容器安全可以归纳分类如下：

* 1、容器(+镜像)是否有病毒——代表项目：[clair](https://github.com/quay/clair), [IBM Vulnerability Advisor](https://cloud.ibm.com/docs/services/Registry?topic=va-va_index)
* 2、容器(+镜像)是否被篡改——代表项目：[Notary](https://github.com/theupdateframework/notary)
* 3、容器是否隔离(container runtime security)——代表项目：[runc](https://github.com/opencontainers/runc)
* 4、容器(+镜像)是否保密，也即镜像是否被目标用户运行——代表项目：[containerd](https://github.com/containerd/containerd), [cri-o](https://github.com/cri-o/cri-o)

本文主要讨论第四点——镜像保密性。社区目前对于镜像保密性的工作主要由IBM公司的[Brandon Lum](https://github.com/lumjjb)和[Harshal Patil](https://github.com/harche)主导进行。由于镜像加密特性会改动[OCI标准](https://www.opencontainers.org/)中的镜像格式部分：the Image Specification [image-spec](https://github.com/opencontainers/image-spec)，因此在详细介绍镜像加密原理之前有必要先了解一下`OCI`

## OCI

OCI(Open Container Initiative)由 Docker，CoreOS以及容器行业中的其他领导者在2015年6月启动，致力于围绕容器格式和运行时创建开放的行业标准，主要包含两部分：

* 1、the Runtime Specification (runtime-spec)——负责容器运行时，contains technologies such as namespaces, cgroups, Linux capabilities, as well as SELinux, AppArmor, and Seccomp profiles
* 2、the Image Specification (image-spec)——负责容器镜像标准，主要规范镜像存储格式

社区对于`OCI`规范的实现情况如下：

* [OCI Runtime Spec Implementations](https://github.com/opencontainers/runtime-spec/blob/master/implementations.md): [runc(Runtime (Container))](https://github.com/opencontainers/runc), [hyperhq/runv(Runtime (Virtual Machine))](https://github.com/hyperhq/runv)
* [OCI Image Spec Implementations](https://github.com/opencontainers/image-spec/blob/master/implementations.md)：[Buildah](https://github.com/containers/buildah)(构建`OCI`镜像)，[Skopeo](https://github.com/containers/skopeo)(传输`OCI`镜像, based on [containers/image](https://github.com/containers/image))

支持`OCI Runtime Spec&Image Spec`的项目如下：

* [containerd](https://github.com/containerd/containerd)：在[runc](https://github.com/opencontainers/runc)基础上实现了`OCI Image Spec`部分
* [cri-o](https://github.com/cri-o/cri-o)：在[runc](https://github.com/opencontainers/runc)基础上实现了`OCI Image Spec`部分(base on [containers/image](https://github.com/containers/image), [containers/storage](https://github.com/containers/storage) and [CNI](https://github.com/containernetworking/cni))
* [Podman](https://github.com/containers/libpod)：在[Buildah](https://github.com/containers/buildah)基础上实现了`OCI Image&Runtime Spec`([Buildah and Podman relationship](https://github.com/containers/buildah#buildah-and-podman-relationship))
* [docker](https://github.com/moby/moby)：在[runc](https://github.com/opencontainers/runc)基础上实现了`OCI Image Spec`部分([计划这部分代码切换到containerd](https://github.com/moby/moby/issues/38043))
* [rkt(pronounced like a "rocket")](https://github.com/rkt/rkt)

为此`containerd`, `cri-o`, `docker`以及`rkt`也被广义地称为`container runtime`(High-Level Container Runtimes)，如图：

![](/public/img/image_encryption/runtimes.png)

详情见[An Introduction to Container Runtimes](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r)

## Encrypted Container Image Principle

由于镜像加密是在`OCI`镜像格式上的扩展，我们先介绍现存镜像格式，如下：

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

>> Unlike the image index, which contains information about a set of images that can span a variety of architectures and operating systems, an image manifest provides a configuration and set of layers for a single container image for a specific architecture and operating system.

对该格式说明如下(refers to [Image Manifest Property Descriptions](https://github.com/opencontainers/image-spec/blob/master/manifest.md))：

* schemaVersion：This REQUIRED property specifies the image manifest schema version
* config(belongs to [descriptor](https://github.com/opencontainers/image-spec/blob/master/descriptor.md))：包含镜像的元文件(配置文件)信息
  * mediaType：标识config文件格式(media type of the referenced content)，目前必须实现的格式为[application/vnd.oci.image.config.v1+json](https://github.com/opencontainers/image-spec/blob/master/config.md)
  * digest：config文件内容的`sha256 hash`值
  * size：config文件内容大小(in bytes)
* layers：包含镜像各分层相关信息，内容是由镜像分层(belongs to [descriptor](https://github.com/opencontainers/image-spec/blob/master/descriptor.md))组成的数组
  * mediaType：标识镜像layer文件格式(media type of the referenced content)，目前必须实现的格式列表如下：
    * [application/vnd.oci.image.layer.v1.tar](https://github.com/opencontainers/image-spec/blob/master/layer.md)
    * [application/vnd.oci.image.layer.v1.tar+gzip](https://github.com/opencontainers/image-spec/blob/master/layer.md#gzip-media-types)
    * [application/vnd.oci.image.layer.nondistributable.v1.tar](https://github.com/opencontainers/image-spec/blob/master/layer.md#non-distributable-layers)
    * [application/vnd.oci.image.layer.nondistributable.v1.tar+gzip](https://github.com/opencontainers/image-spec/blob/master/layer.md#gzip-media-types)
  * digest: layer文件内容的`sha256 hash`值
  * size：layer文件内容大小(in bytes)
* annotations(string-string map)：This OPTIONAL property contains arbitrary metadata for the image manifest.

而基于[Proposal: New Mediatype - Container Image Encryption](https://github.com/opencontainers/image-spec/issues/747)的讨论，镜像加密原理图(draft of modifications to the OCI spec on Encrypted Container Images)如下：

![](/public/img/image_encryption/image-spec-encryption.png)

>> Encrypted container images are based on the OCI image spec. The changes to the spec is the adding of the encrypted layer mediatype. An image manifest contains some metadata and a list of layers. We introduce a new layer mediatype suffix +encrypted to represent an encrypted layer. For example, a regular layer with mediatype 'application/vnd.oci.image.layer.v1.tar' when encrypted would be 'application/vnd.oci.image.layer.v1.tar+encrypted'. Because there is some metadata involved with the encryption, an encrypted layer will also contain several annotations with the prefix org.opencontainers.image.enc.

也即在原来格式的基础上添加了一个`mediaType`类型，表示数据文件被加密；同时在`annotation`中添加具体加密相关信息

例如，没加密之前数据如下：

```json
"layers":[
  {
    "mediaType":"application/vnd.oci.image.layer.v1.tar+gzip",
    "digest":"sha256:7c9d20b9b6cda1c58bc4f9d6c401386786f584437abbe87e58910f8a9a15386b",
    "size":760770
  }
]
```

加密之后数据如下：

```json
"layers":[
  {
    "mediaType":"application/vnd.oci.image.layer.v1.tar+gzip+encrypted",
    "digest":"sha256:c72c69b36a886c268e0d7382a7c6d885271b6f0030ff022fda2b6346b2b274ba",
    "size":760770,
    "annotations": {
      "org.opencontainers.image.enc.keys.jwe":"eyJwcm90ZWN0Z...",
      "org.opencontainers.image.enc.pubopts":"eyJjaXBoZXIiOi..."
    }
  }
]
```

另外，通过将加密粒度细化到镜像层，保留了原有镜像格式的`layering` and `deduplication`特性

整个加密体系使用[混合加密方案](https://medium.com/@lumjjb/encrypting-container-images-with-containerd-imgcrypt-3c07f8e8e8d4)：

* 1、使用对称加密算法(i.e. AES)加密镜像分层数据——主要为了提高加密解密速度
* 2、使用非对称加密算法(eg: OpenPGP, JSON Web Encryption (JWE), and PKCS#7)首先用公钥加密上述对称加密的密钥，然后用私钥对加密后的密钥进行解密——主要为了提高密钥管理的灵活性

流程图如下：

![](/public/img/image_encryption/flow_encryption.png)

大致可以归纳为如下步骤：

* 1、产生对称加密密钥(i.e. LEK)，用于加密镜像分层
* 2、利用非对称加密公钥加密LEK，产生Wrapped Key
* 3、将加密后的镜像(包含Wrapped Key)上传到公开仓库，例如：Docker Hub or quay.io
* 4、非对称加密私钥持有者下载该镜像，并用该私钥对该镜像进行解密并运行

## Encrypted Container Image Build ToolChain——containerd imgcrypt

对于镜像加密工具，社区计划支持：`docker CLI`(To integrate into docker CLI, we are currently waiting on [moby/moby#38043](https://github.com/moby/moby/issues/38043). Tracking with [moby/buildkit#714](https://github.com/moby/buildkit/issues/714)), [containerd imgcrypt](https://github.com/containerd/imgcrypt), [skopeo](https://github.com/containers/skopeo)以及[buildah](https://github.com/containers/buildah)，目前已经实现的有`containerd imgcrypt`和`skopeo`

下面我们基于[containerd imgcrypt](https://github.com/containerd/imgcrypt)对上述加密流程进行实操：

```bash
# Step1: Install containerd imgcrypt
$ git clone https://github.com/containerd/imgcrypt.git github.com/containerd/imgcrypt
$ cd github.com/containerd/imgcrypt && make && make install
install

# Step2: Setting up containerd
$ wget https://github.com/containerd/containerd/releases/download/v1.3.0/containerd-1.3.0.linux-amd64.tar.gz
$ tar -xzf containerd-1.3.0.linux-amd64.tar.gz
$ ls bin/
containerd  containerd-shim  containerd-shim-runc-v1  containerd-shim-runc-v2  containerd-stress  ctr
$ cat <<EOF > config.toml
disable_plugins = ["cri"]
root = "/tmp/var/lib/containerd"
state = "/tmp/run/containerd"
[grpc]
  address = "/tmp/run/containerd/mycontainerd.sock"
  uid = 0
  gid = 0
[stream_processors]
    [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
        accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
        returns = "application/vnd.oci.image.layer.v1.tar+gzip"
        path = "/usr/local/bin/ctd-decoder"
    [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
        accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
        returns = "application/vnd.oci.image.layer.v1.tar"
        path = "/usr/local/bin/ctd-decoder"
EOF
$ bin/containerd -c config.toml
[... truncated ...]
INFO[2020-02-26T11:16:07.704084989+08:00] containerd successfully booted in 0.047800s

# Step3: Generate some RSA keys with openssl
$ openssl genrsa -out mykey.pem
Generating RSA private key, 2048 bit long modulus
.......................................................................................................+++
.............................+++
e is 65537 (0x10001)
$ openssl rsa -in mykey.pem -pubout -out mypubkey.pem
writing RSA key

# Step4: Pulling an image
$ chmod 0666 /tmp/run/containerd/containerd.sock
$ CTR="/usr/local/bin/ctr-enc -a /tmp/run/containerd/containerd.sock"
$ $CTR images pull --all-platforms docker.io/library/bash:latest
[... truncated ...]
elapsed: 8.7 s                                                                    total:  39.2 M (4.5 MiB/s)                                       
unpacking linux/amd64 sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/arm/v6 sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/arm/v7 sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/arm64/v8 sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/386 sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/ppc64le sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
unpacking linux/s390x sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f...
done
$ $CTR images list   
REF                           TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
docker.io/library/bash:latest application/vnd.docker.distribution.manifest.list.v2+json sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -   
$ $CTR images layerinfo --platform linux/amd64 docker.io/library/bash:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:c9b1b535fdd91a9855fb7f82348177e5f019329a58c53c47272962dd60f71fc9   linux/amd64   2802957                          
   1   sha256:a3698fd3137d820dbb91b5f96e89af64b196dde4ab4c326dd1bd6291bbd771cf   linux/amd64   3186680                          
   2   sha256:2538f4b330c99e67e96803569826ba62f7a3353d27f2a15bcf1e33d1814fa7ad   linux/amd64       340

# Step5: Encrypting the image
$ $CTR images encrypt --recipient jwe:mypubkey.pem --platform linux/amd64 docker.io/library/bash:latest bash.enc:latest
Encrypting docker.io/library/bash:latest to bash.enc:latest
# The arguments are:
# --recipient jwe:mypubkey.pem: This indicates that we want to encrypt the image using the public key mypubkey.pem that we just generated, and the prefix jwe: indicates that we want to use the encryption scheme JSON web encryption scheme for our encryption metadata.
# --platform linux/amd64: Encrypt only the linux/amd64 image
# docker.io/library/bash:latest - The image to encrypt
# bash.enc:latest - The tag of the encrypted image to be created
# Optional: it is possible to encrypt just certain layers of the image by using the --layer flag
$ $CTR images list
REF                           TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
bash.enc:latest               application/vnd.oci.image.index.v1+json                   sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
docker.io/library/bash:latest application/vnd.docker.distribution.manifest.list.v2+json sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -   
# We can observe the layerinfo to show the encryption
$ $CTR images layerinfo --platform linux/amd64 bash.enc:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:375abe9071e498e57f1cfcf349266948c3760bfb136e051c22a532d38a020e74   linux/amd64   2802957          jwe        [jwe]
   1   sha256:3298803dec193fdd561ea27d0b1469e3f0b5b796962f543f3bcc57cd12ebdf5d   linux/amd64   3186680          jwe        [jwe]
   2   sha256:bdbc04a60398d706a06e5ae88a4cdbfef770c9fe1b12e31fcf5a50532dafad87   linux/amd64       340          jwe        [jwe]
# Strange enough, the layers 'SIZE' of 'bash.enc:latest' is same with the one of 'docker.io/library/bash:latest'
$ $CTR images layerinfo --platform linux/amd64 docker.io/library/bash:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:c9b1b535fdd91a9855fb7f82348177e5f019329a58c53c47272962dd60f71fc9   linux/amd64   2802957                          
   1   sha256:a3698fd3137d820dbb91b5f96e89af64b196dde4ab4c326dd1bd6291bbd771cf   linux/amd64   3186680                          
   2   sha256:2538f4b330c99e67e96803569826ba62f7a3353d27f2a15bcf1e33d1814fa7ad   linux/amd64       340

# Compare other platforms
$ $CTR images layerinfo --platform linux/arm64/v8 bash.enc:latest
   #                                                                    DIGEST         PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:8fa90b21c985a6fcfff966bdfbde81cdd088de0aa8af38110057f6ac408f4408   linux/arm64/v8   2723075                          
   1   sha256:4d89eb6b628bc985ffcdcb939bf05a16194be6aed18c2848e1424d01ca0012fd   linux/arm64/v8   3193678                          
   2   sha256:4818fb358d8f64c071010cb05fe59285888a36edbe1a45a5eed80576b08d312c   linux/arm64/v8       344                          
$ $CTR images layerinfo --platform linux/arm64/v8 docker.io/library/bash:latest
   #                                                                    DIGEST         PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:8fa90b21c985a6fcfff966bdfbde81cdd088de0aa8af38110057f6ac408f4408   linux/arm64/v8   2723075                          
   1   sha256:4d89eb6b628bc985ffcdcb939bf05a16194be6aed18c2848e1424d01ca0012fd   linux/arm64/v8   3193678                          
   2   sha256:4818fb358d8f64c071010cb05fe59285888a36edbe1a45a5eed80576b08d312c   linux/arm64/v8       344                          

# Step6: Pushing to registry
# Let us set up a local registry that we can push the encrypted image to. Note that by default, the docker/distribution registry version 2.7.1 and above supports encrypted OCI images out of the box.
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2.7.1
Unable to find image 'registry:2.7.1' locally
2.7.1: Pulling from library/registry
486039affc0a: Pull complete 
ba51a3b098e6: Pull complete 
8bb4c43d6c8e: Pull complete 
6f5f453e5f2d: Pull complete 
42bc10b72f42: Pull complete 
Digest: sha256:7d081088e4bfd632a88e3f3bcd9e007ef44a796fddfe3261407a3f9f04abe1e7
Status: Downloaded newer image for registry:2.7.1
f65eff4accdc64eca42b5aa2c2c36e831c63172472800c835e031ec758a3ccbe
$ docker ps |grep registry
f65eff4accdc        registry:2.7.1                                 "/entrypoint.sh /etc…"   28 seconds ago      Up 27 seconds       0.0.0.0:5000->5000/tcp   registry
$ $CTR images tag bash.enc:latest localhost:5000/bash.enc:latest
localhost:5000/bash.enc:latest
$ $CTR images list
REF                            TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
bash.enc:latest                application/vnd.oci.image.index.v1+json                   sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
docker.io/library/bash:latest  application/vnd.docker.distribution.manifest.list.v2+json sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
localhost:5000/bash.enc:latest application/vnd.oci.image.index.v1+json                   sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
$ $CTR images push localhost:5000/bash.enc:latest
[... truncated ...]
elapsed: 0.4 s                                                                    total:  39.2 M (97.7 MiB/s)
$ $CTR images rm localhost:5000/bash.enc:latest bash.enc:latest
localhost:5000/bash.enc:latest
bash.enc:latest
$ $CTR images pull localhost:5000/bash.enc:latest
localhost:5000/bash.enc:latest:                                                   resolved       |++++++++++++++++++++++++++++++++++++++| 
index-sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:043e2439dae1a9634be8b3a589c8a2e6048f0f1c0ca9d1516c53cd1ee8a8bdf8: done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:bdbc04a60398d706a06e5ae88a4cdbfef770c9fe1b12e31fcf5a50532dafad87:    done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:375abe9071e498e57f1cfcf349266948c3760bfb136e051c22a532d38a020e74:    done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:3298803dec193fdd561ea27d0b1469e3f0b5b796962f543f3bcc57cd12ebdf5d:    done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:e155078e7721d0d550b869592482f0e2c9b9c40e6a541fb430ab46a278fde5cd:   exists         |++++++++++++++++++++++++++++++++++++++| 
elapsed: 0.2 s                                                                    total:  3.0 Mi (15.1 MiB/s)                                      
unpacking linux/amd64 sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa...
done
$ $CTR images list
REF                            TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
docker.io/library/bash:latest  application/vnd.docker.distribution.manifest.list.v2+json sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
localhost:5000/bash.enc:latest application/vnd.oci.image.index.v1+json                   sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -     

# Step7: Decrypting the image
$ $CTR images decrypt --key mykey.pem --platform linux/amd64 localhost:5000/bash.enc:latest localhost:5000/bash.dec:latest
Decrypting localhost:5000/bash.enc:latest to localhost:5000/bash.dec:latest
$ $CTR images list
REF                            TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
docker.io/library/bash:latest  application/vnd.docker.distribution.manifest.list.v2+json sha256:8193ca23afc8e1069bd3d982733df79c68dbcfcd66b4da6b5d65da85987dae2f 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
localhost:5000/bash.dec:latest application/vnd.oci.image.index.v1+json                   sha256:52dbac1ba1464568b2de8c9bc0d1b08d26786365e415c2c3ccb52a23191afc22 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
localhost:5000/bash.enc:latest application/vnd.oci.image.index.v1+json                   sha256:e1cea4c43a0d43bf3fdd6af66509668f6fe16e433085fcc7cf5b8a4ec48c80aa 5.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
# Notice that the 'DIGEST' of 'localhost:5000/bash.dec:latest' is different with the one of 'docker.io/library/bash:latest' 
# Let's have a look at layerinfos of these images
$ $CTR images layerinfo --platform linux/amd64 docker.io/library/bash:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:c9b1b535fdd91a9855fb7f82348177e5f019329a58c53c47272962dd60f71fc9   linux/amd64   2802957                          
   1   sha256:a3698fd3137d820dbb91b5f96e89af64b196dde4ab4c326dd1bd6291bbd771cf   linux/amd64   3186680                          
   2   sha256:2538f4b330c99e67e96803569826ba62f7a3353d27f2a15bcf1e33d1814fa7ad   linux/amd64       340
$ $CTR images layerinfo --platform linux/amd64 localhost:5000/bash.enc:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:375abe9071e498e57f1cfcf349266948c3760bfb136e051c22a532d38a020e74   linux/amd64   2802957          jwe        [jwe]
   1   sha256:3298803dec193fdd561ea27d0b1469e3f0b5b796962f543f3bcc57cd12ebdf5d   linux/amd64   3186680          jwe        [jwe]
   2   sha256:bdbc04a60398d706a06e5ae88a4cdbfef770c9fe1b12e31fcf5a50532dafad87   linux/amd64       340          jwe        [jwe]
$ $CTR images layerinfo --platform linux/amd64 localhost:5000/bash.dec:latest
   #                                                                    DIGEST      PLATFORM      SIZE   ENCRYPTION   RECIPIENTS
   0   sha256:c9b1b535fdd91a9855fb7f82348177e5f019329a58c53c47272962dd60f71fc9   linux/amd64   2802957                          
   1   sha256:a3698fd3137d820dbb91b5f96e89af64b196dde4ab4c326dd1bd6291bbd771cf   linux/amd64   3186680                          
   2   sha256:2538f4b330c99e67e96803569826ba62f7a3353d27f2a15bcf1e33d1814fa7ad   linux/amd64       340                          
# layers of 'localhost:5000/bash.dec:latest' is same with 'docker.io/library/bash:latest' but the image 'DIGEST' differs 

# Step8: Running a container
# We can now attempt to run the encrypted container image
$ $CTR run --rm localhost:5000/bash.enc:latest test echo 'Hello World!'
ctr: you are not authorized to use this image: missing private key needed for decryption
# We notice that running the image failed. This is because we did not provide the keys for the encrypted image.We can pass the keys in with the --key flag.
$ $CTR run --rm --key mykey.pem localhost:5000/bash.enc:latest test echo 'Hello World!'
Hello World!
```

## Kubernetes Image Encryption Support

目前社区计划对Kubernetes支持[两种镜像加密模式](https://github.com/cri-o/cri-o/blob/master/tutorials/decryption.md#key-models)：

* Node Key Model：将密钥放在Kubernetes工作节点上，以节点为解密粒度(已实现)
* Multitenant Key Model：多租户模型，支持解密粒度为集群和用户(还未实现，based on [this KEP](https://github.com/kubernetes/enhancements/pull/1066))

本文主要探讨第一种模型，如图所示：

![](/public/img/image_encryption/Node-Key-Model.png)

原理很清晰：

* 1、产生加密公钥和私钥，将密钥放在`Kubernetes`集群`Worker`节点指定路径下
* 2、用公钥加密镜像并上传到镜像仓库
* 3、用加密镜像在`Kubernetes`集群创建服务
* 4、`Worker`节点`Kubelet`会调用`container runtime`拉取加密镜像，利用私钥对该镜像进行解密，最终成功运行解密后的镜像

在了解大致流程后，我们进一步探讨一下`Worker`节点上`container runtime`对镜像加密的支持情况

首先，Kubernetes使用[CRI](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)来对接`container runtime`，如图：

![](/public/img/image_encryption/cri.png)

而对于镜像加密特性，社区计划对`container-runtime`支持：[containerd](https://github.com/containerd/containerd)(+[CRI-Containerd](https://github.com/containerd/cri))以及[cri-o](https://github.com/cri-o/cri-o)，目前已经实现的有`containerd`([CRI-Containerd目前正在合并PR](https://github.com/containerd/cri/pull/1340))以及`cri-o`。因此这里我们主要讨论`containerd`以及`cri-o`等计划支持镜像加密特性的`container runtime`实现

这里给出`cri`结合`cri-o`，`containerd`以及`docker`的使用图示：

![](/public/img/image_encryption/kubernetes-cri.png)

通过上述图示，我们可以更加直观地看出`Kubernetes`对以上`container runtime`在使用上的细节

## Kubernetes Image Encryption Support-CRIO

本章给出`Kubernetes`结合`CRIO`基于`Node Key Model`镜像加密模式的使用示例：

```bash
# Step1: Install cri-o from source code
# Install Runtime dependencies
$ yum install -y \
  containers-common \
  device-mapper-devel \
  git \
  glib2-devel \
  glibc-devel \
  glibc-static \
  go \
  gpgme-devel \
  libassuan-devel \
  libgpg-error-devel \
  libseccomp-devel \
  libselinux-devel \
  pkgconfig \
  make \
  runc

# Compile cri-o from source code
$ git clone https://github.com/cri-o/cri-o go/src/github.com/cri-o/cri-o
$ cd go/src/github.com/cri-o/cri-o && make
mkdir -p "/root/go/src/github.com/cri-o/cri-o/_output/src/github.com/cri-o"
ln -s "/root/go/src/github.com/cri-o/cri-o" "/root/go/src/github.com/cri-o/cri-o/_output/src/github.com/cri-o/cri-o"
touch "/root/go/src/github.com/cri-o/cri-o/_output/.gopathok"
GO111MODULE=on go build --mod=vendor -ldflags '-s -w -X main.gitCommit="186c23056ac25396e9ea9e49c5e5bfbe271b7751" -X main.buildInfo=1582724503' -tags "containers_image_ostree_stub  exclude_graphdriver_btrfs btrfs_noversion    seccomp selinux" -o bin/crio github.com/cri-o/cri-o/cmd/crio
GO111MODULE=on go build --mod=vendor -ldflags '-s -w -X main.gitCommit="186c23056ac25396e9ea9e49c5e5bfbe271b7751" -X main.buildInfo=1582724526' -tags "containers_image_ostree_stub  exclude_graphdriver_btrfs btrfs_noversion    seccomp selinux" -o bin/crio-status github.com/cri-o/cri-o/cmd/crio-status
make -C pinns
make[1]: Entering directory `/root/go/src/github.com/cri-o/cri-o/pinns'
cc -std=c99 -Os -Wall -Wextra -static   -c -o pinns.o pinns.c
cc -o ../bin/pinns pinns.o -std=c99 -Os -Wall -Wextra -static 
make[1]: Leaving directory `/root/go/src/github.com/cri-o/cri-o/pinns'
./bin/crio --config=""  config  > crio.conf
(/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio-status.8.md -out docs/crio-status.8.tmp && touch docs/crio-status.8.tmp && mv docs/crio-status.8.tmp docs/crio-status.8) || \
        (/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio-status.8.md -out docs/crio-status.8.tmp && touch docs/crio-status.8.tmp && mv docs/crio-status.8.tmp docs/crio-status.8)
(/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio.conf.5.md -out docs/crio.conf.5.tmp && touch docs/crio.conf.5.tmp && mv docs/crio.conf.5.tmp docs/crio.conf.5) || \
        (/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio.conf.5.md -out docs/crio.conf.5.tmp && touch docs/crio.conf.5.tmp && mv docs/crio.conf.5.tmp docs/crio.conf.5)
(/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio.8.md -out docs/crio.8.tmp && touch docs/crio.8.tmp && mv docs/crio.8.tmp docs/crio.8) || \
        (/root/go/src/github.com/cri-o/cri-o/build/bin/go-md2man -in docs/crio.8.md -out docs/crio.8.tmp && touch docs/crio.8.tmp && mv docs/crio.8.tmp docs/crio.8)
$ make install  
install  -D -m 755 bin/crio /usr/local/bin/crio
install  -D -m 755 bin/crio-status /usr/local/bin/crio-status
install  -D -m 755 bin/pinns /usr/local/bin/pinns
install  -d -m 755 /usr/local/share/man/man5
install  -d -m 755 /usr/local/share/man/man8
install  -m 644 docs/crio.conf.5 -t /usr/local/share/man/man5
install  -m 644 docs/crio-status.8 docs/crio.8 -t /usr/local/share/man/man8
install  -d -m 755 /usr/local/share/bash-completion/completions
install  -d -m 755 /usr/local/share/fish/completions
install  -d -m 755 /usr/local/share/zsh/site-functions
install  -D -m 644 -t /usr/local/share/bash-completion/completions completions/bash/crio
install  -D -m 644 -t /usr/local/share/fish/completions completions/fish/crio.fish
install  -D -m 644 -t /usr/local/share/zsh/site-functions  completions/zsh/_crio
install  -D -m 644 -t /usr/local/share/bash-completion/completions completions/bash/crio-status
install  -D -m 644 -t /usr/local/share/fish/completions completions/fish/crio-status.fish
install  -D -m 644 -t /usr/local/share/zsh/site-functions  completions/zsh/_crio-status

# Download conmon
$ git clone https://github.com/containers/conmon ~/go/src/github.com/containers/conmon
$ cd ~/go/src/github.com/containers/conmon
$ make  
mkdir -p bin
cc -std=c99 -Os -Wall -Wextra -Werror -I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include   -DVERSION=\"2.0.11-dev\" -DGIT_COMMIT=\""86aa80b908d7532ab9b6edd6f4f27ba4bf6ba17b"\"   -D USE_JOURNALD=0 -o src/conmon.o -c src/conmon.c
cc -std=c99 -Os -Wall -Wextra -Werror -I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include   -DVERSION=\"2.0.11-dev\" -DGIT_COMMIT=\""86aa80b908d7532ab9b6edd6f4f27ba4bf6ba17b"\"   -D USE_JOURNALD=0 -o src/cmsg.o -c src/cmsg.c
cc -std=c99 -Os -Wall -Wextra -Werror -I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include   -DVERSION=\"2.0.11-dev\" -DGIT_COMMIT=\""86aa80b908d7532ab9b6edd6f4f27ba4bf6ba17b"\"   -D USE_JOURNALD=0 -o src/ctr_logging.o -c src/ctr_logging.c
cc -std=c99 -Os -Wall -Wextra -Werror -I/usr/include/glib-2.0 -I/usr/lib64/glib-2.0/include   -DVERSION=\"2.0.11-dev\" -DGIT_COMMIT=\""86aa80b908d7532ab9b6edd6f4f27ba4bf6ba17b"\"   -D USE_JOURNALD=0 -o src/utils.o -c src/utils.c
cc  -o bin/conmon src/conmon.o src/cmsg.o src/ctr_logging.o src/utils.o -lglib-2.0   -lsystemd 
$ make install
install  -D -m 755 bin/conmon /usr/local/bin/conmon

# Setup CNI networking
# Set CNI network configurations
$ cd /root/go/src/github.com/cri-o/cri-o && mkdir -p /etc/cni/net.d
$ cp contrib/cni/*.conf /etc/cni/net.d/
$ ls /etc/cni/net.d/
10-crio-bridge.conf  99-loopback.conf
# Install the CNI plugins
$ git clone https://github.com/containernetworking/plugins ~/go/src/github.com/containernetworking/plugins
$ cd ~/go/src/github.com/containernetworking/plugins && git checkout v0.8.5
$ ./build_linux.sh
Building plugins 
  bandwidth
  firewall
  flannel
  portmap
  sbr
  tuning
  bridge
  host-device
  ipvlan
  loopback
  macvlan
  ptp
  vlan
  dhcp
  host-local
  static
$ mkdir -p /opt/cni/bin
$ cp bin/* /opt/cni/bin/

# Generating CRI-O configuration
$ cd ~/go/src/github.com/cri-o/cri-o/ && make install.config
install  -d /usr/local/share/containers/oci/hooks.d
install  -D -m 644 crio.conf /etc/crio/crio.conf
install  -D -m 644 crio-umount.conf /usr/local/share/oci-umount/oci-umount.d/crio-umount.conf
install  -D -m 644 crictl.yaml /etc
# Path to OCI hooks directories for automatically executed hooks.
hooks_dir = [
        "/usr/local/share/containers/oci/hooks.d",
]

# Validate registries in registries.conf
# Edit /etc/containers/registries.conf and verify that the registries option has valid values in it.
$ cat /etc/containers/registries.conf
[... truncated ...]
[registries.search]
registries = ['registry.access.redhat.com', 'docker.io', 'registry.fedoraproject.org', 'quay.io', 'registry.centos.org']

[registries.insecure]
registries = []

[registries.block]
registries = []

# Recommended - Use systemd cgroups.
# By default, CRI-O uses cgroupfs as a cgroup manager. However, we recommend using systemd as a cgroup manager. You can change your cgroup manager in crio.conf:
cgroup_manager = "systemd"

# Optional - Modify verbosity of logs in /etc/crio/crio.conf
# Users can modify the log_level field in /etc/crio/crio.conf to change the verbosity of the logs. Options are fatal, panic, error (default), warn, info, and debug.
log_level = "info"

# Starting CRI-O
$ make install.systemd
install  -D -m 644 contrib/systemd/crio.service /usr/local/lib/systemd/system/crio.service
ln -sf crio.service /usr/local/lib/systemd/system/cri-o.service
install  -D -m 644 contrib/systemd/crio-shutdown.service /usr/local/lib/systemd/system/crio-shutdown.service
install  -D -m 644 contrib/systemd/crio-wipe.service /usr/local/lib/systemd/system/crio-wipe.service
$ systemctl daemon-reload
$ systemctl enable crio
Created symlink from /etc/systemd/system/multi-user.target.wants/crio.service to /usr/local/lib/systemd/system/crio.service.
# enable crio proxy(pulling image)
$ cat /usr/local/lib/systemd/system/crio.service
[... truncated ...]
[Service]
Environment="GOTRACEBACK=crash" "HTTP_PROXY=x.x.x.x" "HTTPS_PROXY=x.x.x.x" "NO_PROXY=localhost,127.0.0.1"
$ systemctl start crio
$ systemctl status crio
* crio.service - Container Runtime Interface for OCI (CRI-O)
   Loaded: loaded (/usr/local/lib/systemd/system/crio.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-02-26 22:27:38 CST; 1min 11s ago
     Docs: https://github.com/cri-o/cri-o
 Main PID: 16267 (crio)
   CGroup: /system.slice/crio.service
           `-16267 /usr/local/bin/crio

Feb 26 22:27:37 vm-xxx-centos systemd[1]: Starting Container Runtime Interface for OCI (CRI-O)...
Feb 26 22:27:37 vm-xxx-centos crio[16267]: time="2020-02-26 22:27:37.855228868+08:00" level=warning msg="Not using nat...o fix"
Feb 26 22:27:37 vm-xxx-centos crio[16267]: time="2020-02-26 22:27:37.855645605+08:00" level=info msg="Found CNI networ....conf"
Feb 26 22:27:37 vm-xxx-centos crio[16267]: time="2020-02-26 22:27:37.855703783+08:00" level=info msg="Found CNI networ....conf"
Feb 26 22:27:38 vm-xxx-centos crio[16267]: W0226 22:27:38.114732   16267 hostport_manager.go:69] The binary conntrack ...eanup.
Feb 26 22:27:38 vm-xxx-centos crio[16267]: time="2020-02-26 22:27:38.114771405+08:00" level=info msg="no seccomp profi...fault"
Feb 26 22:27:38 vm-xxx-centos systemd[1]: Started Container Runtime Interface for OCI (CRI-O).  

# Step2: Install crictl
$ VERSION="v1.17.0"
$ wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz
$ tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
crictl
$ rm -f crictl-$VERSION-linux-amd64.tar.gz
# To ensure that we’ve setup CRI-O correctly, we will install crictl to verify our setup.
$ crictl -r unix:///run/crio/crio.sock info 
{
  "status": {
    "conditions": [
      {
        "type": "RuntimeReady",
        "status": true,
        "reason": "",
        "message": ""
      },
      {
        "type": "NetworkReady",
        "status": true,
        "reason": "",
        "message": ""
      }
    ]
  }
}

# Step3: Setting up a single node k8s cluster
$ wget https://github.com/kubernetes/kubernetes/archive/v1.17.1.tar.gz
$ tar -xzf v1.17.1.tar.gz && cd kubernetes-1.17.1
$ hack/install-etcd.sh   
Downloading https://github.com/coreos/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz succeed
etcd v3.4.3 installed. To use:
export PATH="/root/go/src/github.com/kubernetes-1.17.1/third_party/etcd:${PATH}"
$ export PATH="/root/go/src/github.com/kubernetes-1.17.1/third_party/etcd:${PATH}"
$ CGROUP_DRIVER=systemd \
CONTAINER_RUNTIME=remote \
CONTAINER_RUNTIME_ENDPOINT='unix:///var/run/crio/crio.sock' \
./hack/local-up-cluster.sh
[... truncated ...]
To start using your cluster, you can open up another terminal/tab and run:

  export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig
  cluster/kubectl.sh

Alternatively, you can write to the default kubeconfig:

  export KUBERNETES_PROVIDER=local

  cluster/kubectl.sh config set-cluster local --server=https://localhost:6443 --certificate-authority=/var/run/kubernetes/server-ca.crt
  cluster/kubectl.sh config set-credentials myself --client-key=/var/run/kubernetes/client-admin.key --client-certificate=/var/run/kubernetes/client-admin.crt
  cluster/kubectl.sh config set-context local --cluster=local --user=myself
  cluster/kubectl.sh config use-context local
  cluster/kubectl.sh
$ cluster/kubectl.sh get nodes
NAME        STATUS   ROLES    AGE   VERSION
127.0.0.1   Ready    <none>   21m   v1.17.1

# Step4: Generating encrypted image with skopeo
# Install skopeo
$ yum install libgpgme-devel device-mapper-devel libbtrfs-devel glib2-devel libassuan-devel
$ git clone https://github.com/containers/skopeo $GOPATH/src/github.com/containers/skopeo
$ cd $GOPATH/src/github.com/containers/skopeo && make binary-local
$ make install
$ mkdir /etc/containers
$ cp default-policy.json /etc/containers/policy.json
# Pull image
$ skopeo copy docker://docker.io/library/nginx:1.15 oci:local_nginx:1.15
Getting image source signatures
Copying blob 743f2d6c1f65 done  
Copying blob 6bfc4ec4420a done  
Copying blob 688a776db95f done  
Copying config 0fb15759be done  
Writing manifest to image destination
Storing signatures
# Encrypt image
$ skopeo  copy --encryption-key jwe:./mypubkey.pem oci:local_nginx:1.15 oci:nginx_encrypted:1.15
Getting image source signatures
Copying blob 743f2d6c1f65 done  
Copying blob 6bfc4ec4420a done  
Copying blob 688a776db95f done  
Copying config 0fb15759be done  
Writing manifest to image destination
Storing signatures
# Push image
$ skopeo  copy --dest-tls-verify=false oci:nginx_encrypted:1.15 docker://x.x.x.x/nginx_encrypted:1.15  
Getting image source signatures
Copying blob cd2a4504255e done  
Copying blob 8df8b398a84d done  
Copying blob 663a05dac0ac done  
Copying config 0fb15759be done  
Writing manifest to image destination
Storing signatures

# Step5: Getting and running an encrypted image
$ cat <<EOF > enc-dply.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: enc-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: x.x.x.x/nginx_encrypted:1.15
        ports:
        - containerPort: 80
EOF
$ cluster/kubectl.sh create -f enc-dply.yaml
deployment.apps/enc-nginx created
$ cluster/kubectl.sh get pods
NAME                         READY   STATUS             RESTARTS   AGE
enc-nginx-7fb4578896-9qhtr   0/1     ImagePullBackOff   0          12s
enc-nginx-7fb4578896-cdrdq   0/1     ErrImagePull       0          12s
enc-nginx-7fb4578896-j94rj   0/1     ErrImagePull       0          12s
# We notice that our pods failed to run due to failure in the image pull process. Let’s describe the pod to find out more.
$ cluster/kubectl.sh describe pods/enc-nginx-7fb4578896-9qhtr
[... truncated ...]
Events:
  Type     Reason     Age                From                Message
  ----     ------     ----               ----                -------
  Normal   Scheduled  <unknown>          default-scheduler   Successfully assigned default/enc-nginx-7fb4578896-9qhtr to 127.0.0.1
  Normal   Pulling    25s (x2 over 36s)  kubelet, 127.0.0.1  Pulling image "x.x.x.x/nginx_encrypted:1.15"
  Warning  Failed     25s (x2 over 36s)  kubelet, 127.0.0.1  Failed to pull image "x.x.x.x/nginx_encrypted:1.15": rpc error: code = Unknown desc = Error decrypting layer sha256:cd2a4504255eaf69fb1ee6c5961625854cd69597a1c5722e86062fb4ab688cf8: missing private key needed for decryption
  Warning  Failed     25s (x2 over 36s)  kubelet, 127.0.0.1  Error: ErrImagePull
  Normal   BackOff    13s (x2 over 36s)  kubelet, 127.0.0.1  Back-off pulling image "x.x.x.x/nginx_encrypted:1.15"
  Warning  Failed     13s (x2 over 36s)  kubelet, 127.0.0.1  Error: ImagePullBackOff
# Configuring decryption keys for CRI-O
$ mkdir -p /etc/crio/keys
$ cat << EOF > /etc/crio/keys/mykey.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA3B07uvDP+NxYXYQxjknPml1zSacijqoj1D79VzSUjjMM6th/
doKJbeMTybehF9PDly6TleluZWZiXMtqagmeEI/sUKaaq0yrMrNrzfIfJYCD0hJx
8tBFZKwUVKaYQjB4Bo6Ij0V04dcP8m6NAfXNG0/MB56zj7WhMndOPP/tuIGmj1jV
jBopq/Pm8KUrzDVCmD+kQLT7hZTi0lapG7A/tE7bc2pGypP2hLR0i9/ckieXh4IL
YKdaNda4THjnA3bHC7/ELOWPinrmQlaxY1mW8hkLCHCQ/MG/SJKkVTY2fX62BscY
KGQPOwh9jNWIS8MAbNJafYBxVueje6Xe/e+OdwIDAQABAoIBAQDXOH4+u1eerVR5
m9gYmHM1LEqdqZ5QgGuoDC8KJY9buu7WcfmvltNpbq7afYI2GgkUuaX03tniq8lh
kkPqipzS9ObLtRtmgwCiAm1WYXey44YA0ag5EwvG87qtSnd1wI6bWqKL9A3lBLPD
B/U4BW8XVV7Z1IMd8So8fgsx+cwmqk2+CqMZsq2m7FBjrCt7fu0QMaOXvuUWka5D
K8F+yvAbD5UMXOqN9fjMuPzbtiOFXvmWtpC3L6wxZjmAPuiR9fNzwM4jCXSasMU8
PNy3x0Esn/ic7IP+1v1PEiXImLui4WPF74v8i6HB6FgxCBht6rCYyl8kYh68gJS6
HTmgf9+BAoGBAPk+ft+R4T/iGDZ0y1YhRNN5zxqtOEcLf1+5/s6W6b+FTRFrxYLL
A8ZMkn0Se8DfVjPald1gMSikE8vj8YJSHAMlstPiHiYQKsBMGbjksZLjBpSapkTy
iMjpk+SmVN3sZRU3jfrcMoPfAQx0hBrCUJdhZFV/aYLiuJ7GUub34gKzAoGBAOIU
mu5aht3MDjiR2mnNzEj2TY7YED7HNWOQJ0FzqLfl4jZTowMbnz8aFUWxkJmC3i1y
0Y8rWbxV7CwcXc9fOR8jRNx2Jn6eaA4A9+a02D1hNHJ6bQivy6yloQXCs4/wardY
EBHvepygs+v507khX146bl8PmUCIYpjVFaeOrRctAoGAehhLPmnP1eODyOld0ktp
086PzZmdP/A57ULHt5vl1ZQPNMF+d5vLtZA9ElfDl6/QIoapc1BzxFzb9b0ryZM/
das59uGFs0+oIZsl3pTpB/N+fb1kRdIpf4IsmI2CdVQgEEyumHzVohPUB63sKM+X
exCSfe90WFGH7v9oDQzRAlECgYBocSRx4JhVdqNLNvYz0sMBIegKiX5XwifD6yB3
eDsFWcn7VwADu4sB18bj/3fRs0d4r4ZoIZq/CuKkLiaYWmFFJUH2pw55iCyB66ia
iAktse5MxIoCbVQmWg3dX2kcofBq6t/hqUR3fzYfWbaZ2/T2zv+WItqlmVwTRr1O
PvdvsQKBgQDA0MQnQ7iV6Uvdm2AFHXlnwtXGelG0EsaNabF31LosHujMf92kUzaC
12NiuE0gEfuPemEw+MFQ/nfBiEDv23NgRz6frZvcvUYWr/xFs8AwPkV5pBFNwoa2
4lYPbMB/1RWf4zDKvDBKEpEGr3pCKBjrPszi0skn2DTCcpiqjH3XGg==
-----END RSA PRIVATE KEY-----
EOF
$ cluster/kubectl.sh delete -f enc-dply.yaml
deployment.apps "enc-nginx" deleted
$ cluster/kubectl.sh create -f enc-dply.yaml
deployment.apps/enc-nginx created
$ cluster/kubectl.sh get pods -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
enc-nginx-7fb4578896-8grjd   1/1     Running   0          61s   10.88.0.102   127.0.0.1   <none>           <none>
enc-nginx-7fb4578896-jjhz4   1/1     Running   0          61s   10.88.0.100   127.0.0.1   <none>           <none>
enc-nginx-7fb4578896-jv9n5   1/1     Running   0          61s   10.88.0.101   127.0.0.1   <none>           <none>
# test with curl
$ curl -v 10.88.0.92
< HTTP/1.1 200 OK
< Server: nginx/1.15.12
< Date: Thu, 27 Feb 2020 09:47:44 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Tue, 16 Apr 2019 13:08:19 GMT
< Connection: keep-alive
< ETag: "5cb5d3c3-264"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## [The main integration points for Encrypted Container Images](https://github.com/opencontainers/image-spec/pull/775#issuecomment-540060318)

#### 镜像加密工具

对于镜像加密工具，社区计划支持：`docker CLI`(To integrate into docker CLI, we are currently waiting on [moby/moby#38043](https://github.com/moby/moby/issues/38043). Tracking with [moby/buildkit#714](https://github.com/moby/buildkit/issues/714)), [containerd imgcrypt](https://github.com/containerd/imgcrypt), [skopeo](https://github.com/containers/skopeo)以及[buildah](https://github.com/containers/buildah)，目前已经实现的有`containerd imgcrypt`和`skopeo`

#### 镜像仓库

[Docker Distribution](https://github.com/docker/distribution) >= `v2.7.1` 支持镜像加密OCI格式

#### Container Runtime

于Container Runtime，社区计划支持：[containerd](https://github.com/containerd/containerd)(+[CRI-Containerd](https://github.com/containerd/cri))以及[cri-o](https://github.com/cri-o/cri-o)(暂不支持Docker)，目前已经实现的有`containerd`([CRI-Containerd目前正在合并PR](https://github.com/containerd/cri/pull/1340))以及`cri-o`

#### Kubernetes

目前社区计划对Kubernetes支持两种镜像加密模式：
* Node Key Model：将密钥放在Kubernetes工作节点上，以节点为解密粒度(CRIO已经实现，CRI-Containerd待合并PR)
* Multitenant Key Model：多租户模型，支持解密粒度为集群和用户(还未实现，based on [this KEP](https://github.com/kubernetes/enhancements/pull/1066))
  
另外，对于`Node Key Model`模式，测试加密流程可行，并对性能评估如下：
* 1、目前社区对镜像加密这块并没有性能测试数据
* 2、利用对称加密算法(i.e. AES)加密镜像Layer，加密后数据长度不变——对镜像pull影响不大&解密速度快，耗费时间少
* 3、Memory would not be that bad since we are using a stream cipher
  
**补充：目前社区对于Docker的支持计划参考如下(我的思考)：应该是等到Docker `OCI`部分切换到`Containerd`，然后对于`CRI-Docker`写插件支持镜像加密，所以目前是阻塞的状态，参考[The main integration points for Encrypted Container Images](https://github.com/opencontainers/image-spec/pull/775#issuecomment-540060318)**

## Conclusion

​在容器安全涉及的内容中，目前还不十分成熟的部分是`镜像加密`，`镜像加密`在数据保密性要求较强的领域中会显得十分重要，例如：金融、银行以及国企等

本文首先介绍了`镜像加密`涉及的`OCI`规范，在此基础上对`镜像加密`原理以及流程进行了说明，并利用[containerd imgcrypt](https://github.com/containerd/imgcrypt)工具对加密流程进行了实操。而`Kubernetes`对镜像加密特性支持`Node Key Model`以及`Multitenant Key Model`这两种使用模式，本文详细讲解了`Kubernetes`基于`Node Key Model`使用模式的原理&流程，并结合`CRIO`演示了镜像加密`Kubernetes-Native`的用法

虽然目前基于社区的工作可以在`Kubernetes`上使用镜像加密特性，但是该特性的支持还并不完善，比如：对于加密工具，还需要支持`docker CLI`(To integrate into docker CLI, we are currently waiting on [moby/moby#38043](https://github.com/moby/moby/issues/38043). Tracking with [moby/buildkit#714](https://github.com/moby/buildkit/issues/714))；对于容器运行时，还需要支持[CRI-Containerd](https://github.com/containerd/cri)以及`Docker`等；对于云原生生态系统，还需要更多地与镜像相关的项目(such as Notary, Clair and so on)进行合作；对于Kubernetes，还需要支持更多Key Models

## Refs

* [Encrypting container images with containerd imgcrypt!](https://medium.com/@lumjjb/encrypting-container-images-with-containerd-imgcrypt-3c07f8e8e8d4)
* [The main integration points for Encrypted Container Images](https://github.com/opencontainers/image-spec/pull/775#issuecomment-540060318)
* [current process](https://github.com/kubernetes/kubernetes/pull/78975#issuecomment-589560959)
* [How Encrypted Images brings about compliance in Kubernetes (via CRI-O)](https://medium.com/@lumjjb/how-encrypted-images-brings-about-compliance-in-kubernetes-via-cri-o-6ab58fad6124)
* [Support for Encrypted Images in Kubernetes](https://github.com/kubernetes/enhancements/issues/1067)
* [Add Image Decryption KEP](https://github.com/kubernetes/enhancements/pull/1066)
* [What’s the difference between runc, containerd, docker? Well, I asked myself the exact same question…](https://medium.com/@alenkacz/whats-the-difference-between-runc-containerd-docker-3fc8f79d4d6e)
* [Container Runtimes Part 1: An Introduction to Container Runtimes](https://www.ianlewis.org/en/container-runtimes-part-1-introduction-container-r)
* [Encrypted container images for container image security at rest](https://developer.ibm.com/articles/encrypted-container-images-for-container-image-security-at-rest/)
* [Running CRI-O on kubernetes cluster](https://github.com/cri-o/cri-o/blob/master/tutorials/kubernetes.md)
* [Encrypting container images with skopeo](https://medium.com/@lumjjb/encrypting-container-images-with-skopeo-f733afb1aed4)
* [OCI Container Tools Guide](https://github.com/containers/buildah/tree/master/docs/containertools)
* [OCI Image Implementations](https://github.com/opencontainers/image-spec/blob/master/implementations.md)
* [OCI Runtime Implementations](https://github.com/opencontainers/runtime-spec/blob/master/implementations.md)
* [OCI Image Manifest Specification](https://github.com/opencontainers/image-spec/blob/master/manifest.md)
* [CRI: the Container Runtime Interface](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md#design-docs-and-proposals)

![](/public/img/duyanghao.png)