# Curator

## 介绍

curator 是 elastic 官方用于日常维护 elasticsearch 的工具.

## 基本使用方法

### 命令行方式

curator [--config CONFIG.YML] [--dry-run] ACTION_FILE.YML

参数 | 说明
---- | ----
config  | 用于指明配置文件路. 没有 --config 选项时, curator 会查找 ~/.curator/curator.yml
dry-run | 用于模拟执行 actions.
help    | 用于查看帮助
version | 查看版本

## 配置

### 环境变量

可以在"配置文件"和"action"文件中使用环境变量. 使用方法: ${VAR} 或 指定默认值: ${VAR:default_value}

v5.5 版本不支持环境变量作为文本的一部分. 比如: logfile: ${LOGPATH}/extra/path/information/file.log

### 配置文件

curator 的配置文件是 yaml 格式的文件.

```yml
---
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
client:
  hosts:
    - 127.0.0.1
  port: 9200
  url_prefix:
  use_ssl: False
  certificate:
  client_cert:
  client_key:
  ssl_no_validate: False
  http_auth:
  timeout: 30
  master_only: False

logging:
  loglevel: INFO
  logfile: /var/log/curator/curator.log
  logformat: default
  blacklist: ['elasticsearch', 'urllib3']
```

client
key    | 说明
-------| ----
hosts  | 指明 ES 服务的 host, 或者 host:port. 字符串数组.
port   | ES 服务的 port. 如果多个服务的端口不一致, 使用 hosts 指定. 默认值: 9200
url_prefix | url 中 host 后面的部分. 比如 url 为: http://example.com/elasticsearch/ , url_prefix 为: elasticsearch. 默认值: 空字符串
use_ssl | 是否使用 SSL. 默认值: False
certificate | CA 证书路径. 默认值: 空字符串
client_cert | 客户端证书(公钥)路径. 默认值: 空字符串
clietn_key  | 客户端私钥路径. 默认值: 空字符串
ssl_no_validate | 设置为 True 时, 禁用 SSL 证书验证. 主要用于自签名证书. 默认值: False
http_auth   | 设置用基础的 HTTP 验证. user:password 方式. 明文传送 user:password, 有安全问题, 不推荐. 默认值: 空字符
timeout     | 设置连接超时. 默认值: 30. 单位: 秒. 可以通过 action 中的 options.timeout_override 设置 action 的超时.
master_only | 设为 True 时, 如果 hosts 为多个值, curator 报错. 默认值: False

logging
key    | 说明
-------| ----
loglevel | 设置日志记录最低级别. 由低到高: DEBUG, INFO, WARNING, ERROR, CRITICAL
logfile  | 设置日志文件路径. 需要确保 curator 能正常写入. 默认值: 空字符串, 输出到 STDOUT.
logformat | 设置日志格式. 可选: default, json, logstash. 默认值: default.
blacklist | 设置不输出日志的模块. 默认值: ['elasticsearch', 'urllib3']

## 常用 Action

action 配置文件的基本格式:

``` yaml
---
# Remember, leave a key empty if there is no value.  None will be a string,
# not a Python "NoneType"
#
# Also remember that all examples have 'disable_action' set to True.  If you
# want to use this action as a template, be sure to set this to False after
# copying it.
actions:
  1:
    action: ACTION1
    description: OPTIONAL DESCRIPTION
    options:
      option1: value1
      ...
      optionN: valueN
      continue_if_exception: False
      disable_action: True
    filters:
    - filtertype: *first*
      filter_element1: value1
      ...
      filter_elementN: valueN
    - filtertype: *second*
      filter_element1: value1
      ...
      filter_elementN: valueN
  2:
    action: ACTION2
    description: OPTIONAL DESCRIPTION
    options:
      option1: value1
      ...
      optionN: valueN
      continue_if_exception: False
      disable_action: True
    filters:
    - filtertype: *first*
      filter_element1: value1
      ...
      filter_elementN: valueN
    - filtertype: *second*
      filter_element1: value1
      ...
      filter_elementN: valueN
  3:
    action: ACTION3
    ...
  4:
    action: ACTION4
    ...
```

### 关闭索引

下面例子中通过 pattern.prefix 和 age 过滤索引. 满足条件的索引将被关闭.

``` yml
actions:
  1:
    action: close
    description: >-
      Close indices older than 30 days (based on index name), for test-
      prefixed indices.
    options:
      ## 忽略空列表. 如果为 False, 如果过滤后的列表为空, 则作为 Error 记录到日志. curator exit code=1.
      ## 大部分情况下, 用 ignore_empty_list 替代 continue_if_exception
      ignore_empty_list: True
      ## 是否删除关联的别名
      delete_aliases: False
      # TODO: 如果要启用该 action, 注释掉 disable_action: True, 或替换 True 为 False
      disable_action: True
      ## close action 默认的超时时间 180 s.
      timeout_override: 180
    filters:
    - filtertype: pattern
      ## 前缀模式
      kind: prefix
      ## 如果索引为: test-2018.10.02. 需要包含 '-'.
      value: test-

    - filtertype: age
      ## 必选项. 选项有:(name | creation_date | field_stats). name是索引名称. creation_date是索引创建时间
      ## 值为 name 时, curator 将根据 timestring 字段格式从索引名称中解析时间, 用于计算时间差.
      ## 值为 field_stats(不推荐使用). 则必须有 field 和 stats_result 键值对. ES 6.X 以上版本不再使用这个字段.
      source: name
      ## 选项有:(older | younger). 符合条件的将被处理.
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 30
```

### 强制合并 segment

``` yml
actions:
  1:
    action: forcemerge
    description: >-
      forceMerge test-v1- prefixed indices older than 2 days (based on index
      creation_date) to 2 segments per shard.  Delay 120 seconds between each
      forceMerge operation to allow the cluster to quiesce. Skip indices that
      have already been forcemerged to the minimum number of segments to avoid
      reprocessing.
    options:
      ignore_empty_list: True
      max_num_segments: 2
      ## 两次 forceMerging indices 之间延迟的时间. 单位: 秒. 没有默认值.
      delay: 120
      ## forcemerge action 默认超时时间: 21600 s
      timeout_override:
      # TODO: 如果要启用该 action, 注释掉 disable_action: True, 或替换 True 为 False
      #disable_action: True
    filters:
    - filtertype: pattern
      kind: prefix
      value: test-v1-
      exclude:
      ## forcemerge 会跳过已经 merge 的索引, 所以使用 age filter 也没问题.
    - filtertype: age
      source: name
      timestring: '%Y.%m.%d'
      ## 下面几项过滤 2 天(unit_count unit)前的索引
      direction: older
      unit: days
      unit_count: 2
    - filtertype: forcemerged
      max_num_segments: 2
      exclude:
  ```

### 备份索引

``` yml
actions:
  1:
    action: snapshot
    description: >-
      snapshot test-v1 prefixed indices older than 60 days (based on index
      creation_date)
    options:
      ## 仓库名称. 必选
      repository: es_test_bak
      ## snapshot 名称
      ## Leaving name blank will result in the default 'curator-%Y%m%d%H%M%S'
      name:
      ignore_unavailable: True
      wait_for_completion: True
      max_wait: 3600
      wait_interval: 10
      ignore_empty_list: True
      # TODO: 如果要启用该 action, 注释掉 disable_action: True, 或替换 True 为 False
      #disable_action: True

    filters:
    - filtertype: pattern
      kind: prefix
      value: test-v1-
      ## 是否排除符合条件的索引. 选项: (True | False). 默认: False
      exclude:
    - filtertype: period ## 周期过滤
      ## 周期类型: (relative | absolute)
      ## relative - 是相对当前时间(执行时间)
      ## absolute - 是指定时间
      period_type: relative
      source: name
      timestring: '%Y.%m.%d'
      ## range_from == range_to 表示: 在 range_to 前(负数)或后(正数) 过滤 1 unit 个索引.
      ## 下面例子就是 32 周前的一周的索引
      range_from: -32
      range_to: -32
      unit: weeks
      ## unit: weeks 时, 指定周的第一天, 默认: sunday
      week_starts_on:
```

### 删除索引

```yml
actions:
  1:
    action: delete_indices
    description: >-
      Delete metric indices older than 3 days (based on index name), for
      .monitoring-es-6-
      .monitoring-kibana-6-
      .monitoring-logstash-6-
      .watcher_alarms-
      prefixed indices. Ignore the error if the filter does not result in an
      actionable list of indices (ignore_empty_list) and exit cleanly.
    options:
      ignore_empty_list: True
      # TODO: 如果要启用该 action, 注释掉 disable_action: True, 或替换 True 为 False
      disable_action: True
    filters:
    - filtertype: pattern
      kind: regex
      value: '^(\.monitoring-(es|kibana|logstash)-6-|\.watcher_alarms-).*$'

      ## filtertype: age 基于 epoch (默认: 当前 UTC 时间)比较. 所有单位都转换为'秒'.
      ## 时间久远的或周期性的用 filtertype: period 比较好. 避免重复执行.
    - filtertype: age
      ## 必选项. 选项有:(name | creation_date | field_stats). name是索引名称. creation_date是索引创建时间
      ## 值为 name 时, curator 将根据 timestring 字段格式从索引名称中解析时间, 用于计算时间差.
      ## 值为 field_stats(不推荐使用). 则必须有 field 和 stats_result 键值对. ES 6.X 以上版本不再使用这个字段.
      source: name
      ## 选项有:(older | younger). 符合条件的将被处理.
      direction: older
      timestring: '%Y.%m.%d'
      unit: days
      unit_count: 7
```

## 参考

[reference](https://www.elastic.co/guide/en/elasticsearch/client/curator/5.5/index.html)
