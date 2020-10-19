# Replication Controller && ReplicaSet 源码剖析

## ReplicaSet 和 ReplicationController 区别

ReplicaSet 和 ReplicationController 的功能基本一致，目前唯一的一个区别就是 ReplicationController 只支持基于等式的selector（env=dev或environment!=qa），但RS还支持基于集合的selector（version in (v1.0, v2.0)）。

## ReplicationController

ReplicationController 确保在任何时候都有特定数量的 Pod 副本处于运行状态。 换句话说，ReplicationController 确保一个 Pod 或一组同类的 Pod 总是可用的。

首先来看ReplicationController的启动

```
func startReplicationController(ctx ControllerContext) (http.Handler, bool, error) {
	go replicationcontroller.NewReplicationManager(
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Core().V1().ReplicationControllers(),
		ctx.ClientBuilder.ClientOrDie("replication-controller"),
		replicationcontroller.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicationController.ConcurrentRCSyncs), ctx.Stop)
	return nil, true, nil
}
```

diaoyong了NewReplicationManager

```
// NewReplicationManager configures a replication manager with the specified event recorder
func NewReplicationManager(podInformer coreinformers.PodInformer, rcInformer coreinformers.ReplicationControllerInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicationManager {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return &ReplicationManager{
		*replicaset.NewBaseController(informerAdapter{rcInformer}, podInformer, clientsetAdapter{kubeClient}, burstReplicas,
			v1.SchemeGroupVersion.WithKind("ReplicationController"),
			"replication_controller",
			"replicationmanager",
			podControlAdapter{controller.RealPodControl{
				KubeClient: kubeClient,
				Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replication-controller"}),
			}},
		),
	}
}
```

以下是ReplicationManager的定义

```
// ReplicationManager is responsible for synchronizing ReplicationController objects stored
// in the system with actual running pods.
// It is actually just a wrapper around ReplicaSetController.
type ReplicationManager struct {
	replicaset.ReplicaSetController
}
```

ReplicationManager实际上是ReplicaController包了一层

## ReplicationController

ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

首先来看ReplicationController的启动

```
func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
		return nil, false, nil
	}
	go replicaset.NewReplicaSetController(
		ctx.InformerFactory.Apps().V1().ReplicaSets(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
		replicaset.BurstReplicas,
	).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
	return nil, true, nil
}
```

```
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(0)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}
```

```
// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
	gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
	if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
		ratelimiter.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
	}

	rsc := &ReplicaSetController{
		GroupVersionKind: gvk,
		kubeClient:       kubeClient,
		podControl:       podControl,
		burstReplicas:    burstReplicas,
		expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
	}

	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    rsc.addRS,
		UpdateFunc: rsc.updateRS,
		DeleteFunc: rsc.deleteRS,
	})
	rsc.rsLister = rsInformer.Lister()
	rsc.rsListerSynced = rsInformer.Informer().HasSynced

	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: rsc.addPod,
		// This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
		// overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
		// local storage, so it should be ok.
		UpdateFunc: rsc.updatePod,
		DeleteFunc: rsc.deletePod,
	})
	rsc.podLister = podInformer.Lister()
	rsc.podListerSynced = podInformer.Informer().HasSynced

	rsc.syncHandler = rsc.syncReplicaSet

	return rsc
}
```

replicaSetController 初始化完成后会调用 Run 方法启动多个 goroutine 处理 informer 队列中的事件并进行 sync 操作，kube-controller-manager 中每个 controller 的启动操作都是如下所示流程。

```
// Run begins watching and syncing.
func (rsc *ReplicaSetController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer rsc.queue.ShutDown()

	controllerName := strings.ToLower(rsc.Kind)
	klog.Infof("Starting %v controller", controllerName)
	defer klog.Infof("Shutting down %v controller", controllerName)

	if !cache.WaitForNamedCacheSync(rsc.Kind, stopCh, rsc.podListerSynced, rsc.rsListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(rsc.worker, time.Second, stopCh)
	}

	<-stopCh
}

// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem() bool {
	key, quit := rsc.queue.Get()
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

	err := rsc.syncHandler(key.(string))
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

### EventHandler

初始化 replicaSetController 时所注册的 eventHandler，replicaSetController 会 watch pod 和 replicaSet 两个对象，eventHandler 中注册了对这两种对象的 add、update、delete 三个操作。

addPod/updatePod/deletePod

```
// When a pod is created, enqueue the replica set that manages it and update its expectations.
func (rsc *ReplicaSetController) addPod(obj interface{}) {
	pod := obj.(*v1.Pod)

	if pod.DeletionTimestamp != nil {
		// on a restart of the controller manager, it's possible a new pod shows up in a state that
		// is already pending deletion. Prevent the pod from being a creation observation.
		rsc.deletePod(pod)
		return
	}

	// If it has a ControllerRef, that's all that matters.
	if controllerRef := metav1.GetControllerOf(pod); controllerRef != nil {
		rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
		if rs == nil {
			return
		}
		rsKey, err := controller.KeyFunc(rs)
		if err != nil {
			return
		}
		klog.V(4).Infof("Pod %s created: %#v.", pod.Name, pod)
		rsc.expectations.CreationObserved(rsKey)
		rsc.queue.Add(rsKey)
		return
	}

	// Otherwise, it's an orphan. Get a list of all matching ReplicaSets and sync
	// them to see if anyone wants to adopt it.
	// DO NOT observe creation because no controller should be waiting for an
	// orphan.
	rss := rsc.getPodReplicaSets(pod)
	if len(rss) == 0 {
		return
	}
	klog.V(4).Infof("Orphan Pod %s created: %#v.", pod.Name, pod)
	for _, rs := range rss {
		rsc.enqueueRS(rs)
	}
}
```

```
// When a pod is updated, figure out what replica set/s manage it and wake them
// up. If the labels of the pod have changed we need to awaken both the old
// and new replica set. old and cur must be *v1.Pod types.
func (rsc *ReplicaSetController) updatePod(old, cur interface{}) {
	curPod := cur.(*v1.Pod)
	oldPod := old.(*v1.Pod)
	if curPod.ResourceVersion == oldPod.ResourceVersion {
		// Periodic resync will send update events for all known pods.
		// Two different versions of the same pod will always have different RVs.
		return
	}

	labelChanged := !reflect.DeepEqual(curPod.Labels, oldPod.Labels)
	if curPod.DeletionTimestamp != nil {
		// when a pod is deleted gracefully it's deletion timestamp is first modified to reflect a grace period,
		// and after such time has passed, the kubelet actually deletes it from the store. We receive an update
		// for modification of the deletion timestamp and expect an rs to create more replicas asap, not wait
		// until the kubelet actually deletes the pod. This is different from the Phase of a pod changing, because
		// an rs never initiates a phase change, and so is never asleep waiting for the same.
		rsc.deletePod(curPod)
		if labelChanged {
			// we don't need to check the oldPod.DeletionTimestamp because DeletionTimestamp cannot be unset.
			rsc.deletePod(oldPod)
		}
		return
	}

	curControllerRef := metav1.GetControllerOf(curPod)
	oldControllerRef := metav1.GetControllerOf(oldPod)
	controllerRefChanged := !reflect.DeepEqual(curControllerRef, oldControllerRef)
	if controllerRefChanged && oldControllerRef != nil {
		// The ControllerRef was changed. Sync the old controller, if any.
		if rs := rsc.resolveControllerRef(oldPod.Namespace, oldControllerRef); rs != nil {
			rsc.enqueueRS(rs)
		}
	}

	// If it has a ControllerRef, that's all that matters.
	if curControllerRef != nil {
		rs := rsc.resolveControllerRef(curPod.Namespace, curControllerRef)
		if rs == nil {
			return
		}
		klog.V(4).Infof("Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		rsc.enqueueRS(rs)
		// TODO: MinReadySeconds in the Pod will generate an Available condition to be added in
		// the Pod status which in turn will trigger a requeue of the owning replica set thus
		// having its status updated with the newly available replica. For now, we can fake the
		// update by resyncing the controller MinReadySeconds after the it is requeued because
		// a Pod transitioned to Ready.
		// Note that this still suffers from #29229, we are just moving the problem one level
		// "closer" to kubelet (from the deployment to the replica set controller).
		if !podutil.IsPodReady(oldPod) && podutil.IsPodReady(curPod) && rs.Spec.MinReadySeconds > 0 {
			klog.V(2).Infof("%v %q will be enqueued after %ds for availability check", rsc.Kind, rs.Name, rs.Spec.MinReadySeconds)
			// Add a second to avoid milliseconds skew in AddAfter.
			// See https://github.com/kubernetes/kubernetes/issues/39785#issuecomment-279959133 for more info.
			rsc.enqueueRSAfter(rs, (time.Duration(rs.Spec.MinReadySeconds)*time.Second)+time.Second)
		}
		return
	}

	// Otherwise, it's an orphan. If anything changed, sync matching controllers
	// to see if anyone wants to adopt it now.
	if labelChanged || controllerRefChanged {
		rss := rsc.getPodReplicaSets(curPod)
		if len(rss) == 0 {
			return
		}
		klog.V(4).Infof("Orphan Pod %s updated, objectMeta %+v -> %+v.", curPod.Name, oldPod.ObjectMeta, curPod.ObjectMeta)
		for _, rs := range rss {
			rsc.enqueueRS(rs)
		}
	}
}
```

```
// When a pod is deleted, enqueue the replica set that manages the pod and update its expectations.
// obj could be an *v1.Pod, or a DeletionFinalStateUnknown marker item.
func (rsc *ReplicaSetController) deletePod(obj interface{}) {
	pod, ok := obj.(*v1.Pod)

	// When a delete is dropped, the relist will notice a pod in the store not
	// in the list, leading to the insertion of a tombstone object which contains
	// the deleted key/value. Note that this value might be stale. If the pod
	// changed labels the new ReplicaSet will not be woken up till the periodic resync.
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %+v", obj))
			return
		}
		pod, ok = tombstone.Obj.(*v1.Pod)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a pod %#v", obj))
			return
		}
	}

	controllerRef := metav1.GetControllerOf(pod)
	if controllerRef == nil {
		// No controller should care about orphans being deleted.
		return
	}
	rs := rsc.resolveControllerRef(pod.Namespace, controllerRef)
	if rs == nil {
		return
	}
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}
	klog.V(4).Infof("Pod %s/%s deleted through %v, timestamp %+v: %#v.", pod.Namespace, pod.Name, utilruntime.GetCaller(), pod.DeletionTimestamp, pod)
	rsc.expectations.DeletionObserved(rsKey, controller.PodKey(pod))
	rsc.queue.Add(rsKey)
}
```

AddRS/DeleteRS/UpdateRS

```
func (rsc *ReplicaSetController) addRS(obj interface{}) {
	rs := obj.(*apps.ReplicaSet)
	klog.V(4).Infof("Adding %s %s/%s", rsc.Kind, rs.Namespace, rs.Name)
	rsc.enqueueRS(rs)
}
```

```
// callback when RS is updated
func (rsc *ReplicaSetController) updateRS(old, cur interface{}) {
	oldRS := old.(*apps.ReplicaSet)
	curRS := cur.(*apps.ReplicaSet)

	// TODO: make a KEP and fix informers to always call the delete event handler on re-create
	if curRS.UID != oldRS.UID {
		key, err := controller.KeyFunc(oldRS)
		if err != nil {
			utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", oldRS, err))
			return
		}
		rsc.deleteRS(cache.DeletedFinalStateUnknown{
			Key: key,
			Obj: oldRS,
		})
	}

	// You might imagine that we only really need to enqueue the
	// replica set when Spec changes, but it is safer to sync any
	// time this function is triggered. That way a full informer
	// resync can requeue any replica set that don't yet have pods
	// but whose last attempts at creating a pod have failed (since
	// we don't block on creation of pods) instead of those
	// replica sets stalling indefinitely. Enqueueing every time
	// does result in some spurious syncs (like when Status.Replica
	// is updated and the watch notification from it retriggers
	// this function), but in general extra resyncs shouldn't be
	// that bad as ReplicaSets that haven't met expectations yet won't
	// sync, and all the listing is done using local stores.
	if *(oldRS.Spec.Replicas) != *(curRS.Spec.Replicas) {
		klog.V(4).Infof("%v %v updated. Desired pod count change: %d->%d", rsc.Kind, curRS.Name, *(oldRS.Spec.Replicas), *(curRS.Spec.Replicas))
	}
	rsc.enqueueRS(curRS)
}
```

```
func (rsc *ReplicaSetController) deleteRS(obj interface{}) {
	rs, ok := obj.(*apps.ReplicaSet)
	if !ok {
		tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("couldn't get object from tombstone %#v", obj))
			return
		}
		rs, ok = tombstone.Obj.(*apps.ReplicaSet)
		if !ok {
			utilruntime.HandleError(fmt.Errorf("tombstone contained object that is not a ReplicaSet %#v", obj))
			return
		}
	}

	key, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for object %#v: %v", rs, err))
		return
	}

	klog.V(4).Infof("Deleting %s %q", rsc.Kind, key)

	// Delete expectations for the ReplicaSet so if we create a new one with the same name it starts clean
	rsc.expectations.DeleteExpectations(key)

	rsc.queue.Add(key)
}
```

#### expectations 机制

通过上面的分析可知，在 rs 每次入队后进行 sync 操作时，首先需要判断该 rs 是否满足 expectations 机制，那么这个 expectations 的目的是什么？其实，rs 除了有 informer 的缓存外，还有一个本地缓存就是 expectations，expectations 会记录 rs 所有对象需要 add/del 的 pod 数量，若两者都为 0 则说明该 rs 所期望创建的 pod 或者删除的 pod 数已经被满足，若不满足则说明某次在 syncLoop 中创建或者删除 pod 时有失败的操作，则需要等待 expectations 过期后再次同步该 rs。

通过上面对 eventHandler 的分析，再来总结一下触发 replicaSet 对象发生同步事件的条件：

1、与 rs 相关的：AddRS、UpdateRS、DeleteRS；
2、与 pod 相关的：AddPod、UpdatePod、DeletePod；
3、informer 二级缓存的同步；
但是所有的更新事件是否都需要执行 sync 操作？对于除 rs.Spec.Replicas 之外的更新操作其实都没必要执行 sync 操作，因为 spec 其他字段和 status 的更新都不需要创建或者删除 pod。

在 sync 操作真正开始之前，依据 expectations 机制进行判断，确定是否要真正地启动一次 sync，因为在 eventHandler 阶段也会更新 expectations 值，从上面的 eventHandler 中可以看到在 addPod 中会调用 rsc.expectations.CreationObserved 更新 rsKey 的 expectations，将其 add 值 -1，在 deletePod 中调用 rsc.expectations.DeletionObserved 将其 del 值 -1。所以等到 sync 时，若 controllerKey(name 或者 ns/name)满足 expectations 机制则进行 sync 操作，而 updatePod 并不会修改 expectations，所以，expectations 的设计就是当需要创建或删除 pod 才会触发对应的 sync 操作，expectations 机制的目的就是减少不必要的 sync 操作。

什么条件下 expectations 机制会满足？

1、当 expectations 中不存在 rsKey 时，也就说首次创建 rs 时；
2、当 expectations 中 del 以及 add 值都为 0 时，即 rs 所需要创建或者删除的 pod 数都已满足；
3、当 expectations 过期时，即超过 5 分钟未进行 sync 操作；
最后再看一下 expectations 中用到的几个方法：

```
// ControllerExpectationsInterface is an interface that allows users to set and wait on expectations.
// Only abstracted out for testing.
// Warning: if using KeyFunc it is not safe to use a single ControllerExpectationsInterface with different
// types of controllers, because the keys might conflict across types.
type ControllerExpectationsInterface interface {
	GetExpectations(controllerKey string) (*ControlleeExpectations, bool, error)
	SatisfiedExpectations(controllerKey string) bool
	DeleteExpectations(controllerKey string)
	SetExpectations(controllerKey string, add, del int) error
	ExpectCreations(controllerKey string, adds int) error
	ExpectDeletions(controllerKey string, dels int) error
	CreationObserved(controllerKey string)
	DeletionObserved(controllerKey string)
	RaiseExpectations(controllerKey string, add, del int)
	LowerExpectations(controllerKey string, add, del int)
}
```

### syncReplicaSet

syncReplicaSet 是 controller 的核心方法，它会驱动 controller 所控制的对象达到期望状态，主要逻辑如下所示：

1、根据 ns/name 获取 rs 对象；
2、调用 expectations.SatisfiedExpectations 判断是否需要执行真正的 sync 操作；
3、获取所有 pod list；
4、根据 pod label 进行过滤获取与该 rs 关联的 pod 列表，对于其中的孤儿 pod 若与该 rs label 匹配则进行关联，若已关联的 pod 与 rs label 不匹配则解除关联关系；
5、调用 manageReplicas 进行同步 pod 操作，add/del pod；
6、计算 rs 当前的 status 并进行更新；
7、若 rs 设置了 MinReadySeconds 字段则将该 rs 加入到延迟队列中；

```
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return err
	}

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key)
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("error converting pod selector to selector: %v", err))
		return nil
	}

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything())
	if err != nil {
		return err
	}
	// Ignore inactive pods.
	filteredPods := controller.FilterActivePods(allPods)

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first.
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}

	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil {
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs)
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr)

	// Always updates status as pods come up or die.
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.queue.AddAfter(key, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}

```

#### manageReplicas

manageReplicas 是最核心的方法，它会计算 replicaSet 需要创建或者删除多少个 pod 并调用 apiserver 的接口进行操作，在此阶段仅仅是调用 apiserver 的接口进行创建，并不保证 pod 成功运行，如果在某一轮，未能成功创建的所有 Pod 对象，则不再创建剩余的 pod。一个周期内最多只能创建或删除 500 个 pod，若超过上限值未创建完成的 pod 数会在下一个 syncLoop 继续进行处理。

该方法主要逻辑如下所示：

1、计算已存在 pod 数与期望数的差异；
2、如果 diff < 0 说明 rs 实际的 pod 数未达到期望值需要继续创建 pod，首先会将需要创建的 pod 数在 expectations 中进行记录，然后调用 slowStartBatch 创建所需要的 pod，slowStartBatch 以指数级增长的方式批量创建 pod，创建 pod 过程中若出现 timeout err 则忽略，若为其他 err 则终止创建操作并更新 expectations；
3、如果 diff > 0 说明可能是一次缩容操作需要删除多余的 pod，如果需要删除全部的 pod 则直接进行删除，否则会通过 getPodsToDelete 方法筛选出需要删除的 pod，具体的筛选策略在下文会将到，然后并发删除这些 pod，对于删除失败操作也会记录在 expectations 中；
在 slowStartBatch 中会调用 rsc.podControl.CreatePodsWithControllerRef 方法创建 pod，若创建 pod 失败会判断是否为创建超时错误，或者可能是超时后失败，但此时认为超时并不影响后续的批量创建动作，大家知道，创建 pod 操作提交到 apiserver 后会经过认证、鉴权、以及动态访问控制三个步骤，此过程有可能会超时，即使真的创建失败了，等到 expectations 过期后在下一个 syncLoop 时会重新创建。

```
// manageReplicas checks and updates replicas for the given ReplicaSet.
// Does NOT modify <filteredPods>.
// It will requeue the replica set in case of an error while creating/deleting pods.
func (rsc *ReplicaSetController) manageReplicas(filteredPods []*v1.Pod, rs *apps.ReplicaSet) error {
	diff := len(filteredPods) - int(*(rs.Spec.Replicas))
	rsKey, err := controller.KeyFunc(rs)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("couldn't get key for %v %#v: %v", rsc.Kind, rs, err))
		return nil
	}
	if diff < 0 {
		diff *= -1
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		// TODO: Track UIDs of creates just like deletes. The problem currently
		// is we'd need to wait on the result of a create to record the pod's
		// UID, which would require locking *across* the create, which will turn
		// into a performance bottleneck. We should generate a UID for the pod
		// beforehand and store it via ExpectCreations.
		rsc.expectations.ExpectCreations(rsKey, diff)
		klog.V(2).InfoS("Too few replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "creating", diff)
		// Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
		// and double with each successful iteration in a kind of "slow start".
		// This handles attempts to start large numbers of pods that would
		// likely all fail with the same error. For example a project with a
		// low quota that attempts to create a large number of pods will be
		// prevented from spamming the API service with the pod create requests
		// after one of its pods fails.  Conveniently, this also prevents the
		// event spam that those failures would generate.
		successfulCreations, err := slowStartBatch(diff, controller.SlowStartInitialBatchSize, func() error {
			err := rsc.podControl.CreatePodsWithControllerRef(rs.Namespace, &rs.Spec.Template, rs, metav1.NewControllerRef(rs, rsc.GroupVersionKind))
			if err != nil {
				if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
					// if the namespace is being terminated, we don't have to do
					// anything because any creation will fail
					return nil
				}
			}
			return err
		})

		// Any skipped pods that we never attempted to start shouldn't be expected.
		// The skipped pods will be retried later. The next controller resync will
		// retry the slow start process.
		if skippedPods := diff - successfulCreations; skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for %v %v/%v", skippedPods, rsc.Kind, rs.Namespace, rs.Name)
			for i := 0; i < skippedPods; i++ {
				// Decrement the expected number of creates because the informer won't observe this pod
				rsc.expectations.CreationObserved(rsKey)
			}
		}
		return err
	} else if diff > 0 {
		if diff > rsc.burstReplicas {
			diff = rsc.burstReplicas
		}
		klog.V(2).InfoS("Too many replicas", "replicaSet", klog.KObj(rs), "need", *(rs.Spec.Replicas), "deleting", diff)

		relatedPods, err := rsc.getIndirectlyRelatedPods(rs)
		utilruntime.HandleError(err)

		// Choose which Pods to delete, preferring those in earlier phases of startup.
		podsToDelete := getPodsToDelete(filteredPods, relatedPods, diff)

		// Snapshot the UIDs (ns/name) of the pods we're expecting to see
		// deleted, so we know to record their expectations exactly once either
		// when we see it as an update of the deletion timestamp, or as a delete.
		// Note that if the labels on a pod/rs change in a way that the pod gets
		// orphaned, the rs will only wake up after the expectations have
		// expired even if other pods are deleted.
		rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))

		errCh := make(chan error, diff)
		var wg sync.WaitGroup
		wg.Add(diff)
		for _, pod := range podsToDelete {
			go func(targetPod *v1.Pod) {
				defer wg.Done()
				if err := rsc.podControl.DeletePod(rs.Namespace, targetPod.Name, rs); err != nil {
					// Decrement the expected number of deletes because the informer won't observe this deletion
					podKey := controller.PodKey(targetPod)
					rsc.expectations.DeletionObserved(rsKey, podKey)
					if !apierrors.IsNotFound(err) {
						klog.V(2).Infof("Failed to delete %v, decremented expectations for %v %s/%s", podKey, rsc.Kind, rs.Namespace, rs.Name)
						errCh <- err
					}
				}
			}(pod)
		}
		wg.Wait()

		select {
		case err := <-errCh:
			// all errors have been reported before and they're likely to be the same, so we'll only return the first one we hit.
			if err != nil {
				return err
			}
		default:
		}
	}

	return nil
}
```

```
// slowStartBatch tries to call the provided function a total of 'count' times,
// starting slow to check for errors, then speeding up if calls succeed.
//
// It groups the calls into batches, starting with a group of initialBatchSize.
// Within each batch, it may call the function multiple times concurrently.
//
// If a whole batch succeeds, the next batch may get exponentially larger.
// If there are any failures in a batch, all remaining batches are skipped
// after waiting for the current batch to complete.
//
// It returns the number of successful calls to the function.
func slowStartBatch(count int, initialBatchSize int, fn func() error) (int, error) {
	remaining := count
	successes := 0
	for batchSize := integer.IntMin(remaining, initialBatchSize); batchSize > 0; batchSize = integer.IntMin(2*batchSize, remaining) {
		errCh := make(chan error, batchSize)
		var wg sync.WaitGroup
		wg.Add(batchSize)
		for i := 0; i < batchSize; i++ {
			go func() {
				defer wg.Done()
				if err := fn(); err != nil {
					errCh <- err
				}
			}()
		}
		wg.Wait()
		curSuccesses := batchSize - len(errCh)
		successes += curSuccesses
		if len(errCh) > 0 {
			return successes, <-errCh
		}
		remaining -= batchSize
	}
	return successes, nil
}
```

若 diff > 0 时再删除 pod 阶段会调用getPodsToDelete 对 pod 进行筛选操作，此阶段会选出最劣质的 pod，下面是用到的 6 种筛选方法：

1、判断是否绑定了 node：Unassigned < assigned；
2、判断 pod phase：PodPending < PodUnknown < PodRunning；
3、判断 pod 状态：Not ready < ready；
4、若 pod 都为 ready，则按运行时间排序，运行时间最短会被删除：empty time < less time < more time；
5、根据 pod 重启次数排序：higher restart counts < lower restart counts；
6、按 pod 创建时间进行排序：Empty creation time pods < newer pods < older pods；
上面的几个排序规则遵循互斥原则，从上到下进行匹配，符合条件则排序完成，代码如下所示：

```
func getPodsToDelete(filteredPods, relatedPods []*v1.Pod, diff int) []*v1.Pod {
	// No need to sort pods if we are about to delete all of them.
	// diff will always be <= len(filteredPods), so not need to handle > case.
	if diff < len(filteredPods) {
		podsWithRanks := getPodsRankedByRelatedPodsOnSameNode(filteredPods, relatedPods)
		sort.Sort(podsWithRanks)
	}
	return filteredPods[:diff]
}
```

```
// getPodsRankedByRelatedPodsOnSameNode returns an ActivePodsWithRanks value
// that wraps podsToRank and assigns each pod a rank equal to the number of
// active pods in relatedPods that are colocated on the same node with the pod.
// relatedPods generally should be a superset of podsToRank.
func getPodsRankedByRelatedPodsOnSameNode(podsToRank, relatedPods []*v1.Pod) controller.ActivePodsWithRanks {
	podsOnNode := make(map[string]int)
	for _, pod := range relatedPods {
		if controller.IsPodActive(pod) {
			podsOnNode[pod.Spec.NodeName]++
		}
	}
	ranks := make([]int, len(podsToRank))
	for i, pod := range podsToRank {
		ranks[i] = podsOnNode[pod.Spec.NodeName]
	}
	return controller.ActivePodsWithRanks{Pods: podsToRank, Rank: ranks}
}
```

```
// ActivePodsWithRanks is a sortable list of pods and a list of corresponding
// ranks which will be considered during sorting.  The two lists must have equal
// length.  After sorting, the pods will be ordered as follows, applying each
// rule in turn until one matches:
//
// 1. If only one of the pods is assigned to a node, the pod that is not
//    assigned comes before the pod that is.
// 2. If the pods' phases differ, a pending pod comes before a pod whose phase
//    is unknown, and a pod whose phase is unknown comes before a running pod.
// 3. If exactly one of the pods is ready, the pod that is not ready comes
//    before the ready pod.
// 4. If the pods' ranks differ, the pod with greater rank comes before the pod
//    with lower rank.
// 5. If both pods are ready but have not been ready for the same amount of
//    time, the pod that has been ready for a shorter amount of time comes
//    before the pod that has been ready for longer.
// 6. If one pod has a container that has restarted more than any container in
//    the other pod, the pod with the container with more restarts comes
//    before the other pod.
// 7. If the pods' creation times differ, the pod that was created more recently
//    comes before the older pod.
//
// If none of these rules matches, the second pod comes before the first pod.
//
// The intention of this ordering is to put pods that should be preferred for
// deletion first in the list.
type ActivePodsWithRanks struct {
	// Pods is a list of pods.
	Pods []*v1.Pod

	// Rank is a ranking of pods.  This ranking is used during sorting when
	// comparing two pods that are both scheduled, in the same phase, and
	// having the same ready status.
	Rank []int
}

func (s ActivePodsWithRanks) Len() int {
	return len(s.Pods)
}

func (s ActivePodsWithRanks) Swap(i, j int) {
	s.Pods[i], s.Pods[j] = s.Pods[j], s.Pods[i]
	s.Rank[i], s.Rank[j] = s.Rank[j], s.Rank[i]
}

// Less compares two pods with corresponding ranks and returns true if the first
// one should be preferred for deletion.
func (s ActivePodsWithRanks) Less(i, j int) bool {
	// 1. Unassigned < assigned
	// If only one of the pods is unassigned, the unassigned one is smaller
	if s.Pods[i].Spec.NodeName != s.Pods[j].Spec.NodeName && (len(s.Pods[i].Spec.NodeName) == 0 || len(s.Pods[j].Spec.NodeName) == 0) {
		return len(s.Pods[i].Spec.NodeName) == 0
	}
	// 2. PodPending < PodUnknown < PodRunning
	if podPhaseToOrdinal[s.Pods[i].Status.Phase] != podPhaseToOrdinal[s.Pods[j].Status.Phase] {
		return podPhaseToOrdinal[s.Pods[i].Status.Phase] < podPhaseToOrdinal[s.Pods[j].Status.Phase]
	}
	// 3. Not ready < ready
	// If only one of the pods is not ready, the not ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) != podutil.IsPodReady(s.Pods[j]) {
		return !podutil.IsPodReady(s.Pods[i])
	}
	// 4. Doubled up < not doubled up
	// If one of the two pods is on the same node as one or more additional
	// ready pods that belong to the same replicaset, whichever pod has more
	// colocated ready pods is less
	if s.Rank[i] != s.Rank[j] {
		return s.Rank[i] > s.Rank[j]
	}
	// TODO: take availability into account when we push minReadySeconds information from deployment into pods,
	//       see https://github.com/kubernetes/kubernetes/issues/22065
	// 5. Been ready for empty time < less time < more time
	// If both pods are ready, the latest ready one is smaller
	if podutil.IsPodReady(s.Pods[i]) && podutil.IsPodReady(s.Pods[j]) {
		readyTime1 := podReadyTime(s.Pods[i])
		readyTime2 := podReadyTime(s.Pods[j])
		if !readyTime1.Equal(readyTime2) {
			return afterOrZero(readyTime1, readyTime2)
		}
	}
	// 6. Pods with containers with higher restart counts < lower restart counts
	if maxContainerRestarts(s.Pods[i]) != maxContainerRestarts(s.Pods[j]) {
		return maxContainerRestarts(s.Pods[i]) > maxContainerRestarts(s.Pods[j])
	}
	// 7. Empty creation time pods < newer pods < older pods
	if !s.Pods[i].CreationTimestamp.Equal(&s.Pods[j].CreationTimestamp) {
		return afterOrZero(&s.Pods[i].CreationTimestamp, &s.Pods[j].CreationTimestamp)
	}
	return false
}
```
