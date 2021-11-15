## 简介

本文是通过`ansible-playbook`的roles功能实现二进制批量自动安装部署`Kubernetes`集群服务。本想做成离线版本，但由于coredns,ingress,dashboard插件需要拉取镜像，（这里把flannel做成非容器安装版）如需容器版去https://github.com/flannel-io/flannel中获取yaml文件

![](https://images.boysec.cn/k8s/202111152020.png)

## 部署思路

### 系统初始化

> 1. 关闭selinux，firewalld
> 2. 关闭swap
> 3. 时间同步
> 4. 写hosts

### Etcd集群部署

> 1. 生成etcd证书
> 2. 部署三个etcd集群
> 3. 查看集群状态

### 部署Master

> 1. 生成apiserver证书
> 2. 部署apiserver、controller-manager和scheduler组件
> 3. 启动TLS Bootstrapping

### 部署Node

> 1. 安装Docker
> 2. 部署kubelet和kube-proxy
> 3. 在Master上允许为新Node颁发证书
> 4. 授权apiserver访问kubelet

#### 部署插件（准备好镜像）

> 1. Web UI
> 2. CoreDNS
> 3. Ingress Controller

## 一键部署角色

### Ansible安装

[Ansible自动化批量管理入门](https://www.boysec.cn/boy/64009.html)

[Ansible之角色详解](https://www.boysec.cn/boy/28429.html)

**目录结构：**

```shell
[root@ceph01 ~]$tree -L 2 k8s/
k8s/
├── ansible.cfg
├── group_vars
│   └── all.yml
├── hosts
├── roles
│   ├── addons              # 部署k8s插件目录
│   ├── common              # 系统初始化目录
│   ├── docker              # docker安装
│   ├── etcd                # etcd安装
│   ├── master              # master节点
│   ├── nginx               # ingrees代理Nginx
│   ├── node                # node节点
│   └── tls                 # 证书生成
└── single.yml
```
