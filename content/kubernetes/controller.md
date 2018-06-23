---
title: "kubernetes中如何编写自定义的controller"
date: 2018-06-23T15:09:51+08:00
draft: false
---

# 编写自定义的controller

&emsp;&emsp;在Kubernetes中，controller是一个控制循环，通过informer监视集群的资源对象，并进行相应的更改，尝试将当前状态移至期望的状态。伪代码如下：

```Go
for {
  desired := getDesiredState()
  current := getCurrentState()
  makeChanges(desired, current)
}
```

## Informer与SharedInformer区别

&emsp;&emsp;controller的重要作用是观察资源对象的目标状态和实际状态，然后尝试将当前状态移至期望的状态。为了获取资源对象的信息，controller需要向Kubernetes的apiserver发送请求 。但是，反复从apiserver检索信息会使apiserver压力过大同时时延也是个问题。 因此，Kubernetes通过informer实现了在代码中多次方便快速地获取资源对象，把最新的状态反映到本地的 cache 中，而不用每次都去请求apiserver。

&emsp;&emsp;controller有两个主要组件：Informer/SharedInformer和Workqueue。 Informer/SharedInformer监视Kubernetes资源对象的当前状态的变化，并将事件发送到Workqueue，然后由处理逻辑弹出事件进行处理。

&emsp;&emsp;informer创建仅由controller自身使用的一组资源的本地缓存。但是，在Kubernetes中，有一组controller运行并关心多种资源，也就是一个资源对象正受到多个controller的关注。在这种情况下，SharedInformer有助于在controller间创建单个共享缓存。这意味着缓存的资源不会被复制多份，降低了系统的内存开销。此外，每个SharedInformer仅在上游apiserver上创建一个监视，无论有多少下游controller正在读取资源对象的事件。这也减少了上游apiserver的负载。这对于拥有如此多内部控制器的kube-controller-manager来说很常见。

&emsp;&emsp;使用SharedInformer无法在回调函数里处理每个controller的具体逻辑（因为它是共享的，Informer可以直接在回调函数里处理controller的具体逻辑），**所以使用SharedInformer的controller必须提供自己的队列（Workqueue）和重试机制（如果需要）**。此时的回调函数只是将解析事件并把资源对象的key放置到每个controller消费者的Workqueue中。key的格式为<resource_namespace> / <resource_name>，除非<resource_namespace>为空，那么它只是<resource_name>。 同时一个相同的key被多次加入 queue 的话会进行合并处理，保证了多个work不会同时处理同一个key。

&emsp;&emsp;workqueue提供了三种队列，包括延迟队列，定时队列和速率限制队列。以下是创建速率限制队列的示例：

```Go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())
```

&emsp;&emsp;workqueue提供了便利的功能来管理key。 下图描述了在workqueue中key的生命周期：

![workqueue](/images/workqueue.png)

&emsp;&emsp;在处理事件失败的情况下controller调用AddRateLimited()函数将key重新推回到workqueue，以便稍后使用预定义次数的重试进行重试。如果处理成功，可以通过调用Forget()函数将该key从工作队列中移除。但是，该功能只能停止追踪事件历史。为了完全从workqueue中移除事件，controller必须触发Done()函数。

&emsp;&emsp;需要注意的是: 需要等待本地 cache sync 完成后， 才能启动 workers（避免没有sync完成前过多的无用功）。下面的伪代码描述了正确的用法：

```Go
controller.informer = cache.NewSharedInformer(...)
controller.queue = workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

controller.informer.Run(stopCh)

if !cache.WaitForCacheSync(stopCh, controller.HasSynched)
{
	log.Errorf("Timed out waiting for caches to sync"))
}

controller.runWorker()
```

&emsp;&emsp;那么我们在编写自定义的controller时应该选择那个呢：

1. 如果要编写的自定义controller关注的是Kubernetes内置的资源对象如pod，service，Deployment等，建议使用SharedInformer。

2. 如果要编写的自定义controller关注的是用户新定义的资源对象（如通过CRD定义的）且不被其他controller共享，且不易变化的。建议使用Informer。

3. SharedInformer必须使用Workqueue，而Informer可以根据自己需要选择是否使用Workqueue（重试，延迟，定时等功能）

&emsp;&emsp;我们的stackube项目就是使用的Informer且没有使用Workqueue，因为controller关注的资源只有它自己使用，且不易变化。这样代码量少，控制逻辑清晰。（我们最初的commit使用的是SharedInformer+Workqueue）

&emsp;&emsp;下面分别通过具体的例子介绍下Informer和SharedInformer+Workqueue编写自定义controller

## 使用Informer构建自定义controller

&emsp;&emsp;在本小节中，我将通过stackube项目中的[TenantController](https://github.com/openstack/stackube/tree/master/pkg/auth-controller/tenant)实际例子展示如何在Kubernetes中编写自定义控制器。TenantController监视Kubernetes中通过CRD新定义的tenant资源对象的更改并相应的操作openstack中的tenant跟user对象，以及Kubernetes中namespace（因为一个tenant对应一个namespace）。

### 编写controller结构体

一般来说，我们首先需要根据自己的业务需求编写一个控制器的结构体，TenantController的结构体如下：

```go
// TenantController manages the life cycle of Tenant.
type TenantController struct {
	k8sClient       kubernetes.Interface
	kubeCRDClient   crdClient.Interface
	openstackClient openstack.Interface
}
```

k8sClient：持有Kubernetes的client 接口，用于控制器与Kubernetes API server进行交互。

kubeCRDClient：持有Kubernetes的CRD client 接口，用于控制器与Kubernetes API server中的CRD资源对象进行交互。

openstackClient：持有openstack的client 接口，用于控制器与openstack进行交互。

### 定义ListerWatcher

&emsp;&emsp;Listerwatcher是特定命名空间中特定资源的list函数和watch函数的组合。 这有助于controller只关注它想要查看的特定资源。 field选择器是一种过滤器，它缩小搜索资源的范围，例如controller想要检索与特定字段匹配的资源。 Listerwatcher的结构如下：

```Go
cache.ListWatch {
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Do().
			Get()
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Watch()
	}
}
```

&emsp;&emsp;TenantController这里我们用`cache.NewListWatchFromClient()`函数构建Listerwatcher：

```Go
	source := cache.NewListWatchFromClient(
		c.kubeCRDClient.Client(),
		crv1.TenantResourcePlural,
		apiv1.NamespaceAll,
		fields.Everything())
```

### 定义ResourceEventHandlerFuncs

&emsp;&emsp;ResourceEventHandlerFuncs是controller根据特定资源的更改进行逻辑处理的地方，这里是业务逻辑的核心部分。

```Go
// ResourceEventHandlerFuncs is an adaptor to let you easily specify as many or
// as few of the notification functions as you want while still implementing
// ResourceEventHandler.
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```

- AddFunc在创建新资源时被调用。
- UpdateFunc在修改现有资源时调用。 oldObj是资源的最后已知状态。 当re-list发生时，UpdateFunc也被调用，即使没有任何变化，它也会被调用。
- DeleteFunc在删除现有资源时调用。 它获得资源的最终状态（如果它是已知的）。 否则，它将获得类型为DeletedFinalStateUnknown的对象。 如果watch关闭并且错过了删除事件，并且controller在随后的re-list中没有注意到删除，则会发生这种情况。

例如TenantController的AddFunc函数为：

```go
func (c *TenantController) onAdd(obj interface{}) {
	tenant := obj.(*crv1.Tenant)
	glog.V(3).Infof("Tenant controller received new object %#v\n", tenant)

	copyObj, err := c.kubeCRDClient.Scheme().Copy(tenant)
	if err != nil {
		glog.Errorf("ERROR creating a deep copy of tenant object: %#v\n", err)
		return
	}

	newTenant := copyObj.(*crv1.Tenant)
	c.syncTenant(newTenant)
}
```

### 构建Informer并运行

NewInformer的定义为：

```go
func NewInformer(
	lw ListerWatcher,
	objType runtime.Object,
	resyncPeriod time.Duration,
	h ResourceEventHandler,
) (Store, Controller){
    ·····
}
```

&emsp;&emsp;其中resyncPeriod定义controller再次re-list的时间间隔。 用于周期性地验证当前状态并使其像期望状态那样，在controller可能错过更新或之前的操作失败的情况下，它非常有用。 但是，如果您构建自定义控制器，则如果周期时间太短，会增大CPU负载。

&emsp;&emsp;但目前现有的这种 List/Watch 机制，完全能够保证永远不会漏掉任何事件，因此完全没有必要再添加 re-list 方法去 resync informer 的缓存。所目前该值一般配置为0

例如初始化TenantController的Informor并运行：

```go
	_, tenantInformor := cache.NewInformer(
		source,
		&crv1.Tenant{},
		0,
		cache.ResourceEventHandlerFuncs{
			AddFunc:    c.onAdd,
			UpdateFunc: c.onUpdate,
			DeleteFunc: c.onDelete,
		})

	go tenantInformor.Run(stopCh)
	<-stopCh
```

## 使用SharedInformer+Workqueue构建自定义controller

&emsp;&emsp;在本小节中，我将通过[Kubewatch](https://github.com/bitnami-labs/kubewatch)项目展示如何通过SharedInformer+Workqueue构建自定义控制器。 Kubewatch监视pod中发生的任何更改并将通知发送到Slack。

### 构建controller结构体

```Go
type Controller struct {
      logger       *logrus.Entry
      clientset    kubernetes.Interface
      queue        workqueue.RateLimitingInterface
      informer     cache.SharedIndexInformer
      eventHandler handlers.Handler
}
```

logger管理controller日志。

clientset用于与Kubernetes apiserver进行交互。

queue是controller的workqueue。

informer是controller的SharedInformer。
eventHandler用于与Slack的通信。

### 构建WorkQueue 和SharedInformer

&emsp;&emsp;首先，我们为controller构建一个WorkQueue和一个SharedInformer。

Kubewatch相关代码如下：

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

informer := cache.NewSharedIndexInformer(
      &cache.ListWatch{
             ListFunc: func(options meta_v1.ListOptions) (runtime.Object, error) {
                    return client.CoreV1().Pods(meta_v1.NamespaceAll).List(options)
             },
             WatchFunc: func(options meta_v1.ListOptions) (watch.Interface, error) {
                    return client.CoreV1().Pods(meta_v1.NamespaceAll).Watch(options)
             },
      },
      &api_v1.Pod{},
      0, //Skip resync
      cache.Indexers{},
)
```

&emsp;&emsp;ListWatch表示controller想要list watch所有命名空间中的所有pod。
这里使用SharedIndexInformer而不是SharedInformer，因为它允许controller维护缓存中所有对象的索引。

### 初始化AddEventHandler函数

&emsp;&emsp;SharedIndexInformer因为是共享缓存的，所以自定义controller的回调函数AddEventHandler需要解析资源变化事件并把资源对象的key放置到controller的Workqueue中。

Kubewatch相关代码如下：

```go
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc: func(obj interface{}) {
             key, err := cache.MetaNamespaceKeyFunc(obj)
             if err == nil {
                    queue.Add(key)
             }
      },
      DeleteFunc: func(obj interface{}) {
             key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
             if err == nil {
                    queue.Add(key)
             }
      },
})
```

&emsp;&emsp;Workqueue中的事件由pod_namespace/pod_name格式的key来表示。在pod删除的情况下，我们必须在该key入队列之前检查缓存中该pod是否是DeletedFinalStateUnknown状态。 如果是DeletedFinalStateUnknown状态则意味着该pod已被删除，但watch删除事件被遗漏（可能是controller重启了）。这时候不需要controller作出反应。

### 编写Run()函数以及处理Workqueue中key的work()函数

&emsp;&emsp;构建好Workqueue和SharedInformer后。需要编写Run()函数来启动controller。

Kubewatch相关代码如下：

```go
// Run will start the controller.
// StopCh channel is used to send interrupt signal to stop it.
func (c *Controller) Run(stopCh <-chan struct{}) {
      // don't let panics crash the process
      defer utilruntime.HandleCrash()
      // make sure the work queue is shutdown which will trigger workers to end
      defer c.queue.ShutDown()

      c.logger.Info("Starting kubewatch controller")

      go c.informer.Run(stopCh)

      // wait for the caches to synchronize before starting the worker
      if !cache.WaitForCacheSync(stopCh, c.HasSynced) {
             utilruntime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
             return
      }

      c.logger.Info("Kubewatch controller synced and ready")

     // runWorker will loop until "something bad" happens.  The .Until will
     // then rekick the worker after one second
      wait.Until(c.runWorker, time.Second, stopCh)
}
```

​	SharedInformer启动后会list watch群集中的pod并将其key发送到Workqueue。这里我们必须定work函数如何pop并处理key（这里是controller业务逻辑的核心）。key在workqueue中的生命周期前文已经详细描述。

Kubewatch相关代码如下：

```go
func (c *Controller) runWorker() {
// processNextWorkItem will automatically wait until there's work available
      for c.processNextItem() {
             // continue looping
      }
}

// processNextWorkItem deals with one key off the queue.  It returns false
// when it's time to quit.
func (c *Controller) processNextItem() bool {
       // pull the next work item from queue.  It should be a key we use to lookup
	// something in a cache
      key, quit := c.queue.Get()
      if quit {
             return false
      }

       // you always have to indicate to the queue that you've completed a piece of
	// work
      defer c.queue.Done(key)

      // do your work on the key.
      err := c.processItem(key.(string))

      if err == nil {
             // No error, tell the queue to stop tracking history
             c.queue.Forget(key)
      } else if c.queue.NumRequeues(key) < maxRetries {
             c.logger.Errorf("Error processing %s (will retry): %v", key, err)
             // requeue the item to work on later
c.queue.AddRateLimited(key)
      } else {
             // err != nil and too many retries
             c.logger.Errorf("Error processing %s (giving up): %v", key, err)
             c.queue.Forget(key)
             utilruntime.HandleError(err)
      }

      return true
}
```
