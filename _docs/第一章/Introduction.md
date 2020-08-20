---
title: 第一章
category: 简介
order: 2
---

第一章、简介
============

编程Kubernetes对不同的人可能意味着不同的事情。在本章中，我们将首先确定本书的范围和重点。另外，我们将分享有关我们所处的环境以及您需要带到桌面上的假设的一组假设，理想情况下，这将使您从本书中受益最多。我们将通过编程Kubernetes，了解什么是Kubernetes原生应用程序以及通过看一个具体示例来了解它们的确切含义，来定义我们的确切含义，我们将讨论控制器和操作员的基础知识，以及事件驱动的Kubernetes原则上如何控制平面功能，让我们开始吧。

Programming Kubernetes 是什么意思?
==================================

我们假设您可以访问正在运行的Kubernetes集群，如Amazon EKS、Microsoft
AKS、Google GKE或OpenShift产品之一。

###### 建议

您将花费大量时间在笔记本电脑或台式机环境上进行本地开发;也就是说，您要针对其开发的Kubernetes集群是本地的，而不是在云中或数据中心中。在本地进行开发时，您可以使用许多选项。根据您的操作系统和其他首选项，您可以选择以下一种（或什至更多种）以下解决方案在本地运行Kubernetes：kind，k3d或Docker
Desktop.1

我们还假设您是Go程序员-也就是说，您具有Go编程语言的经验或至少基本熟悉。现在是个好时机，如果其中任何假设都不适合您，请进行培训：对于Go，我们建议使用Alan
AA Donovan和Brian
W.Kernighan（Addison-Wesley）编写的Go编程语言以及Katherine编写的Go并发Cox-Buday（O\'Reilly）。对于Kubernetes，请查阅以下一本或多本书籍：

-   [*Kubernetes in Action*](http://bit.ly/2Tb8Ydo) by Marko Lukša
    (Manning)

-   [*Kubernetes: Up and Running*, 2nd
    Edition](https://oreil.ly/2SaANU4) by Kelsey Hightower et al.
    (O'Reilly)

-   [*Cloud Native DevOps with Kubernetes*](https://oreil.ly/2BaE1iq) by
    John Arundel and Justin Domingus (O'Reilly)

-   [*Managing Kubernetes*](https://oreil.ly/2wtHcAm) by Brendan Burns
    and Craig Tracey (O'Reilly)

-   [*Kubernetes Cookbook*](http://bit.ly/2FTgJzk) by Sébastien Goasguen
    and Michael Hausenblas (O'Reilly)

###### 注意

为什么我们专注于在Go中编程Kubernetes？好吧，这里的类比可能很有用：Unix是用C编程语言编写的，并且如果您要为Unix编写应用程序或工具，则默认为C。此外，为了扩展和自定义Unix，即使您要使用非C的语言-您至少需要能够阅读C。

现在，用Go编写Kubernetes和许多相关的云原生技术，从容器运行时到监控，例如Prometheus。我们相信大多数本机应用程序将基于Go，因此在本书中我们将重点介绍它。如果您喜欢其他语言，请关注kubernetes-client
GitHub组织。将来，它可能包含您喜欢的编程语言的客户端。

###### 通过在本书上下文中对Kubernetes进行编程，我们的意思是：您将要开发一个Kubernetes本地应用程序，该应用程序可以直接与API服务器进行交互，查询或更新资源状态。我们并不是说要运行诸如WordPress或Rocket Chat或您最喜欢的企业CRM系统之类的现成应用程序，这些应用程序通常被称为商业可用的现成（COTS）应用程序。此外，在第7章中，我们并没有真正过多地关注运营问题，而是主要关注开发和测试阶段。因此，简而言之，这本书是关于开发真正的云原生应用程序的。图1-1可能会帮助您更好地吸收它。![ifferent types of apps running on Kubernetes](media/image1.png){width="5.763888888888889in" height="3.56332895888014in"} * 图1-1.Kubernetes上运行的不同类型的应用程序*

如您所见，您可以使用不同的样式：

1.取得诸如Rocket
Chat之类的COTS，并在Kubernetes上运行它。该应用程序本身并不知道它可以在Kubernetes上运行，通常也不一定是这样。
Kubernetes控制着应用程序的生命周期，即找到要运行的节点，拉取镜像，启动容器，执行运行状况检查，安装卷等等。

2.选择一个定制的应用程序，从头开始编写，并考虑或不考虑使用Kubernetes作为运行时环境，然后在Kubernetes上运行它。适用与COTS相同的操作方式。

3.我们在本书中关注的案例是一个云原生或Kubernetes原生应用程序，该应用程序完全意识到它正在Kubernetes上运行，并在一定程度上利用了Kubernetes
API和资源。

您使用Kubernetes
API进行开发所付出的代价是有回报的：一方面，您获得了可移植性，因为您的应用程序现在可以在任何环境中运行（从本地部署到任何公共云提供商），另一方面您将从中受益从Kubernetes提供的干净的声明式机制中。

现在我们来看一个具体的例子。

Motivational 例子
=================

为了演示Kubernetes原生应用程序的功能，假设您要在特定时间实施-即计划在给定时间执行命令。

我们称其为CNat或cloud-native，其工作原理如下。假设您要执行命令echo"Kubernetes
native rocks！" 在2019年7月3日凌晨2点。使用cnat的操作如下：

> \$ cat cnat-rocks-example.yaml
>
> apiVersion: cnat.programming-kubernetes.info/v1alpha1
>
> kind: At
>
> metadata:
>
> name: cnrex
>
> spec:
>
> schedule: \"2019-07-03T02:00:00Z\"
>
> containers:
>
> \- name: shell
>
> image: centos:7
>
> command:
>
> \- \"bin/bash\"
>
> \- \"-c\"
>
> \- echo \"Kubernetes native rocks!\"
>
> \$ kubectl apply -f cnat-rocks-example.yaml
>
> cnat.programming-kubernetes.info/cnrex created

在背后涉及以下组件：

•一个名为cnat.programming-kubernetes.info/cnrex的自定义资源，代表计划。

•控制器在正确的时间执行计划的命令。

此外，用于CLI UX的Kubectl插件将非常有用，它可以通过诸如" 02:00 Jul
3"中的kubectl之类的命令进行简单处理，如"Kubernetes native
rocks!"。我们不会在本书中写这本书，但是您可以参考Kubernetes文档中的说明。

在整本书中，我们将使用该示例来讨论Kubernetes的各个方面，其内部工作原理以及如何扩展它。

对于第8章和第9章中更高级的示例，我们将模拟一个比萨饼餐厅，该餐厅在群集中具有比萨饼和顶部对象。有关详细信息，请参见\`\`示例：披萨餐厅\'\'。

扩展模式
========

Kubernetes是一个功能强大且本质上可扩展的系统。通常，有多种方法可以自定义或扩展Kubernetes：使用配置文件和标志用于控制平面组件（如kubelet或Kubernetes
API服务器），并通过许多定义的扩展点：

•所谓的云提供商，传统上在树中作为控制器管理器的一部分。从1.11版本开始，Kubernetes通过提供与云集成的自定义云控制器管理程序来实现树外开发。云提供商允许使用特定于云提供商的工具，例如负载平衡器或虚拟机（VM）。

•用于网络，设备（例如GPU），存储和容器运行时的二进制kubelet插件。

•二进制Kubectl插件。

•API服务器中的访问扩展，例如带webhooks的动态准入控制（请参阅第9章）。

•自定义资源（请参见第4章）和自定义控制器；请参阅以下部分。

•自定义API服务器（请参阅第8章）。

•计划程序扩展，例如使用Webhook来实现自己的计划决策。

•使用Webhook进行身份验证。

在本书的上下文中，我们专注于自定义资源，控制器，Webhooks和自定义API服务器以及Kubernetes扩展模式。如果您对其他扩展点（例如存储或网络插件）感兴趣，请查阅官方文档。

现在，您已经对Kubernetes扩展模式和本书的范围有了基本的了解，让我们继续学习Kubernetes控制平面的核心，看看如何扩展它。

Controllers 和Operators
=======================

在本部分中，您将了解Kubernetes中的controllers和operators及其工作方式。

根据Kubernetes词汇表，controller执行控制循环，通过API服务器监视集群的共享状态，并进行更改以尝试将当前状态移至所需状态。

在深入研究控制器的内部工作原理之前，让我们定义一下术语：

•
Controllers可以对核心资源（例如deployments或service）进行操作，这些资源通常是控制平面中的Kubernetes控制器管理器的一部分，或者可以监视和操作用户定义的自定义资源。

• Operators是对一些operational
knowledge进行编码的控制器，例如应用程序生命周期管理以及第4章中定义的自定义资源。

自然地，鉴于后者的概念是基于前者的，因此我们将首先研究Controllers，然后讨论Operators的更特殊情况。

控制循环
--------

一般控制循环如下所示：

1.阅读资源状态，最好是事件驱动的资源（如第3章所述，使用监视）。有关详细信息，请参见\`\`事件\'\'和\`\`边缘水平驱动触发器\'\'。

2.更改群集或群集外部环境中对象的状态。例如，启动Pod，创建网络端点或查询cloud
API。有关详细信息，请参见\`\`更改Cluster Objects或the External
World\'\'。

3.通过etcd中的API服务器更新步骤1中资源的状态。有关详细信息，请参见\`\`
Optimistic Concurrency\'\'。

4.重复循环；返回步骤1。

无论您的控制器多么复杂或简单，这三个步骤（读取资源状态˃ change the world
˃更新资源状态）都保持不变。让我们更深入地研究如何在Kubernetes控制器中实际实现这些步骤。control
loop如图1-2所示，图中显示了典型的运动部件，控制器的主循环位于中间。这个主循环在控制器进程内部持续运行。此过程通常在集群的Pod内部运行。

![ubernetes control loop](media/image2.png){width="5.763888888888889in"
height="2.447344706911636in"}

###### *图1-2. Kubernetes control loop*

从架构的角度来看，控制器通常使用以下数据结构（如第3章中详细讨论的那样）：

Informers

Informers
以可扩展且可持续的方式监视所需的资源状态。它们还实现了强制执行定期协调的重新同步机制（有关详细信息，请参见\`\`信息和缓存\'\'），通常用于确保群集状态和缓存在内存中的假定状态不会漂移（例如由于错误或网络问题））。

Work queues

本质上，工作队列是事件处理程序可以使用的组件，用于处理状态更改的排队并帮助实现重试。在client-go中，可以通过工作队列包使用此功能（请参阅\`\`工作队列\'\'）。
可以在更新环境或写入状态（循环中的步骤2和3）时发生错误的情况下重新分配资源，或者仅是由于某些原因我们不得不在一段时间后重新考虑资源。

有关Kubernetes作为声明性引擎和状态转换的更正式讨论，请阅读Andrew
Chen和Dominik Tornow撰写的\`\`Kubernetes的原理\'\'。

现在，从Kubernetes事件驱动的架构开始，让我们仔细看看控制循环。

事件
----

Kubernetes控制平面大量采用事件和松散耦合组件的原理。
其他分布式系统使用远程过程调用（RPC）触发行为。
Kubernetes没有，Kubernetes控制器监视对API服务器中Kubernetes对象的更改：添加，更新和删除。
当发生此类事件时，控制器将执行其业务逻辑。

例如，为了通过部署启动容器，许多控制器和其他控制平面组件一起工作：

1.部署控制器（在kube-controller-manager内部）通知（通过部署通知程序）用户创建了一个部署。它在其业务逻辑中创建一个副本集。

2.副本集控制器（同样在kube-controller-manager内部）（通过副本集通知程序）通知新副本集并随后运行其业务逻辑，该逻辑创建了pod对象。

3.调度程序（在kube-scheduler二进制文件内部）（也是一个控制器）（通过pod通知程序）使用空的spec.nodeName字段通知Pod。它的业务逻辑将pod放入其调度队列中。

4.同时，kubelet（另一个控制器）通知新Pod（通过其Pod通知程序）。但是新Pod的spec.nodeName字段为空，因此与kubelet的节点名不匹配。它会忽略吊舱并返回睡眠状态（直到下一个事件）。

5.调度程序通过更新Pod中的spec.nodeName字段并将其写入API服务器，将Pod从工作队列中移出并将其调度到具有足够可用资源的节点上。

6.由于pod更新事件，小组件再次唤醒。再次将spec.nodeName与自己的节点名进行比较。名称匹配，因此小程序将启动容器的容器，并将此信息写入容器状态并报告回API服务器，以报告容器已启动。

7.副本集控制器会注意到已更改的Pod，但与它无关。

8.最终pod终止。小程序将注意到这一点，从API服务器获取pod对象，并在pod状态中设置"终止"条件，然后将其写回到API服务器。

9.副本集控制器会注意到终止的Pod，并决定必须更换此Pod。它将在API服务器上删除终止的pod，然后创建一个新的pod。

10.依此类推。

如您所见，许多独立的控制循环纯粹通过API服务器上的对象更改进行通信，并且这些更改通过告示程序触发的事件。

这些事件通过监视（请参阅\`\`监视\'\'）从API服务器发送到控制器内部的通知程序-即监视事件的流式连接。
所有这些几乎都是用户看不见的。
甚至API服务器审核机制都无法使这些事件可见。仅对象更新可见。但是，当控制器对事件做出反应时，它们通常会使用日志输出。

##### 监视事件和事件对象

Watch events和Kubernetes中的top-level events对象是两件事：

> •Watch
> events通过API服务器和控制器之间的流HTTP连接发送，以驱动通知程序。
>
> •top-level events是诸如Pod，deployments或service之类的资源，具有特殊的属性，即生存时间为一个小时，然后自动从etcd中清除。

事件对象仅仅是用户可见的日志记录机制。
许多控制器创建这些事件，以便将其业务逻辑的各个方面传达给用户。例如kubelet报告pod的生命周期事件（即容器启动，重新启动和终止的时间）。

您可以使用Kubectl列出集群中发生的第二类事件。
通过发出以下命令，您可以看到kube-system命名空间中发生了什么：

> \$ **kubectl -n kube-system get events**
>
> LAST SEEN FIRST SEEN COUNT NAME KIND
>
> 3m 3m 1 kube-controller-manager-master.15932b6faba8e5ad Pod
>
> 3m 3m 1 kube-apiserver-master.15932b6fa3f3fbbc Pod
>
> 3m 3m 1 etcd-master.15932b6fa8a9a776 Pod
>
> ...
>
> 2m 3m 2 weave-net-7nvnf.15932b73e61f5bc6 Pod
>
> 2m 3m 2 weave-net-7nvnf.15932b73efeec0b3 Pod
>
> 2m 3m 2 weave-net-7nvnf.15932b73e8f7d318 Pod

如果您想了解更多有关事件的信息，请阅读Michael
Gasch的博客文章\`\`事件，Kubernetes的DNA\'\'，他在其中提供了更多背景知识和示例。

Edge- Versus Level-Driven 触发器
--------------------------------

让我们退后一步，更抽象地看一下我们如何构造在控制器中实现的业务逻辑，以及Kubernetes为什么选择使用事件（即状态更改）来驱动其逻辑。

有两种原则上的选项可检测状态变化（事件本身）：

Edge-driven triggers

在状态更改发生的时间点，将触发处理程序。例如从无Pod到Pod运行。

Level-driven triggers

定期检查状态，如果满足某些条件（例如pod运行），则会触发处理程序。

后一种是轮询的形式。
它不能随对象的数量很好地扩展，并且控制器注意到更改的延迟取决于轮询的间隔以及API服务器可以响应的速度。
如\`\`事件\'\'中所述，涉及到许多异步控制器，结果是系统需要很长时间才能实现用户的需求。

对于许多对象，前一种方法效率更高。
延迟主要取决于控制器处理事件中的工作线程数。
因此，Kubernetes是基于事件（即边缘驱动的触发器）的。

在Kubernetes控制平面中，许多组件会更改API服务器上的对象，每次更改都会导致一个事件（即边缘）。
我们称这些组件为事件源或事件生产者。
另一方面，在控制器的上下文中，我们对使用事件感兴趣，即何时以及如何对事件做出反应（通过通知者）。

在分布式系统中，有许多参与者并行运行，并且事件以任何顺序异步出现。
当我们的控制器逻辑有问题，状态机出现一些错误或外部服务故障时，就很容易丢失事件，因为我们没有完全处理它们。
因此，我们必须更深入地研究如何应对错误。

在图1-3中，您可以看到不同的工作策略：

1.仅edge-driven-only的示例，其中可能错过第二个状态更改。

2\. edge-triggered的示例，在处理事件时始终会获得最新状态（即级别）。
换句话说，逻辑是边沿触发的，但是是电平驱动的。

3.具有附加重新同步功能的edge-triggered，level-driven的示例。

![rigger options (edge vs.
level)](media/image3.png){width="5.763888888888889in"
height="4.217791994750656in"}

###### *图1-3. Trigger options (edge-driven versus level-driven)*

策略1不能很好地应对错过的事件，这可能是因为断开的网络使其丢失了事件，还是因为控制器本身存在错误或某些外部cloud
API已关闭。想象一下，副本集控制器仅在pod终止时才替换pod。缺少事件将意味着副本集将始终以较少的Pod运行，因为它永远不会协调整个状态。

当收到另一个事件时，策略2从这些问题中恢复，因为它基于集群中的最新状态实施其逻辑。对于副本集控制器，它将始终将指定的副本数与集群中正在运行的Pod进行比较。当丢失事件时，它将在下次收到Pod更新时替换所有丢失的Pod。

策略3添加了连续的重新同步（例如，每五分钟一次）。如果没有Pod事件进入，即使应用程序运行非常稳定并且不会导致很多Pod事件，它也至少每五分钟进行一次协调。

考虑到纯edge-driven触发器的挑战，Kubernetes控制器通常会实施第三种策略。

如果您想了解更多有关Kubernetes中触发的起源以及进行和解的水平触发的动机的信息，请阅读James
Bowes的文章，《 Kubernetes中的水平触发与和解》。

到此结束了有关检测外部变化并对之做出反应的不同抽象方法的讨论。
图1-2控制回路中的下一步是按照规范更改群集对象或更改外部环境，我们现在来看。

更改群集对象或外部世界
----------------------

在此阶段中，控制器更改其正在监视的对象的状态。例如，控制器管理器中的ReplicaSet控制器正在监控容器。在每个事件（边缘触发）上，它将观察其pod的当前状态，并将其与所需状态进行比较（水平驱动）。

由于更改资源状态的行为是特定于域或任务的，因此我们几乎无法提供指导。取而代之的是，我们将继续研究我们之前介绍的ReplicaSet控制器。副本集用于部署中，相应控制器的底线是：维护用户定义数量的相同Pod副本。也就是说，如果pod数量少于用户指定的数量（例如，由于pod失效或副本价值增加），则控制器将启动新的pod。但是，如果pod太多，它将选择一些pod作为终端。可通过replica_set.go软件包获得控制器的整个业务逻辑，以下Go代码摘录涉及状态更改（为清楚起见进行了编辑）：

> *// manageReplicas checks and updates replicas for the given
> ReplicaSet.*
>
> *// It does NOT modify \<filteredPods\>.*
>
> *// It will requeue the replica set in case of an error while
> creating/deleting pods.*
>
> **func** (rsc \*ReplicaSetController) manageReplicas(
>
> filteredPods \[\]\*v1.Pod, rs \*apps.ReplicaSet,
>
> ) **error** {
>
> diff := len(filteredPods) - int(\*(rs.Spec.Replicas))
>
> rsKey, err := controller.KeyFunc(rs)
>
> **if** err != **nil** {
>
> utilruntime.HandleError(
>
> fmt.Errorf(\"Couldn\'t get key for %v %\#v: %v\", rsc.Kind, rs, err),
>
> )
>
> **return** **nil**
>
> }
>
> **if** diff \< 0 {
>
> diff \*= -1
>
> **if** diff \> rsc.burstReplicas {
>
> diff = rsc.burstReplicas
>
> }
>
> rsc.expectations.ExpectCreations(rsKey, diff)
>
> klog.V(2).Infof(\"Too few replicas for %v %s/%s, need %d, creating
> %d\",
>
> rsc.Kind, rs.Namespace, rs.Name, \*(rs.Spec.Replicas), diff,
>
> )
>
> successfulCreations, err := slowStartBatch(
>
> diff,
>
> controller.SlowStartInitialBatchSize,
>
> **func**() **error** {
>
> ref := metav1.NewControllerRef(rs, rsc.GroupVersionKind)
>
> err := rsc.podControl.CreatePodsWithControllerRef(
>
> rs.Namespace, &rs.Spec.Template, rs, ref,
>
> )
>
> **if** err != **nil** && errors.IsTimeout(err) {
>
> **return** **nil**
>
> }
>
> **return** err
>
> },
>
> )
>
> **if** skippedPods := diff - successfulCreations; skippedPods \> 0 {
>
> klog.V(2).Infof(\"Slow-start failure. Skipping creation of %d pods,\"
> +
>
> \" decrementing expectations for %v %v/%v\",
>
> skippedPods, rsc.Kind, rs.Namespace, rs.Name,
>
> )
>
> **for** i := 0; i \< skippedPods; i++ {
>
> rsc.expectations.CreationObserved(rsKey)
>
> }
>
> }
>
> **return** err
>
> } **else** **if** diff \> 0 {
>
> **if** diff \> rsc.burstReplicas {
>
> diff = rsc.burstReplicas
>
> }
>
> klog.V(2).Infof(\"Too many replicas for %v %s/%s, need %d, deleting
> %d\",
>
> rsc.Kind, rs.Namespace, rs.Name, \*(rs.Spec.Replicas), diff,
>
> )
>
> podsToDelete := getPodsToDelete(filteredPods, diff)
>
> rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))
>
> errCh := make(**chan** **error**, diff)
>
> **var** wg sync.WaitGroup
>
> wg.Add(diff)
>
> **for** \_, pod := **range** podsToDelete {
>
> **go** **func**(targetPod \*v1.Pod) {
>
> **defer** wg.Done()
>
> **if** err := rsc.podControl.DeletePod(
>
> rs.Namespace,
>
> targetPod.Name,
>
> rs,
>
> ); err != **nil** {
>
> podKey := controller.PodKey(targetPod)
>
> klog.V(2).Infof(\"Failed to delete %v, decrementing \" +
>
> \"expectations for %v %s/%s\",
>
> podKey, rsc.Kind, rs.Namespace, rs.Name,
>
> )
>
> rsc.expectations.DeletionObserved(rsKey, podKey)
>
> errCh \<- err
>
> }
>
> }(pod)
>
> }
>
> wg.Wait()
>
> **select** {
>
> **case** err := \<-errCh:
>
> **if** err != **nil** {
>
> **return** err
>
> }
>
> **default**:
>
> }
>
> }
>
> **return** **nil**
>
> }
>
> 您可以看到控制器在diff行中计算了规格和当前状态之间的差异：=
> len（filteredPods）-int（\*（rs.Spec.Replicas）），然后根据以下情况实现了两种情况：

•diff \<0：副本太少； 必须创建更多pods。

•diff\> 0：副本太多； pods必须删除。

它还实现了一种策略，选择在getPodsToDelete中删除它们危害最小的Pod。

但是，更改资源状态并不一定意味着资源本身必须是Kubernetes集群的一部分。换句话说，控制器可以更改位于Kubernetes外部的资源（例如云存储服务）的状态。例如，\`\`AWS服务运营商\'\'使您可以管理AWS资源。除其他事项外，它还允许您管理S3存储桶-也就是说，S3控制器正在监视Kubernetes外部存在的资源（S3存储桶），并且状态更改反映了其生命周期中的具体阶段：创建了S3存储桶并在某些时候删除。

这应该使您确信，使用自定义控制器，您不仅可以管理核心资源（例如Pod），还可以管理自定义资源（例如我们的cnat示例），甚至可以计算或存储Kubernetes之外的资源。这使控制器具有非常灵活和强大的集成机制，提供了一种跨平台和环境使用资源的统一方式。

乐观并发
--------

在\`\` Control
Loop\'\'中，我们在步骤3中讨论了一个控制器-在根据规范更新群集对象或外部世界之后-将结果写入触发该控制器在步骤1中运行的资源状态。

这以及实际上任何其他写入（也在步骤2中）都可能出错。在分布式系统中，此控制器可能只是许多更新资源中的一个。
由于写入冲突，并发写入可能会失败。

为了更好地了解正在发生的事情，让我们退后一步来看看图1-4.2

![cheduling architectures in distributed
systems](media/image4.png){width="5.763888888888889in"
height="2.855021872265967in"}

###### *图1-4. 调度分布式系统中的体系结构*

来源定义了Omega的并行调度程序架构，如下所示：

我们的解决方案是围绕共享状态构建的新并行调度程序体系结构，它使用无锁的乐观并发控制来实现实现的可扩展性和性能可伸缩性。这种架构已在Google的下一代集群管理系统Omega中使用。

虽然Kubernetes继承了从Borg中学到的许多特征和经验教训，但这种特定的事务控制平面功能来自Omega：为了进行无锁的并发操作，Kubernetes
API服务器使用了乐观并发性。

简而言之，这意味着如果并且当API服务器检测到并发写入尝试时，它将拒绝两个写入操作中的后者。
然后由客户端（控制器，调度程序，kubectl等）处理冲突并可能重试写入操作。

以下内容展示了Kubernetes中的乐观并发的想法：

> var err error
>
> **for** retries := 0; retries \< 10; retries++ {
>
> foo, err = client.Get(\"foo\", metav1.GetOptions{})
>
> **if** err != nil {
>
> break
>
> }
>
> \<update-the-world-and-foo\>
>
> \_, err = client.Update(foo)
>
> **if** err != nil && errors.IsConflict(err) {
>
> **continue**
>
> } **else** **if** err != nil {
>
> break
>
> }
>
> }

该代码显示了一个重试循环，该循环在每次迭代中获取最新的对象foo，然后尝试更新世界和foo的状态以符合foo的规范。
在Update调用之前所做的更改是乐观的。

从client.Get调用返回的对象foo包含资源版本（嵌入式ObjectMeta结构的一部分-有关详细信息，请参见\`\`ObjectMeta\'\'），它将告诉etcd客户背后的写操作.Update调用集群中的另一个参与者
同时写了foo对象。 在这种情况下，我们的重试循环将收到资源版本冲突错误。
这意味着乐观并发逻辑失败。 换句话说，client.Update调用也很乐观。

###### 注意

资源版本实际上是etcd键/值版本。每个对象的资源版本是Kubernetes中的一个字符串，其中包含一个整数。该整数直接来自etcd，etcd拥有一个计数器，该计数器在每次修改键值（保存对象的序列化）时增加。

在整个API机器代码中，资源版本（因此或多或少地）像任意字符串一样进行处理，但是带有一些顺序。
存储整数的事实只是当前etcd存储后端的实现细节。

让我们看一个具体的例子。想象一下，您的客户不是集群中修改Pod的唯一参与者。
还有另一个参与者，即kubelet，由于容器不断崩溃而不断修改某些字段。现在，您的控制器将读取Pod对象的最新状态，如下所示：

> **kind**: Pod
>
> **metadata**:
>
> **name**: foo
>
> **resourceVersion**: 57
>
> **spec**:
>
> \...
>
> **status**:
>
> \...

现在假设控制器需要几秒钟的时间来更新世界。七秒钟后，它尝试更新其读取的Pod，例如，它设置了一个注释。同时，小组件注意到另一个容器重新启动并更新了容器的状态以反映这一点;也就是说，resourceVersion已增加到58。

您的控制器在更新请求中发送的对象的资源版本为57。API服务器尝试使用该值为Pod设置etcd键。etcd通知资源版本不匹配，并报告57与58冲突。更新失败。

此示例的底线是，对于您的控制器，您负责实施重试策略，并负责优化操作是否失败。您永远不知道其他人可能正在操纵状态，是其他自定义控制器还是核心控制器（例如部署控制器）。

其实质是：冲突错误在控制器中是完全正常的。始终期望他们并优雅地处理它们。

重要的是要指出，乐观并发非常适合基于级别的逻辑，因为通过使用基于级别的逻辑，您可以重新运行控制循环（请参阅"边缘对级别驱动触发器"）。该循环的另一轮运行将自动撤消先前失败的乐观尝试的乐观更改，并将尝试将世界更新到最新状态。

让我们继续研究自定义控制器（以及自定义资源）的特定情况：运算符。

Operators
---------

CoreOS于2016年将运算符作为Kubernetes中的一个概念引入.CoreOS首席技术官布兰登·菲利普（Brandon
Philips）在其开创性的博客文章\`\`介绍运算符：将操作知识纳入软件\'\'中将运算符定义为：

站点可靠性工程师（SRE）是通过编写软件来操作应用程序的人。他们是工程师，开发人员，他们知道如何专门为特定应用程序域开发软件。
最终的软件将其中已编程了应用程序的操作领域知识。

我们称这类新的软件操作员。操作员是特定于应用程序的控制器，它扩展了Kubernetes
API以代表Kubernetes用户创建，配置和管理复杂的有状态应用程序实例。它建立在基本的Kubernetes资源和控制器概念的基础上，但包括领域或应用程序特定知识，可自动执行常见任务。

在本书的上下文中，我们将使用Philips所描述的运算符，并且更正式地要求满足以下三个条件（另请参见图1-5）：

•您想自动化一些特定领域的操作知识。

•此操作知识的最佳实践是已知的，可以将其明确化-例如，对于Cassandra运算符，何时以及如何重新平衡节点，或者对于服务网格的运算符，如何创建一条路线。

•与操作员相关的工件为：

custom一组自定义资源定义（CRD），用于捕获特定于域的架构和自定义资源，这些实例和自定义资源遵循在实例级别代表感兴趣域的CRD。

♣一个自定义控制器，负责监督自定义资源，以及潜在的核心资源。例如，自定义控制器可能会旋转一个Pod。

![he concept of an
operator](media/image5.png){width="5.763888888888889in"
height="4.475474628171479in"}

###### *图1-5. The concept of an operator*

从2016年的概念工作和原型设计到Red
Hat（于2018年收购CoreOS并继续建立这个构想）于2019年初推出OperatorHub.io，运营商已经走了很长一段路要走。见图1-6的屏幕截图
该中心将于2019年中期投入运营，届时将有约17名运营商投入使用。

![peratorHub.io screen
shot](media/image6.png){width="5.763888888888889in"
height="3.585123578302712in"}

###### *图1-6. OperatorHub.io screenshot*

摘要
====

在第一章中，我们定义了本书的范围以及我们对您的期望。
我们在本书的上下文中解释了对Kubernetes进行编程的含义并定义了Kubernetes原生应用程序。
为了准备后面的示例，我们还对controllers和operators进行了高级介绍。

因此，既然您知道该书的期望以及如何从中受益，那么让我们跳入深层次。
在下一章中，我们将详细介绍Kubernetes
API，API服务器的内部工作原理以及如何使用命令行工具（例如Curl）与API进行交互。

1有关此主题的更多信息，请参见Megan
O'Keefe的"用于MacOS的Kubernetes开发人员工作流程"，中，2019年1月24日；
以及Alex Ellis在2018年12月14日发表的博客文章" ["Be KinD to
yourself"](http://bit.ly/2XkK9C1)。

2资料来源：\`["Omega: Flexible, Scalable Schedulers for Large Compute
Clusters"](http://bit.ly/2PjYZ59),，作者Malte
Schwarzkopf等人，谷歌AI，2013年。
