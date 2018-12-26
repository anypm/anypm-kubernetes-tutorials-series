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
* 本文提供了简单的基于go语言的示例代码库，您可以直接Fork使用：[CI/CD示例代码](https://github.com/anypm/rancher-pipeline-example-go)
* 本文的Docker镜像仓库使用的是滴滴云的镜像仓库，相关地址如下：[滴滴云容器镜像服务](https://app.didiyun.com/#/docker/repositories)

> 您也可以使用[Docker官方仓库](hub.docker.com)、[Quay](quay.io)或自建的私有仓库，


### 第一步：搭建Kubernetes集群

开始之前请确保已按上述**目标**中的资源要求准备好了服务器资源，可以是物理机、虚拟机或云主机，本文所有资源均为云主机。
关于如何准备Docker环境并搭建Kubernetes集群，请参考[使用Rancher创建Kubernetes集群并进行多集群可视化管理](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/how-to-create-a-kubernetes-1-11-cluster-using-rancher-and-manage-clusters.md),本文将不再赘述。

完成此步骤后本文后续步骤将**默认您已经配置好一个Kubernetes集群**，含2个Etcd节点、2个控制(Controller)节点、2个工作(Worker)节点。


### 第二步：设置代码源


### 第三步：设置Docker仓库



### 第四步：创建并定义流水线


### 结论


### CI/CD相关概念
* 持续集成(Continuous Integration, CI):  代码合并，构建，部署，测试都在一起，不断地执行这个过程，并对结果反馈。
* 持续部署(Continuous Deployment, CD):　部署到测试环境、预生产环境、生成环境。　
* 持续部署(Continuous Delivery, CD):  将最终产品发布到生成环境、给用户使用。
