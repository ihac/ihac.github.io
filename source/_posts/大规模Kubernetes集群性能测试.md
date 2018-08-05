---
title: 大规模Kubernetes集群性能测试
date: 2018-08-05 22:57:19
tags: [Kubernetes, Kubemark, Benchmark]
visible: hide
---

本文主要介绍如何使用kubemark来模拟大规模Kubernetes集群（1000 node），并对其进行e2e性能测试，最后附上测试报告。

以下是本文将用到的相关名词：

- kubemark集群：表示模拟的Kubernetes集群；
- kubemark-master：表示kubemark集群的master节点，以vm形式存在；
- kubemark-hollow-node：表示kubemark集群的hollow node，即模拟的slave节点，以pod形式存在；
- 物理集群：表示以pod形式运行hollow node的底层集群；
- 工作机：表示用于编译Kubernetes、运行kubemark脚本的机器，可以是个人主机。

大致原理：

首先创建一台虚拟机，运行Kubernetes相关组件（比如apiserver、scheduler等等），作为kubemark集群的master节点；然后在物理集群上运行大量pod，向kubemark-master注册，成为kubemark集群中的slave节点；最后在搭建完成的kubemark集群上运行e2e测试。

## GCE

## Aliyun

走的一些弯路

1. 直接运行start-kubemark-master.sh脚本
2. 使用kubeadm搭建kubemark-master

