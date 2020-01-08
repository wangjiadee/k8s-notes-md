## 19、基于kubernetes构建企业Jenkins CI/CD平台

持续集成（CI）： 代码合并，构建，部署，测试都在一起，不断的执行这个过程，并对结果进行反馈

持续部署（CD）：部署到测试环境、预生产环境、生产环境

持续交付（CD）：将最终产品发布到生产环境，给用户使用

~非容器化CI/CD 

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml14624\wps15.jpg) 

缺点：测试与生产环境 相似度程度不高

~容器化CI/CD

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml14624\wps16.jpg) 

镜像仓库 起到中转的作用 以及对镜像业务镜像集中管理

同时可以部署到任何的环境中（对环境的影响小）

~基于k8s化CI/CD

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml14624\wps17.jpg) 

### 19.1项目发布方案概述（蓝绿 灰度发布 和滚动发布）

#### 19.1.1  蓝绿发布Blue/Green Deployment

蓝绿部署是不停老版本，部署新版本然后进行测试，确认OK，将流量切到新版本，然后老版本同时也升级到新版本。

特点：蓝绿部署无需停机，并且风险较小

###### 第一步、部署版本1（老版本）的应用（一开始的状态）

所有外部请求的流量都打到这个版本上。

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml14624\wps18.jpg) 

###### 第二步、部署版本2的应用

版本2的代码与版本1不同(新功能、Bug修复等)。

###### 第三步、将流量从版本1切换到版本2。

 

![img](file:///C:\Users\WANGJI~1\AppData\Local\Temp\ksohtml14624\wps19.jpg) 

###### 第四步、如版本2测试正常，就删除版本1正在使用的资源（例如实例），从此正式用版本2。

在部署的过程中，我们的应用始终在线。并且，新版本上线的过程中，并没有修改老版本的任何内容，在部署期间，老版本的状态不受影响。这样风险很小，并且，只要老版本的资源不被删除，理论上，我们可以在任何时间回滚到老版本。
当你切换到蓝色环境时，需要妥当处理未完成的业务和新的业务。如果你的数据库后端无法处理，会是一个比较麻烦的问题；

- 可能会出现需要同时处理“微服务架构应用”和“传统架构应用”的情况，如果在蓝绿部署中协调不好这两者，还是有可能会导致服务停止。
- 需要提前考虑数据库与应用部署同步迁移 /回滚的问题。
- 蓝绿部署需要有基础设施支持。
- 在非隔离基础架构（ VM 、 Docker 等）上执行蓝绿部署，蓝色环境和绿色环境有被摧毁的风险。

#### 19.1.2滚动发布Rolling update

滚动发布：一般是取出一个或者多个服务器停止服务，执行更新，并重新将其投入使用。周而复始，直到集群中所有的实例都更新成新版本。（k8s 就是使用的这种方法）

特点：这种部署方式相对于蓝绿部署，更加节约资源——它不需要运行两个集群、两倍的实例数。我们可以部分部署，例如每次只取出集群的20%进行升级。

(1) 没有一个确定OK的环境。使用蓝绿部署，我们能够清晰地知道老版本是OK的，而使用滚动发布，我们无法确定。

(2) 修改了现有的环境。

(3) 如果需要回滚，很困难。举个例子，在某一次发布中，我们需要更新100个实例，每次更新10个实例，每次部署需要5分钟。当滚动发布到第80个实例时，发现了问题，需要回滚，这个回滚却是一个痛苦，并且漫长的过程。

(4) 有的时候，我们还可能对系统进行动态伸缩，如果部署期间，系统自动扩容/缩容了，我们还需判断到底哪个节点使用的是哪个代码。尽管有一些自动化的运维工具，但是依然令人心惊胆战。

(5) 因为是逐步更新，那么我们在上线代码的时候，就会短暂出现新老版本不一致的情况，如果对上线要求较高的场景，那么就需要考虑如何做好兼容的问题。

#### 19.1.3灰度发布/金丝雀部署

灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式。AB test就是一种灰度发布方式，让一部分用户继续用A，一部分用户开始用B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度，而我们平常所说的金丝雀部署也就是灰度发布的一种方式。


优点:提前获得目标用户的使用反馈；

根据反馈结果，做到查漏补缺；

发现重大问题，可回滚“旧版本”；

补充完善产品不足；

快速验证产品的 idea。

### 19.2 jenkins的安装+使用git作为代码仓库&使用harbor作为镜像仓库

| 主机    | ip              | 角色                   |
| ------- | --------------- | ---------------------- |
| jenkins | 192.168.208.196 | **docker, jenkins**    |
| volumes | 192.168.208.195 | **docker, harbor,git** |
| **win** | 本机            | **浏览器**             |

**分别在jenkins 和volumes上安装 docker-ce**

```
    1  systemctl stop firewalld
    2  systemctl disable firewalld
    3  sed -i s/SELINUX=enforcing/SELINUX=disabled/ /etc/selinux/config
    4  setenforce 0
    5  getenforce 
    6  yum -y install wget curl vim 
    7  wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
    8  yum install -y yum-utils device-mapper-persistent-data lvm2
    9  yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   10  yum makecache fast
   11  yum -y install docker-ce

```

**volumes 上安装 harbor**（详情看docker篇章）

```
[root@volumes ~]# ls
anaconda-ks.cfg docker-compose
chmod +x docker-composmv docker-compose /usr/bin/
[root@volumes~]# ls
anaconda-ks.cfg harbor-offline-installer-v1.8.1.tgz
tar zxvf harbor-offline-installer-v1.8.1.tgz
cd harbor/
vim harbor.yml
hostname: 192.168.208.195 # IP 地址或者域名访问
harbor_admin_password: gomo # Web 登录密码
[root@volumes harbor]# ./install.sh
[root@volumes harbor]#docker-compose ps
[root@volumes ~]# vim /etc/docker/daemon.json
{
"insecure-registries": ["192.168.208.195"]
}
```

![image-20200107103050361](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107103050361.png)

**安装 git 代码管理版本仓库**

```
[root@volumes home]# useradd git
[root@volumes home]# echo git |passwd --stdin git
Changing password for user git.
passwd: all authentication tokens updated successfully.
[root@volumes home]# yum install -y git
[root@volumes home]# su - git
[git@volumes ~]$ mkdir tomcat-java-demo.git
[git@volumes ~]$ cd tomcat-java-demo.git/
# 把文件夹初始化为一个 git 代码仓库
[git@volumes tomcat-java-demo.git]$ git --bare init
Initialized empty Git repository in /home/git/tomcat-java-demo.git/
[git@volumes tomcat-java-demo.git]$ ls
branches  config  description  HEAD  hooks  info  objects  refs
[git@volumes tomcat-java-demo.git]$ pwd
/home/git/tomcat-java-demo.git

#jenkins去拉取 git 代码仓库里的代码
[root@jenkins ~]# yum -y install git
[root@jenkins ~]# git clone https://github.com/lizhenliang/tomcat-java-demo
[root@jenkins ~]# ls
anaconda-ks.cfg  tomcat-java-demo
#因为代码是从 github 上拉取的，需要修改隐藏的 .git/config 文件， 修改 url 为私有 git 仓库地址
[root@jenkins ~]# vim tomcat-java-demo/.git/config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = git@192.168.208.195:/home/git/tomcat-java-demo.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
[user]
        name = git
        email = git@gomo.com

```

\#提交代码到本地暂存区

```
[root@jenkins ~]# cd tomcat-java-demo/
[root@jenkins tomcat-java-demo]# git add .
#提交代码到本地代码仓库
[root@jenkins tomcat-java-demo]# git commit -m 'all'
# On branch master
nothing to commit, working directory clean
[root@jenkins tomcat-java-demo]# git push origin master
#推送代码到中央代码仓库，至于 origin 和 master 是什么，请看 .git/config 文件

[root@jenkins ~]# git push origin master
fatal: Not a git repository (or any of the parent directories): .git
[root@jenkins ~]# cd tomcat-java-demo/
[root@jenkins tomcat-java-demo]# git add .
[root@jenkins tomcat-java-demo]# git commit -m 'all'
# On branch master
nothing to commit, working directory clean
[root@jenkins tomcat-java-demo]# git push origin master
The authenticity of host '192.168.208.195 (192.168.208.195)' can't be established.
ECDSA key fingerprint is SHA256:gn8cm1tRrOixZ12cmZdelGhTzK+a+5ucyhqb2re2QIo.
ECDSA key fingerprint is MD5:92:cc:ff:21:44:bd:07:50:ff:72:42:ce:1c:20:72:31.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.208.195' (ECDSA) to the list of known hosts.
git@192.168.208.195's password:  git
Counting objects: 546, done.
Compressing objects: 100% (313/313), done.
Writing objects: 100% (546/546), 5.08 MiB | 0 bytes/s, done.
Total 546 (delta 208), reused 546 (delta 208)
To git@192.168.208.195:/home/git/tomcat-java-demo.git
 * [new branch]      master -> master

```

**在 jenkins上部署 jenkins,安装 jdk,maven**

```
root@jenkins home]# ls
apache-maven-3.5.0-bin.tar.gz  apache-tomcat-8.0.46.tar.gz  jdk-8u45-linux-x64.tar.gz  jenkins.war
[root@jenkins home]# tar xf jdk-8u45-linux-x64.tar.gz
[root@jenkins home]# ls
apache-maven-3.5.0-bin.tar.gz  apache-tomcat-8.0.46.tar.gz  jdk1.8.0_45  jdk-8u45-linux-x64.tar.gz  jenkins.war
[root@jenkins home]# mv jdk1.8.0_45/ /usr/local/jdk
[root@jenkins home]# tar xf apache-maven-3.5.0-bin.tar.gz
[root@jenkins home]# mv apache-maven-3.5.0 /usr/local/maven
[root@jenkins home]# vim /etc/profile
    JAVA_HOME=/usr/local/jdk
    PATH=$PATH:$JAVA_HOME/bin
    export JAVA_HOME PATH
[root@jenkins home]# source /etc/profile
[root@jenkins home]# java -version
java version "1.8.0_45"
Java(TM) SE Runtime Environment (build 1.8.0_45-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.45-b02, mixed mode)


#安装 jenkins
[root@jenkins home]# tar xf apache-tomcat-8.0.46.tar.gz
[root@jenkins home]# mv apache-tomcat-8.0.46 /usr/local/jenkins_tomcat
[root@jenkins home]# cd /usr/local/jenkins_tomcat
[root@jenkins jenkins_tomcat]# ls
bin  conf  lib  LICENSE  logs  NOTICE  RELEASE-NOTES  RUNNING.txt  temp  webapps  work
#删除 webapps 下的所有内容(默认都是一些测试页面，这里用不到，所以删除)
[root@jenkins jenkins_tomcat]# cd webapps
[root@jenkins webapps]# rm -rf *
[root@jenkins webapps]# mv /home/jenkins.war ROOT.war
[root@jenkins webapps]# cd ../bin
[root@jenkins bin]# ./startup.sh
Using CATALINA_BASE:   /usr/local/jenkins_tomcat
Using CATALINA_HOME:   /usr/local/jenkins_tomcat
Using CATALINA_TMPDIR: /usr/local/jenkins_tomcat/temp
Using JRE_HOME:        /usr/local/jdk
Using CLASSPATH:       /usr/local/jenkins_tomcat/bin/bootstrap.jar:/usr/local/jenkins_tomcat/bin/tomcat-juli.jar
Tomcat started.

```

到此，jenkins 部署好了，可以通过浏览器 http://192.168.208.196:8080 进行访问了

![image-20200107105443239](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105443239.png)

\# jenkins 登陆页面的初始密

```
[root@jenkins bin]# cat /root/.jenkins/secrets/initialAdminPassword
f0151fdc31ad4efe896222f2fc530920
```

![image-20200107105602362](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105602362.png)



![image-20200107105645796](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105645796.png)

![image-20200107105706538](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105706538.png)

![image-20200107105723304](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105723304.png)

![image-20200107105736089](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105736089.png)

**jenkins 插件安装**

登陆后，系统管理-->插件管理-->advanced 标签页-->拉到最下面 Update site 里， 默认的地址， https 修改成 http。(在上面 jenkins 提示 offline,所以连接官方安装插件 会有问题，这里 https 修改成 http 后，大多数情况，能解决插件安装连接不上的问题)， submit 后，点 check now

![image-20200107105948718](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107105948718.png)

```
如果 Check now 失败，把 URL 改成
http://mirror.esuni.jp/jenkins/updates/update-center.json
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
http://ftp.yz.yamagata-u.ac.jp/pub/misc/jenkins/updates/update-center.json
【详细的 Jenkins 的镜像地址查询：http://mirrors.jenkins-ci.org/status.html】
```

安装插件: Available 标签里选择 pipeline,SCM to job,Git ,-> intall without restart

![image-20200107111613359](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111613359.png)

![image-20200107111624190](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111624190.png)

![image-20200107111636590](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111636590.png)

```
依赖错误，可以手动下载依赖
http://updates.jenkins-ci.org/download/plugins/
重启 jenkins
http://192.168.28.161:8080/restart
```



**Jenkins 里创建 job**

```
[root@jenkins ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:q6WIGqSICX7HIExB9CHP4ScPhn4pSkWOxlzKg0+a9PU root@jenkins
The key's randomart image is:
+---[RSA 2048]----+
|o= =             |
|= # o            |
|.%.@ o           |
|*== B .          |
|+B.= . ES        |
|X.+ o    .       |
|B. . o  o        |
| ..... +         |
|... . o          |
+----[SHA256]-----+
[root@jenkins ~]# ssh-copy-id git@192.168.208.195
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
git@192.168.208.195's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'git@192.168.208.195'"
and check to make sure that only the key(s) you wanted were added.

[root@jenkins ~]# cat .ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEArFY/AykxS9k90B29F/1D5hjY4N98qiC9BWDickWfL7zMdPQv
3aJQT77SUgbHOVpix9yC/OjTLPGfT3I7mb1a1xOc9PU2Y62/r4XEVgFQid0v7o/A
7jXjuBSAleppWb1ZD4pSOO8LNSbFEivl8bP4SnJ+sPZhVCb4a22u+DBF13vSdYaz
rnDn/2rM+qlvWJLINgVsXkiqVI3fzkQXY/2Pbao32Ti/NCo8DdzpGncgbvlTEaUh
KRz0qK5uWty8qRr6wyvX82UwW8WtaMhXCe2OLlBpuI1amw8/jOWI+RpcGnMBfsyI
1Sy8ug1aZcQk+drL3Bnz3gsjeYDhM8fz47792QIDAQABAoIBAQCHd5Q4q9ywPqg0
O+w0O0VwTf/NZF/ea7Wp0KqwIMItCD+/f2NQ2RJAXUN+bw2Tq9USPehJXcsB/Ty5
epYXF52cizJJ66dBW4beNkxLPuVMOa4/3IhPt9S1EoixT35YqFqluJlBX8ZzlXI8
An3SLSHzg2TLPiDrwWZtK97qASglZUk/oOI0F7rIMXsRY+tq5TTIgy6D8ggKNFyO
DbgkybkHjRABSUtXyFv3VBcD6WPdO6oH35Soc1o1J9FjiAuA0zzCq41xq5KX+jF5
EBOwH/zFeYEn4GXJIJZqQt1lKYs1+fkQh80q5s02W3MqBQ4s40+5syRJ2y0auJMn
UgOpaUZxAoGBANyorRX3Tst+r0XPrvibbhagK9N2+GuiefcMKgnyNXaxS3zBeJYw
UvEfZHCKtFyfBVP13boAkFgrmcgD9Ykyq+ptqtSg5bkd9Am9QtK4+ZiyYRhw9Q/u
SIK5EKA2Za1eLIxzTixVLwpK9m6mlUygNZbuu/nuv/9MnUu319DXbZ+rAoGBAMfw
TPR+sSJd9z4+XSF9rAUkHLU7q4qgNMCxg0WqOVQkjnp1vnevIBWuhmSvKjo6dUhC
ZH2jtYaFidBdBKn8nKyjx39UrerzGJ0xlhxWQwbFJs1JkyzKrg1gR0j3dykQF40u
80+gO292JId/iONum8gnS5R0wKQtBRFhRtjKD+SLAoGADHl4t4owqS5zSDYShTl8
QskxURYjuyoHTSEh60gHH7usMdRaNdtrhPgqXHZq9eWDjpiSvWY0wtdMLVOT+Pql
X25tvvGNqyZ3WmmZsoIEkk5bUN9p60mkTceamgQZQXDWgeYu4DC8pQ9R2TWPsTJJ
dUvv0pRdxFgXeGVfTQ4ww1sCgYBhQXO9jo8Nf5XP8jgNHXt6uLk6Mz9bXFiszuxj
C819L5ca3IF86HP79/wpp8crsdnw/1Kwhty7BeQmtciaA2YW2EgnmQJMglmbxU4W
lKNf/LDGNR7hL+oAWa/zP2T4VXqPU6JJPlELA/X67z+gGeKvNuYd3bkDY17OuHnk
5E1cxwKBgBxZlBJPGhLkccavb4TNEXAzLVBsWpBegsCPDYDKq1QjWoag6WmUIWrH
+T6IF8oWQ7yqRM9PEm4yQwe10tc9Ikji1ACfX3EKVYFLRF5NPRVkT/oiMJryvaWV
8m0dpUWinLQSeLAqY+2PCHLc0mzdBtWMRFcuNZOXfyMYOQwOVTcc
-----END RSA PRIVATE KEY-----

```



![image-20200107111844117](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111844117.png)



![image-20200107111903525](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111903525.png)



记录ID：9eb47da4-887d-48f5-a066-0c92daf937db

![image-20200107111924972](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107111924972.png)



![image-20200107113609968](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107113609968.png)

![image-20200107114620656](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107114620656.png)





![image-20200107114628793](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107114628793.png)



![image-20200107114633225](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107114633225.png)

```
jenkins 机器需要往 Harbor 镜像仓库推送镜像，需要将 Harbor 服务器的地址设置成为
jenkins 所在机器的 docker 服务可信任 Harbor 地址
[root@jenkins tomcat-java-demo]# cat /etc/docker/daemon.json 
{"registry-mirrors": ["http://f1361db2.m.daocloud.io"],
 "insecure-registries": ["192.168.208.195"]
}

```

![image-20200107114725349](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107114725349.png)



![image-20200107114730584](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107114730584.png)

### 19.3在k8s中部署jenkins

1.可以使用helm 一键部署

```
helm install --name gomo-jenkins --set Persistence.Enabled=false stable/jenkins
```

**说明** **Persistence.Enabled**设置为false，会禁用PersistentVolume。在您的生产环境下，可以创建一个NAS作为PersistentVolume。

2 yaml 方式：

再次之前 创建我们需要添加创建⼀个 namespace：

```
$ kubectl create namespace kube-ops
```

我们这⾥使⽤⼀个名为 jenkins/jenkins:lts 的镜像，这是 jenkins 官⽅的 Docker 镜像，然后也有⼀些环境变量，当然我们也可以根据⾃⼰的需求来定制⼀个镜像，⽐如我们可以将⼀些插件打包在⾃定义的镜像当中，可以参考⽂档：https://github.com/jenkinsci/docker，我们这⾥使⽤默认的官⽅镜像就⾏。最后为了⽅便我们测试，我们这⾥通过 NodePort 的形式来暴露 Jenkins 的 web 服务，固定为31234 端⼝，另外还需要暴露⼀个 agent 的端⼝，这个端⼝主要是⽤于 Jenkins 的 master 和 slave 之间通信 使⽤的。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: jenkins2
  name: jenkins2
  namespace: kube-ops
spec:
  progressDeadlineSeconds: 2147483647
  replicas: 1
  revisionHistoryLimit: 2147483647
  selector:
    matchLabels:
      app: jenkins2
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: jenkins2
    spec:
      containers:
      - env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              divisor: 1Mi
              resource: limits.memory
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0
            -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
            -Duser.timezone=Asia/Shanghai
        image: 192.168.208.195/library/jenkins:lts
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 12
          httpGet:
            path: /login
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: jenkins
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        readinessProbe:
          failureThreshold: 12
          httpGet:
            path: /login
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/jenkins_home
          name: jenkinshome
          subPath: jenkins2
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext:
        fsGroup: 1000
      serviceAccount: jenkins2
      serviceAccountName: jenkins2
      terminationGracePeriodSeconds: 10
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: opspvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins2
  namespace: kube-ops
  labels:
    app: jenkins2
spec:
  selector:
    app: jenkins2
  type: NodePort
  ports:
  - name: web
    port: 8080
    targetPort: web
    nodePort: 31234
  - name: agent
    port: 50000
    targetPort: agent


---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: opspv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  local:
    path: /var/lib/docker/jenkins

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: opspvc
  namespace: kube-ops
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi


---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins2
  namespace: kube-ops

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins2
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins2
  namespace: kube-ops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins2
subjects:
  - kind: ServiceAccount
    name: jenkins2
    namespace: kube-ops


```

 

如果是1.13-的k8s deployment 的老文件：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins2
  namespace: kube-ops
spec:
  template:
    metadata:
      labels:
        app: jenkins2
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccount: jenkins2
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts
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
          - name: jenkinshome
            subPath: jenkins2
            mountPath: /var/jenkins_home
        env:
        - name: LIMITS_MEMORY
          valueFrom:
            resourceFieldRef:
              resource: limits.memory
              divisor: 1Mi
        - name: JAVA_OPTS
          value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
      securityContext:
        fsGroup: 1000
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: opspvc
```



等到服务启动成功后，我们就可以根据任意节点的 IP:31234 端⼝就可以访问 jenkins 服务了，可以根 据提示信息进⾏安装配置即可：![image-20200107134301710](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107134301710.png)

```
[root@adm-master1 jenkins]# kubectl exec jenkins2-9f55b98b6-54fq9 -it -n kube-ops /bin/sh
$ cat /var/jenkins_home/secrets/initialAdminPassword
```

然后选择安装推荐的插件即可。(也可以自定义 参照19.2)

![image-20200107134344824](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107134344824.png)



![image-20200107131711780](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107131711780.png)

安装完成创建管理员用户：

![image-20200107135532846](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107135532846.png)

- ## 优点

Jenkins 安装完成了，接下来我们不用急着就去使用，我们要了解下在 Kubernetes 环境下面使用 Jenkins 有什么好处。

我们知道持续构建与发布是我们日常工作中必不可少的一个步骤，目前大多公司都采用 Jenkins 集群来搭建符合需求的 CI/CD 流程，然而传统的 Jenkins Slave 一主多从方式会存在一些痛点，比如：

- 主 Master 发生单点故障时，整个流程都不可用了
- 每个 Slave 的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导致管理起来非常不方便，维护起来也是比较费劲
- 资源分配不均衡，有的 Slave 要运行的 job 出现排队等待，而有的 Slave 处于空闲状态
- 资源有浪费，每台 Slave 可能是物理机或者虚拟机，当 Slave 处于空闲状态时，也不会完全释放掉资源。

正因为上面的这些种种痛点，我们渴望一种更高效更可靠的方式来完成这个 CI/CD 流程，而 Docker 虚拟化容器技术能很好的解决这个痛点，又特别是在 Kubernetes 集群环境下面能够更好来解决上面的问题，下图是基于 Kubernetes 搭建 Jenkins 集群的简单示意图： ![k8s-jenkins](https://www.qikqiak.com/k8s-book/docs/images/k8s-jenkins-slave.png)

从图上可以看到 Jenkins Master 和 Jenkins Slave 以 Pod 形式运行在 Kubernetes 集群的 Node 上，Master 运行在其中一个节点，并且将其配置数据存储到一个 Volume 上去，Slave 运行在各个节点上，并且它不是一直处于运行状态，它会按照需求动态的创建并自动删除。

这种方式的工作流程大致为：当 Jenkins Master 接受到 Build 请求时，会根据配置的 Label 动态创建一个运行在 Pod 中的 Jenkins Slave 并注册到 Master 上，当运行完 Job 后，这个 Slave 会被注销并且这个 Pod 也会自动删除，恢复到最初状态。

那么我们使用这种方式带来了哪些好处呢？

- **服务高可用**，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。
- **动态伸缩**，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
- **扩展性好**，当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。

- ## 配置

接下来我们就需要来配置 Jenkins，让他能够动态的生成 Slave 的 Pod。

第1步. 我们需要安装**kubernetes plugin**， 点击 Manage Jenkins -> Manage Plugins -> Available -> Kubernetes plugin 勾选安装即可。（截止2020.1.7号）      jenkins中的kubernetes plugin 已经改名为kubernetes 安装完后默认就有了Kubernetes plugin。![image-20200107145504838](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107145504838.png)

![image-20200107145356906](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107145356906.png)

第2步. 安装完毕后，点击 Manage Jenkins —> Configure System —> (拖到最下方)Add a new cloud —> 选择 Kubernetes，然后填写 Kubernetes 和 Jenkins 配置信息。

![image-20200107150130215](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107150130215.png)

![image-20200107151513972](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151513972.png)

注意 namespace，我们这⾥填 kube-ops，然后点击Test Connection，如果出现 Connection test successful 的提示信息证明 Jenkins 已经可以和 Kubernetes 系统正常通信了，然后下⽅的 JenkinURL 地址：http://jenkins2.kube-ops.svc.cluster.local:8080，这⾥的格式为：服务 名.namespace.svc.cluster.local:8080，根据上⾯创建的jenkins 的服务名填写，我这⾥是之前创建的名 为jenkins，如果是⽤上⾯我们创建的就应该是jenkins2

![image-20200107151528323](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151528323.png)

另外需要注意我们这⾥需要在下⾯挂载两个主机⽬录，⼀个是 /var/run/docker.sock，该⽂件是⽤于 Pod 中的容器能够共享宿主机的 Docker，这就是⼤家说的 docker in docker 的⽅式，Docker ⼆进制 ⽂件我们已经打包到上⾯的镜像中了，另外⼀个⽬录下 /root/.kube ⽬录，我们将这个⽬录挂载到容器的 /home/jenkins/.kube ⽬录下⾯这是为了让我们能够在 Pod 的容器中能够使⽤ kubectl ⼯具来访问我 们的 Kubernetes 集群，⽅便我们后⾯在 Slave Pod 部署 Kubernetes 应⽤。

![image-20200107151544050](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151544050.png)

注意：由于新版本的 Kubernetes 插件变化较多，如果你使用的 Jenkins 版本在 2.176.x 版本以上，注意将上面的镜像替换成`cnych/jenkins:jnlp6`，否则使用会报错，配置如下图所示：

![kubernetes slave image config](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/jenkins-slave-new.png)

![kubernetes plugin config3](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/jenkins-slave-volume.png)



发现启动 Jenkins Slave Pod 的时候，出现 Slave Pod 连接不上，然后尝试100次连接之后销毁 Pod，然后会再创建一个 Slave Pod 继续尝试连接，无限循环，类似于下面的信息： 

![img](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/slave-pod-reconnect-100-times.png)

如果出现这种情况的话就需要将 Slave Pod 中的运行命令和参数两个值给清空掉 ![img](https://bxdc-static.oss-cn-beijing.aliyuncs.com/images/clean-slave-pod-cmd-args.png)

另外还有⼏个参数需要注意，如下图中的Time in minutes to retain slave when idle，这个参数表示的 意思是当处于空闲状态的时候保留 Slave Pod 多⻓时间，这个参数最好我们保存默认就⾏了，如果你 设置过⼤的话，Job 任务执⾏完成后，对应的 Slave Pod 就不会⽴即被销毁删除。

![image-20200107151553697](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151553697.png)

![image-20200107151806562](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151806562.png)

到这⾥我们的 Kubernetes Plugin 插件就算配置完成了。

测试：

Kubernetes 插件的配置⼯作完成了，接下来我们就来添加⼀个 Job 任务，看是否能够在 Slave Pod 中 执⾏，任务执⾏完成后看 Pod 是否会被销毁。在 Jenkins ⾸⻚点击create new jobs，创建⼀个测试的任务，输⼊任务名称，然后我们选择 Freestyle project 类型的任务：

![image-20200107151903639](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151903639.png)

注意在下⾯的 Label Expression 这⾥要填⼊gomo-jnlp，就是前⾯我们配置的 Slave Pod 中的 Label，这两个地⽅必须保持⼀致

![image-20200107151936539](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107151936539.png)

然后往下拉，在 Build 区域选择Execute shell然后输⼊我们测试命令，最后点击保存

![image-20200107152113688](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107152113688.png)



现在我们直接在⻚⾯点击做成的 Build now 触发构建即可，然后观察 Kubernetes 集群中 Pod 的变化

![image-20200107154658391](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107154658391.png)

```

[root@adm-master1 ~]# kubectl get pods -n kube-ops -o wide -w
NAME                       READY   STATUS    RESTARTS   AGE    IP            NODE          NOMINATED NODE   READINESS GATES
jenkins2-9f55b98b6-54fq9   1/1     Running   0          4h9m   10.20.2.35    adm-worker2   <none>           <none>
jnlp-c1p3t                 1/1     Running   0          17s    10.20.3.135   adm-worker1   <none>           <none>


```



### 19.4Jenkins Pipeline 

要实现在 Jenkins 中的构建⼯作，可以有多种⽅式，我们这⾥采⽤⽐较常⽤的 Pipeline 这种⽅式。 Pipeline，简单来说，就是⼀套运⾏在 Jenkins 上的⼯作流框架，将原来独⽴运⾏于单个或者多个节点 的任务连接起来，实现单个任务难以完成的复杂流程编排和可视化的⼯作。



Jenkins Pipeline 有⼏个核⼼概念：

- Node：节点，⼀个 Node 就是⼀个 Jenkins 节点，Master 或者 Agent，是执⾏ Step 的具体运⾏ 环境，⽐如我们之前动态运⾏的 Jenkins Slave 就是⼀个 Node 节点 
- Stage：阶段，⼀个 Pipeline 可以划分为若⼲个 Stage，每个 Stage 代表⼀组操作，⽐如： Build、Test、Deploy，Stage 是⼀个逻辑分组的概念，可以跨多个 Node 
- Step：步骤，Step 是最基本的操作单元，可以是打印⼀句话，也可以是构建⼀个 Docker 镜像， 由各类 Jenkins 插件提供，⽐如命令：sh 'make'，就相当于我们平时 shell 终端中执⾏ make 命令 ⼀样。



Pipeline ⽀持两种语法：Declarative(声明式)和 Scripted Pipeline(脚本式)语法

Pipeline 也有两种创建⽅法：可以直接在 Jenkins 的 Web UI 界⾯中输⼊脚本；也可以通过创建⼀ 个 Jenkinsfile 脚本⽂件放⼊项⽬源码库中





创建⼀个简单的 Pipeline：直接在 Jenkins 的 Web UI 界⾯中输⼊脚本运⾏。

新建 Job：在 Web UI 中点击 New Item -> 输⼊名称：pipeline-demo -> 选择下⾯的 Pipeline -> 点 击 OK 配置：在最下⽅的 Pipeline 区域输⼊如下 Script 脚本，然后点击保存。

![image-20200107163451881](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107163451881.png)

![image-20200107163631095](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107163631095.png)

构建：

![image-20200107163759192](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107163759192.png)

```
node('gomo-jnlp') {
  stage('Clone') {
    echo "1.Clone Stage"
  }
  stage('Test') {
    echo "2.Test Stage"
  }
  stage('Build') {
    echo "3.Build Stage"
  }
  stage('Deploy') {
    echo "4. Deploy Stage"
  }
}
```

![image-20200107164020820](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107164020820.png)

![image-20200107174206829](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200107174206829.png)

再一次执行 然后-w 查看 

```
[root@adm-master1 ~]# kubectl get pods -n kube-ops -o wide -w
NAME                       READY   STATUS    RESTARTS   AGE     IP           NODE          NOMINATED NODE   READINESS GATES
jenkins2-9f55b98b6-54fq9   1/1     Running   3          5h21m   10.20.2.52   adm-worker2   <none>           <none>
jnlp-017z7                 0/1     Pending   0          0s      <none>       <none>        <none>           <none>
jnlp-017z7                 0/1     Pending   0          0s      <none>       adm-worker1   <none>           <none>
jnlp-017z7                 0/1     ContainerCreating   0          0s      <none>       adm-worker1   <none>           <none>
jnlp-017z7                 1/1     Running             0          10s     10.20.3.141   adm-worker1   <none>           <none>
jnlp-017z7                 1/1     Terminating         0          44s     10.20.3.141   adm-worker1   <none>           <none>
jnlp-017z7                 0/1     Terminating         0          47s     <none>        adm-worker1   <none>           <none>
jnlp-017z7                 0/1     Terminating         0          48s     <none>        adm-worker1   <none>           <none>
jnlp-017z7                 0/1     Terminating         0          48s     <none>        adm-worker1   <none>           <none>

```

### 19.5利用pipeline部署 Kubernetes 应用

要部署 Kubernetes 应⽤，我们就得对我们之前部署应⽤的流程要⾮常熟悉才⾏，我们之前的流程是怎样的：

- 编写代码
- 测试
- 编写 Dockerfile
- 构建打包 Docker 镜像
- 推送 Docker 镜像到仓库
- 编写 Kubernetes YAML ⽂件
- 更改 YAML ⽂件中 Docker 镜像 TAG
- 利⽤ kubectl ⼯具部署应⽤

我们之前在 Kubernetes 环境中部署⼀个原⽣应⽤的流程应该基本上是上⾯这些流程吧？现在我们就需 要把上⾯这些流程放⼊ Jenkins 中来⾃动帮我们完成(当然编码除外)，从测试到更新 YAML ⽂件属于 CI 流程，后⾯部署属于 CD 的流程。如果按照我们上⾯的示例，我们现在要来编写⼀个 Pipeline 的脚 本，应该怎么编写呢？

```
node {
  stage('Clone') {
    echo "1.Clone Stage"
  }
  stage('Test') {
    echo "2.Test Stage"
  }
  stage('Build') {
    echo "3.Build Stage"
  }
  stage('Deploy') {
    echo "4. Deploy Stage"
  }
}
```

- 构建：点击左侧区域的 Build Now，可以看到 Job 开始构建了

隔一会儿，构建完成，可以点击左侧区域的 Console Output，我们就可以看到如下输出信息：

![console output](https://www.qikqiak.com/k8s-book/docs/images/pipeline-demo1.png)

上面我们创建了一个简单的 Pipeline 任务，但是我们可以看到这个任务并没有在 Jenkins 的 Slave 中运行，那么如何让我们的任务跑在 Slave 中呢？还记得上节课我们在添加 Slave Pod 的时候，一定要记住添加的 label 吗？没错，我们就需要用到这个 label，我们重新编辑上面创建的 Pipeline 脚本，给 node 添加一个 label 属性，如下：

```
node('gomo-jnlp') {
    stage('Clone') {
      echo "1.Clone Stage"
    }
    stage('Test') {
      echo "2.Test Stage"
    }
    stage('Build') {
      echo "3.Build Stage"
    }
    stage('Deploy') {
      echo "4. Deploy Stage"
    }
}
```

我们这里只是给 node 添加了一个gomo-jnlp 这样的一个label，然后我们保存，构建之前查看下 kubernetes 集群中的 Pod：

```shell
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS              RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running             4          6d
```

然后重新触发立刻构建：

```shell
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running   4          6d
jnlp-017z7                 1/1       Running   0          23s
```

我们发现多了一个名叫**jnlp-017z7**的 Pod 正在运行，隔一会儿这个 Pod 就不再了：

```
$ kubectl get pods -n kube-ops
NAME                       READY     STATUS    RESTARTS   AGE
jenkins-7c85b6f4bd-rfqgv   1/1       Running   4          6d
```

这也证明我们的 Job 构建完成了，同样回到 Jenkins 的 Web UI 界面中查看 Console Output，可以看到如下的信息：

![image-20200108100945847](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200108100945847.png)

利用pipeline部署 Kubernetes 应用：

```
node('gomo-jnlp') {
stage('Clone') {
echo "1.Clone Stage"
}
stage('Test') {
echo "2.Test Stage"
}
stage('Build') {
echo "3.Build Docker Image Stage"
}
stage('Push') {
echo "4.Push Docker Image Stage"
}
stage('YAML') {
echo "5. Change YAML File Stage"
}
stage('Deploy') {
echo "6. Deploy Stage"
}
}
```

这⾥我们来将⼀个简单 golang 程序，部署到 kubernetes 环境中，代码链 接：https://github.com/cnych/jenkins-demo。如果按照之前的示例，我们是不是应该像这样来编写 Pipeline 脚本：

- 第⼀步，clone 代码，这个没得说吧 
- 第⼆步，进⾏测试，如果测试通过了才继续下⾯的任务 
- 第三步，由于 Dockerfile 基本上都是放⼊源码中进⾏管理的，所以我们这⾥就是直接构建 Docker 镜像了
- 第四步，镜像打包完成，就应该推送到镜像仓库中吧
- 第五步，镜像推送完成，是不是需要更改 YAML ⽂件中的镜像 TAG 为这次镜像的 TAG
-  第六步，万事俱备，只差最后⼀步，使⽤ kubectl 命令⾏⼯具进⾏部署了

第一步，Clone 代码

```shell
stage('Clone') {
    echo "1.Clone Stage"
    git url: "https://github.com/wangjiadee/Jenkins-cicd-demo.git"
}
```

第二步，测试

由于我们这里比较简单，忽略该步骤即可

第三步，构建镜像

```shell
stage('Build') {
    echo "3.Build Docker Image Stage"
    sh "docker build -t cnych/jenkins-demo:${build_tag} ."
}
```

我们平时构建的时候是不是都是直接使用`docker build`命令进行构建就行了，那么这个地方呢？我们上节课给大家提供的 Slave Pod 的镜像里面是不是采用的 Docker In Docker 的方式，也就是说我们也可以直接在 Slave 中使用 docker build 命令，所以我们这里直接使用 sh 直接执行 docker build 命令即可，但是镜像的 tag 呢？如果我们使用镜像 tag，则每次都是 latest 的 tag，这对于以后的排查或者回滚之类的工作会带来很大麻烦，我们这里采用和**git commit**的记录为镜像的 tag，这里有一个好处就是镜像的 tag 可以和 git 提交记录对应起来，也方便日后对应查看。但是由于这个 tag 不只是我们这一个 stage 需要使用，下一个推送镜像是不是也需要，所以这里我们把这个 tag 编写成一个公共的参数，把它放在 Clone 这个 stage 中，这样一来我们前两个 stage 就变成了下面这个样子：

```shell
stage('Clone') {
    echo "1.Clone Stage"
    git url: "https://github.com/wangjiadee/Jenkins-cicd-demo.git"
    script {
        build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
    }
}
stage('Build') {
    echo "3.Build Docker Image Stage"
    sh "docker build -t cnych/jenkins-demo:${build_tag} ."
}
```

第四步，推送镜像

镜像构建完成了，现在我们就需要将此处构建的镜像推送到镜像仓库中去，当然如果你有私有镜像仓库也可以，我们这里还没有自己搭建私有的仓库，所以直接使用 docker hub 即可。

> 在后面的课程中我们学习了私有仓库 Harbor 的搭建后，再来更改成 Harbor 仓库

我们知道 docker hub 是公共的镜像仓库，任何人都可以获取上面的镜像，但是要往上推送镜像我们就需要用到一个帐号了，所以我们需要提前注册一个 docker hub 的帐号，记住用户名和密码，我们这里需要使用。正常来说我们在本地推送 docker 镜像的时候，是不是需要使用**docker login**命令，然后输入用户名和密码，认证通过后，就可以使用**docker push**命令来推送本地的镜像到 docker hub 上面去了，如果是这样的话，我们这里的 Pipeline 是不是就该这样写了：

```shell
stage('Push') {
    echo "4.Push Docker Image Stage"
    sh "docker login -u wangjiadee -p xxxxx"
    sh "docker push wangjiadee/jenkins-demo:${build_tag}"
}
```

如果我们只是在 Jenkins 的 Web UI 界面中来完成这个任务的话，我们这里的 Pipeline 是可以这样写的，但是我们是不是推荐使用 Jenkinsfile 的形式放入源码中进行版本管理，这样的话我们直接把 docker 仓库的用户名和密码暴露给别人这样很显然是非常非常不安全的，更何况我们这里使用的是 github 的公共代码仓库，所有人都可以直接看到我们的源码，所以我们应该用一种方式来隐藏用户名和密码这种私密信息，幸运的是 Jenkins 为我们提供了解决方法。

在首页点击 Credentials -> Stores scoped to Jenkins 下面的 Jenkins -> Global credentials (unrestricted) -> 左侧的 Add Credentials：添加一个 Username with password 类型的认证信息，如下：

![image-20200108102204887](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200108102204887.png)

输入 docker hub 的用户名和密码，ID 部分我们输入**mydockerAuth**，注意，这个值非常重要，在后面 Pipeline 的脚本中我们需要使用到这个 ID 值。

有了上面的 docker hub 的用户名和密码的认证信息，现在我们可以在 Pipeline 中使用这里的用户名和密码了：

```shell
stage('Push') {
    echo "4.Push Docker Image Stage"
    withCredentials([usernamePassword(credentialsId: 'mydockerAuth', passwordVariable: 'mydockerAuthPassword', usernameVariable: 'mydockerAuthUser')]) {
        sh "docker login -u ${mydockerAuthUser} -p ${mydockerAuthPassword}"
        sh "docker push wangjiadee/jenkins-demo:${build_tag}"
    }
}
```

注意我们这里在 stage 中使用了一个新的函数**withCredentials**，其中有一个 credentialsId 值就是我们刚刚创建的 ID 值，而对应的用户名变量就是 ID 值加上 User，密码变量就是 ID 值加上 Password，然后我们就可以在脚本中直接使用这里两个变量值来直接替换掉之前的登录 docker hub 的用户名和密码，现在是不是就很安全了，我只是传递进去了两个变量而已，别人并不知道我的真正用户名和密码，只有我们自己的 Jenkins 平台上添加的才知道。

**还可以使用私有仓库 （推荐这种）**

```
    stage('Push') {
    echo "4.Push Docker Image Stage"
    sh "docker login 192.168.208.195 -u admin -p gomo"
    sh "docker push 192.168.208.195/library/jenkins-demo:${build_tag}"
    }
```

第五步，更改 YAML

上面我们已经完成了镜像的打包、推送的工作，接下来我们是不是应该更新 Kubernetes 系统中应用的镜像版本了，当然为了方便维护，我们都是用 YAML 文件的形式来编写应用部署规则，比如我们这里的 YAML 文件：(k8s.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: jenkins-demo
  template:
    metadata:
      labels:
        app: jenkins-demo
    spec:
      containers:
      - image: 192.168.208.195/library/jenkins-demo:<BUILD_TAG>
        imagePullPolicy: IfNotPresent
        name: jenkins-demo
        env:
        - name: branch
          value: <BRANCH_NAME>
```

对于 Kubernetes 比较熟悉的同学，对上面这个 YAML 文件一定不会陌生，我们使用一个 Deployment 资源对象来管理 Pod，该 Pod 使用的就是我们上面推送的镜像，唯一不同的地方是 Docker 镜像的 tag 不是我们平常见的具体的 tag，而是一个 的标识，实际上如果我们将这个标识替换成上面的 Docker 镜像的 tag，是不是就是最终我们本次构建需要使用到的镜像？怎么替换呢？其实也很简单，我们使用一个**sed**命令就可以实现了：

```shell
stage('YAML') {
    echo "5. Change YAML File Stage"
    sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
    sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s.yaml"
}
```

上面的 sed 命令就是将 k8s.yaml 文件中的 标识给替换成变量 build_tag 的值。

第六步，部署

Kubernetes 应用的 YAML 文件已经更改完成了，之前我们手动的环境下，是不是直接使用 kubectl apply 命令就可以直接更新应用了啊？当然我们这里只是写入到了 Pipeline 里面，思路都是一样的：

```shell
stage('Deploy') {
    echo "6. Deploy Stage"
    sh "kubectl apply -f k8s.yaml"
}
```

这样到这里我们的整个流程就算完成了。

人工确认

理论上来说我们上面的6个步骤其实已经完成了，但是一般在我们的实际项目实践过程中，可能还需要一些人工干预的步骤，这是为什么呢？比如我们提交了一次代码，测试也通过了，镜像也打包上传了，但是这个版本并不一定就是要立刻上线到生产环境的，对吧，我们可能需要将该版本先发布到测试环境、QA 环境、或者预览环境之类的，总之直接就发布到线上环境去还是挺少见的，所以我们需要增加人工确认的环节，一般都是在 CD 的环节才需要人工干预，比如我们这里的最后两步，我们就可以在前面加上确认，比如：

```shell
stage('YAML') {
    echo "5. Change YAML File Stage"
    def userInput = input(
        id: 'userInput',
        message: 'Choose a deploy environment',
        parameters: [
            [
                $class: 'ChoiceParameterDefinition',
                choices: "Dev\nQA\nProd",
                name: 'Env'
            ]
        ]
    )
    echo "This is a deploy step to ${userInput.Env}"
    sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
    sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s.yaml"
}
```

我们这里使用了 input 关键字，里面使用一个 Choice 的列表来让用户进行选择，然后在我们选择了部署环境后，我们当然也可以针对不同的环境再做一些操作，比如可以给不同环境的 YAML 文件部署到不同的 namespace 下面去，增加不同的标签等等操作：

```shell
stage('Deploy') {
    echo "6. Deploy Stage"
    if (userInput.Env == "Dev") {
      // deploy dev stuff
    } else if (userInput.Env == "QA"){
      // deploy qa stuff
    } else {
      // deploy prod stuff
    }
    sh "kubectl apply -f k8s.yaml"
}
```

由于这一步也属于部署的范畴，所以我们可以将最后两步都合并成一步，我们最终的 Pipeline 脚本如下：

```
node('gomo-jnlp') {
    stage('Clone') {
        echo "1.Clone Stage"
        git url: "https://github.com/wangjiadee/Jenkins-cicd-demo.git"
        script {
            build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
        }
    }
    stage('Test') {
      echo "2.Test Stage"
    }
    stage('Build') {
        echo "3.Build Docker Image Stage"
        sh "docker build -t 192.168.208.195/library/jenkins-demo:${build_tag} ."
    }
    stage('Push') {
    echo "4.Push Docker Image Stage"
    sh "docker login 192.168.208.195 -u admin -p gomo"
    sh "docker push 192.168.208.195/library/jenkins-demo:${build_tag}"
    }
    stage('Deploy') {
        echo "5. Deploy Stage"
        def userInput = input(
            id: 'userInput',
            message: 'Choose a deploy environment',
            parameters: [
                [
                    $class: 'ChoiceParameterDefinition',
                    choices: "Dev\nQA\nProd",
                    name: 'Env'
                ]
            ]
        )
        echo "This is a deploy step to ${userInput}"
        sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
        sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s.yaml"
        if (userInput == "Dev") {
            // deploy dev stuff
        } else if (userInput == "QA"){
            // deploy qa stuff
        } else {
            // deploy prod stuff
        }
        sh "kubectl apply -f k8s.yaml"
    }
}
```

现在我们在 Jenkins Web UI 中重新配置 jenkins-demo 这个任务，将上面的脚本粘贴到 Script 区域，重新保存，然后点击左侧的 Build Now，触发构建，然后过一会儿我们就可以看到 Stage View 界面出现了暂停的情况：

![image-20200108103011938](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200108103011938.png)

这就是我们上面 Deploy 阶段加入了人工确认的步骤，所以这个时候构建暂停了，需要我们人为的确认下，比如我们这里选择 QA，然后点击 Proceed，就可以继续往下走了，然后构建就成功了，我们在 Stage View 的 Deploy 这个阶段可以看到如下的一些日志信息：

![image-20200108103030646](C:\Users\wangjiadesx\AppData\Roaming\Typora\typora-user-images\image-20200108103030646.png)

打印出来了 QA，和我们刚刚的选择是一致的，现在我们去 Kubernetes 集群中观察下部署的应用：

```shell
$ kubectl get deployment -n kube-ops
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jenkins2        1         1         1            1           7d
jenkins-demo   1         1         1            0           1m
$ kubectl get pods -n kube-ops
NAME                           READY     STATUS      RESTARTS   AGE
jenkins2-7c85b6f4bd-rfqgv       1/1       Running     4          7d
jenkins-demo-f6f4f646b-2zdrq   0/1       Completed   4          1m
$ kubectl logs jenkins-demo-f6f4f646b-2zdrq -n kube-ops
Hello, Kubernetes！I'm from Jenkins CI！
```

我们可以看到我们的应用已经正确的部署到了 Kubernetes 的集群环境中了。



### 19.6Jenkins BlueOcean

上节课我们讲解了使用 Jenkins Pipeline 来自动化部署一个 Kubernetes 应用的方法，在实际的项目中，往往一个代码仓库都会有很多分支的，比如开发、测试、线上这些分支都是分开的，一般情况下开发或者测试的分支我们希望提交代码后就直接进行 CI/CD 操作，而线上的话最好增加一个人工干预的步骤，这就需要 Jenkins 对代码仓库有多分支的支持，当然这个特性是被 Jenkins 支持的。

#### 19.6.1Jenkinsfile

提到过最佳的方式是将脚本写入一个名为 Jenkinsfile 的文件中，跟随代码库进行统一的管理。

我们这里在之前的 git 库中新建一个 dev 分支，然后更改 main.go 的代码，打印当前运行的代码分支，通过环境变量注入进去，所以我们我们通过 k8s.yaml 文件的环境变量把当前代码分支注入进去，具体代码可以参考https://github.com/cnych/jenkins-demo/tree/dev。

然后新建一个 Jenkinsfile 文件，将构建脚本拷贝进来，但是我们需要对其做一些修改：(Jenkinsfile)

```shell
node('haimaxy-jnlp') {
    stage('Prepare') {
        echo "1.Prepare Stage"
        checkout scm
        script {
            build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
            if (env.BRANCH_NAME != 'master') {
                build_tag = "${env.BRANCH_NAME}-${build_tag}"
            }
        }
    }
    stage('Test') {
      echo "2.Test Stage"
    }
    stage('Build') {
        echo "3.Build Docker Image Stage"
        sh "docker build -t 192.168.208.195/library/jenkins-demo:${build_tag} ."
    }
    stage('Push') {
        echo "4.Push Docker Image Stage"
        withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'dockerHubPassword', usernameVariable: 'dockerHubUser')]) {
            sh "docker login -u ${dockerHubUser} -p ${dockerHubPassword}"
            sh "docker push 192.168.208.195/library/jenkins-demo:${build_tag}"
        }
    }
    stage('Deploy') {
        echo "5. Deploy Stage"
        if (env.BRANCH_NAME == 'master') {
            input "确认要部署线上环境吗？"
        }
        sh "sed -i 's/<BUILD_TAG>/${build_tag}/' k8s.yaml"
        sh "sed -i 's/<BRANCH_NAME>/${env.BRANCH_NAME}/' k8s.yaml"
        sh "kubectl apply -f k8s.yaml --record"
    }
}
```

在第一步中我们增加了**checkout scm**命令，用来检出代码仓库中当前分支的代码，为了避免各个环境的镜像 tag 产生冲突，我们为非 master 分支的代码构建的镜像增加了一个分支的前缀，在第五步中如果是 master 分支的话我们才增加一个确认部署的流程，其他分支都自动部署，并且还需要替换 k8s.yaml 文件中的环境变量的值。

更改完成后，提交 dev 分支到 github 仓库中。

## BlueOcean

我们这里使用 BlueOcean 这种方式来完成此处 CI/CD 的工作，BlueOcean 是 Jenkins 团队从用户体验角度出发，专为 Jenkins Pipeline 重新设计的一套 UI 界面，仍然兼容以前的 fressstyle 类型的 job，BlueOcean 具有以下的一些特性：

- 连续交付（CD）Pipeline 的复杂可视化，允许快速直观的了解 Pipeline 的状态
- 可以通过 Pipeline 编辑器直观的创建 Pipeline
- 需要干预或者出现问题时快速定位，BlueOcean 显示了 Pipeline 需要注意的地方，便于异常处理和提高生产力
- 用于分支和拉取请求的本地集成可以在 GitHub 或者 Bitbucket 中与其他人进行代码协作时最大限度提高开发人员的生产力。

BlueOcean 可以安装在现有的 Jenkins 环境中，也可以使用 Docker 镜像的方式直接运行，我们这里直接在现有的 Jenkins 环境中安装 BlueOcean 插件：登录 Jenkins Web UI -> 点击左侧的 Manage Jenkins -> Manage Plugins -> Available -> 搜索查找 BlueOcean -> 点击下载安装并重启

![install BlueOcean](https://www.qikqiak.com/k8s-book/docs/images/blue-demo1.png)

> 一般来说 Blue Ocean 在安装后不需要额外的配置，现有 Pipeline 和 Job 将继续照常运行。但是，Blue Ocean 在首次创建或添加 Pipeline的时候需要访问您的存储库（Git或GitHub）的权限，以便根据这些存储库创建 Pipeline。

安装完成后，我们可以在 Jenkins Web UI 首页左侧看到会多一个 Open Blue Ocean 的入口，我们点击就可以打开，如果之前没有创建过 Pipeline，则打开 Blue Ocean 后会看到一个**Create a new pipeline**的对话框：

![blue demo2](https://www.qikqiak.com/k8s-book/docs/images/blue-demo2.png)

然后我们点击开始创建一个新的 Pipeline，我们可以看到可以选择 Git、Bitbucket、GitHub，我们这里选择 GitHub，可以看到这里需要一个访问我们 GitHub 仓库权限的 token，在 GitHub 的仓库中创建一个 Personal access token:

![blue demo3](https://www.qikqiak.com/k8s-book/docs/images/blue-demo3.png)blue demo3

然后将生成的 token 填入下面的创建 Pipeline 的流程中，然后我们就有权限选择自己的仓库，包括下面需要构建的仓库，比如我们这里需要构建的是 jenkins-demo 这个仓库，然后创建 Pipeline 即可：

![blue demo4](https://www.qikqiak.com/k8s-book/docs/images/blue-demo4.png)blue demo4

Blue Ocean 会自动扫描仓库中的每个分支，会为根文件夹中包含**Jenkinsfile**的每个分支创建一个 Pipeline，比如我们这里有 master 和 dev 两个分支，并且两个分支下面都有 Jenkinsfile 文件，所以创建完成后会生成两个 Pipeline:

![blue demo5](https://www.qikqiak.com/k8s-book/docs/images/blue-demo5.png)blue demo5

我们可以看到有两个任务在运行了，我们可以把 master 分支的任务停止掉，我们只运行 dev 分支即可，然后我们点击 dev 这个 pipeline 就可以进入本次构建的详细页面：

![blue demo6](https://www.qikqiak.com/k8s-book/docs/images/blue-demo6.png)blue demo6

在上面的图中每个阶段我们都可以点击进去查看对应的构建结果，比如我们可以查看 Push 阶段下面的日志信息：

```shell
...
[jenkins-demo_dev-I2WMFUIFQCIFGRPNHN3HU7IZIMHEQMHWPUN2TP6DCYSWHFFFFHOA] Running shell script

+ docker push ****/jenkins-demo:dev-ee90aa5

The push refers to a repository [docker.io/****/jenkins-demo]

...
```

我们可以看到本次构建的 Docker 镜像的 Tag 为**dev-ee90aa5**，是符合我们在 Jenkinsfile 中的定义的吧

现在我们更改下 k8s.yaml 将 环境变量的值的标记改成 BRANCH_NAME，当然 Jenkinsfile 也要对应的更改，然后提交代码到 dev 分支并且 push 到 Github 仓库，我们可以看到 Jenkins Blue Ocean 里面自动触发了一次构建工作，最好同样我们可以看到本次构建能够正常完成，最后我们查看下本次构建的结果：

```shell
$ kubectl get pods
NAME                                      READY     STATUS        RESTARTS   AGE
...
jenkins-demo-648876568d-q5mbx             0/1       Completed     3          57s
...
$ kubectl logs jenkins-demo-648876568d-q5mbx
Hello, Kubernetes！I'm from Jenkins CI！
BRANCH: dev
```

我们可以看到打印了一句 BRANCH: dev ，证明我本次 CI/CD 是正常的。

现在我们来把 dev 分支的代码合并到 master 分支，然后来触发一次自动构建：

```shell
☁  jenkins-demo [dev] git status
On branch dev
nothing to commit, working directory clean
☁  jenkins-demo [dev] git checkout master
Switched to branch 'master'
Your branch is up-to-date with 'origin/master'.
☁  jenkins-demo [master] git merge dev
Updating 50e0401..ee90aa5
Fast-forward
 Jenkinsfile | 29 +++++++++--------------------
 k8s.yaml    |  3 +++
 main.go     |  2 ++
 3 files changed, 14 insertions(+), 20 deletions(-)
☁  jenkins-demo [master] git push origin master
Total 0 (delta 0), reused 0 (delta 0)
To git@github.com:cnych/jenkins-demo.git
   50e0401..ee90aa5  master -> master
```

然后我们回到 Jenkins 的 Blue Ocean 页面中，可以看到一个 master 分支下面的任务被自动触发了，同样我们进入详情页可以查看 Push 阶段下面的日志：

```shell
...
[jenkins-demo_master-XA3VZ5LP4XTCFAHHXIN3G5ZB4XA4J5H6I4DNKOH6JAXZXARF7LYQ] Running shell script

+ docker push ****/jenkins-demo:ee90aa5
...
```

我们可以查看到此处推送的镜像 TAG 为 ee90aa5，没有分支的前缀，是不是和我们前面在 Jenkinsfile 中的定义是一致的，镜像推送完成后，进入 Deploy 阶段的时候我们可以看到出现了一个暂停的操作，让我们选择是否需要部署到线上，我们前面是不是定义的如果是 master 分支的话，在部署的阶段需要我们人工确认：

![bule demo7](https://www.qikqiak.com/k8s-book/docs/images/blue-demo7.png)bule demo7

然后我们点击**Proceed**才会继续后面的部署工作，确认后，我们同样可以去 Kubernetes 环境中查看下部署的结果：

```shell
$ kubectl get pods
NAME                                      READY     STATUS             RESTARTS   AGE
...
jenkins-demo-c69dc6fdf-6ssjf              0/1       Completed   5          4m
...
$ kubectl logs jenkins-demo-c69dc6fdf-6ssjf
Hello, Kubernetes！I'm from Jenkins CI！
BRANCH: master
```

现在我们可以看到打印出来的信息是 master，证明部署是没有问题的。

到这里我们就实现了多分支代码仓库的完整的 CI/CD 流程。

当然我们这里的示例还是太简单，只是单纯为了说明 CI/CD 的步骤，在后面的课程中，我们会结合其他的工具进一步对我们现有的方式进行改造，比如使用 Helm、Gitlab 等等。

另外如果你对声明式的 Pipeline 比较熟悉的话，我们推荐使用这种方式来编写 Jenkinsfile 文件，因为使用声明式的方式编写的 Jenkinsfile 文件在 Blue Ocean 中不但支持得非常好，我们还可以直接在 Blue Ocean Editor 中可视化的对我们的 Pipeline 进行编辑操作，非常方便。

