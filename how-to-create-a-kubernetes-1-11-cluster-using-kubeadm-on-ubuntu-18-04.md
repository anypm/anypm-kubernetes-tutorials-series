# 如何在Ubuntu 18.04上使用Kubeadm创建Kubernetes 1.11集群

## 目标

您的群集将包含以下物理资源：

* 一个Master节点
主节点（Kubernetes中的节点指服务器）负责管理集群的状态。它运行Etcd，它在将工作负载调度到工作节点的组件之间存储集群数据。


* 两个Worker节点
工作节点是运行工作负载（即容器化应用程序和服务）的服务器。一旦工作人员分配了工作负载，工作人员将继续运行您的工作负载，即使主计划在调度完成后停止工作也是如此。通过添加工作人员可以增加群集的容量。


完成本指南后，您将拥有一个可以运行容器化应用程序的集群，前提是集群中的服务器具有足够的CPU和RAM资源供应用程序使用。几乎任何传统的Unix应用程序（包括Web应用程序，数据库，守护程序和命令行工具）都可以进行容器化，并在集群上运行。群集本身将在每个节点上消耗大约300-500MB的内存和10％的CPU。

设置群集后，您将部署Web服务器Nginx以确保它正确运行工作负载。


## 先决条件

* 本地Linux / macOS / BSD计算机上的SSH密钥对。如果您之前没有使用过SSH密钥，可以按照如何在本地计算机上设置SSH密钥的说明来学习如何设置它们。
* 运行Ubuntu 16.04且内存至少为1GB的三台服务器。您应该能够以SSH密钥对的root用户身份SSH到每个服务器。
* Ansible安装在您的本地计算机上。如果您正在运行Ubuntu 18.04作为您的操作系统，请按照如何在Ubuntu 18.04上安装和配置Ansible中的“步骤1 - 安装Ansible”部分来安装Ansible。有关其他平台（如macOS或CentOS）的安装说明，请遵循Ansible官方安装文档。
* 熟悉Ansible剧本。有关查看，请查看配置管理101：编写Ansible Playbooks。
* 了解如何从Docker镜像启动容器。如果需要复习，请参阅如何在Ubuntu 18.04上安装和使用Docker中的“步骤5 - 运行Docker容器” 。

## 第1步 - 设置工作区目录和Ansible清单文件


在本节中，您将在本地计算机上创建一个用作工作区的目录。您将在本地配置Ansible，以便它可以与远程服务器上的命令进行通信并执行命令。完成后，您将创建一个hosts包含库存信息的文件，例如服务器的IP地址和每个服务器所属的组。

在三台服务器中，一台服务器将显示为主服务器`master_ip`。另外两台服务器将是工作人员，并拥有IP `worker_1_ip`和`worker_2_ip`。

创建一个`~/kube-cluster`在本地计算机的主目录中指定的目录并cd进入该目录：

```
$ mkdir ~/kube-cluster
$ cd ~/kube-cluster
```

该目录将是本教程其余部分的工作区，并包含所有Ansible剧本。它也将是您将在其中运行所有本地命令的目录。

创建一个名为`~/kube-cluster/hosts`using `nano`或您喜欢的文本编辑器的文件：

```
$ nano ~/kube-cluster/hosts
```

将以下文本添加到文件中，该文件将指定有关集群逻辑结构的信息：

>〜/ KUBE群集/主机

```
[masters]
master ansible_host=master_ip ansible_user=root

[workers]
worker1 ansible_host=worker_1_ip ansible_user=root
worker2 ansible_host=worker_2_ip ansible_user=root

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

您可能还记得Ansible中的库存文件用于指定服务器信息，例如IP地址，远程用户和服务器分组，以作为执行命令的单个单元进行目标。~/kube-cluster/hosts将是您的库存文件，并且您已向其添加了两个Ansible组（主服务器和工作服务器），用于指定集群的逻辑结构。


在主服务器组中，有一个名为“master”的服务器条目，其中列出了主节点的IP（`master_ip`），并指定Ansible应以root用户身份运行远程命令。

同样，在workers组中，有两个工作服务器（`worker_1_ip`和`worker_2_ip`）条目，它们也指定`ansible_user`为root用户。

该文件的最后一行告诉Ansible使用远程服务器的Python 3解释器进行管理操作。

添加文本后保存并关闭文件。

使用组设置服务器清单后，我们继续安装操作系统级依赖关系并创建配置设置。

## 步骤2 - 在所有远程服务器上创建非root用户

在本节中，您将在所有服务器上创建一个具有sudo权限的非root用户，以便您可以作为非特权用户手动SSH到它们。例如，如果您希望通过命令查看系统信息（例如top/htop，查看正在运行的容器列表或更改root拥有的配置文件），这将非常有用。这些操作通常在维护群集期间执行，并且使用非root用户执行此类任务可以最大程度地降低修改或删除重要文件或无意中执行其他危险操作的风险。

创建`~/kube-cluster/initial.yml`工作空间中指定的文件：

```
$ nano ~/kube-cluster/initial.yml
```

接下来，将以下播放添加到该文件以创建在所有服务器上具有sudo权限的非root用户。Ansible中的游戏是针对特定服务器和组执行的一系列步骤。以下播放将创建一个非root sudo用户：

> 〜/ KUBE群集/ initial.yml

```
- hosts: all
  become: yes
  tasks:
    - name: create the 'ubuntu' user
      user: name=ubuntu append=yes state=present createhome=yes shell=/bin/bash

    - name: allow 'ubuntu' to have passwordless sudo
      lineinfile:
        dest: /etc/sudoers
        line: 'ubuntu ALL=(ALL) NOPASSWD: ALL'
        validate: 'visudo -cf %s'

    - name: set up authorized keys for the ubuntu user
      authorized_key: user=ubuntu key="{{item}}"
      with_file:
        - ~/.ssh/id_rsa.pub
```

这是这个剧本的作用细分：

* 创建非root用户ubuntu。
* 配置sudoers文件以允许ubuntu用户在sudo没有密码提示的情况下运行命令。
* 将本地计算机中的公钥（通常~/.ssh/id_rsa.pub）添加到远程ubuntu用户的授权密钥列表中。这将允许您以ubuntu用户身份SSH到每个服务器。

添加文本后保存并关闭文件。

接下来，通过本地运行执行playbook：

```
ansible-playbook -i hosts ~/kube-cluster/initial.yml

```
该命令将在两到五分钟内完成。完成后，您将看到类似于以下内容的输出：
```
Output
PLAY [all] ****

TASK [Gathering Facts] ****
ok: [master]
ok: [worker1]
ok: [worker2]

TASK [create the 'ubuntu' user] ****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [allow 'ubuntu' user to have passwordless sudo] ****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [set up authorized keys for the ubuntu user] ****
changed: [worker1] => (item=ssh-rsa AAAAB3...)
changed: [worker2] => (item=ssh-rsa AAAAB3...)
changed: [master] => (item=ssh-rsa AAAAB3...)

PLAY RECAP ****
master                     : ok=5    changed=4    unreachable=0    failed=0   
worker1                    : ok=5    changed=4    unreachable=0    failed=0   
worker2                    : ok=5    changed=4    unreachable=0    failed=0   
```

现在初步设置已完成，您可以继续安装特定于Kubernetes的依赖项。


## 第3步 - 安装Kubernetetes的依赖项

在本节中，您将使用Ubuntu的软件包管理器安装Kubernetes所需的操作系统级软件包。这些包是：

* Docker - 容器运行时。它是运行容器的组件。Kubernetes正在积极开发对rkt 等其他运行时的支持。
* kubeadm - CLI工具，以标准方式安装和配置群集的各个组件。
* kubelet - 在所有节点上运行并处理节点级操作的系统服务/程序。
* kubectl - 用于通过其API服务器向集群发出命令的CLI工具。

创建`~/kube-cluster/kube-dependencies.yml`工作空间中指定的文件：

```
$ nano ~/kube-cluster/kube-dependencies.yml
```

将以下播放添加到文件以将这些包安装到您的服务器：

> 〜/ KUBE群集/ KUBE-dependencies.yml

```
- hosts: all
  become: yes
  tasks:
   - name: install Docker
     apt:
       name: docker.io
       state: present
       update_cache: true

   - name: install APT Transport HTTPS
     apt:
       name: apt-transport-https
       state: present

   - name: add Kubernetes apt-key
     apt_key:
       url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
       state: present

   - name: add Kubernetes' APT repository
     apt_repository:
      repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
      state: present
      filename: 'kubernetes'

   - name: install kubelet
     apt:
       name: kubelet
       state: present
       update_cache: true

   - name: install kubeadm
     apt:
       name: kubeadm
       state: present

- hosts: master
  become: yes
  tasks:
   - name: install kubectl
     apt:
       name: kubectl
       state: present
```

剧本中的第一部戏剧如下：

* 安装Docker，容器运行时。
* 安装`apt-transport-https`，允许您将外部HTTPS源添加到APT源列表。
* 添加Kubernetes APT存储库的apt-key进行密钥验证。
* 将Kubernetes APT存储库添加到远程服务器的APT源列表中。
* 安装`kubelet`和`kubeadm`。

第二个游戏包含安装kubectl在主节点上的单个任务。

完成后保存并关闭文件。

接下来，通过本地运行执行playbook：

```
$ ansible-playbook -i hosts ~/kube-cluster/kube-dependencies.yml
```

完成后，您将看到类似于以下内容的输出：

```
Output
PLAY [all] ****

TASK [Gathering Facts] ****
ok: [worker1]
ok: [worker2]
ok: [master]

TASK [install Docker] ****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [install APT Transport HTTPS] *****
ok: [master]
ok: [worker1]
changed: [worker2]

TASK [add Kubernetes apt-key] *****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [add Kubernetes' APT repository] *****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [install kubelet] *****
changed: [master]
changed: [worker1]
changed: [worker2]

TASK [install kubeadm] *****
changed: [master]
changed: [worker1]
changed: [worker2]

PLAY [master] *****

TASK [Gathering Facts] *****
ok: [master]

TASK [install kubectl] ******
ok: [master]

PLAY RECAP ****
master                     : ok=9    changed=5    unreachable=0    failed=0   
worker1                    : ok=7    changed=5    unreachable=0    failed=0  
worker2                    : ok=7    changed=5    unreachable=0    failed=0  
```

执行后，码头工人，`kubeadm`和`kubelet`将在所有远程服务器的安装。kubectl不是必需组件，仅用于执行集群命令。在此上下文中仅在主节点上安装它是有意义的，因为您将仅从主节点运行`kubectl`命令。但请注意，`kubectl`命令可以从任何工作节点运行，也可以从可以安装和配置为指向集群的任何计算机运行。

现在安装了所有系统依赖项。让我们设置主节点并初始化集群。



## 第4步 - 设置主节点

在本节中，您将设置主节点。创建任何剧本之前，然而，它的价值涵盖了几个概念，如豆荚和波德网络插件，因为集群将包括。

pod是运行一个或多个容器的原子单元。这些容器共享资源，例如文件卷和网络接口。Pod是Kubernetes中的基本调度单元：pod中的所有容器都保证在调度pod的同一节点上运行。

每个pod都有自己的IP地址，一个节点上的pod应该能够使用pod的IP访问另一个节点上的pod。单个节点上的容器可以通过本地接口轻松进行通信。然而，pod之间的通信更复杂，并且需要单独的网络组件，该组件可以透明地将流量从一个节点上的pod传送到另一个节点上的pod。

此功能由pod网络插件提供。对于这个群集，您将使用Flannel，一个稳定且高性能的选项。

创建一个`master.yml`在本地计算机上命名的Ansible playbook ：

```
$ nano ~/kube-cluster/master.yml
```

将以下播放添加到文件中以初始化集群并安装Flannel：

> 〜/ KUBE群集/ master.yml

```
- hosts: master
  become: yes
  tasks:
    - name: initialize the cluster
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16 >> cluster_initialized.txt
      args:
        chdir: $HOME
        creates: cluster_initialized.txt

    - name: create .kube directory
      become: yes
      become_user: ubuntu
      file:
        path: $HOME/.kube
        state: directory
        mode: 0755

    - name: copy admin.conf to user's kube config
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ubuntu/.kube/config
        remote_src: yes
        owner: ubuntu

    - name: install Pod network
      become: yes
      become_user: ubuntu
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml >> pod_network_setup.txt
      args:
        chdir: $HOME
        creates: pod_network_setup.txt
```

这是这个戏剧的细分：

* 第一个任务通过运行初始化集群`kubeadm init`。传递参数`--pod-network-cidr=10.244.0.0/16`指定将从中分配pod IP的私有子网。法兰绒默认使用上述子网; 我们告诉`kubeadm`使用相同的子网。
* 第二个任务创建一个`.kube`目录`/home/ubuntu`。此目录将保存配置信息，例如连接到群集所需的管理密钥文件以及群集的API地址。
* 第三个任务将`/etc/kubernetes/admin.conf`生成的文件复制`kubeadm init`到非root用户的主目录。这将允许您用于`kubectl`访问新创建的群集。
* 最后一个任务运行`kubectl apply`安装`Flannel`。`kubectl apply -f descriptor.[yml|json]`是告诉`kubectl`创建`descriptor.[yml|json]`文件中描述的对象的语法。该`kube-flannel.yml`文件包含`Flannel`在群集中设置所需的对象的说明。

完成后保存并关闭文件。

通过运行以下命令在本地执行Playbook：

```
$ ansible-playbook -i hosts ~/kube-cluster/master.yml
```

完成后，您将看到类似于以下内容的输出：

```
Output

PLAY [master] ****

TASK [Gathering Facts] ****
ok: [master]

TASK [initialize the cluster] ****
changed: [master]

TASK [create .kube directory] ****
changed: [master]

TASK [copy admin.conf to user's kube config] *****
changed: [master]

TASK [install Pod network] *****
changed: [master]

PLAY RECAP ****
master                     : ok=5    changed=4    unreachable=0    failed=0  
```

要检查主节点的状态，请使用以下命令通过SSH连接到该节点：

```
$ ssh ubuntu@master_ip
```

进入主节点后，执行：

```
$ kubectl get nodes
```
您现在将看到以下输出：

```
Output
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.11.1
```

输出表明`master`节点已完成所有初始化任务，并且处于Ready可以开始接受工作节点并执行发送到API服务器的任务的状态。您现在可以从本地计算机添加工作程序。


## 第5步 - 设置工作节点


将工作程序添加到集群涉及在每个集群上执行单个命令。此命令包括必要的群集信息，例如主服务器API服务器的IP地址和端口以及安全令牌。只有传入安全令牌的节点才能加入群集。

导航回您的工作区并创建一个名为的剧本`workers.yml`：

```
nano ~/kube-cluster/workers.yml
```

将以下文本添加到文件中以将工作程序添加到集群：

> 〜/ KUBE群集/ workers.yml

```
- hosts: master
  become: yes
  gather_facts: false
  tasks:
    - name: get join command
      shell: kubeadm token create --print-join-command
      register: join_command_raw

    - name: set join command
      set_fact:
        join_command: "{{ join_command_raw.stdout_lines[0] }}"


- hosts: workers
  become: yes
  tasks:
    - name: join cluster
      shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
      args:
        chdir: $HOME
        creates: node_joined.txt
```

以下是剧本的作用：

* 第一个play获取需要在worker节点上运行的join命令。该命令将采用以下格式：`kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>`。一旦它获得具有适当的令牌和哈希值的实际命令，该任务就将其设置为事实，以便下一个游戏将能够访问该信息。
* 第二个游戏有一个任务，它在所有工作节点上运行join命令。完成此任务后，两个工作节点将成为群集的一部分。

完成后保存并关闭文件。

通过本地运行执行playbook：
```
$ ansible-playbook -i hosts ~/kube-cluster/workers.yml
```
完成后，您将看到类似于以下内容的输出：
```
Output
PLAY [master] ****

TASK [get join command] ****
changed: [master]

TASK [set join command] *****
ok: [master]

PLAY [workers] *****

TASK [Gathering Facts] *****
ok: [worker1]
ok: [worker2]

TASK [join cluster] *****
changed: [worker1]
changed: [worker2]

PLAY RECAP *****
master                     : ok=2    changed=1    unreachable=0    failed=0   
worker1                    : ok=2    changed=1    unreachable=0    failed=0  
worker2                    : ok=2    changed=1    unreachable=0    failed=0  
```

通过添加工作节点，您的群集现在已完全设置并正常运行，工作人员可以随时运行工作负载。在安排应用程序之前，让我们验证群集是否按预期工作。



## 第6步 - 验证群集

集群有时可能在安装过​​程中失败，因为节点已关闭或主服务器与工作服务器之间的网络连接无法正常工作。让我们验证集群并确保节点正常运行。

您需要从主节点检查群集的当前状态，以确保节点已准备就绪。如果从主节点断开连接，可以使用以下命令通过SSH重新连接到主节点：

```
$ ssh ubuntu@master_ip
```

然后执行以下命令以获取集群的状态：

```
$ kubectl get nodes
```

您将看到类似于以下内容的输出：

```
Output
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    1d        v1.11.1
worker1   Ready     <none>    1d        v1.11.1 
worker2   Ready     <none>    1d        v1.11.1
```


如果所有的节点都具有价值`Ready`的`STATUS`，这意味着它们是集群的一部分，并准备运行的工作负载。

但是，如果几个节点拥有`NotReady`的`STATUS`，它可能意味着工作节点还没有完成自己的设置呢。等待大约五到十分钟再重新运行`kubectl get nodes`并检查新输出。如果一些节点仍具有`NotReady`状态，则可能必须验证并重新运行前面步骤中的命令。

现在您的集群已成功验证，让我们在集群上安排一个示例Nginx应用程序。




## 步骤7 - 在群集上运行应用程序

您现在可以将任何容器化应用程序部署到您的群集。为了保持熟悉，让我们使用部署和服务部署Nginx ，以了解如何将此应用程序部署到集群。如果更改Docker镜像名称和任何相关标志（例如`ports`和`volumes`），您也可以将以下命令用于其他容器化应用程序。

仍在主节点内，执行以下命令以创建名为的部署`nginx`：

```
$ kubectl run nginx --image=nginx --port 80
```

部署是一种Kubernetes对象，可确保始终根据定义的模板运行指定数量的pod，即使pod在群集生命周期内崩溃也是如此。上面的部署将使用Docker注册表的Nginx Docker Image创建一个包含一个容器的pod 。

接下来，运行以下命令以创建名为`nginx`将公开公开应用程序的服务。它将通过NodePort实现，该方案将通过在群集的每个节点上打开的任意端口访问pod：

```
$ kubectl expose deploy nginx --port 80 --target-port 80 --type NodePort
```

服务是另一种类型的Kubernetes对象，它向内部和外部客户端公开集群内部服务。它们还能够对多个pod进行负载均衡请求，并且是Kubernetes中不可或缺的组件，经常与其他组件交互。

运行以下命令：

```
$ kubectl get services
```
这将输出类似于以下内容的文本：

```
Output
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP           PORT(S)             AGE
kubernetes   ClusterIP   10.96.0.1        <none>                443/TCP             1d
nginx        NodePort    10.109.228.209   <none>                80:nginx_port/TCP   40m
```
从上面输出的第三行，您可以检索运行Nginx的端口。Kubernetes将分配一个大于`30000`自动的随机端口，同时确保该端口尚未受到其他服务的约束。

要测试一切正常，请访问或通过本地计算机上的浏览器。您将看到Nginx熟悉的欢迎页面。`http://worker_1_ip:nginx_porthttp://worker_2_ip:nginx_port`

如果要删除Nginx应用程序，请先`nginx`从主节点删除该服务：

```
$ kubectl delete service nginx
```

运行以下命令以确保已删除该服务：

```
$ kubectl get services
```

您将看到以下输出：

```
Output
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP           PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>                443/TCP        1d
```
然后删除部署：

```
$ kubectl delete deployment nginx
```

运行以下命令以确认这是否有效：

```
$ kubectl get deployments
```
```
Output
No resources found.
```


## 结论

在本指南中，您已使用Kubeadm和Ansible在Ubuntu 16.04上成功建立了Kubernetes集群，以实现自动化。

如果您想知道如果要在集群设置的情况下如何处理集群，那么下一步就是将自己的应用程序和服务部署到集群上。这是一个链接列表，其中包含可以指导您完成此过程的更多信息：

* Dockerizing应用程序 - 列出了详细说明如何使用Docker对应用程序进行容器化的示例。
* Pod概述 - 详细描述了Pod如何工作以及它们与其他Kubernetes对象的关系。豆荚在Kubernetes中无处不在，因此了解它们将有助于您的工作。
* 部署概述 - 提供部署概述。了解部署之类的控制器如何工作非常有用，因为它们经常在无状态应用程序中用于扩展和自动修复不健康的应用程序。
* 服务概述 - 涵盖服务，Kubernetes集群中另一个常用对象。了解服务类型及其选项对于运行无状态和有状态应用程序至关重要。

您可以研究的其他重要概念是Volumes，Ingresses和Secrets，所有这些在部署生产应用程序时都会派上用场。

Kubernetes提供了许多功能和特性。Kubernetes官方文档是了解概念，查找特定于任务的指南以及查找各种对象的API参考的最佳位置。





