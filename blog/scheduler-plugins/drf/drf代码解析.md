# Volcano调度器中的DRF插件
### 引言
DRF插件是Volcano实现公平调度的核心组件之一，它基于Google提出的Dominant Resource Fairness算法，能够在多租户环境中实现多维资源的公平分配。本文将通过分析DRF插件的完整源代码，深入解析其实现原理和设计思想。
### 一、DRF插件基础结构
#### 1.1 基本类型定义
DRF插件的实现始于一系列基本类型定义，这些定义构成了插件内部数据结构的基石。
```go
// PluginName indicates name of volcano scheduler plugin.
const PluginName = "drf"

var shareDelta = 0.000001

// hierarchicalNode represents the node hierarchy
// and the corresponding weight and drf attribute
type hierarchicalNode struct {
   parent *hierarchicalNode
   attr   *drfAttr
   // If the node is a leaf node,
   // request represents the request of the job.
   request   *api.Resource
   weight    float64
   saturated bool
   hierarchy string
   children  map[string]*hierarchicalNode
}
```
这里定义了插件名称常量PluginName和用于浮点数比较的容差值shareDelta。在DRF插件中，shareDelta用于在比较作业的DRF份额时提供一个容差值。具体来说，当两个作业的份额非常接近时（差异小于shareDelta），插件会将它们视为相等，从而避免因浮点数精度问题导致的不准确排序或决策。这确保了调度决策的稳定性和公平性。  
核心数据结构hierarchicalNode定义了层次化调度的节点结构，包含父节点指针、DRF属性、资源请求、权重、饱和状态等信息。
#### 1.2 节点克隆与资源判断
```go
func (node *hierarchicalNode) Clone(parent *hierarchicalNode) *hierarchicalNode {
   newNode := &hierarchicalNode{
      parent: parent,
      attr: &drfAttr{
      share:            node.attr.share,
      dominantResource: node.attr.dominantResource,
      allocated:        node.attr.allocated.Clone(),
   },
      request:   node.request.Clone(),
      weight:    node.weight,
      saturated: node.saturated,
      hierarchy: node.hierarchy,
      children:  nil,
   }
   if node.children != nil {
      newNode.children = map[string]*hierarchicalNode{}
      for _, child := range node.children {
         newNode.children[child.hierarchy] = child.Clone(newNode)
      }
   }
   return newNode
}

func resourceSaturated(allocated *api.Resource,
   jobRequest *api.Resource, demandingResources map[v1.ResourceName]bool) bool {
   for _, rn := range allocated.ResourceNames() {
      if allocated.Get(rn) != 0 && jobRequest.Get(rn) != 0 &&
         allocated.Get(rn) >= jobRequest.Get(rn) {
         return true
      }
      if !demandingResources[rn] && jobRequest.Get(rn) != 0 {
         return true
      }
   }
   return false
}
```
hierarchicalNode的Clone方法实现了节点的深拷贝，这对于保持调度状态的一致性非常重要。  
resourceSaturated函数则判断资源是否已饱和，这是决定任务是否需要被抢占的关键依据。
- allocated (*api.Resource): 
  - 表示当前已经分配给作业的资源量。这是一个资源对象，包含了各种资源类型（如 CPU、内存等）的分配情况。
- jobRequest (*api.Resource):
  - 表示作业请求的资源量。这也是一个资源对象，包含了作业所需的各种资源类型的数量。
- demandingResources (map[v1.ResourceName]bool):
  - 一个映射，键为资源名称（v1.ResourceName），值为布尔值。该映射指示哪些资源是被作业“需求”的资源。如果某个资源在映射中的值为 false，则表示该资源不是作业的必需资源，即使作业请求了该资源，也不会被视为必须满足的需求。

###### 条件解释：
- allocated.Get(rn) != 0：已分配的资源量不为零。
- jobRequest.Get(rn) != 0：作业请求的资源量不为零。
- allocated.Get(rn) >= jobRequest.Get(rn)：已分配的资源量大于或等于作业请求的资源量。

结论：如果以上三个条件同时满足，说明该资源已经被完全分配，作业无法再获取更多的该资源，因此返回 true，表示资源饱和。
###### 条件解释：
- !demandingResources[rn]：该资源不在需求资源列表中，即作业并不真正需要该资源。
- jobRequest.Get(rn) != 0：作业仍然请求了该资源。  

结论：如果作业请求了一个非需求的资源，也返回 true，表示资源饱和。这可能是为了防止作业占用不必要的资源，从而影响其他作业的资源分配。 

如果遍历完所有资源后，没有任何资源满足饱和条件，则返回 false，表示资源未饱和。
#### 1.3 DRF属性定义
```go
type drfAttr struct {
   share            float64
   dominantResource string
   allocated        *api.Resource
}

func (attr *drfAttr) String() string {
   return fmt.Sprintf("dominant resource <%s>, dominant share %f, allocated %s",
   attr.dominantResource, attr.share, attr.allocated)
}
```
drfAttr结构体存储了DRF算法所需的核心属性，包括资源份额、主导资源类型和已分配资源量。String方法提供了友好的字符串表示，便于调试和日志记录。
### 二、DRF插件核心实现
#### 2.1 插件初始化
```go
type drfPlugin struct {
   totalResource  *api.Resource
   totalAllocated *api.Resource
   jobAttrs       map[api.JobID]*drfAttr
   namespaceOpts  map[string]*drfAttr
   hierarchicalRoot *hierarchicalNode
   pluginArguments framework.Arguments
}

func New(arguments framework.Arguments) framework.Plugin {
   return &drfPlugin{
      totalResource:  api.EmptyResource(),
      totalAllocated: api.EmptyResource(),
      jobAttrs:       map[api.JobID]*drfAttr{},
      namespaceOpts:  map[string]*drfAttr{},
      hierarchicalRoot: &hierarchicalNode{
         attr:      &drfAttr{allocated: api.EmptyResource()},
         request:   api.EmptyResource(),
         hierarchy: "root",
         weight:    1,
         children:  map[string]*hierarchicalNode{},
      },
         pluginArguments: arguments,
   }
}

func (drf *drfPlugin) Name() string {
   return PluginName
}
```
drfPlugin结构体是插件的主体，包含了调度所需的所有状态信息。  
New函数负责初始化插件实例，设置初始状态。  
Name方法返回插件名称，这是Kubernetes调度框架要求的接口。
#### 2.2 层次化调度支持
```go
func (drf *drfPlugin) HierarchyEnabled(ssn *framework.Session) bool {
   for _, tier := range ssn.Tiers {
      for _, plugin := range tier.Plugins {
         if plugin.Name != PluginName {
            continue
         }
         return plugin.EnabledHierarchy != nil && *plugin.EnabledHierarchy
      }
   }
   return false
}
// 返回值为负数：表示 lqueue 应该优先于 rqueue。
// 返回值为正数：表示 rqueue 应该优先于 lqueue。
// 返回值为零：表示两个队列的优先级相同。
func (drf *drfPlugin) compareQueues(root *hierarchicalNode, lqueue *api.QueueInfo, rqueue *api.QueueInfo) float64 {
   lnode := root
   lpaths := strings.Split(lqueue.Hierarchy, "/")
   rnode := root
   rpaths := strings.Split(rqueue.Hierarchy, "/")
   //min(len(lpaths), len(rpaths)) 确定遍历的最小深度，避免因路径长度不一致导致的越界访问。
   //使用 for 循环逐层比较两个队列的节点。
   for i, depth := 0, min(len(lpaths), len(rpaths)); i < depth; i++ {
	  // 优先级规则：
      //如果左节点未饱和而右节点饱和，优先选择左节点（返回 -1）。
      //如果右节点未饱和而左节点饱和，优先选择右节点（返回 1）。
      //饱和状态的含义：
      //饱和节点表示其资源已被充分利用，无法再接受更多任务。
      //未饱和节点表示还有可用资源，可以接受更多任务。
      //目的：
      //优先选择未饱和的节点，以确保资源能够被有效利用。
      if !lnode.saturated && rnode.saturated {
         return -1
      }
      if lnode.saturated && !rnode.saturated {
         return 1
      }
      //计算 DRF 份额比率：
      //lnode.attr.share / lnode.weight：左节点的 DRF 份额比率。
      //rnode.attr.share / rnode.weight：右节点的 DRF 份额比率。
      //比较份额比率：
      //如果两个节点的 DRF 份额比率相等，且当前层级不是最后一层，则继续深入下一层进行比较。
      //如果份额比率不相等，直接返回两者的差值，作为优先级比较的结果。
      //目的：
      //通过逐层比较，综合考虑不同层级的权重和资源分配情况，确保整体的公平性。
      if lnode.attr.share/lnode.weight == rnode.attr.share/rnode.weight {
         if i < depth-1 {
            lnode = lnode.children[lpaths[i+1]]
            rnode = rnode.children[rpaths[i+1]]
         }
      } else {
         return lnode.attr.share/lnode.weight - rnode.attr.share/rnode.weight
      }
   }
   return 0
}
```
HierarchyEnabled方法检查当前调度会话是否启用了层次化调度。
compareQueues方法实现了队列间的比较逻辑，考虑了层次结构和权重因素，用于决定队列的调度顺序。
### 三、调度会话管理
#### 3.1 会话初始化(OnSessionOpen)
```go
func (drf *drfPlugin) OnSessionOpen(ssn *framework.Session) {
   // Prepare scheduling data for this session.
   drf.totalResource.Add(ssn.TotalResource)
   klog.V(4).Infof("Total Allocatable %s", drf.totalResource)

	hierarchyEnabled := drf.HierarchyEnabled(ssn)
	
	for _, job := range ssn.Jobs {
		attr := &drfAttr{
			allocated: api.EmptyResource(),
		}
		
		for status, tasks := range job.TaskStatusIndex {
			if api.AllocatedStatus(status) {
				for _, t := range tasks {
					attr.allocated.Add(t.Resreq)
				}
			}
		}
		
		drf.updateJobShare(job.Namespace, job.Name, attr)
		drf.jobAttrs[job.UID] = attr
		
		if hierarchyEnabled {
			queue := ssn.Queues[job.Queue]
			drf.totalAllocated.Add(attr.allocated)
			drf.UpdateHierarchicalShare(drf.hierarchicalRoot, drf.totalAllocated, job, attr, queue.Hierarchy, queue.Weights)
		}
	}
	
	// 设置抢占回调函数
	preemptableFn := func(preemptor *api.TaskInfo, preemptees []*apiTaskInfo) ([]*api.TaskInfo, int) {
		// 抢占逻辑实现
	}
	ssn.AddPreemptableFn(drf.Name(), preemptableFn)
	
	// 如果启用层次化调度，设置队列排序和资源回收回调
	if hierarchyEnabled {
		queueOrderFn := func(l interface{}, r interface{}) int {
			// 队列排序逻辑
		}
		ssn.AddQueueOrderFn(drf.Name(), queueOrderFn)
		
		reclaimFn := func(reclaimer *api.TaskInfo, reclaimees []*apiTaskInfo) ([]*api.TaskInfo, int) {
			// 资源回收逻辑
		}
		ssn.AddReclaimableFn(drf.Name(), reclaimFn)
	}
	
	// 设置作业排序回调
	jobOrderFn := func(l interface{}, r interface{}) int {
		// 作业排序逻辑
	}
	ssn.AddJobOrderFn(drf.Name(), jobOrderFn)
	
	// 注册事件处理器
	ssn.AddEventHandler(&framework.EventHandler{
		AllocateFunc: func(event *framework.Event) {
			// 处理资源分配事件
		},
		DeallocateFunc: func(event *framework.Event) {
			// 处理资源释放事件
		},
	})
}

func (drf *drfPlugin) buildHierarchy(root *hierarchicalNode, job *api.JobInfo, attr *drfAttr, hierarchy, hierarchicalWeights string) {
   // 构建层次结构
}

func (drf *drfPlugin) updateHierarchicalShare(node *hierarchicalNode, demandingResources map[v1.ResourceName]bool) {
   // 更新层次化份额
}

func (drf *drfPlugin) UpdateHierarchicalShare(root *hierarchicalNode, totalAllocated *api.Resource, job *api.JobInfo, attr *drfAttr, hierarchy, hierarchicalWeights string) {
   // 更新层次化份额(带参数)
}

func (drf *drfPlugin) updateJobShare(jobNs, jobName string, attr *drfAttr) {
   // 更新作业份额
}

func (drf *drfPlugin) updateShare(attr *drfAttr) {
   // 更新份额
}
```
OnSessionOpen方法是DRF插件的核心初始化方法，它:

- 准备调度数据，包括总资源和各作业初始状态
- 设置各种回调函数(抢占、回收、排序等)
- 注册事件处理器响应资源变化

辅助方法如buildHierarchy、updateHierarchicalShare等实现了层次化调度的关键逻辑，确保资源能在不同层次的队列间公平分配。
#### 3.2 份额计算核心算法
```go
func (drf *drfPlugin) calculateShare(allocated, totalResource *api.Resource) (string, float64) {
   res := float64(0)
   dominantResource := ""
   for _, rn := range totalResource.ResourceNames() {
      share := helpers.Share(allocated.Get(rn), totalResource.Get(rn))
      if share > res {
         res = share
         dominantResource = string(rn)
      }
   }
   return dominantResource, res
}
```
calculateShare方法是DRF算法的核心，它:

- 遍历所有资源类型
- 计算每种资源的分配份额(已分配量/总量)
- 确定份额最大的资源作为主导资源
- 返回主导资源名称和对应份额

这个方法体现了DRF算法的核心思想:关注作业对哪种资源的需求比例最大，确保所有作业的主导资源份额趋于公平。
#### 3.3 会话关闭清理(OnSessionClose)
```go
func (drf *drfPlugin) OnSessionClose(session *framework.Session) {
   // Clean schedule data.
   drf.totalResource = api.EmptyResource()
   drf.totalAllocated = api.EmptyResource()
   drf.jobAttrs = map[api.JobID]*drfAttr{}
}
```
OnSessionClose方法负责清理调度会话使用的资源:

- 重置总资源记录
- 清空已分配资源记录
- 清除所有作业属性

这确保了插件状态的干净释放，为下一次调度会话做好准备。
### 四、DRF插件优势与应用场景
通过上述代码分析，我们可以总结DRF插件的主要优势:

- 多维资源公平性: 同时考虑CPU、内存等多种资源，实现真正的多维公平调度
- 层次化支持: 支持基于命名空间或队列的层次化资源分配策略
- 动态适应性: 能够根据作业资源需求变化动态调整调度决策
- 抢占与回收: 支持资源抢占和回收机制，提高集群资源利用率

#### DRF插件特别适合以下场景:

- 多租户共享集群环境
- 混合工作负载(CPU密集型和内存密集型混合)
- 需要层次化资源管理的组织结构

### 五、总结
Volcano调度器的DRF插件实现了一个完整的多资源公平调度系统。通过深入分析其源代码，我们不仅理解了DRF算法的具体实现细节，还学习了如何设计一个生产级的调度插件。
DRF算法通过关注作业的主导资源，实现了多维资源的公平分配，特别适合混合工作负载的场景。Volcano的实现还通过层次化调度、事件驱动架构等设计，使其能够适应复杂的集群环境和调度需求。
对于Kubernetes用户和开发者来说，理解DRF插件的工作原理有助于:

- 更有效地配置和使用Volcano调度器
- 诊断和解决资源分配问题
- 设计更适合自身工作负载的调度策略
