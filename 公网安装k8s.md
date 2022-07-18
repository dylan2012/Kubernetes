# 公网安装k8s

## 初始化

```shell
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum update -y
yum install -y yum-utils

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.6-3.3.el7.x86_64.rpm
yum install -y docker-ce docker-ce-cli
mkdir -p /etc/docker
cat <<EOF | tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://mirror.ccs.tencentyun.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
systemctl daemon-reload
systemctl enable docker
systemctl start docker
#配置内核参数
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
#sysctl -p /etc/sysctl.d/k8s.conf
```

## k8s安装

```shell
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet
systemctl start kubelet

#自动补全
kubeadm completion bash > /etc/bash_completion.d/kubeadm  
kubectl completion bash > /etc/bash_completion.d/kubectl  
source /usr/share/bash-completion/bash_completion
```

## 创建虚拟网卡

```shell
cd /etc/sysconfig/network-scripts/
cp ifcfg-eth0 ifcfg-eth0:0
vim ifcfg-eth0:0  #编辑IPADDR=${当前节点的IP}


BOOTPROTO=static
DEVICE=eth0:0
IPADDR=101.101.101.101
NETMASK=255.255.255.0
ONBOOT=yes
PERSISTENT_DHCLIENT=yes
TYPE=Ethernet
USERCTL=no
```

## 初始化主节点

```shell
k8s_version="v1.24.2" # k8s版本
apiserver_advertise_address="101.101.101.101" # 节点公网IP
pod_network_cidr="10.244.0.0/16" # pod网段设置

# k8s master init
kubeadm init --kubernetes-version=${k8s_version}  \
--apiserver-advertise-address=${apiserver_advertise_address}   \
--image-repository=registry.aliyuncs.com/google_containers  \
--pod-network-cidr=${pod_network_cidr}  \
--dry-run

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubectl taint nodes --all node-role.kubernetes.io/master- # 干掉主节点的污点

# 生成创建命令

kubeadm join kubernetes:6443 --token r9m0f9.8p0jqlukp43qw9cz --discovery-token-ca-cert-hash sha256:592a680db77d441771cad836ad7f8257307e8ccb65f3b859f747f944de85f53f

kubeadm join kubernetes:6443 --token okpcew.lri3i4cje9ksrk7s --discovery-token-ca-cert-hash sha256:592a680db77d441771cad836ad7f8257307e8ccb65f3b859f747f944de85f53f --ignore-preflight-errors=Swap 

#pod 之间通信
# 检查dns服务
dig -t A mongodb-svc.dev.svc.cluster.local @10.244.0.2

#防火墙
firewall-cmd --zone=public --permanent --add-port=6443/tcp
firewall-cmd --zone=public --permanent --add-port=8010/tcp
firewall-cmd --permanent --add-rich-rule 'rule family=ipv4 source address=172.24.0.0/16'
```
