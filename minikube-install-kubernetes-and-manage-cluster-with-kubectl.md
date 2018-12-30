## 使用minikube安装kubernetes集群并使用kubectl管理集群

### 第一步：本地安装虚拟机(Virtualbox，xhyve， VMWare)或在云厂商创建云主机即可
本文环境在 滴滴云 中



### 第一步：安装kubectl

* 使用curl下载kubectl最新版

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

* 将kubectl配置可执行权限

```
chmod +x ./kubectl
```

* 将kubectl移到环境变量目录中

```
sudo mv ./kubectl /usr/local/bin/kubectl
```

滴滴云环境下：

```
sudo mv ./kubectl /usr/bin/kubectl
```

### 第二步：安装minikube

3. 安装minikube命令行工具

使用curl下载并安装minikube最新版：

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

滴滴云环境下：

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/lbin/minikube
```

### 第三步：运行minikube start命令

```
minikube start
```

首次启动会下载Minikube ISO和VM相关驱动，因此会比较慢

```
OUTPUT

Starting local Kubernetes v1.12.4 cluster...
Starting VM...
Downloading Minikube ISO

......
```






