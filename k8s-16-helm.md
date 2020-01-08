16 HELM

### 16.1	k8s的几种编排方式 

二进制部署：（v3版）

```bash
wget https://get.helm.sh/helm-v3.0.1-linux-amd64.tar.gz
tar zxvf helm-v3.0.1-linux-amd64.tar.gz 
cd linux-amd64/
cp helm /usr/local/bin/
scp helm root@k8s-master2:/usr/local/bin/
scp helm root@k8s-master3:/usr/local/bin/

[root@k8s-master1 linux-amd64]# helm version
version.BuildInfo{Version:"v3.0.1", GitCommit:"7c22ef9ce89e0ebeb7125ba2ebf7d421f3e82ffa", GitTreeState:"clean", GoVersion:"go1.13.4"}
```

脚本安装 v2版本：

```
https://raw.githubusercontent.com/helm/helm/master/scripts/get
```



二进制部署v2版本：

```
$ helm version
Client: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f9
3090", GitTreeState:"clean"}
Error: could not find tille

由于 Helm 默认会去 gcr.io 拉取镜像，所以如果你当前执⾏的机器没有配置科学上⽹的话可以实现下
⾯的命令代替：
$ helm init --upgrade --tiller-image cnych/tiller:v2.10.0
$HELM_HOME has been configured at /root/.helm.
Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users'
policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#s
ecuring-your-helm-installation
Happy Helming!

如果⼀直卡住或者报 google api 之类的错误，可以使⽤下⾯的命令进⾏初始化：
$ helm init --upgrade --tiller-image cnych/tiller:v2.10.0 --stable-repo-url https://cnych.
github.io/kube-charts-mirror/

这个命令会把默认的 google 的仓库地址替换成我同步的⼀个镜像地址。
如果在安装过程中遇到了⼀些其他问题，⽐如初始化的时候出现了如下错误：
E0125 14:03:19.093131 56246 portforward.go:331] an error occurred forwarding 55943 -> 44
134: error forwarding port 44134 to pod d01941068c9dfea1c9e46127578994d1cf8bc34c971ff109dc
6faa4c05043a6e, uid : unable to do port forwarding: socat not found.
2018/01/25 14:03:19 (0xc420476210) (0xc4203ae1e0) Stream removed, broadcasting: 3
2018/01/25 14:03:19 (0xc4203ae1e0) (3) Writing data frame
2018/01/25 14:03:19 (0xc420476210) (0xc4200c3900) Create stream
2018/01/25 14:03:19 (0xc420476210) (0xc4200c3900) Stream added, broadcasting: 5
Error: cannot connect to Tille解决⽅案：在节点上安装 socat 可以解决
$ sudo yum install -y socat

Helm 服务端正常安装完成后， Tiller 默认被部署在 kubernetes 集群的 kube-system 命名空间下：
$ kubectl get pod -n kube-system -l app=helm
NAME READY STATUS RESTARTS AGE
tiller-deploy-86b844d8c6-44fpq 1/1 Running 0 7m

此时，我们查看 Helm 版本就都正常了：
$ helm version
Client: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f9
3090", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.10.0", GitCommit:"9ad53aac42165a5fadc6c87be0dea6b115f9
3090", GitTreeState:"clean"}
```

v2版本 还要部署rbac

```
apiVersion: v1
kind: ServiceAccount
metadata:
name: tiller
namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
name: tiller
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: ClusterRole
name: cluster-admin
subjects:
- kind: ServiceAccount
name: tiller
namespace: kube-system


然后使⽤ kubectl 创建：
$ kubectl create -f rbac-config.yaml
serviceaccount "tiller" created
clusterrolebinding.rbac.authorization.k8s.io "tiller" created
创建了 tiller 的 ServceAccount 后还没完，因为我们的 Tiller 之前已经就部署成功了，⽽且是没有指
定 ServiceAccount 的，所以我们需要给 Tiller 打上⼀个 ServiceAccount 的补丁：
$ kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spe
c":{"serviceAccount":"tiller"}}}}'
```



### 16.2	Helm介绍

我们可以将Helm看作Kubernetes下的apt-get/yum。Helm是Deis (https://deis.com/) 开发的一个用于kubernetes的包管理器。每个包称为一个Chart（图表），一个Chart是一个目录（一般情况下会将目录进行打包压缩，形成name-version.tgz格式的单一文件，方便传输和存储）。

对于应用发布者而言，可以通过Helm打包应用，管理应用依赖关系，管理应用版本并发布应用到软件仓库。

对于使用者而言，使用Helm后不用需要了解Kubernetes的Yaml语法并编写应用部署文件，可以通过Helm下载并在kubernetes上安装需要的应用

除此以外，Helm还提供了kubernetes上的软件部署，删除，升级，回滚应用的强大功能。

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml15500\wps49.jpg) 

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml15500\wps50.jpg) 

Chart在集群内本地部署完就不叫做chart 而是叫做release

核心术语：

- Chart ： 一个helm的程序包；
- Repository：charts的仓库（就是一个http服务器）
- Release：当chart部署在k8s集群上的叫法（实例）

在老版本中 helm v2 有两个组成部分

![image-20191217094814206](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191217094814206.png)

Helm Client 是⽤户命令⾏⼯具，其主要负责如下： 

- 本地 chart 开发 
- 仓库管理 
- 与 Tiller sever 交互 
- 发送预安装的 chart 
- 查询 release 信息 
- 要求升级或卸载已存在的 release

Tiller Server 是⼀个部署在 Kubernetes 集群内部的 server，其与 Helm client、Kubernetes API server 进⾏交互。Tiller server 主要负责如下：

- 监听来⾃ Helm client 的请求
-  通过 chart 及其配置构建⼀次发布 
- 安装 chart 到 Kubernetes 集群，并跟踪随后的发布 
- 通过与 Kubernetes 交互升级或卸载 chart 
- 简单的说，client 管理 charts，⽽ server 管理发布 release

### 16.3	Helm v3重要里程碑 

除了移除 Tiller、让 Helm 变成纯客户端工具之外，Helm v3 相对于 Helm v2 ，还有如下一些重要变化：

- Release 名字可缩小至 Namespace 范围，需要显示启用选项 --generate-name ：这进一步规范和完善了 Helm 应用的名称问题；
- 合并描述应用依赖的 requirements.yaml 到 Chart.yaml：进一步减小用户的学习负担
- 支持 helm push 到远端 Helm Hub ，支持登陆认证；
- 支持在容器镜像 Registry 中存储 Charts：消除 Helm Hub 和 DockerHub 的重合定位
- 支持 LUA 语法：现有的模板变量，实际上问题很大，我们在后续文章中会详细介绍；
- 对 values.yaml 里的内容进行验证；

命令行的变化：

```
helm delete ---> helm uninstall: 曾经完全删除一个release需要helm delete xxx --purge， 现在只需要uninstall就可以，purge会作为一个默认的行为
helm inspect ---> helm show: 这里可以查看Chart的具体信息
helm fetch ---> helm pull: 与docker pull看齐，为下一步兼容registry 做铺垫，像拉取镜像一样拉取Chart部署
```

- ### Namespaces changes

Helm v2 只使用tiller 的namespace 作为release信息的存储，这样全集群的release名字都不能重复。Helm v3只会在release安装的所在namespace记录对应的信息，这样不同的namepsace就可以出现相同名字的release。

同样的原因，如果已经使用Helm v2创建了release，那么就无法使用helm v3来进行升级操作，因为无法将原来的单一namespace信息迁移到所属namespace 下。这一块的迁移功能，社区正在紧锣密鼓的开发中

```
老版本通过requirements.yaml和requirements.lock 来管理Chart的依赖，一个requirements.yaml 一般长相如下

dependencies:
- name: mariadb
  version: 5.x.x
  repository: https://kubernetes-charts.storage.googleapis.com/
  condition: mariadb.enabled
  tags:
- database

新版本直接使用Chart.yaml 来记录依赖信息,新的Chart.yaml格式和requirements.yaml 基本相同
  dependencies:
  - name: mariadb
    version: 5.x.x
    repository: https://kubernetes-charts.storage.googleapis.com/
    condition: mariadb.enabled
    tags:
      - database
```

- 老版本Helm 可以直接安装chart 并不需要指定名称，Helm v3需要指定名称

### 16.4	Helm客户端、服务端

Helm ：客户端     

 	·可以管理本地的chart仓库，管理chart，与Tiller服务器交互，发送chart，实例安装，卸载，查询等操作

Tiller：服务端

 ·监听来自helm发来的charts和config请求，合并生成release。 





### 16.5	使用Helm部署应用（自定义选项、升级、回滚、删除）

#### 16.5.1 helm的安装

看16.1章节

helm v3的bug：

```
helm install <helm-name> stable/xxxx

error: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1: the server is currently unable to handle the request

解决方法：

kubectl get apiservice
NAME                                   SERVICE                      AVAILABLE                      AGE
v1.                                    Local                        True                           40h
v1.admissionregistration.k8s.io        Local                        True                           40h
v1.apiextensions.k8s.io                Local                        True                           40h
v1.apps                                Local                        True                           40h
v1.authentication.k8s.io               Local                        True                           40h
v1.authorization.k8s.io                Local                        True                           40h
v1.autoscaling                         Local                        True                           40h
v1.batch                               Local                        True                           40h
v1.coordination.k8s.io                 Local                        True                           40h
v1.crd.projectcalico.org               Local                        True                           40h
v1.networking.k8s.io                   Local                        True                           40h
v1.rbac.authorization.k8s.io           Local                        True                           40h
v1.scheduling.k8s.io                   Local                        True                           40h
v1.storage.k8s.io                      Local                        True                           40h
v1beta1.admissionregistration.k8s.io   Local                        True                           40h
v1beta1.apiextensions.k8s.io           Local                        True                           40h
v1beta1.authentication.k8s.io          Local                        True                           40h
v1beta1.authorization.k8s.io           Local                        True                           40h
v1beta1.batch                          Local                        True                           40h
v1beta1.certificates.k8s.io            Local                        True                           40h
v1beta1.coordination.k8s.io            Local                        True                           40h
v1beta1.events.k8s.io                  Local                        True                           40h
v1beta1.extensions                     Local                        True                           40h
v1beta1.metrics.k8s.io                 kube-system/metrics-server   False (FailedDiscoveryCheck)   40h
v1beta1.networking.k8s.io              Local                        True                           40h
v1beta1.node.k8s.io                    Local                        True                           40h
v1beta1.policy                         Local                        True                           40h
v1beta1.rbac.authorization.k8s.io      Local                        True                           40h
v1beta1.scheduling.k8s.io              Local                        True                           40h
v1beta1.storage.k8s.io                 Local                        True                           40h
v2beta1.autoscaling                    Local                        True                           40h
v2beta2.autoscaling                    Local                        True                           40h


[root@k8s-master1 helm]# kubectl delete apiservice  v1beta1.metrics.k8s.io
apiservice.apiregistration.k8s.io "v1beta1.metrics.k8s.io" deleted

就可以了

```



#### 16.5.2 helm的常用命令解释（v3版本）

Helm 的 Repo 仓库和 Docker Registry ⽐较类似，Chart 库可以⽤来存储和共享打包 Chart 的位置我们在安装了 Helm 后，默认的仓库地址是 google 的⼀个地址，这对于我们不能科学上⽹的同学就⽐ 较苦恼了，没办法访问到官⽅提供的 Chart 仓库，可以⽤ helm repo list 来查看当前的仓库配置：

如果是v2版 请自行百度......

```bash
[root@k8s-master1 helm]# helm repo list
NAME  	URL                                                   
aliyun	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
stable	https://kubernetes-charts.storage.googleapis.com/     
apphub	https://apphub.aliyuncs.com/     


helm repo update 更新仓库
[root@k8s-master1 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "aliyun" chart repository
...Successfully got an update from the "apphub" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 

查找 chart(v3版本)
[root@k8s-master1 ~]# helm search repo stable/jenkins
NAME          	CHART VERSION	APP VERSION	DESCRIPTION                                       
stable/jenkins	1.9.9        	lts        	Open source continuous integration server. It s...


使⽤ inspect 命令来查看⼀个 chart 的详细信息（v3版本）
[root@k8s-master1 ~]# helm inspect chart stable/mysql
apiVersion: v1
appVersion: 5.7.28
description: Fast, reliable, scalable, and easy to use open-source relational database
  system.
home: https://www.mysql.com/
icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png
keywords:
- mysql
- database
- sql
maintainers:
- email: o.with@sportradar.com
  name: olemarkus
- email: viglesias@google.com
  name: viglesiasce
name: mysql
sources:
- https://github.com/kubernetes/charts
- https://github.com/docker-library/mysql
version: 1.6.2
```

#### 16.5.3helm的应用部署的回滚升级

- 升级：

首先看一下版本：gomo1 的revision的版本号为1

```
[root@k8s-master1 ~]# helm ls
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        APP VERSION
gomo-redis	default  	1       	2019-12-18 03:11:37.832814379 -0500 EST	deployed	redis-ha-4.1.5.0.6      
gomo1     	default  	1       	2019-12-16 22:24:11.835272018 -0500 EST	deployed	gomo-0.1.0   1.16.0     
```

然后到我们自己配置的gomo文件中修改一些参数：

```
[root@k8s-master1 helm]# cd gomo/
[root@k8s-master1 gomo]# ls
charts  Chart.yaml  templates  values.yaml
[root@k8s-master1 gomo]# vi values.yaml 
# 自己修改参数 我修改的是端口号80改掉了 改成了83#
```

创建新的helm

```
[root@k8s-master1 gomo]# helm upgrade gomo1 ./
Release "gomo1" has been upgraded. Happy Helming!
NAME: gomo1
LAST DEPLOYED: Wed Dec 18 04:15:46 2019
NAMESPACE: default
STATUS: deployed
REVISION: 2
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=gomo,app.kubernetes.io/instance=gomo1" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

然后在查看一下

```
[root@k8s-master1 gomo]# helm ls
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        APP VERSION
gomo-redis	default  	1       	2019-12-18 03:11:37.832814379 -0500 EST	deployed	redis-ha-4.1.5.0.6      
gomo1     	default  	2       	2019-12-18 04:15:46.09894648 -0500 EST 	deployed	gomo-0.1.0   1.16.0     

具体验证：
[root@k8s-master1 mysql]# kubectl get svc
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
gomo1                            ClusterIP   10.254.8.233     <none>        83/TCP               30h

```

- 回滚：

```bash
[root@k8s-master1 mysql]# helm history gomo1
REVISION	UPDATED                 	STATUS    	CHART     	APP VERSION	DESCRIPTION     
1       	Mon Dec 16 22:24:11 2019	superseded	gomo-0.1.0	1.16.0     	Install complete
2       	Wed Dec 18 04:15:46 2019	deployed  	gomo-0.1.0	1.16.0     	Upgrade complete

[root@k8s-master1 mysql]# helm history gomo1
REVISION	UPDATED                 	STATUS    	CHART     	APP VERSION	DESCRIPTION     
1       	Mon Dec 16 22:24:11 2019	superseded	gomo-0.1.0	1.16.0     	Install complete
2       	Wed Dec 18 04:15:46 2019	deployed  	gomo-0.1.0	1.16.0     	Upgrade complete
[root@k8s-master1 mysql]# helm rollback gomo1 1
Rollback was a success! Happy Helming!

验证：
[root@k8s-master1 mysql]# helm ls
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        APP VERSION
gomo-redis	default  	1       	2019-12-18 03:11:37.832814379 -0500 EST	deployed	redis-ha-4.1.5.0.6      
gomo1     	default  	3       	2019-12-18 04:28:10.307621129 -0500 EST	deployed	gomo-0.1.0   1.16.0     
[root@k8s-master1 mysql]# helm history gomo1
REVISION	UPDATED                 	STATUS    	CHART     	APP VERSION	DESCRIPTION     
1       	Mon Dec 16 22:24:11 2019	superseded	gomo-0.1.0	1.16.0     	Install complete
2       	Wed Dec 18 04:15:46 2019	superseded	gomo-0.1.0	1.16.0     	Upgrade complete
3       	Wed Dec 18 04:28:10 2019	deployed  	gomo-0.1.0	1.16.0     	Rollback to 1 

[root@k8s-master1 mysql]# kubectl get svc
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
gomo1                            ClusterIP   10.254.8.233     <none>        80/TCP               30h

```

- 删除：

```
[root@k8s-master1 mysql]# helm delete gomysql
release "gomysql" uninstalled
```

v2版本：

```
helm ls --daleted   #查看以前的版本
```

v2 版本 如果不强制删除（彻底删除 --purge） 再次创建同个名字的chart 会报错 而v3不会 删除就直接删除！

### 16.6	部署 MySQL, redis, Ingress 

#### 16.6.1helm 部署 mysql

```
[root@k8s-master1 repository]# helm install gomo-mysql stable/mysql
NAME: gomo-mysql
LAST DEPLOYED: Wed Dec 18 01:30:44 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
gomo-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default gomo-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h gomo-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/gomo-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}

```

检查：

```
[root@k8s-master1 repository]# helm ls
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        APP VERSION
gomo-mysql	default  	1       	2019-12-18 01:30:44.172660012 -0500 EST	deployed	mysql-1.6.2  5.7.28     

[root@k8s-master1 ~]# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
gomo-mysql-6b9fc85fdc-fghpq   1/1     Running   0          2m35s
gomo1-6d5d895d5-t9m5s         1/1     Running   0          27h


记得给配置PV PVC

```



#### 16.6.2 helm搭建redis

```
查看redis-ha的内容：
helm show chart stable/redis-ha

helm v3 在家目录下的.cache文件下
[root@k8s-master1 ~]# cd .cache/helm/repository/ 

解压去查看具体内容：
[root@k8s-master1 helm]# tree redis-ha/
redis-ha/
├── Chart.yaml
├── ci
│   └── haproxy-enabled-values.yaml
├── OWNERS
├── README.md
├── templates
│   ├── _configs.tpl
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   ├── redis-auth-secret.yaml
│   ├── redis-ha-announce-service.yaml
│   ├── redis-ha-configmap.yaml
│   ├── redis-ha-pdb.yaml
│   ├── redis-haproxy-deployment.yaml
│   ├── redis-haproxy-serviceaccount.yaml
│   ├── redis-haproxy-servicemonitor.yaml
│   ├── redis-haproxy-service.yaml
│   ├── redis-ha-rolebinding.yaml
│   ├── redis-ha-role.yaml
│   ├── redis-ha-serviceaccount.yaml
│   ├── redis-ha-servicemonitor.yaml
│   ├── redis-ha-service.yaml
│   ├── redis-ha-statefulset.yaml
│   └── tests
│       ├── test-redis-ha-configmap.yaml
│       └── test-redis-ha-pod.yaml
└── values.yaml
如果需要修改自己的配置可以在里面修改

下载安装：
[root@k8s-master1 ~]# helm install gomo-redis stable/redis-ha
NAME: gomo-redis
LAST DEPLOYED: Wed Dec 18 02:37:52 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Redis can be accessed via port 6379 and Sentinel can be accessed via port 26379 on the following DNS name from within your cluster:
gomo-redis-redis-ha.default.svc.cluster.local

To connect to your Redis server:
1. Run a Redis pod that you can use as a client:

   kubectl exec -it gomo-redis-redis-ha-server-0 sh -n default

2. Connect using the Redis CLI:

  redis-cli -h gomo-redis-redis-ha.default.svc.cluster.local
  
  
  [root@k8s-master1 ~]# kubectl get pods
NAME                           READY   STATUS    RESTARTS   AGE
gomo-redis-redis-ha-server-0   2/2     Running   0          2m11s
gomo-redis-redis-ha-server-1   2/2     Running   0          116s
gomo-redis-redis-ha-server-2   2/2     Running   0          19s
gomo1-6d5d895d5-t9m5s          1/1     Running   0          28h

[root@k8s-master1 ~]# kubectl get svc
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)              AGE
gomo-redis-redis-ha              ClusterIP   None             <none>        6379/TCP,26379/TCP   2m15s
gomo-redis-redis-ha-announce-0   ClusterIP   10.254.22.92     <none>        6379/TCP,26379/TCP   2m15s
gomo-redis-redis-ha-announce-1   ClusterIP   10.254.85.98     <none>        6379/TCP,26379/TCP   2m15s
gomo-redis-redis-ha-announce-2   ClusterIP   10.254.216.182   <none>        6379/TCP,26379/TCP   2m15s

 在创建集群的时候需要自己创建PV
 [root@k8s-master1 yml]# vi pv-redis.yaml 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-to-redis
spec:
  capacity:
    storage: 12Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # storageClassName: nfs
  nfs:
    path: /home/k8sv
    server: 192.168.208.195
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-to-redis2
spec:
  capacity:
    storage: 12Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # storageClassName: nfs
  nfs:
    path: /home/k8sv
    server: 192.168.208.195

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-to-redis3
spec:
  capacity:
    storage: 12Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # storageClassName: nfs
  nfs:
    path: /home/k8sv
    server: 192.168.208.195


```

#### 16.6.3 helm搭建ingress-nginx

来自官网的文档下载

NGINX Ingress controller can be installed via [Helm](https://helm.sh/) using the chart [stable/nginx-ingress](https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress) from the official charts repository. To install the chart with the release name `my-nginx`:

```
helm install stable/nginx-ingress --name my-nginx
```

If the kubernetes cluster has RBAC enabled, then run:

```
helm install stable/nginx-ingress --name my-nginx --set rbac.create=true
```

Detect installed version:

```
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- /nginx-ingress-controller --version
```



### 16.7	Chart模板 

![image-20191218174730117](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191218174730117.png)

​                                                                                   大致流程

```
charts.yaml 配置清单：

apiVersion: The chart API version (required)
name: The name of the chart (required)
version: A SemVer 2 version (required)
kubeVersion: A SemVer range of compatible Kubernetes versions (optional)
description: A single-sentence description of this project (optional)
type: It is the type of chart (optional)
keywords:
  - A list of keywords about this project (optional)
home: The URL of this project's home page (optional)
sources:
  - A list of URLs to source code for this project (optional)
dependencies: # A list of the chart requirements (optional)
  - name: The name of the chart (nginx)
    version: The version of the chart ("1.2.3")
    repository: The repository URL ("https://example.com/charts")
maintainers: # (optional)
  - name: The maintainer's name (required for each maintainer)
    email: The maintainer's email (optional for each maintainer)
    url: A URL for the maintainer (optional for each maintainer)
icon: A URL to an SVG or PNG image to be used as an icon (optional).
appVersion: The version of the app that this contains (optional). This needn't be SemVer.
deprecated: Whether this chart is deprecated (optional, boolean)

```

文件包含 可以使用tree来查看：

helm create gomo #自己创建的一个图表模板（初始chart）

```
[root@k8s-master1 helm]# tree gomo/
gomo/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

```

templates/ 目录用于放置模板文件。当 Tiller 评估 chart 时，它将 templates/ 通过模板渲染引擎发送目录中的所有文件。然后，Tiller 收集这些模板的结果并将它们发送给 Kubernetes。

values.yaml 文件对模板也很重要。该文件包含 chart 默认值。这些值可能在用户在 helm install 或 helm upgrade 期间被覆盖。

Chart.yaml 文件包含 chart 的说明。可以从模板中查看访问它。该 charts/ 目录可能包含其他 chart（我们称之为子 chart）。在本指南的后面，我们将看到它们在模板渲染方面如何起作用。

提示： 模板名称不遵循严格的命名模式。但是，我们建议 .yaml 为 YAML 文件后缀，.tpl 为模板助手后缀。



删除掉templates目录下全部的东西：（自己自定义）

```
[root@k8s-master1 templates]# cat diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap   #这样写 有灵活性 不会写死文件
data:
  myvalue: "Hello Gomo"
  
[root@k8s-master1 gomo]# helm install gomo2 ./
NAME: gomo2
LAST DEPLOYED: Wed Dec 18 04:55:19 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
[root@k8s-master1 gomo]# helm get manifest gomo2
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo2-configmap
data:
  myvalue: "Hello Gomo"
```

Release 前面的前一个小圆点表示我们从这个范围的最上面的 namespace 开始。所以我们可以这样理解 .Release.Name："从顶层命名空间开始，找到 Release 对象，然后在里面查找名为 Name 的对象"。



- 调试： --dry-run  虚假的运行一次 不是真实的运行

```bash
[root@k8s-master1 gomo]# helm install gomo3 --debug --dry-run ./
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /root/helm/gomo

NAME: gomo3
LAST DEPLOYED: Wed Dec 18 20:48:06 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
affinity: {}
fullnameOverride: ""
image:
  pullPolicy: IfNotPresent
  repository: nginx
imagePullSecrets: []
ingress:
  annotations: {}
  enabled: false
  hosts:
  - host: chart-example.local
    paths: []
  tls: []
nameOverride: ""
nodeSelector: {}
podSecurityContext: {}
replicaCount: 1
resources: {}
securityContext: {}
service:
  port: 83
  type: ClusterIP
serviceAccount:
  create: true
  name: null
tolerations: []

HOOKS:
MANIFEST:
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo3-configmap
data:
  myvalue: "Hello Gomo"

[root@k8s-master1 gomo]# helm  ls
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART        APP VERSION
gomo-redis	default  	1       	2019-12-18 03:11:37.832814379 -0500 EST	deployed	redis-ha-4.1.5.0.6      
gomo1     	default  	3       	2019-12-18 04:28:10.307621129 -0500 EST	deployed	gomo-0.1.0   1.16.0     
gomo2     	default  	1       	2019-12-18 04:55:19.512281949 -0500 EST	deployed	gomo-0.1.0   1.16.0     

```



#### 16.7.1内置对象 

```
内置对象首字母大写！
```

刚刚我们使⽤ {{.Release.Name}} 将 release 的名称插⼊到模板中。这⾥的 Release 就是 Helm 的内置对象，下⾯是⼀些常⽤的内置对象，在需要的时候直接使⽤就可以：

1. Release：这个对象描述了 release 本身。它⾥⾯有⼏个对象：

- Release：这个对象描述了 release 本身。它⾥⾯有⼏个对象：
- Release.Name：release 名称
- Release.Time：release 的时间
- Release.Namespace：release 的 namespace（如果清单未覆盖）
- Release.Service：release 服务的名称（始终是 Tiller）。
- Release.Revision：此 release 的修订版本号，从1开始累加。
- Release.IsUpgrade：如果当前操作是升级或回滚，则将其设置为 true。
- Release.IsInstall：如果当前操作是安装，则设置为 true。

   2.Values：从 values.yaml ⽂件和⽤户提供的⽂件传⼊模板的值。默认情况下，Values 是空的

   3.Chart： Chart.yaml ⽂件的内容。所有的 Chart 对象都将从该⽂件中获取。chart 指南中Charts Guide列出了可⽤字段，可以前往查看。

   4.Files：这提供对 chart 中所有⾮特殊⽂件的访问。虽然⽆法使⽤它来访问模板，但可以使⽤它来
访问 chart 中的其他⽂件。请参阅 "访问⽂件" 部分。

- Files.Get 是⼀个按名称获取⽂件的函数（.Files.Get config.ini）
- Files.GetBytes 是将⽂件内容作为字节数组⽽不是字符串获取的函数。这对于像图⽚这样的东⻄很有⽤。

   5.Capabilities：这提供了关于 Kubernetes 集群⽀持的功能的信息。

- Capabilities.APIVersions 是⼀组版本信息。
- Capabilities.APIVersions.Has $version 指示是否在群集上启⽤版本（batch/v1）。
- Capabilities.KubeVersion 提供了查找 Kubernetes 版本的⽅法。它具有以下值：Major，Minor，GitVersion，GitCommit，GitTreeState，BuildDate，GoVersion，Compiler，和Platform。
- Capabilities.TillerVersion 提供了查找 Tiller 版本的⽅法。它具有以下值：SemVer，GitCommit，和 GitTreeState。

   6.Template：包含有关正在执⾏的当前模板的信息.

   7.Name：到当前模板的⽂件路径（例如 mychart/templates/mytemplate.yaml)

   8.BasePath：当前 chart 模板⽬录的路径（例如 mychart/templates）。

上⾯这些值可⽤于任何顶级模板，要注意内置值始终以⼤写字⺟开头。这也符合 Go 的命名约定。当你创建⾃⼰的名字时，你可以⾃由地使⽤适合你的团队的惯例。（秃头吧 少年 ，详情请看go的语法https://godoc.org）



- values ⽂件

上⾯的内置对象中有⼀个对象就是 Values，该对象提供对传⼊ chart 的值的访问，Values 对象的值有 4个来源：

- chart 包中的 values.yaml ⽂件
- ⽗ chart 包的 values.yaml ⽂件
- 通过 helm install 或者 helm upgrade 的 -f 或者 --values 参数传⼊的⾃定义的 yaml ⽂件
- 通过 --set 参数传⼊的值

chart 的 values.yaml 提供的值可以被⽤户提供的 values ⽂件覆盖，⽽该⽂件同样可以被 --set 提供的参数所覆盖。

这⾥我们来重新编辑 mychart/values.yaml ⽂件，将默认的值全部清空，添加⼀个新的数据： (values.yaml)然后我们在上⾯的 templates/configmap.yaml 模板⽂件中就可以使⽤这个值了：(configmap.yaml）

```bash
[root@k8s-master1 gomo]# tac values.yaml 
course: gomo
[root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  course: {{.Values.course }}

#然后在干跑一下：
[root@k8s-master1 gomo]# helm install gomo3 --debug --dry-run ./
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /root/helm/gomo

NAME: gomo3
LAST DEPLOYED: Wed Dec 18 21:16:46 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
course: gomo

HOOKS:
MANIFEST:
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo3-configmap
data:
  myvalue: "Hello Gomo"
  course: gomo

```

我们可以看到 ConfigMap 中 course 的值被渲染成了 k8s，这是因为在默认的 values.yaml ⽂件中该参 数值为 k8s，同样的我们可以通过 --set 参数来轻松的覆盖 course 的值：

```bash
[root@k8s-master1 gomo]# helm install gomo3 --debug --dry-run --set course=xjbb ./
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /root/helm/gomo

NAME: gomo3
LAST DEPLOYED: Wed Dec 18 21:18:30 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
course: xjbb

COMPUTED VALUES:
course: xjbb

HOOKS:
MANIFEST:
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo3-configmap
data:
  myvalue: "Hello Gomo"
  course: xjbb
```

由于 --set ⽐默认 values.yaml ⽂件具有更⾼的优先级，所以我们的模板⽣成为 course: python。 values ⽂件也可以包含更多结构化内容，例如，我们在 values.yaml ⽂件中可以创建 course 部分，然 后在其中添加⼏个键：

```bash
[root@k8s-master1 gomo]# cat values.yaml 
course: 
  yunwei: sosad
  go: ou
  python: emmmm
  
 #修改cm模板：
 [root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: {{.Values.course.yunwei }}
  go: {{.Values.course.go }}
  python: {{.Values.course.python }}
 
 #干跑运行：
 [root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /root/helm/gomo

NAME: gomo-values-test
LAST DEPLOYED: Wed Dec 18 21:25:08 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
course:
  go: ou
  python: emmmm
  yunwei: sosad

HOOKS:
MANIFEST:
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: sosad
  go: ou
  python: emmmm

```

删除默认 key：

如果您需要从默认值中删除一个键，可以覆盖该键的值为 null，在这种情况下，Helm 将从覆盖值合并中删除该键。

可以看到模板中的参数已经被 values.yaml ⽂件中的值给替换掉了。虽然以这种⽅式构建数据是可以 的，但我们还是建议保持 value 树浅⼀些，平⼀些，这样维护起来要简单⼀点。

#### 16.7.2模板函数与管道（涉及go语言）

学习了如何将信息渲染到模板之中，但是这些信息都是直接传⼊模板引擎中进⾏渲染的，有的时候我们想要转换⼀下这些数据才进⾏渲染，这就需要使⽤到 Go 模板语⾔中的⼀些其他⽤法。

⽐如我们需要从 .Values 中读取的值变成**字符串**的时候就可以通过调⽤ quote 模板函数来实现：

```bash
[root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: {{quote .Values.course.yunwei }}
  go: {{.Values.course.go }}
  python: {{.Values.course.python }}

模板函数遵循调⽤的语法为： functionName arg1 arg2... 。在上⾯的模板⽂件中， quote .Values.course.yunwei 调⽤ quote 函数并将后⾯的值作为⼀个参数传递给它。最终被渲染为：

[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
install.go:148: [debug] Original chart version: ""
install.go:165: [debug] CHART PATH: /root/helm/gomo

NAME: gomo-values-test
LAST DEPLOYED: Wed Dec 18 21:32:06 2019
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
course:
  go: ou
  python: emmmm
  yunwei: sosad

HOOKS:
MANIFEST:
---
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: ou
  python: emmmm
```

我们可以看到 .quote .Values.course.yunwei 被渲染成了字符串 sosad。Helm 是⼀种 Go 模板语⾔，拥有超过60多种可⽤的内置函数，⼀部分是由Go 模板语⾔本身定义的，其他⼤部分都 是 Sprig 模板库提供的⼀部分，我们可以前往这两个⽂档中查看这些函数的⽤法。

⽐如我们这⾥使⽤的 quote 函数就是 Sprig 模板库 提供的⼀种字符串函数，⽤途就是⽤双引号将字符 串括起来，如果需要双引号 " ，则需要添加 \ 来进⾏转义，⽽ squote 函数的⽤途则是⽤双引号将字 符串括起来，⽽不会对内容进⾏转义。

所以在我们遇到⼀些需求的时候，⾸先要想到的是去查看下上⾯的两个模板⽂档中是否提供了对应的 模板函数，这些模板函数可以很好的解决我们的需求。

- 管道

模板语⾔除了提供了丰富的内置函数之外，其另⼀个强⼤的功能就是管道的概念。和 UNIX 中⼀样， 管道我们通常称为 Pipeline ，是⼀个链在⼀起的⼀系列模板命令的⼯具，以紧凑地表达⼀系列转换。 简单来说，管道是可以按顺序完成⼀系列事情的⼀种⽅法。⽐如我们⽤管道来重写上⾯的 ConfigMap 模板：

```bash
[root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: {{ .Values.course.yunwei | quote }}  #转义层字符串 ""
  go: {{.Values.course.go | repeat 3 | quote}}  #重复3次 在转义层字符串
  python: {{.Values.course.python | upper |quote }}  #大写后在转义成字符串
  
  
[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
......omit.....
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
```

注意：

```
go: {{.Values.course.go | repeat 3 | quote}}  反过来写
go: {{.Values.course.go | quote | repeat 3}}  会报错
....omit....
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ou""ou""ou"
  python: "EMMMM"
```

#### 16.7.3流程控制（涉及go语言）

模板函数和管道 是通过转换信息并将其插⼊到 YAML ⽂件中的强⼤⽅法。但有时候需要添加⼀些⽐插⼊ 字符串更复杂⼀些的模板逻辑。这就需要使⽤到模板语⾔中提供的控制结构了控制流程为我们提供了控制模板⽣成流程的⼀种能⼒，Helm 的模板语⾔提供了以下⼏种流程控制：

1. if/else 条件块

if/else 块是⽤于在模板中有条件地包含⽂本块的⽅法，条件块的基本结构如下：

```
{{ if PIPELINE }}
# Do something
{{ else if OTHER PIPELINE }# Do something else
{{ else }}
# Default case
{{ end }}
```

当然要使⽤条件块就得判断条件是否为真，如果值为下⾯的⼏种情况，则管道的结果为 false：

- ⼀个布尔类型的 假
-  ⼀个数字 零
-  ⼀个 空 的字符串 
- ⼀个 nil （空或 null ） 
- ⼀个空的集合（ map 、 slice 、 tuple 、 dict 、 array ）

除了上⾯的这些情况外，其他所有条件都为 真 。



```
[root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: {{ .Values.course.yunwei | quote }}
  go: {{.Values.course.go | repeat 3 | quote}}
  python: {{.Values.course.python | upper |quote }}
  {{if eq .Values.course.python "emmmm"}}web: true{{end}}

[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
.....omit.....
# Source: gomo/templates/diy-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
  web: true

```

在上⾯的模板⽂件中我们增加了⼀个条件语句判断  {{if eq .Values.course.python "emmmm"}}web: true{{end}} ，其中运算符 eq 是判断是否相等的操作，除此之外，还 有 ne 、 lt 、 gt 、 and 、 or 等运算符都是 Helm 模板已经实现了的，直接使⽤即可。这⾥{{ .Values.course.python }} 的值在 values.yaml ⽂件中默认被设置为了emmm，所以正常来说下 ⾯的条件语句判断为真，所以模板⽂件最终被渲染后会有 web: true 这样的的上面那个条⽬；如果我们修改了python的值if条件不成立了 那就没有了web了

```
[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run --set course.python=abs ./
.....omit....
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "ABS"

```

`空格控制`

上⾯我们的条件判断语句是在⼀整⾏中的，如果平时经常写代码的同学可能⾮常不习惯了，我们⼀般 会将其格式化为更容易阅读的形式，⽐如：

```
{{ if eq .Values.course.python "emmmm" }}
gomo: true
{{ end }}


[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
.....omit.....
  python: "EMMMM"
  
  gomo: true
```

我们可以看到渲染出来会有多余的空⾏，这是因为当模板引擎运⾏时，它将⼀些值渲染过后，之前的 指令被删除，但它之前所占的位置完全按原样保留剩余的空⽩了，所以就出现了多余的空 ⾏。 YAML ⽂件中的空格是⾮常严格的，所以对于空格的管理⾮常重要，⼀不⼩⼼就会导致你 的 YAML ⽂件格式错误。 我们可以通过使⽤在模板标识 {{ 后⾯添加破折号和空格 {{- 来表示将空⽩左移，⽽在 }} 前⾯添加 ⼀个空格和破折号 -}} 表示应该删除右边的空格，另外需要注意的是换⾏符也是空格！ 使⽤这个语法，我们来修改我们上⾯的模板⽂件去掉多余的空格：

```
[root@k8s-master1 gomo]# cat templates/diy-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: {{ .Values.course.yunwei | quote }}
  go: {{.Values.course.go | repeat 3 | quote}}
  python: {{.Values.course.python | upper |quote }}
  {{- if eq .Values.course.python "emmmm"}}
  gomo: true
  {{- end}}


[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
.....omit.....
  go: "ououou"
  python: "EMMMM"
  gomo: true
```

PS:

```bash
  {{- if eq .Values.course.python "emmmm" -}}
  gomo: true
  {{- end}}
  #如果我们在 if 条件后⾯增加 -}} ，这会渲染成： 因为 -}} 它删除了双⽅的换⾏符，显然这是不正确的。
  
  python: "EWMMM"gomo: true
```

有关模板中空格控制的详细信息，请参阅官⽅ Go [模板⽂档](https://godoc.org/text/template)

2.with  指定范围

接下来我们来看下 with 关键词的使⽤，它⽤来控制变量作⽤域。还记得之前我们的 {{ .Release.xxx}} 或者 {{ .Values.xxx }} 吗？其中的 . 就是表示对当前范围的引⽤， .Values 就是告诉模板在当前范围中查找 Values 对象的值。⽽ with 语句就可以来控制变量的作⽤域范围，其语法和⼀个简单的 if 语句⽐较类似：

```
{{ with PIPELINE }}
# restricted scope
{{ end }}
```

with 语句可以允许将当前范围 . 设置为特定的对象，⽐如我们前⾯⼀直使⽤的 .Values.course ，我 们可以使⽤ with 来将 . 范围指向 .Values.course ：

```bash
[root@k8s-master1 gomo]# cat templates/with-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  {{- if eq .python "emmmm"}}
  gomo: true
  {{- end}}
  {{- end}}

######################################################
{{- with .Values.course}}
..................
{{- end}}
这样的话我们就可以在当前的块⾥⾯直接引⽤ .python 和 .go 了，⽽不需要进⾏限定了，这是因为该 with 声明将 . 指向了 .Values.course ，在 {{- end }} 后 . 就会复原其之前的作⽤范围了，我们可以使⽤模板引擎来渲染上⾯的模板查看是否符合预期结果。不过需要注意的是在 with 声明的范围内，此时将⽆法从⽗范围访问到其他对象了，⽐如下⾯的模板渲染的时候将会报错，因为显然 .Release 根本就不在当前的 . 范围内，当然如果我们最后两⾏交换下位置就正常了，因为 {{- end }} 之后范围就被重置了：
#######################################################
{{- with .Values.course }}
go: {{ .go | upper | quote }}
python: {{ .python | repeat 3 | quote }}
release: {{ .Rease.Name }}
{{- end }}
```

3.range 循环块

如果你对编程语⾔熟悉的话，⼏乎所有的编程语⾔都⽀持类似于 for 、 foreach 或者类似功能的循 环机制，在 Helm 模板语⾔中，是使⽤ range 关键字来进⾏循环操作。

我们在 values.yaml ⽂件中添加上一些女人名字：

```
[root@k8s-master1 gomo]# cat values.yaml 
course:
  yunwei: sosad
  go: ou
  python: emmmm
japenmowen:
- cangjingkong
- yejieyi
- maliya
- huaqiluo
```

现在我们有4个女人，修改 ConfigMap 模板⽂件来循环打印出该列表：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  {{- if eq .python "emmmm"}}
  gomo: true
  {{- end}}
  {{- end}}
  japenmowen:
  {{- range .Values.japenmowen}}
  - {{. | title | quote}}
  {{- end}}
  
  
  
.........omit...........
web: true
japenmowen:
- "Cangjingkong"
- "Yyejieyi"
- "Maliya"
- "Huaqiluo"
  
```

我们可以看到 日本女人按照我们的要求循环出来了。除了 list 或者 tuple，range 还可以⽤于遍历具 有键和值的集合（如map 或 dict），这个就需要⽤到变量的概念了。

除此之外，它还提供了⼀些声明和使⽤命名模板段的操作：

- define 在模板中声明⼀个新的命名模板
-  template 导⼊⼀个命名模板
-  block 声明了⼀种特殊的可填写的模板区域



`变量`

前⾯我们已经学习了函数、管理以及控制流程的使⽤⽅法，我们知道编程语⾔中还有⼀个很重要的概 念叫：变量，在 Helm 模板中，使⽤变量的场合不是特别多，但是在合适的时候使⽤变量可以很好的 解决我们的问题。如下⾯的模板：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- $releaseName := .Release.Name -}}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  release: {{$releaseName}}
  {{- end}}
```

PS： .Release.Name 不在该 with 语句块限制的作⽤范围之内，我们可以将该对象赋值给⼀个变量可以来解决这个问题

我们可以看到我们在 with 语句上⾯增加了⼀句 {{- $releaseName := .Release.Name -}} ，其 中 $releaseName 就是后⾯的对象的⼀个引⽤变量，它的形式就是 $name ，赋值操作使⽤ := ，这 样 with 语句块内部的 $releaseName 变量仍然指向的是 .Release.Name ，同样，我们 DEBUG 下查看 结果：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
  release: gomo-values-test

```



#### 16.7.4命名模板（涉及go语言）

命名模板我们也可以称为⼦模板，是限定在⼀个⽂件内部的模板，然后给⼀个名称。在使⽤命名模板 的时候有⼀个需要特别注意的是：模板名称是全局的，如果我们声明了两个相同名称的模板，最后加 载的⼀个模板会覆盖掉另外的模板，由于⼦ chart 中的模板也是和顶层的模板⼀起编译的，所以在命名 的时候⼀定要注意，不要重名了。为了避免重名，有个通⽤的约定就是为每个定义的模板添加上 chart 名称： {{define "mychart.labels"}} ， define 关键字就是⽤来声明命名模板的，加上 chart 名称就可以避免不同 chart 间的模板出现冲突的情况。

**声明和使⽤命名模板**

使⽤ define 关键字就可以允许我们在模板⽂件内部创建⼀个命名模板，它的语法格式如下：

```
{{ define "ChartName.TplName" }}
# 模板内容区域
{{ end }}
```

比如，现在我们可以定义一个模板来封装一个 label 标签：

```
{{- define "gomo-test.labels"}}
  labels:
    from: helm
    date: {{ now | htmlDate }}
{{- end }}
```

然后我们可以将该模板嵌入到现有的 ConfigMap 中，然后使用`template`关键字在需要的地方包含进来即可：

```bash
{{- define "gomo-test.labels"}}
  labels:
    from: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
  {{- template "gomo-test.labels" }}
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- $releaseName := .Release.Name -}}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  release: {{$releaseName}}
  {{- end}}
  ##########################################
  我们这个模板文件被渲染过后的结果如下所示：
  # Source: gomo/templates/with-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
  labels:
    from: helm
    date: 2019-12-19
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
  release: gomo-values-test

```

我们可以看到`define`区域定义的命名模板被嵌入到了`template`所在的区域，但是如果我们将命名模板全都写入到一个模板文件中的话无疑也会增大模板的复杂性。

还记得我们在创建 chart 包的时候，templates 目录下面默认会生成一个`_helpers.tpl`文件吗？我们前面也提到过 templates 目录下面除了`NOTES.txt`文件和以下划线`_`开头命令的文件之外，都会被当做 kubernetes 的资源清单文件，而这个下划线开头的文件不会被当做资源清单外，还可以被其他 chart 模板中调用，这个就是 Helm 中的`partials`文件，所以其实我们完全就可以将命名模板定义在这些`partials`文件中，默认就是`_helpers.tpl`文件了。

现在我们将上面定义的命名模板移动到 templates/_helpers.tpl 文件中去：

```bash
[root@k8s-master1 gomo]# tree 
.
├── charts
├── Chart.yaml
├── templates
│   ├── diy-cm.yaml
│   ├── _helpers.tpl
│   └── with-cm.yaml
└── values.yaml


########################
[root@k8s-master1 gomo]# cat templates/_helpers.tpl 
{{/* create labels  */}}
{{- define "gomo-test.labels" }}
  labels:
    from: helm
    date: {{ now | htmlDate }}
{{- end }}
[root@k8s-master1 gomo]# cat templates/with-cm.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
  {{- template "gomo-test.labels" }}
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- $releaseName := .Release.Name -}}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  release: {{$releaseName}}
  {{- end}}

#######################
渲染之后的东西还是一样的：
# Source: gomo/templates/with-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
  labels:
    from: helm
    date: 2019-12-19
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
  release: gomo-values-test

```

**模板范围**

上面我们定义的命名模板中，没有使用任何对象，只是使用了一个简单的函数，如果我们在里面来使用 chart 对象相关信息呢：

```
[root@k8s-master1 gomo]# cat templates/_helpers.tpl 
{{/* create labels  */}}
{{- define "gomo-test.labels" }}
  labels:
    from: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}
```

chart 的名称和版本都没有正确被渲染，这是因为他们不在我们定义的模板范围内，当命名模板被渲染时，它会接收由 template 调用时传入的作用域，由于我们这里并没有传入对应的作用域，因此模板中我们无法调用到 .Chart 对象，要解决也非常简单，我们只需要在 template 后面加上作用域范围即可

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{.Release.Name}}-configmap
  {{- template "gomo-test.labels" . }}   #这里加一个点 表示当前的作用域
data:
  myvalue: {{ .Values.hello | default "Hello Gomo" | quote }}
  {{- $releaseName := .Release.Name -}}
  {{- with .Values.course}}
  yunwei: {{ .yunwei | quote }}
  go: {{.go | repeat 3 | quote}}
  python: {{.python | upper |quote }}
  release: {{$releaseName}}
  {{- end}}


##############################
[root@k8s-master1 gomo]# helm install gomo-values-test --debug --dry-run ./
---
# Source: gomo/templates/with-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
  labels:
    from: helm
    date: 2019-12-19
    chart: gomo
    version: 0.1.0
data:
  myvalue: "Hello Gomo"
  yunwei: "sosad"
  go: "ououou"
  python: "EMMMM"
  release: gomo-values-test

```

我们可以看到 chart 的名称和版本号都已经被正常渲染出来了。

**include 函数**

假如现在我们将上面的定义的 labels 单独提取出来放置到 _helpers.tpl 文件中：

```
{{/* create labels  */}}
{{- define "gomo-test.labels" }}
from: helm
date: {{ now | htmlDate }}
chart: {{ .Chart.Name }}
version: {{ .Chart.Version }}
{{- end }}
```

现在我们将该命名模板插入到 configmap 模板文件的 labels 部分和 data 部分：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
    {{- template "gomo-test.labels" . }}
data:
  {{- range $key, $value := .Values.course }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
  {{- template "gomo-test.labels" . }}
  
  #################
  然后同样的查看下渲染的结果：
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-test-configmap
  labels:
from: helm
date: 2019-12-19
chart: gomo
version: 0.1.0
data:
  k8s: "devops"
  python: "django"
from: helm
date: 2019-12-19
chart: gomo
version: 0.1.0
```

我们可以看到渲染结果是有问题的，不是一个正常的 YAML 文件格式，这是因为`template`只是表示一个嵌入动作而已，不是一个函数，所以原本命名模板中是怎样的格式就是怎样的格式被嵌入进来了，比如我们可以在命名模板中给内容区域都空了两个空格，再来查看下渲染的结构：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-test-configmap
  labels:
  from: helm
  date: 2019-12-19
  chart: gomo
  version: 0.1.0
data:
  k8s: "devops"
  python: "django"
  from: helm
  date: 2019-12-19
  chart: gomo
  version: 0.1.0
```

我们可以看到 data 区域里面的内容是渲染正确的，但是上面 labels 区域是不正常的，因为命名模板里面的内容是属于 labels 标签的，是不符合我们的预期的，但是我们又不可能再去把命名模板里面的内容再增加两个空格，因为这样的话 data 里面的格式又不符合预期了。

为了解决这个问题，Helm 提供了另外一个方案来代替`template`，那就是使用`include`函数，在需要控制空格的地方使用`indent`管道函数来自己控制，比如上面的例子我们替换成`include`函数：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{- include "gomo-test.labels" . | indent 4 }}
data:
  {{- range $key, $value := .Values.course }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- include "gomo-test.labels" . | indent 2 }}
```

在 labels 区域我们需要4个空格，所以在管道函数`indent`中，传入参数4就可以，而在 data 区域我们只需要2个空格，所以我们传入参数2即可以，现在我们来渲染下我们这个模板看看是否符合预期呢：

```
# Source: gomo/templates/with-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gomo-values-test-configmap
  labels:    
    from: helm
    date: 2019-12-19
    chart: gomo
    version: 0.1.0
data:
  go: "ou"
  python: "emmmm"
  yunwei: "sosad"  
  from: helm
  date: 2019-12-19
  chart: gomo
  version: 0.1.0
```

可以看到是符合我们的预期，所以在 Helm 模板中我们使用 include 函数要比 template 更好，可以更好地处理 YAML 文件输出格式。

### 16.8Helm模板之其他注意事项

NOTES.txt 文件的使用、子 Chart 的使用、全局值的使用，这节课我们就来和大家一起了解下这些知识点。

**NOTES.txt 文件**

### 16.9 helm Hooks

和 Kubernetes 里面的容器一样，Helm 也提供了 [Hook](https://docs.helm.sh/developing_charts/#hooks) 的机制，允许 chart 开发人员在 release 的生命周期中的某些节点来进行干预，比如我们可以利用 Hooks 来做下面的这些事情：

- 在加载任何其他 chart 之前，在安装过程中加载 ConfigMap 或 Secret
- 在安装新 chart 之前执行作业以备份数据库，然后在升级后执行第二个作业以恢复数据
- 在删除 release 之前运行作业，以便在删除 release 之前优雅地停止服务

值得注意的是 Hooks 和普通模板一样工作，但是它们具有特殊的注释，可以使 Helm 以不同的方式使用它们。

```
apiVersion: ...
kind: ....
metadata:
  annotations:
    "helm.sh/hook": "pre-install"
```

- 预安装`pre-install`：在模板渲染后，kubernetes 创建任何资源之前执行
- 安装后`post-install`：在所有 kubernetes 资源安装到集群后执行
- 预删除`pre-delete`：在从 kubernetes 删除任何资源之前执行删除请求
- 删除后`post-delete`：删除所有 release 的资源后执行
- 升级前`pre-upgrade`：在模板渲染后，但在任何资源升级之前执行
- 升级后`post-upgrade`：在所有资源升级后执行
- 预回滚`pre-rollback`：在模板渲染后，在任何资源回滚之前执行
- 回滚后`post-rollback`：在修改所有资源后执行回滚请求
- `crd-install`：在运行其他检查之前添加 CRD 资源，只能用于 chart 中其他的资源清单定义的 CRD 资源。

**生命周期**

Hooks 允许开发人员在 release 的生命周期中的一些关键节点执行一些钩子函数，我们正常安装一个 chart 包的时候的生命周期如下所示：

- 用户运行`helm install foo`
- chart 被加载到服务端 Tiller Server 中
- 经过一些验证，Tiller Server 渲染 foo 模板
- Tiller 将产生的资源加载到 kubernetes 中去
- Tiller 将 release 名称和其他数据返回给 Helm 客户端
- Helm 客户端退出

如果开发人员在 install 的生命周期中定义了两个 hook：`pre-install`和`post-install`，那么我们安装一个 chart 包的生命周期就会多一些步骤了：

- 用户运行`helm install foo`
- chart 被加载到服务端 Tiller Server 中
- 经过一些验证，Tiller Server 渲染 foo 模板
- Tiller 将 hook 资源加载到 kubernetes 中，准备执行`pre-install` hook
- Tiller 会根据权重对 hook 进行排序（默认分配权重0，权重相同的 hook 按升序排序）
- Tiller 然后加载最低权重的 hook
- Tiller 等待，直到 hook 准备就绪
- Tiller 将产生的资源加载到 kubernetes 中
- Tiller 执行`post-install` hook
- Tiller 等待，直到 hook 准备就绪
- Tiller 将 release 名称和其他数据返回给客户端
- Helm 客户端退出

等待 hook 准备就绪，这是一个阻塞的操作，如果 hook 中声明的是一个 Job 资源，那么 Tiller 将等待 Job 成功完成，如果失败，则发布失败，在这个期间，Helm 客户端是处于暂停状态的。

对于所有其他类型，只要 kubernetes 将资源标记为加载（添加或更新），资源就被视为**就绪**状态，当一个 hook 声明了很多资源是，这些资源是被串行执行的。

另外需要注意的是 hook 创建的资源不会作为 release 的一部分进行跟踪和管理，一旦 Tiller Server 验证了 hook 已经达到了就绪状态，它就不会去管它了。

所以，如果我们在 hook 中创建了资源，那么不能依赖`helm delete`去删除资源，因为 hook 创建的资源已经不受控制了，要销毁这些资源，需要在`pre-delete`或者`post-delete`这两个 hook 函数中去执行相关操作，或者将`helm.sh/hook-delete-policy`这个 annotation 添加到 hook 模板文件中。

```
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Release.Name }}-hook-test-job
  lables:
    release: {{ .Release.Name }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
  annotations:
    #注意，如果没有下面的这个注释的话，当前的这个Job就会被当成release的一部分
    "helm.sh/hook": post-install   # ,可以添加多个钩子
    "helm.sh/hook-weight": "-5"    #添加权重 但是必须要是字符串
    "helm.sh/hook-delete-policy": hook-succeeded     #删除hook资源的策略
spec:
  template:
    metadata:
      name: {{ .Release.Name }}
      labels:
        release: {{ .Release.Name }}
        chart: {{ .Chart.Name }}
        version: {{ .Chart.Version }}
    spec:
      restartPolicy: Never
      containers:
      - name: hook-test-job
        image: alpine
        command: ["/bin/sleep", "{{ default "10" .Values.sleepTime }}"]
```

删除资源的策略可供选择的注释值：

- `hook-succeeded`：表示 Tiller 在 hook 成功执行后删除 hook 资源
- `hook-failed`：表示如果 hook 在执行期间失败了，Tiller 应该删除 hook 资源
- `before-hook-creation`：表示在删除新的 hook 之前应该删除以前的 hook

当 helm 的 release 更新时，有可能 hook 资源已经存在于群集中。默认情况下，helm 会尝试创建资源，并抛出错误**"... already exists"**。

我们可以选择 "helm.sh/hook-delete-policy": "before-hook-creation"，取代 "helm.sh/hook-delete-policy": "hook-succeeded,hook-failed" 因为：

例如为了手动调试，将错误的 hook 作业资源保存在 kubernetes 中是很方便的。 出于某种原因，可能有必要将成功的 hook 资源保留在 kubernetes 中。同时，在 helm release 升级之前进行手动资源删除是不可取的。 "helm.sh/hook-delete-policy": "before-hook-creation" 在 hook 中的注释，如果在新的 hook 启动前有一个 hook 的话，会使 Tiller 将以前的release 中的 hook 删除，而这个 hook 同时它可能正在被其他一个策略使用。

### 16.10开发自己Chart: tomcar应用为例 

```
[root@k8s-master2 ~]# helm create tomcat
Creating tomcat
[root@k8s-master2 ~]# ls
anaconda-ks.cfg  helm  k8s-elk  kubernetes.sh  rbac-config.yaml  tomcat


#在tomcat/values.yaml定义镜像和暴露端口：
image:
  repository: tomcat
  tag: latest
  pullPolicy: IfNotPresent
service:
  type: NodePort
  port: 80
  
  
  #修改deployment.yaml文件 去修改掉探针并修改默认端口；
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          #livenessProbe:
          #  httpGet:
          #    path: /
          #    port: http
          #readinessProbe:
          #  httpGet:
          #    path: /
          #    port: http
          
          
[root@k8s-master1 tomcat]# helm install mytomcat .
NAME: mytomcat
LAST DEPLOYED: Thu Dec 19 02:29:38 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services mytomcat)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

```

![image-20191219153920747](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219153920747.png)

### 16.11 安装私有仓库（minio）

使用docker

```
docker run -p 9000:9000 --name minio1   -v /mnt/data:/home/data   minio/minio server /data
```

退出 查找ID 并启动：

```
docker start bdceb168627d
```

登录192.168.208.195:9000  帐号密码为minioadmin

![image-20191219161409910](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161409910.png)

   浏览器登录 minio，点击右下角的“新增”按钮，选择 "Create bucket"：

![image-20191219161517338](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161517338.png)

  填写 “Bucket Name” 回车，创建 helm 仓库：

![image-20191219161536375](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161536375.png)

   选择创建好的 helm 仓库，点击“更多”图标：如下图所示：

<img src="C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161605562.png" alt="image-20191219161605562" style="zoom:80%;" />![image-20191219161712341](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161712341.png)

在弹出框中选择 “Read and write” ，然后点击“新增（Add）”按钮：

 点击“关闭”按钮结束配置：

![image-20191219161746699](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219161746699.png)

 自此，完成自建 helm 私有仓库。

上面完成了私有仓库的创建。下一步就可以将 helm 跟私有仓库进行关联了。执行如下命令：

\## 注意不要忘记私有仓库名 “helm-repo” 

```
[root@k8s-master2 ~]# helm repo add minio http://192.168.208.195:9000/helm-repo
Error: looks like "http://192.168.208.195:9000/helm-repo" is not a valid chart repository or cannot be reached: failed to fetch http://192.168.208.195:9000/helm-repo/index.yaml : 404 Not Found
```

 执行报错，helm 3 认为创建的私有仓库无效，因为缺少 index.yaml 文件。执行命令生成 index.yaml 文件。

```
[root@k8s-master2 ~]# mkdir -p /root/helm/repo 
[root@k8s-master2 ~]# helm repo index /root/helm/repo
[root@k8s-master2 ~]# cd /root/helm/repo/
[root@k8s-master2 repo]# ls
index.yaml
```

![image-20191219162047197](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191219162047197.png)

```
[root@k8s-master2 repo]# helm repo add minio http://192.168.208.195:9000/helm-repo
"minio" has been added to your repositories
[root@k8s-master2 repo]# helm repo list
NAME  	URL                                              
stable	https://kubernetes-charts.storage.googleapis.com/
minio 	http://192.168.208.195:9000/helm-repo         
```

