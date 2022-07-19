# k8s部署Mongodb

## 1. helm3安装部署 Mongodb 数据库

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo mongodb

bitnami/mongodb                         12.1.20         5.0.9           MongoDB(R) is a relational open source NoSQL da...

mkdir /root/mongodb && cd /root/mongodb

helm pull bitnami/mongodb --version 12.1.20
tar -xvf mongodb-*.tgz
cp mongodb/values.yaml ./values-test.yaml
#查看集群 storageclasses
kubectl get storageclasses.storage.k8s.io
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   8d
```

编辑文件

vi values-test.yaml

```yaml
global:
  # 定义 storageClass 使用的类型
  storageClass: "longhorn"

# 定义 mongodb 集群为副本集模式 
architecture: replicaset
useStatefulSet: true
# 单机模式
#architecture: standalone
#useStatefulSet: false


# 启动集群认证功能，设置超级管理员账户密码
auth:
  enabled: true
  rootUser: root
  rootPassword: "root"

# 设置集群数量，3个
replicaCount: 3

# 定义 pod 的 nodeSelector
#nodeSelector: { "node": "middleware" }

# 启用持久化存储，使用 global.storageClass 自动创建 pvc 
persistence:
  enabled: true
  size: 2Gi

service:
  type: NodePort
  nodePorts:
    mongodb: "30217"

```

```shell
#安装
helm install mongodb mongodb -f values-test.yaml

#测试连接
kubectl run --namespace default mongodb-client --rm --tty -i --restart='Never' --env="MONGODB_ROOT_PASSWORD=$MONGODB_ROOT_PASSWORD" --image docker.io/bitnami/mongodb:5.0.9-debian-10-r15 --command -- bash

/$ mongosh admin --host "mongodb" --authenticationDatabase admin -u root -p

#卸载
# helm uninstall mongodb
```

参考：
https://www.cnblogs.com/evescn/p/16313215.html