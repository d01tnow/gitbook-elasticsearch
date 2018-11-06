# Curator docker

docker 方式使用 ES-Curator. 定时执行 curator actions.

## Dockerfile

``` Dockerfile
FROM alpine:3.8

LABEL elasticsearch-curator="5.5.4"

RUN set -ex && \
    sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk --no-cache add python py-pip tzdata && \
    ln -fs /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    pip install elasticsearch-curator==5.5.4 && \
    pip install requests-aws4auth==0.9 && \
    apk del py-pip && \
    rm -rf /var/cache/apk/*

COPY entrypoint.sh /entrypoint.sh
RUN set -ex && \
    chmod 744 /entrypoint.sh

WORKDIR /usr/share/curator
VOLUME ["/crontabs", "${WORKDIR}/config"]

ENTRYPOINT [ "/entrypoint.sh" ]

```

entrypoint.sh

``` sh
#!/bin/sh
cat /crontabs/root > /var/spool/cron/crontabs/root
## -f 前台运行, -d 8 日志等级为8, 输出到 stderr(默认输出到 syslog), 通过 docker logs 就可以访问到
/usr/sbin/crond -f -d 8
```

用到的 crontab 文件

``` crontab
30 23 * * * /usr/bin/curator --config /usr/share/curator/config/curator.yml /usr/share/curator/config/create_index.yml
30 2 * * * /usr/bin/curator --config /usr/share/curator/config/curator.yml /usr/share/curator/config/maintain_indices.yml
30 3 * * * /usr/bin/curator --config /usr/share/curator/config/curator.yml /usr/share/curator/config/open_provisionally.yml
```

## 实施

## 常见问题
