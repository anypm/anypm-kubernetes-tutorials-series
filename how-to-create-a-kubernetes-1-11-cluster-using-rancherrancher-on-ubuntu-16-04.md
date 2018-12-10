# 如何在Ubuntu 16.04上使用Rancher创建Kubernetes 1.11集群

## 介绍



## 目标



## 第一步：购买滴滴云服务器
登陆[滴滴云控制台](https://app.didiyun.com/#/auth/signin?channel=0&return_to=https%3A%2F%2Fwww.didiyun.com%2F)购买**5台**滴滴云服务器(如果需要完成试验后即删除可以购买按时长配置)，配置推荐至少2核CPU、4GB内存、40GB存储、2M带宽，系统均为Ubuntu 16.04 LTS。5台服务器作用如下：
* Rancher节点：1台。
* Etcd节点：1台。
* 控制节点：1台。
* 工作节点：2台。

登陆滴滴云批量创建云服务器，如下图：

![批量创建DC2云服务器](01-dc2-create.png)

购买成功后DC2云服务器列表如下图：

![DC2列表](02-dc2-list.png)


> 备注：本文实验配置均为4核CPU、8GB内存、200GB存储、5M带宽。

## 第二步：安装Docker
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
添加Docker稳定版的官方软件源
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



## 第三步：安装rancher
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
87e6362368f4c0903d390c07be08636684d568f6cc32dca9ff5edc3a3584fb55

```
在本地主机打开浏览器，输入rancher访问地址：https://dc2公网IP， 如本案例中安装rancher server的DC2云服务器公网IP地址为`116.85.13.192`,所以rancher管理界面的访问地址为https://116.85.13.192, 如下图：
![安全连接]03-connect-adv.png

点击继续
![继续]04-connect-continue.png

设置管理密码
![设置管理密码]05-rancher-setpwd.png

保存url地址
![保存访问url地址]06-rancher-saveurl.png

进入rancher server管理控制台
![rancher server管理控制台]07-rancher-empty.png


## 第四步：安装Etcd节点与控制节点
点击【Add Cluster】进入添加集群页面，如下图
![添加集群]08-rancher-clustername.png

设置集群名称

* 【Node Options】-【Node Role】中只勾选【Etcd】
* 【Node Options】-【Node Address】-【Public Address】中天蝎Etcd云服务器的公网IP地址，如示例中的：`116.85.5.250`
* 【Node Options】-【Node Address】-【Internal Address】中天蝎Etcd云服务器的内网IP地址，如示例中的：`10.10.73.109`
* 【Node Options】-【Node Name】中填写Etcd云服务器的名称，如示例中的：`kubernetes-etcd-x`

设置效果如下图：
![设置Etcd]09-rancher-etcd.png

下方会自动生成Etcd节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.1 --server https://116.85.13.192 --token kcfhmzmtqhbm7lvhkcbmk5zrfdqd4mjrtqxh7684xzbsmcblp7kpmw --ca-checksum 8b256294489a4230cd34035520e790dc5fbc9541f80c28dfb7b1f8f1d16eecfe --node-name kubernetes-etcd-x --address 116.85.5.250 --internal-address 10.10.73.109 --etcd

```

复制配置命令到Ectd云服务器终端并运行,输出如下：
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
303f78d58a7d312a9444832f0536d9ce426c007aba9c24a7488f15fbc1eb0b90
```
Etcd节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`1 new node has registered`。



同样的步骤安装控制节点：
* 【Node Options】-【Node Role】中只勾选【Control Plane】
* 【Node Options】-【Node Address】-【Public Address】中天蝎Etcd云服务器的公网IP地址，如示例中的：`116.85.10.134`
* 【Node Options】-【Node Address】-【Internal Address】中天蝎Etcd云服务器的内网IP地址，如示例中的：`10.10.169.63`
* 【Node Options】-【Node Name】中填写Control Plane云服务器的名称，如示例中的：`kubernetes-controller-x`

设置效果如下图：
![设置ctlplane]10-rancher-ctlplane.png

下方会自动生成Control Plane节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.1 --server https://116.85.13.192 --token kcfhmzmtqhbm7lvhkcbmk5zrfdqd4mjrtqxh7684xzbsmcblp7kpmw --ca-checksum 8b256294489a4230cd34035520e790dc5fbc9541f80c28dfb7b1f8f1d16eecfe --node-name kubernetes-controller-x --address 116.85.10.134 --internal-address 10.10.169.63 --controlplane

```

复制配置命令到Control Plane云服务器终端并运行,输出如下：
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
bdd468b2552aba5db0e5bdd6b10d0f3b44872f0bbbe7292864dac69ea134ad4e
```
Control Plane节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`2 new node has registered`


## 第五步：安装工作节点
重复**第四步**中的步骤安装**第一个工作节点**，详情如下：
* 【Node Options】-【Node Role】中只勾选【Worker】
* 【Node Options】-【Node Address】-【Public Address】中填写第一台工作节点云服务器的公网IP地址，如示例中的：`116.85.5.252`
* 【Node Options】-【Node Address】-【Internal Address】中填写第一台工作节点云服务器的内网IP地址，如示例中的：`10.10.37.29`
* 【Node Options】-【Node Name】中填写第一台工作节点云服务器的名称，如示例中的：`kubernetes-worker01-x`

设置效果如下图：
![设置worker1]11-rancher-worker1.png

下方会自动生成工作节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.1 --server https://116.85.13.192 --token kcfhmzmtqhbm7lvhkcbmk5zrfdqd4mjrtqxh7684xzbsmcblp7kpmw --ca-checksum 8b256294489a4230cd34035520e790dc5fbc9541f80c28dfb7b1f8f1d16eecfe --node-name kubernetes-worker01-x --address 116.85.5.252 --internal-address 10.10.37.29 --worker

```

复制配置命令到`Worker1`云服务器终端并运行,输出如下：
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
638a0126b7a06138d463c08d4e41d37c65b546ef5dddca0619b065519e442acc
```
`Worker1`节点安装完成后会自动连接到Rancher管理服务器，管理控制台底部会提示`3 new node has registered`

重复如上步骤安装**第二个工作节点**，详情如下：
* 【Node Options】-【Node Role】中只勾选【Worker】
* 【Node Options】-【Node Address】-【Public Address】中填写第二台工作节点云服务器的公网IP地址，如示例中的：`116.85.37.182`
* 【Node Options】-【Node Address】-【Internal Address】中填写第二台工作节点云服务器的内网IP地址，如示例中的：`10.10.124.220`
* 【Node Options】-【Node Name】中填写第二台工作节点云服务器的名称，如示例中的：`kubernetes-worker02-x`

设置效果如下图：
![设置worker2]12-rancher-worker2.png

下方会自动生成工作节点的配置命令：
```
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.1 --server https://116.85.13.192 --token kcfhmzmtqhbm7lvhkcbmk5zrfdqd4mjrtqxh7684xzbsmcblp7kpmw --ca-checksum 8b256294489a4230cd34035520e790dc5fbc9541f80c28dfb7b1f8f1d16eecfe --node-name kubernetes-worker02-x --address 116.85.37.182 --internal-address 10.10.124.220 --worker

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

## 第六步：查看集群




## 结论
