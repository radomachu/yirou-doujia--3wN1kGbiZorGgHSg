![image](https://img2024.cnblogs.com/blog/1439869/202510/1439869-20251005222757411-1004827735.jpg)

> 1. [1. pod基本概念](https://github.com)
> 2. [2. pod网络概念](https://github.com)
> 3. [3. pod的生命周期和状态](https://github.com)
> 4. [4. 探针](https://github.com):[ssrdog](https://pi64.org)
> 5. [5. 创建pod](https://github.com)
> 6. [6. 总结](https://github.com)

　　‍

## 1. pod基本概念

　　Kubernetes 中，**Pod 是最小的网络调度单位，** 每个pod可以放多个容器（例如可以放多个docke容器在同一个pod中运行），这些容器共享pod的网络、存储、以及容器规约。每个 Pod 被分配一个唯一的 IP 地址（Pod IP），这个 IP 在集群内是可达的。

　　在介绍其他概念之前，首先先介绍一下相关的网络概念，因为这是容易比较迷糊的地方。在k8s中，node有一个ip，pod也有一个ip，然后一个pod中又有很多pod，对应了很多ip，这容易让人造成误解。

* node的ip：node的是node对应的物理机（虚拟机，云主机）上的真实ip，比如说，你的一台物理机在公网上，那么这个node的ip便是一个公网的ip地址，我可以在任何地方拿着这个ip连接到这个node上。
* pod的ip：pod的ip实际上就是一个内网ip，由k8s的网络插件在pod启动的时候进行动态分配。

> Pod 运行在 Node 上，其网络流量进出需经过宿主机（Node），当外部访问 Pod 时，流量先到达 **Node**，再由相关组件转发到对应的pod
>
> 在一个正常工作的 Kubernetes 集群中，Node 节点之间的网络必须是互通的 —— 不仅 Node 之间要通，更重要的是，运行在不同 Node 上的 Pod 之间也必须能直接通信（三层路由可达）。
>
> NodeB (192.168.1.11)
>
> NodeA (192.168.1.10)
>
> ✅直接访问 Pod IPcurl Unsupported markdown: link
>
> ✅推荐：通过 Servicecurl Unsupported markdown: link
>
> 自动负载均衡
>
> ❌通常无效curl Unsupported markdown: link
>
> ✅显式设置curl Unsupported markdown: link
>
> PodAIP: 10.244.1.5Port: 8080
>
> PodxIP: 10.244.1.xPort: xx
>
> PodBIP: 10.244.2.8Listens: 8080
>
> Service: podB-serviceClusterIP: 10.96.xx.xxPort: 80 → targetPort: 8080

## 2. pod网络概念

> 在同一个 **Pod** 中的所有容器共享同一个网络命名空间（network namespace），因此它们拥有相同的 IP 地址（即 Pod IP），并共享同一个端口空间。容器之间可以通过 localhost 互相访问，但必须避免端口冲突。

　　当你在一个 Pod 中定义多个容器，Kubernetes 会将这些容器放在**同一个 Linux 网络命名空间中**。

　　这意味着：

* 所有容器看到的是同一个网络接口（如 eth0）
* 所有容器共享同一个 IP 地址 —— 即 Pod IP
* 所有容器共享同一个端口命名空间 —— 不能有两个容器监听同一个端口

　　Pod IP 是集群内可路由的。外部（其他 Pod、Service、Node）访问该 Pod 时，访问的是 Pod IP + 某个端口，而这个端口由 Pod 内某个容器监听。

![image](https://img2024.cnblogs.com/blog/1439869/202510/1439869-20251006170403185-1049510979.png)

　　‍

## 3. pod的生命周期和状态

　　pod的生命周期可以分为如下4种，pod的生命周期是单向的，不会回到之前的状态。

1. Pending
2. Running
3. Succeeded or Failed
4. Unknown

镜像拉取中 / 调度中 / PVC绑定 / Init容器运行

所有容器成功退出exit 0

至少一个容器失败退出exit ≠0 或 Crash

节点失联 / Kubelet无响应

readinessProbe 失败

livenessProbe 失败

startupProbe 失败

Init 容器执行中

全部成功

失败

节点恢复

节点永久丢失

创建 Podkubectl apply / Controller

Pending

Running

Succeeded

Failed

Unknown

Pod NotReady不加入 Endpoints不接收流量

重启容器→ CrashLoopBackOff

重启容器

Init:0/2, Init:Error 等

需手动/控制器重建

　　而pod又可以分为如下5种状态：

| 取值 | 描述 |
| --- | --- |
| ​`Pending` | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。 |
| ​`Running` | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。注意**Runing状态≠pod健康** |
| ​`Succeeded` | Pod 中的所有容器都已成功结束，并且不会再重启。 |
| ​`Failed` | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止，且未被设置为自动重启。 |
| ​`Unknown` | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。 |

　　那么如何知道pod究竟可不可用呢，那就得看pod的就绪状态，pod.status.conditions[? type=="Ready"]，如果type=Ready就代表pod可用，能接受流量，否则就代表不可用。

## 4. 探针

　　前文我们提到了pod分为不同的生命周期，以及对应的pod相关状态，那么问题来了，我怎么知道pod就行有没有运行成功呢，有没有ready呢，这就得靠我们的探针大哥帮忙。

　　探针的目的就是为了检测**容器**相关的状态，一共有如下四种探针检查机制：

1. exec：执行命令，检查容器是否ok
2. grpc：使用grpc检查容器是否正常
3. httpGet：http请求检查容器是否正常
4. tcpSocket：检查容器tcp端口是否打开，打开则正常

　　针对于探针的检查结果，无非就三种结果：成功、失败、未知（也就是探测失败，不采取任何行动）。当然，如果你不设置探针，那结果肯定就都是默认成功。

　　有了探针的检查结果，那么应该做什么呢？k8s有如下3种探针类型：

1. ​`livenessProbe`检查容器是否“活着”（进程是否卡死/假死），失败则根据重启策略重启容器。
2. ​`readinessProbe`检查容器是否“准备好服务”。如果失败，则容器容器就不能对外提供服务（其实就是自动将该 Pod 从对应的 **Endpoints** 对象中移除，从而不再将流量路由到这个 Pod，容器的状态为NotReady，对应的pod的状态Ready=False）。
3. ​`startupProbe`指示容器中的应用是否已经启动。如果提供了该类型探针，在成功前，会屏蔽其他类型的探针。当然，如果失败了，则会根据对应的策略进行重启。

> Pod 从创建到“真正可用”，必须等待所有容器的 readinessProbe 成功 —— 这是探针对 Pod “可用性”的核心控制点。
>
> 以下是千问老师总结的使用要点：
>
> ‍♂️ **Readiness** **=** **能不能干活 → 不行就“靠边站”，别重启！**
> **Liveness** **=** **还有没有气 → 不行就“抬走重来”，必须重启！**
> **Startup** **=** **刚出生要呵护 → 启动期特殊保护，长大再考核！**

　　‍

有

无

是

是

Pod 创建

容器启动

是否有 startupProbe?

执行 startupProbe 直到成功

开始 liveness/readinessProbe

周期性探测

liveness 失败?

重启容器 → Pod RestartCount++

readiness 失败?

Pod Ready=False → 从 Endpoints 移除

## 5. 创建pod

　　前面介绍了这么多，现在让我们使用命令来创建一个pod吧，大家可以在这个[Killercoda Interactive Environments](https://github.com)进行在线创建。

```
kubectl run nginxtest --image=nginx:latest --port=80
```

　　这样，我们便创建了一个nginx的pod：

![image](https://img2024.cnblogs.com/blog/1439869/202510/1439869-20251005222729403-1991344971.png)

## 6. 总结

　　Pod 是 Kubernetes 中最小的可部署、可调度的计算单元，一个 Pod 可以包含一个或多个紧密耦合的容器（共享网络、存储、生命周期）。pod的生命周期是不可逆的，而探针能够不断去对pod的状态进行检测，从而保证服务的可用性。我们有通过相关命令创建了一个pod，但是大家可以想一想，这个pod如果挂了，还能够重启吗？如果不能重启，那怎么去解决这个问题呢？让我们在下一章再进行介绍。

　　‍
