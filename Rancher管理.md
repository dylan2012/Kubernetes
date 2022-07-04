# Rancher

## 1.Rancher 介绍

Rancher 是一个开源的企业级多集群 Kubernetes 管理平台，实现了 Kubernetes 集群在混合云+本地数据中心的集中部署与管理，以确保集群的安全性，加速企业数字化转型。
Rancher 官方文档： <https://docs.rancher.cn/>

## 2 安装 rancher

```shell
hostnamectl set-hostname rancher

#配置 hosts 文件
#配置 rancher 到 k8s 主机互信
ssh-keygen -f /root/.ssh/id_rsa -P ''
ssh-copy-id master01

systemctl stop firewalld ; systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
swapoff -a
sed -ri 's@(^.*swap *swap.*0 0$)@#\1@' /etc/fstab
# 其它初始化参考 k8s初装

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io -y
systemctl start docker && systemctl enable docker.service
tee /etc/docker/daemon.json << EOF
{
    "registry-mirrors":["https://rsbud4vc.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload && systemctl restart docker && systemctl status docker

docker run -d --restart=unless-stopped -p 80:80 -p 443:443 --privileged rancher/rancher:latest
docker ps | grep rancher
```

https://<SERVER_IP>
添加集群-导入-根据提示执行命令

成功后安装监控和日志系统
