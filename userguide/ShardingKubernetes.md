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

注意： 这里我们只指定了数据源分片 *test_keyspace/0*
Notice that we've only specified the source shard, *test_keyspace/0*.
The *SplitClone* process will automatically figure out which shards to use
as the destinations based on the key range that needs to be covered.
In this case, shard *0* covers the entire range, so it identifies
*-80* and *80-* as the destination shards, since they combine to cover the
same range.

Next, it will pause replication on one *rdonly* (offline processing) tablet
to serve as a consistent snapshot of the data. The app can continue without
downtime, since live traffic is served by *replica* and *master* tablets,
which are unaffected. Other batch jobs will also be unaffected, since they
will be served only by the remaining, un-paused *rdonly* tablets.

## Check filtered replication

Once the copy from the paused snapshot finishes, *vtworker* turns on
[filtered replication](http://vitess.io/user-guide/sharding.html#filtered-replication)
from the source shard to each destination shard. This allows the destination
shards to catch up on updates that have continued to flow in from the app since
the time of the snapshot.

When the destination shards are caught up, they will continue to replicate
new updates. You can see this by looking at the contents of each shard as
you add new messages to various pages in the Guestbook app. Shard *0* will
see all the messages, while the new shards will only see messages for pages
that live on that shard.

``` sh
# See what's on shard test_keyspace/0:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
# See what's on shard test_keyspace/-80:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
# See what's on shard test_keyspace/80-:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
```

Add some messages on various pages of the Guestbook to see how they get routed.

## Check copied data integrity

The *vtworker* batch process has another mode that will compare the source
and destination to ensure all the data is present and correct.
The following commands will run a diff for each destination shard:

``` sh
vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/-80
vitess/examples/kubernetes$ ./sharded-vtworker.sh SplitDiff test_keyspace/80-
```

If any discrepancies are found, they will be printed.
If everything is good, you should see something like this:

```
I0416 02:10:56.927313      10 split_diff.go:496] Table messages checks out (4 rows processed, 1072961 qps)
```

## Switch over to new shards

Now we're ready to switch over to serving from the new shards.
The [MigrateServedTypes](http://vitess.io/reference/vtctl.html#migrateservedtypes)
command lets you do this one
[tablet type](http://vitess.io/overview/concepts.html#tablet) at a time,
and even one [cell](http://vitess.io/overview/concepts.html#cell-data-center)
at a time. The process can be rolled back at any point *until* the master is
switched over.

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 rdonly
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 replica
vitess/examples/kubernetes$ ./kvtctl.sh MigrateServedTypes test_keyspace/0 master
```

During the *master* migration, the original shard master will first stop
accepting updates. Then the process will wait for the new shard masters to
fully catch up on filtered replication before allowing them to begin serving.
Since filtered replication has been following along with live updates, there
should only be a few seconds of master unavailability.

When the master traffic is migrated, the filtered replication will be stopped.
Data updates will be visible on the new shards, but not on the original shard.
See it for yourself: Add a message to the guestbook page and then inspect
the database content:

``` sh
# See what's on shard test_keyspace/0
# (no updates visible since we migrated away from it):
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000100 "SELECT * FROM messages"
# See what's on shard test_keyspace/-80:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000200 "SELECT * FROM messages"
# See what's on shard test_keyspace/80-:
vitess/examples/kubernetes$ ./kvtctl.sh ExecuteFetchAsDba test-0000000300 "SELECT * FROM messages"
```

## Remove original shard

Now that all traffic is being served from the new shards, we can remove the
original one. To do that, we use the `vttablet-down.sh` script from the
unsharded example:

``` sh
vitess/examples/kubernetes$ ./vttablet-down.sh
### example output:
# Deleting pod for tablet test-0000000100...
# pods/vttablet-100
# ...
```

Then we can delete the now-empty shard:

``` sh
vitess/examples/kubernetes$ ./kvtctl.sh DeleteShard -recursive test_keyspace/0
```

You should then see in the vtctld **Topology** page, or in the output of
`kvtctl.sh ListAllTablets test` that the tablets for shard *0* are gone.

## Tear down and clean up

Before stopping the Container Engine cluster, you should tear down the Vitess
services. Kubernetes will then take care of cleaning up any entities it created
for those services, like external load balancers.

Since you already cleaned up the tablets from the original unsharded example by
running `./vttablet-down.sh`, that step has been replaced with
`./sharded-vttablet-down.sh` to clean up the new sharded tablets.

``` sh
vitess/examples/kubernetes$ ./guestbook-down.sh
vitess/examples/kubernetes$ ./vtgate-down.sh
vitess/examples/kubernetes$ ./sharded-vttablet-down.sh
vitess/examples/kubernetes$ ./vtctld-down.sh
vitess/examples/kubernetes$ ./etcd-down.sh
```

Then tear down the Container Engine cluster itself, which will stop the virtual
machines running on Compute Engine:

``` sh
$ gcloud container clusters delete example
```

It's also a good idea to remove the firewall rules you created, unless you plan
to use them again soon:

``` sh
$ gcloud compute firewall-rules delete vtctld guestbook
```
