第七章. Shipping Controllers 和Operators

现在您已经熟悉了custom
controllers,的开发，接下来让我们继续讨论如何使custom
controllers和operators投入生产的主题。在本章中，我们将讨论controllers和operators的操作方面，向您展示如何打包它们，引导您完成在生产环境中运行controllers的最佳实践，并确保扩展点不会破坏Kubernetes集群，安全性，或性能方面。

生命周期管理和 Packaging
========================

在本节中，我们考虑运营商的生命周期管理。
也就是说，我们将讨论如何包装和运送您的控制器或操作员，以及如何处理升级。
准备好将操作员运送给用户时，需要一种让他们安装的方法。
为此，您需要打包相应的工件，例如定义控制器二进制文件的YAML清单（通常是Kubernetes部署），以及CRD和与安全相关的资源（例如服务帐户和必要的RBAC权限）。
一旦目标用户运行了特定版本的操作员，您还需要一种机制来升级控制器，同时考虑版本控制和潜在的零停机升级。

让我们从low-hanging开始：packaging和交付controllers，以便用户可以直接安装它。

Packaging: The Challenge
------------------------

虽然Kubernetes使用清单来定义资源，清单通常用YAML编写，这是一个用于声明资源状态的低级接口，但这些清单文件有缺点。最重要的是，在打包容器化应用程序的上下文中，YAML清单是静态的；
也就是说，YAML清单中的所有值都是固定的。这意味着，例如，如果您想更改部署清单中的容器镜像，则必须创建一个新清单。

让我们看一个具体的例子。
假设您在名为mycontroller.yaml的YAML清单中编码了以下Kubernetes部署，代表您希望用户安装的自定义控制器：

> **apiVersion**: apps/v1beta1
>
> **kind**: Deployment
>
> **metadata**:
>
> **name**: mycustomcontroller
>
> **spec**:
>
> **replicas**: 1
>
> **template**:
>
> **metadata**:
>
> **labels**:
>
> **app**: customcontroller
>
> **spec**:
>
> **containers**:
>
> \- **name**: thecontroller
>
> **image**: example/controller:0.1.0
>
> **ports**:
>
> \- **containerPort**: 9999
>
> **env**:
>
> \- **name**: REGION
>
> **value**: eu-west-1

想象一下，环境变量REGION定义了控制器的某些运行时属性，例如托管服务网格等其他服务的可用性。换句话说，虽然eu-west-1的默认值可能是一个明智的选择，但用户可以并且很可能会根据自己的偏好或策略覆盖它。

现在，假设清单mycontroller.yaml本身是一个静态文件，并且在编写本文时已定义了所有值-而且诸如Kubectl之类的客户端本身并不支持清单中的可变部分-如何使用户能够提供可变值还是在运行时覆盖现有值？也就是说，在前面的示例中，用户如何在安装时将区域设置为例如us-east-2，例如使用Kubectl？

为了克服Kubernetes中构建时静态YAML清单的这些限制，根据用户提供的值或运行时属性，有一些选项可以对清单进行模板化（例如，Helm）或启用变量输入（Kustomize）。

Helm
----

Helm包为Kubernetes的软件包管理器，最初是由Deis开发的，现在是一个Cloud
Native Computing
Foundation（CNCF）项目，由Microsoft，Google和Bitnami（现已成为VMware的一部分）的主要贡献者。

Helm通过定义和应用所谓的图表，有效参数化的YAML清单来帮助您安装和升级Kubernetes应用程序。
这是图表模板示例的摘录：

> **apiVersion**: apps/v1
>
> **kind**: Deployment
>
> **metadata**:
>
> **name**: {{ include \"flagger.fullname\" . }}
>
> **\...**
>
> **spec**:
>
> **replicas**: 1
>
> **strategy**:
>
> **type**: Recreate
>
> **selector**:
>
> **matchLabels**:
>
> **app.kubernetes.io/name**: {{ template \"flagger.name\" . }}
>
> **app.kubernetes.io/instance**: {{ .Release.Name }}
>
> **template**:
>
> **metadata**:
>
> **labels**:
>
> **app.kubernetes.io/name**: {{ template \"flagger.name\" . }}
>
> **app.kubernetes.io/instance**: {{ .Release.Name }}
>
> **spec**:
>
> **serviceAccountName**: {{ template \"flagger.serviceAccountName\" .
> }}
>
> **containers**:
>
> \- **name**: flagger
>
> **securityContext**:
>
> **readOnlyRootFilesystem**: true
>
> **runAsUser**: 10001
>
> **image**: \"{{ .Values.image.repository }}:{{ .Values.image.tag }}\"

如您所见，变量以{{.\_Some.value.here\_}}格式编码，恰好是Go模板。

要安装helm，您可以运行helm install命令。
虽然Helm有几种查找和安装chart的方法，但最简单的方法是使用官方的稳定chart之一：

> *\# get the latest list of charts:*
>
> \$ helm repo update
>
> *\# install MySQL:*
>
> \$ helm install stable/mysql
>
> Released smiling-penguin
>
> *\# list running apps:*
>
> \$ helm ls
>
> NAME VERSION UPDATED STATUS CHART
>
> smiling-penguin 1 Wed Sep 28 12:59:46 2016 DEPLOYED mysql-0.1.0
>
> *\# remove it:*
>
> \$ helm delete smiling-penguin
>
> Removed smiling-penguin

为了打包您的controller，您将需要为其创建一个Helm
chart并将其发布到默认情况下到通过Helm
Hub索引并可以访问的公共存储库中，如图7-1所示。

![elm Hub screen shot, showing publicly available Helm
charts](media/image1.png){width="5.763888888888889in"
height="3.7671675415573054in"}

图7-1。 Helm Hub屏幕截图显示了公开可用的Helm Chart

有关如何创建Helm charts的进一步指导，请在闲暇时细读以下资源：

•Bitnami的精彩文章"How to Create Your First Helm Chart"。

•如果要将图表保留在自己的组织中，则"Using S3 as a Helm Repository"。

•官方Helm文档：" The Chart Best Practices Guide"。

Helm之所以受欢迎，部分原因是它易于最终用户使用。但是，有人认为当前的Helm体系结构会带来安全风险。好消息是，社区正在积极致力于解决这些问题。

Kustomize
---------

Kustomize遵循熟悉的Kubernetes
API提供了一种声明式方法来配置Kubernetes清单文件的配置自定义。它于2018年中推出，现在是Kubernetes
SIG CLI项目。

您可以将Kustomize独立安装在计算机上，或者，如果您具有较新的Kubectl版本（版本低于1.14），则将它与Kubectl一起装运并通过-k命令行标志激活。

因此，Kustomize允许您自定义原始YAML清单文件，而无需接触原始清单。
但是这在实践中如何工作？ 假设您要打包我们的cnat自定义控制器;
您将定义一个名为kustomize.yaml的文件，如下所示：

> **imageTags**:
>
> \- **name**: quay.io/programming-kubernetes/cnat-operator
>
> **newTag**: 0.1.0
>
> **resources**:
>
> \- cnat-controller.yaml

现在，您可以将其应用于带有以下内容的cnat-controller.yaml文件：

> **apiVersion**: apps/v1beta1
>
> **kind**: Deployment
>
> **metadata**:
>
> **name**: cnat-controller
>
> **spec**:
>
> **replicas**: 1
>
> **template**:
>
> **metadata**:
>
> **labels**:
>
> **app**: cnat
>
> **spec**:
>
> **containers**:
>
> \- **name**: custom-controller
>
> **image**: quay.io/programming-kubernetes/cnat-operator

使用Kustomize构建并保留cnat-controller.yaml文件！-然后输出为：

> **apiVersion**: apps/v1beta1
>
> **kind**: Deployment
>
> **metadata**:
>
> **name**: cnat-controller
>
> **spec**:
>
> **replicas**: 1
>
> **template**:
>
> **metadata**:
>
> **labels**:
>
> **app**: cnat
>
> **spec**:
>
> **containers**:
>
> \- **name**: custom-controller
>
> **image**: quay.io/programming-kubernetes/cnat-operator:0.1.0

例如，然后可以在Kubectl
apply命令中使用Kustomize构建的输出，并自动为您应用所有自定义设置。

有关Kustomize的更详细的演练及其使用方法，请查看以下资源：

•SébastienGoasguen的博客文章"Configuring Kubernetes Applications
with kustomize"。

凯文·戴文（Kevin Davin）的帖子"Kustomize---The right way to do
templating in Kubernetes"。

•视频"TGI Kubernetes 072: Kustomize and friends"，您可以在其中观看Joe
Beda的应用。

鉴于Kubectl中Kustomize的本机支持，可能会有越来越多的用户采用它。
请注意，虽然它解决了一些问题（自定义），但生命周期管理还有其他领域，例如验证和升级，可能需要您将Kustomize与Google的CUE等语言一起使用。

为了总结这个Packaging主题，让我们回顾一下从业人员使用的其他解决方案。

Other Packaging Options
-----------------------

上述Packaging选项的一些值得注意的替代品-以及其他野生packaging-是：

UNIX tooling

为了自定义原始Kubernetes清单的值，您可以在Shell脚本中使用一系列CLI工具，例如sed，awk或jq。这是一种流行的解决方案，至少在Helm到来之前，这可能是使用最广泛的选项-尤其是因为它最小化了依赖关系，并且可以跨\*
nix环境进行移植。

传统配置管理系统

您可以使用任何传统的配置管理系统（例如Ansible，Puppet，Chef或Salt）来package和交付operator。

云原生语言

诸如Pulumi和Ballerina之类的新一代所谓的云原生编程语言除其他外，还允许Kubernetes原生应用的打包和生命周期管理。

ytt

有了ytt，您可以使用一种本身就是Google配置语言Starlark修改版的语言的YAML模板工具选项。它在YAML结构上进行语义操作，并专注于可重用性。

科松

Ksonnet最初由Heptio（现为VMware）开发，是Kubernetes清单的配置管理工具，现已过时，不再使用Ksonnet，请自行承担风险。

在Jesse Suen的帖子\`\`The State of Kubernetes Configuration Management:
An Unsolved Problem\'\'中详细了解此处讨论的选项。

现在，我们已经全面讨论了包装选项，现在让我们看看packaging和shipping
controllers和operators的最佳做法。

Packaging 最佳做法
------------------

在packaging和发布operator时，请确保您了解以下最佳做法。无论您选择哪种机制（Helm，Kustomize，shell脚本等），它们都适用：

•提供适当的访问控制设置：这意味着为控制器定义专用的服务帐户，并以最小特权为基础定义RBAC权限；有关更多详细信息，请参见\`\`
Getting the Permissions Right\'\'。

•考虑您的自定义控制器的范围：它将在一个名称空间或多个名称空间中照顾CR吗？查看有关不同方法的利弊的Alex
Ellis在Twitter上的对话。

•测试并分析您的控制器，以便您了解其占用空间和可扩展性。例如，红帽已将一组详细的要求与OperatorHub贡献指南中的说明放在一起。

•确保CRD和controller的文档记录完整，理想情况下应使用godoc.org上可用的嵌入式文档以及一系列使用示例;请参阅Banzai
Cloud的银行金库操作员以获取灵感。

Lifecycle Management
--------------------

与包裹/运输相比，更广泛，更全面的方法是生命周期管理。基本思想是考虑整个供应链，从开发到运输再到升级，并尽可能实现自动化。在这一领域，CoreOS（和后来的Red
Hat）再次成为开拓者：将导致operator的逻辑应用于operator的生命周期管理。换句话说：为了安装并稍后升级operator的自定义控制器，您需要一个专门的操作员，该操作员知道如何处理操作员。实际上，所谓的\`\`运营商生命周期管理器（OLM）\'\'是运营商框架的一部分，该框架还提供了运营商SDK，如\`\`运营商SDK\'\'中所述。

OLM的主要负责人之一吉米·泽林斯基（Jimmy Zelinskie）表示如下：

OLM对运营商作者有很多帮助，但它也解决了一个尚未有人想到的重要问题：随着时间的推移，您如何有效地管理对Kubernetes的一流扩展？

简而言之，OLM提供了一种声明性的方式来安装和升级操作员及其依赖项，以及补充包装解决方案，例如Helm。如果您要购买功能全面的OLM解决方案或为版本和升级挑战创建临时解决方案，则由您决定；但是，您应该在此处制定一些策略。对于某些领域，例如，对于Red
Hat的Operator
Hub的认证过程，不仅建议使用，而且对于任何非平凡的部署方案都是强制性的，即使您并非针对Hub。

Production-Ready Deployments
============================

在本节中，我们回顾并讨论如何使custom controllers
和operators投入生产。以下是高级清单：

•使用Kubernetes部署或DaemonSet监督您的自定义控制器，以便它们在失败时自动重新启动，并且在失败时会自动重新启动。

•通过专用端点实施健康检查，以进行活跃性和准备情况调查。这加上上一步，使您的操作更具弹性。

•考虑leader-follower/standby模型，以确保即使控制器pod崩溃，也可以由其他人接管。但是请注意，同步状态是一项艰巨的任务。

•应用最小特权原则提供访问控制资源，例如服务帐户和角色；有关详细信息，请参见\`\`
Getting the Permissions Right\'。

•考虑自动构建，包括测试。其他一些技巧可以在"Automated Builds and
Testing"中找到。

•主动处理监视和日志记录；有关内容和方式的信息，请参阅\`\` Custom
Controllers and Observability\'\'。

我们还建议您仔细阅读上述文章《Kubernetes Operator Development Guidelines
for Improved Usability》以了解更多信息。

正确获得权限
------------

您的自定义控制器是Kubernetes控制平面的一部分。它需要读取资源状态，在Kubernetes内部（可能在外部）创建资源，并传达其自身资源的状态。对于所有这些，自定义控制器需要正确的权限集，并通过一组基于角色的访问控制（RBAC）相关设置来表达。正确处理是本节的主题。

首先，始终要创建一个专用的服务帐户来运行控制器。换句话说：切勿在名称空间中使用默认服务帐户.1

为了使您的生活更轻松，您可以定义一个带有必要的RBAC规则的ClusterRole以及一个将其绑定到特定名称空间的RoleBinding绑定，从而有效地在各个名称空间中重用角色，如使用RBAC授权条目中所述。

遵循最小特权原则，仅分配控制器执行其工作所需的权限。例如，如果控制器仅管理窗格，则无需为其提供列出或创建部署或服务的权限。另外，请确保控制器未安装CRD和/或准入webhook。换句话说，控制器不应具有管理CRD和Webhooks的权限。

如第6章所述，用于创建自定义控制器的通用工具通常提供开箱即用生成RBAC规则的功能。例如，Kubebuilder会生成以下RBAC资产以及一个operator：

> \$ ls -al rbac/
>
> total 40
>
> drwx\-\-\-\-\-- 7 mhausenblas staff 224 12 Apr 09:52 .
>
> drwx\-\-\-\-\-- 7 mhausenblas staff 224 12 Apr 09:55 ..
>
> -rw\-\-\-\-\-\-- 1 mhausenblas staff 280 12 Apr 09:49
> auth_proxy_role.yaml
>
> -rw\-\-\-\-\-\-- 1 mhausenblas staff 257 12 Apr 09:49
> auth_proxy_role_binding.yaml
>
> -rw\-\-\-\-\-\-- 1 mhausenblas staff 449 12 Apr 09:49
> auth_proxy_service.yaml
>
> -rw-r\--r\-- 1 mhausenblas staff 1044 12 Apr 10:50 rbac_role.yaml
>
> -rw-r\--r\-- 1 mhausenblas staff 287 12 Apr 10:50
> rbac_role_binding.yaml
>
> 查看自动生成的RBAC角色和绑定，可以发现细粒度的设置。
> 在rbac_role.yaml中，您可以找到：
>
> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **kind**: ClusterRole
>
> **metadata**:
>
> **creationTimestamp**: null
>
> **name**: manager-role
>
> **rules**:
>
> \- **apiGroups**:
>
> \- apps
>
> **resources**:
>
> \- deployments
>
> **verbs**: \[\"get\", \"list\", \"watch\", \"create\", \"update\",
> \"patch\", \"delete\"\]
>
> \- **apiGroups**:
>
> \- apps
>
> **resources**:
>
> \- deployments/status
>
> **verbs**: \[\"get\", \"update\", \"patch\"\]
>
> \- **apiGroups**:
>
> \- cnat.programming-kubernetes.info
>
> **resources**:
>
> \- ats
>
> **verbs**: \[\"get\", \"list\", \"watch\", \"create\", \"update\",
> \"patch\", \"delete\"\]
>
> \- **apiGroups**:
>
> \- cnat.programming-kubernetes.info
>
> **resources**:
>
> \- ats/status
>
> **verbs**: \[\"get\", \"update\", \"patch\"\]
>
> \- **apiGroups**:
>
> \- admissionregistration.k8s.io
>
> **resources**:
>
> \- mutatingwebhookconfigurations
>
> \- validatingwebhookconfigurations
>
> **verbs**: \[\"get\", \"list\", \"watch\", \"create\", \"update\",
> \"patch\", \"delete\"\]
>
> \- **apiGroups**:
>
> \- \"\"
>
> **resources**:
>
> \- secrets
>
> **verbs**: \[\"get\", \"list\", \"watch\", \"create\", \"update\",
> \"patch\", \"delete\"\]
>
> \- **apiGroups**:
>
> \- \"\"
>
> **resources**:
>
> \- services
>
> **verbs**: \[\"get\", \"list\", \"watch\", \"create\", \"update\",
> \"patch\", \"delete\"\]

查看Kubebuilder在v1中生成的这些权限，您可能会有些吃惊.2我们当然是：最佳实践告诉我们，如果控制器没有很好的理由，则应该不能这样做。
：

•编写通常只在代码中读取的资源。例如，如果您仅监视服务和部署，请删除角色中的create，update，patch和delete动词。

•访问所有秘密；也就是说，始终将其限制为必要的最小机密集。

•编写MutingWebhookConfigurations或ValidationWebhookConfigurations。这等效于访问群集中的任何资源。

•编写CustomResourceDefinitions。请注意，在刚刚显示的集群角色中不允许这样做，但是在此处必须提及，但重要的是：CRD的创建应由单独的过程完成，而不是由控制器本身完成。

•编写未管理的外部资源的/状态子资源（请参阅"子资源"）。例如，此处的部署不受cnat控制器管理，因此不应在范围内。

当然，Kubebuilder并不能真正理解您的控制器代码的实际作用。因此，生成的RBAC规则过于宽松也就不足为奇了。我们建议按照前面的清单仔细检查权限并将其减少到绝对最小值。

###### 警告

拥有对系统中所有机密的读访问权限，控制器就可以访问所有服务帐户令牌。
这等效于可以访问群集中的所有密码。
拥有对MutingWebhookConfigurations或ValidatingWebhookConfigurations的写权限，使您可以拦截和处理系统中的每个API请求。
这为Kubernetes集群中的rootkit打开了大门。
两者显然都是非常危险的并且被视为反模式，因此最好避免使用它们。

为了避免拥有太多功能-将访问权限限制为绝对必要的权限-请考虑使用audit2rbac。
该工具使用审核日志生成一组适当的权限，从而导致更安全的设置和更少的麻烦。

从rbac_role_binding.yaml您可以学习：

> **apiVersion**: rbac.authorization.k8s.io/v1
>
> **kind**: ClusterRoleBinding
>
> **metadata**:
>
> **creationTimestamp**: null
>
> **name**: manager-rolebinding
>
> **roleRef**:
>
> **apiGroup**: rbac.authorization.k8s.io
>
> **kind**: ClusterRole
>
> **name**: manager-role
>
> **subjects**:
>
> \- **kind**: ServiceAccount
>
> **name**: default
>
> **namespace**: system

有关RBAC及其周围工具的更多最佳实践，请访问RBAC.dev，这是Kubernetes中专用于RBAC的网站。
现在让我们继续进行自定义控制器的测试和性能注意事项。

自动化构建和测试
----------------

作为云原生领域的最佳实践，请考虑自动构建自定义控制器。这通常称为连续构建或连续集成（CI），包括单元测试，集成测试，构建容器镜像以及可能的完整性或烟雾测试。
Cloud Native Computing
Foundation（CNCF）维护着许多可用的开源CI工具的交互式清单。

在构建控制器时，请记住，它应该消耗尽可能少的计算资源，同时要服务尽可能多的客户端。每个CR（基于您定义的CRD）都是客户端的代理。但是，您如何知道它消耗了多少内存，是否在哪里泄漏内存以及如何扩展内存呢？

一旦您的自定义控制器的开发稳定下来，您就可以并且确实应该进行大量测试。这些可以包括以下内容，但不限于此：

•Kubernetes本身以及kboom工具中发现的与性能相关的测试，可以为您提供有关扩展和资源占用的数据。

•浸泡测试（例如，Kubernetes中使用的测试）旨在从数小时到数天的长期使用，目的是揭示文件或主内存等资源泄漏的情况。

作为最佳实践，这些测试应成为CI管道的一部分。换句话说，从一开始就自动构建自定义控制器，测试和打包。对于具体的示例设置，我们建议您阅读MarkoMudrinić的精彩文章\`\`在CI中生成Kubernetes集群以进行集成和E2E测试\'\'。

接下来，我们将研究为有效排障提供基础的最佳做法：对可观察性的内置支持。

Custom Controllers and Observability
------------------------------------

在本节中，我们将研究自定义控制器的可观察性方面，特别是日志记录和监视。

### LOGGING

确保提供足够的日志记录信息以帮助进行故障排除（生产中）。
通常在容器化设置中，日志信息被发送到stdout，在这里可以使用Kubectl
logs命令按每个容器或以汇总形式使用日志信息。
可以使用特定于云提供商的解决方案（例如Google
Cloud中的Stackdriver或AWS中的CloudWatch）或定制解决方案（例如Elasticsearch-Logstash-Kibana
/ Elasticsearch-Fluentd-Kibana堆栈）来提供聚合。
另请参见SébastienGoasguen和Michael Hausenblas（O'Reilly）撰写的《
Kubernetes Cookbook》，以获取有关此主题的食谱。

让我们看一下CNAT自定义控制器日志的示例摘录：

> { \"level\":\"info\",
>
> \"ts\":1555063927.492718,
>
> \"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling At\" }
>
> { \"level\":\"info\",
>
> \"ts\":1555063927.49283,
>
> \"logger\":\"controller\",
>
> \"msg\":\"Phase: PENDING\" }
>
> { \"level\":\"info\",
>
> \"ts\":1555063927.492857,
>
> \"logger\":\"controller\",
>
> \"msg\":\"Checking schedule\" }
>
> { \"level\":\"info\",
>
> \"ts\":1555063927.492915,
>
> \"logger\":\"controller\",
>
> \"msg\":\"Schedule parsing done\" }

日志记录方式：一般而言，我们更喜欢结构化日志记录和可调整的日志级别，至少是调试和信息。
Kubernetes代码库中广泛使用两种方法，除非有充分的理由，否则应考虑使用这些方法：

•记录器接口-例如在httplog.go中找到的接口以及具体类型（respLogger）-捕获状态和错误之类的信息。

•klog，是Google
glog的一个分支，是整个Kubernetes都使用的结构化记录器，尽管它有其独特之处，但值得了解。

日志记录内容：确保具有正常业务逻辑操作情况下的详细日志信息。
例如，从at_controller.go中我们的cnat控制器的Operator
SDK实现中，如下设置记录器：

> reqLogger := log.WithValues(\"namespace\", request.Namespace, \"at\",
> request.Name)

然后在业务逻辑中的Reconcile（request reconcile.Request）功能中：

> **case** cnatv1alpha1.PhasePending:
>
> reqLogger.Info(\"Phase: PENDING\")
>
> *// As long as we haven\'t executed the command yet, we need to check
> if it\'s*
>
> *// already time to act:*
>
> reqLogger.Info(\"Checking schedule\", \"Target\",
> instance.Spec.Schedule)
>
> *// Check if it\'s already time to execute the command with a
> tolerance of*
>
> *// 2 seconds:*
>
> d, err := timeUntilSchedule(instance.Spec.Schedule)
>
> **if** err != **nil** {
>
> reqLogger.Error(err, \"Schedule parsing failure\")
>
> *// Error reading the schedule. Wait until it is fixed.*
>
> **return** reconcile.Result{}, err
>
> }
>
> reqLogger.Info(\"Schedule parsing done\", \"Result\", \"diff\",
> fmt.Sprintf(\"%v\", d))
>
> **if** d \> 0 {
>
> *// Not yet time to execute the command, wait until the scheduled
> time*
>
> **return** reconcile.Result{RequeueAfter: d}, **nil**
>
> }
>
> reqLogger.Info(\"It\'s time!\", \"Ready to execute\",
> instance.Spec.Command)
>
> instance.Status.Phase = cnatv1alpha1.PhaseRunning

这个Go片段使您可以很好地了解要记录的内容，尤其是何时使用reqLogger.Info和reqLogger.Error。

不再使用Logging 101，让我们继续讨论一个相关主题：metrics！

### MONITORING, INSTRUMENTATION, AND AUDITING

Prometheus是您可以在各种环境（本地和云中）中使用的出色的开源容器就绪监控解决方案。在每个事件上发出警报都是不切实际的，因此您可能要考虑需要向谁通报哪种事件。例如，您可能有一个策略，即与基础节点管理员或节点相关事件或与名称空间相关的事件由基础架构管理员处理，而名称空间管理员或开发人员则针对页面级事件进行分页。在这种情况下，为了可视化您收集的指标，最受欢迎的解决方案当然是格拉法纳（Grafana）;有关从Grafana中可视化的Prometheus度量的示例，请参见图7-2，摘自Prometheus文档。

如果您使用服务网格（例如基于Envoy代理（例如Istio或App
Mesh）或Linkerd），则通常会免费提供工具，或者只需最少的配置工作即可实现。否则，您将必须使用相应的库（例如Prometheus提供的库）在您的代码中公开相关指标。在这种情况下，您可能还想看看2019年初推出的新兴服务网格接口（SMI）项目，该项目旨在基于CR和控制器为服务网格提供标准化接口。

![rometheus metrics visualized in
Grafana](media/image2.png){width="5.763888888888889in"
height="4.45432195975503in"}

###### *图7-2. Prometheus metrics visualized in Grafana*

Kubernetes通过API服务器提供的另一个有用功能是审核，它允许您记录一系列影响集群的活动。
从不记录到记录事件元数据，请求正文和响应正文，审核策略中可以使用不同的策略。
您可以在简单的日志后端和使用Webhook与第三方系统集成之间进行选择。

摘要
====

本章通过讨论controller和operators的操作方面（包括包装，安全性和性能），着重于如何使operators准备就绪。

到此为止，我们已经涵盖了编写和使用自定义Kubernetes控制器和运算符的基础知识，因此，现在我们进入另一种扩展Kubernetes的方法：开发自定义API服务器。

1另请参阅Luc Juggery的帖子"
Kubernetes技巧：使用ServiceAccount"以获取有关服务帐户用法的详细讨论。

2但是，我们确实针对Kubebuilder项目提出了问题748。
