## Volcano调度插件：Conformance插件解析
### 前言
Conformance插件是Volcano调度器中的一个基础插件，它的主要功能是在预抢占(Preemption)和资源回收(Reclaim)过程中，帮助确定哪些任务可以被抢占或回收。简单来说，它定义了一套规则，告诉调度器在资源不足时，哪些任务可以被"踢出去"以腾出资源给更重要的任务。
### 代码解析

#### 会话打开时的处理
```go
func (pp *conformancePlugin) OnSessionOpen(ssn *framework.Session) {
    evictableFn := func(evictor *api.TaskInfo, evictees []*api.TaskInfo) ([]*api.TaskInfo, int) {
        var victims []*api.TaskInfo
        for _, evictee := range evictees {
            className := evictee.Pod.Spec.PriorityClassName
            // Skip critical pod.
            if className == scheduling.SystemClusterCritical ||  //  system-cluster-critical
                className == scheduling.SystemNodeCritical ||    // system-node-critical
                evictee.Namespace == v1.NamespaceSystem {  //"kube-system"
                    continue
                }
                victims = append(victims, evictee)
            }
        return victims, util.Permit
    }
    
    ssn.AddPreemptableFn(pp.Name(), evictableFn)
    ssn.AddReclaimableFn(pp.Name(), evictableFn)
}
```
OnSessionOpen方法在调度会话开始时被调用，这是插件的主要逻辑所在。
###### 抢占和资源回收函数
定义了一个名为evictableFn的函数，它决定了哪些任务可以被抢占或回收。这个函数接收两个参数：

- evictor: 尝试抢占或回收其他任务的任务，抢劫者
- evictees: 可能被抢占或回收的任务列表，受害者

###### 函数逻辑：

1. 初始化一个空的victims切片，用于存储最终确定可以被抢占或回收的任务。
2. 遍历所有可能被抢占或回收的任务(evictees)：
   - 获取任务的优先级类名(PriorityClassName)
   - 检查任务是否属于关键Pod(通过优先级类名或命名空间判断)：
     - 如果优先级类名是SystemClusterCritical或SystemNodeCritical
     - 或者任务在kube-system命名空间中
    - 如果是关键Pod，则跳过不处理(continue)
    - 否则，将任务添加到victims列表中
3. 返回victims列表和util.Permit状态(表示允许抢占或回收)

###### 注册函数到调度会话
```go
ssn.AddPreemptableFn(pp.Name(), evictableFn)
ssn.AddReclaimableFn(pp.Name(), evictableFn)
```
将evictableFn函数注册到调度会话中，分别用于预抢占和资源回收场景。这样当调度器需要进行预抢占或资源回收时，就会调用这个函数来决定哪些任务可以被处理。
#### 会话关闭时的处理
```go
func (pp *conformancePlugin) OnSessionClose(ssn *framework.Session) {}
```
OnSessionClose方法在调度会话结束时被调用，这里是一个空实现，因为这个插件不需要在会话结束时执行任何清理工作。
#### 插件功能总结
Conformance插件的核心功能可以总结为：

- 关键Pod保护：识别并保护系统关键Pod(如SystemClusterCritical、SystemNodeCritical优先级的Pod和kube-system命名空间中的Pod)不被抢占或回收。
- 普通Pod可抢占：对于非关键Pod，在资源不足时允许被抢占或回收，以腾出资源给更重要的任务。

这种设计确保了系统关键服务的稳定性，同时允许普通批处理任务在必要时被调度器调整，以实现资源的高效利用。
实际应用场景
Conformance插件在以下场景中特别有用：

- 混合工作负载环境：当集群中同时运行关键系统服务和批处理作业时，确保系统服务不会被批处理作业抢占。
- 资源紧张时的优雅降级：当集群资源不足时，优先保证关键服务的运行，而让非关键任务等待或被调整。
- 多租户环境：在多租户集群中，保护系统级租户的资源不被普通租户的任务抢占。

#### 总结
Conformance插件虽然代码量不大，但它在Volcano调度器中扮演着重要角色，为调度决策提供了基本的"哪些任务可以被抢占或回收"的规则。通过保护关键Pod同时允许普通Pod被调整，它帮助实现了集群资源的合理分配和系统稳定性。
