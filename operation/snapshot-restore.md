# Snapshot And Restore

elasticsearch v5.6

## Snapshot

前提条件: master, data 节点的配置文件 elasticsearch.yml 中必须要有 path.repo 配置项. 并且在服务启动时, 服务可以正常访问到配置的目录. 如果无法访问, 则服务会退出.

### 查看已注册的仓库

  ```bash
  ## 查看所有已注册仓库
  curl -XGET "http://localhost:9200/_snapshot?pretty"
  ## 查看指定名称的仓库, 下面例子中 repo_name 是仓库名称. 支持通配符 '*'.
  curl -XGET "http://localhost:9200/_snapshot/repo_name?pretty"
  ```

### 注册仓库

* 使用共享文件系统的例子. settings 部分是和具体的仓库 type 相关的.

    ``` shell
    ## 注册一个仓库
    curl -XPUT 'http://localhost:9200/_snapshot/repo_name' -H 'Content-Type: application/json' -d'
    {
      "type": "fs",
      "settings": {
        "location": "/elastic/backup/repo_name"
      }
    }'
    ```

  **注意**: 如果出现错误"cannot create blob store", 可能是 "/elastic/backup" 无写入权限

  | 字段 | 含义 | 是否必需 | 备注 |
  | ---- | ---- | ------- | ---- |
  repo_name | 仓库名称 | 必需 |
  type | 仓库的文件系统类型 | 必需 | fs: 共享的文件系统. ES 还支持 [S3][1], [HDFS][2], [azure][3], [gcs][4]
  settings | 文件系统类型相关的设置 | 必需 |

  支持的设置项有:
  | 字段 | 含义 | 是否必需 | 备注 |
  | ---- | ---- | ------- | ---- |
  location | 指定快照存储位置. | 必需 | 所有的 master 和 data 节点都必需在 elasticsearch.yml 配置过相同的 path.repo: ["/elastic/backup", "/mount/backups"]. 服务要有权限访问这些已挂载的目录.
  compress | 是否压缩元数据文件(index mapping 和 settings), 数据文件不会压缩 | 可选 | 默认: true
  max_restore_bytes_per_sec | 单节点每秒恢复的最大字节数 | 可选 | 默认: 40mb
  max_snapshot_bytes_per_sec | 单节点每秒快照的最大字节数 | 可选 | 默认: 40mb
  readonly | 使仓库只读 | 可选 | 默认: false
  chunk_size | 快照过程中大文件可能会被拆分成多个 chunk. 设置每个 chunk 的大小. 需指定单位. 比如: 1g, 10m, 5k | 可选 | 默认: null . 表示无限制.

### 删除已注册仓库

```shell
curl -X DELETE "localhost:9200/_snapshot/repo_name
```

### 查看备份

查看指定名称的快照信息.

``` shell
curl -X GET "localhost:9200/_snapshot/repo_name/snapshot_1?pretty"
```

查看所有快照信息.

``` shell
curl -X GET "localhost:9200/_snapshot/repo_name/_all?pretty"
```

查看详细的备份状态

```shell
curl -X GET "localhost:9200/_snapshot/repo_name/snapshot_1/_status?pretty"
```

查看正在运行的快照.

```shell
curl -X GET "localhost:9200/_snapshot/repo_name/_current?pretty"
```

返回结果中的状态值如下表:

状态 | 说明
---- | ----
IN_PROGRESS | 快照进行中
SUCCESS | 快照完成, 所有分片存储成功.
FAILED  | 快照完成, 但有错误, 存储失败
PARTIAL | 集群状态存储完成, 但是至少一个分片数据存储失败. failure 节包含更详细的信息.
INCOMPATIBLE | 快照数据版本和当前版本不兼容.

### 备份

在不指定索引的情况下, 默认备份集群内**所有**打开状态的索引.

比较容易出现的错误是: 备份目录权限问题. 即 ES 服务无正确权限访问备份目录. docker 方式下, elasticsearch 容器使用 user:group 为 102:102. 故在创建好备份根目录后 chown -R 102:102 /elastic/backup .
简单地, 使用 top 命令查看 ES 进程的 USER. 或者 docker ps 查看 ES 容器 ID, 然后 docker exec -it $ID bash 进入容器, cat /etc/passwd , 查看 elasticsearch 用户的 user_id:group_id

创建备份的例子: 备份名称为 snapshot_1. 只备份 index_1 和 index_2. 等待备份完成才返回. 不带 wait_for_completion 时立即返回. 通过 GET /_snapshot/repo_name/snapshot_1 方式查看备份情况.

``` shell
curl -XPUT 'http://localhost:9200/_snapshot/repo_name/snapshot_1?wait_for_completion=true' -H 'Content-Type:application/json' -d '{
  "indices": "index_1,index_2",
  "ignore_unavailable": true,
  "include_global_state": false
}'

```

创建的快照名称包含当天日期的方法, 在 ES v5.6 版中报错.

``` shell
# PUT /_snapshot/repo_name/<snapshot-{now/d}>
curl -X PUT "localhost:9200/_snapshot/repo_name/%3Csnapshot-%7Bnow%2Fd%7D%3E"
```

索引名称可以包含通配符 '\*'. 支持排除某索引(用 '-'), 比如: "index*,-index1".

### 删除已完成备份, 停止正在运行的备份或恢复操作

```shell
curl -X DELETE "localhost:9200/_snapshot/repo_name/snapshot_1"
```

### 恢复

恢复操作会自动打开快照时是关闭状态的索引. 如果被恢复索引名在当前集群内不存在, 则新建索引. 如果被恢复的索引名称在当前集群内已经存在, 那么恢复操作只恢复当前集群内处于**已关闭**状态且分片数与被恢复索引相同的索引. 如果 include_global_state: true(默认为 false), 恢复操作新建当前集群内不存在的模板, **覆盖**当前集群内已存在的模板. 持久化设置会被合并.

在不指定索引名称的情况下恢复指定快照中的**所有**索引. 并且**不恢复**集群状态信息(模板, 设置).

```shell
curl -X POST "localhost:9200/_snapshot/repo_name/snapshot_1/_restore"
```

可以指定恢复的索引名称, 同时可以重命名索引, 改变设置. rename_pattern 是符合 Java 正则表达的语法. 但是, 作为 application/json 格式需要转义 '\'.

```shell
curl -X POST "localhost:9200/_snapshot/repo_name/snapshot_1/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "index_2018.01.01,index_2018.01.02",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [
    "index.refresh_interval"
  ],
  "ignore_unavailable": true,
  "include_global_state": true,
  "rename_pattern": "index_(\\d{4}.\\d{2}.\\d{2})",
  "rename_replacement": "restored_index_$1"
}
'
```

在恢复过程修改索引设置. 下面例子中, 修改副本数为 0. 并且忽略 index.refresh_interval, 使之恢复为默认值.

```shell
curl -X POST "localhost:9200/_snapshot/repo_name/snapshot_1/_restore" -H 'Content-Type: application/json' -d'
{
  "indices": "index_1",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [
    "index.refresh_interval"
  ]
}
'

```

### 恢复到不同集群

快照中存储的信息不与特定的集群或集群名称绑定。因此, 只要集群版本兼容就可以恢复.

## 参考

[elasticsearch reference 6.4](https://www.elastic.co/guide/en/elasticsearch/reference/6.4/modules-snapshots.html)

[1]: <https://www.elastic.co/guide/en/elasticsearch/plugins/6.4/repository-s3.html> "repository-s3 for S3 repository support"
[2]: <https://www.elastic.co/guide/en/elasticsearch/plugins/6.4/repository-hdfs.html> "repository-hdfs for HDFS repository support in Hadoop environments"
[3]: <https://www.elastic.co/guide/en/elasticsearch/plugins/6.4/repository-azure.html> "repository-azure for Azure storage repositories"
[4]: <https://www.elastic.co/guide/en/elasticsearch/plugins/6.4/repository-gcs.html> "repository-gcs for Google Cloud Storage repositories"
