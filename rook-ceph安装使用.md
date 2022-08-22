# rook-ceph安装使用

## 概述

Rook 是一个开源云原生存储编排器，为各种存储解决方案提供平台、框架和支持，以与云原生环境进行原生集成。

Rook 将存储软件转变为自我管理、自我扩展和自我修复的存储服务。它通过自动化部署、引导、配置、供应、扩展、升级、迁移、灾难恢复、监控和资源管理来实现这一点。Rook 使用底层云原生容器管理、调度和编排平台提供的设施来履行其职责。

Rook 利用扩展点深度集成到云原生环境中，并为调度、生命周期管理、资源管理、安全、监控和用户体验提供无缝体验。

Ceph是一个高度可扩展的分布式存储解决方案，用于块存储、对象存储和共享文件系统，具有多年的生产部署

## 安装

### 先决条件

Ceph 先决条件
为了配置 Ceph 存储集群，至少需要以下本地存储选项之一：

原始设备（无分区或格式化文件系统）
原始分区（无格式化文件系统）
block模式下存储类中可用的 PV

使用以下命令确认您的分区或设备是否使用文件系统格式化。

```shell
lsblk -f
NAME            FSTYPE      LABEL           UUID                                   MOUNTPOINT
sdb
sda
├─sda2          LVM2_member                 ydIJLG-wn0X-EF3m-0RO3-4vem-JNmR-qijrEf
│ ├─centos-swap swap                        6c2937db-adab-4f48-b530-1d111ac8bea8
│ └─centos-root xfs                         697f7fac-51f9-42ad-a237-a92198f35dcd   /
└─sda1          xfs                         d62ac18b-c30f-4c34-800d-d0eacd32919e   /boot
```

CephFS

如果您要从 Ceph 共享文件系统 (CephFS) 创建卷，建议的最低内核版本为4.17。如果您的内核版本低于 4.17，则不会强制执行请求的 PVC 大小。存储配额只会在较新的内核上实施。

最低版本

Rook 支持Kubernetes v1.17或更高版本

### 部署安装

```shell
git clone --single-branch --branch master https://github.com/rook/rook.git
cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
#检查pod运行情况
kubectl -n rook-ceph get pod

#安装ceph cluster
kubectl create -f cluster.yaml
#检查pod运行情况
kubectl -n rook-ceph get pod
#安装ceph tool
kubectl create -f toolbox.yaml
#进入容器
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
#执行相关命令
ceph status
ceph osd status
ceph df
rados df
```

block存储(被单个pod使用)

```shell
kubectl create -f csi/rbd/storageclass.yaml
```

Shared Filesystem存储(被多个pod共享使用)

```shell
kubectl create -f filesystem.yaml
kubectl create -f csi/cephfs/storageclass.yaml
#查看
kubectl get sc
NAME          PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
rook-cephfs   rook-ceph.cephfs.csi.ceph.com                 Delete          Immediate           true                   49m
```

配置自动扩容

```shell
vim operator.yaml
ROOK_ENABLE_DISCOVERY_DAEMON: "true"
```

配置并登陆 Ceph Dashboard

```shell
#默认是开启名https
kubectl create -f dashboard-external-https.yaml
#获取8443映射的端口
kubectl -n rook-ceph get service
#获取密码，默认用户名为admin
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

打开 https://nodeIP:8443映射的端口

## 补充

参考文档：
rook 官方文档 ： <https://rook.github.io/docs/rook/v1.9/Getting-Started/intro/>
Kubernetes使用Rook部署Ceph存储集群：<https://www.cnblogs.com/bugutian/p/13092189.html>
Docker安装Ceph集群：<https://blog.baipengfei.com/blogs/linux/2021/10/linux-ceph-install.html#%E5%88%9B%E5%BB%BAceph%E7%9B%AE%E5%BD%95>