# 使用Rancher创建Kubernetes集群并进行多集群可视化管理

> 写在前面：[《kubernetes tutorials》](https://github.com/anypm/kubernetes-tutorials-series) 系列文章旨在帮助您从入门到高阶逐步了解并掌握kubernetes技术栈的实操、理论和最佳实践。主题将包括**Docker基础与实操**、**Kubernetes基础与实操**、**基于Kubernetes的应用部署(工作负载版与集群版)**、**基于Kubernetes的CI/CD**、**Kubernetes集群与应用监控**、**Kubernetes运维与最佳生产实践**等，因为平时工作比较忙碌，尽量确保每周1～3篇相关文章，主题可能会比较随机，待全系列完成后再系统整理，尽情期待～ **天才都会三个神操作 `Watching` & `Star` & `Fork`**


## 介绍
Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

通过Kubernetes你可以：
* 快速部署应用
* 快速扩展应用
* 无缝对接新的应用功能
* 节省资源，优化硬件资源的使用

Kubernetes是Google 2014年创建管理的，是Google 10多年大规模容器管理技术Borg的开源版本。Kubernetes的目标是促进完善组件和工具的生态系统，以减轻应用程序在公有云或私有云中运行的负担。

### Kubernetes 特点
* 可移植: 支持公有云，私有云，混合云，多重云（multi-cloud）
* 可扩展: 模块化, 插件化, 可挂载, 可组合
* 自动化: 自动部署，自动重启，自动复制，自动伸缩/扩展


### 常见的创建Kubernetes集群的方式有
* [使用Kubeadm创建Kubernetes集群](https://github.com/anypm/kubernetes-tutorials-series/blob/master/how-to-create-a-kubernetes-1-11-cluster-using-kubeadm-on-ubuntu-18-04.md)
* 使用Kubemini创建Kubernetes集群
* 使用RKE创建Kubernetes集群
* **[使用Rancher创建Kubrnetes集群](https://github.com/anypm/kubernetes-tutorials-series/blob/master/how-to-create-a-kubernetes-1-11-cluster-using-rancher-and-manage-clusters.md)**

本文主要讲述**如何使用Rancher创建Kubernetes集群并进行可视化的集群管理**。后续会陆续发布其他方式创建Kubernetes集群，并在本文中给出相关链接，请您持续关注

### Rancher创建与管理Kubernetes集群的主要优势

Rancher是一套容器管理平台，它可以帮助组织在生产环境中轻松快捷的部署和管理容器。 Rancher可以轻松地管理各种环境的Kubernetes，满足IT需求并为DevOps团队提供支持。

* 企业级容器管理平台
Rancher是业界唯一完全开源的企业级容器管理平台，为企业用户提供在生产环境中落地使用容器所需的一切功能与组件。Rancher2.0基于Kubernetes构建,使用Rancher，DevOps团队可以轻松测试、部署和管理应用程序，运维团队可以部署、管理和维护一切Kubernetes集群，无论集群运行在何基础设施之上。

* 多集群管理
Rancher可以更方便的管理Kubernetes集群，它可以从头开始轻松部署新集群，甚至可以导入现有的Kubernetes集群。

* 统一运营管理
对于Rancher，运营团队在开发，测试和生产Kubernetes集群中拥有相同的部署和管理工具。


## 目标
集群包含以下资源

* 1个Rancher节点：用于部署Rancher Server，通过该节点可以实现可视化多集群、跨云管理Kubernetes集群
* 1个Etcd节点：存储主控制节点和工作节点之间的任务调度等数据信息
* 1个控制(Controller)节点：部署Kunbernetes集群主控制节点，用于管理和监控Kubernetes其它的工作节点和存在状态信息。
* 2个工作(Worker)节点：部署Kubernetes集群的工作节点，用于运行容器化的应用。

完成本指南后您将学会**安装Docker环境**、**搭建Rancher集群管理环境**、**使用Rancher创建Kubernetes环境**和**使用Rancher进行多集群管理**。

> 注意：配置推荐至少2核CPU、4GB内存、40GB存储、2M带宽，系统均为Ubuntu 16.04 LTS。为达到更好的效果，本文创建的5台云服务器配置均为4核CPU、8GB内存、200GB存储、5M带宽，系统选择Ubuntu 16.04 LTS。


## 第一步：购买滴滴云服务器
登陆[滴滴云控制台](https://app.didiyun.com/#/auth/signin?channel=0&return_to=https%3A%2F%2Fwww.didiyun.com%2F)购买**5台**滴滴云服务器(如果需要完成试验后即删除可以购买按时长配置)，配置推荐至少2核CPU、4GB内存、40GB存储、2M带宽，系统均为Ubuntu 16.04 LTS。5台服务器作用如下：


登陆滴滴云批量创建云服务器，如下图：

![批量创建DC2云服务器](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/01-dc2-create.png)

购买成功后DC2云服务器列表如下图：

![DC2列表](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/02-dc2-list.png)


> 注意：为达到更好的效果，本文创建的5台云服务器配置均为4核CPU、8GB内存、200GB存储、5M带宽。

## 第二步：安装Docker

> 本步骤概述安装Docker的通用步骤，上一步准备的5台云服务器均可按本指南安装Docker-CE。更详细的Docker安装与使用教程请参考[如何用滴滴云在Ubuntu 16.04上安装和使用Docker](https://github.com/luneyuyu/usedocker/blob/master/How-to-install-and-use-Docker-on-Ubuntu16.04-by-didiyun.md)

1. 检查内核版本：
```
$ uname -a
```
输出如下结果,内核版本符合要求(确保内核版本3.0以上)：
```
$ Linux 10-10-73-109 4.4.0-138-generic #164-Ubuntu SMP Tue Oct 2 17:16:02 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

2. 为了让Docker使用aufs存储，推荐安装如下两个软件包
```
$ sudo apt-get update
```
```
OUTPUT

Hit:1 http://mirrors.intra.didiyun.com/ubuntu xenial InRelease
Hit:2 http://mirrors.intra.didiyun.com/ubuntu xenial-updates InRelease
Hit:3 http://mirrors.intra.didiyun.com/ubuntu xenial-backports InRelease
Hit:4 http://mirrors.intra.didiyun.com/ubuntu xenial-security InRelease
Reading package lists... Done
```
```
$ sudo apt-get install -y \
linux-image-extra-$(uname -r) \
linux-image-extra-virtual
```
```
OUTPUT

Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  amd64-microcode crda intel-microcode iucode-tool iw libnl-3-200 libnl-genl-3-200 linux-firmware linux-image-4.4.0-140-generic linux-image-extra-4.4.0-140-generic linux-image-generic thermald wireless-regdb
Suggested packages:
  fdutils linux-doc-4.4.0 | linux-source-4.4.0 linux-tools linux-headers-4.4.0-140-generic
The following NEW packages will be installed:
  amd64-microcode crda intel-microcode iucode-tool iw libnl-3-200 libnl-genl-3-200 linux-firmware linux-image-4.4.0-140-generic linux-image-extra-4.4.0-138-generic linux-image-extra-4.4.0-140-generic
  linux-image-extra-virtual linux-image-generic thermald wireless-regdb
0 upgraded, 15 newly installed, 0 to remove and 17 not upgraded.
Need to get 147 MB of archives.
After this operation, 623 MB of additional disk space will be used.
...

```

3. 添加镜像源
* 安装apt-transport-https等软件包支持https协议的源
```
$ sudo apt-get install \
apt-transport-https \
ca-certificates \
curl \
software-properties-common
```
```
OUTPUT

Reading package lists... Done
Building dependency tree       
Reading state information... Done
apt-transport-https is already the newest version (1.2.29).
ca-certificates is already the newest version (20170717~16.04.1).
curl is already the newest version (7.47.0-1ubuntu2.11).
software-properties-common is already the newest version (0.96.20.7).
0 upgraded, 0 newly installed, 0 to remove and 17 not upgraded.
```

添加源的gpg密钥
```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
确认指纹为"9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88"的GPG公钥
```
sudo apt-key fingerprint 0EBFCD88
```


获取当前系统的代号
```
lsb_release -cs
```
```
OUTPUT

xenial
```
添加Docker稳定版的官方软件源,本文中使用16.04 LTS对应的系统代号为`xenial`；若使用其他版本ubuntu，将代号修改为对应版本的系统代号即可
```
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
xenial  \
stable"
```
添加成功后，再次更新apt软件包缓存
```
sudo apt-get update
```
成功添加源之后就可以安装最新版本的Docker了，软件包名为docker-ce
```
sudo apt-get install -y docker-ce
```
除了基于手动添加软件源的方式之外，也可以使用官方提供的脚本来自动化安装docker
```
sudo curl -fsSL https://get.docker.com/ | sh
```
设置开机自动启动
```
$ sudo systemctl enable docker.service
$ sudo systemctl daemon-reload
$ sudo service docker restart
```

检查安装
```
$ sudo service docker status
```
```
OUTPUT

● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2018-11-26 00:41:59 CST; 2 weeks 0 days ago
     Docs: https://docs.docker.com
 Main PID: 5509 (dockerd)
    Tasks: 19
   Memory: 2.2G
      CPU: 5h 57min 9.624s
   CGroup: /system.slice/docker.service
           └─5509 /usr/bin/dockerd -H unix://

Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.
```

每次重启Docker后，可以通过查看Docker信息确保服务已经正常运行
```
sudo docker version
```
```
OUTPUT

Client:
 Version:           18.09.0
 API version:       1.39
 Go version:        go1.10.4
 Git commit:        4d60db4
 Built:             Wed Nov  7 00:48:57 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          18.09.0
  API version:      1.39 (minimum version 1.12)
  Go version:       go1.10.4
  Git commit:       4d60db4
  Built:            Wed Nov  7 00:16:44 2018
  OS/Arch:          linux/amd64
  Experimental:     false
```



## 第三步：安装rancher Server

> 本指南为安装rancher的通用步骤，从上述云服务器中挑选一台按照本指南安装rancher server即可

安装和运行Rancher Server,运行如下命令会从Docker Hub仓库中拉取rancher镜像并完成安装
```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```
```
OUTPUT

Unable to find image 'rancher/rancher:latest' locally
latest: Pulling from rancher/rancher
32802c0cfa4d: Pull complete 
da1315cffa03: Pull complete 
fa83472a3562: Pull complete 
f85999a86bef: Pull complete 
802918c3c5d1: Pull complete 
941c9d7db7cb: Pull complete 
a00bebfc6f0e: Pull complete 
0a145b822324: Pull complete 
1cd1020104e1: Pull complete 
03f3b0fc5689: Pull complete 
07054e1590fd: Pull complete 
db38f96efb72: Pull complete 
Digest: sha256:b5762180fdc05b5be8337453cc9bbadc33645d50cd8d2dac89c6676bf07460b7
Status: Downloaded newer image for rancher/rancher:latest
6a9758a790ebe1b4ee94725023ec98b214304e2f59fc71c658ef025b0533efef

```
打开浏览器，输入https://<server_ip>,server_ip替换为运行Rancher容器主机的ip;如本文中安装rancher server的云服务器公网IP地址为`116.85.46.53`,所以rancher管理界面的访问地址为https://116.85.46.53, 如下图：
![安全连接](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/03-connect-adv.png)

因为是自动使用的自签名证书，在第一次登录会提示安全授信问题，信任即可；
![继续](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/04-connect-continue.png)

设置管理密码(第一次登录会要求设置管理员密码，默认管理员账号为: `admin`)
![设置管理密码](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/05-rancher-setpwd.png)

设置Rancher Server URL(这个Rancher Server URL是agent节点注册的地址，需要保证这个地址能够被其他主机访问)
![保存访问url地址](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/06-rancher-saveurl.png)

进入rancher server管理控制台
![rancher server管理控制台](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/07-rancher-empty.png)


## 第四步：安装Etcd节点与控制节点

> 本指南为安装rancher的通用步骤，从上述云服务器中挑选两台分别作为Etcd节点和控制节点进行安装即可

点击【Add Cluster】进入添加集群页面，设置集群名称:`k8s-cluster-rancher`，如下图

![添加集群](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/08-rancher-clustername.png)

设置Etcd节点选项

* 【Node Options】-【Node Role】中只勾选【Etcd】
* 【Node Options】-【Node Address】-【Public Address】中天蝎Etcd云服务器的公网IP地址，如示例中的：`116.85.13.168`
* 【Node Options】-【Node Address】-【Internal Address】中天蝎Etcd云服务器的内网IP地址，如示例中的：`10.10.150.195`
* 【Node Options】-【Node Name】中填写Etcd云服务器的名称，如示例中的：`kubernetes-etc-x`

设置效果如下图：

![设置Etcd](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/09-rancher-etcd.png)

下方会自动生成Etcd节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.3 --server https://116.85.46.53 --token zs6zxddp7rdfxfvwhcpdw4vfxbfqccm2vg6hh7st2nwr9l9rtbk7j8 --ca-checksum 551014a641a7d23d0ed67153e2797b9fd8670307422cdfc232020cb831e376aa --node-name kubernetes-etc-x --address 116.85.13.168 --internal-address 10.10.150.195 --etcd

```

复制配置命令到Ectd云服务器终端并运行,输出如下：
```
OUTPUT

Unable to find image 'rancher/rancher-agent:v2.1.3' locally
v2.1.3: Pulling from rancher/rancher-agent
32802c0cfa4d: Pull complete 
da1315cffa03: Pull complete 
fa83472a3562: Pull complete 
f85999a86bef: Pull complete 
5bf53f7eb665: Pull complete 
b6dee6425e98: Pull complete 
15612bde45d1: Pull complete 
5bb137229af3: Pull complete 
72fc31ea0fd1: Pull complete 
Digest: sha256:c0c15e3fb32d516a16889765fe9ce62713617dc2f599a516c7d66620c737b705
Status: Downloaded newer image for rancher/rancher-agent:v2.1.3
ab7275fe95b6d2ffeb9c9090f642af311c71de5d4c2ca8e3d39163f4e52ad89c
```
Etcd节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`1 new node has registered`。



同样的步骤安装控制节点：
* 【Node Options】-【Node Role】中只勾选【Control Plane】
* 【Node Options】-【Node Address】-【Public Address】中天蝎Etcd云服务器的公网IP地址，如示例中的：`116.85.60.45`
* 【Node Options】-【Node Address】-【Internal Address】中天蝎Etcd云服务器的内网IP地址，如示例中的：`10.10.0.89`
* 【Node Options】-【Node Name】中填写Control Plane云服务器的名称，如示例中的：`kubernetes-controller-x`

设置效果如下图：

![设置ctlplane](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/10-rancher-ctlplane.png)

下方会自动生成Control Plane节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.3 --server https://116.85.46.53 --token zs6zxddp7rdfxfvwhcpdw4vfxbfqccm2vg6hh7st2nwr9l9rtbk7j8 --ca-checksum 551014a641a7d23d0ed67153e2797b9fd8670307422cdfc232020cb831e376aa --node-name kubernetes-controller-x --address 116.85.60.45 --internal-address 10.10.0.89 --controlplane

```

复制配置命令到Control Plane云服务器终端并运行,输出如下：
```
OUTPUT

Unable to find image 'rancher/rancher-agent:v2.1.3' locally
v2.1.3: Pulling from rancher/rancher-agent
32802c0cfa4d: Pull complete 
da1315cffa03: Pull complete 
fa83472a3562: Pull complete 
f85999a86bef: Pull complete 
5bf53f7eb665: Pull complete 
b6dee6425e98: Pull complete 
15612bde45d1: Pull complete 
5bb137229af3: Pull complete 
72fc31ea0fd1: Pull complete 
Digest: sha256:c0c15e3fb32d516a16889765fe9ce62713617dc2f599a516c7d66620c737b705
Status: Downloaded newer image for rancher/rancher-agent:v2.1.3
cb878cb4c25b985f7a1af4be7658338c62f2a312f4c78da07e4f1e6a8b48c706
```
Control Plane节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`2 new node has registered`


## 第五步：安装工作节点

> 本指南为安装工作节点的通用步骤，从上述云服务器中挑选两台作为工作节点进行安装即可

重复**第四步**中的步骤安装**第一个工作节点**，详情如下：
* 【Node Options】-【Node Role】中只勾选【Worker】
* 【Node Options】-【Node Address】-【Public Address】中填写第一台工作节点云服务器的公网IP地址，如示例中的：`116.85.30.97`
* 【Node Options】-【Node Address】-【Internal Address】中填写第一台工作节点云服务器的内网IP地址，如示例中的：`10.10.232.55`
* 【Node Options】-【Node Name】中填写第一台工作节点云服务器的名称，如示例中的：`kubernetes-worker01-x`

设置效果如下图：

![设置worker1](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/11-rancher-worker1.png)

下方会自动生成工作节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.3 --server https://116.85.46.53 --token zs6zxddp7rdfxfvwhcpdw4vfxbfqccm2vg6hh7st2nwr9l9rtbk7j8 --ca-checksum 551014a641a7d23d0ed67153e2797b9fd8670307422cdfc232020cb831e376aa --node-name kubernetes-worker01-x --address 116.85.30.97 --internal-address 10.10.232.55 --worker

```

复制配置命令到`Worker1`云服务器终端并运行,输出如下：
```
OUTPUT
Unable to find image 'rancher/rancher-agent:v2.1.3' locally
v2.1.3: Pulling from rancher/rancher-agent
32802c0cfa4d: Pull complete 
da1315cffa03: Pull complete 
fa83472a3562: Pull complete 
f85999a86bef: Pull complete 
5bf53f7eb665: Pull complete 
b6dee6425e98: Pull complete 
15612bde45d1: Pull complete 
5bb137229af3: Pull complete 
72fc31ea0fd1: Pull complete 
Digest: sha256:c0c15e3fb32d516a16889765fe9ce62713617dc2f599a516c7d66620c737b705
Status: Downloaded newer image for rancher/rancher-agent:v2.1.3
5213acaac6a21cfa5770415755f924a53fa037aeb1163df50d3a8c9ed12d5c36
```
`Worker1`节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`3 new node has registered`

重复如上步骤安装**第二个工作节点**，详情如下：
* 【Node Options】-【Node Role】中只勾选【Worker】
* 【Node Options】-【Node Address】-【Public Address】中填写第二台工作节点云服务器的公网IP地址，如示例中的：`116.85.37.60`
* 【Node Options】-【Node Address】-【Internal Address】中填写第二台工作节点云服务器的内网IP地址，如示例中的：`10.10.148.43`
* 【Node Options】-【Node Name】中填写第二台工作节点云服务器的名称，如示例中的：`kubernetes-worker02-x`

设置效果如下图：

![设置worker2](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/12-rancher-worker2.png)

下方会自动生成工作节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.3 --server https://116.85.46.53 --token zs6zxddp7rdfxfvwhcpdw4vfxbfqccm2vg6hh7st2nwr9l9rtbk7j8 --ca-checksum 551014a641a7d23d0ed67153e2797b9fd8670307422cdfc232020cb831e376aa --node-name kubernetes-worker02-x --address 116.85.37.60 --internal-address 10.10.148.43 --worker

```

复制配置命令到`Worker2`云服务器终端并运行,输出如下：
```
OUTPUT

Unable to find image 'rancher/rancher-agent:v2.1.1' locally
v2.1.1: Pulling from rancher/rancher-agent
473ede7ed136: Pull complete 
c46b5fa4d940: Pull complete 
93ae3df89c92: Pull complete 
6b1eed27cade: Pull complete 
f21f12a5ca08: Pull complete 
33a2b1a4bb51: Pull complete 
a0876ba0c412: Pull complete 
cbb664d29f0f: Pull complete 
bd32e23bc74b: Pull complete 
Digest: sha256:2236b44b39bf0c2ae2f5c158f2516e3c89c85f8fa664fa3315b3effe66e63395
Status: Downloaded newer image for rancher/rancher-agent:v2.1.1
909549bc0d6035a9df4d68cbe51406da76c3e17e1d633d541dca0c82a06dc69d
```
`Worker2`节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`4 new node has registered`

至此，Kubernetes集群所有节点均已成功安装，点击【Done】完成集群设置即可。

## 第六步：查看与管理集群

> 可以按照上述方法添加多个Kubernetes集群，本文为更好的演示多集群管理效果，添加了多个kubernetes集群

#### 查看集群，可以看到集群各节点状态还在准备中

![集群准备中](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/13-rancher-cluster.png)

#### 【Global】菜单可查看当前存在的集群并切换到您需要管理的集群

![Global](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/14-rancher-cluster-global.png)

#### 【Cluster】菜单可以可视化的查看和管理某个具体的集群信息，也可以图表化查看集群资源的消耗情况

![Cluster](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/15-rancher-cluster-clusterviews.png)

#### 【Node】菜单可以查看与管理节点服务器

![Node](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/16-rancher-cluster-nodeviews.png)

#### 【Project/Namespace】菜单可以查看与管理命名空间
![Namespace](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/17-rancher-cluster-nsviews.png)

#### 【Members】菜单可以管理Rancher管理控制台的成员信息，包括账户、密码、角色/权限等
![Members](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/18-rancher-cluster-membersviews.png)


#### 【Tools】菜单可以管理告警、提醒与日志信息
![Tools](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/19-rancher-cluster-toolsviews.png)


点击右下角的【Language】可以切换语言，如果您更习惯看中文，恭喜～您可以选择【简体中文】获得更好的管理体验:)

![Language](https://github.com/anypm/kubernetes-tutorials-series/blob/master/images/20-rancher-cluster-lang.png)


## 结论
Rancher为DevOps工程师提供了一个直观的用户界面来管理他们的服务容器，用户不需要深入了解Kubernetes概念就可以开始使用Rancher。 Rancher包含应用商店，支持一键式部署Helm和Compose模板。Rancher通过各种云、本地生态系统产品认证，其中包括安全工具，监控系统，容器仓库以及存储和网络驱动程序。

## K8S相关概念

* 节点（Node）

Kubernetes 集群中的计算能力由 Node 提供，Kubernetes 集群中的 Node 是所有 Pod 运行所在的工作主机，可以是物理机也可以是虚拟机。工作主机的统一特征是上面要运行 kubelet 管理节点上运行的容器。

* 命名空间（Namespace）

命名空间为 Kubernetes 集群提供虚拟的隔离作用。Kubernetes 集群初始有 3 个命名空间，分别是默认命名空间 default、系统命名空间 kube-system 和 kube-public ，除此以外，管理员可以创建新的命名空间以满足需求。

* Pod

Pod是 Kubernetes 部署应用或服务的最小的基本单位。一个Pod 封装多个应用容器（也可以只有一个容器）、存储资源、一个独立的网络 IP 以及管理控制容器运行方式的策略选项。

* 副本控制器（Replication Controller，RC）

RC 确保任何时候 Kubernetes 集群中有指定数量的 pod 副本(replicas)在运行。通过监控运行中的 Pod 来保证集群中运行指定数目的 Pod 副本。指定的数目可以是多个也可以是1个；少于指定数目，RC 就会启动运行新的 Pod 副本；多于指定数目，RC 就会终止多余的 Pod 副本。

* 副本集（Replica Set，RS）

ReplicaSet（RS）是 RC 的升级版本，唯一区别是对选择器的支持，RS 能支持更多种类的匹配模式。副本集对象一般不单独使用，而是作为 Deployment 的理想状态参数使用。

* 部署（Deployment）

部署表示用户对 Kubernetes 集群的一次更新操作。部署比 RS 应用更广，可以是创建一个新的服务，更新一个新的服务，也可以是滚动升级一个服务。滚动升级一个服务，实际是创建一个新的 RS，然后逐渐将新 RS 中副本数增加到理想状态，将旧 RS 中的副本数减小到 0 的复合操作；这样一个复合操作用一个RS是不太好描述的，所以用一个更通用的 Deployment 来描述。不建议您手动管理利用 Deployment 创建的 RS。

* 服务（Service）

Service 也是 Kubernetes 的基本操作单元，是真实应用服务的抽象，每一个服务后面都有很多对应的容器来提供支持，通过 Kube-Proxy 的 port 和服务 selector 决定服务请求传递给后端的容器，对外表现为一个单一访问接口，外部不需要了解后端如何运行，这给扩展或维护后端带来很大的好处。

* 标签（labels）

Labels 的实质是附着在资源对象上的一系列 Key/Value 键值对，用于指定对用户有意义的对象的属性，标签对内核系统是没有直接意义的。标签可以在创建一个对象的时候直接赋予，也可以在后期随时修改，每一个对象可以拥有多个标签，但 key 值必须唯一。

* 存储卷（Volume）

Kubernetes 集群中的存储卷跟 Docker 的存储卷有些类似，只不过 Docker 的存储卷作用范围为一个容器，而 Kubernetes 的存储卷的生命周期和作用范围是一个 Pod。每个 Pod 中声明的存储卷由 Pod 中的所有容器共享。支持使用 Persistent Volume Claim 即 PVC 这种逻辑存储，使用者可以忽略后台的实际存储技术，具体关于 Persistent Volumn(pv)的配置由存储管理员来配置。

* 持久存储卷（Persistent Volume，PV）和持久存储卷声明（Persistent Volume Claim，PVC）

PV 和 PVC 使得 Kubernetes 集群具备了存储的逻辑抽象能力，使得在配置 Pod 的逻辑里可以忽略对实际后台存储技术的配置，而把这项配置的工作交给 PV 的配置者。存储的 PV 和 PVC 的这种关系，跟计算的 Node 和 Pod 的关系是非常类似的；PV 和 Node 是资源的提供者，根据集群的基础设施变化而变化，由 Kubernetes 集群管理员配置；而 PVC 和 Pod是资源的使用者，根据业务服务的需求变化而变化，由 Kubernetes 集群的使用者即服务的管理员来配置。

* Ingress

Ingress 是授权入站连接到达集群服务的规则集合。你可以通过 Ingress 配置提供外部可访问的 URL、负载均衡、SSL、基于名称的虚拟主机等。用户通过 POST Ingress 资源到 API server 的方式来请求 Ingress。 Ingress controller 负责实现 Ingress，通常使用负载均衡器，它还可以配置边界路由和其他前端，这有助于以 HA 方式处理流量。


