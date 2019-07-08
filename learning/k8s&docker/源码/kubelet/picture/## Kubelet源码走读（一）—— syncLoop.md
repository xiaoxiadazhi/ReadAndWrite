## Kubelet源码走读（一）—— Pod如何被创建

---

### kubelet功能简介

```
kubelet是k8s中很重要的一个组件，每个节点上都会运行一个kubelet服务进程。kubelet内部组件繁多，功能复杂，但最主要的功能还是管理pod及pod中的容器。今天我们就先了解下pod的创建流程。
下图是kubelet的整体结构有个概览。pod的创建流程也会涉及到图中多个组件。
```

![1-1](F:\git\ReadAndWrite\learning\k8s&docker\源码\kubelet\picture\1-1.png)

简单列举一下kubelet的主要功能：

* pod管理
* 节点管理
* 容器健康检查
* cadvisor资源监控
* kubelet eviction

### 带着问题看代码

```
	对于看源码，尤其是kubelet这样功能复杂的组件源码，最好是从某个点突破，带着问题出发，先看清主脉络，再细化各功能。当然，首先你起码得了解过kubelet的基本功能，并使用过k8s，要不然什么都不懂的前提下，看源码没有什么意义。以我个人为例，由于需要分析创建pod过程的耗时情况，所以我从日志开始，一步步理清kubelet主脉络。
	当我们创建一个pod的时候，我们肯定想知道kubelet是如何根据我们的`kubectl create -f xxx.yaml`请求来创建、启动一个pod。
	我们知道kubernetes中组件的交互都是通过apiserver完成的，所以，首先一个问题就是kubelet如何从apiserver获取pod变化。然后是创建出来的pod到底是什么，和容器的关系是什么？最后pod创建成功，又是如何更新pod状态的？
```

一开始问题肯定是比较宏观的，在看代码过程中会进一步细化。这里先带着我们的问题去 看源码：

1. kubelet如何根据我们的create请求来启动一个pod？
2. kubelet如何获取pod变化？
3. 创建出来的pod到底是什么，和容器是什么关系？
4. pod创建成功，又是如何更新pod状态的？

#### 如何利用日志走读kubelet源码

```
这里简述下我是如何通过kubelet日志来看代码的。
- 首先将kubelet日志级别设置为`--v=5`来重启。一般来说级别5已经能打印出绝大部分细节了。
- 然后创建一个容易识别的pod（deployment或pod名字写特殊点）
- 使用`journalctl -u kubelet --since "5 min ago" > kubelet.log`将创建pod的整个过程的日志重定向到文件
- 在kubelet.log中搜索deployment或pod名字的关键字，可以扫一眼日志的意思。找出第一次出现的地方，在源码中找到相应的日志，一级一级看，了解之前之后的大概行为。
```

### 走读源码

#### 1、kubelet如何获取pod变化？

在我的日志文件中根据pod名字搜索到的第一条日志如下：

```
337 Apr 02 15:30:35 VM-16-7-ubuntu kubelet[6436]: I0402 15:30:35.063832    6436 config.go:404] Receiving a new pod "nginx-deployment-ivan-569477d6d8-m2xrm_default(343e9a5b-5519-11e9-beb7-4e66accebffa)"
```

我们根据它找到源码，从方法的注释和日志可见，kubelet创建pod的所有动作应该是从此开始的：

![1-2](F:\git\ReadAndWrite\learning\k8s&docker\源码\kubelet\picture\1-2.png)

找到这以后，我们看看是谁调用的它。我们先根据一级级的调用关系往上看（调用链是Merge->merge->recordFirstSeenTime），可以看到/pkg/kubelet/config/config.go中的Merge函数：

![1-3](..\picture\1-3.png)

看注释可以了解到：

* **pod的变化有多个来源**
* **最终会将变化push到update channel中**

稍微浏览下代码即可发现：merge()将入参的change解析分类，然后又push到podStorage的updates中。在此，我们重点关注2点：

1. 入参的change从哪里来？
   - 这能回答我们本节的标题：kubelet如何获取pod变化？
2. 这个update channel在哪里被消费了？
   - 也就是podStorage.updates在哪里被消费了？回答了这个问题，就可以知道数据流向了哪里

对于1，我们看看哪里调用了Merge()。一直追溯下去我们可以得到这样一个调用链：

![1-4](..\picture\1-4.png)

我们找一下Merge的入参update的来源。

![1-6](..\picture\1-6.png)

从上图可以看到update来自于listen()的listenChannel，而这里对listenChannel进行range，会一直等待channel的动作，直到自动关闭。也就是说这里只有当listenChannel有数据时才会去执行Merge()，所以只能往上级调用函数看，看哪里往listenChannel中写数据了。

另外Channel()的逻辑是：第一次时，创建一个channel给listen()，并return这个channel

返回channel给上一级函数，那么在上一级函数中应该会往channel中写数据，那就往上级函数看。上一级函数是cfg.Channel，也返回了这个channel，继续往上看。

`makePodSourceConfig`中多处调用了cfg.Channel，可以发现传入了三类source（file、http、api），也就是表示pod的变更来自于这三类source。这里我们标记一下我们了解到的一个知识点：***kubelet接收到的pod的变更来自于三类source***

```
kubelet接收到的pod的变更来自于三类source:
1. api	——	来自于apiserver
2. file	——	来自于StaticPod的文件（比如，TKE中独立集群通过static pod定义了master的组件（apiser、controller-manager、scheduler））
3. http	——	来自于获取staticPod文件的url
```

我们只看apiserver这类源相关的代码就好了，道理都是一样的。

```go
// makePodSourceConfig creates a config.PodConfig from the given
// KubeletConfiguration or returns an error.
func makePodSourceConfig(kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *Dependencies, nodeName types.NodeName, bootstrapCheckpointPath string) (*config.PodConfig, error) {
	manifestURLHeader := make(http.Header)
	...
	...
	if kubeDeps.KubeClient != nil {
		glog.Infof("Watching apiserver")
		if updatechannel == nil {
			updatechannel = cfg.Channel(kubetypes.ApiserverSource)
		}
		config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, updatechannel)
	}
	return cfg, nil
}
```

第12行`NewSourceApiserver`的代码如下

```
// NewSourceApiserver creates a config source that watches and pulls from the apiserver.
func NewSourceApiserver(c clientset.Interface, nodeName types.NodeName, updates chan<- interface{}) {
	lw := cache.NewListWatchFromClient(c.CoreV1().RESTClient(), "pods", metav1.NamespaceAll, fields.OneTermEqualSelector(api.PodHostField, string(nodeName)))
	newSourceApiserverFromLW(lw, updates)
}
```

可以发现实际上kubelet创建了一个listwatch去watch所有namespace的、绑定到本node上的pod。而我们详看`newSourceApiserverFromLW`的代码也能知道是谁往updates这个channel中写了数据。到这里我们基本上可以回答***“kubelet如何获取pod变化”***的问题了。总结一下：

- ***kubelet通过list-watch机制去watch了三类源（api/file/http），看本node上的pod是否有变化，有则获取，并写入updatechannel***



#### 2、获取的pod变化被谁消费了？

那获取到的pod变化最终给谁处理了呢，或者说谁是updatechannel的消费者呢？还记得前面说的listenChannel有数据以后才会执行Merge()吗？Merge()的逻辑前面也说了将pod的变化push到了update channel（podStorage.updates）中。

既然`makePodSourceConfig`函数往下的调用链最终是把pod变化push到了`podStorage.updates`中，那我们只能看看`makePodSourceConfig`函数的返回及上级调用了。我们可以看到`makePodSourceConfig`函数的数据结构中，有podStorage、updates、sources。结合`makePodSourceConfig`中的`NewPodConfig`实现可以知道podStorage就是listen()函数中的`m.merger`：

```go
// PodConfig is a configuration mux that merges many sources of pod configuration into a single
// consistent structure, and then delivers incremental change notifications to listeners
// in order.
type PodConfig struct {
   pods *podStorage
   mux  *config.Mux

   // the channel of denormalized changes passed to listeners
   updates chan kubetypes.PodUpdate

   // contains the list of all configured sources
   sourcesLock       sync.Mutex
   sources           sets.String
   checkpointManager checkpoint.Manager
}
```

而`makePodSourceConfig`的返回值赋值给了`kubeDeps.PodConfig`，我们只要找到谁消费了`kubeDeps.PodConfig`就能知道pod的变化最终在哪里消费了。


![1-5](..\picture\1-5.png)

根据`kubeDeps.PodConfig`的调用关系（使用IDE查看谁调用了PodConfig），发现`kubeDeps.PodConfig`是在`RunKubelet`中被传给了`startKubelet`，然后一路被传到pkg/kubelet/kubelet.go路径下的`syncLoop`了。调用链如下图

![1-7](..\picture\1-7.png)

这里的syncLoop***就是文章开头的kubelet整体组件图中的`syncLoop`***。我们先看一下`syncLoop`的注释：

```go
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	...
	...
}
```

注释说**syncLoop是处理pod变化的主loop，一旦观察变换就会同步期望状态和运行状态。所以，我们最终找到了syncLoop就是pod变化的消费者**。

那syncLoop具体干了什么呢？我们接下来详细看看syncLoop的代码。

#### 3、syncLoop做了什么？

直接看源码：

![1-8](..\picture\1-8.png)

我们可以看到`syncLoop`有个for循环，主要运行`syncLoopIteration`，并且和pleg组件有交互。`syncLoopIteration`代码逻辑比较简单，就是select入参的几个channel，对每个channel的数据进行相应的处理，我们先看configCh，它其实就是上一小节所说的PodConfig.updates，包含了kubelet watch到的pod变化。

![1-9](..\picture\1-9.png)

我们看下`syncLoopIteration`中调用的`HandlePodAdditions`的逻辑：

- 先对要add的pods按创建时间排序
- 然后遍历pods
  1. 先把pod写入podManager（如果podManager中没有某个pod，就意味着这个pod已经在apiserver中被删除了，并且除了cleanup不再会做别的操作）
  2. 处理MirrorPod（为了监控static pod的状态，kubelet通过api server为每个static pod创建一个mirror pod）
  3. 检查是否admit pod
  4. 将pod分发到某个worker去进行sync操作
  5. 将pod传给probeManager做健康检查

![1-10](..\picture\1-10.png)

我们主要看第4步：dispatchwork()做了什么。我们按照dispatchWork --> `podWorkers.UpdatePod` --> `podWorkers.managePodLoop`的代码链路一路追踪下去，发现最终调用了  podWorkers的`syncPodFn`，而`syncPodFn`是在`NewMainKubelet`对podWorkers初始化时赋值的，赋值为`klet.syncPod`，所以真正做同步工作的是`syncPod`.

```go
...
klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)
...
```

我们看看kubelet的`syncPod`的注释（代码太长的话，看注释有助于理解）：

![1-11](..\picture\1-11.png)

`syncPod`是同步单个pod的事务脚本。主要工作流是：

* 如果要创建pod，记录pod worker的启动延时
* 调用generateAPIPodStatus为pod准备一个v1.PodStatus对象，用来保存pod状态，并会写到statusManager，回写API server
* 如果pod被视为第一次running，记录pod启动延迟
* 在status manager中更新pod的状态
* kill掉不该是running的pod
* 如果pod是static pod，且没有mirror pod，创建一个mirror pod
* 如果不存在，则为pod创建数据目录
* 等待volume被attach/mount
* 获取pod的pull secrets
* 调用容器运行时的`SyncPod`回调
* 更新reasonCache（缓存的是所有容器最近创建失败的原因，用于产生容器状态）

上面的注释中比较重要的是：

1. syncPod会通过status Manager去回写apiserver pod的状态
2. 会等待volume被attach/mount之后再继续执行
3. 调用的容器运行时的`SyncPod`

我们直接看第3点。点开`SyncPod`又是一个很长的函数，看看注释。

![1-12](..\picture\1-12.png)

注释写的很清晰了，`SyncPod`就是执行了这6步去把正在运行的pod同步成期望的pod。

1. 计算sandbox和containar的变化
   - 解释：检测pod spec是否发生变化，如果发生变化返回changes
2. 如有必要，kill pod sanbox
3. kill掉所有不该是running的容器
4. 如有必要创建sandbox
5. 创建init containers
6. 创建普通containers

`SyncPod`创建了sandbox、init容器、和业务容器。其实一个pod就是由sandbox和容器组成，而容器包括initContainer（做一些事先配置的工作）和普通container（业务容器），另外，这里的sandbox其实也是个容器。为了了解pod sandbox，我们可以追踪一下`SyncPod`中的`createPodSandbox`，可以发现经过cri调用了dockershim中的`RunPodSandbox`，调用链如下：

![1-13](..\picture\1-13.png)

我们再看看dockershim中`RunPodSandbox`的代码，代码有点长，不过看注释很清晰。`RunPodSandbox`创建和启动了一个pod-level sandbox，对于docker来说，PodSandbox是由一个拥有pod的网络namespace的容器实现的。过程有以下几步：

1. 拉取sandbox镜像。实际上是pause镜像
2. 创建sandbox容器。
3. 创建sandbox checkpoint
4. 启动sandbox容器。
5. 位sandbox设置网络。
   - 在kubelet启动时会通过CNI插件来设置pod的网络。这个插件会为sandbox中的pod分配ip、设置路由等

```go
// RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
// the sandbox is in ready state.
// For docker, PodSandbox is implemented by a container holding the network
// namespace for the pod.
// Note: docker doesn't use LogDirectory (yet).
func (ds *dockerService) RunPodSandbox(ctx context.Context, r *runtimeapi.RunPodSandboxRequest) (*runtimeapi.RunPodSandboxResponse, error) {
	config := r.GetConfig()

	// Step 1: Pull the image for the sandbox.
	image := defaultSandboxImage
	podSandboxImage := ds.podSandboxImage
	if len(podSandboxImage) != 0 {
		image = podSandboxImage
	}

	// NOTE: To use a custom sandbox image in a private repository, users need to configure the nodes with credentials properly.
	// see: http://kubernetes.io/docs/user-guide/images/#configuring-nodes-to-authenticate-to-a-private-repository
	// Only pull sandbox image when it's not present - v1.PullIfNotPresent.
	if err := ensureSandboxImageExists(ds.client, image); err != nil {
		return nil, err
	}

	// Step 2: Create the sandbox container.
	createConfig, err := ds.makeSandboxDockerConfig(config, image)
	if err != nil {
		return nil, fmt.Errorf("failed to make sandbox docker config for pod %q: %v", config.Metadata.Name, err)
	}
	createResp, err := ds.client.CreateContainer(*createConfig)
	if err != nil {
		createResp, err = recoverFromCreationConflictIfNeeded(ds.client, *createConfig, err)
	}

	if err != nil || createResp == nil {
		return nil, fmt.Errorf("failed to create a sandbox for pod %q: %v", config.Metadata.Name, err)
	}
	resp := &runtimeapi.RunPodSandboxResponse{PodSandboxId: createResp.ID}

	ds.setNetworkReady(createResp.ID, false)
	defer func(e *error) {
		// Set networking ready depending on the error return of
		// the parent function
		if *e == nil {
			ds.setNetworkReady(createResp.ID, true)
		}
	}(&err)

	// Step 3: Create Sandbox Checkpoint.
	if err = ds.checkpointHandler.CreateCheckpoint(createResp.ID, constructPodSandboxCheckpoint(config)); err != nil {
		return nil, err
	}

	// Step 4: Start the sandbox container.
	// Assume kubelet's garbage collector would remove the sandbox later, if
	// startContainer failed.
	err = ds.client.StartContainer(createResp.ID)
	if err != nil {
		return nil, fmt.Errorf("failed to start sandbox container for pod %q: %v", config.Metadata.Name, err)
	}

	// Rewrite resolv.conf file generated by docker.
	// NOTE: cluster dns settings aren't passed anymore to docker api in all cases,
	// not only for pods with host network: the resolver conf will be overwritten
	// after sandbox creation to override docker's behaviour. This resolv.conf
	// file is shared by all containers of the same pod, and needs to be modified
	// only once per pod.
	if dnsConfig := config.GetDnsConfig(); dnsConfig != nil {
		containerInfo, err := ds.client.InspectContainer(createResp.ID)
		if err != nil {
			return nil, fmt.Errorf("failed to inspect sandbox container for pod %q: %v", config.Metadata.Name, err)
		}

		if err := rewriteResolvFile(containerInfo.ResolvConfPath, dnsConfig.Servers, dnsConfig.Searches, dnsConfig.Options); err != nil {
			return nil, fmt.Errorf("rewrite resolv.conf failed for pod %q: %v", config.Metadata.Name, err)
		}
	}

	// Do not invoke network plugins if in hostNetwork mode.
	if config.GetLinux().GetSecurityContext().GetNamespaceOptions().GetNetwork() == runtimeapi.NamespaceMode_NODE {
		return resp, nil
	}

	// Step 5: Setup networking for the sandbox.
	// All pod networking is setup by a CNI plugin discovered at startup time.
	// This plugin assigns the pod ip, sets up routes inside the sandbox,
	// creates interfaces etc. In theory, its jurisdiction ends with pod
	// sandbox networking, but it might insert iptables rules or open ports
	// on the host as well, to satisfy parts of the pod spec that aren't
	// recognized by the CNI standard yet.
	cID := kubecontainer.BuildContainerID(runtimeName, createResp.ID)
	err = ds.network.SetUpPod(config.GetMetadata().Namespace, config.GetMetadata().Name, cID, config.Annotations)
	if err != nil {
		// TODO(random-liu): Do we need to teardown network here?
		if err := ds.client.StopContainer(createResp.ID, defaultSandboxGracePeriod); err != nil {
			glog.Warningf("Failed to stop sandbox container %q for pod %q: %v", createResp.ID, config.Metadata.Name, err)
		}
	}
	return resp, err
}
```

看了上面注释，我们可以追溯下创建和启动sandbox容器的代码，发现是往docker发了create和start请求。而创建的是pause镜像，这就可以解释为什么我们创建一个pod以后，通过docker ps可以看到每个pod都有个pause镜像了。这个pause镜像有两个功能：

* 是pod里其他容器共享Linux namespace的基础
* 扮演PID 1的角色，负责处理僵尸进程

现在我们总结一下：

* ***syncLoop主要就是将pod同步成期望状态。***
* ***另外通过grpc与dockershim通信，让dockershim向docker发送创建删除容器的请求，并通过CNI去配置pod网络***
* ***创建出来的pod实际上就是pause容器加上用户自己的容器（如init容器、业务容器）***

到这里`SyncLoop`就完成它的一次循环的工作了，当然每次循环的处理动作要看收到的数据。那么pod创建成功后，我们通过`kubectl get pods`看到的状态变为running了，这是谁更新到apiserver的呢?我们继续分析。

#### 4、kubelet如何给apiserver回写状态

其实我们在上一小节的分析中提到了kubelet.syncPod中会往statusManager中更新pod状态，但是这个步骤在创建容器之前，那创建容器完成后，是谁去获取的状态传进来的呢？亦或是有其他方式更新？

这其实是kubelet中PLEG这个组件去干的事。

![1-8](..\picture\1-8.png)

我们再看一下这张图，pleg watch到数据传入了`syncLoopIteration`。pleg是用来在pod生命周期中生成事件的，它周期性地去监听容器状态。详情可参考<https://github.com/jenkins-x/exposecontroller/blob/master/vendor/k8s.io/kubernetes/docs/proposals/pod-lifecycle-event-generator.md>

我们继续看代码，进入kl.pleg.Watch()，发现返回了eventChannel。

```go
// Returns a channel from which the subscriber can receive PodLifecycleEvent
// events.
// TODO: support multiple subscribers.
func (g *GenericPLEG) Watch() chan *PodLifecycleEvent {
	return g.eventChannel
}
```

那哪里给eventChannel生产数据了呢？我们根据调用关系可以找到是`relist`中第82行：

```go
// relist queries the container runtime for list of pods/containers, compare
// with the internal pods/containers, and generates events accordingly.
func (g *GenericPLEG) relist() {
	glog.V(5).Infof("GenericPLEG: Relisting")

	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInMicroseconds(lastRelistTime))
	}

	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistLatency.Observe(metrics.SinceInMicroseconds(timestamp))
	}()

	// Get all the pods.
	podList, err := g.runtime.GetPods(true)
	if err != nil {
		glog.Errorf("GenericPLEG: Unable to retrieve pods: %v", err)
		return
	}

	g.updateRelistTime(timestamp)

	pods := kubecontainer.Pods(podList)
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err := g.updateCache(pod, pid); err != nil {
				glog.Errorf("PLEG: Ignoring events for pod %s/%s: %v", pod.Name, pod.Namespace, err)

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else if _, found := g.podsToReinspect[pid]; found {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
			}
		}
		// Update the internal storage and send out the events.
		g.podRecords.update(pid)
		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			g.eventChannel <- events[i]
		}
	}
...
}
```

relist是pleg的主要函数，主要功能就是被周期性调用去获取所有pod，比较新旧pod，生成事件。数据流向是`computeEvents() -> updateEvents() -> eventsByPodID -> g.eventChannel`。我们看`computeEvents()`的源码可以看到有调用`generateEvents`，我么直接看`generateEvents`：

```go
func generateEvents(podID types.UID, cid string, oldState, newState plegContainerState) []*PodLifecycleEvent {
	if newState == oldState {
		return nil
	}

	glog.V(4).Infof("GenericPLEG: %v/%v: %v -> %v", podID, cid, oldState, newState)
	switch newState {
	case plegContainerRunning:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerStarted, Data: cid}}
	case plegContainerExited:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}}
	case plegContainerUnknown:
		return []*PodLifecycleEvent{{ID: podID, Type: ContainerChanged, Data: cid}}
	case plegContainerNonExistent:
		switch oldState {
		case plegContainerExited:
			// We already reported that the container died before.
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerRemoved, Data: cid}}
		default:
			return []*PodLifecycleEvent{{ID: podID, Type: ContainerDied, Data: cid}, {ID: podID, Type: ContainerRemoved, Data: cid}}
		}
	default:
		panic(fmt.Sprintf("unrecognized container state: %v", newState))
	}
}
```

发现不同状态会有不同类型的事件。这下就明白了：`syncLoopIteration`中回去watch pleg的eventchannel，而pleg周期性地发现pod（或container）的变化，生成事件，写入eventchannel中。这样`syncLoopIteration`在select到数据以后会进行相应的处理：

![1-14](..\picture\1-14.png)

之后0同上一节说的步骤一路到kubelet的`syncPod`，将pod状态（假如是第一次创建到running，则为ContainerStarted）更新到statusManger。一路追踪代码发现数据写入了statusManager中的`podStatusChannel`,而statusManager在启动时就会select这个channel，并在statusManager的syncPod中去调用kubeClient去更新apiserver中pod的状态：

![1-15](..\picture\1-15.png)

总结一下，kubelet回写apiserver的过程：

1. pleg周期性检测所有pod的状态，并比较新旧状态，生成相应的事件
2. syncLoop获得相应的事件后，将状态同步到statusManager
3. statusManager获取新状态就回写apiserver

### 总结

讲到这里我们已经讲述了从我们执行`kubectl create -f pod.yaml`以后，整个kubelet如何去创建的pod的流程。我们再总结一下：

1. kubelet通过list-watch机制去watch了三类源（api/file/http），看本node上的pod是否有变化，有则获取，并写入updatechannel
2. syncLoop在select到updatechannel中的数据后，会通过grpc调用dockershim去发送请求给docker创建容器，dockershim还会调用CNI为pod sandbox设置网络
3. pleg会周期性检测所有pod的状态，并比较新旧状态，生成相应的事件
4. syncLoop获得相应的事件后，将状态同步到statusManager
5. statusManager获取新状态就回写apiserver

以下kubelet的一个细节的结构图，虽然没有完成，但是放在这里让大家可以容易理解Pod被创建过程中，kubelet中各组件之间的交互。

![1-16](..\picture\1-16.png)

另外，我们通过源码可以知道pod sandbox实际上就是pause容器。而一个pod其实就是这个pause容器再加上业务容器。

当然还有一些细节，比如syncPod中会有等到volume被attach和mount的步骤，这个等待时间略长，可能是我们的需求（优化pod启动时间）的一个优化点。但通过什么方式优化，还需要评估。