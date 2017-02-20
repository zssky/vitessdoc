# 基于minikube的kubernetes集群搭建

## 简介
[minikube](https://github.com/kubernetes/minikube/blob/v0.16.0/README.md)是一个可以很容易本地运行Kubernetes集群的工具,
minikube在电脑上的虚拟机内运行单节点Kubernetes集群，可以很方便的供Kubernetes日常开发使用；minikube在Linux下是会用需要依赖[VirtualBox](https://www.virtualbox.org/wiki/Downloads)或者[KVM](http://www.linux-kvm.org/)，本文中所说的是基于KVM驱动搭建单机集群环境。

## 环境搭建
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
# 可以借助kvm强大的命令行工具集合来操作方便快捷
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
1. 关键词概念  
[Pods](https://www.kubernetes.org.cn/kubernetes-pod)  
[Labels](https://www.kubernetes.org.cn/kubernetes-labels)  
[Replication Controller](https://www.kubernetes.org.cn/replication-controller-kubernetes)  
[Services](https://www.kubernetes.org.cn/kubernetes-services)  
[Volumes](https://www.kubernetes.org.cn/kubernetes-volumes)  
[kubectl命令详细说明](https://www.kubernetes.org.cn/doc-45)  

２. 获取pod列表  
``` sh
# 命令会返回当前kubernetes 已经创建的pods列表，主要会显示以下信息
# NAME                    READY     STATUS    RESTARTS   AGE
$ kubectl get pod
# NAME                    READY     STATUS    RESTARTS   AGE
# etcd-global-9002d       1/1       Running   0          2d
# etcd-global-l3ph8       1/1       Running   0          2d
# etcd-global-psj52       1/1       Running   0          2d
```

３. 查看pod详细信息  
``` sh
# 使用pod名称查看pod的详细信息, 主要是容器的详细信息
$ kubectl describe pod etcd-global-9002d
```  
４. 查询部署列表  
``` sh
# 获取部署列表
$ kubectl get deployment
```
5. 删除部署
``` sh
# 删除名称为etcd-minikube的部署
$ kubectl delete deployment etcd-minikube
```

### 容器相关
1. 拉取容器镜像  
``` sh
# 拉取远端名称为test的镜像
$ docker pull test
# docker pull vitess/etcd:v2.0.13-lite
# docker pull vitess/lite
```
2. 查看容器列表  
``` sh
# 查看当前启动的容器列表
$ docker ps

# 返回以下信息
# CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
```
3. 登录容器  
``` sh
# 通过容器ID登录容器
$ docker exec -it 容器ID /bin/bash　
# docker exec -it 66f92ed4befb /bin/bash
```
4. 保存容器镜像  
``` sh
# 保存已经下载下来的容器到文件，xxx是镜像名称(REPOSITORY)　
$ docker save -o xxx.tar xxx  
```

5. 加载镜像  
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
1. 启动虚拟机  
``` sh
### 启动已经创建的虚拟机xxxx
$ virsh start xxxx
```

2. 暂停虚拟机
``` sh
#  暂停正在运行的虚拟机xxx
$ virsh suspend xxxx
```

3. 设置虚拟机内存
``` sh
# 修改内存
$ virsh setmem xxxxx 512000
```
4. 恢复挂起(暂停)的虚拟机
``` sh
$ virsh resume xxxx
```
5. 修改虚拟机配置文件
  上面所说的修改内存还有一种方法是可以直接修改运行中的虚拟机的配置文件，以达到修改对应参数的效果，修改配置文件相对于其他命令来说比较好用，kvm虚拟机配置都是以xml格式配置的，我们可以使用virsh edit直接修改。
  ``` sh
  # 使用如下命令就会显示配置文件编辑窗口，对应的ｘｍｌ文件记录了虚拟机的各种参数，　修改完成重启虚拟机即可生效
  $ virsh edit xxxx
  ```

6. 其他
  在使用minikube通过kvm创建虚拟机的时候，文件virbr1.status记录着对应的ip信息，如果出现ip冲突可以修改以下文件进行处理，保证以下文件只有一个唯一的ip即可。
``` sh
$ vim /var/lib/libvirt/dnsmasq/virbr1.status
```
