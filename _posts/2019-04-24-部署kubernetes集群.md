---
layout: post
tags: kubernetes
date: 2019-04-24 14:00
title: 部署kubernetes集群
published: true
---

#### 安装docker
```zsh
yum update -y;
yum install -y yum-utils device-mapper-persistent-data lvm2;
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo;
yum install -y docker-ce docker-ce-cli containerd.io;
systemctl enable docker;
systemctl start docker;
```

#### 系统调整
```zsh
# 关闭SWAP分区
swapoff -a;
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab;

# 关闭SELinux
setenforce 0;
sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config;

# 关闭dnsmaq
systemctl stop dnsmasq;
systemctl disable dnsmasq;


# 优化内核参数 
cat > k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

cp k8s.conf  /etc/sysctl.d/k8s.conf;
sysctl -p /etc/sysctl.d/k8s.conf;

# 调整系统 TimeZone
timedatectl set-timezone Asia/Shanghai;

# 将当前的 UTC 时间写入硬件时钟
timedatectl set-local-rtc 0;

# 重启依赖于系统时间的服务
systemctl restart rsyslog;
systemctl restart crond;

# 更新系统时间
ntpdate cn.pool.ntp.org

# 关闭无关的服务
systemctl stop postfix && systemctl disable postfix;
```

#### 安装kubeadm, kubelet, kubectl
```zsh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0;
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config;

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes;

systemctl enable --now kubelet;
```

#### 提前下载镜像
```zsh
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.14.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.14.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.14.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.14.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.3.10
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.3.1

docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.14.1 k8s.gcr.io/kube-apiserver:v1.14.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.14.1 k8s.gcr.io/kube-controller-manager:v1.14.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.14.1 k8s.gcr.io/kube-scheduler:v1.14.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.14.1 k8s.gcr.io/kube-proxy:v1.14.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.14.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.14.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.14.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.14.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.3.10
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.3.1
```

#### 启动kubernetes
```zsh
kubeadm init
```

#### 安装weave
```zsh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

#### 解决coredns NOT READY问题
```zsh
kubectl -n kube-system get deployment coredns -o yaml | \
  sed 's/allowPrivilegeEscalation: false/allowPrivilegeEscalation: true/g' | \
  kubectl apply -f -
```
