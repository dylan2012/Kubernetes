# k8s的DevOps环境构建

## 1.下载并部署 gitlab

最低要求4G内存服务器

```shell
docker pull gitlab/gitlab-ce:latest
docker run -d -p 8090:80 -p 8443:443 -p 8222:22 --name gitlab --restart always --privileged=true -v /data/gitlab/etc:/etc/gitlab -v /data/gitlab/log:/var/log/gitlab -v /data/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce:latest

#修改配置
vi /var/opt/gitlab/gitlab-rails/etc/gitlab.yml
gitlab:
    ## Web server settings (note: host is the FQDN, do not include http://)
    host: 10.199.10.241
    port: 8090
```

最后执行"gitlab-ctl reconfigure"命令使之配置生效（注意最好不要执行"gitlab-ctl restart",只执行本命令即可）

## 2.harbor安装部署

下载地址：https://github.com/vmware/harbor/releases

安装文档： https://goharbor.io/docs/

### 2.1 安装部署

由于Harbor是采用docker-compose一键部署的，所以Harbor服务器也需要安装Docker以及docker-compose

```shell
tar xf harbor-offline-installer-v2.5.1.tgz  -C /usr/local/
cp /usr/local/harbor/harbor.yml.tmpl /usr/local/harbor/harbor.yml

vi /usr/local/harbor/harbor.yml
hostname: 192.168.10.15  #harbor服务器地址
harbor_admin_password: Harbor12345
data_volume: /data/harbor   #数据目录需要手动创建
#注意除了这三项外还需要注释https的全部注释
cd /usr/local/harbor/
./prepare
./install.sh
```

### 2.2 配置docker/continerd

docker配置

```shell
cat >/etc/docker/daemon.json<<EOF
{
   "exec-opts": ["native.cgroupdriver=systemd"],
   "insecure-registries": ["10.199.10.237","harbor"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

continerd配置

```shell
vi /etc/containerd/config.toml
#增加后面
[plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.199.10.237".tls]
          insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.199.10.237".auth]
           username = "admin"
           password = "Harbor12345"

systemctl daemon-reload
systemctl restart containerd
```

## 3.部署jenkins

### 3.1 部署jenkins

需事先部署docker服务。

```shell
mkdir /data/jenkins_data -p
chmod -R 777 /data/jenkins_data

docker pull jenkins/jenkins:latest
docker images | grep jenkins
cat > docker-compose.yml <<-EOF
version: '3'
services:
  docker_jenkins:
    user: root
    restart: always
    privileged: true
    network_mode: host
    image: jenkins/jenkins:latest
    container_name: jenkins
    ports:
      - 8080:8080
      - 50000:50000
    volumes:
      - /data/jenkins_data:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
EOF
docker-compose up -d
```

### 3.2 配置 Jenkins

#### 3.2.1 初始化

在浏览器访问 jenkins 的 web 界面：
http://10.199.10.241:8080/login?from=%2F

获取初始密码

```shell
cat /data/jenkins_data/secrets/initialAdminPassword
docker logs -f jenkins
```

选择安装推荐的插件

完成安装后进入操作界面
Manage Jnekins------>插件管理------>可选插件------>搜索 kubernetes
Manage Jnekins------>插件管理------>可选插件------>搜索 blueocean

需要安装的插件有：

```html
Git
Git Parameter
Git Pipeline for Blue Ocean
GitLab
Credentials
Credentials Binding
Blue Ocean
Blue Ocean Pipeline Editor
Blue Ocean Core JS
Pipeline SCM API for Blue Ocean
Dashboard for Blue Ocean
Build With Parameters
Dynamic Extended Choice Parameter Plug-In
Dynamic Parameter Plug-in
Extended Choice Parameter
List Git Branches Parameter
Pipeline
Pipeline: Declarative
Kubernetes
Kubernetes CLI
Kubernetes Credentials
Image Tag Parameter
Active Choices
GitLab Authentication
Gitlab API
Generic Webhook Trigger
```

重启jenkins

#### 3.2.2 配置 jenkins 连接到我们存在的 k8s 集群

##### 1.配置凭证

Harbor、GitLab 添加凭证  Add Credentials
选择Username with password
Username：Harbor 或者其它平台的用户名；
Password：Harbor 或者其它平台的密码；
ID：该凭证的 ID；
Description：证书的描述。

gitlab可以选择添加SSH Username with private key
找到jenkins主机的~/.ssh/id_rsa

Kubernetes 的证书
选择 Secret file
File：KUBECONFIG 文件或其它加密文件； ~/.kube/config
ID：该凭证的 ID；
Description：证书的描述。

##### 2.jenkins接入Kubernetes

master节点

```shell
kubectl create ns jenkins-k8s
kubectl create sa jenkins-k8s-sa -n jenkins-k8s
kubectl create clusterrolebinding jenkins-k8s-sa-cluster -n jenkins-k8s --clusterrole=cluster-admin --serviceaccount=jenkins-k8s:jenkins-k8s-sa

#实际使用时，可以选择任意的一个或多个节点作为创建 Slave Pod 的节点
kubectl label node node01 build=true
```

通常情况下，Jenkins Slave 会通过 Jenkins Master 节点的 50000 端口与之通信，所以需要开启 Agent的50000端口。

打开http://10.199.10.241:8080/configureClouds/

配置集群
名称：kubernetes
Kubernetes 地址：https://10.199.10.230:16443
Jenkins 地址：http://10.199.10.241:8080/
Kubernetes 命名空间：jenkins-k8s
凭据：config(kubernetes-cluster)

#### 3.2.3 测试

首页-新建任务
"test-slave-job"
选择"流水线"

在 "流水线"-"脚本"

```html
pipeline {
  agent {
    kubernetes { 
      cloud 'kubernetes'  
      slaveConnectTimeout 1200 
      workspaceVolume emptyDirWorkspaceVolume() 
      yaml '''
kind: Pod
metadata:
  name: jenkins-agent
  namespace: jenkins-k8s
spec:
  containers: 
  - args: [\'$(JENKINS_SECRET)\', \'$(JENKINS_NAME)\']
    image: '10.199.10.237/test/jenkins-jnlp:jdk11'
    name: jnlp  
    imagePullPolicy: IfNotPresent
  - command:
      - "cat"
    image: "10.199.10.237/bash/docker:alpine"
    imagePullPolicy: "IfNotPresent"
    name: "docker"
    tty: true
    volumeMounts:
    - mountPath: "/var/run/docker.sock"
      name: "dockersock"
      readOnly: false
  volumes:   
  #注意docker容器必须被调度到存在docker的k8s节点，并且挂载主机的docker.sock文件到容器
  - hostPath:
      path: "/var/run/docker.sock"
    name: "dockersock"
  restartPolicy: Never 
  nodeSelector:  
    build: true 
'''
    }
  }
  stages {
    stage('docker info') {   
      steps {
        container(name: 'docker') {  
          sh "docker info"    //执行docker info有正常输出即可
        }
      }
    }
  stage('kubectl get') {   //stage名称
      environment {
        MY_KUBECONFIG = credentials('kubernetes-cluster')
      }
      steps {
        container(name: 'docker') {   //这里定义这个步骤使用那个container进行执行，指定container的名称
          sh "kubectl get pod -A --kubeconfig $MY_KUBECONFIG"
        }
      }
    }
  }
}
```

保存-立即构建
查看是否执行

```shell
kubectl get pods -n jenkins-k8s
```

#### 3.2.4 结合gitlab自动化构建

##### 1.Jenkins配置

新建item 选择 流水线

构建触发器 - Build when a change is pushed to GitLab. GitLab webhook URL: http://

高级 - Generate 生成 Secret token

记录 webhook URL 和 Secret token

流水线 -- 定义 -- Pipeline script from SCM
SCM git

Repositories: <gitlab地址>

Branches to build - 指定分支（为空时代表any）

脚本路径：Jenkinsfile

保存

##### 2.gitlab配置

项目-配置-webhooks

填入上面获取的GitLab webhook URL: http://

token
