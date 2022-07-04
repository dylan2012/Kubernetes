# 使用NFS StorageClass

## 1. 搭建NFS服务器

```shell
yum install -y nfs-utils
systemctl enable rpcbind
systemctl enable nfs
systemctl start rpcbind
systemctl start nfs
#防火墙需要打开 rpc-bind 和 nfs 的服务
firewall-cmd --zone=public --permanent --add-service={rpc-bind,mountd,nfs}
firewall-cmd --reload

mkdir /data
chmod 755 /data

#编辑并添加一行
vi /etc/exports
/data/     192.168.0.0/24(rw,sync,no_root_squash,no_all_squash)

systemctl restart nfs
showmount -e localhost
```

/data: 共享目录位置。
192.168.0.0/24: 客户端 IP 范围，* 代表所有，即没有限制。
rw: 权限设置，可读可写。
sync: 同步共享目录。
no_root_squash: 可以使用 root 授权。
no_all_squash: 可以使用普通用户授权。

## 2. 创建RBAC授权

```yaml
# rbac.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs
[root@master01 init]# cat nfs-client-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

## 3. 创建StorageClass

```yaml
# storageclass-nfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: fuseim.pri/ifs
parameters:
  archiveOnDelete: "false"
```

## 4. 创建provisioner

```yaml
# nfs-client-provisioner.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          #image: quay.io/external_storage/nfs-client-provisioner:latest
          image: gcr.io/k8s-staging-sig-storage/nfs-subdir-external-provisioner:v4.0.0
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.199.10.235
            - name: NFS_PATH
              value: /data/disk2
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.199.10.235
            path: /data/disk2
```

## 5. 测试PVC

```yaml
#test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```shell
kubectl create -f test-pvc.yaml
kubectl get pvc
kubectl get pv
```
