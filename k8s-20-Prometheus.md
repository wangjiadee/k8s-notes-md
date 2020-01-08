## 20、Prometheus

### 20.1Prometheus 监控⼊⻔简介

prometheus 是什么？

[Prometheus](https://github.com/prometheus) is an open-source systems monitoring and alerting toolkit originally built at [SoundCloud](https://soundcloud.com). Since its inception in 2012, many companies and organizations have adopted Prometheus, and the project has a very active developer and user [community](https://prometheus.io/community). It is now a standalone open source project and maintained independently of any company. To emphasize this, and to clarify the project's governance structure, Prometheus joined the [Cloud Native Computing Foundation](https://cncf.io/) in 2016 as the second hosted project, after [Kubernetes](https://kubernetes.io/).

官网：https://prometheus.io/

prometheus的结构图：

​                数据采集的2总方式                                 服务发现                                                     prometheus 的报警

![image-20191223093846523](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223093846523.png)

~ prometheus 也是一种C/S 模式 

~使用时间序列模型 time series（x,y模型）

~基于K/V的数据模型（不带单位）

~基于数学运算 来查询

~采用HTTP pull/push 两种对应的数据采集传输方式

~自带图形调试（sql查询语句 ）



prometheus 的优点

- 监控数据的精细程度 绝对的第⼀ 可以精确到 1～5秒的采集精度 4 5分钟 理想的状态 集精度越小越好 （但是过小的话  会带来 存储 性能 的问题）
- 集群部署的速度 监控脚本的制作 （指的是熟练之后） ⾮常快速 ⼤⼤缩短监控的搭建时间成本
- 周边插件很丰富 exporter pushgateway ⼤多数都不需要⾃⼰开发了
- 本⾝基于数学计算模型，⼤量的实⽤函数 可以实现很复杂规则的业务逻辑监控（例如QPS的曲线弯曲 凸起 下跌的 ⽐例等等模糊概念）
- 可以嵌⼊很多开源⼯具的内部 进⾏监控 数据更准时 更可信（其他监控很难做到这⼀点）
- 本⾝是开源的，更新速度快，bug修复快。⽀持N多种语⾔做本⾝和插件的⼆次开发
- 图形很⾼⼤上 很美观 ⽼板特别喜欢看这种业务图 （主要是指跟Grafana的结合）

prometheus 的缺点：

- 因其数据采集的精度 如果集群数量太⼤，那么单点的监控有性能瓶颈 ⽬前尚不⽀持集群 只能workaround
- 学习成本太⼤，尤其是其独有的数学命令⾏（⾮常强⼤的同时 又极其难学《=⾃学的情况下），中⽂资料极少，本⾝的各种数学模型的概念很复杂
- 对磁盘资源也是耗费的较⼤，这个具体要看 监控的集群量 和 监控项的多少 和保存时间的长短
- 本⾝的使⽤ 需要使⽤者的数学不能太差 要有⼀定的数学头脑
- 1.0版本 可能会发生数据丢失  2.0之后就没有了

### 20.2Prometheus 运⾏框架介绍

**组件**
Prometheus 由多个组件组成，但是其中许多组件是可选的：

- Prometheus Server：用于抓取指标、存储时间序列数据（核心的程序）

- exporter：暴露指标让任务来抓（node_exporter）

- pushgateway：push 的方式将指标数据推送到该网关

- alertmanager：处理报警的报警组件

- adhoc：用于数据查询

  大多数 Prometheus 组件都是用Go 编写的，因此很容易构建和部署为静态的二进制文件。

  TSDB 时间序列数据库

### 20.3Prometheus 数据格式介绍

prometheus监控中 对于采集过来的数据 统一称为metrics数据

metrics 是一种对采集数据的总称

metrics的几个种类：

- Gauges：

  最简单的度量指标，只有一个简单的返回值，或者叫做瞬时状态。

  监控硬盘的容量和内存的使用量 ，而这两种量 是随着时间的推移 不断的瞬时的 没有规则的变化的

  数据可能是增长 也有可能是降低

  ```
  [root@k8s-worker3 k8sv]# curl 127.0.0.1:9100/metrics
  ...omit....
  # HELP process_max_fds Maximum number of open file descriptors.
  # TYPE process_max_fds gauge
  process_max_fds 65536
  ...omit....
  ```

  

- Counters

  就是一个计数器 数据从0开始计算 在理想状态下 只能是永远增长 不会下降（被清零除外）

  比如 用户的访问量  只会一直的增长

- Histograms

  是统计数据的分布情况。类似于百分比。这是一种特殊的数据类型  例如：

  Http_response_time HTTP的响应时间 代表的是 一次用户HTTP请球 在系统传输和执行过程中总花费的时间

  在现实的环境中 会有一些慢请求等等。。。 如果你算总的平均值的话可能是看不出什么问题来的

  histograms 是通过一段 或者一部分的比例来算  （在1s内的平均值  2s内的平均值）

K/V的数据形式：

### 20.4Prometheus 初探和配置（安装测试）

~：prometheus是用T_S 的 所以时间同步很重要！

普通安装：

从官网上下载 并解压

```
$wget https://github.com/prometheus/prometheus/releases/download/v2.14.0/prometheus-2.14.0.linux-amd64.tar.gz
$tar xvzf prometheus-2.14.0.linux-amd64.tar.gz
$cd prometheus-2.14.0.linux-amd64 && ls 
console_libraries  consoles  LICENSE  NOTICE  prometheus  prometheus.yml  promtool  tsdb

```

使用二进制开启 也是可以的 但是退出即销毁

```
$ ./prometheus --config.file=prometheus.yml
```

查看yml文件信息：（主配置文件+）

```linux
# my global config  控制全局配置
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

```

上面这个配置文件中包含了3个模块：global、rule_files 和 scrape_configs。

其中 global 模块控制 Prometheus Server 的全局配置：

- scrape_interval：表示 prometheus 抓取指标数据的频率，默认是15s，我们可以覆盖这个值
- evaluation_interval：用来控制评估规则的频率，prometheus 使用规则产生新的时间序列数据或者产生警报

rule_files 模块制定了规则所在的位置，prometheus 可以根据这个配置加载规则，用于生成新的时间序列数据或者报警信息，当前我们没有配置任何规则.

scrape_configs 用于控制 prometheus 监控哪些资源。由于 prometheus 通过 HTTP 的方式来暴露的它本身的监控数据，prometheus 也能够监控本身的健康情况。在默认的配置里有一个单独的 job，叫做prometheus，它采集 prometheus 服务本身的时间序列数据。这个 job 包含了一个单独的、静态配置的目标：监听 localhost 上的9090端口。prometheus 默认会通过目标的`/metrics`路径采集 metrics。所以，默认的 job 通过 URL：`http://localhost:9090/metrics`采集 metrics。收集到的时间序列包含 prometheus 服务本身的状态和性能。如果我们还有其他的资源需要监控的话，直接配置在该模块下面就可以了。

由于我们这里是要跑在 Kubernetes 系统中，所以我们直接用 Docker 镜像的方式运行即可。

```
kubectl create ns prometheus
```

为了能够方便的管理配置文件，我们这里将 prometheus.yml 文件用 ConfigMap 的形式进行管理：（prometheus-cm.yaml）

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
```

配置文件创建完成了，以后如果我们有新的资源需要被监控，我们只需要将上面的 ConfigMap 对象更新即可。现在我们来创建 prometheus 的 Pod 资源：(prometheus-deploy.yaml)

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    app: prometheus
spec:
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:latest
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/prometheus"
          subPath: prometheus
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      securityContext:
        runAsUser: 0
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          name: prometheus-config
        name: config-volume
```

我们在启动程序的时候，除了指定了 prometheus.yml 文件之外，还通过参数`storage.tsdb.path`指定了 TSDB 数据的存储路径、通过`storage.tsdb.retention`设置了保留多长时间的数据，还有下面的`web.enable-admin-api`参数可以用来开启对 admin api 的访问权限，参数`web.enable-lifecycle`非常重要，用来开启支持热更新的，有了这个参数之后，prometheus.yml 配置文件只要更新了，通过执行`localhost:9090/-/reload`就会立即生效，所以一定要加上这个参数。

------

在1.16 之后 k8s的api 删除了一些东西 其中deployment的api 有且只有app/vs1

官方提供了转换功能：

```
[root@k8s-master3 prometheus]# kubectl convert -f ./prometheus-deployment.yaml --output-version apps/v1
kubectl convert is DEPRECATED and will be removed in a future version.
In order to convert, kubectl apply the object to the cluster, then kubectl get at the desired version.
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: prometheus
  name: prometheus
  namespace: prometheus
spec:
  progressDeadlineSeconds: 2147483647
  replicas: 1
  revisionHistoryLimit: 2147483647
  selector:
    matchLabels:
      app: prometheus
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: prometheus
    spec:
      containers:
      - args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        - --storage.tsdb.retention=24h
        - --web.enable-admin-api
        - --web.enable-lifecycle
        command:
        - /bin/prometheus
        image: prom/prometheus:latest
        imagePullPolicy: Always
        name: prometheus
        ports:
        - containerPort: 9090
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 512Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /prometheus
          name: data
          subPath: prometheus
        - mountPath: /etc/prometheus
          name: config-volume
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        runAsUser: 0
      serviceAccount: prometheus
      serviceAccountName: prometheus
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          defaultMode: 420
          name: prometheus-config
        name: config-volume
status: {}
// status ｛｝可以不用复制
```

我们这里将 prometheus.yml 文件对应的 ConfigMap 对象通过 volume 的形式挂载进了 Pod，这样 ConfigMap 更新后，对应的 Pod 里面的文件也会热更新的，然后我们再执行上面的 reload 请求，Prometheus 配置就生效了，除此之外，为了将时间序列数据进行持久化，我们将数据目录和一个 pvc 对象进行了绑定，所以我们需要提前创建好这个 pvc 对象：(prometheus-volume.yaml)

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 192.168.208.195
    path: /home/k8sv

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus
  namespace: prometheus
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

除了上面的注意事项外，我们这里还需要配置 rbac 认证，因为我们需要在 prometheus 中去访问 Kubernetes 的相关信息，所以我们这里管理了一个名为 prometheus 的 serviceAccount 对象：(prometheus-rbac.yaml)

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: prometheus

```

由于我们要获取的资源信息，在每一个 namespace 下面都有可能存在，所以我们这里使用的是 ClusterRole 的资源对象，值得一提的是我们这里的权限规则声明中有一个`nonResourceURLs`的属性，是用来对非资源型 metrics 进行操作的权限声明，这个在以前我们很少遇到过，然后直接创建上面的资源对象即可：

```
[root@k8s-master3 prometheus]# kubectl create -f .
configmap/prometheus-config created
deployment.apps/prometheus created
serviceaccount/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
persistentvolume/prometheus created
persistentvolumeclaim/prometheus created


[root@k8s-master3 prometheus]# kubectl get all -o wide -n prometheus 
NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
pod/prometheus-5c9c5fb6cf-xkps2   1/1     Running   0          38m   172.24.24.237   k8s-worker4   <none>           <none>

NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
service/prometheus   NodePort   10.254.54.121   <none>        9090:31044/TCP   38m   app=prometheus

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                   SELECTOR
deployment.apps/prometheus   1/1     1            1           38m   prometheus   prom/prometheus:latest   app=prometheus

NAME                                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                   SELECTOR
replicaset.apps/prometheus-5c9c5fb6cf   1         1         1       38m   prometheus   prom/prometheus:latest   app=prometheus,pod-template-hash=5c9c5fb6cf

```

打开浏览器访问：

![image-20191223130839543](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223130839543.png)



**监控 Kubernetes 集群节点**

对于集群的监控一般我们需要考虑以下几个方面：

- Kubernetes 节点的监控：比如节点的 cpu、load、disk、memory 等指标
- 内部系统组件的状态：比如 kube-scheduler、kube-controller-manager、kubedns/coredns 等组件的详细运行状态
- 编排级的 metrics：比如 Deployment 的状态、资源请求、调度和 API 延迟等数据指标

**监控方案**

ubernetes 集群的监控方案目前主要有以下几种方案：



Heapster：Heapster 是一个集群范围的监控和数据聚合工具，以 Pod 的形式运行在集群中。 ![heapster](https://www.qikqiak.com/k8s-book/docs/images/kubernetes_monitoring_heapster.png) 

- [x] 需要注意的是 Heapster 已经被废弃了，后续版本中会使用 metrics-server 代替。

- cAdvisor：[cAdvisor](https://github.com/google/cadvisor)是`Google`开源的容器资源监控和性能分析工具，它是专门为容器而生，本身也支持 Docker 容器，在 Kubernetes 中，我们不需要单独去安装，cAdvisor 作为 kubelet 内置的一部分程序可以直接使用。
- Kube-state-metrics：[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)通过监听 API Server 生成有关资源对象的状态指标，比如 Deployment、Node、Pod，需要注意的是 kube-state-metrics 只是简单提供一个 metrics 数据，并不会存储这些指标数据，所以我们可以使用 Prometheus 来抓取这些数据然后存储。
- metrics-server：metrics-server 也是一个集群范围内的资源数据聚合工具，是 Heapster 的替代品，同样的，metrics-server 也只是显示数据，并不提供数据存储服务。

不过 kube-state-metrics 和 metrics-server 之间还是有很大不同的，二者的主要区别如下：

- kube-state-metrics 主要关注的是业务相关的一些元数据，比如 Deployment、Pod、副本状态等
- metrics-server 主要关注的是[资源度量 API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md) 的实现，比如 CPU、文件描述符、内存、请求延时等指标。

**监控集群节点**（node_exporter）

现在我们就来开始我们集群的监控工作，首先来监控我们集群的节点，要监控节点其实我们已经有很多非常成熟的方案了，比如 Nagios、zabbix，甚至我们自己来收集数据也可以，我们这里通过 Prometheus 来采集节点的监控指标数据，可以通过[node_exporter](https://github.com/prometheus/node_exporter)来获取，顾名思义，node_exporter 就是抓取用于采集服务器节点的各种运行指标，目前 node_exporter 支持几乎所有常见的监控点，比如 conntrack，cpu，diskstats，filesystem，loadavg，meminfo，netstat等，详细的监控点列表可以参考其[Github repo](https://github.com/prometheus/node_exporter)。

```
#使用helm 来进行安装
[root@k8s-master2 prometheus-node-exporter]# helm install prometheus-node --namespace=prometheus -f values.yaml stable/prometheus-node-exporter

#values.yaml 并没有修改 就是查看了一下
#检测：
[root@k8s-master1 redis-ha]# kubectl get pods -n prometheus
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-5c9c5fb6cf-xkps2                      1/1     Running   0          131m
prometheus-node-prometheus-node-exporter-49z5c   1/1     Running   0          13s
prometheus-node-prometheus-node-exporter-4p5vs   1/1     Running   0          13s
prometheus-node-prometheus-node-exporter-9bwg9   1/1     Running   0          13s
prometheus-node-prometheus-node-exporter-rwzzr   1/1     Running   0          13s


#在主节点上检测
curl 127.0.0.1:9100/metrics
.....omit.....
# TYPE process_virtual_memory_max_bytes gauge
process_virtual_memory_max_bytes -1
# HELP promhttp_metric_handler_requests_in_flight Current number of scrapes being served.
# TYPE promhttp_metric_handler_requests_in_flight gauge
promhttp_metric_handler_requests_in_flight 1
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 1
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0
#返回值 是200 表示成功

#在prometheus-cm.yml 文件中添加对应job
```

 prometheus 去发现 Node 模式的服务的时候，访问的端口默认是**10250**，而现在该端口下面已经没有了`/metrics`指标数据了，现在 kubelet 只读的数据接口统一通过**10255**端口进行暴露了，所以我们应该去替换掉这里的端口，但是我们是要替换成**10255**端口吗？不是的，因为我们是要去配置上面通过`node-exporter`抓取到的节点指标数据，而我们上面是不是指定了`hostNetwork=true`，所以在每个节点上就会绑定一个端口**9100**，所以我们应该将这里的**10250**替换成**9100**，但是应该怎样替换呢？这里我们就需要使用到 Prometheus 提供的`relabel_configs`中的`replace`能力了，relabel 可以在 Prometheus 采集数据之前，通过Target 实例的 Metadata 信息，动态重新写入 Label 的值。除此之外，我们还能根据 Target 实例的 Metadata 信息选择是否采集或者忽略该 Target 实例。比如我们这里就可以去匹配`__address__`这个 Label 标签，然后替换掉其中的端口：这里就是一个正则表达式，去匹配`__address__`，然后将 host 部分保留下来，port 替换成了**9100**，现在我们重新更新配置文件，执行 reload 操作，然后再去看 Prometheus 的 Dashboard 的 Targets 路径下面 kubernetes-nodes 这个 job 任务是否正常了：

```
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: '(.*):10250'
    replacement: '${1}:9100'
    target_label: __address__
    action: replace
    
    #最后在节点上：
    [root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"
```

![image-20191223144010925](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223144010925.png)

可以看到现在已经正常了，但是还有一个问题就是我们采集的指标数据 Label 标签就只有一个节点的 hostname，这对于我们在进行监控分组分类查询的时候带来了很多不方便的地方，要是我们能够将集群中 Node 节点的 Label 标签也能获取到就很好了。

这里我们可以通过`labelmap`这个属性来将 Kubernetes 的 Label 标签添加为 Prometheus 的指标标签：

```
- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
  relabel_configs:
  - source_labels: [__address__]
    regex: '(.*):10250'
    replacement: '${1}:9100'
    target_label: __address__
    action: replace
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
- job_name: 'kubernetes-kubelet'
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
```

添加了一个 action 为`labelmap`，正则表达式是`__meta_kubernetes_node_label_(.+)`的配置，这里的意思就是表达式中匹配都的数据也添加到指标数据的 Label 标签中去。

对于 kubernetes_sd_configs 下面可用的标签如下： 可用元标签：

- __meta_kubernetes_node_name：节点对象的名称
- *_meta_kubernetes_node_label*：节点对象中的每个标签
- *_meta_kubernetes_node_annotation*：来自节点对象的每个注释
- *_meta_kubernetes_node_address*：每个节点地址类型的第一个地址（如果存在） *

> 关于 kubernets_sd_configs 更多信息可以查看官方文档：[kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#)

另外由于 kubelet 也自带了一些监控指标数据，就上面我们提到的**10255**端口，所以我们这里也把 kubelet 的监控任务也一并配置上。**特别需要注意**的是 Kubernetes 1.11+ 版本以后，kubelet 就移除了 10255 端口， metrics 接口又回到了 10250 端口中，所以这里不需要替换端口，但是需要使用 https 的协议。

![image-20191223144825657](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223144825657.png)

现在我们就可以切换到 Graph 路径下面查看采集的一些指标数据了，比如查询 node_load1 指标：

![image-20191223145059052](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223145059052.png)

我们可以看到将4个 node 节点对应的 node_load1 指标数据都查询出来了，同样的，我们还可以使用 PromQL 语句来进行更复杂的一些聚合查询操作，还可以根据我们的 Labels 标签对指标数据进行聚合，比如我们这里只查询 node03 节点的数据，可以使用表达式`node_load1{instance="node03"}`来进行查询：

![image-20191223145148898](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223145148898.png)

**监控 Kubernetes 常用资源对象**

Prometheus 中来自动监控 Kubernetes 中的一些常用资源对象.Prometheus 中用静态的方式来监控 Kubernetes 集群中的普通应用，但是如果针对集群中众多的资源对象都采用静态的方式来进行配置的话显然是不现实的，所以同样我们需要使用到 Prometheus 提供的其他类型的服务发现机制。

**容器监控**

说到容器监控我们自然会想到`cAdvisor`，我们前面也说过`cAdvisor`已经内置在了 kubelet 组件之中，所以我们不需要单独去安装，`cAdvisor`的数据路径为`/api/v1/nodes//proxy/metrics`，同样我们这里使用 node 的服务发现模式，因为每一个节点下面都有 kubelet，自然都有`cAdvisor`采集到的数据指标，配置如下：

```
- job_name: 'kubernetes-cadvisor'
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - target_label: __address__
    replacement: kubernetes.default.svc:443
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

上面的配置和我们之前配置 node-exporter 的时候几乎是一样的，区别是我们这里使用了 https 的协议，另外需要注意的是配置了 ca.cart 和 token 这两个文件，这两个文件是 Pod 启动后自动注入进来的，通过这两个文件我们可以在 Pod 中访问 apiserver，比如我们这里的`__address__`不在是 nodeip 了，而是 kubernetes 在集群中的服务地址，然后加上`__metrics_path__`的访问路径：`/api/v1/nodes/${1}/proxy/metrics/cadvisor`，现在同样更新下配置，然后查看 Targets 路径：

![image-20191223151735652](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223151735652.png)

然后我们可以切换到 Graph 路径下面查询容器相关数据，比如我们这里来查询集群中所有 Pod 的 CPU 使用情况，这里用的数据指标是 container_cpu_usage_seconds_total，然后去除一些无效的数据，查询1分钟内的数据，由于查询到的数据都是容器相关的，最好要安装 Pod 来进行聚合，对应的`promQL`语句如下：

```
rate(container_cpu_usage_seconds_total{image!=""}[1m])
```

![image-20191223151648452](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223151648452.png)

我们可以看到上面的结果就是集群中的所有 Pod 在1分钟之内的 CPU 使用情况的曲线图，当然还有很多数据可以获取到，我们后面在需要的时候再和大家介绍。

**apiserver 监控**

apiserver 作为 Kubernetes 最核心的组件，当然他的监控也是非常有必要的，对于 apiserver 的监控我们可以直接通过 kubernetes 的 Service 来获取：

```
[root@k8s-master1 ~]# kubectl get  svc
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AGE
kubernetes                       ClusterIP   10.254.0.1       <none>        443/TCP                       7d20h
```

上面这个 Service 就是我们集群的 apiserver 在集群内部的 Service 地址，要自动发现 Service 类型的服务，我们就需要用到 role 为 Endpoints 的 kubernetes_sd_configs，我们可以在 ConfigMap 对象中添加上一个 Endpoints 类型的服务的监控任务：

```
- job_name: 'kubernetes-apiservers'
  kubernetes_sd_configs:
  - role: endpoints
```

上面这个任务是定义的一个类型为`endpoints`的`kubernetes_sd_configs`，添加到 Prometheus 的 ConfigMap 的配置文件中，然后更新配置：

```
[root@k8s-master3 prometheus]# kubectl apply -f prometheus-cm.yaml 
configmap/prometheus-config configured
[root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"
```

更新完成后，我们再去查看 Prometheus 的 Dashboard 的 target 页面：

![image-20191223153936834](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223153936834.png)

我们可以看到 kubernetes-apiservers 下面出现了很多实例，这是因为这里我们使用的是 Endpoints 类型的服务发现，所以 Prometheus 把所有的 Endpoints 服务都抓取过来了，同样的，上面我们需要的服务名为`kubernetes`这个 apiserver 的服务也在这个列表之中`keep`，就是只把符合我们要求的给保留下来，哪些才是符合我们要求的呢？我们可以把鼠标放置在任意一个 target 上，可以查看到`Before relabeling`里面所有的元数据，比如我们要过滤的服务是 default 这个 namespace 下面，服务名为 kubernetes 的元数据，所以这里我们就可以根据对应的`__meta_kubernetes_namespace`和`__meta_kubernetes_service_name`这两个元数据来 relabel

另外由于 kubernetes 这个服务对应的端口是443，需要使用 https 协议，所以这里我们需要使用 https 的协议，对应的就需要将对应的 ca 证书配置上，如下：

```
- job_name: 'kubernetes-apiservers'
  kubernetes_sd_configs:
  - role: endpoints
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https
```

现在重新更新配置文件、重新加载 Prometheus，切换到 Prometheus 的 Targets 路径下查看

```
[root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"
[root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"
[root@k8s-master3 prometheus]# kubectl apply -f prometheus-cm.yaml 
configmap/prometheus-config unchanged
```

![image-20191223154657676](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223154657676.png)

现在可以看到 kubernetes-apiserver 这个任务下面只有 apiserver 这一个实例了，证明我们的 relabel 是成功的，现在我们切换到 graph 路径下面查看下采集到数据，比如查询 apiserver 的总的请求数：

```
sum(rate(apiserver_request_count[1m]))
```

![image-20191223154950752](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223154950752.png)

### 20.5Prometheus 数学理论基础学习 

- increase()

increase 函数 在promethes中，是⽤来 针对Counter 这种持续增 长的数值，截取其中⼀段时间的增量

increase(node_cpu[**1m**]) =》 这样 就获取了 CPU总使⽤时间 在1分钟内的增量

![image-20200106165953035](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106165953035.png)



- sum( **increase(node_cpu[1m])** )

外⾯套⽤⼀个sum 即可把所有核数值加合



- node_cpu{mode="idle”} => 空闲  #mode为选取什么的标签  



- by (**instance**)这个函数 可以把 sum加合到⼀起的数值 按照指定的⼀个⽅式进⾏⼀层的拆分

instance代表的是 机器名的

把sum函数中加合再给它强⾏拆分出来

![image-20200106170602331](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106170602331.png)

### 20.6Prometheus 命令⾏使⽤扩展

instance:node_cpu:ratio{instance=~"adm-worker3"}   ~模糊匹配

｛xx=xx｝筛选出复合括号内的标签 



instance:node_cpu:ratio{instance=~"adm-worker3"}  >0.2  表示显示出大于0.2的范围



- **rate()**

rate 函数可以说 是prometheus提供的 最重要的函数之⼀

**rate(. )** 函数 是专门搭配**counter**类型数据使⽤的函数 它的功能 是按照设置⼀个时间段，取**counter**在这个时间段中 的 平均**每秒**的增量

container_network_receive_bytes_total 本⾝是⼀个**counter**类型

rate(container_network_receive_bytes_total[1m]) 获取到 在**1**分钟时间内，平均每秒钟的 增量

![image-20200106171629470](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106171629470.png)

**任何counter数据类型的时候，先给它加上⼀个 rate() 或者 increase() 看看**

### 20.7Grafana 超实⽤企业级监控绘图⼯具的结合

grafana 是一个可视化面板，有着非常漂亮的图表和布局展示，功能齐全的度量仪表盘和图形编辑器，支持 Graphite、zabbix、InfluxDB、Prometheus、OpenTSDB、Elasticsearch 等作为数据源，比 Prometheus 自带的图表展示功能强大太多，更加灵活，有丰富的插件，功能更加强大。同样的，我们将 grafana 安装到 Kubernetes 集群中，第一步同样是去查看 grafana 的 docker 镜像的介绍，我们可以在 dockerhub 上去搜索，也可以在官网去查看相关资料，镜像地址如下：https://hub.docker.com/r/grafana/grafana/，我们可以看到介绍中运行 grafana 容器的命令非常简单：

```
docker run -d --name=grafana -p 3000:3000 grafana/grafana
```



使用helm搭建grafana：

```yaml
rbac:
  create: true
  pspEnabled: true
  pspUseAppArmor: true
  namespaced: false
  extraRoleRules: []
  # - apiGroups: []
  #   resources: []
  #   verbs: []
  extraClusterRoleRules: []
  # - apiGroups: []
  #   resources: []
  #   verbs: []
serviceAccount:
  create: true
  name:
  nameTest:
#  annotations:

replicas: 1

## See `kubectl explain poddisruptionbudget.spec` for more
## ref: https://kubernetes.io/docs/tasks/run-application/configure-pdb/
podDisruptionBudget: {}
#  minAvailable: 1
#  maxUnavailable: 1

## See `kubectl explain deployment.spec.strategy` for more
## ref: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy
deploymentStrategy:
  type: RollingUpdate

readinessProbe:
  httpGet:
    path: /api/health
    port: 3000

livenessProbe:
  httpGet:
    path: /api/health
    port: 3000
  initialDelaySeconds: 60
  timeoutSeconds: 30
  failureThreshold: 10

## Use an alternate scheduler, e.g. "stork".
## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
##
# schedulerName: "default-scheduler"

image:
  repository: grafana/grafana
  tag: 6.5.2
  pullPolicy: IfNotPresent

  ## Optionally specify an array of imagePullSecrets.
  ## Secrets must be manually created in the namespace.
  ## ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
  ##
  # pullSecrets:
  #   - myRegistrKeySecretName

testFramework:
  enabled: true
  image: "dduportal/bats"
  tag: "0.4.0"
  securityContext: {}

securityContext:
  runAsUser: 472
  fsGroup: 472


extraConfigmapMounts: []
  # - name: certs-configmap
  #   mountPath: /etc/grafana/ssl/
  #   subPath: certificates.crt # (optional)
  #   configMap: certs-configmap
  #   readOnly: true


extraEmptyDirMounts: []
  # - name: provisioning-notifiers
  #   mountPath: /etc/grafana/provisioning/notifiers


## Assign a PriorityClassName to pods if set
# priorityClassName:

downloadDashboardsImage:
  repository: appropriate/curl
  tag: latest
  pullPolicy: IfNotPresent

downloadDashboards:
  env: {}

## Pod Annotations
# podAnnotations: {}

## Pod Labels
# podLabels: {}

podPortName: grafana

## Deployment annotations
# annotations: {}

## Expose the grafana service to be accessed from outside the cluster (LoadBalancer service).
## or access it from within the cluster (ClusterIP service). Set the service type and the port to serve it.
## ref: http://kubernetes.io/docs/user-guide/services/
##
service:
  type: NodePort
  port: 80
  targetPort: 3000
    # targetPort: 4181 To be used with a proxy extraContainer
  annotations: {}
  labels: {}
  portName: service

ingress:
  enabled: false
  annotations: {}
    # kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  labels: {}
  path: /
  hosts:
    - chart-example.local
  ## Extra paths to prepend to every host configuration. This is useful when working with annotation based services.
  extraPaths: []
  # - path: /*
  #   backend:
  #     serviceName: ssl-redirect
  #     servicePort: use-annotation
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local

resources: {}
#  limits:
#    cpu: 100m
#    memory: 128Mi
#  requests:
#    cpu: 100m
#    memory: 128Mi

## Node labels for pod assignment
## ref: https://kubernetes.io/docs/user-guide/node-selection/
#
nodeSelector: {}

## Tolerations for pod assignment
## ref: https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/
##
tolerations: []

## Affinity for pod assignment
## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity
##
affinity: {}

extraInitContainers: []

## Enable an Specify container in extraContainers. This is meant to allow adding an authentication proxy to a grafana pod
extraContainers: |
# - name: proxy
#   image: quay.io/gambol99/keycloak-proxy:latest
#   args:
#   - -provider=github
#   - -client-id=
#   - -client-secret=
#   - -github-org=<ORG_NAME>
#   - -email-domain=*
#   - -cookie-secret=
#   - -http-address=http://0.0.0.0:4181
#   - -upstream-url=http://127.0.0.1:3000
#   ports:
#     - name: proxy-web
#       containerPort: 4181

## Enable persistence using Persistent Volume Claims
## ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
##
persistence:
  type: pvc
  enabled: true
  # storageClassName: default
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  # annotations: {}
  finalizers:
    - kubernetes.io/pvc-protection
  # subPath: ""
  # existingClaim:

initChownData:
  ## If false, data ownership will not be reset at startup
  ## This allows the prometheus-server to be run with an arbitrary user
  ##
  enabled: true

  ## initChownData container image
  ##
  image:
    repository: busybox
    tag: "1.30"
    pullPolicy: IfNotPresent

  ## initChownData resource requests and limits
  ## Ref: http://kubernetes.io/docs/user-guide/compute-resources/
  ##
  resources: {}
  #  limits:
  #    cpu: 100m
  #    memory: 128Mi
  #  requests:
  #    cpu: 100m
  #    memory: 128Mi


# Administrator credentials when not using an existing secret (see below)
adminUser: admin
# adminPassword: strongpassword

# Use an existing secret for the admin user.
admin:
  existingSecret: ""
  userKey: admin-user
  passwordKey: admin-password

## Define command to be executed at startup by grafana container
## Needed if using `vault-env` to manage secrets (ref: https://banzaicloud.com/blog/inject-secrets-into-pods-vault/)
## Default is "run.sh" as defined in grafana's Dockerfile
# command:
# - "sh"
# - "/run.sh"

## Use an alternate scheduler, e.g. "stork".
## ref: https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/
##
# schedulerName:

## Extra environment variables that will be pass onto deployment pods
env: {}

## "valueFrom" environment variable references that will be added to deployment pods
## ref: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.17/#envvarsource-v1-core
## Renders in container spec as:
##   env:
##     ...
##     - name: <key>
##       valueFrom:
##         <value rendered as YAML>
envValueFrom: {}

## The name of a secret in the same kubernetes namespace which contain values to be added to the environment
## This can be useful for auth tokens, etc
envFromSecret: ""

## Sensible environment variables that will be rendered as new secret object
## This can be useful for auth tokens, etc
envRenderSecret: {}

## Additional grafana server secret mounts
# Defines additional mounts with secrets. Secrets must be manually created in the namespace.
extraSecretMounts: []
  # - name: secret-files
  #   mountPath: /etc/secrets
  #   secretName: grafana-secret-files
  #   readOnly: true

## Additional grafana server volume mounts
# Defines additional volume mounts.
extraVolumeMounts: []
  # - name: extra-volume
  #   mountPath: /mnt/volume
  #   readOnly: true
  #   existingClaim: volume-claim

## Pass the plugins you want installed as a list.
##
plugins: []
  # - digrich-bubblechart-panel
  # - grafana-clock-panel

## Configure grafana datasources
## ref: http://docs.grafana.org/administration/provisioning/#datasources
##
datasources: {}
#  datasources.yaml:
#    apiVersion: 1
#    datasources:
#    - name: Prometheus
#      type: prometheus
#      url: http://prometheus-prometheus-server
#      access: proxy
#      isDefault: true

## Configure notifiers
## ref: http://docs.grafana.org/administration/provisioning/#alert-notification-channels
##
notifiers: {}
#  notifiers.yaml:
#    notifiers:
#    - name: email-notifier
#      type: email
#      uid: email1
#      # either:
#      org_id: 1
#      # or
#      org_name: Main Org.
#      is_default: true
#      settings:
#        addresses: an_email_address@example.com
#    delete_notifiers:

## Configure grafana dashboard providers
## ref: http://docs.grafana.org/administration/provisioning/#dashboards
##
## `path` must be /var/lib/grafana/dashboards/<provider_name>
##
dashboardProviders: {}
#  dashboardproviders.yaml:
#    apiVersion: 1
#    providers:
#    - name: 'default'
#      orgId: 1
#      folder: ''
#      type: file
#      disableDeletion: false
#      editable: true
#      options:
#        path: /var/lib/grafana/dashboards/default

## Configure grafana dashboard to import
## NOTE: To use dashboards you must also enable/configure dashboardProviders
## ref: https://grafana.com/dashboards
##
## dashboards per provider, use provider name as key.
##
dashboards: {}
  # default:
  #   some-dashboard:
  #     json: |
  #       $RAW_JSON
  #   custom-dashboard:
  #     file: dashboards/custom-dashboard.json
  #   prometheus-stats:
  #     gnetId: 2
  #     revision: 2
  #     datasource: Prometheus
  #   local-dashboard:
  #     url: https://example.com/repository/test.json
  #   local-dashboard-base64:
  #     url: https://example.com/repository/test-b64.json
  #     b64content: true

## Reference to external ConfigMap per provider. Use provider name as key and ConfiMap name as value.
## A provider dashboards must be defined either by external ConfigMaps or in values.yaml, not in both.
## ConfigMap data example:
##
## data:
##   example-dashboard.json: |
##     RAW_JSON
##
dashboardsConfigMaps: {}
#  default: ""

## Grafana's primary configuration
## NOTE: values in map will be converted to ini format
## ref: http://docs.grafana.org/installation/configuration/
##
grafana.ini:
  paths:
    data: /var/lib/grafana/data
    logs: /var/log/grafana
    plugins: /var/lib/grafana/plugins
    provisioning: /etc/grafana/provisioning
  analytics:
    check_for_updates: true
  log:
    mode: console
  grafana_net:
    url: https://grafana.net
## LDAP Authentication can be enabled with the following values on grafana.ini
## NOTE: Grafana will fail to start if the value for ldap.toml is invalid
  # auth.ldap:
  #   enabled: true
  #   allow_sign_up: true
  #   config_file: /etc/grafana/ldap.toml

## Grafana's LDAP configuration
## Templated by the template in _helpers.tpl
## NOTE: To enable the grafana.ini must be configured with auth.ldap.enabled
## ref: http://docs.grafana.org/installation/configuration/#auth-ldap
## ref: http://docs.grafana.org/installation/ldap/#configuration
ldap:
  enabled: false
  # `existingSecret` is a reference to an existing secret containing the ldap configuration
  # for Grafana in a key `ldap-toml`.
  existingSecret: ""
  # `config` is the content of `ldap.toml` that will be stored in the created secret
  config: ""
  # config: |-
  #   verbose_logging = true

  #   [[servers]]
  #   host = "my-ldap-server"
  #   port = 636
  #   use_ssl = true
  #   start_tls = false
  #   ssl_skip_verify = false
  #   bind_dn = "uid=%s,ou=users,dc=myorg,dc=com"

## Grafana's SMTP configuration
## NOTE: To enable, grafana.ini must be configured with smtp.enabled
## ref: http://docs.grafana.org/installation/configuration/#smtp
smtp:
  # `existingSecret` is a reference to an existing secret containing the smtp configuration
  # for Grafana.
  existingSecret: ""
  userKey: "user"
  passwordKey: "password"

## Sidecars that collect the configmaps with specified label and stores the included files them into the respective folders
## Requires at least Grafana 5 to work and can't be used together with parameters dashboardProviders, datasources and dashboards
sidecar:
  image: kiwigrid/k8s-sidecar:0.1.20
  imagePullPolicy: IfNotPresent
  resources: {}
#   limits:
#     cpu: 100m
#     memory: 100Mi
#   requests:
#     cpu: 50m
#     memory: 50Mi
  # skipTlsVerify Set to true to skip tls verification for kube api calls
  # skipTlsVerify: true
  dashboards:
    enabled: false
    SCProvider: true
    # label that the configmaps with dashboards are marked with
    label: grafana_dashboard
    # folder in the pod that should hold the collected dashboards (unless `defaultFolderName` is set)
    folder: /tmp/dashboards
    # The default folder name, it will create a subfolder under the `folder` and put dashboards in there instead
    defaultFolderName: null
    # If specified, the sidecar will search for dashboard config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null
    # provider configuration that lets grafana manage the dashboards
    provider:
      # name of the provider, should be unique
      name: sidecarProvider
      # orgid as configured in grafana
      orgid: 1
      # folder in which the dashboards should be imported in grafana
      folder: ''
      # type of the provider
      type: file
      # disableDelete to activate a import-only behaviour
      disableDelete: false
  datasources:
    enabled: false
    # label that the configmaps with datasources are marked with
    label: grafana_datasource
    # If specified, the sidecar will search for datasource config-maps inside this namespace.
    # Otherwise the namespace in which the sidecar is running will be used.
    # It's also possible to specify ALL to search in all namespaces
    searchNamespace: null

```

运行：

```
[root@k8s-master2 grafana]# helm install prometheus-grafana -f values.yaml --namespace=prometheus stable/grafana
NAME: prometheus-grafana
LAST DEPLOYED: Mon Dec 23 03:21:32 2019
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace prometheus prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 80 on the following DNS name from within your cluster:

   prometheus-grafana.prometheus.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:
export NODE_PORT=$(kubectl get --namespace prometheus -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-grafana)
     export NODE_IP=$(kubectl get nodes --namespace prometheus -o jsonpath="{.items[0].status.addresses[0].address}")
     echo http://$NODE_IP:$NODE_PORT


3. Login with the password from step 1 and the username: admin


检测：
[root@k8s-master1 yml]# kubectl get svc -n prometheus
NAME                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
prometheus                                 NodePort    10.254.54.121   <none>        9090:31044/TCP   4h
prometheus-grafana                         NodePort    10.254.26.231   <none>        80:30400/TCP     44s
prometheus-node-prometheus-node-exporter   ClusterIP   10.254.77.92    <none>        9100/TCP         109m
[root@k8s-master1 yml]# kubectl get pods -n prometheus
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-5c9c5fb6cf-xkps2                      1/1     Running   0          4h
prometheus-grafana-5f7c6685db-45dns              1/1     Running   0          47s
prometheus-node-prometheus-node-exporter-49z5c   1/1     Running   0          109m
prometheus-node-prometheus-node-exporter-4p5vs   1/1     Running   0          109m
prometheus-node-prometheus-node-exporter-9bwg9   1/1     Running   0          109m
prometheus-node-prometheus-node-exporter-rwzzr   1/1     Running   0          109m

```

登录网页 然后输入帐号密码: 略

然后添加数据源：

![image-20191223163035111](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223163035111.png)

选择prometheus

![image-20191223163116281](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223163116281.png)

![image-20191223171854056](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223171854056.png)

寻找官网的模板：

![image-20191223172033724](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223172033724.png)

记住ID

![image-20191223173622425](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223173622425.png)



导入ID号码 11074

![image-20191223173610894](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223173610894.png)

输入成功后等待跳转输入数据源 并导入：

![image-20191223173749971](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223173749971.png)

原本是空的 （所设置的阈值不匹配）：

![image-20191223174829690](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223174829690.png)

配合prometheus去修改阈值：

![image-20191223174913934](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223174913934.png)



![image-20191223175100313](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223175100313.png)

![image-20191223180138097](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223180138097.png)

![image-20191223173909406](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223173909406.png)

理论上prometheus和grafana的走向一样，一般来说 同时是需要是两个的时间选型一样：

prometheus的时间是UTC 所以我们也要把grafana的时间设置成UTC

![image-20191223180455782](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223180455782.png)

![image-20191223180602741](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191223180602741.png)

插件ID 741  并自己修改

**grafana的插件和监控**

上面是我们最常用的 grafana 当中的 dashboard 的功能的使用，然后我们也可以来进行一些其他的系统管理，比如添加用户，为用户添加权限等等，我们也可以安装一些其他插件，比如 grafana 就有一个专门针对 Kubernetes 集群监控的插件：[grafana-kubernetes-app](https://grafana.com/plugins/grafana-kubernetes-app/installation)

```
[root@k8s-master1 ~]# kubectl exec -it prometheus-grafana-5f7c6685db-45dns /bin/bash -n prometheus
bash-5.0$ ls
LICENSE    NOTICE.md  README.md  VERSION    bin        conf       public     scripts    tools
bash-5.0$ grafana-cli plugins install grafana-kubernetes-app
installing grafana-kubernetes-app @ 1.0.1
from: https://grafana.com/api/plugins/grafana-kubernetes-app/versions/1.0.1/download
into: /var/lib/grafana/plugins

✔ Installed grafana-kubernetes-app successfully 

Restart grafana after installing plugins . <service grafana-server restart>

bash-5.0$ 

```

安装完成后需要重启 grafana 才会生效，我们这里直接删除 Pod，重建即可，然后回到 grafana 页面中，切换到 plugins 页面可以发现下面多了一个 Kubernetes 的插件，点击进来启用即可，然后点击`Next up`旁边的链接配置集群

```
[root@k8s-master1 ~]# kubectl delete pod prometheus-grafana-5f7c6685db-45dns -n prometheus
pod "prometheus-grafana-5f7c6685db-45dns" deleted
#等待deployment自动重启
[root@k8s-master1 ~]# kubectl get all -n prometheus
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/prometheus-5c9c5fb6cf-xkps2                      1/1     Running   4          21h
pod/prometheus-grafana-5f7c6685db-4nhnm              1/1     Running   0          53s

```

重启之后：

![image-20191224100210006](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224100210006.png)

![image-20191224100220198](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224100220198.png)

![image-20191224100338502](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224100338502.png)

![image-20191224100352157](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224100352157.png)

![image-20191224102726053](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224102726053.png)

最后点击save 就可以了

 grafana 4 版本以上就支持了报警功能，这使得我们利用 grafana 作为监控面板更为完整，因为报警是监控系统中必不可少的环节，grafana 支持很多种形式的报警功能，比如 email、钉钉、slack、webhook 等等，我们这里来测试下 email 和 钉钉。

**email 报警**

 要启用 email 报警需要在启动配置文件中`/etc/grafana/grafan.ini`开启 SMTP 服务，我们这里同样利用一个 ConfigMap 资源对象挂载到 grafana Pod 中：（grafana-cm.yaml）

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: kube-ops
data:
  grafana.ini: |
    [server]
    root_url = http://<你grafana的url地址>
    [smtp]
    enabled = true
    host = smtp.163.com:25
    user = 17620455319@163.com
    password = <邮箱密码>
    skip_verify = true
    from_address = 17620455319@163.com
    [alerting]
    enabled = true
    execute_alerts = true
```

上面配置了我的 163 邮箱，开启报警功能，当然我们还得将这个 ConfigMap 文件挂载到 Pod 中去：（deployment.yml）记得转化格式

```yaml
  volumeMounts:
  - mountPath: "/etc/grafana"
    name: config
volumes:
- name: config
  configMap:
    name: grafana-config
```

创建 ConfigMap 对象，更新 Deployment：

```shell
$ kubectl create -f grafana-cm.yaml
$ kubectl apply -f grafana-deploy.yaml
```

![image-20191224113939474](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224113939474.png)

![image-20191224113950573](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224113950573.png)

`钉钉报警`+ 添加群机器人 复制webhook 就可以

![grafana graph alert](https://www.qikqiak.com/k8s-book/docs/images/grafana-graph-alert.png)

然后配置相关参数：

- 1、Alert 名称，可以自定义。
- 2、执行的频率，这里我选择每60s检测一次。
- 3、判断标准，默认是 avg，这里是下拉框，自己按需求选择。
- 4、query（A,5m,now），字母A代表选择的metrics 中设置的 sql，也可以选择其它在 metrics中设置的，但这里是单选。`5m`代表从现在起往之前的五分钟，即`5m`之前的那个点为时间的起始点，`now`为时间的结束点，此外这里可以自己手动输入时间。
- 5、设置的预警临界点，这里手动输入，和6是同样功能，6可以手动移动，两种操作是等同的。

然后需要设置报警发送信息，点击侧边的`Notifications`:

![grafana graph notify](https://www.qikqiak.com/k8s-book/docs/images/grafana-graph-notify.png)

其中`Send to`就是前面我们配置过的发送邮件和钉钉的报警频道的名称。

配置完成后需要保存下这个 graph，否则发送报警可能会失败，然后点击 Alert 区域的`Test Rule`可以来测试报警规则，然后邮件和钉钉正常来说就可以收到报警信息了。

邮件报警信息：

![grafana test rule email](https://www.qikqiak.com/k8s-book/docs/images/grafana-email-alert2.png)





### 20.8Alertmanager连⽤（可选）

 Prometheus 包含一个报警模块，就是我们的 AlertManager，Alertmanager 主要用于接收 Prometheus 发送的告警信息，它支持丰富的告警通知渠道，而且很容易做到告警信息进行去重，降噪，分组等，是一款前卫的告警通知系统

![架构](https://www.qikqiak.com/k8s-book/docs/images/prometheus-architecture.png)



从官方文档https://prometheus.io/docs/alerting/configuration/中我们可以看到下载`AlertManager`二进制文件后，可以通过下面的命令运行：

```shell
$ ./alertmanager --config.file=simple.yml
```

其中`-config.file`参数是用来指定对应的配置文件的，由于我们这里同样要运行到 Kubernetes 集群中来，所以我们使用`docker`镜像的方式来安装，使用的镜像是：`prom/alertmanager:latest`

首先，指定配置文件，同样的，我们这里使用一个 ConfigMap 资源对象：(alertmanager-conf.yaml)

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: alert-config
  namespace: prometheus
data:
  config.yml: |-
    global:
      # 在没有报警的情况下声明为已解决的时间
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: '17620455319@163.com'
      smtp_auth_username: '17620455319@163.com'
      smtp_auth_password: 'gomo123456'
      smtp_hello: '163.com'
      smtp_require_tls: false
    # 所有报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 这里的标签列表是接收到报警信息后的重新分组标签，例如，接收到的报警信息里面有许多具有 cluster=A 和 alertname=LatncyHigh 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname', 'cluster']
      # 当一个新的报警分组被创建后，需要等待至少group_wait时间来初始化通知，这种方式可以确保您能有足够的时间为同一分组来获取多个警报，然后一起触发这个报警信息。
      group_wait: 30s

      # 当第一个报警发送后，等待'group_interval'时间来发送新的一组报警信息。
      group_interval: 5m

      # 如果一个报警信息已经发送成功了，等待'repeat_interval'时间来重新发送他们
      repeat_interval: 5m

      # 默认的receiver：如果一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default

      # 上面所有的属性都由所有子路由继承，并且可以在每个子路由上进行覆盖。
      routes:
      - receiver: email
        group_wait: 10s
        match:
          team: node
    receivers:
    - name: 'default'
      email_configs:
      - to: 'wangjiadesx@gomo.com'
        send_resolved: true
    - name: 'email'
      email_configs:
      - to: 'wangjiadesx@gomo.com'
        send_resolved: true
```

这是 AlertManager 的配置文件，我们先直接创建这个 ConfigMap 资源对象

```shell
$ kubectl create -f alertmanager-conf.yaml
configmap "alert-config" created
```

然后修改原来的prometheus的deployment和cm

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    app: prometheus
spec:
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - image: prom/prometheus:latest
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=24h"
        - "--web.enable-admin-api"  # 控制对admin HTTP API的访问，其中包括删除时间序列等功能
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        ports:
        - containerPort: 9090
          protocol: TCP
          name: http
        volumeMounts:
        - mountPath: "/prometheus"
          subPath: prometheus
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      - name: alertmanager
        image: prom/alertmanager:latest
        imagePullPolicy: IfNotPresent
        args:
        - "--config.file=/etc/alertmanager/config.yml"
        - "--storage.path=/alertmanager/data"
        ports:
        - containerPort: 9093
          name: http
        volumeMounts:
        - mountPath: "/etc/alertmanager"
          name: alertcfg
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi
      securityContext:
        runAsUser: 0
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap: 
          name: prometheus-config
        name: config-volume
      - name: alertcfg
        configMap:
          name: alert-config
```

转换后：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: prometheus
  name: prometheus
  namespace: prometheus
spec:
  progressDeadlineSeconds: 2147483647
  replicas: 1
  revisionHistoryLimit: 2147483647
  selector:
    matchLabels:
      app: prometheus
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: prometheus
    spec:
      containers:
      - args:
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        - --storage.tsdb.retention=24h
        - --web.enable-admin-api
        - --web.enable-lifecycle
        command:
        - /bin/prometheus
        image: prom/prometheus:latest
        imagePullPolicy: Always
        name: prometheus
        ports:
        - containerPort: 9090
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 512Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /prometheus
          name: data
          subPath: prometheus
        - mountPath: /etc/prometheus
          name: config-volume
      - args:
        - --config.file=/etc/alertmanager/config.yml
        - --storage.path=/alertmanager/data
        image: prom/alertmanager:latest
        imagePullPolicy: IfNotPresent
        name: alertmanager
        ports:
        - containerPort: 9093
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 256Mi
          requests:
            cpu: 100m
            memory: 256Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/alertmanager
          name: alertcfg
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        runAsUser: 0
      serviceAccount: prometheus
      serviceAccountName: prometheus
      terminationGracePeriodSeconds: 30
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          defaultMode: 420
          name: prometheus-config
        name: config-volume
      - configMap:
          defaultMode: 420
          name: alert-config
        name: alertcfg
```

cm：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    alerting:
      alertmanagers:
        - static_configs:
          - targets: ["localhost:9093"]
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__
        action: replace
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-kubelet'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    - job_name: 'kubernetes-cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https        
        
        
 #修改配置完成后  apply 同时使用prometheus的热重启
[root@k8s-master3 prometheus]# kubectl apply -f prometheus-deployment.yaml 
[root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"


#检测
[root@k8s-master1 dashdoard]# kubectl get pods  -n prometheus
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-dfb4b9965-wzsrm                       2/2     Running   0          16m
.....omit.....
```

报警规则：修改prometheus-cm.yaml

```
    global:
      scrape_interval: 15s
      scrape_timeout: 15s
    alerting:
      alertmanagers:
        - static_configs:
          - targets: ["localhost:9093"]
    rule_files:
      - /etc/prometheus/rules.yml
```

其中`rule_files`就是用来指定报警规则的，这里我们同样将`rules.yml`文件用 ConfigMap 的形式挂载到`/etc/prometheus`目录下面即可:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prometheus
data:
  rules.yml: |
    groups:
    - name: test-rule
      rules:
      - alert: NodeMemoeryUsage
        expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100 > 20
        for: 10s
        labels:
          team: node
        annotations:
          summary: "{{$labels.instance}}: High Memory usage detected"
          description: "{{$labels.instance}}: Memory usage is above 20% (current value is: {{ $value }}"
  prometheus.yml: |
  ........omit........
  
[root@k8s-master3 prometheus]# kubectl apply -f prometheus-cm.yaml 
configmap/prometheus-config configured
[root@k8s-worker4 k8sv]# curl -X POST "http://10.254.54.121:9090/-/reload"
```

上面我们定义了一个名为`NodeMemoryUsage`的报警规则，其中：

- `for`语句会使 Prometheus 服务等待指定的时间, 然后执行查询表达式。
- `labels`语句允许指定额外的标签列表，把它们附加在告警上。
- `annotations`语句指定了另一组标签，它们不被当做告警实例的身份标识，它们经常用于存储一些额外的信息，用于报警信息的展示之类的。

为了方便演示，我们将的表达式判断报警临界值设置为20，重新更新 ConfigMap 资源对象，由于我们在 Prometheus 的 Pod 中已经通过 Volume 的形式将 prometheus-config 这个一个 ConfigMap 对象挂载到了`/etc/prometheus`目录下面，所以更新后，该目录下面也会出现`rules.yml`文件，所以前面配置的`rule_files`路径也是正常的，更新完成后，重新执行`reload`操作，这个时候我们去 Prometheus 的 Dashboard 中切换到`alerts`路径下面就可以看到有报警配置规则的数据了：

![image-20191224145305229](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224145305229.png)

我们可以看到页面中出现了我们刚刚定义的报警规则信息，而且报警信息中还有状态显示。一个报警信息在生命周期内有下面3种状态：

- inactive: 表示当前报警信息既不是firing状态也不是pending状态
- pending: 表示在设置的阈值时间范围内被激活了
- firing: 表示超过设置的阈值时间被激活了

我们这里的状态现在是`firing`就表示这个报警已经被激活了，我们这里的报警信息有一个`team=node`这样的标签，而最上面我们配置 alertmanager 的时候就有如下的路由配置信息了：

```yaml
routes:
- receiver: email
  group_wait: 10s
  match:
    team: node
```

同时 修改prometheus的svc 去修改端口：

```
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: NodePort
  ports:
    - name: web
      port: 9090
      targetPort: http
    - name: alertweb
      port: 9093
      targetPort: 9093
      
[root@k8s-master3 prometheus]# kubectl apply -f prometheus-svc.yaml 
service/prometheus unchanged      
     
[root@k8s-master1 dashdoard]# kubectl get svc  -n prometheus
NAME                                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
prometheus                                 NodePort    10.254.54.121   <none>        9090:31044/TCP,9093:32254/TCP   26h
prometheus-grafana                         NodePort    10.254.26.231   <none>        80:30400/TCP                    22h
prometheus-node-prometheus-node-exporter   ClusterIP   10.254.77.92    <none>        9100/TCP                        24h

```

既可以收到报警了！

我们同样可以通过 NodePort 的形式去访问到 AlertManager 的 Dashboard 页面：

![image-20191224151927895](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191224151927895.png)

在这个页面中我们可以进行一些操作，比如过滤、分组等等，里面还有两个新的概念：Inhibition(抑制)和 Silences(静默)。

- Inhibition：如果某些其他警报已经触发了，则对于某些警报，Inhibition 是一个抑制通知的概念。例如：一个警报已经触发，它正在通知整个集群是不可达的时，Alertmanager 则可以配置成关心这个集群的其他警报无效。这可以防止与实际问题无关的数百或数千个触发警报的通知，Inhibition 需要通过上面的配置文件进行配置。
- Silences：静默是一个非常简单的方法，可以在给定时间内简单地忽略所有警报。Silences 基于 matchers配置，类似路由树。来到的警告将会被检查，判断它们是否和活跃的 Silences 相等或者正则表达式匹配。如果匹配成功，则不会将这些警报发送给接收者。

**webhook接收器**

上面我们配置的是 AlertManager 自带的邮件报警模板，我们也说了 AlertManager 支持很多中报警接收器，比如 slack、微信之类的，其中最为灵活的方式当然是使用 webhook 了，我们可以定义一个 webhook 来接收报警信息，然后在 webhook 里面去进行处理，需要发送怎样的报警信息我们自定义就可以。比如我们这里用 Flask 编写了一个简单的处理钉钉报警的 webhook 的程序：

大家可以根据自己的需求来定制报警数据，上述代码仓库地址：[github.com/cnych/alertmanager-dingtalk-hook](https://github.com/cnych/alertmanager-dingtalk-hook)和步骤







### 20.9Prometheus Operator

（在创建Prometheus Operator时候 如果之前后prometheus 请删除   因为2个会冲突 ）

`Operator`是由[CoreOS](https://coreos.com/)公司开发的，用来扩展 Kubernetes API，特定的应用程序控制器，它用来创建、配置和管理复杂的有状态应用，如数据库、缓存和监控系统。`Operator`基于 Kubernetes 的资源和控制器概念之上构建，但同时又包含了应用程序特定的一些专业知识，比如创建一个数据库的`Operator`，则必须对创建的数据库的各种运维方式非常了解，创建`Operator`的关键是`CRD`（自定义资源）的设计。

> `CRD`是对 Kubernetes API 的扩展，Kubernetes 中的每个资源都是一个 API 对象的集合，例如我们在YAML文件里定义的那些`spec`都是对 Kubernetes 中的资源对象的定义，所有的自定义资源可以跟 Kubernetes 中内建的资源一样使用 kubectl 操作。

`Operator`是将运维人员对软件操作的知识给代码化，同时利用 Kubernetes 强大的抽象来管理大规模的软件应用。目前`CoreOS`官方提供了几种`Operator`的实现，其中就包括我们今天的主角：`Prometheus Operator`，`Operator`的核心实现就是基于 Kubernetes 的以下两个概念：

- 资源：对象的状态定义
- 控制器：观测、分析和行动，以调节资源的分布

当然我们如果有对应的需求也完全可以自己去实现一个`Operator`，接下来我们就来给大家详细介绍下`Prometheus-Operator`的使用方法。

![promtheus opeator](https://www.qikqiak.com/k8s-book/docs/images/prometheus-operator.png)

可以使用helm 一键安装（helm xxx stable/promethus-operator）

```
[root@adm-master1 manifests]# git clone https://github.com/coreos/kube-prometheus.git
[root@adm-master1 manifests]# pwd
/home/kube-prometheus/manifests
```

如果有失败的 就重建一次 apply

进⼊到 manifests ⽬录下⾯，这个⽬录下⾯包含我们所有的资源清单⽂件，我们需要对其中的⽂件 prometheus-serviceMonitorKubelet.yaml 进⾏简单的修改，因为默认情况下，这个 ServiceMonitor 是 关联的 kubelet 的10250端⼝去采集的节点数据，⽽我们前⾯说过为了安全，这个 metrics 数据已经迁 移到10255这个只读端⼝上⾯去了，我们只需要将⽂件中的 https-metrics 更改成 http-metrics 即 可，这个在 Prometheus-Operator 对节点端点同步的代码中有相关定义，（请自己查看 自己所对应的端口 ss -antlp |grep 10250 10255 来查看 开在了哪一个上面 如果是10250 就不用修改了）

```
Subsets: []v1.EndpointSubset{
    {
        Ports: []v1.EndpointPort{
            {
                Name: "https-metrics",
                Port: 10250,
            },
            {
                Name: "http-metrics",
                Port: 10255,
            },
            {
                Name: "cadvisor",
                Port: 4194,
            },
        },
    },
},
```

```
#先create setup里面的内容！！！！！！！！！！！！！！！
[root@adm-master1 manifests]# cd setup/
[root@adm-master1 setup]# ls
0namespace-namespace.yaml                                         prometheus-operator-clusterRoleBinding.yaml
prometheus-operator-0alertmanagerCustomResourceDefinition.yaml    prometheus-operator-clusterRole.yaml
prometheus-operator-0podmonitorCustomResourceDefinition.yaml      prometheus-operator-deployment.yaml
prometheus-operator-0prometheusCustomResourceDefinition.yaml      prometheus-operator-serviceAccount.yaml
prometheus-operator-0prometheusruleCustomResourceDefinition.yaml  prometheus-operator-service.yaml
prometheus-operator-0servicemonitorCustomResourceDefinition.yaml
[root@adm-master1 setup]# kubectl create -f .

#然后在创建外面的全部
[root@adm-master1 manifests]# kubectl create -f .
```

部署完成后，会创建⼀个名为 monitoring 的 namespace，所以资源对象对将部署在改命名空间下 ⾯，此外 Operator 会⾃动创建4个 CRD 资源对象：

```
[root@adm-master1 manifests]# kubectl get crd|grep coreos
alertmanagers.monitoring.coreos.com       2020-01-06T01:47:49Z
podmonitors.monitoring.coreos.com         2020-01-06T01:47:49Z
prometheuses.monitoring.coreos.com        2020-01-06T01:47:50Z
prometheusrules.monitoring.coreos.com     2020-01-06T01:47:50Z
servicemonitors.monitoring.coreos.com     2020-01-06T01:47:50Z
```

可以在 monitoring 命名空间下⾯查看所有的 Pod，其中 alertmanager 和 prometheus 是⽤ StatefulSet 控制器管理的，其中还有⼀个⽐较核⼼的 prometheus-operator 的 Pod，⽤来控制其他资源对象和监 听对象变化的：

```
[root@adm-master1 manifests]# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-main-0                   2/2     Running   0          19m
alertmanager-main-1                   2/2     Running   0          19m
alertmanager-main-2                   2/2     Running   0          19m
grafana-58dc7468d7-pfvgs              1/1     Running   0          19m
kube-state-metrics-769f4fd4d5-n46rf   3/3     Running   0          19m
node-exporter-96dzp                   2/2     Running   0          19m
node-exporter-dh596                   2/2     Running   0          19m
node-exporter-gsgdg                   2/2     Running   0          19m
node-exporter-jh2g8                   2/2     Running   0          19m
node-exporter-jsqnk                   2/2     Running   0          19m
node-exporter-rbpp5                   2/2     Running   0          19m
node-exporter-rzbdd                   2/2     Running   0          19m
prometheus-adapter-5cd5798d96-q67lw   1/1     Running   0          19m
prometheus-k8s-0                      3/3     Running   1          19m
prometheus-k8s-1                      3/3     Running   1          19m
prometheus-operator-99dccdc56-flq4k   1/1     Running   0          32m


[root@adm-master1 manifests]# kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.10.117.130   <none>        9093/TCP                     23m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   23m
grafana                 ClusterIP   10.10.186.121   <none>        3000/TCP                     23m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP            23m
node-exporter           ClusterIP   None            <none>        9100/TCP                     23m
prometheus-adapter      ClusterIP   10.10.73.166    <none>        443/TCP                      23m
prometheus-k8s          ClusterIP   10.10.168.233   <none>        9090/TCP                     23m
prometheus-operated     ClusterIP   None            <none>        9090/TCP                     23m
prometheus-operator     ClusterIP   None            <none>        8080/TCP                     36m

```

可以看到上⾯针对 grafana 和 prometheus 都创建了⼀个类型为 ClusterIP 的 Service，当然如果我们 想要在外⽹访问这两个服务的话可以通过创建对应的 Ingress 对象或者使⽤ NodePort 类型的 Service，我们这⾥为了简单，直接使⽤ NodePort 类型的服务即可，编辑 grafana 和 prometheus-k8s 这两个 Service，将服务类型更改为 NodePort:

```
[root@adm-master1 manifests]# kubectl edit svc grafana -n monitoring
service/grafana edited
[root@adm-master1 manifests]# kubectl edit svc prometheus-k8s -n monitoring
service/prometheus-k8s edited

[root@adm-master1 manifests]# kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.10.117.130   <none>        9093/TCP                     30m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   30m
grafana                 NodePort    10.10.186.121   <none>        3000:32192/TCP               30m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP            30m
node-exporter           ClusterIP   None            <none>        9100/TCP                     30m
prometheus-adapter      ClusterIP   10.10.73.166    <none>        443/TCP                      30m
prometheus-k8s          NodePort    10.10.168.233   <none>        9090:31338/TCP               30m
prometheus-operated     ClusterIP   None            <none>        9090/TCP                     30m
prometheus-operator     ClusterIP   None            <none>        8080/TCP                     43m

```

更改完成后，我们就可以通过去访问上⾯的两个服务了，⽐如查看 prometheus 的 targets ⻚⾯：（全部都是up）

![image-20200106104709605](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106104709605.png)

我们可以看到⼤部分的配置都是正常的，只有两三个没有管理到对应的监控⽬标，⽐如 kubecontroller-manager 和 kube-scheduler 这两个系统组件，这就和 ServiceMonitor 的定义有关系了，我 们先来查看下 kube-scheduler 组件对应的 ServiceMonitor 资源的定义：(prometheus-serviceMonitorKubeScheduler.yaml)

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s # 每30s获取⼀次信息
    port: http-metrics # 对应service的端⼝名
  jobLabel: k8s-app
  namespaceSelector:   # 表示去匹配某⼀命名空间中的service，如果想从所有的namespace中匹配⽤any: t
rue
    matchNames:
    - kube-system
  selector:  # 匹配的 Service 的labels，如果使⽤mathLabels，则下⾯的所有标签都匹配时才会匹配该s
ervice，如果使⽤matchExpressions，则⾄少匹配⼀个标签的service都会被选择
    matchLabels:
      k8s-app: kube-scheduler

```

上⾯是⼀个典型的 ServiceMonitor 资源⽂件的声明⽅式，上⾯我们通过 selector.matchLabels 在 kube-system 这个命名空间下⾯匹配具有 k8s-app=kube-scheduler 这样的 Service，但是我们系统中 根本就没有对应的 Service，所以我们需要⼿动创建⼀个 Service：（prometheuskubeSchedulerService.yaml）

```
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  selector:
    component: kube-scheduler
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
    protocol: TCP
    
    
[root@adm-master1 manifests]# kubectl create -f  prometheus-kubeSchedulerService.yaml
service/kube-scheduler created
```

用同样的方式来创建kube-controller-manager的service

```
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  selector:
    component: kube-controller-manager
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP
```

解释：

标签选择器：

```
[root@adm-master1 manifests]# kubectl describe pods kube-controller-manager-adm-master1 -n kube-system
```

来查看 能看出 component： kube-controller-mangger   scheduler同理

端口：

```
[root@adm-master1 manifests]# ss -antlp|grep kube-scheduler
LISTEN     0      128    127.0.0.1:10259                    *:*                   users:(("kube-scheduler",pid=10841,fd=6))
LISTEN     0      128         :::10251                   :::*                   users:(("kube-scheduler",pid=10841,fd=5))
[root@adm-master1 manifests]# ss -antlp|grep kube-controller
LISTEN     0      128    127.0.0.1:10257                    *:*                   users:(("kube-controller",pid=11126,fd=6))
LISTEN     0      128         :::10252                   :::*                   users:(("kube-controller",pid=11126,fd=5))
```

创建完成后，隔⼀⼩会⼉后去 prometheus 查看 targets 下⾯ kube-scheduler和 kube-controller的状态：

![image-20200106112143747](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112143747.png)

**PS： 老版本1.13 一下的话 可能会出现报错：** 我的不会报错的原因是1.17+的版本会自动创建2个端口 如上的代码所示

![image-20200106112229353](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112229353.png)

我们可以看到现在已经发现了 target，但是抓取数据结果出错了，这个错误是因为我们集群是使⽤ kubeadm 搭建的，其中 kube-scheduler 默认是绑定在 127.0.0.1 上⾯的，⽽上⾯我们这个地⽅是想 通过节点的 IP 去访问，所以访问被拒绝了，我们只要把 kube-scheduler 绑定的地址更改 成 0.0.0.0 即可满⾜要求，由于 kube-scheduler 是以静态 Pod 的形式运⾏在集群中的，所以我们只 需要更改静态 Pod ⽬录下⾯对应的 YAML ⽂件即可：

```
$ ls /etc/kubernetes/manifestsetcd.yaml kube-apiserver.yaml kube-controller-manager.yaml kube-scheduler.yaml
```

将 kube-scheduler.yaml ⽂件中 -command 的 --address 地址更改成 0.0.0.0 

```
containers:
- command:
- kube-scheduler
- --leader-elect=true
- --kubeconfig=/etc/kubernetes/scheduler.conf
- --address=0.0.0.0
```

修改完成后我们将该⽂件从当前⽂件夹中移除，隔⼀会⼉再移回该⽬录，就可以⾃动更新了，然后再 去看 prometheus 中 kube-scheduler 这个 target 是否已经正常了：

![image-20200106112318589](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112318589.png)

登录grafana 默认帐号/密码改成了admin/admin  第一次进入修改先的密码（gomo123）

![image-20200106112627614](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112627614.png)

然后点击home  可以看到自动加载的选项：

![image-20200106112725769](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112725769.png)

随便点开一个 如果没有数据 需要修改时间类型UTC 

![image-20200106112949774](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106112949774.png)

![image-20200106113043917](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106113043917.png)

#### 20.9.1⾃定义 Prometheus Operator 监控项（ETCD)

除了 Kubernetes 集群中的⼀些资源对象、节点以及组件需要监控，有的时候我们可能还需要根据实际 的业务需求去添加⾃定义的监控项，添加⼀个⾃定义监控的步骤也是⾮常简单的。

- 第⼀步建⽴⼀个 ServiceMonitor 对象，⽤于 Prometheus 添加监控项 
- 第⼆步为 ServiceMonitor 对象关联 metrics 数据接⼝的⼀个 Service 对象
- 第三步确保 Service 对象可以正确获取到 metrics 数据

对于 etcd 集群⼀般情况下，为了安全都会开启 https 证书认证的⽅式，所以要想让 Prometheus 访问 到 etcd 集群的监控数据，就需要提供相应的证书校验。

⽆论是 Kubernetes 集群外的还是使⽤ Kubeadm 安装在集群内部的 etcd 集群，我们这⾥都将其视作 集群外的独⽴集群，因为对于⼆者的使⽤⽅法没什么特殊之处。

```
[root@adm-master1 manifests]# kubectl get pod etcd-adm-master1 -n kube-system -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/config.hash: 2f6c1f15e2f7a1dd4bf654c394b4656f
    kubernetes.io/config.mirror: 2f6c1f15e2f7a1dd4bf654c394b4656f
    kubernetes.io/config.seen: "2019-12-30T22:33:48.092860245-05:00"
    kubernetes.io/config.source: file
  creationTimestamp: "2019-12-31T03:33:52Z"
  labels:
    component: etcd
    tier: control-plane
  name: etcd-adm-master1
  namespace: kube-system
  ownerReferences:
  - apiVersion: v1
    controller: true
    kind: Node
    name: adm-master1
    uid: c1e36163-9c7c-4fcc-a968-ae8e1d913b3a
  resourceVersion: "1603031"
  selfLink: /api/v1/namespaces/kube-system/pods/etcd-adm-master1
  uid: 7a85ab74-d263-4c3d-976d-c039d7454da3
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.208.41:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://192.168.208.41:2380
    - --initial-cluster=adm-master1=https://192.168.208.41:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.208.41:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.208.41:2380
    - --name=adm-master1
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: registry.aliyuncs.com/google_containers/etcd:3.4.3-0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /health
        port: 2381
        scheme: HTTP
      initialDelaySeconds: 15
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 15
    name: etcd
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  hostNetwork: true
  nodeName: adm-master1
  priority: 2000000000
  priorityClassName: system-cluster-critical
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    operator: Exists
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd
      type: DirectoryOrCreate
    name: etcd-data
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-01-02T05:27:26Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T01:58:12Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-01-06T01:58:12Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-01-02T05:27:26Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://19ce520f823999c8a52351244e962423fa2f1f2e48221a7dae0b26646d8b9e91
    image: registry.aliyuncs.com/google_containers/etcd:3.4.3-0
    imageID: docker-pullable://registry.aliyuncs.com/google_containers/etcd@sha256:4198ba6f82f642dfd18ecf840ee37afb9df4b596f06eef20e44d0aec4ea27216
    lastState:
      terminated:
        containerID: docker://987883db1e5df680d18f28d923c2b4692a4c43f7f54efba9f802820989dad72b
        exitCode: 0
        finishedAt: "2020-01-06T01:57:20Z"
        reason: Completed
        startedAt: "2019-12-31T03:33:36Z"
    name: etcd
    ready: true
    restartCount: 1
    started: true
    state:
      running:
        startedAt: "2020-01-06T01:58:10Z"
  hostIP: 192.168.208.41
  phase: Running
  podIP: 192.168.208.41
  podIPs:
  - ip: 192.168.208.41
  qosClass: BestEffort
  startTime: "2020-01-02T05:27:26Z"
```

我们可以看到 etcd 使⽤的证书都对应在节点的 /etc/kubernetes/pki/etcd 这个路径下⾯，所以⾸先我们 将需要使⽤到的证书通过 secret 对象保存到集群中去：(在 etcd 运⾏的节点)

```
[root@adm-master1 etcd]# pwd
/etc/kubernetes/pki/etcd
[root@adm-master1 etcd]# ls
ca.crt  ca.key  healthcheck-client.crt  healthcheck-client.key  peer.crt  peer.key  server.crt  server.key
[root@adm-master1 etcd]# kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key --from-file=/etc/kubernetes/pki/etcd/ca.crt
secret/etcd-certs created

[root@adm-master1 etcd]# kubectl get secret etcd-certs -n monitoring
NAME         TYPE     DATA   AGE
etcd-certs   Opaque   3      52s
[root@adm-master1 etcd]# kubectl get secret etcd-certs -n monitoring -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN3akNDQWFxZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkbGRHTmsKTFdOaE1CNFhEVEU1TVRJek1UQXpNek15TVZvWERUSTVNVEl5T0RBek16TXlNVm93RWpFUU1BNEdBMVVFQXhNSApaWFJqWkMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU53a1NXWHpocngrCnZvY2dOdzdrS3A0ZGNQN09hU3NnRWg4Z0Vod3pGR0I3RkhJRWJKYU8ySm1ZYXVPdXJ6SDlSdUExVWdGTnEvWTYKTDk0MFdtRndWWnlDTnRmSVBVUnVENGFHcVBFY3BHOTZqUWw0bzkyMVlnOTBKU0JVbW1vKzhFSk1oOEVuK0d6egp5aHhYdWhpcXI2NnJVUjRTYXh5bGF4RDU4K29mdDB2UXJRbVlJN0ZneWs4MUVOMk1CelBFdGFzTFVjcXg1eGJrCmZIaXJ5SFpWOExtc09yeXFjdmFsL2puMVBDUXZndjBlVFFUYzJ2VE9pQzRBZDBhdWI2eWpkblRUN2p2bnpwOC8KOXNwK0tMeE5SL3VZVGpLUkkrVWZvL3dhYXJ4ZVpLTFBUcVloLytPUG8xOGtzVDdOWFJ1Q0czNlpTQk5hcjZ2YgpaMkxyaWJ3am1vY0NBd0VBQWFNak1DRXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBQ09zUGZkZVNuYWdKQ1BVanBJdCs0OVlMaHErN1NyNHpEZzgKVXRjLzNub0FkYnlrUlBnUGIySzA5NERjcE1NNWFVblhqejBVM2FxVmpVb3U5cXI2U1JKcHN1NC9rZ1BkTlRHTgp3NW1oSFZlcWhZMG9YNVZnSEJ3Wmt4SnRGSzI1QXN1QVduc3g0VDlnU2I2WVBDTndFa3Y0NU5Rczk5NkVXSVQ3CldyQ1RzS2hHdFBWM25aanFFZFJvNHNrcE1YUVQ0bitKMUVqbG1vaHg2aWxyRHRJUkY4ajhoQXdGUURkUUhNT2YKcXIwUkhyVVNzdWJja0dlQkQ5cXQyRUhHWjk0dnhzL0M3RWZyK1BKY25VK0hCclFaV043S0hMa2xEMTRGckM5QQpCSWJIVEVEM2Rla1RQemozTDB3UHJ1WmoyK2lHMTJmclVzWXRzMHZpTlcxWERIZHVJK0k9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  healthcheck-client.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMrekNDQWVPZ0F3SUJBZ0lJWFVSS2xocnd5cHd3RFFZSktvWklodmNOQVFFTEJRQXdFakVRTUE0R0ExVUUKQXhNSFpYUmpaQzFqWVRBZUZ3MHhPVEV5TXpFd016TXpNakZhRncweU1ERXlNekF3TXpNek1qSmFNRUF4RnpBVgpCZ05WQkFvVERuTjVjM1JsYlRwdFlYTjBaWEp6TVNVd0l3WURWUVFERXh4cmRXSmxMV1YwWTJRdGFHVmhiSFJvClkyaGxZMnN0WTJ4cFpXNTBNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXF1eDAKY0NpRjR5R1RZYkplQ1VrNDdZWGk2bVdUeGpLQlRQdHFsTzd0RzFOTkRVS0h4a0hHbDFVS3hBUFpNL2dhSmtDUQoyVFdwMzFvTTYxRnhTZTZWQ1c1aloycEVFVTljek1pOFUyZEVYV1FGdDloMWdFbmFEYyt0ODZXMVFoZTJiUUo2CjFwOW5IczF0aHl6aXAyejVFdndxWXAvV3JaN0lnZjZFaFBDdkpDUW1vMkQ2YzNlR1BIbmdhczZpZ0RzUVEwNkoKemh3YXdYV2VpNENRa0NtNmxrRkRpSmo5czZRM0NkQWt4MlRsaWxKN3JoUHNzaCt2Y1AvaDkwMFNFNFVXYVczcAppK3VLbWQwczhQc2hZeTZjWEM4ampNUWh6WmppbVAzSDM0QUFXaHFjVG9UcFR5MDEyZHpBNFlmREQ1MzZaOTBjCnFzVzNnazJrWmhXaXQ0alZnd0lEQVFBQm95Y3dKVEFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3cKQ2dZSUt3WUJCUVVIQXdJd0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFISGFQWWtVSW1qVmFtUHNneWVKZDVXZwpFQWRSM2plUkowWjJhTVM2OVgwTWV1OTZqLzREZzlwZjJPRDc2aDVQYlAyMW1EYnEzZTV3aytyK21rZE9aOTc3CnJuU2JrNHJqNkltOHJaV0oxUDhjNHJHb3dKd3VQZUJJaUxpYXIrd09lVE9GdkVxN3lrYzFGTlFKZ3BCU2JKQ3cKMUhsbzVJUXZNK3R4MERNN0QwYitiNndKMDkwaXdFMlkyUmQwaXJWbTdXY2dwTVBKOVE2SjR6bjdEZHpTbzdJTQpGNDZTV0dZdFhxM05PM2FSTjNpNzFHLzU1Q2pXa1BCNWFkZi9SaFp5QjYyWmcyZHhBVzUzVHlFMWlMS21tbUNpCkNXYVI4SmFWTTdMOHhTLzdncVluV1diZXZZS0pPZytZdFlId2Zqa2ZuOWF0ZDFnaitIZlhMcGFKQ2FUTWVNND0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  healthcheck-client.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBcXV4MGNDaUY0eUdUWWJKZUNVazQ3WVhpNm1XVHhqS0JUUHRxbE83dEcxTk5EVUtICnhrSEdsMVVLeEFQWk0vZ2FKa0NRMlRXcDMxb002MUZ4U2U2VkNXNWpaMnBFRVU5Y3pNaThVMmRFWFdRRnQ5aDEKZ0VuYURjK3Q4NlcxUWhlMmJRSjYxcDluSHMxdGh5emlwMno1RXZ3cVlwL1dyWjdJZ2Y2RWhQQ3ZKQ1FtbzJENgpjM2VHUEhuZ2FzNmlnRHNRUTA2Snpod2F3WFdlaTRDUWtDbTZsa0ZEaUpqOXM2UTNDZEFreDJUbGlsSjdyaFBzCnNoK3ZjUC9oOTAwU0U0VVdhVzNwaSt1S21kMHM4UHNoWXk2Y1hDOGpqTVFoelpqaW1QM0gzNEFBV2hxY1RvVHAKVHkwMTJkekE0WWZERDUzNlo5MGNxc1czZ2sya1poV2l0NGpWZ3dJREFRQUJBb0lCQVFDVHFPeTZqRGVHVGRaTwpHMUtqd1E4ZUc0RTZNQUNtdzdEeWVXek5OMCs5UUl5YlBQT2c4ZWdIaXA5ZlVWZk9Uck1BZ3R6ZjJUMWt5QjNMCkdUTyt4QThhODdPS2ZzSkpGZis4cGxvVHoyMi9KSTdRRVg4SkVrUC9sSC9ac2psUjNMeHJsaTNheGlERytuOTUKdk93ZDZjV1BnaXQzd2xBcTg3YVNudmVMQllhNHQ5ekJqaWQ5Ukd5V1FudEQ5MlYvdWtFc08zbDJsZ25qNldsWgpUUm1ISkUwRlcweHZLQU1nNXJ4ZE1GNklLSVBqREZlcHFtSDJ5SC95MDZYWU5nbzQwclozTWJpYi9adFJQNFRpClpBRnVNRlpOOFlZVGRjc0ducGd2SzZ1b1A5N2YrYWFrVjQ3d3ZMN1d6VmthaUVTdDhNKzg3WmVmK1FhTjdPZXgKaXFtcDZFcGhBb0dCQU5rTURHVjVpK3dLNWdLcnpBbWxGNVpJLy9OSE4rZE1MVDRvWlQ1UVZ4QlRQTHdjRThJTgpNUnAzNzhtTHA1dW9iS0ZOd2xXTk91byt3QTI3TjFXS1ZyZ3FpMis2UlViWkpxZUZjVmdKS29zNU4xSGcrYkdtCkdCMDNQNmo1dXFsMCt5ZG9veGJyZFE2Z0tZLzlQc284OFN4ZkZ5WXVwUjRuUWxGZThYTHI2ODd6QW9HQkFNbVoKVXloTlRjRHA1RVdJZkU3RTFIOTZkY1lOcEdUNFVVOWVmUFl5Z01ZQ0NCdEdJNVZCWFhxZzhyaExkTkRuZU5NagpHYldoOFBHVlhXYmdMTGFjK1BaTWZCM1ZKZkhLUzRWc3pOMk1mTFhXNlNzK2hhcXZSNVpNb2NWZXVkOC8xNUxrCjlYUVJRWEhkbTRRcGlVM3lJbGgwbTVEU3JWUEcvdCtnRm5WS09TTXhBb0dCQUtZRDRUZDgwTm1yUEdPdXBGSjgKUko1ZkYrY3RBa1dZcnNKc2c0UTJUMkhkU1FkWk1vT3JNM1BiYVQzdjVEUGJqN3VSanFPQmN4N1pBRzJBVmNMSQpIYXlnWGljSGd4Vzk0eU1mbnFLSDRGSzlZT0x3QWcwdnppSUtzRmEvTFZlUWNzcWg3cDBKWEcvamNlY0EvWllUCkp5V1pWa3VPUWgzZVNZdVQ0M3JUbVhxaEFvR0FOaFdLTjYrMWdtRzlPZUpKNXgvckdtQVNKSllZV25ZNzZoMGgKVFROelZLdkszUFpPS1lhbHUzWmVaNDdtd2Z5M2IzMWxNbE5GdnFvaHFxM05rUmcvdW1QK2tFcFVxYTlwMzF1MwpBbURrUEN4eDFZWXFlZ1lZSUh4aWtmNjl3dVR2d3BybU5zTkNXWGZvZHVabHphRitFVmtIT3kwcUR1VytEdVIxCjRmV05xcUVDZ1lCS3NQaG5UamYyOWhxbHpWS2hTTkVkZkc4NmpYR1dwOHoyejgxczJZWUtiY3lDZWtHNGU5ZUsKRzYvUTRkU29IdmYxRFFVNkE5SDNMcFllUEU3T0pkdHRXMkMwSW83aVMyVG1PajRrd0V5MVVTQnZldDdWQzgyUQpBTWs3aFNYdEtIRUorSkdPQSsvb2JQcHY0NW1mdnFYM1BFSGpNSkptQ0xnc0FIMytLejhGMmc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
kind: Secret
metadata:
  creationTimestamp: "2020-01-06T03:49:52Z"
  name: etcd-certs
  namespace: monitoring
  resourceVersion: "1624616"
  selfLink: /api/v1/namespaces/monitoring/secrets/etcd-certs
  uid: 5fbcf283-621f-4294-93ca-ff23da23ca29
type: Opaque

```



然后将上⾯创建的 etcd-certs 对象配置到 prometheus 资源对象中，直接更新 prometheus 资源对象即 可：

```
 baseImage: quay.io/prometheus/prometheus
  secrets:
  - etcd-certs
  nodeSelector:
  .....omit.....
[root@adm-master1 manifests]# kubectl apply -f prometheus-prometheus.yaml 
prometheus.monitoring.coreos.com/k8s configured
```

验证：

```
[root@adm-master1 etcd]# kubectl exec -it prometheus-k8s-0 /bin/sh -n monitoring
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-k8s-0 -n monitoring' to see all of the containers in this pod.
/prometheus $ ls /etc/prometheus/secrets/etcd-certs/
ca.crt                  healthcheck-client.crt  healthcheck-client.key

```

现在 Prometheus 访问 etcd 集群的证书已经准备好了，接下来创建 ServiceMonitor 对象即可 （prometheus-serviceMonitorEtcd.yaml

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
  - port: port
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-certs/healthcheck-client.crt
      keyFile: /etc/prometheus/secrets/etcd-certs/healthcheck-client.key
      insecureSkipVerify: true
  selector:
    matchLabels:
      k8s-app: etcd
  namespaceSelector:
    matchNames:
    - kube-system

```

上⾯我们在 monitoring 命名空间下⾯创建了名为 etcd-k8s 的 ServiceMonitor 对象，基本属性和前⾯章节中的⼀致，匹配 kube-system 这个命名空间下⾯的具有 k8s-app=etcd 这个 label 标签的 Service，jobLabel 表示⽤于检索 job 任务名称的标签，和前⾯不太⼀样的地⽅是 endpoints 属性的写法，配置上访问 etcd 的相关证书，endpoints 属性下⾯可以配置很多抓取的参数，⽐如 relabel、proxyUrl，tlsConfig 表示⽤于配置抓取监控数据端点的 tls 认证，由于证书 serverName 和 etcd 中签发的可能不匹配，所以加上了 insecureSkipVerify=true

```
[root@adm-master1 manifests]# kubectl create -f prometheus-serviceMonitorEtcd.yaml 
servicemonitor.monitoring.coreos.com/etcd-k8s created


[root@adm-master1 etcd]# kubectl get svc -n kube-system -l k8s-app=etcd
No resources found in kube-system namespace.
#ServiceMonitor 创建完成了，但是现在还没有关联的对应的 Service 对象，所以需要我们去⼿动创建
#⼀个 Service 对象（prometheus-etcdService.yaml）：

apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: port
    port: 2379
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    k8s-app: etcd
subsets:
- addresses:
  - ip: 192.168.208.41
    nodeName: etcd-adm-master1
  - ip: 192.168.208.42
    nodeName: etcd-adm-master2
  - ip: 192.168.208.43
    nodeName: etcd-adm-master3
  ports:
  - name: port
    port: 2379
    protocol: TCP




[root@adm-master1 etcd]# kubectl create -f prometheus-etcdService.yaml 
service/etcd-k8s created
endpoints/etcd-k8s created

```

![image-20200106140248348](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106140248348.png)

连接grafana import模板编号：3070  并修改时间UTC

![image-20200106141315098](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106141315098.png)

#### 20.9.2 配置 PrometheusRule

现在我们知道怎么⾃定义⼀个 ServiceMonitor 对象了，但是如果需要⾃定义⼀个报警规则的话呢？⽐ 如现在我们去查看 Prometheus Dashboard 的 Alert ⻚⾯下⾯就已经有⼀些报警规则了，还有⼀些是已 经触发规则的了：

![image-20200106141722672](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106141722672.png)

之前我们使⽤⾃定义的⽅式可以在 Prometheus 的配置⽂件之中指定 AlertManager 实例和 报警的 rules ⽂件，现在我们通过 Operator 部署的呢？我们可以在 Prometheus Dashboard 的 Config ⻚⾯下⾯查看关于AlertManager 的配置：

![image-20200106141824618](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106141824618.png)

上⾯ alertmanagers 实例的配置我们可以看到是通过⻆⾊为 endpoints 的 kubernetes 的服务发现机制获取的，匹配的是服务名为 alertmanager-main，端⼝名未 web 的 Service 服务，我们查看下alertmanager-main 这个 Service：

```
[root@adm-master1 manifests]# kubectl get svc -n monitoring 
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-main       ClusterIP   10.10.117.130   <none>        9093/TCP                     4h15m


[root@adm-master1 manifests]# kubectl describe svc alertmanager-main -n monitoring
Name:              alertmanager-main
Namespace:         monitoring
Labels:            alertmanager=main
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"alertmanager":"main"},"name":"alertmanager-main","namespace":"...
Selector:          alertmanager=main,app=alertmanager
Type:              ClusterIP
IP:                10.10.117.130
Port:              web  9093/TCP
TargetPort:        web/TCP
Endpoints:         10.20.1.43:9093,10.20.3.79:9093,10.20.4.39:9093
Session Affinity:  ClientIP
Events:            <none>



[root@adm-master1 manifests]# kubectl get pod -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
alertmanager-main-0                   2/2     Running   0          4h19m
alertmanager-main-1                   2/2     Running   0          4h19m
alertmanager-main-2                   2/2     Running   0          4h19m
....omit....

```

可以看到服务名正是 alertmanager-main，Port 定义的名称也是 web，符合上⾯的规则，所以Prometheus 和 AlertManager 组件就正确关联上了。⽽对应的报警规则⽂件位于： /etc/prometheus/rules/prometheus-k8s-rulefiles-0/ ⽬录下⾯所有的 YAML ⽂件。我们可以进⼊ Prometheus 的 Pod 中验证下该⽬录下⾯是否有 YAML ⽂件：

```
$ kubectl exec -it prometheus-k8s-0 /bin/sh -n monitoring
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-k8s-0 -n monitoring' to see all of the containers in
this pod.
/prometheus $ ls /etc/prometheus/rules/prometheus-k8s-rulefiles-0/
monitoring-prometheus-k8s-rules.yaml
/prometheus $ cat /etc/prometheus/rules/prometheus-k8s-rulefiles-0/monitoring-pr
ometheus-k8s-rules.yaml
groups:
- name: k8s.rules
rules:
- expr: |
sum(rate(container_cpu_usage_seconds_total{job="kubelet", image!="", container_name!=
""}[5m])) by (namespace)
record: namespace:container_cpu_usage_seconds_total:sum_rate
.....omit.....
```

这个 YAML ⽂件实际上就是我们之前创建的⼀个 PrometheusRule ⽂件包含的：

```
[root@adm-master1 manifests]# pwd 
/home/kube-prometheus/manifests
[root@adm-master1 manifests]# cat prometheus-rules.yaml 
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: prometheus-k8s-rules
.....
```

我们这⾥的 PrometheusRule 的 name 为 prometheus-k8s-rules，namespace 为 monitoring，我们可 以猜想到我们创建⼀个 PrometheusRule 资源对象后，会⾃动在上⾯的 prometheus-k8s-rulefiles-0 ⽬ 录下⾯⽣成⼀个对应的 <namespace>-<name>.yaml ⽂件，所以如果以后我们需要⾃定义⼀个报警选项的 话，只需要定义⼀个 PrometheusRule 资源对象即可。⾄于为什么 Prometheus 能够识别这个 PrometheusRule 资源对象呢？这就需要查看我们创建的 prometheus 这个资源对象了，⾥⾯有⾮常重 要的⼀个属性 ruleSelector，⽤来匹配 rule 规则的过滤器，要求匹配具有 prometheus=k8s 和 role=alert-rules 标签的 PrometheusRule 资源对象，现在明⽩了吧？

```
[root@adm-master1 manifests]# vim prometheus-prometheus.yaml 
...omit....
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules

```

所以我们要想⾃定义⼀个报警规则，只需要创建⼀个具有 prometheus=k8s 和 role=alert-rules 标签的 PrometheusRule 对象就⾏了，⽐如现在我们添加⼀个 etcd 是否可⽤的报警，我们知道 etcd 整个集群 有⼀半以上的节点可⽤的话集群就是可⽤的，所以我们判断如果不可⽤的 etcd 数量超过了⼀半那么就 触发报警，创建⽂件 prometheus-etcdRules.yaml：

```
[root@adm-master1 manifests]# kubectl create -f prometheus-etcdRules.yaml 
prometheusrule.monitoring.coreos.com/etcd-rules created
[root@adm-master1 manifests]# vim prometheus-etcdRules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: etcd-rules
  namespace: monitoring
spec:
  groups:
  - name: etcd
    rules:
    - alert: EtcdClusterUnavailable
      annotations:
        summary: etcd cluster small!
        description: If one more etcd peer goes down the cluster will be unavailable
      expr: |
        count(up{job="etcd"} == 0) > (count(up{job="etcd"}) / 2 - 1)
      for: 3m
      labels:
        severity: critical
```

注意 label 标签⼀定⾄少要有 prometheus=k8s 和 role=alert-rules，创建完成后，隔⼀会⼉再去容器中 查看下 rules ⽂件夹：

```
[root@adm-master1 manifests]# kubectl exec -it prometheus-k8s-0 /bin/sh -n monitoring
Defaulting container name to prometheus.
Use 'kubectl describe pod/prometheus-k8s-0 -n monitoring' to see all of the containers in this pod.
/prometheus $ ls /etc/prometheus/rules/prometheus-k8s-rulefiles-0/
monitoring-etcd-rules.yaml            monitoring-prometheus-k8s-rules.yaml
/prometheus $ cat /etc/prometheus/rules/prometheus-k8s-rulefiles-0/monitoring-etcd-rules.yaml 
groups:
- name: etcd
  rules:
  - alert: EtcdClusterUnavailable
    annotations:
      description: If one more etcd peer goes down the cluster will be unavailable
      summary: etcd cluster small!
    expr: |
      count(up{job="etcd"} == 0) > (count(up{job="etcd"}) / 2 - 1)
    for: 3m
    labels:
      severity: critical
```

再去浏览器中查看：

![image-20200106143735670](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106143735670.png)

#### 20.9.3 配置报警

```
[root@adm-master1 manifests]# kubectl get svc -n monitoring
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       NodePort    10.10.117.130   <none>        9093:30551/TCP               4h39m
....omit.....

```

![image-20200106144054463](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106144054463.png)

这些配置信息实际上是来⾃于我们之前在 prometheus-operator/contrib/kubeprometheus/manifests ⽬录下⾯创建的 alertmanager-secret.yaml ⽂件：

```
apiVersion: v1
data:
  alertmanager.yaml: Imdsb2JhbCI6IAogICJyZXNvbHZlX3RpbWVvdXQiOiAiNW0iCiJyZWNlaXZlcnMiOiAKL
SAibmFtZSI6ICJudWxsIgoicm91dGUiOiAKICAiZ3JvdXBfYnkiOiAKICAtICJqb2IiCiAgImdyb3VwX2ludGVydmF
sIjogIjVtIgogICJncm91cF93YWl0IjogIjMwcyIKICAicmVjZWl2ZXIiOiAibnVsbCIKICAicmVwZWF0X2ludGVyd
mFsIjogIjEyaCIKICAicm91dGVzIjogCiAgLSAibWF0Y2giOiAKICAgICAgImFsZXJ0bmFtZSI6ICJEZWFkTWFuc1N
3aXRjaCIKICAgICJyZWNlaXZlciI6ICJudWxsIg==
kind: Secret
metadata:
  name: alertmanager-main
  namespace: monitoring
type: Opaque
```

可以将 alertmanager.yaml 对应的 value 值做⼀个 base64 解码：

```
"global":
"resolve_timeout": "5m"
"receivers":
- "name": "null"
"route":
"group_by":
- "job"
"group_interval": "5m"
"group_wait": "30s"
"receiver": "null"
"repeat_interval": "12h"
"routes":
- "match":
"alertname": "DeadMansSwitch"
"receiver": "null"
```

我们可以看到内容和上⾯查看的配置信息是⼀致的，所以如果我们想要添加⾃⼰的接收器，或者模板 消息，我们就可以更改这个⽂件：

```
global:
  resolve_timeout: 5m 
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_from: '515002119@qq.com'  
  smtp_auth_username: '515002119@qq.com'
  smtp_auth_password: 'password'
  smtp_hello: 'qq.com'
  smtp_require_tls: false
route:                     
  group_by: ['job', 'severity']
  group_wait: 10s 
  group_interval: 30s
  repeat_interval: 5m
  receiver: default
  routes:
  - receiver: email
    group_wait: 10s
    match:
      alertname: CoreDNSDown
receivers:
- name: 'default'
  email_configs:
  - to: '515002119@qq.com'
    send_resolved: true
- name: 'email'
  email_configs:
  - to: '515002119@qq.com'
    send_resolved: true


```

将上⾯⽂件保存为 alertmanager.yaml，然后使⽤这个⽂件创建⼀个 Secret 对象：

```
[root@adm-master1 manifests]# kubectl delete secret alertmanager-main -n monitoring
secret "alertmanager-main" deleted
[root@adm-master1 manifests]# kubectl create secret generic alertmanager-main --from-file=alertmanager.yaml -n monitoring
secret/alertmanager-main created

```

我们再次查看 AlertManager ⻚⾯的 status ⻚⾯的配置信息可以看到已经变成上⾯我们的配置信息了：

![image-20200106152843413](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106152843413.png)

AlertManager 配置也可以使⽤模板(.tmpl⽂件)，这些模板可以与 alertmanager.yaml 配置⽂件⼀起添加到 Secret 对象中，⽐如：

```
apiVersion：v1
kind：secret
metadata：
	name：alertmanager-example
data：
	alertmanager.yaml：{BASE64_CONFIG}
	template_1.tmpl：{BASE64_TEMPLATE_1}
	template_2.tmpl：{BASE64_TEMPLATE_2}
```

模板会被放置到与配置⽂件相同的路径，当然要使⽤这些模板⽂件，还需要在 alertmanager.yaml 配 置⽂件中指定：

```
templates:
- '*.tmpl'
```

#### 20.9.4 自动发现配置

为解决上⾯的问题，Prometheus Operator 为我们提供了⼀个额外的抓取配置的来解决这个问题，我 们可以通过添加额外的配置来进⾏服务发现进⾏⾃动监控。和前⾯⾃定义的⽅式⼀样，我们想要在 Prometheus Operator 当中去⾃动发现并监控具有 prometheus.io/scrape=true 这个 annotations 的 Service，之前我们定义的 Prometheus 的配置如下：

```
- job_name: 'kubernetes-service-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
    action: replace
    target_label: __scheme__
    regex: (https?)
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
  - action: labelmap
    regex: __meta_kubernetes_service_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_name

```

要想⾃动发现集群中的 Service，就需要我们在 Service 的 annotation 区域添 加 prometheus.io/scrape=true 的声明，将上⾯⽂件直接保存为 prometheus-additional.yaml，然后通 过这个⽂件创建⼀个对应的 Secret 对象：

```
[root@adm-master1 manifests]# kubectl create secret generic additional-configs --from-file=prometheus-additional.yaml -n monitoring
secret/additional-configs created
```

创建完成后，会将上⾯配置信息进⾏ base64 编码后作为 prometheus-additional.yaml 这个 key 对应的 值存在：

```
[root@adm-master1 manifests]# kubectl get secret additional-configs -n monitoring -o yaml
apiVersion: v1
data:
  prometheus-additional.yaml: LSBqb2JfbmFtZTogJ2t1YmVybmV0ZXMtc2VydmljZS1lbmRwb2ludHMnCiAga3ViZXJuZXRlc19zZF9jb25maWdzOgogIC0gcm9sZTogZW5kcG9pbnRzCiAgcmVsYWJlbF9jb25maWdzOgogIC0gc291cmNlX2xhYmVsczogW19fbWV0YV9rdWJlcm5ldGVzX3NlcnZpY2VfYW5ub3RhdGlvbl9wcm9tZXRoZXVzX2lvX3NjcmFwZV0KICAgIGFjdGlvbjoga2VlcAogICAgcmVnZXg6IHRydWUKICAtIHNvdXJjZV9sYWJlbHM6IFtfX21ldGFfa3ViZXJuZXRlc19zZXJ2aWNlX2Fubm90YXRpb25fcHJvbWV0aGV1c19pb19zY2hlbWVdCiAgICBhY3Rpb246IHJlcGxhY2UKICAgIHRhcmdldF9sYWJlbDogX19zY2hlbWVfXwogICAgcmVnZXg6IChodHRwcz8pCiAgLSBzb3VyY2VfbGFiZWxzOiBbX19tZXRhX2t1YmVybmV0ZXNfc2VydmljZV9hbm5vdGF0aW9uX3Byb21ldGhldXNfaW9fcGF0aF0KICAgIGFjdGlvbjogcmVwbGFjZQogICAgdGFyZ2V0X2xhYmVsOiBfX21ldHJpY3NfcGF0aF9fCiAgICByZWdleDogKC4rKQogIC0gc291cmNlX2xhYmVsczogW19fYWRkcmVzc19fLCBfX21ldGFfa3ViZXJuZXRlc19zZXJ2aWNlX2Fubm90YXRpb25fcHJvbWV0aGV1c19pb19wb3J0XQogICAgYWN0aW9uOiByZXBsYWNlCiAgICB0YXJnZXRfbGFiZWw6IF9fYWRkcmVzc19fCiAgICByZWdleDogKFteOl0rKSg/OjpcZCspPzsoXGQrKQogICAgcmVwbGFjZW1lbnQ6ICQxOiQyCiAgLSBhY3Rpb246IGxhYmVsbWFwCiAgICByZWdleDogX19tZXRhX2t1YmVybmV0ZXNfc2VydmljZV9sYWJlbF8oLispCiAgLSBzb3VyY2VfbGFiZWxzOiBbX19tZXRhX2t1YmVybmV0ZXNfbmFtZXNwYWNlXQogICAgYWN0aW9uOiByZXBsYWNlCiAgICB0YXJnZXRfbGFiZWw6IGt1YmVybmV0ZXNfbmFtZXNwYWNlCiAgLSBzb3VyY2VfbGFiZWxzOiBbX19tZXRhX2t1YmVybmV0ZXNfc2VydmljZV9uYW1lXQogICAgYWN0aW9uOiByZXBsYWNlCiAgICB0YXJnZXRfbGFiZWw6IGt1YmVybmV0ZXNfbmFtZQo=
kind: Secret
metadata:
  creationTimestamp: "2020-01-06T07:40:10Z"
  name: additional-configs
  namespace: monitoring
  resourceVersion: "1668971"
  selfLink: /api/v1/namespaces/monitoring/secrets/additional-configs
  uid: 621ecaf6-7582-4ff2-bcb3-40a9a2dd7cdb
type: Opaque

```

然后我们只需要在声明 prometheus 的资源对象⽂件中添加上这个额外的配置：(prometheus-prometheus.yaml)

```
[root@adm-master1 manifests]# cat prometheus-prometheus.yaml
.....omit......
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  additionalScrapeConfigs:
    name: additional-configs
    key: prometheus-additional.yaml
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.11.0

```

添加完成后，直接更新 prometheus 这个 CRD 资源对象：

```
[root@adm-master1 manifests]# kubectl apply -f prometheus-prometheus.yaml
prometheus.monitoring.coreos.com/k8s configured
```

隔⼀⼩会⼉，可以前往 Prometheus 的 Dashboard 中查看配置是否⽣效：

![image-20200106154751343](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106154751343.png)

在 Prometheus Dashboard 的配置⻚⾯下⾯我们可以看到已经有了对应的的配置信息了，但是我们切 换到 targets ⻚⾯下⾯却并没有发现对应的监控任务，查看 Prometheus 的 Pod ⽇志：

```
level=error ts=2020-01-06T07:51:32.221Z caller=klog.go:94 component=k8s_client_runtime func=ErrorDepth msg="/app/discovery/kubernetes/kubernetes.go:264: Failed to list *v1.Service: services is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list resource \"services\" in API group \"\" at the cluster scope"

```

可以看到有很多错误⽇志出现，都是 xxx is forbidden ，这说明是 RBAC 权限的问题，通过 prometheus 资源对象的配置可以知道 Prometheus 绑定了⼀个名为 prometheus-k8s 的 ServiceAccount 对象，⽽这个对象绑定的是⼀个名为 prometheus-k8s 的 ClusterRole： （prometheus-clusterRole.yaml)

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get



[root@adm-master1 manifests]# kubectl apply -f prometheus-clusterRole.yaml 
clusterrole.rbac.authorization.k8s.io/prometheus-k8s configured
#要解决这个 问题，我们只需要添加上需要的权限即可
```

更新上⾯的 ClusterRole 这个资源对象，然后重建下 Prometheus 的所有 Pod，正常就可以看到 targets ⻚⾯下⾯有 kubernetes-service-endpoints 这个监控任务了：

![image-20200106155550195](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200106155550195.png)

#### 20.9.5 数据持久化

上⾯我们在修改完权限的时候，重启了 Prometheus 的 Pod，如果我们仔细观察的话会发现我们之前 采集的数据已经没有了，这是因为我们通过 prometheus 这个 CRD 创建的 Prometheus 并没有做数据 的持久化，我们可以直接查看⽣成的 Prometheus Pod 的挂载情况就清楚了：

```
[root@adm-master1 manifests]# kubectl get pod prometheus-k8s-0 -n monitoring -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-01-06T04:04:50Z"
  generateName: prometheus-k8s-
 ...omit...
    volumeMounts:
    - mountPath: /etc/prometheus/config_out
      name: config-out
      readOnly: true
    - mountPath: /etc/prometheus/certs
      name: tls-assets
      readOnly: true
    - mountPath: /prometheus
      name: prometheus-k8s-db
    - mountPath: /etc/prometheus/rules/prometheus-k8s-rulefiles-0
      name: prometheus-k8s-rulefiles-0
    - mountPath: /etc/prometheus/secrets/etcd-certs
      name: secret-etcd-certs
      readOnly: true
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: prometheus-k8s-token-zgk5p
      readOnly: true
  - args:
    - --log-format=logfmt
    - --reload-url=http://localhost:9090/-/reload
    - --config-file=/etc/prometheus/config/prometheus.yaml.gz
    - --config-envsubst-file=/etc/prometheus/config_out/prometheus.env.yaml
    command:
    - /bin/prometheus-config-reloader
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    
      readOnly: true

    - mountPath: /etc/prometheus/rules/prometheus-k8s-rulefiles-0
      name: prometheus-k8s-rulefiles-0
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: prometheus-k8s-token-zgk5p
      readOnly: true
 
  volumes:
  - name: config
    secret:
      defaultMode: 420
      secretName: prometheus-k8s
  - name: tls-assets
    secret:
      defaultMode: 420
      secretName: prometheus-k8s-tls-assets
  - emptyDir: {}
    name: config-out
  - configMap:
      defaultMode: 420
      name: prometheus-k8s-rulefiles-0
    name: prometheus-k8s-rulefiles-0
  - name: secret-etcd-certs
    secret:
      defaultMode: 420
      secretName: etcd-certs
  - emptyDir: {}
   name: prometheus-k8s-db
  - name: prometheus-k8s-token-zgk5p
....omit.....
```

我们可以看到 Prometheus 的数据⽬录 /prometheus 实际上是通过 emptyDir 进⾏挂载的，我们知道 emptyDir 挂载的数据的⽣命周期和 Pod ⽣命周期⼀致的，所以如果 Pod 挂掉了，数据也就丢失了， 这也就是为什么我们重建 Pod 后之前的数据就没有了的原因，对应线上的监控数据肯定需要做数据的 持久化的，同样的 prometheus 这个 CRD 资源也为我们提供了数据持久化的配置⽅法，由于我们的 Prometheus 最终是通过 Statefulset 控制器进⾏部署的，所以我们这⾥需要通过 storageclass 来做数 据持久化，⾸先创建⼀个 StorageClass 对象：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: prometheus-data-db
provisioner: nfs-client


[root@adm-master1 manifests]# kubectl create -f prometheus-storageclass.yaml
storageclass.storage.k8s.io/prometheus-data-db created
```

这⾥我们声明⼀个 StorageClass 对象，其中 provisioner=nfs-client，则是因为我们集群中使⽤的 是 nfs 作为存储后端，⽽前⾯我们课程中创建的 nfs-client-provisioner 中指定的 PROVISIONER_NAME 就为 nfs-client，这个名字不能随便更改，将该⽂件保存为 prometheus-storageclass.yaml:**(下面列出 nfs-client 的yaml 文件)**

```
kind: ServiceAccount
apiVersion: v1
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
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default      #替换成你要部署NFS Provisioner的 Namespace
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate      #---设置升级策略为删除再创建(默认为滚动更新)
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
          #---由于quay.io仓库国内被墙，所以替换成七牛云的仓库
          image: quay-mirror.qiniu.com/external_storage/nfs-client-provisioner:latest 
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-client     #--- nfs-provisioner的名称，以后设置的storageclass要和这个保持一致
            - name: NFS_SERVER
              value: 192.168.208.195   #---NFS服务器地址，和 valumes 保持一致
            - name: NFS_PATH
              value: /home/k8sv      #---NFS服务器目录，和 valumes 保持一致
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.208.195    #---NFS服务器地址
            path: /home/k8sv        #---NFS服务器目录

```

然后在 prometheus 的 CRD 资源对象中添加如下配置：

```
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: prometheus-data-db
        resources:
          requests:
            storage: 10Gi
            
            
[root@adm-master1 manifests]# kubectl apply -f prometheus-prometheus.yaml 
prometheus.monitoring.coreos.com/k8s configured
```

注意这⾥的 storageClassName 名字为上⾯我们创建的 StorageClass 对象名称，然后更新 prometheus 这个 CRD 资源。更新完成后会⾃动⽣成两个 PVC 和 PV 资源对象：

```
[root@adm-master1 manifests]# kubectl get pods -n monitoring
NAME                                  READY   STATUS    RESTARTS   AGE
....omit.....
prometheus-k8s-0                      3/3     Running   0          48s
prometheus-k8s-1                      3/3     Running   0          48s
prometheus-operator-99dccdc56-flq4k   1/1     Running   0          6h44m


[root@adm-master1 manifests]# kubectl get pvc -n monitoring
NAME                                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS         AGE
prometheus-k8s-db-prometheus-k8s-0   Bound    pvc-befff6fa-bca7-484d-82a7-2396da71efa5   10Gi       RWO            prometheus-data-db   3m19s
prometheus-k8s-db-prometheus-k8s-1   Bound    pvc-d98dc85c-8dbc-46e3-b477-0013e955a0a2   10Gi       RWO            prometheus-data-db   3m19s


[root@adm-master1 manifests]# kubectl get pod prometheus-k8s-0 -n monitoring -o yaml

....omit....
  volumes:
  - name: prometheus-k8s-db
    persistentVolumeClaim:
      claimName: prometheus-k8s-db-prometheus-k8s-0
```



