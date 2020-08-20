第二章. Kubernetes API 基础
===========================

在本章中，我们将带您逐步了解Kubernetes API的基础知识。
其中包括深入研究API服务器的内部工作原理，API本身，以及如何从命令行与API进行交互。
我们将向您介绍Kubernetes API概念，例如资源和种类，以及分组和版本控制。

API Server
==========

Kubernetes由一堆具有不同角色的节点（集群中的机器）组成，如图2-1所示：主节点上的控制平面由API
server，controller和scheduler组成。API服务器是中央管理实体，也是唯一与分布式存储组件etcd直接通信的组件。

API server具有以下核心职责：

•服务Kubernetes
API。主组件，master节点和Kubernetes本地应用程序在群集内部使用此API，以及在client（如Kubectl）在外部使用此API。

•代理Kubernetes仪表板等集群组件，或流式传输日志，服务端口或服务Kubectl执行会话。

提供API意味着：

> 读取状态：获取单个对象，列出它们并进行流式更改\
> 操纵状态：创建，更新和删除对象

状态通过etcd保留。

![ubernetes architecture
overview](media/image1.png){width="5.763888888888889in"
height="3.7460148731408576in"}

######  图2-1. *Kubernetes 架构概述*

Kubernetes的核心是API Server。但是API Server如何工作？ 我们首先将API
Server视为一个黑盒子，并仔细查看其HTTP界面，然后继续介绍API服务器的内部工作原理。

API Server 的 HTTP 接 口
------------------------

从客户端的角度来看，出于性能方面的考虑，API
Server公开了具有JSON或协议缓冲区（简称protobuf）有效负载的RESTful HTTP
API，主要用于集群内部通信。

API Server的HTTP接口使用以下HTTP
verbs（或HTTP方法）处理HTTP请求以查询和操纵Kubernetes资源：

•HTTP
GET动词用于检索具有特定资源（例如某个容器）或资源集合或资源列表（例如名称空间中的所有容器）的数据。

•HTTP POST动词用于创建资源，例如服务或部署。

•HTTP PUT谓词用于更新现有资源，例如，更改容器的容器映像。

•HTTPPATCH谓词用于部分更新现有资源。
阅读Kubernetes文档中的\`\`使用JSON合并补丁更新部署\'\'以在此处了解更多有关可用策略和含义的信息。

•HTTP DELETE动词用于以不可恢复的方式销毁资源。

例如，如果您查看Kubernetes 1.14
API参考，则可以看到实际的不同HTTP动词。例如要使用等效于CLI命令kubectl
-n THENAMESPACE get pods，可以发出GET请求。

/api/v1/namespaces/*THENAMESPACE*/pods (see [Figure 2-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-list-pods)).

![PI server HTTP interface in action: listing pods in a given
names](media/image2.png){width="5.763888888888889in"
height="2.88707239720035in"}

###### 图2-2. *API server HTTP接口的实际 作用: 列出给定命名空间的pod*

有关如何从Go程序中调用API服务器HTTP接口的介绍，请参见"The Client
Library"。

API 术语
--------

在开始从事API业务之前，我们首先定义在Kubernetes
APIServer上下文中使用的术语：

Kind

实体的类型。每个对象都有一个字段Kind（在JSON中为小写字母，在Golang中为大写Kind），该字段告诉诸如Kubectl之类的客户端它代表例如一个Pod。
共有三类：

•对象代表系统中的持久实体，例如Pod或Endpoints。对象具有名称，其中许多名称存活在名称空间中。

•列表是一种或多种实体的集合，列表具有一组有限的通用元数据。示例包括PodLists或NodeLists。当您执行**kubectl
get pods**，这就是您所得到的。

•特殊用途的种类用于对对象的特定操作以及非持久性实体，例如binding或
scale。为了发现，Kubernetes使用APIGroup和APIResource;
对于错误结果，它使用状态。

在Kubernetes程序中，一种类型直接与Golang类型相对应。
因此，作为Golang类型，种类是奇数，以大写字母开头。

API group

逻辑上相关的种类的集合。
例如，所有批处理对象（例如Job或ScheduledJob）都在批处理API group中。

Version

每个API组可以存在多个版本，大多数都可以。例如，一个组首先显示为v1alpha1，然后提升为v1beta1，最后毕业为v1。
可以在每个受支持的版本中检索以一个版本（例如v1beta1）创建的对象。
API服务器进行无损转换以返回请求版本的对象。从集群用户的角度来看，版本只是同一对象的不同表示。

###### TIP

没有诸如"群集中的v1中有一个对象，群集中的v1beta1中有另一个对象"之类的东西。相反，每个对象都可以按照群集用户的需要以v1表示形式或v1beta1表示形式返回。

Resource

通常是小写的复数单词（例如pod），用于标识一组HTTP端点（路径），以暴露系统中特定对象类型的CRUD（创建，读取，更新，删除）语义。
常见路径是：

•根目录，例如.../ pods，列出该类型的所有实例

•单个命名资源的路径，例如.../ pods / nginx

通常，这些端点中的每个端点都会返回并接收一种类型（第一种情况为PodList，第二种情况为Pod）。
但是在其他情况下（例如在发生错误的情况下），将返回状态类对象。

除了具有完整CRUD语义的主要资源外，资源还可以具有其他终结点来执行特定操作（例如，.../
pod / nginx / port-forward，.../ pod / nginx / exec或\... / pod / nginx
/ logs ）。 我们称这些子资源（请参阅\`\`子资源\'\'）。
这些通常实现自定义协议而不是REST，例如，通过WebSocket或命令性API进行某种流连接。

###### TIP

Resource和kind经常混杂在一起。 注意明显的区别：

•resource对应于HTTP路径。

•kind是这些端点返回并接收并持久化到etcd中的对象的类型。

resource始终是API
group和version的一部分，统称为GroupVersionResource（或GVR）。
GVR唯一定义HTTP路径。例如默认名称空间中的具体路径为*/apis/batch/v1/namespaces/default/jobs*.。
图2-3显示了一个用于命名空间资源Job的示例GVR。

![ubernetes API---Group, Version, Resource
(GVR)](media/image3.png){width="5.763888888888889in"
height="1.3736668853893264in"}

###### 图 2-3. *Kubernetes API---GroupVersionResource (GVR)*

与jobs GVR示例相反，node或namespaces等群集范围内的资源本身在路径中没有\$
NAMESPACE部分。例如，一个节点GVR示例可能如下所示：/ api / v1 / nodes。
请注意，名称空间显示在其他资源的HTTP路径中，但本身也是资源，可以在/ api
/ v1 / namespaces中访问。

与GVR相似，每种类型都生活在一个API组中，进行版本控制并通过GroupVersionKind（GVK）进行标识。

##### COHABITATION---KINDS 存活在多个API GROUPS

同一名称的kinds不仅可以同时存在于不同的版本中，还可以同时存在于不同的API组中。
例如，部署在扩展组中以alpha类型开始，最终在其所属的apps.k8s.io组中提升为稳定版本。我们称这种同居。尽管在Kubernetes中并不常见，但其中有一些：

> Ingress，NetworkPolicy和networking.k8s.io\
> Deployment，DaemonSet，ReplicaSet

Group和events.k8s.io中的 Event

GVK和GVR是相关的。
GVK在GVR标识的HTTP路径下提供服务。将GVK映射到GVR的过程称为REST映射。
我们将在" REST Mapping"中看到在Golang中实现REST映射的RESTMappers。

![n example Kubernetes API
space](media/image4.png){width="5.763888888888889in"
height="3.2306496062992127in"}

###### 图2-4. *示例 Kubernetes API space*

Kubernetes API 版本控制
-----------------------

出于扩展性原因，Kubernetes在不同的API路径（例如/ api / v1或/ apis /
extensions /
v1beta1）中支持多个API版本。不同的API版本意味着不同级别的稳定性和支持：

•默认情况下，通常禁用Alpha级别（例如v1alpha1）；
可以随时删除对功能的支持，恕不另行通知，并且应该仅在短期测试集群中使用。

•默认情况下启用Beta级（例如v2beta3），这意味着代码已经过充分测试；
但是，对象的语义可能会在随后的Beta或稳定版本中以不兼容的方式更改。

•对于许多后续版本，已发布的软件中将出现稳定（通常为GA）级别（例如v1）。

让我们看一下HTTP API空间的构造方式：在顶层，我们区分核心组（即/ api /
v1以下的所有内容）和路径中的命名组（apis / \$ NAME / \$ VERSION）。

###### 注意

出于历史原因，核心Group位于/ api / v1下，而不是人们所期望的位于/ apis /
core / v1下。 核心组在引入API组的概念之前就已经存在。

API服务器提供了第三种类型的HTTP路径-一种与资源不匹配的HTTP路径：群集范围内的实体，例如/
metrics，/ logs或/ healthz。
此外，API服务器还支持监视。也就是说，您可以向某些请求添加？watch =
true而不是按固定的时间间隔轮询资源，并且API服务器将更改为watch模式。

声明式状态管理
--------------

大多数API对象会在资源的期望状态规范和当前对象的状态之间进行区分。
规范或简称规范是对资源所需状态的完整描述，通常保存在稳定的存储中，通常为etcd。

###### 注意

为什么我们说"通常是etcd"？好吧，已经有Kubernetes发行版和产品（例如k3s或Microsoft的AKS）已经替换或正在努力用其他东西替换etcd。得益于Kubernetes控制平面的模块化体系结构，它可以很好地工作。

让我们再谈谈API服务器上下文中的规范（所需状态）与状态（观察状态）之间的关系。

规范描述了您所需的资源状态，您需要通过命令行工具（如Kubectl）或通过Go代码以编程方式提供资源。状态描述了资源的观察状态或实际状态，由控制平面管理，由核心组件（例如控制器管理器）或您自己的自定义控制器（请参阅\`\`控制器和操作员\'\'）进行管理。例如在部署中，您可以指定要始终运行该应用程序的20个副本。部署控制器是控制平面中控制器管理器的一部分，它会读取您提供的部署规范并创建一个副本集，然后由副本集来管理副本：它会创建各自数量的Pod，最终（通过kubelet）导致容器在工作节点上启动。如果任何副本失败，则部署控制器会将状态告知您。这就是我们所谓的声明式状态管理-即声明所需的状态并让Kubernetes负责其余的工作。

在下一部分中，随着我们从命令行开始探索API，我们将看到声明式状态管理的实际应用。

从命令行使用API
===============

在本节中，我们将使用Kubectl和Curl来演示Kubernetes API的用法。
如果您不熟悉这些CLI工具，那么现在是安装并试用它们的好时机。

首先，让我们看一下资源的期望状态和观察状态。
我们将在每个集群中使用可能会使用的控制平面组件，即CoreDNS插件（旧的Kubernetes版本使用kube-dns）在kube-system命名空间中（此输出经过大量编辑以突出显示重要部分）
：

> \$ kubectl -n kube-system get deploy/coredns -o=yaml
>
> apiVersion: apps/v1
>
> kind: Deployment
>
> metadata:
>
> name: coredns
>
> namespace: kube-system
>
> \...
>
> spec:
>
> template:
>
> spec:
>
> containers:
>
> \- name: coredns
>
> image: 602401143452.dkr.ecr.us-east-2.amazonaws.com/eks/coredns:v1.2.2
>
> \...
>
> status:
>
> replicas: 2
>
> conditions:
>
> \- type: Available
>
> status: \"True\"
>
> lastUpdateTime: \"2019-04-01T16:42:10Z\"

从该Kubectl命令中可以看到，在部署的规范部分中，您将定义特征，例如要使用的容器镜像以及要并行运行的副本数量，以及在状态部分中将了解数量
当前时间点的副本实际上正在运行。

为了执行与CLI相关的操作，在本章的其余部分中，我们将使用批处理操作作为运行示例。
首先，在终端中执行以下命令：

> \$ kubectl proxy \--port=8080
>
> Starting to serve on 127.0.0.1:8080

该命令将Kubernetes
API代理到我们的本地机器，并且还要处理身份验证和授权位。
它使我们能够直接通过HTTP发出请求并接收JSON负载作为回报。
为此，我们启动第二个终端会话来查询v1：

> \$ curl http://127.0.0.1:8080/apis/batch/v1
>
> {
>
> \"kind\": \"APIResourceList\",
>
> \"apiVersion\": \"v1\",
>
> \"groupVersion\": \"batch/v1\",
>
> \"resources\": \[
>
> {
>
> \"name\": \"jobs\",
>
> \"singularName\": \"\",
>
> \"namespaced\": true,
>
> \"kind\": \"Job\",
>
> \"verbs\": \[
>
> \"create\",
>
> \"delete\",
>
> \"deletecollection\",
>
> \"get\",
>
> \"list\",
>
> \"patch\",
>
> \"update\",
>
> \"watch\"
>
> \],
>
> \"categories\": \[
>
> \"all\"
>
> \]
>
> },
>
> {
>
> \"name\": \"jobs/status\",
>
> \"singularName\": \"\",
>
> \"namespaced\": true,
>
> \"kind\": \"Job\",
>
> \"verbs\": \[
>
> \"get\",
>
> \"patch\",
>
> \"update\"
>
> \]
>
> }
>
> \]
>
> }

###### TIP

您无需将Curl与Kubectl proxy命令一起使用即可直接通过HTTP
API访问Kubernetes API。 您可以改用kubectl get \--raw命令：例如，将curl
http://127.0.0.1:8080/apis/batch/v1替换为kubectl get \--raw / apis /
batch / v1。

将此与v1beta1版本进行比较，请注意，当您查看http://127.0.0.1:8080/apis/batch
v1beta1时，可以获取批处理API组的受支持版本的列表：

> \$ curl http://127.0.0.1:8080/apis/batch/v1beta1
>
> {
>
> \"kind\": \"APIResourceList\",
>
> \"apiVersion\": \"v1\",
>
> \"groupVersion\": \"batch/v1beta1\",
>
> \"resources\": \[
>
> {
>
> \"name\": \"cronjobs\",
>
> \"singularName\": \"\",
>
> \"namespaced\": true,
>
> \"kind\": \"CronJob\",
>
> \"verbs\": \[
>
> \"create\",
>
> \"delete\",
>
> \"deletecollection\",
>
> \"get\",
>
> \"list\",
>
> \"patch\",
>
> \"update\",
>
> \"watch\"
>
> \],
>
> \"shortNames\": \[
>
> \"cj\"
>
> \],
>
> \"categories\": \[
>
> \"all\"
>
> \]
>
> },
>
> {
>
> \"name\": \"cronjobs/status\",
>
> \"singularName\": \"\",
>
> \"namespaced\": true,
>
> \"kind\": \"CronJob\",
>
> \"verbs\": \[
>
> \"get\",
>
> \"patch\",
>
> \"update\"
>
> \]
>
> }
>
> \]
>
> }

如您所见，v1beta1版本还包含类型为CronJob的cronjobs资源。
在撰写本文时，cron作业尚未提升为v1。

如果您想了解集群中支持哪些API资源，包括它们的种类，是否命名空间以及它们的简称（主要用于命令行上的Kubectl），则可以使用以下命令：

> \$ kubectl api-resources
>
> NAME SHORTNAMES APIGROUP NAMESPACED KIND
>
> bindings true Binding
>
> componentstatuses cs false ComponentStatus
>
> configmaps cm true ConfigMap
>
> endpoints ep true Endpoints
>
> events ev true Event
>
> limitranges limits true LimitRange
>
> namespaces ns false Namespace
>
> nodes no false Node
>
> persistentvolumeclaims pvc true PersistentVolumeClaim
>
> persistentvolumes pv false PersistentVolume
>
> pods po true Pod
>
> podtemplates true PodTemplate
>
> replicationcontrollers rc true ReplicationController
>
> resourcequotas quota true ResourceQuota
>
> secrets true Secret
>
> serviceaccounts sa true ServiceAccount
>
> services svc true Service
>
> controllerrevisions apps true ControllerRevision
>
> daemonsets ds apps true DaemonSet
>
> deployments deploy apps true Deployment

以下是一个相关命令，对于确定集群中支持的不同资源版本可能非常有用：

> \$ kubectl api-versions
>
> admissionregistration.k8s.io/v1beta1
>
> apiextensions.k8s.io/v1beta1
>
> apiregistration.k8s.io/v1
>
> apiregistration.k8s.io/v1beta1
>
> appmesh.k8s.aws/v1alpha1
>
> appmesh.k8s.aws/v1beta1
>
> apps/v1
>
> apps/v1beta1
>
> apps/v1beta2
>
> authentication.k8s.io/v1
>
> authentication.k8s.io/v1beta1
>
> authorization.k8s.io/v1
>
> authorization.k8s.io/v1beta1
>
> autoscaling/v1
>
> autoscaling/v2beta1
>
> autoscaling/v2beta2
>
> batch/v1
>
> batch/v1beta1
>
> certificates.k8s.io/v1beta1
>
> coordination.k8s.io/v1beta1
>
> crd.k8s.amazonaws.com/v1alpha1
>
> events.k8s.io/v1beta1
>
> extensions/v1beta1
>
> networking.k8s.io/v1
>
> policy/v1beta1
>
> rbac.authorization.k8s.io/v1
>
> rbac.authorization.k8s.io/v1beta1
>
> scheduling.k8s.io/v1beta1
>
> storage.k8s.io/v1
>
> storage.k8s.io/v1beta1
>
> v1

API Server 如何处理请求
=======================

现在您已经了解了面向外部的HTTP接口，让我们专注于API服务器的内部工作原理。
图2-5概述了API服务器中的请求处理。

![ubernetes API server request processing
overview](media/image5.png){width="5.763888888888889in"
height="1.330719597550306in"}

###### 图2-5. *Kubernetes API server 请求处理概述*

那么，当HTTP请求命中Kubernetes
API时，实际上会发生什么？在较高级别上，发生以下交互：

1\.
HTTP请求由在DefaultBuildHandlerChain（）中注册的一系列过滤器处理。该链在k8s.io/apiserver/pkg/server/config.go中定义，稍后将详细讨论。它对所述请求应用了一系列过滤操作。要么过滤器通过并将相应的信息附加到上下文（准确地说是ctx.RequestInfo，而ctx是Go中的上下文（例如，经过身份验证的用户）），或者，如果请求未通过过滤器，它将返回一个适当的说明原因的HTTP响应代码（例如，如果用户身份验证失败，则为401响应）。

2.接下来，根据HTTP路径，k8s.io / apiserver / pkg / server /
handler.go中的多路复用器将HTTP请求路由到相应的处理程序。

3.为每个API组注册一个处理程序-有关详细信息，请参见k8s.io/apiserver/pkg/endpoints/groupversion.go和k8s.io/apiserver/pkg/endpoints/installer.go。它接受HTTP请求以及上下文（例如，用户和访问权限），并从etcd存储中检索并传递请求的对象。

现在让我们仔细研究一下server /
config.go中的DefaultBuildHandlerChain（）设置的过滤器链，以及它们各自发生的情况：

> **func** DefaultBuildHandlerChain(apiHandler http.Handler, c \*Config)
> http.Handler {
>
> h := WithAuthorization(apiHandler, c.Authorization.Authorizer,
> c.Serializer)
>
> h = WithMaxInFlightLimit(h, c.MaxRequestsInFlight,
>
> c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
>
> h = WithImpersonation(h, c.Authorization.Authorizer, c.Serializer)
>
> h = WithAudit(h, c.AuditBackend, c.AuditPolicyChecker,
> LongRunningFunc)
>
> \...
>
> h = WithAuthentication(h, c.Authentication.Authenticator, failed,
> \...)
>
> h = WithCORS(h, c.CorsAllowedOriginList, **nil**, **nil**, **nil**,
> \"true\")
>
> h = WithTimeoutForNonLongRunningRequests(h, LongRunningFunc,
> RequestTimeout)
>
> h = WithWaitGroup(h, c.LongRunningFunc, c.HandlerChainWaitGroup)
>
> h = WithRequestInfo(h, c.RequestInfoResolver)
>
> h = WithPanicRecovery(h)
>
> **return** h
>
> }

所有软件包都在k8s.io/apiserver/pkg中。 要更具体地审查：

WithPanicRecovery（）

负责恢复和日志恐慌。在server / filters / wrap.go中定义。

WithRequestInfo（）

在上下文中附加一个RequestInfo。定义于endpoints / filters /
requestinfo.go。

WithWaitGroup（）

将所有非长时间运行的请求添加到等待组；用于正常关机。定义在server /
filters / waitgroup.go中。

WithTimeoutForNonLongRunningRequests（）

与非长期运行的请求（例如监视和代理请求）相比，超时非长期运行的请求（例如大多数GET，PUT，POST和DELETE请求）。在server
/ filters / timeout.go中定义。

WithCORS（）

提供CORS实现。
CORS是跨域资源共享的缩写，是一种机制，它允许嵌入在HTML页面中的JavaScript向与该JavaScript起源的域不同的域发出XMLHttpRequests。在server
/ filters / cors.go中定义。

WithAuthentication（）

尝试将给定请求认证为人类或机器用户，并将用户信息存储在提供的上下文中。成功后，将从请求中删除Authorization
HTTP标头。如果身份验证失败，则返回HTTP 401状态代码。在endpoints /
filters / authentication.go中定义。

WithAudit（）

使用所有入局请求的审核日志记录信息来装饰处理程序。审核日志条目包含信息，例如请求的源IP，调用操作的用户以及请求的名称空间。在admission
/ audit.go中定义。

WithImpersonation（）

通过检查尝试更改用户的请求来处理用户模拟（类似于sudo）。在endpoints /
filters / impersonation.go中定义。

WithMaxInFlightLimit（）

限制进行中的请求数。在server / filters / maxinflight.go中定义。

WithAuthorization（）

通过调用授权模块检查权限，并将所有授权的请求传递到多路复用器，多路复用器将请求分派到正确的处理程序。如果用户没有足够的权限，它将返回HTTP
403状态代码。 Kubernetes现在使用基于角色的访问控制（RBAC）。在endpoints
/ filters / authorization.go中定义。

传递完此通用处理程序链之后（图2-5中的第一个方框），实际的请求处理开始（即执行请求处理程序的语义）：

•对/，/ *version*，/ api，/ healthz和其他非RESTful API的请求将直接处理。

•对RESTful资源的请求进入包含以下内容的请求管道：

入场

传入的对象经过一条进入链。该链有大约20个不同的准入插件.1每个插件都可以是变异阶段的一部分（请参见图2-5中的第三个方框），可以是验证阶段的一部分（请参见图中的第四个方框），或两者。

在更改阶段，可以更改传入的请求有效负载；例如，取决于准入配置，图像提取策略设置为\`\`
Always \'\'，\`\`IfNotPresent\'\'或\`\` Never\'\'。

第二个准入阶段纯粹是为了验证；例如，在该命名空间中创建对象之前，先验证Pod中的安全设置，或者验证命名空间的存在。

验证

将根据大型验证逻辑检查传入的对象，该逻辑对于系统中的每种对象类型都存在。例如，检查字符串格式以验证服务名称中仅使用了与DNS兼容的有效字符，或者Pod中的所有容器名称都是唯一的。

etcd支持的CRUD逻辑

这里实现了我们在\`\`API服务器的HTTP接口\'\'中看到的不同动词;例如，更新逻辑从etcd读取对象，检查是否没有其他用户以"乐观并发性"的方式修改对象，如果没有，则将请求对象写入etcd。

我们将在以下各章中更详细地研究所有这些步骤;例如：

定制资源

在第4章中的"验证自定义资源"中进行验证，在" Admission
Webhooks"中进行许可以及常规CRUD语义

Golang原生资源

在"Validation"中进行验证，在"Admission"中进行录入，以及在"Registry and
Strategy"中实施CRUD语义

摘要
====

在本章中，我们首先将Kubernetes
API服务器作为黑盒进行了讨论，并对其HTTP接口进行了介绍。然后，您学习了如何在命令行上与该黑匣子进行交互，最后我们打开了黑匣子并探索了它的内部工作原理。到现在为止，您应该知道API服务器如何在内部工作以及如何使用CLI工具Kubectl与之交互以进行资源探索和操纵。

现在是时候让我们在命令行上进行手动交互了，然后开始使用Go进行程序化API服务器访问：Meet？client-go，这是Kubernetes\`\`标准库\'\'的核心。

1在一个Kubernetes
1.14簇，这些是（以该顺序）：AlwaysAdmit，NamespaceAutoProvision，NamespaceLifecycle，NamespaceExists，SecurityContextDeny，LimitPodHardAntiAffinityTopology，PodPreset，LimitRanger，ServiceAccount，NodeRestriction，TaintNodesByCondition，AlwaysPullImages，ImagePolicyWebhook，PodSecurityPolicy，PodNodeSelector，优先级，DefaultTolerationSeconds，PodTolerationRestriction
，DenyEscalatingExec，DenyExecOnPrivileged，EventRateLimit，ExtendedResourceToleration，PersistentVolumeLabel，DefaultStorageClass，StorageObjectInUseProtection，OwnerReferencesPermissionEnforcement，PersistentVolumeClaimResize，MutatingAdmissionWebhook，ValidatingAdmissionWebhook，ValID
