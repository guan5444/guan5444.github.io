---
title: 24_kubernetes集群java项目上云部署实践
slug: 24_kubernetes-cluster-java-project-shangyun-deployment-practice-23urdh
url: >-
  /post/24_kubernetes-cluster-java-project-shangyun-deployment-practice-23urdh.html
tags: []
categories:
  - post
lastmod: '2023-04-24 19:04:19'
toc: true
keywords: ''
description: >-
  _kubernetes集群java项目上云部署实践一集群环境准备主机规划机ip地址主机名主机配置主机角色ckacmastercgmasterckacmastercgmasterckacmastercgmasterckacnodecgnodeckacnodecgnodeckacnodecgnodeckachacghackachacghaharborservercgharborgitlabservercggitlabjenkinsservercgjenkinsdevcgdevapiservervipingre
isCJKLanguage: true
---

# 24_kubernetes集群java项目上云部署实践

# 一、集群环境准备

## 1.1 主机规划

|机IP地址|主机名|主机配置|主机角色|
| --------------------------------| -------------------| -----------------| ------------------------------|
|192.168.100.241|ckac-master01|4C8G|master|
|192.168.100.242|ckac-master02|4C8G|master|
|192.168.100.243|ckac-master03|4C8G|master|
|192.168.100.244|ckac-node01|4C8G|node|
|192.168.100.245|ckac-node02|4C8G|node|
|192.168.100.246|ckac-node03|4C8G|node|
|192.168.100.247|ckac-ha01|2C2G|ha|
|192.168.100.248|ckac-ha02|2C2G|ha|
|192.168.100.224|harbor-server|4C8G|harbor|
|192.168.100.222|gitlab-server|4C8G|gitlab|
|192.168.100.223|jenkins-server|4C8G|jenkins|
|192.168.100.221|dev|4C8G|Dev|
|192.168.100.240|<br />||apiserver-vip|
|192.168.100.201|<br />||ingress-Loadbalancer|

## 1.2 软件版本

|配置信息|版本|备注|
| ----------------| ---------------------------------| ------|
|操作系统版本|CentOS 7.9 2009||
|kernel版本|kernel-lt  5.4.231-1.el7.elrepo||
|kubernetes版本|1.26.1||
|Docker版本|23.0.1||

## 1.3 系统架构图

​![](https://minio.duob.top:58443/dubodata01/2023/04/1a031a224db1d739bd5df21e3157a291.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=IryuMx1mVNnVAUfO%2F20230424%2Fauto%2Fs3%2Faws4_request&X-Amz-Date=20230424T110419Z&X-Amz-Expires=3600&X-Amz-Signature=1cccac7a4c50ad7200a93e4a4d4640933a48f87ac3ee7f447edf7ad468ca1a53&X-Amz-SignedHeaders=host%3Bif-match&x-id=GetObject)​

# 二、环境准备

## 2.1 存储准备

### 2.1.1 配置SC

‍

```yaml
[root@ckac-master01 sc-provider]# cat 01_kubernetes_sc_provisioner.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  # 命名空间要与定制的rbac的一致
  namespace: javaproject
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-provisioner
          # serviceAccount已经被serviceAccountName替代
          # serviceAccountName的值为SA资源的name
      containers:
      - name: nfs-subdir-external-provisioner
        image: harbor.duob.top/google_containers/nfs-subdir-external-provisioner:v4.0.0
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
        env:
        - name: PROVISIONER_NAME
             # 该变量的值，必须与nfs的storageclass的provisioner的值一致
          value: "nfsprovisioner"
        - name: NFS_SERVER
              # 设置NFS服务器的ip地址
          value: "192.168.100.160"
        - name: NFS_PATH
              # 设置NFS服务器分享的目录
          value: "/volume2/NFSdata"
      volumes:
      - name: nfs-client-root
        nfs:
          # 直接使用nfs来挂载该目录，方便storageclass基于该pod对pv和pvc进行自动处理
          server: "192.168.100.160"
          path: "/volume2/NFSdata"

[root@ckac-master01 sc-provider]# kubectl apply -f 01_kubernetes_sc_provisioner.yaml
```

‍

```yaml
[root@ckac-master01 sc-provider]# cat 02_kubernetes_sc_rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
  namespace: javaproject
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
   name: nfs-provisioner
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
    verbs: ["watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get","create","patch","list", "watch","update"]
  - apiGroups: ["extensions"]
    resources: ["podsecuritypolicies"]
    resourceNames: ["nfs-provisioner"]
    verbs: ["use"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-provisioner
  namespace: javaproject
subjects:
  - kind: ServiceAccount
    name: nfs-provisioner
    namespace: javaproject
roleRef:
  kind: ClusterRole
  name: nfs-provisioner
  apiGroup: rbac.authorization.k8s.io

[root@ckac-master01 sc-provider]# kubectl apply -f 02_kubernetes_sc_rbac.yaml
```

‍

```yaml
[root@ckac-master01 sc-provider]# cat 03_kubernetes_sc_pv.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc-javaproject
  namespace: javaproject
# 每个 StorageClass 都包含 provisioner、parameters 和 reclaimPolicy 字段
# provisioner用来决定使用哪个卷插件分配PV，必须与nfs-client的容器内部的 PROVISIONER_NAME 变量一致
# reclaimPolicy指定创建的Persistent Volume的回收策略
provisioner: "nfsprovisioner"
reclaimPolicy: Delete
parameters:
  # archiveOnDelete: "false"表示在删除时不会对数据进行打包，当设置为true时表示删除时会对数据进行打包
  archiveOnDelete: "false"

[root@ckac-master01 sc-provider]# kubectl apply -f 03_kubernetes_sc_pv.yaml
```

‍

## 2.2 网络准备

### 2.2.1 配置MetalLB

#### 2.2.1.1 编辑kube-proxy的configmap

```
kubectl edit configmap -n kube-system kube-proxy

# 在47行将mode: ""修改为mode: "ipvs"
mode: "ipvs"

# 在40行将strictARP: false修改为strictARP: true
strictARP: true
```

#### 2.2.1.2 使用yaml安装

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
```

#### 2.2.1.3 **创建 IPAdressPool**

```yaml
[root@ckac-master01 metallb]# cat 02_ipAddressPoll.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  # 可分配的 IP 地址,可以指定多个，包括 ipv4、ipv6
  - 192.168.100.201-192.168.100.209

[root@ckac-master01 metallb]# kubectl apply -f 02_ipAddressPoll.yaml
```

#### 2.2.1.4 **创建 L2Advertisement**

```yaml
[root@ckac-master01 metallb]# cat 03_L2Advertisement.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2a
  namespace: metallb-system
spec:
  ipAddressPools:
  #上一步创建的 ip 地址池，通过名字进行关联
  - first-pool

[root@ckac-master01 metallb]# kubectl apply -f 03_L2Advertisement.yaml
```

#### 2.2.1.5 MetalLB测试

```yaml
[root@ckac-master01 metallb]# cat 04_nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
  - name: nginx-port
    protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer


[root@ckac-master01 metallb]# kubectl apply -f 04_nginx-deploy.yaml
[root@ckac-master01 metallb]# kubectl get svc 
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)        AGE
kubernetes   ClusterIP      10.96.0.1       <none>            443/TCP        5d11h
nginx-svc    LoadBalancer   10.96.142.224   192.168.100.201   80:30316/TCP   1s
```

‍

### 2.2.3 配置ingress控制器

#### 2.2.3.1 下载ingress-nginx控制器的yaml文件

版本1.6.4

```yaml
[root@ckac-master01 ingress]# wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml
```

#### 2.2.3.2 修改yaml文件

* 将yaml中镜像拉取到本地，并修改镜像地址为本地harbor仓库
* 633行：failurePolicy由Fail修改为Ignore，避免默认的准入控制限制
* 493行：添加hostNetwork为true，增加cluster和node网络的互通
* 391行：将控制器类型由Deployment修改为DaemonSet
* 494行：无视master节点污点，与DaemonSet类型配合，便于将ingress控制器部署在3个master节点

```yaml
[root@ckac-master01 ingress]# diff ingress-daemonset-metallb.yaml ingress-deploy.yaml.bak
346a347
>   externalTrafficPolicy: Local
391c392
< kind: DaemonSet
---
> kind: Deployment
438c439
<         image: harbor.duob.top/google_containers/controller:v1.6.4
---
>         image: registry.k8s.io/ingress-nginx/controller:v1.6.4@sha256:15be4666c53052484dd2992efacf2f50ea77a78ae8aa21ccd91af6baaa7ea22f
493,495d493
<       hostNetwork: true
<       tolerations:
<       - operator: Exists
497d494
<         ingressSelect: "true"
539c536
<         image: harbor.duob.top/google_containers/kube-webhook-certgen:v20220916-gd32f8c343
---
>         image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f
588c585
<         image: harbor.duob.top/google_containers/kube-webhook-certgen:v20220916-gd32f8c343
---
>         image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f
633,634c630
<   failurePolicy: Ignore
---
>   failurePolicy: Fail

```

#### 2.3.3.3 为master节点添加标签

通过标签将ingress控制器部署在3个master节点

```yaml
[root@ckac-master01 ingress]# kubectl label ckac-master01 ingressSelect=true
[root@ckac-master01 ingress]# kubectl label ckac-master02 ingressSelect=true
[root@ckac-master01 ingress]# kubectl label ckac-master03 ingressSelect=true
```

#### 2.3.3.4 应用yaml文件

```yaml
[root@ckac-master01 ingress]# kubectl -f ingress-daemonset-metallb.yaml
```

## 2.3 环境准备

### 2.3.1 Harbor仓库

* 仓库名称：java-project
* 仓库属性：私有

​![](https://minio.duob.top:58443/dubodata01/2023/04/ccf62046dd9f14154fa023b2ddc59d14.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=IryuMx1mVNnVAUfO%2F20230424%2Fauto%2Fs3%2Faws4_request&X-Amz-Date=20230424T110419Z&X-Amz-Expires=3600&X-Amz-Signature=f99fcb949f5497897115d4ec50b57bed92a87bef4891304e096512c0fa59c8ed&X-Amz-SignedHeaders=host%3Bif-match&x-id=GetObject)​

### 2.3.2 Gitlab

* 项目名称：java-project
* 项目可见性级别：私有

​![](https://minio.duob.top:58443/dubodata01/2023/04/d7f7a86c21b4c2c8f4a0e04a41c9fe4e.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=IryuMx1mVNnVAUfO/20230424/auto/s3/aws4_request&X-Amz-Date=20230424T110325Z&X-Amz-Expires=3600&X-Amz-Signature=12e224373027e88a0249829b3f330eff4bee426706618ff5c57221400e998c40&X-Amz-SignedHeaders=host;if-match&x-id=GetObject)​

### 2.3.3 Jenkins

||软件|版本|备注|
| --| ---------| ---------------------------------------------| ----------------------|
||Jenkins|2.375.2|需要使用JDK11运行|
||maven|maven-3.8.7|需要使用JDK8进行编译|
||jdk|oracle jdk1.8-202/MicroSoft openjdk-11.0.18||

2.3.3.1 JDK安装安装与配置

jdk8为全局JAVA_HOME

‍

2.3.3.2 Maven安装与配置

cp

环境变量

server.xml配置

修改1  m2

修改2  阿里云

2.3.3.3 Jenkins安装与配置  
jenkins指定openjdk为jenkins的jdk

插件

配置

‍

## 2.4 项目源码准备

2.4.1 JAVA代码

在mysql部署后修改数据库连接

2.4.2 Dockerfile

2.4.3 Kubenetes资源清单

secret docker

‍

‍

‍

## 三、项目部署

3.1 数据库部署

3.2 代码发布到gitlab

3.3 jenkins打包并发布为镜像

3.4 应用部署

3.5 自动化CD配置

## 四、访问验证

‍

‍

### 

‍

‍
