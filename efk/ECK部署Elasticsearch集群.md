# ECK部署Elasticsearch集群

## 概述

Elastic Cloud on Kubernetes(ECK)是一个 Elasticsearch Operator，但远不止于此。 ECK 使用 Kubernetes Operator 模式构建而成，需要安装在您的 Kubernetes 集群内，其功能绝不仅限于简化 Kubernetes 上 Elasticsearch 和 Kibana 的部署工作这一项任务。ECK 专注于简化所有后期运行工作，例如：

- 管理和监测多个集群
- 轻松升级至新的版本
- 扩大或缩小集群容量
- 更改集群配置
- 动态调整本地存储的规模（包括 Elastic Local Volume（一款本地存储驱动器））
- 备份
  
ECK 不仅能自动完成所有运行和集群管理任务，还专注于简化在 Kubernetes 上使用 Elasticsearch 的完整体验。ECK 的愿景是为 Kubernetes 上的 Elastic 产品和解决方案提供 SaaS 般的体验。
在 ECK 上启动的所有 Elasticsearch 集群都默认受到保护，这意味着在最初创建的那一刻便已启用加密并受到默认强密码的保护。

通过 ECK 部署的所有集群都包括强大的基础（免费）级功能，例如可实现密集存储的冻结索引、Kibana Spaces、Canvas、Elastic Maps，等等。您甚至可以使用 Elastic Logs 和 Elastic Infrastructure 应用监测 Kubernetes 日志和基础设施。您可以获得在 Kubernetes 上使用 Elastic Stack 完整功能的体验。

ECK 内构建了 Elastic Local Volume，这是一个适用于 Kubernetes 的集成式存储驱动器。ECK 中还融入了很多最佳实践，例如在缩小规模之前对节点进行 drain 操作，在扩大规模的时候对分片进行再平衡，等等。从确保在配置变动过程中不会丢失数据，到确保在规模调整过程中实现零中断。

项目地址：https://github.com/elastic/cloud-on-k8s/
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html

## 安装ECK

### 安装CRS

```shell
kubectl create -f https://download.elastic.co/downloads/eck/2.4.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.4.0/operator.yaml
#查看
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
kubectl get pods -n elastic-system
```

### 部署Elasticsearch cluster

节点必须2G内存以上，要不会处于pending

```shell
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elastic
  namespace: elastic-system
spec:
  version: 8.4.1
  nodeSets:
  - name: default
    config:
      node.store.allow_mmap: false
    count: 1 #集群节点数量
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data # Do not change this name unless you set up a volume mount for the data path.
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 2Gi
        storageClassName: nfs-client
EOF

kubectl get elasticsearch -n elastic-system
NAME      HEALTH   NODES   VERSION   PHASE   AGE
elastic   green    1       8.4.1     Ready   13m

kubectl get service,pod -n elastic-system

PASSWORD=$(kubectl get secret -n elastic-system elastic-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
echo $PASSWORD
#curl -u "elastic:$PASSWORD" -k "https://elastic-es-http.elastic-system.svc:9200"

kubectl port-forward service/elastic-es-http 9200
curl -u "elastic:$PASSWORD" -k "https://localhost:9200"

{
  "name" : "elastic-es-default-0",
  "cluster_name" : "elastic",
  "cluster_uuid" : "WcGkdXMXTyqSH0XopTmBaw",
  "version" : {
    "number" : "8.4.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2bd229c8e56650b42e40992322a76e7914258f0c",
    "build_date" : "2022-08-26T12:11:43.232597118Z",
    "build_snapshot" : false,
    "lucene_version" : "9.3.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

### 部署Kibana

```shell
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana
  namespace: elastic-system
spec:
  version: 8.4.1
  count: 1
  elasticsearchRef:
    name: elastic
EOF

kubectl get kibana -n elastic-system
NAME     HEALTH   NODES   VERSION   AGE
kibana   green    1       8.4.1     115s

kubectl edit service kibana-kb-http -n elastic-system
#修改为 type: NodePort
kubectl get service kibana-kb-http -n elastic-system

# 默认用户： elastic 
# 密码
kubectl get secret -n elastic-system elastic-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```

浏览器打开 https://nodeIP:NodePort/ 
填入上面获取的用户名密码

或者我们可以去访问 kibana 来验证我们的集群，比如我们可以再添加一个 Ingress 对象：(ingress.yaml)

```shell
cat > ingress.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kibana
  namespace: elastic-system
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: kibana.com
    http:
      paths:
      - backend:
          serviceName: kibana-kibana
          servicePort: 5601
        path: /
EOF

kubectl create -f ingress.yaml
kubectl get ingress -n elastic-system
```

为了能够获得磁盘的最佳性能，ECK 支持每个节点使用 local volume，关于在 ECK 中使用 local volume 的方法可以查看下面几篇资料：

https://kubernetes.io/docs/concepts/storage/storage-classes
https://github.com/elastic/cloud-on-k8s/tree/master/local-volume
https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner

## 部署ApmServer

```shell
cat <<EOF | kubectl apply -f -
apiVersion: apm.k8s.elastic.co/v1
kind: ApmServer
metadata:
  name: apm-server-quickstart
  namespace: elastic-system
spec:
  version: 8.4.1
  count: 1
  elasticsearchRef:
    name: elastic
EOF

kubectl get apmservers -n elastic-system
```

## helm 安装 eck

```shell
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
helm install elastic-operator-crds elastic/eck-operator-crds

helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace \
  --set=installCRDs=false \
  --set=managedNamespaces='{namespace-a, namespace-b}' \
  --set=createClusterScopedResources=false \
  --set=webhook.enabled=false \
  --set=config.validateStorageClass=false

#查看配置
helm show values elastic/eck-operator
```
