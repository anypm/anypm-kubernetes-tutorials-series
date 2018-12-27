## 使用Kubernetes构建基于高可用容器集群的全自动可视化CI/CD流水线实操指南

> 写在前面：[《kubernetes tutorials》](https://github.com/anypm/kubernetes-tutorials-series) 系列文章旨在帮助您从入门到高阶逐步了解并掌握kubernetes技术栈的实操、理论和最佳实践。主题将包括**Docker基础与实操**、**Kubernetes基础与实操**、**基于Kubernetes的应用部署(工作负载版与集群版)**、**基于Kubernetes的CI/CD**、**Kubernetes集群与应用监控**、**Kubernetes运维与最佳生产实践**等，因为平时工作比较忙碌，尽量确保每周1～3篇相关文章，主题可能会比较随机，待全系列完成后再系统整理，尽情期待～ **天才都会三个神操作 `Watching` & `Star` & `Fork`**


### 介绍

本文主要讲述使用Rancher构建好Kubernetes集群后，如何在Kebernetes集群中快速构全自动的CI/CD流水， 包括自动签出代码、执行代码、构建Docker镜像、将Docker镜像发布到仓库、从Docker仓库拉取镜像并部署到集群等。
为完成构建流水线本文还会用到以下组件：

* Jenkins:管道的构建引擎。

* Docker Registry:开箱即用，构建发布步骤的默认目标是内部Docker仓库。但是，您可以进行配置以推送到远程仓库。内部Docker Registry只能从群集节点访问，用户无法直接访问。镜像不会在管道的生命周期之后持久存在，并且只应在管道运行中使用。如果您需要在管道运行之外访问您的镜像，请推送到外部仓库。

* Minio:Minio存储用于存储管道执行的日志。

> 托管的Jenkins实例无状态地工作，因此不要担心其数据持久性。默认情况下，Docker Registry和Minio实例使用临时卷，这对大多数用例来说都很好。如果要确保管道日志可以在节点故障中继续存在，则可以为它们配置持久卷



### 目标

使用Kubernetes容器集群构建CI/CD自动流水线，包括自动签出代码、执行代码、构建镜像、发布到仓库、将应用部署到集群等。
本文中容器集群包含以下资源：
* 1个Rancher Server节点：用于部署Rancher Server，通过该节点可以实现可视化多集群、跨云管理Kubernetes集群
* 2个Etcd节点：存储主控制节点和工作节点之间的任务调度等数据信息
* 2个控制(Controller)节点：部署Kunbernetes集群主控制节点，用于管理和监控Kubernetes其它的工作节点和存在状态信息。
* 2个工作(Worker)节点：部署Kubernetes集群的工作节点，用于运行容器化的应用。

> 注意：Etcd、Controller和Worker节点均选择至少两台是为了模拟高可用控制节点和工作节点。配置推荐至少2核CPU、4GB内存、40GB存储、2M带宽，系统均为Ubuntu 16.04 LTS。为达到更好的效果，本文创建的5台云服务器配置均为4核CPU、8GB内存、200GB存储、5M带宽，系统选择Ubuntu 16.04 LTS。


为完成CI/CD流水间自动构建与部署的操作，您还需要一个示例代码库和用于存放docker镜像的仓库：
* 本文提供了简单的基于go语言的示例代码库，您可以直接Fork使用： [CI/CD示例代码](https://github.com/anypm/rancher-pipeline-example-go)
* 本文的Docker镜像仓库使用的是滴滴云的镜像仓库，相关地址如下： [滴滴云容器镜像服务](https://app.didiyun.com/#/docker/repositories)

> 您也可以使用 [Docker官方仓库](hub.docker.com) 、 [Quay](quay.io) 或自建的私有仓库，


### 第一步：搭建Kubernetes集群

开始之前请确保已按上述**目标**中的资源要求准备好了服务器资源，可以是物理机、虚拟机或云主机，本文所有资源均为云主机。
关于如何准备Docker环境并搭建Kubernetes集群，请参考 [使用Rancher创建Kubernetes集群并进行多集群可视化管理](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/how-to-create-a-kubernetes-1-11-cluster-using-rancher-and-manage-clusters.md) ,本文将不再赘述。

完成此步骤后本文后续步骤将**默认您已经配置好一个Kubernetes集群**，含2个Etcd节点、2个控制(Controller)节点、2个工作(Worker)节点。


### 第二步：设置代码源

#### 1.创建项目
选择你的集群，然后在顶部菜单中选择 **「Projects/Namespaces」** ，输入 **「Project Name」** ，其他默认，点击 **「Create」** 即可创建集群下的项目，如下图：

![添加CICD示例项目](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/01-cicd-addproject.png)

#### 2.设置代码源
1)选择上一步创建的项目，顶部主菜单选择 **「Workloads」**，然后选择tab栏中的 **「Pipelines」**，点击 **「Configure Repositories」**，如下图：

![设置代码源](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/02-cicd-config-repo.png)

2) **「Repositories」** 页面中可以看到Rancher官方提供了三个示例代码，为了完整演示配置CI/CD流程，我们不用此处的示例代码，直接点击下方按钮 **「Authorize & Fetch Your Own Repositories」**，如下图:

![设置代码源](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/03-cicd-config-repo-auth01.png)

3)页面会跳转到选择源代码平台的页面，如下图。此处我们选择 **「Github」** 平台(GitLab与Bitbucket步骤类似)。此处注意要记录下 **“Homepage URL”** 和 **“Authorization callback URL”**，以备Github中应用授权使用。点击下方 **「click here」** 链接新开一个页面跳转至github应用授权页面。

![设置代码源](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/04-cicd-config-repo-auth02.png)

4)在Github平台的 **「OAuth Apps」** 页面中点击 **「Register a new application」**，如下图:

![注册app](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/04-cicd-config-repo-auth02.png)

5)在 **「Register a new application」** 自定义一个应用名称到 **"Application name"** ，在**“Homepage URL”** 和 **“Authorization callback URL”** 中填入 **“3)”** 对应的地址即可，点击 **「Register application」**。

![注册app2](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/06-cicd-config-repo-regnewapp02.png)

6)记录Github生成 **“Client ID”** 和 **"Client Secret"**,如下图:

![注册app3](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/07-cicd-config-repo-regnewapp03.png)

7)回到Rancher设置「Pipelines」页面中填入Github生成的 **“Client ID”** 和 **"Client Secret"** ，点击 **「Authenticate」** 进入授权页面：

![注册app4](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/08-cicd-config-repo-regnewapp04.png)

点击绿色按钮授权

![授权](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/09-cicd-config-repo-auth.png)

输入Github密码完成授权

![授权](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/10-cicd-config-repo-authpwd.png)


8)页面自动回到 **「Repositories」**，此时会自动拉取您在github上授权项目下所有的代码源，如图

![代码源列表](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/11-cicd-repo-open01.png)

找到 **“rancher-pipeline-example-go.git”**，点击 **「Enable」** 激活代码源授权

![激活代码源授权](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/12-cicd-repo-open02.png)

点击底部 **「Done」** 完成代码源授权

![激活代码源授权](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/13-cicd-repo-open03.png)

9) **「Pipelines」** 列表中会自动生成一个构建流水线，如图

![构建管道](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/14-cicd-pipeline-list.png)


### 第三步：设置Docker仓库

1)登陆滴滴云控制台，顶部导航依次选择 **「计算」** 和 **「容器镜像服务」**，可以看到 **「我的仓库」**，点击 **「创建仓库」**

![创建仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/15-cicd-didiyun-reg.png)

2)填写命名空间和仓库名称

![创建仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/16-cicd-didiyun-addrepo.png)

3)设置仓库访问密码，此密码用于从docker客户端登陆。在本文实操案例中会用于自动构建流水线将构建好的docker镜像上传至滴滴云docker镜像仓库中

![设置账户和密码](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/17-cicd-didiyun-reg-pwd.png)

4)回到集群页面，选择我们前面创建的项目，此案例中为 **「cicd-demo」**，选择顶导 **「Resources」** -> **「Registries」**

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/18-cicd-reg.png)

进入仓库授权列表页，点击 **「Add Registry」**

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/19-cicd-reg-list.png)


5)仓库授权列表页面点击 **「Add Registry」**，自定义填写仓库授权名称， **「Address」** 选择 **「Custom」**，填写滴滴云仓库访问地址，格式为hub.didiyun.com/命名空间， 此处 **"命名空间"** 为步骤 **"2)"** 中填写的命名空间名称。填写用户名和密码，此处用户名和密码为步骤 **“3)”**中设置的用户名和密码。

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/20-cicd-addreg.png)

设置完成后将自动返回列表页，我们会看到列表页多出一行仓库信息

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/21-cicd-addedreg.png)

回到 **「Pipelines」** 页面继续构建流水线


### 第四步：创建并定义流水线

1)在前面构建的pipeline下拉操作中选择 **「Edit Config」** ，进行进入pipeline编辑页

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/22-cicd-pipeline-edit.png)

2)在 **「Edit Pipeline Configuration」** 中可以看到已经创建好了四个步骤，分别是

* 从仓库获取代码
* 运行编译/构建代码(可选，此案例为必须项)
* 构建镜像并发布到私有仓库中
* 从仓库中拉取镜像并部署到集群中

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/23-cicd-pipeline-editview.png)

下面下面分别展示各步骤细节：

* 选择pipeline步骤类型、代码版本和编译脚本

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/24-cicd-pipline-edit-run.png)

* 选择构建与发布镜像选项，设置Dockerfile路径、镜像名称，勾选 **「Push image to remote repository」** ，下拉列表中选择前面设置好的滴滴云仓库地址

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/25-cicd-pipline-edit-publish.png)

* 选择部署YAML类型，并设置部署配置文件的路径

![设置仓库](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/26-cicd-pipline-edit-deploy.png)

3)设置构建提醒方式

![设置提醒](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/27-cicd-pipline-edit-notify.png)

4)设置流水线自动构建方式

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/28-cicd-pipline-edit-autobuild.png)

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/29-cicd-pipline-edit-autobuild.png)


> 本文为了让读者尽快成功体验自定义流水线的全流程，将案例代码、构建镜像的Dockerfile、用于部署的deployment.yaml都已经写好。您可以尝试根据自己的项目/代码编写合适的Dockerfile、deployment.yaml和pipeline流程


### 第五步：手动构建

* **「Pipelines」** 列表中选择前面设置的构建流水线，点击右边下拉选项中的 **「Run」**

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/30-cicd-pipline-edit-run.png)

* 选择构建分支

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/31-cicd-pipline-edit-run.png)

* 查看构建进度与详情

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/32-cicd-pipline-edit-run.png)

* 构建成功&构建日志

![自动构建](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/33-cicd-pipline-edit-run.png)

* 第一次构建会比较慢，后端会自动部署构建pipeline所必须的Jenkins引擎、Docker Registry和Minio,如下图

![构建引擎](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/34-cicd-pipline-edit-engine.png)

> 后续您的代码源有代码更新或版本更新时，Pipeline将根据您设置的自动构建策略自动发布新版本到集群应用中。

* 进入滴滴云控制台可以看到，每自动构建一次，镜像版本也会同时发布一次到docker镜像仓库中

![构建引擎](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/35-cicd-pipeline-images.png)


### 结论

基于Kubernetes集群可以非常便利的完成代码拉取、代码编译/构建、docker镜像构建、docker镜像分发(发布到私有或公有镜像仓库)、应用部署(从仓库拉取镜像并在集群部署应用)，且可以根据您设置的自动构建策略完成自动化流程，参考本文步骤您可以同时构建多条pipeline同时自动化构建您的应用程序，而无需关心细节。

### CI/CD相关概念
* 持续集成(Continuous Integration, CI):  代码合并，构建，部署，测试都在一起，不断地执行这个过程，并对结果反馈。
* 持续部署(Continuous Deployment, CD):　部署到测试环境、预生产环境、生成环境。　
* 持续部署(Continuous Delivery, CD):  将最终产品发布到生成环境、给用户使用。

### Rancher-Pipelines环境变量

环境变量名称 | 描述
:---- | :-----
CICD_GIT_REPO_NAME | 代码仓库名称(取自Github组织名称)	
CICD_PIPELINE_NAME | Pipeline名称
CICD_GIT_BRANCH | 本次触发的Git代码分支
CICD_TRIGGER_TYPE | 触发构建的事件
CICD_PIPELINE_ID | Pipeline的RancherID
CICD_GIT_URL | Github代码库的URL
CICD_EXECUTION_SEQUENCE | 构建的Pipeline编号
CICD_EXECUTION_ID | Pipeline的执行编号
CICD_GIT_COMMIT | 执行的Git提交ID



> 写在后面：据说完成应用发布只完成了整个产品或服务生命周期的一半甚至更少，后面将会有漫长的监控、运维与迭代过程，后续即将推出一系列监控与运维实践，此处先占坑并预留链接地址： **[《多Kubernetes集群和多租户环境中Prometheus搭建和使用监控指南》](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/how-to-create-use-prometheus-in-kubernetes-clusters.md)** 先欣赏几张Prometheus与Grafana监控容器集群的预览图

* 基于Prometheus的Kubernetes集群治标监控

![监控](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/36-prometheus.png)


* 基于Prometheus的Kubernetes节点指标监控

![监控](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/37-prometheus.png)


* 基于Grafana的Kubernetes集群

![监控](https://github.com/anypm/kubernetes-tutorials-series/blob/master/cicd-images/38-grafana.png)




尽情期待哦：）





