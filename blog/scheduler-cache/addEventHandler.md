源代码位置  ***pkg/scheduler/cache/cache.go***
```go
func (sc *SchedulerCache) addEventHandler() {
	informerFactory := informers.NewSharedInformerFactory(sc.kubeClient, 0)
	sc.informerFactory = informerFactory

	// explicitly register informers to the factory, otherwise resources listers cannot get anything
	// even with no error returned.
	// `Namespace` informer is used by `InterPodAffinity` plugin,
	// `SelectorSpread` and `PodTopologySpread` plugins uses the following four so far.
	informerFactory.Core().V1().Namespaces().Informer()
	informerFactory.Core().V1().Services().Informer()
	if utilfeature.DefaultFeatureGate.Enabled(features.WorkLoadSupport) {
		informerFactory.Core().V1().ReplicationControllers().Informer()
		informerFactory.Apps().V1().ReplicaSets().Informer()
		informerFactory.Apps().V1().StatefulSets().Informer()
	}

	// `PodDisruptionBudgets` informer is used by `Pdb` plugin
	if utilfeature.DefaultFeatureGate.Enabled(features.PodDisruptionBudgetsSupport) {
		informerFactory.Policy().V1().PodDisruptionBudgets().Informer()
	}

	// create informer for node information
	sc.nodeInformer = informerFactory.Core().V1().Nodes()
	sc.nodeInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Node:
					return true
				case cache.DeletedFinalStateUnknown:
					var ok bool
					_, ok = t.Obj.(*v1.Node)
					if !ok {
						klog.Errorf("Cannot convert to *v1.Node: %v", t.Obj)
						return false
					}
					return true
				default:
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sc.AddNode,
				UpdateFunc: sc.UpdateNode,
				DeleteFunc: sc.DeleteNode,
			},
		},
		0,
	)

	sc.podInformer = informerFactory.Core().V1().Pods()
	sc.pvcInformer = informerFactory.Core().V1().PersistentVolumeClaims()
	sc.pvInformer = informerFactory.Core().V1().PersistentVolumes()
	sc.scInformer = informerFactory.Storage().V1().StorageClasses()
	sc.csiNodeInformer = informerFactory.Storage().V1().CSINodes()
	sc.csiNodeInformer.Informer().AddEventHandler(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    sc.AddOrUpdateCSINode,
			UpdateFunc: sc.UpdateCSINode,
			DeleteFunc: sc.DeleteCSINode,
		},
	)

	if options.ServerOpts != nil && options.ServerOpts.EnableCSIStorage && utilfeature.DefaultFeatureGate.Enabled(features.CSIStorage) {
		sc.csiDriverInformer = informerFactory.Storage().V1().CSIDrivers()
		sc.csiStorageCapacityInformer = informerFactory.Storage().V1beta1().CSIStorageCapacities()
	}

	// create informer for pod information
	sc.podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch v := obj.(type) {
				case *v1.Pod:
					if !responsibleForPod(v, sc.schedulerNames, sc.schedulerPodName, sc.c) {
						if len(v.Spec.NodeName) == 0 {
							return false
						}
						if !responsibleForNode(v.Spec.NodeName, sc.schedulerPodName, sc.c) {
							return false
						}
					}
					return true
				case cache.DeletedFinalStateUnknown:
					if _, ok := v.Obj.(*v1.Pod); ok {
						// The carried object may be stale, always pass to clean up stale obj in event handlers.
						return true
					}
					klog.Errorf("Cannot convert object %T to *v1.Pod", v.Obj)
					return false
				default:
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sc.AddPod,
				UpdateFunc: sc.UpdatePod,
				DeleteFunc: sc.DeletePod,
			},
		})

	if options.ServerOpts != nil && options.ServerOpts.EnablePriorityClass && utilfeature.DefaultFeatureGate.Enabled(features.PriorityClass) {
		sc.pcInformer = informerFactory.Scheduling().V1().PriorityClasses()
		sc.pcInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
			AddFunc:    sc.AddPriorityClass,
			UpdateFunc: sc.UpdatePriorityClass,
			DeleteFunc: sc.DeletePriorityClass,
		})
	}

	sc.quotaInformer = informerFactory.Core().V1().ResourceQuotas()
	sc.quotaInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    sc.AddResourceQuota,
		UpdateFunc: sc.UpdateResourceQuota,
		DeleteFunc: sc.DeleteResourceQuota,
	})

	vcinformers := vcinformer.NewSharedInformerFactory(sc.vcClient, 0)
	sc.vcInformerFactory = vcinformers

	// create informer for PodGroup(v1beta1) information
	sc.podGroupInformerV1beta1 = vcinformers.Scheduling().V1beta1().PodGroups()
	sc.podGroupInformerV1beta1.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				var pg *vcv1beta1.PodGroup
				switch v := obj.(type) {
				case *vcv1beta1.PodGroup:
					pg = v
				case cache.DeletedFinalStateUnknown:
					var ok bool
					pg, ok = v.Obj.(*vcv1beta1.PodGroup)
					if !ok {
						klog.Errorf("Cannot convert to podgroup: %v", v.Obj)
						return false
					}
				default:
					return false
				}

				return responsibleForPodGroup(pg, sc.schedulerPodName, sc.c)
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    sc.AddPodGroupV1beta1,
				UpdateFunc: sc.UpdatePodGroupV1beta1,
				DeleteFunc: sc.DeletePodGroupV1beta1,
			},
		})

	// create informer(v1beta1) for Queue information
	sc.queueInformerV1beta1 = vcinformers.Scheduling().V1beta1().Queues()
	sc.queueInformerV1beta1.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    sc.AddQueueV1beta1,
		UpdateFunc: sc.UpdateQueueV1beta1,
		DeleteFunc: sc.DeleteQueueV1beta1,
	})

	if utilfeature.DefaultFeatureGate.Enabled(features.ResourceTopology) {
		sc.cpuInformer = vcinformers.Nodeinfo().V1alpha1().Numatopologies()
		sc.cpuInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
			AddFunc:    sc.AddNumaInfoV1alpha1,
			UpdateFunc: sc.UpdateNumaInfoV1alpha1,
			DeleteFunc: sc.DeleteNumaInfoV1alpha1,
		})
	}
}

```

该函数是Volcano调度器（SchedulerCache）中用于注册事件处理程序的核心逻辑，主要功能是通过Kubernetes的Informer机制监听各类资源（如Pod、Node、Queue等）的变化，并绑定对应的事件处理函数。以下从关键模块和设计意图进行解读：

#### 1. Informer初始化与工厂模式

- SharedInformerFactory：通过informers.NewSharedInformerFactory创建共享的Informer工厂，减少重复的ListAndWatch操作，优化API Server负载。参数sc.kubeClient是Kubernetes客户端，0表示禁用定期全量同步（resync）。
- 显式注册资源Informer：即使某些Informer未直接绑定事件处理器（如Namespace、Service等），仍需显式注册以确保其缓存被正确初始化，供插件（如InterPodAffinity）使用。


#### 2. 核心资源的事件监听
###### Node资源

- 过滤逻辑：通过FilterFunc确保仅处理*v1.Node类型对象或删除状态的节点。
- 事件处理：

    - AddFunc: sc.AddNode（新增节点）
    - UpdateFunc: sc.UpdateNode（节点更新）
    - DeleteFunc: sc.DeleteNode（节点删除）

###### Pod资源

- 责任过滤：responsibleForPod函数确保调度器仅处理其负责的Pod（如匹配schedulerNames或特定节点）。
- 事件处理：绑定Pod的增删改操作到对应方法（如sc.AddPod），用于维护调度器的本地缓存状态。

###### 存储相关资源

- PV/PVC/StorageClass：监听持久化存储资源变化，支持动态卷调度。
- CSI资源：若启用CSIStorage特性，监听CSIDriver和CSIStorageCapacity，用于CSI存储插件集成。


#### 3. Volcano特有资源处理
###### PodGroup与Queue
- PodGroup：通过vcv1beta1.PodGroup的Informer监听任务组变化，过滤非本调度器管理的组（responsibleForPodGroup），并绑定增删改事件。
- Queue：直接监听队列资源（vcv1beta1.Queue），更新队列的配额和状态。

###### NUMA拓扑（可选）
- 若启用ResourceTopology特性，监听Numatopologies资源，用于NUMA感知调度。


#### 4. 动态特性开关

- 条件注册：通过utilfeature.DefaultFeatureGate控制特定Informer的注册，例如：

    - WorkLoadSupport：启用时监听ReplicationController、ReplicaSet、StatefulSet。
    - PodDisruptionBudgetsSupport：启用时监听PodDisruptionBudgets。
    - PriorityClass：启用时监听优先级类资源。

#### 5. 事件处理模式

- FilteringResourceEventHandler：组合过滤与处理逻辑，避免无效事件触发业务逻辑。
- ResourceEventHandlerFuncs：直接绑定增删改事件的通用模式，如处理ResourceQuota的更新。


##### 设计意图与优化

- 性能优化：

    - 共享Informer减少API Server压力。
    - 过滤机制降低无效事件处理开销（如非本调度器的Pod）。


- 扩展性：

    - 通过特性开关（FeatureGate）动态启用高级功能（如CSI、NUMA）。
    - 插件化设计（如InterPodAffinity依赖Namespace Informer）。


- 状态一致性：

    - 本地缓存与事件处理协同，确保调度器视图与集群状态同步。

