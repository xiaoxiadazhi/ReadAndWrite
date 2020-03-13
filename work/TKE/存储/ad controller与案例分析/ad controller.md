## K8s存储 —— Attach/Detach Controller

k8s中涉及存储的组件主要有：attach/detach controller、pv controller、volume manager、volume plugins、scheduler。每个组件分工明确：

- **attach/detach controller**：负责对volume进行attach、detach
- **pv controller**：负责处理pv/pvc对象
- **volume manager**：主要负责对volume进行mount、unmount
- **volume plugins**：包含k8s原生的和各厂商的的存储插件
  - 原生的包括：emptydir、hostpath、flexvolume、csi等
  - 各厂商的包括：aws-ebs、azure、我们的cbs等
- **scheduler**：涉及到volume的调度

控制器模式是k8s非常重要的概念，一般一个controller会去管理一个或多个API对象，以让对象从实际状态/当前状态趋近于期望状态。

而attach/detach controller的作用就是让该被attach的volume能attach成功，该被detach的volume能detach成功。由于最近定位的一个问题，深入研究了下ad controller的代码，所以整理下，记录下来。



### 数据结构

对于ad controller来说，理解了其内部的数据结构，再去理解逻辑就事半功倍。ad controller在内存中维护2个数据结构：

1. `actualStateOfWorld`  ——   表征实际状态（后面简称asw）
2. `desiredStateOfWorld`  ——    表征期望状态（后面简称dsw）

很明显，对于声明式API来说，是需要随时比对实际状态和期望状态的，所以ad controller中就用了2个数据结构来分别表征实际状态和期望状态。



#### actualStateOfWorld

`actualStateOfWorld`  包含2个map：

- `attachedVolumes`： 包含了那些ad controller认为被成功attach到nodes上的volumes
- `nodesToUpdateStatusFor`： 包含要更新`node.Status.VolumesAttached` 的nodes



##### attachedVolumes

###### 如何填充数据？

1、在启动ad controller时，会populate asw，此时会list集群内所有node对象，然后用这些node对象的`node.Status.VolumesAttached` 去填充`attachedVolumes`。

2、之后只要有需要attach的volume被成功attach了，就会调用`MarkVolumeAsAttached`（`GenerateAttachVolumeFunc` 中）来填充到`attachedVolumes中`。

###### 如何删除数据？

1、只有在volume被detach成功后，才会把相关的volume从`attachedVolumes`中删掉。（`GenerateDetachVolumeFunc` 中调用`MarkVolumeDetached`)



##### nodesToUpdateStatusFor

######  如何填充数据？

1、detach volume失败后，将volume add back到`nodesToUpdateStatusFor`

​	- `GenerateDetachVolumeFunc` 中调用`AddVolumeToReportAsAttached`

###### 如何删除数据？

1、在detach volume之前会先调用`RemoveVolumeFromReportAsAttached` 从`nodesToUpdateStatusFor`中先删除该volume相关信息



#### desiredStateOfWorld

`desiredStateOfWorld` 中维护了一个map：

`nodesManaged`：包含被ad controller管理的nodes，以及期望attach到这些node上的volumes。



##### nodesManaged

###### 如何填充数据？

1、在启动ad controller时，会populate asw，list集群内所有node对象，然后把由ad controller管理的node填充到`nodesManaged`

2、ad controller的`nodeInformer` watch到node有更新也会把node填充到`nodesManaged`

3、另外在populate dsw和`podInformer` watch到pod有变化（add, update）时，往`nodesManaged` 中填充volume和pod的信息

4、`desiredStateOfWorldPopulator` 中也会周期性地去找出需要被add的pod，此时也会把相应的volume和pod填充到`nodesManaged` 

###### 如何删除数据？

1、当删除node时，ad controller中的`nodeInformer` watch到变化会从dsw的`nodesManaged` 中删除相应的node

2、当ad controller中的`podInformer` watch到pod的删除时，会从`nodesManaged` 中删除相应的volume和pod

3、`desiredStateOfWorldPopulator` 中也会周期性地去找出需要被删除的pod，此时也会从`nodesManaged` 中删除相应的volume和pod





### 流程简述

ad controller的逻辑比较简单：

1、首先，list集群内所有的node和pod，来populate `actualStateOfWorld` (`attachedVolumes` )和`desiredStateOfWorld` (`nodesManaged`)

2、然后，单独开个goroutine运行`reconciler`，通过触发attach, detach操作周期性地去reconcile asw（实际状态）和dws（期望状态）

 - 触发attach，detach操作也就是，detach该被detach的volume，attach该被attach的volume

3、之后，又单独开个goroutine运行`DesiredStateOfWorldPopulator` ，定期去验证dsw中的pods是否依然存在，如果不存在就从dsw中删除





### 现网案例

接下来结合现网的一个案例，来详细看看`reconciler`的逻辑，也顺便记录下定位问题的过程。



#### 问题描述

- 一个statefulsets(sts)引用了多个pvc cbs，我们更新sts时，会删除旧pod，创建新pod，如果新pod调度到和旧pod相同的节点，此时如果有多个cbs被同时调度到某一个node上挂载，就可能会让这些pod一直处于`ContainerCreating` 。



#### 现象

- `kubectl describe pod`

![88565 describe pod](.\88565 describe pod.png)

-  kubelet log

![88565 kubelet](.\88565 kubelet.png)

- `kubectl get node xxx -oyaml` 的`volumesAttached`和`volumesInUse`

```
volumesAttached:
  - devicePath: /dev/disk/by-id/virtio-disk-6w87j3wv
    name: kubernetes.io/qcloud-cbs/disk-6w87j3wv
volumesInUse:
  - kubernetes.io/qcloud-cbs/disk-6w87j3wv
  - kubernetes.io/qcloud-cbs/disk-7bfqsft5
```



#### 初步分析

- 从pod的事件可以看出来：ad controller认为cbs attach成功了，然后kubelet没有mount成功。
- ***但是***从kubelet日志却发现`Volume not attached according to node status` ，也就是说kubelet认为cbs没有按照node状态挂载。这个从node info也可以得到证实：`volumesAttached` 中的确没有这个cbs盘（disk-7bfqsft5）。
- node info中还有个现象：`volumesInUse` 中还有这个cbs。说明没有unmount成功

很明显，cbs要能被pod成功使用，需要ad controller和volume manager的协同工作。所以这个问题的定位首先要明确：

1. volume manager为什么认为volume没有按照node状态挂载，ad controller却认为volume attch成功了？
2. `volumesAttached`和`volumesInUse` 在ad controller和kubelet之间充当什么角色？

这里只对分析volume manager做简要分析。

- 根据`Volume not attached according to node status` 在代码中找到对应的位置，发现在`GenerateVerifyControllerAttachedVolumeFunc` 中。仔细看代码逻辑，会发现
  - volume manager的reconciler会先确认该被unmount的volume被unmount掉
  - 然后确认该被mount的volume被volume
    - 此时会先从volume manager的dsw缓存中获取要被mount的volumes（`volumesToMount`的`podsToMount` ）
    - 然后遍历，验证每个`volumeToMount`是否已经attach了
      - `这个volumeToMount`是由`podManager`中的`podInformer`加入到相应内存中，然后`desiredStateOfWorldPopulator`周期性同步到dsw中的
    - 验证逻辑中，在`GenerateVerifyControllerAttachedVolumeFunc`中会去遍历本节点的`node.Status.VolumesAttached`，如果没有找到就报错（`Volume not attached according to node status`）
- 所以可以看出来，***volume manager就是根据volume是否存在于`node.Status.VolumesAttached` 中来判断volume有无被attach成功***。
- 那谁去填充`node.Status.VolumesAttached` ？ad controller的数据结构`nodesToUpdateStatusFor` 就是用来存储要更新到`node.Status.VolumesAttached` 上的数据的。
- 所以，***如果ad controller那边没有更新`node.Status.VolumesAttached`，而又新建了pod，`desiredStateOfWorldPopulator` 从podManager中的内存把新建pod引用的volume同步到了`volumesToMount`中，在验证volume是否attach时，就会报错（Volume not attached according to node status）***
  - 当然，之后由于kublet的syncLoop里面会调用`WaitForAttachAndMount` 去等待volumeattach和mount成功，由于前面一直无法成功，等待超时，才会有会面`timeout expired` 的报错

所以接下来主要需要看为什么ad controller那边没有更新`node.Status.VolumesAttached`。



#### Attach/Detach Controller分析

接下来详细分析下ad controller的逻辑，看看为什么会没有更新`node.Status.VolumesAttached`，但从事件看ad controller却又认为volume已经挂载成功。

从[流程简述](#流程简述)中表述可见，ad controller主要逻辑是在reconciler中。

- reconciler定时去运行`reconciliationLoopFunc`，周期为100ms。
- `reconciliationLoopFunc`的主要逻辑在`reconcile()`中：
  - 首先确保该被detach的volume被detach掉
    - 遍历asw中的`attachedVolumes`，对于每个volume，判断其是否存在于dsw中
      - 去dsw.nodesManaged中判断node和volume是否存在
      - 存在的话，再判断volume是否存在
    - 如果volume存在于asw，且不存在于dsw，则意味着需要进行detach
    - 之后，根据`node.Status.VolumesInUse`来判断volume是否已经unmount完成，unmount完成或者等待6min timeout时间到后，会继续detach逻辑
    - 在执行detach volume之前，会先调用`RemoveVolumeFromReportAsAttached`从asw的`nodesToUpdateStatusFor`中去删除要detach的volume
    - 然后patch node，也就等于从`node.status.VolumesAttached`删除这个volume
    - 之后进行detach，如果detach失败，会把volume add back到`nodesToUpdateStatusFor`（之后在attach逻辑结束后，会再次patch node），如果是backoffError就直接跳过detach
  - 之后确保该被attach的volume被attach成功










``reconciliationLoopFunc` -> `sync` -> `syncStates` -> `updateStates` -> `MarkVolumeAsMounted`

```
Ad controller中会根据node.status.VolumesInUse来判断volume是否已经umount，以下2种情况才会去detach
```