# EFK 日志系统

## 概述

Kubernetes 中比较流行的日志收集解决方案是 Elasticsearch、Fluentd 和 Kibana（EFK）技术栈，也是官方现在比较推荐的一种方案。

Elasticsearch 是一个实时的、分布式的可扩展的搜索引擎，允许进行全文、结构化搜索，它通常用于索引和搜索大量日志数据，也可用于搜索许多不同类型的文档。

Elasticsearch 通常与 Kibana 一起部署，Kibana 是 Elasticsearch 的一个功能强大的数据可视化 Dashboard，Kibana 允许你通过 web 界面来浏览 Elasticsearch 日志数据。

Fluentd是一个流行的开源数据收集器，我们将在 Kubernetes 集群节点上安装 Fluentd，通过获取容器日志文件、过滤和转换日志数据，然后将数据传递到 Elasticsearch 集群，在该集群中对其进行索引和存储。

## 创建 Elasticsearch 集群

```shell
kubectl create ns logging

cat > elasticsearch-svc.yaml <<EOF
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node
EOF
kubectl create -f elasticsearch-svc.yaml

cat > elasticsearch-statefulset.yaml <<EOF
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: elasticsearch:7.17.6
        resources:
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
        ports:
        - containerPort: 9200
          name: rest
          protocol: TCP
        - containerPort: 9300
          name: inter-node
          protocol: TCP
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
        env:
          - name: cluster.name
            value: k8s-logs
          - name: node.name
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: discovery.seed_hosts
            value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
          - name: cluster.initial_master_nodes
            value: "es-cluster-0,es-cluster-1,es-cluster-2"
          - name: ES_JAVA_OPTS
            value: "-Xms512m -Xmx512m"
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      - name: increase-vm-max-map
        image: busybox
        command: ["sysctl", "-w", "vm.max_map_count=262144"]
        securityContext:
          privileged: true
      - name: increase-fd-ulimit
        image: busybox
        command: ["sh", "-c", "ulimit -n 65536"]
        securityContext:
          privileged: true
  volumeClaimTemplates:
  - metadata:
      name: data
      labels:
        app: elasticsearch
    spec:
      accessModes: [ "ReadWriteOnce" ]
      #使用nfs-client 可换成其它
      storageClassName: nfs-client
      resources:
        requests:
          storage: 10Gi
EOF

kubectl apply -f elasticsearch-statefulset.yaml
kubectl get sts -n logging
kubectl get pods -n logging

#验证
kubectl port-forward es-cluster-0 9200:9200 --namespace=logging
curl http://localhost:9200/_cluster/state?pretty
```

如果无法初始化请删除 fix-permissions 段落内容

## 创建 Kibana 服务

```shell
cat > kibana.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
  type: NodePort
  selector:
    app: kibana

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:7.17.6
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_URL
            value: http://elasticsearch:9200
        ports:
        - containerPort: 5601
EOF

kubectl apply -f kibana.yaml
kubectl get pods,svc -n logging
```

浏览器打 http://nodeip:nodeport

## 部署 Fluentd

Fluentd配置示例，作为参考

```shell
cat >fluentd-configmap.yaml <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config
  namespace: logging
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  #------系统配置参数-----
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
  #------Kubernetes 容器日志收集配置------
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*
      read_from_head true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          #time_format %Y-%m-%dT%H:%M:%S.%N%:z
          time_format %Y-%m-%dT%H:%M:%S.%L%z
        </pattern>
      </parse>
    </source>
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>
    <filter **>
      @id filter_concat
      @type concat
      key message
      multiline_end_regexp /\n$/
      separator ""
    </filter>
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>
    <filter kubernetes.**>
      @id filter_parser
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type multi_format
        <pattern>
          format json
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
    # 删除一些多余的属性
    <filter kubernetes.**>
      @type record_transformer
      remove_keys $.docker.container_id,$.kubernetes.container_image_id,$.kubernetes.pod_id,$.kubernetes.namespace_id,$.kubernetes.master_url,$.kubernetes.labels.pod-template-hash
    </filter>
    # 只保留具有logging=true标签的Pod日志
    <filter kubernetes.**>
      @id filter_log
      @type grep
      <regexp>
        key $.kubernetes.labels.logging
        pattern ^true$
      </regexp>
    </filter>
  #------系统日志收集-------
  system.input.conf: |-
    <source>
      @id journald-docker
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "docker.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-docker.pos
      </storage>
      read_from_head true
      tag docker
    </source>
    <source>
      @id journald-container-runtime
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "{{ fluentd_container_runtime_service }}.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-container-runtime.pos
      </storage>
      read_from_head true
      tag container-runtime
    </source>
    <source>
      @id journald-kubelet
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "kubelet.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-kubelet.pos
      </storage>
      read_from_head true
      tag kubelet
    </source>
    <source>
      @id journald-node-problem-detector
      @type systemd
      matches [{ "_SYSTEMD_UNIT": "node-problem-detector.service" }]
      <storage>
        @type local
        persistent true
        path /var/log/journald-node-problem-detector.pos
      </storage>
      read_from_head true
      tag node-problem-detector
    </source>
    <source>
      @id kernel
      @type systemd
      matches [{ "_TRANSPORT": "kernel" }]
      <storage>
        @type local
        persistent true
        path /var/log/kernel.pos
      </storage>
      <entry>
        fields_strip_underscores true
        fields_lowercase true
      </entry>
      read_from_head true
      tag kernel
    </source>
  #------输出到 ElasticSearch 配置------
  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      type_name _doc
      include_tag_key true
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      user "#{ENV['FLUENT_ELASTICSEARCH_USER']}"
      password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD']}"
      logstash_format true
      logstash_prefix kubernetes
      logst
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 5
        flush_interval 8s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 5M
        queue_limit_length 10
        overflow_action block
        compress gzip               #开启gzip提高日志采集性能
      </buffer>
    </match>
EOF

#kubectl apply -f fluentd-configmap.yaml

cat > fluentd-rbac.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  namespace: logging
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
EOF
kubectl apply -f fluentd-rbac.yaml

cat > fluentd-daemonset.yaml <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-es
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccountName: fluentd-es
      containers:
      - name: fluentd-es
        image: fluent/fluentd-kubernetes-daemonset:v1.15-debian-elasticsearch7-1
        env:
          - name:  FLUENT_ELASTICSEARCH_HOST
            value: "elasticsearch.logging.svc.cluster.local"
          - name:  FLUENT_ELASTICSEARCH_PORT
            value: "9200"
          - name: FLUENT_ELASTICSEARCH_SCHEME
            value: "http"
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: cri
          - name: FLUENT_CONTAINER_TAIL_PARSER_TIME_FORMAT
            value: "%Y-%m-%dT%H:%M:%S.%L%z"
          - name: FLUENTD_SYSTEMD_CONF
            value: disable
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        #- name: config-volume
        #  mountPath: /fluent/etc/conf.d
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      #- name: config-volume
      #  configMap:
      #    name: fluentd-es-config
EOF

kubectl apply -f fluentd-daemonset.yaml

#给需要日志收集的节点打label
#kubectl label node node01 beta.kubernetes.io/fluentd-ds-ready=true
kubectl get nodes --show-labels

kubectl get pods -n logging
```

Fluentd 启动成功后，我们可以前往 Kibana 的 Dashboard 页面中，点击左侧的Discover

在这里可以配置我们需要的 Elasticsearch 索引，前面 Fluentd 配置文件中我们采集的日志使用的是 logstash 格式，这里只需要在文本框中输入logstash-*即可匹配到 Elasticsearch 集群中的所有日志数据，然后点击下一步，进入以下页面：

### 工作原理

### 配置说明
