# Ubuntu server22.04 k8s 1.24.3 安装笔记

## 初始化

```shell
#root用户下
#时间同步
apt-get install ntpdate -y
ntpdate time2.aliyun.com
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

#允许 iptables 检查桥接流量
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

#关闭swap
swapoff -a # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab # 永久
```

## 安装docker containerd

```shell
#安装docker
apt-get install docker.io
systemcd enable docker
systemcd start docker
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": [ "native.cgroupdriver=systemd" ]
}
EOF
systemctl daemon-reload
systemctl restart docker

#削除旧的版本
apt-get remove docker docker-engine docker.io containerd runc
apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get install containerd.io docker-ce docker-ce-cli

mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep 'SystemdCgroup' -B 11 /etc/containerd/config.toml
systemctl daemon-reload
systemctl restart containerd.service
```

## 安装 K8S

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update && apt-get install -y apt-transport-https

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
#添加 Kubernetes apt 仓库
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

#安装k8s
apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 部署Kubernetes Master

```shell
kubeadm config print init-defaults > kubeadm-config.yaml
#修改advertiseAddress
kubeadm init --config kubeadm-init.yaml
```

其它内容参考centos7安装
