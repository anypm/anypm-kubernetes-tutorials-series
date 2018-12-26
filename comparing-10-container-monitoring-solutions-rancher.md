# 容器领域的十大监控系统对比

容器监测环境有多种形态和大小。有些是开源的，而另一些则是商业性质的。有些可以借助平台一键部署（例如在Rancher容器管理平台的应用目录中一键部署这些监控应用），而另一些则需要手动配置。有些是通用的，有些是专门针对容器环境的。有些托管在公有云中，而另一些则需要在自己的集群主机上安装。

在本文中，我将对容器领域的10个监控解决方案进行全面的分析对比。监控解决方案的数量之多令人望而生畏。新的解决方案不断涌现，同时现有的解决方案不断发展。我没有深入研究每个解决方案，而是采取了high-level的对比方法。通过这种方法，读者可以“缩小列表”，并能针对自身需求进行更认真的评估，从而选出最适合的解决方案。

** 本文将介绍并对比的监控解决方案包括：**

* [原生Docker](https://docs.docker.com/engine/reference/commandline/stats/)
* [cAdvisor](https://github.com/google/cadvisor)
* [Scout](https://scoutapp.com/)
* [Pingdom](https://www.pingdom.com/)
* [Datadog](https://www.datadoghq.com/)
* [Sysdig](https://sysdig.com/)
* [Prometheus](https://prometheus.io/)
* [Heapster / GrafanaPingdom](https://github.com/kubernetes-retired/heapster)
* [ELK](https://www.elastic.co/cn/)
* [Sensu](https://sensu.io/)

我提出了一个对比监控解决方案的架构，并对每个解决方案进行了高级对比，然后通过讨论每个解决方案将如何与Rancher协同工作，从而更详细地讨论每个解决方案的细节。我还会在最后谈谈一些其它的监控解决方案，这些方案未被纳入本文的“十大”中，但你也可能遇到过。

## 对比架构
客观地对比监控解决方案面临的一个挑战是，解决方案的架构、功能、部署模型和成本可能会有很大的差异。一个解决方案可以从单个主机提取和绘制docker相关的数据，而另一个解决方案则可以从许多主机收集数据，测量应用程序响应时间，并在特定条件下发送自动警报。

在对比解决方案时，先确定一个对比架构，将会为后期的对比工作带来很大帮助。我有些武断地提出了如下图的对比架构，以大多数监控解决方案都具有的功能层,来作为我对比的基础。这个对比架构可以分为7层：

![seven_layers_of_docker_monitoring](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/monitering-images/seven_layers_of_docker_monitoring.png)

* **主机代理**——主机代理代表监控解决方案的“肢体”，它会从各种来源（如API和日志文件）提取时间序列数据。主机代理通常安装在每个集群主机上（无论是本地还是云端），并且它们通常被打包成Docker容器，以便部署和管理。

* **数据收集架构**——虽然单主机数据有时很有用，但管理员可能需要所有主机和应用程序的统一视图。监控解决方案通常具备一些机制来收集每个主机的数据，并将其保存在共享数据存储区中。

* **数据存储**——数据存储可能是传统的数据库，但更常见的一种形式是可伸缩的分布式数据库，由键值对组成的时间序列数据进行了优化。有些解决方案具有原生数据存储，而其他解决方案则使用的是开源的数据存储插件。

* **聚合引擎**——要存储来自数十个主机的原始数据，可能遇见的一大问题是数据量会变得过大。监控架构通常提供数据聚合功能，定期将原始数据转换为统一的度量标准(比如每小时或每日进行汇总)，清除不再需要的旧数据，或以某种方式重新分解数据，以支持预期的查询和分析。

* **过滤和分析**——一个监控解决方案就像是你从数据中获得的洞察力。不同的监控解决方案之间，筛选和分析的能力常常差别很大。有些解决方案仅支持以简单的时间序列图表的形式来进行一些预先打包的查询, 而另一些则具有可自定义的仪表板、嵌入式查询语言和复杂的分析功能。

* **可视化层**——监控工具通常具有可视化层，用户可以在其中与 web 界面进行交互以生成图表、制定查询以及在某些情况下定义警报条件。可视化层可能与筛选和分析功能紧密耦合, 也可能根据解决方案而与其分开。

* **告警和通知**——很少管理员有时间整天坐着、时刻关注监控图表。监控系统的另一个常见特性是告警子系统，如果达到或超过了预定义的阈值，可以向管理员发出通知。

除了解每个监控解决方案如何实现上述基本功能之外，如下方面也是用户在选择监控解决方案时应该注意与考量的：
* 解决方案的完整性
* 是否易于安装和配置
* 关于web用户界面的详细信息
* 是否能够将警报转发至外部服务
* 社区支持和参与程度（若该方案为开源项目）
* Rancher应用程序目录中的可用性
* 支持监控非容器环境和应用程序
* 原生Kubernetes支持（Pods、Services、Namespaces等）
* 可扩展性（API，其他接口）
* 部署模式（自主托管、云上托管）
* 成本

## 10大监控解决方案的对比

下图以high-level的形式展示了我们提出的的10个监控解决方案如何对应我们在上文提出的七层对比模型，哪些组件实现了每层功能，以及组件的所在位置。每个框架都是极其复杂的，下图的对比方式毋庸置疑是一种简化，不过它也会给大家提供一个有用的视角来了解各个组件的功能。
![gord_comparing_ten_monitoring_solutions](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/monitering-images/gord_comparing_ten_monitoring_solutions-01.png)

下图的摘要介绍了每个监控解决方案的附加属性。其中某些解决方案有多个部署选项，所以它们之间的对比就变得更加细微。

![gord_monitoring_functionality_comparison](https://github.com/anypm/anypm-kubernetes-tutorials-series/blob/master/monitering-images/gord_monitoring_functionality_comparison-02.png)

## 各解决方案的深入研究

### 1.DOCKER STATS
https://www.docker.com/docker-community 
Docker通过docker stats命令为Docker主机提供了内置命令监控功能。管理员可以查询Docker守护进程，并获取有关容器资源消耗数据的详细实时信息，包括CPU和内存使用、磁盘和网络I/O以及正在运行的进程数。Docker stats利用Docker引擎API来检索这些信息。Docker统计信息没有历史概念，它只能监控单个主机，但聪明的管理员可以编写脚本从多个主机收集数据。

Docker stats本身用处有限，但Docker统计数据可以与其他数据源（如Docker日志文件和Docker事件）结合使用，以满足更高级别的监控服务。Docker只能得到单个主机报告的数据，所以Docker stats对于使用多主机应用程序服务的Kubernetes或对Swarm集群进行监控的能力有限。由于没有可视化界面，没有聚合，没有数据存储，也无法从多个主机收集数据，所以Docker的统计数据对我们的七层模型来说并不太适用。由于Rancher在Docker上运行，Rancher用户可以自动使用基本的docker stats功能。

### 2.CADVISOR
https://github.com/google/cadvisor 
cAdvisor是一个开源项目，好比 Docker stats向用户提供关于运行容器的资源使用信息。cAdvisor最初是由谷歌开发的，用于管理其lmctfy容器，但它现在也支持Docker。作为守护进程，它收集、聚集、处理和导出关于运行容器的信息。

cAdvisor有一个web界面，可以生成多个图表，但是像Docker stats一样，它只监控一个Docker主机。它可以作为容器安装在Docker machine上，也可以在Docker主机本身上安装。

cAdvisor本身只保留60秒的信息。需要将cAdvisor设置为将数据记录到外部数据存储库中。常用于cAdvisor数据的数据存储库包括Prometheus和InfluxDB。虽然cAdvisor本身并不是一个完整的监控解决方案，但它通常是其他监控解决方案的组成部分。在Rancher版本1.2前，Rancher在Rancher agent中嵌入了cAdvisor(用于Rancher的内部使用)，但现在已经不是这样了。最新版本的Rancher使用Docker统计来收集通过Rancher UI公开的信息，因为它们可以减少开销。

管理员可以轻松地在Rancher上部署cAdvisor，它是几个综合监控堆栈的一部分，但是cAdvisor不再是Rancher本身的一部分。

### 3.SCOUT
http://scoutapp.com 
Scout是一家总部位于科罗拉多州的公司，它提供基于云的应用程序和数据库监控服务，主要针对Ruby和Elixir环境。其现有的监控和警报架构使其能够监控Docker容器。

我们提到Scout，因为之前在比较监控Docker的解决方案时就提到了它。通过灵活的告警和与第三方告警服务的集成，Scout提供全面的数据收集、过滤和监控功能。

Scout的团队提供了如何使用Ruby和StatsD编写脚本的指导，以利用Docker Stats API、Docker Event API和传递数据来监控这些脚本。他们还打包了一个Docker – scout容器，可以在Docker Hub(scoutapp / Docker – scout)上使用，这就使安装和配置scout代理变得更简单。易用性取决于用户是自行配置StatsD代理，还是使用打包的docker-scout容器。

作为一种托管云服务，ScoutApp可以在快速启动并运行容器监控解决方案时省去许多麻烦。如果您正在部署Ruby应用程序或运行Scout支持的数据库环境，使用Scout解决方案，可以帮助您很好地整合您的Docker、应用程序和数据库级别的监控。

但是，用户可能需要注意一些事项。在大多数服务级别上，该平台只允许保留30天的数据，而不是每个被监控的主机。至于价格，每月定价的标准套餐价格从99美元到299美元不等。这一开箱即用的解决方案只能提取和传递有限的指标，也不太适用于Kubernetes相关的监控。此外，虽然docker-scout在Docker Hub上可用，但开发是由Pingdom完成的，在过去的两年中，Scout的代理组件只有很小的更新。

Rancher自身并不默认原生支持Scout，但由于Scout是云服务，所以它在Rancher中很容易部署和使用，特别是当使用基于容器的代理时。目前，docker-scout代理不在Rancher应用程序目录中。

### 4.PINGDOM
http://pingdom.com 
上文中我们提到Scout作为云托管的应用程序，因此还需要提到一个类似的解决方案，称为Pingdom。Pingdom是由德克萨斯州奥斯汀市的SolarWinds公司运营的托管云服务，它是一家专注于监控IT基础架构的公司。Pingdom的主要用例是网站监控，作为服务器监控平台的一部分，Pingdom提供了大约90个插件。事实上，Pingdom维护docker-scout，同样地，Scout也使用 StatsD代理。

Pingdom值得关注的原因在于，它灵活的定价方案似乎更适合监控Docker环境。用户可以根据计划收集到的StatsD数据数（每10个数据每月要价1美元）在基于每个服务器的计划之间进行选择。Pingdom易于设置和管理，对于需要一个完整的监视解决方案的用户以及希望监控容器管理平台之外的其他服务的用户而言，Pingdom非常合适。像Scout一样，Pingdom是一种云服务，并且易于同Rancher结合使用。

### 5.DATADOG
https://www.datadoghq.com/ 
Datadog是另一个类似于Scout和Pingdom的商业托管云监控服务。 Datadog还提供了一个Dockerized agent，用于在每个Docker主机上进行安装；然而，Datadog并没有像前面提到的云监控解决方案那样使用StatsD，而是开发了一种增强的StatsD，称为DogStatsD。Datadog代理收集并传递Docker API提供的完整数据，从而进行更详实、细致的监控。

虽然Datadog不能原生支持Rancher，但是Rancher UI中有Datadog目录，用户可以在Rancher上轻松地安装和配置Datadog agent。用户还可以使用Rancher 标签，Datadog中的报告反映了您在Rancher中用于主机和应用程序的标签。与前面提到的云服务相比，Datadog能够提供更好的数据访问权限和更精细的定义警报条件。与其他服务一样，Datadog也可用于监视其他服务和应用程序，并拥有超过200个集成的库。Datadog还能保留18个月的全分辨率数据，这比云服务要更长。

与其他云服务相比，Datadog的优势在于它具有超越Docker的集成功能，并且可以从Kubernetes、Mesos、etcd和其他在您的Rancher环境中运行的服务中收集数据。对于在Rancher上运行Kubernetes的用户来说，这种多功能性是很重要的，因为他们希望能够监控诸如Kubernetes pods、服务、名称空间和kubelet health之类的数据。Datadog-Kubernetes监控解决方案通过Kubernetes中的DaemonSets，能够自动将数据收集代理部署到每个集群节点。

Datadog的定价为每台主机每月约15美元，总价会根据用户需要的服务和每个主机监控的容器数量相应增加。


### 6.SYSDIG
http://sysdig.com

Sysdig是一家加州公司，为用户提供基于云计算的监控解决方案。与前文所描述的几个基于云的监控解决方案不同的是，Sysdig更专注于监控容器环境，包括Docker、集群、Mesos和Kubernetes。此外，Sysdig还在开源项目中提供了一些可用功能，并且可以选择对Sysdig监控服务进行云部署还是本地部署。在这些方面，Sysdig不同于迄今为止所出现的其他基于云的解决方案。

Sysdig，与Datadog类似，其目录可用于Rancher，但Sysdig的本地和云安装都有单独目录。从Rancher Catalog里自动安装的Sysdig无法用于对Kubernetes的监控；不过，它也可以不通过Rancher Catalog来安装到Rancher之上。商用Sysdig监控具有Docker监控、告警和故障排除功能，并且还具有 Kubernetes、Mesos和集群识别的功能。Sysdig能够自动识别Kubernetes Pod和服务，因此选择Kubernetes作为Rancher的编排架构将是一个很好的解决方案。

Sysdig和Datadog一样是按每个主机每月定价。虽然Sysdig入门价格略高，但它每个主机上可以支持更多容器，因此根据用户的环境，实际定价可能非常相似。 Sysdig还提供了一个全面的CLI——csysdig，将其与一些产品区分开来。


### 7.PROMETHEUS
http://prometheus.io

Prometheus是一个很受欢迎的开源监控和警报工具包，它最初是在SoundCloud进行构建的。现在是CNCF项目，也是该公司在Kubernetes之后的第二个托管项目。作为一个工具包，它与目前为止所描述的其他监视解决方案有很大不同。首先一个主要的区别是，Prometheus，作为一种云服务，是模块化的，可以自行托管，这意味着无论是在本地还是在云端，用户都可以在他们的集群上部Prometheus。

值得注意的是，Prometheus不是将数据推送到云服务，而是安装在每个Docker主机上，并通过HTTP从Prometheus提供的各种输出口获取或“抓取”数据。其中，一些输出口被官方保留为Prometheus GitHub项目的一部分，而另一些则是由外部贡献。有些项目本身暴露了Prometheus数据，因此不需要输出口。由于Prometheus可高度扩展，用户需要考虑输出方的数量，并根据收集的数据量适当地配置轮询间隔。

Prometheus的服务器从各种来源检索时间序列数据，并将数据存储在其内部数据存储区中。此外，Prometheus提供服务发现等功能，这是一种针对特定类型数据的独立推送网关，并且有一个嵌入的查询语言(PromQL)，该语言擅长查询多维数据。同时，它也有一个嵌入式的Web UI和API。虽然Prometheus中的Web UI提供了强大的功能，但用户必须对PromQL十分了解，因此一些站点更愿意使用Grafana作为绘制和查看集群相关指标的接口。

Prometheus有一个独立的告警管理器，它也具有独特的UI，并且可以处理存储在Prometheus中的数据。和其他告警管理器一样，它可以与各种外部告警服务一起工作，包括电子邮件、Hipchat、Pagerduty、#Slack、OpsGenie、VictorOps等。

由于Prometheus由许多组件组成，输出方需要根据所监控的服务进行选择和安装，所以安装起来比较困难，但是作为免费产品，在价格上Prometheus具有无可比拟的优势。

虽然不像 Datadog或 Sysdig这样精炼，但是Prometheus提供了类似的功能、广泛的第三方软件集成以及一流的云监控解决方案，并且Prometheus十分了解 Kubernetes和其他容器管理架构。另外，由Infinityworks开发的Rancher Catalog中的条目使得在使用Cattle作为Rancher的编排架构时，Prometheus更容易入门，但由于配置选项的种类繁多，管理员需要花费一些时间才能正确安装和配置。

Infinityworks提供了一些有用的插件，其中包括prometheus-rancher-exporter，这些插件将Rancher stack和从Rancher API获得的主机的健康状况发送给Prometheus兼容端点。因此，Prometheus对于那些愿意花更多精力的管理者来说是最强大的监控解决方案之一。


### 8.HEAPSTER
https://github.com/kubernetes/heapster

Heapster是Kubernetes旗下的一个项目，它有助于实现容器集群监控和性能分析。此外，Heapster对Kubernetes和OpenShift的支持十分良好，也很适用于在Rancher上使用Kuberenetes作为编排工具的用户。Cattle或者Swarm的用户则通常不会选择它。

人们经常将Heapster定义为一个监控解决方案，但更确切地说，它应该是一个“集群范围内的监控和事件数据聚合器”。Heapster从来不单独部署，相反，它是一堆开源组件的一部分。Heapster监控堆栈通常由以下部分组成：

数据收集层：例如，在每个集群主机上使用kubelet访问的cAdvisor
可插入式存储后端：例如，ElasticSearch、InfluxDB、Kafka、Graphite等
数据可视化组件：Grafana或Google Cloud Monitoring
Heapster与InfluxDB、Grafana共同组成了一个流行的堆栈，当用户在Rancher上部署Kubernetes时，此组合便会默认安装在Rancher上。
需要注意的是，这些组件被认为是Kubernetes的附加组件，因此它们可能不会被自动部署到所有Kubernetes发行版中。

InfluxDB受欢迎的其中一个原因是，它是少数几个支持Kubernetes项目和数据的数据后端之一，并且可以对Kubernetes进行更全面的监控。

值得注意的是，Heapster本身不支持在商用云的解决方案或Prometheus中发现的与应用程序性能管理（APM）相关的告警或服务。需要监控服务的用户可以使用Hawkular来弥补这一不足，不过Hawkular并不会自动配置为Rancher部署的一部分，而是需要用户另行操作。


### 9.ELK STACK
https://www.elastic.co/

另一个可用于监视容器环境的开源软件栈是ELK，由Elastic提供的三个开源项目组成。ELK是通用的，广泛用于各种分析应用程序，日志文件监控是其中关键的一环。ELK以其关键组件的首字母命名:

Elasticsearch：基于Lucene的分布式搜索引擎
Logstash：一个数据处理管道，用于获取数据并将其发送到Elastisearch（或其他“托盘”）
Kibana：Elasticsearch的可视化搜索仪表板和分析工具
Elastic栈中一个容易被忽视的成员是Beats，项目开发人员将其描述为“轻量级数据托运器”。现在有许多现成的Beats托运器，包括Filebeat（用于日志文件）、Metricbeat（用于收集各种来源的数据）以及用于简单的uptime监控等。

ELK栈的部署方式有所不同。Kiratech的Lorenzo Fontana在这篇文章中解释了如何使用cAdvisor从Docker Swarm主机收集数据以存储在ElasticSearch中，并使用Kibana进行分析：https://blog.codeship.com/moni … isor/。在另一篇文章中，Aboullaite Mohammed描述了一个不同的用例，其重点是收集Docker日志文件，分析各种Linux和NGINX日志文件（error.log、access.log和syslog）：https://aboullaite.me/docker-m … tack/。有些商用ELK栈提供者，例如logz.io和Elastic Co，向用户提供“ELK即服务”，在原生ELK之外补充提供了告警功能。有关在Docker上使用ELK的更多信息，请访问https://elk-docker.readthedocs.io/。

对于希望尝试使用ELK的Rancher用户，它在Rancher Catalog中已有条目，《如何在Rancher上运行Elasticsearch》一文介绍了如何在Rancher Catalog中部署ELK。《使用容器和Elasticsearch集群对Twitter进行监控》一文介绍了如何使用ELK监控Twitter数据。尽管博洽多闻的管理员可以使用ELK进行容器监控，但与Sysdig、Prometheus或Datadog等直接针对容器监控的解决方案相比，ELK的上手和使用难度都会更大。


### 10.SENSU
http://sensuapp.org

Sensu是一个通用的自主监控解决方案，支持多种监控应用。用户可在MIT许可下获得一个免费的Sensu Core版本，Sensu的企业版则拥有更多的附加功能，价格为每月99美元，可以为50个Sensu客户端提供服务。Sensu使用术语“客户端”来指代其监控代理，因此根据您监控的主机和应用程序环境的数量，企业版可能会变得非常昂贵。Sensu在容器管理之外还拥有非常强大的功能，但就监控容器环境和容器化应用程序这方面来看，它与其他平台并无差别。

Sensu插件的数量持续增长，现在已有数十个Sensu和社区支持的插件可以从各种来源提取数据。2015年Rancher对Sensu进行早期评估时，那时Sensu用户要从Docker中提取信息，需要开发shell脚本。但是现在，Sensu已经有了一个不错的Docker插件，这使得Sensu更易于使用了。

插件往往是用Ruby编写的，使用基于gem的安装脚本，这些脚本需要在Docker主机上运行。用户可以在他们选择的语言中开发额外的插件。与我们讨论过的其他监控解决方案相同的是，Sensu插件不是部署在自身容器中。(这一点毫无疑问，因为Sensu并非在监控容器的基础上构建的。)

由于不同的用户希望根据自己的监控要求混合和匹配插件，因此为每个插件设置单独的容器将会非常棘手，这可能是为什么不使用容器进行部署的原因。不过，插件可以使用Chef、Puppet和Ansible等平台进行部署。例如，对于Docker来说，有6个独立的插件可以从各种来源收集与Docker相关的数据，包括Docker统计信息、容器数量、容器运行状况、Docker ps等等。Sensu插件的数量非常多，包括许多用户可能在容器环境（ElasticSearch、Solr、Redis、MongoDB、RabbitMQ、Graphite和Logstash等）中运行的应用程序栈。此外，Sensu还提供用于管理和编排架构的插件，如AWS服务（EC2、RDS、ELB）。但是在插件列表中，Kubernetes似乎消失了。但是Sensu提供对OpenStack和Mesos的支持。

Sensu通过RabbitMQ使用消息总线，以协助代理/客户端与Sensu服务器之间的通信。Sensu用 Redis存储数据，但它的设计目的是将数据路由到外部的时间序列数据库。支持的数据库包括Graphite、Librato和InfluxDB。

安装和配置Sensu需要花点功夫。安装Sensu前必须先安装Redis和RabbitMQ。 Sensu服务器、Sensu客户端和Sensu仪表板需要单独安装，并且根据部署的是Sensu内核还是企业版本，流程也会有所不同。如前所述，Sensu不提供容器友好的部署模型。为了方便起见，可以使用Docker镜像(hiroakis／Docker-sensu-server)运行redis、rabbitmq-server、uchiwa(开源web层)和Sensu服务器组件，但在评估上，这个软件包比生产部署更有用。

Sensu的特性非常多，但对容器用户而言，它的缺点是架构很难安装、配置和维护，因为这些组件本身没有被Docker化。此外，许多告警功能（例如发送警报给诸如PagerDuty，Slack或HipChat等服务）可以在基于云的解决方案或像Prometheus这样的开源代码解决方案中使用，因此需要购买Sensu企业版许可。特别是若你使用Kubernetes，可能Sensu不是最好的选择。

### 我们忽略的监控解决方案
Graylog是另一个监控Docker的开源解决方案。和ELK一样，Graylog也适用于Docker日志文件分析。它可以接受和解析来自多个数据源的日志和事件数据，并支持像Beats、Fluentd和NXLog这样的第三方收集器。

Nagios通常被认为更适合于监控集群主机而不是容器，但对于那些在监控集群环境中浸淫已久的人来说，Nagios最受欢迎。

Netsil是一家硅谷初创公司，作为一个监控应用程序，它为Docker、Kubernetes、Mesos以及各种应用程序和云提供商提供插件。Netsil的应用运营中心（AOC）与我们讨论的其它监控架构一样，以云/SaaS或自主托管的形式为云应用服务提供架构感知监控。


### 结语
容器监控解决方案很多，新的解决方案不断涌现，同时现有的解决方案不断发展。此次我们采取了high-level的对比方法，希望可以帮助您 “缩小列表”，针对自身需求进行更认真的评估，从而选出最适合的解决方案。

> 原文：http://rancher.com/comparing-10-container-monitoring-solutions-rancher/
