## scheduler支持cbs的maxAttachLimit

### 背景

- 目前我们单个cvm最多可attach 20块cbs，包括系统盘
- 而k8s在调度使用cbs的pod的时候，并没有剔除这些可能会挂载超过20块磁盘的node，从而导致某些pod attach失败而一直处于`ContainerCreating`




### 目的

- 让使用cbs的pod在调度时，不会被调度到即将挂载超过20块cbs的node



### scheduler现状

````
1.14 scheduler中支持aws-ebs，GCE和AzureDisk的`maxAttachLimit`。
	- 也就是说，如果某个pod调度到某个node上去，会导致node上挂载的volume数量超过限制(25/39/16)，则会把这个pod调度到其他符合条件的node上。
````

目前支持的方式有2种：

1、在`Predicates`阶段为每个cloud provider添加一个predicate policy。

- `MaxEBSVolumeCountPred`/`MaxGCEPDVolumeCountPred`/`MaxAzureDiskVolumeCountPred`。 1.14开始标记为`DEPRECATED`，由方式2统一替代。


- 以`MaxEBSVolumeCountPred`为例，大致流程是：

  - ebs有唯一的`volumeLimitKey`（`attachable-volumes-aws-ebs`)和filter（`EBSVolumeFilter`）

    - filter中实现`FilterVolume`和`FilterPersistentVolume`。判断`volume.AWSElasticBlockStore`和`pv.Spec.AWSElasticBlockStore`是否为空

  - 之后根据filter来分别过滤需要新增的volumes（`pod.Spec.Volumes`）和，node上已有的volumes（`existingPod.Spec.Volumes`），过滤出其中包含的ebs，分别得到`newVolumes`和`existingVolumes`

  - 比对`newVolumes`和`existingVolumes`，从`newVolumes`中删除二者交集

  - ***以三种方式获取`maxAttachLimit`，优先级为3>2>1***

    1. 从环境变量获取（`KUBE_MAX_PD_VOLS`）

    2. 使用内部默认值

    3. 若feature gate`AttachVolumeLimit`开启，则根据`volumeLimitKey`从`node.status.allocatable`获取

       ![ebs-node](.\ebs-node.png)

  - 判断如果`len(existingVolumes) + len(newVolumes) > maxAttachLimit ` ，则该predicate返回false

- `AttachVolumeLimit`是kubernetes默认的feature gate

  ![AttachVolumeLimit](.\AttachVolumeLimit.png)

- 目前aws单节点最大支持attach的ebs数量限制是谁patch到`node.status.allocatable`的？

  - 目前是在awsebs intree中实现了`VolumePluginWithAttachLimits`接口，其中包含`GetVolumeLimits`和`VolumeLimitKey`
  - 如果`AttachVolumeLimit`开启，kubelet会在`initialNode`和`tryUpdateNodeStatus`时，将默认的`maxAttachLimit`patch到node上
  - 不排除aws有其他组件会patch以修改



2、统一以csi的方式使用存储，走`MaxCSIVolumeCountPred`

- ***这个方式要求必须以csi方式使用存储***


- 大致流程为：
  - 首先需要feature gate`AttachVolumeLimit`开启
  - 然后，从`node.status.allocatable`中获取所有前缀为`attachable-volumes-`的`maxVolumeLimit`，存入`nodeVolumeLimits`
  - 分别过滤需要新增的volumes（`pod.Spec.Volumes`）和，node上已有的volumes（`existingPod.Spec.Volumes`），分别得到`newVolumes`和`attachedVolumes`
    - 根据volume获取pvc->pv->csi driver，最终以`"attachable-volumes-csi-" + drivername`的方式拼成一个`volumeLimitKey`
  - 比对`newVolumes`和`attachedVolumes`，从`newVolumes`中删除二者交集
  - 从`nodeVolumeLimits`中拿到对应`volumeLimitKey`的`maxVolumeLimit`，判断如果该类型的已有+新增>`maxVolumeLimit`，则返回false



### 方案

#### 1 对比

- 目前tke中cbs的使用大多数是以cbs intree的方式使用的，csi cbs数量上相对较少。到完全切换到csi可能可能是个较为漫长的过程

- 从[scheduler现状](#scheduler现状)可知，想让scheduler支持cbs关于`maxAttachLimit`的调度，可以有两个方案，各有优劣：

  |                                        | 优点                                       | 缺点                                       |
  | -------------------------------------- | ---------------------------------------- | ---------------------------------------- |
  | **方案1**：<br>单独添加关于cbs的predicate policy | 1、更适合我们cbs目前现状（以cbs intree方式为主）<br>2、可以不依赖feature gate`AttachVolumeLimit`（1.10没有该feature gate） | 1、在1.14版本被标记为`DEPRECATED` <br>2、需要在k8s intree（scheduler）中添加代码，这个应该也不好合入社区，pr维护是个问题 |
  | **方案2**：<br> 以csi方式使用存储                | csi是社区趋势，最终应该都会使用csi                     | 1、必须使用csi方式来使用存储<br>2、1.8，1.10版本不支持。（需要feature gate`AttachVolumeLimit`开启。当然我们可以从某个版本（比如1.14）开始才支持cbs的`maxAttachLimit`）。 |

#### 2 设置`maxAttachLimit`

- 首先，无论采用何种方案，都可以在cbs intree中实现`VolumePluginWithAttachLimits`接口，由kubelet去把默认值set到`node.status.allocatable`。之后看由其他方式灵活设置`node.status.allocatable`。
- 采用方案1，需要根据k8s版本分情况讨论如何灵活设置`maxAttachLimit`
  - k8s版本在1.12以上，可以有2种方式来进行来动态设置`maxAttachLimit`
    1. 给scheduler设置环境变量`KUBE_MAX_PD_VOLS`
    2. 通过某个外部组件给node patch `node.status.allocatable`
       - `"attachable-volumes-qcloud-cbs": "16" `
  - k8s版本在1.12以下，仅有一种方式：
    - 给scheduler设置环境变量`KUBE_MAX_PD_VOLS`
- 采用方案2，由于feature gate `AttachVolumeLimit`，也需要根据k8s版本分情况
  - k8s版本在1.12以上，可以通过某个外部组件给node patch `node.status.allocatable`
    - `"attachable-volumes-csi-com.tencent.cloud.csi.cbs": "16" `
  - k8s版本在1.12以下，不支持

```
说明：
- 由scheduler现状可知，方案1在1.14版本获取的maxAttachLimit来自于三个地方（优先级3>2>1）：
  1. 从环境变量获取（KUBE_MAX_PD_VOLS）
  2. 使用内部默认值
  3. 若feature gate `AttachVolumeLimit`开启，则根据volumeLimitKey从node.status.allocatable获取
- 其中3，aws-ebs实现了VolumePluginWithAttachLimits接口，kubelet在initialNode和tryUpdateNodeStatus时会调用VolumePluginWithAttachLimits接口的方法获取默认的maxAttachLimit，patch到node status

- feature gate `AttachVolumeLimit`最低支持版本是1.11，1.12默认开启，所以1.12以下版本的集群无法从node.status.allocatable中获取
```



#### 3 流程图

![方案流程图](.\方案流程图.png)

- 方案流程如上：
  - 0：我们首先在cbs intree中实现`VolumePluginWithAttachLimits`接口，让kubelet在`initialNode`时，将默认值`attachable-volumes-qcloud-cbs: "16"`设置到`Node.Status.Allocatable`中
  - 1、用户创建引用了cbs的pod
  - 2、scheduler在predicates阶段调用`MaxCBSVolumeCountPred`策略预选node，此时从`KUBE_MAX_PD_VOLS`和`Node.Status.Allocatable`获取`maxAttachLimit`，皆无，则使用默认值。如果node上已有cbs数量与新需cbs数量大于`maxAttachLimit`，则该node被淘汰
  - 3、scheduler选出合适node
  - 4、对应node上kubelet watch到新pod，进行启动
- 需要确定的问题：
  - 1、采用何种方案
  - 2、以何种方式灵活设置`maxAttachLimit`



