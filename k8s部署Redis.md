# k8s部署Redis

## 1. helm3安装部署 Redis 数据库

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis

bitnami/redis                           16.12.2         6.2.7           Redis(R) is an open source, advanced key-value ...

mkdir -p /root/redis/ && cd /root/redis/

helm pull bitnami/redis --version 16.12.2
tar -xvf redis-*.tgz
cp redis/values.yaml ./values-test.yaml
#查看集群 storageclasses
kubectl get storageclasses.storage.k8s.io
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   8d
```

编辑文件

vi values-test.yaml

```yaml
global:
  # 全局定义 storageClass 使用 openebs
  storageClass: "longhorn"
  # 定义 redis 集群认证密码
  redis:
    password: "redis123"


fullnameOverride: "redis"


# 定义集群的模式  Allowed values: `standalone` or `replication`
architecture: replication


# redis 服务配置定义
commonConfiguration: |-
  # Enable AOF https://redis.io/topics/persistence#append-only-file
  appendonly yes
  # Disable RDB persistence, AOF persistence already enabled.
  save ""


# master 节点配置信息
master:
  containerPorts:
    redis: 6379

  kind: StatefulSet

  persistence:
    enabled: true
    accessModes:
      - ReadWriteOnce
    size: 1Gi

  service:
    type: ClusterIP
    ports:
      redis: 6379


# replica 节点配置信息
replica:
  replicaCount: 1 #从节点数量
  containerPorts:
    redis: 6379

  persistence:
    enabled: true
    storageClass: ""
    accessModes:
      - ReadWriteOnce
    size: 1Gi

  service:
    type: ClusterIP
    ports:
      redis: 6379


# sentinel 节点配置信息
sentinel:
  enabled: true
  containerPorts:
    sentinel: 26379

  persistence:
    enabled: true
    storageClass: ""
    accessModes:
      - ReadWriteOnce
    size: 100Mi

  service:
    type: ClusterIP
    ports:
      redis: 6379
      sentinel: 26379
```

```shell
#安装
helm install redis redis -f values-test.yaml
helm list
#测试连接
export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)
kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:6.2.7-debian-11-r3 --command -- sleep infinity

kubectl exec --tty -i redis-client \
   --namespace default -- bash

REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis -p 6379 # Read only operations
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis -p 26379 # Sentinel access
kubectl port-forward --namespace default svc/redis 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379
#卸载
# helm uninstall redis
```

参考：
https://www.cnblogs.com/evescn/p/16342899.html