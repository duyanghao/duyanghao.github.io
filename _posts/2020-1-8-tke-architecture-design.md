---
layout: post
title: TKE架构设计
date: 2020-1-8 19:10:31
category: 技术
tags: Kubernetes
excerpt: 本文对TKE架构设计进行分析和讲解……
---

## 前言

[TKE](https://github.com/tkestack/tke)是腾讯Kubernetes-native公有云容器管理平台，其整体架构设计如下所示:

![](/public/img/tke/architecture.png)

本文主要针对Global Cluster的代码架构设计进行分析

## TKE Global Cluster

TKE Global cluster采用Kubernetes标准的[Aggregated APIServer](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/aggregated-api-servers.md)设计模式，目录结构如下所示：

```bash
├── cmd
│   ├── generate-images
│   │   └── main.go
│   ├── tke-auth-api
│   │   ├── app
│   │   └── auth.go
│   ├── tke-auth-controller
│   │   ├── app
│   │   └── controller-manager.go
│   ├── tke-business-api
│   │   ├── apiserver.go
│   │   └── app
│   ├── tke-business-controller
│   │   ├── app
│   │   └── controller-manager.go
│   ├── tke-gateway
│   │   ├── app
│   │   └── gateway.go
│   ├── tke-installer
│   │   ├── app
│   │   ├── assets
│   │   └── installer.go
│   ├── tke-monitor-api
│   │   ├── apiserver.go
│   │   └── app
│   ├── tke-monitor-controller
│   │   ├── app
│   │   └── controller-manager.go
│   ├── tke-notify-api
│   │   ├── apiserver.go
│   │   └── app
│   ├── tke-notify-controller
│   │   ├── app
│   │   └── controller-manager.go
│   ├── tke-platform-api
│   │   ├── apiserver.go
│   │   └── app
│   ├── tke-platform-controller
│   │   ├── app
│   │   └── controller-manager.go
│   └── tke-registry-api
│       ├── apiserver.go
│       └── app
├── pkg
│   ├── auth
│   ├── business
│   │   ├── apiserver
│   │   ├── controller
│   │   ├── registry
│   │   └── util
│   ├── gateway
│   ├── monitor
│   ├── notify
│   ├── platform
│   ├── registry
```

可以看到TKE将Global Cluster分成了若干个模块：gateway、registry、business等。每个模块对应TKE的若干自定义资源类型，例如这里的business对应TKE资源`Project`、`Namespace`、`Portal`。各模块均采用Aggregated APIServer和相应的Controller进行处理，这个会在后续进行分析。这里我们先依次介绍一下Kubernetes扩展工作负载的几个概念：

* Custom Resource
* Controller
* Aggregated APIServer
* Operator

## Custom Resource

>> A resource is an endpoint in the Kubernetes API that stores a collection of API objects of a certain kind. For example, the built-in pods resource contains a collection of Pod objects.

>> A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. However, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.

>> Custom resources can appear and disappear in a running cluster through dynamic registration, and cluster admins can update custom resources independently of the cluster itself. Once a custom resource is installed, users can create and access its objects using kubectl, just as they do for built-in resources like Pods.

Custom Resource，简称CR，是Kubernetes自定义资源类型，与之相对应的就是Kubernetes内置的各种资源类型，例如Pod、Service等。利用CR我们可以定义任何想要的资源类型，例如这里TKE的`Project`等

而对于如何使用CR，官方也给出了两种方式：

>> Kubernetes provides two ways to add custom resources to your cluster:

>> CRDs are simple and can be created without any programming.
>> API Aggregation requires programming, but allows more control over API behaviors like how data is stored and conversion between API versions.
Kubernetes provides these two options to meet the needs of different users, so that neither ease of use nor flexibility is compromised.

>> Aggregated APIs are subordinate APIServers that sit behind the primary API server, which acts as a proxy. This arrangement is called API Aggregation (AA). To users, it simply appears that the Kubernetes API is extended.

>> CRDs allow users to create new types of resources without adding another APIserver. You do not need to understand API Aggregation to use CRDs.

>> Regardless of how they are installed, the new resources are referred to as Custom Resources to distinguish them from built-in Kubernetes resources (like pods).

也即Aggregated APIServer和CRDs，这两种方式各有优缺点，适用场景也不相同，如下：

* CRD更简单，不需要programming，更加轻量级；相比AA则需要专门programming并维护
* Aggregated APIServer更加灵活，可以完成很多CRD不具备的事情，例如：对存储层的CRUD定制化操作

详细比较可以参考[这里](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/compare_with_kubebuilder.md)

## Controller

无论是采用Aggregated APIServer还是CRDs，都必须使用Controller。在Kubernetes中，Controller是一个控制循环，它通过apiserver监视集群的共享状态，并进行更改以尝试将当前状态移向所需状态(Reconcile Loop)：

```bash
for {
  if current != desired {
    change(current, desired)
  }
}
```

![](/public/img/tke/reconciliation.jpg)

而对于custom controller，原理和Kubernetes Controller Manager一致，如下：

![](/public/img/tke/client-go-controller-interaction.jpeg)

详情参考[sample controller](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)

## Aggregated APIServer

>> Aggregated APIServer Motivation

>> * Extensibility: We want to allow community members to write their own API servers to expose APIs they want. Cluster admins should be able to use these servers without having to require any change in the core kubernetes repository.
>> * Unblock new APIs from core kubernetes team review: A lot of new API proposals are currently blocked on review from the core kubernetes team. By allowing developers to expose their APIs as a separate server and enabling the cluster admin to use it without any change to the core kubernetes repository, we unblock these APIs.
>> * Place for staging experimental APIs: New APIs can be developed in separate aggregated servers, and installed only by those willing to take the risk of installing an experimental API. Once they are stable, it is then easy to package them up for installation in other clusters.
>> * Ensure that new APIs follow kubernetes conventions: Without the mechanism proposed here, community members might be forced to roll their own thing which may or may not follow kubernetes conventions.
 
Aggregated APIServer(简称AA)是Kubernetes提出的用于客户定制API需求的解决方案，也是Kubernetes扩展工作负载的一种方式。利用AA我们可以用Kubernetes-native的方式对CR做任何事情，其中最基本的就是存储层的CRUD操作了

AA通过向Kubernetes注册CR的方式来实现api-resource，如下是[apiserver-builder-alpha](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/concepts/api_building_overview.md)对AA的工作流解释：

![](/public/img/tke/extensionserver.jpg)

流程如下：

* 1、对于CR的API请求，先发送给Kubernetes core APIServer，代理给AA
* 2、AA接受请求，然后操作etcd，这个etcd可以与Kubernetes etcd共用，也可以单独部署一套
* 3、Custom Controller Watch core APIServer CR资源变化(注意这个时候Watch会代理给AA)
* 4、如果CR发生变化，Custom Controller接受到变化对象和相应事件，并对该CR进行相关操作，可以是CR本身，也可以是CR所关联的Kubernetes原生资源类型，例如Pod、Deployment等
* 5、如果是CR本身的CRUD操作，则走core APIServer，然后代理给AA；否则直接走core APIServer

## Operator

CRDs是扩展Kubernetes工作负载的第二种方式，也是相对简单的一种：

>> The CustomResourceDefinition API resource allows you to define custom resources. Defining a CRD object creates a new custom resource with a name and schema that you specify. The Kubernetes API serves and handles the storage of your custom resource.

>> This frees you from writing your own API server to handle the custom resource, but the generic nature of the implementation means you have less flexibility than with API server aggregation

CRD通过yaml文件的形式向Kubernetes注册CR实现api-resource，用户无需额外编写AA，但是需要维护一个Custom Controller以处理CR变化，Kubernetes将这种方式称为Operator，也即：Operator=CRD+Controller

## TKE business Aggregated APIServer

在介绍完上述概念后，我们回到主题TKE，由于TKE各模块均采用AA方式(Aggregated APIServer+Custom Controller)编写，这里我们针对其中的business模块进行详细分析

TKE business模块整合了Project、Namespace、Platform等资源的处理：

```sh
├── cmd
│   ├── tke-business-api
│   │   ├── apiserver.go
│   │   └── app
│   ├── tke-business-controller
│   │   ├── app
│   │   └── controller-manager.go
├── pkg
    ├── business
        ├── apiserver
        │   ├── apiserver.go
        │   └── install.go
        ├── controller
        │   ├── chartgroup
        │   ├── imagenamespace
        │   ├── namespace
        │   └── project
        ├── registry
        │   ├── chartgroup
        │   ├── configmap
        │   ├── imagenamespace
        │   ├── namespace
        │   ├── platform
        │   ├── portal
        │   ├── project
        │   └── rest
        └── util
            ├── filter.go
            ├── label.go
            └── resource.go
```

启动TKE business Aggregated APIServer(github.com/tkestack/tke/cmd/tke-business-api/app/server.go:28)：

```go
// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(cfg *config.Config) (*genericapiserver.GenericAPIServer, error) {
	apiServerConfig := createAPIServerConfig(cfg)
	apiServer, err := CreateAPIServer(apiServerConfig, genericapiserver.NewEmptyDelegate())
	if err != nil {
		return nil, err
	}

	apiServer.GenericAPIServer.AddPostStartHookOrDie("start-business-api-server-informers", func(context genericapiserver.PostStartHookContext) error {
		cfg.VersionedSharedInformerFactory.Start(context.StopCh)
		return nil
	})

	return apiServer.GenericAPIServer, nil
}

// CreateAPIServer creates and wires a workable tke-business-api
func CreateAPIServer(apiServerConfig *apiserver.Config, delegateAPIServer genericapiserver.DelegationTarget) (*apiserver.APIServer, error) {
	return apiServerConfig.Complete().New(delegateAPIServer)
}

func createAPIServerConfig(cfg *config.Config) *apiserver.Config {
	return &apiserver.Config{
		GenericConfig: &genericapiserver.RecommendedConfig{
			Config: *cfg.GenericAPIServerConfig,
		},
		ExtraConfig: apiserver.ExtraConfig{
			ServerName:              cfg.ServerName,
			VersionedInformers:      cfg.VersionedSharedInformerFactory,
			StorageFactory:          cfg.StorageFactory,
			APIResourceConfigSource: cfg.StorageFactory.APIResourceConfigSource,
			PlatformClient:          cfg.PlatformClient,
			RegistryClient:          cfg.RegistryClient,
			PrivilegedUsername:      cfg.PrivilegedUsername,
			FeatureOptions:          cfg.FeatureOptions,
		},
	}
}
...
// New returns a new instance of APIServer from the given config.
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*APIServer, error) {
	s, err := c.GenericConfig.New(c.ExtraConfig.ServerName, delegationTarget)
	if err != nil {
		return nil, err
	}

	m := &APIServer{
		GenericAPIServer: s,
	}

	// The order here is preserved in discovery.
	restStorageProviders := []storage.RESTStorageProvider{
		&businessrest.StorageProvider{
			LoopbackClientConfig: c.GenericConfig.LoopbackClientConfig,
			PlatformClient:       c.ExtraConfig.PlatformClient,
			RegistryClient:       c.ExtraConfig.RegistryClient,
			PrivilegedUsername:   c.ExtraConfig.PrivilegedUsername,
			Features:             c.ExtraConfig.FeatureOptions,
		},
	}
	m.InstallAPIs(c.ExtraConfig, c.GenericConfig.RESTOptionsGetter, restStorageProviders...)
	m.GenericAPIServer.AddPostStartHookOrDie("default-administrator", c.postStartHookFunc())

	return m, nil
}
```

注册CR(github.com/tkestack/tke/pkg/business/apiserver/apiserver.go:113)：

```go
// InstallAPIs will install the APIs for the restStorageProviders if they are enabled.
func (m *APIServer) InstallAPIs(extraConfig *ExtraConfig, restOptionsGetter generic.RESTOptionsGetter, restStorageProviders ...storage.RESTStorageProvider) {
	var apiGroupsInfo []genericapiserver.APIGroupInfo

	for _, restStorageBuilder := range restStorageProviders {
		groupName := restStorageBuilder.GroupName()
		if !extraConfig.APIResourceConfigSource.AnyVersionForGroupEnabled(groupName) {
			log.Infof("Skipping disabled API group %q.", groupName)
			continue
		}
		apiGroupInfo, enabled := restStorageBuilder.NewRESTStorage(extraConfig.APIResourceConfigSource, restOptionsGetter)
		if !enabled {
			log.Warnf("Problem initializing API group %q, skipping.", groupName)
			continue
		}
		log.Infof("Enabling API group %q.", groupName)

		if postHookProvider, ok := restStorageBuilder.(genericapiserver.PostStartHookProvider); ok {
			name, hook, err := postHookProvider.PostStartHook()
			if err != nil {
				log.Fatalf("Error building PostStartHook: %v", err)
			}
			m.GenericAPIServer.AddPostStartHookOrDie(name, hook)
		}

		apiGroupsInfo = append(apiGroupsInfo, apiGroupInfo)
	}

	for i := range apiGroupsInfo {
		if err := m.GenericAPIServer.InstallAPIGroup(&apiGroupsInfo[i]); err != nil {
			log.Fatalf("Error in registering group versions: %v", err)
		}
	}
}
 
...
// github.com/tkestack/tke/pkg/business/registry/rest/rest.go:73
func (s *StorageProvider) v1Storage(apiResourceConfigSource serverstorage.APIResourceConfigSource,
	restOptionsGetter generic.RESTOptionsGetter, loopbackClientConfig *restclient.Config,
	features *options.FeatureOptions) map[string]rest.Storage {
	businessClient := businessinternalclient.NewForConfigOrDie(loopbackClientConfig)

	storageMap := make(map[string]rest.Storage)
	{
		projectREST := projectstorage.NewStorage(restOptionsGetter, businessClient, s.PlatformClient, s.PrivilegedUsername, features)
		storageMap["projects"] = projectREST.Project
		storageMap["projects/status"] = projectREST.Status
		storageMap["projects/finalize"] = projectREST.Finalize

		namespaceREST := namespacestorage.NewStorage(restOptionsGetter, businessClient, s.PlatformClient, s.PrivilegedUsername)
		storageMap["namespaces"] = namespaceREST.Namespace
		storageMap["namespaces/status"] = namespaceREST.Status

		platformREST := platformstorage.NewStorage(restOptionsGetter, businessClient, s.PrivilegedUsername)
		storageMap["platforms"] = platformREST.Platform

		portalREST := portalstorage.NewStorage(restOptionsGetter, businessClient)
		storageMap["portal"] = portalREST.Portal

		configMapREST := configmapstorage.NewStorage(restOptionsGetter)
		storageMap["configmaps"] = configMapREST.ConfigMap

		if s.RegistryClient != nil {
			imageNamespaceREST := imagenamespacestorage.NewStorage(restOptionsGetter, businessClient, s.RegistryClient, s.PrivilegedUsername)
			storageMap["imagenamespaces"] = imageNamespaceREST.ImageNamespace
			storageMap["imagenamespaces/status"] = imageNamespaceREST.Status
			storageMap["imagenamespaces/finalize"] = imageNamespaceREST.Finalize

			chartGroupREST := chartgroupstorage.NewStorage(restOptionsGetter, businessClient, s.RegistryClient, s.PrivilegedUsername)
			storageMap["chartgroups"] = chartGroupREST.ChartGroup
			storageMap["chartgroups/status"] = chartGroupREST.Status
			storageMap["chartgroups/finalize"] = chartGroupREST.Finalize
		}
	}

	return storageMap
}
```

business中相应的AA处理逻辑都在pkg/business/registry目录中，如下：

```sh
pkg/business/registry
├── chartgroup
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   └── validation.go
├── configmap
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   └── validation.go
├── imagenamespace
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   └── validation.go
├── namespace
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   ├── validation.go
│   └── validation_test.go
├── platform
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   └── validation.go
├── portal
│   └── storage
│       └── storage.go
├── project
│   ├── storage
│   │   └── storage.go
│   ├── strategy.go
│   ├── validation.go
│   └── validation_test.go
└── rest
    └── rest.go
```

而对应的Controller处理逻辑都在pkg/controller中，如下：

```sh
pkg/business/controller
├── chartgroup
│   ├── chartgroup_cache.go
│   ├── chartgroup_controller.go
│   ├── chartgroup_health.go
│   └── deletion
│       └── chartgroup_resources_deleter.go
├── imagenamespace
│   ├── deletion
│   │   └── imagenamespace_resources_deleter.go
│   ├── imagenamespace_cache.go
│   ├── imagenamespace_controller.go
│   └── imagenamespace_health.go
├── namespace
│   ├── cluster.go
│   ├── deletion
│   │   └── namespaced_resources_deleter.go
│   ├── namespace_cache.go
│   ├── namespace_controller.go
│   └── namespace_health.go
└── project
    ├── deletion
    │   └── projected_resources_deleter.go
    ├── project_cache.go
    └── project_controller.go
```

其中AA采用[sample-apiserver](https://github.com/kubernetes/sample-apiserver)编写，Custom Controller采用[sample-controller](https://github.com/kubernetes/sample-controller)编写。这里我们以创建一个TKE Namespace为例分析Namespace AA和Namespace Controller细节:

1、创建TKE Namespace：

```go
namespace1 := &businessv1.Namespace{
	ObjectMeta: metav1.ObjectMeta{
		Name:      "project1-namespace1",
		Namespace: "project1",
	},
	Spec: businessv1.NamespaceSpec{
		ClusterName: "cluster1",
		Hard: businessv1.ResourceList{
			"cpu":    resource.MustParse("300m"),
			"memory": resource.MustParse("450Mi"),
		},
		Namespace: "project1",
	},
	Status: businessv1.NamespaceStatus{
		Used: businessv1.ResourceList{
			"cpu":    resource.MustParse("200m"),
			"memory": resource.MustParse("300Mi"),
		},
	},
}
_, _ = businessClient.BusinessV1().Namespaces("project1").Create(namespace1)
```

2、Namespace AA接受到请求，并对Namespace对象进行预处理、有效性检查以及ETCD写操作，如下(github.com/tkestack/tke/pkg/business/registry/namespace/storage/storage.go:53)：

```go
// NewStorage returns a Storage object that will work against namespace sets.
func NewStorage(optsGetter genericregistry.RESTOptionsGetter, businessClient *businessinternalclient.BusinessClient, platformClient platformversionedclient.PlatformV1Interface, privilegedUsername string) *Storage {
	strategy := namespace.NewStrategy(businessClient, platformClient)
	store := &registry.Store{
		NewFunc:                  func() runtime.Object { return &business.Namespace{} },
		NewListFunc:              func() runtime.Object { return &business.NamespaceList{} },
		DefaultQualifiedResource: business.Resource("namespaces"),
		PredicateFunc:            namespace.MatchNamespace,

		CreateStrategy: strategy,
		UpdateStrategy: strategy,
		DeleteStrategy: strategy,
		ExportStrategy: strategy,

		ShouldDeleteDuringUpdate: shouldDeleteDuringUpdate,
	}
	options := &genericregistry.StoreOptions{
		RESTOptions: optsGetter,
		AttrFunc:    namespace.GetAttrs,
	}

	if err := store.CompleteWithOptions(options); err != nil {
		log.Panic("Failed to create namespace etcd rest storage", log.Err(err))
	}

	statusStore := *store
	statusStore.UpdateStrategy = namespace.NewStatusStrategy(strategy)
	statusStore.ExportStrategy = namespace.NewStatusStrategy(strategy)

	finalizeStore := *store
	finalizeStore.UpdateStrategy = namespace.NewFinalizeStrategy(strategy)
	finalizeStore.ExportStrategy = namespace.NewFinalizeStrategy(strategy)

	return &Storage{
		Namespace: &REST{store, privilegedUsername},
		Status:    &StatusREST{&statusStore},
		Finalize:  &FinalizeREST{&finalizeStore},
	}
}
...
// Validate validates a new namespace.
func (s *Strategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
	return ValidateNamespace(obj.(*business.Namespace), nil,
		validation.NewObjectGetter(s.businessClient), validation.NewClusterGetter(s.platformClient))
}
...
// PrepareForCreate is invoked on create before validation to normalize
// the object.
func (s *Strategy) PrepareForCreate(ctx context.Context, obj runtime.Object) {
	_, tenantID := authentication.GetUsernameAndTenantID(ctx)
	namespace, _ := obj.(*business.Namespace)
	if len(tenantID) != 0 {
		namespace.Spec.TenantID = tenantID
	}

	if namespace.Spec.ClusterName != "" && namespace.Spec.Namespace != "" {
		namespace.ObjectMeta.GenerateName = ""
		namespace.ObjectMeta.Name = fmt.Sprintf("%s-%s", namespace.Spec.ClusterName, namespace.Spec.Namespace)
	} else {
		namespace.ObjectMeta.GenerateName = "ns"
		namespace.ObjectMeta.Name = ""
	}

	namespace.Spec.Finalizers = []business.FinalizerName{
		business.NamespaceFinalize,
	}
}
```

3、Namespace Controller Watch到Namespace Add事件，对该事件进行处理如下(github.com/tkestack/tke/pkg/business/controller/namespace/namespace_controller.go:201)：

```go
// syncItem will sync the Namespace with the given key if it has had
// its expectations fulfilled, meaning it did not expect to see any more of its
// namespaces created or deleted. This function is not meant to be invoked
// concurrently with the same key.
func (c *Controller) syncItem(key string) error {
	startTime := time.Now()
	defer func() {
		log.Info("Finished syncing namespace", log.String("namespace", key), log.Duration("processTime", time.Since(startTime)))
	}()

	projectName, namespaceName, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	// Namespace holds the latest Namespace info from apiserver
	namespace, err := c.lister.Namespaces(projectName).Get(namespaceName)
	switch {
	case errors.IsNotFound(err):
		log.Info("Namespace has been deleted. Attempting to cleanup resources", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
		err = c.processDeletion(key)
	case err != nil:
		log.Warn("Unable to retrieve namespace from store", log.String("projectName", projectName), log.String("namespaceName", namespaceName), log.Err(err))
	default:
		if namespace.Status.Phase == v1.NamespacePending || namespace.Status.Phase == v1.NamespaceAvailable || namespace.Status.Phase == v1.NamespaceFailed {
			cachedNamespace := c.cache.getOrCreate(key)
			err = c.processUpdate(cachedNamespace, namespace, key)
		} else if namespace.Status.Phase == v1.NamespaceTerminating {
			log.Info("Namespace has been terminated. Attempting to cleanup resources", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
			_ = c.processDeletion(key)
			err = c.namespacedResourcesDeleter.Delete(projectName, namespaceName)
		} else {
			log.Debug(fmt.Sprintf("Namespace %s status is %s, not to process", key, namespace.Status.Phase))
		}
	}
	return err
}
```

由于默认是v1.NamespacePending状态，所以这里会跳转到processUpdate进行处理：

```go
...
func (c *Controller) processUpdate(cachedNamespace *cachedNamespace, namespace *v1.Namespace, key string) error {
	if cachedNamespace.state != nil {
		// exist and the namespace name changed
		if cachedNamespace.state.UID != namespace.UID {
			if err := c.processDelete(cachedNamespace, key); err != nil {
				return err
			}
		}
	}
	// start update machine if needed
	err := c.handlePhase(key, cachedNamespace, namespace)
	if err != nil {
		return err
	}
	cachedNamespace.state = namespace
	// Always update the cache upon success.
	c.cache.set(key, cachedNamespace)
	return nil
}

func (c *Controller) handlePhase(key string, cachedNamespace *cachedNamespace, namespace *v1.Namespace) error {
	switch namespace.Status.Phase {
	case v1.NamespacePending:
		if err := c.calculateProjectUsed(cachedNamespace, namespace); err != nil {
			return err
		}
		if err := c.createNamespaceOnCluster(namespace); err != nil {
			namespace.Status.Phase = v1.NamespaceFailed
			namespace.Status.Message = "CreateNamespaceOnClusterFailed"
			namespace.Status.Reason = err.Error()
			namespace.Status.LastTransitionTime = metav1.Now()
			return c.persistUpdate(namespace)
		}
		namespace.Status.Phase = v1.NamespaceAvailable
		namespace.Status.Message = ""
		namespace.Status.Reason = ""
		namespace.Status.LastTransitionTime = metav1.Now()
		return c.persistUpdate(namespace)
	case v1.NamespaceAvailable, v1.NamespaceFailed:
		c.startNamespaceHealthCheck(key)
	}
	return nil
}
```

这里会先将Namespace的资源限制添加到对应的Project的资源限制，并更新Project，这样Project的资源配额就下发到Namespace中去了：

```go
func (c *Controller) calculateProjectUsed(cachedNamespace *cachedNamespace, namespace *v1.Namespace) error {
	project, err := c.client.BusinessV1().Projects().Get(namespace.ObjectMeta.Namespace, metav1.GetOptions{})
	if err != nil {
		log.Error("Failed to get the project", log.String("projectName", namespace.ObjectMeta.Namespace), log.Err(err))
		return err
	}
	calculatedNamespaceNames := sets.NewString(project.Status.CalculatedNamespaces...)
	if !calculatedNamespaceNames.Has(project.ObjectMeta.Name) {
		project.Status.CalculatedNamespaces = append(project.Status.CalculatedNamespaces, namespace.ObjectMeta.Name)
		if project.Status.Clusters == nil {
			project.Status.Clusters = make(v1.ClusterUsed)
		}
		businessUtil.AddClusterHardToUsed(&project.Status.Clusters,
			v1.ClusterHard{
				namespace.Spec.ClusterName: v1.HardQuantity{
					Hard: namespace.Spec.Hard,
				},
			})
		return c.persistUpdateProject(project)
	}
	if cachedNamespace.state != nil && !reflect.DeepEqual(cachedNamespace.state.Spec.Hard, namespace.Spec.Hard) {
		if project.Status.Clusters == nil {
			project.Status.Clusters = make(v1.ClusterUsed)
		}
		// sub old
		businessUtil.SubClusterHardFromUsed(&project.Status.Clusters,
			v1.ClusterHard{
				namespace.Spec.ClusterName: v1.HardQuantity{
					Hard: cachedNamespace.state.Spec.Hard,
				},
			})
		// add new
		businessUtil.AddClusterHardToUsed(&project.Status.Clusters,
			v1.ClusterHard{
				namespace.Spec.ClusterName: v1.HardQuantity{
					Hard: namespace.Spec.Hard,
				},
			})
		return c.persistUpdateProject(project)
	}
	return nil
}
...
func (c *Controller) persistUpdateProject(project *v1.Project) error {
	var err error
	for i := 0; i < clientRetryCount; i++ {
		_, err = c.client.BusinessV1().Projects().UpdateStatus(project)
		if err == nil {
			return nil
		}
		if errors.IsNotFound(err) {
			log.Info("Not persisting update to projects that no longer exists", log.String("projectName", project.ObjectMeta.Name), log.Err(err))
			return nil
		}
		if errors.IsConflict(err) {
			return fmt.Errorf("not persisting update to projects '%s' that has been changed since we received it: %v", project.ObjectMeta.Name, err)
		}
		log.Warn(fmt.Sprintf("Failed to persist updated status of projects '%s/%s'", project.ObjectMeta.Name, project.Status.Phase), log.String("projectName", project.ObjectMeta.Name), log.Err(err))
		time.Sleep(clientRetryInterval)
	}
	return err
}
```

接着创建TKE Namespace对应的Kubernetes namespace以及相应的[namespace ResourceQuotas](https://k8smeetup.github.io/docs/tasks/administer-cluster/quota-memory-cpu-namespace/)，用于限制该namespace资源使用配额，如下：

```go
func createNamespaceOnCluster(kubeClient *kubernetes.Clientset, namespace *v1.Namespace) error {
	ns, err := kubeClient.CoreV1().Namespaces().Get(namespace.Spec.Namespace, metav1.GetOptions{})
	if err != nil && errors.IsNotFound(err) {
		// create namespace
		nsOnCluster := &corev1.Namespace{
			ObjectMeta: metav1.ObjectMeta{
				Name: namespace.Spec.Namespace,
				Labels: map[string]string{
					util.LabelProjectName:   namespace.ObjectMeta.Namespace,
					util.LabelNamespaceName: namespace.ObjectMeta.Name,
				},
			},
		}
		_, err := kubeClient.CoreV1().Namespaces().Create(nsOnCluster)
		if err != nil {
			log.Error("Failed to create the namespace on cluster", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
			return err
		}
		return nil
	}
	if err != nil {
		log.Error("Failed to get the namespace on cluster", log.String("namespace", namespace.Spec.Namespace), log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
		return err
	}
	projectName, ok := ns.ObjectMeta.Labels[util.LabelProjectName]
	if !ok {
		ns.Labels = make(map[string]string)
		ns.Labels[util.LabelProjectName] = namespace.ObjectMeta.Namespace
		ns.Labels[util.LabelNamespaceName] = namespace.ObjectMeta.Name
		_, err := kubeClient.CoreV1().Namespaces().Update(ns)
		if err != nil {
			log.Error("Failed to update the namespace on cluster", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
			return err
		}
		return nil
	}
	if projectName != namespace.ObjectMeta.Namespace {
		log.Error("The namespace in the cluster already belongs to another project and cannot be attributed to this project", log.String("clusterName", namespace.Spec.ClusterName), log.String("namespace", namespace.Spec.Namespace))
		return fmt.Errorf("namespace in the cluster already belongs to another project(%s) and cannot be attributed to this project(%s)", projectName, namespace.ObjectMeta.Namespace)
	}
	return nil
}
...
func createResourceQuotaOnCluster(kubeClient *kubernetes.Clientset, namespace *v1.Namespace) error {
	resourceQuota, err := kubeClient.CoreV1().ResourceQuotas(namespace.Spec.Namespace).Get(namespace.Spec.Namespace, metav1.GetOptions{})
	if err != nil && errors.IsNotFound(err) {
		// create resource quota
		resourceList := resource.ConvertToCoreV1ResourceList(namespace.Spec.Hard)
		rq := &corev1.ResourceQuota{
			ObjectMeta: metav1.ObjectMeta{
				Name:      namespace.Spec.Namespace,
				Namespace: namespace.Spec.Namespace,
			},
			Spec: corev1.ResourceQuotaSpec{
				Hard: resourceList,
			},
		}
		_, err := kubeClient.CoreV1().ResourceQuotas(namespace.Spec.Namespace).Create(rq)
		if err != nil {
			log.Error("Failed to create the resource quota on cluster", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
			return err
		}
		return nil
	}
	if err != nil {
		log.Error("Failed to get the resource quota on cluster", log.String("namespace", namespace.Spec.Namespace), log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
		return err
	}
	resourceList := resource.ConvertToCoreV1ResourceList(namespace.Spec.Hard)
	if !reflect.DeepEqual(resourceQuota.Spec.Hard, resourceList) {
		resourceQuota.Spec.Hard = resourceList
		_, err := kubeClient.CoreV1().ResourceQuotas(namespace.Spec.Namespace).Update(resourceQuota)
		if err != nil {
			log.Error("Failed to update the resource quota on cluster", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
			return err
		}
	}
	return nil
}
```

最后更新TKE Namespace状态，如下：

```go
func (c *Controller) handlePhase(key string, cachedNamespace *cachedNamespace, namespace *v1.Namespace) error {
	switch namespace.Status.Phase {
	case v1.NamespacePending:
		if err := c.calculateProjectUsed(cachedNamespace, namespace); err != nil {
			return err
		}
		if err := c.createNamespaceOnCluster(namespace); err != nil {
			namespace.Status.Phase = v1.NamespaceFailed
			namespace.Status.Message = "CreateNamespaceOnClusterFailed"
			namespace.Status.Reason = err.Error()
			namespace.Status.LastTransitionTime = metav1.Now()
			return c.persistUpdate(namespace)
		}
		namespace.Status.Phase = v1.NamespaceAvailable
		namespace.Status.Message = ""
		namespace.Status.Reason = ""
		namespace.Status.LastTransitionTime = metav1.Now()
		return c.persistUpdate(namespace)
	case v1.NamespaceAvailable, v1.NamespaceFailed:
		c.startNamespaceHealthCheck(key)
	}
	return nil
}
...
func (c *Controller) persistUpdate(namespace *v1.Namespace) error {
	var err error
	for i := 0; i < clientRetryCount; i++ {
		_, err = c.client.BusinessV1().Namespaces(namespace.ObjectMeta.Namespace).UpdateStatus(namespace)
		if err == nil {
			return nil
		}
		if errors.IsNotFound(err) {
			log.Info("Not persisting update to namespace that no longer exists", log.String("projectName", namespace.ObjectMeta.Namespace), log.String("namespaceName", namespace.ObjectMeta.Name), log.Err(err))
			return nil
		}
		if errors.IsConflict(err) {
			return fmt.Errorf("not persisting update to namespace '%s/%s' that has been changed since we received it: %v", namespace.ObjectMeta.Namespace, namespace.ObjectMeta.Name, err)
		}
		log.Warn(fmt.Sprintf("Failed to persist updated status of namespace '%s/%s/%s'", namespace.ObjectMeta.Namespace, namespace.ObjectMeta.Name, namespace.Status.Phase), log.String("namespaceName", namespace.ObjectMeta.Name), log.Err(err))
		time.Sleep(clientRetryInterval)
	}
	return err
}
```

4、Namespace AA接受到Update请求，进行预处理并检查有效性，最后对该Namespace进行ETCD Update操作，处理如下：

```go
// NewStorage returns a Storage object that will work against namespace sets.
func NewStorage(optsGetter genericregistry.RESTOptionsGetter, businessClient *businessinternalclient.BusinessClient, platformClient platformversionedclient.PlatformV1Interface, privilegedUsername string) *Storage {
	strategy := namespace.NewStrategy(businessClient, platformClient)
	store := &registry.Store{
		NewFunc:                  func() runtime.Object { return &business.Namespace{} },
		NewListFunc:              func() runtime.Object { return &business.NamespaceList{} },
		DefaultQualifiedResource: business.Resource("namespaces"),
		PredicateFunc:            namespace.MatchNamespace,

		CreateStrategy: strategy,
		UpdateStrategy: strategy,
		DeleteStrategy: strategy,
		ExportStrategy: strategy,

		ShouldDeleteDuringUpdate: shouldDeleteDuringUpdate,
	}
	options := &genericregistry.StoreOptions{
		RESTOptions: optsGetter,
		AttrFunc:    namespace.GetAttrs,
	}

	if err := store.CompleteWithOptions(options); err != nil {
		log.Panic("Failed to create namespace etcd rest storage", log.Err(err))
	}

	statusStore := *store
	statusStore.UpdateStrategy = namespace.NewStatusStrategy(strategy)
	statusStore.ExportStrategy = namespace.NewStatusStrategy(strategy)

	finalizeStore := *store
	finalizeStore.UpdateStrategy = namespace.NewFinalizeStrategy(strategy)
	finalizeStore.ExportStrategy = namespace.NewFinalizeStrategy(strategy)

	return &Storage{
		Namespace: &REST{store, privilegedUsername},
		Status:    &StatusREST{&statusStore},
		Finalize:  &FinalizeREST{&finalizeStore},
	}
}
...
// PrepareForUpdate is invoked on update before validation to normalize
// the object.  For example: remove fields that are not to be persisted,
// sort order-insensitive list fields, etc.  This should not remove fields
// whose presence would be considered a validation error.
func (StatusStrategy) PrepareForUpdate(ctx context.Context, obj, old runtime.Object) {
	newNamespace := obj.(*business.Namespace)
	oldNamespace := old.(*business.Namespace)
	newNamespace.Spec = oldNamespace.Spec
}

// ValidateUpdate is invoked after default fields in the object have been
// filled in before the object is persisted.  This method should not mutate
// the object.
func (s *StatusStrategy) ValidateUpdate(ctx context.Context, obj, old runtime.Object) field.ErrorList {
	return ValidateNamespaceUpdate(obj.(*business.Namespace), old.(*business.Namespace),
		validation.NewObjectGetter(s.businessClient), validation.NewClusterGetter(s.platformClient))
}

// Update alters the status subset of an object.
func (r *StatusREST) Update(ctx context.Context, name string, objInfo rest.UpdatedObjectInfo, createValidation rest.ValidateObjectFunc, updateValidation rest.ValidateObjectUpdateFunc, forceAllowCreate bool, options *metav1.UpdateOptions) (runtime.Object, bool, error) {
	// We are explicitly setting forceAllowCreate to false in the call to the underlying storage because
	// subresources should never allow create on update.
	_, err := ValidateGetObjectAndTenantID(ctx, r.store, name, &metav1.GetOptions{})
	if err != nil {
		return nil, false, err
	}
	return r.store.Update(ctx, name, objInfo, createValidation, updateValidation, false, options)
}
```

5、在Update之后，Namespace Controller Watch到该事件，由于这个时候该Namespace对象的namespace.Status.Phase为NamespaceFailed或者NamespaceAvailable，所以会进入`processUpdate`处理：

```go
func (c *Controller) handlePhase(key string, cachedNamespace *cachedNamespace, namespace *v1.Namespace) error {
	switch namespace.Status.Phase {
	case v1.NamespacePending:
		if err := c.calculateProjectUsed(cachedNamespace, namespace); err != nil {
			return err
		}
		if err := c.createNamespaceOnCluster(namespace); err != nil {
			namespace.Status.Phase = v1.NamespaceFailed
			namespace.Status.Message = "CreateNamespaceOnClusterFailed"
			namespace.Status.Reason = err.Error()
			namespace.Status.LastTransitionTime = metav1.Now()
			return c.persistUpdate(namespace)
		}
		namespace.Status.Phase = v1.NamespaceAvailable
		namespace.Status.Message = ""
		namespace.Status.Reason = ""
		namespace.Status.LastTransitionTime = metav1.Now()
		return c.persistUpdate(namespace)
	case v1.NamespaceAvailable, v1.NamespaceFailed:
		c.startNamespaceHealthCheck(key)
	}
	return nil
}
...
func (c *Controller) startNamespaceHealthCheck(key string) {
	if !c.health.Exist(key) {
		c.health.Set(key)
		go func() {
			if err := wait.PollImmediateUntil(1*time.Minute, c.watchNamespaceHealth(key), c.stopCh); err != nil {
				log.Error("Failed to wait poll immediate until", log.Err(err))
			}
		}()
		log.Info("Namespace phase start new health check", log.String("namespace", key))
	} else {
		log.Info("Namespace phase health check exit", log.String("namespace", key))
	}
}
...
// for PollImmediateUntil, when return true ,an err while exit
func (c *Controller) watchNamespaceHealth(key string) func() (bool, error) {
	return func() (bool, error) {
		log.Debug("Check namespace health", log.String("namespace", key))

		if !c.health.Exist(key) {
			return true, nil
		}

		projectName, namespaceName, err := cache.SplitMetaNamespaceKey(key)
		if err != nil {
			log.Error("Failed to split meta namespace key", log.String("key", key))
			c.health.Del(key)
			return true, nil
		}
		namespace, err := c.client.BusinessV1().Namespaces(projectName).Get(namespaceName, metav1.GetOptions{})
		if err != nil && errors.IsNotFound(err) {
			log.Error("Namespace not found, to exit the health check loop", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
			c.health.Del(key)
			return true, nil
		}
		if err != nil {
			log.Error("Check namespace health, namespace get failed", log.String("projectName", projectName), log.String("namespaceName", namespaceName), log.Err(err))
			return false, nil
		}
		// if status is terminated,to exit the  health check loop
		if namespace.Status.Phase == v1.NamespaceTerminating || namespace.Status.Phase == v1.NamespacePending {
			log.Warn("Namespace status is terminated, to exit the health check loop", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
			c.health.Del(key)
			return true, nil
		}

		if err := c.checkNamespaceHealth(namespace); err != nil {
			log.Error("Failed to check namespace health", log.String("projectName", projectName), log.String("namespaceName", namespaceName), log.Err(err))
		}
		return false, nil
	}
}
...
func (c *Controller) checkNamespaceHealth(namespace *v1.Namespace) error {
	// build client
	kubeClient, err := util.BuildExternalClientSetWithName(c.platformClient, namespace.Spec.ClusterName)
	if err != nil {
		return err
	}
	message, reason := checkNamespaceOnCluster(kubeClient, namespace)
	if message != "" {
		namespace.Status.Phase = v1.NamespaceFailed
		namespace.Status.LastTransitionTime = metav1.Now()
		namespace.Status.Message = message
		namespace.Status.Reason = reason
		return c.persistUpdate(namespace)
	}
	message, reason, used := calculateNamespaceUsed(kubeClient, namespace)
	if message != "" {
		namespace.Status.Phase = v1.NamespaceFailed
		namespace.Status.LastTransitionTime = metav1.Now()
		namespace.Status.Message = message
		namespace.Status.Reason = reason
		return c.persistUpdate(namespace)
	}
	if namespace.Status.Phase != v1.NamespaceAvailable || !reflect.DeepEqual(namespace.Status.Used, used) {
		if namespace.Status.Phase != v1.NamespaceAvailable {
			namespace.Status.LastTransitionTime = metav1.Now()
		}
		namespace.Status.Phase = v1.NamespaceAvailable
		namespace.Status.Used = used
		namespace.Status.Message = ""
		namespace.Status.Reason = ""
		return c.persistUpdate(namespace)
	}
	return nil
}
```

这里checkNamespaceHealth会检查Namespace对应的Kubernetes namespace是否存在，检查namespace ResourceQuota是否存在，并更新正确的namespace.Status.Used

之后会设置c.health cache，以便后续不会重复执行健康检查操作：

```go
func (c *Controller) startNamespaceHealthCheck(key string) {
	if !c.health.Exist(key) {
		c.health.Set(key)
		go func() {
			if err := wait.PollImmediateUntil(1*time.Minute, c.watchNamespaceHealth(key), c.stopCh); err != nil {
				log.Error("Failed to wait poll immediate until", log.Err(err))
			}
		}()
		log.Info("Namespace phase start new health check", log.String("namespace", key))
	} else {
		log.Info("Namespace phase health check exit", log.String("namespace", key))
	}
}
```

之后如果用户删除该TKE Namespace，则进入Delete函数(github.com/tkestack/tke/pkg/business/registry/namespace/storage/storage.go:171)：

```go
// Delete enforces life-cycle rules for cluster termination
func (r *REST) Delete(ctx context.Context, name string, deleteValidation rest.ValidateObjectFunc, options *metav1.DeleteOptions) (runtime.Object, bool, error) {
	nsObj, err := ValidateGetObjectAndTenantID(ctx, r.Store, name, &metav1.GetOptions{})
	if err != nil {
		return nil, false, err
	}

	ns := nsObj.(*business.Namespace)

	// Ensure we have a UID precondition
	if options == nil {
		options = metav1.NewDeleteOptions(0)
	}
	if options.Preconditions == nil {
		options.Preconditions = &metav1.Preconditions{}
	}
	if options.Preconditions.UID == nil {
		options.Preconditions.UID = &ns.UID
	} else if *options.Preconditions.UID != ns.UID {
		err = errors.NewConflict(
			business.Resource("namespaces"),
			name,
			fmt.Errorf("precondition failed: UID in precondition: %v, UID in object meta: %v", *options.Preconditions.UID, ns.UID),
		)
		return nil, false, err
	}
	if options.Preconditions.ResourceVersion != nil && *options.Preconditions.ResourceVersion != ns.ResourceVersion {
		err = errors.NewConflict(
			business.Resource("namespaces"),
			name,
			fmt.Errorf("precondition failed: ResourceVersion in precondition: %v, ResourceVersion in object meta: %v", *options.Preconditions.ResourceVersion, ns.ResourceVersion),
		)
		return nil, false, err
	}

	if ns.DeletionTimestamp.IsZero() {
		key, err := r.Store.KeyFunc(ctx, name)
		if err != nil {
			return nil, false, err
		}

		preconditions := storage.Preconditions{UID: options.Preconditions.UID, ResourceVersion: options.Preconditions.ResourceVersion}

		out := r.Store.NewFunc()
		err = r.Store.Storage.GuaranteedUpdate(
			ctx, key, out, false, &preconditions,
			storage.SimpleUpdate(func(existing runtime.Object) (runtime.Object, error) {
				existingNamespace, ok := existing.(*business.Namespace)
				if !ok {
					// wrong type
					return nil, fmt.Errorf("expected *business.Namespace, got %v", existing)
				}
				if err := deleteValidation(ctx, existingNamespace); err != nil {
					return nil, err
				}
				// Set the deletion timestamp if needed
				if existingNamespace.DeletionTimestamp.IsZero() {
					now := metav1.Now()
					existingNamespace.DeletionTimestamp = &now
				}
				// Set the namespace phase to terminating, if needed
				if existingNamespace.Status.Phase != business.NamespaceTerminating {
					existingNamespace.Status.Phase = business.NamespaceTerminating
				}

				// the current finalizers which are on namespace
				currentFinalizers := map[string]bool{}
				for _, f := range existingNamespace.Finalizers {
					currentFinalizers[f] = true
				}
				// the finalizers we should ensure on namespace
				shouldHaveFinalizers := map[string]bool{
					metav1.FinalizerOrphanDependents: apiserverutil.ShouldHaveOrphanFinalizer(options, currentFinalizers[metav1.FinalizerOrphanDependents]),
					metav1.FinalizerDeleteDependents: apiserverutil.ShouldHaveDeleteDependentsFinalizer(options, currentFinalizers[metav1.FinalizerDeleteDependents]),
				}
				// determine whether there are changes
				changeNeeded := false
				for finalizer, shouldHave := range shouldHaveFinalizers {
					changeNeeded = currentFinalizers[finalizer] != shouldHave || changeNeeded
					if shouldHave {
						currentFinalizers[finalizer] = true
					} else {
						delete(currentFinalizers, finalizer)
					}
				}
				// make the changes if needed
				if changeNeeded {
					var newFinalizers []string
					for f := range currentFinalizers {
						newFinalizers = append(newFinalizers, f)
					}
					existingNamespace.Finalizers = newFinalizers
				}
				return existingNamespace, nil
			}),
			dryrun.IsDryRun(options.DryRun),
		)

		if err != nil {
			err = storageerr.InterpretGetError(err, business.Resource("namespaces"), name)
			err = storageerr.InterpretUpdateError(err, business.Resource("namespaces"), name)
			if _, ok := err.(*errors.StatusError); !ok {
				err = errors.NewInternalError(err)
			}
			return nil, false, err
		}

		return out, false, nil
	}

	// prior to final deletion, we must ensure that finalizers is empty
	if len(ns.Spec.Finalizers) != 0 {
		err = errors.NewConflict(business.Resource("namespaces"), ns.Name, fmt.Errorf("the system is ensuring all content is removed from this namespace.  Upon completion, this namespace will automatically be purged by the system"))
		return nil, false, err
	}
	return r.Store.Delete(ctx, name, deleteValidation, options)
}
```

这里会Update TKE Namespace.Status.Phase = business.NamespaceTerminating，TKE Namespace Conroller接受到该事件，进行处理如下：

```go
// syncItem will sync the Namespace with the given key if it has had
// its expectations fulfilled, meaning it did not expect to see any more of its
// namespaces created or deleted. This function is not meant to be invoked
// concurrently with the same key.
func (c *Controller) syncItem(key string) error {
	startTime := time.Now()
	defer func() {
		log.Info("Finished syncing namespace", log.String("namespace", key), log.Duration("processTime", time.Since(startTime)))
	}()

	projectName, namespaceName, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}
	// Namespace holds the latest Namespace info from apiserver
	namespace, err := c.lister.Namespaces(projectName).Get(namespaceName)
	switch {
	case errors.IsNotFound(err):
		log.Info("Namespace has been deleted. Attempting to cleanup resources", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
		err = c.processDeletion(key)
	case err != nil:
		log.Warn("Unable to retrieve namespace from store", log.String("projectName", projectName), log.String("namespaceName", namespaceName), log.Err(err))
	default:
		if namespace.Status.Phase == v1.NamespacePending || namespace.Status.Phase == v1.NamespaceAvailable || namespace.Status.Phase == v1.NamespaceFailed {
			cachedNamespace := c.cache.getOrCreate(key)
			err = c.processUpdate(cachedNamespace, namespace, key)
		} else if namespace.Status.Phase == v1.NamespaceTerminating {
			log.Info("Namespace has been terminated. Attempting to cleanup resources", log.String("projectName", projectName), log.String("namespaceName", namespaceName))
			_ = c.processDeletion(key)
			err = c.namespacedResourcesDeleter.Delete(projectName, namespaceName)
		} else {
			log.Debug(fmt.Sprintf("Namespace %s status is %s, not to process", key, namespace.Status.Phase))
		}
	}
	return err
}
```

首先删除cache数据：

```go
func (c *Controller) processDeletion(key string) error {
	cachedNamespace, ok := c.cache.get(key)
	if !ok {
		log.Debug("Namespace not in cache even though the watcher thought it was. Ignoring the deletion", log.String("name", key))
		return nil
	}
	return c.processDelete(cachedNamespace, key)
}

func (c *Controller) processDelete(cachedNamespace *cachedNamespace, key string) error {
	log.Info("Namespace will be dropped", log.String("name", key))

	if c.cache.Exist(key) {
		log.Info("Delete the namespace cache", log.String("name", key))
		c.cache.delete(key)
	}

	if c.health.Exist(key) {
		log.Info("Delete the namespace health cache", log.String("name", key))
		c.health.Del(key)
	}

	return nil
}
```

接着删除TKE Namespace对应资源，如下：

```go
// Delete deletes all resources in the given namespace.
// Before deleting resources:
// * It ensures that deletion timestamp is set on the
//   namespace (does nothing if deletion timestamp is missing).
// * Verifies that the namespace is in the "terminating" phase
//   (updates the namespace phase if it is not yet marked terminating)
// After deleting the resources:
// * It removes finalizer token from the given namespace.
// * Deletes the namespace if deleteNamespaceWhenDone is true.
//
// Returns an error if any of those steps fail.
// Returns ResourcesRemainingError if it deleted some resources but needs
// to wait for them to go away.
// Caller is expected to keep calling this until it succeeds.
func (d *namespacedResourcesDeleter) Delete(projectName string, namespaceName string) error {
	// Multiple controllers may edit a namespace during termination
	// first get the latest state of the namespace before proceeding
	// if the namespace was deleted already, don't do anything
	namespace, err := d.businessClient.Namespaces(projectName).Get(namespaceName, metav1.GetOptions{})
	if err != nil {
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}
	if namespace.DeletionTimestamp == nil {
		return nil
	}

	log.Infof("namespace controller - syncNamespace - namespace: %s, finalizerToken: %s", namespace.Name, d.finalizerToken)

	// ensure that the status is up to date on the namespace
	// if we get a not found error, we assume the namespace is truly gone
	namespace, err = d.retryOnConflictError(namespace, d.updateNamespaceStatusFunc)
	if err != nil {
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}

	// the latest view of the namespace asserts that namespace is no longer deleting..
	if namespace.DeletionTimestamp.IsZero() {
		return nil
	}

	// Delete the namespace if it is already finalized.
	if d.deleteNamespaceWhenDone && finalized(namespace) {
		return d.deleteNamespace(namespace)
	}

	// there may still be content for us to remove
	err = d.deleteAllContent(namespace)
	if err != nil {
		return err
	}

	// we have removed content, so mark it finalized by us
	namespace, err = d.retryOnConflictError(namespace, d.finalizeNamespace)
	if err != nil {
		// in normal practice, this should not be possible, but if a deployment is running
		// two controllers to do namespace deletion that share a common finalizer token it's
		// possible that a not found could occur since the other controller would have finished the delete.
		if errors.IsNotFound(err) {
			return nil
		}
		return err
	}

	// Check if we can delete now.
	if d.deleteNamespaceWhenDone && finalized(namespace) {
		return d.deleteNamespace(namespace)
	}
	return nil
}
```

删除TKE Namespace：

```go
...
// Deletes the given namespace.
func (d *namespacedResourcesDeleter) deleteNamespace(namespace *v1.Namespace) error {
	var opts *metav1.DeleteOptions
	uid := namespace.UID
	if len(uid) > 0 {
		opts = &metav1.DeleteOptions{Preconditions: &metav1.Preconditions{UID: &uid}}
	}
	err := d.businessClient.Namespaces(namespace.ObjectMeta.Namespace).Delete(namespace.ObjectMeta.Name, opts)
	if err != nil && !errors.IsNotFound(err) {
		return err
	}
	return nil
}
```

重新调整对应Project的资源配额：

```go
// deleteAllContent will use the dynamic client to delete each resource identified in groupVersionResources.
// It returns an estimate of the time remaining before the remaining resources are deleted.
// If estimate > 0, not all resources are guaranteed to be gone.
func (d *namespacedResourcesDeleter) deleteAllContent(namespace *v1.Namespace) error {
	log.Debug("Namespace controller - deleteAllContent", log.String("namespaceName", namespace.ObjectMeta.Name))

	var errs []error
	for _, deleteFunc := range deleteResourceFuncs {
		err := deleteFunc(d, namespace)
		if err != nil {
			// If there is an error, hold on to it but proceed with all the remaining resource.
			errs = append(errs, err)
		}
	}

	if len(errs) > 0 {
		return utilerrors.NewAggregate(errs)
	}

	log.Debug("Namespace controller - deletedAllContent", log.String("namespaceName", namespace.ObjectMeta.Name))
	return nil
}

var deleteResourceFuncs = []deleteResourceFunc{
	recalculateProjectUsed,
	deleteNamespaceOnCluster,
}

func recalculateProjectUsed(deleter *namespacedResourcesDeleter, namespace *v1.Namespace) error {
	log.Debug("Namespace controller - recalculateProjectUsed", log.String("namespaceName", namespace.ObjectMeta.Name))

	project, err := deleter.businessClient.Projects().Get(namespace.ObjectMeta.Namespace, metav1.GetOptions{})
	if err != nil {
		log.Error("Failed to get the project", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("projectName", namespace.ObjectMeta.Namespace), log.Err(err))
		return err
	}
	calculatedNamespaceNames := sets.NewString(project.Status.CalculatedNamespaces...)
	if calculatedNamespaceNames.Has(namespace.ObjectMeta.Name) {
		calculatedNamespaceNames.Delete(namespace.ObjectMeta.Name)
		project.Status.CalculatedNamespaces = calculatedNamespaceNames.List()
		if project.Status.Clusters != nil {
			clusterUsed, clusterUsedExist := project.Status.Clusters[namespace.Spec.ClusterName]
			if clusterUsedExist {
				for k, v := range namespace.Spec.Hard {
					usedValue, ok := clusterUsed.Used[k]
					if ok {
						usedValue.Sub(v)
						clusterUsed.Used[k] = usedValue
					}
				}
				project.Status.Clusters[namespace.Spec.ClusterName] = clusterUsed
			}
		}
		_, err := deleter.businessClient.Projects().Update(project)
		if err != nil {
			log.Error("Failed to update the project status", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("projectName", namespace.ObjectMeta.Namespace), log.Err(err))
			return err
		}
	}

	return nil
}
```

最后删除TKE Namespace对应的Kubernetes namespace：

```go
func deleteNamespaceOnCluster(deleter *namespacedResourcesDeleter, namespace *v1.Namespace) error {
	kubeClient, err := platformutil.BuildExternalClientSetWithName(deleter.platformClient, namespace.Spec.ClusterName)
	if err != nil {
		log.Error("Failed to create the kubernetes client", log.String("namespaceName", namespace.ObjectMeta.Name), log.String("clusterName", namespace.Spec.ClusterName), log.Err(err))
		return err
	}
	ns, err := kubeClient.CoreV1().Namespaces().Get(namespace.Spec.Namespace, metav1.GetOptions{})
	if err != nil && errors.IsNotFound(err) {
		return nil
	}
	projectName, ok := ns.ObjectMeta.Labels[util.LabelProjectName]
	if !ok {
		return fmt.Errorf("no project label were found on the namespace within the business cluster")
	}
	if projectName != namespace.ObjectMeta.Namespace {
		return fmt.Errorf("the namespace in the business cluster currently belongs to another project")
	}
	background := metav1.DeletePropagationBackground
	deleteOpt := &metav1.DeleteOptions{PropagationPolicy: &background}
	if err := kubeClient.CoreV1().Namespaces().Delete(namespace.Spec.Namespace, deleteOpt); err != nil {
		log.Error("Failed to delete the namespace in the business cluster", log.String("clusterName", namespace.Spec.ClusterName), log.String("namespace", namespace.Spec.Namespace), log.Err(err))
		return err
	}
	return nil
}
```
    
通过上述设计，就可以将TKE Project的资源配额分配到TKE Namespace，而TKE Namespace对应Kubernetes的namespace，namespace通过quota-memory-cpu-namespace进行实际的集群资源控制

## 总结

上述对TKE business模块的Namespace CR进行了分析，其它模块原理类似，都是采用Kubernetes标准的Aggregated APIServer和Controller设计，代码精炼且优雅，不失为Kubernetes AA的最佳实践……

## Refs

* [tke](https://github.com/tkestack/tke)
* [What's the difference between AA and CRDs](https://github.com/kubernetes-sigs/apiserver-builder-alpha/blob/master/docs/compare_with_kubebuilder.md)

![](/public/img/duyanghao.png)