**源代码** https://github.com/volcano-sh/volcano/blob/master/pkg/scheduler/plugins/capacity/capacity.go

## Volcano调度插件深度解析：Capacity插件的工作原理与实现
Volcano作为Kubernetes生态中领先的批处理调度引擎，其强大功能很大程度上得益于其插件化架构设计。今天，我们将深入解析Volcano项目中一个核心调度插件——Capacity插件的源代码，帮助初学者理解其工作原理和实现机制。
### 一、Capacity插件概述
Capacity插件是Volcano调度器中的一个重要组件，它主要负责实现*弹性队列容量管理*功能。在Volcano v1.9.0版本中，该插件被引入以替代原有的proportion插件，提供了更直观、更灵活的队列资源管理能力。
简单来说，Capacity插件允许管理员直接为队列设置每一维度资源（如CPU、内存、GPU等）的具体容量值，而不是通过抽象的weight权重来分配资源。这种设计使得资源分配更加直观和精确，特别适合需要精细控制各种资源类型的场景，如AI大模型训练中需要为不同GPU型号（如A100和V100）设置独立配额的情况。
### 二、Capacity插件的工作原理
Capacity插件的核心思想是基于deserved（应得资源量）的弹性容量调度，它支持：      
- 精确资源分配：管理员可以直接为队列设置CPU、内存、GPU等每一维度资源的容量上限
- 弹性资源共享：当集群资源空闲时，队列可以借用其他队列的空闲资源
- 资源回收机制：当资源紧张时，系统会逐步回收借出的资源，直到各队列回到其"应得资源量"

这种机制完美解决了传统按权重分配资源不够直观、无法为不同资源类型单独设置配额的问题。例如，在AI训练场景中，可以为一个队列设置40张A100 GPU和80张V100 GPU的容量，而不必担心资源分配不均衡的问题。
### 三、代码结构解析
让我们来看一下Capacity插件的主要代码结构：
```go
// Capacity插件实现的主要接口
type capacityPlugin struct {
    // 插件参数
    pluginArguments framework.Arguments
    // 各队列的应得资源量
    queueDeserved map[api.QueueID]*api.Resource
    // 各队列的当前实际分配量
    queueAllocated map[api.QueueID]*api.Resource
    // 集群总资源
    totalResource *api.Resource
    // 其他内部状态...
}

// Name返回插件名称
func (cp *capacityPlugin) Name() string {
    return PluginName
}

// OnSessionOpen在调度会话开始时调用
func (cp *capacityPlugin) OnSessionOpen(ssn *framework.Session) {
    // 初始化插件状态
    cp.totalResource = ssn.TotalResource

    // 注册回调函数
    ssn.AddQueueOrderFn(cp.Name(), cp.QueueOrderFn)
    ssn.AddQueueCompareFn(cp.Name(), cp.QueueCompareFn)
    ssn.AddAllocatableFn(cp.Name(), cp.AllocatableFn)
    ssn.AddOverusedFn(cp.Name(), cp.OverusedFn)
    // 其他回调注册...
}

// OnSessionClose在调度会话结束时调用
func (cp *capacityPlugin) OnSessionClose(ssn *framework.Session) {
    // 清理资源...
}
```
#### 结构体参数作用
###### 1. rootQueue（根队列标识）
- 作用：指定层级队列体系中的根队列名称，用于构建队列层级关系（如ParentQueue和LeafQueue的父子关系）。
- 背景：在Volcano v1.9.0后，社区支持了层级队列功能，rootQueue是层级调度的起点，确保资源从根队列向下分配。
###### 2. totalResource（集群总资源）
- 作用：记录集群所有节点的可用资源总量（如CPU、内存、GPU等），是计算队列资源配额（如deserved）的基准。
- 示例：若集群总CPU为100核，队列A的deserved设为40核，则其初始应得资源占比40%。
###### 3. totalGuarantee（总保障资源）
- 作用：存储所有队列配置的最低保障资源（guarantee字段）之和，用于确保队列在资源竞争时至少能获得这部分资源。
- 场景：当集群资源紧张时，优先满足各队列的guarantee值，剩余资源再按deserved比例分配。
###### 4. queueOpts（队列属性映射）
- 作用：以QueueID为键，存储每个队列的动态调度属性，包括：
  - deserved：队列应得资源量（用户显式配置的容量上限）。
  - allocated：队列当前实际分配的资源量。
  - guarantee：队列的最低保障资源量（防止资源饥饿）。
- 功能：插件通过对比allocated与deserved，决定是否允许队列继续分配资源或触发资源回收。
###### 5. pluginArguments（插件参数）
- 作用：接收调度器配置文件中传递给capacity插件的自定义参数，用于动态调整插件行为。
- 示例：可通过参数指定是否启用弹性共享、资源回收策略等高级功能。
###### 参数协同工作流程
- 初始化阶段：插件从集群和队列配置中加载totalResource、totalGuarantee及queueOpts。
- 调度决策：比较队列的allocated与deserved，结合totalResource剩余量，决定是否分配资源。
- 资源回收：当集群资源不足时，优先回收超过deserved的队列资源，确保各队列回归应得量
#### 关键组件解析
- Plugin接口实现：Capacity插件实现了Volcano的标准Plugin接口，包括Name()、OnSessionOpen()和OnSessionClose()方法。
- 回调函数注册：在OnSessionOpen中，插件向调度会话注册了多个关键回调函数，这些函数将在调度过程的不同阶段被调用：
  - QueueOrderFn：决定队列的调度顺序
  - QueueCompareFn：比较两个队列的优先级
  - AllocatableFn：判断队列是否还可分配更多资源
  - OverusedFn：判断队列是否已过度使用资源
- 资源跟踪：插件维护了几个重要映射表：
   - queueDeserved：记录每个队列的"应得资源量"
   - queueAllocated：记录每个队列当前实际分配的资源量
   - totalResource：集群总资源量

### 四、核心算法实现
Capacity插件的核心算法主要体现在几个关键回调函数中：
#### 1. 队列排序(QueueOrderFn)
```go
func (cp *capacityPlugin) QueueOrderFn(l, r interface{}) int {
   lv := l.(*api.QueueInfo)
   rv := r.(*api.QueueInfo)
   
   // 比较两个队列的已分配资源与应得资源的比例
   lRes := cp.queueAllocated[lv.UID].MilliCPU / cp.queueDeserved[lv.UID].MilliCPU
   rRes := cp.queueAllocated[rv.UID].MilliCPU / cp.queueDeserved[rv.UID].MilliCPU
   
   if lRes < rRes {
      return -1
   } else if lRes > rRes {
      return 1
   }
      return 0
}
```
这段代码实现了队列的排序逻辑，基本原则是：***已分配资源与应得资源比例较低的队列优先调度***。这确保了资源分配的公平性，防止某些队列过度占用资源。
#### 2. 可分配性检查(AllocatableFn)
```go
func (cp *capacityPlugin) AllocatableFn(queue *api.QueueInfo, candidate *api.TaskInfo) bool {
   // 计算队列已分配资源加上候选任务需求后的总量
   allocated := cp.queueAllocated[queue.UID].Clone()
   allocated.Add(candidate.ResReq)
   
   // 检查是否超过应得资源量
   return !allocated.Less(cp.queueDeserved[queue.UID])
}
```
这个函数***判断队列是否可以分配更多资源给候选任务***。当队列的已分配资源加上候选任务的需求仍小于应得资源量时，返回true，表示可以分配。
#### 3. 过度使用检查(OverusedFn)
```go
func (cp *capacityPlugin) OverusedFn(queue *api.QueueInfo) bool {
   // 检查队列已分配资源是否超过应得资源量
   return cp.queueAllocated[queue.UID].Less(cp.queueDeserved[queue.UID])
}
```
这个函数***判断队列是否已经过度使用了资源***。当队列的已分配资源超过应得资源量时，返回true，表示该队列已过度使用资源，可能需要回收部分资源。
#### 五、Capacity插件的配置与使用
要使用Capacity插件，需要在Volcano的调度器配置中进行相应设置。以下是一个典型的配置示例：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
   name: volcano-scheduler-configmap
   namespace: volcano-system
data:
   volcano-scheduler.conf: |
      actions: "enqueue, allocate, backfill"
      tiers:
      - plugins:
         - name: priority
         - name: gang
         - name: conformance
      - plugins:
         - name: drf
         - name: predicates
         - name: capacity  # 使用capacity插件替代proportion
         - name: nodeorder
         - name: binpack
```
同时，队列的定义也需要设置deserved字段来指定各维度资源的应得量：
```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
   name: demo-queue
spec:
   reclaimable: true
   deserved:  # 设置应得资源量
      cpu: 64
      memory: 128Gi
      nvidia.com/a100: 40
      nvidia.com/v100: 80
```
这种配置方式比原来的weight权重方式更加直观和灵活。
#### 六、Capacity插件的应用场景
Capacity插件特别适合以下场景：

- AI训练任务：需要为不同GPU型号设置独立配额，如为A100和V100 GPU分别设置容量
- 多部门资源共享：不同部门使用不同类型的计算资源，需要精确控制每种资源的分配量
- 弹性资源需求：业务负载波动大，需要在不影响核心业务的情况下弹性共享资源
- 混合部署环境：在线业务和离线批处理作业混合部署，需要确保关键业务有足够的资源保障

#### 七、Capacity插件的优势
相比于传统的proportion插件，Capacity插件具有以下优势：

- 直观的资源分配：直接设置具体资源量而非抽象权重
- 多维资源独立控制：可以为CPU、内存、GPU等不同资源类型设置独立配额
- 弹性资源共享：空闲资源可被其他队列借用，提高利用率
- 公平的资源回收：当资源紧张时，系统会公平地回收借出的资源

#### 八、总结
Volcano的Capacity插件通过引入"应得资源量"(deserved)的概念，实现了比传统权重分配更直观、更精确的队列资源管理机制。其核心思想是允许队列在资源充足时借用其他队列的空闲资源，在资源紧张时又能公平地回收资源，既提高了集群资源利用率，又确保了各队列的基本资源保障。
通过分析源代码，我们了解了Capacity插件的整体结构、核心算法和关键实现细节。这种插件化设计是Volcano调度器的一大特色，使得Volcano能够灵活支持各种复杂的批处理调度场景。
对于想要深入了解Volcano调度机制或开发自定义插件的开发者来说，理解Capacity插件的实现原理是一个很好的起点。希望这篇解析能帮助你更好地理解和使用Volcano这一强大的批处理调度引擎。