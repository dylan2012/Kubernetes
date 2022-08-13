# kubernetes1.24+二进制安装(版本二)

## 一、基础环境

### 1.1 概述

#### Master节点

Master节点上面主要由四个模块组成，APIServer,schedule,controller-manager,etcd

1. APIServer: APIServer负责对外提供RESTful的kubernetes API的服务，它是系统管理指令的统一接口，任何对资源的增删该查都要交给APIServer处理后再交给etcd，如图，kubectl(kubernetes提供的客户端工具，该工具内部是对kubernetes API的调用）是直接和APIServer交互的。
2. schedule: schedule负责调度Pod到合适的Node上，如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定。 kubernetes目前提供了调度算法，同样也保留了接口。用户根据自己的需求定义自己的调度算法。
3. controller manager: 如果APIServer做的是前台的工作的话，那么controller manager就是负责后台的。每一个资源都对应一个控制器。而control manager就是负责管理这些控制器的，比如我们通过APIServer创建了一个Pod，当这个Pod创建成功后，APIServer的任务就算完成了。
4. etcd：etcd是一个高可用的键值存储系统，kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。

#### Node节点

每个Node节点主要由两个模板组成：kublet, kube-proxy

1. kube-proxy: 该模块实现了kubernetes中的服务发现和反向代理功能。kube-proxy支持TCP和UDP连接转发，默认基Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响，另外，kube-proxy还支持session affinity。
2. kublet：kublet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护和管理该Node上的所有容器，但是如果容器不是通过kubernetes创建的，它并不会管理。本质上，它负责使Pod的运行状态与期望的状态一致。

![Kubernetes部署（一）：架构及功能说明_calico](https://s1.51cto.com/images/blog/201812/24/6b461755ea1f5a5e9812858399f82b4c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

### 1.2 规划信息

集群规划信息：
|  主机名称   | IP地址  | 说明  |
|  ----  | ----  | ----  |
|master01 | 10.199.10.231 | master节点
|master02 | 10.199.10.232 |master节点
|master03 | 10.199.10.233 |master节点
|node01 |10.199.10.234 |node节点
|node02 |10.199.10.235 |node节点
|node03 |10.199.10.236 |node节点
|master-lb |10.199.10.230:16443 |apiserver vip地址
|harbor |10.199.10.237 |docker私有仓库
|gitlab |10.199.10.241 |gitlab私有仓库

说明：

master节点为3台实现高可用，并且通过nginx进行代理master流量实现高可用，master也可以安装node组件。
系统使用centos7.X

### 1.3 基础环境配置

#### 1.3.1 配置hosts

```shell
#每台主机修改主机名
hostnamectl set-hostname master01
#修改hosts
cat >>/etc/hosts<<EOF
10.199.10.231 master01
10.199.10.232 master02
10.199.10.233 master03
10.199.10.234 node01
10.199.10.235 node02
10.199.10.236 node03
10.199.10.237 harbor
10.199.10.241 gitlab
EOF
```

#### 1.3.2 关闭服务

```shell
#关闭防火墙
systemctl disable --now firewalld
#关闭dnsmasq
systemctl disable --now dnsmasq
#关闭postfix
systemctl  disable --now postfix
#关闭NetworkManager
systemctl disable --now NetworkManager
#关闭selinux
sed -ri 's/(^SELINUX=).*/\1disabled/' /etc/selinux/config
setenforce 0
#关闭swap
sed -ri 's@(^.*swap *swap.*0 0$)@#\1@' /etc/fstab
swapoff -a
```

#### 1.3.3 配置时间同步

方法1：使用ntpdate

```shell
#安装ntpdate，需配置yum源
yum install ntpdate -y
#执行同步，可以使用自己的ntp服务器如果没有
ntpdate time2.aliyun.com
#写入定时任务
crontab -e
*/5 * * * * ntpdate time2.aliyun.com
```

方法2：使用chrony(推荐使用)

```shell
#安装chrony
yum install chrony -y
#在其中一台主机配置为时间服务器
cat /etc/chrony.conf
server time2.aliyun.com iburst   #从哪同步时间
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 10.199.0.0/16  #允许的ntp客户端网段
local stratum 10
logdir /var/log/chrony
#重启服务
systemctl restart chronyd
#配置其他节点从服务端获取时间进行同步
cat /etc/chrony.conf
server 10.199.10.231 iburst
#重启验证
systemctl restart chronyd
chronyc sources -v
^* master01                      3   6    17     5    -10us[ -109us] +/-   28ms  #这样表示正常
```

#### 1.3.4 修改资源限制

```shell
cat > /etc/security/limits.conf <<EOF
*       soft        core        unlimited
*       hard        core        unlimited
*       soft        nproc       1000000
*       hard        nproc       1000000
*       soft        nofile      1000000
*       hard        nofile      1000000
*       soft        memlock     32000
*       hard        memlock     32000
*       soft        msgqueue    8192000
EOF
```

#### 1.3.5 升级系统以及内核

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
#安装新版本的kernel
yum --disablerepo=\* --enablerepo=elrepo-kernel install  kernel-ml.x86_64  -y
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y
yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml-tools.x86_64  -y

grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot 
uname -r
```

#### 1.3.6 系统参数优化

```shell
yum install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="10.199.10.231 10.199.10.232 10.199.10.233 10.199.10.234 10.199.10.235 10.199.10.236"
export SSHPASS=123456
for HOST in $IP;do
     sshpass -e ssh-copy-id -o StrictHostKeyChecking=no $HOST
done

cat >/etc/sysctl.conf<<EOF
net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_probes=10
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.conf.all.rp_filter=0 
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.ip_local_port_range= 45001 65000
net.ipv4.ip_forward=1
net.ipv4.tcp_max_tw_buckets=6000
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_synack_retries=2
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.netfilter.nf_conntrack_max=2310720
net.ipv6.neigh.default.gc_thresh1=8192
net.ipv6.neigh.default.gc_thresh2=32768
net.ipv6.neigh.default.gc_thresh3=65536
net.core.netdev_max_backlog=16384 # 每CPU网络设备积压队列长度
net.core.rmem_max = 16777216 # 所有协议类型读写的缓存区大小
net.core.wmem_max = 16777216
net.ipv4.tcp_max_syn_backlog = 8096 # 第一个积压队列长度
net.core.somaxconn = 32768 # 第二个积压队列长度
fs.inotify.max_user_instances=8192 # 表示每一个real user ID可创建的inotify instatnces的数量上限，默认128.
fs.inotify.max_user_watches=524288 # 同一用户同时可以添加的watch数目，默认8192。
fs.file-max=52706963
fs.nr_open=52706963
kernel.pid_max = 4194303
net.bridge.bridge-nf-call-arptables=1
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
vm.max_map_count = 262144
EOF
#加载ipvs模块
cat >/etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
systemctl enable --now systemd-modules-load.service
#重启
reboot
#重启服务器执行检查
lsmod | grep -e ip_vs -e nf_conntrack
```

#### 1.3.7 安装基础软件

```shell
#基础软件
yum install -y yum-utils device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlibdevel python-devel epel-release openssh-server socat ipvsadm conntrack ntpdate telnet git ipvsadm iotop iftop htop
```

#### 1.3.8 优化journald日志

```shell
mkdir -p /var/log/journal
mkdir -p /etc/systemd/journald.conf.d
cat >/etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 1G
SystemMaxUse=1G
# 单日志文件最大 10M
SystemMaxFileSize=10M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 syslog
ForwardToSyslog=no
EOF
systemctl restart systemd-journald && systemctl enable systemd-journald
```

#### 1.3.9 准备软件包

1.下载kubernetes1.24.+的二进制包
github二进制包下载地址：https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md

wget https://dl.k8s.io/v1.24.3/kubernetes-server-linux-amd64.tar.gz

2.下载etcdctl二进制包
github二进制包下载地址：https://github.com/etcd-io/etcd/releases

wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

3.docker-ce二进制包下载地址
二进制包下载地址：https://download.docker.com/linux/static/stable/x86_64/

这里需要下载20.10.+版本

wget https://download.docker.com/linux/static/stable/x86_64/docker-20.10.14.tgz

4.containerd二进制包下载
github下载地址：https://github.com/containerd/containerd/releases

containerd下载时下载带cni插件的二进制包。

wget https://github.com/containerd/containerd/releases/download/v1.6.6/cri-containerd-cni-1.6.6-linux-amd64.tar.gz

5.下载cfssl二进制包
github二进制包下载地址：https://github.com/cloudflare/cfssl/releases

wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl-certinfo_1.6.1_linux_amd64

6.cni插件下载
github下载地址：https://github.com/containernetworking/plugins/releases

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz

7.crictl客户端二进制下载
github下载：https://github.com/kubernetes-sigs/cri-tools/releases

wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz

## 二、安装容器

以下操作docker-ce与containerd选择一个安装即可，需要在所有运行kubelet的节点都需要安装，在kubernetes1.24版本之后如果使用docker-ce作为容器运行时，需要额外安装cri-docker。

### 2.1 安装containerd

```shell
#所有节点安装
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install containerd -y
systemctl start containerd && systemctl enable containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml
systemctl restart containerd
#sed -i "s#https://registry-1.docker.io#https://registry.cnhangzhou.aliyuncs.com#g" /etc/containerd/config.toml
systemctl status containerd
ctr version
runc -version
#crictl工具
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz
tar -zxvf crictl*.tar.gz -C /usr/local/bin
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
EOF
```

### 2.2 安装docker

```shell
#curl -fsSL https://get.docker.com/ | sh
yum-config-manager --add-repo http://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce --showduplicates | sort -r
yum -y install docker-ce-18.06.3.ce
#启动docker
systemctl enable --now docker.socket  && systemctl enable --now docker.service
#验证
docker info
cat >/etc/docker/daemon.json <<EOF
{
   "exec-opts": ["native.cgroupdriver=systemd"],
   "insecure-registries": ["10.199.10.237","harbor"]
}
EOF
systemctl restart docker
```

安装cri-docker

```shell
#解压安装包
tar xf cri-dockerd-0.2.3.amd64.tgz
#拷贝二进制文件
cp cri-dockerd/* /usr/bin/
#生成service文件
cat >/etc/systemd/system/cri-docker.socket<<EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=%t/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
cat >/etc/systemd/system/cri-docker.service<<EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd:// --network-plugin=cni --pod-infra-container-image=192.168.10.254:5000/k8s/pause:3.7
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

StartLimitBurst=3
StartLimitInterval=60s

LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
#启动
systemctl enable --now cri-docker.socket
systemctl enable --now cri-docker
```

## 三、安装ETCD

### 3.1 安装ETCD 服务

#### 3.1.1签名证书

```shell
#证书工具
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl_1.6.1_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssljson_1.6.1_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.1/cfssl-certinfo_1.6.1_linux_amd64 
chmod +x cfssl_1.6.1_linux_amd64 cfssljson_1.6.1_linux_amd64 cfssl-certinfo_1.6.1_linux_amd64 
mv cfssl_1.6.1_linux_amd64 /usr/bin/cfssl
mv cfssljson_1.6.1_linux_amd64 /usr/bin/cfssljson
mv cfssl-certinfo_1.6.1_linux_amd64  /usr/bin/cfssl-certinfo

mkdir -p ~/ssl/{etcd,k8s}
cd ~/ssl/etcd

#自签CA：
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

#生成证书：
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

#使用自签CA签发Etcd HTTPS证书
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "127.0.0.1",
    "10.199.10.231",
    "10.199.10.232",
    "10.199.10.233",
    "10.199.10.234",
    "10.199.10.235",
    "10.199.10.236",
    "etcd1",
    "etcd2",
    "etcd3"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF

#生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server

```

注：上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。

会生成server.pem和server-key.pem文件。

### 3.2 部署etcd

#### 3.2.1 初始化配置文件

Etcd 是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储，所以先准备一个Etcd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群，可容忍1台机器故障，当然，你也可以使用5台组建集群，可容忍2台机器故障。

下载地址： https://github.com/etcd-io/etcd/releases

三个master节点安装配置etcd

```shell
mkdir /etc/etcd/ssl -p
cd ~
wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz
tar zxvf etcd-v3.5.4-linux-amd64.tar.gz
mv etcd-v3.5.4-linux-amd64/{etcd,etcdctl} /usr/local/bin

#etcd-1
IP=`ifconfig ens33 | awk -rn 'NR==2{print $2}'`
cat > /etc/etcd/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$IP:2380"
ETCD_LISTEN_CLIENT_URLS="https://$IP:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$IP:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://$IP:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.199.10.231:2380,etcd-2=https://10.199.10.232:2380,etcd-3=https://10.199.10.233:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/server-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/server-key.pem"
EOF

#etcd-2
IP=`ifconfig ens33 | awk -rn 'NR==2{print $2}'`
cat > /etc/etcd/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$IP:2380"
ETCD_LISTEN_CLIENT_URLS="https://$IP:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$IP:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://$IP:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.199.10.231:2380,etcd-2=https://10.199.10.232:2380,etcd-3=https://10.199.10.233:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/server-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/server-key.pem"
EOF

#etcd-3
IP=`ifconfig ens33 | awk -rn 'NR==2{print $2}'`
cat > /etc/etcd/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://$IP:2380"
ETCD_LISTEN_CLIENT_URLS="https://$IP:2379"
#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://$IP:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://$IP:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.199.10.231:2380,etcd-2=https://10.199.10.232:2380,etcd-3=https://10.199.10.233:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_KEY_FILE="/etc/etcd/ssl/server-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/etc/etcd/ssl/server.pem"
ETCD_PEER_KEY_FILE="/etc/etcd/ssl/server-key.pem"
EOF
```

ETCD_NAME：节点名称，集群中唯一
ETCD_DATA_DIR：数据目录
ETCD_LISTEN_PEER_URLS：集群通信监听地址
ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
ETCD_INITIAL_ADVERTISE_PEER_URLS：集群通告地址
ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
ETCD_INITIAL_CLUSTER：集群节点地址
ETCD_INITIAL_CLUSTER_TOKEN：集群Token
ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

#### 2.systemd管理etcd

```shell
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/local/bin/etcd --logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

#复制证书
cp ~/ssl/etcd/ca*pem /etc/etcd/ssl
cp ~/ssl/etcd/server*pem /etc/etcd/ssl
#同步etcd所有主机
scp -r /etc/etcd/ssl root@master02:/etc/etcd/
scp -r /etc/etcd/ssl root@master03:/etc/etcd/
scp /usr/lib/systemd/system/etcd.service root@master02:/usr/lib/systemd/system
scp /usr/lib/systemd/system/etcd.service root@master03:/usr/lib/systemd/system

systemctl daemon-reload
systemctl enable --now etcd
systemctl status etcd

#设置全局变量
cat > /etc/profile.d/etcdctl.sh <<EOF
#!/bin/bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS="https://10.199.10.231:2379,https://10.199.10.232:2379,https://10.199.10.233:2379"
export ETCDCTL_CACERT=/etc/etcd/ssl/ca.pem
export ETCDCTL_CERT=/etc/etcd/ssl/server.pem
export ETCDCTL_KEY=/etc/etcd/ssl/server-key.pem
EOF
#生效
source /etc/profile
#验证集群状态
etcdctl member list
etcdctl endpoint health --write-out=table
```

## 四、部署apiserver高可用

### 4.1 安装nginx

```shell
mkdir /var/log/nginx -p
yum install nginx nginx-mod-stream keepalived -y

cat >/etc/nginx/nginx.conf<<EOF 
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections  3000;
}

stream {
    log_format main '\$remote_addr \$upstream_addr - [\$time_local] \$status \$upstream_bytes_sent';
    access_log /var/log/nginx/k8s-access.log main;

    upstream apiservers {
        server 10.199.10.231:6443  max_fails=2 fail_timeout=3s;
        server 10.199.10.232:6443  max_fails=2 fail_timeout=3s;
        server 10.199.10.233:6443  max_fails=2 fail_timeout=3s;
    }

    server {
        listen 16443;
        proxy_connect_timeout 1s;
        proxy_pass apiservers;
    }
}
EOF

systemctl daemon-reload
systemctl enable --now nginx.service
#验证
ss -ntl | grep 16443
```

### 4.2 安装keepalive

主keepalived 配置文件

```shell
cat >/etc/keepalived/keepalived.conf<<EOF 
global_defs {
   router_id NGINX
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
}

vrrp_script check_nginx {
     script "/etc/keepalived/check_nginx.sh"
     interval 2
     weight 50
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33 # 修改为实际网卡名
    virtual_router_id 51 # 路由 ID 实例，每个实例是唯一的
    priority 100 # 优先级，备服务器设置 90
    advert_int 1 # 指定 VRRP 心跳包通告间隔时间，默认 1 秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.199.10.230 #虚拟IP地址
    }
    track_script {
      check_nginx
    }
}
EOF

cat >/etc/keepalived/check_nginx.sh<<EOF
#!/bin/bash
count=\$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|\$\$")
if [ "\$count" -eq 0 ];then
     systemctl stop keepalived
fi
EOF

chmod +x /etc/keepalived/check_nginx.sh
```

备 keepalive

```shell
cat >/etc/keepalived/keepalived.conf<<EOF 
global_defs {
   router_id NGINX
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
}

vrrp_script check_nginx {
     script "/etc/keepalived/check_nginx.sh"
     interval 2
     weight 50
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33 # 修改为实际网卡名
    virtual_router_id 51 # 路由 ID 实例，每个实例是唯一的
    priority 90 # 优先级，备服务器设置 90
    advert_int 1 # 指定 VRRP 心跳包通告间隔时间，默认 1 秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.199.10.230 #虚拟IP地址
    }
    track_script {
      check_nginx
    }
}
EOF

cat >/etc/keepalived/check_nginx.sh<<EOF
#!/bin/bash
count=\$(ps -ef |grep nginx | grep sbin | egrep -cv "grep|\$\$")
if [ "\$count" -eq 0 ];then
     systemctl stop keepalived
fi
EOF

chmod +x /etc/keepalived/check_nginx.sh
```

主备启动服务

```shell
#添加执行用户
groupadd -r keepalived_script
useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script

systemctl daemon-reload
systemctl enable --now keepalived
systemctl status keepalived
#systemctl restart keepalived
#测试 vip 是否绑定成功
ip addr
```

## 五、部署 kubernetes

### 5.1 软件下载

从Github下载二进制文件
下载地址：
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md
注：打开链接你会发现里面有很多包，下载一个server包就够了，包含了Master和Worker Node二进制文件。

```shell
#基础包
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
cd ~
wget https://dl.k8s.io/v1.24.3/kubernetes-server-linux-amd64.tar.gz
tar -zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kubectl /usr/bin/
cp kubectl /usr/local/bin/
```

分发二进制文件

```shell
master="master01 master02 master03"
node="node01 node02 node03"
for i in $master;do
  scp {kubeadm,kube-apiserver,kube-controller-manager,kube-scheduler,kube-proxy,kubelet,kubectl} $i:/opt/kubernetes/bin
done
#分发node组件
for i in $node;do
  scp {kube-proxy,kubelet} $i:/opt/kubernetes/bin
done
```

### 5.2 生成kubernetes集群所需证书

#### 5.2.1 CA证书制作

```shell
cd ~/ssl/k8s
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

#生成证书：
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
```

会生成ca.pem和ca-key.pem文件。

#### 5.2.2 kube-apiserver证书

```shell
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.200.0.1",
      "10.199.10.231",
      "10.199.10.232",
      "10.199.10.233",
      "10.199.10.234",
      "10.199.10.235",
      "10.199.10.236",
      "10.199.10.237",
      "10.199.10.238",
      "10.199.10.239",
      "10.199.10.230",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
```

注：上述文件hosts字段中IP为所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP。
会生成server.pem和server-key.pem文件。

#### 5.2.3 创建kube-controller-manager证书与认证文件

```shell
cd ~/ssl/k8s

# 创建证书请求文件
cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing", 
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

#生成kubeconfig文件
KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
KUBE_APISERVER="https://10.199.10.230:16443"

kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

#### 5.2.4 生成kube-scheduler证书文件

```shell
cd ~/ssl/k8s
cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

#生成kubeconfig文件：
KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
KUBE_APISERVER="https://10.199.10.230:16443"

kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

#### 5.2.5 启用 TLS Bootstrapping 机制

```shell
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
#格式：token，用户名，UID，用户组
cat > /opt/kubernetes/cfg/token.csv << EOF
6e80d5442227f1ae80a6957df007c4c2,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
KUBE_APISERVER="https://10.199.10.230:16443" # apiserver IP:PORT
TOKEN="6e80d5442227f1ae80a6957df007c4c2" # 与token.csv里保持一致

#在master节点 生成 kubelet bootstrap kubeconfig 配置文件
kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

#### 5.2.6 生成kube-proxy.kubeconfig文件

```shell
cd ~/ssl/k8s
# 创建证书请求文件
cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
#生成kubeconfig文件：
KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
KUBE_APISERVER="https://10.199.10.230:16443"

kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

#### 5.2.7 分发证书和配置文件

```shell
cd ~/ssl/k8s
master="master01 master02 master03"
node="node01 node02 node03"
for i in $master;do
  scp {ca.pem,ca-key.pem,server.pem,server-key.pem} $i:/opt/kubernetes/ssl
  scp /opt/kubernetes/cfg/{bootstrap.kubeconfig,kube-proxy.kubeconfig,token.csv} $i:/opt/kubernetes/cfg
done
#分发node
for i in $node;do
  scp ca.pem $i:/opt/kubernetes/ssl
  scp /opt/kubernetes/cfg/{bootstrap.kubeconfig,kube-proxy.kubeconfig} $i:/opt/kubernetes/cfg
done
```

### 5.3 安装kube-apiserver

```shell
IP=`ifconfig ens33 | awk -rn 'NR==2{print $2}'`
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://10.199.10.231:2379,https://10.199.10.232:2379,https://10.199.10.233:2379 \\
--bind-address=$IP \\
--secure-port=6443 \\
--advertise-address=$IP \\
--allow-privileged=true \\
--service-cluster-ip-range=10.200.0.0/16 \\
--enable-admission-plugins=NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--service-account-issuer=https://kubernetes.default.svc.cluster.local \\
--service-account-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/etc/etcd/ssl/ca.pem \\
--etcd-certfile=/etc/etcd/ssl/server.pem \\
--etcd-keyfile=/etc/etcd/ssl/server-key.pem \\
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \\
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \\
--requestheader-allowed-names=kubernetes \\
--requestheader-extra-headers-prefix=X-Remote-Extra- \\
--requestheader-group-headers=X-Remote-Group \\
--requestheader-username-headers=X-Remote-User \\
--enable-aggregator-routing=true \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```

注：上面两个\ \ 第一个是转义符，第二个是换行符，使用转义符是为了使用EOF保留换行符。
• --logtostderr：启用日志
• ---v：日志等级
• --log-dir：日志目录
• --etcd-servers：etcd集群地址
• --bind-address：监听地址
• --secure-port：https安全端口
• --advertise-address：集群通告地址
• --allow-privileged：启用授权
• --service-cluster-ip-range：Service虚拟IP地址段
• --enable-admission-plugins：准入控制模块
• --authorization-mode：认证授权，启用RBAC授权和节点自管理
• --enable-bootstrap-token-auth：启用TLS bootstrap机制
• --token-auth-file：bootstrap token文件
• --service-node-port-range：Service nodeport类型默认分配端口范围
• --kubelet-client-xxx：apiserver访问kubelet客户端证书
• --tls-xxx-file：apiserver https证书
• 1.20版本必须加的参数：--service-account-issuer，--service-account-signing-key-file
• --etcd-xxxfile：连接Etcd集群证书
• --audit-log-xxx：审计日志
• 启动聚合层相关配置：--requestheader-client-ca-file，--proxy-client-cert-file，--proxy-client-key-file，--requestheader-allowed-names，--requestheader-extra-headers-prefix，--requestheader-group-headers，--requestheader-username-headers，--enable-aggregator-routing

启用 TLS Bootstrapping 机制
TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。

```shell
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart kube-apiserver 
systemctl enable kube-apiserver
systemctl status kube-apiserver
```

### 5.4 安装kube-controller-manager

```shell
#创建配置文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.200.0.0/16 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--cluster-signing-duration=87600h0m0s"
EOF

#systemd管理controller-manager
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart kube-controller-manager
systemctl enable kube-controller-manager
systemctl status kube-controller-manager
```

### 5.5 安装kube-scheduler

```shell
#创建配置文件
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect \\
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \\
--bind-address=127.0.0.1"
EOF
#systemd管理scheduler
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart kube-scheduler
systemctl enable kube-scheduler
systemctl status kube-scheduler
```

### 5.6 master节点配置kubectl工具

生成kubectl连接集群的证书

```shell
cd ~/ssl/k8s/
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

生成kubeconfig文件

```shell
KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://10.199.10.230:16443"

kubectl config set-cluster kubernetes \
  --certificate-authority=./ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

#授权kubelet-bootstrap用户允许请求证书
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap

#通过kubectl工具查看当前集群组件状态
kubectl get cs

Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
etcd-2               Healthy   {"health":"true","reason":""}
etcd-1               Healthy   {"health":"true","reason":""}
```

### 5.7 同步其它master节点

```shell
master2="master02 master03"
for i in $master2;do
  ssh $i "mkdir /root/.kube -p"
  scp /root/.kube/config $i:/root/.kube/
  scp -r /opt/kubernetes/cfg $i:/opt/kubernetes
  scp /usr/lib/systemd/system/{kube-apiserver.service,kube-controller-manager.service,kube-scheduler.service} $i:/usr/lib/systemd/system/
done
#其它master修改配置文件
vi /opt/kubernetes/cfg/kube-apiserver.conf
#后面的内容改为当前主机的IP
#--bind-address=
#--advertise-address=

systemctl daemon-reload
systemctl enable --now kube-apiserver.service
systemctl enable --now kube-controller-manager.service
systemctl enable --now kube-scheduler.service

systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```

### 5.8 授权apiserver访问kubelet

应用场景：例如kubectl logs

```shell
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

### 5.9 部署kubelet

node节点创建配置文件 #注意不同节点主机名修改--hostname-override=

```shell
HOSTNAME=`hostname`
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=$HOSTNAME \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--container-runtime=remote  \\
--runtime-request-timeout=15m  \\
--container-runtime-endpoint=unix:///run/containerd/containerd.sock  \\
--cgroup-driver=systemd \\
--node-labels=node.kubernetes.io/node='' \\
--feature-gates=IPv6DualStack=true"
EOF

#配置参数文件
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: systemd
clusterDNS:
- 10.200.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

#systemd管理kubelet
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now kubelet.service
# systemctl restart kubelet
# systemctl status kubelet
```

批准kubelet证书申请并加入集群

```shell
kubectl get csr

NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-HiE30nQZo_oXPdE7gDkb7vSJDydoE1GxsAr_NPoUkcU   11m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending

# 批准申请
kubectl certificate approve node-csr-HiE30nQZo_oXPdE7gDkb7vSJDydoE1GxsAr_NPoUkcU

# 查看节点
kubectl get node
```

### 5.10 部署kube-proxy

work节点创建配置文件 #不同节点的主机名修改 hostnameOverride:

```shell
HOSTNAME=`hostname`
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF

cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: $HOSTNAME
clusterCIDR: 10.244.0.0/16
mode: ipvs
ipvs:
  scheduler: "rr"
iptables:
  masqueradeAll: true
EOF

cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now kube-proxy.service
# systemctl restart kube-proxy
# systemctl status kube-proxy

#允许master节点调度
kubectl taint nodes master01 node-role.kubernetes.io/master-
#不允许master节点调度
kubectl taint nodes master01 node-role.kubernetes.io/master=:NoSchedule
```

## 六、安装其他组件

### 6.1 安装calico网络插件

master节点执行

```shell
curl -Lsk https://docs.projectcalico.org/manifests/calico.yaml>calico.yaml 
vim +4434 calico.yaml
...
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"

kubectl apply -f calico.yaml
#查看集群状态
kubectl get pods -n kube-system
#calico 的 STATUS 状态是 Ready，说明 k8s 集群正常运行了

#安装calicoctl工具
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.15.5/calicoctl
chmod +x calicoctl 
mv calicoctl /usr/bin/
#配置calicoctl
mkdir /etc/calico -p
cat >/etc/calico/calicoctl.cfg <<EOF
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/root/.kube/config"
EOF
#验证
calicoctl node status

Calico process is running.
```

### 6.2 安装cilium网络插件(大型集群使用)

```shell
#下载客户端工具
wget https://github.com/cilium/cilium-cli/releases/download/v0.9.3/cilium-linux-amd64.tar.gz

#解压
tar xf cilium-linux-amd64.tar.gz -C /usr/bin/

#使用命令安装
cilium install
#查看运行状态
cilium status

    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium             Running: 2
                  cilium-operator    Running: 1
Cluster Pods:     23/46 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.10.5: 2
                  cilium-operator    quay.io/cilium/operator-generic:v1.10.5: 1
```

安装hubble组件

```shell
cilium hubble enable

🔑 Found existing CA in secret cilium-ca
✨ Patching ConfigMap cilium-config to enable Hubble...
♻️  Restarted Cilium pods
⌛ Waiting for Cilium to become ready before deploying other Hubble component(s)...
🔑 Generating certificates for Relay...
✨ Deploying Relay from quay.io/cilium/hubble-relay:v1.10.5...
⌛ Waiting for Hubble to be installed...
✅ Hubble was successfully enabled!
#启用hubble-ui
cilium hubble enable --ui

#查看状态
cilium status

#修改service
kubectl edit service -n kube-system hubble-ui

spec:
  clusterIP: 10.200.40.172
  ports:  
  - port: 80
    protocol: TCP
    targetPort: 8081
    nodePort: 30002  #添加
  selector:
    k8s-app: hubble-ui
  sessionAffinity: None
  type: NodePort  #修改
```

浏览器打开 http://10.199.10.231:30002/

### 6.3 安装coredns组件

https://github.com/coredns/deployment/tree/master/kubernetes

```shell
#helm 安装
helm repo add coredns https://coredns.github.io/helm
helm --namespace=kube-system install coredns coredns/coredns

#kubectl 安装
git clone https://github.com/coredns/deployment.git
mv deployment coredns
cd coredns/kubernetes

export CLUSTER_DNS_SVC_IP="10.200.0.2"
export CLUSTER_DNS_DOMAIN="cluster.local"

./deploy.sh -i ${CLUSTER_DNS_SVC_IP} -d ${CLUSTER_DNS_DOMAIN} | kubectl apply -f -
#./deploy.sh 10.3.0.0/24 | kubectl apply -f -
#kubectl delete --namespace=kube-system deployment kube-dns
#kubectl apply -f coredns.yaml

#测试网络是否正常
kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh

/ # ping www.baidu.com
PING www.baidu.com (39.156.66.18): 56 data bytes
64 bytes from 39.156.66.18: seq=0 ttl=127 time=39.3 ms

#通过上面可以看到能访问网络，说明 calico 网络插件已经被正常安装了

/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.200.0.2
Address 1: 10.200.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.200.0.1 kubernetes.default.svc.cluster.local
#10.200.0.2 就是我们 coreDNS 的 clusterIP，说明 coreDNS 配置好了。

#创建一个nginx 测试
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get deploy,svc,pod 
```

### 6.4 安装helm3

https://github.com/helm/helm/releases  选择对应的版本

```shell
wget https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
tar xf helm-v3.9.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
#helm repo add aliyun https://kubernetes.oss-cnhangzhou.aliyuncs.com/charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add google https://charts.helm.sh/stable
helm repo update
```

### 6.5 安装dashboard

https://github.com/kubernetes/dashboard

```shell
curl -Lsk https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml>dashboard.yaml
#修改dashboard.yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort  #添加
  ports:
    - port: 443 
      targetPort: 8443
      nodePort: 30001  #添加


kubectl apply -f dashboard.yaml
#验证
kubectl get pods -n kubernetes-dashboard
kubectl get pods,svc -n kubernetes-dashboard

#创建用户
cat >admin.yaml<<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
kubectl apply -f admin.yaml
kubectl describe secrets -n kubernetes-dashboard admin-user

#另一种创建token 
#kubectl -n kubernetes-dashboard create token admin-user


#查看 dashboard 前端的 service
#kubectl get svc -n kubernetes-dashboard
#修改 service type 类型变成 NodePort
#kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
#把 type: ClusterIP 变成 type: NodePort，保存退出即可。
```

复制token填入浏览器的
浏览器打开 https://<node节点>:30001
[https://10.199.10.235:30001/](https://10.199.10.235:30001/)

### 6.6 安装Metrics-server

https://github.com/kubernetes-sigs/metrics-server

选择版本

```shell
curl -Lsk https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.1/components.yaml>metrics.yaml
#需要修改配置
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  #添加这行

kubectl apply -f metrics.yaml
#验证
kubectl top nodes 
kubectl top pods 
```

### 6.7 安装ingress控制器

https://github.com/kubernetes/ingress-nginx
https://kubernetes.github.io/ingress-nginx/deploy/

```shell
#1.helm安装
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

#2.kubectl安装
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.2.0/deploy/static/provider/cloud/deploy.yaml

#出现错误 Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": failed to call webhook: Post "https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1/ingresses?timeout=10s": context deadline exceeded
kubectl delete -A ValidatingWebhookConfiguration ingress-nginx-admission
```

### 6.8 安装部署velero备份插件

需要先部署对象存储服务，minio或者ceph。

下载地址：https://github.com/vmware-tanzu/velero

```shell
#先下载镜像，上传到自己的镜像仓库
docker pull velero/velero:v1.7.0
docker pull velero/velero-plugin-for-aws:latest

#部署，准备对象存储认证文件
cat > credentials-velero <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
#部署
velero install \
#--image 10.199.10.237/velero/velero:v1.7.0 \   #velero镜像地址
--provider aws  \   #后端存储插件aws
#--plugins 10.199.10.237/velero/velero-plugin-for-aws:latest \ #插件镜像
--bucket velero \  #对象存储桶名称
--use-restic  \ 
--use-volume-snapshots=false \
--secret-file ./credentials-velero \  #认证文件
--backup-location-config region=us-east-2,s3ForcePathStyle="true",s3Url=http://minio-server.server:9000  #对象存储定义

#验证，没有error即为正常
kubectl logs deployment/velero -n velero | tail
```

### 6.9 部署nfs-provisioner

官方github地址：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner

#### 1.准备nfs服务器

```shell
#创建nfs数据目录
mkdir /nfs
chmod 777 -R /nfs
#所有节点安装nfs工具
yum install nfs-utils -y
#修改配置文件
cat /etc/exports
/nfs *(rw,async,no_all_squash)
#重启服务
systemctl enable --now nfs-server
#exportfs -av
#验证nfs服务
showmount -e 127.0.0.1

Export list for 127.0.0.1:
/nfs *
```

#### 2.在k8s部署nfs-provisioner

```shell
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/deploy
vi deployment.yaml
#修改IP和PATH
- name: NFS_SERVER
  value: 10.199.10.232
- name: NFS_PATH
  value: /nfs

- name: nfs-client-root
  nfs:
    server: 10.199.10.232
    path: /nsf

#修改允许多个副本
spec:
  replicas: 3 #高可用，配置为3个副本
env:
  - name: ENABLE_LEADER_ELECTION  #设置高可用允许选举
    value: "True"

vi class.yaml
#新增class参数
parameters:
  archiveOnDelete: "false"
  pathPattern: "${.PVC.namespace}/${.PVC.name}"
  namePattern: "${.PVC.name}-${.PVC.annotations.volume.beta.kubernetes.io/storage-class}"
  #pathPattern: "${.PVC.namespace}/${.PVC.annotations.nfs.io/storage-path}"

kubectl apply -f rbac.yaml
kubectl apply -f class.yaml
kubectl apply -f deployment.yaml

#验证
kubectl get sc

NAME         PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   k8s-sigs.io/nfs-subdir-external-provisioner   Delete          Immediate           false                  53s
```

## 七、补充内容

### 7.1 配置登录harbor

```shell
#docker
cat >/etc/docker/daemon.json<<EOF
{
   "exec-opts": ["native.cgroupdriver=systemd"],
   "insecure-registries": ["10.199.10.237","harbor"]
}
EOF

systemctl daemon-reload
systemctl restart docker
#containerd
vi /etc/containerd/config.toml
#增加后面
[plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.199.10.237".tls]
          insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.199.10.237".auth]
           username = "admin"
           password = "Harbor12345"

systemctl daemon-reload
systemctl restart containerd
```
