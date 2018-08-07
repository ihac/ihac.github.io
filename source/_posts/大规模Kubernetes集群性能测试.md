---
title: 使用kubemark进行大规模Kubernetes集群性能测试
date: 2018-08-05 22:57:19
tags: [Kubernetes, Kubemark, Benchmark]
---

本文主要介绍如何使用kubemark来模拟大规模Kubernetes集群（1000 node），并对其进行e2e性能测试，最后附上测试报告。

**NOTE**
本次测试日期为*2018-08-02*，最新commit为*[Merge pull request #66671 from hanxiaoshuai/cleanup07261](https://github.com/kubernetes/kubernetes/commit/0e9b1dd20f8c202d5118b8712c4a9dcfe67dbf4a)*，release版本为*[v1.12.0-alpha.1](https://github.com/kubernetes/kubernetes/releases/tag/v1.12.0-alpha.1)*.
kubemark目前尚属`开发中`、`不成熟`的状态，在之后的版本中可能会有大量代码改动，因此本文内容并不一定适用于更新或更老的Kubernetes版本，仅作为参考，切忌照搬。

## 名词解释

以下是本文将用到的相关名词：

- kubemark集群：表示模拟的Kubernetes集群；
- kubemark-master：表示kubemark集群的master节点，以vm形式存在；
- kubemark-hollow-node：表示kubemark集群的hollow node，即模拟的slave节点，以pod形式存在；
- 物理集群：表示以pod形式运行hollow node的底层集群；
- 工作机：表示用于编译Kubernetes、运行kubemark脚本的机器，可以是个人主机。



## 背景

kubemark的设计初衷就是为了让用户能够非常方便地模拟任意规模的Kubernetes集群，并对集群进行性能测试，根据我在网上搜集到的资料，kubemark模拟出来的集群与真实集群在性能方面是非常接近的，因此kubemark应当是一个可行且易用的benchmark工具。

那为什么我想要将这次测试记录成博文呢？事实上，目前kubemark的易用性仅在GCE平台上有效，查看`start-kubemark.sh`脚本可以发现，GCE是kubemark目前唯一支持的cloud provider，虽然还留有`pre-existing`这么一个可选项，理论上来说可以支持任意Kubernetes平台，但官方文档对这一feature言之甚少，仅有一句：

> The pre-existing provider is an advanced use case that requires the developer to have knowledge of setting up a kubemark master.

我的实践经历也证明，在其它云平台搭建kubemark集群并进行性能测试确实不是一件容易的差事。因此，我写这篇总结博文主要出于两点考虑：一方面，是对自己连续3天高压工作、解决多个bug的感慨，以及对最终成果的珍惜；另一方面，网上针对kubemark的指导文档几乎不存在，官方文档虽然很赞（详细介绍了kubemark的工作流程），但没能介绍如何在GCE以外的云平台上（或标准Kubernetes集群）搭建kubemark集群，希望我的经验能够帮到后来人吧。


## 大致流程（原理）

首先创建一台虚拟机，运行Kubernetes相关组件（比如apiserver、scheduler等等），作为kubemark集群的master节点；然后在物理集群上运行大量pod，向kubemark-master注册，成为kubemark集群中的slave节点；最后在搭建完成的kubemark集群上运行e2e测试。

## GCE + kubemark

### 步骤

在GCE平台搭建kubemark集群比较简单，大致步骤如下：

1. 创建一个Kubernetes集群（即物理集群）与一个VM实例（即工作机）；
2. 执行命令`gcloud auth login`完成gcloud认证；
3. 在工作机上配置kubeconfig（具体命令参见集群connect说明），使其可以直接通过kubectl访问物理集群；
4. 在工作机上克隆Kubernetes仓库，在repo根目录下，执行命令`make quick-release`进行编译打包；
5. 在repo根目录下，执行`test/kubemark/start-kubemark.sh`脚本，自动创建kubemark集群；
6. kubemark集群创建成功后，执行`test/kubemark/run-e2e-tests.sh`脚本，运行e2e性能测试。

### FAQ

1. 默认创建的kubemark集群只有10个hollow node，如何指定hollow node的个数，比如1000个？
    **answer**:
    参见代码：
    ``` bash
    NUM_NODES=${KUBEMARK_NUM_NODES:-10} # cluster/kubemark/gce/config-default.sh
    ```
    可知，将环境变量`KUBEMARK_NUM_NODES`设置为1000即可创建1000-node规模的kubemark集群。

2. 创建kubemark集群失败，报错信息如下，大致意思是找不到相应的subnetwork。
``` bash
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
 - Invalid value for field 'resource.networkInterfaces[0].subnetwork': 'https://www.googleapis.com/compute/v1/projects/xxx/regions/us-central1/subnetworks/e2e'. The refer
enced subnetwork resource cannot be found.
Attempt 1 failed to gcloud compute instances. Retrying.
```
    **answer**:
    查看GCE上的安全组与subnetwork信息，可以发现其默认名字是`default`，而非`e2e`。参见代码：
    ``` bash
    NETWORK=${KUBE_GCE_NETWORK:-e2e} # cluster/kubemark/gce/config-default.sh
    ```
    可知，将环境变量`KUBE_GCE_NETWORK`设置为`default`即可。

3. 具体使用的命令？
    **answer**:
``` bash
KUBE_GCE_NETWORK=default KUBEMARK_NUM_NODES=20 test/kubemark/start-kubemark.sh
test/kubemark/run-e2e-tests.sh --ginkgo.focus=\[Feature:Performance\] --gather-metrics-at-teardown=true --output-print-type=json --report-dir=/xxx/yyy
```

4. 如何彻底删除kubemark集群，重新安装？
    **answer**:
    首先删除kubemark-master实例，然后删除额外绑定的磁盘、静态ip地址、防火墙规则，最后在物理集群中删除kubemark这一namespace。

5. 如何立即中断e2e测试，重新运行？
    **answer**:
    连按两次Ctrl+C中断测试，然后在kubemark集群中删除所有与e2e测试相关的namespace。

6. 为什么不使用GCE测试1000节点规模的集群？
    **answer**:
    因为GCE免费账户限制了资源使用量，参见[官方说明](https://cloud.google.com/free/docs/frequently-asked-questions#limitations)，几十个node估计就是上限了。


## Aliyun + kubemark

### 一些说明

我接到的任务是，在阿里云的平台上进行大规模Kubernetes集群性能测试，自然就不能采用GCE + kubemark这种简易模式了。本来想着，kubemark集群自身的运行是与底层云平台无关的，只有kubemark集群的搭建过程涉及到云平台的接口交互，因此我需要做的应该就是简单的接口适配了。但万万没想到，实际操作过程中会有如此多的坑（当然很重要的一部分原因是我没仔细读脚本源码），解决完一个（或绕开一个）又出现新的，现在估算一下，我在非常熟悉GCE + kubemark的工作流程的基础上，完成Aliyun + kubemark的工作的有效用时大概为2天1夜（不包括测试用时）。工作量确实不算大，困难之处大概在于需要承受各种坑带来的心理折磨吧。

需要说明的是，这部分内容没办法如GCE + kubemark那样列出完整细致的步骤与具体参数，因为我自己在搭建kubemark集群的时候走了很多弯路，基本一直处于“遇到bug - 解决bug”的循环中，没法总结出一套“一蹴而就”的Aliyun + kubemark攻略，我能做到的，只有介绍大致的步骤流程、可能遇到的问题以及解决这些问题的思路方法。

### 基本思路与流程

根据文档描述，kubemark代码在之后会有一定的重构，方便开发者适配不同的cloud provider，因此我一开始就打消了给脚本添加Aliyun支持的念头，因为不会被merge，且过一段时间可能就失效了。所以，我决定采用`pre-existing`的模式来搭建kubemark集群。

在`pre-existing`模式下，用户需要自己提供一个可用的kubemark-master节点，`start-kubemark.sh`脚本将跳过申请vm资源、创建master的这一步骤，直接访问用户指定的kubemark-master，然后注册hollow node。

### 整体建议

在Aliyun平台（或任意标准Kubernetes集群）上部署kubemark的几点建议：

1. 仔细阅读kubemark官方[文档](https://github.com/kubernetes/community/blob/master/contributors/devel/kubemark-guide.md)以及pre-existing相关[文档](https://github.com/kubernetes/kubernetes/blob/master/test/kubemark/pre-existing/README.md)；
2. 预先在GCE上搭建kubemark，熟悉其架构，**释放资源时保留kubemark-master与工作机**；
3. 仔细阅读脚本源码，包括但不限于`start-kubemark.sh`、`start-kubemark-master.sh`，结合文档，熟悉kubemark集群的搭建步骤（比如证书、manifest文件的生成）；
4. 不要使用kubeadm或其它方式搭建kubemark-master；
5. 自行安装kubelet，但其它kubernetes组件均采用kubemark提供的manifest文件；
6. 使用GCE上保留的工作机单独生成证书与token，即仅执行`generate-pki-config`函数，注意需要将master ip地址替换为新kubemark-master的地址，否则证书无法在新机器上使用；
7. 注意`known_token.csv`文件中随机生成的token，确保kubernetes各组件、hollow node各组件的manifest与其保持一致，否则apiserver会报认证失败的错误；
8. 如果遇到容器（比如apiserver）一启动就异常退出的问题，一个好的debug方法是，将manifest中的command字段替换为一段循环语句，比如`while true; do echo 123; sleep 1; done`，在容器正常运行以后，通过`docker exec`获取终端，然后手动运行原command，观察命令报错原因；
9. 保留GCE上的kubemark-master的好处在于，我们在搭建新的kubemark-master时，可以经常和正常运行的kubemark-master进行对比，找到自己出错的原因；
10. 在使用`pre-existing`模式时，脚本可能会报错，比如unbound variable或者函数未定义等等，这是kubemark代码对pre-existing支持不完善所致，对于unbound variable，我们可以随便赋值声明一下即可，对于未定义的函数，建议参照GCE模式下该函数的定义，自己重写一个；


### 如何搭建kubemark-master？

1. 申请一台vm，用于搭建kubemark-master；
2. 安装docker以及kubelet（v1.11.1）；
3. 创建用户kubernetes，并将工作机的公钥写入authorized_keys中，使其能够以kubernetes用户的身份免密码ssh登入kubemark-master；
```bash
adduser kubernetes # 创建用户kubernetes
sudo vim /etc/sudoers # 追加一行：kubernetes ALL=(ALL) NOPASSWD:ALL，赋予无密码sudo权限
sudo usermod -aG docker kubernetes # 加入docker组，允许直接操作docker
```
4. 阅读`start-kubemark.sh`脚本中的`copy-resource-files-to-master`函数源码，手动将对应文件拷贝到kubemark-master节点上；
5. 准备一块磁盘（或单独分一个分区），用于etcd存储，（假设分区名为`/dev/vdb1`，注意到`start-kubemark-master.sh`源码中通过`/dev/disk/by-id/google-master-pd`来索引该分区，这只在GCE下生效，我们需要修改源码，如下所示；
``` bash
# local -r pd_path="/dev/disk/by-id/${pd_name}"
local -r pd_path="/dev/vdb1"
### 在mount-pd函数中，注释第一行，改为第二行
```
6. 在`start-kubemark-master.sh`脚本中，删除`params+=" --cloud-provider=gce"`一行，并注释掉`start-kubelet`函数调用；
7. 尝试第一次运行`start-kubemark-master.sh`脚本，这一步主要是为了挂载磁盘、加载编译好的kubernetes镜像、填写各组件manifest模板等等，脚本最后会因为apiserver无响应而超时结束；
8. 使用GCE上保留的工作机来生成证书，即执行`start-kubemark.sh`中的`generate-pki-config`函数，并仿照`write-pki-config-to-master`函数将证书、token写到本地，注意需要跟踪到`create-certs`函数内部，将机器ip地址替换为新的kubemark-master的IP地址，否则证书无法在新机器中使用，最后将生成的所有文件按原目录结构拷贝到新master节点的`/etc/srv/kubernetes`目录下;
9. 根据生成的`known_tokens.csv`，修改`start-kubemark.sh`源码，修改hollow node中各组件的token，使其保持一致；
10. 仔细观察kubernetes组件的manifest文件中的command字段，修复其中留空的参数项（具体可以参照GCE中保留的原kubemark-master），否则这会导致组件启动失败。

### 个人体会

个人认为，解决这类问题的最佳方法是**对比实验**：比如我不确定apiserver的参数项要怎么填，这时候我可以参照GCE上正常运行的kubemark-master；或是，我不知道该怎么生成证书、token，但我可以跟踪脚本在GCE上的证书生成流程，通过替换代码，直接运行脚本生成在本地可用的证书等等。
还有一个重要的点是，遇到问题，比如kubernetes组件无法启动、kubemark集群无法部署应用以及e2e测试失败（这些我都遇到过）等等，第一件事应该是查找日志或具体报错信息。比如我发现apiserver和controller manager两个容器始终无法启动，于是我按照上文提到的方法（参见`整体建议 -> 8`）找到错误原因在于启动命令的参数存在留空；还有，kubemark集群搭建起来后，集群状态看似正常，但是e2e测试中的Density总会失败，于是我登录到apiserver容器内，查看日志发现有大量实时产生的认证失败信息，于是我猜测是hollow node的证书不对，果不其然，我忘记修改`start-kubemark.sh`源码（参见`如何搭建kubemark-master？ -> 9`），hollow node组件的token仍然是随机生成的，将其替换为固定的token即可解决这一问题。
