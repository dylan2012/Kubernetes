# Elasticsearch7 集群安装

## elasticsearch集群规划

Centos7.x 主机 内存2g+

|  主机名称   | IP地址  | 说明  |
|  ----  | ----  | ----  |
|node01 | 10.199.10.231 | 节点
|node02 |10.199.10.232 |节点
|node03 |10.199.10.233 |节点

## elasticsearch集群安装

### 安装前准备

官网下载地址
https://www.elastic.co/cn/downloads/elasticsearch
https://www.elastic.co/cn/downloads/past-releases/elasticsearch-7-17-6

所有节点

```shell
cat > /etc/security/limits.conf <<EOF
*       soft        core        unlimited
*       hard        core        unlimited
*       soft        nproc       1000000
*       hard        nproc       1000000
*       soft        nofile      1000000
*       hard        nofile      1000000
*       soft        memlock     32000
*       hard        memlock     32000
*       soft        msgqueue    8192000
EOF

echo 'vm.max_map_count = 655360'>>/etc/sysctl.conf
sysctl -p
```

### 主节点安装

```shell
mkdir -p /data/elasticsearch
mkdir -p /data/elasticsearch/logs
mkdir -p /data/elasticsearch/data
cd /data/elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.6-linux-x86_64.tar.gz
tar xf elasticsearch-7.17.6-linux-x86_64.tar.gz

#创建elasticsearch用户
useradd elasticsearch

cd /data/elasticsearch/elasticsearch-7.17.6/config

#配置jvm内存参数,这里设置为4G
vi jvm.options
-Xms4g
-Xmx4g

cp elasticsearch.yml elasticsearch.yml.bak
#更改elasticsearch配置文件
vi elasticsearch.yml
cluster.name: my-application
node.name: node-1
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["10.199.10.231", "10.199.10.232", "10.199.10.233"]
cluster.initial_master_nodes: ["node-1"]

chown -R elasticsearch:elasticsearch /data/elasticsearch

```

配置systemd管理

```shell
cat >/usr/lib/systemd/system/elasticsearch.service<<EOF
[Unit]
Description=elasticsearch
After=network.target

[Service]
Type=simple
User=elasticsearch
Group=elasticsearch
LimitNOFILE=100000
LimitNPROC=100000
Restart=no
ExecStart=/data/elasticsearch/elasticsearch-7.17.6/bin/elasticsearch
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start elasticsearch.service
systemctl enable elasticsearch.service
systemctl status elasticsearch.service
#systemctl stop elasticsearch.service
#启动检查
ss -lntup|grep 9200
ps -ef|grep elasticsearch
```

### 从节点安装

```shell
mkdir -p /data/elasticsearch/logs
mkdir -p /data/elasticsearch/data

cd /data/elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.6-linux-x86_64.tar.gz
tar xf elasticsearch-7.17.6-linux-x86_64.tar.gz

useradd elasticsearch
cd /data/elasticsearch/elasticsearch-7.17.6/config
#配置jvm内存参数,这里设置为4G
vi jvm.options
-Xms4g
-Xmx4g

cp elasticsearch.yml elasticsearch.yml.bak
#更改elasticsearch配置文件
vi elasticsearch.yml
cluster.name: my-application
node.name: node-2 #不同节点修改成不同的名称
path.data: /data/elasticsearch/data
path.logs: /data/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["10.199.10.231", "10.199.10.232", "10.199.10.233"]

chown -R elasticsearch:elasticsearch /data/elasticsearch
```

配置systemd管理

```shell
cat >/usr/lib/systemd/system/elasticsearch.service<<EOF
[Unit]
Description=elasticsearch
After=network.target

[Service]
Type=simple
User=elasticsearch
Group=elasticsearch
LimitNOFILE=100000
LimitNPROC=100000
Restart=no
ExecStart=/data/elasticsearch/elasticsearch-7.17.6/bin/elasticsearch
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start elasticsearch.service
systemctl enable elasticsearch.service
systemctl status elasticsearch.service
#systemctl stop elasticsearch.service
#启动检查
ss -lntup|grep 9200
ps -ef|grep elasticsearch
```

### 安装分词器ik

官网地址
https://github.com/medcl/elasticsearch-analysis-ik/releases

选择对应的ES版本号进行下载

所有主机

```shell
mkdir /data/elasticsearch/elasticsearch-7.17.6/plugins/ik
cd /data/elasticsearch/elasticsearch-7.17.6/plugins/ik
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.6/elasticsearch-analysis-ik-7.17.6.zip
unzip elasticsearch-analysis-ik-7.17.6.zip
chown -R elasticsearch:elasticsearch ../ik
#重启ES
systemctl restart elasticsearch
```

## 集群功能验证

```shell
#查看集群健康
curl http://10.199.10.231:9200/_cat/health?v
epoch      timestamp cluster        status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1666690807 09:40:07  my-application green           3         3      8   4    0    0        0             0                  -                100.0%

#查看节点信息
curl http://10.199.10.231:9200/_cat/nodes?v
ip            heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
10.199.10.231           26          98   0    0.14    0.13     0.09 cdfhilmrstw *      node-1
10.199.10.232            2          98   0    0.16    0.08     0.08 cdfhilmrstw -      node-2
10.199.10.233           22          98   0    0.18    0.14     0.10 cdfhilmrstw -      node-3

#创建一个索引
curl -X PUT 'http://10.199.10.231:9200/testindex?pretty'
#列出所有的索引
curl http://10.199.10.231:9200/_cat/indices?v
```
