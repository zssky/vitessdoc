# ETCD运维手册

## 1. 查看成员列表
  ``` sh
  # 查看成员列表
  $ export ETCDCTL_API=3
  $ etcdctl --endpoints=http://etcd-global:4001 member list

  # 3c7b0b72a83cb207, started, etcd-global-l2ng6, http://b449bd7.etcd-global-srv.etcd2test.svc.hades.local:7001, http://192.168.81.75:4001
  # 446539d2bec101ad, started, etcd-global-dhzvw, http://790c7240.etcd-global-srv.etcd2test.svc.hades.local:7001, http://192.168.81.78:4001
  # e5e76d85dd6a2837, started, etcd-global-xkvps, http://add369ac.etcd-global-srv.etcd2test.svc.hades.local:7001, http://192.168.81.74:4001
  ```

## 2. 查看可用服务列表
  ``` sh
  # 查看可用服务列表
  $ getsrv etcd-server tcp etcd-global-srv
  # b449bd7.etcd-global-srv.etcd2test.svc.hades.local.:7001
  # 790c7240.etcd-global-srv.etcd2test.svc.hades.local.:7001
  # add369ac.etcd-global-srv.etcd2test.svc.hades.local.:7001
  ```


## 3. 删除成员
  ``` sh
  # 446539d2bec101ad, started, etcd-global-dhzvw, http://790c7240.etcd-global-srv.etcd2test.svc.hades.local:7001, http://192.168.81.78:4001
  # member_id = 446539d2bec101ad
  $ export ETCDCTL_API=3
  $ etcdctl --endpoints=http://etcd-global:4001 member remove $member_id

  ```
## 4. 数据备份

## 5. 集群节点之间时间偏差问题

## 6. 权限控制

## 7. 使用备份数据重新启动集群

## 8. 异常情况说明
  正常情况下容器销毁的时候会自动把自己从集群中移除， 如果出现异常情况，比如宿主机直接挂了，可能还没有来得及自己删除就退出了。 这种情况如果需要新加入节点可以手动删除成员，然后再添加即可。

  通过测试ETCD正常容器关闭的时候会自动把自己从集群中摘除， 比如启动后集群有三个节点， 关闭一个节点后会自动从集群中删除（member remove xxxxx); 详细的可以参考etcd集群启动脚本里面的preStop
  关闭一个后集群中两个节点依然会正常处理请求， 然后如果我们再关闭一个节点， 同样会自动摘除，集群中就会剩下一个节点， 集群中剩下一个节点的时候依然可以保证正常的服务。 这样使用就大大提高的系统可用性，
  如果集群中部分节点关闭同样可以确保系统是正常的。
