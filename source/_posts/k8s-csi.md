---
title: Kubernetes CSI Volume 插件设计
date: 2019-10-12 11:21:08
categories: Kubernetes
tags: [CSI,k8s,storage]
---

# CSI Volume Plugins in Kubernetes Design Doc

## 术语

术语 | 解析
---|---
Container Storage Interface (CSI) | 一种试图建立行业标准接口的规范，容器编排系统(COs)可以用来将任意存储系统暴露给它们的容器化工作负载。
in-tree | Kubernetes 核心代码库中的代码。
out-of-tree | Kubernetes 核心代码库之外的代码。
CSI Volume Plugin | 核心代码库中一个新的volume插件，用作适配器，支持第三方Kubernetes CSI volume驱动程序。
CSI Volume Driver | 第三方CSI插件驱动实现，可以用过CSI Volume Plugin在Kubernetes中使用。

<!-- more -->
## 背景&动机

Kubernetes volume 插件当前是“in-tree”方式实现，这样意味着它们与核心库本内二进制文件链接、编译、构建和交付。 向Kubernes(volume)添加新的存储系统需要将需改Kubernes核心代码库。这是不可取的，原因有很多，包括:

1. Volume插件发布是一个紧密的过程而且依赖于Kubernetes releases版本。
2. Kubernetes开发人员/社区负责测试和维护所有卷插件，而不仅仅是测试和维护稳定的插件应用编程接口。
3. Volume插件中的错误会导致关键的Kubernetes组件崩溃，而不仅仅是插件。 
4. Volume插件获得Kubernetes组件(kubeletkube-controller-manager)的完全权限。
5. 插件开发人员被迫提供插件源代码，不能选择只发布可执行包。

## 概述

为了支持CSI兼容卷插件，Kubernetes将引入一个新的树内CSI卷插件。这个新的Volume插件将成为Kubernetes用户(应用程序开发人员和集群管理员)与外部CSI卷驱动程序交互的机制。

对新的`in-tree`CSI卷插件的`设置`/`拆卸`调用将通过节点计算机上的unix domain socket(UDS)直接调用`NodePublishVolume`和`NodeUnpublishVolume`CSI RPCs。

提供/删除和附加/分离必须由某个外部组件来处理，该组件代表CSI卷驱动程序监视Kubernetes API，并调用适当的CSI RPCs接口。

为了简化集成，Kubernetes团队将提供一个容器来捕获所有Kubernetes特定的逻辑，并充当第三方容器化CSI卷驱动程序和Kubernetes之间的适配器(CSI驱动程序的每个部署都有自己的适配器实例)。

## 设计详情

### 第三方 CSI Volume Drivers

Kubernetes对CSI卷驱动程序的打包和部署尽可能不严格。使用通信信道(记录在下面)是在Kubernetes启用任意外部CSI兼容存储驱动程序的唯一要求。
