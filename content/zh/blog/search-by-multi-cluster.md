---
title: 视频讲解｜Clusterpedia -- 多云环境下的资源复杂检索
date: 2022-03-01
---

Clusterpedia 的发起人 --「Daocloud 道客」的云原生研发工程师蔡威，为大家详细介绍 Clusterpedia 在资源检索上提供的功能，让大家可以直观的了解到使用 Clusterepdia 可以解决哪些问题。

<div align="center"><a href="https://www.bilibili.com/video/BV1qi4y117Vq"><img class="aligncenter wp-image-7957 size-full" src="http://blog.daocloud.io/wp-content/uploads/clusterpedia.png" alt="" width="800" height="500" data-tag="bdshare"></a></div>

## Clusterpedia 多集群资源检索神器
随着云原生技术的发展、承载业务量的增加以及集群规模的不断扩大，单个 Kubernetes 集群已经无法满足很多企业的需求，我们在逐渐的步入多云时代，多集群内部资源管理和检索变得越发复杂和困难。

由此，社区不断出现了很多优秀的的开源项目，例如用于集群生命周期管理的 cluster api，以及多云应用管理的 karmada， clusternet 等。而 Clusterpedia 便是建立在这些云管平台之上，为用户提供多集群资源的复杂检索。

在单集群中，我们通常使用 kubectl 来查看资源，或者直接访问 Kubernetes 的 OpenAPI，在代码中也可以借助 client-go 来对资源进行检索。

而在多集群环境下，Clusterpedia 通过兼容 Kubernetes OpenAPI ，用户可以依然使用单集群的方式，来对多集群资源进行复杂检索，无需从每个集群中拉取数据到本地进行过滤。

当然 Clusterpedia 的能力并不仅仅只是检索查看，未来还会支持对资源的简单控制，就像 wiki 同样支持编辑词条一样。Clusterpedia 具有许多特性和功能：

* 支持复杂的检索条件，过滤条件，排序，分页等等
* 支持查询资源时请求附带关系资源
* 统一主集群和多集群资源检索入口
* 兼容 kubernetes OpenAPI, 可以直接使用 kubectl 进行多集群检索, 而无需第三方插件或者工具
* 兼容收集不同版本的集群资源，不受主集群版本约束，
* 资源收集高性能，低内存
* 根据集群当前的健康状态，自动启停资源收集
* 插件化存储层，用户可以根据自己需求使用其他存储组件来自定义存储层
* 高可用

## 下期内容
除了支持多集群的复杂检索，Clusterpedia 还有很多其他优点，例如通过聚合式 API 来统一主集群和多集群资源的访问入口，在实时同步子集群资源时的低内存占用以及弱网优化，另外还有通过插件化存储层来解耦对存储组件的依赖。

下一期将为大家介绍具体设计和实现原理，详细解读 Clusterpedia 的优点，敬请期待。
