# kube-apiserver 中 apiserver service 的实现

在 kubernetes，可以从集群外部和内部两种方式访问 kubernetes API，在集群外直接访问 apiserver 提供的 API，在集群内即 pod 中可以通过访问 service 为 kubernetes 的 ClusterIP。kubernetes 集群在初始化完成后就会创建一个 kubernetes service，该 service 是 kube-apiserver 创建并进行维护的，如下所示：

```
$ kubectl get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d22h

$ kubectl get endpoints kubernetes
NAME         ENDPOINTS             AGE
kubernetes   192.168.99.113:6443   4d22h
```

内置的 kubernetes service 无法删除，其 ClusterIP 为通过 --service-cluster-ip-range 参数指定的 ip 段中的首个 ip，kubernetes endpoints 中的 ip 以及 port 可以通过 --advertise-address 和 --secure-port 启动参数来指定。

kubernetes service 是由 kube-apiserver 中的 bootstrap controller 进行控制的，其主要以下几个功能：

创建 kubernetes service；
创建 default、kube-system 和 kube-public 命名空间，如果启用了 NodeLease 特性还会创建 kube-node-lease 命名空间；
提供基于 Service ClusterIP 的修复及检查功能；
提供基于 Service NodePort 的修复及检查功能；
kubernetes service 默认使用 ClusterIP 对外暴露服务，若要使用 nodePort 的方式可在 kube-apiserver 启动时通过 --kubernetes-service-node-port 参数指定对应的端口。

## bootstrap controller 源码分析

bootstrap controller 的初始化以及启动是在 CreateKubeAPIServer 调用链的 InstallLegacyAPI 方法中完成的，bootstrap controller 的启停是由 apiserver 的 PostStartHook 和 ShutdownHook 进行控制的。

```
func (m *Master) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter, legacyRESTStorageProvider corerest.LegacyRESTStorageProvider) error {
	legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(restOptionsGetter)
	if err != nil {
		return fmt.Errorf("error building core storage: %v", err)
	}

	controllerName := "bootstrap-controller"
	coreClient := corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
	bootstrapController := c.NewBootstrapController(legacyRESTStorage, coreClient, coreClient, coreClient, coreClient.RESTClient())
	m.GenericAPIServer.AddPostStartHookOrDie(controllerName, bootstrapController.PostStartHook)
	m.GenericAPIServer.AddPreShutdownHookOrDie(controllerName, bootstrapController.PreShutdownHook)

	if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}
```

postStartHooks 会在 kube-apiserver 的启动方法 prepared.Run 中调用 RunPostStartHooks 启动所有 Hook。

### NewBootstrapController

bootstrap controller 在初始化时需要设定多个参数，主要有 PublicIP、ServiceCIDR、PublicServicePort 等。PublicIP 是通过命令行参数 --advertise-address 指定的，如果没有指定，系统会自动选出一个 global IP。PublicServicePort 通过 --secure-port 启动参数来指定（默认为 6443），ServiceCIDR 通过 --service-cluster-ip-range 参数指定（默认为 10.0.0.0/24）。

```
func (c *completedConfig) NewBootstrapController(legacyRESTStorage corerest.LegacyRESTStorage, serviceClient corev1client.ServicesGetter, nsClient corev1client.NamespacesGetter, eventClient corev1client.EventsGetter, healthClient rest.Interface) *Controller {
	_, publicServicePort, err := c.GenericConfig.SecureServing.HostPort()
	if err != nil {
		klog.Fatalf("failed to get listener address: %v", err)
	}

	systemNamespaces := []string{metav1.NamespaceSystem, metav1.NamespacePublic, corev1.NamespaceNodeLease}

	return &Controller{
		ServiceClient:   serviceClient,
		NamespaceClient: nsClient,
		EventClient:     eventClient,
		healthClient:    healthClient,

		EndpointReconciler: c.ExtraConfig.EndpointReconcilerConfig.Reconciler,
		EndpointInterval:   c.ExtraConfig.EndpointReconcilerConfig.Interval,

		SystemNamespaces:         systemNamespaces,
		SystemNamespacesInterval: 1 * time.Minute,

		ServiceClusterIPRegistry:          legacyRESTStorage.ServiceClusterIPAllocator,
		ServiceClusterIPRange:             c.ExtraConfig.ServiceIPRange,
		SecondaryServiceClusterIPRegistry: legacyRESTStorage.SecondaryServiceClusterIPAllocator,
		SecondaryServiceClusterIPRange:    c.ExtraConfig.SecondaryServiceIPRange,

		ServiceClusterIPInterval: 3 * time.Minute,

		ServiceNodePortRegistry: legacyRESTStorage.ServiceNodePortAllocator,
		ServiceNodePortRange:    c.ExtraConfig.ServiceNodePortRange,
		ServiceNodePortInterval: 3 * time.Minute,

		PublicIP: c.GenericConfig.PublicAddress,

		ServiceIP:                 c.ExtraConfig.APIServerServiceIP,
		ServicePort:               c.ExtraConfig.APIServerServicePort,
		ExtraServicePorts:         c.ExtraConfig.ExtraServicePorts,
		ExtraEndpointPorts:        c.ExtraConfig.ExtraEndpointPorts,
		PublicServicePort:         publicServicePort,
		KubernetesServiceNodePort: c.ExtraConfig.KubernetesServiceNodePort,
	}
}
```

自动选出 global IP 的代码如下所示：

```
func ChooseHostInterface() (net.IP, error) {
	return chooseHostInterface(preferIPv4)
}
```

```
func chooseHostInterface(addressFamilies AddressFamilyPreference) (net.IP, error) {
	var nw networkInterfacer = networkInterface{}
	if _, err := os.Stat(ipv4RouteFile); os.IsNotExist(err) {
		return chooseIPFromHostInterfaces(nw, addressFamilies)
	}
	routes, err := getAllDefaultRoutes()
	if err != nil {
		return nil, err
	}
	return chooseHostInterfaceFromRoute(routes, nw, addressFamilies)
}func chooseHostInterface(addressFamilies AddressFamilyPreference) (net.IP, error) {
	var nw networkInterfacer = networkInterface{}
	if _, err := os.Stat(ipv4RouteFile); os.IsNotExist(err) {
		return chooseIPFromHostInterfaces(nw, addressFamilies)
	}
	routes, err := getAllDefaultRoutes()
	if err != nil {
		return nil, err
	}
	return chooseHostInterfaceFromRoute(routes, nw, addressFamilies)
}
```

### bootstrapController.Start

上文已经提到了 bootstrap controller 主要的四个功能：修复 ClusterIP、修复 NodePort、更新 kubernetes service、创建系统所需要的名字空间（default、kube-system、kube-public）。bootstrap controller 在启动后首先会完成一次 ClusterIP、NodePort 和 Kubernets 服务的处理，然后异步循环运行上面的4个工作。以下是其 start 方法：

```
func (c *Controller) Start() {
	if c.runner != nil {
		return
	}

	// Reconcile during first run removing itself until server is ready.
	endpointPorts := createEndpointPortSpec(c.PublicServicePort, "https", c.ExtraEndpointPorts)
	if err := c.EndpointReconciler.RemoveEndpoints(kubernetesServiceName, c.PublicIP, endpointPorts); err != nil {
		klog.Errorf("Unable to remove old endpoints from kubernetes service: %v", err)
	}

	repairClusterIPs := servicecontroller.NewRepair(c.ServiceClusterIPInterval, c.ServiceClient, c.EventClient, &c.ServiceClusterIPRange, c.ServiceClusterIPRegistry, &c.SecondaryServiceClusterIPRange, c.SecondaryServiceClusterIPRegistry)
	repairNodePorts := portallocatorcontroller.NewRepair(c.ServiceNodePortInterval, c.ServiceClient, c.EventClient, c.ServiceNodePortRange, c.ServiceNodePortRegistry)

	// run all of the controllers once prior to returning from Start.
	if err := repairClusterIPs.RunOnce(); err != nil {
		// If we fail to repair cluster IPs apiserver is useless. We should restart and retry.
		klog.Fatalf("Unable to perform initial IP allocation check: %v", err)
	}
	if err := repairNodePorts.RunOnce(); err != nil {
		// If we fail to repair node ports apiserver is useless. We should restart and retry.
		klog.Fatalf("Unable to perform initial service nodePort check: %v", err)
	}

	c.runner = async.NewRunner(c.RunKubernetesNamespaces, c.RunKubernetesService, repairClusterIPs.RunUntil, repairNodePorts.RunUntil)
	c.runner.Start()
}
```

### c.RunKubernetesNamespaces

c.RunKubernetesNamespaces 主要功能是创建 kube-system 和 kube-public 命名空间，如果启用了 NodeLease 特性功能还会创建 kube-node-lease 命名空间，之后每隔一分钟检查一次。

```
func (c *Controller) RunKubernetesNamespaces(ch chan struct{}) {
	wait.Until(func() {
		// Loop the system namespace list, and create them if they do not exist
		for _, ns := range c.SystemNamespaces {
			if err := createNamespaceIfNeeded(c.NamespaceClient, ns); err != nil {
				runtime.HandleError(fmt.Errorf("unable to create required kubernetes system namespace %s: %v", ns, err))
			}
		}
	}, c.SystemNamespacesInterval, ch)
}
```

### c.RunKubernetesService

c.RunKubernetesService 主要是检查 kubernetes service 是否处于正常状态，并定期执行同步操作。首先调用 /healthz 接口检查 apiserver 当前是否处于 ready 状态，若处于 ready 状态然后调用 c.UpdateKubernetesService 服务更新 kubernetes service 状态。

```
func (c *Controller) RunKubernetesService(ch chan struct{}) {
	// wait until process is ready
	wait.PollImmediateUntil(100*time.Millisecond, func() (bool, error) {
		var code int
		c.healthClient.Get().AbsPath("/healthz").Do(context.TODO()).StatusCode(&code)
		return code == http.StatusOK, nil
	}, ch)

	wait.NonSlidingUntil(func() {
		// Service definition is not reconciled after first
		// run, ports and type will be corrected only during
		// start.
		if err := c.UpdateKubernetesService(false); err != nil {
			runtime.HandleError(fmt.Errorf("unable to sync kubernetes service: %v", err))
		}
	}, c.EndpointInterval, ch)
}
```

#### c.UpdateKubernetesService

c.UpdateKubernetesService 的主要逻辑为：

1、调用 createNamespaceIfNeeded 创建 default namespace；
2、调用 c.CreateOrUpdateMasterServiceIfNeeded 为 master 创建 kubernetes service；
3、调用 c.EndpointReconciler.ReconcileEndpoints 更新 master 的 endpoint；

```
func (c *Controller) UpdateKubernetesService(reconcile bool) error {
	// Update service & endpoint records.
	// TODO: when it becomes possible to change this stuff,
	// stop polling and start watching.
	// TODO: add endpoints of all replicas, not just the elected master.
	if err := createNamespaceIfNeeded(c.NamespaceClient, metav1.NamespaceDefault); err != nil {
		return err
	}

	servicePorts, serviceType := createPortAndServiceSpec(c.ServicePort, c.PublicServicePort, c.KubernetesServiceNodePort, "https", c.ExtraServicePorts)
	if err := c.CreateOrUpdateMasterServiceIfNeeded(kubernetesServiceName, c.ServiceIP, servicePorts, serviceType, reconcile); err != nil {
		return err
	}
	endpointPorts := createEndpointPortSpec(c.PublicServicePort, "https", c.ExtraEndpointPorts)
	if err := c.EndpointReconciler.ReconcileEndpoints(kubernetesServiceName, c.PublicIP, endpointPorts, reconcile); err != nil {
		return err
	}
	return nil
}
```

#### c.EndpointReconciler.ReconcileEndpoints

EndpointReconciler 的具体实现由 EndpointReconcilerType 决定，EndpointReconcilerType 是 --endpoint-reconciler-type 参数指定的，可选的参数有 master-count, lease, none，每种类型对应不同的 EndpointReconciler 实例，在 v1.16 中默认为 lease，此处仅分析 lease 对应的 EndpointReconciler 的实现。

一个集群中可能会有多个 apiserver 实例，因此需要统一管理 apiserver service 的 endpoints，c.EndpointReconciler.ReconcileEndpoints 就是用来管理 apiserver endpoints 的。一个集群中 apiserver 的所有实例会在 etcd 中的对应目录下创建 key，并定期更新这个 key 来上报自己的心跳信息，ReconcileEndpoints 会从 etcd 中获取 apiserver 的实例信息并更新 endpoint。

```
func (r *leaseEndpointReconciler) ReconcileEndpoints(serviceName string, ip net.IP, endpointPorts []corev1.EndpointPort, reconcilePorts bool) error {
	r.reconcilingLock.Lock()
	defer r.reconcilingLock.Unlock()

	if r.stopReconcilingCalled {
		return nil
	}

	// Refresh the TTL on our key, independently of whether any error or
	// update conflict happens below. This makes sure that at least some of
	// the masters will add our endpoint.
	if err := r.masterLeases.UpdateLease(ip.String()); err != nil {
		return err
	}

	return r.doReconcile(serviceName, endpointPorts, reconcilePorts)
}
```

### repairClusterIPs.RunUntil

repairClusterIP 主要解决的问题有：

保证集群中所有的 ClusterIP 都是唯一分配的；
保证分配的 ClusterIP 不会超出指定范围；
确保已经分配给 service 但是因为 crash 等其他原因没有正确创建 ClusterIP；
自动将旧版本的 Kubernetes services 迁移到 ipallocator 原子性模型；
repairClusterIPs.RunUntil 其实是调用 repairClusterIPs.runOnce 来处理的，其代码中的主要逻辑如下所示：

```
func (c *Repair) runOnce() error {
	// TODO: (per smarterclayton) if Get() or ListServices() is a weak consistency read,
	// or if they are executed against different leaders,
	// the ordering guarantee required to ensure no IP is allocated twice is violated.
	// ListServices must return a ResourceVersion higher than the etcd index Get triggers,
	// and the release code must not release services that have had IPs allocated but not yet been created
	// See #8295

	// If etcd server is not running we should wait for some time and fail only then. This is particularly
	// important when we start apiserver and etcd at the same time.
	var snapshot *api.RangeAllocation
	var secondarySnapshot *api.RangeAllocation

	var stored, secondaryStored ipallocator.Interface
	var err, secondaryErr error

	err = wait.PollImmediate(time.Second, 10*time.Second, func() (bool, error) {
		var err error
		snapshot, err = c.alloc.Get()
		if err != nil {
			return false, err
		}

		if c.shouldWorkOnSecondary() {
			secondarySnapshot, err = c.secondaryAlloc.Get()
			if err != nil {
				return false, err
			}
		}

		return true, nil
	})
	if err != nil {
		return fmt.Errorf("unable to refresh the service IP block: %v", err)
	}
	// If not yet initialized.
	if snapshot.Range == "" {
		snapshot.Range = c.network.String()
	}

	if c.shouldWorkOnSecondary() && secondarySnapshot.Range == "" {
		secondarySnapshot.Range = c.secondaryNetwork.String()
	}
	// Create an allocator because it is easy to use.

	stored, err = ipallocator.NewFromSnapshot(snapshot)
	if c.shouldWorkOnSecondary() {
		secondaryStored, secondaryErr = ipallocator.NewFromSnapshot(secondarySnapshot)
	}

	if err != nil || secondaryErr != nil {
		return fmt.Errorf("unable to rebuild allocator from snapshots: %v", err)
	}

	// We explicitly send no resource version, since the resource version
	// of 'snapshot' is from a different collection, it's not comparable to
	// the service collection. The caching layer keeps per-collection RVs,
	// and this is proper, since in theory the collections could be hosted
	// in separate etcd (or even non-etcd) instances.
	list, err := c.serviceClient.Services(metav1.NamespaceAll).List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		return fmt.Errorf("unable to refresh the service IP block: %v", err)
	}

	var rebuilt, secondaryRebuilt *ipallocator.Range
	rebuilt, err = ipallocator.NewCIDRRange(c.network)
	if err != nil {
		return fmt.Errorf("unable to create CIDR range: %v", err)
	}

	if c.shouldWorkOnSecondary() {
		secondaryRebuilt, err = ipallocator.NewCIDRRange(c.secondaryNetwork)
	}

	if err != nil {
		return fmt.Errorf("unable to create CIDR range: %v", err)
	}

	// Check every Service's ClusterIP, and rebuild the state as we think it should be.
	for _, svc := range list.Items {
		if !helper.IsServiceIPSet(&svc) {
			// didn't need a cluster IP
			continue
		}
		ip := net.ParseIP(svc.Spec.ClusterIP)
		if ip == nil {
			// cluster IP is corrupt
			c.recorder.Eventf(&svc, v1.EventTypeWarning, "ClusterIPNotValid", "Cluster IP %s is not a valid IP; please recreate service", svc.Spec.ClusterIP)
			runtime.HandleError(fmt.Errorf("the cluster IP %s for service %s/%s is not a valid IP; please recreate", svc.Spec.ClusterIP, svc.Name, svc.Namespace))
			continue
		}

		// mark it as in-use
		actualAlloc := c.selectAllocForIP(ip, rebuilt, secondaryRebuilt)
		switch err := actualAlloc.Allocate(ip); err {
		case nil:
			actualStored := c.selectAllocForIP(ip, stored, secondaryStored)
			if actualStored.Has(ip) {
				// remove it from the old set, so we can find leaks
				actualStored.Release(ip)
			} else {
				// cluster IP doesn't seem to be allocated
				c.recorder.Eventf(&svc, v1.EventTypeWarning, "ClusterIPNotAllocated", "Cluster IP %s is not allocated; repairing", ip)
				runtime.HandleError(fmt.Errorf("the cluster IP %s for service %s/%s is not allocated; repairing", ip, svc.Name, svc.Namespace))
			}
			delete(c.leaks, ip.String()) // it is used, so it can't be leaked
		case ipallocator.ErrAllocated:
			// cluster IP is duplicate
			c.recorder.Eventf(&svc, v1.EventTypeWarning, "ClusterIPAlreadyAllocated", "Cluster IP %s was assigned to multiple services; please recreate service", ip)
			runtime.HandleError(fmt.Errorf("the cluster IP %s for service %s/%s was assigned to multiple services; please recreate", ip, svc.Name, svc.Namespace))
		case err.(*ipallocator.ErrNotInRange):
			// cluster IP is out of range
			c.recorder.Eventf(&svc, v1.EventTypeWarning, "ClusterIPOutOfRange", "Cluster IP %s is not within the service CIDR %s; please recreate service", ip, c.network)
			runtime.HandleError(fmt.Errorf("the cluster IP %s for service %s/%s is not within the service CIDR %s; please recreate", ip, svc.Name, svc.Namespace, c.network))
		case ipallocator.ErrFull:
			// somehow we are out of IPs
			cidr := actualAlloc.CIDR()
			c.recorder.Eventf(&svc, v1.EventTypeWarning, "ServiceCIDRFull", "Service CIDR %v is full; you must widen the CIDR in order to create new services", cidr)
			return fmt.Errorf("the service CIDR %v is full; you must widen the CIDR in order to create new services", cidr)
		default:
			c.recorder.Eventf(&svc, v1.EventTypeWarning, "UnknownError", "Unable to allocate cluster IP %s due to an unknown error", ip)
			return fmt.Errorf("unable to allocate cluster IP %s for service %s/%s due to an unknown error, exiting: %v", ip, svc.Name, svc.Namespace, err)
		}
	}

	c.checkLeaked(stored, rebuilt)
	if c.shouldWorkOnSecondary() {
		c.checkLeaked(secondaryStored, secondaryRebuilt)
	}

	// Blast the rebuilt state into storage.
	err = c.saveSnapShot(rebuilt, c.alloc, snapshot)
	if err != nil {
		return err
	}

	if c.shouldWorkOnSecondary() {
		err := c.saveSnapShot(secondaryRebuilt, c.secondaryAlloc, secondarySnapshot)
		if err != nil {
			return nil
		}
	}
	return nil
}
```

### repairNodePorts.RunUnti

repairNodePorts 主要是用来纠正 service 中 nodePort 的信息，保证所有的 ports 都基于 cluster 创建的，当没有与 cluster 同步时会触发告警，其最终是调用 repairNodePorts.runOnce 进行处理的，主要逻辑与 ClusterIP 的处理逻辑类似。

```
func (c *Repair) runOnce() error {
	// TODO: (per smarterclayton) if Get() or ListServices() is a weak consistency read,
	// or if they are executed against different leaders,
	// the ordering guarantee required to ensure no port is allocated twice is violated.
	// ListServices must return a ResourceVersion higher than the etcd index Get triggers,
	// and the release code must not release services that have had ports allocated but not yet been created
	// See #8295

	// If etcd server is not running we should wait for some time and fail only then. This is particularly
	// important when we start apiserver and etcd at the same time.
	var snapshot *api.RangeAllocation

	err := wait.PollImmediate(time.Second, 10*time.Second, func() (bool, error) {
		var err error
		snapshot, err = c.alloc.Get()
		return err == nil, err
	})
	if err != nil {
		return fmt.Errorf("unable to refresh the port allocations: %v", err)
	}
	// If not yet initialized.
	if snapshot.Range == "" {
		snapshot.Range = c.portRange.String()
	}
	// Create an allocator because it is easy to use.
	stored, err := portallocator.NewFromSnapshot(snapshot)
	if err != nil {
		return fmt.Errorf("unable to rebuild allocator from snapshot: %v", err)
	}

	// We explicitly send no resource version, since the resource version
	// of 'snapshot' is from a different collection, it's not comparable to
	// the service collection. The caching layer keeps per-collection RVs,
	// and this is proper, since in theory the collections could be hosted
	// in separate etcd (or even non-etcd) instances.
	list, err := c.serviceClient.Services(metav1.NamespaceAll).List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		return fmt.Errorf("unable to refresh the port block: %v", err)
	}

	rebuilt, err := portallocator.NewPortAllocator(c.portRange)
	if err != nil {
		return fmt.Errorf("unable to create port allocator: %v", err)
	}
	// Check every Service's ports, and rebuild the state as we think it should be.
	for i := range list.Items {
		svc := &list.Items[i]
		ports := collectServiceNodePorts(svc)
		if len(ports) == 0 {
			continue
		}

		for _, port := range ports {
			switch err := rebuilt.Allocate(port); err {
			case nil:
				if stored.Has(port) {
					// remove it from the old set, so we can find leaks
					stored.Release(port)
				} else {
					// doesn't seem to be allocated
					c.recorder.Eventf(svc, corev1.EventTypeWarning, "PortNotAllocated", "Port %d is not allocated; repairing", port)
					runtime.HandleError(fmt.Errorf("the node port %d for service %s/%s is not allocated; repairing", port, svc.Name, svc.Namespace))
				}
				delete(c.leaks, port) // it is used, so it can't be leaked
			case portallocator.ErrAllocated:
				// port is duplicate, reallocate
				c.recorder.Eventf(svc, corev1.EventTypeWarning, "PortAlreadyAllocated", "Port %d was assigned to multiple services; please recreate service", port)
				runtime.HandleError(fmt.Errorf("the node port %d for service %s/%s was assigned to multiple services; please recreate", port, svc.Name, svc.Namespace))
			case err.(*portallocator.ErrNotInRange):
				// port is out of range, reallocate
				c.recorder.Eventf(svc, corev1.EventTypeWarning, "PortOutOfRange", "Port %d is not within the port range %s; please recreate service", port, c.portRange)
				runtime.HandleError(fmt.Errorf("the port %d for service %s/%s is not within the port range %s; please recreate", port, svc.Name, svc.Namespace, c.portRange))
			case portallocator.ErrFull:
				// somehow we are out of ports
				c.recorder.Eventf(svc, corev1.EventTypeWarning, "PortRangeFull", "Port range %s is full; you must widen the port range in order to create new services", c.portRange)
				return fmt.Errorf("the port range %s is full; you must widen the port range in order to create new services", c.portRange)
			default:
				c.recorder.Eventf(svc, corev1.EventTypeWarning, "UnknownError", "Unable to allocate port %d due to an unknown error", port)
				return fmt.Errorf("unable to allocate port %d for service %s/%s due to an unknown error, exiting: %v", port, svc.Name, svc.Namespace, err)
			}
		}
	}

	// Check for ports that are left in the old set.  They appear to have been leaked.
	stored.ForEach(func(port int) {
		count, found := c.leaks[port]
		switch {
		case !found:
			// flag it to be cleaned up after any races (hopefully) are gone
			runtime.HandleError(fmt.Errorf("the node port %d may have leaked: flagging for later clean up", port))
			count = numRepairsBeforeLeakCleanup - 1
			fallthrough
		case count > 0:
			// pretend it is still in use until count expires
			c.leaks[port] = count - 1
			if err := rebuilt.Allocate(port); err != nil {
				runtime.HandleError(fmt.Errorf("the node port %d may have leaked, but can not be allocated: %v", port, err))
			}
		default:
			// do not add it to the rebuilt set, which means it will be available for reuse
			runtime.HandleError(fmt.Errorf("the node port %d appears to have leaked: cleaning up", port))
		}
	})

	// Blast the rebuilt state into storage.
	if err := rebuilt.Snapshot(snapshot); err != nil {
		return fmt.Errorf("unable to snapshot the updated port allocations: %v", err)
	}

	if err := c.alloc.CreateOrUpdate(snapshot); err != nil {
		if errors.IsConflict(err) {
			return err
		}
		return fmt.Errorf("unable to persist the updated port allocations: %v", err)
	}
	return nil
}
```

## 总结

本文主要分析了 kube-apiserver 中 apiserver service 的实现，apiserver service 是通过 bootstrap controller 控制的，bootstrap controller 会保证 apiserver service 以及其 endpoint 处于正常状态，需要注意的是，apiserver service 的 endpoint 根据启动时指定的参数分为三种控制方式，本文仅分析了 lease 的实现方式，如果使用 master-count 方式，需要将每个 master 实例的 port、apiserver-count 等配置参数改为一致。
