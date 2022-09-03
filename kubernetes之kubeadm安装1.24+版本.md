# kubernetes之kubeadm安装1.24.+版本

## 一、基础环境准备

集群规划信息：
|  主机名称   | IP地址  | 说明  |
|  ----  | ----  | ----  |
|master01 | 10.199.10.241 | master节点
|master02 |10.199.10.242 |master节点
|master03 |10.199.10.243 |master节点
|node01 |10.199.10.244 |node节点
|node02 |10.199.10.245 |node节点
|master-lb |10.199.10.240:16443 |nginx组件监听地址

说明：

master节点为3台实现高可用，并且通过nginx进行代理master流量实现高可用，master也安装node组件。
node节点为2台
nginx在所有节点安装，监听127.0.0.1:16443端口
系统使用centos7.X

### 1.1 基础环境配置

#### 1.所有节点配置hosts

```shell
#每台主机修改主机名
hostnamectl set-hostname master01
#修改hosts
cat >>/etc/hosts<<EOF
10.199.10.241 master01
10.199.10.242 master02
10.199.10.243 master03
10.199.10.244 node01
10.199.10.245 node02
10.199.10.237 harbor
EOF
```

#### 2.所有节点关闭防火墙、selinux、dnsmasq、swap

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

#### 3.配置时间同步

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
server 10.199.10.241 iburst
#重启验证
systemctl restart chronyd
chronyc sources -v
^* master01                      3   6    17     5    -10us[ -109us] +/-   28ms  #这样表示正常
```

#### 4.所有节点修改资源限制

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

#### 5.ssh认证、升级系统以及内核

```shell
yum install -y sshpass
ssh-keygen -f /root/.ssh/id_rsa -P ''
export IP="10.199.10.241 10.199.10.242 10.199.10.243 10.199.10.244 10.199.10.245"
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

#### 6.安装基础软件

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlibdevel python-devel epel-release openssh-server socat ipvsadm conntrack ntpdate telnet git ipvsadm iotop iftop htop bash-completion
```

#### 7.优化journald日志

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

#### 8.配置kubernetes的yum源

```shell
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#测试
yum list --showduplicates | grep kubeadm
```

### 1.2 安装 containerd 服务

```shell
#所有节点安装
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install containerd -y
#yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
#yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
systemctl start containerd && systemctl enable containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
#sed -i "s#k8s.gcr.io#registry.cnhangzhou.aliyuncs.com/google_containers#g" /etc/containerd/config.toml
sed -i "s#SystemdCgroup = false#SystemdCgroup = true#g" /etc/containerd/config.toml
#sed -i "s#https://registry-1.docker.io#https://registry.cnhangzhou.aliyuncs.com#g" /etc/containerd/config.toml
systemctl restart containerd
systemctl status containerd
ctr version
runc -version
#crictl工具
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.24.2/crictl-v1.24.2-linux-amd64.tar.gz
tar -zxvf crictl*.tar.gz -C /usr/local/bin
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
EOF
#验证
crictl images
```

### 1.3 安装kubernetes组件安装

kubernetes1.24.+版本说明：https://github.com/kubernetes/kubernetes/blob/v1.24.0-alpha.1/CHANGELOG/CHANGELOG-1.24.md

```shell
#所有节点安装kubeadm
yum list kubeadm.x86_64 --showduplicates | sort -r #查看所有版本
#安装
yum install -y kubeadm-1.24.4-0 kubelet-1.24.4-0 kubectl-1.24.4-0 
#重启kubelet
systemctl daemon-reload && systemctl restart kubelet && systemctl enable kubelet && systemctl status kubelet
#设置容器运行时
#crictl config runtime-endpoint /run/containerd/containerd.sock
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

### 1.4 安装高可用组件nginx

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
    log_format main '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    access_log /var/log/nginx/k8s-access.log main;

    upstream apiservers {
        server 10.199.10.241:6443  max_fails=2 fail_timeout=3s;
        server 10.199.10.242:6443  max_fails=2 fail_timeout=3s;
        server 10.199.10.243:6443  max_fails=2 fail_timeout=3s;
    }

    server {
        listen 16443;
        proxy_connect_timeout 1s;
        proxy_pass apiservers;
    }
}
EOF

systemctl enable --now nginx.service
#验证
ss -ntl | grep 16443
LISTEN     0      511    127.0.0.1:16443                    *:*
```

### 1.5 keepalive 配置

主 keepalived

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
        10.199.10.240/24 #虚拟IP地址
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
        10.199.10.240/24 #虚拟IP地址
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
groupadd -r keepalived_script
useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script

systemctl daemon-reload
systemctl start nginx
systemctl start keepalived
systemctl enable nginx keepalived
systemctl status keepalived
#测试 vip 是否绑定成功
ip addr
```

## 二、k8s组件安装

### 2.1 准备kubeadm-config.yaml配置文件

```shell
cat >kubeadm-config.yaml<<EOF
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
---
apiVersion: kubeadm.k8s.io/v1beta3
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: 1.24.4
apiServer:
  certSANs:
  - 127.0.0.1
controlPlaneEndpoint: "10.199.10.240:16443"
networking:
  dnsDomain: cluster.local
  podSubnet: "10.100.0.0/16"
  serviceSubnet: 10.200.0.0/16
EOF

kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
#预下载镜像
kubeadm config images pull --config new.yaml
```

### 2.2 初始化k8s集群

在一个master节点执行即可

```shell
kubeadm init --config new.yaml  --upload-certs

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
#验证
kubectl get nodes
#重新生成加入命令
kubeadm token create --print-join-command
```

### 2.3 初始化master

在master节点执行初始化节点显示的命令

```shell
kubeadm join 10.199.10.240:16443 --token fpuiwu.tm61kndykjmizwdy \
        --discovery-token-ca-cert-hash sha256:b1e9bd1f34f445137377022e7d156644ba122ba0410eab5fb3a1b2f0766e75a3 \
        --control-plane --certificate-key 7b5fc359059ea75dc977f109a16a22319c78d15c63ffe4a0646a54056284667e
```

### 2.4 初始化node节点

在node节点执行初始化节点显示的命令

```shell
kubeadm join 10.199.10.240:16443 --token fpuiwu.tm61kndykjmizwdy \
        --discovery-token-ca-cert-hash sha256:b1e9bd1f34f445137377022e7d156644ba122ba0410eab5fb3a1b2f0766e75a3
```

## 三、其他组件安装

### 3.1 calico网络组件安装

master节点执行

```shell
curl -Lsk https://docs.projectcalico.org/manifests/calico.yaml>calico.yaml 
vi calico.yaml
#修改配置
            - name: CALICO_IPV4POOL_CIDR
              value: "10.100.0.0/16"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=ens,eth"

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

IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 10.199.10.242 | node-to-node mesh | up    | 09:37:48 | Established |
| 10.199.10.245 | node-to-node mesh | up    | 09:37:53 | Established |
| 10.199.10.243 | node-to-node mesh | up    | 09:38:52 | Established |
| 10.199.10.244 | node-to-node mesh | up    | 09:39:13 | Established |
+---------------+-------------------+-------+----------+-------------+
```

测试在 k8s 创建 pod 是否可以正常访问网络

```shell
kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh

/ # ping www.baidu.com
PING www.baidu.com (39.156.66.18): 56 data bytes
64 bytes from 39.156.66.18: seq=0 ttl=127 time=39.3 ms

#通过上面可以看到能访问网络，说明 calico 网络插件已经被正常安装了

/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.200.0.10
Address 1: 10.200.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.200.0.1 kubernetes.default.svc.cluster.local
#10.200.0.10 就是我们 coreDNS 的 clusterIP，说明 coreDNS 配置好了。
```

### 3.2 安装Metrics-server

https://github.com/kubernetes-sigs/metrics-server

选择版本

```shell
curl -Lsk https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml>metrics.yaml
vi metrics.yaml
#需要修改配置
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  #添加这个

kubectl apply -f metrics.yaml
#验证
kubectl top node
```

注意某些版本不正常时

```shell
vim /etc/kubernetes/manifests/kube-apiserver.yaml
#- kube-apiserver后面新增加如下内容：
- --enable-aggregator-routing=true
```


### 3.3 安装dashboard

https://github.com/kubernetes/dashboard

```shell
curl -Lsk https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml>dashboard.yaml
#修改dashboard.yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  type: NodePort  #添加
  ports:
    - port: 8000
      targetPort: 8000
      nodePort: 30001  #添加，如果冲突修改端口


kubectl apply -f dashboard.yaml
#验证
kubectl get pods -n kubernetes-dashboard
#查看 dashboard 前端的 service
#kubectl get svc -n kubernetes-dashboard
#修改 service type 类型变成 NodePort
#kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
#把 type: ClusterIP 变成 type: NodePort，保存退出即可。
```



通过 Token 登陆 dashboard

```shell
kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard
kubectl get secret -n kubernetes-dashboard
#找到对应的带有 token 的 kubernetes-dashboard-token-ppc8c
kubectl describe secret kubernetes-dashboard-token-ppc8c -n kubernetes-dashboard
```

复制token填入浏览器的
浏览器打开 https://<node节点>:30001

### 3.4 安装cilium网络插件（可选）

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

# 浏览器打开 http://10.199.10.241:30002/

```

## 四、一些必要的修改

### 4.1 修改kube-proxy工作模式为ipvs

将Kube-proxy改为ipvs模式，因为在初始化集群的时候注释了ipvs配置，所以需要自行修改一下：

```shell
#在控制节点修改configmap
kubectl edit cm -n kube-system kube-proxy
mode: "ipvs"  #默认没有值是iptables工作模式
#更新kube-proxy的pod
kubectl patch daemonset kube-proxy -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" -n kube-system
#验证工作模式
curl 127.0.0.1:10249/proxyMode
ipvs
```

### 4.2 安装helm3

https://github.com/helm/helm/releases

选择对应的版本

```shell
wget https://get.helm.sh/helm-v3.9.0-linux-amd64.tar.gz
tar xf helm-v3.9.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add google https://charts.helm.sh/stable
helm repo update
```

### 4.3 安装Ingress-nginx

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

## 五 补充内容

### 5.1 测试网络

```shell
kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping www.baidu.com

nslookup kubernetes.default.svc.cluster.local
```

### 5.2 命令

```shell
#重新生成加入节点命令
kubeadm token create --print-join-command
#标志工作节点
kubectl label node node01 node-role.kubernetes.io/worker=worker
#允许master节点调度
kubectl taint nodes master03 node-role.kubernetes.io/control-plane-
kubectl taint nodes master03 node-role.kubernetes.io/master-
#不允许master节点调度
kubectl taint nodes master01 node-role.kubernetes.io/master=:NoSchedule

#新增master节点命令
echo "$(kubeadm token create --print-join-command) --control-plane --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)"
```

### 5.3 配置登录harbor

```shell
#docker
cat /etc/docker/daemon.json
{
   "exec-opts": ["native.cgroupdriver=systemd"],
   "insecure-registries": ["10.199.10.237"]
}
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

#批量上传脚本
cat >1.sh<<EOF
name="
`docker images | grep gcr | awk -v OFS=':' 'NR!=1{print $1,$2}'`
"
host='10.199.10.237'
for var in \$name;do
  tmp=\${var##*/}
  eval new_image_url=\${host}/k8s/\${tmp}
  docker tag \$var \$new_image_url
  docker push \$new_image_url
done
EOF
```
