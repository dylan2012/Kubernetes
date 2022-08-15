# kubernetes集群备份与恢复

## 一、概述

k8s集群服务所有组件都是无状态服务，所有数据都存储在etcd集群当中，所以为保证k8s集群的安全可以直接备份etcd集群数据，备份etcd的数据相当于直接备份k8s整个集群。

但是备份etcd及备份整个集群，有些场景比如迁移服务，只想备份一个namespace，就无法使用备份etcd的方式来备份，所以我们这里引用velero工具，Velero（以前称为Heptio Ark）可以为您提供了备份和还原Kubernetes集群资源和持久卷的能力，你可以在公有云或本地搭建的私有云环境安装Velero。

## 二、etcd集群备份与恢复

etcd有多个不同的API访问版本，v1版本已经废弃，etcd v2 和 v3 本质上是共享同一套 raft 协议代码的两个独立的应用，接口不一样，存储不一样，数据互相隔离。也就是说如果从 Etcd v2 升级到 Etcd v3，原来v2 的数据还是只 能用 v2 的接口访问，v3 的接口创建的数据也只能访问通过 v3 的接口访问。

### 2.1 etcd v2版本数据备份与恢复

备份数据

```shell
#帮助说明
ETCDCTL_API=2 etcdctl backup --help
NAME:
   etcdctl backup - backup an etcd directory

USAGE:
   etcdctl backup [command options]  

OPTIONS:
   --data-dir value        源数据目录
   --wal-dir value         wal日志源目录
   --backup-dir value      备份到那个目录
   --backup-wal-dir value  备份日志到那个目录
   --with-v3               Backup v3 backend data

#备份
ETCDCTL_API=2 etcdctl backup --data-dir /var/lib/etcd --backup-dir /opt/etcd_backup
file /opt/etcd_backup/member/snap/db 
```

恢复数据

```shell
etcd --help | grep force
  --force-new-cluster 'false'

#恢复数据，恢复时会覆盖 snapshot 的元数据，所以需要启动一个新的集群。
etcd --data-dir=/opt/etcd_backup --force-new-cluster

#修改service文件
vi /etc/systemd/system/etcd.service

--data-dir=/var/lib/etcd  -force-new-cluster \ #强制设置为为新集群
systemctl restart etcd.service
```

### 2.2 etcd v3版本数据备份与恢复

数据备份

一般的k8s集群数据都是存放在etcd的api版本3中所以不需要使用v2版本的备份方式直接使用v3版本，不过有些k8s组件要是版本比较旧可能会使用v2的api，需要提前确认

```shell
export ETCDCTL_ENDPOINTS=https://10.199.10.231:2379 #etcd某台服务器地址
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
file snapshot.db 
```

数据恢复

```shell
export ETCDCTL_ENDPOINTS=https://10.199.10.231:2379 #etcd某台服务器地址
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db --data-dir=/mnt/etcd  #将数据恢复到一个新的不存在的目录中
ls -hl /mnt/etcd/member/snap/db 
```

### 2.3 etcd数据备份脚本

```shell
#!/bin/bash

#备份目录
backup_dir="/root/etcd"
#保持5个最新的备份，其余删除
num=5

#etcdctl配置参数
export ETCDCTL_ENDPOINTS=https://10.199.10.231:2379 #etcd服务器地址
export ETCDCTL_CACERT=/etc/etcd/ssl/ca.pem  #etcd的CA证书
export ETCDCTL_CERT=/etc/etcd/ssl/etcd.pem  #etcd证书
export ETCDCTL_KEY=/etc/etcd/ssl/etcd-key.pem #etcd私钥

source /etc/profile
DATE=`date +%Y-%m-%d-%H-%M-%S`
ETCDCTL_ENDPOINTS=https://10.199.10.231:2379 #etcd服务器地址
ETCDCTL_API=3 etcdctl snapshot save ${backup_dir}/etcd-snapshot-${DATE}.db &>/dev/null && echo "etcd备份完成" || { echo "etcd备份失败";exit; }
cd $backup_dir && tar czf etcd-backup-${DATE}.tar.gz etcd-snapshot-${DATE}.db --remove-files

file_name=`ls -lt ${backup_dir} | awk  'NR!=1{print $9}' | awk 'NR>5{print}'`
cd $backup_dir
for name in ${file_name};do
  rm -f $name
done
```

### 2.4 k8s集群etcd备份恢复流程

```html
1、恢复服务器系统
2、重新部署ETCD集群
3、停止kube-apiserver、controller-manager、scheduler、kubelet、kube-proxy
4、停止ETCD集群
5、各ETCD节点恢复同一份备份数据
6、启动各节点并验证ETCD集群
7、启动kube-apiserver、controller-manager、scheduler、kubelet、kube-proxy
8、验证k8s master状态及pod数据
```

## 三、k8s备份-velero

### 3.1 velero简介

官方网站：https://velero.io/

github网站：https://github.com/vmware-tanzu/velero

Velero（以前称为Heptio Ark）可以为您提供了备份和还原Kubernetes集群资源和持久卷的能力，你可以在公有云或本地搭建的私有云环境安装Velero,可以为你提供以下能力：

备份集群数据，并在集群故障的情况下进行还原
将集群资源迁移到其他集群
将您的生产集群复制到开发和测试集群
Velero包含：

在集群上运行的服务器端
在本地运行的命令行客户端
velero工作原理

每个Velero的操作（如按需备份，计划备份，还原）都是自定义资源，使用Kubernetes 自定义资源定义（CRD）定义并存储在 etcd中，Velero还包括处理自定义资源以执行备份，还原以及所有相关操作的控制器，可以备份或还原群集中的所有对象，也可以按类型，命名空间或标签过滤对象。Velero是kubernetes用来灾难恢复的理想选择，也是在集群上执行系统操作（如升级）之前对应用程序状态进行快照的理想选择。

### 3.2 velero部署

#### 1.安装minio对象存储

```shell
yum install -y docker-compose
mkdir -p /data/docker-compose/minio
cd /data/docker-compose/minio  # 创建工作目录

#如果不支持中文把注释删除
cat > docker-compose.yaml << EOF
version: '3.1'
services: 
    minio:
      image: minio/minio
      container_name: minio
      restart: always
      privileged: true
      network_mode: host
      ports:
        - 9000:9000
      command: server /data  #指定容器中的目录 /data
      environment:
        MINIO_ACCESS_KEY: minio    #管理后台用户名
        MINIO_SECRET_KEY: minio123 #管理后台密码，最小8个字符
      volumes:
        - ./data:/data              #映射当前目录下的data目录至容器内/data目录
        - ./config:/root/.minio/     #映射配置目录
EOF

docker-compose up -d  # 启动容器
docker-compose logs -f  # 启动后观察容器启动情况并获取Console
```

浏览器打开 Console: http://10.199.10.237:53424
登录后台创建Bucket: velero

#### 2.安装velero

```shell
mkdir -p  /data/velero/
cd /data/velero/
#对应minio安装的用户名和密码
cat > credentials-velero << EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```

包下载地址：https://github.com/vmware-tanzu/velero/releases

```shell
wget https://github.com/vmware-tanzu/velero/releases/download/v1.9.0/velero-v1.9.0-linux-amd64.tar.gz
tar xf velero*.tar.gz
mv velero-v1.9.0-linux-amd64/velero /usr/local/bin/
velero version 

Client:
        Version: v1.9.0
        Git commit: 6021f148c4d7721285e815a3e1af761262bff029
<error getting server version: no matches for kind "ServerStatusRequest" in version "velero.io/v1">

cd /data/velero/
velero install \
    --image velero/velero:v1.9.0 \
    --provider aws \
    --bucket velero \
    --namespace velero \
    --secret-file ./credentials-velero \
    --velero-pod-cpu-request 200m \
    --velero-pod-mem-request 200Mi \
    --velero-pod-cpu-limit 1000m \
    --velero-pod-mem-limit 1000Mi \
    --use-volume-snapshots=false \
    --use-restic \
    --restic-pod-cpu-request 200m \
    --restic-pod-mem-request 200Mi \
    --restic-pod-cpu-limit 1000m \
    --restic-pod-mem-limit 1000Mi \
    --plugins velero/velero-plugin-for-aws:v1.2.1 \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://10.199.10.237:9000

#验证是否安装成功
kubectl logs deployment/velero -n velero
```

#### 3.velero启动参数说明

```html
#velero install下参数
--image  velero/velero:v1.9.0  #velero镜像名称，不写默认去官方仓库拉取
--plugins     #插件镜像
--provider      
--bucket         #对象存储bucket名称
--secret-file    #认证文件
--backup-location-config #备份地址配置
--use-restic    #启用restic备份恢复插件，如果使用需要配合标签进行使用
--use-volume-snapshots=false #当在没有Velero支持快照的存储提供商上使用恢复性时，该标志可防止在安装时创建未使用的机标。
```

#### 4.备份测试

```shell
#全部备份
velero backup create all-tidy --exclude-namespaces kube-system,ingress-nginx,cattle-system,velero  # 创建备份，排除不想要的命名空间
#删除备份
velero backup delete all-tidy

#使用示例
cd velero-v1.9.0-linux-amd64/examples
kubectl apply -f nginx-app/base.yaml
velero backup create nginx-backup --include-namespaces nginx-example --default-volumes-to-restic

Backup request "nginx-backup" submitted successfully.
Run `velero backup describe nginx-backup` or `velero backup logs nginx-backup` for more details.

#查看进度
velero backup describe nginx-backup
velero backup logs nginx-backup
```

#### 5.恢复测试

```shell
kubectl delete ns nginx-example
kubectl get ns

#恢复
velero restore create --from-backup nginx-backup

Restore request "nginx-backup-20220815172557" submitted successfully.
Run `velero restore describe nginx-backup-20220815172557` or `velero restore logs nginx-backup-20220815172557` for more details.

#查看恢复
velero restore describe nginx-backup-20220815172557

Name:         nginx-backup-20220815172557
Namespace:    velero
Labels:       <none>
Annotations:  <none>

Phase:                       Completed
Total items to be restored:  10
Items restored:              10

Started:    2022-08-15 17:25:57 +0800 CST
Completed:  2022-08-15 17:25:57 +0800 CST

#查看运行
kubectl get ns
kubectl get pod -n nginx-example

NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-57f59b7b9b-52dkp   1/1     Running   0          2m4s
nginx-deployment-57f59b7b9b-78b4v   1/1     Running   0          2m4s
```

### 3.3 velero工具使用详解

如果准备定期备份集群的资源，则能够在发生意外事故（如服务中断）时恢复到以前的状态。也可以进单次的手动备份，默认备份保留期为(720小时即30天标签为TTL)也可以在必要时进行指定保留期。备份时可以进行资源过滤选择资源以及排除不需要要的资源，可以通过备份挂钩实现备份前或者备份后在容器中执行命令，高级一点的功能可以实现备份pod持久数据目录时冻结文件系统。

velero还可以实现集群资源的迁移从一个集群移植到另一个集群，只需要您将每个velero实例指向相同的存储位置。确保kubernetes集群版本一致，镜像仓库访问地址镜像名称一致即可。

#### 3.3.1 定期自动备份

如果确保集群，安全请设置定期自动备份，最好设置为每日备份。
自动备份创建命示例令如下

```shell
velero schedule create NAME --schedule [flags]
velero create schedule NAME --schedule="0 */6 * * *"
velero create schedule NAME --schedule="@every 6h"   #6小时备份一次，所有集群资源
velero create schedule NAME --schedule="@every 24h" --include-namespaces web   #24小时备份一次web命名空间下所有资源
velero create schedule NAME --schedule="@every 168h" --ttl 2160h0m0s   #168小时备份一次所有集群资源，并且设置保留时间为2160小时

#常用的参数为
--include-namespaces     #选择哪个命名空间备份
--ttl                    #设置保留备份时间
--schedule               #周期时间，跟系统crontab设置类似
```

示例

```shell
#创建一个每日备份集群业务命名空间的定时任务
velero schedule create default --schedule="@every 24h" --include-namespaces=default,minio --ttl 72h
#这个是每隔24小时，创建一次备份，24小时周期为你创建定时任务的时间为开始时间

#验证创建的备份
velero backup get
#查看定时备份任务
velero schedule get
#删除定时备份任务
velero schedule delete default
#查看定时备份任务的详细信息
velero schedule describe default
#再次创建
velero schedule create default --schedule="@every 24h" --include-namespaces=default,minio --ttl 72h
```

#### 3.3.2 手动备份

```shell
#单次备份，可以增加各种参数，设置备份策略
velero backup create test --include-namespaces=default

#查看备份列表，定时备份任务备份的内容也是在这里查看
velero backup get 
#下载备份内容到本地
velero backup download test
#查看备份的详细信息，主要查看备份的状态以及一些元数据
velero backup describe test
#查看备份日志，主要用来查看备份过程中是否有错误，会有错误的详细信息
velero backup logs test
#删除备份，备份任务默认保存30天，即使不手动处理velero也会自动清理
velero backup delete test
```

#### 3.3.3 资源过滤

资源过滤的方式有

Includes(包括)
–include-namespaces
–include-resources
–include-cluster-resources
–selector
Excludes(排除)
–exclude-namespaces
–exclude-resources
velero.io/exclude-from-backup=true

##### 1.Includes(包括)

仅包含特定资源，不包括所有其他资源。当包括通配符和特定资源时，通配符优先。
–include-namespaces(包括命名空间)
主要用来备份命名空间时，选择备份那个命名空间

```shell
#备份命名空间及其子对象
velero backup create <backup-name> --include-namespaces <namespace>
#恢复俩个命名空间及其对象
velero restore create <backup-name> --include-namespaces <namespace1>,<namespace2>
```

–include-resources(包括资源)
主要用来指定备份的资源

```shell
#备份集群中的所有部署。
velero backup create <backup-name> --include-resources deployments
#恢复集群中的所有部署和配置。
velero restore create <backup-name> --include-resources deployments,configmaps
#在特定的命名空间备份部署
velero backup create <backup-name> --include-resources deployments --include-namespaces <namespace>
```

–include-cluster-resources(包括集群资源)
此选项有三个值：

true：所有集群范围的资源都包括在内。
false：不包括集群范围资源。

```shell
#备份整个集群，包括集群范围的资源
velero backup create <backup-name>
#仅恢复集群中的名称速度资源
velero restore create <backup-name> --include-cluster-resources=false
#备份命名空间，并包括集群范围的资源
velero backup create <backup-name> --include-namespaces <namespace> --include-cluster-resources=true
```

-selector(选择器)

```shell
#包括与标签选择器匹配的资源
velero backup create <backup-name> --selector <key>=<value>
```

##### 2.Excludes(排除)

从备份中排除特定资源。通配符排除被忽略。
–exclude-namespaces(排除命名空间)

```shell
#排除特定的命名空间
velero backup create <backup-name> --exclude-namespaces kube-system
#在恢复过程中排除两个命名空间
velero restore create <backup-name> --exclude-namespaces <namespace1>,<namespace2>
```

–exclude-resources(排除资源)

```shell
#排除备份中的secrets
velero backup create <backup-name> --exclude-resources secrets
```

velero.io/exclude-from-backup=true(带有这个标签的资源排除在备份之外)

带有标签的资源不包括在备份中，即使它包含匹配的选择器标签。velero.io/exclude-from-backup=true

#### 3.3.4 灾后恢复

集群资源出现问题时，您需要手动进行资源恢复，恢复的资源可以全部恢复，也可以指定过滤

```shell
#恢复指定备份，恢复备份中的所有资源
velero restore create --from-backup=default-20220815142030

#恢复备份中的特定资源，可以写多个
velero restore create --from-backup=default-20220815142030 --include-resources=Deployments

#恢复状态查看
velero restore describe server-20220815142030
#恢复日志查看
velero restore logs server-20220815142030
```

#### 3.3.5 使用restic备份pod持久卷

velero如果使用restic备份pvc中的数据需要打标签否则默认是不备份的。

```shell
#备份的标签
kubectl annotate -n 命名空间 pod/pod名称 backup.velero.io/backup-volumes=pod定义的voluem名称

kubectl annotate -n server pod/redis-5c5975fdcc-5plsh backup.velero.io/backup-volumes=redis-data
#排除备份的标签，注意如果设置这个标签velero会不进行备份这个pvc的资源信息
kubectl annotate -n 命名空间 pod/pod名称 backup.velero.io/backup-volumes-excludes=pod定义的voluem名称

kubectl annotate -n server pod/redis-5c5975fdcc-5plsh backup.velero.io/backup-volumes-excludes=redis-data

#使用标签后如何查看
kubectl describe pod -n server redis-5c5975fdcc-5plsh 
Name:         redis-5c5975fdcc-5plsh
Namespace:    server
Annotations:  backup.velero.io/backup-volumes: redis-data   #标签在这里查看

#去除标签，只需要把=换成-即表示去除标签，跟k8s中标签lable使用一致
kubectl annotate -n 命名空间 pod/pod名称 backup.velero.io/backup-volumes-
```

具体恢复过程

恢复带pvc的pod时，会生成一个初始化容器，用来同步之前备份的pvc中的数据，同步数据完成后会启动之前的容器。

minio中备份的pvc数据，会在备份数据的velero存储桶生成一个restic的备份数据目录，所有的pvc中的数据都会被备份到此处。
