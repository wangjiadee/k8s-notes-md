## 17.K8S存储之Ceph分布式存储系统	

### 17.1	Ceph系统架构及核心组件

### 17.2	Ceph三大客户端介绍 

### 17.3Ceph 核心概念(Crush、Pool. PG、OSD、Object) 

### 17.4部署Ceph集群 

#### 17.4.1ceph版本选择 

#### 17.4.2安装ceph-deploy工具 

#### 17.4.3ceph.conf配置文件

#### 17.4.4 MON/OSD/MDS/MGR组件部署 

#### 17.4.5 cephfs文件系统部署 

### 17.5 RBD客户端安装与使用

#### 17.5.1 RBD的工作厕与配置 

#### 17.5.2RBD的常用命令 

#### 17.5.3客户挂载RBD的几种方式 

### 17.6CephFS客户端安装与使用 

#### 17.6.1CephFS的工作原理与配置 

#### 17.6.2挂载CephFS的几种方式 

### 17.7 K8S使用Ceph作为存储 

#### 17.7.1PV,PVC 概述 

#### 17.7.2 PV自动蝠 	

#### 17.7.3 Pod使用RBD作为持久数据卷 

#### 17.7.4 Pod使用CephFS作为持久数据卷 

### 17.8Ceph监控 

#### 17.8.1内置Dashboard 

#### 17.8.2Prometheus+Grafana 监控 Ceph 

### 17.9	Ceph 日常运维管理