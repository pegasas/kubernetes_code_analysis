# podManager 源码分析

podManager存储和管理对Pod的访问，并维护静态容器和镜像容器之间的映射。
kubelet从3个来源发现pod更新：file，http和apiserver。 来自非API服务器来源的Pod称为静态Pod和API
//服务器不知道静态容器的存在。 为了监控
//此类容器的状态，kubelet为每个静态对象创建一个镜像容器
//通过API服务器的pod。


//镜像容器与其静态名称具有相同的容器全名（名称和名称空间）
//对应项（尽管元数据不同，例如UID等）。 通过利用
//事实上，kubelet使用广告连播全名（即
//镜像窗格的状态始终反映静态对象的实际状态
// pod。 删除静态Pod时，关联的孤立镜像Pod
//也将被删除。

```
// Manager stores and manages access to pods, maintaining the mappings
// between static pods and mirror pods.
//
// The kubelet discovers pod updates from 3 sources: file, http, and
// apiserver. Pods from non-apiserver sources are called static pods, and API
// server is not aware of the existence of static pods. In order to monitor
// the status of such pods, the kubelet creates a mirror pod for each static
// pod via the API server.
//
// A mirror pod has the same pod full name (name and namespace) as its static
// counterpart (albeit different metadata such as UID, etc). By leveraging the
// fact that the kubelet reports the pod status using the pod full name, the
// status of the mirror pod always reflects the actual status of the static
// pod. When a static pod gets deleted, the associated orphaned mirror pod
// will also be removed.
type Manager interface {
	// GetPods returns the regular pods bound to the kubelet and their spec.
	GetPods() []*v1.Pod
	// GetPodByFullName returns the (non-mirror) pod that matches full name, as well as
	// whether the pod was found.
	GetPodByFullName(podFullName string) (*v1.Pod, bool)
	// GetPodByName provides the (non-mirror) pod that matches namespace and
	// name, as well as whether the pod was found.
	GetPodByName(namespace, name string) (*v1.Pod, bool)
	// GetPodByUID provides the (non-mirror) pod that matches pod UID, as well as
	// whether the pod is found.
	GetPodByUID(types.UID) (*v1.Pod, bool)
	// GetPodByMirrorPod returns the static pod for the given mirror pod and
	// whether it was known to the pod manager.
	GetPodByMirrorPod(*v1.Pod) (*v1.Pod, bool)
	// GetMirrorPodByPod returns the mirror pod for the given static pod and
	// whether it was known to the pod manager.
	GetMirrorPodByPod(*v1.Pod) (*v1.Pod, bool)
	// GetPodsAndMirrorPods returns the both regular and mirror pods.
	GetPodsAndMirrorPods() ([]*v1.Pod, []*v1.Pod)
	// SetPods replaces the internal pods with the new pods.
	// It is currently only used for testing.
	SetPods(pods []*v1.Pod)
	// AddPod adds the given pod to the manager.
	AddPod(pod *v1.Pod)
	// UpdatePod updates the given pod in the manager.
	UpdatePod(pod *v1.Pod)
	// DeletePod deletes the given pod from the manager.  For mirror pods,
	// this means deleting the mappings related to mirror pods.  For non-
	// mirror pods, this means deleting from indexes for all non-mirror pods.
	DeletePod(pod *v1.Pod)
	// GetOrphanedMirrorPodNames returns names of orphaned mirror pods
	GetOrphanedMirrorPodNames() []string
	// TranslatePodUID returns the actual UID of a pod. If the UID belongs to
	// a mirror pod, returns the UID of its static pod. Otherwise, returns the
	// original UID.
	//
	// All public-facing functions should perform this translation for UIDs
	// because user may provide a mirror pod UID, which is not recognized by
	// internal Kubelet functions.
	TranslatePodUID(uid types.UID) kubetypes.ResolvedPodUID
	// GetUIDTranslations returns the mappings of static pod UIDs to mirror pod
	// UIDs and mirror pod UIDs to static pod UIDs.
	GetUIDTranslations() (podToMirror map[kubetypes.ResolvedPodUID]kubetypes.MirrorPodUID, mirrorToPod map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID)
	// IsMirrorPodOf returns true if mirrorPod is a correct representation of
	// pod; false otherwise.
	IsMirrorPodOf(mirrorPod, pod *v1.Pod) bool

	MirrorClient
}
```

```
// basicManager is a functional Manager.
//
// All fields in basicManager are read-only and are updated calling SetPods,
// AddPod, UpdatePod, or DeletePod.
type basicManager struct {
	// Protects all internal maps.
	lock sync.RWMutex

	// Regular pods indexed by UID.
	podByUID map[kubetypes.ResolvedPodUID]*v1.Pod
	// Mirror pods indexed by UID.
	mirrorPodByUID map[kubetypes.MirrorPodUID]*v1.Pod

	// Pods indexed by full name for easy access.
	podByFullName       map[string]*v1.Pod
	mirrorPodByFullName map[string]*v1.Pod

	// Mirror pod UID to pod UID map.
	translationByUID map[kubetypes.MirrorPodUID]kubetypes.ResolvedPodUID

	// basicManager is keeping secretManager and configMapManager up-to-date.
	secretManager    secret.Manager
	configMapManager configmap.Manager

	// A mirror pod client to create/delete mirror pods.
	MirrorClient
}
```
