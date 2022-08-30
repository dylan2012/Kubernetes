# 安装kube-prometheus

## 概述

官方地址：<https://github.com/prometheus-operator/kube-prometheus>
官方文档：<https://prometheus-operator.dev/>

kube-prometheus 是一整套监控解决方案，它使用 Prometheus 采集集群指标，Grafana 做展示，包含如下组件：

The Prometheus Operator
Highly available Prometheus
Highly available Alertmanager
Prometheus node-exporter
Prometheus Adapter for Kubernetes Metrics APIs
kube-state-metrics
Grafana

该堆栈用于集群监控，因此已预先配置为从所有 Kubernetes 组件收集指标。除此之外，它还提供一组默认仪表板和警报规则。许多有用的仪表板和警报都来自kubernetes-mixin 项目，类似于这个项目，它提供了可组合的 jsonnet 作为库，供用户根据自己的需要进行定制。

## 部署

查看kubernetes版本对应选择对应的版本
修改持久化配置

```shell
git clone https://github.com/coreos/kube-prometheus
cd kube-prometheus/manifests/

#修改Prometheus 持久化
vi prometheus-prometheus.yaml
#文件末尾新增
  serviceMonitorSelector: {}
  version: 2.38.0
  retention: 3d
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: nfs-client #使用sc
        resources:
          requests:
            storage: 5Gi

#修改grafana持久化配置
cat > grafana-pvc.yaml<<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
EOF

vi grafana-deployment.yaml
      serviceAccountName: grafana
      volumes:
#      - emptyDir: {}            #注释
#        name: grafana-storage
      - name: grafana-storage       # 新增持久化配置
        persistentVolumeClaim:
          claimName: grafana-pvc    # 设置为创建的PVC名称
```

修改各组件service

```shell
vi prometheus-service.yaml
spec:
  type: NodePort #新增
  ports:
  - name: web
    nodePort: 30090  #新增
    port: 9090

vi alertmanager-service.yaml
spec:
  type: NodePort #新增
  ports:
  - name: web
    nodePort: 30093 #新增

vi grafana-service.yaml
spec:
  type: NodePort #新增
  ports:
  - name: http
    nodePort: 32000 #新增

#安装CRD和prometheus-operator
kubectl apply --server-side -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl apply -f .

#查看是否安装完成
kubectl get pods -n monitoring

#修改网络策略 在以下文件中新增ingress-nginx支持
alertmanager-networkPolicy.yaml
grafana-networkPolicy.yaml
prometheus-networkPolicy.yaml
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx #新增

cat > ingress-monitor.yaml<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: prome.k8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-k8s
            port:
              number: 9090
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.k8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: alert.k8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: alertmanager-main
            port:
              number: 9093
EOF
kubectl apply -f ingress-monitor.yaml

#或者删除服务网络配置
#kubectl delete -f alertmanager-networkPolicy.yaml
#kubectl delete -f grafana-networkPolicy.yaml
#kubectl delete -f prometheus-networkPolicy.yaml
```

prometheus
打开浏览器 http://nodeip:30090
alertmanager
浏览器打开 http://nodeip:30093/
grafana
浏览器打开 http://nodeip:32000/
首次登录，用户名和密码，都是admin

或者配置用ingress域名访问

## 企业微信监控告警

```shell
cat > wechat.tmpl<<EOF
{{ define "wechat.default.message" }}
{{- if gt (len .Alerts.Firing) 0 -}}
{{- range $index, $alert := .Alerts -}}
{{- if eq $index 0 -}}
=====================
告警类型: {{ $alert.Labels.alertname }}
告警级别: {{ $alert.Labels.severity }}

{{- end }}
*****告警详情*****
告警详情: {{ $alert.Annotations.message }}
故障时间: {{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}
*****参考信息*****
{{ if gt (len $alert.Labels.instance) 0 -}}故障实例ip: {{ $alert.Labels.instance }};{{- end -}}
{{- if gt (len $alert.Labels.namespace) 0 -}}故障实例所在namespace: {{ $alert.Labels.namespace }};{{- end -}}
{{- if gt (len $alert.Labels.node) 0 -}}故障物理机ip: {{ $alert.Labels.node }};{{- end -}}
{{- if gt (len $alert.Labels.pod_name) 0 -}}故障pod名称: {{ $alert.Labels.pod_name }}{{- end }}
=====================
{{- end }}
{{- end }}



{{- if gt (len .Alerts.Resolved) 0 -}}
{{- range $index, $alert := .Alerts -}}
{{- if eq $index 0 -}}
=====================
告警类型: {{ $alert.Labels.alertname }}
告警级别: {{ $alert.Labels.severity }}

{{- end }}
*****告警详情*****
告警详情: {{ $alert.Annotations.message }}
故障时间: {{ $alert.StartsAt.Format "2006-01-02 15:04:05" }}
恢复时间: {{ $alert.EndsAt.Format "2006-01-02 15:04:05" }}
*****参考信息*****
{{ if gt (len $alert.Labels.instance) 0 -}}故障实例ip: {{ $alert.Labels.instance }};{{- end -}}
{{- if gt (len $alert.Labels.namespace) 0 -}}故障实例所在namespace: {{ $alert.Labels.namespace }};{{- end -}}
{{- if gt (len $alert.Labels.node) 0 -}}故障物理机ip: {{ $alert.Labels.node }};{{- end -}}
{{- if gt (len $alert.Labels.pod_name) 0 -}}故障pod名称: {{ $alert.Labels.pod_name }};{{- end }}
=====================
{{- end }}
{{- end }}
{{- end }}
EOF

cat > alertmanager.yaml<<EOF
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: 'xxxxxx@qq.com'
  smtp_auth_username: xxxxxx@qq.com
  smtp_auth_password: password #发送邮箱的授权码,非邮箱密码
  smtp_hello: 'qq.com'
  smtp_require_tls: false
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
  wechat_api_secret: 'xxxx-xxxxx-xxxxx'
  wechat_api_corp_id: 'xxxxxx'
# 自定义 通知的模板的 目录 或者 文件.
templates:
  - "/etc/alertmanager/config/wechat.tmpl" ##注意：此路径不能更改 
route:
  receiver: wechat
  group_by:
  - alertname
  - cluster
  - service
  group_wait: 30s
  group_interval: 1m  # 当第一个通知发送，等待多久发送压缩的警报
  repeat_interval: 5m #如果报警发送成功, 等待多久重新发送一次
  routes:
  - receiver: wechat 
    match_re:
      severity: critical
  - receiver: email 
    #match:
    #  alertname: CPUThrottlingHigh
    match_re:
      severity: warning 
  - receiver: "null"
    match:
      alertname: Watchdog

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    #equal: ['alertname', 'dev', 'instance']

receivers:
- name: email 
  email_configs:
  - to: 'xxxxxx@qq.com'
    headers: { Subject: "报警邮件" } 
    send_resolved: true
- name: 'dingtalk'
  webhook_configs:
  - url: 'http://dingtalk-hook:8060/dingtalk/webhook1/send'
    send_resolved: true
- name: 'wechat'
  wechat_configs:
  - corp_id: 'ww782826ac8ab69a1f'
    to_party: '1' #部门ID
    to_user: "@all"
    agent_id: '1000002'
    api_secret: 'xxxx-xxxxx-xxxxx' #api
    send_resolved: true
- name: "null"
EOF

kubectl delete secret alertmanager-main -n monitoring
kubectl create secret generic alertmanager-main --from-file=alertmanager.yaml  --from-file=wechat.tmpl  -n monitoring
```

## 钉钉监控告警

创建钉钉机器人，并生成token,参考 https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq
注意：自定义关键字需要添加Prometheus
修改 dingtalk-hook.yaml 中的token，换成你自己的

dingtalk.tmpl

```yaml
{{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
{{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}

{{ define "__text_alert_list" }}{{ range . }}
**Labels**
{{ range .Labels.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
{{ end }}
**Annotations**
{{ range .Annotations.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
{{ end }}
**Source:** [{{ .GeneratorURL }}]({{ .GeneratorURL }})
{{ end }}{{ end }}

{{ define "default.__text_alert_list" }}{{ range . }}
---
**告警级别:** {{ .Labels.severity | upper }}

**概览:** {{ .Annotations.summary }}

**Trigger Time:** {{ dateInZone "2006.01.02 15:04:05" (.StartsAt) "Asia/Shanghai" }}

**描述:** {{ .Annotations.description }}

**图表:** [查看图表]({{ .GeneratorURL }})

**详情:**
{{ range .Labels.SortedPairs }}{{ if and (ne (.Name) "severity") (ne (.Name) "summary") }}> - {{ .Name }}: {{ .Value | markdown | html }}
{{ end }}{{ end }}
{{ end }}
{{ end }}

{{/* Default */}}
{{ define "default.title" }}{{ template "__subject" . }}{{ end }}
{{ define "default.content" }}#### \[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}\] **[{{ index .GroupLabels "alertname" }}]({{ template "__alertmanagerURL" . }})**
{{ if gt (len .Alerts.Firing) 0 -}}
**发生告警**
{{ template "default.__text_alert_list" .Alerts.Firing }}
{{- end }}
{{ if gt (len .Alerts.Resolved) 0 -}}
**告警恢复**
{{ template "default.__text_alert_list" .Alerts.Resolved }}
{{- end }}
{{- end }}

{{/* Legacy */}}
{{ define "legacy.title" }}{{ template "__subject" . }}{{ end }}
{{ define "legacy.content" }}#### \[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}\] **[{{ index .GroupLabels "alertname" }}]({{ template "__alertmanagerURL" . }})**
{{ template "__text_alert_list" .Alerts.Firing }}
{{- end }}

{{/* Following names for compatibility */}}
{{ define "ding.link.title" }}{{ template "default.title" . }}{{ end }}
{{ define "ding.link.content" }}{{ template "default.content" . }}{{ end }}
```

dingtalk-config.yaml

```yaml
apiVersion: v1
data:
  dingding.tmpl: >-
    #告警消息标题
    title=prometheus
    {{ define "__subject" }}[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }} {{ if gt (len .CommonLabels) (len .GroupLabels) }}({{ with .CommonLabels.Remove .GroupLabels.Names }}{{ .Values | join " " }}{{ end }}){{ end }}{{ end }}
    {{ define "__alertmanagerURL" }}{{ .ExternalURL }}/#/alerts?receiver={{ .Receiver }}{{ end }}
    {{ define "__text_alert_list" }}{{ range . }}
    **Labels**
    {{ range .Labels.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
    {{ end }}
    **Annotations**
    {{ range .Annotations.SortedPairs }}> - {{ .Name }}: {{ .Value | markdown | html }}
    {{ end }}
    **Source:** [{{ .GeneratorURL }}]({{ .GeneratorURL }})
    {{ end }}{{ end }}
    {{ define "default.__text_alert_list" }}{{ range . }}
    #### \[{{ .Labels.severity | upper }}\] {{ .Annotations.summary }}
    **Description:** {{ .Annotations.description }}
    **Graph:** [📈]({{ .GeneratorURL }})
    **Details:**
    {{ range .Labels.SortedPairs }}{{ if and (ne (.Name) "severity") (ne (.Name) "summary") }}> - {{ .Name }}: {{ .Value | markdown | html }}
    {{ end }}{{ end }}
    {{ end }}{{ end }}
    {{/* Default */}}
    {{ define "default.title" }}{{ template "__subject" . }}{{ end }}
    {{ define "default.content" }}#### \[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}\] **[{{ index .GroupLabels "alertname" }}]({{ template "__alertmanagerURL" . }})**
    {{ if gt (len .Alerts.Firing) 0 -}}
    **Alerts Firing**
    {{ template "default.__text_alert_list" .Alerts.Firing }}
    {{- end }}
    {{ if gt (len .Alerts.Resolved) 0 -}}
    **Alerts Resolved**
    {{ template "default.__text_alert_list" .Alerts.Resolved }}
    {{- end }}
    {{- end }}
    {{/* Legacy */}}
    {{ define "legacy.title" }}{{ template "__subject" . }}{{ end }}
    {{ define "legacy.content" }}#### \[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}\] **[{{ index .GroupLabels "alertname" }}]({{ template "__alertmanagerURL" . }})**
    {{ template "__text_alert_list" .Alerts.Firing }}
    {{- end }}
    {{/* Following names for compatibility */}}
    {{ define "ding.link.title" }}{{ template "default.title" . }}{{ end }}
    {{ define "ding.link.content" }}{{ template "default.content" . }}{{ end }}
kind: ConfigMap
apiVersion: v1
metadata:
  name: dingding
  namespace: monitoring
```

dingtalk-hook.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dingtalk-hook
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: dingtalk-hook
  template:
    metadata:
      labels:
        app: dingtalk-hook
    spec:
      containers:
      - name: dingtalk-hook
        image: timonwong/prometheus-webhook-dingtalk:latest
        args:
          - '--web.listen-address=0.0.0.0:8060'
          - '--ding.profile=webhook1=https://oapi.dingtalk.com/robot/send?access_token=cee80c5dfff988b20ec626f54ac2fd099dfd9c227e1cfea778b1dfffa1025acc'
          - '--log.level=info'
          - '--web.enable-lifecycle'
          - '--web.enable-ui'
          - '--template.file=/etc/prometheus-webhook-dingtalk/templates/dingtalk.tmpl'
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8060
        resources:
          requests:
            cpu: 100m
            memory: 32Mi
          limits:
            cpu: 200m
            memory: 64Mi
        volumeMounts:
          - mountPath: /etc/prometheus-webhook-dingtalk/templates
            name: dingtalk-hook 
      volumes:
        - configMap:
            defaultMode: 420
            name: dingding
          name: dingtalk-hook 
---
apiVersion: v1
kind: Service
metadata:
  name: dingtalk-hook
  namespace: monitoring
spec:
  ports:
    - port: 8060
      protocol: TCP
      targetPort: 8060
      name: http
  selector:
    app: dingtalk-hook
  type: ClusterIP
```

alertmanager.yaml

```yaml
global:
  resolve_timeout: 5m
  #title: prometheus
# 自定义 通知的模板的 目录 或者 文件.
#templates:
#  - "/etc/alertmanager/config/dingtalk.tmpl" ##注意：此路径不能更改 
route:
  group_by:
  - alertname
  - cluster
  - service
  group_wait: 10m
  group_interval: 10s  # 当第一个通知发送，等待多久发送压缩的警报
  repeat_interval: 10m #如果报警发送成功, 等待多久重新发送一次
  receiver: dingtalk 
  routes:
  - receiver: dingtalk 
    match_re:
      severity: critical
  - receiver: "null"
    match:
      alertname: Watchdog
receivers:
- name: 'dingtalk'
  webhook_configs:
  - url: 'http://dingtalk-hook:8060/dingtalk/webhook1/send'
    send_resolved: true  
- name: 'null'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    #equal: ['alertname']
    #equal: ['alertname', 'dev', 'instance']
```

部署安装

```shell
kubectl delete configmap dingding -n monitoring >/dev/null 2>&1
kubectl create configmap dingding --from-file=dingtalk.tmpl -n monitoring
kubectl delete -f dingtalk-hook.yaml
kubectl apply -f dingtalk-hook.yaml

kubectl delete secret alertmanager-main -n monitoring
kubectl create secret generic alertmanager-main --from-file=alertmanager.yaml -n monitoring

kubectl logs -f `kubectl get pod -n monitoring |grep dingtalk |awk '{print $1}'` -n monitoring
```

参考：<https://github.com/chinaboy007/kube-prometheus>

## 监控mysql

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
            value: "username:password@(mysql.default.svc:3306)/"
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

cat >servicemonitor.yaml<<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: mysqld-exporter
    release: prometheus
  name: mysqld-exporter
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    # Mysqld Grafana 模版 ID 为 7362
    # 填写 service.yaml 中 Prometheus Exporter 对应的 Port 的 Name 字段的值
    port: mysqld-exporter
    #填写 Prometheus Exporter 代码中暴露的地址
    path: /metrics
  namespaceSelector:
    #any: true
    #service 所在命名空间
    matchNames:
    - default
  selector:
    matchLabels:
      app: mysqld-exporter
EOF

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f servicemonitor.yaml
```

登录grafana 导入模板 ID 为 7362
监控外部mysql

```shell
cat > service.yaml <<EOF
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-metrics
  namespace: monitoring
  labels:
    k8s-app: mysql-metrics
subsets:
- addresses:
    #外部exporter
    - ip: 10.100.196.167
  ports:
  - name: mysql-exporter
    port: 9104
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9104"
  name: mysql-metrics
  namespace: monitoring
  labels:
    k8s-app: mysql-metrics
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: mysql-exporter
    port: 9104
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mysql-metrics
  namespace: monitoring
  labels:
    app: mysql-metrics
    k8s-app: mysql-metrics
    prometheus: kube-prometheus
    release: kube-prometheus
spec:
  endpoints:
  - port: mysql-exporter
    interval: 15s
  selector:
    matchLabels:
      k8s-app: mysql-metrics
  namespaceSelector:
    matchNames:
    - monitoring
EOF
```

## 监控redis

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
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-exporter
  namespace: monitoring
  labels:
    app: redis-exporter
    prometheus: kube-prometheus
    release: kube-prometheus
spec:
  endpoints:
  - interval: 30s
    # Redis Grafana模版ID为763
    # 填写service.yaml中prometheus exporter对应的Port的Name字段的值
    port: redis-exporter
    # 填写Prometheus Exporter代码中暴露的地址
    path: /metrics
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      # 填写service.yaml的Label字段以定位目标service.yaml
      app: redis-exporter
EOF

kubectl apply -f redis-exporter.yaml
```

登录grafana导入模版ID为763

监控外部redis

```shell
cat >redis-out-exporter.yaml <<EOF
apiVersion: v1
kind: Endpoints
metadata:
  name: redis-metrics
  namespace: monitoring
  labels:
    k8s-app: redis-metrics
subsets:
- addresses:
    - ip: 10.100.196.168
  ports:
  - name: redis-exporter
    port: 9121
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
  name: redis-metrics
  namespace: monitoring
  labels:
    k8s-app: redis-metrics
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: redis-exporter
    port: 9121
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-metrics
  namespace: monitoring
  labels:
    app: redis-metrics
    k8s-app: redis-metrics
    prometheus: kube-prometheus
    release: kube-prometheus
spec:
  endpoints:
  - port: redis-exporter
    interval: 15s
  selector:
    matchLabels:
      k8s-app: redis-metrics
  namespaceSelector:
    matchNames:
    - monitoring
EOF

kubectl apply -f redis-out-exporter.yaml
```

## Prometheus Operator 高级配置

### 自动发现配置

我们想要在 Prometheus Operator 当中去自动发现并监控具有prometheus.io/scrape=true这个 annotations 的 Service。
之前我们的prometheus配置：

```shell
cd kube-prometheus/manifests/
cat > prometheus-additional.yaml << EOF
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
    replacement: \$1:\$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name
EOF
```

要想自动发现集群中的 Service，就需要我们在 Service 的annotation区域添加prometheus.io/scrape=true的声明，将上面文件直接保存为 prometheus-additional.yaml，然后通过这个文件创建一个对应的 Secret 对象：

```shell
kubectl create secret generic additional-configs --from-file=prometheus-additional.yaml -n monitoring
secret "additional-configs" created

kubectl get secret additional-configs -n monitoring -o yaml
```

然后我们只需要在声明 prometheus 的资源对象文件中添加上这个额外的配置：(prometheus-prometheus.yaml)

```yaml
securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  #新增下面三行
  additionalScrapeConfigs:
    name: additional-configs
    key: prometheus-additional.yaml
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
```

添加完成后，直接更新 prometheus 这个 CRD 资源对象：

```shell
kubectl apply -f prometheus-prometheus.yaml
prometheus.monitoring.coreos.com/k8s configured
```

隔一小会儿，可以前往 Prometheus 的 Dashboard 中查看配置是否生效：

- job_name: kubernetes-service-endpoints
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics

```shell
kubectl logs -f prometheus-k8s-0 prometheus -n monitoring
ts=2022-08-30T08:57:37.729Z caller=klog.go:116 level=error component=k8s_client_runtime func=ErrorDepth msg="pkg/mod/k8s.io/client-go@v0.24.3/tools/cache/reflector.go:167: Failed to watch *v1.Endpoints: failedto list *v1.Endpoints: endpoints is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list resource \"endpoints\" in API group \"\" at the cluster scope"
```

可以看到有很多错误日志出现，都是xxx is forbidden，这说明是 RBAC 权限的问题，通过 prometheus 资源对象的配置可以知道 Prometheus 绑定了一个名为 prometheus-k8s 的 ServiceAccount 对象，而这个对象绑定的是一个名为 prometheus-k8s 的 ClusterRole：（prometheus-clusterRole.yaml）

```shell
cat > prometheus-clusterRole.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s
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
EOF
kubectl apply -f prometheus-clusterRole.yaml
```

更新上面的 ClusterRole 这个资源对象，然后重建下 Prometheus 的所有 Pod，正常就可以看到 targets 页面下面有 kubernetes-service-endpoints 这个监控任务了：
kubernetes-service-endpoints (2/2 up)

在配置 service 时加入下面的配置内容

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
```

当然我们也可以用同样的方式去配置 Pod、Ingress 这些资源对象的自动发现。

