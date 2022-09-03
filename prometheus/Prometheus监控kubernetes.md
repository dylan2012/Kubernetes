# Prometheus监控kubernetes

## 一、k8s监控架构介绍

Prometheus（普罗米修斯）是一个最初在SoundCloud上构建的监控系统。自2012年成为社区开源项目，拥有非常活跃的开发人员和用户社区。为强调开源及独立维护，Prometheus于2016年加入云原生云计算基金会（CNCF），成为继Kubernetes之后的第二个托管项目。

![avatar](https://zhangzhuo-1257627961.cos.ap-beijing.myqcloud.com//Typora/56a25ca71b52f63de33614300e65994b.png?imageView2/2/w/1280/format/jpg/interlace/1/q/100)

### Prometheus 组件介绍

1. Prometheus Server：Prometheus 生态最重要的组件，主要用于抓取和存储时间 序列数据，同时提供数据的查询和告警策略的配置管理；
2. Alertmanager：Prometheus 生态用于告警的组件，Prometheus Server 会将告警发送给 Alertmanager，Alertmanager 根据路由配置，将告警信息发送给指定的人或组。Alertmanager 支持邮件、Webhook、微信、钉钉、短信等媒介进行告 警通知；
3. Grafana：用于展示数据，便于数据的查询和观测；
4. Push Gateway：Prometheus 本身是通过 Pull 的方式拉取数据，但是有些监控数 据可能是短期的，如果没有采集数据可能会出现丢失。Push Gateway 可以用来 解决此类问题，它可以用来接收数据，也就是客户端可以通过 Push 的方式将数据推送到 Push Gateway，之后 Prometheus 可以通过 Pull 拉取该数据；
5. Exporter：主要用来采集监控数据，比如主机的监控数据可以通过 node_exporter 采集，MySQL 的监控数据可以通过 mysql_exporter 采集，之后 Exporter 暴露一 个接口，比如/metrics，Prometheus 可以通过该接口采集到数据；
6. PromQL：PromQL 其实不算 Prometheus 的组件，它是用来查询数据的一种语法，比如查询数据库的数据，可以通过SQL语句，查询Loki的数据，可以通过LogQL，查询 Prometheus 数据的叫做 PromQL；
7. Service Discovery：用来发现监控目标的自动发现，常用的有基于 Kubernetes、 Consul、Eureka、文件的自动发现等。

官网：<https://prometheus.io/>

github: <https://github.com/prometheus>

|  监控指标   | 具体实现  |举例  |
|  ----  | ----  | ----  |
|  Pod性能   | cAdvisor(kubelet已经集成)  |容器CPU，内存使用率  |
|  Node性能   | node-exporter(需单独部署)  |节点CPU，内存使用率  |
|  k8s资源对象   | kube-state-metrics(需单独部署)  |Pod/Deploy/Service  |

### Prometheus 工作流程

1.Prometheus server 可定期从活跃的（up）目标主机上（target）拉取监控指标数据，目标主机的
监控数据可通过配置静态 job 或者服务发现的方式被 prometheus server 采集到，这种方式默认的 pull
方式拉取指标；也可通过 pushgateway 把采集的数据上报到 prometheus server 中；还可通过一些组件
自带的 exporter 采集相应组件的数据；
2.Prometheus server 把采集到的监控指标数据保存到本地磁盘或者数据库；
3.Prometheus 采集的监控指标数据按时间序列存储，通过配置报警规则，把触发的报警发送到
alertmanager
4.Alertmanager 通过配置报警接收方，发送报警到邮件，微信或者钉钉等
5.Prometheus 自带的 web ui 界面提供 PromQL 查询语言，可查询监控数据
6.Grafana 可接入 prometheus 数据源，把监控数据以图形化形式展示出

### prometheus对于k8s的服务发现详解

prometheus支持k8s的服务发现：<https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config>
从Kubernetes的REST API上，Kubernets SD配置检索和获取目标，并且始终保持与集群状态同步。

role可以指定特定的服务发现类型用来发现k8s中各种资源分别有node,service,pod,endpoints，endpointslice，ingress。

#### 1.服务发现配置文件详解

```yaml
scrape_configs:   #数据采集配置
  - job_name: 'k8s-cadvisor'  #采集job名称
    scheme: https             #监控指标收集协议,k8s中大多服务的接口协议通常为https
    metrics_path: /metrics/cadvisor  #采集数据指标的url地址
    bearer_token_file: k8s.token     #这里是Prometheus访问服务时的token文件
    tls_config:                      #这里是tls认证配置，选择其中一种即可
      insecure_skip_verify: true     #强制认证证书，无需配置ca证书公钥
      ca_file: ca.pem                #通过ca公钥认证
    kubernetes_sd_configs:           #以下为prometheus的k8s服务发现配置
    - role: node                     #发现方式,node,service,pod,endpoints,endpointslice,ingress选择其中一种即可，特定的资源需要选择特定的发现方式      
      api_server: https://192.168.10.71:6443  #这里为k8s的kube-apiserver的通信地址与端口
      bearer_token_file: k8s.token #这里是Prometheus访问kube-apiserver的token文件位置，token获取一般使用kubectl  describe secrets 获取，需提前准备并且给予特定权限
      kubeconfig_file: config.kubeconfig  #k8s的认证文件，如果使用kubeconfig认证文件及不需要配置api_server以及bearer_token_file
      tls_config: #这里是tls认证配置，一般没有特殊要求直接可以强制认证通过，如果需要认证需配置kube-apiserver的证书ca文件，俩种方式选择其中一个
        insecure_skip_verify: true   #强制认证证书，无需配置ca证书公钥
        ca_file: ca.pem              #通过ca公钥认证
      namespaces:                    #选择发现资源的命名空间，不写默认全部命名空间
        names:
          - default
    relabel_configs:                 #以下为标签的重写配置，可根据自己的需求进行重写，配置方法较多这里不进行详细列举。
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

#### 2.node详解

Kubelet组件运行在Kubernetes集群的各个节点中，其负责维护和管理节点上Pod的运行状态。kubelet组件的正常运行直接关系到该节点是否能够正常的被Kubernetes集群正常使用。

kubelet组件中默认集成了cadvisor组件，无需重复安装，数据默认采集路径/metrics/cadvisor，所以如果需要采集数据，也是通过node发现服务进行采集。

基于Node模式，Prometheus会自动发现Kubernetes中所有Node节点的信息并作为监控的目标Target。 而这些Target的访问地址实际上就是Kubelet的访问地址，并且Kubelet实际上直接内置了对Promtheus的支持。

可用的meta标签：

* __meta_kubernetes_node_name: 节点对象的名称
* __meta_kubernetes_node_label_<labelname>: 节点对象的每个标签
* __meta_kubernetes_node_labelpresent_<labelname>: 节点对象中的每个标签都为true。
* __meta_kubernetes_node_annotation_<annotationname>: 节点对象的每个注解
* __meta_kubernetes_node_annotationpresent_<annotationname>: 节点对象的每个注释都为true。
* __meta_kubernetes_node_address_<address_type>: 如果存在，每一个节点对象类型的第一个地址

另外，对于节点的instance标签，将会被设置成从API服务中获取的节点名称。

#### 3.service

对于每个服务每个服务端口，service角色发现一个目标。对于一个服务的黑盒监控是通常有用的。这个地址被设置成这个服务的Kubernetes DNS域名, 以及各自的服务端口。

可用的meta标签：

* __meta_kubernetes_namespace: 服务对象的命名空间
* __meta_kubernetes_service_annotation_<annotationname>: 服务对象的注释
* __meta_kubernetes_service_annotationpresent_<annotationname>: 服务对象的每个注解为“true”。
* __meta_kubernetes_service_cluster_ip: 服务的群集IP地址。（不适用于ExternalName类型的服务）
* __meta_kubernetes_service_external_name: 服务的DNS名称。（适用于ExternalName类型的服务）
* __meta_kubernetes_service_label_<labelname>: 服务对象的标签。
* __meta_kubernetes_service_labelpresent_<labelname>: 对于服务对象的每个标签为true。
* __meta_kubernetes_service_name: 服务对象的名称
* __meta_kubernetes_service_port_name: 目标服务端口的名称
* __meta_kubernetes_service_port_protocol: 目标服务端口的协议
* __meta_kubernetes_service_type: 服务的类型

#### 4.Pod

pod角色会察觉所有的pod，并将它们的容器作为目标暴露出来。对于容器的每个声明的端口，都会生成一个目标。如果一个容器没有指定的端口，则会为每个容器创建一个无端口的目标，以便通过重新标注来手动添加端口。

可用的meta标签：

* __meta_kubernetes_namespace: pod对象的命名空间
* __meta_kubernetes_pod_name: pod对象的名称
* __meta_kubernetes_pod_ip: pod对象的IP地址
* __meta_kubernetes_pod_label_<labelname>: pod对象的标签
* __meta_kubernetes_pod_labelpresent_<labelname>: 对来自pod对象的每个标签都是true。
* __meta_kubernetes_pod_annotation_<annotationname>: pod对象的注释
* __meta_kubernetes_pod_annotationpresent_<annotationname>: 对于来自pod对象的每个注解都是true。
* __meta_kubernetes_pod_container_init: 如果容器是 InitContainer，则为 true。
* __meta_kubernetes_pod_container_name: 目标地址的容器名称
* __meta_kubernetes_pod_container_port_name: 容器端口名称
* __meta_kubernetes_pod_container_port_number: 容器端口的数量
* __meta_kubernetes_pod_container_port_protocol: 容器端口的协议
* __meta_kubernetes_pod_ready: 设置pod ready状态为true或者false
* __meta_kubernetes_pod_phase: 在生命周期中设置 Pending, Running, Succeeded, Failed 或 Unknown
* __meta_kubernetes_pod_node_name: pod调度的node名称
* __meta_kubernetes_pod_host_ip: 节点对象的主机IP
* __meta_kubernetes_pod_uid: pod对象的UID。
* __meta_kubernetes_pod_controller_kind: pod控制器的kind对象.
* __meta_kubernetes_pod_controller_name: pod控制器的名称.

#### 5.endpoints

endpoints角色发现来自于一个服务的列表端点目标。对于每一个终端地址，一个目标被一个port发现。如果这个端点被写入到pod中，这个节点的所有其他容器端口，未绑定到端点的端口，也会被目标发现。

可用的meta标签：

* __meta_kubernetes_namespace: 端点对象的命名空间
* __meta_kubernetes_endpoints_name: 端点对象的名称
对于直接从端点列表中获取的所有目标，下面的标签将会被附加上。
* __meta_kubernetes_endpoint_hostname: 端点的Hostname
* __meta_kubernetes_endpoint_node_name: 端点所在节点的名称。
* __meta_kubernetes_endpoint_ready: endpoint ready状态设置为true或者false。
* __meta_kubernetes_endpoint_port_name: 端点的端口名称
* __meta_kubernetes_endpoint_port_protocol: 端点的端口协议
* __meta_kubernetes_endpoint_address_target_kind: 端点地址目标的kind。
* __meta_kubernetes_endpoint_address_target_name: 端点地址目标的名称。
如果端点属于一个服务，这个角色的所有标签：服务发现被附加上。
对于在pod中的所有目标，这个角色的所有表掐你：pod发现被附加上

#### 6.ingress

ingress角色为每个ingress的每个路径发现一个目标。这通常对黑盒监控一个ingress很有用。地址将被设置为 ingress 规范中指定的主机。

可用的meta标签：

* __meta_kubernetes_namespace: ingress对象的命名空间
* __meta_kubernetes_ingress_name: ingress对象的名称
* __meta_kubernetes_ingress_label_<labelname>: ingress对象的每个label。
* __meta_kubernetes_ingress_labelpresent_<labelname>: ingress对象的每个label都为true。
* __meta_kubernetes_ingress_annotation_<annotationname>: ingress对象的每个注释.
* __meta_kubernetes_ingress_annotationpresent_<annotationname>: 每个ingress对象的注解都是true。
* __meta_kubernetes_ingress_scheme: 协议方案，如果设置了TLS配置，则为https。默认为http。
* __meta_kubernetes_ingress_path: ingree spec的路径。默认为/。

## 二、Prometheus server 安装和配置

### 创建rbac授权

```shell
#创建命名空间
kubectl create namespace kube-monitor
#创建一个 sa 账号 monitor
kubectl create serviceaccount monitor -n kube-monitor
#把 sa 账号 monitor 通过 clusterrolebing 绑定到 clusterrole 上
kubectl create clusterrolebinding monitor-clusterrolebinding -n kube-monitor --clusterrole=cluster-admin --serviceaccount=kube-monitor:monitor
```

如果不绑定cluster-admin，使用具体RABC授权prometheus-rbac.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor
  namespace: kube-monitor
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitor
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitor
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitor
subjects:
- kind: ServiceAccount
  name: monitor
  namespace: kube-monitor
```

### 创建数据目录

```shell
cat > pvc.yaml <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus-pvc
  namespace: kube-monitor
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: alertmanager-pvc
  namespace: kube-monitor
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
kubectl apply -f pvc.yaml
```

### 部署Prometheus server

#### 创建configmap 存储卷，用来存放 prometheus 配置信息

```shell
cat > prometheus-cfg.yaml <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus-config
  namespace: kube-monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
      evaluation_interval: 1m
    scrape_configs:
    - job_name: 'kubernetes-node'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-node-cadvisor'
      kubernetes_sd_configs:
      - role:  node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiserver'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
EOF
kubectl apply -f prometheus-cfg.yaml
```

#### node-exporter 组件安装和配置

```shell
cat >node-export.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: kube-monitor
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
     name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 0.15
        securityContext:
          privileged: true
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      - key: "node-role.kubernetes.io/control-plane"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: dev
          hostPath:
            path: /dev
        - name: sys
          hostPath:
            path: /sys
        - name: rootfs
          hostPath:
            path: /
EOF
kubectl apply -f node-export.yaml
```

#### 通过 deployment 部署 prometheus

```shell
cat > prometheus-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: kube-monitor
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
      component: server
    #matchExpressions:
    #- {key: app, operator: In, values: [prometheus]}
    #- {key: component, operator: In, values: [server]}
  template:
    metadata:
      labels:
        app: prometheus
        component: server
      annotations:
        prometheus.io/scrape: 'false'
    spec:
      serviceAccountName: monitor
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        imagePullPolicy: IfNotPresent
        command:
          - prometheus
          - --config.file=/etc/prometheus/prometheus.yml
          - --storage.tsdb.path=/prometheus
          - --storage.tsdb.retention=720h
          - --web.enable-lifecycle
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: /etc/prometheus/prometheus.yml
          name: prometheus-config
          subPath: prometheus.yml
        - mountPath: /prometheus/
          name: prometheus-storage-volume
      volumes:
        - name: prometheus-config
          configMap:
            name: prometheus-config
            items:
              - key: prometheus.yml
                path: prometheus.yml
                mode: 0644
        - name: prometheus-storage-volume
          persistentVolumeClaim:
            claimName: prometheus-pvc
EOF
kubectl apply -f prometheus-deploy.yaml
```

#### 给 prometheus创建 service

```shell
cat > prometheus-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-monitor
  labels:
    app: prometheus
spec:
  type: NodePort
  ports:
    - port: 9090
      targetPort: 9090
      protocol: TCP
      nodePort: 30090
  selector:
    app: prometheus
    component: server
EOF
kubectl apply -f prometheus-svc.yaml
```

浏览器打开 http://nodeIp:30090
点击Status-Targets 查看监控

另外的部署方式 kube-Prometheus部署
官方项目地址：<https://github.com/prometheus-operator/kube-prometheus>

## 三、Grafana 的安装和配置

### Grafana 介绍

### 安装 Grafana

```shell
cat >grafana.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: kube-monitor
  labels:
    app: grafana
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin321
        readinessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 30
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - mountPath: /var/lib/grafana
          subPath: grafana
          name: storage
      securityContext:
        fsGroup: 472
        runAsUser: 472
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana-pvc
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-pvc
  namespace: kube-monitor
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-monitor
  labels:
    app: grafana
spec:
  type: NodePort
  ports:
    - port: 3000
  selector:
    app: grafana
---
EOF

cat > grafana-chown.yaml<<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: grafana-chown
  namespace: kube-monitor
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: grafana-chown
        command: ["chown", "-R", "472:472", "/var/lib/grafana"]
        image: busybox
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: storage
          subPath: grafana
          mountPath: /var/lib/grafana
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana-pvc
EOF

kubectl apply -f grafana.yaml
#验证
kubectl get pods -n kube-monitor -l app=grafana
#如果有权限报错可执行以下
#kubectl apply -f grafana-chown.yaml
#获取nodePort
kubectl get svc -n kube-monitor | grep grafana
```

### Grafana 接入Prometheus

浏览器打开 http://nodeIp:nodePort
开始配置 grafana 的 web 界面：
选择 Create your first data source

Settings面板下填写

Name: Prometheus
Type: Prometheus
URL: http://prometheus.kube-monitor.svc:9090

点击左下角 Save & Test，出现如下 Data source is working，说明 prometheus 数据源成功的被
grafana 接入了

点击左边+Create->Import->Upload .json File
选择node_exporter.json

### 安装 kube-state-metrics 组件

kube-state-metrics 通过监听 API Server 生成有关资源对象的状态指标，比如 Deployment、
Node、 Pod，需要注意的是 kube-state-metrics 只是简单的提供一个 metrics 数据，并不会存储这
些指标数据，所以我们可以使用 Prometheus 来抓取这些数据然后存储，主要关注的是业务相关的一
些元数据，比如 Deployment、 Pod、副本状态等；调度了多少个 replicas？现在可用的有几个？多
少个 Pod 是 running/stopped/terminated 状态？ Pod 重启了多少次？我有多少 job 在运行中。

```shell
cat >kube-state-metrics-rbac.yaml <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "resourcequotas", "replicationcontrollers", "limitranges", "persistentvolumeclaims", "persistentvolumes", "namespaces", "endpoints"]
  verbs: ["list", "watch"]
- apiGroups: ["extensions"]
  resources: ["daemonsets", "deployments", "replicasets"]
  verbs: ["list", "watch"]
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["list", "watch"]
- apiGroups: ["batch"]
  resources: ["cronjobs", "jobs"]
  verbs: ["list", "watch"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
- kind: ServiceAccount
  name: kube-state-metrics
  namespace: kube-system
EOF
kubectl apply -f kube-state-metrics-rbac.yaml

cat > kube-state-metrics-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-state-metrics
  template:
    metadata:
      labels:
        app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: quay.io/coreos/kube-state-metrics:v1.9.0
        ports:
        - containerPort: 8080
EOF
kubectl apply -f kube-state-metrics-deploy.yaml
kubectl get pods -n kube-system -l app=kube-state-metrics

#Service
cat > kube-state-metrics-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: 'true'
  name: kube-state-metrics
  namespace: kube-system
  labels:
    app: kube-state-metrics
spec:
  ports:
  - name: kube-state-metrics
    port: 8080
    protocol: TCP
  selector:
    app: kube-state-metrics
EOF
kubectl apply -f kube-state-metrics-svc.yaml
kubectl get svc -n kube-system | grep kube-state-metrics
```

在 grafana web 界面导入
Kubernetes Cluster (Prometheus).json
Kubernetes cluster monitoring (via Prometheus).json

除此之外，我们也可以前往 grafana dashboard 的页面去搜索其他的关于 Kubernetes 的监控页面，地址：https://grafana.com/dashboards，比如id 为747和741的这两个 dashboard。

## 四、配置 alertmanager

官方告警方案：
<https://prometheus.io/docs/alerting/latest/configuration/>

### 配置发送报警到 qq 邮箱

```shell
cat > alertmanager-cm.yaml <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: kube-monitor
data:
  alertmanager.yml: |-
    global:
      resolve_timeout: 1m
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'username@163.com'
      smtp_auth_username: 'username' #用户名
      smtp_auth_password: 'password' #密码
      smtp_require_tls: false
    route:
      group_by: [alertname] # 采用哪个标签来作为分组依据
      group_wait: 10s # 组告警等待时间
      group_interval: 10s # 上下两组发送告警的间隔时间
      repeat_interval: 10m # 重复发送告警的时间
      receiver: default-receiver #定义谁来收告警
    receivers:
    - name: 'default-receiver'
      email_configs:
      - to: 'touser@qq.com' #接收邮箱
        send_resolved: true
EOF
kubectl apply -f alertmanager-cm.yaml
```

重新部署

```shell
kubectl delete -f prometheus-cfg.yaml
kubectl apply -f prometheus-alertmanager-cfg.yaml
#在master节点创建etcd证书
kubectl -n kube-monitor create secret generic etcd-certs --from-file=/etc/kubernetes/pki/etcd/server.key --from-file=/etc/kubernetes/pki/etcd/server.crt --from-file=/etc/kubernetes/pki/etcd/ca.crt

#删除原来的重新部署
kubectl delete -f prometheus-deploy.yaml
kubectl apply -f prometheus-alertmanager-deploy.yaml
kubectl apply -f alertmanager-svc.yaml

kubectl get pods -n kube-monitor | grep prometheus
```

浏览器打开 http://nodeIP:NodePort/#/alerts

访问 prometheus 的 web 界面
点击 status->targets，可看到如下报错

从上面可以发现 kubernetes-controller-manager 和kubernetes-schedule 都显示连接不上对应的端口

vim /etc/kubernetes/manifests/kube-scheduler.yaml
修改如下内容：
把--bind-address=127.0.0.1 变成--bind-address=0.0.0.0
把 httpGet:字段下的 hosts 由 127.0.0.1 变成 10.199.10.241
把—port=0 删除

vim /etc/kubernetes/manifests/kube-controller-manager.yaml
把--bind-address=127.0.0.1 变成--bind-address=0.0.0.0
把 httpGet:字段下的 hosts 由 127.0.0.1 变成 10.199.10.241
把—port=0 删除

修改之后在 k8s 各个节点执行
systemctl restart kubelet

kube-proxy 默认端口 10249 是监听在 127.0.0.1 上的，需要改成监听到物理节点上，按如下
方法修改，线上建议在安装 k8s 的时候就做修改，这样风险小一些：
kubectl edit configmap kube-proxy -n kube-system
把 metricsBindAddress 这段修改成 metricsBindAddress: 0.0.0.0:10249
然后重新启动 kube-proxy 这个 pod
kubectl get pods -n kube-system | grep kube-proxy |awk '{print
$1}' | xargs kubectl delete pods -n kube-system

热加载：curl -X POST  http://10.199.10.244:30090/-/reload

### 配置发送报警到钉钉

新版grafana后台的Alert添加

### 配置发送报警到微信

使用官方方案

### 配置发送到telegram

#### 创建 Telegrambot

遵循 Telegram 文档中的指南：https ://core.telegram.org/bots#creating-a-new-bot 您将收到电报令牌 ID。我们将其存储并稍后使用。
更改机器人加入群组，获取群组id： 我们需要用这个bot来发送消息，首先需要创建一个group，加入一些人，也可以不加人，同时将这个bot也加进去。然后在这个group中发送消息。类似 /hello @xxxxx_Bot 然后访问 https://api.telegram.org/botxxx:xxx/getUpdates xxx:xxx部分就是机器人的api token 需要妥善保管。 注意： 群组的chat_id就是上面的id，注意是负数，必须有-，机器人的chat_id则可以是正数。

主要获取到两个内容：
telegram-token：xxxxxx:xxxxxxxx
chat_id: xxxxxx

以下为另一种方案

#### 创建和构建电报机器人 docker 镜像

<https://github.com/inCaller/prometheus_bot>

```shell
git clone https://github.com/inCaller/prometheus_bot
cd prometheus_bot/
docker build -t prometheus-bot:0.0.1 .

#在需要部署的目录创建配置文件
mkdir -pv etc/telegrambot
cat > etc/config.yaml <<EOF
telegram_token: "telegram-token"
template_path: "/etc/telegrambot/template.tmpl"
time_zone: "Asia/Shanghai"
split_token: "|"
time_outdata: "02/01/2006 15:04:05"
split_msg_byte: 4000
EOF

#模板文件，可以项目testdata里面复制
cat > etc/telegrambot/template.tmpl <<EOF
Type: {{.CommonAnnotations.description}}
Summary: {{.CommonAnnotations.summary}}
Alertname: {{ .CommonLabels.alertname }}
Instance: {{ .CommonLabels.instance }}
Serverity: {{ .CommonLabels.serverity}}
Status:  {{ .Status }}
EOF

cat > docker-compose.yml <<EOF
version: '3.3'
services:
  prometheus-bot:
    container_name: telegrambot
    image: prometheus-bot:0.0.1  #你的镜像仓库地址
    volumes:
      - ./etc/telegrambot/:/etc/telegrambot/:rw
      - ./etc/config.yaml:/config.yaml:rw
    ports:
      - 9087:9087
    restart: always
EOF
#启动
docker-compose up -d
#替换<chat-id> 测试发送
curl -X POST http://127.0.0.1:9087/alert/<chat-id>
```

修改alertmanager-cm.yaml

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: alertmanager
  namespace: kube-monitor
data:
  alertmanager.yml: |-
    global:
      resolve_timeout: 1m
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'xxxxxx@163.com'
      smtp_auth_username: 'xxxxxx'
      smtp_auth_password: 'password'
      smtp_require_tls: false
    route:
      group_by: [alertname]
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 10m
      receiver: telegram-webhook
    receivers:
    - name: 'telegram-webhook'
      webhook_configs:
      #替换IP 10.199.10.245
      - url: http://10.199.10.245:9087/alert/<chat-id>
        send_resolved: true
    - name: 'default-receiver'
      email_configs:
      - to: 'xxxxxx@qq.com'
        send_resolved: true
```

```shell
kubectl delete -f alertmanager-cm.yaml
kubectl apply -f alertmanager-cm.yaml
#快速生效
kubectl delete -f prometheus-alertmanager-deploy.yaml
kubectl apply -f prometheus-alertmanager-deploy.yaml
```

## 五、Prometheus 监控扩展

已经配置好kubernetes-service-endpoints自动发现
在配置 service 时加入下面的配置内容

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
```

### promethues 采集 tomcat 监控数据

### promethues 采集 redis 监控数据

```shell
cat > redis-exporter.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
    spec:
      containers:
      - name: redis-exporter
        imagePullPolicy: Always
        env:
        - name: REDIS_ADDR
          value: "redis-master.default.svc:6379"
        - name: REDIS_PASSWORD
          value: "redis123"
        - name: REDIS_EXPORTER_DEBUG
          value: "1"
        image: oliver006/redis_exporter
        ports:
        - containerPort: 9121
          name: redis-exporter
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
  labels:
    app: redis-exporter
  name: redis-exporter
spec:
  ports:
  - name: redis-exporter
    port: 9121
    protocol: TCP
    targetPort: 9121
  type: NodePort
  selector:
    app: redis-exporter
EOF

kubectl apply -f redis-exporter.yaml
```

登录grafana导入模版ID为763

### Prometheus 监控 mysql

https://github.com/prometheus/mysqld_exporter

```shell
#mysql授权
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'exporter123' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
```

```shell
cat >deployment.yaml<<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysqld-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysqld-exporter 
  template:
    metadata:
      labels:
        app: mysqld-exporter
    spec:
      containers:
      - name: mysqld-exporter
        imagePullPolicy: IfNotPresent 
        env:
          - name: DATA_SOURCE_NAME
            value: "exporter:exporter123@(mysql.default.svc:3306)/"
        image: prom/mysqld-exporter
        ports:
        - containerPort: 9104
          name: mysqld-exporter
EOF

cat > service.yaml<<EOF
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9104"
  labels:
    app: mysqld-exporter
  name: mysqld-exporter
spec:
  ports:
  - name: mysqld-exporter
    port: 9104
    protocol: TCP
    targetPort: 9104
  type: NodePort
  selector:
    app: mysqld-exporter
EOF

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

登录grafana 导入模板 ID 为 7362

### Prometheus 监控 nginx

#### 自行编译

1、下载模块
https://github.com/vozlt/nginx-module-vts/releases
2、编译
编译参数添加--add-module=/path/to/nginx-module-vts
3、配置状态访问接口

```html
location /status{
        allow  127.0.0.1;
        deny all;
        vhost_traffic_status_display;
        vhost_traffic_status_display_format html;
}


http {
...
      # nginx-vts
        vhost_traffic_status_zone;
        vhost_traffic_status_filter_by_host on;   #开启此功能，会根据不同的server_name进行流量的统计，否则默认会把流量全部计算到第一个上。
        vhost_traffic_status_filter on;
        vhost_traffic_status_filter_by_set_key $status $server_name;
...
}  
```

此外还有基于地理信息的统计，根据访问量或访问流量对nginx
做访问限制，详细使用见文档：https://github.com/vozlt/nginx-module-vts#installation

#### 准备镜像

Nginx with the VTS module (https://github.com/vozlt/nginx-module-vts) and the prometheus exporter (https://github.com/hnlq715/nginx-vts-exporter)

直接找带vts模块的Nginx，可以用 https://hub.docker.com/r/skyscrapers/nginx
再找vts-exporter，可以用 https://hub.docker.com/r/arquivei/nginx-vts

```shell
cat > nginx-vts.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: skyscrapers/nginx:latest
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 80
      - name: nginx-exporter
        image: arquivei/nginx-vts:latest
        command: ["/bin/sh","-c","--"]
        args: ["nginx-vts-exporter -nginx.scrape_uri=http://localhost:9999/status/format/json"]
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 9913
---
kind: Service
apiVersion: v1
metadata:
  name: nginx
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9913"
spec:
  type: NodePort
  selector:
   app: nginx
  ports:
  - name: nginx
    port: 80
    targetPort: 80
  - name: prom
    port: 9913
    targetPort: 9913
EOF

kubectl apply -f nginx-vts.yaml
kubectl get svc
```

浏览器打开 http://nodeip:nodeport/metrics

grafana导入模板2948

### Prometheus 监控 mongodb
