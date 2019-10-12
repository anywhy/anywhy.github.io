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

