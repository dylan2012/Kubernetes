# 一、Longhorn基础介绍

官方github：<https://github.com/longhorn/longhorn>

官方网站：<https://longhorn.io>

Longhorn是一个轻量级、可靠且功能强大的分布式块存储系统，适用于 Kubernetes。使用容器和微服务实现分布式块存储。Longhorn 为每个块储存设备卷创建一个专用的存储控制器，并在存储在多个节点上的多个副本之间同步复制该卷。存储控制器和副本本身是使用 Kubernetes 编排的。Longhorn 是免费的开源软件。它最初由Rancher Labs开发，现在作为云原生计算基金会的孵化项目进行开发。

Longhorn 支持以下架构：
AMD64
ARM64（实验性）

使用Longhorn，您可以：

使用 Longhorn 卷作为 Kubernetes 集群中分布式有状态应用程序的持久存储
将您的块存储分区为 Longhorn 卷，以便您可以在有或没有云提供商的情况下使用 Kubernetes 卷。
跨多个节点和数据中心复制块存储以提高可用性
将备份数据存储在外部存储（如 NFS 或 AWS S3）中
创建跨集群灾难恢复卷，以便从第二个 Kubernetes 集群中的备份中快速恢复主 Kubernetes 集群中的数据
计划卷的定期快照，并计划定期备份到 NFS 或与 S3 兼容的辅助存储
从备份还原卷
在不中断持久卷的情况下升级 Longhorn
Longhorn带有一个独立的UI，可以使用Helm，kubectl或Rancher应用程序目录进行安装。

## 二、部署Longhorn

我这里部署的版本为v1.2.4最新版本，在部署 Longhorn v1.2.4 之前，请确保您的 Kubernetes 集群至少为 v1.18，因为支持的 Kubernetes 版本已在 v1.2.4 中更新 （>= v1.18）。如果是低版本请部署选择相应的版本

### 2.1 安装前准备

在安装了 Longhorn 的 Kubernetes 集群中，每个节点都必须满足以下要求

与 Kubernetes 兼容的容器运行时（Docker v1.13+、containerd v1.3.7+ 等）
Kubernetes v1.18+，v1.2.4要求。
open-iscsi已安装，并且守护程序正在所有节点上运行。

```shell
#centos安装
yum install -y iscsi-initiator-utils
systemctl enable iscsid
systemctl start iscsid
```

RWX 支持要求每个节点都安装了 NFSv4 客户端。

```shell
yum install -y nfs-utils
```

主机文件系统支持存储数据的功能。
ext4/xfs
bash必须安装curl findmnt grep awk blkid lsblk

```shell
yum -y install curl util-linux grep gawk jq 
```

必须启用装载传播。
官方提供了一个测试脚本，需要在kube-master节点执行：https://raw.githubusercontent.com/longhorn/longhorn/v1.2.4/scripts/environment_check.sh

```shell
curl https://raw.githubusercontent.com/longhorn/longhorn/v1.2.4/scripts/environment_check.sh | bash
```

### 2.2 使用Helm安装Longhorn

官方提供了俩种安装方式，我们在这里使用helm进行安装

首先在kube-master安装Helm v2.0+。

#### 1.部署

```shell
[root@master01 longhorn]# helm version
version.BuildInfo{Version:"v3.9.0", GitCommit:"7ceeda6c585217a19a1131663d8cd1f7d641b2a7", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

添加 Longhorn Helm 存储库并安装

```shell
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm  pull longhorn/longhorn
tar xf longhorn*.tgz
cd longhorn
helm install longhorn  --namespace longhorn-system --create-namespace .
```

验证

```shell
#pod正常情况下都会是运行状态
kubectl get pod -n longhorn-system 
#验证存储类
[root@master01 longhorn]# kubectl get storageclasses.storage.k8s.io
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   125m
```

创建pvc并且挂载验证

```yaml
apiVersion: v1  
apiVersion: apps/v1 
kind: Deployment    
metadata:           
  name: nginx       
  namespace: default
spec:               
  replicas: 2      
  selector:          
    matchLabels:     
      app: nginx   
  template:          #以下为定义pod模板信息,请查看pod详解
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - name: test
        image: nginx
        imagePullPolicy: IfNotPresent
        volumeMounts: 
        - mountPath: "/data"
          name: data
      volumes:              
        - name: data          
          persistentVolumeClaim: 
            claimName: claim
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi

```

创建后验证

```shell
[root@master01 longhorn]# kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim   Bound    pvc-5f8d1f9b-4dd1-444a-9d09-2ecf7e1d19e0   1Gi        RWX            longhorn       6s

[root@master01 longhorn]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-864d99d8db-98dhp   1/1     Running   0          25s
nginx-864d99d8db-l8pxc   1/1     Running   0          25s
```

#### 2.修改配置

存储类配置

```yaml
persistence:
#是否设置为默认存储类
  defaultClass: true
#文件系统类型ext4与xfs选择一个
  defaultFsType: xfs
#副本数，建议三个
  defaultClassReplicaCount: 1
#删除pvc的策略
  reclaimPolicy: Delete
```

全局配置

```yaml
defaultSettings:
#备份相关
  backupTarget: ~
  backupTargetCredentialSecret: ~
  allowRecurringJobWhileVolumeDetached: ~
#仅在具有node.longhorn.io/create-default-disk=true标签的节点上初始化数据目录，默认为false在所有节点初始化数据目录
  createDefaultDiskLabeledNodes: ~
#默认数据目录位置，不配置默认为/var/lib/longhorn/
  defaultDataPath: /var/lib/longhorn
#默认数据位置，默认选项为disabled，还有best-effort，表示使用卷的pod是否与卷尽量在同一个node节点，默认为不进行这个限制。
  defaultDataLocality: ~
#卷副本是否为软亲和，默认false表示相同卷的副本强制调度到不同节点，如果为true则表示同一个卷的副本可以在同一个节点
  replicaSoftAntiAffinity: ~
#副本是否进行自动平衡。默认为disabled关闭，least-effort平衡副本以获得最小冗余，best-effort此选项指示 Longhorn 尝试平衡副本以实现冗余。
  replicaAutoBalance: ~
#存储超配置百分比默认200，已调度存储+已用磁盘空间（存储最大值-保留存储）未超过之后才允许调度新副本实际可用磁盘容量的 200%
  storageOverProvisioningPercentage: ~
#存储最小可用百分比默认,默认设置为 25，Longhorn 管理器仅在可用磁盘空间（可用存储空间）减去磁盘空间量且可用磁盘空间仍超过实际磁盘容量（存储空间）的 25%后才允许调度新副本）。否则磁盘将变得不可调度，直到释放更多空间。
  storageMinimalAvailablePercentage: ~
#定期检查版本更新，默认true启用
  upgradeChecker: ~
#创建卷时的默认副本数默认为3
  defaultReplicaCount: ~
#默认Longhorn静态存储类名称，默认值longhorn-static
  defaultLonghornStaticStorageClass: ~
#轮询备份间隔默认300，单位秒
  backupstorePollInterval: ~
#优先级配置
  priorityClass: ~
#卷出现问题自动修复，默认为true
  autoSalvage: ~
#驱逐相关
  disableSchedulingOnCordonedNode: ~
#副本亲和相关
  replicaZoneSoftAntiAffinity: ~
  nodeDownPodDeletionPolicy: ~
#存储安全相关，默认false，如果节点包含卷的最后一个健康副本，Longhorn将阻塞该节点上的kubectl清空操作。
  allowNodeDrainWithLastHealthyReplica: ~
#如果使用ext4文件系统允许设置其他创建参数，以支持旧版本系统内核
  mkfsExt4Parameters: ~
#副本有问题的重建间隔默认600秒 
  replicaReplenishmentWaitInterval: ~
#一个节点同时重建副本的个数，默认5个，超过会阻塞
  concurrentReplicaRebuildPerNodeLimit: ~
#允许引擎控制器和引擎副本在每次数据写入时禁用修订计数器文件更新。默认false
  disableRevisionCounter: ~
#Pod镜像拉取策略与k8s一致
  systemManagedPodsImagePullPolicy: ~
#此设置允许用户创建和挂载在创建时没有计划所有副本的卷，默认true，生产建议关闭
  allowVolumeCreationWithDegradedAvailability: ~
#此设置允许Longhorn在副本重建后自动清理系统生成的快照，默认true
  autoCleanupSystemGeneratedSnapshot: ~
#升级相关
  concurrentAutomaticEngineUpgradePerNodeLimit: ~
#备份相关
  backingImageCleanupWaitInterval: ~
  backingImageRecoveryWaitInterval: ~
#资源限制相关
  guaranteedEngineManagerCPU: ~
  guaranteedReplicaManagerCPU: ~
```

### 三、运维相关

#### 3.1 为UI控制台设置认证

UI控制台自己并没有认证功能，依赖与ingress提供认证功能，这里使用ingress-nginx演示：

```shell
#创建认证文件
USER=admin; PASSWORD=admin123; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
#创建密钥
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```

创建ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/ssl-redirect: 'false'
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
    nginx.ingress.kubernetes.io/proxy-body-size: 10000m
spec:
  ingressClassName: nginx
  rules:
  - host: www.longhornio.org
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: longhorn-frontend
            port:
              number: 80
```

```shell
kubectl apply -f longhorn-ingress.yaml
```

#### 3.2 卷扩展

使用StorageClass创建的卷，可以直接修改pvc的spec.resources.requests.storage字段进行扩容,如果卷是被挂载状态修改完成后并不会立马进行扩容操作，必须停止挂载pvc的Pod使pvc处于detached状态后才会进行扩容操作。

在扩展期间不允许重建和添加复制副本，在重建或添加复制副本时不允许扩展。

示例：

```shell
$ kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim   Bound    pvc-5119939d-ff12-42c4-b6f8-5d2fcc393bc2   1Gi        RWX            longhorn       11s
$ kubectl edit pvc claim
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 2Gi  #修改
$kubectl get pvc
NAME    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
claim   Bound    pvc-5119939d-ff12-42c4-b6f8-5d2fcc393bc2   2Gi        RWX            longhorn       63s
```

#### 3.3 单节点多磁盘实现

Longhorn 支持在节点上使用多个磁盘来存储卷数据，默认情况下会初始k8s所有的node节点作为Longhorn的数据节点，默认的数据目录为/var/lib/longhorn。但是可以通过安装时设置createDefaultDiskLabeledNodes为true，改变默认操作表示只有 node.longhorn.io/create-default-disk=true 标签的node才会初始化数据目录，可以不给节点打这个标签，之后手动在UI页面配置各个节点的磁盘，配置方式如下。

修改部署helm的valuse文件，重新部署

```shell
defaultSettings:
  createDefaultDiskLabeledNodes: true  #修改这个值

#重新部署
$ helm install longhorn -n longhorn-system .
```

node添加磁盘
选择一个node节点,点击Edit Node and Disks进入编辑状态。

点击Add Disk进行数据目录的添加，数据目录需事先进行格式化硬盘并且挂载到相应目录

可以重复添加多个数据目录，每个数据目录对应一个物理磁盘

填写的内容有

Name: 磁盘名称
Path：数据目录路径
Storage Reserved: 保留空间
Scheduling: 是否允许调度
Eviction Requested: 是否驱逐副本

#### 3.4 驱逐节点上的副本与磁盘

驱逐节点上的副本需要在UI页面进行操作，在Node选项卡找到对应的节点，需要先设置禁用节点相应磁盘后在设置驱逐副本。

驱逐节点磁盘副本

到选项卡，选择其中一个节点，然后在下拉菜单中进行选择。Node Edit Node and Disks
确保已禁用磁盘以进行调度并设置为 。Scheduling Disable
设置为并保存。Eviction Requested true
之后等待磁盘中的副本变为0，即可进行删除磁盘操作

驱逐节点所有副本

转到选项卡，选择一个或多个节点，然后单击 。Node Edit Node
确保已禁用节点以进行调度并设置为 。Scheduling Disable
设置为 ，然后保存。Eviction Requested true

#### 3.5 移除node节点步骤

禁用磁盘调度。
逐出节点上的所有副本。
驱逐节点所有Pod
kubectl drain <node-name>
使用选项卡中的 从 Longhorn 中删除节点。Delete Node

或者，从 Kubernetes 中删除该节点，使用：

kubectl delete node <node-name>
Longhorn 将自动从群集中删除该节点。

#### 3.6 高可用相关

1.自动平衡副本
当副本在节点或区域上的调度不均匀时，Longhorn 设置允许副本在新节点可用于群集时进行自动平衡。

设置参数为eplicaAutoBalance，全局配置，可以安装时在valuse中设置

disabled.这是默认选项，不会执行副本自动平衡。
least-effort.此选项指示 Longhorn 平衡副本以实现最小冗余。例如，添加节点 2 后，具有 4 个不平衡副本的卷将仅重新平衡 1 个副本。
best-effort.此选项指示 Longhorn 尝试平衡副本以实现均匀冗余。例如，添加节点 2 后，具有 4 个不平衡副本的卷将重新平衡 2 个副本。
在存储类中设置

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: hyper-converged
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  replicaAutoBalance: "least-effort"  #这里设置
  staleReplicaTimeout: "2880" # 48 hours in minutes
  fromBackup: ""
```

### 四、监控告警

官方文档：https://longhorn.io/docs/1.2.4/monitoring

这里采用kube-Prometheus项目进行监控。

Longhorn 在 REST 端点上以 Prometheus 文本格式原生公开指标。http://LONGHORN_MANAGER_IP:PORT/metrics

官方示例。监控系统使用Prometheus来收集数据和警报，Grafana用于可视化/仪表板收集的数据。从高级概述来看，监控系统包含：

Prometheus服务器，从Longhorn指标端点抓取和存储时间序列数据。Prometheus还负责根据配置的规则和收集的数据生成警报。然后，Prometheus服务器向警报管理器发送警报。
然后，AlertManager 管理这些警报，包括静音、抑制、聚合以及通过电子邮件、待命通知系统和聊天平台等方法发送通知。
Grafana查询Prometheus服务器以获取数据并绘制用于可视化的仪表板。

#### 4.1 使用ServiceMonitor获取指标数据

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: longhorn
  namespace: monitoring
  labels:
    name: longhorn
spec:
  endpoints:
  - port: manager
  selector:
    matchLabels:
      app: longhorn-manager
  namespaceSelector:
    matchNames:
    - longhorn-system
```

#### 4.2 创建rule告警规则

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: longhorn
    role: alert-rules
  name: prometheus-longhorn-rules
  namespace: monitoring
spec:
  groups:
  - name: longhorn.rules
    rules:
    - alert: LonghornVolumeActualSpaceUsedWarning
      annotations:
        description: The actual space used by Longhorn volume {{$labels.volume}} on {{$labels.node}} is at {{$value}}% capacity for
          more than 5 minutes.
        summary: The actual used space of Longhorn volume is over 90% of the capacity.
      expr: (longhorn_volume_actual_size_bytes / longhorn_volume_capacity_bytes) * 100 > 90
      for: 5m
      labels:
        issue: The actual used space of Longhorn volume {{$labels.volume}} on {{$labels.node}} is high.
        severity: warning
    - alert: LonghornVolumeStatusCritical
      annotations:
        description: Longhorn volume {{$labels.volume}} on {{$labels.node}} is Fault for
          more than 2 minutes.
        summary: Longhorn volume {{$labels.volume}} is Fault
      expr: longhorn_volume_robustness == 3
      for: 5m
      labels:
        issue: Longhorn volume {{$labels.volume}} is Fault.
        severity: critical
    - alert: LonghornVolumeStatusWarning
      annotations:
        description: Longhorn volume {{$labels.volume}} on {{$labels.node}} is Degraded for
          more than 5 minutes.
        summary: Longhorn volume {{$labels.volume}} is Degraded
      expr: longhorn_volume_robustness == 2
      for: 5m
      labels:
        issue: Longhorn volume {{$labels.volume}} is Degraded.
        severity: warning
    - alert: LonghornNodeStorageWarning
      annotations:
        description: The used storage of node {{$labels.node}} is at {{$value}}% capacity for
          more than 5 minutes.
        summary:  The used storage of node is over 70% of the capacity.
      expr: (longhorn_node_storage_usage_bytes / longhorn_node_storage_capacity_bytes) * 100 > 70
      for: 5m
      labels:
        issue: The used storage of node {{$labels.node}} is high.
        severity: warning
    - alert: LonghornDiskStorageWarning
      annotations:
        description: The used storage of disk {{$labels.disk}} on node {{$labels.node}} is at {{$value}}% capacity for
          more than 5 minutes.
        summary:  The used storage of disk is over 70% of the capacity.
      expr: (longhorn_disk_usage_bytes / longhorn_disk_capacity_bytes) * 100 > 70
      for: 5m
      labels:
        issue: The used storage of disk {{$labels.disk}} on node {{$labels.node}} is high.
        severity: warning
    - alert: LonghornNodeDown
      annotations:
        description: There are {{$value}} Longhorn nodes which have been offline for more than 5 minutes.
        summary: Longhorn nodes is offline
      expr: (avg(longhorn_node_count_total) or on() vector(0)) - (count(longhorn_node_status{condition="ready"} == 1) or on() vector(0)) > 0
      for: 5m
      labels:
        issue: There are {{$value}} Longhorn nodes are offline
        severity: critical
    - alert: LonghornIntanceManagerCPUUsageWarning
      annotations:
        description: Longhorn instance manager {{$labels.instance_manager}} on {{$labels.node}} has CPU Usage / CPU request is {{$value}}% for
          more than 5 minutes.
        summary: Longhorn instance manager {{$labels.instance_manager}} on {{$labels.node}} has CPU Usage / CPU request is over 300%.
      expr: (longhorn_instance_manager_cpu_usage_millicpu/longhorn_instance_manager_cpu_requests_millicpu) * 100 > 300
      for: 5m
      labels:
        issue: Longhorn instance manager {{$labels.instance_manager}} on {{$labels.node}} consumes 3 times the CPU request.
        severity: warning
    - alert: LonghornNodeCPUUsageWarning
      annotations:
        description: Longhorn node {{$labels.node}} has CPU Usage / CPU capacity is {{$value}}% for
          more than 5 minutes.
        summary: Longhorn node {{$labels.node}} experiences high CPU pressure for more than 5m.
      expr: (longhorn_node_cpu_usage_millicpu / longhorn_node_cpu_capacity_millicpu) * 100 > 90
      for: 5m
      labels:
        issue: Longhorn node {{$labels.node}} experiences high CPU pressure.
        severity: warning
```

#### 4.3 导入grafana模板

模板：https://grafana.com/grafana/dashboards/13032

### 五、备份相关

#### 5.1 快照功能

快照是 Kubernetes 卷在任何给定时间点的状态。

要创建现有集群的快照，

在 Longhorn UI 的顶部导航栏中，单击Volume。
单击要为其拍摄快照的卷的名称。这将进入卷详细页面。
单击Take Snapshot按钮
创建快照后，您将在卷头之前的卷快照列表中看到它

#### 5.2 卷克隆

Longhorn 支持 CSI 卷克隆。官方说明https://kubernetes.io/docs/concepts/storage/volume-pvc-datasource/

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

您可以通过应用以下 yaml 文件来创建一个内容与 完全相同的新 PVC，除了官方的一些要求之外还必须满足cloned-claim和claim的resources.requests.storage相同。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-claim
spec:
  storageClassName: longhorn
  dataSource:
    name: claim
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

#### 5.3 备份详解

Longhorn可以备份数据到s3对象存储或者nfs当中，并且可以设置定期备份。

##### 1.备份目标设置

备份目标是用于访问 Longhorn 中的备份存储的端点。备份存储是 NFS 服务器或 S3 兼容服务器，用于存储 Longhorn 卷的备份。

我这里使用minio与nfs进行演示，如果使用其他s3对象存储请查看官方文档：https://longhorn.io/docs/1.2.4/snapshots-and-backups/backup-and-restore/set-backup-target/

备份目标的配置有俩项分别是

backupTarget：备份目标设置

##### 2.手动创建备份

必须设置备份目标。确定备份目标配置没有问题，并且备份的volume必须处于挂载状态才可以进行备份。

导航到Volume菜单。
选择要备份的卷。
单击Create backup
添加任何适当的标签，然后单击确定。
创建完成后可以在Backup导航菜单看到所创建的备份。

##### 3.定期备份

可以在UI中配置，也可以在k8s中进行配置，他是一个CRD资源RecurringJob。

```yaml
apiVersion: longhorn.io/v1beta1
kind: RecurringJob
metadata:
  name: backup
  namespace: longhorn-system
spec:
  cron: "*/5 * * * *"
  task: "backup"
  groups:
  - default
  - group1
  retain: 1
  concurrency: 2
  labels:
    label/1: a
    label/2: b
```

name：定期作业的名称。不要使用重复的名称。并且的长度不应超过40个字符。
task：作业类型。它支持（定期创建快照）或（定期创建快照然后进行备份）。选项有snapshot，backup
cron：克伦表达式。它告诉作业的执行时间。
retain：Longhorn 将为每个卷作业保留多少个快照/备份。它应该不小于1。
concurrency：要并发运行的作业数。它应该不小于1。
可以指定可选参数：

groups：作业应属于的组，默认volues创建后都属于default组，所以如果对所有卷进行备份请设置default，如果不同的卷有不同的备份请配置不同组
labels：创建快照与备份后添加相应的标签。