# kube-scheduler 源码分析

## kube-scheduler 的设计

Kube-scheduler 是 kubernetes 的核心组件之一，也是所有核心组件之间功能比较单一的，其代码也相对容易理解。kube-scheduler 的目的就是为每一个 pod 选择一个合适的 node，整体流程可以概括为三步，获取未调度的 podList，通过执行一系列调度算法为 pod 选择一个合适的 node，提交数据到 apiserver，其核心则是一系列调度算法的设计与执行。

官方对 kube-scheduler 的调度流程描述 The Kubernetes Scheduler：

```
For given pod:

    +---------------------------------------------+
    |               Schedulable nodes:            |
    |                                             |
    | +--------+    +--------+      +--------+    |
    | | node 1 |    | node 2 |      | node 3 |    |
    | +--------+    +--------+      +--------+    |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Pred. filters: node 3 doesn't have enough resource

    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+
    |             remaining nodes:                |
    |   +--------+                 +--------+     |
    |   | node 1 |                 | node 2 |     |
    |   +--------+                 +--------+     |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Priority function:    node 1: p=2
                          node 2: p=5

    +-------------------+-------------------------+
                        |
                        |
                        v
            select max{node priority} = node 2
```

kube-scheduler 目前包含两部分调度算法 predicates 和 priorities，首先执行 predicates 算法过滤部分 node 然后执行 priorities 算法为所有 node 打分，最后从所有 node 中选出分数最高的最为最佳的 node。

## kube-scheduler 源码分析

kubernetes 中所有组件的启动流程都是类似的，首先会解析命令行参数、添加默认值，kube-scheduler 的默认参数在 k8s.io/kubernetes/pkg/scheduler/apis/config/v1alpha1/defaults.go 中定义的。然后会执行 run 方法启动主逻辑，下面直接看 kube-scheduler 的主逻辑 run 方法执行过程。

Run() 方法主要做了以下工作：

启动 kube-scheduler server，kube-scheduler 监听 10251 和 10259 端口，10251 端口不需要认证，可以获取 healthz metrics 等信息，10259 为安全端口，需要认证
启动所有的 informer
执行 sched.Run() 方法，执行主调度逻辑

```
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
	// To help debugging, immediately log version
	klog.V(1).Infof("Starting Kubernetes Scheduler version %+v", version.Get())

	// Configz registration.
	if cz, err := configz.New("componentconfig"); err == nil {
		cz.Set(cc.ComponentConfig)
	} else {
		return fmt.Errorf("unable to register configz: %s", err)
	}

	// Prepare the event broadcaster.
	if cc.Broadcaster != nil && cc.EventClient != nil {
		cc.Broadcaster.StartRecordingToSink(ctx.Done())
	}
	if cc.CoreBroadcaster != nil && cc.CoreEventClient != nil {
		cc.CoreBroadcaster.StartRecordingToSink(&corev1.EventSinkImpl{Interface: cc.CoreEventClient.Events("")})
	}
	// Setup healthz checks.
	var checks []healthz.HealthChecker
	if cc.ComponentConfig.LeaderElection.LeaderElect {
		checks = append(checks, cc.LeaderElection.WatchDog)
	}

	// Start up the healthz server.
	if cc.InsecureServing != nil {
		separateMetrics := cc.InsecureMetricsServing != nil
		handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, separateMetrics, checks...), nil, nil)
		if err := cc.InsecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start healthz server: %v", err)
		}
	}
	if cc.InsecureMetricsServing != nil {
		handler := buildHandlerChain(newMetricsHandler(&cc.ComponentConfig), nil, nil)
		if err := cc.InsecureMetricsServing.Serve(handler, 0, ctx.Done()); err != nil {
			return fmt.Errorf("failed to start metrics server: %v", err)
		}
	}
	if cc.SecureServing != nil {
		handler := buildHandlerChain(newHealthzHandler(&cc.ComponentConfig, false, checks...), cc.Authentication.Authenticator, cc.Authorization.Authorizer)
		// TODO: handle stoppedCh returned by c.SecureServing.Serve
		if _, err := cc.SecureServing.Serve(handler, 0, ctx.Done()); err != nil {
			// fail early for secure handlers, removing the old error loop from above
			return fmt.Errorf("failed to start secure server: %v", err)
		}
	}

	// Start all informers.
	go cc.PodInformer.Informer().Run(ctx.Done())
	cc.InformerFactory.Start(ctx.Done())

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	// If leader election is enabled, runCommand via LeaderElector until done and exit.
	if cc.LeaderElection != nil {
		cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
			OnStartedLeading: sched.Run,
			OnStoppedLeading: func() {
				klog.Fatalf("leaderelection lost")
			},
		}
		leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
		if err != nil {
			return fmt.Errorf("couldn't create leader elector: %v", err)
		}

		leaderElector.Run(ctx)

		return fmt.Errorf("lost lease")
	}

	// Leader election is disabled, so runCommand inline until done.
	sched.Run(ctx)
	return fmt.Errorf("finished without leader elect")
}
```

下面看一下 scheduler.New() 方法是如何初始化 scheduler 结构体的，该方法主要的功能是初始化默认的调度算法以及默认的调度器 GenericScheduler。

创建 scheduler 配置文件
根据默认的 DefaultProvider 初始化 schedulerAlgorithmSource 然后加载默认的预选及优选算法，然后初始化 GenericScheduler
若启动参数提供了 policy config 则使用其覆盖默认的预选及优选算法并初始化 GenericScheduler，不过该参数现已被弃用

```
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	podInformer coreinformers.PodInformer,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

	schedulerCache := internalcache.New(30*time.Second, stopEverything)
	volumeBinder := scheduling.NewVolumeBinder(
		client,
		informerFactory.Core().V1().Nodes(),
		informerFactory.Storage().V1().CSINodes(),
		informerFactory.Core().V1().PersistentVolumeClaims(),
		informerFactory.Core().V1().PersistentVolumes(),
		informerFactory.Storage().V1().StorageClasses(),
		time.Duration(options.bindTimeoutSeconds)*time.Second,
	)

	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	snapshot := internalcache.NewEmptySnapshot()

	configurator := &Configurator{
		client:                   client,
		recorderFactory:          recorderFactory,
		informerFactory:          informerFactory,
		podInformer:              podInformer,
		volumeBinder:             volumeBinder,
		schedulerCache:           schedulerCache,
		StopEverything:           stopEverything,
		disablePreemption:        options.disablePreemption,
		percentageOfNodesToScore: options.percentageOfNodesToScore,
		bindTimeoutSeconds:       options.bindTimeoutSeconds,
		podInitialBackoffSeconds: options.podInitialBackoffSeconds,
		podMaxBackoffSeconds:     options.podMaxBackoffSeconds,
		enableNonPreempting:      utilfeature.DefaultFeatureGate.Enabled(kubefeatures.NonPreemptingPriority),
		profiles:                 append([]schedulerapi.KubeSchedulerProfile(nil), options.profiles...),
		registry:                 registry,
		nodeInfoSnapshot:         snapshot,
		extenders:                options.extenders,
		frameworkCapturer:        options.frameworkCapturer,
	}

	metrics.Register()

	var sched *Scheduler
	source := options.schedulerAlgorithmSource
	switch {
	case source.Provider != nil:
		// Create the config from a named algorithm provider.
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
	case source.Policy != nil:
		// Create the config from a user specified policy source.
		policy := &schedulerapi.Policy{}
		switch {
		case source.Policy.File != nil:
			if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
				return nil, err
			}
		case source.Policy.ConfigMap != nil:
			if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
				return nil, err
			}
		}
		// Set extenders on the configurator now that we've decoded the policy
		// In this case, c.extenders should be nil since we're using a policy (and therefore not componentconfig,
		// which would have set extenders in the above instantiation of Configurator from CC options)
		configurator.extenders = policy.Extenders
		sc, err := configurator.createFromConfig(*policy)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler from policy: %v", err)
		}
		sched = sc
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
	// Additional tweaks to the config produced by the configurator.
	sched.DisablePreemption = options.disablePreemption
	sched.StopEverything = stopEverything
	sched.podConditionUpdater = &podConditionUpdaterImpl{client}
	sched.podPreemptor = &podPreemptorImpl{client}
	sched.scheduledPodsHasSynced = podInformer.Informer().HasSynced

	addAllEventHandlers(sched, informerFactory, podInformer)
	return sched, nil
}
```

下面是 pod informer 的启动逻辑，只监听 status.phase 不为 succeeded 以及 failed 状态的 pod，即非 terminating 的 pod。

```
func NewPodInformer(client clientset.Interface, resyncPeriod time.Duration) coreinformers.PodInformer {
	selector := fields.ParseSelectorOrDie(
		"status.phase!=" + string(v1.PodSucceeded) +
			",status.phase!=" + string(v1.PodFailed))
	lw := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), string(v1.ResourcePods), metav1.NamespaceAll, selector)
	return &podInformer{
		informer: cache.NewSharedIndexInformer(lw, &v1.Pod{}, resyncPeriod, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}),
	}
}
```

然后继续看 Run() 方法中最后执行的 sched.Run() 调度循环逻辑，若 informer 中的 cache 同步完成后会启动一个循环逻辑执行 sched.SchedulingQueue.Run 方法。

```
func (sched *Scheduler) Run(ctx context.Context) {
	if !cache.WaitForCacheSync(ctx.Done(), sched.scheduledPodsHasSynced) {
		return
	}
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)
	sched.SchedulingQueue.Close()
}
```

运行启动goroutine将其从podBackoffQ送到到activeQ

```
func (p *PriorityQueue) Run() {
	go wait.Until(p.flushBackoffQCompleted, 1.0*time.Second, p.stop)
	go wait.Until(p.flushUnschedulableQLeftover, 30*time.Second, p.stop)
}
```

flushBackoffQCompleted 把已完成的所有pod从backoffQ移到activeQ

```
func (p *PriorityQueue) flushBackoffQCompleted() {
	p.lock.Lock()
	defer p.lock.Unlock()
	for {
		rawPodInfo := p.podBackoffQ.Peek()
		if rawPodInfo == nil {
			return
		}
		pod := rawPodInfo.(*framework.QueuedPodInfo).Pod
		boTime := p.getBackoffTime(rawPodInfo.(*framework.QueuedPodInfo))
		if boTime.After(p.clock.Now()) {
			return
		}
		_, err := p.podBackoffQ.Pop()
		if err != nil {
			klog.Errorf("Unable to pop pod %v from backoff queue despite backoff completion.", nsNameForPod(pod))
			return
		}
		p.activeQ.Add(rawPodInfo)
		metrics.SchedulerQueueIncomingPods.WithLabelValues("active", BackoffComplete).Inc()
		defer p.cond.Broadcast()
	}
}
```

flushUnschedulableQLeftover把所有停留在unschedulableQ的时间大于unschedulableQTimeInterval移动到activeQ

```
func (p *PriorityQueue) flushUnschedulableQLeftover() {
	p.lock.Lock()
	defer p.lock.Unlock()

	var podsToMove []*framework.QueuedPodInfo
	currentTime := p.clock.Now()
	for _, pInfo := range p.unschedulableQ.podInfoMap {
		lastScheduleTime := pInfo.Timestamp
		if currentTime.Sub(lastScheduleTime) > unschedulableQTimeInterval {
			podsToMove = append(podsToMove, pInfo)
		}
	}

	if len(podsToMove) > 0 {
		p.movePodsToActiveOrBackoffQueue(podsToMove, UnschedulableTimeout)
	}
}
```

## 总结

本文主要对于 kube-scheduler 的调度流程进行了分析，但其中有大量的细节都暂未提及，包括预选算法以及优选算法的具体实现、优先级与抢占调度的实现、framework 的使用及实现，因篇幅有限，部分内容会在后文继续说明。
