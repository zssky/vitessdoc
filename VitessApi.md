## Vitess API接口说明文档

1. 数据库列表(keyspaces list)
   GET :15000/api/keyspaces/

2. 创建数据库
   POST :15000/api/vtctl/

   content-type:application/json
   ["CreateKeyspace","gwgggg"]

3. 删除数据库
   POST :15000/api/vtctl/

   content-type:application/json
   ["DeleteKeyspace","-recursive","gwgggg"]

4. 数据库验证
   POST :15000/api/vtctl/

   Content-Type:application/json;charset=UTF-8
   ["ValidateKeyspace","-ping-tablets","gw_keyspace"]

   ["ValidateSchemaKeyspace","gw_keyspace"]

   ["ValidateVersionKeyspace","test"]

5. RebuildKeyspaceGraph
   POST :15000/api/vtctl/

   Content-Type:application/json;charset=UTF-8
   ["RebuildKeyspaceGraph","gw_keyspace"]


6. 创建分片
   POST :15000/api/vtctl/

   Content-Type:application/json;charset=UTF-8
   ["CreateShard","test/-10"]

7. 删除分片
   POST :15000/api/vtctl/

   Content-Type:application/json;charset=UTF-8
   ["DeleteShard","-recursive","-even_if_serving","test/-10"]

8. 获取分片列表
   GET :15000/api/shards/gw_keyspace/

   ["-10","10-"]

9. 获取Tablets列表
   POST :15000/api/tablets/

   content-type:application/x-www-form-urlencoded
   shard=gw_keyspace/-10


  [
    {
      "cell": "test",
      "uid": 1201
    },
    {
      "cell": "test",
      "uid": 1202
    },
    {
      "cell": "test",
      "uid": 1203
    },
    {
      "cell": "test",
      "uid": 1204
    }
  ]  

10. 获取Tablets中表信息
    POST :15000/api/vtctl/

    content-type:application/json
    ["GetSchema","test-1201"]

11. 执行sql
   POST :15000/api/schema/apply
   Content-Type:application/json;charset=UTF-8

   {"Keyspace":"gw_keyspace","SQL":"CREATE TABLE user_info(\n `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '用户id',\n `username` varchar(32) NOT NULL COMMENT '用户名称',\n `userpasswd` varchar(32) NOT NULL COMMENT '用户密码',\n `email` varchar(32) DEFAULT NULL COMMENT '用户密码',\n `remark` varchar(128) DEFAULT NULL COMMENT ' 备注',\n `createtime` timestamp NOT NULL DEFAULT '2017-01-01 00:00:00' COMMENT '创建时间',\n `updatetime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',\n PRIMARY key(`id`)\n)ENGINE=InnoDB AUTO_INCREMENT = 1 DEFAULT CHARSET=utf8;"}


12. 初始化Master
   POST :15000/api/vtctl/

   Content-Type:application/json;charset=UTF-8
   ["InitShardMaster", "-force", "gw_keyspace/10-","test-0000001301"]


13. ReparentTablet
   POST :15000/api/vtctl/

   application/json; charset=utf-8

   ["ReparentTablet","test-1301"]


14. 获取POD列表
    GET /api/v1/namespaces/{namespace}/pods

15. 创建POD
    POST /api/v1/namespaces/{namespace}/pods

16. 删除POD
    DELETE /api/v1/namespaces/{namespace}/pods/{name}

17. 获取POD详细信息
    GET /api/v1/namespaces/{namespace}/pods/{name}

18. 获取Service列表
    GET /api/v1/namespaces/{namespace}/services

19. 创建Service
    POST /api/v1/namespaces/{namespace}/services

20. 删除Service
    DELETE /api/v1/namespaces/{namespace}/services/{name}

21. 获取ReplicationControllerList
    GET /api/v1/namespaces/{namespace}/replicationcontrollers

22. 创建ReplicationController
    POST /api/v1/namespaces/{namespace}/replicationcontrollers

23. 删除ReplicationController
    DELETE /api/v1/namespaces/{namespace}/replicationcontrollers/{name}

24. 获取ReplicationController详细信息
    GET /api/v1/namespaces/{namespace}/replicationcontrollers/{name}

25. 获取Namespace列表
    GET /api/v1/namespaces

26. 创建Namespace
    POST /api/v1/namespaces

27. 删除Namespace
    DELETE /api/v1/namespaces/{name}

28. 获取Namespace详细信息
    GET /api/v1/namespaces/{name}
