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

![批量创建DC2云服务器](images/01-dc2-create.png)

购买成功后DC2云服务器列表如下图：
![DC2列表](images/01-dc2-list.png)


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
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
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


## 第三步：安装rancher


## 第四步：安装Etcd节点与控制节点


## 第五步：安装工作节点


## 第六步：查看集群


## 结论
