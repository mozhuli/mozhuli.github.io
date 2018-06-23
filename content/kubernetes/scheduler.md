---
title: "kube-schdeuler 1.10解析"
date: 2018-06-23T15:09:30+08:00
draft: false
---

# Kube-scheduler解析

&emsp;&emsp;Kube-scheduler具有丰富的调度策略，能够感知集群的拓扑，同时能够适应于特定的工作负载等功能，Kube-scheduler可显着影响集群的可用性，性能和容量。 Kube-scheduler需要考虑个人和集群的资源需求，服务质量要求，硬件/软件/政策约束，亲和力和反亲和力要求，数据位置，工作负载间的干扰，工作负载到期日期等。 必要时，特定工作负载对调度的要求将通过API公开。

## kube-scheduler原理

&emsp;&emsp;kube-scheduler一般以pod的形式部署在 master 节点上，它会 watch kube-apiserver 进程去发现 PodSpec.NodeName 为空的 Pod，然后根据指定的算法（1.默认的DefaultProvider，2.也可以通过配置文件policy配置, 3.也可以是自定义调度器）将 Pod 调度到合适的 Node 上进行绑定(Bind)。scheduler 的输入就是需要被调度的 Pod 和 Node 的信息，输出是经过调度算法选出条件最优的 Node，并将该 Pod 绑定到这个 Node 上。

&emsp;&emsp;调度的过程就像是一个漏斗，根据要调度pod的一些信息，以及其他k8s的信息，如PV，service，rc，rs等，从所有的node中一步步筛选出最优的node并进行调度（Bind）。

![scheduler主要逻辑](/images/process.png)

调度过程主要分为两个阶段：

- **Predicates（筛选阶段）**

   ​&emsp;&emsp;从所有的node中筛选出一部分符合要求的node（也就是过滤掉不符合的node）。此阶段包含一系列的筛选算法，如PodFitsHost：节点是否满足pod的spec node name；PodFitsHostPorts：插件节点的port对于pod请求的port是否可用；PodMatchNodeSelector：node的label是否满足pod的node selector等等

   ​&emsp;&emsp;此阶段的原则是提前发现不满足要求的node，过滤掉，从而提高调度的效率。所以就有了各个筛选算法的优先级，默认优先级(原则是**过滤面越大的算法优先级越高**)如下：

   ```yaml
   []string{
     CheckNodeConditionPred,
     GeneralPred,
     HostNamePred,
     PodFitsHostPortsPred,
     MatchNodeSelectorPred,
     PodFitsResourcesPred,
     NoDiskConflictPred,
     PodToleratesNodeTaintsPred,
     PodToleratesNodeNoExecuteTaintsPred,
     CheckNodeLabelPresencePred,
     checkServiceAffinityPred,
     MaxEBSVolumeCountPred,
     MaxGCEPDVolumeCountPred,
     MaxAzureDiskVolumeCountPred,
     CheckVolumeBindingPred,
     NoVolumeZoneConflictPred,
     CheckNodeMemoryPressurePred,
     CheckNodeDiskPressurePred,
     MatchInterPodAffinityPred,
   }
   ```

&emsp;&emsp;同时我们也可以通过policy调度策略配置文件自定义优先级：

```Json
{
   "kind" : "Policy",
   "apiVersion" : "v1",
   "predicates" : [
   	{"name" : "PodFitsHostPorts", "order": 2},
   	{"name" : "PodFitsResources", "order": 3},
   	{"name" : "NoDiskConflict", "order": 5},
   	{"name" : "PodToleratesNodeTaints", "order": 4},
   	{"name" : "MatchNodeSelector", "order": 6},
   	{"name" : "PodFitsHost", "order": 1}
   	],
   "priorities" : [
   	{"name" : "LeastRequestedPriority", "weight" : 1},
   	{"name" : "BalancedResourceAllocation", "weight" : 1},
   	{"name" : "ServiceSpreadingPriority", "weight" : 1},
   	{"name" : "EqualPriority", "weight" : 1}
   	],
   "hardPodAffinitySymmetricWeight" : 10
}
```

- **Priorities（优选阶段）**

   ​&emsp;&emsp;经过 Predicates 剩下的 Node，需要经过Priorities 选出一个最优的 Node。

```text
1. 首先计算通过Predicates 的每个 Node的在每一项Priorities算法的得分并乘以此算法的权重, 每项得分相加得到node的总的得分。

2. 然后根据优先级队列 (由heap实现)排序，随机选取得分最高的node，进行调度。
```

   此阶段的原则是：

   1. 资源利用率达到最优

   2. 优先满足用户的需求（亲和性）

   3. 分散应用，应用的高可用

## kube-scheduler源码解析

kube-scheduler时序图如下所示：

![kube-scheduler时序图](/images/kube-scheduler.png)

&emsp;&emsp;cmd/kube-scheduler/scheduler.go 为程序入口文件 (main.go)

&emsp;&emsp;cmd/kube-scheduler/app/server.go 包含 scheduler 的基础配置项, 以及调度的整体框架逻辑。

&emsp;&emsp;pkg/scheduler/factory/factory.go 主要包含 scheduler.Configurator的默认实现，调度配置的工厂实现。

&emsp;&emsp;pkg/scheduler/scheduler.go 主要包含Scheduler行为的定义实现。

&emsp;&emsp;pkg/scheduler/core/generic_scheduler.go 具体包含genericScheduler具体的实现，此处是scheduler的核心抽象。其中最总要的方法就是Schedule，可以说这里是scheduler的核心。它使用设计模式里面的模板方法模式，也就是不同的实现可以修改特定的步骤，但是这些步骤的执行顺序仍然是固定的。

```go
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	trace := utiltrace.New(fmt.Sprintf("Scheduling %s/%s", pod.Namespace, pod.Name))
	defer trace.LogIfLong(100 * time.Millisecond)

	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return "", err
	}

	nodes, err := nodeLister.List()
	if err != nil {
		return "", err
	}
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// Used for all fit and priority funcs.
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	if err != nil {
		return "", err
	}

	trace.Step("Computing predicates")
	startPredicateEvalTime := time.Now()
	filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders, g.predicateMetaProducer, g.equivalenceCache, g.schedulingQueue, g.alwaysCheckAllPredicates)
	if err != nil {
		return "", err
	}

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			NumAllNodes:      len(nodes),
			FailedPredicates: failedPredicateMap,
		}
	}
	metrics.SchedulingAlgorithmPredicateEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPredicateEvalTime))

	trace.Step("Prioritizing")
	startPriorityEvalTime := time.Now()
	// When only one node after predicate, just use it.
	if len(filteredNodes) == 1 {
		metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
		return filteredNodes[0].Name, nil
	}

	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	if err != nil {
		return "", err
	}
	metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))

	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```

大概的调度流程如下：

1. NewSchedulerCommand 初始化一个 scheduler命令行实例，用来对 scheduler 的命令行参数进行解析校验，并且初始化 scheduler 程序的入口函数 Run 的定义。

2. command.Execute() 会执行NewSchedulerCommand 初始化后的 options.Run 方法。

3. 在 options.Run 中主要进行了如下的操作：

   1. loadConfigFromFile 加载 scheduler 的配置文件信息。

   2. ApplyFeatureGates根据FeatureGates调整算法。

   3. NewSchedulerServer 根据配置文件初始化 schedulerserver 实例:

      1. createClients 创建一系列 client，如连接 k8s 的 client，进行 scheduler 选主的 client 及 event client.
      2. makeLeaderElectionConfig 生成 Leader Election 配置信息 (scheduler做了 HA，可以同时运行多个实例进程，但只有一个能正常工作，如果主的 scheduler 挂了，会重新进行选举)。
      3. makeHealthzServer 初始化 healthz server，用于健康检查。
      4. makeMetricsServer 初始化 metrics server，用于 prometheus 性能监控。

   4. SchedulerServer.Run 启动 SchedulerServer,用于监控还是否有 Pod 待调度，并且进行相应的调度工作。具体的代码实现：

      1. SchedulerConfig() 创建Scheduler.Config，其中关键性函数是 NewConfigFactory 和 CreateFromProvider。

         1. NewConfigFactory 定义了 podQueue , 它用来存储需要被调度的 Pod，每当新的 Pod 建立后，就会将 Pod 添加到该 queue 中。

         2. CreateFromProvider 根据 algorithmprovider 名称创建一个 scheduler 配置信息。其中 GetAlgorithmProvider 则根据 provider 名称去获取指定的 provider。scheduler 默认使用的 provider 是 DefaultProvider。

            ```go
            // AlgorithmProviderConfig is used to store the configuration of algorithm providers.
            type AlgorithmProviderConfig struct {
            	FitPredicateKeys     sets.String
            	PriorityFunctionKeys sets.String
            }
            ```

            AlgorithmProviderConfig 这个数据结构包含筛选和优选相关算法 key 的集合(一个算法对应一个key，key 是算法的名字，value 是算法的具体实现 funtion)，而这些算法注册是在 scheduler/algorithmprovider/defaults/defaults.go 文件的 init() 方法中进行注册 (工厂模式)。

            通过 GetAlgorithmProvider 得到了 provider 关联的筛选和优选算法集合的 Key。然后通过调用 CreateFromKeys (筛选和优选的 Key作为参数) 来获取筛选和优选算法的具体实现 (funtion),并对 NewGenericScheduler 实例进行初始化，返回最终的 scheduler 配置信息。

      2. NewFromConfig 由 Scheduler.Config 创建一个 scheduler。

      3. Start up the healthz server 启动健康检查服务

      4. Start up the metrics server 启动 metrics 服务，供 Prometheus 进行性能监控数据的抓取。

      5. LeaderElection 如果指定选举的方式来启动scheduler，则使用这种方式来执行scheduler。(使用 CallBack 的方式执行 Run 方法。如果主的 scheduler 出现问题，还会指定优雅处理函数对其进行处理)。

   5. Scheduler.Run()调度 Pod 的具体逻辑：

      ```go
      // Run begins watching and scheduling. It waits for cache to be synced, then starts a goroutine and returns immediately.
      func (sched *Scheduler) Run() {
      	if !sched.config.WaitForCacheSync() {
      		return
      	}
      
      	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
      		go sched.config.VolumeBinder.Run(sched.bindVolumesWorker, sched.config.StopEverything)
      	}
      
      	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
      }
      ```

      1. WaitForCacheSync() 将最新的数据同步到 SchedulerCache 缓存中。
      2. scheduleOne() 调度 Pod 的整体逻辑：
         1. NextPod() 从 PodQueue 中获取一个未绑定的Pod。
         2. schedule(pod) 执行对应 Algorithm 的 Schedule，进行筛选和优选。Schedule 主要逻辑：
            1. nodeLister.List() 获取可用的 Node 列表。
            2. findNodesThatFit() 进行筛选。
            3. PrioritizeNodes() 进行优选。
            4. selectHost() 如果优选出的多个得分相同的 Node，则随机选取一个 Node。
         3. assume() 更新 SchedulerCache 中 Pod 的状态，标志该 Pod 为 scheduled，并更新到 NodeInfo 中。
         4. bind() 调用 kube-apiserver API，将 Pod 绑定到选出的 Node，之后 Kube-apiserver 会将元数据写入 etcd 中。

## kube-scheduler目录结构

```go
.
├── BUILD
├── OWNERS
├── algorithm
│   ├── BUILD
│   ├── doc.go
│   ├── predicates
│   │   ├── BUILD
│   │   ├── error.go
│   │   ├── metadata.go   //PredicateMetadataFactory实现，产生筛选的时需要用到的一些metadata.
│   │   ├── metadata_test.go
│   │   ├── predicates.go    //predicates各个算法的具体实现
│   │   ├── predicates_test.go
│   │   ├── testing_helper.go
│   │   ├── utils.go
│   │   └── utils_test.go
│   ├── priorities // priorities各个优先算法的具体实现
│   │   ├── BUILD
│   │   ├── balanced_resource_allocation.go
│   │   ├── balanced_resource_allocation_test.go
│   │   ├── image_locality.go
│   │   ├── image_locality_test.go
│   │   ├── interpod_affinity.go
│   │   ├── interpod_affinity_test.go
│   │   ├── least_requested.go
│   │   ├── least_requested_test.go
│   │   ├── metadata.go
│   │   ├── metadata_test.go
│   │   ├── most_requested.go
│   │   ├── most_requested_test.go
│   │   ├── node_affinity.go
│   │   ├── node_affinity_test.go
│   │   ├── node_label.go
│   │   ├── node_label_test.go
│   │   ├── node_prefer_avoid_pods.go
│   │   ├── node_prefer_avoid_pods_test.go
│   │   ├── reduce.go
│   │   ├── resource_allocation.go
│   │   ├── resource_limits.go
│   │   ├── resource_limits_test.go
│   │   ├── selector_spreading.go
│   │   ├── selector_spreading_test.go
│   │   ├── taint_toleration.go
│   │   ├── taint_toleration_test.go
│   │   ├── test_util.go
│   │   └── util
│   │       ├── BUILD
│   │       ├── non_zero.go
│   │       ├── non_zero_test.go
│   │       ├── topologies.go
│   │       ├── topologies_test.go
│   │       ├── util.go
│   │       └── util_test.go
│   ├── scheduler_interface.go //定义了ScheduleAlgorithm跟SchedulerExtender接口
│   ├── scheduler_interface_test.go
│   ├── types.go // 定义了priorities通用的一些类型，结构体，接口
│   ├── types_test.go
│   └── well_known_labels.go
├── algorithmprovider // algorithmprovider包包含了调度器的实现
│   ├── BUILD
│   ├── defaults // 默认调度器的实现
│   │   ├── BUILD
│   │   ├── compatibility_test.go
│   │   ├── defaults.go
│   │   └── defaults_test.go
│   ├── plugins.go 
│   └── plugins_test.go
├── api // policy api相关的定义，注册，scheme验证等
│   ├── BUILD
│   ├── doc.go
│   ├── latest
│   │   ├── BUILD
│   │   └── latest.go
│   ├── register.go
│   ├── types.go
│   ├── v1
│   │   ├── BUILD
│   │   ├── doc.go
│   │   ├── register.go
│   │   ├── types.go
│   │   └── zz_generated.deepcopy.go
│   ├── validation
│   │   ├── BUILD
│   │   ├── validation.go
│   │   └── validation_test.go
│   └── zz_generated.deepcopy.go
├── core //调度单个pod的核心实现逻辑
│   ├── BUILD
│   ├── equivalence_cache.go //EquivalenceCache的实现，用于筛选阶段，避免筛选阶段一些数据的重复计算，例如相同controller生成的pod只会用第一个pod的结果，从而优化了调度执行时间
│   ├── equivalence_cache_test.go
│   ├── extender.go //HTTPExtender的实现，用于根据不是Kubernetes所管理的资源信息进行调度抉择
│   ├── extender_test.go
│   ├── generic_scheduler.go //genericScheduler的实现，调度一个pod的核心逻辑
│   ├── generic_scheduler_test.go
│   ├── scheduling_queue.go //调度队列的实现，包括FIFO，PriorityQueue
│   └── scheduling_queue_test.go
├── factory 
│   ├── BUILD
│   ├── factory.go //主要用于scheduler配置的初始化，用于启动scheduler
│   ├── factory_test.go
│   ├── plugins.go
│   └── plugins_test.go
├── metrics
│   ├── BUILD
│   └── metrics.go //调度相关的一些metrics，对接prometheus
├── scheduler.go //Scheduler实现，调度逻辑
├── scheduler_test.go
├── schedulercache //schedulerCache实现，主要更新缓存中的pod，node信息
│   ├── BUILD
│   ├── cache.go
│   ├── cache_test.go
│   ├── interface.go
│   ├── node_info.go
│   ├── node_info_test.go
│   └── util.go
├── testing
│   ├── BUILD
│   ├── fake_cache.go
│   ├── fake_lister.go
│   └── pods_to_cache.go
├── testutil.go
├── util
│   ├── BUILD
│   ├── backoff_utils.go
│   ├── backoff_utils_test.go
│   ├── testutil.go
│   ├── testutil_test.go
│   ├── utils.go
│   └── utils_test.go
└── volumebinder // VolumeBinder管理volume的bind操作
    ├── BUILD
    └── volume_binder.go
```

## 自定义调度器

默认情况下，k8s中的所有pod都使用DefaultProvider。 可以通过pod的字段：`“scheulderName”：“defaultScheduler”`自定义调度器。

1. 你可以实现自己的调度器并重新编译Kube-scheduler，新增调度器必须实现[algorithm.ScheduleAlgorithm](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/algorithm/scheduler_interface.go#L46-L61).
2. K8s也支持使用自定义调度器作为单独的进程。自定义调度器像k8s中的其他正常部署的应用一样运行（使用deployment资源对象），由Kube-scheduler调度管理。比如我们有一个my-scheduler的调度器，它将负责调度所有具有`“schedulerName”：“my-scheduler”`的pod。

## k8s调度相关的Feature

kubernetes调度相关的一些Feature：

- PodPriority=true|false (ALPHA - default=false)：pod优先级调度
- EnableEquivalenceClassCache=true|false (ALPHA - default=false): 等价类cache

- VolumeScheduling=true|false (BETA - default=true): 调度时考虑PV的拓扑

- ScheduleDaemonSetPods=true|false (ALPHA - default=false): 由scheduler调度DaemonSetPod而不是DaemonSet controller

- ExperimentalCriticalPodAnnotation=true|false (ALPHA - default=false): 保护有“scheduler.alpha.kubernetes.io/critical-pod“annotation的pod不被node驱逐。

## FIFO和PriorityQueue

kube-scheduler目前有两种调度队列，分别是FIFO和PriorityQueue，使用PriorityQueue需要开启PodPriority功能。

```go
// NewSchedulingQueue initializes a new scheduling queue. If pod priority is
// enabled a priority queue is returned. If it is disabled, a FIFO is returned.
func NewSchedulingQueue() SchedulingQueue {
	if util.PodPriorityEnabled() {
		return NewPriorityQueue()
	}
	return NewFIFO()
}
```

他们都实现了如下接口：

```Go
// SchedulingQueue is an interface for a queue to store pods waiting to be scheduled.
// The interface follows a pattern similar to cache.FIFO and cache.Heap and
// makes it easy to use those data structures as a SchedulingQueue.
type SchedulingQueue interface {
	Add(pod *v1.Pod) error
	AddIfNotPresent(pod *v1.Pod) error
	AddUnschedulableIfNotPresent(pod *v1.Pod) error
	Pop() (*v1.Pod, error)
	Update(oldPod, newPod *v1.Pod) error
	Delete(pod *v1.Pod) error
	MoveAllToActiveQueue()
	AssignedPodAdded(pod *v1.Pod)
	AssignedPodUpdated(pod *v1.Pod)
	WaitingPodsForNode(nodeName string) []*v1.Pod
}
```

### FIFO

​FIFO调度队列以先到先服务的顺序为pod进行调度。 它将所有的pod统一无区别对待。这是默认的调度队列（逻辑简单，方便实现，不需要复杂的并发控制，可以加快并发速度）

```Go
// FIFO is basically a simple wrapper around cache.FIFO to make it compatible
// with the SchedulingQueue interface.
type FIFO struct {
	*cache.FIFO
}
```

### PriorityQueue

​PriorityQueue是另外一种可选的调度队列，它实现了根据pod的优先级进行调度。实现了schedulingQueue的接口。PriorityQueue头部的pod是待调度的最高优先级的pod。

```Go
type PriorityQueue struct {
	lock sync.RWMutex
	cond sync.Cond

	// activeQ is heap structure that scheduler actively looks at to find pods to
	// schedule. Head of heap is the highest priority pod.
	activeQ *Heap
	// unschedulableQ holds pods that have been tried and determined unschedulable.
	unschedulableQ *UnschedulablePodsMap
	// nominatedPods is a map keyed by a node name and the value is a list of
	// pods which are nominated to run on the node. These are pods which can be in
	// the activeQ or unschedulableQ.
	nominatedPods map[string][]*v1.Pod
	// receivedMoveRequest is set to true whenever we receive a request to move a
	// pod from the unschedulableQ to the activeQ, and is set to false, when we pop
	// a pod from the activeQ. It indicates if we received a move request when a
	// pod was in flight (we were trying to schedule it). In such a case, we put
	// the pod back into the activeQ if it is determined unschedulable.
	receivedMoveRequest bool
}
```

​	PriorityQueue包含两个子队列：activeQ中pod是等待被调度的pod。unschedulableQ中的pod是已经进行了调度决策并决定暂时不调度，等k8s中pod，node状态发生变化时候会重新回到activeQ队列中。

大概逻辑如下：

![](/images/PriorityQueue.png)

## DefaultProvider默认调度策略

### Default Predicates

- NoVolumeZoneConflict: 如果pod使用的pvc的pv里声明了zone，检查node是否与卷有冲突

- MaxEBSVolumeCount：确保已挂载的EBS存储卷加上pod新请求的不超过设置的最大值，默认39

- MaxGCEPDVolumeCount：确保已挂载的GCE存储卷PD加上pod新请求的不超过设置的最大值，默认16

- MaxAzureDiskVolumeCount：确保已挂载的Azure存储卷加上pod新请求的不超过设置的最大值，默认16

- MatchInterPodAffinity：检查pod的pod affinity/antiaffinity是否允许

- NoDiskConflict：检查在此主机上是否存在卷冲突。如果这个主机已经挂载了卷，其它同样使用这个卷的Pod不能调度到这个主机上，不同的存储后端具体规则不同

- GeneralPredicates ：包括非关键筛选算法的基本筛选算法。非关键筛选算法是非关键pod需要检查，而基本筛选算法是所有pod都需要检查。

  1. 非关键筛选算法

     PodFitsResources： pod请求的资源是否满足，CPU，内存，GPU等等

  2. 基本筛选算法

     PodFitsHost：检查node的NodeName是否是pod.Spec中指定的

     PodFitsHostPorts：检查pod申请的端口映射的主机端口是否被node上已经运行的pod占用

     PodMatchNodeSelector：检查node上标签是否满足pod的nodeSelector和NodeAffinity，这两项需要同时满足

- CheckNodeMemoryPressure：如果pod的QoS级别为BestEffort，当node处在MemoryPressureCondition时，不允许调度。（BestEffort级别的pod的oom_score分数会很高，是omm killer首要的kill对象，因此内存在有压力状态即使pod调度过去也会被马上杀掉）

- CheckNodeDiskPressure：检查pod是否可以调度到已经报告了主机的存储压力过大的节点（如果node的处于DiskPressureCondition状态，则不允许任何pod调度在上

- CheckNodeCondition： 检查pod是否可调度到有以下node状况的节点 out of disk, network不可达，not ready

- PodToleratesNodeTaints：检查pod上的toleration能否适配node上taints

- CheckVolumeBinding：检查所有请求的PVC，如果PVC绑定了PV，这检查node与此PV的亲和性；如果PVC没有绑定PV，则检查可用的PV与node的亲和性。

### Default Priorities

priority会分配一个二维切片，每个算法为一行，每个node为一列，以此临时存储每个priority算法对每个node的打分，最后再使用每个算法的Weight权重乘以每个node分数，累加起来得到每个node最后的总分数。priority算法目前分为两类：

```go
// PriorityConfigFactory produces a PriorityConfig from the given function and weight
type PriorityConfigFactory struct {
	Function          PriorityFunctionFactory
	MapReduceFunction PriorityFunctionFactory2
	Weight            int
}
```

 1. 第一类是早期版本使用的PriorityConfigFactory.Function，

    ```Go
    // PriorityFunctionFactory produces a PriorityConfig from the given args.
    // DEPRECATED
    // Use Map-Reduce pattern for priority functions.
    type PriorityFunctionFactory func(PluginFactoryArgs) algorithm.PriorityFunction
    ```

    这些Function类型的算法都被标记了DEPRECATED即未来都会重构为第二类。

	2.  第二类使用的PriorityConfigFactory.MapReduceFunction

     ```Go
     // PriorityFunctionFactory2 produces map & reduce priority functions
     // from a given args.
     // FIXME: Rename to PriorityFunctionFactory.
     type PriorityFunctionFactory2 func(PluginFactoryArgs) (algorithm.PriorityMapFunction, algorithm.PriorityReduceFunction)
     ```

     每次PriorityMapFunction只处理一个node，使用workqueue.Parallelize生产者消费者的并行方式，最后由PriorityReduceFunction把所有node的得分映射到0 -10这个分数段。

默认的使用DefaultProvider注册了下列priority算法:

- SelectorSpreadPriority：同一个Service、ReplicationController、ReplicaSet、StatefulSet下的pod 分配到不同的节点 ，权重为1。

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式。PriorityMapFunction为计算每个node上的相关的pod数，PriorityReduceFunction计算每个node的最终得分，公式如下所示：

```Bash
  fScore = MaxPriorityFloat64 * (float64(maxCountByNodeName-result[i].Score)/maxCountByNodeNameFloat64)
  //maxCountByNodeName=所有node中最大的匹配pod个数
  
  zoneScore = MaxPriorityFloat64 * (float64(maxCountByZone-countsByZone[zoneID])/maxCountByZoneFloat64)
  // maxCountByZone=所有zone中最大匹配的pod个数
  
  fScore = (fScore * (1.0 - zoneWeighting)) + (zoneWeighting * zoneScore)
  
  zoneWeighting默认权重为2.0/3.0，2/3=zone spread权重，1/3=node spread权重
```

- InterPodAffinityPriority：pod与其他亲和性的pod调度在在同一个的拓扑域中，比如同一个node，机架，zone，供电域。 权重为1。

  ​	在PodAffinity中需要进行双向检查，即待调度的pod的Affinity检查已存在pod，已存在pod的Affinity检查待调度pod。优先算法需要处理每个node和每个node上的每个正在运行的pod（所以大规模集群不建议开启pod affinity）。

  ​	该算法使用PriorityConfigFactory.Function第一类方式，计算过程如下：

  1. 首先检查了待调度pod的Affinity和AntiAffinity的PreferredDuringSchedulingIgnoredDuringExecution能否匹配当前检查已存在的pod，如果匹配成功则会给已运行pod所在node及相同拓扑域的所有node加上（AntiAffinity对应减去）`1*Weight`得分。
  2. 而对于已存在pod检查待调度pod除了常规的PreferredDuringSchedulingIgnoredDuringExecution外，还特别检查了Affinity的RequiredDuringSchedulingIgnoredDuringExecution，Require应该都是出现在Predicate算法中，而在这Priority出现原因通过官方设计文档解读，还是由于类似的对称性，这里特意给了这个Require一个特殊的参数hardPodAffinityWeight，这个参数是由DefaultProvider提供的（默认值是1），因此已存在的pod的RequiredDuringSchedulingIgnoredDuringExecution如果匹配到待调度pod，与其运行的node具有相同拓扑域的全部node都会增加`hardPodAffinityWeight*Weight`得分。
  3. 最后得到全部node得分后在将映射到0-10段。

- LeastRequestedPriority：优选资源利用率最少的节点。 权重为1。

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式，PriorityMapFunction公式如下，PriorityReduceFunction为空：

```
  cpu((capacity-sum(requested))*10/capacity) + memory((capacity-sum(requested))*10/capacity)/2
```

- BalancedResourceAllocation：优选CPU，memory利用率相近的node，必须与LeastRequestedPriority一起使用。 权重为1。

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式，PriorityMapFunction公式如下，PriorityReduceFunction为空：

```bash
  score = 10 – abs(cpuFraction-memoryFraction)*10  Fraction = requested/capacity
```

- NodePreferAvoidPodsPriority：根据"scheduler.alpha.kubernetes.io/preferAvoidPods" 此node annotation 优选node，禁止rc或者rs这种controller的pod调度在上面，默认权重是10000，即一旦该函数的结果不为0，就由它决定排序结果。

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式，PriorityMapFunction代码如下，PriorityReduceFunction为空。

```go
  	if controllerRef != nil {
  		// Ignore pods that are owned by other controller than ReplicationController
  		// or ReplicaSet.
  		if controllerRef.Kind != "ReplicationController" && controllerRef.Kind != "ReplicaSet" {
  			controllerRef = nil
  		}
  	}
  	if controllerRef == nil {
  		return schedulerapi.HostPriority{Host: node.Name, Score: schedulerapi.MaxPriority}, nil
  	}
  
  	avoids, err := v1helper.GetAvoidPodsFromNodeAnnotations(node.Annotations)
  	if err != nil {
  		// If we cannot get annotation, assume it's schedulable there.
  		return schedulerapi.HostPriority{Host: node.Name, Score: schedulerapi.MaxPriority}, nil
  	}
  	for i := range avoids.PreferAvoidPods {
  		avoid := &avoids.PreferAvoidPods[i]
  		if avoid.PodSignature.PodController.Kind == controllerRef.Kind && avoid.PodSignature.PodController.UID == controllerRef.UID {
  			return schedulerapi.HostPriority{Host: node.Name, Score: 0}, nil
  		}
  	}
  	return schedulerapi.HostPriority{Host: node.Name, Score: schedulerapi.MaxPriority}, nil
  }
```

- NodeAffinityPriority：根据node亲和性优选node，匹配的label的次数为几次，此值就加几次。

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式，PriorityMapFunction代码如下，PriorityReduceFunction正归一化结果到[0,10]。

```go
  	// A nil element of PreferredDuringSchedulingIgnoredDuringExecution matches no objects.
  	// An element of PreferredDuringSchedulingIgnoredDuringExecution that refers to an
  	// empty PreferredSchedulingTerm matches all objects.
  	if affinity != nil && affinity.NodeAffinity != nil && affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution != nil {
  		// Match PreferredDuringSchedulingIgnoredDuringExecution term by term.
  		for i := range affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution {
  			preferredSchedulingTerm := &affinity.NodeAffinity.PreferredDuringSchedulingIgnoredDuringExecution[i]
  			if preferredSchedulingTerm.Weight == 0 {
  				continue
  			}
  
  			// TODO: Avoid computing it for all nodes if this becomes a performance problem.
  			nodeSelector, err := v1helper.NodeSelectorRequirementsAsSelector(preferredSchedulingTerm.Preference.MatchExpressions)
  			if err != nil {
  				return schedulerapi.HostPriority{}, err
  			}
  			if nodeSelector.Matches(labels.Set(node.Labels)) {
  				count += preferredSchedulingTerm.Weight
  			}
  		}
  	}
```

  ​	首先检查了待调度pod的NodeAffinity的PreferredDuringSchedulingIgnoredDuringExecution能否匹配当前检查的node，如果匹配成功则会给node的得分加上`preferredSchedulingTerm.Weight`。

- TaintTolerationPriority：优选pod有最少无法容忍label的node，无法容忍的key为PreferNoSchedule。权重为1.

  ​	该算法使用PriorityConfigFactory.MapReduceFunction第二类方式，PriorityMapFunction代码如下，PriorityReduceFunction负归一化结果到[0,10]。

```go
// ComputeTaintTolerationPriorityMap prepares the priority list for all the nodes based on the number of intolerable taints on the node
func ComputeTaintTolerationPriorityMap(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (schedulerapi.HostPriority, error) {
	node := nodeInfo.Node()
	if node == nil {
		return schedulerapi.HostPriority{}, fmt.Errorf("node not found")
	}
	// To hold all the tolerations with Effect PreferNoSchedule
	var tolerationsPreferNoSchedule []v1.Toleration
	if priorityMeta, ok := meta.(*priorityMetadata); ok {
		tolerationsPreferNoSchedule = priorityMeta.podTolerations

	} else {
		tolerationsPreferNoSchedule = getAllTolerationPreferNoSchedule(pod.Spec.Tolerations)
	}

	return schedulerapi.HostPriority{
		Host:  node.Name,
		Score: countIntolerableTaintsPreferNoSchedule(node.Spec.Taints, tolerationsPreferNoSchedule),
	}, nil
}

// 计算node节点有多少个key为PreferNoSchedule且不被pod所有的tolerations所容忍的Taints
```