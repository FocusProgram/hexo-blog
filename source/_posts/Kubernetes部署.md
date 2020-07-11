---
title: Kubernetes部署
date: 2020-07-11 16:45:02
tags: 
 - Docker 
 - Kubernetes
categories: Devops
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/20200401114021.png
---

**Kubernetes部署**

---

# 1. Kubernetes是什么？

> [Kubernetes](https://kubernetes.io/zh/docs/home/) 是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。Kubernetes 拥有一个庞大且快速增长的生态系统。Kubernetes 的服务、支持和工具广泛可用

# 2. Kubernetes 组件

## 2.1 Master 组件

> Master 组件提供集群的控制平面。Master 组件对集群进行全局决策（例如，调度），并检测和响应集群事件（例如，当不满足部署的 replicas 字段时，启动新的 pod）。
>
> Master 组件可以在集群中的任何节点上运行。然而，为了简单起见，安装脚本通常会启动同一个计算机上所有 Master 组件，并且不会在计算机上运行用户容器。请参阅[构建高可用性集群](https://kubernetes.io/docs/admin/high-availability/)示例对于多主机 VM 的安装   

### 2.1.1 kube-apiserver

> 主节点上负责提供 Kubernetes API 服务的组件；它是 Kubernetes 控制面的前端。
>
> kube-apiserver 在设计上考虑了水平扩缩的需要。 换言之，通过部署多个实例可以实现扩缩。 参见[构造高可用集群](https://kubernetes.io/docs/admin/high-availability/)。

### 2.2.2 etcd

> etcd 是兼具一致性和高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。
> 
> 您的 Kubernetes 集群的 etcd 数据库通常需要有个备份计划。要了解 etcd 更深层次的信息，请参考[etcd 文档](https://etcd.io/docs)。

### 2.2.3 kube-scheduler

> 主节点上的组件，该组件监视那些新创建的未指定运行节点的 Pod，并选择节点让 Pod 在上面运行。
>
> 调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

### 2.2.4 kube-controller-manager

> 在主节点上运行控制器的组件。
>
> 从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。
>

这些控制器包括:

- 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应。
- 副本控制器（Replication Controller）: - 负责为系统中的每个副本控制器对象维护正确数量的 Pod。
- 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)。
- 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌.

### 2.2.5 云控制器管理器-(cloud-controller-manager)

> cloud-controller-manager 运行与基础云提供商交互的控制器。cloud-controller-manager 二进制文件是 Kubernetes 1.6 版本中引入的 alpha 功能。
>
> cloud-controller-manager 仅运行云提供商特定的控制器循环。您必须在 kube-controller-manager 中禁用这些控制器循环，您可以通过在启动 kube-controller-manager 时将 --cloud-provider 参数设置为 external 来禁用控制器循环。
>
> cloud-controller-manager 允许云供应商的代码和 Kubernetes 代码彼此独立地发展。在以前的版本中，核心的 Kubernetes 代码依赖于特定云提供商的代码来实现功能。在将来的版本中，云供应商专有的代码应由云供应商自己维护，并与运行 Kubernetes 的云控制器管理器相关联。

以下控制器具有云提供商依赖性:

- 节点控制器（Node Controller）: 用于检查云提供商以确定节点是否在云中停止响应后被删除
- 路由控制器（Route Controller）: 用于在底层云基础架构中设置路由
- 服务控制器（Service Controller）: 用于创建、更新和删除云提供商负载均衡器
- 数据卷控制器（Volume Controller）: 用于创建、附加和装载卷、并与云提供商进行交互以编排卷

## 2.2 Node组件

> 节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行环境。

### 2.2.1 kubelet

> 一个在集群中每个节点上运行的代理。它保证容器都运行在 Pod 中。
>
> kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。

### 2.2.2 kube-proxy

> kube-proxy 是集群中每个节点上运行的网络代理,实现 Kubernetes Service 概念的一部分。
>
> kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。
>
> 如果有 kube-proxy 可用，它将使用操作系统数据包过滤层。否则，kube-proxy 会转发流量本身

### 2.2.3 容器运行环境(Container Runtime)

> 容器运行环境是负责运行容器的软件。
>
> Kubernetes 支持多个容器运行环境: [Docker](http://www.docker.com/)、 [containerd](https://containerd.io/)、[cri-o](https://cri-o.io/)、  [rktlet](https://github.com/kubernetes-incubator/rktlet)以及任何实现 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)。

## 2.3 插件(Addons)

> 插件使用 Kubernetes 资源 (DaemonSet, Deployment等) 实现集群功能。因为这些提供集群级别的功能，所以插件的命名空间资源属于 kube-system 命名空间。
> 
> 所选的插件如下所述：有关可用插件的扩展列表，请参见[插件 (Addons)](https://kubernetes.io/docs/concepts/cluster-administration/addons/)。

### 2.3.1 DNS

> 尽管并非严格要求其他附加组件，但所有示例都依赖[集群 DNS](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)，因此所有 Kubernetes 集群都应具有 DNS。
>
> 除了您环境中的其他 DNS 服务器之外，集群 DNS 还是一个 DNS 服务器，它为 Kubernetes 服务提供 DNS 记录。
>
> Cluster DNS 是一个 DNS 服务器，和您部署环境中的其他 DNS 服务器一起工作，为 Kubernetes 服务提供DNS记录。
>
> Kubernetes 启动的容器自动将 DNS 服务器包含在 DNS 搜索中。

### 2.3.2 用户界面(Dashboard)

> [Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) 是 Kubernetes 集群的通用基于 Web 的 UI。它使用户可以管理集群中运行的应用程序以及集群本身并进行故障排除。

### 2.3.3 容器资源监控

> [容器资源监控](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供用于浏览这些数据的界面。

### 2.3.4 集群层面日志

> [集群层面日志](https://kubernetes.io/docs/concepts/cluster-administration/logging/) 机制负责将容器的日志数据保存到一个集中的日志存储中，该存储能够提供搜索和浏览接口。

# 3. Kubernetes安装

## 3.1 前提概要

> K8s进群搭建的方式有kubeadm（1.13版本后可以使用生产环境）、二进制（不用编译，直接可用，但需要修改配置参数）、minikube（单机版一般是开发测试使用）、yum（使用相对较少）。目前相对使用较多为kubeadm和二进制方式，kubeadm为一键方式，故障不宜解决，二进制方式需自己修改参数，会比较了解信息。

官方提供Kubernetes部署3种方式:

- [minikube](https://kubernetes.io/docs/setup/minikube/)
是一个工具，可以在本地快速运行一个单点的Kubernetes，尝试Kubernetes或日常开发的用户使用。不能用于生产环境。

- [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm/)
可帮助你快速部署一套kubernetes集群。kubeadm设计目的为新用户开始尝试kubernetes提供一种简单的方法。目前是Beta版。
官方文档：
https://kubernetes.io/docs/setup/independent/install-kubeadm/

- [二进制包](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.11.md#v1113)
从官方下载发行版的二进制包，手动部署每个组件，组成Kubernetes集群。目前企业生产环境中主要使用该方式。

## 3.2 服务器信息

| 主机IP              | 主机名            | 配置   | 操作系统                |
|-------------------|----------------|------|---------------------|
| 192\.168\.80\.128 | k8s\-master    | 2核4G | CentOS7\.x\-86\_x64 |
| 192\.168\.80\.129 | k8s\-node\-one | 2核4G | CentOS7\.x\-86\_x64 |
| 192\.168\.80\.130 | k8s\-node\-two | 2核4G | CentOS7\.x\-86\_x64 |

## 3.3 准备工作

### 3.3.1 同步时间

```
$ yum install ntpdate -y

$ ntpdate ntp.api.bz
```

### 3.3.2 修改主机名

> hostnamectl set-hostname your-new-host-name
>
> 查看修改结果：
> hostnamectl status

```
$ hostnamectl set-hostname k8s-master

$ hostnamectl set-hostname k8s-node-one

$ hostnamectl set-hostname k8s-node-two
```

### 3.3.3 设置 hostname 解析

```
$ echo "127.0.0.1 $(hostname)" >> /etc/hosts
```

三台主机分别设置 vim /etc/sysconfig/network

```
HOSTNAME=k8s-master    192.168.80.128

HOSTNAME=k8s-node-one  192.168.80.129

HOSTNAME=k8s-node-two  192.168.80.130
```

### 3.3.4 修改主机DNS

```
$ vim /etc/resolv.conf

nameserver 114.114.114.114

nameserver 8.8.8.8
```

### 3.3.5 关闭防火墙

```
$ systemctl stop firewalld && systemctl disable firewalld
```

#### 3.3.6 关闭selinux

```
$ sed -i 's/enforcing/disabled/' /etc/selinux/config 

$ setenforce 0
```

### 3.3.7 关闭swap

#### 3.3.7.1 临时关闭

```
$ swapoff -a $
```

#### 3.3.7.2 永久关闭

```
$ vim /etc/fstab $

$ swapoff -a && sysctl -w vm.swappiness=0 #关闭swap

$ sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab #取消开机挂载swap
```

### 3.3.8 设置hosts

```
$ vim /etc/hosts

192.168.80.128 k8s-master
192.168.80.129 k8s-node-one
192.168.80.130 k8s-node-two
```

### 3.3.9 设置路由（Centos7）

```
$ yum install -y bridge-utils.x86_64
```

加载br_netfilter模块，使用lsmod查看开启的模块:

```
modprobe br_netfilter
```

将桥接的IPv4流量传递到iptables的链：

```
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

### 3.4.0 重新加载所有配置文件重启

```
$ sysctl --system && reboot
```

## 3.4 开始安装

> 安装流程如下：

- 在所有节点上安装Docker和kubeadm
- 部署Kubernetes Master
- 部署容器网络插件
- 部署 Kubernetes Node，将节点加入Kubernetes集群中
- 部署Dashboard Web页面，可视化查看Kubernetes资源


### 3.4.1 所有节点安装Docker

> Kubernetes默认CRI（容器运行时）为Docker

![](https://gitee.com/FocusProgram/PicGo/raw/master/k8s-docker.png)

#### 3.4.1.1 卸载Docker

> 若之前安装过Docker，防止冲突，请卸载，若未安装，忽略此步

```
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

#### 3.4.1.2 安装依赖

> 这个过程可能会升级内核，如果升级了建议重启下切换到新内核

```
sudo yum update -y && yum install -y yum-utils device-mapper-persistent-data lvm2 && reboot
```

#### 3.4.1.3 添加镜像库(二者选其一)

##### 3.4.1.3.1 官方yum库

```
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

##### 3.4.1.3.2 阿里云镜像库

```
$ yum install wget && wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

#### 3.4.1.4 安装Docker

##### 3.4.1.4.1 安装最新版本

```
$ yum list docker-ce #查看最新版本

$ yum -y install docker-ce docker-ce-cli containerd.io #安装最新版本
```

##### 3.4.1.4.2 安装指定版本

```
$ list docker-ce --showduplicates | sort -r #查看历史版本

$ yum -y install docker-ce-指定版本号      #指定版本号安装
```

#### 3.4.1.5 脚本一键安装

```
$ curl -fsSL "https://get.docker.com/" | sh && systemctl enable --now docker
```

> 修改docker cgroup驱动，与k8s一致，使用systemd方式
>
> 修改docker cgroup驱动：native.cgroupdriver=systemd

```
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

$ systemctl restart docker  # 重启使配置生效
```

> 备注：systemd是系统自带的cgroup管理器, 系统初始化就存在的, 和cgroups联系紧密,为每一个进程分配cgroups,  用它管理就行了. 如果设置成cgroupfs就存在2个cgroup控制管理器, 实验证明在资源有压力的情况下,会存在不稳定的情况.cgroupfs是docker自带的。

#### 3.4.1.6 配置docker镜像加速地址

```
vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://l5arlel6.mirror.aliyuncs.com"]
}
```

#### 3.4.1.7 启动Docker并设置开机启动

```
$ systemctl enable docker && systemctl start docker

$ docker --version #查看Docker版本
```

### 3.4.2 添加阿里云YUM软件源

> master、node节点都需要安装kubelet kubeadm kubectl。
安装kubernetes的时候，需要安装kubelet, kubeadm等包，但k8s官网给的yum源是packages.cloud.google.com，国内访问不了，此时我们可以使用阿里云的yum仓库镜像。

```
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 3.4.3 安装kubeadm、kubelet、kubectl

> 由于版本更新频繁，可以指定版本号部署：
>
> kubeadm： 引导集群的命令
>
> kubelet：集群中运行任务的代理程序
>
> kubectl：命令行管理工具

```
$ yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes 安装最新版

$ yum install -y kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0  安装制定版本

$ systemctl enable kubelet && systemctl start kubelet

$ systemctl enable kubelet.service
```

启动k8s，查看安装k8s的安装目录:

```
$ rpm -ql kubelet
/etc/kubernetes/manifests 	清单目录
/etc/sysconfig/kubelet   配置文件
/usr/bin/kubelet
/usr/lib/systemd/system/kubelet.service
```

### 3.4.4 部署Kubernetes Master

#### 3.4.4.1 可以科学上网访问国外

```
$ kubeadm config images pull
```

#### 3.4.4.2 不能连接到外网解决办法

```
$ kubeadm config images list # 列出所需镜像

k8s.gcr.io/kube-apiserver:v1.18.0
k8s.gcr.io/kube-controller-manager:v1.18.0
k8s.gcr.io/kube-scheduler:v1.18.0
k8s.gcr.io/kube-proxy:v1.18.0
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401114021.png)

##### 3.4.4.2.1 第一种安装方式

编写安装脚本vim install.sh

```
K8S_VERSION=v1.18.0
ETCD_VERSION=3.4.3-0
DASHBOARD_VERSION=v1.8.3
FLANNEL_VERSION=v0.12.0-amd64
DNS_VERSION=1.6.7
PAUSE_VERSION=3.2
# 基本组件
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION
# 网络组件
docker pull quay.io/coreos/flannel:$FLANNEL_VERSION
# 修改tag
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$K8S_VERSION k8s.gcr.io/kube-apiserver:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$K8S_VERSION k8s.gcr.io/kube-controller-manager:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$K8S_VERSION k8s.gcr.io/kube-scheduler:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$K8S_VERSION k8s.gcr.io/kube-proxy:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd-amd64:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$DNS_VERSION k8s.gcr.io/coredns:$DNS_VERSION
```

##### 3.4.4.2.2 第二种安装方式

```
#下载镜像
$ kubeadm config images list | sed -e 's/^/docker pull /g' -e 's#k8s.gcr.io#registry.cn-hangzhou.aliyuncs.com/google_containers#g' | sh -x 

#修改镜像名称
$ docker images | grep registry.cn-hangzhou.aliyuncs.com/google_containers | awk '{print "docker tag",$1":"$2,$1":"$2}' | sed -e 's/registry.cn-hangzhou.aliyuncs.com\/google_containers/k8s.gcr.io/2' | sh -x 

#删除原始镜像
$ docker images | grep registry.cn-hangzhou.aliyuncs.com/google_containers | awk '{print "docker rmi """$1""":"""$2}' | sh -x 
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401131713.png)

### 3.4.5 部署Kubernetes Node

编写安装脚本vim install.sh

```
K8S_VERSION=v1.18.0

PAUSE_VERSION=3.2

FLANNEL_VERSION=v0.12.0-amd64

# 根据所需镜像名字先拉取国内资源
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull quay.io/coreos/flannel:$FLANNEL_VERSION

# 修改镜像tag
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION k8s.gcr.io/kube-proxy:$K8S_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION

# 删除原来的镜像
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:$K8S_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401122908.png)

### 3.4.6 初始化Master

```
kubeadm init --pod-network-cidr 10.244.0.0/16 --apiserver-advertise-address=192.168.80.128
```

无法拉取镜像的可使用此命令192.168.20.61（Master）执行 使用阿里云节点在线方式，节点会自动安装阿里云镜像

```
由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。kubeadm init --pod-network-cidr 10.244.0.0/16 --image-repository= registry.cn-hangzhou.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.80.128
```

指定版本实例:

```
$ kubeadm init \
--apiserver-advertise-address=192.168.80.128 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.18.0 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16
```

执行完毕保存kubeadm token，以便node节点加入
#每个机器创建的master以下部分都不同,需要自己保存好

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401132225.png)

```
$ kubeadm join 192.168.80.128:6443 --token 9c963m.as0v97vbtdajyszd \
    --discovery-token-ca-cert-hash sha256:9e213498e36fe6044c5022a39911989c417b649a667825239a536033c68e18cb
```

### 3.4.7 查看kubeadm init的帮助信息

```
# kubeadm init –help
--apiserver-advertise-address：API服务器将通知它正在监听的IP地址，监听的地址为“0.0.0.0”，即本机所有IP地址。
--apiserver-bind-port：API服务器绑定到的端口。(默认：6443)
--cert-dir：加载证书的相关目录（默认：/etc/kubernetes/pki）
--config：配置文件的路径。警告:配置文件目前属于实验性，还不稳定。
--ignore-preflight-errors：将错误显示为警告的检查列表进行忽略。例如:“IsPrivilegedUser,Swp”。Value 'all'忽略所有检查中的错误。
--pod-network-cidr：指定pod网络的IP地址范围。如果设置，控制平面将为每个节点自动分配CIDRs。
--service-cidr：为service VIPs使用不同的IP地址。(默认“10.96.0.0/12”)。
```

> 运行初始化，程序会检验环境一致性，可以根据实际错误提示进一步修复问题。程序会访问https://dl.k8s.io/release/stable-1.txt获取最新的k8s版本，访问这个连接需要FQ，如果无法访问，则会使用kubeadm client的版本作为安装的版本号，使用kubeadm version查看client版本。也可以使用--kubernetes-version明确指定版本

### 3.4.8 使用kubectl工具

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

### 3.4.9 安装Pod网络插件（CNI）

```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 3.5.0 加入Kubernetes Node

> 在192.168.80.129/130（Node）执行确保之前节点已拉取所需镜像。
向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
$ kubeadm join 192.168.80.128:6443 --token 9c963m.as0v97vbtdajyszd \
    --discovery-token-ca-cert-hash sha256:9e213498e36fe6044c5022a39911989c417b649a667825239a536033c68e18cb
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401133645.png)

查看已经运行了的容器

```
$ kubectl get pods -n kube-system  查看运行空间为名称为kube-system

$ kubectl get pods --all-namespaces  查看所有运行空间

$ kubectl get ns 查看运行的名称空间

$ kubectl get pods -n kube-system -o wide 查看容器详细信息

$ kubectl get svc -n kube-system 查看服务信息
```
查看所有节点是否正常运行，显示如下，为全部正常运行：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401151922.png)

如果存在penging等状态，请使用如下命令查看问题：

```
kubectl describe pod pod节点名 -n 命名空间
```

## 3.5 测试Kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx

$ kubectl expose deployment nginx --port=80 --type=NodePort

$ kubectl get pod,svc
```

使用master节点部署，显示如下错误： 1 node(s) had taints that the pod didn't tolerate

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401134920.png)

> kubernetes增加污点，达到pod是否能在做节点运行
>
> master node参与工作负载 (只在主节点执行)
使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。
>
> 这里搭建的是测试环境可以使用下面的命令使Master Node参与工作负载：
k8s是master节点的hostname


允许master节点部署pod，使用命令如下:

```
kubectl taint nodes --all node-role.kubernetes.io/master-1

输出如下:

node “k8s” untainted
error: taint “node-role.kubernetes.io/master:” not found 错误忽略。
```

禁止master部署pod
```
kubectl taint nodes k8s node-role.kubernetes.io/master=true:NoSchedule
```

通过命令查看服务端口 kubectl get pod,svc

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401151107.png)

查看服务部署节点 kubectl get pods -A -o wide

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401151922.png)

分别访问192.168.80.128/129/130节点上运行的Nginx

[http://192.168.80.128:30482/](http://192.168.80.128:30482/)

[http://192.168.80.129:30482/](http://192.168.80.129:30482/)

[http://192.168.80.130:30482/](http://192.168.80.130:30482/)

显示均为下，则运行正常：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401152349.png)

## 3.6 部署 Dashboard

### 3.6.1 k8s为v1.16之前安装步骤(v1.10)

#### 3.6.1.1 安装kubernetes-dashboard

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

#### 3.6.1.2 查看dashboard部署服务器

```
$ kubectl get pods -A -o wide
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/97db9b8f-6c72-4fa6-9ff5-1297de4d4ca7.jpg)

从图中可以看出 pod在node1节点 其中节点IP为192.168.20.62，但是状态为ErrImagePull

可使用命令查看具体原因
```
$ kubectl describe pod kubernetes-dashboard -n kube-system
```

原因：因为默认镜像国内无法访问

解决办法：使用阿里云镜像下载，在远程k8s-node-one服务器，输入下面命令拉取

```
# 拉取镜像
$ docker pull registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.1     

# 修改镜像名
$ docker tag registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

# 删除原镜像
$ docker rmi registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.1  
```

查看状态
```
$ kubectl get pods -A -o wide
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/5389ea55-a5cd-47b9-80f0-415ce404b2bd.jpg)

查看服务详情
```
$ kubectl get svc -n kube-system
```
默认Dashboard只能集群内部访问

![](https://gitee.com/FocusProgram/PicGo/raw/master/75bcb537-4d4d-4e83-8932-8d6c9d464695.jpg)

修改Service为NodePort类型，暴露到外部：
修改node为NodePort模式

```
$ kubectl patch svc -n kube-system kubernetes-dashboard -p '{"spec":{"type":"NodePort"}}'
```

或者修改service配置，找到type，将ClusterIP改成NodePort：

```
$ kubectl edit service kubernetes-dashboard --namespace=kube-system
```
查看暴露端口：

```
$ kubectl get service --namespace=kube-system
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/95ba8de3-f3a8-489b-a352-17e7bf6e6089.jpg)

访问地址为[https://192.168.20.62:32146](https://192.168.20.62:32146)

![](https://gitee.com/FocusProgram/PicGo/raw/master/f09af693-63f4-4e60-a617-a9754bc55657.jpg)

### 3.6.2 k8s为v1.16+安装步骤(v2.0)

#### 3.6.2.1 下载recommended.yaml

```
$ mkdir /data && cd /data

$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
```

#### 3.6.2.2 编辑配置文件

> 解决认证问题，端口问题，镜像问题

```
$ vim recommended.yaml

## 设置对外开放端口
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort # 设置为NodePort
  ports:
    - port: 443
      nodePort: 30001 # 不设置随机端口，设置为固定端口30001
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---    

## 由于证书问题，只能firefox浏览器才能打开，通过修改证书的方式，使得所有浏览器都能打开：注释此段
#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs # 生成证书会用到该名字
#  namespace: kubernetes-dashboard  # 生成证书会用到该名字
#type: Opaque

---

## 修改获取镜像相关
spec:
  nodeSelector:
    type: master # 选择部署到哪一台节点
  containers:
    - name: kubernetes-dashboard
      image: kubernetesui/dashboard:v2.0.0-beta4
      imagePullPolicy: Always # 注释原来的镜像拉取规则
      imagePullPolicy: IfNotPresent   # 设置为本地没有则拉取
   ports:
    - containerPort: 8443
      protocol: TCP
  args:
    - --token-ttl=43200 # 修改token的实效时间60*60*12 为12小时,默认15分钟
    - --auto-generate-certificates
    - --namespace=kubernetes-dashboard

```

#### 3.6.2.3 配置证书

> 由于证书问题，只能firefox浏览器能打开，通过修改证书的方式，使得所有浏览器都能打开

```
# 进入默认证书目录进行配置
$ cd /etc/kubernetes/pki/

# 查看是否存在namespace为kubernetes-dashboard
$ kubectl get namespaces

# 不存在namespace为创建kubernetes-dashboard创建namespace
$ kubectl create namespace kubernetes-dashboard

# 创建一个证书
$ (umask 077; openssl genrsa -out dashboard.key 2048)

# 建立证书的签署请求(二者选其一即可)

方式一：
$ openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=HTI/CN=kubernetes-dashboard"

方式二：master节点IP
$ openssl req -days 3650 -new -out dashboard.csr -key dashboard.key -subj '/CN=**192.168.80.128**'

# 使用集群的ca来签署证书
$ openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 3650

# 我们需要把我们创建的证书创建为secret给k8s使用
$ kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.crt=./dashboard.crt --from-file=dashboard.key=./dashboard.key  -n kubernetes-dashboard

# 在master节点上这是label，拉取镜像
# 设置node选择器label为master
$ kubectl label node k8s-master type=master

# 拉取镜像
$ docker pull kubernetesui/dashboard:v2.0.0-beta4

# 进入配置文件所在目录,应用配置文件
$ kubectl apply -f recommended.yaml

# 获取默认用户登录token
$ kubectl describe secrets  $(kubectl  get secrets -n kubernetes-dashboard | awk  '/kubernetes-dashboard-token/{print $1}' ) -n kubernetes-dashboard |sed -n '/token:.*/p'

eyJhbGciOiJSUzI1NiIsImtpZCI6ImdEa1F3eU0yckVWTDhWSkFac1Uyc0RkcUF3VmlTTlA0ZEp6UzdlNmRpNm8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1ocnhwdiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjhkMmI2MWNiLTY1OTEtNGRkYy1iYjFmLWVjNWZjODg0ZDg0ZCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.i6oCoHDpKMt9b-yIXVULYf-QPdAnHQignKNvJTxCR7Bzw_GGKx3iSnBF366C0NaWVm5nWRXJSd61qp89Z-02OrVWAJbZw0FEiQ9cudM2ynE6J_sid1hat24q-M7thwr0rz_2E7uxmB1NBN9X1mw9KIMYwMeRvhdhLgMdD_t_o40pX05IMYhGKKEFf3GdEBrBQ4sIoY2FTbxgN2tWolAaU3gUaJtMw1iuMz7ePDEsw_exv5An-DNnfYeKcKhqN4FGmHCR-YNzYtsWXBEN3GxYmzX16U4eLC1ir9W32nVDaT5bisPaZ53Z_OJnSMJsEvXoz27_9_6lEAzuv022j0TFVQ
```

查看状态

```
$ kubectl get pods -A -o wide
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401212246.png)

查看服务详情

```
$ kubectl get svc -n kubernetes-dashboard
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401212316.png)

查看pod 与service是否运行正常

```
$ kubectl get pod,svc -n kubernetes-dashboard -o wide
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401212338.png)

可根据需求查看kubernetes-dashboard-75d8b49cf6-7q8nk信息

```
$ kubectl describe pod kubernetes-dashboard-649b495f5d-c2f58 -n kubernetes-dashboard
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401212420.png)

### 3.6.3 访问Dashboard

使用默认token登录 [https://192.168.80.128:30001](https://192.168.80.128:30001/#/login)

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImdEa1F3eU0yckVWTDhWSkFac1Uyc0RkcUF3VmlTTlA0ZEp6UzdlNmRpNm8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1ocnhwdiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjhkMmI2MWNiLTY1OTEtNGRkYy1iYjFmLWVjNWZjODg0ZDg0ZCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.i6oCoHDpKMt9b-yIXVULYf-QPdAnHQignKNvJTxCR7Bzw_GGKx3iSnBF366C0NaWVm5nWRXJSd61qp89Z-02OrVWAJbZw0FEiQ9cudM2ynE6J_sid1hat24q-M7thwr0rz_2E7uxmB1NBN9X1mw9KIMYwMeRvhdhLgMdD_t_o40pX05IMYhGKKEFf3GdEBrBQ4sIoY2FTbxgN2tWolAaU3gUaJtMw1iuMz7ePDEsw_exv5An-DNnfYeKcKhqN4FGmHCR-YNzYtsWXBEN3GxYmzX16U4eLC1ir9W32nVDaT5bisPaZ53Z_OJnSMJsEvXoz27_9_6lEAzuv022j0TFVQ
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401223006.png)

默认用户权限不足---授权管理员，创建service account并绑定默认cluster-admin管理员集群角色：

```
# 创建service account
$ kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard

# 绑定集群管理员
$ kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

# 获取token
$ kubectl describe secrets  $(kubectl  get secrets -n kubernetes-dashboard | awk  '/dashboard-admin-token/{print $1}' ) -n kubernetes-dashboard |sed -n '/token:.*/p'

eyJhbGciOiJSUzI1NiIsImtpZCI6ImdEa1F3eU0yckVWTDhWSkFac1Uyc0RkcUF3VmlTTlA0ZEp6UzdlNmRpNm8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tZGZtZmQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNWQ0ZTQwYjUtN2QxNi00NjFhLWE1N2YtYTZiZGJiZjYzZjc5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.JvfId92C8sOJ60mgKT39RDuvmA9ANZJuG8VwtJoC8qRRVD-CDL_sTsDR_GXqpAZDvs3q_HtUhWMMkF1awqPnQQNMdQ36K-tLI7ZAquPhjX71mEQq-Smto9XVOJziHlkXPizgBnf5kemaG-SYOOIbesFBk2Ky9264K2APoaafrDCtgwrCHQz9HrCDrbGb6ffjd_DeoMYRSu9pYr4baQsl57DEwLL5CLZ56W37IaFyiGA90s8ZDc_RzKx5nRftOXv4j0xiR-eGRjhuzUb35SMQsVIvejIVcHUwCGKNAE74PRDmSbauhcab0SMgR5DW6Lk5lsGnIPAXKs3BUp9iURoi1A
```

记录，以备不时之需或者获取token信息知晓管理员token

```
$ kubectl get secret -n kubernetes-dashboard |grep dashboard
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401224525.png)

获取token

```
$ kubectl describe secrets kubernetes-dashboard-token-9tkf9 -n kubernetes-dashboard
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401224620.png)

使用输出的token登录Dashboard [https://192.168.80.128:30001/](https://192.168.80.129:30001/)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200401224832.png)


至此，Kubernetes搭建完毕

## 3.7 Dashboard中文版本安装(扩展)

### 3.7.1 卸载Dashboard:v2.0版本

#### 3.7.1.1 卸载Dashboard

```
$ kubectl delete -f kubernetes-dashboard.yaml
```

#### 3.7.1.2 删除证书

```
$ kubectl delete secret generic kubernetes-dashboard-certs -n kube-system
```

### 3.7.2 安装Dashboard:v1.8.3版本

#### 3.7.2.1 重新生成证书

```
## 生成证书请求的key
$ openssl genrsa -out dashboard.key 2048

## 生成证书请求
$ openssl req -days 3650 -new -out dashboard.csr -key dashboard.key -subj '/CN=**192.168.80.128**'

## 生成自签证书
$ openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt

## 创建证书

kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kube-system
```

#### 3.7.2.2 下载配置文件

```
$ docker pull mirrorgooglecontainers/kubernetes-dashboard-amd64:v1.8.3

$ wget http://down.i4t.com/k8s-passwd-dashboard.yaml| kubectl apply -f k8s-passwd-dashboard.yaml
```

#### 3.7.2.3 编辑配置文件

```
$ vim k8s-passwd-dashboard.yaml

## 注释证书鉴证，免得只能用火狐浏览器访问

#apiVersion: v1
#kind: Secret
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard-certs
#  namespace: kube-system
#type: Opaque
```

#### 3.7.2.4 basic认证

方式一：修改kube-api的启动参数

```
## 增加配置文件:
$ vim /etc/kubernetes/pki/basic_auth_file.csv

admin,admin,1
system,system,2

$ vim /etc/kubernetes/manifests/kube-apiserver.yaml

## 增加如下参数：
- --basic-auth-file=/etc/kubernetes/pki/basic_auth_file.csv

## 对admin用户授权
$ kubectl create clusterrolebinding cluster-test2 --clusterrole=cluster-admin --user=admin

$ kubectl -s="https://192.168.80.128:6443" --username="admin" --password="admin" get pods -n kube-system
```

方式二：给匿名用户授权

```
$ kubectl create clusterrolebinding test:anonymous --clusterrole=cluster-admin --user=system:anonymous
```

#### 3.7.2.5 发布

```
$ kubectl apply -f k8s-passwd-dashboard.yaml
```

#### 3.7.2.6 访问

访问 [https://192.168.80.128:30000/](https://192.168.80.128:30000/)

使用 admin/admin 登陆

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200403155621.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200403155637.png)

## 3.8 安装常见问题汇总

### 3.8.1 克隆github代码速度慢

解决办法：
```
https://github.com/x 换成 git://github.com/x
```

### 3.8.2 boot空间不足

解决办法：
```
内核信息
1.查看现运行的内核版本
$ uname -r

3.10.0-1062.18.1.el7.x86_64
 
2.列出所有的内核文件
$ rpm -q kernel

kernel-3.10.0-862.el7.x86_64
kernel-3.10.0-1062.18.1.el7.x86_64
 
3.删除所有旧的内核文件
（注意不要删除当前系统正在运行的内核文件）
$ rpm -ekernel-3.10.0-862.el7.x86_64
到此，旧的内核文件就安全删除

$ rpm -q kernel
kernel-3.10.0-1062.18.1.el7.x86_64
```

### 3.8.3 pod 一直未处于Running状态

解决办法：
```
# 使用 describe 命令查看具体原因：

$ kubectl describe pod kubernetes-dashboard -n kube-system
```

### 3.8.4 彻底删除pod

```
## 删除pod

$ kubectl get pod -n 命名空间

$ kubectl delete pod 节点名 -n 命名空间

## 删除deployment

$ kubectl get deployment -n 命名空间

$ kubectl delete deployment 部署名 -n 命名空间
```

### 3.8.5 开机运行kubectl失败

```
$ systemctl status kubelet  命令查看kubelet的情况
```

# 4. Metrics-server安装

> 在k8s早期版本中，对资源的监控使用的是heapster的资源监控工具。
>
> 但是从 Kubernetes 1.8 开始，Kubernetes 通过 Metrics API 获取资源使用指标，例如容器 CPU 和内存使用情况。
>
> 这些度量指标可以由用户直接访问，例如通过使用kubectl top 命令，或者使用集群中的控制器。
>
> Metrics API: 通过 Metrics API，您可以获得 node 或 pod 当前的资源使用情况（但是不存储）。
>
> metres-server比 heapster 优势在于： 访问不需要 apiserver 的代理机制，提供认证和授权等; 很多集群内组件依赖它（HPA,scheduler,kubectl top），因此它应该在集群中默认运行

## 4.1 Kubernetes安装版本信息

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402112923.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402113046.png)

## 4.2 部署metrics-server

### 4.2.1 下载metrics-server
```
$ cd /data

$ git clone https://github.com/kubernetes-incubator/metrics-server
```

### 4.2.2 查看依赖镜像

```
$ cd /data/metrics-server/deploy/kubernetes/

$ grep 'image:' *

metrics-server-deployment.yaml: image: k8s.gcr.io/metrics-server-amd64:v0.3.6

# 无法访问外网，修改为阿里云镜像
$ sed -i "s/image: .*/image: registry.cn-hangzhou.aliyuncs.com\/centosos\/metrics-server-amd64:v0.3.6/g" metrics-server-deployment.yaml
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402114356.png)

### 4.2.3 安装metrics-server

```
$ kubectl create -f /data/metrics-server/deploy/kubernetes/
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402114521.png)

查看pod是否运行
```
$ kubectl -n kube-system get pods -l k8s-app=metrics-server
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402114549.png)

查看pod的具体信息

```
$ kubectl get pods -n kube-system -o wide
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402114617.png)

验证是否安全成功查看 apiserver

```
$ kubectl get apiservices
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402114655.png)

查看服务是否正常

```
$ kubectl top node 出现error: metrics not available yet

## 查看pod信息诊断
$ kubectl describe pods metrics-server-bfcf447f7-vk9bm -n kube-system
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402115616.png)

```
## 查看日志
$ kubectl logs metrics-server-bfcf447f7-vk9bm -n kube-system 
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402115824.png)

### 4.2.4 问题总结

- 问题1：metrics-server默认使用节点hostname通过kubelet 10250端口获取数据，但是coredns里面没有该数据无法解析(10.96.0.10:53)，可以在metrics server启动命令添加参数 --kubelet-preferred-address-types=InternalIP 直接使用节点IP地址获取数据

- 问题2：kubelet 的10250端口使用的是https协议，连接需要验证tls证书。可以在metrics server启动命令添加参数--kubelet-insecure-tls不验证客户端证书

- 问题3：yaml文件中的image地址k8s.gcr.io/metrics-server-amd64:v0.3.6 需要梯子，需要改成中国可以访问的image地址，可以使用aliyun的。这里使用hub.docker.com里的google镜像地址 image: mirrorgooglecontainers/metrics-server-amd64:v0.3.6

### 4.2.5 问题修正

编辑 vim /data/metrics-server/deploy/kubernetes/metrics-server-deployment.yaml

```
 - name: metrics-server
        #image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        image: registry.cn-hangzhou.aliyuncs.com/centosos/metrics-server-amd64:v0.3.6
        #imagePullPolicy: IfNotPresent
        imagePullPolicy: Always
        command:
            - /metrics-server
            # - --kubelet-preferred-address-types=InternalIP
            - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
            - --kubelet-insecure-tls
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402125032.png)

然后重新部署metrics-server

```
$ kubectl apply -f /data/metrics-server/deploy/kubernetes/metrics-server-deployment.yaml
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402125008.png)

查看部署状态并验证

```
$ kubectl get pods -n kube-system

$ kubectl top node
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402132122.png)

<font color="red">

出现该报错信息：the server is currently unable to handle the request (get nodes.metrics.k8s.io)

</font>

解决方法：

```
## 编辑配置文件
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml

## 添加配置

- --enable-aggregator-routing
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402171535.png)

再次验证：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402163020.png)

## 4.3 访问Dashboard查看metrics-server

显示如下,部署成功：[https://192.168.80.128:30001/](https://192.168.80.128:30001/)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200402163034.png)