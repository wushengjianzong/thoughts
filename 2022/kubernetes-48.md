# Kubernetes九剑——总诀篇

## 口诀

> 控制模式，声明设计。  
> 任务调度，资源管理。  
> 三关六将，流批一体。  
> 进程线程，软硬双笔。  
> 配置存储，内外各异。  
> 设备流量，下四上七。

## 控制模式，声明设计

通俗说明如下：
+ 控制器模式：欢迎内驱潜力股，磨炼听话伸手党；
+ 声明式设计：顾客只递菜单，不教厨师做饭。

专业说明如下：
+ 前端：React单向数据流，浏览器内核（猜的）；
+ 后端：编译原理（形式语言与自动机），Linux内核。

> 万恶之源控制论。

## 任务调度，资源管理

+ 满足拓扑关系
    - `node affinity` / `pod affinity`
    - `taint` / `toleration`
    - `topology spread constraints`
+ 满足资源需求(QoS)
    - `Guaranteed`
    - `Burstable`
    - `BestEffort`

## 三关六将，流批一体

三关指授权（`Authuntion`）、鉴权（`Authorization`）和准入控制（`Admission`）
- 先验证书(`x509`)，后验`token`
- 先查租户(`Role`/`RoleBinding`)，后查集群(`ClusterRole`/`ClusterRoleBinding`)
- 先验操作(`ValidationWebhook`)，后验对象

流批一体是大数据里的一个名词，指系统同时支持批处理和流处理。

∵ `grpc`既能【问答调用】，也能【流式调用】；  
∴ `etcd`既做【数据存储】，也做【事件监听】；  
∴ `kube-apiserver`既是【控制入口】，也是【消息总线】。

## 进程线程，软硬双笔

+ 若将负载比进程，Pod即是那线程。
+ 若将Pod比进程，容器即是那线程。

> docker/containerd/cri-o是systemd托管的容器生命周期管理服务，使用CRI接口对接kubelet；  
> runc/kata/gvisor是利用软硬隔离技术实现的容器运行时，本体是二进制可执行文件。

+ 软隔离容器——共享内核
+ 硬隔离容器——独立内核

## 配置存储，内外各异

+ 内部配置
    - ConfigMap
    - Secret
    - ServiceAccount
+ 外部存储
    - StorageClass
    - PVC
    - PV


## 设备流量，下四上七

> 这里的分层指OSI七层模型。

CNI插件做两件事：装设备，连网线。确保容器网络二至四层功能完善。
- Service
- NetworkPolicy

Ingress/Gateway Controller引入L7服务器解决应用层的流量治理问题。
- Ingress/IngressClass
- Gateway/Route/Policy

## 附录

半成品的网络口诀如下（缺L7部分，还在酝酿）。

> 见与不见，网卡网线。  
> 架桥铺路，隧道相连。  
> 内核加速，用户网栈。  
> 卸力控流，还问硬件。

