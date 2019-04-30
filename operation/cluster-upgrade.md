# 集群全量重启升级(5.6->6.x)

5.6版本的 ES 可以滚动升级到 6.x. 5.6 之前版本需要全量停机维护重启升级到 6.x. 具体查看[升级路径表格](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html#upgrade-paths)

集群全量重启过程, 关闭所有集群内节点, 升级, 然后重启集群.

## 具体流程

**注意**: 这个过程是不备份数据, 直接使用旧版本数据目录的方案. 个人觉得更保险的方案是先备份然后新集群内恢复的方案(该方案需要2倍于原集群的磁盘容量, 过程较慢)

1. 做好准备. 比如: 下载 image, 修改配置文件等.
2. 关闭分片分配功能. 当节点下线后, 分配进程会等待 index.unassigned.node_left.delayed_timeout (默认1分钟), 然后开始分片复制到其他节点. 这会消耗大量 I/O. 为了避免不必要的 I/O, 关闭节点前先停止分片分配功能.

    ```shell
    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "transient": {
        "cluster.routing.allocation.enable": "none"
      }
    }
    '
    ```

3. 停止索引,执行刷盘: curl -X POST "localhost:9200/_flush/synced"
4. 在 Kibana - Management - SavedObjects 界面 Export Objects
5. 删除 .kibana 索引数据(它里面记录了 xpack license). curl -XDELETE "localhost:9200/.kabana"
6. 删除 xpack license. curl -XDELETE "localhost:9200/_xpack/license"
7. 关闭数据节点和非主节点, 最后关闭主节点
8. 修改"备份目录所有者为 1000:1000": sudo chown -R 1000:1000 /path/to/backup
9. 移动节点数据中的 _state 目录和 node.lock 文件, 到节点数据目录下的 bak. sudo mkdir /path/to/es-data/nodes/0/bak && sudo mv /path/to/es-data/nodes/0/node.lock /path/to/es-data/nodes/0/bak && sudo mv /path/to/es-data/nodes/0/_state /path/to/es-data/nodes/0/bak
10. 启动主节点, 候选主节点和其他节点
11. 查看集群状态. 在集群状态为 green 后再启用分片分配功能
12. 启用分片分配功能

    ```shell
    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "transient": {
        "cluster.routing.allocation.enable": null
      }
    }
    '
    ```

13. 查看集群状态: curl -X GET "localhost:9200/_cat/health"
14. 查看集群恢复过程: curl -X GET "localhost:9200/_cat/recovery"
15. 进入 Kibana - Management - SavedObject 界面 import 备份的对象

参考 [Full cluster restart upgrade](https://www.elastic.co/guide/en/elasticsearch/reference/current/restart-upgrade.html)
