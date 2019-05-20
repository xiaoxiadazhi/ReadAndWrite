## Kubelet源码走读（一）—— SyncLoop

---

### kubelet功能简介

```
kubelet是k8s中很重要的一个组件，每个节点上都会运行一个kubelet服务进程。kubelet内部组件繁多，功能复杂，但最主要的功能还是管理pod及pod中的容器。今天我们先了解下syncLoop。我们可以先从下图开始，对kubelet整体结构有个概览。
```

![1-1](F:\git\ReadAndWrite\learning\k8s&docker\源码\kubelet\picture\1-1.png)

稍作总结，kubelet大致有以下功能：

* pod管理
* 节点管理
* 容器健康检查
* cadvisor资源监控
* kubelet eviction

### 带着问题看代码

```
	对于看源码，尤其是kubelet这样功能复杂的组件源码，最好是从某个点突破，带着问题出发，先看清主脉络，再细化各功能。当然，首先你起码得了解过kubelet的基本功能，并使用过k8s，要不然什么都不懂的前提下，看源码没有什么意义。以我个人为例，由于需要分析创建pod过程的耗时情况，所以我从日志开始，一步步理清kubelet主脉络。
	当我们创建一个pod的时候，我们肯定想知道：kubelet如何根据我们的create请求来启动一个pod。所以，首先一个问题就是kubelet如何获取pod变化。然后是创建出来的pod到底是什么，和容器的关系是什么？最后pod创建成功，又是如何更新pod状态的？
```

一开始问题肯定是比较宏观的，在看代码过程中会一步步细化。这里先记录下我们的问题：

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

### 正文

#### 1、kubelet如何获取pod变化？

我的日志文件中搜索到相应pod名字的第一条日志是

```
337 Apr 02 15:30:35 VM-16-7-ubuntu kubelet[6436]: I0402 15:30:35.063832    6436 config.go:404] Receiving a new pod "nginx-deployment-ivan-569477d6d8-m2xrm_default(343e9a5b-5519-11e9-beb7-4e66accebffa)"
```

所以，kubelet创建pod的所有动作都是从此开始的。我们在源码中很轻易就找到了相应的地方：

![1-2](F:\git\ReadAndWrite\learning\k8s&docker\源码\kubelet\picture\1-2.png)

根据一级级的调用关系往上看（调用链是Merge->merge->recordFirstSeenTime），可以看到/pkg/kubelet/config/config.go中的Merge函数：

![1-3](..\picture\1-3.png)

看注释可以了解到：

* pod的变化有多个来源
* 最终会将变化push到update channel中

稍微浏览下代码即可发现：merge()将入参的change解析分类，然后push到updates中。在此，我们重点关注2点：

1. 入参的change从哪里来？
   - 这能回答我们本节的标题：kubelet如何获取pod变化？
2. 这个update channel在哪里被使用？
   - 这个可以知道数据流向了那里

对于1，我们看看那里调用了Merge()。一直追溯下去我们可以得到这样一个调用链：

![1-4](..\picture\1-4.png)

我们找一下Merge的入参update的来源。

![1-6](..\picture\1-6.png)

从上图可以看到update来自于listen()的listenChannel，而这里对listenChannel进行range，会一直等待channel的动作，直到自动关闭。而Channel()的逻辑是：

* 第一次创建一个channel给listen()，并返回这个channel
* 返回channel给上一级函数，那么在上一级函数中肯定会往chan中写数据

再看上一级函数cfg.Channel，也返回了这个channel，那就继续往上看。

`makePodSourceConfig`中多处调用了cfg.Channel，可以发现传入了三类source，也就是表示pod的变更来自于这三类source

* file
* http
* api

file表示来自于定义的staticPod的文件路径，http表示获取staticPod文件的url。（比如使用static pod定义master的几个组件的话，有变动就是通过这两种方式获取pod变化）。api表示pod变化来自于apiserver。我们只看下这里面关于apiserver这类源的代码：

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

可以发现实际上创建了一个listwatch去watch所有namespace的、绑定到本node上的pod。而由第4行我们也能知道updates这个channel是谁往里面写了数据。到这里我们基本上可以回答标题的问题了，总结一下：

- ***kubelet通过list-watch机制去watch了三类源（api/file/http），看本node上的pod是否有变化，有则获取***

那获取到的pod变化最终给谁处理了呢？可以看一下`makePodSourceConfig`的返回值赋值给了`kubeDeps.PodConfig`：


![1-5](..\picture\1-5.png)

而`kubeDeps.PodConfig`在`RunKubelet`中被传给了`startKubelet`，然后一路被传给了pkg/kubelet/kubelet.go路径下的`syncLoop`了。我们接下来详细分析下这个过程以及updates如何被处理。

#### 2、pod如何被创建？（SyncLoop）

接续上节结尾，我们循着podConfig的数据流向，最终将podConfig中的update传递到了`syncLoop`中

![1-7](..\picture\1-7.png)

`syncloop`是处理pod变化的主loop，本节我们详细探索创建pod的过程。

![1-8](..\picture\1-8.png)

我们可以看到`syncLoop`主要在运行`syncLoopIteration`，并且和pleg组件有交互。`syncLoopIteration`代码逻辑比较简单，就是select入参的几个channel，对每个channel的数据进行相应的处理，我们主要看configCh，因为前面已经说明了kubelet watch到的pod变化会写入该channel。

![1-9](..\picture\1-9.png)

我们看下`syncLoopIteration`中的`HandlePodAdditions`的逻辑，就是对新建pod的变化进行处理。

![1-10](..\picture\1-10.png)