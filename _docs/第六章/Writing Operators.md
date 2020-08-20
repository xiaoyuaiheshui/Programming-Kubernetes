第六章 6. Writing Operators的解决方案
=====================================

到目前为止，我们已经在\`\` Controllers and
Operators\'\'的概念级别上研究了自定义controllers和Operators，并在第5章中介绍了如何使用Kubernetes代码生成器-一种相当低级的方式来处理该主题。
在本章中，我们将详细介绍三种用于编写自定义控制器和运算符的解决方案，并讨论更多替代方案。

使用本章中讨论的解决方案之一可以帮助您避免编写大量重复的代码，并使您能够专注于业务逻辑，而不是样板代码。它可以帮助您更快地开始工作，并使您的工作效率更高。

###### 注意

截至2019年中期，总体而言，运营商以及我们在本章中专门讨论的工具仍在迅速发展。当我们尽力而为时，此处显示的某些命令和/或其输出可能会更改。
考虑到这一点，并确保您始终使用相应工具的最新版本，同时注意相应的问题跟踪器，邮件列表和Slack频道。

虽然在线上有可用的资源可以与我们在此处讨论的解决方案进行比较，但我们不会为您推荐特定的解决方案。但是，我们鼓励您自己进行评估和比较，并选择最适合您的组织和环境的方案。

Preparation
===========

我们将使用cnat（本机云，在"A Motivational
Example"中介绍过）作为本章中不同解决方案的运行示例。如果您想继续，请注意我们假设您：

1.已安装Go版本1.12或更高版本并正确设置。

2.可以访问本地版本1.12或更高版本的Kubernetes群集-例如通过Kind或k3d本地访问，或通过您最喜欢的云提供商远程访问-并且已配置Kubectl对其进行访问。

3\. git clone
GitHub存储库。此处提供了以下各节中显示的完整，有效的源代码和必要的命令。请注意此处显示的是一切从头开始的工作方式。如果您想查看结果而不是自己执行步骤，也欢迎克隆存储库并仅运行命令来安装CRD，安装CR和启动自定义控制器。

整理完这些管家用品后，让我们开始编写operators：我们将sample-controller,，Kubebuilder和Operator
SDK

开始吧- pun intended!

Following sample-controller
===========================

让我们从基于k8s.io/sample-controller的cnat实现开始，它直接使用client-go库。样本控制器使用k8s.io/code-generator生成类型化的客户端，通知程序，列表程序和深度复制功能。
每当您的自定义controller中的API类型发生更改时（例如在自定义资源中添加新字段），您都必须使用update-codegen.sh脚本（另请参见GitHub中的源代码）来重新生成上述源文件。

###### 警告

您可能已经注意到k8s.io被用作整本书的基本URL。 我们在第3章介绍了它的用法;
提醒一下，它实际上是kubernetes.io的别名，在Go软件包管理的上下文中，它解析为github.com/kubernetes。
请注意，k8s.io没有自动重定向。 因此，例如，k8s.io /
sample-controller确实意味着您应该查看github.com/kubernetes/sample-controller等。

好的，让我们在示例控制器之后使用client-go实现我们的CNat operator。
（请参阅我们仓库中的相应目录。）

Bootstrapping
-------------

首先go get
k8s.io/sample-controller获取系统的源代码和依赖项，该文件应位于\$ GOPATH
/ src / k8s.io / sample- \\ controller中。

如果您是从头开始的，请将sample-controller目录的内容复制到您选择的目录中（例如，我们在仓库中使用cnat-client-go），然后可以运行以下命令序列来构建和运行
基本控制器（具有默认实现，尚未使用CNAT业务逻辑）：

> *\# build custom controller binary:*
>
> \$ go build -o cnat-controller .
>
> *\# launch custom controller locally:*
>
> \$ ./cnat-controller -kubeconfig=\$HOME/.kube/config

此命令将启动定制控制器，并等待您注册CRD并创建定制资源。
让我们现在做，看看会发生什么。 在第二个终端会话中，输入：

> \$ kubectl apply -f artifacts/examples/crd.yaml

确保CRD已正确注册并可用，如下所示：

> \$ kubectl get crds
>
> NAME CREATED AT
>
> foos.samplecontroller.k8s.io 2019-05-29T12:16:57Z

请注意，您可能会在这里看到其他CRD，具体取决于您所使用的Kubernetes发行版。但是，至少应列出foos.samplecontroller.k8s.io。

接下来，我们创建示例自定义资源foo.samplecontroller.k8s.io/example-foo并检查controller是否完成其工作：

> \$ kubectl apply -f artifacts/examples/example-foo.yaml
>
> foo.samplecontroller.k8s.io/example-foo created
>
> \$ kubectl get po,rs,deploy,foo
>
> NAME READY STATUS RESTARTS AGE
>
> pod/example-foo-5b8c9679d8-xjhdf 1/1 Running 0 67s
>
> NAME DESIRED CURRENT READY AGE
>
> replicaset.extensions/example-foo-5b8c9679d8 1 1 1 67s
>
> NAME READY UP-TO-DATE AVAILABLE AGE
>
> deployment.extensions/example-foo 1/1 1 1 67s
>
> NAME AGE
>
> foo.samplecontroller.k8s.io/example-foo 67s

是的，它按预期工作！ 现在我们可以继续实施特定于CNAT的实际业务逻辑。

Business Logic
--------------

为了开始实施business logic，我们首先将现有目录pkg / apis /
samplecontroller重命名为pkg / apis /
cnat，然后创建我们自己的CRD和自定义资源，如下所示：

> \$ cat artifacts/examples/cnat-crd.yaml
>
> apiVersion: apiextensions.k8s.io/v1beta1
>
> kind: CustomResourceDefinition
>
> metadata:
>
> name: ats.cnat.programming-kubernetes.info
>
> spec:
>
> group: cnat.programming-kubernetes.info
>
> version: v1alpha1
>
> names:
>
> kind: At
>
> plural: ats
>
> scope: Namespaced
>
> \$ cat artifacts/examples/cnat-example.yaml
>
> apiVersion: cnat.programming-kubernetes.info/v1alpha1
>
> kind: At
>
> metadata:
>
> labels:
>
> controller-tools.k8s.io: \"1.0\"
>
> name: example-at
>
> spec:
>
> schedule: \"2019-04-12T10:12:00Z\"
>
> command: \"echo YAY\"

请注意，每当API类型更改时（例如，当您向At
CRD添加新字段时），您都必须执行update-codegen.sh脚本，如下所示：

> \$ ./hack/update-codegen.sh

这将自动生成以下内容：

-   *pkg/apis/cnat/v1alpha1/zz_generated.deepcopy.go*

-   *pkg/generated/\**

在business logic方面，我们需要在operator中实现两个部分：

•在types.go中，我们修改AtSpec
struct以包括相应的字段，例如schedule和command。请注意，无论何时在此处进行更改，都必须运行update-codegen.sh以重新生成相关文件。

•在controller.go中，我们更改了NewController（）和syncHandler（）函数，并添加了辅助函数，包括创建容器和检查计划时间。

在types.go中，注意代表资源三个阶段的三个常数：直到PENDING中的计划时间，然后从RUNNING到完成，最后处于DONE状态：

> *// +genclient*
>
> *//
> +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object*
>
> **const** (
>
> PhasePending = \"PENDING\"
>
> PhaseRunning = \"RUNNING\"
>
> PhaseDone = \"DONE\"
>
> )
>
> *// AtSpec defines the desired state of At*
>
> **type** AtSpec **struct** {
>
> *// Schedule is the desired time the command is supposed to be
> executed.*
>
> *// Note: the format used here is UTC time https://www.utctime.net*
>
> Schedule **string** \`json:\"schedule,omitempty\"\`
>
> *// Command is the desired command (executed in a Bash shell) to be*
>
> *// executed.*
>
> Command **string** \`json:\"command,omitempty\"\`
>
> }
>
> *// AtStatus defines the observed state of At*
>
> **type** AtStatus **struct** {
>
> *// Phase represents the state of the schedule: until the command is*
>
> *// executed it is PENDING, afterwards it is DONE.*
>
> Phase **string** \`json:\"phase,omitempty\"\`
>
> }

请注意构建标记+
k8s：deepcopy-gen：interfaces的显式用法（请参阅第5章），以便自动生成各个源。

现在，我们可以实现自定义控制器的业务逻辑。也就是说，我们在controller.go中实现了从PhasePending到PhaseRunning到PhaseDone这三个阶段之间的状态转换。

在\`\` Work Queue\'\'中，我们介绍并解释了client去提供的Work Queue。
现在，我们可以运用这些知识了：在controller.go的processNextWorkItem（）中-更准确地说，在176至186行中，您可以找到以下（生成的）代码：

> **if** when, err := c.syncHandler(key); err != **nil** {
>
> c.workqueue.AddRateLimited(key)
>
> **return** fmt.Errorf(\"error syncing \'%s\': %s, requeuing\", key,
> err.Error())
>
> } **else** **if** when != time.Duration(0) {
>
> c.workqueue.AddAfter(key, when)
>
> } **else** {
>
> *// Finally, if no error occurs we Forget this item so it does not*
>
> *// get queued again until another change happens.*
>
> c.workqueue.Forget(obj)
>
> }

该代码段显示了如何调用（尚待编写）自定义syncHandler（）函数（稍后说明），并涵盖了以下三种情况：

1.第一个if分支通过AddRateLimited（）函数调用使该项目重新排队，以处理瞬态错误。

2.第二个分支else
if通过AddAfter（）函数调用重新排队该项目，以避免热循环。

3.最后一种情况是其他情况，该项已成功处理，并通过Forget（）函数调用将其丢弃。

现在，我们对通用处理有了充分的了解，让我们继续进行特定于业务逻辑的功能。
关键是前面提到的syncHandler（）函数，我们在其中实现自定义控制器的业务逻辑。
它具有以下签名：

> *// syncHandler将实际状态与所需状态进行比较并尝试*
>
> *//将两者融合。 然后，它更新At资源的Status块*
>
> *//以及资源的当前状态。 返回等待时间*
>
> *//直到时间表到期为止。*
>
> **func** (c \*Controller) syncHandler(key **string**) (time.Duration,
> **error**) {
>
> \...
>
> }
>
> 这个syncHandler（）函数实现以下状态转换：1\...
>
> *//如果未设置任何阶段，则默认为pending（初始阶段）：*
>
> **if** instance.Status.Phase == \"\" {
>
> instance.Status.Phase = cnatv1alpha1.PhasePending
>
> }
>
> */现在让我们区分主要情况：实现*
>
> *//状态图PENDING-\> RUNNING-\> DONE*
>
> **switch** instance.Status.Phase {
>
> **case** cnatv1alpha1.PhasePending:
>
> klog.Infof(\"instance %s: phase=PENDING\", key)
>
> *//只要还没有执行命令，就需要*
>
> *//检查是否该执行了：*
>
> klog.Infof(\"instance %s: checking schedule %q\", key,
> instance.Spec.Schedule)
>
> *// /检查是否已经是时候使用*
>
> *// 2秒的容限：*

d, err := timeUntilSchedule(instance.Spec.Schedule)

> **if** err != **nil** {
>
> utilruntime.HandleError(fmt.Errorf(\"schedule parsing failed: %v\",
> err))
>
> *//读取*schedule*时出错-重新排队请求：*
>
> **return** time.Duration(0), err
>
> }
>
> klog.Infof(\"instance %s: schedule parsing done: diff=%v\", key, d)
>
> **if** d \> 0 {
>
> *//还没有时间执行命令，请等待*
>
> *         //* schedule *time*
>
> **return** d, **nil**
>
> }
>
> klog.Infof(
>
> \"instance %s: it\'s time! Ready to execute: %s\", key,
>
> instance.Spec.Command,
>
> )
>
> instance.Status.Phase = cnatv1alpha1.PhaseRunning
>
> **case** cnatv1alpha1.PhaseRunning:
>
> klog.Infof(\"instance %s: Phase: RUNNING\", key)
>
> pod := newPodForCR(instance)
>
> *//将At实例设置为own和controller*
>
> owner := metav1.NewControllerRef(
>
> instance, cnatv1alpha1.SchemeGroupVersion.
>
> WithKind(\"At\"),
>
> )
>
> pod.ObjectMeta.OwnerReferences =
> append(pod.ObjectMeta.OwnerReferences, \*owner)
>
> *//尝试查看容器是否已经存在，如果不存在*
>
> *//（我们期望），然后根据规格创建一个一次性Pod：*
>
> found, err := c.kubeClientset.CoreV1().Pods(pod.Namespace).
>
> Get(pod.Name, metav1.GetOptions{})
>
> **if** err != **nil** && errors.IsNotFound(err) {
>
> found, err = c.kubeClientset.CoreV1().Pods(pod.Namespace).Create(pod)
>
> **if** err != **nil** {
>
> **return** time.Duration(0), err
>
> }
>
> klog.Infof(\"instance %s: pod launched: name=%s\", key, pod.Name)
>
> } **else** **if** err != **nil** {
>
> *// requeue with error*
>
> **return** time.Duration(0), err
>
> } **else** **if** found.Status.Phase == corev1.PodFailed \|\|
>
> found.Status.Phase == corev1.PodSucceeded {
>
> klog.Infof(
>
> \"instance %s: container terminated: reason=%q message=%q\",
>
> key, found.Status.Reason, found.Status.Message,
>
> )
>
> instance.Status.Phase = cnatv1alpha1.PhaseDone
>
> } **else** {
>
> *// Don\'t requeue because it will happen automatically*
>
> *// when the pod status changes.*
>
> **return** time.Duration(0), **nil**
>
> }
>
> **case** cnatv1alpha1.PhaseDone:
>
> klog.Infof(\"instance %s: phase: DONE\", key)
>
> **return** time.Duration(0), **nil**
>
> **default**:
>
> klog.Infof(\"instance %s: NOP\")
>
> **return** time.Duration(0), **nil**
>
> }
>
> *//更新At实例，将状态设置为相应的阶段：*
>
> \_, err = c.cnatClientset.CnatV1alpha1().Ats(instance.Namespace).
>
> UpdateStatus(instance)
>
> **if** err != **nil** {
>
> **return** time.Duration(0), err
>
> }
>
> *// Don\'t requeue. We should be reconcile because either the pod or*
>
> *// the CR changes.*
>
> **return** time.Duration(0), **nil**

Further, to set up informers and the controller at large, we implement
the following in NewController():

> *// NewController returns a new cnat controller*
>
> **func** NewController(
>
> kubeClientset kubernetes.Interface,
>
> cnatClientset clientset.Interface,
>
> atInformer informers.AtInformer,
>
> podInformer corev1informer.PodInformer) \*Controller {
>
> *// Create event broadcaster*
>
> *// Add cnat-controller types to the default Kubernetes Scheme so
> Events*
>
> *// can be logged for cnat-controller types.*
>
> utilruntime.Must(cnatscheme.AddToScheme(scheme.Scheme))
>
> klog.V(4).Info(\"Creating event broadcaster\")
>
> eventBroadcaster := record.NewBroadcaster()
>
> eventBroadcaster.StartLogging(klog.Infof)
>
> eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{
>
> Interface: kubeClientset.CoreV1().Events(\"\"),
>
> })
>
> source := corev1.EventSource{Component: controllerAgentName}
>
> recorder := eventBroadcaster.NewRecorder(scheme.Scheme, source)
>
> rateLimiter := workqueue.DefaultControllerRateLimiter()
>
> controller := &Controller{
>
> kubeClientset: kubeClientset,
>
> cnatClientset: cnatClientset,
>
> atLister: atInformer.Lister(),
>
> atsSynced: atInformer.Informer().HasSynced,
>
> podLister: podInformer.Lister(),
>
> podsSynced: podInformer.Informer().HasSynced,
>
> workqueue: workqueue.NewNamedRateLimitingQueue(rateLimiter, \"Ats\"),
>
> recorder: recorder,
>
> }
>
> klog.Info(\"Setting up event handlers\")
>
> *// Set up an event handler for when At resources change*
>
> atInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
>
> AddFunc: controller.enqueueAt,
>
> UpdateFunc: **func**(old, new **interface**{}) {
>
> controller.enqueueAt(new)
>
> },
>
> })
>
> *// Set up an event handler for when Pod resources change*
>
> podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
>
> AddFunc: controller.enqueuePod,
>
> UpdateFunc: **func**(old, new **interface**{}) {
>
> controller.enqueuePod(new)
>
> },
>
> })
>
> **return** controller
>
> }
>
> 为了使它起作用，我们还需要两个辅助函数：一个计算到schedule的时间，如下所示：
>
> **func** timeUntilSchedule(schedule **string**) (time.Duration,
> **error**) {
>
> now := time.Now().UTC()
>
> layout := \"2006-01-02T15:04:05Z\"
>
> s, err := time.Parse(layout, schedule)
>
> **if** err != **nil** {
>
> **return** time.Duration(0), err
>
> }
>
> **return** s.Sub(now), **nil**
>
> }
>
> 另一个使用busybox容器镜像创建一个带有要执行的命令的pod：**func**
> newPodForCR(cr \*cnatv1alpha1.At) \*corev1.Pod {
>
> labels := **map**\[**string**\]**string**{
>
> \"app\": cr.Name,
>
> }
>
> **return** &corev1.Pod{
>
> ObjectMeta: metav1.ObjectMeta{
>
> Name: cr.Name + \"-pod\",
>
> Namespace: cr.Namespace,
>
> Labels: labels,
>
> },
>
> Spec: corev1.PodSpec{
>
> Containers: \[\]corev1.Container{
>
> {
>
> Name: \"busybox\",
>
> Image: \"busybox\",
>
> Command: strings.Split(cr.Spec.Command, \" \"),
>
> },
>
> },
>
> RestartPolicy: corev1.RestartPolicyOnFailure,
>
> },
>
> }
>
> }

我们将在本章稍后的syncHandler（）函数中重用这两个辅助函数和业务逻辑的基本流程，因此请确保您熟悉它们的详细信息。

请注意，从资源的角度来看，pod是次要资源，控制器必须确保清理这些pod，否则将对孤立的pod造成风险。

现在，sample-controller是学习sausage制作方法的好工具，但是通常您要专注于创建业务逻辑而不是处理样板代码。为此，您可以选择两个相关项目：Kubebuilder和Operator
SDK。 让我们看看它们的各个方面以及如何实现它们。

Kubebuilder
===========

由Kubernetes特别兴趣小组（SIG）API
Machinery拥有并维护的Kubebuilder是一种工具和一组库，使您可以轻松有效地构建operators。深入了解Kubebuilder的最佳资源是在线《
Kubebuilder》一书，其中会带您逐步了解其组成和用法。
但是，我们这里将重点介绍如何使用Kubebuilder来实现cnat运算符（请参阅Git存储库中的相应目录）。

首先，请确保已安装所有依赖项-dep，kustomize（请参阅\`\`Kustomize\'\'）和Kubebuilder本身。

> \$ dep version
>
> dep:
>
> version : v0.5.1
>
> build date : 2019-03-11
>
> git hash : faa6189
>
> go version : go1.12
>
> go compiler : gc
>
> platform : darwin/amd64
>
> features : ImportDuringSolve=false
>
> \$ kustomize version
>
> Version: {KustomizeVersion:v2.0.3
> GitCommit:a6f65144121d1955266b0cd836ce954c04122dc8
>
> BuildDate:2019-03-18T22:15:21+00:00 GoOs:darwin GoArch:amd64}
>
> \$ kubebuilder version
>
> Version: version.Version{
>
> KubeBuilderVersion:\"1.0.8\",
>
> KubernetesVendor:\"1.13.1\",
>
> GitCommit:\"1adf50ed107f5042d7472ba5ab50d5e1d357169d\",
>
> BuildDate:\"2019-01-25T23:14:29Z\", GoOs:\"unknown\",
> GoArch:\"unknown\"
>
> }

我们将引导您完成从头编写cnat operator的步骤。
首先，创建一个您选择的目录（我们在我们的仓库中使用cnat-kubebuilder），它将用作所有其他命令的基础。

###### 警告

在撰写本文时，Kubebuilder正在迁移到新版本（v2）。由于尚不稳定，我们将显示（稳定）版本v1的命令和设置。

Bootstrapping
-------------

要引导cnat运算符，我们使用init命令，如下所示（请注意，这可能需要几分钟的时间，具体取决于您的环境）：

> \$ kubebuilder init **\\**
>
> \--domain programming-kubernetes.info **\\**
>
> \--license apache2 **\\**
>
> \--owner \"Programming Kubernetes authors\"
>
> Run \`dep ensure\` to fetch dependencies (Recommended) \[y/n\]?
>
> y
>
> dep ensure
>
> Running make\...
>
> make
>
> go generate ./pkg/\... ./cmd/\...
>
> go fmt ./pkg/\... ./cmd/\...
>
> go vet ./pkg/\... ./cmd/\...
>
> go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go
> all
>
> CRD manifests generated under \'config/crds\'
>
> RBAC manifests generated under \'config/rbac\'
>
> go test ./pkg/\... ./cmd/\... -coverprofile cover.out
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/apis \[no test files\]
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/controller \[no test
> files\]
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/webhook \[no test
> files\]
>
> ? github.com/mhausenblas/cnat-kubebuilder/cmd/manager \[no test
> files\]
>
> go build -o bin/manager
> github.com/mhausenblas/cnat-kubebuilder/cmd/manager

完成此命令后，Kubebuilder搭建了operator的脚手架，有效地生成了从自定义控制器到示例CRD的一堆文件。
您的基本目录现在应类似于以下内容（为清晰起见，不包括巨大的vendor目录）：

> \$ tree -I vendor
>
> .
>
> ├── Dockerfile
>
> ├── Gopkg.lock
>
> ├── Gopkg.toml
>
> ├── Makefile
>
> ├── PROJECT
>
> ├── bin
>
> │   └── manager
>
> ├── cmd
>
> │   └── manager
>
> │   └── main.go
>
> ├── config
>
> │   ├── crds
>
> │   ├── default
>
> │   │   ├── kustomization.yaml
>
> │   │   ├── manager_auth_proxy_patch.yaml
>
> │   │   ├── manager_image_patch.yaml
>
> │   │   └── manager_prometheus_metrics_patch.yaml
>
> │   ├── manager
>
> │   │   └── manager.yaml
>
> │   └── rbac
>
> │   ├── auth_proxy_role.yaml
>
> │   ├── auth_proxy_role_binding.yaml
>
> │   ├── auth_proxy_service.yaml
>
> │   ├── rbac_role.yaml
>
> │   └── rbac_role_binding.yaml
>
> ├── cover.out
>
> ├── hack
>
> │   └── boilerplate.go.txt
>
> └── pkg
>
> ├── apis
>
> │   └── apis.go
>
> ├── controller
>
> │   └── controller.go
>
> └── webhook
>
> └── webhook.go
>
> 13 directories, 22 files
>
> 接下来，我们使用create
> api命令创建一个API（即自定义控制器）（这应该比上一个命令更快，但仍然需要一些时间）：\$
> kubebuilder create api **\\**
>
> \--group cnat **\\**
>
> \--version v1alpha1 **\\**
>
> \--kind At
>
> Create Resource under pkg/apis \[y/n\]?
>
> y
>
> Create Controller under pkg/controller \[y/n\]?
>
> y
>
> Writing scaffold **for** you to edit\...
>
> pkg/apis/cnat/v1alpha1/at_types.go
>
> pkg/apis/cnat/v1alpha1/at_types_test.go
>
> pkg/controller/at/at_controller.go
>
> pkg/controller/at/at_controller_test.go
>
> Running make\...
>
> go generate ./pkg/\... ./cmd/\...
>
> go fmt ./pkg/\... ./cmd/\...
>
> go vet ./pkg/\... ./cmd/\...
>
> go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go
> all
>
> CRD manifests generated under \'config/crds\'
>
> RBAC manifests generated under \'config/rbac\'
>
> go test ./pkg/\... ./cmd/\... -coverprofile cover.out
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/apis \[no test files\]
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat \[no test
> files\]
>
> ok github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat/v1alpha1
> 9.011s
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/controller \[no test
> files\]
>
> ok github.com/mhausenblas/cnat-kubebuilder/pkg/controller/at 8.740s
>
> ? github.com/mhausenblas/cnat-kubebuilder/pkg/webhook \[no test
> files\]
>
> ? github.com/mhausenblas/cnat-kubebuilder/cmd/manager \[no test
> files\]
>
> go build -o bin/manager
> github.com/mhausenblas/cnat-kubebuilder/cmd/manager
>
> 让我们看看发生了什么变化，重点关注已收到更新和添加的两个目录：
>
> \$ tree config/ pkg/
>
> config/
>
> ├── crds
>
> │   └── cnat_v1alpha1_at.yaml
>
> ├── default
>
> │   ├── kustomization.yaml
>
> │   ├── manager_auth_proxy_patch.yaml
>
> │   ├── manager_image_patch.yaml
>
> │   └── manager_prometheus_metrics_patch.yaml
>
> ├── manager
>
> │   └── manager.yaml
>
> ├── rbac
>
> │   ├── auth_proxy_role.yaml
>
> │   ├── auth_proxy_role_binding.yaml
>
> │   ├── auth_proxy_service.yaml
>
> │   ├── rbac_role.yaml
>
> │   └── rbac_role_binding.yaml
>
> └── samples
>
> └── cnat_v1alpha1_at.yaml
>
> pkg/
>
> ├── apis
>
> │   ├── addtoscheme_cnat_v1alpha1.go
>
> │   ├── apis.go
>
> │   └── cnat
>
> │   ├── group.go
>
> │   └── v1alpha1
>
> │   ├── at_types.go
>
> │   ├── at_types_test.go
>
> │   ├── doc.go
>
> │   ├── register.go
>
> │   ├── v1alpha1_suite_test.go
>
> │   └── zz_generated.deepcopy.go
>
> ├── controller
>
> │   ├── add_at.go
>
> │   ├── at
>
> │   │   ├── at_controller.go
>
> │   │   ├── at_controller_suite_test.go
>
> │   │   └── at_controller_test.go
>
> │   └── controller.go
>
> └── webhook
>
> └── webhook.go
>
> 11 directories, 27 files

请注意在config / crds /中添加了Cat的cnat_v1alpha1_at.yaml，以及在config
/ samples
/中的cnat_v1alpha1_at.yaml（是，同名），代表了CRD的自定义资源示例实例。
此外，在pkg /中，我们看到了许多新文件，最重要的是apis / cnat / v1alpha1
/ at_types.go和controller / at /
at_controller.go，接下来我们将对其进行修改。

接下来，我们在Kubernetes中创建一个专用名称空间cnat并将其用作默认名称，并按如下所示设置上下文（作为一种好习惯，请始终使用专用名称空间而不是默认名称空间）：

> \$ kubectl create ns cnat && **\\**
>
> kubectl config set-context **\$(**kubectl config current-context**)**
> \--namespace=cnat
>
> 我们通过以下方式安装CRD：
>
> \$ make install
>
> go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go
> all
>
> CRD manifests generated under \'config/crds\'
>
> RBAC manifests generated under \'config/rbac\'
>
> kubectl apply -f config/crds
>
> customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info
> created
>
> 现在我们可以在本地启动operator：
>
> \$ make run
>
> go generate ./pkg/\... ./cmd/\...
>
> go fmt ./pkg/\... ./cmd/\...
>
> go vet ./pkg/\... ./cmd/\...
>
> go run ./cmd/manager/main.go
>
> {\"level\":\"info\",\"ts\":1559152740.0550249,\"logger\":\"entrypoint\",
>
> \"msg\":\"setting up client for manager\"}
>
> {\"level\":\"info\",\"ts\":1559152740.057556,\"logger\":\"entrypoint\",
>
> \"msg\":\"setting up manager\"}
>
> {\"level\":\"info\",\"ts\":1559152740.1396701,\"logger\":\"entrypoint\",
>
> \"msg\":\"Registering Components.\"}
>
> {\"level\":\"info\",\"ts\":1559152740.1397,\"logger\":\"entrypoint\",
>
> \"msg\":\"setting up scheme\"}
>
> {\"level\":\"info\",\"ts\":1559152740.139773,\"logger\":\"entrypoint\",
>
> \"msg\":\"Setting up controller\"}
>
> {\"level\":\"info\",\"ts\":1559152740.139831,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting EventSource\",\"controller\":\"at-controller\",
>
> \"source\":\"kind source: /, Kind=\"}
>
> {\"level\":\"info\",\"ts\":1559152740.139929,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting EventSource\",\"controller\":\"at-controller\",
>
> \"source\":\"kind source: /, Kind=\"}
>
> {\"level\":\"info\",\"ts\":1559152740.139971,\"logger\":\"entrypoint\",
>
> \"msg\":\"setting up webhooks\"}
>
> {\"level\":\"info\",\"ts\":1559152740.13998,\"logger\":\"entrypoint\",
>
> \"msg\":\"Starting the Cmd.\"}
>
> {\"level\":\"info\",\"ts\":1559152740.244628,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting Controller\",\"controller\":\"at-controller\"}
>
> {\"level\":\"info\",\"ts\":1559152740.344791,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting workers\",\"controller\":\"at-controller\",\"worker
> count\":1}
>
> 保持终端会话运行，并在新会话中安装CRD，对其进行验证并创建示例自定义资源，如下所示：
>
> \$ kubectl apply -f config/crds/cnat_v1alpha1_at.yaml
>
> customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info
>
> configured
>
> \$ kubectl get crds
>
> NAME CREATED AT
>
> ats.cnat.programming-kubernetes.info 2019-05-29T17:54:51Z
>
> \$ kubectl apply -f config/samples/cnat_v1alpha1_at.yaml
>
> at.cnat.programming-kubernetes.info/at-sample created
>
> 如果现在查看运行make运行的会话的输出，则应注意以下输出：\...
>
> {\"level\":\"info\",\"ts\":1559153311.659829,\"logger\":\"controller\",
>
> \"msg\":\"Creating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}
>
> {\"level\":\"info\",\"ts\":1559153311.678407,\"logger\":\"controller\",
>
> \"msg\":\"Updating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}
>
> {\"level\":\"info\",\"ts\":1559153311.6839428,\"logger\":\"controller\",
>
> \"msg\":\"Updating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}
>
> {\"level\":\"info\",\"ts\":1559153311.693443,\"logger\":\"controller\",
>
> \"msg\":\"Updating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}
>
> {\"level\":\"info\",\"ts\":1559153311.7023401,\"logger\":\"controller\",
>
> \"msg\":\"Updating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}
>
> {\"level\":\"info\",\"ts\":1559153332.986961,\"logger\":\"controller\",\#
>
> \"msg\":\"Updating
> Deployment\",\"namespace\":\"cnat\",\"name\":\"at-sample-deployment\"}

这告诉我们总体设置成功！
现在我们已经完成了脚手架并成功启动了cnat运算符，我们可以继续进行实际的核心任务：使用Kubebuilder实现cnat业务逻辑。

Business Logic
--------------

对于初学者，我们将重新使用与\`\` Following
sample-controller\'\'中相同的结构，将config / crds /
cnat_v1alpha1_at.yaml和config / samples /
cnat_v1alpha1_at.yaml更改为我们自己的cnat CRD和自定义资源值的定义。

在业务逻辑方面，我们需要在运营商中实现两个部分：

•在pkg / apis / cnat / v1alpha1 /
at_types.go中，我们修改AtSpec结构以包括相应的字段，例如时间表和命令。
请注意，无论何时在此处进行任何更改，都必须运行make来重新生成相关文件。
Kubebuilder使用Kubernetes生成器（在第5章中进行了介绍）并附带了自己的一组生成器（例如，生成CRD清单）。

•在pkg / controller / at / at_controller.go中，我们修改协调（request
reconcile.Request）方法以在Spec.Schedule中定义的时间创建一个pod。

在at_types.go中：

> **const** (
>
> PhasePending = \"PENDING\"
>
> PhaseRunning = \"RUNNING\"
>
> PhaseDone = \"DONE\"
>
> )
>
> *//* *AtSpec定义At的所需状态*
>
> **type** AtSpec **struct** {
>
> *// Schedule是应该执行命令的期望时间。*
>
> *//注意：此处使用的格式为UTC时间<https://www.utctime.ne>t*

Schedule **string** \`json:\"schedule,omitempty\"\`

> *// Command是要执行的所需命令（在Bash shell中执行）。*

Command **string** \`json:\"command,omitempty\"\`

> }
>
> *// AtStatus定义At的观察状态*
>
> **type** AtStatus **struct** {
>
> *//阶段表示调度的状态：直到命令执行*
>
> *//它是PENDING，之后是DONE。*

Phase **string** \`json:\"phase,omitempty\"\`

> }
>
> 在at_controller.go中，我们实现了三个阶段之间的状态转换，从PENDING到RUNNING到DONE：

**func** (r \*ReconcileAt) Reconcile(req reconcile.Request)
(reconcile.Result, **error**) {

> reqLogger := log.WithValues(\"namespace\", req.Namespace, \"at\",
> req.Name)
>
> reqLogger.Info(\"=== Reconciling At\")
>
> *// Fetch the At instance*
>
> instance := &cnatv1alpha1.At{}
>
> err := r.Get(context.TODO(), req.NamespacedName, instance)
>
> **if** err != **nil** {
>
> **if** errors.IsNotFound(err) {
>
> *// Request object not found, could have been deleted after*
>
> *// reconcile request---return and don\'t requeue:*
>
> **return** reconcile.Result{}, **nil**
>
> }
>
> *// Error reading the object---requeue the request:*
>
> **return** reconcile.Result{}, err
>
> }
>
> *// If no phase set, default to pending (the initial phase):*
>
> **if** instance.Status.Phase == \"\" {
>
> instance.Status.Phase = cnatv1alpha1.PhasePending
>
> }
>
> *// Now let\'s make the main case distinction: implementing*
>
> *// the state diagram PENDING -\> RUNNING -\> DONE*
>
> **switch** instance.Status.Phase {
>
> **case** cnatv1alpha1.PhasePending:
>
> reqLogger.Info(\"Phase: PENDING\")
>
> *// As long as we haven\'t executed the command yet, we need to check
> if*
>
> *// it\'s already time to act:*
>
> reqLogger.Info(\"Checking schedule\", \"Target\",
> instance.Spec.Schedule)
>
> *// Check if it\'s already time to execute the command with a
> tolerance*
>
> *// of 2 seconds:*
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
>
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
>
> **case** cnatv1alpha1.PhaseRunning:
>
> reqLogger.Info(\"Phase: RUNNING\")
>
> pod := newPodForCR(instance)
>
> *// Set At instance as the owner and controller*
>
> err := controllerutil.SetControllerReference(instance, pod, r.scheme)
>
> **if** err != **nil** {
>
> *// requeue with error*
>
> **return** reconcile.Result{}, err
>
> }
>
> found := &corev1.Pod{}
>
> nsName := types.NamespacedName{Name: pod.Name, Namespace:
> pod.Namespace}
>
> err = r.Get(context.TODO(), nsName, found)
>
> *// Try to see if the pod already exists and if not*
>
> *// (which we expect) then create a one-shot pod as per spec:*
>
> **if** err != **nil** && errors.IsNotFound(err) {
>
> err = r.Create(context.TODO(), pod)
>
> **if** err != **nil** {
>
> *// requeue with error*
>
> **return** reconcile.Result{}, err
>
> }
>
> reqLogger.Info(\"Pod launched\", \"name\", pod.Name)
>
> } **else** **if** err != **nil** {
>
> *// requeue with error*
>
> **return** reconcile.Result{}, err
>
> } **else** **if** found.Status.Phase == corev1.PodFailed \|\|
>
> found.Status.Phase == corev1.PodSucceeded {
>
> reqLogger.Info(\"Container terminated\", \"reason\",
>
> found.Status.Reason, \"message\", found.Status.Message)
>
> instance.Status.Phase = cnatv1alpha1.PhaseDone
>
> } **else** {
>
> *// Don\'t requeue because it will happen automatically when the*
>
> *// pod status changes.*
>
> **return** reconcile.Result{}, **nil**
>
> }
>
> **case** cnatv1alpha1.PhaseDone:
>
> reqLogger.Info(\"Phase: DONE\")
>
> **return** reconcile.Result{}, **nil**
>
> **default**:
>
> reqLogger.Info(\"NOP\")
>
> **return** reconcile.Result{}, **nil**
>
> }
>
> *// Update the At instance, setting the status to the respective
> phase:*
>
> err = r.Status().Update(context.TODO(), instance)
>
> **if** err != **nil** {
>
> **return** reconcile.Result{}, err
>
> }
>
> *// Don\'t requeue. We should be reconcile because either the pod*
>
> *// or the CR changes.*
>
> **return** reconcile.Result{}, **nil**
>
> }
>
> 请注意，最后的Update调用在/
> status子资源上运行（请参见\`\`Status子资源\'\'），而不是整个CR。
> 因此，这里我们遵循规范状态拆分的最佳实践。
>
> 现在，一旦创建了CR example-at，我们将看到本地执行的运算符的以下输出：
>
> \$ make run
>
> \...
>
> {\"level\":\"info\",\"ts\":1555063897.488535,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063897.488621,\"logger\":\"controller\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063897.4886441,\"logger\":\"controller\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T10:12:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555063897.488703,\"logger\":\"controller\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 10:12:00 +0000 UTC with a diff of
> 22.511336s\"}
>
> {\"level\":\"info\",\"ts\":1555063907.489264,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063907.489402,\"logger\":\"controller\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063907.489428,\"logger\":\"controller\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T10:12:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555063907.489486,\"logger\":\"controller\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 10:12:00 +0000 UTC with a diff of
> 12.510551s\"}
>
> {\"level\":\"info\",\"ts\":1555063917.490178,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063917.4902349,\"logger\":\"controller\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063917.490247,\"logger\":\"controller\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T10:12:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555063917.490278,\"logger\":\"controller\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 10:12:00 +0000 UTC with a diff of 2.509743s\"}
>
> {\"level\":\"info\",\"ts\":1555063927.492718,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063927.49283,\"logger\":\"controller\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063927.492857,\"logger\":\"controller\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T10:12:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555063927.492915,\"logger\":\"controller\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 10:12:00 +0000 UTC with a diff of
> -7.492877s\"}
>
> {\"level\":\"info\",\"ts\":1555063927.4929411,\"logger\":\"controller\",
>
> \"msg\":\"It\'s time!\",\"namespace\":\"cnat\",\"at\":
>
> \"example-at\",\"Ready to execute\":\"echo YAY\"}
>
> {\"level\":\"info\",\"ts\":1555063927.626236,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063927.626303,\"logger\":\"controller\",
>
> \"msg\":\"Phase:
> RUNNING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063928.07445,\"logger\":\"controller\",
>
> \"msg\":\"Pod launched\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"name\":\"example-at-pod\"}
>
> {\"level\":\"info\",\"ts\":1555063928.199562,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063928.199645,\"logger\":\"controller\",
>
> \"msg\":\"Phase: DONE\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063937.631733,\"logger\":\"controller\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555063937.631783,\"logger\":\"controller\",
>
> \"msg\":\"Phase: DONE\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> \...

要验证我们的自定义控制器是否已完成其工作，请执行以下命令：

> \$ kubectl get at,pods
>
> NAME AGE
>
> at.cnat.programming-kubernetes.info/example-at 11m
>
> NAME READY STATUS RESTARTS AGE
>
> pod/example-at-pod 0/1 Completed 0 38s

已经创建了pod-example实例，现在是时候查看操作结果了：

> \$ kubectl logs example-at-pod
>
> YAY

在完成自定义控制器的开发后，使用此处所示的本地模式，您可能需要在其中构建一个容器镜像。
此自定义控制器容器映像随后可以在例如Kubernetes部署中使用。您可以使用以下命令生成容器镜像并将其推送到repoquay.io/pk/cnat中：

> \$ export IMG=quay.io/pk/cnat:v1
>
> \$ make docker-build
>
> \$ make docker-push

通过此操作，我们继续使用Operator
SDK，该SDK共享Kubebuilder的一些代码库和API。

The Operator SDK
================

为了使构建Kubernetes应用程序更加容易，CoreOS / Red Hat整合了Operator
Framework。其中的一部分是\`\`Operator
SDK\'\'，它使开发人员可以构建operators而无需对Kubernetes API的深入了解。

Operator SDK提供了用于构建，测试和打包operators的工具。
虽然SDK提供了更多功能，尤其是在测试方面，但我们在这里着重于用SDK实施cnat运算符（请参阅Git存储库中的相应目录）。

首先，请确保安装了Operator SDK，并检查所有依赖项是否可用：

> \$ dep version
>
> dep:
>
> version : v0.5.1
>
> build date : 2019-03-11
>
> git hash : faa6189
>
> go version : go1.12
>
> go compiler : gc
>
> platform : darwin/amd64
>
> features : ImportDuringSolve=false
>
> \$ operator-sdk \--version
>
> operator-sdk version v0.6.0

Bootstrapping
-------------

> 现在是时候按照以下步骤引导CNat运算符了：

\$ operator-sdk new cnat-operator && cd cnat-operator

> 接下来，与Kubebuilder非常相似，我们添加一个API或简单地说：初始化自定义控制器，如下所示：

\$ operator-sdk add api **\\**

> \--api-version=cnat.programming-kubernetes.info/v1alpha1 **\\**
>
> \--kind=At
>
> \$ operator-sdk add controller **\\**
>
> \--api-version=cnat.programming-kubernetes.info/v1alpha1 **\\**
>
> \--kind=At
>
> 这些命令生成必要的样板代码以及许多帮助程序功能，例如Deep-copy函数Deepeep（），DeepCopyInto（）和DeepCopyObject（）。
>
> 现在我们可以将自动生成的CRD应用于Kubernetes集群：

\$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_crd.yaml

> \$ kubectl get crds
>
> NAME CREATED AT
>
> ats.cnat.programming-kubernetes.info 2019-04-01T14:03:33Z
>
> 让我们在本地启动cnat自定义控制器。 这样，它可以开始处理请求：
>
> \$ OPERATOR_NAME=cnatop operator-sdk up local \--namespace \"cnat\"
>
> INFO\[0000\] Running the operator locally.
>
> INFO\[0000\] Using namespace cnat.
>
> {\"level\":\"info\",\"ts\":1555041531.871706,\"logger\":\"cmd\",
>
> \"msg\":\"Go Version: go1.12.1\"}
>
> {\"level\":\"info\",\"ts\":1555041531.871785,\"logger\":\"cmd\",
>
> \"msg\":\"Go OS/Arch: darwin/amd64\"}
>
> {\"level\":\"info\",\"ts\":1555041531.8718028,\"logger\":\"cmd\",
>
> \"msg\":\"Version of operator-sdk: v0.6.0\"}
>
> {\"level\":\"info\",\"ts\":1555041531.8739321,\"logger\":\"leader\",
>
> \"msg\":\"Trying to become the leader.\"}
>
> {\"level\":\"info\",\"ts\":1555041531.8743382,\"logger\":\"leader\",
>
> \"msg\":\"Skipping leader election; not running in a cluster.\"}
>
> {\"level\":\"info\",\"ts\":1555041536.1611362,\"logger\":\"cmd\",
>
> \"msg\":\"Registering Components.\"}
>
> {\"level\":\"info\",\"ts\":1555041536.1622112,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting EventSource\",\"controller\":\"at-controller\",
>
> \"source\":\"kind source: /, Kind=\"}
>
> {\"level\":\"info\",\"ts\":1555041536.162519,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting EventSource\",\"controller\":\"at-controller\",
>
> \"source\":\"kind source: /, Kind=\"}
>
> {\"level\":\"info\",\"ts\":1555041539.978822,\"logger\":\"metrics\",
>
> \"msg\":\"Skipping metrics Service creation; not running in a
> cluster.\"}
>
> {\"level\":\"info\",\"ts\":1555041539.978875,\"logger\":\"cmd\",
>
> \"msg\":\"Starting the Cmd.\"}
>
> {\"level\":\"info\",\"ts\":1555041540.179469,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting Controller\",\"controller\":\"at-controller\"}
>
> {\"level\":\"info\",\"ts\":1555041540.280784,\"logger\":\"kubebuilder.controller\",
>
> \"msg\":\"Starting workers\",\"controller\":\"at-controller\",\"worker
> count\":1}
>
> 我们的自定义控制器将保持此状态，直到我们创建CR（ats.cnat.programming-kubernetes.info）。
> 因此，让我们这样做：
>
> \$ cat deploy/crds/cnat_v1alpha1_at_cr.yaml
>
> apiVersion: cnat.programming-kubernetes.info/v1alpha1
>
> kind: At
>
> metadata:
>
> name: example-at
>
> spec:
>
> schedule: \"2019-04-11T14:56:30Z\"
>
> command: \"echo YAY\"
>
> \$ kubectl apply -f deploy/crds/cnat_v1alpha1_at_cr.yaml
>
> \$ kubectl get at
>
> NAME AGE
>
> at.cnat.programming-kubernetes.info/example-at 54s

Business Logic
--------------

用业务逻辑的术语来说，我们需要在运营商中实现两个部分：

•在pkg / apis / cnat / v1alpha1 /
at_types.go中，我们修改AtSpec结构以包括相应的字段，例如时间表和命令，并使用operator-sdk生成k8s来重新生成代码，以及使用operator-sdk生成
用于OpenAPI位的openapi命令。

•在pkg / controller / at / at_controller.go中，我们修改协调（request
reconcile.Request）方法以在Spec.Schedule中定义的时间创建一个pod。

应用于自举代码的更详细的更改如下（集中在相关位上）。 在at_types.go中：

> *// AtSpec defines the desired state of At*
>
> *// +k8s:openapi-gen=true*
>
> **type** AtSpec **struct** {
>
> *// Schedule is the desired time the command is supposed to be
> executed.*
>
> *// Note: the format used here is UTC time https://www.utctime.net*
>
> Schedule **string** \`json:\"schedule,omitempty\"\`
>
> *// Command is the desired command (executed in a Bash shell) to be
> executed.*
>
> Command **string** \`json:\"command,omitempty\"\`
>
> }
>
> *// AtStatus defines the observed state of At*
>
> *// +k8s:openapi-gen=true*
>
> **type** AtStatus **struct** {
>
> *// Phase represents the state of the schedule: until the command is
> executed*
>
> *// it is PENDING, afterwards it is DONE.*
>
> Phase **string** \`json:\"phase,omitempty\"\`
>
> }

在at_controller.go中，我们实现了三个阶段的状态图，即PENDING to RUNNING
to DONE。

###### 注意

controller-runtime 是SIG API
Machinery公司的另一个项目，旨在以Go软件包的形式为构建控制器提供一套通用的低级功能。有关更多详细信息，请参见第4章。

由于Kubebuilder和Operator
SDK共享控制器运行时，因此Reconcile（）函数实际上是相同的

> **func** (r \*ReconcileAt) Reconcile(request reconcile.Request)
> (reconcile.Result, **error**) {
>
> *the-same-as-**for**-kubebuilder*
>
> }
>
> 一旦创建了CR example-at，我们将看到本地执行的operator的以下输出：

\$ OPERATOR_NAME=cnatop operator-sdk up local \--namespace \"cnat\"

> INFO\[0000\] Running the operator locally.
>
> INFO\[0000\] Using namespace cnat.
>
> \...
>
> {\"level\":\"info\",\"ts\":1555044934.023597,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044934.023713,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044934.0237482,\"logger\":\"controller_at\",
>
> \"msg\":\"Checking schedule\",\"namespace\":\"cnat\",\"at\":
>
> \"example-at\",\"Target\":\"2019-04-12T04:56:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555044934.02382,\"logger\":\"controller_at\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 04:56:00 +0000 UTC with a diff of
> 25.976236s\"}
>
> {\"level\":\"info\",\"ts\":1555044934.148148,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044934.148224,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044934.148243,\"logger\":\"controller_at\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T04:56:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555044934.1482902,\"logger\":\"controller_at\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 04:56:00 +0000 UTC with a diff of 25.85174s\"}
>
> {\"level\":\"info\",\"ts\":1555044944.1504588,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044944.150568,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044944.150599,\"logger\":\"controller_at\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T04:56:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555044944.150663,\"logger\":\"controller_at\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 04:56:00 +0000 UTC with a diff of 15.84938s\"}
>
> {\"level\":\"info\",\"ts\":1555044954.385175,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044954.3852649,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044954.385288,\"logger\":\"controller_at\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T04:56:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555044954.38534,\"logger\":\"controller_at\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 04:56:00 +0000 UTC with a diff of 5.614691s\"}
>
> {\"level\":\"info\",\"ts\":1555044964.518383,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044964.5184839,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> PENDING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044964.518566,\"logger\":\"controller_at\",
>
> \"msg\":\"Checking
> schedule\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Target\":\"2019-04-12T04:56:00Z\"}
>
> {\"level\":\"info\",\"ts\":1555044964.5186381,\"logger\":\"controller_at\",
>
> \"msg\":\"Schedule parsing
> done\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Result\":\"2019-04-12 04:56:00 +0000 UTC with a diff of
> -4.518596s\"}
>
> {\"level\":\"info\",\"ts\":1555044964.5186849,\"logger\":\"controller_at\",
>
> \"msg\":\"It\'s time!\",\"namespace\":\"cnat\",\"at\":\"example-at\",
>
> \"Ready to execute\":\"echo YAY\"}
>
> {\"level\":\"info\",\"ts\":1555044964.642559,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044964.642622,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> RUNNING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044964.911037,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044964.9111192,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase:
> RUNNING\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044966.038684,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044966.038771,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase: DONE\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044966.708663,\"logger\":\"controller_at\",
>
> \"msg\":\"=== Reconciling
> At\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> {\"level\":\"info\",\"ts\":1555044966.708749,\"logger\":\"controller_at\",
>
> \"msg\":\"Phase: DONE\",\"namespace\":\"cnat\",\"at\":\"example-at\"}
>
> \...

在这里您可以看到我们operator的三个阶段：PENDING，直到时间戳记1555044964.518566，然后是RUNNING，然后是DONE。

要验证我们的自定义控制器的功能并检查操作结果，请输入：

> \$ kubectl get at,pods
>
> NAME AGE
>
> at.cnat.programming-kubernetes.info/example-at 23m
>
> NAME READY STATUS RESTARTS AGE
>
> pod/example-at-pod 0/1 Completed 0 46s
>
> \$ kubectl logs example-at-pod
>
> YAY

在完成自定义控制器的开发后，使用此处所示的本地模式，您可能需要在其中构建一个容器镜像。
此自定义控制器容器镜像随后可以在例如Kubernetes部署中使用。
您可以使用以下命令来生成容器镜像：

> \$ operator-sdk build \$REGISTRY/PROJECT/IMAGE

以下是一些其他资源，以了解有关Operator SDK及其示例的更多信息：

•Toader Sebastian在BanzaiCloud上撰写的" Kubernetes Operator SDK完整指南"

•Rob Szumski的博客文章"为Prometheus和Thanos构建Kubernetes运营商"

•在ITNEXT上来自CloudARK的"提高可用性的Kubernetes运营商开发指南"

作为本章的最后，让我们看看编写自定义控制器和运算符的其他方法。

Other Approaches
================

除了我们已经讨论过的方法之外，或者可能与之结合的方法，您可能需要查看以下项目，库和工具：

Metacontroller

Metacontroller的基本思想是基于级别触发的对帐循环为您提供状态和更改的声明性规范，并与JSON接口。也就是说，您将收到描述所观察到状态的JSON并返回描述您想要的状态的JSON。这对于以动态脚本语言（例如Python或JavaScript）快速开发自动化特别有用。除了简单的控制器外，Metacontroller还允许您将API组合成更高级别的抽象-例如BlueGreenDeployment。

KUDO

类似于Metacontroller，KUDO提供了一种声明式方法来构建Kubernetes运算符，涵盖了整个应用程序生命周期。简而言之，它将Mesosphere从Apache
Mesos框架中获得的经验移植到Kubernetes。
KUDO固执己见，但易于使用，几乎不需要编码。本质上，您只需指定一个带有内置逻辑的Kubernetes清单的集合即可定义何时执行。

Rook operator kit

这是用于实现运算符的通用库。它起源于Rook运算符，但已分解为一个单独的独立项目。

ericchiang / k8s

这是Eric
Chiang瘦身的Go客户端，它使用Kubernetes协议缓冲区支持生成。它的行为类似于官方的Kubernetes
client-go，但是仅导入两个外部依赖项。虽然它具有某些限制-例如在集群访问配置方面-它是一个易于使用的Go软件包。

kutil

AppsCode通过kutil提供Kubernetes客户端附加组件。

基于CLI客户端的方法

一种主要用于实验和测试的客户端方法是通过编程利用Kubectl（例如kubecuddler库）。

###### 注意

在本书中我们专注于使用Go编程语言编写运算符时，您可以使用其他语言编写运算符。
两个著名的例子是Flant的Shell-operator，它使您可以使用良好的旧Shell脚本编写运算符，以及Zalando的Kopf（Kubernetes运算符框架），Python框架和库。

如本章开头所述，operator领域正在迅速发展，越来越多的从业者以代码和最佳实践的形式共享他们的知识，因此请在此处关注新工具。
确保检查在线资源和论坛，例如Kubernetes
Slack上的＃kubernetes-operators，＃kubebuilder和＃client-go-docs频道，以了解新方法和/或讨论问题并在您获得建议时获得帮助
重新卡住。

Uptake and Future Directions
============================

关于哪种写操作符的方法将是最流行和广泛使用的问题，尚无定论。
在Kubernetes项目的上下文中，涉及CR和控制器的几个SIG中都有活动。
主要利益相关者是SIG API
Machinery公司，它拥有CR和控制器，并负责Kubebuilder项目。 Operator
SDK已加倍努力以与Kubebuilder API保持一致，因此存在很多重叠之处。

摘要
====

在本章中，我们介绍了不同的工具，这些工具可让您更有效地编写自定义控制器和运算符。
传统上，只有sample-controller是唯一的选择，但有了Kubebuilder和Operator
SDK，您现在有了两个选择，可让您专注于自定义控制器的业务逻辑而不是处理样板。幸运的是，这两个工具共享许多API和代码，因此从一个工具迁移到另一个工具应该不会太困难。

现在，让我们看看如何交付我们的工作成果，即如何package和ship我们一直在编写的控制器。

1\'我们仅在此处显示相关部分； 函数本身还有很多我们不关心的其他样板代码。
