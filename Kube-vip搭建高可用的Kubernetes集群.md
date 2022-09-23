# 使用 Kube-vip 搭建高可用的 kubeadm Kubernetes 集群

## 概述

kube-vip 可以在你的控制平面节点上提供一个 Kubernetes 原生的 HA 负载均衡，我们不需要再在外部设置 HAProxy 和 Keepalived 来实现集群的高可用了。kube-vip 是一个为 Kubernetes 集群内部和外部提供高可用和负载均衡的开源项目，在 Vmware 的 Tanzu 项目中已经使用 kube-vip 替换了用于 vSphere 部署的 HAProxy 负载均衡器，本文我们将先来了解 kube-vip 如何用于 Kubernetes 控制平面的高可用和负载均衡功能。

特点
Kube-Vip 最初是为 Kubernetes 控制平面提供 HA 解决方案而创建的，随着时间的推移，它已经发展为将相同的功能合并到 Kubernetes 的 LoadBalancer 类型的 Service 中了。

- VIP 地址可以是 IPv4 或 IPv6
- 带有 ARP（第2层）或 BGP（第3层）的控制平面
- 使用领导选举或 raft 控制平面
- 带有 kubeadm（静态 Pod）的控制平面 HA
- 带有 K3s/和其他（DaemonSets）的控制平面 HA
- 使用 ARP 领导者选举的 Service LoadBalancer（第 2 层）
- 通过 BGP 使用多个节点的 Service LoadBalancer
- 每个命名空间或全局的 Service LoadBalancer 地址池
- Service LoadBalancer 地址通过 UPNP 暴露给网关

kube-vip(<https://kube-vip.io/>) 可以在你的控制平面节点上提供一个 Kubernetes 原生的 HA 负载均衡，我们不需要再在外部设置 HAProxy 和 Keepalived 来实现集群的高可用了。

kube-vip 可以通过静态 pod 运行在控制平面节点上，这些 pod 通过 ARP 会话来识别每个节点上的其他主机，我们可以选择 BGP 或 ARP 来设置负载平衡器，这与 Metal LB 比较类似。在 ARP 模式下，会选出一个领导者，这个节点将继承虚拟 IP 并成为集群内负载均衡的 Leader，而在 BGP 模式下，所有节点都会通知 VIP 地址。

集群中的 Leader 将分配 vip，并将其绑定到配置中声明的选定接口上。当 Leader 改变时，它将首先撤销 vip，或者在失败的情况下，vip 将直接由下一个当选的 Leader 分配。当 vip 从一个主机移动到另一个主机时，任何使用 vip 的主机将保留以前的 vip <-> MAC 地址映射，直到 ARP 过期(通常是30秒)并检索到一个新的 vip <-> MAC 映射，这可以通过使用无偿的 ARP 广播来优化。

kube-vip 可以被配置为广播一个无偿的 arp(可选)，通常会立即通知所有本地主机 vip <-> MAC 地址映射已经改变。

## 安装

### 运行流程

使用 kube-vip 构建高可用 Kubernetes 集群的事件顺序kubeadm如下：

1. 在静态 Pods 清单目录中生成 kube-vip 清单（请参阅下面的生成清单部分）。
2. kubeadm init使用生成静态 Pod 清单时提供的 VIP 地址使用--control-plane-endpoint标志运行。
3. 将kubelet解析并执行所有清单，包括第一步生成的 kube-vip 清单和其他控制平面组件，包括kube-apiserver.
4. kube-vip 启动并公布 VIP 地址。
5. 第kubelet一个控制平面上的 将连接到上一步中通告的 VIP。
6. kubeadm init在第一个控制平面上成功完成。
7. 使用第一个控制平面上命令的输出，在其余kubeadm init控制平面上运行该命令。kubeadm join
8. 将生成的 kube-vip 清单复制到控制平面的其余部分，并放在它们的静态 Pod 清单目录中（默认为/etc/kubernetes/manifests/）

### 静态 Pod安装

```shell
mkdir -p /etc/kubernetes/manifests/ 
export VIP=10.199.10.240
export INTERFACE=ens33
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
#手动设置
export KVVERSION=v0.5.0

#Docker安装
#alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION"

#containerd 安装
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml

```

安装完成k8s后 control-panel节点都需要执行以上命令

### DaemonSet 方式安装（可选）

```shell
#创建 RBAC 设置
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml

mkdir -p /etc/kubernetes/manifests/ 
export VIP=10.199.10.240
export INTERFACE=ens33
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
#手动设置
export KVVERSION=v0.5.0
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection
```

### 初始化k8s

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
controlPlaneEndpoint: "10.199.10.240:6443"
networking:
  dnsDomain: cluster.local
  podSubnet: "10.100.0.0/16"
  serviceSubnet: 10.200.0.0/16
EOF

kubeadm config migrate --old-config kubeadm-config.yaml --new-config new.yaml
#预下载镜像
kubeadm config images pull --config new.yaml
kubeadm init --config new.yaml  --upload-certs

#安装网络插件
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.12.2 \
--namespace kube-system

#控制平面节点
kubeadm join 10.199.10.240:6443 --token hash.hash\
     --discovery-token-ca-cert-hash sha256:hash \
     --control-plane --certificate-key key
#工作节点
kubeadm join 10.199.10.240:6443 --token hash.hash\
    --discovery-token-ca-cert-hash sha256:hash
```

