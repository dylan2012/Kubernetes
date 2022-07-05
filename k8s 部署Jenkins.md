# k8s 部署Jenkins

## 1.安装nfs

k8s节点都执行

```shell
yum install nfs-utils -y
systemctl start nfs
systemctl enable nfs

mkdir /data/jenkins -p
echo '/data/jenkins 192.168.10.0/24(rw,no_root_squash)' >> /etc/exports
exportfs -arv
systemctl restart nfs
```

## 2.在 kubernetes 中部署 jenkins

### 2.1 创建PVC

在master节点

```shell
kubectl create namespace jenkins-k8s
cd /data/
cat >>pv.yaml<<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-k8s-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: 192.168.10.51 #替换IP
    path: /data/jenkins
EOF
kubectl apply -f pv.yaml
kubectl get pv

cat >>pvc.yaml<<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-k8s-pvc
  namespace: jenkins-k8s
spec:
  resources:
    requests:
      storage: 10Gi
  accessModes:
  - ReadWriteMany
EOF
kubectl apply -f pvc.yaml
#查看 pvc 是否创建成功
kubectl get pvc -n jenkins-k8s
```

### 2.2 Jenkins部署

```shell
kubectl create sa jenkins-k8s-sa -n jenkins-k8s
kubectl create clusterrolebinding jenkins-k8s-sa-cluster -n jenkins-k8s --clusterrole=cluster-admin --serviceaccount=jenkins-k8s:jenkins-k8s-sa

cat >>jenkins-deployment.yaml<<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: jenkins
  namespace: jenkins-k8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccount: jenkins-k8s-sa
      containers:
      - name: jenkins
        image:  jenkins/jenkins:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 5
          failureThreshold: 12
        volumeMounts:
        - name: jenkins-volume
          subPath: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-volume
        persistentVolumeClaim:
          claimName: jenkins-k8s-pvc
EOF
kubectl apply -f jenkins-deployment.yaml
#查看 jenkins 是否创建成功
kubectl get pods -n jenkins-k8s
#POD出现CrashLoopBackOff 状态
kubectl delete -f jenkins-deployment.yaml
chown -R 1000.1000 /data/jenkins
kubectl apply -f jenkins-deployment.yaml
#查看 jenkins 是否创建成功
kubectl get pods -n jenkins-k8s
```

把 jenkins 前端加上 service，提供外部网络访问

```shell

cat >>jenkins-service.yaml<<EOF
apiVersion: v1
kind: Service
metadata:
  name: jenkins-service
  namespace: jenkins-k8s
  labels:
    app: jenkins
spec:
  selector:
    app: jenkins
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
    nodePort: 30008
  - name: agent
    port: 50000
    targetPort: agent
EOF
kubectl apply -f jenkins-service.yaml
kubectl get svc -n jenkins-k8s
```

### 2.3 配置 Jenkins

#### 2.3.1 初始化

在浏览器访问 jenkins 的 web 界面：
http://192.168.10.51:30008/login?from=%2F

在 nfs 服务端，也就是我们的 master01 节点获取密码

```shell
cat /data/jenkins/jenkins-home/secrets/initialAdminPassword
#或者查看pod
kubectl logs -n jenkins-k8s jenkins-59fb776588-lngpl 
```

选择安装推荐的插件

完成安装后进入操作界面
Manage Jnekins------>插件管理------>可选插件------>搜索 kubernetes
Manage Jnekins------>插件管理------>可选插件------>搜索 blueocean

重启jenkins
http://192.168.10.51:30008/restart

#### 2.3.2 配置 jenkins 连接到我们存在的 k8s 集群

#### 1.jenkins安装在k8s

访问 http://192.168.10.51:30008/configureClouds/

新增一个云,在下拉菜单中选择 kubernets 并添加
填写云 kubernetes 配置内容 - "Kubernetes Cloud details"

名称：kubernetes
Kubernetes 地址：https://kubernetes.default.svc.cluster.local
Kubernetes 命名空间：jenkins-k8s

测试 jenkins 和 k8s 是否可以通信 点击 “连接测试”
Connected to Kubernetes v1.24.1
说明测试成功， Jenkins 可以和 k8s 进行通信

Jenkins 地址：http://jenkins-service.jenkins-k8s.svc.cluster.local:8080

应用------>保存

#### 2.jenkins是外部接入

在master节点

```shell
cat /root/.kube/config
#替换<>值
echo <certificate-authority-data>|base64 -d >ca.crt
echo <client-key-data>|base64 -d >client.key
echo <client-certificate-data>|base64 -d >client.crt
openssl pkcs12 -export -out cert.pfx -inkey client.key -in client.crt -certfile ca.crt
#输入密码并记住
```

添加凭据选择Certificate 把生成的cert.pfx上传并填入密码
Kubernetes 地址：https://192.168.10.51:16443
Jenkins 地址：http://192.168.10.55:8080/
Kubernetes 命名空间：jenkins-k8s

#### 2.3.3 配置 pod-template

点击"Pod-Templates"-"添加 Pod 模板"-"Pod Template detail"

名称: jnlp
命名空间: jenkins-k8s
标签列表: my-jnlp #后面pipline会使用到
用法: 只允许运行绑定到这台机器的Job

点击"添加容器"

名称: jnlp
Docker 镜像: cnych/jenkins:jnlp6
运行的命令: 清空
命令参数: 清空
勾选"分配伪终端"

另外需要注意我们这里需要在下面挂载两个主机目录，一个是/var/run/docker.sock，该文件是用于 Pod 中的容器能够共享宿主机的 Docker，这就是大家说的 docker in docker 的方式，Docker 二进制文件我们已经打包到上面的镜像中了，另外一个目录下/root/.kube目录，我们将这个目录挂载到容器的/root/.kube目录下面这是为了让我们能够在 Pod 的容器中能够使用 kubectl 工具来访问我们的 Kubernetes 集群，方便我们后面在 Slave Pod 部署 Kubernetes 应用。

卷-挂载到 Pod 代理中的卷列表
点击 “添加卷” 选择 "Host Path Volume"

主机路径: /var/run/docker.sock
挂载路径: /var/run/docker.sock

主机路径: /root/.kube
挂载路径: /root/.kube

最后 填入上面创建的sa
Service Account: jenkins-k8s-sa

应用-保存

#### 2.3.4 测试

首页-新建任务
"test-slave-job"
选择"构建一个自由风格的软件项目"

限制项目的运行节点-标签表达式
"my-jnlp"

在 "构建" 区域选择 "执行 shell"

```shell
echo "测试 Kubernetes 动态生成 jenkins slave"
echo "==============docker in docker==========="
docker info

echo "=============kubectl============="
kubectl get pods
```

保存
查看是否执行

```shell
kubectl get pods -n jenkins-k8s
```

harbor流水线测试
首页-新建任务
"test-slave-pipeline-job"
选择"流水线"
流水线-脚本

```shell

node('my-jnlp') {
    stage('Clone') {
        echo "1.Clone Stage"
        git url: "https://github.com/luckylucky421/jenkins-sample.git"
        script {
            build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        }
    }
    stage('Test') {
      echo "2.Test Stage"

    }
    stage('Build') {
        echo "3.Build Docker Image Stage"
        sh "docker build -t 192.168.10.56/jenkins-demo/jenkins-demo:${build_tag} ."
    }
    stage('Push') {
        echo "4.Push Docker Image Stage"
        withCredentials([usernamePassword(credentialsId: 'dockerharbor', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
            sh "docker login 192.168.10.56 -u ${dockerHubUser} -p ${dockerHubPassword}"
            sh "docker push 192.168.10.56/jenkins-demo/jenkins-demo:${build_tag}"
        }
    }
    stage('Deploy to dev') {
        echo "5. Deploy DEV"
		sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s-dev-harbor.yaml"
        sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s-dev-harbor.yaml"
        sh "sed -i 's/192.168.40.182/192.168.10.56/' k8s-dev-harbor.yaml"
        sh "kubectl apply -f k8s-dev-harbor.yaml  --validate=false"
	}	
	stage('Promote to qa') {	
		def userInput = input(
            id: 'userInput',

            message: 'Promote to qa?',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "YES\nNO",
                    name: 'Env'
                ]
            ]
        )
        echo "This is a deploy step to ${userInput}"
        if (userInput == "YES") {
            sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s-qa-harbor.yaml"
            sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s-qa-harbor.yaml"
			sh "sed -i 's/192.168.40.182/192.168.10.56/' k8s-qa-harbor.yaml"
            sh "kubectl apply -f k8s-qa-harbor.yaml --validate=false"
            sh "sleep 6"
            sh "kubectl get pods -n qatest"
        } else {
            //exit
        }
    }
	stage('Promote to pro') {	
		def userInput = input(

            id: 'userInput',
            message: 'Promote to pro?',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "YES\nNO",
                    name: 'Env'
                ]
            ]
        )
        echo "This is a deploy step to ${userInput}"
        if (userInput == "YES") {
            sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s-prod-harbor.yaml"
            sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s-prod-harbor.yaml"
			sh "sed -i 's/192.168.40.182/192.168.10.56/' k8s-prod-harbor.yaml"
            sh "cat k8s-prod-harbor.yaml"
            sh "kubectl apply -f k8s-prod-harbor.yaml --record --validate=false"
        }
    }
}

```
