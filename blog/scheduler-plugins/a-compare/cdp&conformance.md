### Volcano Conformance插件与Cooldown Protection插件的区别详解
作为Volcano调度器中的两个不同插件，Conformance和Cooldown Protection有着完全不同的设计目标和应用场景。下面我将从多个维度进行详细对比：
#### 1. 核心功能对比

| 特性   |      Conformance插件      | Cooldown Protection插件  |
|:--|:-----------------------:|:----------------------:|
| 主要功能 | 控制Pod抢占/回收规则，保护关键任务不被抢占 |  防止调度决策过于频繁，提供冷却时间窗口   |
| 触发时机 |      在资源不足需要抢占时触发       |       在每次调度决策后触发       |
| 决策类型 |     二元决策(可抢占/不可抢占)      |     时间控制(强制等待冷却期)      |

#### 2. 设计目的差异
- Conformance插件：
  - 解决"哪些任务可以被抢占"的问题
  - 保护系统关键Pod不被抢占
  - 实现批处理作业间的资源公平性
- Cooldown Protection插件：
    - 解决"调度决策频率过高"的问题
    - 防止因指标波动导致的频繁调度
    -为系统提供稳定性缓冲期
#### 3. 具体工作流程对比
Conformance插件工作流程：

1. 资源不足触发抢占流程
2. 插件检查每个候选Pod
   - 是否在kube-system命名空间
   - 是否有system-*优先级类
3. 标记符合条件的Pod为可抢占
4. 返回可抢占Pod列表给调度器

Cooldown Protection插件工作流程：
1. 调度器做出调度决策(如扩容)
2. 插件检查该资源最近是否有调度操作
3. 如果在冷却期内：
   - 阻止新调度决策
   - 记录被抑制的操作
4. 如果冷却期已过：
   - 允许新调度决策
   - 重置冷却计时器



#### 4. 典型应用场景
Conformance插件适用场景：

- 当集群GPU资源不足时，决定哪些计算任务可以被暂停
- 在混合关键性系统中保护系统服务Pod
- 多租户环境中防止低优先级租户抢占高优先级租户资源

Cooldown Protection插件适用场景：

- 自动伸缩组频繁扩容/缩容导致的不稳定
- 因指标采集延迟导致的误判(如CPU瞬时峰值)
- 需要时间验证调度决策效果的场景

#### 5. 配置参数差异
Conformance插件主要参数：

- 通过framework.Arguments传递
- 可能包含优先级类白名单等配置

Cooldown Protection插件典型参数：

- cooldown-period: 冷却时间长度(如5分钟)
- resource-types: 需要保护的资源类型列表
- excluded-actions: 不受冷却限制的操作类型

#### 6. 相互关系
这两个插件可以：

- 同时工作：Cooldown Protection控制调度频率，Conformance控制抢占规则
- 互补而非冲突：一个解决"何时做决策"，一个解决"做什么决策"
- 共同提高系统稳定性：前者防止过度反应，后者保护关键任务

#### 7. 实际影响示例
假设一个AI训练集群：

- 没有Cooldown Protection：
  - 可能因指标波动每分钟触发多次扩容
  - 导致资源碎片化
  - 增加调度器负载
- 没有Conformance：
  - 可能抢占系统监控Pod导致监控失效
  - 关键服务可能被普通训练任务挤占资源
- 两者结合：
  - Cooldown限制扩容频率(如每5分钟最多1次)
  - Conformance确保扩容时不会抢占监控Pod
  - 既保证响应速度又确保系统稳定


### 总结
这两个插件虽然都属于Volcano调度器，但：

- Conformance是微观层面的抢占控制，关注"哪些任务可以被牺牲"
- Cooldown Protection是宏观层面的调度节流，关注"调度决策的频率"

它们共同构成了Volcano调度器的"质量控制"体系，一个确保调度决策的正确性，一个确保调度决策的合理性。



https://dl-cdn.alpinelinux.org/v3.22/main/x86_64/openssl-3.5.3-r1.apk
https://dl-cdn.alpinelinux.org/v3.22/main/x86_64/libssl3-3.5.3-r1.apk
https://dl-cdn.alpinelinux.org/v3.22/main/x86_64/libcrypto3-3.5.3-r1.apk
https://dl-cdn.alpinelinux.org/v3.22/main/x86_64/ca-certificates-20250619-r0.apk
https://cdn.dl.k8s.io/release/v1.34.0/bin/linux/amd64/kubectl
FROM alpine:latest

COPY ./installer/dockerfile/webhook-manager/ca-certificates-20250619-r0.apk /ca-certificates.apk
COPY ./installer/dockerfile/webhook-manager/libcrypto3-3.5.3-r1.apk /libcrypto3.apk
COPY ./installer/dockerfile/webhook-manager/libssl3-3.5.3-r1.apk /libssl3.apk

RUN apk add --no-cache --allow-untrusted \
    /libcrypto3.apk \
    /libssl3.apk \
    /ca-certificates.apk

COPY ./installer/dockerfile/webhook-manager/kubectl /usr/local/bin/kubectl
COPY ./installer/dockerfile/webhook-manager/openssl /usr/bin/openssl


RUN chmod +x /usr/local/bin/kubectl && \
    chmod +x /usr/bin/openssl

COPY ./_output/bin/vc-webhook-manager /vc-webhook-manager
ADD ./installer/dockerfile/webhook-manager/gen-admission-secret.sh /gen-admission-secret.sh
ENTRYPOINT ["/vc-webhook-manager"]
FROM alpine:latest

COPY ./installer/dockerfile/webhook-manager/ca-certificates-20250619-r0.apk /ca-certificates.apk
COPY ./installer/dockerfile/webhook-manager/libcrypto3-3.5.3-r1.apk /libcrypto3.apk
COPY ./installer/dockerfile/webhook-manager/libssl3-3.5.3-r1.apk /libssl3.apk

RUN apk add --no-cache --allow-untrusted \
    /libcrypto3.apk \
    /libssl3.apk \
    /ca-certificates.apk

COPY ./installer/dockerfile/webhook-manager/kubectl /usr/local/bin/kubectl
COPY ./installer/dockerfile/webhook-manager/openssl /usr/bin/openssl


RUN chmod +x /usr/local/bin/kubectl && \
    chmod +x /usr/bin/openssl

COPY ./_output/bin/vc-webhook-manager /vc-webhook-manager
ADD ./installer/dockerfile/webhook-manager/gen-admission-secret.sh /gen-admission-secret.sh
ENTRYPOINT ["/vc-webhook-manager"]
