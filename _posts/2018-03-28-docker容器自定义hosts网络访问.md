---
layout: post
tags: docker hosts
date: 2018-03-28 17:00
title: docker容器自定义hosts网络访问
published: true
---

### 搭建 dnsmaq DNS 服务
```shell
yum install dnsmasq -y

touch /etc/dnsmasq.hosts

# 重启
service dnsmasq restart
```

### 使用 Docker 的 DNS 解决
```shell
vim /etc/docker/daemon.json
```

```json
{
    "dns": ["10.123.12.14", "8.8.8.8"]
}
```

```shell
# 重启
service docker restart
```
