### 基本信息

* 操作系统与内核版本：Linux version 4.4.0-104-generic (buildd@lgw01-amd64-022) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.5) ) #127-Ubuntu SMP Mon Dec 11 12:16:42 UTC 2017
* 4核 , 8GB , 1Mbps
* 创建一个无volume、镜像已存在node上的nginxpod



### 步骤

```
创建一个pod的整个流程的时序图如下图所示，相应的时间和步骤说明也列于下表。相应的日志见于附录2
```

![1557112608419](C:\Users\ivanscai\AppData\Roaming\Typora\typora-user-images\1557112608419.png)

| 序号 | 时间            | 组件       | 说明                                             | 备注                                                         |
| ---- | --------------- | ---------- | ------------------------------------------------ | ------------------------------------------------------------ |
| 1    | 10:35:18.636925 | API Server | API Server第一次接收到create pod的请求           |                                                              |
| 2    | 10:35:18.744457 | scheduler  | scheduler watch到新pod后尝试调度                 |                                                              |
| 3    | 10:35:18.754567 | scheduler  | scheduler完成绑定                                |                                                              |
| 4    | 10:35:18.759653 | kubelet    | kubelet接收到了新pod                             |                                                              |
| 5    | 10:35:18.778065 | kubelet    | kubelet（volume manager）等待volumeattach和mount |                                                              |
| 6    | 10:35:19.078148 | kubelet    | kubelet等到volume被attach和mount完               |                                                              |
| 7    | 10:35:19.078453 | kubelet    | kubelet开始创建sandbox                           | 从开始等待volume挂载到volume挂载成功，至少要300ms（wait.poll总是需要等一个interval） |
| 8    |                 | kubelet    | kubelet创建sandbox checkpoint                    |                                                              |
| 9    | 10:35:19.430912 | kubelet    | kubelet rewrite sandbox容器的resolv.conf         |                                                              |
| 10   | 10:35:19.430973 | kubelet    | 开始设置sanbox网络                               |                                                              |
| 11   | 10:35:19.467557 | kubelet    | 设置sanbox网络完成                               |                                                              |
| 12   | 10:35:19.468886 | kubelet    | 开始创建普通容器                                 |                                                              |
| 13   | 10:35:19.576669 | kubelet    | 普通容器创建成功                                 |                                                              |
| 14   | 10:35:19.801254 | kubelet    | kubelet(PLEG)获取pod状态之后去 更新pod状态       | PLEG是kubelet进程中的一个goroutine，每1s执行一次relist，所以运气不好的话可能会在pod创建成功后会需要等1s。（本次看起来只等了200多ms） |
| 15   | 10:35:19.811688 | kubelet    | kubelet（status manager）更新pod状态成功         |                                                              |



### 总结

* 整体
  * 本次实验整体花费**约1.2s**，但是这个受PLEG的relist影响，并非固定值。relist每次花费（0~1s）
* API Server
  * 在接收到create请求以后，到scheduler watch到该pod，共花费约100ms。占<10%
* scheduler
  * 调度绑定整个过程花费约10ms，占比极小。
* kubelet
  * 等待volume被attach和mount花费至少300ms。本次约占25%。
    * 本次没有volume，可见引用volume时这个过程会耗时更多
  * 创建sandbox过程花费<400ms。本次约占33%。
  * relist过程。本次等待230ms，约占19%；而经由status manager向API Server更新pod状态过程花费10ms。
    * 但等待时间占比是变化的，这里的19%没有参考意义，等待时间为0~1s



### 附录

#### 1、nginx.yaml

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-ivans
spec:
  replicas: 1 # tells deployment to run 2 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      # unlike pod-nginx.yaml, the name is not included in the meta data as a unique name is
      # generated from the deployment name
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

#### 2、步骤对应日志

| 步骤序号 | 日志                                                         |
| -------- | ------------------------------------------------------------ |
| 1        | PUT "/apis/extensions/v1beta1/namespaces/default/deployments/nginx-ivans/status"
satisfied by nonGoRestful |
| 2        | scheduler watch到新pod后尝试调度（10:35:18.744457）（About to try and schedule pod nginx-ivans-75675f5897-kvb74 |
| 3        | Finished binding for pod ee68eb93-67cb-11e9-b2f6-5254001a7461. Can be expired. |
| 4        | Receiving a new pod "nginx-ivans-75675f5897-kvb74_default(ee68eb93-67cb-11e9-b2f6-5254001a7461)" |
| 5        | Waiting for volumes to attach and mount for pod
"nginx-ivans-75675f5897-kvb74_default |
| 6        | All volumes are attached and mounted for pod
"nginx-ivans-75675f5897-kvb74_default |
| 7        | Creating sandbox for pod
"nginx-ivans-75675f5897-kvb74_default(ee68eb93-67cb-11e9-b2f6-5254001a7461)" |
| 8        |                                                              |
| 9        | Will attempt to re-write config file /var/lib/docker/containers/xxx/resolv.conf with: |
| 10       | Calling network plugin kubenet to set up pod "nginx-ivans-75675f5897-kvb74_default" |
| 11       | SetUpPod took 36.572139ms for default/nginx-ivans-75675f5897-kvb74 |
| 12       | Creating container &Container{Name:nginx,Image:nginx:1.7.9…  |
| 13       | Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-ivans-75675f5897-kvb74",
UID:"ee68eb93-67cb-11e9-b2f6-5254001a7461",
APIVersion:"v1", ResourceVersion:"118376",
FieldPath:"spec.containers{nginx}"}): type: 'Normal' reason:
'Created' Created container |
| 14       | PLEG: Write status for nginx-ivans-75675f5897-kvb74…         |
| 15       | Status for pod "nginx-ivans-75675f5897-kvb74_default(ee68eb93-67cb-11e9-b2f6-5254001a7461)"
updated successfully… |



