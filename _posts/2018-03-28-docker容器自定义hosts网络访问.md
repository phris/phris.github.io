---
layout: post
tags: docker hosts dns
date: 2018-03-28 17:00
title: docker容器自定义hosts网络访问
published: true
---

### 搭建 dnsmaq DNS 服务
```shell
# 安装dnsmasq
yum install dnsmasq -y

# 监听对127.0.0.1, 192.168.1.100的请求，192.168.1.100为局域网地址，可继续添加多个
echo 'listen-address=127.0.0.1, 192.168.1.100' >> /etc/dnsmasq.conf
echo 'addn-hosts=/etc/dnsmasq.hosts' >> /etc/dnsmasq.conf

# 添加解析
touch /etc/dnsmasq.hosts
echo '192.168.1.1 www.server.com' >> /etc/dnsmasq.hosts

# 重启
service dnsmasq restart

# 自动启动
systemctl enable dnsmasq

# 开放dns服务端口
firewall-cmd --zone=public --add-port=53/udp --permanent
firewall-cmd --reload
```

<!--more-->

### 使用 Docker 的 DNS 解决
```shell
vim /etc/docker/daemon.json
```

```json
{
    "dns": ["192.168.1.100", "8.8.8.8"]
}
```

```shell
# 重启
service docker restart
```

### centos永久设置dns
```
echo 'dns=none' >> /etc/NetworkManager/NetworkManager.conf
echo 'nameserver 192.168.1.100' > /etc/resolve.conf
echo 'nameserver 8.8.8.8' >> /etc/resolve.conf
systemctl restart NetworkManager.service
```

#### 参考链接
* [Docker使用dnsmasq替代/etc/hosts解析](http://knktc.com/2014/08/16/docker-use-dnsmasq-as-hosts)
* [archlinux:dnsmasq (简体中文)](https://wiki.archlinux.org/index.php/Dnsmasq_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))
* [firewall-cmd](http://wangchujiang.com/linux-command/c/firewall-cmd.html)