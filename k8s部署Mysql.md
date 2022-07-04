# k8s部署mysql

## 1. helm3安装部署 Mysql 数据库

```shell
kubectl create ns mysql

helm repo list
NAME            URL
google          https://charts.helm.sh/stable

helm search repo mysql
helm fetch google/mysql --version 1.6.9
#解压
tar xvf mysql-*.tgz
vi mysql/values.yaml
#新增内容
mysqlRootPassword: qaz123
mysqlUser: lr365_admin
mysqlPassword: qaz123
mysqlDatabase: lr365_admin
#修改内容
service:
  type: NodePort
  nodePort: 31306

persistence:
  size: 8Gi #分配磁盘大小

metrics:
  enabled: true #开户prom监控

helm install -n mysql mysql mysql
#第一个 mysql 是命名空间，第二个是 helm release 名，第三个是解压缩目录
#查看部署
helm list -n mysql

cat > mysql-pv.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: "mysql-pv-10g"
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /data/nfs/mysql
    server: 192.168.6.251
EOF

#创建 pv 
kubectl apply -f mysql-pv.yaml
#查看pod是否正常启动
kubectl get pods -n mysql
#测试连接
kubectl exec -it mysql-74d48db7d4-zvrhp -n mysql -- /bin/bash
mysql -uroot -p

#卸载
#helm uninstall mysql -n mysql
```

## 2. 部署Mysql-Ha(集群) 推荐

```shell
#添加新的远程仓库
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo mysql

mkdir /data/mysql-ha && cd /data/mysql-ha
#拉取 chart 到本地目录
helm pull bitnami/mysql --version 9.1.7
tar xvf mysql-*.tgz
cp mysql/values.yaml ./values-test.yaml

#查看集群 storageclasses
kubectl get storageclasses.storage.k8s.io
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   8d
longhorn-test        driver.longhorn.io   Delete          Immediate           true                   8d

```

修改配置values-test.yaml

```yaml
architecture: replication #standalone单机 replication 集群
image:
  debug: true #打开日志
auth:
  rootPassword: "qaz123" # 设置 MySQL root密码
  replicationPassword: "replicator" # 同步密码
primary:
  service:
    nodePorts:
      mysql: "31307" #读写入口
    type: NodePort
  port: 3306
  persistence :
    size: 5Gi
    storageClass: "longhorn"   # 设置 storageClass
secondary:
  replicaCount: 1
  service:
    nodePorts:
      mysql: "31308" #读入口
    type: NodePort
  port: 3306
  persistence :
    size: 5Gi
    storageClass: "longhorn"   # 设置 storageClass

```

```shell
#安装
helm install mysql-cluster mysql -f values-test.yaml

#测试连接
kubectl run mysql-cluster-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.29-debian-11-r3 --namespace default --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

/$ mysql -h mysql-cluster-primary.default.svc.cluster.local -uroot -p
#卸载
# helm uninstall mysql-cluster
```

参考：
https://www.cnblogs.com/evescn/p/16249207.html
https://www.snycloud.com/2021/07/02/kubernetes-%E4%BD%BF%E7%94%A8-helm-%E9%83%A8%E7%BD%B2mysql-ha/
https://developer.aliyun.com/article/760332