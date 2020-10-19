# Pod AutoScaler

## ReplicationController

ReplicationController 确保在任何时候都有特定数量的 Pod 副本处于运行状态。 换句话说，ReplicationController 确保一个 Pod 或一组同类的 Pod 总是可用的。

首先来看ReplicationController的启动

```
func startHPAController(ctx ControllerContext) (http.Handler, bool, error) {
	if !ctx.AvailableResources[schema.GroupVersionResource{Group: "autoscaling", Version: "v1", Resource: "horizontalpodautoscalers"}] {
		return nil, false, nil
	}

	if ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerUseRESTClients {
		// use the new-style clients if support for custom metrics is enabled
		return startHPAControllerWithRESTClient(ctx)
	}

	return startHPAControllerWithLegacyClient(ctx)
}

func startHPAControllerWithRESTClient(ctx ControllerContext) (http.Handler, bool, error) {
	clientConfig := ctx.ClientBuilder.ConfigOrDie("horizontal-pod-autoscaler")
	hpaClient := ctx.ClientBuilder.ClientOrDie("horizontal-pod-autoscaler")

	apiVersionsGetter := custom_metrics.NewAvailableAPIsGetter(hpaClient.Discovery())
	// invalidate the discovery information roughly once per resync interval our API
	// information is *at most* two resync intervals old.
	go custom_metrics.PeriodicallyInvalidate(
		apiVersionsGetter,
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerSyncPeriod.Duration,
		ctx.Stop)

	metricsClient := metrics.NewRESTMetricsClient(
		resourceclient.NewForConfigOrDie(clientConfig),
		custom_metrics.NewForConfig(clientConfig, ctx.RESTMapper, apiVersionsGetter),
		external_metrics.NewForConfigOrDie(clientConfig),
	)
	return startHPAControllerWithMetricsClient(ctx, metricsClient)
}

func startHPAControllerWithLegacyClient(ctx ControllerContext) (http.Handler, bool, error) {
	hpaClient := ctx.ClientBuilder.ClientOrDie("horizontal-pod-autoscaler")
	metricsClient := metrics.NewHeapsterMetricsClient(
		hpaClient,
		metrics.DefaultHeapsterNamespace,
		metrics.DefaultHeapsterScheme,
		metrics.DefaultHeapsterService,
		metrics.DefaultHeapsterPort,
	)
	return startHPAControllerWithMetricsClient(ctx, metricsClient)
}

func startHPAControllerWithMetricsClient(ctx ControllerContext, metricsClient metrics.MetricsClient) (http.Handler, bool, error) {
	hpaClient := ctx.ClientBuilder.ClientOrDie("horizontal-pod-autoscaler")
	hpaClientConfig := ctx.ClientBuilder.ConfigOrDie("horizontal-pod-autoscaler")

	// we don't use cached discovery because DiscoveryScaleKindResolver does its own caching,
	// so we want to re-fetch every time when we actually ask for it
	scaleKindResolver := scale.NewDiscoveryScaleKindResolver(hpaClient.Discovery())
	scaleClient, err := scale.NewForConfig(hpaClientConfig, ctx.RESTMapper, dynamic.LegacyAPIPathResolverFunc, scaleKindResolver)
	if err != nil {
		return nil, false, err
	}

	go podautoscaler.NewHorizontalController(
		hpaClient.CoreV1(),
		scaleClient,
		hpaClient.AutoscalingV1(),
		ctx.RESTMapper,
		metricsClient,
		ctx.InformerFactory.Autoscaling().V1().HorizontalPodAutoscalers(),
		ctx.InformerFactory.Core().V1().Pods(),
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerSyncPeriod.Duration,
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerDownscaleStabilizationWindow.Duration,
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerTolerance,
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerCPUInitializationPeriod.Duration,
		ctx.ComponentConfig.HPAController.HorizontalPodAutoscalerInitialReadinessDelay.Duration,
	).Run(ctx.Stop)
	return nil, true, nil
}
```

```
// NewHorizontalController creates a new HorizontalController.
func NewHorizontalController(
	evtNamespacer v1core.EventsGetter,
	scaleNamespacer scaleclient.ScalesGetter,
	hpaNamespacer autoscalingclient.HorizontalPodAutoscalersGetter,
	mapper apimeta.RESTMapper,
	metricsClient metricsclient.MetricsClient,
	hpaInformer autoscalinginformers.HorizontalPodAutoscalerInformer,
	podInformer coreinformers.PodInformer,
	resyncPeriod time.Duration,
	downscaleStabilisationWindow time.Duration,
	tolerance float64,
	cpuInitializationPeriod,
	delayOfInitialReadinessStatus time.Duration,

) *HorizontalController {
	broadcaster := record.NewBroadcaster()
	broadcaster.StartStructuredLogging(0)
	broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: evtNamespacer.Events("")})
	recorder := broadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "horizontal-pod-autoscaler"})

	hpaController := &HorizontalController{
		eventRecorder:                recorder,
		scaleNamespacer:              scaleNamespacer,
		hpaNamespacer:                hpaNamespacer,
		downscaleStabilisationWindow: downscaleStabilisationWindow,
		queue:                        workqueue.NewNamedRateLimitingQueue(NewDefaultHPARateLimiter(resyncPeriod), "horizontalpodautoscaler"),
		mapper:                       mapper,
		recommendations:              map[string][]timestampedRecommendation{},
		scaleUpEvents:                map[string][]timestampedScaleEvent{},
		scaleDownEvents:              map[string][]timestampedScaleEvent{},
	}

	hpaInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    hpaController.enqueueHPA,
			UpdateFunc: hpaController.updateHPA,
			DeleteFunc: hpaController.deleteHPA,
		},
		resyncPeriod,
	)
	hpaController.hpaLister = hpaInformer.Lister()
	hpaController.hpaListerSynced = hpaInformer.Informer().HasSynced

	hpaController.podLister = podInformer.Lister()
	hpaController.podListerSynced = podInformer.Informer().HasSynced

	replicaCalc := NewReplicaCalculator(
		metricsClient,
		hpaController.podLister,
		tolerance,
		cpuInitializationPeriod,
		delayOfInitialReadinessStatus,
	)
	hpaController.replicaCalc = replicaCalc

	return hpaController
}
```

```
// HorizontalController is responsible for the synchronizing HPA objects stored
// in the system with the actual deployments/replication controllers they
// control.
type HorizontalController struct {
	scaleNamespacer scaleclient.ScalesGetter
	hpaNamespacer   autoscalingclient.HorizontalPodAutoscalersGetter
	mapper          apimeta.RESTMapper

	replicaCalc   *ReplicaCalculator
	eventRecorder record.EventRecorder

	downscaleStabilisationWindow time.Duration

	// hpaLister is able to list/get HPAs from the shared cache from the informer passed in to
	// NewHorizontalController.
	hpaLister       autoscalinglisters.HorizontalPodAutoscalerLister
	hpaListerSynced cache.InformerSynced

	// podLister is able to list/get Pods from the shared cache from the informer passed in to
	// NewHorizontalController.
	podLister       corelisters.PodLister
	podListerSynced cache.InformerSynced

	// Controllers that need to be synced
	queue workqueue.RateLimitingInterface

	// Latest unstabilized recommendations for each autoscaler.
	recommendations map[string][]timestampedRecommendation

	// Latest autoscaler events
	scaleUpEvents   map[string][]timestampedScaleEvent
	scaleDownEvents map[string][]timestampedScaleEvent
}
```

```
// ScalesGetter can produce a ScaleInterface
type ScalesGetter interface {
	// Scales produces a ScaleInterface for a particular namespace.
	// Set namespace to the empty string for non-namespaced resources.
	Scales(namespace string) ScaleInterface
}

// ScaleInterface can fetch and update scales for
// resources in a particular namespace which implement
// the scale subresource.
type ScaleInterface interface {
	// Get fetches the scale of the given scalable resource.
	Get(ctx context.Context, resource schema.GroupResource, name string, opts metav1.GetOptions) (*autoscalingapi.Scale, error)

	// Update updates the scale of the given scalable resource.
	Update(ctx context.Context, resource schema.GroupResource, scale *autoscalingapi.Scale, opts metav1.UpdateOptions) (*autoscalingapi.Scale, error)

	// Patch patches the scale of the given scalable resource.
	Patch(ctx context.Context, gvr schema.GroupVersionResource, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions) (*autoscalingapi.Scale, error)
}
```

```
// HorizontalPodAutoscalersGetter has a method to return a HorizontalPodAutoscalerInterface.
// A group's client should implement this interface.
type HorizontalPodAutoscalersGetter interface {
	HorizontalPodAutoscalers(namespace string) HorizontalPodAutoscalerInterface
}

// HorizontalPodAutoscalerInterface has methods to work with HorizontalPodAutoscaler resources.
type HorizontalPodAutoscalerInterface interface {
	Create(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.CreateOptions) (*v1.HorizontalPodAutoscaler, error)
	Update(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.UpdateOptions) (*v1.HorizontalPodAutoscaler, error)
	UpdateStatus(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.UpdateOptions) (*v1.HorizontalPodAutoscaler, error)
	Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
	DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
	Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.HorizontalPodAutoscaler, error)
	List(ctx context.Context, opts metav1.ListOptions) (*v1.HorizontalPodAutoscalerList, error)
	Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
	Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.HorizontalPodAutoscaler, err error)
	HorizontalPodAutoscalerExpansion
}

// horizontalPodAutoscalers implements HorizontalPodAutoscalerInterface
type horizontalPodAutoscalers struct {
	client rest.Interface
	ns     string
}

// newHorizontalPodAutoscalers returns a HorizontalPodAutoscalers
func newHorizontalPodAutoscalers(c *AutoscalingV1Client, namespace string) *horizontalPodAutoscalers {
	return &horizontalPodAutoscalers{
		client: c.RESTClient(),
		ns:     namespace,
	}
}

// Get takes name of the horizontalPodAutoscaler, and returns the corresponding horizontalPodAutoscaler object, and an error if there is any.
func (c *horizontalPodAutoscalers) Get(ctx context.Context, name string, options metav1.GetOptions) (result *v1.HorizontalPodAutoscaler, err error) {
	result = &v1.HorizontalPodAutoscaler{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		Name(name).
		VersionedParams(&options, scheme.ParameterCodec).
		Do(ctx).
		Into(result)
	return
}

// List takes label and field selectors, and returns the list of HorizontalPodAutoscalers that match those selectors.
func (c *horizontalPodAutoscalers) List(ctx context.Context, opts metav1.ListOptions) (result *v1.HorizontalPodAutoscalerList, err error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	result = &v1.HorizontalPodAutoscalerList{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Do(ctx).
		Into(result)
	return
}

// Watch returns a watch.Interface that watches the requested horizontalPodAutoscalers.
func (c *horizontalPodAutoscalers) Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	opts.Watch = true
	return c.client.Get().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Watch(ctx)
}

// Create takes the representation of a horizontalPodAutoscaler and creates it.  Returns the server's representation of the horizontalPodAutoscaler, and an error, if there is any.
func (c *horizontalPodAutoscalers) Create(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.CreateOptions) (result *v1.HorizontalPodAutoscaler, err error) {
	result = &v1.HorizontalPodAutoscaler{}
	err = c.client.Post().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(horizontalPodAutoscaler).
		Do(ctx).
		Into(result)
	return
}

// Update takes the representation of a horizontalPodAutoscaler and updates it. Returns the server's representation of the horizontalPodAutoscaler, and an error, if there is any.
func (c *horizontalPodAutoscalers) Update(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.UpdateOptions) (result *v1.HorizontalPodAutoscaler, err error) {
	result = &v1.HorizontalPodAutoscaler{}
	err = c.client.Put().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		Name(horizontalPodAutoscaler.Name).
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(horizontalPodAutoscaler).
		Do(ctx).
		Into(result)
	return
}

// UpdateStatus was generated because the type contains a Status member.
// Add a +genclient:noStatus comment above the type to avoid generating UpdateStatus().
func (c *horizontalPodAutoscalers) UpdateStatus(ctx context.Context, horizontalPodAutoscaler *v1.HorizontalPodAutoscaler, opts metav1.UpdateOptions) (result *v1.HorizontalPodAutoscaler, err error) {
	result = &v1.HorizontalPodAutoscaler{}
	err = c.client.Put().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		Name(horizontalPodAutoscaler.Name).
		SubResource("status").
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(horizontalPodAutoscaler).
		Do(ctx).
		Into(result)
	return
}

// Delete takes name of the horizontalPodAutoscaler and deletes it. Returns an error if one occurs.
func (c *horizontalPodAutoscalers) Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error {
	return c.client.Delete().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		Name(name).
		Body(&opts).
		Do(ctx).
		Error()
}

// DeleteCollection deletes a collection of objects.
func (c *horizontalPodAutoscalers) DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error {
	var timeout time.Duration
	if listOpts.TimeoutSeconds != nil {
		timeout = time.Duration(*listOpts.TimeoutSeconds) * time.Second
	}
	return c.client.Delete().
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		VersionedParams(&listOpts, scheme.ParameterCodec).
		Timeout(timeout).
		Body(&opts).
		Do(ctx).
		Error()
}

// Patch applies the patch and returns the patched horizontalPodAutoscaler.
func (c *horizontalPodAutoscalers) Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.HorizontalPodAutoscaler, err error) {
	result = &v1.HorizontalPodAutoscaler{}
	err = c.client.Patch(pt).
		Namespace(c.ns).
		Resource("horizontalpodautoscalers").
		Name(name).
		SubResource(subresources...).
		VersionedParams(&opts, scheme.ParameterCodec).
		Body(data).
		Do(ctx).
		Into(result)
	return
}
```

```
// ReplicaCalculator bundles all needed information to calculate the target amount of replicas
type ReplicaCalculator struct {
	metricsClient                 metricsclient.MetricsClient
	podLister                     corelisters.PodLister
	tolerance                     float64
	cpuInitializationPeriod       time.Duration
	delayOfInitialReadinessStatus time.Duration
}

// NewReplicaCalculator creates a new ReplicaCalculator and passes all necessary information to the new instance
func NewReplicaCalculator(metricsClient metricsclient.MetricsClient, podLister corelisters.PodLister, tolerance float64, cpuInitializationPeriod, delayOfInitialReadinessStatus time.Duration) *ReplicaCalculator {
	return &ReplicaCalculator{
		metricsClient:                 metricsClient,
		podLister:                     podLister,
		tolerance:                     tolerance,
		cpuInitializationPeriod:       cpuInitializationPeriod,
		delayOfInitialReadinessStatus: delayOfInitialReadinessStatus,
	}
}

// GetResourceReplicas calculates the desired replica count based on a target resource utilization percentage
// of the given resource for pods matching the given selector in the given namespace, and the current replica count
func (c *ReplicaCalculator) GetResourceReplicas(currentReplicas int32, targetUtilization int32, resource v1.ResourceName, namespace string, selector labels.Selector) (replicaCount int32, utilization int32, rawUtilization int64, timestamp time.Time, err error) {
	metrics, timestamp, err := c.metricsClient.GetResourceMetric(resource, namespace, selector)
	if err != nil {
		return 0, 0, 0, time.Time{}, fmt.Errorf("unable to get metrics for resource %s: %v", resource, err)
	}
	podList, err := c.podLister.Pods(namespace).List(selector)
	if err != nil {
		return 0, 0, 0, time.Time{}, fmt.Errorf("unable to get pods while calculating replica count: %v", err)
	}

	itemsLen := len(podList)
	if itemsLen == 0 {
		return 0, 0, 0, time.Time{}, fmt.Errorf("no pods returned by selector while calculating replica count")
	}

	readyPodCount, ignoredPods, missingPods := groupPods(podList, metrics, resource, c.cpuInitializationPeriod, c.delayOfInitialReadinessStatus)
	removeMetricsForPods(metrics, ignoredPods)
	requests, err := calculatePodRequests(podList, resource)
	if err != nil {
		return 0, 0, 0, time.Time{}, err
	}

	if len(metrics) == 0 {
		return 0, 0, 0, time.Time{}, fmt.Errorf("did not receive metrics for any ready pods")
	}

	usageRatio, utilization, rawUtilization, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
	if err != nil {
		return 0, 0, 0, time.Time{}, err
	}

	rebalanceIgnored := len(ignoredPods) > 0 && usageRatio > 1.0
	if !rebalanceIgnored && len(missingPods) == 0 {
		if math.Abs(1.0-usageRatio) <= c.tolerance {
			// return the current replicas if the change would be too small
			return currentReplicas, utilization, rawUtilization, timestamp, nil
		}

		// if we don't have any unready or missing pods, we can calculate the new replica count now
		return int32(math.Ceil(usageRatio * float64(readyPodCount))), utilization, rawUtilization, timestamp, nil
	}

	if len(missingPods) > 0 {
		if usageRatio < 1.0 {
			// on a scale-down, treat missing pods as using 100% of the resource request
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: requests[podName]}
			}
		} else if usageRatio > 1.0 {
			// on a scale-up, treat missing pods as using 0% of the resource request
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: 0}
			}
		}
	}

	if rebalanceIgnored {
		// on a scale-up, treat unready pods as using 0% of the resource request
		for podName := range ignoredPods {
			metrics[podName] = metricsclient.PodMetric{Value: 0}
		}
	}

	// re-run the utilization calculation with our new numbers
	newUsageRatio, _, _, err := metricsclient.GetResourceUtilizationRatio(metrics, requests, targetUtilization)
	if err != nil {
		return 0, utilization, rawUtilization, time.Time{}, err
	}

	if math.Abs(1.0-newUsageRatio) <= c.tolerance || (usageRatio < 1.0 && newUsageRatio > 1.0) || (usageRatio > 1.0 && newUsageRatio < 1.0) {
		// return the current replicas if the change would be too small,
		// or if the new usage ratio would cause a change in scale direction
		return currentReplicas, utilization, rawUtilization, timestamp, nil
	}

	newReplicas := int32(math.Ceil(newUsageRatio * float64(len(metrics))))
	if (newUsageRatio < 1.0 && newReplicas > currentReplicas) || (newUsageRatio > 1.0 && newReplicas < currentReplicas) {
		// return the current replicas if the change of metrics length would cause a change in scale direction
		return currentReplicas, utilization, rawUtilization, timestamp, nil
	}

	// return the result, where the number of replicas considered is
	// however many replicas factored into our calculation
	return newReplicas, utilization, rawUtilization, timestamp, nil
}

// GetRawResourceReplicas calculates the desired replica count based on a target resource utilization (as a raw milli-value)
// for pods matching the given selector in the given namespace, and the current replica count
func (c *ReplicaCalculator) GetRawResourceReplicas(currentReplicas int32, targetUtilization int64, resource v1.ResourceName, namespace string, selector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	metrics, timestamp, err := c.metricsClient.GetResourceMetric(resource, namespace, selector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get metrics for resource %s: %v", resource, err)
	}

	replicaCount, utilization, err = c.calcPlainMetricReplicas(metrics, currentReplicas, targetUtilization, namespace, selector, resource)
	return replicaCount, utilization, timestamp, err
}

// GetMetricReplicas calculates the desired replica count based on a target metric utilization
// (as a milli-value) for pods matching the given selector in the given namespace, and the
// current replica count
func (c *ReplicaCalculator) GetMetricReplicas(currentReplicas int32, targetUtilization int64, metricName string, namespace string, selector labels.Selector, metricSelector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	metrics, timestamp, err := c.metricsClient.GetRawMetric(metricName, namespace, selector, metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get metric %s: %v", metricName, err)
	}

	replicaCount, utilization, err = c.calcPlainMetricReplicas(metrics, currentReplicas, targetUtilization, namespace, selector, v1.ResourceName(""))
	return replicaCount, utilization, timestamp, err
}

// calcPlainMetricReplicas calculates the desired replicas for plain (i.e. non-utilization percentage) metrics.
func (c *ReplicaCalculator) calcPlainMetricReplicas(metrics metricsclient.PodMetricsInfo, currentReplicas int32, targetUtilization int64, namespace string, selector labels.Selector, resource v1.ResourceName) (replicaCount int32, utilization int64, err error) {

	podList, err := c.podLister.Pods(namespace).List(selector)
	if err != nil {
		return 0, 0, fmt.Errorf("unable to get pods while calculating replica count: %v", err)
	}

	if len(podList) == 0 {
		return 0, 0, fmt.Errorf("no pods returned by selector while calculating replica count")
	}

	readyPodCount, ignoredPods, missingPods := groupPods(podList, metrics, resource, c.cpuInitializationPeriod, c.delayOfInitialReadinessStatus)
	removeMetricsForPods(metrics, ignoredPods)

	if len(metrics) == 0 {
		return 0, 0, fmt.Errorf("did not receive metrics for any ready pods")
	}

	usageRatio, utilization := metricsclient.GetMetricUtilizationRatio(metrics, targetUtilization)

	rebalanceIgnored := len(ignoredPods) > 0 && usageRatio > 1.0

	if !rebalanceIgnored && len(missingPods) == 0 {
		if math.Abs(1.0-usageRatio) <= c.tolerance {
			// return the current replicas if the change would be too small
			return currentReplicas, utilization, nil
		}

		// if we don't have any unready or missing pods, we can calculate the new replica count now
		return int32(math.Ceil(usageRatio * float64(readyPodCount))), utilization, nil
	}

	if len(missingPods) > 0 {
		if usageRatio < 1.0 {
			// on a scale-down, treat missing pods as using 100% of the resource request
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: targetUtilization}
			}
		} else {
			// on a scale-up, treat missing pods as using 0% of the resource request
			for podName := range missingPods {
				metrics[podName] = metricsclient.PodMetric{Value: 0}
			}
		}
	}

	if rebalanceIgnored {
		// on a scale-up, treat unready pods as using 0% of the resource request
		for podName := range ignoredPods {
			metrics[podName] = metricsclient.PodMetric{Value: 0}
		}
	}

	// re-run the utilization calculation with our new numbers
	newUsageRatio, _ := metricsclient.GetMetricUtilizationRatio(metrics, targetUtilization)

	if math.Abs(1.0-newUsageRatio) <= c.tolerance || (usageRatio < 1.0 && newUsageRatio > 1.0) || (usageRatio > 1.0 && newUsageRatio < 1.0) {
		// return the current replicas if the change would be too small,
		// or if the new usage ratio would cause a change in scale direction
		return currentReplicas, utilization, nil
	}

	// return the result, where the number of replicas considered is
	// however many replicas factored into our calculation
	return int32(math.Ceil(newUsageRatio * float64(len(metrics)))), utilization, nil
}

// GetObjectMetricReplicas calculates the desired replica count based on a target metric utilization (as a milli-value)
// for the given object in the given namespace, and the current replica count.
func (c *ReplicaCalculator) GetObjectMetricReplicas(currentReplicas int32, targetUtilization int64, metricName string, namespace string, objectRef *autoscaling.CrossVersionObjectReference, selector labels.Selector, metricSelector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	utilization, timestamp, err = c.metricsClient.GetObjectMetric(metricName, namespace, objectRef, metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get metric %s: %v on %s %s/%s", metricName, objectRef.Kind, namespace, objectRef.Name, err)
	}

	usageRatio := float64(utilization) / float64(targetUtilization)
	replicaCount, timestamp, err = c.getUsageRatioReplicaCount(currentReplicas, usageRatio, namespace, selector)
	return replicaCount, utilization, timestamp, err
}

// getUsageRatioReplicaCount calculates the desired replica count based on usageRatio and ready pods count.
// For currentReplicas=0 doesn't take into account ready pods count and tolerance to support scaling to zero pods.
func (c *ReplicaCalculator) getUsageRatioReplicaCount(currentReplicas int32, usageRatio float64, namespace string, selector labels.Selector) (replicaCount int32, timestamp time.Time, err error) {
	if currentReplicas != 0 {
		if math.Abs(1.0-usageRatio) <= c.tolerance {
			// return the current replicas if the change would be too small
			return currentReplicas, timestamp, nil
		}
		readyPodCount := int64(0)
		readyPodCount, err = c.getReadyPodsCount(namespace, selector)
		if err != nil {
			return 0, time.Time{}, fmt.Errorf("unable to calculate ready pods: %s", err)
		}
		replicaCount = int32(math.Ceil(usageRatio * float64(readyPodCount)))
	} else {
		// Scale to zero or n pods depending on usageRatio
		replicaCount = int32(math.Ceil(usageRatio))
	}

	return replicaCount, timestamp, err
}

// GetObjectPerPodMetricReplicas calculates the desired replica count based on a target metric utilization (as a milli-value)
// for the given object in the given namespace, and the current replica count.
func (c *ReplicaCalculator) GetObjectPerPodMetricReplicas(statusReplicas int32, targetAverageUtilization int64, metricName string, namespace string, objectRef *autoscaling.CrossVersionObjectReference, metricSelector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	utilization, timestamp, err = c.metricsClient.GetObjectMetric(metricName, namespace, objectRef, metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get metric %s: %v on %s %s/%s", metricName, objectRef.Kind, namespace, objectRef.Name, err)
	}

	replicaCount = statusReplicas
	usageRatio := float64(utilization) / (float64(targetAverageUtilization) * float64(replicaCount))
	if math.Abs(1.0-usageRatio) > c.tolerance {
		// update number of replicas if change is large enough
		replicaCount = int32(math.Ceil(float64(utilization) / float64(targetAverageUtilization)))
	}
	utilization = int64(math.Ceil(float64(utilization) / float64(statusReplicas)))
	return replicaCount, utilization, timestamp, nil
}

// @TODO(mattjmcnaughton) Many different functions in this module use variations
// of this function. Make this function generic, so we don't repeat the same
// logic in multiple places.
func (c *ReplicaCalculator) getReadyPodsCount(namespace string, selector labels.Selector) (int64, error) {
	podList, err := c.podLister.Pods(namespace).List(selector)
	if err != nil {
		return 0, fmt.Errorf("unable to get pods while calculating replica count: %v", err)
	}

	if len(podList) == 0 {
		return 0, fmt.Errorf("no pods returned by selector while calculating replica count")
	}

	readyPodCount := 0

	for _, pod := range podList {
		if pod.Status.Phase == v1.PodRunning && podutil.IsPodReady(pod) {
			readyPodCount++
		}
	}

	return int64(readyPodCount), nil
}

// GetExternalMetricReplicas calculates the desired replica count based on a
// target metric value (as a milli-value) for the external metric in the given
// namespace, and the current replica count.
func (c *ReplicaCalculator) GetExternalMetricReplicas(currentReplicas int32, targetUtilization int64, metricName, namespace string, metricSelector *metav1.LabelSelector, podSelector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	metricLabelSelector, err := metav1.LabelSelectorAsSelector(metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, err
	}
	metrics, timestamp, err := c.metricsClient.GetExternalMetric(metricName, namespace, metricLabelSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get external metric %s/%s/%+v: %s", namespace, metricName, metricSelector, err)
	}
	utilization = 0
	for _, val := range metrics {
		utilization = utilization + val
	}

	usageRatio := float64(utilization) / float64(targetUtilization)
	replicaCount, timestamp, err = c.getUsageRatioReplicaCount(currentReplicas, usageRatio, namespace, podSelector)
	return replicaCount, utilization, timestamp, err
}

// GetExternalPerPodMetricReplicas calculates the desired replica count based on a
// target metric value per pod (as a milli-value) for the external metric in the
// given namespace, and the current replica count.
func (c *ReplicaCalculator) GetExternalPerPodMetricReplicas(statusReplicas int32, targetUtilizationPerPod int64, metricName, namespace string, metricSelector *metav1.LabelSelector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	metricLabelSelector, err := metav1.LabelSelectorAsSelector(metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, err
	}
	metrics, timestamp, err := c.metricsClient.GetExternalMetric(metricName, namespace, metricLabelSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get external metric %s/%s/%+v: %s", namespace, metricName, metricSelector, err)
	}
	utilization = 0
	for _, val := range metrics {
		utilization = utilization + val
	}

	replicaCount = statusReplicas
	usageRatio := float64(utilization) / (float64(targetUtilizationPerPod) * float64(replicaCount))
	if math.Abs(1.0-usageRatio) > c.tolerance {
		// update number of replicas if the change is large enough
		replicaCount = int32(math.Ceil(float64(utilization) / float64(targetUtilizationPerPod)))
	}
	utilization = int64(math.Ceil(float64(utilization) / float64(statusReplicas)))
	return replicaCount, utilization, timestamp, nil
}
```

```
// RESTMapper allows clients to map resources to kind, and map kind and version
// to interfaces for manipulating those objects. It is primarily intended for
// consumers of Kubernetes compatible REST APIs as defined in docs/devel/api-conventions.md.
//
// The Kubernetes API provides versioned resources and object kinds which are scoped
// to API groups. In other words, kinds and resources should not be assumed to be
// unique across groups.
//
// TODO: split into sub-interfaces
type RESTMapper interface {
	// KindFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	KindFor(resource schema.GroupVersionResource) (schema.GroupVersionKind, error)

	// KindsFor takes a partial resource and returns the list of potential kinds in priority order
	KindsFor(resource schema.GroupVersionResource) ([]schema.GroupVersionKind, error)

	// ResourceFor takes a partial resource and returns the single match.  Returns an error if there are multiple matches
	ResourceFor(input schema.GroupVersionResource) (schema.GroupVersionResource, error)

	// ResourcesFor takes a partial resource and returns the list of potential resource in priority order
	ResourcesFor(input schema.GroupVersionResource) ([]schema.GroupVersionResource, error)

	// RESTMapping identifies a preferred resource mapping for the provided group kind.
	RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error)
	// RESTMappings returns all resource mappings for the provided group kind if no
	// version search is provided. Otherwise identifies a preferred resource mapping for
	// the provided version(s).
	RESTMappings(gk schema.GroupKind, versions ...string) ([]*RESTMapping, error)

	ResourceSingularizer(resource string) (singular string, err error)
}
```

```
// EventRecorder knows how to record events on behalf of an EventSource.
type EventRecorder interface {
	// Event constructs an event from the given information and puts it in the queue for sending.
	// 'object' is the object this event is about. Event will make a reference-- or you may also
	// pass a reference to the object directly.
	// 'type' of this event, and can be one of Normal, Warning. New types could be added in future
	// 'reason' is the reason this event is generated. 'reason' should be short and unique; it
	// should be in UpperCamelCase format (starting with a capital letter). "reason" will be used
	// to automate handling of events, so imagine people writing switch statements to handle them.
	// You want to make that easy.
	// 'message' is intended to be human readable.
	//
	// The resulting event will be created in the same namespace as the reference object.
	Event(object runtime.Object, eventtype, reason, message string)

	// Eventf is just like Event, but with Sprintf for the message field.
	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})

	// AnnotatedEventf is just like eventf, but with annotations attached
	AnnotatedEventf(object runtime.Object, annotations map[string]string, eventtype, reason, messageFmt string, args ...interface{})
}
```
