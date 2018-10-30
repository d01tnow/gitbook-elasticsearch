# 集群全量重启

5.6版本的 ES 可以滚动升级到 6.x. 5.6 之前版本需要全量停机维护重启升级到 6.x. 具体查看[升级路径表格](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html#upgrade-paths)

集群全量重启过程, 关闭所有集群内节点, 维护/升级, 然后重启集群.

## 具体流程

1. 做好准备. 比如: 下载 image, 修改配置文件等.
2. 关闭分片分配功能. 当节点下线后, 分配进程会等待 index.unassigned.node_left.delayed_timeout (默认1分钟), 然后开始分片复制到其他节点. 这会消耗大量 I/O. 为了避免不必要的 I/O, 关闭节点前先停止分片分配功能.

  ```shell
  curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {
    "persistent": {
      "cluster.routing.allocation.enable": "none"
    }
  }
  '
  ```

3. 停止索引,执行刷盘: curl -X POST "localhost:9200/_flush/synced"
4. 关闭数据节点和非主节点, 最后关闭主节点
5. 维护/升级
6. 启动主节点, 候选主节点和其他节点
7. 查看集群状态. 在集群状态为 yellow 后再启用分片分配功能
8. 启用分片分配功能

  ```shell
  curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
  {
    "persistent": {
      "cluster.routing.allocation.enable": null
    }
  }
  '
  ```

9. 查看集群状态: curl -X GET "localhost:9200/_cat/health"
10. 查看集群恢复过程: curl -X GET "localhost:9200/_cat/recovery"

参考 [Full cluster restart upgrade](https://www.elastic.co/guide/en/elasticsearch/reference/current/restart-upgrade.html)
