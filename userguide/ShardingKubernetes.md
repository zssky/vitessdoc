本指南将指导我们完成，从[Kubernetes](http://kubernetes.io/)上一个未分片的Vitess[keyspace](http://vitess.io/overview/concepts.html#keyspace)进
行分片的处理过程。

## 先决条件

我们假设读者已经按照[Kubernetes基本环境搭建指南](http://vitess.io/getting-started/)完成相应的操作, 并且就只剩下集群的运行了。

## 概述

我们将按照类似通常[水平拆分](http://vitess.io/user-guide/horizontal-sharding.html)指南一样的步骤去处理，除此之外我们还会给出Vitess集群在Kubernetes上运行的相关命令。

因为Vitess的[分片](http://vitess.io/user-guide/sharding.html)操作对应用层是透明的，所以[Guestbook](https://github.com/youtube/vitess/tree/master/examples/kubernetes/guestbook)
实例将会在[resharding](http://vitess.io/user-guide/sharding.html#resharding)的过程中一直提供服务； 确保Vitess集群在分片的过程中一直保提供服务不会停机。

## 配置分片信息

第一步就是需要让Vitess知道我们需要怎样对数据进行分片，我们通过提供如下的VSchema配置来实现：
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

这说明我们想通过 `page` 列的hash来对数据进行拆分。换一种说法就是，保证相同的page的messages数据在同一个分片是上，但是page的分布
会被随机放置在不同的分片是上。

我们可以通过以下命令把VSchema 信息加载到Vitess中：

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh ApplyVSchema -vschema "$(cat vschema.json)" test_keyspace
```

## 新分片tablets启动

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

因为， Guestbook应用的拆分键是page, 这就会导致pages的数据各有一半会落在不同的分片上； *0x80* 是[拆分键范围](http://vitess.io/user-guide/sharding.html#key-ranges-and-partitions)的终点。

在数据过渡期间，新的分片和老的分片将会并行运行， 但是在我们做切换前所有的流量还是由老的分片提供。

我们可以通过`vtctld`界面或者`kvtctl.sh ListAllTablets test`命令的输出查看tablets状态，当tablets启动成功后，每个分片应该有5个对应的tablets

一旦tablets启动成功， 我们可以通过为每个新分片选择一个主分片来初始化复制：

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/-80 test-0000000200
vitess/examples/kubernetes$ ./kvtctl.sh InitShardMaster -force test_keyspace/80- test-0000000300
```

Now there should be a total of 15 tablets, with one master for each shard:

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

## 从原始分片复制数据

新的tablets开始是空的， 因此我们需要将所有内容从原始分片复制到两个新的分片上，首先就从数据库开始：

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh CopySchemaShard test_keyspace/0 test_keyspace/-80
vitess/examples/kubernetes$ ./kvtctl.sh CopySchemaShard test_keyspace/0 test_keyspace/80-
```

下面我们拷贝数据。由于要复制的数据量可能非常大，我们使用一个称作 *vtworker* 的特殊批处理程序，根据 *keyspace_id* 路由将每一行数据从
单个源传输到多个目标。

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

注意： 这里我们只指定了数据源分片 *test_keyspace/0* 。 *SplitClone* 进程会根据key值覆盖和重叠范围自动判断需要访问的分片。
本例中， 分片 *0* 覆盖整个范围， 所以可以识别 *-80* 和 *80-* 作为目标分片。因为它们结合起来覆盖范围相同；


接下来，我们将在一个 *rdonly* tablet上暂停复制（离线处理)， 为数据一致性提供快照。 应用程序可以继续服务不停机；
因为实时流量处理由 *replica* 和 *master* 负责响应，不会受到任何影响。 其他批处理任务同样也不会受到影响，
因为还有一台未暂停的 *rdonly* tablets可以提供服务。

## 检查过滤复制

一旦从已经暂停的快照数据复制完成， *vtworker* 会开启从源分片到每个目标分片的[过滤复制](http://vitess.io/user-guide/sharding.html#filtered-replication)
过滤复制会从快照创建时间起，继续同步应用数据。

当追赶上目标分片数据时，还会继续复制新更新。 你可以通过查看每个分片的内容来看到这个数据同步的变化， 您可以向留言板应用程序中的各个页面添加新邮件。
分片 *0* 可以看到所有的消息， 而新的分片仅能看到分布在这个分片上的消息。

``` sh
# See what's on shard test_keyspace/0:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
# See what's on shard test_keyspace/-80:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
# See what's on shard test_keyspace/80-:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
```

可以通过在Guestbook上的不同的页面上添加一些消息， 来观察他们是如何进行数据路由的。

## 检查复制的数据完整性

*vtworker* 批处理有另一种模式，将比较源和目标，以确保所有数据的存在和正确。
以下命令将在每个目标分片上校验数据差异:

``` sh
vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/-80
vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/80-
```

如果发现任何差异， 程序将会输出差异信息。
如果所有检测都正常， 你将会看到如下信息:

```
I0416 02:10:56.927313      10 split_diff.go:496] Table messages checks out (4 rows processed, 1072961 qps)
```

## 切换到新的分片

现在，我们就可以把所有服务切换到新的分片上，由新的分片为应用提供服务。
我们可以使用[MigrateServedTypes](http://vitess.io/reference/vtctl.html#migrateservedtypes)命令，一次迁移一
个[cell](http://vitess.io/overview/concepts.html#cell-data-center)上的一个[tablet type](http://vitess.io/overview/concepts.html#tablet)
在master切换完成之前，我们可以在任何点都可以进行回滚。

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 rdonly
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 replica
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 master
```


在 *master* 迁移过程中， 首先会停止老master接收更新请求； 然后进程需要等待新的分片通过过滤
复制数据完全一直之后， 才会开启新的服务。 由于过滤复制已跟随实时更新，因此应该只有几秒钟的主机不可用。

master完全钱以后就会停止过滤复制， 新分片的数据更新就会被开启， 但是老分片的更新依然是不可用。
读者可以自己尝试下： 将消息添加到留言板页面，然后检查数据库内容

``` sh
# See what's on shard test_keyspace/0
# (no updates visible since we migrated away from it):
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
# See what's on shard test_keyspace/-80:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
# See what's on shard test_keyspace/80-:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
```

## 移除老的分片

现在，所有的服务都由新的分片进行提供， 我们可以移除老的分片。 通过运行脚本`vttablet-down.sh`关闭一组
非拆分的分片：

``` sh
vitess/examples/kubernetes$ ./vttablet-down.sh
### example output:
# Deleting pod for tablet test-0000000100...
# pods/vttablet-100
# ...
```

下面我们可以删除空置分片，通过以下命令可以删除：

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh DeleteShard -recursive test_keyspace/0
```

我们通过 **Topology** 页面或者使用`kvtctl.sh ListAllTablets test`命令，发现分片 *0* 已经不存在了，
说明我们已经成功删除了不可用的分片， 当系统中存在不可用的分片的时候就可以通过这种方式删除。

## 清理

在关闭容器引擎之前， 我们需要关闭Vitess服务； 然后Kubernetes会负责清理它创建的其他实体， 例如：外部负载平衡器

我们可以通过运行`./vttablet-down.sh`来清理非分片状态的tablets, 可以通过运行`./sharded-vttablet-down.sh`来关闭拆分状态下的tablets

``` sh
vitess/examples/kubernetes$ ./guestbook-down.sh
vitess/examples/kubernetes$ ./vtgate-down.sh
vitess/examples/kubernetes$ ./sharded-vttablet-down.sh
vitess/examples/kubernetes$ ./vtctld-down.sh
vitess/examples/kubernetes$ ./etcd-down.sh
```
