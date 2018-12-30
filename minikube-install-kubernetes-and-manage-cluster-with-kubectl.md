## 使用minikube安装kubernetes集群并使用kubectl管理集群

### 第一步：本地安装虚拟机(Virtualbox，xhyve， VMWare)或在云厂商创建云主机即可
本文环境在 滴滴云 中

对于OS X，需要Homebrew来安装xhyve 驱动程序。安装Homebrew：

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```



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

> 滴滴云环境下：`sudo mv ./kubectl /usr/bin/kubectl`

> Mac环境使用Homebrew下载kubectl命令管理工具： `brew install kubectl` 



### 第二步：安装minikube

3. 安装minikube命令行工具

* 使用curl下载并安装minikube最新版：

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

* 滴滴云环境下：

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && sudo install minikube-linux-amd64 /usr/lbin/minikube
```

* Mac下：

使用curl下载并安装最新版本Minikube：
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64 && \
  chmod +x minikube && \
  sudo mv minikube /usr/local/bin/
```

使用Homebrew安装xhyve驱动程序并设置其权限：

```
brew install docker-machine-driver-xhyve
sudo chown root:wheel $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
sudo chmod u+s $(brew --prefix)/opt/docker-machine-driver-xhyve/bin/docker-machine-driver-xhyve
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

Mac下

```
minikube start --vm-driver=xhyve
```











