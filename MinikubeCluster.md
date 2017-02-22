# 基于minikube的kubernetes集群部署Vitess最佳实践

## 简介
[minikube](https://github.com/kubernetes/minikube/blob/v0.16.0/README.md)是一个可以很容易在本地运行Kubernetes集群的工具,
minikube在电脑上的虚拟机内运行单节点Kubernetes集群，可以很方便的供Kubernetes日常开发使用；minikube在Linux下是部署需要依赖[VirtualBox](https://www.virtualbox.org/wiki/Downloads)或者[KVM](http://www.linux-kvm.org/)，本文中所说的是基于KVM驱动搭建单机集群环境。  
minikube运行的大致结构如下：  
![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/minikube_struct.png)


## minikube环境搭建
### minikube安装
Minikube使用go语言编写，发布形式是一个独立的二进制文件，因此只需要下载，然后放到对应的位置即可正常使用。
``` sh
# 下载minikube-linux-amd64文件
$ wget https://storage.googleapis.com/minikube/releases/v0.16.0/minikube-linux-amd64

# 增加可执行权限
$ chmod +x minikube-linux-amd64

# 将可执行文件移动到/usr/local/bin
$ mv minikube-linux-amd64 /usr/local/bin/minikube

# 查看版本确认是否安装成功
$ minikube version
# minikube version: v0.16.0
```
### kubectl安装
kubectl同样是go语言编写， 发布形式是一个独立的二进制文件，我们只需要下载该文件就可以正常使用。
``` sh
# 下载kubectl文件
$ wget http://storage.googleapis.com/kubernetes-release/release/v1.3.0/bin/linux/amd64/kubectl

# 增加可执行权限
$ chmod +x kubectl

# 移动文件到/usr/local/bin下
$ mv kubectl /usr/local/bin

# 查看版本确认安装成功
$ kubectl version
```

### docker-machine-driver-kvm安装
docker-machine-driver-kvm是提供在kvm虚拟机上安装docker的驱动程序，二进制发布，可以直接下载使用。
``` sh
# 下载驱动
$ wget https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.7.0/docker-machine-driver-kvm

# 增加可执行权限
$ chmod +x docker-machine-driver-kvm

# 移动文件到/usr/local/bin
$ mv docker-machine-driver-kvm /usr/local/bin
```

### kvm驱动安装
安装kvm驱动主要是为了在本机运行kvm虚拟机， kvm驱动需要根据自己系统到官网下载对应的驱动进行安装。
minikube官方对kvm驱动的安装说明请参考： [kvm驱动](https://github.com/kubernetes/minikube/blob/v0.16.0/DRIVERS.md#kvm-driver)   
以下是我个人整理的需要安装的内容：  
``` sh
# centos 系统
# 安装驱动
$ yum install libvirt-daemon-kvm kvm

# 安装驱动相关工具
$ yum install libguestfs libguestfs-tools libvirt


# ubuntu 系统
# 安装驱动和相应的工具
$ sudo apt install libvirt-bin qemu-kvm
```

### 启动kvm相关服务
kvm安装好后需要启动相应的服务才能保证虚拟机正常启动使用

``` sh
$ libvirtd -d
$ systemctl start virtlogd.socket
```

### 启动minikube
通过上面的安装下面我们就可以正常启动minikube了，由于minikube启动参数比较多，以下我们只列出两条，简单说明下，需要详细了解请自行阅读minikube帮助文档，帮助信息可以通过以下命令查看：
``` sh
$ minikube -h
```

正常启动命令
``` sh
# 正常启动minikube服务，指定驱动；不开启日志
# --vm 参数指定了需要使用的驱动程序， linux下默认使用的是virtualbox程序来启动， 由于virtualbox操作安装问题较多，所以这里选用了kvm
# 可以借助kvm强大的命令行工具集合来操作, 方便快捷
$ minikube start --vm-driver=kvm
```

开启日志启动命令
``` sh
# 启动minikube并且开启日志， --v参数是可以指定minikube的日志级别
# --v=0 INFO level logs
# --v=1 WARNING level logs
# --v=2 ERROR level logs
# --v=3 libmachine logging
# --v=7 libmachine --debug level logging
$ minikube start --v=7 --vm-driver=kvm
```

以上只说明了--v和--vm参数的使用说明，由于minikube涉及的参数比较多，使用的时候可以根据自己需要查看帮助文档，或者参考官方说明。本文只列出了linux下的部署说明，windows和mac系统下请参阅[官方说明](https://github.com/kubernetes/minikube/blob/v0.16.0/README.md)。

经过上面的步骤我就可以正常使用minikube和kubectl来管理操作我们的集群了。


## 集群操作常用命令
### kubectl相关
* 关键词概念  
[Pods](https://www.kubernetes.org.cn/kubernetes-pod)  
[Labels](https://www.kubernetes.org.cn/kubernetes-labels)  
[Replication Controller](https://www.kubernetes.org.cn/replication-controller-kubernetes)  
[Services](https://www.kubernetes.org.cn/kubernetes-services)  
[Volumes](https://www.kubernetes.org.cn/kubernetes-volumes)  
[kubectl命令详细说明](https://www.kubernetes.org.cn/doc-45)  

* 获取pod列表  
``` sh
# 命令会返回当前kubernetes 已经创建的pods列表，主要会显示以下信息
# NAME                    READY     STATUS    RESTARTS   AGE
$ kubectl get pod
# NAME                    READY     STATUS    RESTARTS   AGE
# etcd-global-9002d       1/1       Running   0          2d
# etcd-global-l3ph8       1/1       Running   0          2d
# etcd-global-psj52       1/1       Running   0          2d
```

* 查看pod详细信息  
``` sh
# 使用pod名称查看pod的详细信息, 主要是容器的详细信息
$ kubectl describe pod etcd-global-9002d
```  
* 查询部署列表  
``` sh
# 获取部署列表
$ kubectl get deployment
```
* 删除部署
``` sh
# 删除名称为etcd-minikube的部署
$ kubectl delete deployment etcd-minikube
```

### 容器相关
* 拉取容器镜像  
``` sh
# 拉取远端名称为test的镜像
$ docker pull test
# docker pull vitess/etcd:v2.0.13-lite
# docker pull vitess/lite
```
* 查看容器列表  
``` sh
# 查看当前启动的容器列表
$ docker ps

# 返回以下信息
# CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
```
* 登录容器  
``` sh
# 通过容器ID登录容器
$ docker exec -it 容器ID /bin/bash　
# docker exec -it 66f92ed4befb /bin/bash
```
* 保存容器镜像  
``` sh
# 保存已经下载下来的容器到文件，xxx是镜像名称(REPOSITORY)　
$ docker save -o xxx.tar xxx  
```

* 加载镜像  
``` sh
# 加载导出的镜像文件
$ docker load --input xxx.tar
```
如果有多个镜像文件，可以使用脚本进行批量导入
``` sh
$ ls -l | awk -F ' ' '{print "docker load --input="$NF}' | sh
```

### kvm相关
kvm命令也很多，下面介绍部分命令，详细的命令信息可以参见virsh -h
* 启动虚拟机  
``` sh
### 启动已经创建的虚拟机xxxx
$ virsh start xxxx
```

* 暂停虚拟机
``` sh
#  暂停正在运行的虚拟机xxx
$ virsh suspend xxxx
```

* 设置虚拟机内存
``` sh
# 修改内存
$ virsh setmem xxxxx 512000
```
* 恢复挂起(暂停)的虚拟机
``` sh
$ virsh resume xxxx
```
* 修改虚拟机配置文件
  上面所说的修改内存还有一种方法是可以直接修改运行中的虚拟机的配置文件，以达到修改对应参数的效果，修改配置文件相对于其他命令来说比较好用，kvm虚拟机配置都是以xml格式配置的，我们可以使用virsh edit直接修改。
  ``` sh
  # 使用如下命令就会显示配置文件编辑窗口，对应的ｘｍｌ文件记录了虚拟机的各种参数，　修改完成重启虚拟机即可生效
  $ virsh edit xxxx
  ```

* 其他
  在使用minikube通过kvm创建虚拟机的时候，文件virbr1.status记录着对应的ip信息，如果出现ip冲突可以修改以下文件进行处理，保证以下文件只有一个唯一的ip即可。
``` sh
$ vim /var/lib/libvirt/dnsmasq/virbr1.status
```

## Vitess部署

  本文假定用户已经具备[本地部署Vitess](http://vitess.io/getting-started/local-instance.html)的经验，需要将Vitess部署在Kubernetes，所以对于相关的环境依赖就不在做过多的说明；　如果有不明白的地方请先参阅官方文档。

### 编译安装vtctlclient

  `vtctlclient`工具可以用来向Vitess发送命令，　所以安全环境前我们需要先安装`vtctlclient`。

  ``` sh
  $ go get github.com/youtube/vitess/go/cmd/vtctlclient
  ```
  该命令会在$GOPATH/src/github.com/youtube/vitess/目录下载并且编译`vitess`源码，　同时也会把编译好的vtctlclient二进制文件拷贝到目录$GOPATH/bin下。

### 本地kubectl

    如果正常按照文档说明本地`kubectl`已经安装完成，这里我们需要再次校验一下，确保`kubectl`处于正常可用状态。
    检查`kubectl`是否已经正常安装并设置环境变量PATH:
    ``` sh
    $ which kubectl
    ### example output:
    # /usr/local/bin/kubectl
    ```
    如果kubectl没有包含在$PATH环境变量中，　就需要设置`KUBECTL`环境变量，否则执行启动脚本的时候无法获取`kubectl`位置。

    ``` sh
    $ export KUBECTL=/export/working/bin/kubectl
    ```
### 启动Vitess集群
１. 跳转到本地Vitess源代码目录

    经过上面的步骤后，我们就可以尝试着运行Vitess官方提供的实例了，　切换到$GOPATH/src/github.com/youtube/vitess/examples/kubernetes目录下：
    ``` sh
    $ cd $GOPATH/src/github.com/youtube/vitess/examples/kubernetes
    ```
2. 修改本地配置

    运行configure.sh脚本来生成config.sh文件，config.sh用于自定义的集群设置。对于备份官方支持两种方式file和gcs方式，我们这里使用file方式创建备份。
    ``` sh
    vitess/examples/kubernetes$ ./configure.sh
    ### example output:
    # Vitess Docker image (leave empty for default) []:
    # Backup Storage (file, gcs) [gcs]: file
    # Root directory for backups (usually an NFS mount): /backup
    # NOTE: You must add your NFS mount to the vtctld-controller-template
    #  and vttablet-pod-template as described in the Kubernetes docs:
    #  http://kubernetes.io/v1.0/docs/user-guide/volumes.html#nfs
    ```
    注意：　对于使用file方式备份的我们需要在vttablet和vtctld pod中安装一个读写网络卷，　可以通过NFS(Network File System)将任何存
    储服务mount到Kubernetes中；这样我们就可以很方便的备份了。

3. 启动etcd集群

    Vitess[拓扑服务](http://vitess.io/overview/concepts.html#topology-service)存储Vitess集群中所有服务器的协作元数据， 他将此数据存储在支持数据一致性存储系统中。本例中我们使用[etcd](https://github.com/coreos/etcd)来存储，注意，我们需要自己的etcd集群，与Kubernetes本身使用的集群分开。
    ``` sh
    vitess/examples/kubernetes$ ./etcd-up.sh
    ### example output:
    # Creating etcd service for global cell...
    # service "etcd-global" created
    # service "etcd-global-srv" created
    # Creating etcd replicationcontroller for global cell...
    # replicationcontroller "etcd-global" created
    # ...
    ```

    这条命令创建了两个集群， 一个是[全局数据中心](/user-guide/topology-service.html#global-vs-local)集群，另一个是[本地数据中心](http://vitess.io/overview/concepts.html#cell-data-center)集群。你可以通过运行以下命令来检查群集中[pods](http://kubernetes.io/v1.1/docs/user-guide/pods.html)的状态:

    ``` sh
    $ kubectl get pods
    ### example output:
    # NAME                READY     STATUS    RESTARTS   AGE
    # etcd-global-8oxzm   1/1       Running   0          1m
    # etcd-global-hcxl6   1/1       Running   0          1m
    # etcd-global-xupzu   1/1       Running   0          1m
    # etcd-test-e2y6o     1/1       Running   0          1m
    # etcd-test-m6wse     1/1       Running   0          1m
    # etcd-test-qajdj     1/1       Running   0          1m
    ```

    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/etcd_pods.png)

    Kubernetes节点第一次下载需要的Docker镜像的时候会耗费较长的时间， 在下载镜像的过程中Pod的状态是Pending状态。

    **注意:** 本例中, 每个以`-up.sh`结尾的脚本都有一个以`-down.sh`结尾的脚本相对应。 你可以用来停止Vitess集群中的某些组件，而不会关闭整个集群；例如：移除`etcd`的部署可以使用一下命令：
    ``` sh
    vitess/examples/kubernetes$ ./etcd-down.sh
    ```

4.  **启动vtctld**
    `vtctld`提供了检查Vitess集群状态的相关接口， 同时还可以接收`vtctlclient`的RPC命令来修改集群信息。

    ``` sh
    vitess/examples/kubernetes$ ./vtctld-up.sh
    ### example output:
    # Creating vtctld ClusterIP service...
    # service "vtctld" created
    # Creating vtctld replicationcontroller...
    # replicationcontroller "vtctld" create createdd
    ```

5.  **使用vtctld web界面**

    在Kubernetes外面使用vtctld需要使用[kubectl proxy]
    (http://kubernetes.io/v1.1/docs/user-guide/kubectl/kubectl_proxy.html)在工作站上创建一个通道。

    **注意:** proxy命令是运行在前台， 所以如果你想启动proxy需要另外开启一个终端。

    ``` sh
    $ kubectl proxy --port=8001
    ### example output:
    # Starting to serve on localhost:8001
    ```

    你可以在`本地`打开vtctld web界面:

    http://localhost:8001/api/v1/proxy/namespaces/default/services/vtctld:web/

    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/vtctld_web.png)

    同时，还可以通过proxy进入[Kubernetes Dashboard]
    (http://kubernetes.io/v1.1/docs/user-guide/ui.html), 监控nodes, pods和服务器状态:

    http://localhost:8001/ui

    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/Kubernetes_ui.png)

6. 


## 其他
  [Kubernetes中文文档](https://www.kubernetes.org.cn/k8s)  
