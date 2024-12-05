# 简介

本文介绍了如何通过 **Ansible Playbook** 的 **roles** 功能，自动化部署 Kubernetes 集群。本文将重点讲解如何使用离线方式批量安装 Kubernetes 二进制文件及必要的依赖组件，尽可能减少外部网络访问。考虑到 Kubernetes 集群的正常运行，某些插件（如 CoreDNS、Ingress、Dashboard 等）需要从外部镜像仓库拉取相应镜像，因此在完全离线的环境下，可能需要进行额外的处理。此外，本教程还将介绍如何选择适当的 **Kubernetes** 和 **etcd** 版本，并为 `flannel` 网络插件提供了非容器化版本的安装方式。

 **Kubernetes 版本选择**

在本文的部署过程中，支持多个版本的 Kubernetes，用户可以根据需求选择合适的版本。支持的 Kubernetes 版本从 **v1.24.x** 到 **v1.31.x**，同时也会兼容不同版本的 **etcd**。 

## 软件环境准备

- **操作系统**：是基于 CentOS 操作系统编写的。
- **etcd：**https://github.com/etcd-io/etcd/releases
- **证书签发工具CFSSL：**https://github.com/cloudflare/cfssl/releases
- **Containerd：**https://github.com/containerd/containerd/releases
- **Kubernetes：**https://github.com/kubernetes/kubernetes/releases
- **Nginx：**https://nginx.org/en/download.html
- **flanneld：**https://github.com/flannel-io/flannel/releases
- **flannel-cni：**https://github.com/flannel-io/cni-plugin/releases/

## 架构图

![](https://images.boysec.cn/k8s/202111152020.png)

# 部署思路

## 系统初始化

> 1. 关闭selinux，firewalld
> 2. 关闭swap
> 3. 时间同步
> 4. 写hosts
> 5. 优化sysctl

## Etcd集群部署

> 1. 生成etcd证书
> 2. 部署三个etcd集群
> 3. 查看集群状态

## 部署Master

> 1. 生成 kube-apiserver 证书
> 2. 部署kube-apiserver、controller-manager和 scheduler 组件
> 3. 启动TLS Bootstrapping

## 部署Node

> 1. 安装 Containerd
> 2. 部署 kubelet 和 kube-proxy
> 3. 在Master上允许为新Node颁发证书
> 4. 授权kube-apiserver访问kubelet

## 部署插件

> 1. Dashboard
> 2. CoreDNS
> 3. Traefik Controller

# 一键部署脚本

## Ansible安装

[Ansible自动化批量管理入门](https://www.boysec.cn/boy/64009.html)

[Ansible之角色详解](https://www.boysec.cn/boy/28429.html)

**目录结构：**

```shell
[root@ansible ~]$tree -L 2 k8s/
k8s/
├── ansible.cfg
├── group_vars
│   └── all.yml              # 全局环境变量
├── hosts                    # 主机地址
├── multi-install.yml        # 高可用集群部署脚本
├── roles
│   ├── addons               # 部署k8s插件目录
│   ├── common               # 系统初始化目录
│   ├── containerd           # containerd安装部署
│   ├── etcd                 # etcd安装部署
│   ├── ha                   # 高可用集群插件(keepalived\nginx)
│   ├── master               # k8s的master节点
│   ├── nginx                # nginx 代理
│   ├── node                 # k8s的node节点
│   └── tls                  # 证书生成
├── single.yml               # 单Master集群部署脚本
```

代码 GitHub下载：{% psw https://github.com/5279314/ansible-k8s %}

关注公众号回复 {% psw ansible-k8s %} 获取下载地址

## 使用介绍

### 修改所需变量

```shell
[root@ansible ~/k8s]$cat group_vars/all.yml 
# 安装软件存放目录 
software_dir: '/server/tools'
# k8s安装目录
k8s_work_dir: '/opt/kubernetes'
# etcd安装目录
etcd_work_dir: '/opt/etcd'
# etcd 数据存放目录
etcd_data_dir: '/var/lib/etcd'
# 临时软件存放
tmp_dir: '/tmp/k8s'

#CONTAINERD 配置
CONTAINERD_STORAGE_DIR: "/var/lib/containerd"
SANDBOX_IMAGE: "registry.aliyuncs.com/google_containers/pause:3.9"
ENABLE_MIRROR_REGISTRY: true   # 开启镜像加速，不开启下面则不会生效
MIRROR_DIR: "/etc/containerd/registry.d"
DOCKER_REPO:                   # 定义镜像加速地址
  - docker.io
  - XXX.XXX.XXX

# 集群网络
service_cidr: '192.168.0.0/16'
localhost_dns: '223.5.5.5'
cluster_dns: '192.168.0.2'
pod_cidr: "172.17.0.0/16"
service_nodeport_range: '30000-32767'
cluster_domain: 'cluster.local'
node_pod_ip: "172.17.{{ inventory_hostname.split('.')[3] }}.1/24"

## 代理模式
PROXY_MODE: "ipvs" # iptables 模式下面可以不填写,填写也不生效
IPVS_SCHEDULER: "nq"
ENABLE_IPVS_STRICT_ARP: true
# flanneld 监听网卡 
flanneld_nic: eth0

# 集群TOKEN认证信息
TOKEN: 'bc43e407e311d78b60da186fdd347fc8'

# 高可用，如果部署单Master，该项忽略
vip: '10.1.1.50'
nic: 'eth0'        # 修改为实际内网网卡名
listen: '16443'    # 高可用监听端口

# 自签证书可信任IP列表，为方便扩展，可添加多个预留IP
cert_hosts:
  # 包含所有LB、VIP、Master IP和service_cidr的第一个IP
  k8s:
    - 127.0.0.1
    - 192.168.0.1
    - 10.1.1.50
    - 10.1.1.60
  # 包含所有etcd节点IP
  etcd:
    - 10.1.1.100
    - 10.1.1.120
    - 10.1.1.130

# k8s插件镜像
coredns_images: 'coredns/coredns:1.10.1'
dashboard_images: 'kubernetesui/dashboard:v2.6.1'
traefik_images: 'traefik:v2.9'

# 外部访问域名
web_domain: "od.com"
Traefike_Domain: "traefik.od.com"
Dashboard_Domain: "dashboard.od.com"

# Nginx 安装版本
nginx_ver: nginx-1.26.2
# user configure
Ngx_Group: www
Ngx_User: www
Ngx_Gid: 666
Ngx_Uid: 666

# nginx configure
install_dir: "/usr/local/"
sbin_path: "/usr/sbin/nginx"
conf_path: "/etc/nginx/nginx.conf"
error_log: "/var/log/nginx/error.log"
http_log: "/var/log/nginx/access.log"

# 内核修改
sysctl_file: "/etc/sysctl.d/99-kubernetes-cri.conf"
sysctl_params:
  - "net.ipv4.ip_forward = 1"
  - "net.bridge.bridge-nf-call-iptables = 1"   #将桥接的IPv4流量传递到iptables
  - "net.bridge.bridge-nf-call-ip6tables = 1"  #将桥接的IPv6流量传递到iptables
  - "net.bridge.bridge-nf-call-arptables=1"
  - "net.ipv4.ip_nonlocal_bind = 1"            #允许服务绑定一个本机不存在的IP地址，haproxy部署高可用使用vip时会用到
```

### 修改主机地址

```shell
[root@ansible ~/k8s]$ cat hosts   # 单Master集群部署hosts
# 所有主机通过 node_domain 设置主机名的
[master]
10.1.1.100 node_name=k8s-master node_domain=k8s-master.local
#10.1.1.110 node_name=k8s-master2 node_domain=k8s-master2.local

[node]
10.1.1.120 node_name=k8s-node1 node_domain=k8s-node01.local
10.1.1.130 node_name=k8s-node2 node_domain=k8s-node02.local

[etcd]
10.1.1.100 etcd_name=etcd-1 etcd_domain=etcd1.local
10.1.1.120 etcd_name=etcd-2 etcd_domain=etcd2.local
10.1.1.130 etcd_name=etcd-3 etcd_domain=etcd3.local

[lb]
# 如果部署单Master，该项忽略,填写也不会影响
10.1.1.100 lb_name=lb-master
10.1.1.110 lb_name=lb-backup

[nginx]
10.1.1.10 node_name=nginx node_domain=nginx.local

[k8s:children]
master
node

[newnode]
#192.168.31.74 node_name=k8s-node3
#10.1.1.12 lb_name=lb-master

[all:children]
master
node
nginx
```

### 本地软件存放目录

```shell
[root@ansible ~/k8s]$ ls /server/tools/ -l
total 390368
-rw-r--r-- 1 root root  14891716 Nov 11  2021 cfssl.tar.gz            # 证书生成软件
-rw-r--r-- 1 root root   8295080 Oct 30 16:27 cni-plugin-flannel-linux-amd64-v1.0.1.tar.gz # cni flannel网络
-rw-r--r-- 1 root root 151721930 Oct 31  2024 cri-containerd-cni-1.7.23-linux-amd64.tar.gz # containerd
-rw-r--r-- 1 root root  16171146 Sep 20  2023 etcd-v3.4.27-linux-amd64.tar.gz              # etcd
-rw-r--r-- 1 root root  12320405 Sep 27  2023 flannel-v0.22.3-linux-amd64.tar.gz           # flannel服务网络
-rw-r--r-- 1 root root 195079575 Oct 29 18:21 kubernetes-v1.31.2.tar.gz                    # kubernetes
-rw-r--r-- 1 root root   1244789 Nov  1  2024 nginx-1.26.2.tar.gz                          # nginx
```

### 执行部署命令

```shell
[root@ansible ~/k8s]$ ansible-playbook -i hosts multi-install.yml  # 部署高可用版本
[root@ansible ~/k8s]$ ansible-playbook -i hosts single.yml         # 部署单Master集群版本
```
