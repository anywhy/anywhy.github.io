---
title: Kubernetes CSI(Container Storage Interface) 设计
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
4. Volume插件获得Kubernetes组件(kubelet和kube-controller-manager)的完全权限。
5. 插件开发人员被迫提供插件源代码，不能选择只发布可执行包。

## 概述

为了支持CSI兼容卷插件，Kubernetes将引入一个新的树内CSI卷插件。这个新的Volume插件将成为Kubernetes用户(应用程序开发人员和集群管理员)与外部CSI卷驱动程序交互的机制。

对新的`in-tree`CSI卷插件的`设置`/`拆卸`调用将通过节点计算机上的unix domain socket(UDS)直接调用`NodePublishVolume`和`NodeUnpublishVolume`CSI RPCs。

提供/删除和附加/分离必须由某个外部组件来处理，该组件代表CSI卷驱动程序监视Kubernetes API，并调用适当的CSI RPCs接口。

为了简化集成，Kubernetes团队将提供一个容器来捕获所有Kubernetes特定的逻辑，并充当第三方容器化CSI卷驱动程序和Kubernetes之间的适配器(CSI驱动程序的每个部署都有自己的适配器实例)。

## 设计详情

### 第三方 CSI Volume Drivers

Kubernetes对CSI卷驱动程序的打包和部署尽可能不严格。使用通信信道(记录在下面)是在Kubernetes启用任意外部CSI兼容存储驱动程序的唯一要求。

该文档推荐了一种在Kubernetes部署任意容器化CSI驱动程序的标准机制。存储提供商可以使用它来简化Kubernetes上容器化CSI兼容卷驱动程序的部署(参见下面的“Kubernetes上部署CSI驱动程序的推荐机制”一节)。然而，这个机制是严格可选的。

### 通信机制

#### Kubelet 到 CSI驱动

Kubelet(负责装载和卸载)将通过Unix域套接字与运行在同一主机(无论是否容器化)上的外部“CSI卷(Volume)驱动程序”通信。

CSI卷(Volume)驱动程序应该在节点机器上的以下路径上创建一个套接字：`/var/lib/kubelet/plugins/[SanitizedCSIDriverName]/csi.sock`。对于alpha版本的实现, kubelet将假定这是Unix Domain Socket与CSI卷驱动程序通信的位置。对于beta版的实现，我们可以考虑使用[Device Plugin Unix Domain Socket Registration](/contributors/design-proposals/resource-management/device-plugin.md#unix-socket)向kubelet注册Unix Domain Socket。这个机制需要扩展，以独立地支持CSI卷驱动程序和设备插件的注册。

`Sanitized CSIDriverName`是CSI驱动程序名，不包含危险字符，可以用作注释名。它可以遵循我们在[volume plugins](https://git.k8s.io/kubernetes/pkg/util/strings/escape.go#L27).中使用的相同模式。太长或难看的驱动程序名可以被拒绝使用，也就是说，所有在这个文档中描述的组件将报告一个错误，不会与这个CSI驱动程序通信。使用简洁驱动名名称是实施细则(最坏情况下为SHA)。

在外部“CSI卷驱动程序”初始化之后，kubelet必须调用CSI方法`NodeGetInfo`来获得从Kubernetes节点名到CSI驱动程序`NodeID`和相关的`accessible_topology`的映射。它必须:

* 使用`accessible_topology`中的`NodeID`和拓扑键为节点创建/更新一个`CSINodeInfo`对象实例。
  * 这将允许发出`ControllerPublishVolume`调用的组件使用`CSINodeInfo`作为从集群节点ID到存储节点ID的映射。
  * 这将使发出`CreateVolume`的组件能够重构`accessible_topology`，并提供来自特定节点的可访问卷。
  * 每个驱动程序必须完全覆盖它以前版本的`NodeID`和拓扑键(如果它们存在的话)。
  * 如果`NodeGetInfo`调用失败，kubelet必须删除此驱动程序之前的所有`NodeID`和拓扑键。
  * 当kubelet插件取消注册机制被实现时，当驱动程序取消注册时，删除`NodeID`和拓扑密钥。

* 以CSI驱动程序`NodeID`作为更新节点API对象`csi.volume.kubernetes.io/nodeid`注解的值。注解的值是一个JSON对象，包含每个CSI驱动程序的键/值对。例如: `csi.volume.kubernetes.io/nodeid: "{ \"driver1\": \"name1\", \"driver2\": \"name2\" }`
此注释已被弃用，将根据弃用策略(弃用后1年)删除。TODO标记弃用日期。

  * 如果`NodeGetInfo`调用失败，kubelet必须删除此驱动程序之前的任何`NodeID`。
  * 当kubelet插件取消注册机制实现时，当驱动程序取消注册时，删除`NodeID`和拓扑密钥。

* 使用`accessible_topology`作为标签创建/更新节点API对象。标签格式没有硬性限制，但是对于推荐设置使用的格式，请查看[Topology Representation in Node Objects](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#topology-representation-in-node-objects)。

为了方便部署外部容器化的CSI卷驱动程序，Kubernetes团队将提供一个边车(sidecar)“Kubernetes CSI助手”容器，该容器可以管理UDS(unix domain socket)注册和`NodeID`初始化。下面的“在Kubernetes上部署CSI驱动程序的建议机制”小节对此进行了详细说明。

新的API对象`CSINodeInfo`将被定义如下:

```go
// CSINodeInfo holds information about status of all CSI drivers installed on a node.
type CSINodeInfo struct {
    metav1.TypeMeta
    // ObjectMeta.Name must be node name.
    metav1.ObjectMeta

    // List of CSI drivers running on the node and their properties.
    CSIDrivers []CSIDriverInfo
}

// Information about one CSI driver installed on a node.
type CSIDriverInfo struct {
    // CSI driver name.
    Name string

    // ID of the node from the driver point of view.
    NodeID string

    // Topology keys reported by the driver on the node.
    TopologyKeys []string
}
```

选择新的对象类型`CSINodeInfo`替代`Node.Status`，因为Node对象已经足够大了与它的大小有问题有关。`CSINodeInfo`是在集群启动时由TODO (jsafrane)安装的CRD，在`kubernetes/kubernetes/pkg/apis/storage-csi/v1alpha1/types.go`中定义, 因此k8s.io/client-go和k8s.io/api自动生成。`CSINodeInfo`的所有用户将容忍如果CRD没有安装和重试任何他们需要做的指数后退和正确的错误报告。特别是当CRD缺失时，kubelet能够履行它的正常职责。

每个节点必须有0个或一个`CSINodeInfo`实例。这是由`CSINodeInfo.Name=Node.Name`名称。TODO:如何验证这个?每个`CSINodeInfo`都归属相应的Node，用于垃圾收集。

#### Master到CSI驱动通信

因为CSI卷驱动程序代码被认为是不可信的，所以可能不允许它在主机上运行。因此，Kube controller manager(负责创建、删除、附加和分离)不能通过UDS(Unix Domain Socket)与“CSI卷驱动程序”容器进行通信。相反，Kube controller manager管理器将通过Kubernetes API与外部“CSI卷驱动程序”通信。

更具体地说，一些外部组件必须代表外部CSI卷驱动程序监视Kubernetes API，并触发针对它的适当操作。这消除了在kube-controller-manager和CSI卷驱动程序之间发现和保护通道的问题。

在Kubernetes上轻松部署外部容器化的CSI卷驱动程序，而无需让驱动程序Kubernetes知道，Kubernetes将提供一个边车(sidecar)“Kubernetes to CSI”代理容器，该代理容器将监视Kubernetes API并触发针对“CSI卷驱动程序”容器的适当操作。下面的“在Kubernetes上部署CSI驱动程序的建议机制”小节对此进行了详细说明。

外部组件CSI卷驱动程序监视Kubernetes API处理配置、删除、附加和分离操作实现。

##### 配置和删除

配置和删除操作使用现有的`外部供应器机制`进行处理，其中代表外部CSI卷驱动程序监视Kubernetes API的外部组件将充当外部供应器。

简而言之，为了动态地提供一个新的CSI卷，集群管理员将创建一个`StorageClass`，其中的供应者与代表CSI卷驱动程序处理供应请求的外部供应者的名称对应。

为了提供一个新的CSI卷，终端用户将创建一个引用这个存储类的`PersistentVolumeClaim`对象。外部供应程序将对PVC的创建做出反应，并发出针对CSI卷驱动程序的`CreateVolume`调用来供应卷。与其他动态供应的卷一样，`CreateVolume`名称将自动生成。`CreateVolume`能力将从`PersistentVolumeClaim`对象中获取。`CreateVolume`参数将通过`StorageClass`参数传递(对Kubernetes不透明)。

如果`PersistentVolumeClaim`的卷名是`volume.alpha.kubernetes.io/selected-node`注解(只有在`StorageClass`中启用延迟卷绑定时才添加)，供应者将从相应的`CSINodeInfo`实例中获取相关的拓扑键和节点标签中的拓扑值，并使用它们在`CreateVolume()`请求中生成首选拓扑。如果未设置注解，则不会指定首选拓扑(除非PVC遵循本节后面讨论的`StatefulSet`命名格式)。来自`StorageClass`的`allowedtopology`作为必需的拓扑传递。如果`AllowedTopologies`是未指定的，则供应程序将在整个集群中传递一组聚合的拓扑值作为必需的拓扑。

要执行此拓扑聚合，外部供应程序将缓存所有现有节点对象。为了防止受影响的节点影响供应过程，它将选择单个节点作为密钥的真值来源，而不是依赖于存储在`CSINodeInfo`中的密钥来获取每个节点对象。对于使用后期绑定提供的pvc，所选节点是事实的来源;否则将选择一个随机节点。然后，供应者将遍历包含来自驱动程序的节点ID的所有缓存节点，使用这些键聚合标签。注意，如果集群中的拓扑键不同，则只会考虑与所选节点的拓扑键匹配的节点子集进行供应。

为了生成首选拓扑，外部供应程序将在`CreateVolume()`调用中为首选拓扑生成N个段，其中N是必需拓扑的大小。包含多个段来支持跨多个拓扑段可用的卷。来自所选节点的拓扑段总是首选拓扑中的第一个。所有其他段都是对其余必要拓扑的一些重新排序，这样，对于给定的必要拓扑(或任意对其重新排序)和选择的节点，可以保证首选拓扑集总是相同的。

如果设置了即时卷绑定模式，并且PVC遵循状态集命名格式，那么供应程序将选择一个来自必需的基于PVC名称的拓扑的段作为首选拓扑中的第一个段，这将确保拓扑在状态集的所有卷中均匀分布。该逻辑与GCE持久磁盘供应程序中的名称哈希逻辑类似。优先拓扑中的其他段的顺序与上面描述的相同。这个特性将在作为推荐部署方法的一部分提供的外部供应程序中标记门控。

一旦操作成功完成，外部供应程序将创建一个`PersistentVolume`对象，使用`CreateVolume`响应中返回的信息来表示该卷。返回卷的拓扑被转换为`PersistentVolume` `NodeAffinity`字段。然后将`PersistentVolume`对象绑定到`PersistentVolumeClaim`并可供使用。

拓扑密钥/值对的格式由用户定义，并且必须在以下位置之间匹配:

* `Node`拓扑标签
* `PersistentVolume` `NodeAffinity` 字段
* 当`StorageClass`启用了延迟卷绑定时，调度器将以以下方式使用节点的拓扑信息:
  1. 在动态供应期间，调度器通过将每个节点的拓扑与`StorageClass`中的`AllowedTopologies`进行比较，为供应程序选择一个候选节点。
  2. 在卷绑定和pod调度期间，调度器通过比较`PersistentVolume`的`Node`拓扑和`VolumeNodeAffinity`来为pod选择一个候选节点。

更详细的描述可以在[topology-aware volume scheduling design doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/volume-topology-scheduling.md)中找到。有关推荐部署方法使用的格式，请参阅[Topology Representation in Node Objects](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md#topology-representation-in-node-objects)。

要删除CSI卷，最终用户将删除相应的`PersistentVolumeClaim`对象。外部供应程序将对PVC的删除做出反应，并且基于其回收策略，它将针对CSI卷驱动程序命令发出`DeleteVolume`调用来删除卷。然后它将删除`PersistentVolume`对象。

##### 附加和分离

附加/分离操作也必须由外部组件(“attacher”)处理。attacher代表外部CSI卷驱动程序监视Kubernetes API，以获取新的`VolumeAttachment`对象(在下文中定义)，并触发针对CSI卷驱动程序的适当调用来附加卷。即使底层的CSI驱动程序不支持`ControllerPublishVolume`调用，因为Kubernetes对此一无所知，attacher也必须监视`VolumeAttachment`对象并将其标记为attached。

更具体地说，外部“attacher”必须代表外部CSI卷驱动程序监视Kubernetes API，以处理附加/分离请求。

一旦下列条件为真，外部提供者应该针对CSI卷驱动程序调用`ControllerPublishVolume`来将卷附加到指定的节点:

1. 由Kubernetes attach/detach控制器创建一个新的`VolumeAttachment`Kubernetes API对象。
2. `VolumeAttachment.Spec.Attacher`值对应于外部Attacher的名称。
