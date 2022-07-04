# centos7系统内核升级

## 小版本升级

### 1. 查看当前和可升级版本

```shell
[root@master01 ~]# yum list kernel
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
 * base: repo.virtualhosting.hk
 * elrepo: hkg.mirror.rackspace.com
 * epel: hkg.mirror.rackspace.com
 * extras: repo.virtualhosting.hk
 * updates: repo.virtualhosting.hk
已安装的软件包
kernel.x86_64                                                                         3.10.0-1160.el7                                                                               @anaconda
kernel.x86_64                                                                         3.10.0-1160.66.1.el7                                                                          @updates
```

### 2. 升级

```shell
yum update kernel -y
reboot
uname -r
```

## 大版本升级

### 1.准备

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo=\* --enablerepo=elrepo-kernel repolist
```

### 2.查看可用的rpm包

```shell
[root@master01 ~]# yum --disablerepo=\* --enablerepo=elrepo-kernel list kernel*
已加载插件：fastestmirror
Repodata is over 2 weeks old. Install yum-cron? Or run: yum makecache fast
Loading mirror speeds from cached hostfile
 * elrepo-kernel: hkg.mirror.rackspace.com
已安装的软件包
kernel.x86_64                                                                                 3.10.0-1160.el7                                                                  @anaconda
kernel.x86_64                                                                                 3.10.0-1160.66.1.el7                                                             @updates
kernel-headers.x86_64                                                                         3.10.0-1160.66.1.el7                                                             @updates
kernel-ml.x86_64                                                                              5.17.8-1.el7.elrepo                                                              @elrepo-kernel
kernel-tools.x86_64                                                                           3.10.0-1160.66.1.el7                                                             @updates
kernel-tools-libs.x86_64                                                                      3.10.0-1160.66.1.el7                                                             @updates
可安装的软件包
kernel-lt.x86_64                                                                              5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-devel.x86_64                                                                        5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-doc.noarch                                                                          5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-headers.x86_64                                                                      5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-tools.x86_64                                                                        5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-tools-libs.x86_64                                                                   5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-lt-tools-libs-devel.x86_64                                                             5.4.194-1.el7.elrepo                                                             elrepo-kernel
kernel-ml-devel.x86_64                                                                        5.17.8-1.el7.elrepo                                                              elrepo-kernel
kernel-ml-doc.noarch                                                                          5.17.8-1.el7.elrepo                                                              elrepo-kernel
kernel-ml-headers.x86_64                                                                      5.17.8-1.el7.elrepo                                                              elrepo-kernel
kernel-ml-tools.x86_64                                                                        5.17.8-1.el7.elrepo                                                              elrepo-kernel
kernel-ml-tools-libs.x86_64                                                                   5.17.8-1.el7.elrepo                                                              elrepo-kernel
kernel-ml-tools-libs-devel.x86_64                                                             5.17.8-1.el7.elrepo                                                              elrepo-kernel
```

说明：

lt  ：long term support，长期支持版本；

ml：mainline，主线版本；

### 3. 安装最新版本的kernel

```shell
yum --disablerepo=\* --enablerepo=elrepo-kernel install  kernel-ml.x86_64  -y
yum remove kernel-tools-libs.x86_64 kernel-tools.x86_64  -y
yum --disablerepo=\* --enablerepo=elrepo-kernel install kernel-ml-tools.x86_64  -y
```

### 4.查看内核插入顺序

```shell
[root@master01 ~]# awk -F \' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (5.17.8-1.el7.elrepo.x86_64) 7 (Core)
1 : CentOS Linux (3.10.0-1160.66.1.el7.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-1160.el7.x86_64) 7 (Core)
3 : CentOS Linux (0-rescue-cd184487f2884f67ae090706ddfdd1f0) 7 (Core)
```

说明：默认新内核是从头插入，默认启动顺序也是从0开始（当前顺序还未生效）

### 5.查看当前实际启动顺序

```shell
[root@master01 ~]# grub2-editenv list
saved_entry=0
```

### 6.设置默认启动

```shell
grub2-set-default 0
grub2-editenv list
```

### 7.重启并检查

```shell
reboot 
uname -r
```
