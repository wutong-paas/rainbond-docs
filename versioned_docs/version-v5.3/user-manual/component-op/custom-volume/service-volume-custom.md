---
title: 组件自定义存储设置
description: Rainbond组件自定义存储的管理文档
weight: 5011
hidden: false
---

### 组件为什么需要自定义存储

上篇文档 [组件存储设置](./service-volume/) 讲到了 Rainbond 默认支持的几种存储类型，不同的企业不同的业务对存储的要求是不一样的。比如大数据业务对高性能存储的需求等。Rainbond 实现了自定义存储类型的需求，用户可以自己安装自定义存储类型供 Rainbond 使用，可以很好的解决 Rainbond 默认存储类型不能满足的高性能存储需求。

现已支持的 [阿里云盘](../../../user-operations/storage/ali-disk/)、[ceph-rbd 块存储](../../../user-operations/storage/ceph-rbd/) 两种自定义存储，都是经过详细测试并成功的案例。原则上其他存储类型也能正常使用。

### 实现原理

了解 Rainbond 自定义存储类型实现原理之前，需要先了解 kubernetes 通过 storageClass 实现的动态提供存储功能。

Rainbond 在 kubernetes 实现动态提供存储功能的基础上，与 Rainbond 平台现有存储结合，支持用户自定义存储类型供 Rainbond 平台上的组件使用。

- 自定义存储类型的创建

自定义存储类型的创建需要考虑存储类型的不同，结合各种存储类型特性以及配置等，统一实现资源的定义。kubernetes 针对该需求抽象出了 storageClass 的资源定义，Rainbond 在 kubernetes 的基础上，抽象了 Rainbond 的存储类型的定义，用户可以通过 Rainbond 平台自定义存储类型，启用之后会生成对应的 storageClass，供控制台用户在组件管理的存储模块选择使用。

- 存储的创建

Rainbond 平台会检测 kubernetes 集群中存在的 storageClass 资源，将 storageClass 资源对象认为是用户自定义的存储类型，供 Rainbond 控制台组件选择使用。

在用户通过 Rainbond 控制台组件存储管理中，选择了对应的 storageClass 类型的自定义存储类型之后，需要通过组件的重启或者升级来使存储的变化生效。此时，Rainbond 集群组件 rbd-worker 会根据选择的存储类型找到对应的 storageClass，并根据用户的存储需求生成相应的 pvc，挂载到组件对应的 pod 实例上。

待驱动根据 pvc 创建好 pv 之后，pvc 和 pv 会形成一对一的绑定关系，此时 pvc 的状态为绑定状态，此时挂载了该 pvc 的实例才会继续生命周期的下一步。否则 pvc 的未绑定状态将会阻止 pod 实例的正常创建，难以实现 Rainbond 组件正常运行的效果。

- 存储的回收

组件被删除时会发生存储的回收，指定 storageClass 创建的存储会根据 storageClass 对应的存储回收策略选择在 pvc 被删除是是回收该存储还是将该存储恢复并保留或者什么都不做。

用户在添加存储类型的时候可以自己指定相应的存储回收策略。但 Rainbond 在组件删除时只会删除组件对应的 pvc，pvc 的删除意味着此时 pvc 绑定的存储会根据 storageClass 的回收策略进行选择。

### 如何新增存储

对接阿里云盘存储到 Rainbond 平台可参考 [Rainbond 平台对接阿里云盘](../../../user-operations/storage/ali-disk/)

对接 ceph 块存储到 Rainbond 平台可参考 [Rainbond 平台对接 ceph-rbd 块存储](../../../user-operations/storage/ceph-rbd/)

### 自定义存储缺陷

#### 限制使用

不同的存储类型有着不同的存储服务支持，比如块存储、文件存储和对象存储有着不同的读写模式，单读单写的读写模式只能挂在一次，不能共享读写。目前 Rainbond 平台仅对有状态组件开放使用自定义存储，并且自定义存储类型不可共享给其他组件共同使用。

#### 存储的备份和迁移

自定义存储的实现完全依赖 kubernetes，在挂载存储时只会挂载到实例对应的宿主机，并非集群共享，Rainbond 平台在进行组件备份时不能很好的获取自定义存储的数据并进行数据备份，所以 Rainbond 平台目前针对使用了自定义存储的组件在应用备份时会跳过数据的备份，仅备份组件使用的存储类型。

### 自定义存储的选择

Rainbond 平台在应用的分享和备份时都会记录组件使用的存储类型，当使用了自定义存储类型的应用被迁移到其他集群或者通过分享安装的方式安装到其他集群之后，会出现找不到存储类型的情况。针对这个异常情况，Rainbond 平台设计了一套最优存储类型选择的逻辑。在找不到对应的存储类型时，Rainbond 平台会根据组件类型以及对存储的期望选择最适合组件的存储类型，默认选择 Rainbond 共享存储。

> 最优存储类型的选择会在后期进行更细致的优化迭代，会综合考虑存储的读写特性、共享特性以及备份恢复策略等相关特性共同决定如何正确且最快的存储类型