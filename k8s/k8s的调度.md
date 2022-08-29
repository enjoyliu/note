## k8s的调度

k8s作为云原生的编排工具，调度自然是其核心所在，在 k8s 中，*调度* 是指将 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 放置到合适的 [Node](https://kubernetes.io/zh/docs/concepts/architecture/nodes/) 上，然后由对应 Node 上的 [kubelet](https://kubernetes.io/docs/reference/generated/kubelet) 运行这些 pod。

k8s集群默认的调度器为[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)，同时在设计上k8s支持调度器的自定义。对每一个新创建的 Pod 或者是未被调度的 Pod，kube-scheduler 会选择一个最优的 Node 去运行这个 Pod。然而，Pod 内的每一个容器对资源都有不同的需求，而且 Pod 本身也有不同的资源需求。因此，Pod 在被调度到 Node 上之前， 根据这些特定的资源调度需求，需要对集群中的 Node 进行一次过滤。

在一个集群中，满足一个 Pod 调度请求的所有 Node 称之为 *可调度节点*。 如果没有任何一个 Node 能满足 Pod 的资源请求，那么这个 Pod 将一直停留在 未调度状态直到调度器能够找到合适的 Node。

调度器先在集群中找到一个 Pod 的所有可调度节点，然后根据一系列函数对这些可调度节点打分， 选出其中得分最高的 Node 来运行 Pod。之后，调度器将这个调度决定通知给 kube-apiserver，这个过程叫做 *绑定*。

### 调度器的触发

在了解具体的调度逻辑之前，首先介绍一下k8s的一个调度事件是如何产生和处理的。

![image-20210607191247355](https://github.com/weixiao619/pic/raw/main/image-20210607191247355.png)

如上图所示为pod的调度过程: 

1. reflector通过List的Watch方式监听k8s的API，从而获取事件
2. 将获取到的事件对象存入 Delta 的FIFO队列中
3. Informer从Delta队列中获取事件对象
4. Informer首先传递给Indexer
5. Indexer负责调用存储组件以k-v的形式存储下来，其中的key为该对象的`<namespace> / <name>`组合
6. 随后，Informer将事件发送到资源事件处理器，
7. 资源事件的Handler将事件放入工作队列workqueue中
8. 事件处理器获取到事件的key，并交给具体的Handle
9. Handle通过Indexer的引用，根据key拿到存储的事件对象，进行相应处理

#####  组件

在上述的调度过程中，提到了k8s的三个组件，下面是各个组件功能的简要介绍

1.`reflector`:用来watch特定的k8s API资源。具体的实现是通过`ListAndWatch`的方法，watch可以是k8s内建的资源或者是自定义的资源。当reflector通过watch API接收到有关新资源实例存在的通知时，它使用相应的列表API获取新创建的对象，并将其放入watchHandler函数内的Delta Fifo队列中。

2.`Informer`：informer从Delta Fifo队列中弹出对象。执行此操作的功能是processLoop。base controller的作用是保存对象以供以后检索，并调用我们的控制器将对象传递给它。

3.`Indexer`：索引器提供对象的索引功能。典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 在Store中定义了一个名为`MetaNamespaceKeyFunc`的默认函数，该函数生成对象的键作为该对象的`<namespace> / <name>`组合

- `Informer reference`：指的是Informer实例的引用，定义如何使用自定义资源对象。 自定义控制器代码需要创建对应的Informer。
- `Indexer reference`: 自定义控制器对Indexer实例的引用。自定义控制器需要创建对应的Indexser。

> client-go中提供`NewIndexerInformer`函数可以创建Informer 和 Indexer。

- `Resource Event Handlers`：资源事件回调函数，当它想要将对象传递给控制器时，它将被调用。 编写这些函数的典型模式是获取调度对象的key，并将该key排入工作队列以进行进一步处理。
- `Workqueue`：任务队列。 编写资源事件处理程序函数以提取传递的对象的key并将其添加到任务队列。
- `Process Item`：处理任务队列中对象的函数， 这些函数通常使用Indexer引用或Listing包装器来重试与该key对应的对象。

### 调度的逻辑

上述流程的最后触发的Handle即为调度的具体逻辑，调度的过程由**Kube-scheduler**组件来负责，k8s为了保证调度的效率和调度流程的简洁性，将调度大致分为了两个阶段，分别为**过滤**和**打分**。对应的过程如下：

![](https://github.com/weixiao619/pic/raw/main/640-20210607192255383.png)



过滤阶段将所有满足 Pod 调度需求的 Node 选出来。例如，PodFitsResources 过滤函数会检查候选 Node 的可用资源能否满足 Pod 的资源请求。在过滤之后，得出一个 Node 列表，里面包含了所有可调度节点；通常情况下，这个 Node 列表包含不止一个 Node。如果这个列表是空的，代表这个 Pod 不可调度；

打分阶段根据过滤阶段得到的Node列表，为每一个可调度节点进行打分。最后，**kube-scheduler** 会将 Pod 调度到得分最高的 Node 上。如果存在多个得分最高的 Node，kube-scheduler 会从中随机选取一个。

k8s将调度过程分为

下面分别对两个流程进行介绍

### 1.过滤

过滤过程中，会过滤出可调度的Node节点，过滤策略有如下几种：

#### 1.1. GeneralPredicates

负责最基础的调度策略，过滤宿主机的CPU和内存等资源信息

#### 1.2. Volume过滤规则

负责与容器持久化Volume相关的调度策略。例如：

检查多个 Pod 声明挂载的持久化 Volume 是否有冲突；

检查一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目；

检查Pod 对应的 PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配

#### 1.3. 检查调度Pod是否满足Node本身的条件

如PodToleratesNodeTaints负责检查的就是我们前面经常用到的 Node 的“污点”机制。NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足。

#### 1.4. affinity and anti-affinity

根据pod的node之间的亲和性与反亲和性进行过滤

### 2. 打分

```go
//筛选函数
// scheduleOne does the entire scheduling workflow for a single pod.
scheduleOne()

// 根据pod中的描述选择对应的调度器

// 1. 超时打日志


// frameworkImpl is the component responsible for initializing and running scheduler
// plugins.
type frameworkImpl struct {
	registry             Registry
	snapshotSharedLister framework.SharedLister
	waitingPods          *waitingPodsMap
	scorePluginWeight    map[string]int
	queueSortPlugins     []framework.QueueSortPlugin
	preFilterPlugins     []framework.PreFilterPlugin
	filterPlugins        []framework.FilterPlugin
	postFilterPlugins    []framework.PostFilterPlugin
	preScorePlugins      []framework.PreScorePlugin
	scorePlugins         []framework.ScorePlugin
	reservePlugins       []framework.ReservePlugin
	preBindPlugins       []framework.PreBindPlugin
	bindPlugins          []framework.BindPlugin
	postBindPlugins      []framework.PostBindPlugin
	permitPlugins        []framework.PermitPlugin

	clientSet       clientset.Interface
	kubeConfig      *restclient.Config
	eventRecorder   events.EventRecorder
	informerFactory informers.SharedInformerFactory

	metricsRecorder *metricsRecorder
	profileName     string

	extenders []framework.Extender
	framework.PodNominator

	parallelizer parallelize.Parallelizer

	// Indicates that RunFilterPlugins should accumulate all failed statuses and not return
	// after the first failure.
	runAllFilters bool
}


// 2. 预选策略选出满足调度条件的Node节点
findNodesThatFit()
// Run "prefilter" plugins.
	s := fwk.RunPreFilterPlugins(ctx, state, pod)

	preFilter依次执行 frameworkImpl中定义的preFilterPlugins中的所有插件的PreFilter实现

// "NominatedNodeName" can potentially be set in a previous scheduling cycle as a result of preemption.
	// This node is likely the only candidate that will fit the pod, and hence we try it first before iterating over all nodes.
feasibleNodes, err := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)


// 用不同的过滤插件{NodePort, Volume, Label等}进行过滤
runFilterPlugin()
// 1 运行过滤器，并考虑pod的提名情况
func (f *frameworkImpl) RunFilterPluginsWithNominatedPods(ctx context.Context, state *framework.CycleState, pod *v1.Pod, info *framework.NodeInfo) *framework.Status {
	var status *framework.Status

	podsAdded := false
  // 可以看到在某些情况下过滤插件执行了两次： 如果该node中有优先级大于等于当前pod的已提名pod，就把这些pod考虑进nodeInfo中并再执行一次预过滤，所有过滤器全部通过后，再执行一次不考虑已提名pod的过滤，如果第一次过滤失败或者该节点没有已提名的pod,则跳过第二次过滤。
  // 在第一次过滤中只考虑优先级大于等于当前pod的情况是因为，优先级低于他的pod所占据的资源可被抢占，
  // 为了确保新pod在两种情况下的均可被调度，这里采用了一种相对保守的调度策略，将被提名的pod视为运行中，pod间的反亲和性设置更可能失败，但是不将被提名的pod视为运行中，pod间的亲和性设置更容易失败。我们不能直接假设被提名的pod处于运行中，因为事实上他们有可能被调度到其他节点上。
	for i := 0; i < 2; i++ {
		stateToUse := state
		nodeInfoToUse := info
		if i == 0 {
			var err error
			podsAdded, stateToUse, nodeInfoToUse, err = addNominatedPods(ctx, f, pod, state, info)
			if err != nil {
				return framework.AsStatus(err)
			}
		} else if !podsAdded || !status.IsSuccess() {
			break
		}

		statusMap := f.RunFilterPlugins(ctx, stateToUse, pod, nodeInfoToUse)
		status = statusMap.Merge()
		if !status.IsSuccess() && !status.IsUnschedulable() {
			return status
		}
	}

	return status
}



// 3. 对节点进行打分
prioritizeNodes()



// 运行预打分插件
// InterPodAffinity, NodeAffinity, PodTopologySpread, 
RunPreScorePlugins()

// 运行打分插件
RunScorePlugins()


```

![image-20210602215451585](https://github.com/weixiao619/pic/raw/main/image-20210602215451585.png)

### 代码逻辑

![img](https://github.com/weixiao619/pic/raw/main/1599387171_7_w1514_h1290.png)

1. 通过sched.NextPod()函数从优先队列中获取一个优先级最高的待调度Pod资源对象，如果没有获取到，那么该方法会阻塞住；
2. 通过sched.Algorithm.Schedule调度函数执行Predicates的调度算法与Priorities算法，挑选出一个合适的节点；
3. 当没有找到合适的节点时，调度器会尝试调用prof.RunPostFilterPlugins抢占低优先级的Pod资源对象的节点；
4. 当调度器为Pod资源对象选择了一个合适的节点时，通过sched.bind函数将合适的节点与Pod资源对象绑定在一起；



```
kubectl -n 20190905164919672-9b8b-6b2e04122 get pods | grep Evicted | awk '{print $1}' | xargs kubectl -n 20190905164919672-9b8b-6b2e04122 delete pod
```

