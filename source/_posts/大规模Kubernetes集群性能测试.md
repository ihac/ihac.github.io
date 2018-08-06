---
title: 大规模Kubernetes集群性能测试
date: 2018-08-05 22:57:19
tags: [Kubernetes, Kubemark, Benchmark]
visible: hide
---

本文主要介绍如何使用kubemark来模拟大规模Kubernetes集群（1000 node），并对其进行e2e性能测试，最后附上测试报告。

**NOTE**
本次测试

## 背景

kubemark的设计初衷就是为了让用户能够非常方便地模拟任意规模的Kubernetes集群，并对集群进行性能测试，根据我在网上搜集到的资料，kubemark模拟出来的集群与真实集群在性能方面是非常接近的，因此kubemark应当是一个可行且易用的benchmark工具。

那为什么我想要将这次测试记录成博文呢？事实上，目前kubemark的易用性仅在GCE平台上有效，查看`start-kubemark.sh`脚本可以发现，GCE是kubemark目前唯一支持的cloud provider，虽然还留有`pre-existing`这么一个可选项，理论上来说可以支持任意Kubernetes平台，但官方文档对这一feature言之甚少，仅有一句：

> The pre-existing provider is an advanced use case that requires the developer to have knowledge of setting up a kubemark master.

我的实践经历也证明，在其它云平台搭建kubemark集群并进行性能测试确实不是一件容易的差事。因此，我写这篇总结博文主要出于两点考虑：一方面，是对自己连续3天高压工作、攻克多个难关的肯定，以及对最终成果的珍惜；另一方面，网上针对kubemark的指导文档几乎不存在，官方文档虽然很赞（详细介绍了kubemark的工作流程），但没能介绍如何在GCE以外的云平台上（或标准Kubernetes集群）搭建kubemark集群，希望我的经验能够帮到后来人吧。

## 名词解释

以下是本文将用到的相关名词：

- kubemark集群：表示模拟的Kubernetes集群；
- kubemark-master：表示kubemark集群的master节点，以vm形式存在；
- kubemark-hollow-node：表示kubemark集群的hollow node，即模拟的slave节点，以pod形式存在；
- 物理集群：表示以pod形式运行hollow node的底层集群；
- 工作机：表示用于编译Kubernetes、运行kubemark脚本的机器，可以是个人主机。

## 大致流程（原理）

首先创建一台虚拟机，运行Kubernetes相关组件（比如apiserver、scheduler等等），作为kubemark集群的master节点；然后在物理集群上运行大量pod，向kubemark-master注册，成为kubemark集群中的slave节点；最后在搭建完成的kubemark集群上运行e2e测试。

## GCE + kubemark

### 步骤

在GCE平台搭建kubemark集群比较简单，大致步骤如下：

1. 创建一个Kubernetes集群（即物理集群）与一个VM实例（即工作机）；
2. 在工作机上配置kubeconfig（具体命令参见集群connect说明），使其可以直接通过kubectl访问物理集群；
3. 在工作机上克隆Kubernetes仓库，在repo根目录下，执行命令`make quick-release`进行编译打包；
4. 在repo根目录下，执行`test/kubemark/start-kubemark.sh`脚本，自动创建kubemark集群；
5. kubemark集群创建成功后，执行`test/kubemark/run-e2e-tests.sh`脚本，运行e2e性能测试。

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

4. 为什么不使用GCE测试1000节点规模的集群？
    **answer**:
    因为GCE免费账户限制了资源使用量，参见[官方说明](https://cloud.google.com/free/docs/frequently-asked-questions#limitations)，几十个node估计就是上限了。

## Aliyun

走的一些弯路

1. 直接运行start-kubemark-master.sh脚本
2. 使用kubeadm搭建kubemark-master

