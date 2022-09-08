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

fluentd是一个针对日志的收集、处理、转发系统。通过丰富的插件系统， 可以收集来自于各种系统或应用的日志，转化为用户指定的格式后，转发到用户所指定的日志存储系统之中。
Fluentd 是一个高效的日志聚合器，是用 Ruby 编写的，并且可以很好地扩展。对于大部分企业来说，Fluentd 足够高效并且消耗的资源相对较少，另外一个工具Fluent-bit更轻量级，占用资源更少，但是插件相对 Fluentd 来说不够丰富，所以整体来说，Fluentd 更加成熟，使用更加广泛，所以我们这里也同样使用 Fluentd 来作为日志收集工具。

**[Fluentd官方文档](https://docs.fluentd.org/)**
**[基于kubernetes的fluentd镜像](https://github.com/fluent/fluentd-kubernetes-daemonset)**
**[fluentd正则匹配](https://rubular.com/)**
**[cri日志格式解析器](https://github.com/fluent/fluent-plugin-parser-cri#log-and-configuration-example)**

fluentd部署

```shell
#RBAC授权
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

#部署镜像
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
#kubectl label node node01 logging=true
kubectl get nodes --show-labels

kubectl get pods -n logging
```

Fluentd 启动成功后，我们可以前往 Kibana 的 Dashboard 页面中，点击左侧的Discover

在这里可以配置我们需要的 Elasticsearch 索引，前面 Fluentd 配置文件中我们采集的日志使用的是 logstash 格式，这里只需要在文本框中输入logstash-*即可匹配到 Elasticsearch 集群中的所有日志数据，然后点击下一步

日志多行解析参考：<https://www.qikqiak.com/post/collect-multiline-logs/>
参考内容：
<https://www.qikqiak.com/post/install-efk-stack-on-k8s/>
<https://bbs.huaweicloud.com/blogs/303136>

### 配置说明

Fluentd配置示例

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-es-config-v0.2.0
  namespace: kube-system
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  ###### 系统配置，默认即可 #######
  system.conf: |-
    <system>
      root_dir /tmp/fluentd-buffers/
    </system>
    
  ###### 容器日志—收集配置 #######    
  containers.input.conf: |-
    # ------采集 Kubernetes 容器日志-------
    <source>                                  
      @id fluentd-containers.log
      @type tail                              #---Fluentd 内置的输入方式，其原理是不停地从源文件中获取新的日志。
      path /var/log/containers/*.log          #---挂载的服务器Docker容器日志地址
      pos_file /var/log/es-containers.log.pos
      tag raw.kubernetes.*                    #---设置日志标签
      read_from_head true
      <parse>                                 #---多行格式化成JSON
        @type multi_format                    #---使用multi-format-parser解析器插件
        <pattern>
          format json                         #---JSON解析器
          time_key time                       #---指定事件时间的时间字段
          time_format %Y-%m-%dT%H:%M:%S.%NZ   #---时间格式
        </pattern>
        <pattern>
          format /^(?<time>.+) (?<stream>stdout|stderr) [^ ]* (?<log>.*)$/
          time_format %Y-%m-%dT%H:%M:%S.%N%:z
        </pattern>
      </parse>
    </source>
    
    # -----检测Exception异常日志连接到一条日志中------
    # 关于插件请查看地址：https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions
    <match raw.kubernetes.**>     #---匹配tag为raw.kubernetes.**日志信息
      @id raw.kubernetes
      @type detect_exceptions     #---使用detect-exceptions插件处理异常栈信息，放置异常只要一行而不完整
      remove_tag_prefix raw       #---移出raw前缀
      message log                 #---JSON记录中包含应扫描异常的单行日志消息的字段的名称。
                                  #   如果将其设置为''，则插件将按此顺序尝试'message'和'log'。
                                  #   此参数仅适用于结构化（JSON）日志流。默认值：''。
      stream stream               #---JSON记录中包含“真实”日志流中逻辑日志流名称的字段的名称。
                                  #   针对每个逻辑日志流单独处理异常检测，即，即使逻辑日志流 的
                                  #   消息在“真实”日志流中交织，也将检测到异常。因此，仅组合相
                                  #   同逻辑流中的记录。如果设置为''，则忽略此参数。此参数仅适用于
                                  #   结构化（JSON）日志流。默认值：''。
      multiline_flush_interval 5  #---以秒为单位的间隔，在此之后将转发（可能尚未完成）缓冲的异常堆栈。
                                  #   如果未设置，则不刷新不完整的异常堆栈。
      max_bytes 500000
      max_lines 1000
    </match>
  
    # -------日志拼接-------
    <filter **>
      @id filter_concat
      @type concat                #---Fluentd Filter插件，用于连接多个事件中分隔的多行日志。
      key message
      multiline_end_regexp /\n$/  #---以换行符“\n”拼接
      separator ""
    </filter> 
  
    # ------过滤Kubernetes metadata数据使用pod和namespace metadata丰富容器日志记录-------
    # 关于插件请查看地址：https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter
    <filter kubernetes.**>
      @id filter_kubernetes_metadata
      @type kubernetes_metadata
    </filter>
    
    # ------修复ElasticSearch中的JSON字段------
    # 关于插件请查看地址：https://github.com/repeatedly/fluent-plugin-multi-format-parser
    <filter kubernetes.**>
      @id filter_parser
      @type parser                #---multi-format-parser多格式解析器插件
      key_name log                #---在要解析的记录中指定字段名称。
      reserve_data true           #---在解析结果中保留原始键值对。
      remove_key_name_field true  #---key_name解析成功后删除字段。
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
    
  ###### Kuberntes集群节点机器上的日志收集 ######    
  system.input.conf: |-
    # ------Kubernetes minion节点日志信息，可以去掉------
    #<source>
    #  @id minion
    #  @type tail
    #  format /^(?<time>[^ ]* [^ ,]*)[^\[]*\[[^\]]*\]\[(?<severity>[^ \]]*) *\] (?<message>.*)$/
    #  time_format %Y-%m-%d %H:%M:%S
    #  path /var/log/salt/minion
    #  pos_file /var/log/salt.pos
    #  tag salt
    #</source>
    
    # ------启动脚本日志，可以去掉------
    # <source>
    #   @id startupscript.log
    #   @type tail
    #   format syslog
    #   path /var/log/startupscript.log
    #   pos_file /var/log/es-startupscript.log.pos
    #   tag startupscript
    # </source>
    
    # ------Docker 程序日志，可以去掉------
    # <source>
    #   @id docker.log
    #   @type tail
    #   format /^time="(?<time>[^)]*)" level=(?<severity>[^ ]*) msg="(?<message>[^"]*)"( err="(?<error>[^"]*)")?( statusCode=($<status_code>\d+))?/
    #   path /var/log/docker.log
    #   pos_file /var/log/es-docker.log.pos
    #   tag docker
    # </source>
    
    #------ETCD 日志，因为ETCD现在默认启动到容器中，采集容器日志顺便就采集了，可以去掉------
    # <source>
    #   @id etcd.log
    #   @type tail
    #   # Not parsing this, because it doesn't have anything particularly useful to
    #   # parse out of it (like severities).
    #   format none
    #   path /var/log/etcd.log
    #   pos_file /var/log/es-etcd.log.pos
    #   tag etcd
    # </source>
    
    #------Kubelet 日志------
    # <source>
    #   @id kubelet.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kubelet.log
    #   pos_file /var/log/es-kubelet.log.pos
    #   tag kubelet
    # </source>
    
    #------Kube-proxy 日志------
    # <source>
    #   @id kube-proxy.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-proxy.log
    #   pos_file /var/log/es-kube-proxy.log.pos
    #   tag kube-proxy
    # </source>
    
    #------kube-apiserver日志------
    # <source>
    #   @id kube-apiserver.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-apiserver.log
    #   pos_file /var/log/es-kube-apiserver.log.pos
    #   tag kube-apiserver
    # </source>

    #------Kube-controller日志------
    # <source>
    #   @id kube-controller-manager.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-controller-manager.log
    #   pos_file /var/log/es-kube-controller-manager.log.pos
    #   tag kube-controller-manager
    # </source>
    
    #------Kube-scheduler日志------
    # <source>
    #   @id kube-scheduler.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/kube-scheduler.log
    #   pos_file /var/log/es-kube-scheduler.log.pos
    #   tag kube-scheduler
    # </source>
    
    #------glbc日志------
    # <source>
    #   @id glbc.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/glbc.log
    #   pos_file /var/log/es-glbc.log.pos
    #   tag glbc
    # </source>
    
    #------Kubernetes 伸缩日志------
    # <source>
    #   @id cluster-autoscaler.log
    #   @type tail
    #   format multiline
    #   multiline_flush_interval 5s
    #   format_firstline /^\w\d{4}/
    #   format1 /^(?<severity>\w)(?<time>\d{4} [^\s]*)\s+(?<pid>\d+)\s+(?<source>[^ \]]+)\] (?<message>.*)/
    #   time_format %m%d %H:%M:%S.%N
    #   path /var/log/cluster-autoscaler.log
    #   pos_file /var/log/es-cluster-autoscaler.log.pos
    #   tag cluster-autoscaler
    # </source>

    # -------来自system-journal的日志------
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

    # -------Journald-container-runtime日志信息------
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
    
    # -------Journald-kubelet日志信息------
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
    
    # -------journald节点问题检测器------
    #关于插件请查看地址：https://github.com/reevoo/fluent-plugin-systemd
    #systemd输入插件，用于从systemd日志中读取日志
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
    
    # -------kernel日志------ 
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
    
  ###### 监听配置，一般用于日志聚合用 ######
  forward.input.conf: |-
    #监听通过TCP发送的消息
    <source>
      @id forward
      @type forward
    </source>

  ###### Prometheus metrics 数据收集 ######
  monitoring.conf: |-
    # input plugin that exports metrics
    # 输出 metrics 数据的 input 插件
    <source>
      @id prometheus
      @type prometheus
    </source>
    <source>
      @id monitor_agent
      @type monitor_agent
    </source>
    # 从 MonitorAgent 收集 metrics 数据的 input 插件 
    <source>
      @id prometheus_monitor
      @type prometheus_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # ------为 output 插件收集指标的 input 插件------
    <source>
      @id prometheus_output_monitor
      @type prometheus_output_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>
    # ------为in_tail 插件收集指标的input 插件------
    <source>
      @id prometheus_tail_monitor
      @type prometheus_tail_monitor
      <labels>
        host ${hostname}
      </labels>
    </source>

  ###### 输出配置，在此配置输出到ES的配置信息 ######
  # ElasticSearch fluentd插件地址：https://docs.fluentd.org/v1.0/articles/out_elasticsearch
  output.conf: |-
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info     #---指定日志记录级别。可设置为fatal，error，warn，info，debug，和trace，默认日志级别为info。
      type_name _doc
      include_tag_key true              #---将 tag 标签的 key 到日志中。
      #host elasticsearch-logging        #---指定 ElasticSearch 服务器地址。
      #port 9200                         #---指定 ElasticSearch 端口号。
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      user "#{ENV['FLUENT_ELASTICSEARCH_USER']}"
      password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD']}"
      #index_name fluentd.${tag}.%Y%m%d #---要将事件写入的索引名称（默认值:) fluentd。
      logstash_format true              #---使用传统的索引名称格式logstash-%Y.%m.%d，此选项取代该index_name选项。
      #logstash_prefix logstash         #---用于logstash_format指定为true时写入logstash前缀索引名称，默认值:logstash。
      <buffer>
        @type file                      #---Buffer 插件类型，可选file、memory
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff  #---重试模式，可选为exponential_backoff、periodic。
                                        #   exponential_backoff 模式为等待秒数，将在每次失败时成倍增长
        flush_thread_count 2
        flush_interval 10s
        retry_forever
        retry_max_interval 30           #---丢弃缓冲数据之前的尝试的最大间隔。
        chunk_limit_size 5M             #---每个块的最大大小：事件将被写入块，直到块的大小变为此大小。
        queue_limit_length 8            #---块队列的长度。
        overflow_action block           #---输出插件在缓冲区队列已满时的行为方式，有throw_exception、block、
                                        #   drop_oldest_chunk，block方式为阻止输入事件发送到缓冲区。
        compress gzip               #开启gzip提高日志
      </buffer>
    </match>
```
