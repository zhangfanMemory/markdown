# sentinel

sentinel 与 hystrix对比

<img title="" src="file:///Users/Zhuanz1/markdown_study/markdown/image/2025-01-15-16-38-02-image.png" alt="" width="459">

官网的对比：[Sentinel 与 Hystrix 的对比 · alibaba/Sentinel Wiki · GitHub](https://github.com/alibaba/Sentinel/wiki/Sentinel-%E4%B8%8E-Hystrix-%E7%9A%84%E5%AF%B9%E6%AF%94)

## 各个slot的作用

- `NodeSelectorSlot` 负责收集资源的路径，并将这些资源的调用路径，以树状结构存储起来，用于根据调用路径来限流降级；  
- `ClusterBuilderSlot` 则用于存储资源的统计信息以及调用者信息，例如该资源的 RT, QPS, thread count 等等，这些信息将用作为多维度限流，降级的依据；  
- `StatistcSlot` 则用于记录，统计不同纬度的 runtime 信息；  
- `FlowSlot` 则用于根据预设的限流规则，以及前面 slot 统计的状态，来进行限流；  
- `AuthorizationSlot` 则根据黑白名单，来做黑白名单控制；  
- `DegradeSlot` 则通过统计信息，以及预设的规则，来做熔断降级；  
- `SystemSlot` 则通过系统的状态，例如 load1 等，来控制总的入口流量；


