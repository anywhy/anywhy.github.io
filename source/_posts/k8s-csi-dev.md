---
title: Kubernetes CSI(Container Storage Interface) 开发文档
categories: Kubernetes
tags:
  - Kubernetes
  - CSI
  - storage
date: 2019-10-24 11:44:55
---


## 简介

本文讲解了如何在Kubernetes上开发、部署和测试[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)(CSI)驱动程序。

[Container Storage Interface](https://github.com/container-storage-interface/spec/blob/master/spec.md)(CSI)是将任意块和文件存储系统暴露给容器编排系统(COs，如Kubernetes)上的容器化工作。使用CSI第三方存储提供商可以编写和部署在Kubernetes中暴露新存储系统的插件，而无需触及Kubernetes的核心代码。

### Kubernetes版本

| Kubernetes | CSI规范兼容 | 状态 |
| ---------- | -------- | ------ |
| v1.9       | [v0.1.0](https://github.com/container-storage-interface/spec/releases/tag/v0.1.0)     | Alpha  |
| v1.10      | [v0.2.0](https://github.com/container-storage-interface/spec/releases/tag/v0.2.0)     | Beta   |
| v1.11      | [v0.3.0](https://github.com/container-storage-interface/spec/releases/tag/v0.3.0)     | Beta   |
| v1.13      | [v0.3.0](https://github.com/container-storage-interface/spec/releases/tag/v0.3.0), [v1.0.0](https://github.com/container-storage-interface/spec/releases/tag/v1.0.0) | GA     |

### 开发与部署

#### 最低要求(为Kubernetes开发和部署CSI驱动程序)

Kubernetes尽可能少地规定了CSI卷驱动程序的打包和部署。唯一的需求是Kubernetes(Master和Node)组件如何查找和与CSI驱动程序通信。

具体来说，以下是Kubernetes关于CSI的描述:

* Kubelet与CSI通信：
  * Kubelet通过UDS(Unix Domain Socket)直接向CSI驱动程序发出CSI调用(如`NodeStageVolume`、`NodePublishVolume`等)来挂载和卸载卷。
  * Kubelet通过[Kubelet插件注册机制](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins#device-plugin-registration)发现CSI驱动程序(以及用于与CSI驱动程序交互的UDS)。
  * 因此，部署在Kubernetes上的所有CSI驱动程序必须在每个Node上使用kubelet插件注册机制进行注册。
* Master与CSI驱动程序通信：
  * Kubernetes主组件不直接(UDS或其他方式)与CSI驱动程序通信。
  * Kubernetes主组件只与Kubernetes API交互。
  * 因此，需要依赖于Kubernetes API的操作的CSI驱动程序(如卷创建、卷附加、卷快照等)必须监视Kubernetes API并触发相应的CSI操作。

这些需求是最低限度的规定，CSI驱动程序开发人员可以自由地实现和部署他们认为合适的驱动程序。

>为了简化开发和部署，建议使用下面描述的机制。

#### 推荐机制(用于开发和部署Kubernetes的CSI驱动程序)

Kubernetes开发团队已经建立了一个“推荐机制”来开发、部署和测试Kubernetes上的CSI驱动程序。它旨在减少样板代码和简化CSI驱动程序开发人员的整个过程。

“推荐机制”利用了以下组件:

* Kubernetes CSI [边车(Sidecar)容器](https://github.com/kubernetes-csi/docs/blob/master/book/src/sidecar-containers.md)。
* Kubernetes CSI [Objects](https://github.com/kubernetes-csi/docs/blob/master/book/src/csi-objects.md)。
* CSI驱动程序测试工具。

要使用这种机制实现CSI驱动程序，CSI驱动程序开发人员应该:

1. 创建一个容器化的应用程序，实现CSI规范(CSI驱动程序容器)中描述的`Identity`、`Node`和可选的`Controller`服务。

    * 有关更多信息，请参见[开发CSI驱动程序](https://github.com/kubernetes-csi/docs/blob/master/book/src/developing.md)。
2. 使用CSI完整性对其进行单元测试。

    * 有关更多信息，请参见[驱动程序单元测试](https://github.com/kubernetes-csi/docs/blob/master/book/src/unit-testing.md)。
3. 定义Kubernetes API YAML文件，该文件部署CSI驱动程序容器和适当的边车(Sidecar)容器。
    * 有关更多信息，请参见在[Kubernetes部署](https://github.com/kubernetes-csi/docs/blob/master/book/src/deploying.md)。
4. 在Kubernetes集群上部署驱动程序，并在其上运行端到端的功能测试。
    * 参见[驱动程序-功能测试](https://github.com/kubernetes-csi/docs/blob/master/book/src/functional-testing.md)。

## 为Kubernetes开发CSI驱动程序

创建CSI驱动程序的第一步是编写实现CSI规范中描述的gRPC服务的应用程序。

至少，CSI驱动程序必须实现下列CSI接口服务:

* CSI`Identity`服务
  * 允许调用者(Kubernetes组件和CSI边车(Sidecar)容器)识别驱动程序及其支持的可选功能。
* CSI`Node`服务
  * 只需要`NodePublishVolume`、`NodeUnpublishVolume`和`NodeGetCapabilities`。
  * 所需的方法使调用方能够在指定的路径上使卷可用，并发现驱动程序支持哪些可选功能。

所有CSI服务可以在同一个CSI驱动程序应用程序中实现。CSI驱动程序应用程序应该被封装起来，以便于部署到Kubernetes上。一旦容器部署后，CSI驱动程序就可以与CSI[边车(Sidecar)容器](https://kubernetes-csi.github.io/docs/sidecar-containers.html)配对，并在适当的情况下部署在节点或z着成为控制器(Controller)服务。

### Capabilities

如果你的驱动支持附加功能，CSI“Capabilities”可以用来说明它支持的可选方法/服务，例如:

* `CONTROLLER_SERVICE` (`PluginCapability`)
  * 整个CSI`控制器(Controller)`服务是可选的。此功能指示驱动程序实现CSI`控制器(Controller)`服务中的一个或多个方法。
* `VOLUME_ACCESSIBILITY_CONSTRAINTS` (`PluginCapability`)
  * 此功能表明此驱动程序的卷可能无法从集群中的所有节点平等地访问，并且驱动程序将返回与拓扑相关的附加信息，Kubernetes可以使用这些信息来更智能地调度工作负载，或影响卷的供应位置。
* `VolumeExpansion` (`PluginCapability`)
  * 此功能表明驱动程序支持在创建后调整(扩展)卷。
* `CREATE_DELETE_VOLUME` (`ControllerServiceCapability`)
  * 此功能表明驱动程序支持动态卷挂载和卸载。
* `PUBLISH_UNPUBLISH_VOLUME` (`ControllerServiceCapability`)
  * 此功能指示驱动程序实现`ControllerPublishVolume`和`ControllerUnpublishVolume`这些操作对应于Kubernetes卷附加/分离操操作。例如，这可能导致对谷歌云控制平面执行`volume attach`操作，将指定的卷附加到谷歌Cloud PD CSI驱动程序的指定节点。
* `CREATE_DELETE_SNAPSHOT` (`ControllerServiceCapability`)
  * 此功能表明驱动程序支持供应卷快照，并支持使用这些快照供应新卷。
* `CLONE_VOLUME` (`ControllerServiceCapability`)
  * 此功能表明驱动程序支持克隆卷。
* `STAGE_UNSTAGE_VOLUME` (`NodeServiceCapability`)
  * 这个能力表明驱动程序实现了`NodeStageVolume`和`NodeUnstageVolume`这些操作对应于Kubernetes卷设备的挂载/卸载操作。例如，这可以用于创建块存储设备的全局(每个节点)卷挂载。
这是一个部分列表，请参阅[CSI规范](https://github.com/container-storage-interface/spec/blob/master/spec.md)中的完整功能列表。

### Kubernetes CSI边车(Sidecar)容器

Kubernetes CSI边车(Sidecar)容器是一套标准容器，旨在简化在Kubernetes上开发和部署CSI驱动程序。

这些容器包含监视Kubernetes API的通用逻辑，触发针对“CSI卷驱动程序”容器的适当操作，并根据需要更新Kubernetes API。

这些容器将与第三方CSI驱动容器捆绑在一起，并作为POD一起部署。

这些容器由Kubernetes存储社区开发和维护。

> 使用容器是严格可选的，但强烈建议使用。

使用边车(Sidecar)容器的好处包括:

* 减少“引用”代码。
  * CSI驱动程序开发者不必担心复杂的“Kubernetes专用”代码。
* 代码隔离
  * 与Kubernetes API交互的代码与实现CSI接口的代码隔离(然后在不同的容器中)。

Kubernetes开发团队维护以下Kubernetes CSI边车(Sidecar)容器:

* [external-provisioner](https://kubernetes-csi.github.io/docs/external-provisioner.html)
* [external-attacher](https://kubernetes-csi.github.io/docs/external-attacher.html)
* [external-snapshotter](https://kubernetes-csi.github.io/docs/external-snapshotter.html)
* [external-resizer](https://kubernetes-csi.github.io/docs/external-resizer.html)
* [node-driver-registrar](https://kubernetes-csi.github.io/docs/node-driver-registrar.html)
* [cluster-driver-registrar](https://kubernetes-csi.github.io/docs/cluster-driver-registrar.html) (废弃)
* [livenessprobe](https://kubernetes-csi.github.io/docs/livenessprobe.html)

## CSI Objects

状态: Beta
Kubernetes API包含以下CSI特定对象:

* CSIDriver Object
* CSINode Object

两者都是`storage.k8s.io/v1beta1` API的部分。

对象的模式定义可以在这里找到:[https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/storage/types.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/apis/storage/types.go)

### CSIDriver Object

#### 状态

* Kubernetes 1.12 - 1.13: Alpha
* Kubernetes 1.14: Beta

#### 什么是 CSIDriver Object

`CSIDriver` Kubernetes API对象有两个用途:

1. 简化驱动发现
    * 如果CSI驱动程序创建了CSIDriver对象，Kubernetes用户可以很容易地发现安装在其集群上的CSI驱动程序(只需发出`kubectl get CSIDriver`)
2. 定制Kubernetes行为
    * Kubernetes在处理CSI驱动程序时有一组默认的行为(例如，它默认调用附加/分离操作)。这个对象允许CSI驱动程序指定Kubernetes如何与之交互。

#### CSIDriver 对象属性

下面是一个v1beta1 `CSIDriver`对象的例子:

```yml
    apiVersion: storage.k8s.io/v1beta1
    kind: CSIDriver
    metadata:
    name: mycsidriver.example.com
    spec:
    attachRequired: true
    podInfoOnMount: true
    volumeLifecycleModes: # added in Kubernetes 1.16
    - Persistent
    - Ephemeral
```

有三个重要的字段:

* `name`
  * 他的名字应该与CSI驱动程序的全名对应。
* `attachRequired`
  * 指示此CSI卷驱动程序需要附加操作(因为它实现了CSI `ControllerPublishVolume`方法)，并且Kubernetes应该调用附加并等待任何附加操作完成后再继续挂载。
  * 如果给定的CSI驱动程序不存在`CSIDriver`对象，则默认值为`true`,这意味着将调用`attach`。
  * 如果给定的CSI驱动程序存在`CSIDriver`对象，但是没有指定该字段，那么它也默认为`true`,这意味着将调用`attach`。
有关更多信息，请参见[Skip Attach](https://kubernetes-csi.github.io/docs/skip-attach.html)。
* `podInfoOnMount`
  * 指示此CSI卷驱动程序在安装操作期间需要附加的pod信息(如pod名称、pod UID等)。
  * 如果未指定值或为`false`，将不会在挂载上传递pod信息。
  * 如果值设置为`true`, Kubelet将在CSI `NodePublishVolume`中以`volume_context`的形式传递pod信息:
    * `"csi.storage.k8s.io/pod.name": pod.Name`
    * `"csi.storage.k8s.io/pod.namespace": pod.Namespace`
    * `"csi.storage.k8s.io/pod.uid": string(pod.UID)`
  * 有关更多信息，请参见[POD挂载信息](https://kubernetes-csi.github.io/docs/pod-info.html)。
* `volumeLifecycleModes`
  * 这个字段是在Kubernetes 1.16中添加的，在使用较老的Kubernetes版本时无法设置。
  * 它通知Kubernetes有关驱动程序支持的降级模式。这可以确保用户没有错误地使用驱动程序。默认值是`Persistent`，这是常规的PVC/PV机制。此外，`Ephemeral`还支持[inline ephemeral volumes](https://kubernetes-csi.github.io/docs/ephemeral-local-volumes.html)(当两者都被列出时)或代替普通卷(当它是列表中唯一的一个条目时)。

#### 是什么创建了CSIDriver对象

要安装，CSI驱动程序的部署清单必须包含如上例所示的CSIDriver对象。

> 注意:Kubernetes 1.13中用于创建CSIDriver对象的cluster-driver-registrar边车(Sidecar)容器已经被Kubernetes 1.16所弃用。Kubernetes 1.14及以后版本还没有发布集群驱动程序注册表。

`CSIDriver`实例应该存在于所有使用相应的CSI驱动程序提供的卷的POD整个生命周期中，所以[Skip Attach](https://github.com/kubernetes-csi/docs/blob/master/book/src/skip-attach.md)和[Pod Info on Mount](https://github.com/kubernetes-csi/docs/blob/master/book/src/pod-info.md)可以正确工作。

#### 查看CSDriver注册清单

使用CSIDriver对象，现在可以查询Kubernetes来获得在集群中运行的注册驱动程序列表，如下所示:

```shell
$> kubectl get csidrivers.storage.k8s.io
NAME                  CREATED AT
hostpath.csi.k8s.io   2019-09-13T09:58:43Z
```

或着取得更详细的信息，你的注册司机与:

``` shell
$> kubectl describe csidrivers.storage.k8s.io
Name:         hostpath.csi.k8s.io
Namespace:    
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"storage.k8s.io/v1beta1","kind":"CSIDriver","metadata":{"annotations":{},"name":"hostpath.csi.k8s.io"},"spec":{"podInfoOnMou...
API Version:  storage.k8s.io/v1beta1
Kind:         CSIDriver
Metadata:
  Creation Timestamp:  2019-09-13T09:58:43Z
  Resource Version:    341
  Self Link:           /apis/storage.k8s.io/v1beta1/csidrivers/hostpath.csi.k8s.io
  UID:                 1860f2a1-85f8-4357-a933-c45e54f0c8e0
Spec:
  Attach Required:    true
  Pod Info On Mount:  true
  Volume Lifecycle Modes:
    Persistent
    Ephemeral
Events:  <none>
```

#### Alpha to Beta版本的变化

在开发过程中，CSIDriver对象也被定义为自定义资源定义(CRD)。作为升级到beta的一部分，该对象已被移动到内置的Kubernetes API。

在从alpha迁移到beta的过程中，该对象的API组从`csi.storage.k8s.io/v1alpha1`更改为`storage.k8s.io/v1beta1`。

在Kubernetes更新到新的内置类型时，不会自动更新现有的CRD及其CRs。

#### Kubernetes启用CSIDriver

在Kubernetes v1.12和v1.13中，因为这个特性是alpha版本，所以默认情况下是禁用的。要在这些版本中使用CSIDriver，请执行以下操作:

1. 通过以下Kubernetes特性标志确保启用特性门: `--feature-gates=CSIDriverRegistry=true`
2. 要么通过Kubernetes存储CRD插件确保CSIDriver CRD是自动安装的，要么使用以下命令在Kubernetes集群上手动安装CSIDriver CRD:

```shell
$> kubectl create -f https://raw.githubusercontent.com/kubernetes/csi-api/master/pkg/crd/manifests/csidriver.yaml
```

Kubernetes v1.14+使用了相同的Kubernetes特性标志，但是由于该特性是beta版，所以默认启用了它。由于API类型(从beta版开始)内建在Kubernetes API中，因此不再需要安装CRD。

### CSINode Object

#### 状态

* Kubernetes 1.12 - 1.13: Alpha
* Kubernetes 1.14: Beta

#### 什么是 CSINode Object

CSI驱动程序生成特定于节点的信息。不是将其存储在Kubernetes节点API对象中，而是创建了一个新的CSI特定的Kubernetes CSINode对象。

它的用途如下:

1. Kubernetes Node name 和 CSI Node name之间的映射

* CSI `GetNodeInfo`调用返回存储系统引用节点的名称。Kubernetes必须在将来的`ControllerPublishVolume`调用中使用这个名称。因此，当注册一个新的CSI驱动程序时，Kubernetes将存储系统节点ID存储在`CSINode`对象中，以供将来参考。

2. 驱动程序可用性

*  kubelet与kube-controller-manager和kubernetes调度器通信的一种方式，不管驱动程序在节点上是否可用(已注册)。

3. 卷的拓扑

* CSI `GetNodeInfo`调用返回一组标识该节点拓扑结构的键/值标签。Kubernetes使用这些信息进行拓扑感知供应(有关详细信息，请参阅PVC卷绑定模式)。它将键/值作为标签存储在Kubernetes节点对象上。为了回忆起哪个节点标签键属于一个特定的CSI驱动程序，kubelet将这些键存储在`CSINode`对象中以供将来参考。

#### CSINode 对象属性

下面是一个v1beta1 `CSINode`对象的例子:

```yaml
apiVersion: storage.k8s.io/v1beta1
kind: CSINodeInfo
metadata:
  name: node1
spec:
  drivers:
  - name: mycsidriver.example.com
    nodeID: storageNodeID1
    topologyKeys: ['mycsidriver.example.com/regions', "mycsidriver.example.com/zones"]
```

字段的含义：

* `drivers`- 运行在节点上的CSI驱动程序及其属性列表。
* `name` - 此对象引用的CSI驱动程序。
* `nodeID` - 由驱动程序确定的节点的指定标识符。
* `topologyKeys` - 驱动程序支持的分配给节点的拓扑密钥列表。

#### 是什么创建了CSINode对象

CSI驱动程序不需要直接创建`CSINode`对象。相反，它们应该使用`node-driver-registrar`边车(Sidecar)容器。这个边车(Sidecar)容器将通过kubelet插件注册机制与kubelet交互，代表CSI驱动程序自动填充`CSINode`对象。

#### Alpha to Beta版本的变化

alpha对象称为`CSINodeInfo`，而beta对象称为`CSINode`。alpha `CSINodeInfo`对象也被定义为自定义资源定义(CRD)。作为升级到beta的一部分，该对象已被移动到内置的Kubernetes API。

在从alpha迁移到beta的过程中，该对象的API组从`csi.storage.k8s.io/v1alpha1`更改为`csi.storage.k8s.io/v1beta1`。

在Kubernetes更新到新的内置类型时，不会自动更新现有的CRD及其CRs。


#### Kubernetes启用CSIDriver

在Kubernetes v1.12和v1.13中，因为这个特性是alpha版本，所以默认情况下是禁用的。要在这些版本中使用CSINodeInfo，请执行以下操作:

1. 确保使用`--features-gates=CSINodeInfo=true`启用功能。
2. 要么通过Kubernetes存储CRD插件确保`CSIDriver`CRD是自动安装的，要么使用以下命令在Kubernetes集群上手动安装`CSINodeInfo`CRD:

```shell
$> kubectl create -f https://raw.githubusercontent.com/kubernetes/csi-api/master/pkg/crd/manifests/csinodeinfo.yaml
```

Kubernetes v1.14+使用了相同的Kubernetes特性标志，但是由于该特性是beta版，所以默认启用了它。由于API类型(从beta版开始)内建在Kubernetes API中，因此不再需要安装CRD。

### Kubernetes上安装CSI插件

#### 概述

CSI驱动程序通常作为两个组件部署在Kubernetes中:控制器组件和每个节点组件。

#### Controller 插件

控制器组件可以作为部署或状态集部署在集群中的任何节点上。它由实现CSI控制器服务的CSI驱动程序和一个或多个边车(Sidecar)容器组成。这些控制器边车(Sidecar)容器通常与Kubernetes对象交互，并调用驱动程序的CSI控制器服务。

它通常不需要直接访问主机，可以通过Kubernetes API和外部控制平面服务执行所有操作。可以为HA部署控制器组件的多个副本，但是建议使用leader election来确保一次只有一个活动控制器。

控制器(Sidecar)容器包含`external-provisioner`,`external-attacher`,`external-snapshotter`和`external-resizer`。 在部署中包含一个边车(Sidecar)容器是可选的。

##### 与边车(Sidecar)通信

![CSI Controller Deployment Diagram](sidecar-container.png?raw=true "CSI Controller Deployment Diagram")

边车(Sidecar)容器管理Kubernetes事件并适当地调用CSI驱动程序。调用是通过边车(Sidecars)容器和CSI驱动程序之间的emptyDir卷共享一个UDS进行的。

##### RBAC 策略

大多数控制器边车与Kubernetes对象交互，因此需要设置RBAC策略。每个sidecar存储库都包含示例RBAC配置。

#### Node 插件

节点组件应该通过一个守护进程部署在集群中的每个节点上。它由实现CSI节点服务的CSI驱动程序和node-driver-registrar边车(Sidecar)容器组成。

##### 与Kubelet通信

![CSI Node Deployment Diagram](kubelet.png?raw=true "CSI Node Deployment Diagram")

Kubernetes kubelet在每个节点上运行，负责发出CSI节点服务调用。这些调用从存储系统挂载和卸载存储卷，使之可供Pod使用。Kubelet通过主机上通过主机路径卷共享的UDS调用CSI驱动程序。node-driver-registrar还使用第二个UDS将CSI驱动程序注册到kubelet。

#### 驱动卷挂载

节点插件需要直接访问主机，以使块设备和/或文件系统安装对Kubernetes kubelet可用。

CSI驱动程序使用的挂载点必须设置为双向，以允许主机上的Kubelet看到CSI驱动程序容器创建的挂载。请看下面的例子:

``` yaml
      containers:
      - name: my-csi-driver
        ...
        volumeMounts:
        - name: socket-dir
          mountPath: /csi
        - name: mountpoint-dir
          mountPath: /var/lib/kubelet/pods
          mountPropagation: "Bidirectional"
      - name: node-driver-registrar
        ...
        volumeMounts:
        - name: registration-dir
          mountPath: /registration
      volumes:
      # This volume is where the socket for kubelet->driver communication is done
      - name: socket-dir
        hostPath:
          path: /var/lib/kubelet/plugins/<driver-name>
          type: DirectoryOrCreate
      # This volume is where the driver mounts volumes
      - name: mountpoint-dir
        hostPath:
          path: /var/lib/kubelet/pods
          type: Directory
      # This volume is where the node-driver-registrar registers the plugin
      # with kubelet
      - name: registration-dir
        hostPath:
          path: /var/lib/kubelet/plugins_registry
          type: Directory
```

#### 启用特权POD

要使用CSI驱动程序，Kubernetes集群必须允许特权pod(即API服务器和kubelet必须将--allow-privileged标志设置为true)。这是某些环境(例如GCE、GKE、kubeadm)的默认设置。

确保你的API服务器启动与特权标志:

```shell
$ ./kube-apiserver ...  --allow-privileged=true ...
```

```shell
$ ./kubelet ...  --allow-privileged=true ...
```

> 注意:从Kubernetes 1.13.0开始，——对kubelet来说允许特权是正确的。在未来的kubernetes版本中，它将被弃用。
