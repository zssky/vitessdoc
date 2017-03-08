# 基于kubernetes集群的Vitess最佳实践

## 概要
  本文主要说明基于kubernetes集群部署并使用Vitess; 本文假定用户已经具备了kubernetes集群使用环境，如果不具备请先参阅[基于minikube的kubernetes集群搭建](https://github.com/zssky/vitessdoc/blob/master/MinikubeCluster.md), 或者参阅[Kubernetes 官方文档](https://kubernetes.io/docs/)搭建正式的集群环境。

  [Kubernetes中文文档](https://www.kubernetes.org.cn/k8s)

  以下就先简要介绍下基本操作命令：

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

* 删除容器
  ``` sh
  # 删除rc，即删除该rc控制的所有容器
  $ kubectl delete rc my-nginx

  # 删除svc，即删除分配的虚拟IP
  $ kubectl delete svc my-ngin
  ```

* 获取Replication Controller
  ``` sh
  # 获取Replication Controller列表
  $ kubectl get rc
  ```

* 通过外部访问kubectl内部的端口
  ``` sh
  # expose命令将会创建一个service，将本地（某个节点上）的一个随机端口关联到容器中的80端口
  $ kubectl expose rc my-nginx --port=80 --type=LoadBalancer
  ```

* 查询服务信息
  ``` sh
  # 以上通过expose创建了一个叫my-nginx的service，我们可以通过以下命令查询服务信息
  $ kubectl get svc my-nginx
  ```

* 根据配置文件创建pod
  ``` sh
  # 根据配置文件*.yaml创建容器
  $ kubectl create -f ./hello-world.yaml
  ```

* 配置文件正确性校验
  ``` sh
  # 使用--vaildate参数可以校验配置文件正确性
  $ kubectl create -f ./hello-world.yaml --validate
  ```

* 查看日志
  ``` sh
  # 查看vttablet的日志
  $ kubectl logs vttablet-100 vttablet

  # 查看vttablet中mysql的日志
  $ kubectl logs vttablet-100 mysql
  ```

* shell登录
  ``` sh
  # 通过kubectl exec 可以直接连接到对应的节点
  $ kubectl exec vttablet-100 -c vttablet -t -i -- bash -il
  ```

* 查看service详细信息
  ``` sh
  kubectl describe service etcd-global
  ```

* RC副本数量修改
  ``` sh
  # 可以通过本地动态修改RC副本数量实现动态扩容缩容
  kubectl scale rc xxxx --replicas=3
  ```
* 查询Replica Set
  ``` sh
  kubectl get rs
  ```

* 查看Endpoints列表
  ``` sh
  # 查看Endpoints 列表
  # Endpoint => (Pod Ip + ContainerPort)
  kubectl get endpoints
  ```

* 查看namespaces
  ``` sh
  kubectl get namespace
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

## Vitess部署

  本文假定用户已经具备[本地部署Vitess](http://vitess.io/getting-started/local-instance.html)的经验，需要将Vitess部署在Kubernetes，所以对于相关的环境依赖就不在做过多的说明；　如果有不明白的地方请先参阅官方文档。

### 编译安装vtctlclient

  `vtctlclient`工具可以用来向Vitess发送命令，　所以安装环境前我们需要先安装`vtctlclient`。

  ```ssh
  $ go get github.com/youtube/vitess/go/cmd/vtctlclient
  ```
  该命令会在`$GOPATH/src/github.com/youtube/vitess/`目录下下载并且编译`vtctlclient`源码，　同时也会把编译好的vtctlclient二进制文件拷贝到目录`$GOPATH/bin`下。

### 本地kubectl

  如果正常按照文档说明安装，本地kubectl就应该已经安装完成，这里我们需要再次校验一下，确保kubectl处于正常可用状态。
  检查kubectl是否已经正常安装并设置环境变量PATH:

  ```sh
  $ which kubectl
  ### example output:
  # /usr/local/bin/kubectl
  ```

  如果kubectl没有包含在$PATH环境变量中，　就需要设置`KUBECTL`环境变量，否则执行启动脚本的时候无法获取`kubectl`位置。


    ``` sh
    $ export KUBECTL=/export/working/bin/kubectl
    ```

### 启动Vitess集群  

1. 跳转到本地Vitess源代码目录  
     经过上面的步骤后，我们就可以尝试着运行Vitess官方提供的实例了，切换到$GOPATH/src/github.com/youtube/vitess/examples/kubernetes目录下：  

      ```
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

    Vitess[拓扑服务](http://vitess.io/overview/concepts.html#topology-service)存储Vitess集群中所有服务器的协作元数据， 他将此数据存储在支持数据一致性的分布式存储系统中。本例中我们使用[etcd](https://github.com/coreos/etcd)来存储，注意：我们需要自己的etcd集群，与Kubernetes本身使用的集群分开。

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

    界面截图如下:  
    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/vtctld_web.png)

    同时，还可以通过proxy进入[Kubernetes Dashboard]
    (http://kubernetes.io/v1.1/docs/user-guide/ui.html), 监控nodes, pods和服务器状态:

    http://localhost:8001/ui
    控制台截图如下:  
    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/Kubernetes_ui.png)

6. **启动vttablets**

    [tablet](http://vitess.io/overview/concepts.html#tablet)是Vitess扩展的基本单位。tablet由运行在相同的机器上的`vttablet` 和 `mysqld`组成。
    我们在用Kubernetes的时候通过将vttablet和mysqld的容器放在单个[pod](http://kubernetes.io/v1.1/docs/user-guide/pods.html)中来实现耦合。

    运行以下脚本以启动vttablet pod，其中也包括mysqld：

    ``` sh
    vitess/examples/kubernetes$ ./vttablet-up.sh
    ### example output:
    # Creating test_keyspace.shard-0 pods in cell test...
    # Creating pod for tablet test-0000000100...
    # pod "vttablet-100" created
    # Creating pod for tablet test-0000000101...
    # pod "vttablet-101" created
    # Creating pod for tablet test-0000000102...
    # pod "vttablet-102" created
    # Creating pod for tablet test-0000000103...
    # pod "vttablet-103" created
    # Creating pod for tablet test-0000000104...
    # pod "vttablet-104" created
    ```
    启动后在vtctld Web管理界面中很快就会看到一个名为`test_keyspace`的[keyspace](http://vitess.io/overview/concepts.html#keyspace)，其中有一个名为`0`的分片。点击分片名称可以查看
    tablets列表。当5个tablets全部显示在分片状态页面上，就可以继续下一步操作。注意，当前状态tablets不健康是正常的，因为在tablets上面还没有初始化数据库。

    tablets第一次创建的时候， 如果pod对应的node上尚未下载对应的[Vitess镜像](https://hub.docker.com/u/vitess/)文件，那么创建就需要花费较多的时间。同样也可以通过命令行使用`kvtctl.sh`查看tablets的状态。

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
    ### example output:
    # test-0000000100 test_keyspace 0 spare 10.64.1.6:15002 10.64.1.6:3306 []
    # test-0000000101 test_keyspace 0 spare 10.64.2.5:15002 10.64.2.5:3306 []
    # test-0000000102 test_keyspace 0 spare 10.64.0.7:15002 10.64.0.7:3306 []
    # test-0000000103 test_keyspace 0 spare 10.64.1.7:15002 10.64.1.7:3306 []
    # test-0000000104 test_keyspace 0 spare 10.64.2.6:15002 10.64.2.6:3306 []
    ```

    ![Alt text](https://github.com/davygeek/vitessdoc/blob/master/res/kvtctl_list.png)

7.  **初始化MySQL数据库**

    一旦所有的tablets都启动完成， 我们就可以初始化底层数据库了。

    **注意:** 许多`vtctlclient`命令在执行成功时不返回任何输出。

    首先，指定tablets其中一个作为初始化的master。Vitess会自动连接其他slaves的mysqld实例，以便他们开启从master复制数据； 默认数据库创建也是如此。 因为我们的keyspace名称为`test_keyspace`，所以MySQL的数据库会被命名为`vt_test_keyspace`。
    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/0 test-0000000100
    ### example output:
    # master-elect tablet test-0000000100 is not the shard master, proceeding anyway as -force was used
    # master-elect tablet test-0000000100 is not a master in the shard, proceeding anyway as -force was used
    ```

    **注意:** 因为分片是第一次启动， tablets还没有准备做任何复制操作， 也不存在master。如果分片不是一个全新的分片，`InitShardMaster`命令增加`-force`标签可以绕过应用的完整性检查。

    tablets更新完成后，你可以看到一个　**master**, 多个 **replica** 和 **rdonly** tablets:

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
    ### example output:
    # test-0000000100 test_keyspace 0 master 10.64.1.6:15002 10.64.1.6:3306 []
    # test-0000000101 test_keyspace 0 replica 10.64.2.5:15002 10.64.2.5:3306 []
    # test-0000000102 test_keyspace 0 replica 10.64.0.7:15002 10.64.0.7:3306 []
    # test-0000000103 test_keyspace 0 rdonly 10.64.1.7:15002 10.64.1.7:3306 []
    # test-0000000104 test_keyspace 0 rdonly 10.64.2.6:15002 10.64.2.6:3306 []
    ```

    **replica** tablets通常用于提供实时网络流量, 而 **rdonly** tablets通常用于离线处理, 例如批处理作业和备份。
    每个[tablet type](http://vitess.io/overview/concepts.html#tablet)的数量可以在配置脚本`vttablet-up.sh`中配置。

9.  **创建表**

    `vtctlclient`命令可以跨越keyspace里面的所有tablets来应用数据库变更。以下命令可以通过文件`create_test_table.sql`的内容来创建表：

    ``` sh
    # Make sure to run this from the examples/kubernetes dir, so it finds the file.
    vitess/examples/kubernetes$ ./kvtctl.sh ApplySchema -sql "$(cat create_test_table.sql)" test_keyspace
    ```

    创建表的SQL如下所示：

    ``` sql
    CREATE TABLE messages (
      page BIGINT(20) UNSIGNED,
      time_created_ns BIGINT(20) UNSIGNED,
      message VARCHAR(10000),
      PRIMARY KEY (page, time_created_ns)
    ) ENGINE=InnoDB
    ```

    我们可以通过运行此命令来确认在给定的tablet上表是否创建成功，`test-0000000100`是`ListAllTablets`命令显示
    tablet列表中一个tablet的别名：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh GetSchema test-0000000100
    ### example output:
    # {
    #   "DatabaseSchema": "CREATE DATABASE `{{.DatabaseName}}` /*!40100 DEFAULT CHARACTER SET utf8 */",
    #   "TableDefinitions": [
    #     {
    #       "Name": "messages",
    #       "Schema": "CREATE TABLE `messages` (\n  `page` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `time_created_ns` bigint(20) unsigned NOT NULL DEFAULT '0',\n  `message` varchar(10000) DEFAULT NULL,\n  PRIMARY KEY (`page`,`time_created_ns`)\n) ENGINE=InnoDB DEFAULT CHARSET=utf8",
    #       "Columns": [
    #         "page",
    #         "time_created_ns",
    #         "message"
    #       ],
    # ...
    ```

10.  **执行备份**

    现在， 数据库初始化已经应用， 可以开始执行第一次[备份](http://vitess.io/user-guide/backup-and-restore.html)了。在他们连上master并且复制之前， 这个备份将用于自动还原运行的任何其他副本。
    如果一个已经存在的tablet出现故障，并且没有备份数据， 那么他将会自动从最新的备份恢复并且恢复复制。

    选择其中一个 **rdonly** tablets并且执行备份。因为在数据复制期间创建快照的tablet会暂停复制并且停止服务，所以我们使用 **rdonly** 代替 **replica**。
    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh Backup test-0000000104
    ```

    备份完成后，可以通过一下命令查询备份列表：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh ListBackups test_keyspace/0
    ### example output:
    # 201７-０２-21.１42940.test-0000000104
    ```

11. **初始化Vitess路由**  

    在本例中， 我们只使用了没有特殊配置的单节点数据库。因此，我们只需要确保当前配置的服务处于可用状态。
    我们可以通过运行以下命令完成：

    ``` sh
    vitess/examples/kubernetes$ ./kvtctl.sh RebuildVSchemaGraph
    ```

    （此命令执行完成后将不显示任何输出）

12.  **启动vtgate**

    Vitess通过使用[vtgate](http://vitess.io/overview/#vtgate)来路由每个客户端的查询到正确的`vttablet`。
    在KubernetesIn中`vtgate`服务将连接分发到一个`vtgate`pods池中。pods由[replication controller](http://kubernetes.io/v1.1/docs/user-guide/replication-controller.html)来制定。

    ``` sh
    vitess/examples/kubernetes$ ./vtgate-up.sh
    ### example output:
    # Creating vtgate service in cell test...
    # service "vtgate-test" created
    # Creating vtgate replicationcontroller in cell test...
    # replicationcontroller "vtgate-test" created
    ```
13. **说明**

    到目前为止，我们整体的Vitess环境就搭建好了，可以使用命令连接服务进行测试，也可以自己部署对应的应用进行测试。　测试用例可以参考官方提供的[测试用例](http://vitess.io/getting-started/#test-your-cluster-with-a-client-app)。

    通过以上操作我们现在可以通过VitessClient或者Mysql-Client访问数据库了;

## 数据拆分  

### 配置分片信息
  首先， 我们需要做的就是让Vitess知道我们需要怎样对数据进行分片，我们通过提供如下的VSchema配置来实现数据分片配置：
  ``` json
  {
    "Sharded": true,
    "Vindexes": {
      "hash": {
        "Type": "hash"
      }
    },
    "Tables": {
      "messages": {
        "ColVindexes": [
          {
            "Col": "page",
            "Name": "hash"
          }
        ]
      }
    }
  }
  ```

  以上配置我们想通过 `page` 列的以`hash`方式来对数据进行拆分。换一种说法就是，保证相同的`page`的messages数据在同一个分片是上，但是page的分布
  会被随机打散放置在不同的分片是上。

  我们可以通过以下命令把VSchema信息应用到Vitess中：

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh ApplyVSchema -vschema "$(cat vschema.json)" test_keyspace
  ```


### 新分片tablets启动

  在未分片的示例中， 我们在 *test_keyspace* 中启动了一个名称为 *0* 的分片，可以这样表示 *test_keyspace/0*。
  现在，我们将会分别为两个不同的分片启动tablets，命名为 *test_keyspace/-80* 和 *test_keyspace/80-*:

  ``` sh
  vitess/examples/kubernetes$ ./sharded-vttablet-up.sh
  ### example output:
  # Creating test_keyspace.shard--80 pods in cell test...
  # ...
  # Creating test_keyspace.shard-80- pods in cell test...
  # ...
  ```

  因为， Guestbook应用的拆分键是page, 这就会导致pages的数据各有一半会落在不同的分片上； *0x80* 是[拆分键范围](http://vitess.io/user-guide/sharding.html#key-ranges-and-partitions)的中点。
  数据分片范围如下：
  [0x00 - 0x80][0x80-0xFF]

  在数据迁移过渡期间，新的分片和老的分片将会并行运行， 但是在我们正式做服务切换前所有的流量还是由老的分片提供服务。
  我们可以通过`vtctld`web界面或者`kvtctl.sh ListAllTablets test`命令查看tablets状态，当tablets启动成功后，每个分片应该有5个对应的tablets
  一旦tablets启动成功， 我们可以通过为每个新分片指定一个master来初始化同步复制数据：

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/-80 test-0000000200
  vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/80- test-0000000300
  ```

  现在我们应该一共存在15个tablets进程了， 可以通过以下命令查看：

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh ListAllTablets test
  ### example output:
  # test-0000000100 test_keyspace 0 master 10.64.3.4:15002 10.64.3.4:3306 []
  # ...
  # test-0000000200 test_keyspace -80 master 10.64.0.7:15002 10.64.0.7:3306 []
  # ...
  # test-0000000300 test_keyspace 80- master 10.64.0.9:15002 10.64.0.9:3306 []
  # ...
  ```

### 数据复制

  新的tablets默认都是空的， 因此我们需要将原始分片的所有数据复制到两个新的分片上，首先就从Schema开始：

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh CopySchemaShard test_keyspace/0 test_keyspace/-80
  vitess/examples/kubernetes$ ./kvtctl.sh CopySchemaShard test_keyspace/0 test_keyspace/80-
  ```

  下面我们开始拷贝数据， 由于要复制的数据量可能非常大，所以我们使用一个称作 *vtworker* 的特殊批处理程序，根据 *keyspace_id* 路由将每一行数据从
  单个源分片传输到多个目标分片。

  ``` sh
  vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitClone test_keyspace/0
  ### example output:
  # Creating vtworker pod in cell test...
  # pods/vtworker
  # Following vtworker logs until termination...
  # I0416 02:08:59.952805       9 instance.go:115] Starting worker...
  # ...
  # State: done
  # Success:
  # messages: copy done, copied 11 rows
  # Deleting vtworker pod...
  # pods/vtworker
  ```

  注意： 这里我们只指定了数据源分片 *test_keyspace/0* 没有指定目标分片的信息。 *SplitClone* 进程会根据key值覆盖和重叠范围自动判断需要访问的目标分片。
  本例中， 分片 *0* 覆盖整个范围， 所以程序可以自动识别 *-80* 和 *80-* 作为目标分片。因为它们结合起来覆盖范围和 *0* 相同；


  接下来，我们将在老分片上摘除一个 *rdonly* tablet（离线处理)， 作为数据复制一致性提供快照， 提供静态数据同步数据源。 整个服务可以继续服务不停机；
  因为实时流量可以由 *replica* 和 *master* 负责响应处理，不会受到任何影响。 其他批处理任务同样也不会受到影响，
  因为还有一台未暂停的 *rdonly* tablets可以提供服务。


### 过滤复制检查

  当数据从 *rdonly* tablet 复制完成后， *vtworker* 会开启从源分片到每个目标分片的[过滤复制](http://vitess.io/user-guide/sharding.html#filtered-replication)；
  过滤复制会从快照创建时间起，继续同步应用数据。

  当源分片和目标分片数据基本一直时，还会继续复制更新。 你可以通过查看每个分片的内容来看到这个数据同步的变化， 您还可以向留言板应用程序中的各个页面添加新信息，然后在分
  片 *0* 中可以看到所有的消息， 而新的分片仅能看到分布在这个分片上的消息。

  ``` sh
  # See what's on shard test_keyspace/0:
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
  # See what's on shard test_keyspace/-80:
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
  # See what's on shard test_keyspace/80-:
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
  ```

  说明： 可以通过在Guestbook上的不同的页面上添加一些消息， 来观察他们是如何进行数据路由的。

### 数据完整性检查

  *vtworker* 批处理程序还有另一种模式，可以比较源分片和目标分片所有数据的一致性和正确性。
  以下命令将在每个目标分片上校验数据差异:

  ``` sh
  vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/-80
  vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/80-
  ```

  如果发现有任何差异， 程序将会输出差异信息。
  如果所有检测都正常， 你将会看到如下信息:

  ```
  I0416 02:10:56.927313      10 split_diff.go:496] Table messages checks out (4 rows processed, 1072961 qps)
  ```


### 服务切换

  现在，我们就可以把所有服务切换到新的分片上，由新的分片为应用提供服务。
  我们可以使用[MigrateServedTypes](http://vitess.io/reference/vtctl.html#migrateservedtypes)命令，一次迁移同一
  个[cell](http://vitess.io/overview/concepts.html#cell-data-center)上的一个[tablet type](http://vitess.io/overview/concepts.html#tablet)；
  在master切换完成之前，在任何时候我们都可以进行数据回滚。

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 rdonly
  vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 replica
  vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 master
  ```


  在 *master* 迁移过程中， 首先会停止老master接收数据更新请求； 然后进程需要等待新的分片通过过滤
  复制数据完全一致， 其次才会开启新的服务。 由于过滤复制已经确保数据实时更新，因此在切换过程中应用应该只会出现几秒钟的不可用。

  master完全迁移后就会停止过滤复制， 新分片的数据更新就会被开启， 但是老分片的更新依然是不可用。
  读者可以自己尝试下： 将消息添加到留言板页面，然后检查数据库内容是否有同步更新。

  ``` sh
  # See what's on shard test_keyspace/0
  # (no updates visible since we migrated away from it):
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
  # See what's on shard test_keyspace/-80:
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
  # See what's on shard test_keyspace/80-:
  vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
  ```



### 老分片下线

  现在，所有的服务都由新的分片进行提供， 我们可以老的分片进行下线处理，使资源可以回收利用。 通过运行脚本`vttablet-down.sh`关闭一组
  非拆分的分片：

  ``` sh
  vitess/examples/kubernetes$ ./vttablet-down.sh
  ### example output:
  # Deleting pod for tablet test-0000000100...
  # pods/vttablet-100
  # ...
  ```
  通过以上命令，我们已经关闭了老分片中的服务，下面可以通过以下命令删除控制的分片信息，保证元数据一致。

  ``` sh
  vitess/examples/kubernetes$ ./kvtctl.sh DeleteShard -recursive test_keyspace/0
  ```

  我们可以通过 **Topology** 页面或者使用`kvtctl.sh ListAllTablets test`命令，来查看元数据信息， 通过运行命令发现
  分片 *0* 已经不存在了，说明我们已经成功删除了分片 *0*， 当系统中存在不可用或者闲置的分片的时候就可以通过这种方式删除。



## 其他
    基础环境的搭建完全是依赖于Kubernetes，以下列出了对应的Kubernetes文档，有需要的可以根据需要进行查阅。

    * [Kubernetes官方文档](https://kubernetes.io/docs)
    * [Kubernetes中文文档](https://www.kubernetes.org.cn/k8s)
    * [测试用例](http://vitess.io/getting-started/#test-your-cluster-with-a-client-app)
