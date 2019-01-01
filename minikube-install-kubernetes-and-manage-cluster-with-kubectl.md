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

### 第四步：安装Helm

mac下： `brew install kubernetes-helm`

```
OUTPUT

Updating Homebrew...
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).
==> New Formulae
atomist-cli                                                                                            ruby@2.5
==> Updated Formulae
dcd                          docker-machine-parallels     gobject-introspection        imagemagick                  neovim                       pgbadger                     wtf
dfmt                         dscanner                     grakn                        ldc                          nnn                          tika                         youtube-dl
diamond                      easyengine                   http-parser                  meson                        node-build                   verilator

==> Downloading https://homebrew.bintray.com/bottles/kubernetes-helm-2.12.1.mojave.bottle.tar.gz
######################################################################## 100.0%
==> Pouring kubernetes-helm-2.12.1.mojave.bottle.tar.gz
==> Caveats
Bash completion has been installed to:
  /usr/local/etc/bash_completion.d

zsh completions have been installed to:
  /usr/local/share/zsh/site-functions
==> Summary
🍺  /usr/local/Cellar/kubernetes-helm/2.12.1: 51 files, 79.4MB

```

**初始化helm：**  `helm init`

```
OUTPUT

Creating /Users/didi/.helm 
Creating /Users/didi/.helm/repository 
Creating /Users/didi/.helm/repository/cache 
Creating /Users/didi/.helm/repository/local 
Creating /Users/didi/.helm/plugins 
Creating /Users/didi/.helm/starters 
Creating /Users/didi/.helm/cache/archive 
Creating /Users/didi/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /Users/didi/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!

```



### 第五步：安装Catlog

* 添加catlog安装源： `helm repo add svc-cat https://svc-catalog-charts.storage.googleapis.com`

```
OUTPUT

"svc-cat" has been added to your repositories
daniel:~ didi$ helm repo list
NAME   	URL                                              
stable 	https://kubernetes-charts.storage.googleapis.com 
local  	http://127.0.0.1:8879/charts                     
svc-cat	https://svc-catalog-charts.storage.googleapis.com

```

* 查看repo清单： `helm repo list`

```
OUTPUT

NAME   	URL                                              
stable 	https://kubernetes-charts.storage.googleapis.com 
local  	http://127.0.0.1:8879/charts                     
svc-cat	https://svc-catalog-charts.storage.googleapis.com

```


* 搜索是否含有catalog源： `helm search service-catalog`

```
OUTPUT

NAME           	CHART VERSION	APP VERSION	DESCRIPTION                                                 
svc-cat/catalog	0.1.38       	           	service-catalog API server and controller-manager helm chart

```

* 安装Catalog服务: `helm install svc-cat/catalog --name catalog --namespace catalog`

```
OUTPUT

NAME:   catalog
LAST DEPLOYED: Mon Dec 31 09:42:01 2018
NAMESPACE: catalog
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                                      AGE
servicecatalog.k8s.io:apiserver           1s
servicecatalog.k8s.io:controller-manager  1s

==> v1/ServiceAccount
NAME                                SECRETS  AGE
service-catalog-apiserver           1        1s
service-catalog-controller-manager  1        1s

==> v1/ClusterRoleBinding
NAME                                            AGE
servicecatalog.k8s.io:apiserver                 1s
servicecatalog.k8s.io:apiserver-auth-delegator  1s
servicecatalog.k8s.io:controller-manager        1s

==> v1/RoleBinding
NAME                                                   AGE
servicecatalog.k8s.io:apiserver-authentication-reader  1s
service-catalog-controller-manager-cluster-info        1s
service-catalog-controller-manager-leader-election     1s

==> v1/Role
NAME                                                     AGE
servicecatalog.k8s.io:cluster-info-configmap             1s
servicecatalog.k8s.io:leader-locking-controller-manager  1s

==> v1/Secret
NAME                            TYPE    DATA  AGE
catalog-catalog-apiserver-cert  Opaque  2     5s

==> v1/Service
NAME                       TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)        AGE
catalog-catalog-apiserver  NodePort  10.102.219.44  <none>       443:30443/TCP  5s

==> v1beta1/Deployment
NAME                                DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
catalog-catalog-apiserver           1        0        0           0          5s
catalog-catalog-controller-manager  1        0        0           0          5s

==> v1beta1/APIService
NAME                           AGE
v1beta1.servicecatalog.k8s.io  1s

```


* 查看catalog的pod运行状态： `kubectl get pods -n catalog`

```
OUTPUT

NAME                                                  READY     STATUS    RESTARTS   AGE
catalog-catalog-apiserver-74ccfdccb5-mk85g            2/2       Running   1          1d
catalog-catalog-controller-manager-578b5f4c54-9j6r2   1/1       Running   2          1d

```










