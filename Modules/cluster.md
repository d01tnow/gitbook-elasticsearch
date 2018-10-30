# Cluster

## Shard Allocation Filtering

在集群范围内控制分片分配.
通过动态设置 API 进行配置. 支持的配置有:

配置项 | 说明
------ | ------
cluster.routing.allocation.include.{attribute} | 分片可以被分配到包含指定的{attribute}节点上. 字符串类型. 多个属性值用 ',' 分隔. 节点包含一个属性值即可.
cluster.routing.allocation.require.{attribute} | 只分配到拥有指定的{attribute}节点上. 字符串类型. 多个属性值用 ',' 分隔. 节点必须满足所有所有属性值.
cluster.routing.allocation.exclude.{attribute} | **排除**包含指定的{attribute}的节点. 字符串类型. 多个属性值用 ',' 分隔. 节点包含一个属性值即可.

支持的属性(支持通配符 '*'):

名称 | 说明
---- | ----
_name | 通过名称匹配节点
_ip   | 通过 IP 匹配节点
_host | 通过 hostname 匹配节点

配置命令

```shell
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "transient" : {
    "cluster.routing.allocation.exclude._ip" : "10.0.0.1,192.168.2.*"
  }
}
'
```
