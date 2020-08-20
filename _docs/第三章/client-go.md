---
title: 第三章
category: client-go 基础
order: 2
---

第三章.  client-go 基础
=======================

现在，我们将重点介绍Go中的Kubernetes编程界面。您将学习如何访问Pod，service和deploymentploy等知名本机类型的Kubernetes
API。在后面的章节中，这些技术将扩展到用户定义的类型。不过，在这里我们首先关注每个Kubernetes集群附带的所有API对象。

仓库
====

Kubernetes项目在GitHub的kubernetes组织下提供了许多第三方可消耗的Git存储库。您需要将所有这些都以域别名k8s.io
/ （不是github.com/kubernetes / ）导入项目中。
我们将在以下各节中介绍这些存储库中最重要的存储库。

客户端库
--------

Go中的Kubernetes编程接口主要由k8s.io/client-go库组成（为简便起见，我们将其简称为client-go-going）。
client-go是一个典型的Web服务客户端库，它支持Kubernetes正式包含的所有API类型。它可以用来执行常用的REST动词：

-   *Create*

-   *Get*

-   *List*

-   *Update*

-   *Delete*

-   *Patch*

每个REST动词都使用\` ["The HTTP Interface of the API
Server"](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)实现。
此外，还支持动词Watch（监视），这对于类似Kubernetes的API来说是特殊的，并且是与其他API相比的主要区别之一。

client-go在GitHub上可用（请参见图3-1），并在Go代码中以k8s.io/client-go软件包名称使用。
它与Kubernetes本身并行交付；也就是说，对于每个Kubernetes
1.x.y版本，都有一个带有匹配标签kubernetes-1.x.y的客户端发布版本。

![he \`client-go\` repository on
Github](media/image1.png){width="5.763888888888889in"
height="5.431208442694663in"}

###### *图3-1.  client-go GitHub 仓库*

此外，还有一个语义版本转换方案。 例如，client-go 9..0.0与Kubernetes
1.12版本匹配，client-go 10..0.0与Kubernetes 1.13版本匹配，依此类推。
将来可能会有更多的细粒度版本。除了Kubernetes
API对象的客户端代码外，client-go还包含许多通用库代码。这也用于第4章中的用户定义的API对象。有关软件包列表，请参见图3-1。

虽然所有软件包都可以使用，但是与Kubernetes
API通讯的大多数代码将使用tools / clientcmd
/从kubeconfig文件中设置客户端，并为实际的Kubernetes
API客户端使用kubernetes /。 我们将很快看到代码执行此操作。
在此之前，让我们快速浏览一下其他相关的存储库和软件包。

Kubernetes API 类型
-------------------

如我们所见，client-go保持客户端接口。 Pod，服务和部署等对象的Kubernetes
API Go类型位于其自己的存储库中。 在Go代码中以k8s.io/api访问。

Pod是旧版API组（通常也称为"核心"组）版本v1的一部分。
因此，可以在k8s.io/api/core/v1中找到Pod
Go类型，对于Kubernetes中的所有其他API类型也是如此。
有关软件包列表，请参见图3-2，其中大多数软件包对应于Kubernetes
API组及其版本。

实际的Go类型包含在types.go文件中（例如k8s.io/api/core/v1/types.go）。
此外，还有其他文件，其中大多数由代码生成器自动生成。

###### ![](media/image2.png){width="5.763888888888889in" height="5.312623578302712in"} *图3-2. The API repository on GitHub*

API Machinery
-------------

最后但并非最不重要的是，还有第三个存储库称为API
Machinery，在Go中用作k8s.io/apimachinery。 它包括用于实现类似Kubernetes
API的所有通用构建块。 API
Machinery不限于容器管理，因此，例如，它可以用于为在线商店或任何其他特定于业务的域构建API。

不过，您将在Kubernetes本地Go代码中遇到很多API Machinery软件包。
重要的是k8s.io/apimachinery/pkg/apis/meta/v1。
它包含许多通用API类型，例如ObjectMeta，TypeMeta，GetOptions和ListOptions（请参见图3-3）。

![PI Machinery repository on
Github](media/image3.png){width="5.763888888888889in"
height="4.035748031496063in"}

###### *图3-3. The API Machinery repository on GitHub*

创建和使用 Client
-----------------

现在我们知道创建Kubernetes客户端对象的所有构件，这意味着我们可以访问Kubernetes集群中的资源。
假设您有权访问本地环境中的集群（即正确设置了Kubectl并配置了凭据），则以下代码说明了如何在Go项目中使用client-go：

> **import** (
>
> metav1 \"k8s.io/apimachinery/pkg/apis/meta/v1\"
>
> \"k8s.io/client-go/tools/clientcmd\"
>
> \"k8s.io/client-go/kubernetes\"
>
> )
>
> kubeconfig = flag.String(\"kubeconfig\", \"\~/.kube/config\",
> \"kubeconfig file\")
>
> flag.Parse()
>
> config, err := clientcmd.BuildConfigFromFlags(\"\", \*kubeconfig)
>
> clientset, err := kubernetes.NewForConfig(config)
>
> pod, err := clientset.CoreV1().Pods(\"book\").Get(\"example\",
> metav1.GetOptions{})

该代码导入了meta / v1软件包以访问metaav1.GetOptions。
此外，它从client-go导入clientcmd，以读取和解析kubeconfig（即具有服务器名称，凭据等的客户端配置）。然后导入带有Kubernetes资源客户端集的client-go
Kubernetes软件包。

kubeconfig文件的默认位置在用户主目录中的.kube / config中。
这也是Kubectl获取Kubernetes集群的凭据的地方。

然后使用clientcmd.BuildConfigFromFlags读取并解析该kubeconfig。
我们在整个代码中都省略了强制性错误处理，但如果kubeconfig格式不正确，则err变量通常会包含例如语法错误。
由于语法错误在Go代码中很常见，因此应正确检查此类错误，如下所示：

> config, err := clientcmd.BuildConfigFromFlags(\"\", \*kubeconfig)
>
> **if** err != **nil** {
>
> fmt.Printf(\"The kubeconfig cannot be loaded: %v\\n\", err
>
> os.Exit(1)
>
> }

从clientcmd.BuildConfigFromFlags中获得一个rest.Config，您可以在k8s.io/client-go/rest软件包中找到）。
这被传递给kubernetes.NewForConfig以创建实际的Kubernetes
Client。之所以称为客户端集，是因为它包含所有本地Kubernetes资源的多个客户端。

当在集群中的Pod中运行二进制文件时，小组件将自动将服务帐户安装到容器中的/var/run/secrets/kubernetes.io/serviceaccount。
它替换了刚才提到的kubeconfig文件，可以通过rest.InClusterConfig（）方法轻松地将其转换为rest.Config您通常会发现rest.InClusterConfig（）和clientcmd.BuildConfigFromFlags（）的以下组合，包括对KUBECONFIG环境变量的支持：

> config, err := rest.InClusterConfig()
>
> **if** err != **nil** {
>
> *// fallback to kubeconfig*
>
> kubeconfig := filepath.Join(\"\~\", \".kube\", \"config\")
>
> **if** envvar := os.Getenv(\"KUBECONFIG\"); len(envvar) \>0 {
>
> kubeconfig = envvar
>
> }
>
> config, err = clientcmd.BuildConfigFromFlags(\"\", kubeconfig)
>
> **if** err != **nil** {
>
> fmt.Printf(\"The kubeconfig cannot be loaded: %v\\n\", err
>
> os.Exit(1)
>
> }
>
> }

在以下示例代码中，我们使用clientset.CoreV1（）在v1中选择核心组，然后在"
book"命名空间中访问pod" example"

> pod, err := clientset.CoreV1().Pods(\"book\").Get(\"example\",
> metav1.GetOptions{})

请注意，只有最后一个函数调用Get才能实际访问服务器。CoreV1和Pod都选择客户端并仅为以下Get调用设置名称空间（通常称为构建器模式，在这种情况下为构建请求）。

Get调用向服务器上的/ api / v1 / namespaces / book / pods /
example发送一个HTTP GET请求，该请求在kubeconfig中设置。 如果Kubernetes
API服务器用HTTP代码200回答，则响应的主体将以JSON（这是client-go的默认连线格式）或协议缓冲区的形式携带编码的pod对象。

###### 注意

您可以在通过REST配置创建客户端之前，通过修改REST配置为本地Kubernetes资源客户端启用protobuf：

> cfg, err := clientcmd.BuildConfigFromFlags(\"\", \*kubeconfig)
>
> cfg.AcceptContentTypes = \"application/vnd.kubernetes.protobuf,
>
> application/json\"
>
> cfg.ContentType = \"application/vnd.kubernetes.protobuf\"
>
> clientset, err := kubernetes.NewForConfig(cfg)

请注意，第4章介绍的自定义资源不支持协议缓冲区

版本和兼容性
------------

Kubernetes API已版本化。我们在上一节中看到Pod在core
Group的v1中。如今，core
Group实际上只存在一个版本。但是，还有其他组-例如app组，它存在于v1，v1beta2和v1beta1中（截至撰写本文时）。如果查看k8s.io/api/apps软件包，您将找到这些版本的所有API对象。在k8s.io/client-go/kubernetes/typed/apps软件包中，您会看到所有这些版本的client实现。

所有这些仅是客户端。它没有说明Kubernetes集群及其API服务器。将客户端与API服务器不支持的API组版本一起使用会失败。客户端被硬编码为一个版本，并且应用程序开发人员必须选择正确的API组版本才能与手头的集群对话。有关API组兼容性保证的更多信息，请参见\` ["API
Versions and Compatibility
Guarantees"](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#api-versions)。

兼容性的第二个方面是客户端访问的API服务器的元API功能。例如，存在CRUD动词的选项结构，例如CreateOptions，GetOptions，UpdateOptions和DeleteDelete。另一个重要的对象是ObjectMeta（在"
ObjectMeta"中详细讨论），它是每种类型的一部分。所有这些都经常通过新功能进行扩展。我们通常称它们为API机制功能。在其字段的Go文档中，注释指定何时将功能视为alpha或beta。相同的API兼容性保证适用于任何其他API字段。

在下面的示例中，DeleteOptions结构：

package [*k8s.io/apimachinery/pkg/apis/meta/v1/types.go*](http://bit.ly/2MZ9flL):

> *// DeleteOptions may be provided when deleting an API object.*
>
> *// 删除API对象时，提供DeleteOptions*
>
> **type** DeleteOptions **struct** {
>
> TypeMeta \`json:\",inline\"\`
>
> GracePeriodSeconds \***int64**
> \`json:\"gracePeriodSeconds,omitempty\"\`
>
> Preconditions \*Preconditions \`json:\"preconditions,omitempty\"\`
>
> OrphanDependents \***bool** \`json:\"orphanDependents,omitempty\"\`
>
> PropagationPolicy \*DeletionPropagation
> \`json:\"propagationPolicy,omitempty\"\`
>
> //如果存在，则指示不应持久化修改。无效或无法识别的dryRun指令将导致错误响应，并且不会进一步处理该请求。有效值为：
>
> \- All: all dry run stages will be processed
>
> +optional
>
> DryRun \[\]**string** \`json:\"dryRun,omitempty\"
> protobuf:\"bytes,5,rep,name=dryRun\"\`
>
> }

最后一个字段DryRun在Kubernetes
1.12中添加为alpha，在1.13中添加为beta（默认启用）。
早期版本的API服务器无法识别。根据功能的不同，传递这样的选项可能会被忽略甚至拒绝。因此，拥有一个与客户端版本相距不太远的client-go版本非常重要。

###### TIP

k8s.io/api中的来源是哪个质量级别可用的参考，例如，版本1.13分支中的Kubernetes
1.13可以访问。 Alpha字段在其描述中进行了标记。

有生成的API文档，以便于使用。但是，它与k8s.io/api中的信息相同。

最后但并非最不重要的一点是，许多alpha和beta功能都有相应的功能门（请在此处查看主要来源）。
在问题中跟踪功能。

集群版本和客户端版本之间的正式保证支持矩阵发布在客户端版本自述文件中（请参阅表3-1）。

                                                                  **Kubernetes 1.9**   **Kubernetes 1.10**   **Kubernetes 1.11**   **Kubernetes 1.12**   **Kubernetes 1.13**   **Kubernetes 1.14**   **Kubernetes 1.15**
  --------------------------------------------------------------- -------------------- --------------------- --------------------- --------------------- --------------------- --------------------- ---------------------
  client-go 6.0                                                   ✓                    +--                   +--                   +--                   +--                   +--                   +--
  client-go 7.0                                                   +--                  ✓                     +--                   +--                   +--                   +--                   +--
  client-go 8.0                                                   +--                  +--                   ✓                     +--                   +--                   +--                   +--
  client-go 9.0                                                   +--                  +--                   +--                   ✓                     +--                   +--                   +--
  client-go 10.0                                                  +--                  +--                   +--                   +--                   ✓                     +--                   +--
  client-go 11.0                                                  +--                  +--                   +--                   +--                   +--                   ✓                     +--
  client-go 12.0                                                  +--                  +--                   +--                   +--                   +--                   +--                   ✓
  client-go HEAD                                                  +--                  +--                   +--                   +--                   +--                   +--                   +--
  *Table 3-1. client-go compatibility with Kubernetes versions*                                                                                                                                      

•✓：client-go版本和Kubernetes版本具有相同的功能和相同的API组版本。

•+：client-go具有Kubernetes集群中可能缺少的功能或API组版本。这可能是由于client-go中增加了功能或Kubernetes删除了过时的旧功能。但是，它们的所有共同点（即大多数API）都可以使用。

•--：client-go已知与Kubernetes集群不兼容。

表3-1的要点是客户端数据库具有相应的集群版本。如果出现版本偏差，开发人员必须仔细考虑他们使用的功能和API组，以及应用程序所针对的群集版本是否支持这些功能。

在表3-1中，列出了客户端版本。我们在\`\`客户端库\'\'中简要提到了客户端使用正式的语义版本控制（Semver），尽管每次增加客户端的主要版本都会增加Kubernetes的次要版本（1.13.2中的13）。在针对Kubernetes
1.4发布了client-go 1.0之后，我们现在在Kubernetes 1.15上处于client-go
12.0（在撰写本文时）。

此服务仅适用于客户端本身，不适用于API
Machinery或API存储库。相反，后者使用Kubernetes版本标记，如图3-4所示。请参阅["Vendoring"](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#vendoring) 以了解在项目中供应k8s.io/client-go、k8s.io/apimachinery和k8s.io/api意味着什么。

![lient-go versioning](media/image4.png){width="5.763888888888889in"
height="2.6293897637795274in"}

###### *图3-4. client-go versioning*

API 版本和兼容性保证
--------------------

如上一节所述，如果您使用代码定位不同的集群版本，那么选择正确的API组版本可能至关重要。Kubernetes版本所有API组。使用了通用的Kubernetes风格的版本控制方案，该方案由alpha，beta和GA（通用）版本组成。

模式是：

•v1alpha1，v1alpha2，v2alpha1等被称为alpha版本，并被认为是不稳定的。这表示：

•它们可能以任何不兼容的方式随时消失或改变。

•Kubernetes的各个版本之间，数据可能会丢失，丢失或变得不可访问。

如果管理员未手动选择，则通常默认情况下禁用它们。

•v1beta1，v1beta2，v2beta1等称为Beta版本。它们正在走向稳定，这意味着：

•它们至少与对应的稳定API版本并行存在至少一个Kubernetes版本仍然存在。

•它们通常不会以不兼容的方式进行更改，但是对此没有严格的保证。

stored存储在beta版本中的对象将不会被丢弃或变得不可访问。

•默认情况下，通常在群集中启用Beta版本。但这可能取决于所使用的Kubernetes发行版或云提供商。

v1，v2等是稳定的，通用的API；那是：他们会留下来，且兼容。

###### TIP

在这些经验法则的背后，Kubernetes有一个正式的弃用政策。您可以在Kubernetes社区GitHub上找到更多有关哪些API构造被认为兼容的详细信息。

关于API组版本，有两点需要牢记：

•API Group version总体上适用于API资源，例如pod或service的格式。 除了API
Group version以外，API资源可能还包含单个字段，这些字段是正交版本的。
例如，稳定的API中的字段可能会在其Go内联代码文档中标记为alpha质量。
与刚才为API组列出的规则相同的规则将应用于这些字段。 例如：

a稳定的API中的Alpha字段可能会变得不兼容，丢失数据或随时消失。
例如，从未提升过alpha范围的ObjectMeta.Initializers字段将在不久的将来消失（1.14中已弃用）：

-   *// DEPRECATED - initializers are an alpha field and will be
    > removed*

-   *// in v1.15.*

> Initializers \*Initializers \`json:\"initializers,omitempty\"

通常默认情况下将禁用它，并且必须使用API服务器功能门启用它，如下所示：

-   **type** JobSpec **struct** {

-   \...

-   *// This field is alpha-level and is only honored by servers that*

-   *// enable the TTLAfterFinished feature.*

-   TTLSecondsAfterFinished \***int32**
    > \`json:\"ttlSecondsAfterFinished,omitempty\"

> }

API服务器的行为因字段而异。如果未启用相应的功能门，则某些Alpha字段将被拒绝，而某些Alpha字段将被忽略。这在字段描述中进行了记录（请参见上一示例中的TTLSecondsAfterFinished）。

此外，API Group
version在访问API中也起作用。在同一资源的不同版本之间，API服务器会进行即时转换。也就是说，您可以访问在任何其他受支持版本（例如v1）中的一个版本（例如v1beta1）中创建的对象，而无需在应用程序中进行任何进一步的工作。这对于构建向后和向前兼容的应用程序非常方便。

存储在etcd中的每个对象都以特定版本存储。默认情况下，这称为该资源的存储版本。虽然存储版本可以从Kubernetes版本更改到版本，但是存储在etcd中的对象在撰写本文时不会自动更新。因此，集群管理员必须确保在删除旧版本支持之前，在更新Kubernetes集群时及时进行迁移。对此没有通用的迁移机制，迁移因Kubernetes发行版而异。

但是，对于应用程序开发人员而言，此操作工作根本不重要。快速转换将确保应用程序对群集中的对象具有统一的了解。该应用程序甚至不会注意到正在使用哪个存储版本。存储版本控制对于编写的Go代码是透明的。

Go中的 Kubernetes对象
=====================

在"Creating and Using a
Client"中，我们介绍了如何为核心组创建客户端以访问Kubernetes集群中的Pod。
在下文中，我们想更详细地了解Go领域中的Pod或任何其他Kubernetes资源。

Kubernetes资源（或更确切地说是对象）是一种kind1的实例，并被API服务器用作资源，以结构表示。根据所讨论的种类，它们的领域当然会有所不同。但另一方面，它们具有相同的结构。

从类型系统的角度来看，Kubernetes对象从包k8s.io/apimachinery/pkg/runtime中实现了一个名为runtime.Object的Go接口，实际上非常简单：

> *在Scheme上注册的所有API类型都必须支持对象接口。*
>
> *由于期望方案中的对象被序列化到线路，因此对象必须提供给方案的接口允许序列化程序设置对象所表示的kind，version和group。
> 在不希望序列化对象的情况下，对象可以选择返回无操作ObjectKindAccessor。*
>
> **type** Object **interface** {
>
> GetObjectKind() schema.ObjectKind
>
> DeepCopyObject() Object
>
> }

Here, schema.ObjectKind (from
the *k8s.io/apimachinery/pkg/runtime/schema* package) is another simple
interface:

*//
从Scheme序列化的所有对象都对其类型信息进行编码。序列化使用此接口将来自Scheme的类型信息设置到对象的序列化版本上。
对于无法序列化或具有独特要求的对象，此接口可能是无操作的。*

> **type** ObjectKind **interface** {
>
> *//SetGroupVersionKind设置或清除对象的预期序列化类型。
> 传递nil应该清除当前设置。*

SetGroupVersionKind(kind GroupVersionKind)

> *// GroupVersionKind返回存储的group，version和kind*
>
> *     object，如果对象未公开或不提供这些字段，则为nil。*
>
> GroupVersionKind() GroupVersionKind
>
> }

换句话说，Go中的Kubernetes对象是一种数据结构，可以：

返回并设置GroupVersionKind

深度复制

深拷贝是数据结构的克隆，因此它不与原始对象共享任何内存。它在代码必须更改对象而不修改原始对象的任何地方使用。
有关在Kubernetes中如何实现深度复制的详细信息，请参见关于代码生成的\`\`全局标签\'\'。

简而言之，对象存储其类型并允许克隆。

TypeMeta
--------

虽然runtime.Object只是一个接口，但我们想知道它是如何实现的。通过从包k8s.io/apimachinery/meta/v1中嵌入metaav1.TypeMeta结构，来自k8s.io/api的Kubernetes对象实现了Schema.ObjectKind的类型获取器和设置器：

> *//
> TypeMeta使用表示对象类型及其API架构版本的字符串描述API响应或请求中的单个对象。
> 版本化或持久化的结构应内联TypeMeta。TypeMeta.*
>
> *// +k8s:deepcopy-gen=false*
>
> **type** TypeMeta **struct** {
>
> *Kind是一个字符串值，表示此对象的REST资源*
>
> *  代表。 服务器可以从客户端向其提交请求的端点推断出这一点。*
>
> *无法更新*
>
> *// +optional*
>
> Kind **string** \`json:\"kind,omitempty\"
> protobuf:\"bytes,1,opt,name=kind\"\`
>
> *APIVersion定义了此对象表示形式的版本控制架构。
> 服务器应将已识别的架构转换为最新的内部值，并可能拒绝无法识别的值。*
>
> APIVersion **string** \`json:\"apiVersion,omitempty\"\`
>
> }

Go中的pod声明如下所示：

> *//Pod是可以在主机上运行的容器的集合。该资源是由客户端创建并调度到主机上。*
>
> **type** Pod **struct** {
>
> metav1.TypeMeta \`json:\",inline\"\`
>
> *//标准对象的元数据。// +可选的*
>
> metav1.ObjectMeta \`json:\"metadata,omitempty\"\`
>
> *//pod所需行为的规范。*

*// +可选*

> Spec PodSpec \`json:\"spec,omitempty\"\`
>
> *// 最近观察到的容器状态。*
>
> *//此数据可能不是最新的。*
>
> *//由系统填充。*

*//只读* *// +可选*

> Status PodStatus \`json:\"status,omitempty\"\`
>
> }

如您所见，TypeMeta已嵌入。此外，pod类型具有JSON标记，这些标记也声明了TypeMeta被内联。

###### 注意

这个"inline"标签实际上与Golang JSON en /
decoders无关：嵌入的结构是自动内联的。

这与YAML en / decoder go-yaml /
yaml不同，后者在很早的Kubernetes代码中与JSON并行使用。
我们从那时起继承了inline标记，但是今天它只是文档，没有任何作用。

K8s.io/apimachinery/pkg/runtime/serializer/yaml中的YAML序列化器使用sigs.k8s.io/yaml的编组和解组功能。
然后通过接口{}对YAML进行编码和解码，并使用JSON编码器编码和从Golang
API结构解码器。

这与Pod的YAML表示匹配，所有Kubernetes用户都知道：

> **apiVersion**: v1
>
> **kind**: Pod
>
> **metadata**:
>
> **namespace**: default
>
> **name**: example
>
> **spec**:
>
> **containers**:
>
> \- **name**: hello
>
> **image**: debian:latest
>
> **command**:
>
> \- /bin/sh
>
> **args**:
>
> \- -c
>
> \- echo \"hello world\"; sleep 10000

版本存储在TypeMeta.APIVersion中，类型存储在TypeMeta.Kind中。

##### 核心组因历史原因而不同

Pod和很早就添加到Kubernetes的许多其他类型是core组的一部分-通常也称为Legacy组-由空字符串表示。因此，apiVersion仅设置为"
v1"。

最终，API组被添加到Kubernetes中，并且组名之间用斜杠分隔，被添加到apiVersion之前。如果是应用程序，则版本为apps
/ v1。 因此，apiVersion字段实际上被错误命名;
它存储API组名称和版本字符串。
这是出于历史原因，因为仅在存在核心组而没有其他API组时定义了apiVersion。

运行\`\`创建和使用客户端\'\'中的示例以从集群中获取Pod时，请注意客户端返回的Pod对象实际上没有设置kind和version。基于客户端的应用程序中的约定是，这些字段在内存中为空，并且仅在将它们编组为JSON或protobuf时，才会在网络上填充实际值。但是，这是由客户端自动完成的，或更确切地说，是由版本控制序列化程序完成的。

##### BEHIND THE幕后花絮：KINDS，PACKAGES，GROUP NAMES关联？

您可能想知道客户端如何知道填充TypeMeta字段的kind和API Group。
尽管起初这个问题听起来很琐碎，但事实并非如此：

•看起来种类只是Go类型的名称，可以通过反射从对象派生。多数情况下是正确的-也许在99％的情况下-但也有例外（在第4章中，您将了解如何使用的自定义资源）。

•该组似乎只是Go软件包名称（apps API组的类型在k8s.io/api/apps中声明）。
这通常是匹配的，但并非在所有情况下都匹配：如我们所见，核心组的组名称字符串为空。例如，组rbac.authorization.k8s.io的类型在k8s.io/api/rbac中，而不在k8s.io/api/rbac.authorization.k8s.io中。

有关如何填写TypeMeta字段的问题的正确答案涉及方案的概念，将在\`\`方案\'\'中详细讨论。

换句话说，基于客户端的应用程序会检查对象的Golang类型以确定手边的对象。
这在其他框架中可能会有所不同，例如Operator SDK（请参阅\`\`Operator
SDK\'\'）。

ObjectMeta
----------

除了TypeMeta之外，大多数顶级对象都具有一个类型为metaav1.ObjectMeta的字段，同样来自k8s.io/apimachinery/pkg/meta/v1包：

> **type** ObjectMeta **struct** {
>
> Name **string** \`json:\"name,omitempty\"\`
>
> Namespace **string** \`json:\"namespace,omitempty\"\`
>
> UID types.UID \`json:\"uid,omitempty\"\`
>
> ResourceVersion **string** \`json:\"resourceVersion,omitempty\"\`
>
> CreationTimestamp Time \`json:\"creationTimestamp,omitempty\"\`
>
> DeletionTimestamp \*Time \`json:\"deletionTimestamp,omitempty\"\`
>
> Labels **map**\[**string**\]**string** \`json:\"labels,omitempty\"\`
>
> Annotations **map**\[**string**\]**string**
> \`json:\"annotations,omitempty\"\`
>
> \...
>
> }

在JSON或YAML中，这些字段位于元数据下。例如，对于上一个pod，metav1.ObjectMeta存储：

> **metadata**:
>
> **namespace**: default
>
> **name**: example

通常，它包含所有元级别信息，例如名称，名称空间，资源版本（请勿与API组版本混淆），多个时间戳，并且众所周知的标签和注释是ObjectMeta的一部分。有关ObjectMeta字段的更详细讨论，请参见\`\`类型的解剖\'\'。

资源版本在前面的\`\`乐观并发性\'\'中讨论过。
几乎不会从client-go代码中读取或写入它。但是，使整个系统正常工作的是Kubernetes的领域之一。
resourceVersion是ObjectMeta的一部分，因为每个嵌入了ObjectMeta的对象都对应于源自resourceVersion值的etcd中的键。

规格和状态
----------

最后，几乎每个顶级对象都有一个规范和一个状态部分。 该约定来自Kubernetes
API的声明性：规范是用户的需求，状态是该需求的结果，通常由系统中的控制器填充。有关Kubernetes中控制器的详细讨论，请参见\`\`控制器和操作员\'\'。

系统中的规范和状态约定只有少数例外-例如，核心组中的端点或RBAC对象（如ClusterRole）。

客户端集
========

在\`\`创建和使用客户端\'\'的介绍性示例中，我们看到kubernetes.NewForConfig（config）为我们提供了一个客户端集。客户端集为客户端提供了多个API组和资源的访问权限。对于来自k8s.io/client-go/kubernetes的kubernetes.NewForConfig（config），我们可以访问k8s.io/api中定义的所有API组和资源。这是Kubernetes
API服务器提供的全部资源，但有一些例外，例如APIServices（用于聚合的API服务器）和CustomResourceDefinition（请参见第4章）。

在第5章中，我们将解释如何从API类型（在本例中为k8s.io/api）实际生成这些客户端集。具有自定义API的第三方项目不仅仅使用Kubernetes客户端集。所有客户端集的共同点是REST配置（例如，如示例中那样，由clientcmd.BuildConfigFromFlags（""，\*
kubeconfig）返回）。

k8s.io/client-go/kubernetes/typed中的Kubernetes本地资源客户端设置主界面如下所示：

> **type** Interface **interface** {
>
> Discovery() discovery.DiscoveryInterface
>
> AppsV1() appsv1.AppsV1Interface
>
> AppsV1beta1() appsv1beta1.AppsV1beta1Interface
>
> AppsV1beta2() appsv1beta2.AppsV1beta2Interface
>
> AuthenticationV1() authenticationv1.AuthenticationV1Interface
>
> AuthenticationV1beta1()
> authenticationv1beta1.AuthenticationV1beta1Interface
>
> AuthorizationV1() authorizationv1.AuthorizationV1Interface
>
> AuthorizationV1beta1()
> authorizationv1beta1.AuthorizationV1beta1Interface
>
> \...
>
> }

此接口中曾经有未版本化的方法-例如，仅Apps（）appsv1.AppsV1Interface-但自基于Kubernetes
1.14的client-go 11.0起已弃用。
如前所述，对于应用程序使用的API组版本非常明确是一种很好的做法。

##### 过去版本的CLIENTS 和 INTERNAL CLIENTS 

过去，Kubernetes拥有所谓的内部客户端。它们对称为"内部"的对象使用了通用的内存版本，并与在线版本进行了转换。

希望从使用的实际API版本中提取控制器代码，并能够通过单行更改切换到另一个版本。在实践中，实施转换的巨大额外复杂性以及该转换代码需要有关对象语义的知识，导致得出这样的结论：这种模式不值得。

此外，Client与API
serve之间从来没有任何形式的自动协商。即使具有内部类型和客户端，也可以将控制器硬编码为线路上的特定版本。因此，在客户端和服务器之间版本偏斜的情况下，使用内部类型的控制器与使用版本化API类型的控制器之间的兼容性更高。

在最新的Kubernetes版本中，重写了大量代码以完全摆脱这些内部版本。如今，k8s.io
/ api中没有内部版本，k8s.io / client-go中也没有客户端可用。

每个客户端集还提供对发现客户端的访问权限（RESTMappers将使用它;请参阅\`\`REST映射\'\'和\`\`从命令行使用API\'\'）。

在每个GroupVersion方法（例如AppsV1beta1）的背后，我们找到了API组的资源-例如：

> **type** AppsV1beta1Interface **interface** {
>
> RESTClient() rest.Interface
>
> ControllerRevisionsGetter
>
> DeploymentsGetter
>
> StatefulSetsGetter
>
> }

RESTClient是通用的REST客户端，每个资源一个接口，如下所示：

> *// DeploymentsGetter具有一种返回DeploymentInterface的方法。*
>
> *//组的client应实现此接口。*
>
> **type** DeploymentsGetter **interface** {
>
> Deployments(namespace **string**) DeploymentInterface
>
> }
>
> // DeploymentInterface具有使用Deployment资源的方法。
>
> **type** DeploymentInterface **interface** {
>
> Create(\*v1beta1.Deployment) (\*v1beta1.Deployment, **error**)
>
> Update(\*v1beta1.Deployment) (\*v1beta1.Deployment, **error**)
>
> UpdateStatus(\*v1beta1.Deployment) (\*v1beta1.Deployment, **error**)
>
> Delete(name **string**, options \*v1.DeleteOptions) **error**
>
> DeleteCollection(options \*v1.DeleteOptions, listOptions
> v1.ListOptions) **error**
>
> Get(name **string**, options v1.GetOptions) (\*v1beta1.Deployment,
> **error**)
>
> List(opts v1.ListOptions) (\*v1beta1.DeploymentList, **error**)
>
> Watch(opts v1.ListOptions) (watch.Interface, **error**)
>
> Patch(name **string**, pt types.PatchType, data \[\]**byte**,
> subresources \...**string**)
>
> (result \*v1beta1.Deployment, err **error**)
>
> DeploymentExpansion
>
> }

根据资源的范围（即是集群作用域还是命名空间作用域），访问器（在这里为DeploymentGetter）可能有也可能没有命名空间参数。

部署接口可访问资源的所有受支持动词。它们中的大多数是不言自明的，但是接下来需要描述的内容将在下面进行描述。

状态子资源: UpdateStatus
------------------------

部署具有所谓的状态子资源。 这意味着UpdateStatus使用附加了/
status后缀的其他HTTP端点。 虽然/ apis / apps / v1beta1 / namespaces / ns
/ deployments / name端点上的更新只能更改部署的规格，但端点/ apis / apps
/ v1beta1 / namespaces / ns / deployments / name /
status只能更改对象的状态。
为了为规范更新（由人工完成）和状态更新（由控制器完成）设置不同的权限，这很有用。

默认情况下，client-gen（请参见"
client-gen标签"）生成UpdateStatus（）方法。
该方法的存在不能保证资源实际支持子资源。当我们在"子资源"中使用CRD时，这一点很重要。

清单和删除
----------

DeleteCollection允许我们一次删除命名空间的多个对象。使用ListOptions参数，我们可以使用字段或标签选择器定义应删除哪些对象：

> **type** ListOptions **struct** {
>
> *//一个选择器，用于通过其标签限制返回对象的列表。*
>
> *     //默认为所有内容。* *// +可选*
>
> LabelSelector **string** \`json:\"labelSelector,omitempty\"\`
>
> *//一个选择器，用于按其字段限制返回对象的列表。*
>
> *// 默认所有内容*
>
> *// +可选*
>
> FieldSelector **string** \`json:\"fieldSelector,omitempty\"\`}

Watches
-------

监视事件接口，了解对象的所有更改（添加，删除和更新）。
从k8s.io/apimachinery/pkg/watch返回的watch.Interface看起来像这样：

> *// Interface can be implemented by anything that knows how to watch
> and*
>
> *// report changes.*
>
> **type** Interface **interface** {
>
> *// Stops watching. Will close the channel returned by ResultChan().
> Releases*
>
> *// any resources used by the watch.*
>
> Stop()
>
> *// Returns a chan which will receive all the events. If an error
> occurs*
>
> *// or Stop() is called, this channel will be closed, in which case
> the*
>
> *// watch should be completely cleaned up.*
>
> ResultChan() \<-**chan** Event
>
> }

watch界面的结果通道返回三种事件：

> *// EventType defines the possible types of events.*
>
> **type** EventType **string**
>
> **const** (
>
> Added EventType = \"ADDED\"
>
> Modified EventType = \"MODIFIED\"
>
> Deleted EventType = \"DELETED\"
>
> Error EventType = \"ERROR\"
>
> )
>
> *// Event represents a single event to a watched resource.*
>
> *// +k8s:deepcopy-gen=true*
>
> **type** Event **struct** {
>
> Type EventType
>
> *// Object is:*
>
> *// \* If Type is Added or Modified: the new state of the object.*
>
> *// \* If Type is Deleted: the state of the object immediately before*
>
> *// deletion.*
>
> *// \* If Type is Error: \*api.Status is recommended; other types may*
>
> *// make sense depending on context.*
>
> Object runtime.Object
>
> }

虽然很想直接使用此接口，但实际上不建议使用通知者（请参见\`\`通知者和缓存\'\'）。
通知者是此事件接口和带有索引查找的内存缓存的组合。
这是迄今为止最常见的watch用例。
幕后告密者首先调用客户端上的List以获取所有对象的集合（作为缓存的基准），然后观看以更新缓存。
它们可以正确处理错误情况，即从网络问题或其他群集问题中恢复。

Client 拓展

DeploymentExpansion实际上是一个空接口。
它用于添加自定义客户端行为，但如今在Kubernetes中很少使用。
相反，客户端生成器允许我们以声明的方式添加自定义方法（请参见\`\`客户端标签\'\'）。

再次注意，DeploymentInterface中的所有这些方法都不希望在TypeMeta字段的Kind和APIVersion中获得有效的信息，也不能在Get（）和List（）上设置这些字段（另请参见\`\`TypeMeta\'\'）。
这些字段仅在导线上填充有实际值。

Client 选择
-----------

值得一看的是我们在创建客户集时可以设置的不同选项。
在\`\`版本和兼容性\'\'之前的注释中，我们看到可以为本地Kubernetes类型切换到protobuf有线格式。
Protobuf比JSON更有效（在空间上以及客户端和服务器的CPU负载方面），因此是可取的。

出于调试目的和指标的可读性，通常有助于区分访问API服务器的不同客户端。
为此，我们可以在REST配置中设置用户代理字段。 默认值为二进制/版本（os /
arch）kubernetes / commit; 例如，kubectl将使用kubectl / v1.14.0（darwin
/ amd64）kubernetes / d654b49这样的用户代理。
如果该模式不足以进行设置，则可以像这样自定义：

> cfg, err := clientcmd.BuildConfigFromFlags(\"\", \*kubeconfig)
>
> cfg.AcceptContentTypes =
> \"application/vnd.kubernetes.protobuf,application/json\"
>
> cfg.UserAgent = fmt.Sprintf(
>
> \"book-example/v1.0 (%s/%s) kubernetes/v1.0\",
>
> runtime.GOOS, runtime.GOARCH
>
> )
>
> clientset, err := kubernetes.NewForConfig(cfg)

REST配置中经常覆盖的其他值是客户端速率限制和超时的值：

> *// Config holds the common attributes that can be passed to a
> Kubernetes*
>
> *// client on initialization.*
>
> **type** Config **struct** {
>
> \...
>
> *// QPS indicates the maximum QPS to the master from this client.*
>
> *// If it\'s zero, the created RESTClient will use DefaultQPS: 5*
>
> QPS **float32**
>
> *// Maximum burst for throttle.*
>
> *// If it\'s zero, the created RESTClient will use DefaultBurst: 10.*
>
> Burst **int**
>
> *// The maximum length of time to wait before giving up on a server
> request.*
>
> *// A value of zero means no timeout.*
>
> Timeout time.Duration
>
> }

QPS值默认为每秒5个请求，突发数量为10。

超时没有默认值，至少在客户端REST配置中没有。 默认情况下，Kubernetes
API服务器将在60秒后使不是长时间运行请求的每个请求超时。长时间运行的请求可以是监视请求或对子资源（如/
exec，/ portforward或/ proxy）的无限制请求。

##### 平稳关机且避免连接错误

请求分为长期运行和非长期运行。Watches长时间运行，而GET，LIST，UPDATE等不是长时间运行的。许多子资源（例如，用于日志流，exec，端口转发）也在长时间运行。

重新启动Kubernetes
API服务器时（例如在更新过程中），它会等待长达60秒的时间才能正常关闭。在这段时间内，它会完成非长时间运行的请求，然后终止。当它终止时，诸如持续的Watches连接之类的长时间运行的请求将被切断。

无论如何，非长时间​​运行的请求都会受到60秒的限制（然后它们超时）。因此，从客户端的角度来看，关闭是正常的。

通常，应始终为不成功的请求准备应用程序代码，并且应以对应用程序无害的方式进行响应。在分布式系统的世界中，那些连接错误是正常现象，无需担心。但是需要特别注意以谨慎处理错误情况并从中恢复。

错误处理对于手表尤为重要。Watches运行时间很长，但它们随时可能失败。下一节中描述的通知程序提供了一种围绕监视的弹性实现，并且可以优雅地处理错误-也就是说，它们通过使用新连接从断开连接中恢复。应用程序代码通常不会发出通知。

通知和缓存
==========

\`client
set\'\'中的客户端界面包括Watch动词，该动词提供了一个事件接口，可对对象的更改（添加，删除，更新）做出反应。通知者针对Watch的最常见用例提供了更高级别的编程接口：内存中缓存以及按名称或内存中其他属性进行的对象的快速索引查找。

每次需要对象时访问API服务器的控制器都会给系统带来很大的负担。使用通知程序的内存中缓存是解决此问题的方法。
而且通知可以几乎实时地对对象的变化做出反应，而无需轮询请求。

图3-5显示了通知的概念模式; 具体来说，他们：

•从API服务器获取输入作为事件。

•提供类似于客户端的接口，称为Lister，以从内存缓存中获取和列出对象。

•注册事件处理程序以进行添加，删除和更新。

•使用存储实现内存中缓存。

![nformers](media/image5.png){width="5.763888888888889in"
height="1.739037620297463in"}

###### *图3-5. Informers*

通知还具有高级错误行为：长时间运行的监视连接中断时，他们通过尝试另一个监视请求从其恢复，从而拾取事件流而不会丢失任何事件。
如果中断时间很长，并且API服务器丢失了事件，因为在新监视请求成功之前etcd从其数据库中清除了事件，通知程序将重新列出所有对象。

在相对者旁边，有一个可配置的重新同步时间段，用于协调内存中缓存和业务逻辑：每次经过此时间段时，将为所有对象调用已注册的事件处理程序。
常用值以分钟为单位（例如10或30分钟）。

###### 警告

重新同步纯粹是在内存中，不会触发对服务器的调用。以前是不同的，但最终被更改了，因为监视机制的错误行为已得到足够的改进，因此不需要重新列出。

所有这些高级且经过战斗验证的错误行为是使用通知的好理由，而不是直接使用客户端Watch（）方法推出自定义逻辑。通知在Kubernetes本身中随处可见，并且是Kubernetes
API设计中的主要架构概念之一。

尽管通知者优先于轮询，但通知者会在API服务器上创建负载。每个GroupVersionResource一个二进制文件应仅实例化一个通知程序。为了使通知者共享变得容易，我们可以使用共享的通知者工厂实例化一个通知者。

共享告密工厂允许在应用程序中为同一资源共享informers。换句话说，不同的控制回路可以使用与引擎盖下的API服务器相同的监视连接。例如，kube-controller-manager是Kubernetes集群的主要组件之一（请参见\`\`API服务器\'\'），具有更大的两位数的控制器。但是，对于每种资源（例如Pod），在此过程中只有一个informers。

###### TIP

始终使用共享的通知者工厂实例化通知者。 不要尝试手动实例化通知者。
开销是最小的，并且不使用共享通知者的非平凡的控制器二进制文件可能正在某个地方为同一资源打开多个监视连接。

从REST配置开始（请参阅\`\`创建和使用客户端\'\'），使用客户端集轻松创建共享告密工厂。
通知程序由代码生成器生成，并作为k8s.io/client-go/informers中标准Kubernetes资源的client-go的一部分提供：

> **import** (
>
> \...
>
> \"k8s.io/client-go/informers\"
>
> )
>
> \...
>
> clientset, err := kubernetes.NewForConfig(config)
>
> informerFactory := informers.NewSharedInformerFactory(clientset,
> time.Second\*30)
>
> podInformer := informerFactory.Core().V1().Pods()
>
> podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
>
> AddFunc: **func**(new **interface**{}) {\...},
>
> UpdateFunc: **func**(old, new **interface**{}) {\...},
>
> DeleteFunc: **func**(obj **interface**{}) {\...},
>
> })
>
> informerFactory.Start(wait.NeverStop)
>
> informerFactory.WaitForCacheSync(wait.NeverStop)
>
> pod, err :=
> podInformer.Lister().Pods(\"programming-kubernetes\").Get(\"client-go\")

该示例说明了如何获取pod的共享通知者。

您可以看到informers允许添加，更新和删除三种情况的事件处理程序。
这些通常用于触发控制器的业务逻辑-即再次处理某个对象（请参阅\`\`控制器和操作员\'\'）。
通常，这些处理程序只是将修改后的对象添加到工作队列中。

#####  其他事件处理程序和内部存储更新逻辑

不要将这些处理程序与通知程序内部的内存存储的更新逻辑（可通过示例最后一行的列表器访问）混淆。
通知者将始终更新其商店，但是刚刚描述的其他事件处理程序是可选的，并且打算由通知者的消费者使用。

另请注意，可以添加许多事件处理程序。整个共享告密工厂的概念之所以存在，仅是因为这在具有许多控制循环的控制器二进制文件中如此普遍，每个二进制安装事件处理程序以将对象添加到自己的工作队列中。

注册处理程序后，必须启动共享的通知者工厂。幕后有Go例程，可以对API服务器进行实际调用。启动方法（具有控制生命周期的停止通道）启动这些Go例程，WaitForCacheSync（）方法使代码等待对客户端的第一个List调用完成。如果控制器逻辑要求填充缓存，则此WaitForCacheSync调用必不可少。

通常，watch背后的事件接口会导致一定的滞后。在具有适当容量规划的设置中，此延迟并不大。当然，优良作法是使用指标来测量此滞后。但是滞后无论如何都存在，因此应用程序逻辑的构建方式使得滞后不会损害代码的行为。

###### 警告

通知者的滞后可能导致控制器在客户端直接通过API服务器在客户端上进行的更改与通知者所知的世界状况之间产生竞争。

如果控制器更改了对象，则同一过程中的通知者必须等待，直到相应的事件到达，然后更新内存中的存储。
该过程不是瞬时的，在以前的更改变为可见之前，可能会通过另一个触发器启动另一个控制器工作循环运行。

在此示例中，重新同步间隔为30秒会导致将一组完整的事件发送到已注册的UpdateFunc，以便控制器逻辑能够将其状态与API服务器的状态进行协调。
通过比较ObjectMeta.resourceVersion字段，可以将实际更新与重新同步区分开。

##### 不改变 INFORMERS对象

重要的是要记住，从列表程序传递到事件处理程序的任何对象均由通知程序拥有。
如果您以任何方式对其进行变异，则可能会在应用程序中引入难以调试的缓存一致性问题。
更改对象之前，请务必进行深度复制（请参阅\`\`Go中的Kubernetes对象\'\'）。

通常：在更改对象之前，请始终问自己是谁拥有该对象或其中的数据结构。
根据经验：

•
Informers和列表者拥有返回的对象。因此，消费者必须在突变之前进行深度复制。

•客户端返回调用方拥有的新对象。

•转换返回共享对象。如果调用者确实拥有输入对象，则它不拥有输出。

在示例中，通知程序构造函数NewSharedInformerFactory在存储区的所有命名空间中缓存资源的所有对象。
如果这对于应用程序来说太多了，那么可以使用另一种具有更大灵活性的构造函数：

> *// NewFilteredSharedInformerFactory constructs a new instance of*
>
> *// sharedInformerFactory. Listers obtained via this
> sharedInformerFactory will be*
>
> *// subject to the same filters as specified here.*
>
> **func** NewFilteredSharedInformerFactory(
>
> client versioned.Interface, defaultResync time.Duration,
>
> namespace **string**,
>
> tweakListOptions internalinterfaces.TweakListOptionsFunc
>
> ) SharedInformerFactor
>
> **type** TweakListOptionsFunc **func**(\*v1.ListOptions)

它允许我们指定一个名称空间并传递一个TweakListOptionsFunc，它可以使用客户端的List和Watch调用来改变用于列出和监视对象的ListOptions结构。
例如，它可用于设置标签或字段选择器。

告密者是控制器的组成部分之一。
在第6章中，我们将看到典型的基于客户端运行的控制器的外观。
在client和informers之后，第三个主要组成部分是工作队列。
让我们现在来看它。

工作队列
--------

工作队列是一种数据结构。您可以按照队列预定义的顺序添加元素并将元素从队列中取出。正式地，这种队列称为优先级队列。
client-go在k8s.io/client-go/util/workqueue中提供了强大的实现以构建控制器。

更准确地说，该包装包含许多用于不同目的的变体。所有变体实现的基本接口如下所示：

> **type** Interface **interface** {
>
> Add(item **interface**{})
>
> Len() **int**
>
> Get() (item **interface**{}, shutdown **bool**)
>
> Done(item **interface**{})
>
> ShutDown()
>
> ShuttingDown() **bool**
>
> }

在这里Add（item）添加一个项目，Len（）给出长度，Get（）返回具有最高优先级的项目（并且阻塞直到有一个可用）。
当控制器完成处理后，Get（）返回的每个项目都需要一个完成（item）调用。
同时，重复的Add（item）仅将项目标记为脏项，以便在调用Done（item）时将其读取。

以下队列类型从该通用接口派生：

•DelayingInterface可以在以后添加一个项目。
这样可以在失败后更轻松地重新排队项目，而不会出现在热循环中：

-   **type** DelayingInterface **interface** {

-   Interface

-   *// AddAfter adds an item to the workqueue after the*

-   *// indicated duration has passed.*

-   AddAfter(item **interface**{}, duration time.Duration)

> }

-   RateLimitingInterface rate-limits items being added to the queue. It
    extends the DelayingInterface:

-   **type** RateLimitingInterface **interface** {

-   DelayingInterface

-   

-   *// AddRateLimited adds an item to the workqueue after the rate*

-   *// limiter says it\'s OK.*

-   AddRateLimited(item **interface**{})

-   

-   *// Forget indicates that an item is finished being retried.*

-   *// It doesn\'t matter whether it\'s for perm failing or success;*

-   *// we\'ll stop the rate limiter from tracking it. This only clears*

-   *// the \`rateLimiter\`; you still have to call \`Done\` on the
    > queue.*

-   Forget(item **interface**{})

-   

-   *// NumRequeues returns back how many times the item was requeued.*

NumRequeues(item **interface**{}) **int**

此处最有趣的是\`\`忘记（项目）\'\'方法：它重置给定项目的退避。通常，将在成功处理项目后调用它。

速率限制算法可以传递给构造函数NewRateLimitingQueue。
同一包中定义了多个速率限制器，例如BucketRateLimiter，ItemExponentialFailureRateLimiter，ItemFastSlowRateLimiter和MaxOfRateLimiter。
有关更多详细信息，请参阅软件包说明文件。
大多数控制器只会使用DefaultControllerRateLimiter（）\*
RateLimiter函数，该函数提供：

♣从5毫秒开始的指数补偿，直至1000秒，使每个错误的延迟加倍

♣最高每秒10个项目和100个项目的爆发率

根据上下文，您可能需要自定义值。
对于某些控制器应用程序，每项最大1000秒的退避时间很多。

API Machinery in Depth
======================

API
Machinery存储库实现了Kubernetes类型系统的基础。但是这种类型的系统到底是什么？
首先是什么类型？

API Machinery术语中实际上不存在术语类型。相反，它指的是种类。

Kinds
-----

正如我们在" API术语"中已经看到的那样，种类分为API组和版本。 因此，API
Machinery存储库中的核心术语是GroupVersionKind或简称为GVK。

在Go中，每个GVK对应一个Go类型。 相反，Go类型可以属于多个GVK。

kind不会正式一对一映射到HTTP路径。 许多类型具有HTTP
REST端点，用于访问给定类型的对象。
但也有一些没有任何HTTP端点的类型（例如，admission.k8s.io/v1beta1.AdmissionReview，用于调出Webhook）。
许多端点还返回了一些种类，例如meta.k8s.io/v1.Status，所有端点都返回了它们以报告非对象状态（例如错误）。

按照惯例，种类会以CamelCase之类的格式格式化，通常是单数形式。
根据上下文，它们的具体格式会有所不同。
对于CustomResourceDefinition类型，它必须是DNS路径标签（RFC 1035）。

Resources
---------

正如我们在\`\`API术语\'\'中所看到的，与种类并行的是资源的概念。
资源再次进行分组和版本控制，从而导致术语GroupVersionResource或简称为GVR。

每个GVR对应一个HTTP（基本）路径。 GVR用于标识Kubernetes API的REST端点。
例如，GVR apps / v1.deployments映射到/ apis / apps / v1 / namespaces /
namespace / deployments。

客户端库使用此映射来构造访问GVR的HTTP路径。

##### 了解资源命名和集群分布

您必须知道GVR是命名空间还是群集作用域才能知道HTTP路径。
例如，部署具有命名空间，因此将名称空间作为其HTTP路径的一部分。
其他GVR，例如rbac.authorization.k8s.io/v1.clusterroles，属于群集作用域；例如，可以在apis
/ rbac.authorization.k8s.io / v1 / clusterroles中访问群集角色

按照惯例，资源是小写和复数形式，通常对应于并行种类的复数个单词。
它们必须符合DNS路径标签格式（RFC 1025）。
由于资源直接映射到HTTP路径，这不足为奇。

REST Mapping
------------

GVK到GVR的映射称为REST映射。

RESTMapper是Golang接口，使我们能够为GVK请求GVR：

> RESTMapping(gk schema.GroupKind, versions \...**string**)
> (\*RESTMapping, **error**)

右侧的类型RESTMapping如下所示：

> **type** RESTMapping **struct** {
>
> *// Resource is the GroupVersionResource (location) for this
> endpoint.*
>
> Resource schema.GroupVersionResource.
>
> *// GroupVersionKind is the GroupVersionKind (data format) to submit*
>
> *// to this endpoint.*
>
> GroupVersionKind schema.GroupVersionKind
>
> *// Scope contains the information needed to deal with REST Resources*
>
> *// that are in a resource hierarchy.*
>
> Scope RESTScope
>
> }

此外，RESTMapper提供了许多便利功能

> *// KindFor takes a partial resource and returns the single match.*
>
> *// Returns an error if there are multiple matches.*
>
> KindFor(resource schema.GroupVersionResource)
> (schema.GroupVersionKind, **error**)
>
> *// KindsFor takes a partial resource and returns the list of
> potential*
>
> *// kinds in priority order.*
>
> KindsFor(resource schema.GroupVersionResource)
> (\[\]schema.GroupVersionKind, **error**)
>
> *// ResourceFor takes a partial resource and returns the single
> match.*
>
> *// Returns an error if there are multiple matches.*
>
> ResourceFor(input schema.GroupVersionResource)
> (schema.GroupVersionResource, **error**)
>
> *// ResourcesFor takes a partial resource and returns the list of
> potential*
>
> *// resource in priority order.*
>
> ResourcesFor(input schema.GroupVersionResource)
> (\[\]schema.GroupVersionResource, **error**)
>
> *// RESTMappings returns all resource mappings for the provided group
> kind*
>
> *// if no version search is provided. Otherwise identifies a preferred
> resource*
>
> *// mapping for the provided version(s).*
>
> RESTMappings(gk schema.GroupKind, versions \...**string**)
> (\[\]\*RESTMapping, **error**)

在此，部分GVR表示未设置所有字段。 例如，假设您键入Kubectl get pods。
在这种情况下，组和版本将丢失。具有足够信息的RESTMapper可能仍设法将其映射到v1
Pod类型。

对于前面的部署示例，一个知道部署的RESTMapper（稍后会更多地了解这意味着什么）将把apps
/ v1.Deployment映射为apps / v1.deployments作为命名空间资源。

RESTMapper接口有多种不同的实现。
对于客户端应用程序而言，最重要的一个是在软件包k8s.io/client-go/restmapper中基于发现的DeferredDiscoveryRESTMapper：它使用Kubernetes
API服务器中的发现信息来动态构建REST映射。
它还将与非核心资源（如自定义资源）一起使用。

Scheme
------

我们想在Kubernetes类型系统的上下文中提出的最终核心概念是软件包k8s.io/apimachinery/pkg/runtime中的方案。

一个方案将Golang的世界与GVK的与实现无关的世界联系起来。方案的主要特征是将Golang类型映射到可能的GVK：

> **func** (s \*Scheme) ObjectKinds(obj Object)
> (\[\]schema.GroupVersionKind, **bool**, **error**)

如我们在\`\`Go中的Kubernetes对象\'\'中所见，对象可以通过GetObjectKind（）模式返回对象的组和种类.ObjectKind方法。
但是，这些值大多数时候都是空的，因此对于识别几乎没有用。

相反，该方案通过反射获取给定对象的Golang类型，并将其映射到该Golang类型的已注册GVK。
为了使它正常工作，必须将Golang类型注册到该方案中，如下所示：

> **func** (s \*Scheme) ObjectKinds(obj Object)
> (\[\]schema.GroupVersionKind, **bool**, **error**)

该方案不仅用于注册Golang类型及其GVK，还用于存储转换函数和默认值的列表（参见图3-6）。
我们将在第8章中详细讨论转换和默认值。它也是实现编码器和解码器的数据源。

![he scheme, connecting Golang data types with the GVK, conversions and
defaulters](media/image6.png){width="5.763888888888889in"
height="5.3023676727909015in"}

###### *图3-6. scheme* *Golang数据类型与GVK，转换和默认值连接*

对于Kubernetes核心类型，软件包k8s.io/client-go/kubernetes/scheme中的client-go
client set中有一个预定义的方案，所有类型都已预先注册。
实际上，由client-gen代码生成器生成的每个客户端集（请参见第5章）都具有子包方案，该客户端包中的所有类型和类型都属于该客户端集。

通过该方案，我们结束了对API机械概念的深入研究。
如果您只记得关于这些概念的一件事，那就如图3-7所示

![rom Golang types to GVKs to GVRs to an HTTP path --- API machinery in
a ](media/image7.png){width="4.058333333333334in"
height="5.930555555555555in"}

###### *图3-7. 从Golang类型到GVK，再到GVR到HTTP路径-简而言之，API机制*

供应商
======

我们已经在本章中看到k8s.io/client-go、k8s.io/api和k8s.io/apimachinery是Golang
Kubernetes编程的核心。
Golang使用供应商将这些库包含在第三方应用程序源代码存储库中。

供应商是Golang社区的目标。在撰写本文时，有几种通用的销售工具，例如godeps，dep和glide。同时，Go
1.12获得了对Go模块的支持，该模块将来可能会成为Go社区中的标准供应商方法，但目前尚未在Kubernetes生态系统中准备就绪。

当今大多数项目都使用dep或glide。
Kubernetes本身在github.com/kubernetes/kubernetes中跳转到了1.15开发周期的Go模块。以下注释与所有这些供应商工具有关。

每个k8s.io/\*存储库中受支持的依赖项版本的真实来源是出厂的Godeps /
Godeps.json文件。需要强调的是，任何其他依赖项选择都可能破坏库的功能。

有关k8s.io/client-go、k8s.io/api和k8s.io/apimachinery的已发布标签以及哪些标签相互兼容的更多信息，请参见\`\`客户端库\'\'。

glide
-----

使用glide的项目可以使用其在任何依赖项更改时读取Godeps /
Godeps.json文件的功能。
事实证明，这非常可靠：开发人员只需声明正确的k8s.io/client-go版本，并且glide将选择正确的k8s.io/apimachinery、k8s.io/api版本以及其他依赖项。

对于GitHub上的某些项目，glide.yaml文件可能如下所示：

> **package**: github.com/book/example
>
> **import**:
>
> \- **package**: k8s.io/client-go
>
> **version**: v10.0.0
>
> **\...**

这样，glide install
-v将把k8s.io/client-go及其依赖项下载到本地供应商/软件包中。
这里-v表示从供应商库中删除供应商/软件包。 这是我们需要的。

如果您通过编辑glide.yaml更新到新版本的client-go，则glide update
-v将再次以正确的版本下载新的依赖项。

dep
---

dep通常被认为比glide更强大，更先进。
长期以来，它被视为在生态系统中滑行的继任者，似乎注定是Go供应商工具。
在撰写本文时，它的未来还不确定，Go模块似乎是前进的道路。

在client-go的上下文中，了解dep的两个限制非常重要：

•dep在首次运行dep init时会读取Godeps / Godeps.json。

•dep在以后不读取Godeps / Godeps.json，请确保-update调用。

这意味着当在Godep.toml中更新client-go版本时，client-go依赖项的解析很可能是错误的。
这是不幸的，因为它要求开发人员显式且通常手动声明所有依赖项。

一个有效且一致的Godep.toml文件如下所示：

> \[\[constraint\]\]
>
> name = \"k8s.io/api\"
>
> version = \"kubernetes-1.13.0\"
>
> \[\[constraint\]\]
>
> name = \"k8s.io/apimachinery\"
>
> version = \"kubernetes-1.13.0\"
>
> \[\[constraint\]\]
>
> name = \"k8s.io/client-go\"
>
> version = \"10.0.0\"
>
> \[prune\]
>
> go-tests = true
>
> unused-packages = true
>
> \# the following overrides are necessary to enforce
>
> \# the given version, even though our
>
> \# code does not import the packages directly.
>
> \[\[override\]\]
>
> name = \"k8s.io/api\"
>
> version = \"kubernetes-1.13.0\"
>
> \[\[override\]\]
>
> name = \"k8s.io/apimachinery\"
>
> version = \"kubernetes-1.13.0\"
>
> \[\[override\]\]
>
> name = \"k8s.io/client-go\"
>
> version = \"10.0.0\"

###### 警告

Gopkg.toml不仅为k8s.io/apimachinery和k8s.io/api都声明了显式版本，还为它们提供了替代项。
对于启动项目时没有从这两个存储库中显式导入软件包的情况，这是必需的。
在这种情况下，没有这些覆盖，dep将在一开始就忽略约束，而开发人员将从一开始就获得错误的依赖关系。

即使此处显示的Gopkg.toml文件在技术上也是不正确的，因为它不完整，因为它没有声明对client-go所需的所有其他库的依赖关系。
过去，上游库破坏了client-go的编译。
因此，如果您使用dep进行依赖项管理，请做好准备。

Go Modules
----------

Go模块是Golang中依赖管理的未来。 它们在Go
1.11中引入了初步支持，并在1.12中进一步稳定下来。 通过设置GO111MODULE =
on环境变量，许多命令（例如go run和go get）与Go模块一起使用。 在Go
1.13中，这是默认设置。

Go模块由项目根目录中的go.mod文件驱动。
这是第8章中github.com/programming-kubernetes/pizza-apiserver项目的go.mod文件的摘录：

> module github.com/programming-kubernetes/pizza-apiserver
>
> require (
>
> \...
>
> k8s.io/api v0.0.0-20190222213804-5cb15d344471 // indirect
>
> k8s.io/apimachinery v0.0.0-20190221213512-86fb29eff628
>
> k8s.io/apiserver v0.0.0-20190319190228-a4358799e4fe
>
> k8s.io/client-go
> v2.0.0-alpha.0.0.20190307161346-7621a5ebb88b+incompatible
>
> k8s.io/klog v0.2.1-0.20190311220638-291f19f84ceb
>
> k8s.io/kube-openapi v0.0.0-20190320154901-c59034cc13d5 // indirect
>
> k8s.io/utils v0.0.0-20190308190857-21c4ce38f2a7 // indirect
>
> sigs.k8s.io/yaml v1.1.0 // indirect
>
> )

client-go v11.0.0-与Kubernetes
1.14匹配-以及较旧的版本不明确支持Go模块。如上例所示，仍然可以将Go模块与Kubernetes库一起使用。

只要（client-go）和其他Kubernetes存储库不附带go.mod文件（至少在Kubernetes
1.15之前），就必须手动选择正确的版本。也就是说，您需要在client-go中与Godeps
/ Godeps.json的依赖项修订版本相匹配的所有依赖项的完整列表。

还要注意上一个示例中不太可读的修订。它们是从现有标签派生的伪版本，如果没有标签，则使用v0.0.0作为前缀。更糟糕的是，您可以在该文件中引用标记的版本，但是Go模块命令将在下次运行时将其替换为伪版本。

使用client-go v12.0.0（与Kubernetes
1.15匹配），我们提供了一个go.mod文件，并弃用了对所有其他供应商工具的支持（请参阅相应的建议文档）。出厂的go.mod文件包含所有依赖关系，您的项目go.mod文件不再需要手动列出所有传递依赖关系。在更高版本中，也可能会更改标记方案以修复丑陋的伪修订并将其替换为适当的模拟器标记。但是，在撰写本文时，这还没有完全实现或确定。

致谢
====

在本章中，我们的重点是Go中的Kubernetes编程接口。
我们讨论了访问知名核心类型的Kubernetes
API（即，每个Kubernetes群集附带的API对象）。

到此为止，我们已经介绍了Kubernetes API的基础知识及其在Go语言中的表示。
现在，我们准备继续讨论自定义资源这一主题，它是运营商的支柱之一。

1参见\`\`API术语\'\'。

2个Kubectl解释Pod，可让您向API服务器查询对象的架构，包括字段文档。
