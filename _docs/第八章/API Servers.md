---
title: 第八章
category: 自定义 API Servers
order: 2
---

第8章. 自定义 API Servers
=========================

作为CustomResourceDefinitions的替代方法，您可以使用自定义API Server。
自定义API Server可以使用与Kubernetes API主服务器相同的方式为API
groups提供资源。与CRD相比，自定义API Server的功能几乎没有任何限制。

本章首先列出了CRD可能不是您的用例的正确解决方案的许多原因。它描述了一种聚合模式，该模式使使用自定义API
Server扩展Kubernetes
API表面成为可能。最后，您将学习使用Golang实际实现自定义API服务器。

自定义API Servers用例
=====================

可以使用自定义API
Server代替CRD。它可以执行CRD可以做的所有事情，并提供几乎无限的灵活性。当然，这是有代价的：开发和操作都非常复杂。

让我们看一下在撰写本文时（当Kubernetes 1.14是稳定版本时）CRD的一些限制：

•使用etcd作为它们的存储介质（或Kubernetes API Server使用的任何介质）。

•不支持protobuf，仅支持JSON。

•仅支持两种子资源：status和scale（请参阅"Subresources"）。

•不支持正常删除。1
Finalizers可以模拟此操作，但不允许自定义正常删除time。

•因为所有算法都是以通用方式实现的（例如validation），所以会大大增加Kubernetes
API server的CPU负载。

•仅为API端点实现标准的CRUD语义。

•不支持资源共存（即，不同API
Group中的resources或共享存储的不同名称的资源）。

相比之下，自定义API Server没有这些限制。

自定义API Server：

•可以使用任何存储介质。有自定义API Server，例如：

API Server将数据存储在内存中以实现最佳性能

API Server在OpenShift中镜像Docker注册表

API Server写入时间序列数据库

API Server镜像clouds APIs

API Server镜像其他API对象，比如OpenShift中镜像Kubernetes名称空间的项目

•像所有本地Kubernetes资源一样提供protobuf支持。为此，您必须使用go-to-protobuf创建一个.proto文件，然后使用protobuf编译器protoc生成序列化器，然后将其编译为二进制文件。

•可以提供任何自定义子资源；例如，Kubernetes API Server提供/ exec，/
logs，/
port-forward等功能，其中大多数使用非常定制的协议，例如WebSockets或HTTP /
2 streaming。

•可以像Kubernetes对Pod一样实现优美的删除。
kubectl等待删除，用户甚至可以提供自定义的正常终止期限。

•可以使用Golang以最有效的方式实现所有操作，例如验证，准入和转换，而无需通过webhooks来回传递，这会增加进一步的延迟。对于高性能用例或存在大量对象，这可能很重要。考虑一下具有数千个nodes和两个数量级以上的巨大集群中的pod对象。

•可以实现自定义语义，例如核心v1服务类型中Service
IP的原子保留。在创建服务的那一刻，分配了唯一的Service
IP并直接将其返回。在一定程度上，当然可以使用Admission
Webhooks来实现这样的特殊语义（请参阅\`\` Admission
Webhooks\'\'），尽管这些Webhooks永远无法可靠地知道所传递的对象是实际创建还是更新的：它们被乐观地调用，但是稍后请求pipeline中的步骤可能会取消请求。换句话说：Webhooks中的副作用非常棘手，因为如果请求失败，则没有撤消trigger。

•可以服务具有通用存储机制（即通用的etcd密钥路径前缀）但位于不同API
group中或name不同的resources。例如，Kubernetes将deployments和其他resources存储在API
group extensions / v1中，然后将其移动到特定的API Group中，例如apps /
v1。

换句话说，自定义API Server是CRD仍然受限的情况的解决方案。
在过渡方案中，在迁移到新语义时不要破坏资源兼容性很重要，自定义API
Server通常更加灵活。

实例: A Pizza Restaurant
========================

要了解如何实现自定义API
Server，在本节中，我们将看一个示例项目：一个实现a pizza restaurant
API的自定义API Server。 让我们来看看这些要求。

我们要在restaurant.programming-kubernetes.info API组中创建两种：

Topping（配料）

披萨馅料（例如萨拉米香肠，芝士或番茄）

Pizza（比萨）

餐厅提供的披萨类型

toppings是群集范围内的资源，并且仅包含一个浮点值，表示一个toppings单元的成本,实例很简单：

> **apiVersion**: restaurant.programming-kubernetes.info/v1alpha1
>
> **kind**: Topping
>
> **metadata**:

**name**: mozzarella(奶酪)

> **spec**:
>
> **cost**: 1.0

每个披萨可以有任意数量的toppings； 例如：

> **apiVersion**: restaurant.programming-kubernetes.info/v1alpha1
>
> **kind**: Pizza
>
> **metadata**:
>
> **name**: margherita(玛格丽塔)
>
> **spec**:
>
> **toppings**:
>
> \- mozzarella(奶酪)
>
> \- tomato(番茄)

toppings列表是有序的（就像YAML或JSON中的任何列表一样），但是该顺序对于类型的语义并不重要。在任何情况下，客户都会得到相同的披萨。我们允许列表中的重复项，以便带有额外奶酪的披萨。

所有这些都可以使用CRD轻松实现。现在，我们添加一些超出基本CRD功能的要求：

•我们只允许比萨规格中具有相应"toppings"对象的toppings。

•我们还想假设我们首先将此API作为v1alpha1版本引入，但最终了解到我们希望在同一API的v1beta1版本中使用其他表示形式。

换句话说，我们希望有两个版本并在它们之间无缝转换。

可以在本书的GitHub存储库中找到此API作为自定义API
Server的完整实现。在本章的其余部分，我们将遍历该项目的所有主要部分，并了解其工作方式。在此过程中，您将以不同的方式看到上一章中介绍的许多概念：即，Kubernetes
API Server背后的Golang实现。 CRD中强调的许多设计决策也将变得更加清晰。

因此，即使您不打算使用自定义API
Server，也强烈建议您通读本章。也许将来介绍的概念也将适用于CRD，在这种情况下，具有自定义API
Server的知识将对您有用。

体系结构: Aggregation

在讨论技术实现细节之前，我们希望在Kubernetes集群的上下文中对定制API
Server体系结构有一个更高层次的了解。

自定义API Server是为API Group提供服务的进程，通常使用通用API
Server库k8s.io/API
server构建。这些进程可以在集群内部或外部运行。在前一种情况下，它们在pod内运行，前面有一个服务。

Main Kubernetes API
Server，称为kube-apiserver,总是kubectl和其他API客户机的第一个联系点。自定义API
Server提供的API
Group由kube-apiserver,进程代理到自定义API服务器进程。换句话说，kube-apiserver,进程知道所有定制API
Server及其服务的API Group，以便能够将正确的请求代理给它们。

执行此代理的组件位于kube-apiserver,进程内，称为kube聚合器。将API请求代理到自定义API服务器的过程称为API
aggregation。

让我们进一步了解一下针对自定义API Server的请求的路径，但进入Kubernetes
API Server TCP socket（请参见图8-1）：

1.请求由Kubernetes API Server接收。

2.它们通过由身份验证、审核日志记录、impersonation、max-in-flight
throttling,授权等组成的处理程序链（图只是一个示意图，并不完整）。

3.由于Kubernetes API Server知道aggregation
APIs，所以它可以拦截对HTTP路径/ /apis/aggregated-API-group-name的请求。

4.Kubernetes API Server将请求转发到自定义API服务器。

![ubernetes main API server \`kube-apiserver\` with an integrated
\`kube-aggrega](media/image1.png){width="5.763888888888889in"
height="3.1768055555555557in"}

###### *图 8-1. Kubernetes main API server kube-apiserver with an integrated kube-aggregator*

kube-aggregator代理API Group版本（即/apis/group
name/version下的所有内容）的HTTP路径下的请求。它不必知道API组版本中的实际服务资源。

相反，kube-aggregator服务于所有聚合的自定义API服务器本身的发现端点/API和/API/group名称（它使用下一节中解释的定义顺序），并返回结果而不与aggregator的自定义API
Server交谈。相反，它使用来自APIService资源的信息。让我们详细看看这个过程。

API Services
------------

为了让Kubernetes API Server了解自定义API Server所服务的API
group，必须在apiregistration.k8s.io/v1
API组中创建一个APIService对象。这些对象仅列出API
group和versions，而不列出资源或任何其他详细信息：

> **apiVersion**: apiregistration.k8s.io/v1beta1
>
> **kind**: APIService
>
> **metadata**:
>
> **name**: *name*
>
> **spec**:
>
> **group**: *API-group-name*
>
> **version**: *API-group-version*
>
> **service**:
>
> **namespace**: *custom-API-server-service-namespace*
>
> **name**: *-API-server-service*
>
> **caBundle**: *base64-caBundle*
>
> **insecureSkipTLSVerify**: *bool*
>
> **groupPriorityMinimum**: 2000
>
> **versionPriority**: 20

名称是任意的，但是为了清楚起见，我们建议您使用一个标识API group
name和version的名称，例如*group-name-version*.。

该service可以是群集中的普通ClusterIP
Service，也可以是具有给定DNS名称的外部名称服务，用于群外自定义API
Server。在这两种情况下，端口都必须是443。不支持其他服务端口（在编写本文时）。服务目标端口映射允许任何选择的、最好是非限制的、更高的端口用于自定义API
Server pod，因此这不是一个主要限制。

证书颁发机构（CA）包用于Kubernetes
API服务器以信任所联系的服务。注意，API请求可以包含机密数据。为了避免中间人攻击，强烈建议您设置caBundle字段，而不要使用不安全的skiptlsvify选项。这对于任何生产集群都特别重要，包括证书循环机制。

最后，APIService对象中有两个优先级。它们具有一些复杂的语义，如APIService类型的Golang代码文档中所述：

> *// GroupPriorityMininum is the priority this group should have at
> least. Higher*
>
> *// priority means that the group is preferred by clients over lower
> priority ones.*
>
> *// Note that other versions of this group might specify even higher*
>
> *// GroupPriorityMinimum values such that the whole group gets a
> higher priority.*
>
> *//*
>
> *// The primary sort is based on GroupPriorityMinimum, ordered highest
> number to*
>
> *// lowest (20 before 10). The secondary sort is based on the
> alphabetical*
>
> *// comparison of the name of the object (v1.bar before v1.foo). We\'d
> recommend*
>
> *// something like: \*.k8s.io (except extensions) at 18000 and PaaSes*
>
> *// (OpenShift, Deis) are recommended to be in the 2000s*
>
> GroupPriorityMinimum **int32** \`json:\"groupPriorityMinimum\"\`
>
> *// VersionPriority controls the ordering of this API version inside
> of its*
>
> *// group. Must be greater than zero. The primary sort is based on*
>
> *// VersionPriority, ordered highest to lowest (20 before 10). Since
> it\'s inside*
>
> *// of a group, the number can be small, probably in the 10s. In case
> of equal*
>
> *// version priorities, the version string will be used to compute the
> order*
>
> *// inside a group. If the version string is \"kube-like\", it will
> sort above non*
>
> *// \"kube-like\" version strings, which are ordered
> lexicographically. \"Kube-like\"*
>
> *// versions start with a \"v\", then are followed by a number (the
> major version),*
>
> *// then optionally the string \"alpha\" or \"beta\" and another
> number (the minor*
>
> *// version). These are sorted first by GA \> beta \> alpha (where GA
> is a version*
>
> *// with no suffix such as beta or alpha), and then by comparing major
> version,*
>
> *// then minor version. An example sorted list of versions:*
>
> *// v10, v2, v1, v11beta2, v10beta3, v3beta1, v12alpha1, v11alpha2,
> foo1, foo10.*
>
> VersionPriority **int32** \`json:\"versionPriority\"\`

换句话说，GroupPriorityMinimum值确定了对组进行优先级排序的位置。
如果不同版本的多个APIService对象不同，则以最高价值规则为准。

第二优先级只是将版本相互排序，以定义动态客户端要使用的首选版本。

以下是本地Kubernetes API组的GroupPriorityMinimum值的列表：

> **var** apiVersionPriorities = **map**\[schema.GroupVersion\]priority{
>
> {Group: \"\", Version: \"v1\"}: {group: 18000, version: 1},
>
> {Group: \"extensions\", Version: \"v1beta1\"}: {group: 17900, version:
> 1},
>
> {Group: \"apps\", Version: \"v1beta1\"}: {group: 17800, version: 1},
>
> {Group: \"apps\", Version: \"v1beta2\"}: {group: 17800, version: 9},
>
> {Group: \"apps\", Version: \"v1\"}: {group: 17800, version: 15},
>
> {Group: \"events.k8s.io\", Version: \"v1beta1\"}: {group: 17750,
> version: 5},
>
> {Group: \"authentication.k8s.io\", Version: \"v1\"}: {group: 17700,
> version: 15},
>
> {Group: \"authentication.k8s.io\", Version: \"v1beta1\"}: {group:
> 17700, version: 9},
>
> {Group: \"authorization.k8s.io\", Version: \"v1\"}: {group: 17600,
> version: 15},
>
> {Group: \"authorization.k8s.io\", Version: \"v1beta1\"}: {group:
> 17600, version: 9},
>
> {Group: \"autoscaling\", Version: \"v1\"}: {group: 17500, version:
> 15},
>
> {Group: \"autoscaling\", Version: \"v2beta1\"}: {group: 17500,
> version: 9},
>
> {Group: \"autoscaling\", Version: \"v2beta2\"}: {group: 17500,
> version: 1},
>
> {Group: \"batch\", Version: \"v1\"}: {group: 17400, version: 15},
>
> {Group: \"batch\", Version: \"v1beta1\"}: {group: 17400, version: 9},
>
> {Group: \"batch\", Version: \"v2alpha1\"}: {group: 17400, version: 9},
>
> {Group: \"certificates.k8s.io\", Version: \"v1beta1\"}: {group: 17300,
> version: 9},
>
> {Group: \"networking.k8s.io\", Version: \"v1\"}: {group: 17200,
> version: 15},
>
> {Group: \"networking.k8s.io\", Version: \"v1beta1\"}: {group: 17200,
> version: 9},
>
> {Group: \"policy\", Version: \"v1beta1\"}: {group: 17100, version: 9},
>
> {Group: \"rbac.authorization.k8s.io\", Version: \"v1\"}: {group:
> 17000, version: 15},
>
> {Group: \"rbac.authorization.k8s.io\", Version: \"v1beta1\"}: {group:
> 17000, version: 12},
>
> {Group: \"rbac.authorization.k8s.io\", Version: \"v1alpha1\"}: {group:
> 17000, version: 9},
>
> {Group: \"settings.k8s.io\", Version: \"v1alpha1\"}: {group: 16900,
> version: 9},
>
> {Group: \"storage.k8s.io\", Version: \"v1\"}: {group: 16800, version:
> 15},
>
> {Group: \"storage.k8s.io\", Version: \"v1beta1\"}: {group: 16800,
> version: 9},
>
> {Group: \"storage.k8s.io\", Version: \"v1alpha1\"}: {group: 16800,
> version: 1},
>
> {Group: \"apiextensions.k8s.io\", Version: \"v1beta1\"}: {group:
> 16700, version: 9},
>
> {Group: \"admissionregistration.k8s.io\", Version: \"v1\"}: {group:
> 16700, version: 15},
>
> {Group: \"admissionregistration.k8s.io\", Version: \"v1beta1\"}:
> {group: 16700, version: 12},
>
> {Group: \"scheduling.k8s.io\", Version: \"v1\"}: {group: 16600,
> version: 15},
>
> {Group: \"scheduling.k8s.io\", Version: \"v1beta1\"}: {group: 16600,
> version: 12},
>
> {Group: \"scheduling.k8s.io\", Version: \"v1alpha1\"}: {group: 16600,
> version: 9},
>
> {Group: \"coordination.k8s.io\", Version: \"v1\"}: {group: 16500,
> version: 15},
>
> {Group: \"coordination.k8s.io\", Version: \"v1beta1\"}: {group: 16500,
> version: 9},
>
> {Group: \"auditregistration.k8s.io\", Version: \"v1alpha1\"}: {group:
> 16400, version: 1},
>
> {Group: \"node.k8s.io\", Version: \"v1alpha1\"}: {group: 16300,
> version: 1},
>
> {Group: \"node.k8s.io\", Version: \"v1beta1\"}: {group: 16300,
> version: 9},
>
> }

因此，对于类似PaaS的API使用2000意味着将它们放在此列表的末尾.4\\

API组的顺序在Kubectl中的REST映射过程中起作用（请参阅\`\`REST
Mapping\'\'）。这意味着它会对用户体验产生实际影响。如果存在冲突的资源名称或简称，则以GroupPriorityMinimum值最高的资源为准。

此外，在使用自定义API Server替换API group
version的特殊情况下，此优先级排序可能会有用。例如，您可以通过将自定义API
Server放置在GroupPriorityMinimum值低于上表中的位置的位置来将本地Kubernetes
API组替换为修改后的Kubernetes API组（无论出于何种原因）。

再次注意，Kubernetes API Server不需要知道发现端点/ apis和/ apis /
group-name或代理的资源列表。资源列表仅通过第三个发现端点/ apis /
group-name /
version返回。但是正如我们在上一节中看到的那样，此终结点由聚合的自定义API服务器提供服务，而不是由kube-aggregator服务。

自定义API Server的内部结构
--------------------------

自定义API Server类似于组成Kubernetes
API服务器的大多数部分，尽管当然具有不同的API
group实现，并且没有嵌入式kube-aggregator或嵌入式apiextension-apiserver（用于CRD）。这导致了与图8-1中几乎相同的架构图（如图8-2所示）：

![n aggregated custom API server based on
k8s.io/apiserver](media/image2.png){width="5.763888888888889in"
height="3.660120297462817in"}

###### *图8-2. An aggregated custom API server based on k8s.io/apiserver*

我们观察到许多事情。aggregated的API Server：

•具有与Kubernetes API Server相同的基本内部结构。

•拥有自己的处理程序链，包括身份验证，审核，模拟，最大运行限制和授权（我们将在本章中解释为什么这样做是必要的；例如，参见"Delegated
Authorization"）。

•拥有自己的资源处理程序管道，包括解码，转换，接纳，REST映射和编码。

•呼叫webhooks。

•可能写到etcd（尽管它可以使用其他存储后端）。 etcd集群不必与Kubernetes
API Server使用的集群相同。

•对于自定义API组具有自己的方案和注册表实现。注册表的实现可能会有所不同，并且会在任何程度上进行自定义。

•再次进行身份验证。它通常会执行客户端证书身份验证和基于令牌的身份验证，并使用TokenAccessReview请求回调到Kubernetes
API Server。我们将在稍后详细讨论身份验证和信任体系结构。

•自己进行审核。这意味着Kubernetes API
Server审核某些字段，但仅在元级别上。对象级审核是在聚合的自定义API
Server中完成的。

•使用对Kubernetes
API服务器的SubjectAccessReview请求进行自己的身份验证。我们将在短期内更详细地讨论授权。

委托身份验证和信任
------------------

aggregated的自定义API Server（基于k8s.io/apiserver）在与Kubernetes API
Server相同的身份验证库上构建。它可以使用客户端证书或令牌来认证用户。

由于在架构上将aggregated的自定义API Server放置在Kubernetes API
Server的后面（即Kubernetes API
Server接收请求并将其代理到聚合的自定义API服务器），因此KubernetesAPI
Server已经对请求进行了身份验证。 Kubernetes
API服务器将身份验证的结果（即用户名和组成员身份）存储在HTTP请求标头中，通常是X-Remote-User和X-Remote-Group（可以使用\--requestheader-username-标头和\--requestheader-group-headers标志）。

aggregated的自定义API
Server必须知道何时信任这些标头。否则，任何其他调用者都可以声称已完成身份验证并可以设置这些标头。这由特殊请求标头客户端CA处理。它存储在配置映射kube-system
/
extension-apiserver-authentication（文件名requestheader-client-ca-file）中。这是一个例子：

> **apiVersion**: v1
>
> **kind**: ConfigMap
>
> **metadata**:
>
> **name**: extension-apiserver-authentication
>
> **namespace**: kube-system
>
> **data**:
>
> **client-ca-file**: \|
>
> \-\-\-\--BEGIN CERTIFICATE\-\-\-\--
>
> \...
>
> \-\-\-\--END CERTIFICATE\-\-\-\--
>
> **requestheader-allowed-names**: \'\[\"aggregator\"\]\'
>
> **requestheader-client-ca-file**: \|
>
> \-\-\-\--BEGIN CERTIFICATE\-\-\-\--
>
> \...
>
> \-\-\-\--END CERTIFICATE\-\-\-\--
>
> **requestheader-extra-headers-prefix**: \'\[\"X-Remote-Extra-\"\]\'
>
> **requestheader-group-headers**: \'\[\"X-Remote-Group\"\]\'
>
> **requestheader-username-headers**: \'\[\"X-Remote-User\"\]\'

利用此信息，具有默认设置的aggregated自定义API
Server将对以下内容进行身份验证：

•使用与给定的client-ca文件匹配的客户端证书的客户端

•由Kubernetes API
Server预先认证的客户端，其客户端使用给定的requestheader-client-ca-file转发请求，其用户名和组成员身份存储在给定的HTTP标头X-Remote-Group和X-Remote-User中

最后但并非最不重要的一点是，存在一种称为TokenAccessReview的机制，该机制将承载令牌（通过HTTP标头授权：承载令牌接收）转发回Kubernetes
API
Server，以验证它们是否有效。令牌访问检查机制默认情况下处于禁用状态，但可以选择启用；请参阅\`\`
Options and Config Pattern and Startup Plumbing\'\'。

我们将在以下各节中看到如何实际设置委托身份验证。虽然我们在这里详细介绍了此机制，但是在聚合的自定义API
Server内部，这通常是由k8s.io/apiserver库自动完成的。但是，知道幕后发生的事情无疑是有价值的，尤其是在涉及安全性的地方。

委托授权
--------

身份验证完成后，必须授权每个请求。 授权基于用户名和组列表。
Kubernetes中的默认授权机制是基于角色的访问控制（RBAC）。

RBAC将身份映射到角色，将角色映射到授权规则，这些规则最终接受或拒绝请求。
我们不会在此处介绍有关RBAC授权对象的所有详细信息，例如角色和集群角色，或角色绑定和集群角色绑定（有关更多信息，请参见Getting
the Permissions Right"）。 从架构的角度来看，知道聚合的自定义API
Server通过SubjectAccessReviews使用委托授权来授权请求就足够了。
它不会评估RBAC规则本身，而是将评估委托给Kubernetes API Server。

##### 为什么已整合的API Server总是必须 执行另一个 授权步骤

Kubernetes API Server收到并转发到聚合的自定义API
Server的每个请求都通过身份验证和授权（请参见图8-1）。
这意味着聚合的自定义API Server可以跳过此类请求的委托授权部分。

但是，这种预授权不能保证并且可能随时消失（为了将来更好的安全性和可扩展性，有计划将kube-aggregator与kube-apiserver分开）。
此外，直接转到汇总的自定义API
Server的请求（例如，通过客户端证书或令牌访问审核进行验证的请求）不会通过Kubernetes
API Server，因此不会得到预授权。

换句话说，跳过授权委托会带来安全漏洞，因此强烈建议您不要这样做。

现在，让我们详细了解委托授权。

根据请求（如果它在其授权缓存中未找到答案），主题访问审核从aggregated的自定义API
Server发送到Kubernetes API Server。 这是一个此类检查对象的示例：

> **apiVersion**: authorization.k8s.io/v1
>
> **kind**: SubjectAccessReview
>
> **spec**:
>
> **resourceAttributes**:
>
> **group**: apps
>
> **resource**: deployments
>
> **verb**: create
>
> **namespace**: default
>
> **version**: v1
>
> **name**: example
>
> **user**: michael
>
> **groups**:
>
> \- system:authenticated
>
> \- admins
>
> \- authors

Kubernetes API Server从聚合的自定义API
Server接收此消息，评估集群中的RBAC规则，并做出决定，返回设置了状态字段的SubjectAccessReview对象;
例如：

> **apiVersion**: authorization.k8s.io/v1
>
> **kind**: SubjectAccessReview
>
> **status**:
>
> **allowed**: true
>
> **denied**: false
>
> **reason**: \"rule foo allowed this request\"

请注意，此处允许和拒绝都有可能是错误的。这意味着Kubernetes API
Server无法做出决定，在这种情况下，聚合的自定义API
Server内部的另一个授权者可以做出决定（API
Server实现一个由一个授权者一个人查询的授权链，委托授权是其中一个授权者在那个链中）。这可以用于对非标准授权逻辑进行建模-也就是说，在某些情况下，如果没有RBAC规则，而是使用外部授权系统。

请注意，出于性能原因，委托授权机制在每个聚合的自定义API
Server中维护一个本地缓存。默认情况下，它将缓存1,024个授权条目，其中包括：

•允许的授权请求有效期为5分钟

•拒绝授权请求的有效期为30秒

可以通过\--authorization-webhook-cache-authorized-ttl和\--authorization-webhook-cache-unauthorized-ttl自定义这些值。

在以下各节中，我们将介绍如何在代码中设置委派授权。同样，与身份验证一样，在聚合的自定义API
Server中，委托授权通常由k8s.io/apiserver库自动完成。

编写自定义API Servers
=====================

在前面的部分中，我们介绍了aggregated API
Server的体系结构。在本节中，我们要研究Golang中聚合自定义API
Server的实现。

Main API Server是通过k8s.io/apiserver库实现的。自定义API
Server将使用完全相同的代码。主要区别在于我们的自定义API
Server将在集群中运行。这意味着它可以假设集群中有一个kube-apiserver可用，并用它来进行委托授权和检索其他kube-native资源。

我们还假设有一个etcd集群可用，并且可供聚集的自定义API
Server使用。这个etcd是专用还是与Kubernetes API
Server共享并不重要。我们的自定义APIServer将使用不同的etcd密钥空间来避免冲突。

本章中的代码示例引用了GitHub上的示例代码，因此请在此处查找完整的源代码。我们将仅在此处显示最有趣的摘录，但您始终可以转到完整的示例项目，对其进行试验，并且（对于学习非常重要）在真实的集群中运行它。

这个pizza-apiserver项目实现了"示例：披萨餐厅"中显示的示例API

Options and Config Pattern and Startup Plumbing
-----------------------------------------------

k8s.io/apiserver库使用选项和配置模式创建正在运行的API Server。

我们将从绑定到标志的几个选项结构开始。
从k8s.io/apiserver中获取它们并添加我们的自定义选项。
可以针对特殊用例在代码中调整k8s.io/apiserver中的选项结构，并且可以将提供的标志应用于标志集以便用户访问。

在示例中，我们非常简单地将所有内容都基于RecommendedOptions。
这些建议的选项可根据需要设置简单API的"normal"聚合自定义API
Server所需的所有内容，例如：

> **import** (
>
> \...
>
> informers \"github.com/programming-kubernetes/pizza-apiserver/pkg/
>
> generated/informers/externalversions\"
>
> )
>
> **const** defaultEtcdPathPrefix =
> \"/registry/restaurant.programming-kubernetes.info\"
>
> **type** CustomServerOptions **struct** {
>
> RecommendedOptions \*genericoptions.RecommendedOptions
>
> SharedInformerFactory informers.SharedInformerFactory
>
> }
>
> **func** NewCustomServerOptions(out, errOut io.Writer)
> \*CustomServerOptions {
>
> o := &CustomServerOptions{
>
> RecommendedOptions: genericoptions.NewRecommendedOptions(
>
> defaultEtcdPathPrefix,
>
> apiserver.Codecs.LegacyCodec(v1alpha1.SchemeGroupVersion),
>
> genericoptions.NewProcessInfo(\"pizza-apiserver\",
> \"pizza-apiserver\"),
>
> ),
>
> }
>
> **return** o
>
> }

CustomServerOptions嵌入RecommendedOptions并在顶部添加一个字段。
NewCustomServerOptions是使用默认值填充CustomServerOptions结构的构造函数。

让我们研究一些更有趣的细节：

•defaultEtcdPathPrefix是我们所有密钥的etcd前缀。作为密钥空间，我们使用/registry/pizza-apiserver.programming-kubernetes.info，与Kubernetes密钥明显不同。

•SharedInformerFactory是我们自己的CR的全过程共享informers工厂，以避免不必要的informers使用相同的资源（见图3-5）。
请注意，它是从我们项目中生成的informers代码而不是从client-go导入的。

•NewRecommendedOptions使用默认值为聚合的自定义API服务器设置所有内容。

让我们快速浏览一下NewRecommendedOptions：

> **return** &RecommendedOptions{
>
> Etcd: NewEtcdOptions(storagebackend.NewDefaultConfig(prefix, codec)),
>
> SecureServing: sso.WithLoopback(),
>
> Authentication: NewDelegatingAuthenticationOptions(),
>
> Authorization: NewDelegatingAuthorizationOptions(),
>
> Audit: NewAuditOptions(),
>
> Features: NewFeatureOptions(),
>
> CoreAPI: NewCoreAPIOptions(),
>
> ExtraAdmissionInitializers:
>
> **func**(c \*server.RecommendedConfig)
> (\[\]admission.PluginInitializer, **error**) {
>
> **return** **nil**, **nil**
>
> },
>
> Admission: NewAdmissionOptions(),
>
> ProcessInfo: processInfo,
>
> Webhook: NewWebhookOptions(),
>
> }

所有这些都可以在必要时进行调整。例如，如果需要自定义默认服务端口，则可以设置RecommendedOptions.SecureServing.SecureServingOptions.BindPort。

让我们简要介绍一下现有的选项结构：

•Etcd配置读取和写入etcd的存储堆栈。

•SecureServing可配置HTTPS周围的所有内容（即端口，证书等）

•身份验证按照"Delegated Authentication and
Trust"中的说明设置委托的身份验证。

•授权按照"Delegated Authorization"中所述设置委托授权。

•审核设置审核输出堆栈。默认情况下禁用此功能，但可以将其设置为输出审核日志文件或将审核事件发送到外部后端。

•功能配置Alpha和Beta功能的功能门。

•CoreAPI拥有一个kubeconfig文件的路径来访问主API
Server。默认情况下，使用集群内配置。

•准入是一堆针对每个传入API请求执行的变异和验证准入插件的堆栈。可以使用定制的代码内接纳插件来扩展它，或者可以为定制API服务器调整默认的接纳链。

•ExtraAdmissionInitializers允许我们添加更多的初始值设定项以用于接纳。初始化程序通过自定义API
Server实现通知程序或客户端的管道。有关自定义Admission的更多信息，请参见\`\`
Admission\'\'。

•ProcessInfo保存用于创建事件对象的信息（即进程名称和名称空间）。对于两个值，我们都将其设置为Pizza-apiserver。

•Webhook配置Webhook的操作方式（例如，身份验证和许可Webhook的常规设置）。对于在群集内部运行的自定义API服务器，使用默认设置进行设置。对于集群外部的API服务器，这里是配置其如何到达Webhook的地方。

选项与标志结合；也就是说，它们通常与标志处于相同的抽象级别。根据经验，选项不包含"运行中"的数据结构。它们在启动期间使用，然后转换为配置或服务器对象，然后再运行。

可以通过Validate（）错误方法来验证选项。此方法还将检查用户提供的标志值是否具有逻辑意义。

可以完成选项以设置默认值，这些默认值不应显示在标志的帮助文本中，但对于获得完整的选项集而言是必需的。

通过Config（）（\*
apiserver.Config，error）方法将选项转换为Server配置（"
config"）。首先从建议的默认配置开始，然后对其应用选项：

> **func** (o \*CustomServerOptions) Config() (\*apiserver.Config,
> **error**) {
>
> err :=
> o.RecommendedOptions.SecureServing.MaybeDefaultWithSelfSignedCerts(
>
> \"localhost\", **nil**, \[\]net.IP{net.ParseIP(\"127.0.0.1\")},
>
> )
>
> **if** err != **nil** {
>
> **return** **nil**, fmt.Errorf(\"error creating self-signed cert:
> %v\", err)
>
> }
>
> \[\... omitted o.RecommendedOptions.ExtraAdmissionInitializers \...\]
>
> serverConfig :=
> genericapiserver.NewRecommendedConfig(apiserver.Codecs)
>
> err = o.RecommendedOptions.ApplyTo(serverConfig, apiserver.Scheme);
>
> **if** err != **nil** {
>
> **return** **nil**, err
>
> }
>
> config := &apiserver.Config{
>
> GenericConfig: serverConfig,
>
> ExtraConfig: apiserver.ExtraConfig{},
>
> }
>
> **return** config, **nil**
>
> }

此处创建的配置包含可运行的数据结构；
换句话说，与选项相对应，配置是运行时对象，与选项相对应。
o.RecommendedOptions.SecureServing.MaybeDefaultWithSelfSignedCerts这一行将创建自签名证书，以防用户未通过预生成证书的标志的情况。

如前所述，genericapiserver.NewRecommendedConfig返回默认的推荐配置，RecommendedOptions.ApplyTo根据标志（和其他自定义选项）对其进行更改。

Pizza-apiserver项目本身的配置结构只是我们示例自定义API
Server的RecommendedConfig的包装：

> **type** ExtraConfig **struct** {
>
> *// Place your custom config here.*
>
> }
>
> **type** Config **struct** {
>
> GenericConfig \*genericapiserver.RecommendedConfig
>
> ExtraConfig ExtraConfig
>
> }
>
> *// CustomServer contains state for a Kubernetes custom api server.*
>
> **type** CustomServer **struct** {
>
> GenericAPIServer \*genericapiserver.GenericAPIServer
>
> }
>
> **type** completedConfig **struct** {
>
> GenericConfig genericapiserver.CompletedConfig
>
> ExtraConfig \*ExtraConfig
>
> }
>
> **type** CompletedConfig **struct** {
>
> *// Embed a private pointer that cannot be instantiated outside of*
>
> *// this package.*
>
> \*completedConfig
>
> }

如果需要为正在运行的自定义API
Server提供更多状态，请使用ExtraConfig放置它。

与选项结构类似，该配置具有设置默认值的Complete（）CompletedConfig方法。
由于必须为基础配置实际调用Complete（），因此通常通过引入未导出的completedConfig数据类型来通过类型系统强制执行该操作。
这里的想法是只有对Complete（）的调用才能将Config转换为completeConfig。
如果未完成此调用，编译器将抱怨(complain)：

> **func** (cfg \*Config) Complete() completedConfig {
>
> c := completedConfig{
>
> cfg.GenericConfig.Complete(),
>
> &cfg.ExtraConfig,
>
> }
>
> c.GenericConfig.Version = &version.Info{
>
> Major: \"1\",
>
> Minor: \"0\",
>
> }
>
> **return** completedConfig{&c}
>
> }
>
> 最后，可以通过New（）构造函数将完成的配置转换为CustomServer运行时结构
>
> *// New returns a new instance of CustomServer from the given
> config.、*
>
> **func** (c completedConfig) New() (\*CustomServer, **error**) {
>
> genericServer, err := c.GenericConfig.New(
>
> \"pizza-apiserver\",
>
> genericapiserver.NewEmptyDelegate(),
>
> )
>
> **if** err != **nil** {
>
> **return** **nil**, err
>
> }
>
> s := &CustomServer{
>
> GenericAPIServer: genericServer,
>
> }
>
> \[ \... omitted API installation \...\]
>
> **return** s, **nil**
>
> }

请注意，我们在此故意省略了API安装部分。 我们将在\`\`API
Installation\'\'（即您如何在启动过程中将注册表连接到自定义API
Server）中返回到此内容。 注册表实现API组的API和存储语义。 我们将在\`\`
Registry and Strategy\'\'中的餐厅API组中看到这一点。

最终可以使用Run（stopCh \<-chan struct
{}）错误方法启动CustomServer对象。
在我们的示例中，这是通过选项的运行方法来调用的。
也就是说，CustomServerOptions.Run：

•创建配置

•完成配置

•创建CustomServer

•调用CustomServer.Run

这是代码：

> **func** (o CustomServerOptions) Run(stopCh \<-**chan** **struct**{})
> **error** {
>
> config, err := o.Config()
>
> **if** err != **nil** {
>
> **return** err
>
> }
>
> server, err := config.Complete().New()
>
> **if** err != **nil** {
>
> **return** err
>
> }
>
> server.GenericAPIServer.AddPostStartHook(\"start-pizza-apiserver-informers\",
>
> **func**(context genericapiserver.PostStartHookContext) **error** {
>
> config.GenericConfig.SharedInformerFactory.Start(context.StopCh)
>
> o.SharedInformerFactory.Start(context.StopCh)
>
> **return** **nil**
>
> },
>
> )
>
> **return** server.GenericAPIServer.PrepareRun().Run(stopCh)
>
> }

PrepareRun（）调用连接了OpenAPI规范，并且可能会执行其他API安装后操作。
调用后，运行方法将启动实际服务器。 它会一直阻塞直到stopCh关闭。

这个示例还连接了一个名为start-pizza-apiserver-informers的启动后hook。
顾名思义，HTTPS服务器启动并侦听后，将调用启动后hook。在这里，它将启动共享的
informer工厂。

请注意，即使是有关自定义API服务器本身提供的资源的本地进程内通知程序，也都通过HTTPS向本地主机接口讲话。
因此，在服务器启动且HTTPS端口正在侦听之后启动它们很有意义。

还要注意，/ healthz端点仅在所有启动hook成功完成后才返回成功。

在所有小管道都准备就绪的情况下，pizza-apiserver项目将所有内容包装到一个cobra命令中：

> *// NewCommandStartCustomServer provides a CLI handler for \'start
> master\' command*
>
> *// with a default CustomServerOptions.*
>
> **func** NewCommandStartCustomServer(
>
> defaults \*CustomServerOptions,
>
> stopCh \<-**chan** **struct**{},
>
> ) \*cobra.Command {
>
> o := \*defaults
>
> cmd := &cobra.Command{
>
> Short: \"Launch a custom API server\",
>
> Long: \"Launch a custom API server\",
>
> RunE: **func**(c \*cobra.Command, args \[\]**string**) **error** {
>
> **if** err := o.Complete(); err != **nil** {
>
> **return** err
>
> }
>
> **if** err := o.Validate(); err != **nil** {
>
> **return** err
>
> }
>
> **if** err := o.Run(stopCh); err != **nil** {
>
> **return** err
>
> }
>
> **return** **nil**
>
> },
>
> }
>
> flags := cmd.Flags()
>
> o.RecommendedOptions.AddFlags(flags)
>
> **return** cmd
>
> }

使用 NewCommandStartCustomServer the main() 方法非常简单:

> **func** main() {
>
> logs.InitLogs()
>
> **defer** logs.FlushLogs()
>
> stopCh := genericapiserver.SetupSignalHandler()
>
> options := server.NewCustomServerOptions(os.Stdout, os.Stderr)
>
> cmd := server.NewCommandStartCustomServer(options, stopCh)
>
> cmd.Flags().AddGoFlagSet(flag.CommandLine)
>
> **if** err := cmd.Execute(); err != **nil** {
>
> klog.Fatal(err)
>
> }
>
> }

特别注意对SetupSignalHandler的调用：它连接Unix信号处理。
在SIGINT（在终端中按Ctrl-C时触发）和SIGKILL上，停止通道已关闭。
停止通道将传递到正在运行的自定义API
Server，并且在停止通道关闭时它将关闭。
因此，当接收到信号之一时，主回路将启动关闭。
从某种意义上说，这种关闭很合适，因为在终止之前，正在运行的请求已完成（默认情况下最长60秒）。
它还确保将所有请求发送到审核后端，并且不删除审核数据。
之后，cmd.Execute（）将返回并且该过程将终止。

The First Start
---------------

现在，我们已经准备就绪，可以首次启动自定义API Server。 假设您在〜/ .kube
/ config中配置了一个集群，则可以将其用于委派的身份验证和授权：

> \$ cd \$GOPATH/src/github.com/programming-kubernetes/pizza-apiserver
>
> \$ etcd &
>
> \$ go run . \--etcd-servers localhost:2379 **\\**
>
> \--authentication-kubeconfig \~/.kube/config **\\**
>
> \--authorization-kubeconfig \~/.kube/config **\\**
>
> \--kubeconfig \~/.kube/config
>
> I0331 11:33:25.702320 64244 plugins.go:158\]
>
> Loaded 3 mutating admission controller(s) successfully in the
> following order:
>
> NamespaceLifecycle,MutatingAdmissionWebhook,PizzaToppings.
>
> I0331 11:33:25.702344 64244 plugins.go:161\]
>
> Loaded 1 validating admission controller(s) successfully in the
> following order:
>
> ValidatingAdmissionWebhook.
>
> I0331 11:33:25.714148 64244 secure_serving.go:116\] Serving securely
> on \[::\]:443
>
> 它将启动并开始为通用API端点提供服务
>
> \$ curl -k https://localhost:443/healthz
>
> ok
>
> 我们还可以列出发现端点，但是结果还不是很令人满意-我们尚未创建API，因此发现为空：
>
> \$ curl -k https://localhost:443/apis
>
> {
>
> \"kind\": \"APIGroupList\",
>
> \"groups\": \[\]
>
> }

让我们从更高的层次看一下：

•我们已经使用推荐的选项和配置启动了自定义APIServer。

•我们有一个标准的处理程序链，其中包括委托认证，委托授权和审核。

•我们有一个HTTPS服务器运行并服务于以下通用端点的请求：/logs, /metrics, /version, /healthz,
and /apis.

图8-3从10,000 feet处显示了这一点。

![he custom API server without
APIs](media/image3.png){width="5.763888888888889in"
height="3.617174103237095in"}

###### *Figure 8-3. The custom API server without APIs*

内部类型和转换
--------------

现在我们已经设置了正在运行的自定义API Server，是时候实际实现API了。
在这样做之前，我们必须了解API版本以及如何在API Server内部处理它们。

每个API服务器都提供许多资源和版本（请参见图2-3）。 一些资源有多个版本。
为了使资源的多个版本成为可能，APIServer在版本之间进行转换。

为了避免版本之间必要转换的二次增长，在实现实际API逻辑时，API
Server使用内部版本。
内部版本通常也称为集线器版本，因为它是所有其他版本之间相互转换的集线器（见图8-4）。
内部API逻辑仅对该集线器版本实施一次。

![onversion from and to the hub
version](media/image4.png){width="5.763888888888889in"
height="6.064673009623797in"}

图8-5显示了API Server在API请求的生命周期中如何利用内部版本：

•用户使用特定版本（例如v1）发送请求。

•API Server解码有效负载并将其转换为内部版本。

•API Server通过允许和验证来传递内部版本。

•API逻辑是为注册表中的内部版本实现的。

•etcd读取和写入版本控制的对象（例如，v2-存储版本）；
也就是说，它会与内部版本相互转换。

•最后，结果将转换为请求版本，在这种情况下为v1。

![onversion of API objects during the life-cycle of a
request](media/image5.png){width="5.763888888888889in"
height="4.508166010498687in"}

###### *Figure 8-5. Conversion of API objects during the lifecycle of a request*

在内部hub版本和外部版本之间的每个边缘上都会进行转换。
在图8-6中，您可以计算每个请求处理程序的转换次数。
在写入操作（如创建和更新）中，至少完成了四次转换，如果在集群中部署了准入webhooks，则转换会更多。
如您所见，在每个API实现中，转换都是至关重要的操作。

![onversions and Defaulting during the life-cycle of a
request](media/image6.png){width="5.763888888888889in"
height="3.3056463254593176in"}

###### *Figure 8-6. Conversions and defaulting during the lifecycle of a request*

除了转换外，图8-6还显示了发生默认设置的时间。
默认是填写未指定字段值的过程。
默认值与转换高度相关，并且总是在外部版本中从用户请求，etcd或准入Webhook传入外部版本时进行，但从hub转换为外部版本时绝不执行。

###### WARNING

转换对于API
Server机制至关重要。从可双向转换的意义上说，所有转换（来回）必须正确，这一点也至关重要。
Roundtrippable意味着我们可以在版本图（图8-4）中来回转换，从随机值开始，并且我们永远不会丢失任何信息;也就是说，转换是双射的或一对一的。例如，我们必须能够从随机（但有效）的v1对象转到内部集线器类型，然后再回到v1alpha1，再回到内部集线器类型，然后再回到v1。生成的对象必须与原始对象相同。

使类型可双向访问通常需要很多思考；它几乎总是驱动新版本的API设计，并且还影响旧类型的扩展，以便存储新版本附带的信息。

简而言之：正确地往返是很难的，有时很难。请参阅\`\` Roundtrip
Testing\'\'以了解如何有效测试往返。

默认逻辑可以在API服务器的生命周期内更改。假设您向类型添加了新字段。用户可能将旧对象存储在磁盘上，或者etcd可能具有旧对象。如果该新字段具有默认值，则在将旧的存储对象发送到API服务器时，或者当用户从etcd中检索一个旧对象时，将设置此字段值。看起来新字段已经永远存在，而实际上，API服务器中的默认过程会在处理请求期间设置字段值。

编写 API Types
--------------

如我们所见，要将API添加到自定义API
Server，我们必须编写内部集线器版本类型和外部版本类型并在它们之间进行转换。这就是比萨示例项目的内容。

传统上将API类型放置在项目的pkg / apis /
group-name软件包中，内部类型使用pkg / apis / group-name /
types.go，外部类型使用pkg / apis / group-name / version /
types.go版本）。因此，对于我们的示例，pkg / apis / restaurant，pkg /
apis / restaurant / v1alpha1 / types.go和pkg / apis / restaurant /
v1beta1 / types.go。

转换将在开发人员编写的自定义转换的pkg / apis / group-name / version /
zz_genic.conversion.go和pkg / apis / group-name / version /
conversion.go中创建。

以类似的方式，将在pkg / apis / group-name / version /
zz_generation.defaults.go和pkg / apis / group-name / version /
defaults.go为defaulter-gen输出创建默认代码以用于自定义默认代码由开发人员编写。在我们的示例中，我们同时具有pkg
/ apis / restaurant / v1alpha1 / defaults.go和pkg / apis / restaurant /
v1beta1 / defaults.go。

我们将在\`\`转换\'\'和\`\`默认设置\'\'中详细介绍转换和默认设置。

除了转换和默认设置外，我们已经在\`\` Anatomy of a
type\'\'中看到了大部分针对CustomResourceDefinitions的过程。在我们的自定义APIServer中，外部版本的本机类型的定义方式完全相同。

此外，对于内部类型（集线器类型），我们有pkg / apis / group-name /
types.go。主要区别在于在后者中register.go文件中的SchemeGroupVersion引用了runtime.APIVersionInternal（这是"
\_\_internal"的快捷方式）。

> *// SchemeGroupVersion is group version used to register these
> objects*
>
> **var** SchemeGroupVersion = schema.GroupVersion{Group: GroupName,
> Version:
>
> runtime.APIVersionInternal}

pkg / apis / group-name /
types.go与外部类型文件之间的另一个区别是缺少JSON和protobuf标签。

###### TIP

某些生成器使用JSON标签来检测types.go文件是用于外部版本还是内部版本。
因此，在复制和粘贴外部类型时请务必删除这些标签，以创建或更新内部类型。

最后但并非最不重要的一点是，有一个帮助程序可将API组的所有版本安装到方案中。
传统上，该助手位于pkg / apis / group-name / install / install.go中。
对于我们的自定义API Server pkg / apis / restaurant / install /
install.go，它看起来像这样简单：

> *// Install registers the API group and adds types to a scheme*
>
> **func** Install(scheme \*runtime.Scheme) {
>
> utilruntime.Must(restaurant.AddToScheme(scheme))
>
> utilruntime.Must(v1beta1.AddToScheme(scheme))
>
> utilruntime.Must(v1alpha1.AddToScheme(scheme))
>
> utilruntime.Must(scheme.SetVersionPriority(
>
> v1beta1.SchemeGroupVersion,
>
> v1alpha1.SchemeGroupVersion,
>
> ))
>
> }

因为我们有多个版本，所以必须定义优先级。
此顺序将用于确定资源的默认存储版本。
它过去也曾在内部客户端（返回内部版本对象的客户端;请参考注释\`\`以前的版本客户端和内部客户端\'\'）中的版本选择中发挥作用。
但是内部客户已弃用，并且即将废弃。甚至API
Server内部的代码将来都将使用外部版本客户端。

转换次数
--------

转换将一个版本的对象转换为另一个版本的对象。
转换是通过转换功能实现的，其中一些功能是手动编写的（按惯例放置到pkg /
apis / group-name / version /
conversion.go中），而另一些则是由conversion-gen自动生成的（按惯例放置在pkg
/ apis / group-中 名称/版本/zz_genic.conversion.go）。

使用Convert（）方法通过方案（请参见"Scheme"）启动转换，将源对象传入和目标对象传出：

> **func** (s \*Scheme) Convert(in, out **interface**{}, context
> **interface**{}) **error**

上下文描述如下：

// \...一个可选字段，调用者可以使用该字段将信息传递给转换函数。

它仅在非常特殊的情况下使用，通常为零。在本章的后面，我们将讨论转换函数的作用域，它使我们能够从转换函数内部访问此上下文。

为了进行实际的转换，该方案知道所有Golang
API类型，它们的GroupVersionKinds以及GroupVersionKinds之间的转换函数。
为此，conversion-gen通过本地方案构建器注册生成的转换函数。在我们的示例自定义API
Server中，zz_genic.conversion.go文件开始如下：

> **func** init() {
>
> localSchemeBuilder.Register(RegisterConversions)
>
> }
>
> *// RegisterConversions adds conversion functions to the given
> scheme.*
>
> *// Public to allow building arbitrary schemes.*
>
> **func** RegisterConversions(s \*runtime.Scheme) **error** {
>
> **if** err := s.AddGeneratedConversionFunc(
>
> (\*Topping)(**nil**),
>
> (\*restaurant.Topping)(**nil**),
>
> **func**(a, b **interface**{}, scope conversion.Scope) **error** {
>
> **return** Convert_v1alpha1_Topping_To_restaurant_Topping(
>
> a.(\*Topping),
>
> b.(\*restaurant.Topping),
>
> scope,
>
> )
>
> },
>
> ); err != **nil** {
>
> **return** err
>
> }
>
> \...
>
> **return** **nil**
>
> }
>
> \...

生成函数Convert_v1alpha1_Topping_To_restaurant_Topping（）。
它接受一个v1alpha1对象并将其转换为内部类型。

###### NOTE

前面的复杂类型转换将类型转换函数转换为统一类型的func（a，b interface
{}，scope conversion.Scope）错误。
该方案使用后一种类型，因为它可以在不使用反射的情况下调用它们。
由于有许多必要的分配，反射速度很慢。

如果conversion_gen在包中找到具有Convert_source-package-base-basename_KindTo_target-package-basename_Kind转换功能命名模式的包中的手动编写功能，则conversion.go中的手动编写转换优先于生成。例如：

> **func** Convert_v1alpha1_PizzaSpec_To_restaurant_PizzaSpec(
>
> in \*PizzaSpec,
>
> out \*restaurant.PizzaSpec,
>
> s conversion.Scope,
>
> ) **error** {
>
> \...
>
> **return** **nil**
>
> }

在最简单的情况下，转换函数只是将值从源复制到目标对象。但是对于前面的示例，它将v1alpha1
Pizza规范转换为内部类型，简单复制是不够的。我们必须适应不同的结构，实际上如下所示：

> **func** Convert_v1alpha1_PizzaSpec_To_restaurant_PizzaSpec(
>
> in \*PizzaSpec,
>
> out \*restaurant.PizzaSpec,
>
> s conversion.Scope,
>
> ) **error** {
>
> idx := **map**\[**string**\]**int**{}
>
> **for** \_, top := **range** in.Toppings {
>
> **if** i, duplicate := idx\[top\]; duplicate {
>
> out.Toppings\[i\].Quantity++
>
> **continue**
>
> }
>
> idx\[top\] = len(out.Toppings)
>
> out.Toppings = append(out.Toppings, restaurant.PizzaTopping{
>
> Name: top,
>
> Quantity: 1,
>
> })
>
> }
>
> **return** **nil**
>
> }

显然，没有任何代码生成可以如此聪明地预见到用户在定义这些不同类型时的意图。

请注意，在转换期间，源对象绝不能突变。但这是完全正常的，并且通常出于性能原因，强烈建议在类型匹配时重用目标对象中源的数据结构。

这是如此重要，以至于我们在警告中重申它，因为它不仅对转换的实现有影响，而且对转换的调用者和转换输出的使用者也有影响。

###### WARNING

转换函数不得更改源对象，但允许输出与源共享数据结构。这意味着，如果原始对象必须不发生突变，则转换输出的使用者必须确保不使该对象发生突变。

例如，假设您在内部版本中有一个pod \* core.Pod，并将其转换为v1作为podv1
\*
corev1.Pod，然后对生成的Podv1进行突变。这也可能会改变原始的Pod。如果来自informers的Pod，这是非常危险的，因为informers具有共享的缓存，并且变异Pod会使缓存不一致。

因此，请注意转换的这种特性，并在必要时进行深入复制，以避免不必要的和潜在的危险突变。

虽然这种共享数据结构会带来一定的风险，但在许多情况下它也可以避免不必要的分配。生成的代码是如此广泛，以至于生成器比较源和目标结构，并使用Golang的不安全软件包通过简单的类型转换将指针转换为相同内存布局的结构。因为在我们的示例中披萨的内部类型和v1beta1类型具有相同的内存布局，所以我们得到以下信息：

> **func** autoConvert_restaurant_PizzaSpec_To_v1beta1_PizzaSpec(
>
> in \*restaurant.PizzaSpec,
>
> out \*PizzaSpec,
>
> s conversion.Scope,
>
> ) **error** {
>
> out.Toppings = \*(\*\[\]PizzaTopping)(unsafe.Pointer(&in.Toppings))
>
> **return** **nil**
>
> }

在机器语言级别上，这是一个NOOP，因此要尽可能快。在这种情况下，它避免了分配一个分片并逐项复制的方式。

最后但并非最不重要的一点是关于转换函数的第三个参数的一些单词：转换范围，conversion.Scope。

转换范围提供对许多转换元级别值的访问。例如，它允许我们通过以下方式访问传递给方案的转换in,
out interface{}, context interface{})错误方法的上下文值：

> s.Meta().Context

它还允许我们通过s.Convert调用子类型的方案转换，或者根本不通过s.DefaultConvert调用注册的转换函数。

但是，在大多数转换情况下，根本不需要使用范围。为了简单起见，您可以忽略它的存在，直到遇到棘手的情况，其中需要比源对象和目标对象更多的上下文。

Defaulting
----------

默认设置是API请求生命周期中的步骤，用于设置传入对象（来自client或来自etcd）中省略字段的默认值。例如，pod有一个restartPolicy字段。如果用户未指定，则默认值为始终。

想象一下，我们在2014年左右使用的是非常旧的Kubernetes版本。那时restartPolicy字段刚刚在当时的最新版本中引入了系统。集群升级后，etcd中有一个pod，没有restartPolicy字段。一个Kubectl
get
pod将从etcd读取旧的pod，默认代码将始终添加默认值。从用户的角度来看，神奇的是旧的pod突然有了新的restartPolicy字段。

请参考图8-6，以查看今天在Kubernetes请求管道中发生默认设置的位置。请注意，仅对外部类型（而不是内部类型）进行默认设置。

现在让我们看一下默认代码。与转换类似，默认设置由k8s.io/apiserver代码通过该方案启动。因此，我们必须将默认函数注册到我们的自定义类型的方案中。

同样，与转换类似，大多数默认代码只是使用defaulter-gen二进制文件生成的。它遍历API类型并在pkg
/ apis / group-name / version /
zz_genic.defaults.go中创建默认函数。除了为子结构调用默认函数外，该代码默认不会执行任何操作。

您可以按照默认功能命名模式SetDefaultsKind定义自己的默认逻辑：

> **func** SetDefaults*Kind*(obj \**Type*) {
>
> \...
>
> }

此外，与转换不同，我们必须在本地方案构建器上手动调用生成函数的注册。
不幸的是，这不是自动完成的：

> **func** init() {
>
> localSchemeBuilder.Register(RegisterDefaults)
>
> }

这里，RegisterDefaults是在包pkg / apis / group-name / version /
zz_genic.defaults.go中生成的。

对于defaulting代码，了解用户何时设置字段以及何时未设置字段至关重要。在许多情况下这还不清楚。

Golang的每种类型都有零值，如果在传递的JSON或protobuf中未找到字段，则将其设置为零。
想象一下boolean类型foo的默认值为true。 零值为false。
不幸的是，不清楚是由于用户输入而设置了false还是因为false只是布尔值的零值。

为了避免这种情况，通常必须在Golang
API类型中使用指针类型（例如，在前面的情况下为\* bool）。
用户提供的false会导致指向虚假值的非空布尔值指针，而用户提供的true将导致非零指向布尔型的指针和真实值。未提供的字段将导致nil。可以在defaulting代码中检测到：

> **func** SetDefaults*Kind*(obj \**Type*) {
>
> **if** obj.Foo == **nil** {
>
> x := **true**
>
> obj.Foo = &x
>
> }
>
> }

这提供了所需的语义：" foo默认为true"。

###### TIP

这种使用指针的技巧适用于诸如字符串之类的原始类型。对于maps和arrays，如果不标识nil maps/arrays和empty
maps/arrays.，通常很难达到往返能力。 因此，Kubernetes中empty
maps/arrays.的大多数默认设置都在两种情况下都采用默认设置，以解决编码和解码错误。

Roundtrip Testing
-----------------

正确实现转化非常困难。Roundtrip
tests是在随机测试中自动检查转换是否按计划进行且在从所有已知组版本进行转换时不会丢失数据的基本工具。

Roundtrip tests通常与install.go文件一起放置（例如，放入pkg / apis /
restaurant / install / roundtrip_test.go中），只需从API
Machinery调用Roundtrip tests功能即可：

> **import** (
>
> \...
>
> \"k8s.io/apimachinery/pkg/api/apitesting/roundtrip\"
>
> restaurantfuzzer
> \"github.com/programming-kubernetes/pizza-apiserver/pkg/apis/
>
> restaurant/fuzzer\"
>
> )
>
> **func** TestRoundTripTypes(t \*testing.T) {
>
> roundtrip.RoundTripTestForAPIGroup(t, Install, restaurantfuzzer.Funcs)
>
> }

在内部，RoundTripTestForAPIGroup调用使用Install函数将API组安装到临时方案中。然后使用给定的fuzzer在内部版本中创建随机对象，然后将它们转换为某些外部版本并返回到内部。结果对象必须与原始对象等效。这个测试在所有外部版本中都要进行成百上千次。

fuzzer是一个函数，它返回内部类型及其子类型的随机化函数片段。在我们的示例中，fuzzer放在pkg/api/restaurant/fuzzer/fuzzer.go包中，并包含spec结构的随机化器：

> *// Funcs returns the fuzzer functions for the restaurant api group.*
>
> **var** Funcs = **func**(codecs runtimeserializer.CodecFactory)
> \[\]**interface**{} {
>
> **return** \[\]**interface**{}{
>
> **func**(s \*restaurant.PizzaSpec, c fuzz.Continue) {
>
> c.FuzzNoCustom(s) *// fuzz first without calling this function again*
>
> *// avoid empty Toppings because that is defaulted*
>
> **if** len(s.Toppings) == 0 {
>
> s.Toppings = \[\]restaurant.PizzaTopping{
>
> {\"salami\", 1},
>
> {\"mozzarella\", 1},
>
> {\"tomato\", 1},
>
> }
>
> }
>
> seen := **map**\[**string**\]**bool**{}
>
> **for** i := **range** s.Toppings {
>
> *// make quantity strictly positive and of reasonable size*
>
> s.Toppings\[i\].Quantity = 1 + c.Intn(10)
>
> *// remove duplicates*
>
> **for** {
>
> **if** !seen\[s.Toppings\[i\].Name\] {
>
> **break**
>
> }
>
> s.Toppings\[i\].Name = c.RandString()
>
> }
>
> seen\[s.Toppings\[i\].Name\] = **true**
>
> }
>
> },
>
> }
>
> }

如果没有给定随机化器函数，底层库github.com/google/gofuzz通常会尝试通过为基类型设置随机值来模糊对象，并递归地跳入指针、结构、映射和切片中，最终调用自定义随机化器函数（如果由开发人员给定）。

为其中一种类型编写随机化函数时，首先调用c.FuzzNoCustom是很方便的。它随机化给定的对象，并为子结构调用自定义函数，但不为s本身调用自定义函数。然后开发人员可以限制和修复随机值，使对象有效。

###### WARNING

为了覆盖尽可能多的有效对象，使模糊器尽可能通用是很重要的。如果fuzzer过于严格，那么测试覆盖率将很差。在许多情况下，在Kubernetes的发展过程中，没有捕捉到回归，因为fuzzers的位置不好。

另一方面，fuzzer只需要考虑验证的对象，并且是可以在外部版本中定义的实际对象的投影。通常必须限制c.FuzzNoCustom设置的随机值，使随机化对象变得有效。例如，如果验证仍将拒绝任意字符串，则包含URL的字符串不必对任意值进行往返。

前面的PizzaSpec示例首先调用c.FuzzNoCustom，然后通过以下方法修复对象：

•违约情况下的toppings

•为每个顶部设置一个合理的数量（没有这个，v1alpha1的转换将在复杂性上爆炸，将大量引入字符串列表）

•规范化toppings名称，因为我们知道比萨规范中的重复toppings永远不会往返（对于内部类型，请注意v1alpha1类型有重复）

Validation
----------

传入对象在被反序列化、默认设置并转换为内部版本后不久进行验证。图8-5显示了在执行实际创建或更新逻辑之前很久，如何在更改允许插件和验证允许插件之间进行验证。

这意味着验证只需对内部版本执行一次，而不是对所有外部版本执行一次。这样做的好处是，它明显节省了实现工作，还确保了版本之间的一致性。另一方面，这意味着验证错误不引用外部版本。这实际上可以用Kubernetes的资源来观察，但实际上这并不是什么大问题。

在本节中，我们将讨论验证函数的实现。下一节将介绍到自定义API
Server的连接，即从配置通用注册表的策略调用验证。换言之，图8-5有点误导，有利于视觉简洁。

现在，看看strategy内部验证的切入点就足够了：

> **func** (pizzaStrategy) Validate(
>
> ctx context.Context, obj runtime.Object,
>
> ) field.ErrorList {
>
> pizza := obj.(\*restaurant.Pizza)
>
> **return** validation.ValidatePizza(pizza)
>
> }

这将调用API group
pkg/API/group/validation的验证包中的ValidateKind（obj\*Kind）字段.ErrorList验证函数。

验证函数返回一个错误列表。它们通常以相同的样式编写，将返回值附加到错误列表，同时递归地跳入类型，每个结构一个验证函数：

> *// ValidatePizza validates a Pizza.*
>
> **func** ValidatePizza(f \*restaurant.Pizza) field.ErrorList {
>
> allErrs := field.ErrorList{}
>
> errs := ValidatePizzaSpec(&f.Spec, field.NewPath(\"spec\"))
>
> allErrs = append(allErrs, errs\...)
>
> **return** allErrs
>
> }
>
> *// ValidatePizzaSpec validates a PizzaSpec.*
>
> **func** ValidatePizzaSpec(
>
> s \*restaurant.PizzaSpec,
>
> fldPath \*field.Path,
>
> ) field.ErrorList {
>
> allErrs := field.ErrorList{}
>
> prevNames := **map**\[**string**\]**bool**{}
>
> **for** i := **range** s.Toppings {
>
> **if** s.Toppings\[i\].Quantity \<= 0 {
>
> allErrs = append(allErrs, field.Invalid(
>
> fldPath.Child(\"toppings\").Index(i).Child(\"quantity\"),
>
> s.Toppings\[i\].Quantity,
>
> \"cannot be negative or zero\",
>
> ))
>
> }
>
> **if** len(s.Toppings\[i\].Name) == 0 {
>
> allErrs = append(allErrs, field.Invalid(
>
> fldPath.Child(\"toppings\").Index(i).Child(\"name\"),
>
> s.Toppings\[i\].Name,
>
> \"cannot be empty\",
>
> ))
>
> } **else** {
>
> **if** prevNames\[s.Toppings\[i\].Name\] {
>
> allErrs = append(allErrs, field.Invalid(
>
> fldPath.Child(\"toppings\").Index(i).Child(\"name\"),
>
> s.Toppings\[i\].Name,
>
> \"must be unique\",
>
> ))
>
> }
>
> prevNames\[s.Toppings\[i\].Name\] = **true**
>
> }
>
> }
>
> **return** allErrs
>
> }

请注意如何使用子调用和索引调用维护字段路径。字段路径是JSON路径，在出现错误时打印。

通常会有一组附加的验证函数，这些函数在更新时略有不同（而前面的一组用于创建）。在我们的示例API
Server中，这可能如下所示：

> **func** (pizzaStrategy) ValidateUpdate(
>
> ctx context.Context,
>
> obj, old runtime.Object,
>
> ) field.ErrorList {
>
> objPizza := obj.(\*restaurant.Pizza)
>
> oldPizza := old.(\*restaurant.Pizza)
>
> **return** validation.ValidatePizzaUpdate(objPizza, oldPizza)
>
> }

这可用于验证没有更改只读字段。通常，更新验证也会调用常规验证函数，并且只添加与更新相关的检查。

###### NOTE

Validation是在创建时限制对象名称的正确位置，例如，仅限于单个单词，或不包括任何非字母数字字符。

实际上，任何ObjectMeta字段在技术上都可以通过自定义方式进行限制，但这对于许多字段来说并不理想，因为它可能会破坏核心API机器的行为。许多资源限制了名称，因为例如，该名称将显示在其他系统或其他需要特殊格式名称的上下文中。

但是，即使在自定义API
Server中存在特殊的ObjectMeta验证，在任何情况下，在自定义验证通过后，通用注册表都将根据通用规则进行验证。这允许我们首先从自定义代码返回更具体的错误消息。

Registry and Strategy
---------------------

到目前为止，我们已经了解了如何定义和验证API类型。下一步是这些API类型的REST逻辑的实现，图8-7显示了注册表作为API组实现的中心部分。k8s.io/apiserver中的通用REST请求处理程序代码调用注册表。

![esource storage and generic
registry](media/image7.png){width="5.763888888888889in"
height="4.389581146106736in"}

###### *Figure 8-7. Resource storage and generic registry*

### GENERIC REGISTRY

REST逻辑通常由泛型注册表实现。顾名思义，它是包k8s.io/apiserver/pkg/registry/rest中注册表接口的通用实现。

泛型注册表实现"normal"资源的默认REST行为。几乎所有的Kubernetes资源都使用这个实现。只有少数，特别是那些不持久化对象的对象（例如SubjectAccessReview；请参阅"Delegated
Authorization"）具有自定义实现。

在k8s.io/apiserver/pkg/registry/rest/rest.go中，您将发现许多接口，大致对应于HTTP动词和某些API功能。如果接口是由注册表实现的，那么API端点代码将提供某些REST特性。因为通用注册表实现了大部分k8s.io/API
Server/pkg/registry/rest接口，所以使用它的资源将支持所有默认的Kubernetes
HTTP动词（请参阅"The HTTP Interface of the API
Server"）。下面是实现的那些接口的列表，其中的GoDoc描述来自Kubernetes源代码：

CollectionDeleter

> An object that can delete a collection of RESTful resources
>
> 可以删除RESTful资源集合的对象

Creater

> An object that can create an instance of a RESTful object
>
> 可以创建RESTful对象实例的对象

CreaterUpdater

> A storage object that must support both create and update operations
>
> 必须同时支持创建和更新操作的存储对象

Exporter

> An object that knows how to strip a RESTful resource for export
>
> 知道如何剥离RESTful资源以便导出的对象

Getter

> An object that can retrieve a named RESTful resource
>
> 可以检索命名RESTful资源的对象

GracefulDeleter

> An object that knows how to pass deletion options to allow delayed
> deletion of a RESTful object
>
> 传递删除选项以允许延迟删除RESTful对象的对象

Lister

> An object that can retrieve resources that match the provided field
> and label criteria
>
> 可以检索与提供的字段和标签条件匹配的资源的对象

Patcher

> A storage object that supports both get and update
>
> 同时支持get和update的存储对象

Scoper

> An object that must be specified and indicates what scope the resource
>
> 必须指定并指示资源作用域的对象

Updater

> An object that can update an instance of a RESTful object
>
> 可以更新RESTful对象实例的对象

Watcher

> An object that should be implemented by all storage objects that want
> to offer the ability to watch for changes through the Watch API
>
> 应该由所有希望通过监视API提供监视更改能力的存储对象实现的对象

Let's look at one of the interfaces, Creater:

> *// Creater is an object that can create an instance of a RESTful
> object.*
>
> **type** Creater **interface** {
>
> *// New returns an empty object that can be used with Create after
> request*
>
> *// data has been put into it.*
>
> *// This object must be a pointer type for use with
> Codec.DecodeInto(\[\]byte,*
>
> *// runtime.Object)*
>
> New() runtime.Object
>
> *// Create creates a new version of a resource.*
>
> Create(
>
> ctx context.Context,
>
> obj runtime.Object,
>
> createValidation ValidateObjectFunc,
>
> options \*metav1.CreateOptions,
>
> ) (runtime.Object, **error**)
>
> }

实现此接口的注册表将能够创建对象。与NamedCreater相反，新对象的名称要么来自ObjectMeta.name，要么通过ObjectMeta.GenerateName生成。如果注册表实现NamedCreater，则名称也可以通过HTTP路径传递。

重要的是要理解，实现的接口确定在将API安装到自定义API
Server时创建的API端点将支持哪些谓词。请参阅"API
Installation"，了解如何在代码中完成此操作。

### STRATEGY

通用注册表可以使用名为策略的对象在一定程度上进行自定义。正如我们在"Validation"中看到的，该策略提供了对验证等功能的回调。

该策略实现了此处列出的REST策略接口及其GoDoc描述（有关其定义，请参见k8s.io/apiser/pkg/registry/REST）：

RESTCreateStrategy

> Defines the minimum validation, accepted input, and name generation
> behavior to create an object that follows Kubernetes API conventions.
>
> 定义最小验证、接受的输入和名称生成行为，以创建遵循Kubernetes
> API约定的对象。

RESTDeleteStrategy

> Defines deletion behavior on an object that follows Kubernetes API
> conventions.
>
> 在遵循Kubernetes API约定的对象上定义删除行为。

RESTGracefulDeleteStrategy

> Must be implemented by the registry that supports graceful deletion.
>
> 必须由支持正常删除的注册表实现

GarbageCollectionDeleteStrategy

> Must be implemented by the registry that wants to orphan dependents by
> default.
>
> 必须由希望在默认情况下孤立依赖项的注册表实现

RESTExportStrategy

> Defines how to export a Kubernetes object.
>
> 定义如何导出Kubernetes对象。

RESTUpdateStrategy

> Defines the minimum validation, accepted input, and name generation
> behavior to update an object that follows Kubernetes API conventions.

定义最小验证、接受的输入和名称生成行为，以更新遵循Kubernetes
API约定的对象。

让我们再来看看创建strategy：

> **type** RESTCreateStrategy **interface** {
>
> runtime.ObjectTyper
>
> *// The name generator is used when the standard GenerateName field is
> set.*
>
> *// The NameGenerator will be invoked prior to validation.*
>
> names.NameGenerator
>
> *// NamespaceScoped returns true if the object must be within a
> namespace.*
>
> NamespaceScoped() **bool**
>
> *// PrepareForCreate is invoked on create before validation to
> normalize*
>
> *// the object. For example: remove fields that are not to be
> persisted,*
>
> *// sort order-insensitive list fields, etc. This should not remove
> fields*
>
> *// whose presence would be considered a validation error.*
>
> *//*
>
> *// Often implemented as a type check and an initailization or
> clearing of*
>
> *// status. Clear the status because status changes are internal.
> External*
>
> *// callers of an api (users) should not be setting an initial status
> on*
>
> *// newly created objects.*
>
> PrepareForCreate(ctx context.Context, obj runtime.Object)
>
> *// Validate returns an ErrorList with validation errors or nil.
> Validate*
>
> *// is invoked after default fields in the object have been filled in*
>
> *// before the object is persisted. This method should not mutate the*
>
> *// object.*
>
> Validate(ctx context.Context, obj runtime.Object) field.ErrorList
>
> *// Canonicalize allows an object to be mutated into a canonical form.
> This*
>
> *// ensures that code that operates on these objects can rely on the
> common*
>
> *// form for things like comparison. Canonicalize is invoked after*
>
> *// validation has succeeded but before the object has been
> persisted.*
>
> *// This method may mutate the object. Often implemented as a type
> check or*
>
> *// empty method.*
>
> Canonicalize(obj runtime.Object)
>
> }

嵌入式ObjectTyper识别对象;
也就是说，它检查注册表中是否支持请求中的对象。
这对于创建正确的对象类型非常重要（例如，通过" foo"资源，仅应创建"
Foo"资源）。

NameGenerator显然是从ObjectMeta.GenerateName字段生成名称的。

通过NamespaceScoped，该策略可以通过返回false或true来支持群集范围或命名空间的资源。

验证之前，传入对象将调用PrepareForCreate方法。

我们之前在"Validation"中看到的"Validation
functions"：这是验证功能的入口点。

最后，规范化方法进行规范化（例如，切片的排序）。

### WIRING A STRATEGY INTO THE GENERIC REGISTRY

策略对象已插入通用注册表实例。
这是我们在GitHub上的自定义API服务器的REST存储构造函数：

> *// NewREST returns a RESTStorage object that will work against API
> services.*
>
> **func** NewREST(
>
> scheme \*runtime.Scheme,
>
> optsGetter generic.RESTOptionsGetter,
>
> ) (\*registry.REST, **error**) {
>
> strategy := NewStrategy(scheme)
>
> store := &genericregistry.Store{
>
> NewFunc: **func**() runtime.Object { **return** &restaurant.Pizza{} },
>
> NewListFunc: **func**() runtime.Object { **return**
> &restaurant.PizzaList{} },
>
> PredicateFunc: MatchPizza,
>
> DefaultQualifiedResource: restaurant.Resource(\"pizzas\"),
>
> CreateStrategy: strategy,
>
> UpdateStrategy: strategy,
>
> DeleteStrategy: strategy,
>
> }
>
> options := &generic.StoreOptions{
>
> RESTOptions: optsGetter,
>
> AttrFunc: GetAttrs,
>
> }
>
> **if** err := store.CompleteWithOptions(options); err != **nil** {
>
> **return** **nil**, err
>
> }
>
> **return** &registry.REST{store}, **nil**
>
> }

它实例化通用注册表对象genericregistry.Store并设置一些字段。
这些字段中的许多都是可选的，store.CompleteWithOptions将是默认值，如果开发人员未设置它们。

您可以看到如何首先通过NewStrategy构造函数实例化自定义策略，然后将其插入注册表以创建，更新和删除运算符。

此外，将NewFunc设置为创建新的对象实例，并将NewListFunc字段设置为创建新的对象列表。
PredicateFunc将选择器（可以传递给列表请求）转换为谓词函数，过滤运行时对象。

返回的对象是一个REST注册表，只是我们的示例项目中的一个简单包装器，围绕通用注册表对象创建自己的类型：

> **type** REST **struct** {
>
> \*genericregistry.Store
>
> }

有了这个，我们就可以实例化我们的API并将其连接到自定义API Server中。
在下一节中，我们将介绍如何从中创建HTTP处理程序。

API Installation
----------------

要在API Server中激活API，需要执行两个步骤：

1.必须将API版本安装到API类型的（以及转换和默认函数）服务器方案中。

2.必须将API版本安装到服务器HTTP多路复用器（mux）中。

第一步通常是在API Server引导程序的任意位置使用初始化函数来完成的。
这是在示例自定义API服务器的pkg / apiserver /
apiserver.go中完成的，其中定义了serverConfig和CustomServer对象（请参见\`\`
Options and Config Pattern and Startup Plumbing \'\'）：

> **import** (
>
> \...
>
> \"k8s.io/apimachinery/pkg/runtime\"
>
> \"k8s.io/apimachinery/pkg/runtime/serializer\"
>
> \"github.com/programming-kubernetes/pizza-apiserver/pkg/apis/restaurant/install\"
>
> )
>
> **var** (
>
> Scheme = runtime.NewScheme()
>
> Codecs = serializer.NewCodecFactory(Scheme)
>
> )
>
> 然后对于每个应提供的API组，我们调用Install（）函数：
>
> **func** init() {
>
> install.Install(Scheme)
>
> }

由于技术原因，我们还必须在方案中添加一些与发现相关的类型（在将来的k8s.io/apiserver版本中可能会消失）：

> **func** init() {
>
> *// we need to add the options to empty v1*
>
> *// TODO: fix the server code to avoid this*
>
> metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: \"v1\"})
>
> *// TODO: keep the generic API server from wanting this*
>
> unversioned := schema.GroupVersion{Group: \"\", Version: \"v1\"}
>
> Scheme.AddUnversionedTypes(unversioned,
>
> &metav1.Status{},
>
> &metav1.APIVersions{},
>
> &metav1.APIGroupList{},
>
> &metav1.APIGroup{},
>
> &metav1.APIResourceList{},
>
> )
>
> }

这样，我们就在全局方案中注册了我们的API类型，包括conversion和defaulting功能。
换句话说，图8-3的空方案现在了解有关我们类型的所有信息。

第二步是将API组添加到HTTP多路复用器。
嵌入到我们的CustomServer结构中的通用API
Server代码提供了InstallAPIGroup（apiGroupInfo \*
APIGroupInfo）错误方法，该方法为API组设置了整个请求管道。

我们唯一要做的就是提供一个正确填充的APIGroupInfo结构。
我们在completedConfig类型的构造函数New（）（\*
CustomServer，错误）中进行此操作：

> *// New returns a new instance of CustomServer from the given config.*
>
> **func** (c completedConfig) New() (\*CustomServer, **error**) {
>
> genericServer, err := c.GenericConfig.New(\"pizza-apiserver\",
>
> genericapiserver.NewEmptyDelegate())
>
> **if** err != **nil** {
>
> **return** **nil**, err
>
> }
>
> s := &CustomServer{
>
> GenericAPIServer: genericServer,
>
> }
>
> apiGroupInfo :=
> genericapiserver.NewDefaultAPIGroupInfo(restaurant.GroupName,
>
> Scheme, metav1.ParameterCodec, Codecs)
>
> v1alpha1storage := **map**\[**string**\]rest.Storage{}
>
> pizzaRest := pizzastorage.NewREST(Scheme,
> c.GenericConfig.RESTOptionsGetter)
>
> v1alpha1storage\[\"pizzas\"\] = customregistry.RESTInPeace(pizzaRest)
>
> toppingRest := toppingstorage.NewREST(
>
> Scheme, c.GenericConfig.RESTOptionsGetter,
>
> )
>
> v1alpha1storage\[\"toppings\"\] =
> customregistry.RESTInPeace(toppingRest)
>
> apiGroupInfo.VersionedResourcesStorageMap\[\"v1alpha1\"\] =
> v1alpha1storage
>
> v1beta1storage := **map**\[**string**\]rest.Storage{}
>
> pizzaRest = pizzastorage.NewREST(Scheme,
> c.GenericConfig.RESTOptionsGetter)
>
> v1beta1storage\[\"pizzas\"\] = customregistry.RESTInPeace(pizzaRest)
>
> apiGroupInfo.VersionedResourcesStorageMap\[\"v1beta1\"\] =
> v1beta1storage
>
> **if** err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err
> != **nil** {
>
> **return** **nil**, err
>
> }
>
> **return** s, **nil**
>
> }

APIGroupInfo引用了我们在"registry and
Strategy"中通过策略定制的通用注册表。对于每个组版本和资源，我们使用实现的构造函数创建注册表的实例。

customregistry.RESTInPeace包装器只是一个在注册表构造函数返回错误时死机的帮助器：

> **func** RESTInPeace(storage rest.StandardStorage, err **error**)
> rest.StandardStorage {
>
> **if** err != **nil** {
>
> err = fmt.Errorf(\"unable to create REST storage: %v\", err)
>
> panic(err)
>
> }
>
> **return** storage
>
> }

注册表本身是独立于版本的，因为它对内部对象进行操作；请参阅图8-5。因此，我们为每个版本调用相同的注册表构造函数。

对installappigroup的调用最终会引导我们找到一个完整的定制API服务器，可以为我们的定制API组提供服务，如图8-7所示。

在经历了这么多繁重的工作之后，是时候看看我们的新API组在起作用了。为此，我们启动服务器，如"第一次启动"所示。但这次发现信息不是空的，而是显示我们新注册的资源：

> \$ curl -k https://localhost:443/apis
>
> {
>
> \"kind\": \"APIGroupList\",
>
> \"groups\": \[
>
> {
>
> \"name\": \"restaurant.programming-kubernetes.info\",
>
> \"versions\": \[
>
> {
>
> \"groupVersion\": \"restaurant.programming-kubernetes.info/v1beta1\",
>
> \"version\": \"v1beta1\"
>
> },
>
> {
>
> \"groupVersion\": \"restaurant.programming-kubernetes.info/v1alpha1\",
>
> \"version\": \"v1alpha1\"
>
> }
>
> \],
>
> \"preferredVersion\": {
>
> \"groupVersion\": \"restaurant.programming-kubernetes.info/v1beta1\",
>
> \"version\": \"v1beta1\"
>
> },
>
> \"serverAddressByClientCIDRs\": \[
>
> {
>
> \"clientCIDR\": \"0.0.0.0/0\",
>
> \"serverAddress\": \":443\"
>
> }
>
> \]
>
> }
>
> \]
>
> }

这样，我们几乎达到了为餐厅提供API的目标。我们已经连接了API组版本，转换已经就位，验证正在工作。

缺少的是检查批萨中提到的配料是否确实存在于集群中。我们可以将其添加到验证函数中。但传统上这些只是格式验证函数，它们是静态的，不需要其他资源来运行。

相比之下，在接受下一节的主题时实现了更复杂的检查。

Admission
---------

每个请求在被解组、默认设置和转换为内部类型后都会通过许可插件链；请参阅图8-2。更准确地说，请求通过两次准入：

• mutating插件

• validating插件

许可插件可以是mutating和validating，因此可能会被许可机制调用两次：

•一旦进入mutation阶段，依次调用所有mutation插件

•在validation阶段，为所有验证validation调用（可能并行化）

更准确地说，一个插件可以实现validation和mutation接纳接口，两种情况下都有两种不同的方法。

###### NOTE

在分离到validation和mutation之前，每个插件只有一个调用。几乎不可能一直关注每个插件都做了哪些变异，以及哪些允许插件这样做才有意义，从而为用户带来一致的行为。

这种两步体系结构至少确保在最后对所有插件进行验证，从而保证一致性。

此外，链（即，两个接纳阶段的插件顺序）是相同的。插件总是在两个阶段同时启用或禁用。

允许插件（至少如本章所述在Golang中实现的插件）使用内部类型。相反，webhook允许插件（请参阅"Admission
webhook"）是基于外部类型的，涉及到到到webhook的转换和返回（在webhook发生变异的情况下）。

在所有这些理论之后，让我们进入代码。

### IMPLEMENTATION

admission是一种实现以下功能的类型：

• admission插件接口

•可选的MutatingInterface

•可选的ValidatingInterface

这三个都可以在包k8s.io/apiserver/pkg/admission中找到：

> *// Operation is the type of resource operation being checked for*
>
> *// admission control*
>
> **type** Operation **string**.
>
> *// Operation constants*
>
> **const** (
>
> Create Operation = \"CREATE\"
>
> Update Operation = \"UPDATE\"
>
> Delete Operation = \"DELETE\"
>
> Connect Operation = \"CONNECT\"
>
> )
>
> *// Interface is an abstract, pluggable interface for Admission
> Control*
>
> *// decisions.*
>
> **type** Interface **interface** {
>
> *// Handles returns true if this admission controller can handle the
> given*
>
> *// operation where operation can be one of CREATE, UPDATE, DELETE,
> or*
>
> *// CONNECT.*
>
> Handles(operation Operation) **bool**.
>
> }
>
> **type** MutationInterface **interface** {
>
> Interface
>
> *// Admit makes an admission decision based on the request
> attributes.*
>
> Admit(a Attributes, o ObjectInterfaces) (err **error**)
>
> }
>
> *// ValidationInterface is an abstract, pluggable interface for
> Admission Control*
>
> *// decisions.*
>
> **type** ValidationInterface **interface** {
>
> Interface
>
> *// Validate makes an admission decision based on the request
> attributes.*
>
> *// It is NOT allowed to mutate.*
>
> Validate(a Attributes, o ObjectInterfaces) (err **error**)
>
> }

可以看到接口方法句柄负责对操作进行筛选。mutating插件通过Admit调用，验证插件通过Validate调用。

ObjectInterfaces允许访问通常由方案实现的助手：

> **type** ObjectInterfaces **interface** {
>
> *// GetObjectCreater is the ObjectCreater for the requested object.*
>
> GetObjectCreater() runtime.ObjectCreater
>
> *// GetObjectTyper is the ObjectTyper for the requested object.*
>
> GetObjectTyper() runtime.ObjectTyper
>
> *// GetObjectDefaulter is the ObjectDefaulter for the requested
> object.*
>
> GetObjectDefaulter() runtime.ObjectDefaulter
>
> *// GetObjectConvertor is the ObjectConvertor for the requested
> object.*
>
> GetObjectConvertor() runtime.ObjectConvertor
>
> }

传递给插件的属性（通过accept或Validate或两者）基本上包含从请求中提取的所有信息，这些信息对于实现高级检查非常重要：

> *// Attributes is an interface used by AdmissionController to get
> information*
>
> *// about a request that is used to make an admission decision.*
>
> **type** Attributes **interface** {
>
> *// GetName returns the name of the object as presented in the
> request.*
>
> *// On a CREATE operation, the client may omit name and rely on the*
>
> *// server to generate the name. If that is the case, this method
> will*
>
> *// return the empty string.*
>
> GetName() **string**
>
> *// GetNamespace is the namespace associated with the request (if
> any).*
>
> GetNamespace() **string**
>
> *// GetResource is the name of the resource being requested. This is
> not the*
>
> *// kind. For example: pods.*
>
> GetResource() schema.GroupVersionResource
>
> *// GetSubresource is the name of the subresource being requested.
> This is a*
>
> *// different resource, scoped to the parent resource, but it may have
> a*
>
> *// different kind.*
>
> *// For instance, /pods has the resource \"pods\" and the kind
> \"Pod\", while*
>
> *// /pods/foo/status has the resource \"pods\", the sub resource
> \"status\", and*
>
> *// the kind \"Pod\" (because status operates on pods). The binding
> resource for*
>
> *// a pod, though, may be /pods/foo/binding, which has resource
> \"pods\",*
>
> *// subresource \"binding\", and kind \"Binding\".*
>
> GetSubresource() **string**
>
> *// GetOperation is the operation being performed.*
>
> GetOperation() Operation
>
> *// IsDryRun indicates that modifications will definitely not be
> persisted for*
>
> *// this request. This is to prevent admission controllers with side
> effects*
>
> *// and a method of reconciliation from being overwhelmed.*
>
> *// However, a value of false for this does not mean that the
> modification will*
>
> *// be persisted, because it could still be rejected by a subsequent*
>
> *// validation step.*
>
> IsDryRun() **bool**
>
> *// GetObject is the object from the incoming request prior to default
> values*
>
> *// being applied.*
>
> GetObject() runtime.Object
>
> *// GetOldObject is the existing object. Only populated for UPDATE
> requests.*
>
> GetOldObject() runtime.Object
>
> *// GetKind is the type of object being manipulated. For example:
> Pod.*
>
> GetKind() schema.GroupVersionKind
>
> *// GetUserInfo is information about the requesting user.*
>
> GetUserInfo() user.Info
>
> *// AddAnnotation sets annotation according to key-value pair. The
> key*
>
> *// should be qualified, e.g.,
> podsecuritypolicy.admission.k8s.io/admit-policy,*
>
> *// where \"podsecuritypolicy\" is the name of the plugin,
> \"admission.k8s.io\"*
>
> *// is the name of the organization, and \"admit-policy\" is the key*
>
> *// name. An error is returned if the format of key is invalid. When*
>
> *// trying to overwrite annotation with a new value, an error is*
>
> *// returned. Both ValidationInterface and MutationInterface are*
>
> *// allowed to add Annotations.*
>
> AddAnnotation(key, value **string**) **error**
>
> }

在mutating情况下，也就是说，在允许（a
Attributes）错误方法的实现中，可以对属性进行变异，或者更准确地说，可以对从GetObject（）runtime.object返回的对象进行变异。

在验证情况下，不允许变异。

这两种情况都允许调用AddAnnotation（key，value
string）错误，这允许我们添加最终出现在API服务器的审计输出中的注释。这有助于理解为什么许可插件会发生变异或拒绝请求。

拒绝通过从允许或验证返回非零错误来发出信号。

###### TIP

对许可插件进行mutating也是一种很好的实践，可以在验证许可阶段验证更改。原因是其他插件，包括webhook允许插件，可能会添加进一步的更改。如果允许插件保证满足某些不变量，则只有验证步骤才能确保确实如此。

许可插件必须从acception.Interface接口实现Handles（操作操作）bool方法。同一个包中有一个名为Handler的助手。它可以使用NewHandler（ops...Operation）\*Handler进行实例化，并通过将Handles方法嵌入到自定义许可插件中来实现：

> **type** CustomAdmissionPlugin **struct** {
>
> \*admission.Handler
>
> \...
>
> }

允许插件应始终首先检查传递对象的groupversion类型：

> **func** (d \*PizzaToppingsPlugin) Admit(
>
> a admission.Attributes,
>
> o ObjectInterfaces,
>
> ) **error** {
>
> *// we are only interested in pizzas*
>
> **if** a.GetKind().GroupKind() != restaurant.Kind(\"Pizza\") {
>
> **return** **nil**
>
> }
>
> \...
>
> }

同样，对于validating案例：

> **func** (d \*PizzaToppingsPlugin) Validate(
>
> a admission.Attributes,
>
> o ObjectInterfaces,
>
> ) **error** {
>
> *// we are only interested in pizzas*
>
> **if** a.GetKind().GroupKind() != restaurant.Kind(\"Pizza\") {
>
> **return** **nil**
>
> }
>
> \...
>
> }

##### 为什么Api Server管道不预先过滤对象

对于本机许可插件，没有注册机制使API
Server机器可以使用受支持对象的信息，以便仅为其支持的对象调用插件。一个原因是Kubernetes
API服务器中的许多插件（在那里发明了许可机制）支持大量对象。

完整的允许实现示例如下：

> *// Admit ensures that the object in-flight is of kind Pizza.*
>
> *// In addition checks that the toppings are known.*
>
> **func** (d \*PizzaToppingsPlugin) Validate(
>
> a admission.Attributes,
>
> \_ admission.ObjectInterfaces,
>
> ) **error** {
>
> *// we are only interested in pizzas*
>
> **if** a.GetKind().GroupKind() != restaurant.Kind(\"Pizza\") {
>
> **return** **nil**
>
> }
>
> **if** !d.WaitForReady() {
>
> **return** admission.NewForbidden(a, fmt.Errorf(\"not yet ready\"))
>
> }
>
> obj := a.GetObject()
>
> pizza := obj.(\*restaurant.Pizza)
>
> **for** \_, top := **range** pizza.Spec.Toppings {
>
> err := \_, err := d.toppingLister.Get(top.Name)
>
> **if** err != **nil** && errors.IsNotFound(err) {
>
> **return** admission.NewForbidden(
>
> a,
>
> fmt.Errorf(\"unknown topping: %s\", top.Name),
>
> )
>
> }
>
> }
>
> **return** **nil**
>
> }

它采取以下步骤：

1.检查传递的对象是否是正确的类型

2.在informers准备好之前禁止进入

3.通过toppings informer
lister验证pizza规范中提到的每个topping实际上作为topping对象存在于集群中

注意这里的lister只是内存存储中的informer的接口，因此 Get calls会很快。

### REGISTERING

Admission插件必须注册。这是通过注册功能完成的：

> **func** Register(plugins \*admission.Plugins) {
>
> plugins.Register(
>
> \"PizzaTopping\",
>
> **func**(config io.Reader) (admission.Interface, **error**) {
>
> **return** New()
>
> },
>
> )
>
> }

此功能已添加到\`\`推荐的选项\'\'中的插件列表中（请参阅\`\` Options and
Config Pattern and Startup Plumbing\'\'）：

> **func** (o \*CustomServerOptions) Complete() **error** {
>
> *// register admission plugins*
>
> pizzatoppings.Register(o.RecommendedOptions.Admission.Plugins)
>
> *// add admisison plugins to the RecommendedPluginOrder*
>
> oldOrder := o.RecommendedOptions.Admission.RecommendedPluginOrder
>
> o.RecommendedOptions.Admission.RecommendedPluginOrder =
>
> append(oldOrder, \"PizzaToppings\")
>
> **return** **nil**
>
> }

在这里，\`\`RecommendedPluginOrder\'\'列表中预填充了通用准入插件，每个API服务器都应保持启用状态，以使其成为集群中良好的API约定公民。

最好的做法是不要碰订单。 原因之一是正确执行订单绝非易事。
当然，如果严格要求插件行为的话，可以在列表末尾以外的位置添加定制插件。

自定义API服务器的用户将能够使用通常的准入链配置标志（例如\--disable-admission-plugins）禁用自定义准入插件。
默认情况下，我们自己的插件处于启用状态，因为我们没有明确禁用它。

可以使用配置文件配置准入插件。
为此，我们在前面显示的Register函数中解析io.Reader的输出。
\--admission-control-config-file允许我们将配置文件传递给插件，如下所示：

> **kind**: AdmissionConfiguration
>
> **apiVersion**: apiserver.k8s.io/v1alpha1
>
> **plugins**:
>
> \- **name**: CustomAdmissionPlugin
>
> **path**: custom-admission-plugin.yaml

另外，我们可以进行内联配置，以将所有准入配置放在一个位置：

> **kind**: AdmissionConfiguration
>
> **apiVersion**: apiserver.k8s.io/v1alpha1
>
> **plugins**:
>
> \- **name**: CustomAdmissionPlugin
>
> **configuration**:
>
> *your-custom-yaml-inline-config*

我们简要地提到了我们的admission插件使用toppings通知程序来检查披萨中是否提到了toppings。
我们还没有讨论如何将其连接到admission插件。 现在开始吧

### PLUMBING RESOURCES

admission插件通常需要客户端和通知者或其他资源来实现其行为。
我们可以使用插件初始化程序来完成此资源探测。

有许多标准的插件初始化程序。
如果您的插件要被它们调用，则必须使用回调方法实现某些接口（有关更多信息，请参见k8s.io/apiserver/pkg/admission/initializer）：

> *// WantsExternalKubeClientSet defines a function that sets external
> ClientSet*
>
> *// for admission plugins that need it.*
>
> **type** WantsExternalKubeClientSet **interface** {
>
> SetExternalKubeClientSet(kubernetes.Interface)
>
> admission.InitializationValidator
>
> }
>
> *// WantsExternalKubeInformerFactory defines a function that sets
> InformerFactory*
>
> *// for admission plugins that need it.*
>
> **type** WantsExternalKubeInformerFactory **interface** {
>
> SetExternalKubeInformerFactory(informers.SharedInformerFactory)
>
> admission.InitializationValidator
>
> }
>
> *// WantsAuthorizer defines a function that sets Authorizer for
> admission*
>
> *// plugins that need it.*
>
> **type** WantsAuthorizer **interface** {
>
> SetAuthorizer(authorizer.Authorizer)
>
> admission.InitializationValidator
>
> }
>
> *// WantsScheme defines a function that accepts runtime.Scheme for
> admission*
>
> *// plugins that need it.*
>
> **type** WantsScheme **interface** {
>
> SetScheme(\*runtime.Scheme)
>
> admission.InitializationValidator
>
> }

实现其中一些功能，然后在启动过程中调用该插件，以访问Kubernetes资源或API服务器全局方案。

此外，应实现\`\`mission.InitializationValidator\'\'接口来进行最终检查以确保正确设置了插件：

> *// InitializationValidator holds ValidateInitialization functions,
> which are*
>
> *// responsible for validation of initialized shared resources and
> should be*
>
> *// implemented on admission plugins.*
>
> **type** InitializationValidator **interface** {
>
> ValidateInitialization() **error**
>
> }

标准的初始化程序很棒，但是我们需要访问toppings通知程序。因此，让我们看一下如何添加我们自己的初始化程序。
初始化程序包括：

•Wants
\*接口（例如，WantsRestaurantInformerFactory），该接口应由准入插件实现：

-   *// WantsRestaurantInformerFactory defines a function that sets*

-   *// InformerFactory for admission plugins that need it.*

-   **type** WantsRestaurantInformerFactory **interface** {

-   SetRestaurantInformerFactory(informers.SharedInformerFactory)

-   admission.InitializationValidator

> }

•初始化程序结构，实现admission.PluginInitializer：

-   **func** (i restaurantInformerPluginInitializer) Initialize(

-   plugin admission.Interface,

-   ) {

-   **if** wants, ok := plugin.(WantsRestaurantInformerFactory); ok {

-   wants.SetRestaurantInformerFactory(i.informers)

-   }

}

换句话说，Initialize（）方法检查所传递的插件是否实现了相应的自定义初始化程序Wants
\*接口。 如果是这种情况，则初始化程序将在插件上调用该方法。

•将初始化构造函数的管道插入到RecommendedOptions.Extra \\
AdmissionInitializers中（请参见"Options and Config Pattern and Startup
Plumbing"）：

-   **func** (o \*CustomServerOptions) Config() (\*apiserver.Config,
    > **error**) {

-   \...

-   o.RecommendedOptions.ExtraAdmissionInitializers =

-   **func**(c \*genericapiserver.RecommendedConfig) (

-   \[\]admission.PluginInitializer, **error**,

-   ) {

-   client, err := clientset.NewForConfig(c.LoopbackClientConfig)

-   **if** err != **nil** {

-   **return** **nil**, err

-   }

-   informerFactory := informers.NewSharedInformerFactory(

-   client, c.LoopbackClientConfig.Timeout,

-   )

-   o.SharedInformerFactory = informerFactory

-   **return** \[\]admission.PluginInitializer{

-   custominitializer.New(informerFactory),

-   }, **nil**

-   }

-   

-   \...

> }

这段代码为餐厅API组创建了一个回送客户端，创建了一个相应的notifyer工厂，将其存储在选项o中，并为其返回了一个插件初始化器。

##### SYNCING INFORMERS

如果准入插件中使用了informers，请先检查informers是否已同步，然后再在实际的Admit（）或Validate（）函数中使用informers。在这种情况下，请拒绝带有\`\`禁止\'\'错误的请求。

使用"Implementation"中所述的Handler帮助程序结构，我们可以使用Handler.WaitForReady（）函数轻松地做到这一点：

> **if** !d.WaitForReady() {
>
> **return** admission.NewForbidden(
>
> a, fmt.Errorf(\"not yet ready to handle request\"),
>
> )
>
> }

要在此WaitForReady（）方法中包括自定义informers的HasSynced（）方法，请将其添加到初始化程序实现的ready函数中，如下所示：

> **func** (d \*PizzaToppingsPlugin) SetRestaurantInformerFactory(
>
> f informers.SharedInformerFactory) {
>
> d.toppingLister = f.Restaurant().V1Alpha1().Toppings().Lister()
>
> d.SetReadyFunc(f.Restaurant().V1Alpha1().Toppings().Informer().HasSynced)
>
> }

如所承诺的那样，admission是为餐厅API组完成我们的自定义API
Server的实现的最后一步。
现在我们希望看到它的作用，而不是在本地机器上人为地看到，而是在真实的Kubernetes集群中。
这意味着我们必须看一看聚合定制API Server的部署。

部署自定义API Servers
=====================

在"API Service"中，我们看到了APIService对象，它用于向Kubernetes
API服务器内的聚合器注册自定义API Server API组版本：

> **apiVersion**: apiregistration.k8s.io/v1beta1
>
> **kind**: APIService
>
> **metadata**:
>
> **name**: *name*
>
> **spec**:
>
> **group**: *API-group-name*
>
> **version**: *API-group-version*
>
> **service**:
>
> **namespace**: *custom-API-server-service-namespace*
>
> **name**: *custom-API-server-service*
>
> **caBundle**: *base64-caBundle*
>
> **insecureSkipTLSVerify**: *bool*
>
> **groupPriorityMinimum**: 2000
>
> **versionPriority**: 20

APIService对象指向 Service。通常，这个service将是一个普通的集群IP
Service：也就是说，自定义API
Server是使用pods部署到集群中的。服务将请求转发到pods。

让我们看看Kubernetes清单来实现这一点。

Deployment Manifests
--------------------

我们有以下清单（在GitHub上的示例代码中找到），这些清单将是自定义APIservice的集群内部署的一部分：

-   An APIService for both versions v1alpha1:

-   **apiVersion**: apiregistration.k8s.io/v1beta1

-   **kind**: APIService

-   **metadata**:

-   **name**: v1alpha1.restaurant.programming-kubernetes.info

-   **spec**:

-   **insecureSkipTLSVerify**: true

-   **group**: restaurant.programming-kubernetes.info

-   **groupPriorityMinimum**: 1000

-   **versionPriority**: 15

-   **service**:

-   **name**: api

-   **namespace**: pizza-apiserver

> **version**: v1alpha1
>
> ...and v1beta1:
>
> **apiVersion**: apiregistration.k8s.io/v1beta1
>
> **kind**: APIService
>
> **metadata**:
>
> **name**: v1alpha1.restaurant.programming-kubernetes.info
>
> **spec**:
>
> **insecureSkipTLSVerify**: true
>
> **group**: restaurant.programming-kubernetes.info
>
> **groupPriorityMinimum**: 1000
>
> **versionPriority**: 15
>
> **service**:
>
> **name**: api
>
> **namespace**: pizza-apiserver
>
> **version**: v1alpha1
>
> 注意这里我们设置了insecureskiplsverify。这对于开发来说是可以的，但对于任何生产部署来说都是不够的。我们将在"Certificates
> and Trust"中看到如何解决这个问题。

-   在群集中运行的自定义API Server实例前面的服务

-   **apiVersion**: v1

-   **kind**: Service

-   **metadata**:

-   **name**: api

-   **namespace**: pizza-apiserver

-   **spec**:

-   **ports**:

-   \- **port**: 443

-   **protocol**: TCP

-   **targetPort**: 8443

-   **selector**:

> **apiserver**: \"true\"

-   A Deployment (as shown here) or DaemonSet for the custom API server
    pods:

-   **apiVersion**: apps/v1

-   **kind**: Deployment

-   **metadata**:

-   **name**: pizza-apiserver

-   **namespace**: pizza-apiserver

-   **labels**:

-   **apiserver**: \"true\"

-   **spec**:

-   **replicas**: 1

-   **selector**:

-   **matchLabels**:

-   **apiserver**: \"true\"

-   **template**:

-   **metadata**:

-   **labels**:

-   **apiserver**: \"true\"

-   **spec**:

-   **serviceAccountName**: apiserver

-   **containers**:

-   \- **name**: apiserver

-   **image**: quay.io/programming-kubernetes/pizza-apiserver:latest

-   **imagePullPolicy**: Always

-   **command**: \[\"/pizza-apiserver\"\]

-   **args**:

-   \- \--etcd-servers=http://localhost:2379

-   \- \--cert-dir=/tmp/certs

-   \- \--secure-port=8443

-   \- \--v=4

-   \- **name**: etcd

-   **image**: quay.io/coreos/etcd:v3.2.24

> **workingDir**: /tmp

-   A namespace for the service and the deployment to live in:

-   **apiVersion**: v1

-   **kind**: Namespace

-   **metadata**:

-   **name**: pizza-apiserver

> **spec**: {}

通常，聚合（aggregated）的API服务器被部署到为控制平面pod（通常称为master）保留的一些节点上。在这种情况下，守护程序是每个主节点运行一个自定义API
Server实例的好选择。这将导致高可用性设置。注意，API
Server是无状态的，这意味着它们可以很容易地被多次部署，而且不需要leader选举。

有了这些清单，我们就快完成了。不过，与通常情况一样，安全部署需要更多考虑。您可能已经注意到pod（通过前面的部署定义）使用一个自定义service
account apiserver。这可以通过另一个清单创建：

> **kind**: ServiceAccount
>
> **apiVersion**: v1
>
> **metadata**:
>
> **name**: apiserver
>
> **namespace**: pizza-apiserver

此服务帐户需要多个权限，我们可以通过RBAC对象添加这些权限。

Setting Up RBAC
---------------

API服务的服务帐户首先需要一些通用权限才能参与：

namespace lifecycle

对象只能在现有namespaces中创建，并且在删除namespaces时被删除。为此，API
Server必须获取、列出和watch namespaces。

admission webhooks

通过MutatingWebhookConfigurations和ValidatedWebhookConfigurations配置的许可webhook是从每个API
Server独立调用的。为此，自定义API
Server中的许可机制必须获取、列出和watch这些资源。

我们通过创建RBAC集群角色来配置两者：

> **kind**: ClusterRole
>
> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **metadata**:
>
> **name**: aggregated-apiserver-clusterrole
>
> **rules**:
>
> \- **apiGroups**: \[\"\"\]
>
> **resources**: \[\"namespaces\"\]
>
> **verbs**: \[\"get\", \"watch\", \"list\"\]
>
> \- **apiGroups**: \[\"admissionregistration.k8s.io\"\]
>
> **resources**: \[\"mutatingwebhookconfigurations\",
> \"validatingwebhookconfigurations\"\]
>
> **verbs**: \[\"get\", \"watch\", \"list\"\]

并通过ClusterRoleBinding将其绑定到我们的service account  apiserver：

> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **kind**: ClusterRoleBinding
>
> **metadata**:
>
> **name**: pizza-apiserver-clusterrolebinding
>
> **roleRef**:
>
> **apiGroup**: rbac.authorization.k8s.io
>
> **kind**: ClusterRole
>
> **name**: aggregated-apiserver-clusterrole
>
> **subjects**:
>
> \- **kind**: ServiceAccount
>
> **name**: apiserver
>
> **namespace**: pizza-apiserver

对于委派的身份验证和授权，service
account必须绑定到先前存在的RBAC角色扩展apiserver身份验证读取器：

> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **kind**: RoleBinding
>
> **metadata**:
>
> **name**: pizza-apiserver-auth-reader
>
> **namespace**: kube-system
>
> **roleRef**:
>
> **apiGroup**: rbac.authorization.k8s.io
>
> **kind**: Role
>
> **name**: extension-apiserver-authentication-reader
>
> **subjects**:
>
> \- **kind**: ServiceAccount
>
> **name**: apiserver
>
> **namespace**: pizza-apiserver

以及现有的RBAC集群角色系统：auth delegator:

> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **kind**: ClusterRoleBinding
>
> **metadata**:
>
> **name**: pizza-apiserver:system:auth-delegator
>
> **roleRef**:
>
> **apiGroup**: rbac.authorization.k8s.io
>
> **kind**: ClusterRole
>
> **name**: system:auth-delegator
>
> **subjects**:
>
> \- **kind**: ServiceAccount
>
> **name**: apiserver
>
> **namespace**: pizza-apiserver

Running the Custom API Server Insecurely（不安全地运行自定义API Server）
------------------------------------------------------------------------

现在，所有清单都就位并设置了RBAC，让我们将API
Server部署到一个真正的集群。

从GitHub存储库的checkout，并使用已配置的具有群集管理权限的kubectl（这是必需的，因为RBAC规则永远无法提升访问权限）：

> \$ cd \$GOPATH/src/github.com/programming-kubernetes/pizza-apiserver
>
> \$ cd artifacts/deployment
>
> \$ kubectl apply -f ns.yaml *\# create the namespace first*
>
> \$ kubectl apply -f . *\# creating all manifests described above*

现在，自定义API Server正在启动：

> \$ kubectl get pods -A
>
> NAMESPACE NAME READY STATUS AGE
>
> pizza-apiserver pizza-apiserver-7779f8d486-8fpgj 0/2 ContainerCreating
> 1s
>
> \$ *\# some moments later*
>
> \$ kubectl get pods -A
>
> pizza-apiserver pizza-apiserver-7779f8d486-8fpgj 2/2 Running 75s

当它运行时，我们再次检查Kubernetes API
Server是否进行aggregation（即请求代理）。首先通过APIServices检查Kubernetes
API Server是否认为我们的自定义API Server可用：

> \$ kubectl get apiservices
> v1alpha1.restaurant.programming-kubernetes.info
>
> NAME SERVICE AVAILABLE
>
> v1alpha1.restaurant.programming-kubernetes.info pizza-apiserver/api
> True

这看起来不错。让我们尝试列出比萨饼，并启用日志记录以查看是否出了问题：

> \$ kubectl get pizzas \--v=7
>
> \...
>
> \... GET https://localhost:58727/apis?timeout=32s
>
> \...
>
> \... GET
> https://localhost:58727/apis/restaurant.programming-kubernetes.info/
>
> v1alpha1?timeout=32s
>
> \...
>
> \... GET
> https://localhost:58727/apis/restaurant.programming-kubernetes.info/
>
> v1beta1/namespaces/default/pizzas?limit=500
>
> \... Request Headers:
>
> \... Accept: application/json;as=Table;v=v1beta1;g=meta.k8s.io,
> application/json
>
> \... User-Agent: kubectl/v1.15.0 (darwin/amd64) kubernetes/f873d2a
>
> \... Response Status: 200 OK in 6 milliseconds
>
> No resources found.

这看起来很不错。我们看到kubectl查询发现信息以了解比萨饼是什么。它查询餐厅，编程kubernetes.info/v1beta1
API来列出比萨饼。不出所料，现在还没有。但我们当然可以改变：

> \$ cd ../examples
>
> \$ *\# install toppings first*
>
> \$ ls topping\* \| xargs -n 1 kubectl create -f
>
> \$ kubectl create -f pizza-margherita.yaml
>
> pizza.restaurant.programming-kubernetes.info/margherita created
>
> \$ kubectl get pizza -o yaml margherita
>
> apiVersion: restaurant.programming-kubernetes.info/v1beta1
>
> kind: Pizza
>
> metadata:
>
> creationTimestamp: \"2019-05-05T13:39:52Z\"
>
> name: margherita
>
> namespace: default
>
> resourceVersion: \"6\"
>
> pizzas/margherita
>
> uid: 42ab6e88-6f3b-11e9-8270-0e37170891d3
>
> spec:
>
> toppings:
>
> \- name: mozzarella
>
> quantity: 1
>
> \- name: tomato
>
> quantity: 1
>
> status: {}

这看起来棒极了。但是玛格丽塔比萨饼很简单。让我们尝试通过创建一个不列出任何配料的空比萨饼来：

> apiVersion: restaurant.programming-kubernetes.info/v1alpha1
>
> kind: Pizza
>
> metadata:
>
> name: salami
>
> spec:

我们的defaulting应该会把它变成一个上面有意大利腊肠的意大利披萨。让我们试试：

> \$ kubectl create -f empty-pizza.yaml
>
> pizza.restaurant.programming-kubernetes.info/salami created
>
> \$ kubectl get pizza -o yaml salami
>
> apiVersion: restaurant.programming-kubernetes.info/v1beta1
>
> kind: Pizza
>
> metadata:
>
> creationTimestamp: \"2019-05-05T13:42:42Z\"
>
> name: salami
>
> namespace: default
>
> resourceVersion: \"8\"
>
> pizzas/salami
>
> uid: a7cb7af2-6f3b-11e9-8270-0e37170891d3
>
> spec:
>
> toppings:
>
> \- name: salami
>
> quantity: 1
>
> \- name: mozzarella
>
> quantity: 1
>
> \- name: tomato
>
> quantity: 1
>
> status: {}

这看起来像是美味的萨拉米披萨。

现在让我们检查一下我们的自定义admission插件是否工作正常。我们首先删除所有比萨饼和配料，然后尝试重新创建比萨饼：

> \$ kubectl delete pizzas \--all
>
> pizza.restaurant.programming-kubernetes.info \"margherita\" deleted
>
> pizza.restaurant.programming-kubernetes.info \"salami\" deleted
>
> \$ kubectl delete toppings \--all
>
> topping.restaurant.programming-kubernetes.info \"mozzarella\" deleted
>
> topping.restaurant.programming-kubernetes.info \"salami\" deleted
>
> topping.restaurant.programming-kubernetes.info \"tomato\" deleted
>
> \$ kubectl create -f pizza-margherita.yaml
>
> Error from server (Forbidden): error when creating
> \"pizza-margherita.yaml\":
>
> pizzas.restaurant.programming-kubernetes.info \"margherita\" is
> forbidden:
>
> unknown topping: mozzarella

没有马苏里拉就没有玛格丽特，就像在任何一家好的意大利餐馆一样。

看起来我们已经完成了我们在"示例：比萨餐厅"中所描述的。但不完全是。安全。再一次。我们没有处理好适当的证书。恶意的比萨饼销售商可能会试图在我们的用户和自定义API
Server之间进行交易，因为Kubernetes API
Server只接受任何服务证书而不检查它们。我们来解决这个问题。

Certificates and Trust（证书和信任）
------------------------------------

APIService对象包含caBundle字段。这将配置aggregator（在Kubernetes
API服务器内）如何信任自定义API Server。此CA捆绑包包含用于验证aggregator
APIServer是否具有其声称拥有的标识的证书（和中间证书）。对于任何严重的部署，请将相应的CA包放入此字段。

###### WARNING

虽然APIService中允许使用insecureskiptsverify以禁用证书验证，但在生产设置中使用此选项是一个坏主意。Kubernetes
API Server向受信任的aggregator API
Server发送请求。将insecureskiptsverify设置为true意味着任何其他参与者都可以声明自己是aggregator的API
Server。这显然是不安全的，不应在生产环境中使用。

从自定义API Server到Kubernetes API
Server的反向信任及其对请求的预身份验证在"Delegated Authentication and
Trust"中进行了描述。我们不需要做任何额外的事情。

回到pizza示例：为了确保安全，我们需要一个服务证书和部署中自定义API
Server的密钥。我们将两者都放入一个服务证书机密中，并将其装载到位于/var/run/apiserver/serving
cert/tls.{crt，key}的pod中。然后我们在APIService中将tls.crt文件用作CA。这些都可以在GitHub的示例代码中找到。

证书生成逻辑是在Makefile中编写的。

请注意，在实际的场景中，我们可能会有某种集群或公司CA，我们可以插入到APIService中。

要查看它的运行情况，可以从一个新的集群开始，也可以重用前一个集群并应用新的安全清单：

> \$ cd ../deployment-secure
>
> \$ make
>
> openssl req -new -x509 -subj \"/CN=api.pizza-apiserver.svc\"
>
> -nodes -newkey rsa:4096
>
> -keyout tls.key -out tls.crt -days 365
>
> Generating a 4096 bit RSA private key
>
> \...\...\...\...\...\...\....++
>
> \...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\...\....++
>
> writing new private key to \'tls.key\'
>
> \...
>
> \$ ls \*.yaml \| xargs -n 1 kubectl apply -f
>
> clusterrolebinding.rbac.authorization.k8s.io/pizza-apiserver:system:auth-delegator
> unchanged
>
> rolebinding.rbac.authorization.k8s.io/pizza-apiserver-auth-reader
> unchanged
>
> deployment.apps/pizza-apiserver configured
>
> namespace/pizza-apiserver unchanged
>
> clusterrolebinding.rbac.authorization.k8s.io/pizza-apiserver-clusterrolebinding
> unchanged
>
> clusterrole.rbac.authorization.k8s.io/aggregated-apiserver-clusterrole
> unchanged
>
> serviceaccount/apiserver unchanged
>
> service/api unchanged
>
> secret/serving-cert created
>
> apiservice.apiregistration.k8s.io/v1alpha1.restaurant.programming-kubernetes.info
> configured
>
> apiservice.apiregistration.k8s.io/v1beta1.restaurant.programming-kubernetes.info
> configured

请注意证书中正确的公用名CN=api.pizza apiserver.svc。Kubernetes
API服务器将请求代理到API/pizza API
server服务，因此它的DNS名称必须放入证书中。

我们再次检查是否确实禁用了APIService中的不安全skiplsverify标志：

> \$ kubectl get apiservices
> v1alpha1.restaurant.programming-kubernetes.info -o yaml
>
> apiVersion: apiregistration.k8s.io/v1
>
> kind: APIService
>
> metadata:
>
> name: v1alpha1.restaurant.programming-kubernetes.info
>
> \...
>
> spec:
>
> caBundle: LS0tLS1C\...
>
> group: restaurant.programming-kubernetes.info
>
> groupPriorityMinimum: 1000
>
> service:
>
> name: api
>
> namespace: pizza-apiserver
>
> version: v1alpha1
>
> versionPriority: 15
>
> status:
>
> conditions:
>
> \- lastTransitionTime: \"2019-05-05T14:07:07Z\"
>
> message: all checks passed
>
> reason: Passed
>
> status: \"True\"
>
> type: Available
>
> artifacts/deploymen

这看起来和预期的一样：insecureskipltsverify已消失，caBundle字段中填充了证书的base64值，并且：该服务仍然可用。

现在让我们看看kubectl是否仍然可以查询API：

> \$ kubectl get pizzas
>
> No resources found.
>
> \$ cd ../examples
>
> \$ ls topping\* \| xargs -n 1 kubectl create -f
>
> topping.restaurant.programming-kubernetes.info/mozzarella created
>
> topping.restaurant.programming-kubernetes.info/salami created
>
> topping.restaurant.programming-kubernetes.info/tomato created
>
> \$ kubectl create -f pizza-margherita.yaml
>
> pizza.restaurant.programming-kubernetes.info/margherita created

玛格丽塔披萨回来了。这次是完全安全的。恶毒的披萨销售商没有机会发动中间人攻击。

Sharing etcd（共享ETCD）
------------------------

使用推荐选项（请参阅"Options and Config Pattern and Startup
Plumbing"）的聚合API Server使用etcd进行存储。这意味着任何自定义API
Server的部署都需要etcd集群可用。

这个集群可以在集群中，例如，使用etcd操作符部署。这个操作符允许我们以声明的方式启动和管理etcd集群。操作员将进行更新、上下缩放和备份。这大大减少了操作开销。

或者可以使用集群控制平面的etcd（即kube-apiserver的etcd）。取决于自行部署的环境、内部部署的环境或像Google
Container
Engine（GKE）这样的托管服务，这可能是可行的，也可能是不可能的，因为用户根本无法访问集群（GKE就是这样）。在可行的情况下，自定义API
Server必须使用与Kubernetes
API服务器或其他etcd用户使用的密钥路径不同的密钥路径。在我们的示例自定义API服务器中，如下所示：

> **const** defaultEtcdPathPrefix =
>
> \"/registry/pizza-apiserver.programming-kubernetes.github.com\"
>
> **func** NewCustomServerOptions() \*CustomServerOptions {
>
> o := &CustomServerOptions{
>
> RecommendedOptions: genericoptions.NewRecommendedOptions(
>
> defaultEtcdPathPrefix,
>
> \...
>
> ),
>
> }
>
> **return** o
>
> }

这个etcd路径前缀不同于Kubernetes API
Server路径，后者使用不同的组API名称。

最后但并非最不重要的是，etcd可以被代理。项目etcdproxy控制器使用operator模式实现此机制；也就是说，etcd代理可以自动部署到集群，并使用etcdproxy对象进行配置。

etcd代理将自动进行密钥映射，因此保证etcd密钥前缀不会冲突。这允许我们为多个聚合API服务器共享etcd集群，而不必担心一个聚合API服务器读取或更改另一个aggregated
API
Server的数据。这将在需要共享etcd集群的环境中提高安全性，例如，由于资源限制或避免操作开销。

根据上下文，必须选择其中一个选项。最后，aggregated API
Server当然也可以使用其他存储后端，至少在理论上是这样，因为实现k8s.io/apiserver存储接口需要很多自定义代码。

Summary
=======

这是一个相当大的章节，你做到了最后。关于Kubernetes中的api以及它们是如何实现的，您已经了解了很多背景知识。

我们看到了如何将自定义aggregated API
Server到Kubernetes集群的体系结构中。我们看到了自定义API
Server如何接收来自kubernetesapi
Server的代理请求。我们已经看到了Kubernetes API
Server是如何预先验证这些请求的，以及API组是如何使用外部版本和内部版本实现的。我们学习了如何将对象解码为Golang结构，如何将其默认设置，如何将其转换为内部类型，以及如何通过许可和验证最终到达注册表。我们看到了如何将策略插入到通用注册表中以实现"normal"的Kubernetes（如REST资源），如何添加自定义许可，以及如何使用自定义初始值设定项配置自定义许可插件。我们现在知道如何使用多版本API组启动自定义API
Server，以及如何使用APIServices在集群中部署API组。我们看到了如何配置RBAC规则以允许自定义API服务器执行其任务。我们讨论了kubectl如何查询API组。最后，我们学习了如何使用证书保护到自定义API
Server的连接。

这太多了。现在，您对Kubernetes中的api以及它们是如何实现的有了更好的理解，希望您有动机执行以下一项或多项操作：

•实现自己的自定义API Server

•了解Kubernetes的内部工作原理

•为将来的库伯内特斯做出贡献

我们希望你已经找到了一个好的起点。

1正常删除意味着客户端可以作为删除调用的一部分通过正常删除周期。实际的删除是由一个controller通过强制删除异步完成的（kubelet对pods执行此操作）。这样pod就有时间彻底关闭了。

2 Kubernetes使用cohabitation来将资源（例如，从扩展/v1beta1
API组的部署）迁移到特定于主题的API组（例如，apps/v1）。crd没有共享存储的概念。

3我们将在第9章中看到，最新Kubernetes版本中提供的CRD转换和许可webhook也允许我们将这些特性添加到CRD中。

4 PaaS代表平台即服务
