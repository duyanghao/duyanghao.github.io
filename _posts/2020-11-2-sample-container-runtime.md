---
layout: post
title: sample container runtime
date: 2020-11-2 19:10:31
category: 技术
tags: Kubernetes Docker runc container-runtime
excerpt: 本文描述了sample-container-runtime初始版本的核心实现细节
---

## 前言

无论是虚拟化技术还是容器技术都是为了最大程度解决母机资源利用率的问题。虚拟化技术利用Hypervisor(运行在宿主机OS上)将底层硬件进行了虚拟，使得在每台VM看来，硬件都是独占的，并且由VM Guest OS直接操作(具备最高操作权限)；而容器共享母机OS，每个容器只包含应用以及应用所依赖的库和二进制文件；宿主机内核的namespace隔离特性，cgroups(资源控制)，以及联合文件系统使得多个容器之间相互隔离，同时文件系统视图也各不相同。总的来说：容器技术相比虚拟机而言，更加轻量级，同时也具备更高的执行效率

![](/public/img/sample-container-runtime/docker-vs-vm.png)

对于容器来说，最具有代表性的项目就是Docker。Docker自2013年由DotCloud开源后，便席卷整个容器技术圈。它通过设计和封装用户友好的操作接口，使得整个容器技术使用门槛大大降低；同时它也系统地构建了应用打包(Docker build)，分发(Docker pull&push)标准和工具，使得整个容器生命周期管理更加容易和可实施。对于Docker来说，可以简单的认为它并没有创造新的技术，而是将内核的namespace(进程隔离)，cgroups(进程资源控制)，Capabilities，Apparmro以及seccomp(安全防护)，以及联合文件系统进行了组合，并最终呈现给用户一个可操作和管理的容器引擎

![](/public/img/sample-container-runtime/docker-life.png)

这里为了研究容器技术，我在参考了阿里云三位同学编写的《自己动手写Docker》这本书后，基于[mydocker](https://github.com/xianlubird/mydocker/tree/code-6.5)项目开始编写自己的容器运行态，希望能更加贴近容器本质，并计划补充mydocker没有涉及的OCI，CRI等部分以及一些高级命令

下面我将依次介绍[sample-container-runtime](https://github.com/duyanghao/sample-container-runtime)实现过程中的一些核心细节

## 联合文件系统(UnionFS)

在深入介绍namespace以及cgroups之前，我们先介绍联合文件系统。联合文件系统(UnionFS)用于将不同文件系统的文件和目录联合挂载到同一个文件系统。它使用branch把不同文件系统的文件和目录进行覆盖，形成一个单一一致的文件系统(对于同一路径，上层覆盖下层)视图，这些branch具备不同的读写权限，read-only or read-write；同时利用了写时拷贝(copy on write)技术将对只读层的写操作复制到了读写层。Cow是一种对可修改资源实现高效复制的资源管理技术，当一个资源是重复的且没有发生任何修改时，并不需要创建新的资源，该资源可被多个实例共享；只有当第一次写操作发生时，才会创建新的资源。通过CoW，可以显著减少未修改资源复制带来的消耗，但另一方面也增加了资源修改时的开销

AUFS重写了早期的UnionFS，并引入了一些新的功能，增加了可靠性和性能，同时也是Docker选用的第一个storage driver。sample-container-runtime采用AUFS作为联合文件系统实现，将容器镜像的多层内容呈现为统一的rootfs(根文件系统)

AUFS具备如下特性：

* 上层覆盖下层
* 新增文件存放在可写层
* 写时拷贝(CoW)
* 通过whiteout标识删除文件

![](/public/img/sample-container-runtime/aufs.png)

这里，我们将容器使用的镜像分为三个目录(参考Docker)，如下：

* 只读目录存放容器基础镜像，不可修改(/var/lib/sample-container-runtime/imageXXX)
* 读写目录存放容器运行时修改的内容(/var/lib/sample-container-runtime/writeLayer/containerXXX)
* 挂载目录存放容器aufs联合挂载点(/var/lib/sample-container-runtime/mnt/containerXXX)

```bash
$ make build
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -ti --name container1 -v /root/tmp/from1:/to1 busybox sh
# on node
$ ls /var/lib/sample-container-runtime/  
busybox  busybox.tar  mnt  writeLayer
$ mount|grep aufs
none on /var/lib/sample-container-runtime/mnt/container1 type aufs (rw,relatime,si=b7a28d49e64d71ad)
$ cat /sys/fs/aufs/si_b7a28d49e64d71ad/*
/var/lib/sample-container-runtime/writeLayer/container1=rw
/var/lib/sample-container-runtime/busybox=ro
64
65
/var/lib/sample-container-runtime/writeLayer/container1/.aufs.xino
# container1
/ # mount
none on / type aufs (rw,relatime,si=b7a28d49e87289ad)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,mode=755)
/ # echo "hello, world" > tmpfile
/ # ls
bin      dev      etc      home     proc     root     sys      tmp      tmpfile  usr      var
# switch to node
$ ls -al /var/lib/sample-container-runtime/writeLayer/container1/
drwxr-xr-x 5 root root 4096 Nov  2 19:52 .
drwxr-xr-x 5 root root 4096 Nov  2 19:52 ..
-r--r--r-- 1 root root    0 Nov  2 19:52 .wh..wh.aufs
drwx------ 2 root root 4096 Nov  2 19:52 .wh..wh.orph
drwx------ 2 root root 4096 Nov  2 19:52 .wh..wh.plnk
drwx------ 2 root root 4096 Nov  2 19:52 root
-rw-r--r-- 1 root root   13 Nov  2 19:52 tmpfile
# container1
/ # echo "testline" >> tmp/work_dir_onion/install_exec.sh
# switch to node
$ ls -al /var/lib/sample-container-runtime/writeLayer/container1/
total 36
drwxr-xr-x 8 root root 4096 Nov  2 20:18 .
drwxr-xr-x 5 root root 4096 Nov  2 19:52 ..
-r--r--r-- 1 root root    0 Nov  2 19:52 .wh..wh.aufs
drwx------ 2 root root 4096 Nov  2 19:52 .wh..wh.orph
drwx------ 2 root root 4096 Nov  2 19:52 .wh..wh.plnk
drwxr-xr-x 2 root root 4096 Nov  2 20:03 bin
drwx------ 2 root root 4096 Nov  2 19:52 root
drwxrwxrwt 3 root root 4096 Nov  2 20:18 tmp
-rw-r--r-- 1 root root   13 Nov  2 19:52 tmpfile
drwxr-xr-x 3 root root 4096 Nov  2 20:01 usr
# ls -al /var/lib/sample-container-runtime/writeLayer/container1/
tmp
`-- work_dir_onion
    `-- install_exec.sh
```

从运行可以看到添加文件到aufs mnt，文件实际添加到可写层；另外，修改可读层的文件会复制该文件到可写层，同时只读层该文件并没有修改

这里我们看一下实现：

```go
// Create a AUFS filesystem as container root workspace
func NewWorkSpace(volume, imageName, containerName string) {
	CreateReadOnlyLayer(imageName)
	CreateWriteLayer(containerName)
	CreateMountPoint(containerName, imageName)
	if volume != "" {
		volumeURLs := strings.Split(volume, ":")
		length := len(volumeURLs)
		if length == 2 && volumeURLs[0] != "" && volumeURLs[1] != "" {
			MountVolume(volumeURLs, containerName)
			log.Infof("NewWorkSpace volume urls %q", volumeURLs)
		} else {
			log.Infof("Volume parameter input is not correct.")
		}
	}
}

// Decompression tar image
func CreateReadOnlyLayer(imageName string) error {
	unTarFolderUrl := RootUrl + "/" + imageName + "/"
	imageUrl := RootUrl + "/" + imageName + ".tar"
	exist, err := PathExists(unTarFolderUrl)
	if err != nil {
		log.Infof("Fail to judge whether dir %s exists. %v", unTarFolderUrl, err)
		return err
	}
	if !exist {
		if err := os.MkdirAll(unTarFolderUrl, 0622); err != nil {
			log.Errorf("Mkdir %s error %v", unTarFolderUrl, err)
			return err
		}

		if _, err := exec.Command("tar", "-xvf", imageUrl, "-C", unTarFolderUrl).CombinedOutput(); err != nil {
			log.Errorf("Untar dir %s error %v", unTarFolderUrl, err)
			return err
		}
	}
	return nil
}

// Create read-write layer
func CreateWriteLayer(containerName string) {
	writeURL := fmt.Sprintf(WriteLayerUrl, containerName)
	if err := os.MkdirAll(writeURL, 0777); err != nil {
		log.Infof("Mkdir write layer dir %s error. %v", writeURL, err)
	}
}

// Create aufs mount point
func CreateMountPoint(containerName, imageName string) error {
	mntUrl := fmt.Sprintf(MntUrl, containerName)
	if err := os.MkdirAll(mntUrl, 0777); err != nil {
		log.Errorf("Mkdir mountpoint dir %s error. %v", mntUrl, err)
		return err
	}
	tmpWriteLayer := fmt.Sprintf(WriteLayerUrl, containerName)
	tmpImageLocation := RootUrl + "/" + imageName
	mntURL := fmt.Sprintf(MntUrl, containerName)
	dirs := "dirs=" + tmpWriteLayer + ":" + tmpImageLocation
	_, err := exec.Command("mount", "-t", "aufs", "-o", dirs, "none", mntURL).CombinedOutput()
	if err != nil {
		log.Errorf("Run command for creating mount point failed %v", err)
		return err
	}
	return nil
}
```

核心命令如下：

```bash
mount -t aufs -o dirs=/var/lib/sample-container-runtime/writeLayer/container1:/var/lib/sample-container-runtime/busybox none . /var/lib/sample-container-runtime/mnt/container1
```

上述的aufs联合挂载点作为容器rootfs，这里利用了mount namespace，会在接下来的namespace章节-Mount namespaces介绍

通过上述操作，我们实现了容器重复利用只读层，并构建可写层运行容器的方法，而这实际上也是目前Docker采用的原理。通过联合文件系统可以使Docker镜像最大程度利用磁盘空间，同时也提高了Docker容器启动的效率

## [namespace隔离](https://lwn.net/Articles/531114/)

namespace提供了一种内核级别资源隔离的方法：

>> The purpose of each namespace is to wrap a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. 

Linux目前提供了6种namespace类型，每种namespace用途各不相同：

* Mount namespaces (CLONE_NEWNS, Linux 2.4.19) isolate the set of filesystem mount points seen by a group of processes
* UTS namespaces (CLONE_NEWUTS, Linux 2.6.19) UTS namespaces provide isolation of two system identifiers: the hostname and the NIS domain name.
* IPC namespaces (CLONE_NEWIPC, Linux 2.6.19) isolate certain interprocess communication (IPC) resources, namely, System V IPC objects and (since Linux 2.6.30) POSIX message queues.
* PID namespaces (CLONE_NEWPID, Linux 2.6.24) isolate the process ID number space. In other words, processes in different PID namespaces can have the same PID. 
* Network namespaces (CLONE_NEWNET, started in Linux 2.4.19 2.6.24 and largely completed by about Linux 2.6.29) provide isolation of the system resources associated with networking. Thus, each network namespace has its own network devices, IP addresses, IP routing tables, /proc/net directory, port numbers, and so on.
* User namespaces (CLONE_NEWUSER, started in Linux 2.6.23 and completed in Linux 3.8) isolate the user and group ID number spaces. In other words, a process's user and group IDs can be different inside and outside a user namespace.

下面我将依次介绍各个namespace的应用实现(由易到难)：

### UTS namespaces

UTS namespace实现了进程hostname以及domain name的隔离，它允许我们给容器设置与母机不同的hostname以及domainname。通过给Cloneflags设置CLONE_NEWUTS来实现隔离，并在容器内部使用syscall.Sethostname()函数设置hostname，如下：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}

	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	} else {
		dirURL := fmt.Sprintf(DefaultInfoLocation, containerName)
		if err := os.MkdirAll(dirURL, 0622); err != nil {
			log.Errorf("NewParentProcess mkdir %s error %v", dirURL, err)
			return nil, nil
		}
		stdLogFilePath := dirURL + ContainerLogFile
		stdLogFile, err := os.Create(stdLogFilePath)
		if err != nil {
			log.Errorf("NewParentProcess create file %s error %v", stdLogFilePath, err)
			return nil, nil
		}
		cmd.Stdout = stdLogFile
	}

	cmd.ExtraFiles = []*os.File{readPipe}
	cmd.Env = append(os.Environ(), envSlice...)
	NewWorkSpace(volume, imageName, containerName)
	cmd.Dir = fmt.Sprintf(MntUrl, containerName)
	return cmd, writePipe
}

...
var InitCommand = cli.Command{
	Name:  "init",
	Usage: "Init container process run user's process in container. Do not call it outside",
	Action: func(context *cli.Context) error {
		log.Infof("init come on")
		err := container.RunContainerInitProcess()
		return err
	},
}

func RunContainerInitProcess() error {
	cmdArray := readUserCommand()
	if cmdArray == nil || len(cmdArray) == 0 {
		return fmt.Errorf("Run container get user command error, cmdArray is nil")
	}

	hostname := util.RandomSeq(10)

	if err := syscall.Sethostname([]byte(hostname)); err != nil {
		log.Errorf("set hostname error: %v", err)
		return err
	}
	setUpMount()

	path, err := exec.LookPath(cmdArray[0])
	if err != nil {
		log.Errorf("Exec loop path error %v", err)
		return err
	}
	log.Infof("Find path %s", path)
	if err := syscall.Exec(path, cmdArray[0:], append(os.Environ(), fmt.Sprintf("PS1=%s # ", hostname))); err != nil {
		log.Errorf(err.Error())
	}
	return nil
}
```

运行如下：

```bash
# on container
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -ti --name container1 busybox sh
{"level":"info","msg":"createTty true","time":"2020-11-03T11:20:56+08:00"}
{"level":"info","msg":"init come on","time":"2020-11-03T11:20:56+08:00"}
{"level":"info","msg":"command all is sh","time":"2020-11-03T11:20:56+08:00"}
{"level":"info","msg":"Current location is /var/lib/sample-container-runtime/mnt/container1","time":"2020-11-03T11:20:56+08:00"}
{"level":"info","msg":"Find path /bin/sh","time":"2020-11-03T11:20:56+08:00"}
MbNtIFraOd # hostname
MbNtIFraOd
# on node
$ hostname
VM-xxx-centos
```

可以看到容器中hostname为MbNtIFraOd，而母机为VM-xxx-centos

### IPC namespaces

IPC用于隔离进程某些IPC(进程间通信)资源，具体来说就是：System V IPC objects and (since Linux 2.6.30) POSIX message queues。其中System V IPC objects又包括：Shared Memory(共享内存), Semaphore(信号量) and Message Queues(消息队列)

这里我们通过给Cloneflags设置CLONE_NEWIPC来实现隔离，如下：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}

	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	} else {
		dirURL := fmt.Sprintf(DefaultInfoLocation, containerName)
		if err := os.MkdirAll(dirURL, 0622); err != nil {
			log.Errorf("NewParentProcess mkdir %s error %v", dirURL, err)
			return nil, nil
		}
		stdLogFilePath := dirURL + ContainerLogFile
		stdLogFile, err := os.Create(stdLogFilePath)
		if err != nil {
			log.Errorf("NewParentProcess create file %s error %v", stdLogFilePath, err)
			return nil, nil
		}
		cmd.Stdout = stdLogFile
	}

	cmd.ExtraFiles = []*os.File{readPipe}
	cmd.Env = append(os.Environ(), envSlice...)
	NewWorkSpace(volume, imageName, containerName)
	cmd.Dir = fmt.Sprintf(MntUrl, containerName)
	return cmd, writePipe
}
```

验证如下：

```bash
# inside of container
gyZQRmcHMr # ipcs -a

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      

------ Semaphore Arrays --------
key        semid      owner      perms      nsems     

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
gyZQRmcHMr # readlink /proc/$$/ns/ipc
ipc:[4026532515]

# outside of container
$ ipcmk -Q
Message queue id: 0
$ ipcs -a

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x11df483b 0          root       644        0            0                    

------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x00005feb 0          root       666        12000      3                                            
...                     

------ Semaphore Arrays --------
key        semid      owner      perms      nsems     
0x00008708 0          root       666        1                
...

$ readlink /proc/$$/ns/ipc
ipc:[4026531839]
```

可以看到母机和容器中的IPC资源不同了，两者处于不同的ipc namespace

## [USER namespaces](https://medium.com/@teddyking/namespaces-in-go-user-a54ef9476f2a)

USER namespace用于隔离进程用户ID以及组ID资源，它允许我们设置进程在容器和母机中的用户和组ID映射，也就是说一个进程在容器中可以具有root最高权限，但是在母机上该进程实际上并不具备root用户权限，而只具备普通用户权限

![](/public/img/sample-container-runtime/user_namespace.png)

每个进程通过如下文件路径存储映射关系(inside a USER namespace to a corresponding set of user IDs and group IDs outside the namespace)：

* /proc/PID/uid_map：uid的映射文件
* /proc/PID/gid_map：gid的映射文件

这里，我们通过给[syscall.SysProcAttr](https://golang.org/pkg/syscall/#SysProcAttr)分别设置UidMappings(uid映射)以及GidMappings(gid映射)来实现容器进程USER namespace隔离：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC | syscall.CLONE_NEWUSER,
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getuid(),
				Size:        1,
			},
		},
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,
				HostID:      os.Getgid(),
				Size:        1,
			},
		},
	}

	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	} else {
		dirURL := fmt.Sprintf(DefaultInfoLocation, containerName)
		if err := os.MkdirAll(dirURL, 0622); err != nil {
			log.Errorf("NewParentProcess mkdir %s error %v", dirURL, err)
			return nil, nil
		}
		stdLogFilePath := dirURL + ContainerLogFile
		stdLogFile, err := os.Create(stdLogFilePath)
		if err != nil {
			log.Errorf("NewParentProcess create file %s error %v", stdLogFilePath, err)
			return nil, nil
		}
		cmd.Stdout = stdLogFile
	}

	cmd.ExtraFiles = []*os.File{readPipe}
	cmd.Env = append(os.Environ(), envSlice...)
	NewWorkSpace(volume, imageName, containerName)
	cmd.Dir = fmt.Sprintf(MntUrl, containerName)
	return cmd, writePipe
}
```

这里将容器init进程(1号进程)的uid和gid设置为0(root)，且分别映射为母机当前的用户ID和组ID(非root)

## Mount namespaces

Mount namespace用于隔离进程文件系统的挂载点视图，在不同namespace的进程中，看到的文件系统层次是不一样的，同时，在 Mount Namespace 中调用 mount()和 umount()仅仅只会影响当前Namespace内的文件系统，而对全局的文件系统是没有影响的（Mount Namespace是Linux第一个实现的Namespace类型，因此，它的系统调用参数是NEWNS ( New Namespace的缩写））

这里我将主要探讨如何实现使用mount namespace实现容器挂载联合文件系统作为它的rootfs，这也是上述讲解aufs时遗留的一个问题

通常来说我们需要在容器中按照如下步骤进行挂载：

* remounts current root filesystem with MS_PRIVATE(This mount point is private; it does not have a peer group. Mount and unmount events do not propagate into or out of this mount point.)
* Bind mount newRoot to itself - this is a slight hack needed to satisfy the pivot_root requirement that newRoot and putold must not be on the same filesystem as the current root
* creates temporary directory, where the old root will be stored
* [pivots root (swaps the mount at `/` with another (the `rootfs-dir` in this case).](https://lwn.net/Articles/689856/)
  pivot_root() changes the root directory and the current working directory of each process or thread in the same mount namespace to new_root if they point to the old root directory. (See also NOTES.) On the other hand, pivot_root() does not change the caller's current working directory (unless it is on the old root directory), and thus it should be followed by a chdir("/") call.
  The following restrictions apply:
  - new_root and put_old must be directories.
  - new_root and put_old must not be on the same mount as the current root.
  - put_old must be at or underneath new_root; that is, adding some nonnegative number of "/.." prefixes to the pathname pointed to by put_old must yield the same directory as new_root.
  - new_root must be a path to a mount point, but can't be "/". A path that is not already a mount point can be converted into one by bind mounting the path onto itself.
  - The propagation type of the parent mount of new_root and the parent mount of the current root directory must not be MS_SHARED; similarly, if put_old is an existing mount point, its propagation type must not be MS_SHARED. These restrictions ensure that pivot_root() never propagates any changes to another mount namespace.
  - The current root directory must be a mount point.
* ensures current working directory is set to new root(os.Chdir("/"))
* umounts and removes the old root

另外，需要给Cloneflags设置syscall.CLONE_NEWNS实现mnt namespace隔离。核心代码如下：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}
  
	...
	return cmd, writePipe
}

...
func RunContainerInitProcess() error {
    ...
	setUpMount()
    ...
	return nil
}

/**
Init 挂载点
*/
func setUpMount() {
	pwd, err := os.Getwd()
	if err != nil {
		log.Errorf("Get current location error %v", err)
		return
	}
	log.Infof("Current location is %s", pwd)
	pivotRoot(pwd)
    ...
}

func pivotRoot(root string) error {
	// Remounts current root filesystem with MS_PRIVATE
	if err := syscall.Mount("", "/", "", syscall.MS_REC|syscall.MS_PRIVATE, ""); err != nil {
		return fmt.Errorf("syscall Mount current root failure: %v", err)
	}
	/**
	  为了使当前root的老 root 和新 root 不在同一个文件系统下，我们把root重新mount了一次
	  bind mount是把相同的内容换了一个挂载点的挂载方法
	*/
	if err := syscall.Mount(root, root, "bind", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
		return fmt.Errorf("Mount rootfs to itself error: %v", err)
	}
	// 创建 rootfs/.pivot_root 存储 old_root
	pivotDir := filepath.Join(root, ".pivot_root")
	if err := os.Mkdir(pivotDir, 0777); err != nil {
		return err
	}
	// pivot_root 到新的rootfs, 现在老的 old_root 是挂载在rootfs/.pivot_root
	// 挂载点现在依然可以在mount命令中看到
	if err := syscall.PivotRoot(root, pivotDir); err != nil {
		return fmt.Errorf("pivot_root %v", err)
	}
	// 修改当前的工作目录到根目录
	if err := syscall.Chdir("/"); err != nil {
		return fmt.Errorf("chdir / %v", err)
	}

	pivotDir = filepath.Join("/", ".pivot_root")
	// umount rootfs/.pivot_root
	if err := syscall.Unmount(pivotDir, syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("unmount pivot_root dir %v", err)
	}
	// 删除临时文件夹
	return os.Remove(pivotDir)
}
```

通过上述步骤，我们可以成功做到将容器的rootfs设置为aufs联合文件系统，如下：

```bash
# inside of container
JJMhAjPfRh # mount
none on / type aufs (rw,relatime,si=b7a28d49e33081ad)
JJMhAjPfRh # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
# outside of container
$ cat /sys/fs/aufs/si_b7a28d49e33081ad/*
/var/lib/sample-container-runtime/writeLayer/container1=rw
/var/lib/sample-container-runtime/busybox=ro
64
65
/var/lib/sample-container-runtime/writeLayer/container1/.aufs.xino
```

### PID namespaces

PID namespace用于隔离进程的PID，它会导致容器进程只能看到属于该namespace空间下的进程，同时不同PID namespace下的进程可以拥有相同的PID。当我们通过上述mnt namespace将联合文件系统作为容器的根文件系统后，由于没有/proc目录，我们通过ps命令看到的会是空返回(linux通过/proc目录存储操作系统所有进程的信息)。因此我们必须在运行容器指定进程前设置proc文件系统

具体来说我们需要给Cloneflags设置syscall.CLONE_NEWPID实现PID namespace隔离，同时在容器中设置proc文件系统(mount -t proc proc /proc)，核心代码如下：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}

  ...
	return cmd, writePipe
}

/**
Init 挂载点
*/
func setUpMount() {
	pwd, err := os.Getwd()
	if err != nil {
		log.Errorf("Get current location error %v", err)
		return
	}
	log.Infof("Current location is %s", pwd)
	pivotRoot(pwd)

	// mount proc
	defaultMountFlags := syscall.MS_NOEXEC | syscall.MS_NOSUID | syscall.MS_NODEV
	syscall.Mount("proc", "/proc", "proc", uintptr(defaultMountFlags), "")

	syscall.Mount("tmpfs", "/dev", "tmpfs", syscall.MS_NOSUID|syscall.MS_STRICTATIME, "mode=755")
}
```

运行容器如下：

```bash
JJMhAjPfRh # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 sh
   11 root      0:00 ps -ef
JJMhAjPfRh # mount
none on / type aufs (rw,relatime,si=b7a28d49e33081ad)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
```

可以看到这里多了proc文件系统，同时ps命令可以正常输出，且显示为容器进程PID namespace可以看到的进程(PID=1为容器进程)

另外在容器外，我们可以发现该进程实际对应PID为`19879`，如下：

```bash
xxx     14180 14177  0 Nov02 pts/0    00:00:00 -bash
xxx     19872 14180  0 12:45 pts/0    00:00:00 ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -ti --name container1 busybox sh
xxx     19879 19872  0 12:45 pts/0    00:00:00 sh
```

### Network namespaces

network namespace用于隔离进程网络资源，每个network namespace都有属于它自己的网络设备，IP地址，IP路由表，端口信息，以及/proc/net目录等。通过network namespace可以使每个容器拥有自己独立的(虚拟的)网络设备，而且容器内的应用可以绑定到自己的端口，并且不会产生冲突。通过一定的网络方法就可以实现容器与容器，容器与宿主机，以及容器与容器跨主机通信，而容器网络将会在后续章节进行介绍，这里不展开

下面我们将展示network namespace隔离的最直观现象，如下：

```bash
JJMhAjPfRh # ifconfig
```

可以看到容器中执行ifconfig命令显示为空，表明容器中没有任何网络设备；而宿主机是存在网络设备的，从这一点可以看出网络命令空间确实隔离了，而实现这一现象的操作只需要给Cloneflags设置syscall.CLONE_NEWNET即可：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}

  ...
	return cmd, writePipe
}
```

## cgroups控制

cgroups提供了对一组进程以及子进程资源限制，控制以及统计的能力，包括：CPU，内存，I/O，网络等。通过cgroups，我们可以实现对容器资源的限制和统计，下面介绍cgroups的几个组件：

* cgroup是对进程分组管理的一种机制， 一个 cgroup包含一组进程，井可以在这个cgroup上增加Linux subsystem的各种参数配置，将一组进程和一组subsystem的系统参数关联起来
* subsystem是一组资源控制的模块，包括：blkio(块设备(比如硬盘)输入输出的访问控制)，cpu(进程CPU调度策略)，cpuacct(进程CPU占用)，cpuset(在多核机器上设置 cgroup 中进程可以使用的 CPU 和内存)，memory(进程内存占用)，net_cls(用于将 cgroup 中进程产生的网络包分类)。每个subsystem会关联到定义了相应限制的cgroup上，并对该cgroup中的进程做资源控制
* hierarchy的功能是把一组cgroup串成一个树状的结构，一个这样的树便是一个hierarchy，通过这种树状结构，cgroups可以做到继承

cgroups是通过这三个组件之间相互协作实现的，它们之间的关系可以归纳如下：

* 一个hierarchy可以附加多个subsystem
* 一个subsystem只能附加到一个hierarchy上面 
* 一个进程可以作为多个cgroup的成员，但是这些cgroup必须在不同的hierarchy中
* 一个进程fork出子进程时，子进程是和父进程在同一个cgroup中的，也可以根据需要将其移动到其它cgroup中

Kernel为了使对cgroups的配置更直观，是通过一个虚拟的树状文件系统配置cgroups的，通过层级的目录虚拟出cgroup树。系统默认已经为每个subsystem创建了一个hierarchy，如下：

```bash
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
```

可以看到，/sys/fs/cgroup/memory目录便是挂在了memory subsystem的hierarchy上

接下来将详细讲解`sample-container-runtime`对cgroups的使用细节：

```go
type CgroupManager struct {
	// cgroup在hierarchy中的路径 相当于创建的cgroup目录相对于root cgroup目录的路径
	Path string
	// 资源配置
	Resource *subsystems.ResourceConfig
}
```

这里首先封装了一层CgroupManager，表示对cgroups的管理，同时设置了相应的操作接口：

```go
// 将进程pid加入到这个cgroup中
func (c *CgroupManager) Apply(pid int) error {
	for _, subSysIns := range subsystems.SubsystemsIns {
		subSysIns.Apply(c.Path, pid)
	}
	return nil
}

// 设置cgroup资源限制
func (c *CgroupManager) Set(res *subsystems.ResourceConfig) error {
	for _, subSysIns := range subsystems.SubsystemsIns {
		subSysIns.Set(c.Path, res)
	}
	return nil
}

//释放cgroup
func (c *CgroupManager) Destroy() error {
	for _, subSysIns := range subsystems.SubsystemsIns {
		if err := subSysIns.Remove(c.Path); err != nil {
			log.Warnf("remove cgroup fail %v", err)
		}
	}
	return nil
}
```

上述接口含义如下：

* Apply：调用subsystem.Apply接口将进程PID添加到相应的cgroup tasks文件中(such as /sys/fs/cgroup/memory/ContainerID/tasks)
* Set：调用subsystem.Set接口设置cgroup资源配额(such as /sys/fs/cgroup/memory/ContainerID/memory.limit_in_bytes)
* Destory：调用subsystem.Remove接口删除cgroup(such as /sys/fs/cgroup/memory/ContainerID)

下面展开介绍一下subsystem接口(目前只实现了三种subsystem cgroup)：

```go
package subsystems

type ResourceConfig struct {
	MemoryLimit string
	CpuShare    string
	CpuSet      string
}

type Subsystem interface {
	Name() string
	Set(path string, res *ResourceConfig) error
	Apply(path string, pid int) error
	Remove(path string) error
}

var (
	SubsystemsIns = []Subsystem{
		&CpusetSubSystem{},
		&MemorySubSystem{},
		&CpuSubSystem{},
	}
)
```

这里举例说明memory subsystem实现：

```go
func (s *MemorySubSystem) Set(cgroupPath string, res *ResourceConfig) error {
	if subsysCgroupPath, err := GetCgroupPath(s.Name(), cgroupPath, true); err == nil {
		if res.MemoryLimit != "" {
			if err := ioutil.WriteFile(path.Join(subsysCgroupPath, "memory.limit_in_bytes"), []byte(res.MemoryLimit), 0644); err != nil {
				return fmt.Errorf("set cgroup memory fail %v", err)
			}
		}
		return nil
	} else {
		return err
	}

}

func (s *MemorySubSystem) Remove(cgroupPath string) error {
	if subsysCgroupPath, err := GetCgroupPath(s.Name(), cgroupPath, false); err == nil {
		return os.RemoveAll(subsysCgroupPath)
	} else {
		return err
	}
}

func (s *MemorySubSystem) Apply(cgroupPath string, pid int) error {
	if subsysCgroupPath, err := GetCgroupPath(s.Name(), cgroupPath, false); err == nil {
		if err := ioutil.WriteFile(path.Join(subsysCgroupPath, "tasks"), []byte(strconv.Itoa(pid)), 0644); err != nil {
			return fmt.Errorf("set cgroup proc fail %v", err)
		}
		return nil
	} else {
		return fmt.Errorf("get cgroup %s error: %v", cgroupPath, err)
	}
}

func (s *MemorySubSystem) Name() string {
	return "memory"
}
```

可以看到MemorySubSystem.Set将res.MemoryLimit限制写入到了`/sys/fs/cgroup/memory/ContainerID/memory.limit_in_bytes`文件，对ContainerID命名的cgroup进行了内存限制；同时Apply函数将进程PID写入到`/sys/fs/cgroup/memory/ContainerID/tasks`文件中，使得容器进程被添加到该cgroup中；最后Remove函数删除`/sys/fs/cgroup/memory/ContainerID`目录，也即删除该cgroup。其它subsystem原理依次类推

运行程序如下：

```bash
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -ti -m 100m -cpuset 1 -cpushare 512 -name container1 busybox sh
{"level":"info","msg":"createTty true","time":"2020-11-03T16:34:16+08:00"}
{"level":"info","msg":"init come on","time":"2020-11-03T16:34:16+08:00"}
{"level":"info","msg":"command all is sh","time":"2020-11-03T16:34:16+08:00"}
{"level":"info","msg":"Current location is /var/lib/sample-container-runtime/mnt/container1","time":"2020-11-03T16:34:16+08:00"}
{"level":"info","msg":"Find path /bin/sh","time":"2020-11-03T16:34:16+08:00"}
IkIucFXIzM # 

# outside of container
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
6123430628   container1   23965       running     sh          2020-11-03 16:34:16
$ ls /sys/fs/cgroup/memory/6123430628/
cgroup.clone_children       memory.kmem.max_usage_in_bytes      memory.limit_in_bytes            memory.numa_stat            memory.use_hierarchy
cgroup.event_control        memory.kmem.slabinfo                memory.max_usage_in_bytes        memory.oom_control          notify_on_release
cgroup.procs                memory.kmem.tcp.failcnt             memory.memsw.failcnt             memory.pressure_level       tasks
memory.failcnt              memory.kmem.tcp.limit_in_bytes      memory.memsw.limit_in_bytes      memory.soft_limit_in_bytes
memory.force_empty          memory.kmem.tcp.max_usage_in_bytes  memory.memsw.max_usage_in_bytes  memory.stat
memory.kmem.failcnt         memory.kmem.tcp.usage_in_bytes      memory.memsw.usage_in_bytes      memory.swappiness
memory.kmem.limit_in_bytes  memory.kmem.usage_in_bytes          memory.move_charge_at_immigrate  memory.usage_in_bytes
$ cat /sys/fs/cgroup/memory/6123430628/tasks 
23965
$ cat /sys/fs/cgroup/memory/6123430628/memory.limit_in_bytes 
104857600
$ ps -ef|grep 23958
xxx     23958 14180  0 16:34 pts/0    00:00:00 ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -ti -m 100m -cpuset 1 -cpushare 512 -name container1 busybox sh
xxx     23965 23958  0 16:34 pts/0    00:00:00 sh
```

通过将容器init进程PID添加到对应subsystem的cgroup.tasks文件中，我们就可以实现对容器资源的限制(一个进程fork出子进程时，子进程是和父进程在同一个cgroup中)

## 容器进阶

在介绍完容器的底层核心技术(aufs，namespace隔离以及cgroups资源控制)之后，我们将进行更加高阶的操作，构建实际可用的容器命令行工具。包括：ps(容器列表)，logs(日志查看)，exec(进入容器命名空间)，stop(停止容器)，start(启动容器)，rm(删除容器)，commit(通过容器创建镜像)，env(容器指定环境变量运行)等

### sample-container-runtime ps

sample-container-runtime ps用于查看容器信息列表。这里我们使用/var/run/sample-container-runtime/containerName/config.json文件存储每个容器相关信息，包括：容器ID，容器名称，容器PID(init process PID)，容器运行状态，容器启动命令，容器创建时间等。如下：

```bash
# outside of container
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
6123430628   container1   23965       running     sh          2020-11-03 16:34:16
$ cat /var/run/sample-container-runtime/container1/config.json 
{"pid":"23965","id":"6123430628","name":"container1","command":"sh","createTime":"2020-11-03 16:34:16","status":"running","volume":"","portmapping":null}
```

核心代码实现参考：

```go
type ContainerInfo struct {
	Pid         string   `json:"pid"`         //容器的init进程在宿主机上的 PID
	Id          string   `json:"id"`          //容器Id
	Name        string   `json:"name"`        //容器名
	Command     string   `json:"command"`     //容器内init运行命令
	CreatedTime string   `json:"createTime"`  //创建时间
	Status      string   `json:"status"`      //容器的状态
	Volume      string   `json:"volume"`      //容器的数据卷
	PortMapping []string `json:"portmapping"` //端口映射
}

func recordContainerInfo(containerPID int, commandArray []string, containerName, id, volume string) (string, error) {
	createTime := time.Now().Format("2006-01-02 15:04:05")
	command := strings.Join(commandArray, "")
	containerInfo := &container.ContainerInfo{
		Id:          id,
		Pid:         strconv.Itoa(containerPID),
		Command:     command,
		CreatedTime: createTime,
		Status:      container.RUNNING,
		Name:        containerName,
		Volume:      volume,
	}

	jsonBytes, err := json.Marshal(containerInfo)
	if err != nil {
		log.Errorf("Record container info error %v", err)
		return "", err
	}
	jsonStr := string(jsonBytes)

	dirUrl := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	if err := os.MkdirAll(dirUrl, 0622); err != nil {
		log.Errorf("Mkdir error %s error %v", dirUrl, err)
		return "", err
	}
	fileName := dirUrl + "/" + container.ConfigName
	file, err := os.Create(fileName)
	defer file.Close()
	if err != nil {
		log.Errorf("Create file %s error %v", fileName, err)
		return "", err
	}
	if _, err := file.WriteString(jsonStr); err != nil {
		log.Errorf("File write string error %v", err)
		return "", err
	}

	return containerName, nil
}

func Run(tty bool, comArray []string, res *subsystems.ResourceConfig, containerName, volume, imageName string,
	envSlice []string, nw string, portmapping []string) {
	containerID := randStringBytes(10)
	if containerName == "" {
		containerName = containerID
	}

	parent, writePipe := container.NewParentProcess(tty, containerName, volume, imageName, envSlice)
	if parent == nil {
		log.Errorf("New parent process error")
		return
	}

	if err := parent.Start(); err != nil {
		log.Error(err)
	}

	//record container info
	containerName, err := recordContainerInfo(parent.Process.Pid, comArray, containerName, containerID, volume)
	if err != nil {
		log.Errorf("Record container info error %v", err)
		return
	}

	// use containerID as cgroup name
	cgroupManager := cgroups.NewCgroupManager(containerID)
	defer cgroupManager.Destroy()
	cgroupManager.Set(res)
	cgroupManager.Apply(parent.Process.Pid)

	if nw != "" {
		// config container network
		network.Init()
		containerInfo := &container.ContainerInfo{
			Id:          containerID,
			Pid:         strconv.Itoa(parent.Process.Pid),
			Name:        containerName,
			PortMapping: portmapping,
		}
		if err := network.Connect(nw, containerInfo); err != nil {
			log.Errorf("Error Connect Network %v", err)
			return
		}
	}

	sendInitCommand(comArray, writePipe)

	if tty {
		parent.Wait()
		deleteContainerInfo(containerName)
		container.DeleteWorkSpace(volume, containerName)
	}

}
```

从上述代码可以看出当容器创建时，父进程会设置容器进程的相关信息到指定文件中，而sample-container-runtime ps命令其实也就是简单地读取这些文件(/var/run/sample-container-runtime/containerName/config.json)，并进行排列输出，如下：

```go
var ListCommand = cli.Command{
	Name:  "ps",
	Usage: "list all the containers",
	Action: func(context *cli.Context) error {
		ListContainers()
		return nil
	},
}

func ListContainers() {
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, "")
	dirURL = dirURL[:len(dirURL)-1]
	files, err := ioutil.ReadDir(dirURL)
	if err != nil {
		log.Errorf("Read dir %s error %v", dirURL, err)
		return
	}

	var containers []*container.ContainerInfo
	for _, file := range files {
		if file.Name() == "network" {
			continue
		}
		tmpContainer, err := getContainerInfo(file)
		if err != nil {
			log.Errorf("Get container info error %v", err)
			continue
		}
		containers = append(containers, tmpContainer)
	}

	w := tabwriter.NewWriter(os.Stdout, 12, 1, 3, ' ', 0)
	fmt.Fprint(w, "ID\tNAME\tPID\tSTATUS\tCOMMAND\tCREATED\n")
	for _, item := range containers {
		fmt.Fprintf(w, "%s\t%s\t%s\t%s\t%s\t%s\n",
			item.Id,
			item.Name,
			item.Pid,
			item.Status,
			item.Command,
			item.CreatedTime)
	}
	if err := w.Flush(); err != nil {
		log.Errorf("Flush error %v", err)
		return
	}
}

func getContainerInfo(file os.FileInfo) (*container.ContainerInfo, error) {
	containerName := file.Name()
	configFileDir := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	configFileDir = configFileDir + container.ConfigName
	content, err := ioutil.ReadFile(configFileDir)
	if err != nil {
		log.Errorf("Read file %s error %v", configFileDir, err)
		return nil, err
	}
	var containerInfo container.ContainerInfo
	if err := json.Unmarshal(content, &containerInfo); err != nil {
		log.Errorf("Json unmarshal error %v", err)
		return nil, err
	}

	return &containerInfo, nil
}
```

注意：该实现只是在容器创建时设置了容器的状态为RUNNING，但没有对该容器进行监控，当容器状态发生改变时，通过ps命令查看到的会是错误的状态，这个可以在后续改进

### sample-container-runtime logs

sample-container-runtime logs用于查看容器日志，这里我们将容器的标准输出重定向到文件中(/var/run/sample-container-runtime/containerName/container.log )，在调用logs命令时读取相应容器的container.log即可：

```go
func NewParentProcess(tty bool, containerName, volume, imageName string, envSlice []string) (*exec.Cmd, *os.File) {
	readPipe, writePipe, err := NewPipe()
	if err != nil {
		log.Errorf("New pipe error %v", err)
		return nil, nil
	}
	initCmd, err := os.Readlink("/proc/self/exe")
	if err != nil {
		log.Errorf("get init process error %v", err)
		return nil, nil
	}

	cmd := exec.Command(initCmd, "init")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWPID | syscall.CLONE_NEWNS |
			syscall.CLONE_NEWNET | syscall.CLONE_NEWIPC,
	}

	if tty {
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr
	} else {
		dirURL := fmt.Sprintf(DefaultInfoLocation, containerName)
		if err := os.MkdirAll(dirURL, 0622); err != nil {
			log.Errorf("NewParentProcess mkdir %s error %v", dirURL, err)
			return nil, nil
		}
		stdLogFilePath := dirURL + ContainerLogFile
		stdLogFile, err := os.Create(stdLogFilePath)
		if err != nil {
			log.Errorf("NewParentProcess create file %s error %v", stdLogFilePath, err)
			return nil, nil
		}
		cmd.Stdout = stdLogFile
	}

	cmd.ExtraFiles = []*os.File{readPipe}
	cmd.Env = append(os.Environ(), envSlice...)
	NewWorkSpace(volume, imageName, containerName)
	cmd.Dir = fmt.Sprintf(MntUrl, containerName)
	return cmd, writePipe
}

...
var LogCommand = cli.Command{
	Name:  "logs",
	Usage: "print logs of a container",
	Action: func(context *cli.Context) error {
		if len(context.Args()) < 1 {
			return fmt.Errorf("Please input your container name")
		}
		containerName := context.Args().Get(0)
		logContainer(containerName)
		return nil
	},
}

func logContainer(containerName string) {
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	logFileLocation := dirURL + container.ContainerLogFile
	file, err := os.Open(logFileLocation)
	defer file.Close()
	if err != nil {
		log.Errorf("Log container open file %s error %v", logFileLocation, err)
		return
	}
	content, err := ioutil.ReadAll(file)
	if err != nil {
		log.Errorf("Log container read file %s error %v", logFileLocation, err)
		return
	}
	fmt.Fprint(os.Stdout, string(content))
}
```

执行如下：

```bash
# ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -d -name container4 busybox top  
{"level":"info","msg":"createTty false","time":"2020-11-03T18:01:39+08:00"}
{"level":"info","msg":"command all is top","time":"2020-11-03T18:01:39+08:00"}
# ./build/pkg/cmd/sample-container-runtime/sample-container-runtime logs container4
Mem: 264484K used, 237248K free, 1256K shrd, 19616K buff, 115258K cached
CPU:  0.0% usr  0.0% sys  0.0% nic 99.8% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.05 0.05 0.01 1/303 9
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
```

### sample-container-runtime exec

sample-container-runtime exec可以进入到容器命名空间中。由于对Mount Namespace来说， 一个具有多线程的进程是无法使用 setns调用进入到对应的命名空间的，而Go每启动一个程序就会进入多线程状态，因此无法简简单单地在 Go 里面直接调用系统调用，使当前的进程进入对应的Mount Namespace

这里我们采用借助Cgo来实现这个功能。Cgo允许Go程序以一种特殊的方式调用C函数与标准库，Cgo会将C 源码文件和Go文件整合成一个包，如下是使用C根据指定PID进入其对应的namespace的函数：

```c
package nsenter

/*
#include <errno.h>
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>

__attribute__((constructor)) void enter_namespace(void) {
	char *scr_pid;
	scr_pid = getenv("scr_pid");
	if (scr_pid) {
		//fprintf(stdout, "got scr_pid=%s\n", scr_pid);
	} else {
		//fprintf(stdout, "missing scr_pid env skip nsenter");
		return;
	}
	char *scr_cmd;
	scr_cmd = getenv("scr_cmd");
	if (scr_cmd) {
		//fprintf(stdout, "got scr_cmd=%s\n", scr_cmd);
	} else {
		//fprintf(stdout, "missing scr_cmd env skip nsenter");
		return;
	}
	int i;
	char nspath[1024];
	char *namespaces[] = { "ipc", "uts", "net", "pid", "mnt" };

	for (i=0; i<5; i++) {
		sprintf(nspath, "/proc/%s/ns/%s", scr_pid, namespaces[i]);
		int fd = open(nspath, O_RDONLY);

		if (setns(fd, 0) == -1) {
			//fprintf(stderr, "setns on %s namespace failed: %s\n", namespaces[i], strerror(errno));
		} else {
			//fprintf(stdout, "setns on %s namespace succeeded\n", namespaces[i]);
		}
		close(fd);
	}
	int res = system(scr_cmd);
	exit(0);
	return;
}
*/
import "C"
```

可以看到，这里使用了构造函数，然后导入了C模块，一旦这个包(nsenter)被引入，它就会在所有Go运行的环境启动之前执行，这段程序执行完毕后，Go程序才会执行。这样就避免了Go多线程导致的无法进入mnt Namespace的问题，同时通过在enter_namespace中设置开关避免了sample-container-runtime run启动容器时也会执行该函数，如下：

```c
__attribute__((constructor)) void enter_namespace(void) {
	char *scr_pid;
	scr_pid = getenv("scr_pid");
	if (scr_pid) {
		//fprintf(stdout, "got scr_pid=%s\n", scr_pid);
	} else {
		//fprintf(stdout, "missing scr_pid env skip nsenter");
		return;
	}
	char *scr_cmd;
	scr_cmd = getenv("scr_cmd");
	if (scr_cmd) {
		//fprintf(stdout, "got scr_cmd=%s\n", scr_cmd);
	} else {
		//fprintf(stdout, "missing scr_cmd env skip nsenter");
		return;
	}
  ...
}    
```

setns可以让进程加入已经存在的namespace：int setns(int fd, int nstype)；

* fd参数是一个文件描述符，可通打开namespace文件获取
* 调用这个函数的进程就会被加入到fd所代表的namesapce

通过enter_namespace可以实现成功进入指定PID的5种namespace(ipc, uts, network, pid, mount)，接下来我们再来看一下sample-container-runtime是如何使用enter_namespace的：

```go
var ExecCommand = cli.Command{
	Name:  "exec",
	Usage: "exec a command into container",
	Action: func(context *cli.Context) error {
		//This is for callback
		if os.Getenv(ENV_EXEC_PID) != "" {
			log.Infof("pid callback pid %s", os.Getgid())
			return nil
		}

		if len(context.Args()) < 2 {
			return fmt.Errorf("Missing container name or command")
		}
		containerName := context.Args().Get(0)
		var commandArray []string
		for _, arg := range context.Args().Tail() {
			commandArray = append(commandArray, arg)
		}
		ExecContainer(containerName, commandArray)
		return nil
	},
}

const ENV_EXEC_PID = "scr_pid"
const ENV_EXEC_CMD = "scr_cmd"

func ExecContainer(containerName string, comArray []string) {
	pid, err := GetContainerPidByName(containerName)
	if err != nil {
		log.Errorf("Exec container getContainerPidByName %s error %v", containerName, err)
		return
	}

	cmdStr := strings.Join(comArray, " ")
	log.Infof("container pid %s", pid)
	log.Infof("command %s", cmdStr)

	cmd := exec.Command("/proc/self/exe", "exec")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	os.Setenv(ENV_EXEC_PID, pid)
	os.Setenv(ENV_EXEC_CMD, cmdStr)
	containerEnvs := getEnvsByPid(pid)
	cmd.Env = append(os.Environ(), containerEnvs...)

	if err := cmd.Run(); err != nil {
		log.Errorf("Exec container %s error %v", containerName, err)
	}
}

func GetContainerPidByName(containerName string) (string, error) {
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	configFilePath := dirURL + container.ConfigName
	contentBytes, err := ioutil.ReadFile(configFilePath)
	if err != nil {
		return "", err
	}
	var containerInfo container.ContainerInfo
	if err := json.Unmarshal(contentBytes, &containerInfo); err != nil {
		return "", err
	}
	return containerInfo.Pid, nil
}

func getEnvsByPid(pid string) []string {
	path := fmt.Sprintf("/proc/%s/environ", pid)
	contentBytes, err := ioutil.ReadFile(path)
	if err != nil {
		log.Errorf("Read file %s error %v", path, err)
		return nil
	}
	//env split by \u0000
	envs := strings.Split(string(contentBytes), "\u0000")
	return envs
}
```

通过如上代码，我们知道当执行exec命令时，会执行ExecContainer函数，而ExecContainer会先设置子进程的ENV_EXEC_PID(容器PID)以及ENV_EXEC_CMD(exec执行的shell命令)环境变量，之后clone出子进程(/proc/self/exe)，并一直等待

子进程(/proc/self/exe)会先执行enter_namespace函数，进入到容器namespace，然后执行system函数，该函数会fork出一个子进程(孙)，并在该子进程(孙)中执行shell命令(**The system() library function uses fork(2) to create a child process that executes the shell command specified in command using execl(3) as follows: execl("/bin/sh", "sh", "-c", command, (char *) NULL);**)，在子进程(孙)执行完shell命令后，子进程(/proc/self/exe)会回到ExecCommand，并由于ENV_EXEC_PID设置并不为空而直接退出，如下：

```go
var ExecCommand = cli.Command{
	Name:  "exec",
	Usage: "exec a command into container",
	Action: func(context *cli.Context) error {
		//This is for callback
		if os.Getenv(ENV_EXEC_PID) != "" {
			log.Infof("pid callback pid %s", os.Getgid())
			return nil
		}

		if len(context.Args()) < 2 {
			return fmt.Errorf("Missing container name or command")
		}
		containerName := context.Args().Get(0)
		var commandArray []string
		for _, arg := range context.Args().Tail() {
			commandArray = append(commandArray, arg)
		}
		ExecContainer(containerName, commandArray)
		return nil
	},
}
```

通过如上的调用流程，很巧妙地解决了在不影响启动容器功能前提下，如何在Go运行的环境启动之前进入到容器命名空间的问题，如下是执行结果：

```bash
# outside of container
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
2845654752   container4   5571        running     top         2020-11-03 18:01:39
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime exec container4 sh
{"level":"info","msg":"container pid 5571","time":"2020-11-03T19:05:53+08:00"}
{"level":"info","msg":"command sh","time":"2020-11-03T19:05:53+08:00"}
OHdBBchsCx # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 top
   12 root      0:00 sh
   13 root      0:00 ps -ef
$ pstree 8812
bash---sample-containe-+-exe---sh
                       `-5*[{sample-containe}]   
# ps -ef|grep 8812
xxx      8812 14177  0 11:33 pts/1    00:00:00 -bash
xxx     15555  8812  0 19:05 pts/1    00:00:00 ./build/pkg/cmd/sample-container-runtime/sample-container-runtime exec container4 sh
xxx     15561 15555  0 19:05 pts/1    00:00:00 /proc/self/exe exec
xxx     15562 15561  0 19:05 pts/1    00:00:00 sh
```

可以看到调用链：bash---sample-containe-+-exe---sh。在容器中看到了top以及sh进程，说明exec成功进入到了容器命名空间

这里有一个小疑问：为什么子进程(proc/self/exec)没有显示出来？

答案就是：**CLONE_NEWPID 和其他 namespace 不同，把进程加入到PID namespace 并不会修改该进程的 PID namespace，而只修改它所有子进程的 PID namespace**

```bash
# PID namespace diffs between proc/self/exec and sh
$ readlink /proc/15555/ns/pid
pid:[4026531836]
$ readlink /proc/15561/ns/pid 
pid:[4026531836]
$ readlink /proc/15562/ns/pid
pid:[4026532322]
# other namespaces are same as below
$ readlink /proc/15555/ns/ipc
ipc:[4026531839]
$ readlink /proc/15561/ns/ipc
ipc:[4026532321]
$ readlink /proc/15562/ns/ipc
ipc:[4026532321]
```

### sample-container-runtime stop

stop命令用于停止容器执行，具体来说就是向容器主进程(init process)发送SIGKILL信号，实现流程如下：

* 获取容器PID
* 对该PID发送kill信号(SIGKILL)
* 修改容器相关信息(STATUS&PID)
* 将信息重新写入相应文件(/var/run/sample-container-runtime/containerName/config.json)

核心代码如下：

```go
var StopCommand = cli.Command{
	Name:  "stop",
	Usage: "stop a container",
	Action: func(context *cli.Context) error {
		if len(context.Args()) < 1 {
			return fmt.Errorf("Missing container name")
		}
		containerName := context.Args().Get(0)
		stopContainer(containerName)
		return nil
	},
}

func stopContainer(containerName string) {
	pid, err := GetContainerPidByName(containerName)
	if err != nil {
		log.Errorf("Get contaienr pid by name %s error %v", containerName, err)
		return
	}
	pidInt, err := strconv.Atoi(pid)
	if err != nil {
		log.Errorf("Conver pid from string to int error %v", err)
		return
	}
	if err := syscall.Kill(pidInt, syscall.SIGKILL); err != nil {
		log.Errorf("Stop container %s error %v", containerName, err)
		return
	}
	containerInfo, err := getContainerInfoByName(containerName)
	if err != nil {
		log.Errorf("Get container %s info error %v", containerName, err)
		return
	}
	containerInfo.Status = container.STOP
	containerInfo.Pid = " "
	newContentBytes, err := json.Marshal(containerInfo)
	if err != nil {
		log.Errorf("Json marshal %s error %v", containerName, err)
		return
	}
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	configFilePath := dirURL + container.ConfigName
	if err := ioutil.WriteFile(configFilePath, newContentBytes, 0622); err != nil {
		log.Errorf("Write file %s error", configFilePath, err)
	}
}

func getContainerInfoByName(containerName string) (*container.ContainerInfo, error) {
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	configFilePath := dirURL + container.ConfigName
	contentBytes, err := ioutil.ReadFile(configFilePath)
	if err != nil {
		log.Errorf("Read file %s error %v", configFilePath, err)
		return nil, err
	}
	var containerInfo container.ContainerInfo
	if err := json.Unmarshal(contentBytes, &containerInfo); err != nil {
		log.Errorf("GetContainerInfoByName unmarshal error %v", err)
		return nil, err
	}
	return &containerInfo, nil
}
```

运行如下：

```bash
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -d -name container5 busybox top
{"level":"info","msg":"createTty false","time":"2020-11-03T20:10:40+08:00"}
{"level":"info","msg":"command all is top","time":"2020-11-03T20:10:40+08:00"}
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
2845654752   container4   5571        running     top         2020-11-03 18:01:39
7060111258   container5   26338       running     top         2020-11-03 20:10:40
$ ps -ef|grep 26338
xxx     26338     1  0 20:10 pts/0    00:00:00 top
xxx     26451  8812  0 20:11 pts/1    00:00:00 grep --color=auto 26338
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime stop container5
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
2845654752   container4   5571        running     top         2020-11-03 18:01:39
7060111258   container5               stopped     top         2020-11-03 20:10:40
# ps -ef|grep 26338
xxx     26634  8812  0 20:12 pts/1    00:00:00 grep --color=auto 26338
```

**注意：如果容器主进程(init process)存在子进程，那么子进程也会一起被kill掉**

### sample-container-runtime start

### sample-container-runtime rm

rm命令用于删除已经停止的容器，删除内容包括：容器存储信息，容器日志，容器aufs文件系统以及容器cgroup。步骤如下：

* 根据容器名查找容器信息(getContainerInfoByName)
* 判断容器是否处于停止状态(containerInfo.Status)
* 删除容器云数据存储目录(容器信息+日志)
* 删除容器对应的aufs文件系统(包括读写层，不包括只读层)
* 删除容器对应的各subsystem cgroup

核心代码如下：

```go
var RemoveCommand = cli.Command{
	Name:  "rm",
	Usage: "remove unused containers",
	Action: func(context *cli.Context) error {
		if len(context.Args()) < 1 {
			return fmt.Errorf("Missing container name")
		}
		containerName := context.Args().Get(0)
		removeContainer(containerName)
		return nil
	},
}

func removeContainer(containerName string) {
	containerInfo, err := getContainerInfoByName(containerName)
	if err != nil {
		log.Errorf("Get container %s info error %v", containerName, err)
		return
	}
	if containerInfo.Status != container.STOP {
		log.Errorf("Couldn't remove running container")
		return
	}
	dirURL := fmt.Sprintf(container.DefaultInfoLocation, containerName)
	if err := os.RemoveAll(dirURL); err != nil {
		log.Errorf("Remove file %s error %v", dirURL, err)
		return
	}
	container.DeleteWorkSpace(containerInfo.Volume, containerName)
	// clear cgroup
	cgroupManager := cgroups.NewCgroupManager(containerInfo.Id)
	cgroupManager.Destroy()
}

// Delete the AUFS filesystem while container exit
func DeleteWorkSpace(volume, containerName string) {
	if volume != "" {
		volumeURLs := strings.Split(volume, ":")
		length := len(volumeURLs)
		if length == 2 && volumeURLs[0] != "" && volumeURLs[1] != "" {
			DeleteMountPointWithVolume(volumeURLs, containerName)
		} else {
			DeleteMountPoint(containerName)
		}
	} else {
		DeleteMountPoint(containerName)
	}
	DeleteWriteLayer(containerName)
}

func DeleteMountPoint(containerName string) error {
	mntURL := fmt.Sprintf(MntUrl, containerName)
	_, err := exec.Command("umount", mntURL).CombinedOutput()
	if err != nil {
		log.Errorf("Unmount %s error %v", mntURL, err)
		return err
	}
	if err := os.RemoveAll(mntURL); err != nil {
		log.Errorf("Remove mountpoint dir %s error %v", mntURL, err)
		return err
	}
	return nil
}

func DeleteWriteLayer(containerName string) {
	writeURL := fmt.Sprintf(WriteLayerUrl, containerName)
	if err := os.RemoveAll(writeURL); err != nil {
		log.Infof("Remove writeLayer dir %s error %v", writeURL, err)
	}
}
```

运行如下：

```bash
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime run -d -m 100m -cpuset 1 -cpushare 512 -name container6 busybox top
{"level":"info","msg":"createTty false","time":"2020-11-03T20:38:00+08:00"}
{"level":"info","msg":"command all is top","time":"2020-11-03T20:38:00+08:00"}
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
9373732263   container6   31067       running     top         2020-11-03 20:38:00
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime stop container6
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
9373732263   container6               stopped     top         2020-11-03 20:38:00
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime rm container6
$ ./build/pkg/cmd/sample-container-runtime/sample-container-runtime ps
ID           NAME         PID         STATUS      COMMAND     CREATED
empty
```

### sample-container-runtime commit



### sample-container-runtime env

## 容器网络

## Roadmap

## Conclusion

## Refs

* [sample-container-runtime](https://github.com/duyanghao/sample-container-runtime)