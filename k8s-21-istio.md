## 21、Istio

### 21.1为什么我们需要Istio 

随着单片应用程序向分布式微服务架构过渡，Istio解决了开发人员和运营商面临的许多挑战。术语**服务网格**通常用于描述构成此类应用程序的微服务网络以及它们之间的交互。随着服务网格的大小和复杂性的增加，理解和管理变得更加困难。其要求可包括发现，负载平衡，故障恢复，指标和监控，以及通常更复杂的操作要求，如A / B测试，金丝片发布，速率限制，访问控制和端到端身份验证。

Istio通过提供整体服务网格的行为洞察和操作控制，提供完整的解决方案，以满足微服务应用的各种需求。它在服务网络中统一提供了许多关键功能：

- **交通管理**。控制服务之间的流量和API调用流，使呼叫更可靠，并在面对不利条件时使网络更加健壮。
- **服务身份和安全**。在网格中提供具有可验证身份的服务，并提供在流经不同可信度的网络时保护服务流量的能力。
- **政策执行**。将组织策略应用于服务之间的交互，确保实施访问策略，并在消费者之间公平地分配资源。通过配置网格而不是通过更改应用程序代码来进行策略更改。
- **遥测**。了解服务之间的依赖关系以及它们之间的流量的性质和流量，提供快速识别问题的能力。

除了这些行为，Istio还可以扩展以满足不同的部署需求：

- **平台支持**。Istio旨在运行在各种环境中，包括云，内部部署，Kubernetes，Mesos等。我们最初专注于Kubernetes，但很快就会努力支持其他环境。
- **集成和定制**。策略实施组件可以扩展和定制，以与现有的ACL，日志记录，监控，配额，审计等解决方案集成。

这些功能极大地减少了应用程序代码，底层平台和策略之间的耦合。这种减少的耦合不仅使服务更容易实现，而且使操作员更容易在环境之间移动应用程序部署或新的策略方案。因此，应用程序本身更具可移植性。







- Kubernetes 提供平台基础设施层强大的容器编排与调度能力

 – 服务部署与弹性伸缩：Deployment

 – 服务拆分与服务发现：Service

-  Kubernetes 提供简单的负载均衡 

– 负载均衡：基于IPVS或Iptables的简单均衡机制

service mesh （服务网格）

• 治理能力独立（Sidecar）          • 应用程序无感知             • 服务通信的基础设施层

![image-20191225134523936](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225134523936.png)



istio：

• 连接（Connect）                  • 安全（Secure）                • 控制（Control）            • 观察（Observe）



istio 的关键能力

![image-20191225134847900](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225134847900.png)

说白了 就是可以补充k8s 服务治理的能力

### 21.2 Istio介绍及核心功能 

istio架构图

![image-20191225135053050](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225135053050.png)

Pilot：配置规则  注册中心，配置中心

![image-20191225135328776](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225135328776.png)

用户自己配置 + k8s的元数据   pilot 根据你的配置在进行服务转发 



mixer：  policy checks （策略管理）  +telemetry （监控）   指标收集，对接外部组件（Prometheus，Fluentd，Jaeger）

![image-20191225135504852](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225135504852.png)

citadel ：安全 证书

![image-20191225135532863](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225135532863.png)



与k8s集群结合：

![image-20191225135550367](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225135550367.png)

ENVOY：（作用是转发）动态服务获取，负载平衡，路由，断路器，超时，指标上报。

• 基于C++的 L4/L7 Proxy转发器                                      • CNCF第三个毕业的项目



• Listeners （LDS）               • Routes （RDS）                • Clusters （CDS）             • Endpoints （EDS）

envoy的配置文件:(静态配置在实际中 envoy是动态配置)

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 static_resources: 9001 }   #管理端口
static_resources:    
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }       #listeners （LDS）
    filter_chains:
    - filters:
    - name: envoy.http_connection_manager
      config:
        stat_prefix: ingress_http
        codec_type: AUTO
        route_config:
          name: local_route
          virtual_hosts:
          - name: local_service                                 #Routes  来匹配cluster
            domains: ["*"]
            routes:
            - match: { prefix: "/" }
              route: { cluster: service_envoy }
        http_filters:
        - name: envoy.router
    clusters:
      - name: service_envoy                      #cluster
      connect_timeout: 0.25s
      type: LOGICAL_DNS
      lb_policy: ROUND_ROBIN
      hosts: [{ socket_address: { address: 192.168.0.1, port_value: 443 }}]    #endpoint 转发到哪里去
```

**Istio 基础概念**

![image-20191225141123430](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191225141123430.png)

- Gateway   提供外部服务访问接入，可发布任意内部端口的服务，供外部访问。配合VirtualService使用，使用标准Istio规则治理

     

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: Gateway
  metadata:
    name: bookinfo-gateway
  spec:
    selector:
      istio: ingressgateway # use istio default controller
    servers:
    - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
    ---
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: bookinfo
  spec:
    hosts:
    - "*"
    gateways:
    - bookinfo-gateway
    http:
    - match:
      - uri:
        exact: /productpage
    route:
      - destination:
      ...
  ```

   

- VirtualServic  最核心的配置接口 定义指定服务的所有路由规则         （进行版本选择）

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: VirtualService
  metadata:
    name: reviews-route
    namespace: istio
  spec:
    hosts:
    - reviews
    http:
    - match:
      - uri:
          prefix: "/gomolog"
      - uri:
          prefix: "/3glog"
      rewrite:
        uri: "/newcatalog"
      route:
      - destination:
          host: reviews
          subset: v2
    - route:
      - destination:
        host: reviews
        subset: v1
  ```

- DestinationRule  其定义的策略，决定了路由处理之后的流 量访问策略。负载均衡设置，断路器， TLS设置等   

  ```yaml
  apiVersion: networking.istio.io/v1alpha3
  kind: DestinationRule
  metadata:
    name: bookinfo-ratings
  spec:
    host: ratings
    trafficPolicy:       #默认策略
      loadBalancer:
        simple: LEAST_CONN
    subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
    - name: v3
      labels:
        version: v3
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN    
  ```

- ServiceEntry  将外部服务接入到服务注册表中，让Istio中自动发现的服务能够访问和路由到这些手工加入的服务。与VirtualService或DestinationRule配合使用

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain
  addresses:
  - 192.168.208.192/24 # VIPs
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: mtls-mongocluster
spec:
  host: mymongodb.somedomain
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /etc/certs/myclientcert.pem
      privateKey: /etc/certs/client_private_key.pem
      caCertificates: /etc/certs/rootcacerts.pem
```



### 21.3 Istio使用场景 

​      2019 istio 最愿意支持的是k8s   在1.0之后 eureka 被取消

### 21.4  Istio在K8S中部署 

**Istioctl** • proxy-status：状态同步情况 

• proxy-config：envoy中具体规则查询

​	 • listener  • route • cluster • endpoint 

• kube-inject • ……

搭建部署：

来自官方文档：

Before you can install Istio, you need a cluster running a compatible version of Kubernetes. Istio 1.4 has been tested with Kubernetes releases 1.13, 1.14, 1.15.（**1.16 1.17 搭建出错误 不是很牛逼的 别去看日志了 删除重新建pods就可以了    1.16.2 日志看不出任何内容！kiali 一直起不来 删除重新建就好了  同时 istio 不支持helm v3！**）

Create a cluster by selecting the appropriate [platform-specific setup instructions](https://istio.io/docs/setup/platform-setup/).

Some platforms provide a managed control plane which you can use instead of installing Istio manually. If this is the case with your selected platform, and you choose to use it, you will be finished installing Istio after creating the cluster, so you can skip the following instructions. Refer to your platform service provider for further details and instructions.

Download the Istio release which includes installation files, samples, and the [istioctl](https://istio.io/docs/reference/commands/istioctl/) command line utility.

1. Go to the [Istio release](https://github.com/istio/istio/releases/tag/1.4.2) page to download the installation file corresponding to your OS. Alternatively, on a macOS or Linux system, you can run the following command to download and extract the latest release automatically:

   ```
   $ curl -L https://istio.io/downloadIstio | sh -
   ```

Move to the Istio package directory. For example, if the package is `istio-1.4.2`:

```
$ cd istio-1.4.2
```

The installation directory contains:

- Installation YAML files for Kubernetes in `install/kubernetes`
- Sample applications in `samples/`
- The [`istioctl`](https://istio.io/docs/reference/commands/istioctl) client binary in the `bin/` directory. `istioctl` is used when manually injecting Envoy as a sidecar proxy.

Add the `istioctl` client to your path, on a macOS or Linux system:

```
$ export PATH=$PWD/bin:$PATH
```

You can optionally enable the [auto-completion option](https://istio.io/docs/ops/diagnostic-tools/istioctl#enabling-auto-completion) when working with a bash or ZSH console.

These instructions assume you are new to Istio, providing streamlined instruction to install Istio’s built-in `demo` [configuration profile](https://istio.io/docs/setup/additional-setup/config-profiles/). This installation lets you quickly get started evaluating Istio. If you are already familiar with Istio or interested in installing other configuration profiles or a more advanced [deployment model](https://istio.io/docs/ops/deployment/deployment-models/), follow the [installing with istioctl instructions](https://istio.io/docs/setup/install/istioctl) instead.

The demo configuration profile is not suitable for performance evaluation. It is designed to showcase Istio functionality with high levels of tracing and access logging.

1. Install the `demo` profile

   ```
   $ istioctl manifest apply --set profile=demo
   Preparing manifests for these components:
   - Telemetry
   - Galley
   - EgressGateway
   - Grafana
   - PrometheusOperator
   - CoreDNS
   - Kiali
   - NodeAgent
   - Injector
   - Prometheus
   - Citadel
   - Pilot
   - IngressGateway
   - Cni
   - Policy
   - CertManager
   - Base
   - Tracing
   
   Applying manifest for component Base
   Finished applying manifest for component Base
   Applying manifest for component Tracing
   Applying manifest for component IngressGateway
   Applying manifest for component EgressGateway
   Applying manifest for component Prometheus
   Applying manifest for component Injector
   Applying manifest for component Kiali
   Applying manifest for component Galley
   Applying manifest for component Pilot
   Applying manifest for component Citadel
   Applying manifest for component Policy
   Applying manifest for component Telemetry
   Applying manifest for component Grafana
   Finished applying manifest for component Kiali
   Finished applying manifest for component Galley
   Finished applying manifest for component Citadel
   Finished applying manifest for component Injector
   Finished applying manifest for component Prometheus
   Finished applying manifest for component Pilot
   Finished applying manifest for component Tracing
   Finished applying manifest for component Policy
   Finished applying manifest for component IngressGateway
   Finished applying manifest for component EgressGateway
   Finished applying manifest for component Grafana
   Finished applying manifest for component Telemetry
   
   Component CertManager installed successfully:
   =============================================
   
   Component Base installed successfully:
   ======================================
   
   Component Tracing installed successfully:
   =========================================
   
   Component IngressGateway installed successfully:
   ================================================
   
   Component Cni installed successfully:
   =====================================
   
   Component Policy installed successfully:
   ========================================
   
   Component EgressGateway installed successfully:
   ===============================================
   
   Component Grafana installed successfully:
   =========================================
   
   Component PrometheusOperator installed successfully:
   ====================================================
   
   Component Telemetry installed successfully:
   ===========================================
   
   Component Galley installed successfully:
   ========================================
   
   Component NodeAgent installed successfully:
   ===========================================
   
   Component CoreDNS installed successfully:
   =========================================
   
   Component Kiali installed successfully:
   =======================================
   
   Component Citadel installed successfully:
   =========================================
   
   Component Pilot installed successfully:
   =======================================
   
   Component Injector installed successfully:
   ==========================================
   
   Component Prometheus installed successfully:
   ============================================
   ```

Verify the installation by ensuring the following Kubernetes services are deployed and verify they all have an appropriate `CLUSTER-IP` except the `jaeger-agent` service:

```
[root@k8s-master1 ~]# kubectl get svc -n istio-system
NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                      AGE
grafana                  ClusterIP      10.254.122.35    <none>        3000/TCP                                                                                                                     12m
istio-citadel            ClusterIP      10.254.39.91     <none>        8060/TCP,15014/TCP                                                                                                           12m
istio-egressgateway      ClusterIP      10.254.151.93    <none>        80/TCP,443/TCP,15443/TCP                                                                                                     12m
istio-galley             ClusterIP      10.254.215.233   <none>        443/TCP,15014/TCP,9901/TCP,15019/TCP                                                                                         12m
istio-ingressgateway     LoadBalancer   10.254.130.92    <pending>     15020:31881/TCP,80:32497/TCP,443:31428/TCP,15029:31350/TCP,15030:30989/TCP,15031:30814/TCP,15032:30239/TCP,15443:31921/TCP   12m
istio-pilot              ClusterIP      10.254.142.240   <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                       12m
istio-policy             ClusterIP      10.254.237.95    <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                 12m
istio-sidecar-injector   ClusterIP      10.254.46.150    <none>        443/TCP                                                                                                                      12m
istio-telemetry          ClusterIP      10.254.67.210    <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                       12m
jaeger-agent             ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                                                   13m
jaeger-collector         ClusterIP      10.254.24.215    <none>        14267/TCP,14268/TCP,14250/TCP                                                                                                13m
jaeger-query             ClusterIP      10.254.182.86    <none>        16686/TCP                                                                                                                    13m
kiali                    ClusterIP      10.254.74.12     <none>        20001/TCP                                                                                                                    12m
prometheus               ClusterIP      10.254.241.113   <none>        9090/TCP                                                                                                                     12m
tracing                  ClusterIP      10.254.84.3      <none>        80/TCP                                                                                                                       13m
zipkin                   ClusterIP      10.254.152.37    <none>        9411/TCP                                                                                                                     13m
```

If your cluster is running in an environment that does not support an external load balancer (e.g., minikube), the `EXTERNAL-IP` of `istio-ingressgateway` will say ``. To access the gateway, use the service’s `NodePort`, or use port-forwarding instead.

Also ensure corresponding Kubernetes pods are deployed and have a `STATUS` of `Running`:

```
[root@k8s-master1 ~]# kubectl get pods -n istio-system
NAME                                      READY   STATUS    RESTARTS   AGE
grafana-6b65874977-fqr2w                  1/1     Running   0          12m
istio-citadel-86dcf4c6b-jp9t5             1/1     Running   0          13m
istio-egressgateway-68f754ccdd-qjmxh      1/1     Running   0          13m
istio-galley-5fc6d6c45b-dh2tc             1/1     Running   0          12m
istio-ingressgateway-6d759478d8-s6xpb     1/1     Running   0          13m
istio-pilot-5c4995d687-gsbtm              1/1     Running   0          12m
istio-policy-57b99968f-lfvw6              1/1     Running   3          13m
istio-sidecar-injector-746f7c7bbb-9fncq   1/1     Running   0          12m
istio-telemetry-854d8556d5-t6t54          1/1     Running   3          13m
istio-tracing-c66d67cd9-qnfpp             1/1     Running   0          13m
kiali-8559969566-769dd                    1/1     Running   0          12m
prometheus-66c5887c86-t7fhl               1/1     Running   0          13m
```

With Istio installed, you can now deploy your own application or one of the sample applications provided with the installation.

The application must use either the HTTP/1.1 or HTTP/2.0 protocols for all its HTTP traffic; HTTP/1.0 is not supported.

When you deploy your application using `kubectl apply`, the [Istio sidecar injector](https://istio.io/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection) will automatically inject Envoy containers into your application pods if they are started in namespaces labeled with `istio-injection=enabled`:

```
$ kubectl label namespace <namespace> istio-injection=enabled
$ kubectl create -n <namespace> -f <your-app-spec>.yaml
```

```
[root@k8s-master1 ~]# kubectl get ns -L istio-injection
NAME                   STATUS   AGE     ISTIO-INJECTION
default                Active   9d      enable
efk                    Active   5d22h   
ingress-nginx          Active   9d      
istio-system           Active   18m     disabled
kube-node-lease        Active   9d      
kube-public            Active   9d      
kube-system            Active   9d      
kubernetes-dashboard   Active   9d      
prometheus             Active   2d4h    
```

In namespaces without the `istio-injection` label, you can use [`istioctl kube-inject`](https://istio.io/docs/reference/commands/istioctl/#istioctl-kube-inject) to manually inject Envoy containers in your application pods before deploying them:

```
$ istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
```

示例Bookinfo 应用
        部署一个样例应用，它由四个单独的微服务构成，用来演示多种 Istio 特性。这个应用模仿在线书店的一个分类，显示一本书的信息。页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

    Bookinfo 应用分为四个单独的微服务：

productpage ：productpage 微服务会调用 details 和 reviews 两个微服务，用来生成页面。
details ：这个微服务包含了书籍的信息。
reviews ：这个微服务包含了书籍相关的评论。它还会调用 ratings 微服务。
ratings ：ratings 微服务中包含了由书籍评价组成的评级信息。
   reviews 微服务有 3 个版本：

v1 版本不会调用 ratings 服务。
v2 版本会调用 ratings 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
v3 版本会调用 ratings 服务，并使用 1 到 5 个红色星形图标来显示评分信息。
    下图展示了这个应用的端到端架构。


![Bookinfo Application without Istio](https://istio.io/docs/examples/bookinfo/noistio.svg)



`Start the application services`

If you use GKE, please ensure your cluster has at least 4 standard GKE  nodes. If you use Minikube, please ensure you have at least 4GB RAM.

1. Change directory to the root of the Istio installation.

2. The default Istio installation uses [automatic sidecar injection](https://istio.io/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection). Label the namespace that will host the application with `istio-injection=enabled`:

   ```
   $ kubectl label namespace default istio-injection=enabled
   ```

If you use OpenShift, make sure to give appropriate permissions to service accounts on the namespace as described in [OpenShift setup page](https://istio.io/docs/setup/platform-setup/openshift/#privileged-security-context-constraints-for-application-sidecars).

Deploy your application using the `kubectl` command:

```
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
[root@k8s-master1 kube]# kubectl apply -f bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
deployment.apps/productpage-v1 created

```

If you disabled automatic sidecar injection during installation and rely on [manual sidecar injection](https://istio.io/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection), use the [`istioctl kube-inject`](https://istio.io/docs/reference/commands/istioctl/#istioctl-kube-inject) command to modify the `bookinfo.yaml` file before deploying your application.

```
$ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
```

The command launches all four services shown in the `bookinfo` application architecture diagram. All 3 versions of the reviews service, v1, v2, and v3, are started.

In a realistic deployment, new versions of a microservice are deployed over time instead of deploying all versions simultaneously.

Confirm all services and pods are correctly defined and running:

```
[root@k8s-master1 kube]# kubectl get svc
NAME                             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                       AG
ratings                          ClusterIP   10.254.42.6      <none>        9080/TCP                      39s
reviews                          ClusterIP   10.254.165.214   <none>        9080/TCP                      39s
[root@k8s-master1 kube]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-78d78fbddf-jdp59       1/1     Running   0          5m15s
productpage-v1-596598f447-pmsz6   1/1     Running   0          5m13s
ratings-v1-6c9dbf6b45-jdhqp       1/1     Running   0          5m14s
reviews-v1-7bb8ffd9b6-cr2rl       1/1     Running   0          5m15s
reviews-v2-d7d75fff8-rmzxw        1/1     Running   0          5m15s
reviews-v3-68964bc4c8-gs89g       1/1     Running   0          5m15s

```

```
[root@k8s-master1 kube]# kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
                                   <title>Simple Bookstore App</title>

```

Determine the ingress IP and port

Now that the Bookinfo services are up and running, you need to make the application accessible from outside of your Kubernetes cluster, e.g., from a browser. An [Istio Gateway](https://istio.io/docs/concepts/traffic-management/#gateways) is used for this purpose.

1. Define the ingress gateway for the application:

   ```
   $ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
   ```

Confirm the gateway has been created:

```
$ kubectl get gateway
NAME               AGE
bookinfo-gateway   32s
```

Follow [these instructions](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports) to set the `INGRESS_HOST` and `INGRESS_PORT` variables for accessing the gateway. Return here, when they are set.

Set `GATEWAY_URL`:

```
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

To confirm that the Bookinfo application is accessible from outside the cluster, run the following `curl` command:

```
$ curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

You can also point your browser to `http://$GATEWAY_URL/productpage` to view the Bookinfo web page. If you refresh the page several times, you should see different versions of reviews shown in `productpage`, presented in a round robin style (red stars, black stars, no stars), since we haven’t yet used Istio to control the version routing.

![image-20191230093641206](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230093641206.png)

Apply default destination rules

Before you can use Istio to control the Bookinfo version routing, you need to define the available versions, called *subsets*, in [destination rules](https://istio.io/docs/concepts/traffic-management/#destination-rules).

Run the following command to create default destination rules for the Bookinfo services:

- If you did **not** enable mutual TLS, execute this command:

  

  Choose this option if you are new to Istio and are using the `demo` [configuration profile](https://istio.io/docs/setup/additional-setup/config-profiles/).

  ```
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
  ```

If you **did** enable mutual TLS, execute this command:

```
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
```

Wait a few seconds for the destination rules to propagate.

You can display the destination rules with the following command:

```
$ kubectl get destinationrules -o yaml
```



### 21.5 Istio Pilot与服务发现

**服务发现基本原理**

![image-20191230094850098](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230094850098.png)

一个服务调用另一个服务 （通过服务名字 来访问 在服务列表中查询在调用）

例如 当你访问一个名字为gomo的服务 ，不用去后面加IP  因为IP 可能是会变的 可能是11 有时候可能是12 有时候可能是13 。

```
gomo 11.11.11.11 
gomo 11.11.11.12 
gomo 11.11.11.13 
```

**SpringCloud的服务(注册与)发现流程**

![image-20191230095359752](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230095359752.png)

• 服务注册表：如Springcloud中一般Eureka服 务；

 • 服务注册：服务配置文件中配置服务名和本实 例地址，实例启动时自动注册到服务注册表；

 • 服务发现：访问目标服务时连服务注册表，获 取服务实例列表。根据LB根据策略选择一个服 务实例，建立连接去访问。

**Istio服务发现流程**

![image-20191230095501149](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230095501149.png)

但serviceA 去发出连接请求的时候 会被Proxy所阻断掉，然后连接到Pilot上面，通过Pilot 拿到 服务发现的数据 和负载均衡的策略。在istio里面 serviceA 要想通过serviceB，serviceA的流量必须经过serviceB 里面的 Proxy。（Proxy 来拦截流量，但是2个service之间的是感觉不到对方的proxy的存在）

• 服务注册表：Pilot 从平台获取服务发 现数据，并提供统一的服务发现接口。

• 服务注册：无

• 服务发现：Envoy 实现服务发现，动 态更新负载均衡池。在服务请求时使 用对应的负载均衡策略将请求路由到 对应的后端。

**Pilot服务发现机制的Adapter机制**

![image-20191230102022967](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230102022967.png)

**Istio服务发现实现：基于 Eureka**

1. Pilot 实现若干服务发现的接口定义 
2. Controller使用EurekaClient来获取服 务列表，提供转换后的标准的服务发现 接口和数据结构； 
3. Discoveryserver基于Controller上维 护的服务发现数据，发布成gRPC协议 的服务供Envoy使用。
4. 当有服务访问时 ， Envoy 在处理 Outbound请求时，根据配置的LB策略选择一个服务实例发起访问

![image-20191230102745326](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230102745326.png)

**Istio服务发现实现：基于Kubernetes**

![image-20191230103323761](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230103323761.png)

1. Pilot 实现若干服务发现的接口定义
2. Pilot 的Controller List/Watch  KubeAPIserver上service、 endpoint等资源对象并转换成标准格式。 
3. Envoy从Pilot获取xDS，动态更新 
4. 当有服务访问时，Envoy在处理 Outbound请求时，根据配置的LB 策略，选择一个服务实例发起访问。

**Kubernetes & Istio 服务模型**

![image-20191230103432876](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230103432876.png)

istio 的service 其实就是k8s的service  情况对等的

![image-20191230103637644](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230103637644.png)

做分流规则  来做灰度发布 一个service 有不同的版本 可以对应 k8s 上面的 副本控制器



对比k8s的服务发现（通过域名来访问）

![image-20191230104231576](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230104231576.png)

**服务部署和运维 主要由 kubernetes的本身来做,而服务治理主要由istio来做**，因为istio 的所有能力都是基于k8s 来构建的



**Istio （upon Kubernetes）服务发现和配置**

![image-20191230105103709](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230105103709.png)

服务发现 是pilot 通过 kube-apiserver 去查看 

**istio服务配置管理**

![image-20191230105322168](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230105322168.png)

运维人员配置一个规则 然后通过pilot 来进行发布。

1. 配置：管理员通过Pilot配置治 疗规则 
2. 下发：Envoy从Pilot获取治理 规则 
3. 执行：在流量访问的时候执行 治理规则



**istio的治理规则：**

官网https://istio.io/docs/reference/config/networking/

![image-20191230110254529](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230110254529.png)

1.virtual service:服务访问路由控制。满足特定条件的请求流到哪里，过程中治理。包括请求重写、重试、故障注入等。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - name: "reviews-v2-routes"
    match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v2
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
```

The following example on Kubernetes, routes all HTTP traffic by default to pods of the reviews service with label “version: v1”. In addition, HTTP requests with path starting with /wpcatalog/ or /consumercatalog/ will be rewritten to /newcatalog and sent to pods with label “version: v2”.

A subset/version of a route destination is identified with a reference to a named service subset which must be declared in a corresponding `DestinationRule`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews.prod.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**HTTPRoute**

Describes match conditions and actions for routing HTTP/1.1, HTTP2, and gRPC traffic. See VirtualService for usage examples.

| Field           | Type                     | Description                                                  | Required |
| --------------- | ------------------------ | ------------------------------------------------------------ | -------- |
| `name`          | `string`                 | The name assigned to the route for debugging purposes. The route’s name will be concatenated with the match’s name and will be logged in the access logs for requests matching this route/match. | No       |
| `match`         | `HTTPMatchRequest[]`     | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. | No       |
| `route`         | `HTTPRouteDestination[]` | A http rule can either redirect or forward (default) traffic. The forwarding target can be one of several versions of a service (see glossary in beginning of document). Weights associated with the service version determine the proportion of traffic it receives. | No       |
| `redirect`      | `HTTPRedirect`           | A http rule can either redirect or forward (default) traffic. If traffic passthrough option is specified in the rule, route/redirect will be ignored. The redirect primitive can be used to send a HTTP 301 redirect to a different URI or Authority. | No       |
| `rewrite`       | `HTTPRewrite`            | Rewrite HTTP URIs and Authority headers. Rewrite cannot be used with Redirect primitive. Rewrite will be performed before forwarding. | No       |
| `timeout`       | `Duration`               | Timeout for HTTP requests.                                   | No       |
| `retries`       | `HTTPRetry`              | Retry policy for HTTP requests.                              | No       |
| `fault`         | `HTTPFaultInjection`     | Fault injection policy to apply on HTTP traffic at the client side. Note that timeouts or retries will not be enabled when faults are enabled on the client side. | No       |
| `mirror`        | `Destination`            | Mirror HTTP traffic to a another destination in addition to forwarding the requests to the intended destination. Mirrored traffic is on a best effort basis where the sidecar/gateway will not wait for the mirrored cluster to respond before returning the response from the original destination. Statistics will be generated for the mirrored destination. | No       |
| `mirrorPercent` | `UInt32Value`            | Percentage of the traffic to be mirrored by the `mirror` field. If this field is absent, all the traffic (100%) will be mirrored. Max value is 100. | No       |
| `corsPolicy`    | `CorsPolicy`             | Cross-Origin Resource Sharing policy (CORS). Refer to [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) for further details about cross origin resource sharing. | No       |
| `headers`       | `Headers`                | Header manipulation rules                                    | No       |

**DestinationRule**

目标服务的策略，包括目标服务的负载均衡，连接池管理等。

详细的介绍 官方文档：https://istio.io/docs/reference/config/networking/destination-rule/



### 21.6Gateway设计与技术

#### 21.6.1Gateway简介

官方文档:https://istio.io/docs/reference/config/networking/gateway/

在Istio中，Gateway控制着网格边缘的服务暴露。就类似于k8s 中的ingress

![image-20191230134815557](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230134815557.png)

Gateway也可以看作网格的负载均衡器， 提供以下功能:

- L4-L6的负载均衡 
- 对外的mTLS

Istio服务网格中，Gateway可以部署任意多个，可以共用一个， 也可以每个租户、namespace单独隔离。

`Gateway` 描述了一个运行在网格边缘的负载均衡器，它接收传入或传出的HTTP/TCP连接。该规范描述了一组应该公开的端口、要使用的协议类型、负载均衡器的SNI配置，等等。

例如，下面的网关配置设置了一个代理，作为用于ingress的负载平衡器开放端口80和9080 (http)、443(https)、9443(https)和端口2379 (TCP)。网关将被应用到带有标签app: my-gateway-controller的pod上运行的代理。虽然Istio将配置代理来监听这些端口，但是用户有责任确保这些端口的外部通信能够进入网格。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
  namespace: some-config-namespace
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 80   #gateway 监听的端口和协议
      name: http
      protocol: HTTP
    hosts:           #允许外部访问的host：uk.bookinfo eu.bookinfo 的http流量进入网格内
    - uk.bookinfo.com 
    - eu.bookinfo.com
    tls:
      httpsRedirect: true # sends 301 redirect for http requests
  - port:
      number: 443
      name: https-443
      protocol: HTTPS
    hosts:
    - uk.bookinfo.com
    - eu.bookinfo.com
    tls:
      mode: SIMPLE # enables HTTPS on this port
      serverCertificate: /etc/certs/servercert.pem
      privateKey: /etc/certs/privatekey.pem
  - port:
      number: 9443
      name: https-9443
      protocol: HTTPS
    hosts:
    - "bookinfo-namespace/*.bookinfo.com"
    tls:
      mode: SIMPLE # enables HTTPS on this port
      credentialName: bookinfo-secret # fetches certs from Kubernetes secret
  - port:
      number: 9080
      name: http-wildcard
      protocol: HTTP
    hosts:
    - "*"
  - port:
      number: 2379 # to expose internal service via external port 2379
      name: mongo
      protocol: MONGO
    hosts:
    - "*"
```

这里体现了gateway 在4-6层的负载均衡的代理能力，然后VirtualService可以绑定到网关，以控制到达特定主机或网关端口的流量的转发。（virtualservice 可以做L7层路由）

将VirtualService中的urlhttps://uk.bookinfo.com/reviews、https://eu.bookinfo.com/reviews、http://uk.bookinfo.com:9080/reviews、http://eu.bookinfo.com:9080/reviews的流量分成两个版本(prod和qa)在内部服务端口9080.包含cookie“user: dev-123”的请求将被发送到qa版本中的特殊端口7777。而下面的第而个match 也是同样的道理 但是要注意此规则适用于端口443、9080。同时http://uk.bookinfo.com被重定向到https://uk.bookinfo.com(即80重定向到443)。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-rule
  namespace: bookinfo-namespace
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  - uk.bookinfo.com
  - eu.bookinfo.com
  gateways:
  - some-config-namespace/my-gateway
  - mesh # applies to all the sidecars in the mesh
  http:
  - match:
    - headers:
        cookie:
          exact: "user=dev-123"
    route:
    - destination:
        port:
          number: 7777
        host: reviews.qa.svc.cluster.local
  - match:
    - uri:
        prefix: /reviews/
    route:
    - destination:
        port:
          number: 9080 # can be omitted if it's the only port for reviews
        host: reviews.prod.svc.cluster.local
      weight: 80
    - destination:
        host: reviews.qa.svc.cluster.local
      weight: 20
```

下面的VirtualService将到达(外部)端口27017的流量转发到端口5555上的内部Mongo服务器。此规则在网格内部不适用，因为网关列表忽略了保留的名称mesh。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
  namespace: bookinfo-namespace
spec:
  hosts:
  - mongosvr.prod.svc.cluster.local # name of internal Mongo service
  gateways:
  - some-config-namespace/my-gateway # can omit the namespace if gateway is in same
                                       namespace as virtual service.
  tcp:
  - match:
    - port: 27017
    route:
    - destination:
        host: mongo.prod.svc.cluster.local
        port:
          number: 5555
```

可以使用hosts字段中的namespace/hostname语法限制可以绑定到网关服务器的虚拟服务集。例如，下面的网关允许gomo1名称空间中的任何虚拟服务绑定到它，同时限制只有gomo2名称空间中的foo.bar.com主机的虚拟服务绑定到它

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
  namespace: some-config-namespace
spec:
  selector:
    app: my-gateway-controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "gomo1/*"
    - "gomo2/foo.bar.com"
```



Gateway根据流入流出方向分为ingress gateway和egresgateway

```
[root@k8s-master1 ~]# kubectl get pods -n istio-system|grep gateway
istio-egressgateway-68f754ccdd-gp78k      1/1     Running       0          4d15h
istio-ingressgateway-6d759478d8-4sfrd     1/1     Running       0          4d15h
```

-  Ingress gateway: 控制外部服务访问网格内服务，配合VirtualService
-  Egress gateway: 控制网格内服务访问外部服务, 配合DestinationRule ServiceEntry使用

#### 21.6.2Gateway vs Kubernetes Ingress 

Kubernetes Ingress集群边缘负载均衡， 提供集群内部服务的 访问入口，仅支持L7负载均衡，功能单一

 Istio 1.0以前，利用Kubernetes Ingress实现网格内服务暴露。 但是Ingress无法实现很多功能：

1. L4-L6负载均衡 
2.  对外mTLS 
3. SNI的支持 
4. 其他istio中已经实现的内部网络功能： Fault Injection， Traffic Shifting，Circuit Breaking， Mirroring

为了解决这些这些问题，Istio在1.0版本设计了新的v1alpha3  API。

- Gateway允许管理员指定L4-L6的设置：端口及TLS设置。
- 对于ingress 的L7设置，Istio允许将VirtualService与Gateway绑定起来。
- 分离的好处：用户可以像使用传统的负载均衡设备一样管理进入网格内部的流量，绑定虚拟IP到虚拟服务器上 。便于传统技术用户无缝迁移到微服务。

#### 21.6.3Gateway原理及实现 

![image-20191230141950923](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191230141950923.png)

Pilot 监听，通过平台设备层感知，再通过聚合层产生统一的接口调用，然后ADS生成 istio中的所有的规则



Gateway 与 普通sidecar均是使用Envoy作为proxy实行流量控制。Pilot为不同类型的proxy生成相应的配置，Gateway的类

型为router，sidecar的类型为sidecar。

启动参数：

```
[root@adm-master1 istio-1.4.2]# kubectl exec -it istio-ingressgateway-6d759478d8-nlzts -n istio-system -- /bin/bash
root@istio-ingressgateway-6d759478d8-nlzts:/# ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 03:46 ?        00:00:06 /usr/local/bin/pilot-agent proxy router --domain istio-system.svc.cluster.local --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --log_output_level=default:info --drainDuration 45s --parentShutdownDuration 1m0s --connectTimeout 10s --serviceCluster istio-ingressgateway --zipkinAddress zipkin.istio-system:9411 --proxyAdminPort 15000 --statusPort 15020 --controlPlaneAuthPolicy NONE --discoveryAddress istio-pilot.istio-system:15010 --trust-domain=cluster.local
root        28     1  0 03:48 ?        00:00:18 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev1.json --restart-epoch 1 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster istio-ingressgateway --service-node router~10.20.3.2~istio-ingressgateway-6d759478d8-nlzts.istio-system~istio-system.svc.cluster.local --max-obj-name-len 189 --local-address-ip-version v4 --log-format [Envoy (Epoch 1)] [%Y-%m-%d %T.%e][%t][%l][%n] %v -l warning --component-log-level misc:error
root        39     0  0 05:33 pts/0    00:00:00 /bin/bash
root        49    39  0 05:33 pts/0    00:00:00 ps -efww

```

Pilot如何得知proxy类型？

Envoy发现服务使用的是xDS协议，Envoy向server端pilot发起请求 **DiscoveryRequest** 时会携带自身信息node，node有一个ID标识， pilot会解析node标识获取proxy类型。

![image-20191231133656322](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231133656322.png)

Envoy的节点标识可以通过静态配置文件指定，也可以通过启动参数 --service-node指定



Gateway对象在Istio中是用CRD声明的

```
[root@adm-master1 istio-1.4.2]# kubectl get crd gateways.networking.istio.io
NAME                           CREATED AT
gateways.networking.istio.io   2019-12-31T03:46:21Z
[root@adm-master1 istio-1.4.2]# kubectl get crd gateways.networking.istio.io -o yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apiextensions.k8s.io/v1beta1","kind":"CustomResourceDefinition","metadata":{"annotations":{},"labels":{"app":"istio-pilot","chart":"istio","heritage":"Tiller","operator.istio.io/component":"Base","operator.istio.io/managed":"Reconcile","operator.istio.io/version":"1.4.0","release":"istio"},"name":"gateways.networking.istio.io"},"spec":{"group":"networking.istio.io","names":{"categories":["istio-io","networking-istio-io"],"kind":"Gateway","plural":"gateways","shortNames":["gw"],"singular":"gateway"},"scope":"Namespaced","subresources":{"status":{}},"validation":{"openAPIV3Schema":{"properties":{"spec":{"description":"Configuration affecting edge load balancer. See more details at: https://istio.io/docs/reference/config/networking/v1alpha3/gateway.html","properties":{"selector":{"additionalProperties":{"format":"string","type":"string"},"type":"object"},"servers":{"description":"A list of server specifications.","items":{"properties":{"bind":{"format":"string","type":"string"},"defaultEndpoint":{"format":"string","type":"string"},"hosts":{"description":"One or more hosts exposed by this gateway.","items":{"format":"string","type":"string"},"type":"array"},"port":{"properties":{"name":{"description":"Label assigned to the port.","format":"string","type":"string"},"number":{"description":"A valid non-negative integer port number.","type":"integer"},"protocol":{"description":"The protocol exposed on the port.","format":"string","type":"string"}},"type":"object"},"tls":{"description":"Set of TLS related options that govern the server's behavior.","properties":{"caCertificates":{"description":"REQUIRED if mode is `MUTUAL`.","format":"string","type":"string"},"cipherSuites":{"description":"Optional: If specified, only support the specified cipher list.","items":{"format":"string","type":"string"},"type":"array"},"credentialName":{"format":"string","type":"string"},"httpsRedirect":{"type":"boolean"},"maxProtocolVersion":{"description":"Optional: Maximum TLS protocol version.","enum":["TLS_AUTO","TLSV1_0","TLSV1_1","TLSV1_2","TLSV1_3"],"type":"string"},"minProtocolVersion":{"description":"Optional: Minimum TLS protocol version.","enum":["TLS_AUTO","TLSV1_0","TLSV1_1","TLSV1_2","TLSV1_3"],"type":"string"},"mode":{"enum":["PASSTHROUGH","SIMPLE","MUTUAL","AUTO_PASSTHROUGH","ISTIO_MUTUAL"],"type":"string"},"privateKey":{"description":"REQUIRED if mode is `SIMPLE` or `MUTUAL`.","format":"string","type":"string"},"serverCertificate":{"description":"REQUIRED if mode is `SIMPLE` or `MUTUAL`.","format":"string","type":"string"},"subjectAltNames":{"items":{"format":"string","type":"string"},"type":"array"},"verifyCertificateHash":{"items":{"format":"string","type":"string"},"type":"array"},"verifyCertificateSpki":{"items":{"format":"string","type":"string"},"type":"array"}},"type":"object"}},"type":"object"},"type":"array"}},"type":"object"}},"type":"object"}},"versions":[{"name":"v1alpha3","served":true,"storage":true}]}}
  creationTimestamp: "2019-12-31T03:46:21Z"
  generation: 1
  labels:
    app: istio-pilot
    chart: istio
    heritage: Tiller
    operator.istio.io/component: Base
    operator.istio.io/managed: Reconcile
    operator.istio.io/version: 1.4.0
    release: istio
  name: gateways.networking.istio.io
  resourceVersion: "2634"
  selfLink: /apis/apiextensions.k8s.io/v1/customresourcedefinitions/gateways.networking.istio.io
  uid: 2b4c5347-82cf-464a-92ab-47e72ee33ce0
spec:
  conversion:
    strategy: None
  group: networking.istio.io
  names:
    categories:
    - istio-io
    - networking-istio-io
    kind: Gateway
    listKind: GatewayList
    plural: gateways
    shortNames:
    - gw
    singular: gateway
  preserveUnknownFields: true
  scope: Namespaced
  versions:
  - name: v1alpha3
    schema:
      openAPIV3Schema:
        properties:
          spec:
            description: 'Configuration affecting edge load balancer. See more details
              at: https://istio.io/docs/reference/config/networking/v1alpha3/gateway.html'
            properties:
              selector:
                additionalProperties:
                  format: string
                  type: string
                type: object
              servers:
                description: A list of server specifications.
                items:
                  properties:
                    bind:
                      format: string
                      type: string
                    defaultEndpoint:
                      format: string
                      type: string
                    hosts:
                      description: One or more hosts exposed by this gateway.
                      items:
                        format: string
                        type: string
                      type: array
                    port:
                      properties:
                        name:
                          description: Label assigned to the port.
                          format: string
                          type: string
                        number:
                          description: A valid non-negative integer port number.
                          type: integer
                        protocol:
                          description: The protocol exposed on the port.
                          format: string
                          type: string
                      type: object
                    tls:
                      description: Set of TLS related options that govern the server's
                        behavior.
                      properties:
                        caCertificates:
                          description: REQUIRED if mode is `MUTUAL`.
                          format: string
                          type: string
                        cipherSuites:
                          description: 'Optional: If specified, only support the specified
                            cipher list.'
                          items:
                            format: string
                            type: string
                          type: array
                        credentialName:
                          format: string
                          type: string
                        httpsRedirect:
                          type: boolean
                        maxProtocolVersion:
                          description: 'Optional: Maximum TLS protocol version.'
                          enum:
                          - TLS_AUTO
                          - TLSV1_0
                          - TLSV1_1
                          - TLSV1_2
                          - TLSV1_3
                          type: string
                        minProtocolVersion:
                          description: 'Optional: Minimum TLS protocol version.'
                          enum:
                          - TLS_AUTO
                          - TLSV1_0
                          - TLSV1_1
                          - TLSV1_2
                          - TLSV1_3
                          type: string
                        mode:
                          enum:
                          - PASSTHROUGH
                          - SIMPLE
                          - MUTUAL
                          - AUTO_PASSTHROUGH
                          - ISTIO_MUTUAL
                          type: string
                        privateKey:
                          description: REQUIRED if mode is `SIMPLE` or `MUTUAL`.
                          format: string
                          type: string
                        serverCertificate:
                          description: REQUIRED if mode is `SIMPLE` or `MUTUAL`.
                          format: string
                          type: string
                        subjectAltNames:
                          items:
                            format: string
                            type: string
                          type: array
                        verifyCertificateHash:
                          items:
                            format: string
                            type: string
                          type: array
                        verifyCertificateSpki:
                          items:
                            format: string
                            type: string
                          type: array
                      type: object
                  type: object
                type: array
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
status:
  acceptedNames:
    categories:
    - istio-io
    - networking-istio-io
    kind: Gateway
    listKind: GatewayList
    plural: gateways
    shortNames:
    - gw
    singular: gateway
  conditions:
  - lastTransitionTime: "2019-12-31T03:46:21Z"
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: "2019-12-31T03:46:21Z"
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
  storedVersions:
  - v1alpha3
```

Gateway配置下发：

遵循 make-before-break原则，杜绝规则更新过程中出现503

![image-20191231134250809](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231134250809.png)

router类型与sidecar类型的proxy最本质的区别是没有 inbound cluster，endpoint， listener 这也从侧面证明Gateway不是流量的终点只是充当一个代理转发。



#### 21.6.4Gateway demo演示

• 控制Ingress HTTP流量（官方文档：https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/）

If you have enabled [automatic sidecar injection](https://istio.io/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection), deploy the `httpbin` service:

```
$ kubectl apply -f samples/httpbin/httpbin.yaml
```

create rule for gateway ：

```
$ kubectl apply -f samples/httpbin/httpbin-gateway.yaml
```

通过端口访问：

```
[root@adm-master1 istio-1.4.2]# kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.10.77.255   <pending>     15020:30123/TCP,80:32013/TCP,443:32268/TCP,15029:32571/TCP,15030:32370/TCP,15031:31230/TCP,15032:30744/TCP,15443:31992/TCP   127m

```

![image-20191231140112065](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231140112065.png)

**理解外部请求如何到达应用** 

1) Client发起请求到特定端口 

2) Load Balancer 监听在这个端口，并转发到后端（或者暴露在nodeport 上面） 

3) 在Istio中，LB将请求转发到IngressGateway 服务 

4) Service将请求转发到IngressGateway pod 

5) Pod 获取Gateway 和 VirtualService配置，获取端口、协 议、证书，创建监听器 （gateway 在80端口监听）

6) Gateway pod 根据路由将请求转发到应用pod（不是 service）



• 利用HTTPS保护后端服务

需要配置密钥证书等：

1. [使用官网的git脚本库来生成证书和密钥](https://github.com/nicholasjackson/mtls-go-example)：

   ```
   $ git clone https://github.com/nicholasjackson/mtls-go-example
   ```

   如果没有git 安装（yum -y install git）

2. 进入目录：

```
[root@adm-master1 ~]# cd mtls-go-example/
[root@adm-master1 mtls-go-example]# ls
generate.sh  intermediate_openssl.cnf  LICENSE  main.go  openssl.cnf  README.md
```

为生成证书`httpbin.example.com`。 gomo 为密码

```
[root@adm-master1 mtls-go-example]# ./generate.sh httpbin.example.com gomo
generate.sh httpbin.example.com 
```

出现提示时，回答`y`所有问题。该命令生成四个目录：`1_root`，`2_intermediate`，`3_application`，和 `4_client`含有在下面的程序使用客户端和服务器证书。（y 一路按下去）

![image-20191231143101504](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231143101504.png)

```
[root@adm-master1 mtls-go-example]# ls
1_root          3_application  generate.sh               LICENSE  openssl.cnf
2_intermediate  4_client       intermediate_openssl.cnf  main.go  README.md

```

将证书移到名为的目录中`httpbin.example.com`：

```
[root@adm-master1 mtls-go-example]# mkdir ../httpbin.example.com && mv 1_root 2_intermediate 3_application 4_client ../httpbin.example.com
```

使用SDS配置TLS入口网关

您可以配置TLS入口网关，以通过秘密发现服务（SDS）从入口网关代理中获取凭据。入口网关代理与入口网关在同一Pod中运行，并监视在与入口网关相同的名称空间中创建的凭据。在入口网关上启用SDS具有以下好处。

- 入口网关可以动态添加，删除或更新其密钥/证书对及其根证书。您不必重新启动入口网关。
- 不需要秘密卷安装。创建`kubernetes` 密钥后，网关代理会捕获该密钥，并将其作为密钥/证书或根证书发送到入口网关。
- 网关代理可以监视多个密钥/证书对。您只需要为多个主机创建机密并更新网关定义。

1. 在入口网关上启用SDS并部署入口网关代理。由于默认情况下禁用此功能，因此您需要启用 `istio-ingressgateway.sds.enabled`安装选项并生成`istio-ingressgateway.yaml`文件：

   ```
   $ istioctl manifest generate \
   --set values.gateways.istio-egressgateway.enabled=false \
   --set values.gateways.istio-ingressgateway.sds.enabled=true > \
   $HOME/istio-ingressgateway.yaml
   $ kubectl apply -f $HOME/istio-ingressgateway.yaml
   ```

2. 设置环境变量`INGRESS_HOST`并`SECURE_INGRESS_PORT`：

   ```
   $ export SECURE_INGRESS_PORT=$(kubectl -n istio-system \
   get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
   $ export INGRESS_HOST=$(kubectl -n istio-system \
   get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   ```

为单个主机配置TLS入口网关

1. 开始`httpbin`示例：

   ```
   $ cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Service
   metadata:
     name: httpbin
     labels:
       app: httpbin
   spec:
     ports:
     - name: http
       port: 8000
     selector:
       app: httpbin
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: httpbin
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: httpbin
         version: v1
     template:
       metadata:
         labels:
           app: httpbin
           version: v1
       spec:
         containers:
         - image: docker.io/citizenstig/httpbin
           imagePullPolicy: IfNotPresent
           name: httpbin
           ports:
           - containerPort: 8000
   EOF
   ```

2. 为入口网关创建一个秘密：

   ```
   $ kubectl create -n istio-system secret generic httpbin-credential \
   --from-file=key=httpbin.example.com/3_application/private/httpbin.example.com.key.pem \
   --from-file=cert=httpbin.example.com/3_application/certs/httpbin.example.com.cert.pem
   ```

   secret名称**不应**以`istio`或开头`prometheus`，并且secret**不应**包含`token`字段。

3. 定义一个网关和`servers:`端口443节，并指定值 `credentialName`是`httpbin-credential`。这些值与secret名称相同。TLS模式的值应为`SIMPLE`。

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: Gateway
   metadata:
     name: mygateway
   spec:
     selector:
       istio: ingressgateway # use istio default ingress gateway
     servers:
     - port:
         number: 443
         name: https
         protocol: HTTPS
       tls:
         mode: SIMPLE
         credentialName: "httpbin-credential" # must be the same as secret
       hosts:
       - "httpbin.example.com"
   EOF
   ```

4. 配置网关的入口流量路由。定义相应的虚拟服务。

   ```yaml
   $ cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
   spec:
     hosts:
     - "httpbin.example.com"
     gateways:
     - mygateway
     http:
     - match:
       - uri:
           prefix: /status
       - uri:
           prefix: /delay
       route:
       - destination:
           port:
             number: 8000
           host: httpbin
   EOF
   ```

5. 发送HTTPS请求以`httpbin`通过HTTPS 访问服务：

   ```yaml
   $ curl -v -HHost:httpbin.example.com \
   --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
   --cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
   https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   ```

   该`httpbin`服务将返回 [418 I’m a Teapot](https://tools.ietf.org/html/rfc7168#section-2.3.3)代码。

6. 删除网关的secret并创建一个新密码以更改入口网关的凭据。

   ```
   $ kubectl -n istio-system delete secret httpbin-credential
   ```

   ```
   $ pushd mtls-go-example
   $ ./generate.sh httpbin.example.com gomo4321
   $ mkdir ../httpbin.new.example.com && mv 1_root 2_intermediate 3_application 4_client ../httpbin.new.example.com
   $ cd
   $ kubectl create -n istio-system secret generic httpbin-credential \
   --from-file=key=httpbin.new.example.com/3_application/private/httpbin.example.com.key.pem \
   --from-file=cert=httpbin.new.example.com/3_application/certs/httpbin.example.com.cert.pem
   ```

7. 使用访问`httpbin`服务`curl`

   ```
   $ curl -v -HHost:httpbin.example.com \
   --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
   --cacert httpbin.new.example.com/2_intermediate/certs/ca-chain.cert.pem \
   https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   ...
   HTTP/2 418
   ...
   -=[ teapot ]=-
   
      _...._
    .'  _ _ `.
   | ."` ^ `". _,
   \_;`"---"`|//
     |       ;/
     \_     _/
       `"""`
   ```

8. 如果您尝试使用`httpbin`先前的证书链进行访问，则尝试现在将失败。

   ```
   $ curl -v -HHost:httpbin.example.com \
   --resolve httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST \
   --cacert httpbin.example.com/2_intermediate/certs/ca-chain.cert.pem \
   https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418
   ...
   * TLSv1.2 (OUT), TLS handshake, Client hello (1):
   * TLSv1.2 (IN), TLS handshake, Server hello (2):
   * TLSv1.2 (IN), TLS handshake, Certificate (11):
   * TLSv1.2 (OUT), TLS alert, Server hello (2):
   * SSL certificate problem: unable to get local issuer certificate
   ```

为多个主机配置 和 配置双向入口网关 请看[官方文档](https://istio.io/docs/tasks/traffic-management/ingress/ingress-sni-passthrough/)

• mTLS 



Istio网格内默认不能访问外部服务，如果需要访问外部服务有三种方式：

1Istio安装时设置： --set global.proxy.includeIPRanges="10.0.0.1/24"  

2创建应用时指定pod annotation traffic.sidecar.istio.io/includeOutboundIPRanges: "127.0.0.1/24,10.96.0.1/24“ 

3创建ServiceEntry

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: cnn
spec:
  hosts:
  - edition.cnn.com
  ports:
  - number: 80
    name: http-port
    protocol: HTTP
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
```

• 控制egress流量

https://istio.io/docs/tasks/traffic-management/egress/egress-gateway/

### 21.7 Istio的灰度发布	

#### 21.7.1典型发布类型对比

蓝绿发布   

全量的切换  如果 v2版本有问题 那就用户全部影响           

![image-20191231145439647](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231145439647.png)       

灰度发布（金丝雀发布）    

对于安全的角度来讲  要比蓝绿发布好很多   

![image-20191231145531842](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231145531842.png)

​      

 A/B Test

####  ![image-20191231145616850](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231145616850.png)

A/B Test主要对特定用户采样后，对收集到的反馈数据做相关对比，然后根据比对结果作出决策。 用来测试应用功能表现的方法，侧重应用的**可用性，受欢迎程度**等

#### 21.7.2 Istio流量治理技术解析 

istio的四个概念：

• **VirtualService 在 Istio 服务网格中定义路由规则，控制路由如何路由到服务上。**

VirtualService 定义了一系列针对指定服务的流量路由规则。  （和destinationRule的区别是D 只对于管理一个 而V 可以是多个service 可以配置一组的配置）

- hosts —— 流量的目标主机 
- gateways —— Gateway名称列表         match 对网格内的都生效

- http —— HTTP 流量规则（HTTPRoute）的列表     HttpMatchRequest 匹配的条件   DestinationWeight动作

- tcp —— tcp流量规则（TCPRoute）的列表 

- tls —— tls和https（TLSRoute）流量规则的列表

• **DestinationRule 是 VirtualService 路由生效后，配置应用与请求的策略集。**

DestinationRule 所定义的策略，决定了经过路由处理之后的流量的访 问策略。

-  host —— 目标服务的名称
-  trafficPolicy —— 流量策略（负载均衡配置、连接池配置和熔断配置）。
-  subsets —— 一个或多个服务版本

• ServiceEntry 是通常用于在 Istio 服务网格之外启用对服务的请求。 

• Gateway 为 HTTP/TCP 流量配置负载均衡器，最常见的是在网格的边缘的操作，以启用 应用程序的入口流量。

要想达到此效果：

![image-20191231151903158](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231151903158.png)

```yaml
apiVersion: …
kind: VirtualService
metadata:
	name: vs-svcb
spec:
	hosts: 
	- svcb
	http:
		route: 
		- destination:
            name: v1
            weight: 20
         - destination:
            name: v2
            weight: 80
```

![image-20191231152056319](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231152056319.png)

```
apiVersion: …
kind: VirtualService
metadata:
	name: vs-svcb
spec:
	hosts: 
	- svcb
	http:
    - match:
        - headers:
            cookie:
                exact: “group=dev”
		route: 
		- destination:
            name: v1
            weight: 20
         - destination:
            name: v2
            weight: 80
```

**复杂灰度场景下的**VirtualService

通过cookie 或者指定的网页url

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: helloworld
spec:
 hosts:
  - helloworld
http:
 - match:
  - headers:
   cookie:
    regex: "^(.*?;)?(email=[^;]*@somecompany-name.com)(;.*)?$"
 route:
  - destination:
    host: helloworld
    subset: v1
   weight: 50
  - destination:
    host: helloworld
    subset: v2
   weight: 50
 - route:
  - destination:
	host: helloworld
	subset: v1
```

灰度发布的存在形式以及灰度发布的流程：

![image-20191231152920211](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231152920211.png)

![image-20191231153733572](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231153733572.png)

#### 21.7.3 智能灰度发布介绍 

目标：细粒度控制的自动化的持续交付 

特点： 1用户细分   2 流量管理     3 关键指标可观测    4 发布流程自动化

![image-20191231154004183](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20191231154004183.png)

**自适应灰度发布参数** 

• 负载健康状态        • 请求成功率            • 平均请求时延            • 流量权重步长            • 回滚门限值

