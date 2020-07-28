# daemonset controller 源码分析

在前面的文章中已经分析过 deployment、statefulset 两个重要对象了，本文会继续分析 kubernetes 中另一个重要的对象 daemonset，在 kubernetes 中 daemonset 类似于 linux 上的守护进程会运行在每一个 node 上，在实际场景中，一般会将日志采集或者网络插件采用 daemonset 的方式部署。

## DaemonSet 的基本操作

### 创建

daemonset 在创建后会在每个 node 上都启动一个 pod。

```
$ kubectl create -f nginx-ds.yaml
```

### 扩缩容

由于 daemonset 是在每个 node 上启动一个 pod，其不存在扩缩容操作，副本数量跟 node 数量保持一致。

### 更新

daemonset 有两种更新策略 OnDelete 和 RollingUpdate，默认为 RollingUpdate。滚动更新时，需要指定 .spec.updateStrategy.rollingUpdate.maxUnavailable（默认为1）和 .spec.minReadySeconds（默认为 0）。

```
// 更新镜像
$ kubectl set image ds/nginx-ds nginx-ds=nginx:1.16

// 查看更新状态
$ kubectl rollout status ds/nginx-ds
```

### 回滚

在 statefulset 源码分析一节已经提到过 controllerRevision 这个对象了，其主要用来保存历史版本信息，在更新以及回滚操作时使用，daemonset controller 也是使用 controllerrevision 保存历史版本信息，在回滚时会使用历史 controllerrevision 中的信息替换 daemonset 中 Spec.Template。

```
// 查看 ds 历史版本信息
$ kubectl get controllerrevision
NAME                  CONTROLLER                REVISION   AGE
nginx-ds-5c4b75bdbb   daemonset.apps/nginx-ds   2          122m
nginx-ds-7cd7798dcd   daemonset.apps/nginx-ds   1          133m

// 回滚到版本 1
$ kubectl rollout undo daemonset nginx-ds --to-revision=1

// 查看回滚状态
$ kubectl rollout status ds/nginx-ds
```

### 暂停

daemonset 目前不支持暂停操作。

### 删除

daemonset 也支持两种删除操作。

```
// 非级联删除
$ kubectl delete ds/nginx-ds --cascade=false

// 级联删除
$ kubectl delete ds/nginx-ds
```

## DaemonSetController 源码分析

首先还是看 startDaemonSetController 方法，在此方法中会初始化 DaemonSetsController 对象并调用 Run方法启动 daemonset controller，从该方法中可以看出 daemonset controller 会监听 daemonsets、controllerRevision、pod 和 node 四种对象资源的变动。其中 ConcurrentDaemonSetSyncs的默认值为 2。

```
func startDaemonSetController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "daemonsets"}] {
		return nil, false, nil
	}
	dsc, err := daemon.NewDaemonSetsController(
		ctx.InformerFactory.Apps().V1().DaemonSets(),
		ctx.InformerFactory.Apps().V1().ControllerRevisions(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.InformerFactory.Core().V1().Nodes(),
		ctx.ClientBuilder.ClientOrDie("daemon-set-controller"),
		flowcontrol.NewBackOff(1*time.Second, 15*time.Minute),
	)
	if err != nil {
		return nil, true, fmt.Errorf("error creating DaemonSets controller: %v", err)
	}
	go dsc.Run(int(ctx.ComponentConfig.DaemonSetController.ConcurrentDaemonSetSyncs), ctx.Stop)
	return nil, true, nil
}
```

在 Run 方法中会启动两个操作，一个就是 dsc.runWorker 执行的 sync 操作，另一个就是 dsc.failedPodsBackoff.GC 执行的 gc 操作，主要逻辑为：

1、等待 informer 缓存同步完成；
2、启动两个 goroutine 分别执行 dsc.runWorker；
3、启动一个 goroutine 每分钟执行一次 dsc.failedPodsBackoff.GC，从 startDaemonSetController 方法中可以看到 failedPodsBackoff 的 duration为1s，max duration为15m，failedPodsBackoff 的主要作用是当发现 daemon pod 状态为 failed 时，会定时重启该 pod；

```
func (dsc *DaemonSetsController) Run(workers int, stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()
	defer dsc.queue.ShutDown()

	klog.Infof("Starting daemon sets controller")
	defer klog.Infof("Shutting down daemon sets controller")

	if !cache.WaitForNamedCacheSync("daemon sets", stopCh, dsc.podStoreSynced, dsc.nodeStoreSynced, dsc.historyStoreSynced, dsc.dsStoreSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.Until(dsc.runWorker, time.Second, stopCh)
	}

	go wait.Until(dsc.failedPodsBackoff.GC, BackoffGCInterval, stopCh)

	<-stopCh
}
```

### syncDaemonSet

daemonset 中 pod 的创建与删除是与 node 相关联的，所以每次执行 sync 操作时需要遍历所有的 node 进行判断。syncDaemonSet 的主要逻辑为：

1、通过 key 获取 ns 和 name；
2、从 dsLister 中获取 ds 对象；
3、从 nodeLister 获取所有 node；
4、获取 dsKey；
5、判断 ds 是否处于删除状态；
6、调用 constructHistory 获取 current 和 old controllerRevision；
7、调用 dsc.expectations.SatisfiedExpectations 判断是否满足 expectations 机制，expectations 机制的目的就是减少不必要的 sync 操作，关于 expectations 机制的详细说明可以参考笔者以前写的 "replicaset controller 源码分析"一文；
8、调用 dsc.manage 执行实际的 sync 操作；
9、判断是否为更新操作，并执行对应的更新操作逻辑；
10、调用 dsc.cleanupHistory 根据 spec.revisionHistoryLimit字段清理过期的 controllerrevision；
11、调用 dsc.updateDaemonSetStatus 更新 ds 状态；

```
func (dsc *DaemonSetsController) syncDaemonSet(key string) error {
	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing daemon set %q (%v)", key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	ds, err := dsc.dsLister.DaemonSets(namespace).Get(name)
	if errors.IsNotFound(err) {
		klog.V(3).Infof("daemon set has been deleted %v", key)
		dsc.expectations.DeleteExpectations(key)
		return nil
	}
	if err != nil {
		return fmt.Errorf("unable to retrieve ds %v from store: %v", key, err)
	}

	nodeList, err := dsc.nodeLister.List(labels.Everything())
	if err != nil {
		return fmt.Errorf("couldn't get list of nodes when syncing daemon set %#v: %v", ds, err)
	}

	everything := metav1.LabelSelector{}
	if reflect.DeepEqual(ds.Spec.Selector, &everything) {
		dsc.eventRecorder.Eventf(ds, v1.EventTypeWarning, SelectingAllReason, "This daemon set is selecting all pods. A non-empty selector is required.")
		return nil
	}

	// Don't process a daemon set until all its creations and deletions have been processed.
	// For example if daemon set foo asked for 3 new daemon pods in the previous call to manage,
	// then we do not want to call manage on foo until the daemon pods have been created.
	dsKey, err := controller.KeyFunc(ds)
	if err != nil {
		return fmt.Errorf("couldn't get key for object %#v: %v", ds, err)
	}

	// If the DaemonSet is being deleted (either by foreground deletion or
	// orphan deletion), we cannot be sure if the DaemonSet history objects
	// it owned still exist -- those history objects can either be deleted
	// or orphaned. Garbage collector doesn't guarantee that it will delete
	// DaemonSet pods before deleting DaemonSet history objects, because
	// DaemonSet history doesn't own DaemonSet pods. We cannot reliably
	// calculate the status of a DaemonSet being deleted. Therefore, return
	// here without updating status for the DaemonSet being deleted.
	if ds.DeletionTimestamp != nil {
		return nil
	}

	// Construct histories of the DaemonSet, and get the hash of current history
	cur, old, err := dsc.constructHistory(ds)
	if err != nil {
		return fmt.Errorf("failed to construct revisions of DaemonSet: %v", err)
	}
	hash := cur.Labels[apps.DefaultDaemonSetUniqueLabelKey]

	if !dsc.expectations.SatisfiedExpectations(dsKey) {
		// Only update status. Don't raise observedGeneration since controller didn't process object of that generation.
		return dsc.updateDaemonSetStatus(ds, nodeList, hash, false)
	}

	err = dsc.manage(ds, nodeList, hash)
	if err != nil {
		return err
	}

	// Process rolling updates if we're ready.
	if dsc.expectations.SatisfiedExpectations(dsKey) {
		switch ds.Spec.UpdateStrategy.Type {
		case apps.OnDeleteDaemonSetStrategyType:
		case apps.RollingUpdateDaemonSetStrategyType:
			err = dsc.rollingUpdate(ds, nodeList, hash)
		}
		if err != nil {
			return err
		}
	}

	err = dsc.cleanupHistory(ds, old)
	if err != nil {
		return fmt.Errorf("failed to clean up revisions of DaemonSet: %v", err)
	}

	return dsc.updateDaemonSetStatus(ds, nodeList, hash, true)
}
```

syncDaemonSet 中主要有 manage、rollingUpdate和updateDaemonSetStatus 三个方法，分别对应创建、更新与状态同步，下面主要来分析这三个方法。

### manage

manage 主要是用来保证 ds 的 pod 数正常运行在每一个 node 上，其主要逻辑为：

1、调用 dsc.getNodesToDaemonPods 获取已存在 daemon pod 与 node 的映射关系；
2、遍历所有 node，调用 dsc.podsShouldBeOnNode 方法来确定在给定的节点上需要创建还是删除 daemon pod；
3、判断是否启动了 ScheduleDaemonSetPodsfeature-gates 特性，若启动了则需要删除通过默认调度器已经调度到不存在 node 上的 daemon pod；
4、调用 dsc.syncNodes 为对应的 node 创建 daemon pod 以及删除多余的 pods；

```
func (dsc *DaemonSetsController) manage(ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
	// Find out the pods which are created for the nodes by DaemonSet.
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	// For each node, if the node is running the daemon pod but isn't supposed to, kill the daemon
	// pod. If the node is supposed to run the daemon pod, but isn't, create the daemon pod on the node.
	var nodesNeedingDaemonPods, podsToDelete []string
	for _, node := range nodeList {
		nodesNeedingDaemonPodsOnNode, podsToDeleteOnNode, err := dsc.podsShouldBeOnNode(
			node, nodeToDaemonPods, ds)

		if err != nil {
			continue
		}

		nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, nodesNeedingDaemonPodsOnNode...)
		podsToDelete = append(podsToDelete, podsToDeleteOnNode...)
	}

	// Remove unscheduled pods assigned to not existing nodes when daemonset pods are scheduled by scheduler.
	// If node doesn't exist then pods are never scheduled and can't be deleted by PodGCController.
	podsToDelete = append(podsToDelete, getUnscheduledPodsWithoutNode(nodeList, nodeToDaemonPods)...)

	// Label new pods using the hash label value of the current history when creating them
	if err = dsc.syncNodes(ds, podsToDelete, nodesNeedingDaemonPods, hash); err != nil {
		return err
	}

	return nil
}
```

在 manage 方法中又调用了 getNodesToDaemonPods、podsShouldBeOnNode 和 syncNodes 三个方法，继续来看这几种方法的作用。

#### getNodesToDaemonPods

getNodesToDaemonPods 是用来获取已存在 daemon pod 与 node 的映射关系，并且会通过 adopt/orphan 方法关联以及释放对应的 pod。

```
func (dsc *DaemonSetsController) getNodesToDaemonPods(ds *apps.DaemonSet) (map[string][]*v1.Pod, error) {
	claimedPods, err := dsc.getDaemonPods(ds)
	if err != nil {
		return nil, err
	}
	// Group Pods by Node name.
	nodeToDaemonPods := make(map[string][]*v1.Pod)
	for _, pod := range claimedPods {
		nodeName, err := util.GetTargetNodeName(pod)
		if err != nil {
			klog.Warningf("Failed to get target node name of Pod %v/%v in DaemonSet %v/%v",
				pod.Namespace, pod.Name, ds.Namespace, ds.Name)
			continue
		}

		nodeToDaemonPods[nodeName] = append(nodeToDaemonPods[nodeName], pod)
	}

	return nodeToDaemonPods, nil
}
```

#### podsShouldBeOnNode

podsShouldBeOnNode 方法用来确定在给定的节点上需要创建还是删除 daemon pod，主要逻辑为：

1、调用 dsc.nodeShouldRunDaemonPod 判断该 node 是否需要运行 daemon pod 以及 pod 能不能调度成功，该方法返回三个值 wantToRun, shouldSchedule, shouldContinueRunning；
2、通过判断 wantToRun, shouldSchedule, shouldContinueRunning 将需要创建 daemon pod 的 node 列表以及需要删除的 pod 列表获取到， wantToRun主要检查的是 selector、taints 等是否匹配，shouldSchedule 主要检查 node 上的资源是否充足，shouldContinueRunning 默认为 true；

```
func (dsc *DaemonSetsController) podsShouldBeOnNode(
	node *v1.Node,
	nodeToDaemonPods map[string][]*v1.Pod,
	ds *apps.DaemonSet,
) (nodesNeedingDaemonPods, podsToDelete []string, err error) {

	shouldRun, shouldContinueRunning, err := dsc.nodeShouldRunDaemonPod(node, ds)
	if err != nil {
		return
	}

	daemonPods, exists := nodeToDaemonPods[node.Name]

	switch {
	case shouldRun && !exists:
		// If daemon pod is supposed to be running on node, but isn't, create daemon pod.
		nodesNeedingDaemonPods = append(nodesNeedingDaemonPods, node.Name)
	case shouldContinueRunning:
		// If a daemon pod failed, delete it
		// If there's non-daemon pods left on this node, we will create it in the next sync loop
		var daemonPodsRunning []*v1.Pod
		for _, pod := range daemonPods {
			if pod.DeletionTimestamp != nil {
				continue
			}
			if pod.Status.Phase == v1.PodFailed {
				// This is a critical place where DS is often fighting with kubelet that rejects pods.
				// We need to avoid hot looping and backoff.
				backoffKey := failedPodsBackoffKey(ds, node.Name)

				now := dsc.failedPodsBackoff.Clock.Now()
				inBackoff := dsc.failedPodsBackoff.IsInBackOffSinceUpdate(backoffKey, now)
				if inBackoff {
					delay := dsc.failedPodsBackoff.Get(backoffKey)
					klog.V(4).Infof("Deleting failed pod %s/%s on node %s has been limited by backoff - %v remaining",
						pod.Namespace, pod.Name, node.Name, delay)
					dsc.enqueueDaemonSetAfter(ds, delay)
					continue
				}

				dsc.failedPodsBackoff.Next(backoffKey, now)

				msg := fmt.Sprintf("Found failed daemon pod %s/%s on node %s, will try to kill it", pod.Namespace, pod.Name, node.Name)
				klog.V(2).Infof(msg)
				// Emit an event so that it's discoverable to users.
				dsc.eventRecorder.Eventf(ds, v1.EventTypeWarning, FailedDaemonPodReason, msg)
				podsToDelete = append(podsToDelete, pod.Name)
			} else {
				daemonPodsRunning = append(daemonPodsRunning, pod)
			}
		}
		// If daemon pod is supposed to be running on node, but more than 1 daemon pod is running, delete the excess daemon pods.
		// Sort the daemon pods by creation time, so the oldest is preserved.
		if len(daemonPodsRunning) > 1 {
			sort.Sort(podByCreationTimestampAndPhase(daemonPodsRunning))
			for i := 1; i < len(daemonPodsRunning); i++ {
				podsToDelete = append(podsToDelete, daemonPodsRunning[i].Name)
			}
		}
	case !shouldContinueRunning && exists:
		// If daemon pod isn't supposed to run on node, but it is, delete all daemon pods on node.
		for _, pod := range daemonPods {
			if pod.DeletionTimestamp != nil {
				continue
			}
			podsToDelete = append(podsToDelete, pod.Name)
		}
	}

	return nodesNeedingDaemonPods, podsToDelete, nil
}
```

然后继续看 nodeShouldRunDaemonPod 方法的主要逻辑：

1、调用 NewPod 为该 node 构建一个 daemon pod object；
2、判断 ds 是否指定了 .Spec.Template.Spec.NodeName 字段；
3、调用 dsc.simulate 执行 GeneralPredicates 预选算法检查该 node 是否能够调度成功；
4、判断 GeneralPredicates 预选算法执行后的 reasons 确定 wantToRun, shouldSchedule, shouldContinueRunning 的值；

```
func (dsc *DaemonSetsController) nodeShouldRunDaemonPod(node *v1.Node, ds *apps.DaemonSet) (bool, bool, error) {
	pod := NewPod(ds, node.Name)

	// If the daemon set specifies a node name, check that it matches with node.Name.
	if !(ds.Spec.Template.Spec.NodeName == "" || ds.Spec.Template.Spec.NodeName == node.Name) {
		return false, false, nil
	}

	taints := node.Spec.Taints
	fitsNodeName, fitsNodeAffinity, fitsTaints := Predicates(pod, node, taints)
	if !fitsNodeName || !fitsNodeAffinity {
		return false, false, nil
	}

	if !fitsTaints {
		// Scheduled daemon pods should continue running if they tolerate NoExecute taint.
		shouldContinueRunning := v1helper.TolerationsTolerateTaintsWithFilter(pod.Spec.Tolerations, taints, func(t *v1.Taint) bool {
			return t.Effect == v1.TaintEffectNoExecute
		})
		return false, shouldContinueRunning, nil
	}

	return true, true, nil
}
```

#### syncNodes

syncNodes 方法主要是为需要 daemon pod 的 node 创建 pod 以及删除多余的 pod，其主要逻辑为：

1、将 createDiff 和 deleteDiff 与 burstReplicas 进行比较，burstReplicas 默认值为 250 即每个 syncLoop 中创建或者删除的 pod 数最多为 250 个，若超过其值则剩余需要创建或者删除的 pod 在下一个 syncLoop 继续操作；
2、将 createDiff 和 deleteDiff 写入到 expectations 中；
3、并发创建 pod，创建 pod 有两种方法:（1）创建的 pod 不经过默认调度器，直接指定了 pod 的运行节点(即设定pod.Spec.NodeName)；（2）若启用了 ScheduleDaemonSetPods feature-gates 特性，则使用默认调度器进行创建 pod，通过 nodeAffinity来保证每个节点都运行一个 pod；
4、并发删除 deleteDiff 中的所有 pod；
ScheduleDaemonSetPods 是一个 feature-gates 特性，其出现在 v1.11 中，在 v1.12 中处于 Beta 版本，v1.17 为 GA 版。最初 daemonset controller 只有一种创建 pod 的方法，即直接指定 pod 的 spec.NodeName 字段，但是目前这种方式已经暴露了许多问题，在以后的发展中社区还是希望能通过默认调度器进行调度，所以才出现了第二种方式，原因主要有以下五点：

1、DaemonSet 无法感知 node 上资源的变化 (#46935, #58868)：当 pod 第一次因资源不够无法创建时，若其他 pod 退出后资源足够时 DaemonSet 无法感知到；
2、Daemonset 无法支持 Pod Affinity 和 Pod AntiAffinity 的功能(#29276)；
3、在某些功能上需要实现和 scheduler 重复的代码逻辑, 例如：critical pods (#42028), tolerant/taint；
4、当 DaemonSet 的 Pod 创建失败时难以 debug，例如：资源不足时，对于 pending pod 最好能打一个 event 说明；
5、多个组件同时调度时难以实现抢占机制：这也是无法通过横向扩展调度器提高调度吞吐量的一个原因；
更详细的原因可以参考社区的文档：schedule-DS-pod-by-scheduler.md。

```
func (dsc *DaemonSetsController) syncNodes(ds *apps.DaemonSet, podsToDelete, nodesNeedingDaemonPods []string, hash string) error {
	// We need to set expectations before creating/deleting pods to avoid race conditions.
	dsKey, err := controller.KeyFunc(ds)
	if err != nil {
		return fmt.Errorf("couldn't get key for object %#v: %v", ds, err)
	}

	createDiff := len(nodesNeedingDaemonPods)
	deleteDiff := len(podsToDelete)

	if createDiff > dsc.burstReplicas {
		createDiff = dsc.burstReplicas
	}
	if deleteDiff > dsc.burstReplicas {
		deleteDiff = dsc.burstReplicas
	}

	dsc.expectations.SetExpectations(dsKey, createDiff, deleteDiff)

	// error channel to communicate back failures.  make the buffer big enough to avoid any blocking
	errCh := make(chan error, createDiff+deleteDiff)

	klog.V(4).Infof("Nodes needing daemon pods for daemon set %s: %+v, creating %d", ds.Name, nodesNeedingDaemonPods, createDiff)
	createWait := sync.WaitGroup{}
	// If the returned error is not nil we have a parse error.
	// The controller handles this via the hash.
	generation, err := util.GetTemplateGeneration(ds)
	if err != nil {
		generation = nil
	}
	template := util.CreatePodTemplate(ds.Spec.Template, generation, hash)
	// Batch the pod creates. Batch sizes start at SlowStartInitialBatchSize
	// and double with each successful iteration in a kind of "slow start".
	// This handles attempts to start large numbers of pods that would
	// likely all fail with the same error. For example a project with a
	// low quota that attempts to create a large number of pods will be
	// prevented from spamming the API service with the pod create requests
	// after one of its pods fails.  Conveniently, this also prevents the
	// event spam that those failures would generate.
	batchSize := integer.IntMin(createDiff, controller.SlowStartInitialBatchSize)
	for pos := 0; createDiff > pos; batchSize, pos = integer.IntMin(2*batchSize, createDiff-(pos+batchSize)), pos+batchSize {
		errorCount := len(errCh)
		createWait.Add(batchSize)
		for i := pos; i < pos+batchSize; i++ {
			go func(ix int) {
				defer createWait.Done()

				podTemplate := template.DeepCopy()
				// The pod's NodeAffinity will be updated to make sure the Pod is bound
				// to the target node by default scheduler. It is safe to do so because there
				// should be no conflicting node affinity with the target node.
				podTemplate.Spec.Affinity = util.ReplaceDaemonSetPodNodeNameNodeAffinity(
					podTemplate.Spec.Affinity, nodesNeedingDaemonPods[ix])

				err := dsc.podControl.CreatePodsWithControllerRef(ds.Namespace, podTemplate,
					ds, metav1.NewControllerRef(ds, controllerKind))

				if err != nil {
					if errors.HasStatusCause(err, v1.NamespaceTerminatingCause) {
						// If the namespace is being torn down, we can safely ignore
						// this error since all subsequent creations will fail.
						return
					}
				}
				if err != nil {
					klog.V(2).Infof("Failed creation, decrementing expectations for set %q/%q", ds.Namespace, ds.Name)
					dsc.expectations.CreationObserved(dsKey)
					errCh <- err
					utilruntime.HandleError(err)
				}
			}(i)
		}
		createWait.Wait()
		// any skipped pods that we never attempted to start shouldn't be expected.
		skippedPods := createDiff - (batchSize + pos)
		if errorCount < len(errCh) && skippedPods > 0 {
			klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods, decrementing expectations for set %q/%q", skippedPods, ds.Namespace, ds.Name)
			dsc.expectations.LowerExpectations(dsKey, skippedPods, 0)
			// The skipped pods will be retried later. The next controller resync will
			// retry the slow start process.
			break
		}
	}

	klog.V(4).Infof("Pods to delete for daemon set %s: %+v, deleting %d", ds.Name, podsToDelete, deleteDiff)
	deleteWait := sync.WaitGroup{}
	deleteWait.Add(deleteDiff)
	for i := 0; i < deleteDiff; i++ {
		go func(ix int) {
			defer deleteWait.Done()
			if err := dsc.podControl.DeletePod(ds.Namespace, podsToDelete[ix], ds); err != nil {
				klog.V(2).Infof("Failed deletion, decrementing expectations for set %q/%q", ds.Namespace, ds.Name)
				dsc.expectations.DeletionObserved(dsKey)
				errCh <- err
				utilruntime.HandleError(err)
			}
		}(i)
	}
	deleteWait.Wait()

	// collect errors if any for proper reporting/retry logic in the controller
	errors := []error{}
	close(errCh)
	for err := range errCh {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```

### RollingUpdate

daemonset update 的方式有两种 OnDelete 和 RollingUpdate，当为 OnDelete 时需要用户手动删除每一个 pod 后完成更新操作，当为 RollingUpdate 时，daemonset controller 会自动控制升级进度。

当为 RollingUpdate 时，主要逻辑为：

1、获取 daemonset pod 与 node 的映射关系；
2、根据 controllerrevision 的 hash 值获取所有未更新的 pods；
3、获取 maxUnavailable, numUnavailable 的 pod 数值，maxUnavailable 是从 ds 的 rollingUpdate 字段中获取的默认值为 1，numUnavailable 的值是通过 daemonset pod 与 node 的映射关系计算每个 node 下是否有 available pod 得到的；
4、通过 oldPods 获取 oldAvailablePods, oldUnavailablePods 的 pod 列表；
5、遍历 oldUnavailablePods 列表将需要删除的 pod 追加到 oldPodsToDelete 数组中。oldUnavailablePods 列表中的 pod 分为两种，一种处于更新中，即删除状态，一种处于未更新且异常状态，处于异常状态的都需要被删除；
6、遍历 oldAvailablePods 列表，此列表中的 pod 都处于正常运行状态，根据 maxUnavailable 值确定是否需要删除该 pod 并将需要删除的 pod 追加到 oldPodsToDelete 数组中；
7、调用 dsc.syncNodes 删除 oldPodsToDelete 数组中的 pods，syncNodes 方法在 manage 阶段已经分析过，此处不再详述；
rollingUpdate 的结果是找出需要删除的 pods 并进行删除，被删除的 pod 在下一个 syncLoop 中会通过 manage 方法使用最新版本的 daemonset template 进行创建，整个滚动更新的过程是通过先删除再创建的方式一步步完成更新的，每次操作都是严格按照 maxUnavailable 的值确定需要删除的 pod 数。

```
func (dsc *DaemonSetsController) rollingUpdate(ds *apps.DaemonSet, nodeList []*v1.Node, hash string) error {
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	_, oldPods := dsc.getAllDaemonSetPods(ds, nodeToDaemonPods, hash)
	maxUnavailable, numUnavailable, err := dsc.getUnavailableNumbers(ds, nodeList, nodeToDaemonPods)
	if err != nil {
		return fmt.Errorf("couldn't get unavailable numbers: %v", err)
	}
	oldAvailablePods, oldUnavailablePods := util.SplitByAvailablePods(ds.Spec.MinReadySeconds, oldPods)

	// for oldPods delete all not running pods
	var oldPodsToDelete []string
	klog.V(4).Infof("Marking all unavailable old pods for deletion")
	for _, pod := range oldUnavailablePods {
		// Skip terminating pods. We won't delete them again
		if pod.DeletionTimestamp != nil {
			continue
		}
		klog.V(4).Infof("Marking pod %s/%s for deletion", ds.Name, pod.Name)
		oldPodsToDelete = append(oldPodsToDelete, pod.Name)
	}

	klog.V(4).Infof("Marking old pods for deletion")
	for _, pod := range oldAvailablePods {
		if numUnavailable >= maxUnavailable {
			klog.V(4).Infof("Number of unavailable DaemonSet pods: %d, is equal to or exceeds allowed maximum: %d", numUnavailable, maxUnavailable)
			break
		}
		klog.V(4).Infof("Marking pod %s/%s for deletion", ds.Name, pod.Name)
		oldPodsToDelete = append(oldPodsToDelete, pod.Name)
		numUnavailable++
	}
	return dsc.syncNodes(ds, oldPodsToDelete, []string{}, hash)
}
```

总结一下，manage 方法中的主要流程为：

```
|->  dsc.getNodesToDaemonPods
|
|
manage ---- |->  dsc.podsShouldBeOnNode  ---> dsc.nodeShouldRunDaemonPod
|
|
|->  dsc.syncNodes
```

### updateDaemonSetStatus

updateDaemonSetStatus 是 syncDaemonSet 中最后执行的方法，主要是用来计算 ds status subresource 中的值并更新其 status。status 如下所示：

```
status:
  currentNumberScheduled: 1  // 已经运行了 DaemonSet Pod的节点数量
  desiredNumberScheduled: 1  // 需要运行该DaemonSet Pod的节点数量
  numberMisscheduled: 0      // 不需要运行 DeamonSet Pod 但是已经运行了的节点数量
  numberReady: 0             // DaemonSet Pod状态为Ready的节点数量
  numberAvailable: 1          // DaemonSet Pod状态为Ready且运行时间超过                                                                                                      // Spec.MinReadySeconds 的节点数量
  numberUnavailable: 0             // desiredNumberScheduled - numberAvailable 的节点数量
  observedGeneration: 3
  updatedNumberScheduled: 1  // 已经完成DaemonSet Pod更新的节点数量
```

updateDaemonSetStatus 主要逻辑为：

1、调用 dsc.getNodesToDaemonPods 获取已存在 daemon pod 与 node 的映射关系；
2、遍历所有 node，调用 dsc.nodeShouldRunDaemonPod 判断该 node 是否需要运行 daemon pod，然后计算 status 中的部分字段值；
3、调用 storeDaemonSetStatus 更新 ds status subresource；
4、判断 ds 是否需要 resync；

```
func (dsc *DaemonSetsController) updateDaemonSetStatus(ds *apps.DaemonSet, nodeList []*v1.Node, hash string, updateObservedGen bool) error {
	klog.V(4).Infof("Updating daemon set status")
	nodeToDaemonPods, err := dsc.getNodesToDaemonPods(ds)
	if err != nil {
		return fmt.Errorf("couldn't get node to daemon pod mapping for daemon set %q: %v", ds.Name, err)
	}

	var desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable int
	for _, node := range nodeList {
		shouldRun, _, err := dsc.nodeShouldRunDaemonPod(node, ds)
		if err != nil {
			return err
		}

		scheduled := len(nodeToDaemonPods[node.Name]) > 0

		if shouldRun {
			desiredNumberScheduled++
			if scheduled {
				currentNumberScheduled++
				// Sort the daemon pods by creation time, so that the oldest is first.
				daemonPods, _ := nodeToDaemonPods[node.Name]
				sort.Sort(podByCreationTimestampAndPhase(daemonPods))
				pod := daemonPods[0]
				if podutil.IsPodReady(pod) {
					numberReady++
					if podutil.IsPodAvailable(pod, ds.Spec.MinReadySeconds, metav1.Now()) {
						numberAvailable++
					}
				}
				// If the returned error is not nil we have a parse error.
				// The controller handles this via the hash.
				generation, err := util.GetTemplateGeneration(ds)
				if err != nil {
					generation = nil
				}
				if util.IsPodUpdated(pod, hash, generation) {
					updatedNumberScheduled++
				}
			}
		} else {
			if scheduled {
				numberMisscheduled++
			}
		}
	}
	numberUnavailable := desiredNumberScheduled - numberAvailable

	err = storeDaemonSetStatus(dsc.kubeClient.AppsV1().DaemonSets(ds.Namespace), ds, desiredNumberScheduled, currentNumberScheduled, numberMisscheduled, numberReady, updatedNumberScheduled, numberAvailable, numberUnavailable, updateObservedGen)
	if err != nil {
		return fmt.Errorf("error storing status for daemon set %#v: %v", ds, err)
	}

	// Resync the DaemonSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if ds.Spec.MinReadySeconds > 0 && numberReady != numberAvailable {
		dsc.enqueueDaemonSetAfter(ds, time.Duration(ds.Spec.MinReadySeconds)*time.Second)
	}
	return nil
}
```

最后，再总结一下 syncDaemonSet 方法的主要流程：

```
|-> dsc.getNodesToDaemonPods
|
|
|-> manage -->|-> dsc.podsShouldBeOnNode  ---> dsc.nodeShouldRunDaemonPod
|             |
|             |
syncDaemonSet --> |             |-> dsc.syncNodes
|
|-> rollingUpdate
|
|
|-> updateDaemonSetStatus
```

## 总结

在 daemonset controller 中可以看到许多功能都是 deployment 和 statefulset 已有的。在创建 pod 的流程与 replicaset controller 创建 pod 的流程是相似的，都使用了 expectations 机制并且限制了在一个 syncLoop 中最多创建或删除的 pod 数。更新方式与 statefulset 一样都有 OnDelete 和 RollingUpdate 两种， OnDelete 方式与 statefulset 相似，都需要手动删除对应的 pod，而 RollingUpdate 方式与 statefulset 和 deployment 都有点区别， RollingUpdate方式更新时不支持暂停操作并且 pod 是先删除再创建的顺序进行。版本控制方式与 statefulset 的一样都是使用 controllerRevision。最后要说的一点是在 v1.12 及以后的版本中，使用 daemonset 创建的 pod 已不再使用直接指定 .spec.nodeName的方式绕过调度器进行调度，而是走默认调度器通过 nodeAffinity 的方式调度到每一个节点上。

参考：

https://yq.aliyun.com/articles/702305
