## 多Kubernetes集群和多租户环境中搭建和使用Prometheus监控指南


### 介绍

先了解一些背景内容。应用程序运行在容器上并由Kubernetes负责调度，在此环境中它们是高度自动化并且动态的。传统的监控工具一般是基于服务器，只监控静态的服务，所以当要在这种动态环境监控应用程序时，传统的监控工具往往很难满足这一需求。

这时就需要Prometheus出马了。

Prometheus是一个开源项目，最初由SoundCloud的工程师开发。它专门用于监控那些运行在容器中的微服务。每经过一个时间间隔，数据都会从运行的服务中流出，存储到一个时间序列数据库中，这个数据库之后可以通过PromQL语言查询。另外，因为数据是以时间序列存储的，当出现问题时，可以根据这些时间间隔进行诊断，另外还可以预测基础设施的长期监控趋势—-这是Prometheus的两大功能。



### 目标

### 第一步：搭建kubernetes集群


### 第二步：部署prometheus监控

### 启用prometheus的另一种快捷方法

```
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:v2.2.0-alpha3
```

### 第三步：prometheus使用指南


### 第四步：grafana使用指南






### 结论
