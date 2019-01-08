# 集群缩容

## 背景

需要对某些节点做永久下线处理. 要求不能丢失数据, 且集群正常运行.

## 平滑下线节点

### data类型节点

1. 将节点从集群路由策略中排除. 下面例子中使用 _name 属性下线指定的节点. 还有 _ip, _host 属性. 具体参考[Cluster-Shards-Routing-Filter](../Modules/cluster.md)

    ```shell
    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "transient" : {
        "cluster.routing.allocation.exclude._name" : "node3,node5"
      }
    }
    '
    ```

2. 查看集群状态
3. 等待分片从被排除节点迁移到其他节点上.
4. 关闭节点服务
5. 取消节点禁用策略

    ```shell
    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "transient" : {
        "cluster.routing.allocation.exclude._name" : null
      }
    }
    '
    ```

### master 类型节点

如果计划下线后,集群中 master 节点个数 **大于等于** 配置文件中 'discovery.zen.minimum_master_nodes' 个数. 则可以进行不停机, 缩容 master 节点. 步骤如下:

1. 查看配置文件, 确保该节点仅仅是 master 类型. 如果同时是 data 类型, 那么将节点从集群路由策略中排除. 同 [data类型节点](#data类型节点), 并且确保该 data 节点上的分片都迁移到其他 data 节点中才能进行下面步骤.
2. 查看将要下线的节点是否是集群的主节点. curl http://localhost:9200/_cat/nodes?v . master 列为 '*' 即是集群主节点. 如非必要, 请下线非主节点.
3. 关闭节点服务
4. 查看集群状态, 如果下线的是当前集群的主节点, 那么集群在选举超时配置(默认选举时间为 '3s')后重新选举主节点. 集群在选举过程中, 集群的功能不可用.

如果计划下线后, 集群中 master 节点个数 **小于** 配置文件中 'discovery.zen.minimum_master_nodes' 个数. 那么, 必须下线所有 master 节点后, 修改配置文件, 再重启所有 master 节点. 下面步骤仅适用于 master , data 角色分离的情况. 步骤如下:

1. 修改配置文件. discovery.zen.minimum_master_nodes, discovery.zen.ping.unicast.hosts
2. 查看哪个节点是主节点. curl http://localhost:9200/_cat/nodes?v . master 列为 '*' 即是集群主节点.
3. 确保即将下线节点仅仅是 master 候选节点. curl http://localhost:9200/_cat/nodes?v . node.role 列为 "mi" 角色.
4. 先关闭非主节点服务, 最后关闭主节点服务
5. 先重启主节点服务, 然后再启动非主节点服务
6. 查看集群状态. curl http://localhost:9200/_cat/health?pretty

如果计划下线后, 集群中 master 节点个数 **小于** 配置文件中 'discovery.zen.minimum_master_nodes' 个数. 并且所有节点都是 'mdi' 角色(即是候选 master 节点, 又是 data 节点). 那么, 进行[集群全量重启](./cluster-all-restart.md).
