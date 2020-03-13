## sts+cbs VS sts+ebs

### 测试场景

1. sts挂10块盘，包含创建pvc的创建时间
2. sts挂10块盘，不包含创建pvc的创建时间
3. sts挂10块盘，failover（删除pod之后立马重新创建pod）
4. sts挂16块盘，包含创建pvc的创建时间
   - 尝试了aws的eks，只能调度成功16块盘（官方写每个node最大挂载39块）



### 基本信息

- 集群版本
  - tke：v1.14.3-tke.9
  - eks(aws)：v1.14.9-eks-502bfb

### 结论

1、从pod开始有调度事件，到kubelet启动完容器

| 测试场景\块存储类别 | sts+cbs      | sts+ebs     |
| ---------- | ------------ | ----------- |
| 场景1        | 386s（6m26s）  | 40s         |
| 场景2        | 376s         | 9s          |
| 场景3        | 638s(10m38s) | 136s(2m16s) |
| 场景4        | 约21min       | 25s         |



2、detach时间（通过控制台观察1个node上10块盘detach花费总时间）

- cbs：310s
- ebs：70s



3、分析

- 单块cbs和ebs创建时间差别不大。但sts+多块cbs相比+多块ebs耗时巨大，主要是因为cbs不支持并发attach/detach，导致controller manager一直backoff。这从pod的事件中也可以看出来。
- 另外从场景1和场景4对比来看，由于cbs不支持并发attach/detach，pod启动时间也与所引用的cbs数量正相关，而使用ebs的pod启动时间并没有和所偶引用的ebs数量正相关。



### 测试过程与日志

#### 场景1

- sts+cbs 10盘

```
  Warning  FailedScheduling        13m (x11 over 13m)  default-scheduler        pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled               12m                 default-scheduler        Successfully assigned default/ivantestweb-0 to 172.21.33.130
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351788d3-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(20f4bed65694)resource is busy, please retry later, RequestId=63532bef-5571-4df9-8887-20f4bed65694
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351d45f6-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(86d2199a7e48)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=bc5e8d80-8e06-4dd5-839e-86d2199a7e48
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-35168c3a-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(4b890245304a)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=4ade9be2-a2b7-47fa-a043-4b890245304a
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351a005c-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(c3d6c897692c)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is attaching disk, please retry later, RequestId=46331708-40ce-44c3-aa26-c3d6c897692c
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351e2465-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(b9122ff5387d)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=395b26e5-f142-42a5-8988-b9122ff5387d
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351f2457-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(35c9394ab3b5)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=c905be31-b9a0-4a50-b249-35c9394ab3b5
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351cc344-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(b826061ab647)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=70354552-f9bc-40c6-b667-b826061ab647
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351bc13a-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(a7eb86632798)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is attaching disk, please retry later, RequestId=ac7bab72-2a35-45ae-b422-a7eb86632798
  Warning  FailedAttachVolume      12m                 attachdetach-controller  AttachVolume.Attach failed for volume "pvc-351b1909-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(80224e223de2)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is attaching disk, please retry later, RequestId=a1faffc2-22ea-47c0-bf7a-80224e223de2
  Normal   SuccessfulAttachVolume  12m                 attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-3518a490-59dc-11ea-bce1-de73adcfd030"
  Warning  FailedAttachVolume      12m (x15 over 12m)  attachdetach-controller  (combined from similar events): AttachVolume.Attach failed for volume "pvc-351cc344-59dc-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(60c66053cf5b)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=f888de6d-1b67-4879-ada4-60c66053cf5b
  Warning  FailedMount             10m                 kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(3520324b-59dc-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www4 www10 www1]. list of unattached volumes=[www6 www7 www9 www3 www2 www4 www5 www8 www10 www1 default-token-rd5v7]
  Warning  FailedMount             8m39s               kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(3520324b-59dc-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www10]. list of unattached volumes=[www6 www7 www9 www3 www2 www4 www5 www8 www10 www1 default-token-rd5v7]
  Normal   Pulled                  6m40s               kubelet, 172.21.33.130   Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created                 6m40s               kubelet, 172.21.33.130   Created container nginx
  Normal   Started                 6m40s               kubelet, 172.21.33.130   Started container nginx
```



- sts+ebs 10盘

```
Events:
  Type     Reason                  Age                From                                                 Message
  ----     ------                  ----               ----                                                 -------
  Warning  FailedScheduling        87s                default-scheduler                                    Operation cannot be fulfilled on persistentvolumeclaims "www1-ivantestweb-0": the object has been modified; please apply your changes to the latest version and try again
  Warning  FailedScheduling        77s (x3 over 85s)  default-scheduler                                    AssumePod failed: pod 5c7cfd7c-59cd-11ea-b07d-068d9c1a98d6 is in the cache, so can't be assumed
  Normal   Scheduled               76s                default-scheduler                                    Successfully assigned default/ivantestweb-0 to ip-172-31-46-51.us-west-2.compute.internal
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c77d01b-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c78a345-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7b30de-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c765a6a-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7a52d8-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c75b866-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c79982b-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7729d4-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  53s                attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7514ca-59cd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  52s                attachdetach-controller                              (combined from similar events): AttachVolume.Attach succeeded for volume "pvc-5c7bdf15-59cd-11ea-b07d-068d9c1a98d6"
  Normal   Pulled                  37s                kubelet, ip-172-31-46-51.us-west-2.compute.internal  Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created                 37s                kubelet, ip-172-31-46-51.us-west-2.compute.internal  Created container nginx
  Normal   Started                 37s                kubelet, ip-172-31-46-51.us-west-2.compute.internal  Started container nginx
```



#### 场景2

- sts+cbs 10盘

```
Events:
  Type     Reason                  Age                     From                     Message
  ----     ------                  ----                    ----                     -------
  Normal   Scheduled               6m31s                   default-scheduler        Successfully assigned default/ivantestweb-0 to 172.21.33.130
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52600a5-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(176e756f91f7)resource is busy, please retry later, RequestId=7c01a7db-f8af-48b5-b812-176e756f91f7
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52cc601-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(d8e140e71c0c)resource is busy, please retry later, RequestId=5e954c13-b3bf-4c9c-a51e-d8e140e71c0c
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52bcaf9-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(db44b7ccd57c)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=b4290fd0-146d-4573-a226-db44b7ccd57c
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f528fcfc-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(879e3ae47bbb)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=a55c298b-3294-4269-97d6-879e3ae47bbb
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f526bf7a-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(37db7580bb35)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=46b4ff5b-1bc2-4a36-b44e-37db7580bb35
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52ae21e-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(bfd5e1218f86)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=434d26e9-d0a5-4e3a-8e17-bfd5e1218f86
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52e7c7f-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(080db395c50d)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=273a11b9-55eb-40df-a695-080db395c50d
  Warning  FailedAttachVolume      6m27s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f529f4eb-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(804347180ad6)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=34794bb8-2061-4efa-931b-804347180ad6
  Warning  FailedAttachVolume      6m26s                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f527cb37-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(9d6e25f92b88)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=501bcd54-5b3e-413e-b57d-9d6e25f92b88
  Normal   SuccessfulAttachVolume  6m20s                   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-f52d8a4b-59d5-11ea-bce1-de73adcfd030"
  Warning  FailedAttachVolume      6m17s (x15 over 6m22s)  attachdetach-controller  (combined from similar events): AttachVolume.Attach failed for volume "pvc-f527cb37-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(94765c5a36be)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=ae2aa395-8acc-4f96-a5c7-94765c5a36be
  Warning  FailedMount             4m28s                   kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(026e8260-59da-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www10 www2 www4 www5]. list of unattached volumes=[www6 www8 www9 www10 www1 www2 www3 www4 www5 www7 default-token-rd5v7]
  Warning  FailedMount             2m13s                   kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(026e8260-59da-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www5]. list of unattached volumes=[www6 www8 www9 www10 www1 www2 www3 www4 www5 www7 default-token-rd5v7]
  Normal   Pulled                  15s                     kubelet, 172.21.33.130   Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created                 15s                     kubelet, 172.21.33.130   Created container nginx
  Normal   Started                 15s                     kubelet, 172.21.33.130   Started container nginx
```



- sts+ebs 10盘

```
Events:
  Type    Reason                  Age   From                                                 Message
  ----    ------                  ----  ----                                                 -------
  Normal  Scheduled               10s   default-scheduler                                    Successfully assigned default/ivantestweb-0 to ip-172-31-46-51.us-west-2.compute.internal
  Normal  SuccessfulAttachVolume  8s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c765a6a-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  8s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c75b866-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  8s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c78a345-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7514ca-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7729d4-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7a52d8-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c7bdf15-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c79982b-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-5c77d01b-59cd-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  6s    attachdetach-controller                              (combined from similar events): AttachVolume.Attach succeeded for volume "pvc-5c7b30de-59cd-11ea-b07d-068d9c1a98d6"
  Normal  Pulled                  1s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal  Created                 1s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Created container nginx
  Normal  Started                 1s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Started container nginx
```



#### 场景3

- cbs failover 10盘

```
Events:
  Type     Reason              Age                   From                     Message
  ----     ------              ----                  ----                     -------
  Warning  FailedScheduling    10m (x10 over 11m)    default-scheduler        pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled           10m                   default-scheduler        Successfully assigned default/ivantestweb-0 to 172.21.33.130
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52600a5-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(2aed9f21acb0)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=ff0955bb-4afa-4590-8d41-2aed9f21acb0
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52d8a4b-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(fe0f3ff44c3a)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=7ecce928-4394-49bc-b960-fe0f3ff44c3a
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f528fcfc-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(a39efc9142ee)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=a9904397-c745-4af2-a6dc-a39efc9142ee
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52cc601-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(2ddbdef41d6a)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=22df468e-17fb-4fce-bde7-2ddbdef41d6a
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52e7c7f-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(f4ecf034ac2c)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=22dcfd4e-505c-40a3-ae80-f4ecf034ac2c
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52bcaf9-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(51e3bb83415f)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=bd831a4a-246f-4cda-b219-51e3bb83415f
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f527cb37-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(b09b8b757453)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=9920dab5-5e33-48a0-b880-b09b8b757453
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f529f4eb-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(910f280f38c2)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=a920e5d6-9984-4b6d-addb-910f280f38c2
  Warning  FailedAttachVolume  10m                   attachdetach-controller  AttachVolume.Attach failed for volume "pvc-f52ae21e-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(0f26e2565f2a)InstanceId: ins-an54ec79, cvm(ins-an54ec79) is detaching disk, please retry later, RequestId=2e6e3956-11d2-4d7d-8cf3-0f26e2565f2a
  Warning  FailedMount         8m39s                 kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(f52f597a-59d5-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www5 www9 www10 www3 www4 www6]. list of unattached volumes=[www1 www2 www5 www8 www9 www10 www3 www4 www6 www7 default-token-rd5v7]
  Warning  FailedMount         6m23s                 kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(f52f597a-59d5-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www5 www9 www3]. list of unattached volumes=[www1 www2 www5 www8 www9 www10 www3 www4 www6 www7 default-token-rd5v7]
  Warning  FailedMount         4m8s                  kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(f52f597a-59d5-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www5 www3]. list of unattached volumes=[www1 www2 www5 www8 www9 www10 www3 www4 www6 www7 default-token-rd5v7]
  Warning  FailedAttachVolume  3m44s (x52 over 10m)  attachdetach-controller  (combined from similar events): AttachVolume.Attach failed for volume "pvc-f527cb37-59d5-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(a9eaa7f1a0c9)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=ac13ecd9-c034-41b0-bfb5-a9eaa7f1a0c9
  Warning  FailedMount         113s                  kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(f52f597a-59d5-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www5]. list of unattached volumes=[www1 www2 www5 www8 www9 www10 www3 www4 www6 www7 default-token-rd5v7]
  Normal   Pulled              23s                   kubelet, 172.21.33.130   Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created             23s                   kubelet, 172.21.33.130   Created container nginx
  Normal   Started             23s                   kubelet, 172.21.33.130   Started container nginx
```



- ebs failover 10盘

```
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason                  Age    From                                                 Message
  ----     ------                  ----   ----                                                 -------
  Normal   Scheduled               2m33s  default-scheduler                                    Successfully assigned default/ivantestweb-0 to ip-172-31-46-51.us-west-2.compute.internal
  Normal   SuccessfulAttachVolume  101s   attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-adfdb60c-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  101s   attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-ae004f47-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  101s   attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-adfacb99-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  101s   attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-ae016a43-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  101s   attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-ae026328-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  90s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-ae033763-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  90s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-adfbbe04-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  78s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-adff8f76-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  78s    attachdetach-controller                              AttachVolume.Attach succeeded for volume "pvc-adfe7fff-59dd-11ea-b07d-068d9c1a98d6"
  Normal   SuccessfulAttachVolume  77s    attachdetach-controller                              (combined from similar events): AttachVolume.Attach succeeded for volume "pvc-adfcb880-59dd-11ea-b07d-068d9c1a98d6"
  Warning  FailedMount             30s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Unable to mount volumes for pod "ivantestweb-0_default(1fa45ea0-59e0-11ea-b07d-068d9c1a98d6)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www1 www8 www10]. list of unattached volumes=[www1 www2 www3 www8 www9 www10 www4 www5 www6 www7 default-token-grltz]
  Normal   Pulled                  17s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created                 17s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Created container nginx
  Normal   Started                 17s    kubelet, ip-172-31-46-51.us-west-2.compute.internal  Started container nginx
```



#### 场景4

- cbs 16盘

```
Events:
  Type     Reason              Age                    From                     Message
  ----     ------              ----                   ----                     -------
  Warning  FailedScheduling    21m (x15 over 22m)     default-scheduler        pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled           21m                    default-scheduler        Successfully assigned default/ivantestweb-0 to 172.21.33.130
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ae40dc1-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=RequestLimitExceeded, Message=Your current request times equals to `21` in a second, which exceeds the frequency limit `20` for a second. Please reduce the frequency of calls., RequestId=635a452b-1b69-42ff-963c-572257a44758
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ae292ea-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=RequestLimitExceeded, Message=Your current request times equals to `22` in a second, which exceeds the frequency limit `20` for a second. Please reduce the frequency of calls., RequestId=e3717c58-e38c-45f2-8df9-d77533bef889
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8aebbdb3-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=RequestLimitExceeded, Message=Your current request times equals to `23` in a second, which exceeds the frequency limit `20` for a second. Please reduce the frequency of calls., RequestId=d9b0b919-672d-4566-923c-e130d8b88c62
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ad6f1a1-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(27ae8a2d6226)resource is busy, please retry later, RequestId=babd66cb-381b-4766-a4b0-27ae8a2d6226
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ade15fd-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(060882469e4a)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=88cc4698-9e91-45f1-a788-060882469e4a
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ae7119d-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(f503e28e9208)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=0990b0df-ffc2-4105-96c4-f503e28e9208
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8adc050a-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(c4bebf4a033b)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=bf3a1a32-1127-48b1-bbde-c4bebf4a033b
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ae5cc91-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(1d2e7d4b4d2f)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=5d691b41-6ead-48ec-ab2e-1d2e7d4b4d2f
  Warning  FailedAttachVolume  21m                    attachdetach-controller  AttachVolume.Attach failed for volume "pvc-8ae1a2a9-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(f2e156432348)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=3fcd22b7-4125-4052-8549-f2e156432348
  Warning  FailedMount         19m                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www11 www1 www10 www13 www4 www5 www2 www12 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         17m                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www11 www1 www10 www13 www2 www12 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         14m                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www11 www1 www10 www13 www2 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         12m                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www11 www1 www10 www2 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         10m                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www1 www10 www2 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         8m4s                   kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www10 www2 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         5m47s                  kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www2 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedAttachVolume  4m26s (x126 over 21m)  attachdetach-controller  (combined from similar events): AttachVolume.Attach failed for volume "pvc-8ae06306-59e1-11ea-bce1-de73adcfd030" : [TencentCloudSDKError] Code=ResourceBusy, Message=(0ce116feddf4)1adffd60-a38c-44ae-b20d-247e808da021,ins-an54ec79 is busy, please retry later, RequestId=9ebd5bd2-63a3-4587-b1c2-0ce116feddf4
  Warning  FailedMount         3m31s                  kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www6 www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Warning  FailedMount         77s                    kubelet, 172.21.33.130   Unable to mount volumes for pod "ivantestweb-0_default(8aecabc6-59e1-11ea-bce1-de73adcfd030)": timeout expired waiting for volumes to attach or mount for pod "default"/"ivantestweb-0". list of unmounted volumes=[www9]. list of unattached volumes=[www3 www6 www7 www11 www1 www10 www13 www4 www5 www8 www14 www2 www12 www15 www16 www9 default-token-rd5v7]
  Normal   Pulled              48s                    kubelet, 172.21.33.130   Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal   Created             47s                    kubelet, 172.21.33.130   Created container nginx
  Normal   Started             47s                    kubelet, 172.21.33.130   Started container nginx
```



- ebs 16盘

```
Events:
  Type    Reason                  Age                From                                                  Message
  ----    ------                  ----               ----                                                  -------
  Normal  Scheduled               31s                default-scheduler                                     Successfully assigned default/ivantestweb-0 to ip-172-31-50-189.us-west-2.compute.internal
  Normal  SuccessfulAttachVolume  28s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043f1ad3-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  28s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043c0659-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  24s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043cac50-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  24s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043abc5d-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  24s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043d2bce-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  24s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043e7687-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  18s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-0436b161-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  18s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-04396625-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  18s                attachdetach-controller                               AttachVolume.Attach succeeded for volume "pvc-043b6edc-59e1-11ea-b07d-068d9c1a98d6"
  Normal  SuccessfulAttachVolume  18s (x7 over 18s)  attachdetach-controller                               (combined from similar events): AttachVolume.Attach succeeded for volume "pvc-04377145-59e1-11ea-b07d-068d9c1a98d6"
  Normal  Pulled                  6s                 kubelet, ip-172-31-50-189.us-west-2.compute.internal  Container image "ccr.ccs.tencentyun.com/qcloud/nginx:1.9" already present on machine
  Normal  Created                 6s                 kubelet, ip-172-31-50-189.us-west-2.compute.internal  Created container nginx
  Normal  Started                 6s                 kubelet, ip-172-31-50-189.us-west-2.compute.internal  Started container nginx
```

