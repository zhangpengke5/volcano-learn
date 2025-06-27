## 深入理解Volcano调度插件：deviceshare

#### 引言
deviceshare插件是Volcano调度器的一个扩展组件，专门用于管理和调度GPU等设备资源。在现代计算环境中，GPU已经成为加速机器学习、深度学习和其他计算密集型任务的关键硬件。然而，GPU资源的管理和分配并不像CPU那样简单，因为：

- GPU资源通常是稀缺且昂贵的
- 不同的工作负载可能需要不同数量的GPU资源
- 有些工作负载可能需要共享GPU资源以提高利用率

deviceshare插件正是为了解决这些问题而设计的，它提供了灵活的GPU资源管理和调度能力。

#### 插件的基本结构
先从代码的整体结构开始了解：
```go
import (
	v1 "k8s.io/api/core/v1"
	"k8s.io/klog/v2"
	k8sframework "k8s.io/kubernetes/pkg/scheduler/framework"

	"volcano.sh/volcano/pkg/scheduler/api"
	"volcano.sh/volcano/pkg/scheduler/api/devices"
	"volcano.sh/volcano/pkg/scheduler/api/devices/nvidia/gpushare"
	"volcano.sh/volcano/pkg/scheduler/api/devices/nvidia/vgpu"
	"volcano.sh/volcano/pkg/scheduler/framework"
)
```
这段代码导入了必要的包，包括Kubernetes的核心API、Volcano的调度框架API，以及特定于NVIDIA GPU设备的包。

#### 插件常量定义
```go
const (
    PluginName = "deviceshare"
    // GPUSharingPredicate is the key for enabling GPU Sharing Predicate in YAML
    GPUSharingPredicate = "deviceshare.GPUSharingEnable"
    NodeLockEnable      = "deviceshare.NodeLockEnable"
    GPUNumberPredicate  = "deviceshare.GPUNumberEnable"

	VGPUEnable = "deviceshare.VGPUEnable"

	SchedulePolicyArgument = "deviceshare.SchedulePolicy"
	ScheduleWeight         = "deviceshare.ScheduleWeight"
)
```
这里定义了一些常量，它们用于：

- PluginName: 插件的名称，用于在Volcano配置中引用
- 各种启用标志的键名，用于在YAML配置中控制插件的行为
- 调度策略和权重的参数名

#### 设备共享插件结构体
```go
type deviceSharePlugin struct {
    // Arguments given for the plugin
    pluginArguments framework.Arguments
    schedulePolicy  string
    scheduleWeight  int
}
```
deviceSharePlugin结构体是插件的核心，它包含：

- pluginArguments: 插件接收的配置参数
- schedulePolicy: 调度策略，决定如何分配设备资源
- scheduleWeight: 调度权重，影响设备资源的分配优先级

#### 插件初始化
```go
// New return priority plugin
func New(arguments framework.Arguments) framework.Plugin {
    dsp := &deviceSharePlugin{pluginArguments: arguments, schedulePolicy: "", scheduleWeight: 0}
    enablePredicate(dsp)
    return dsp
}

func (dp *deviceSharePlugin) Name() string {
    return PluginName
}
```
New函数是插件的构造函数，它接收配置参数并初始化插件实例。Name方法返回插件的名称，这是Volcano识别插件所必需的。
启用谓词配置
```go
func enablePredicate(dsp *deviceSharePlugin) {
    // Checks whether predicate.GPUSharingEnable is provided or not, if given, modifies the value in predicateEnable struct.
    nodeLockEnable := false
    args := dsp.pluginArguments
    args.GetBool(&gpushare.GpuSharingEnable, GPUSharingPredicate)
    args.GetBool(&gpushare.GpuNumberEnable, GPUNumberPredicate)
    args.GetBool(&nodeLockEnable, NodeLockEnable)
    args.GetBool(&vgpu.VGPUEnable, VGPUEnable)
    
    gpushare.NodeLockEnable = nodeLockEnable
    vgpu.NodeLockEnable = nodeLockEnable
    
    _, ok := args[SchedulePolicyArgument]
    if ok {
        dsp.schedulePolicy = args[SchedulePolicyArgument].(string)
    }
    args.GetInt(&dsp.scheduleWeight, ScheduleWeight)
    
    if gpushare.GpuSharingEnable && gpushare.GpuNumberEnable {
        klog.Fatal("can not define true in both gpu sharing and gpu number")
    }
    if (gpushare.GpuSharingEnable || gpushare.GpuNumberEnable) && vgpu.VGPUEnable {
        klog.Fatal("gpu-share and vgpu can't be used together")
    }
}
```
enablePredicate函数是插件的核心配置部分，它：

- 从配置参数中读取各种启用标志
- 设置节点锁定标志
- 读取调度策略和权重
- 进行一些互斥性检查，确保不会同时启用冲突的功能

***特别值得注意的是，代码中明确禁止了同时启用GPU共享和VGPU功能，因为它们是互斥的资源管理方式。***
#### 设备评分机制
```go
func getDeviceScore(ctx context.Context, pod *v1.Pod, node *api.NodeInfo, schedulePolicy string) (int64, *k8sframework.Status) {
    s := float64(0)
    for _, devices := range node.Others {
        if devices.(api.Devices).HasDeviceRequest(pod) {
            ns := devices.(api.Devices).ScoreNode(pod, schedulePolicy)
            s += ns
        }
    }
    klog.V(4).Infof("deviceScore for task %s/%s is: %v", pod.Namespace, pod.Name, s)
    return int64(math.Floor(s + 0.5)), nil
}
```
getDeviceScore函数计算节点对特定Pod的设备资源评分。它：

- 遍历节点上的所有设备资源
- 检查Pod是否请求了该设备
- 如果请求了，则调用设备的评分方法计算分数
- 累加所有设备的分数
- 返回四舍五入后的整数值

***这个评分机制允许Volcano根据设备资源的可用性和Pod的需求来做出更智能的调度决策。***

#### 插件生命周期方法
###### OnSessionOpen
```go
func (dp *deviceSharePlugin) OnSessionOpen(ssn *framework.Session) {
    // Register event handlers to update task info in PodLister & nodeMap
    ssn.AddPredicateFn(dp.Name(), func(task *api.TaskInfo, node *api.NodeInfo) error {
        predicateStatus := make([]*api.Status, 0)
        // Check PredicateWithCache
        for _, val := range api.RegisteredDevices {
            if dev, ok := node.Others[val].(api.Devices); ok {
                if reflect.ValueOf(dev).IsNil() {
                    // TODO When a pod requests a device of the current type, but the current node does not have such a device, an error is thrown
                    if dev == nil || dev.HasDeviceRequest(task.Pod) {
                        predicateStatus = append(predicateStatus, &api.Status{
                            Code:   devices.Unschedulable,
                            Reason: "node not initialized with device" + val,
                            Plugin: PluginName,
                        })
                        return api.NewFitErrWithStatus(task, node, predicateStatus...)
                    }
                    klog.V(4).Infof("pod %s/%s did not request device %s on %s, skipping it", task.Pod.Namespace, task.Pod.Name, val, node.Name)
                    continue
                }
                code, msg, err := dev.FilterNode(task.Pod, dp.schedulePolicy)
                if err != nil {
                    predicateStatus = append(predicateStatus, createStatus(code, msg))
                    return api.NewFitErrWithStatus(task, node, predicateStatus...)
                }
                filterNodeStatus := createStatus(code, msg)
                if filterNodeStatus.Code != api.Success {
                    predicateStatus = append(predicateStatus, filterNodeStatus)
                    return api.NewFitErrWithStatus(task, node, predicateStatus...)
                }
            } else {
                klog.Warningf("Devices %s assertion conversion failed, skip", val)
            }
        }
        
        klog.V(4).Infof("checkDevices predicates Task <%s/%s> on Node <%s>: fit ",
        task.Namespace, task.Name, node.Name)
        
        return nil
    })
    
    ssn.AddNodeOrderFn(dp.Name(), func(task *api.TaskInfo, node *api.NodeInfo) (float64, error) {
        // DeviceScore
        if len(dp.schedulePolicy) > 0 {
            score, status := getDeviceScore(context.TODO(), task.Pod, node, dp.schedulePolicy)
            if !status.IsSuccess() {
                klog.Warningf("Node: %s, Calculate Device Score Failed because of Error: %v", node.Name, status.AsError())
                return 0, status.AsError()
            }
            
            // TODO: we should use a seperate plugin for devices, and seperate them from predicates and nodeOrder plugin.
            nodeScore := float64(score) * float64(dp.scheduleWeight)
            klog.V(5).Infof("Node: %s, task<%s/%s> Device Score weight %d, score: %f", node.Name, task.Namespace, task.Name, dp.scheduleWeight, nodeScore)
        }
        return 0, nil
    })
}
```
OnSessionOpen方法在调度会话开始时被调用，它注册了两个重要的事件处理器：

- 谓词函数(PredicateFn): 用于检查Pod是否可以调度到特定节点上。它检查节点是否初始化了所需的设备，以及设备是否满足Pod的需求。
- 节点排序函数(NodeOrderFn): 用于对节点进行评分，决定哪个节点最适合运行Pod。它调用getDeviceScore函数计算设备分数，并乘以调度权重得到最终分数。


###### OnSessionClose
```go
func (dp *deviceSharePlugin) OnSessionClose(ssn *framework.Session) {}
```
OnSessionClose方法在调度会话结束时被调用，目前它是一个空实现，但可以用于清理资源或记录日志。

### 插件的工作原理
现在，让我们从更高层次理解deviceshare插件是如何工作的：

1. 配置阶段: 通过YAML配置文件或命令行参数配置插件，指定是否启用GPU共享、VGPU等功能，以及调度策略和权重。
2. 会话初始化: 当Volcano启动一个新的调度会话时，OnSessionOpen方法被调用，注册谓词和节点排序函数。
3. 调度决策:
   - 谓词阶段: 对于每个Pod和节点对，检查设备资源是否满足需求。如果节点没有初始化所需的设备，或者设备不满足Pod的需求，则Pod不会被调度到该节点。
   - 节点排序阶段: 对满足谓词条件的节点进行评分，评分基于设备资源的可用性和调度策略。分数高的节点更有可能被选中运行Pod。
4. 会话结束: 当调度会话结束时，OnSessionClose方法被调用。


##### 插件的关键特性

- 灵活的GPU资源管理: 支持多种GPU资源管理方式，包括GPU共享、VGPU等。
- 可配置的调度策略: 通过配置参数可以灵活调整调度行为。


- 细粒度的资源评分: 基于设备资源的可用性和需求进行精确评分，提高资源利用率。


- 与Volcano框架的无缝集成: 作为Volcano调度器的一个插件，可以与其他调度策略协同工作。


#### 使用场景
deviceshare插件特别适用于以下场景：

- 机器学习训练: 需要大量GPU资源，且不同训练任务可能需要不同数量的GPU。
- 深度学习推理: 需要高效利用GPU资源，可能需要在多个推理任务之间共享GPU。
- 高性能计算: 需要精细控制GPU资源分配的计算密集型应用。


#### 总结
deviceshare插件是Volcano调度器中一个强大而灵活的组件，它解决了GPU等设备资源管理和调度的复杂问题。通过支持多种资源管理策略和可配置的调度行为，它可以帮助用户更高效地利用宝贵的GPU资源，提高集群的整体利用率和性能。
对于需要在Kubernetes环境中运行GPU密集型工作负载的用户来说，理解和配置deviceshare插件是优化资源使用和提高作业效率的关键步骤。随着AI和机器学习应用的普及，像deviceshare这样的设备资源调度插件将变得越来越重要。 