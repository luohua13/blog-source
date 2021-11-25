---
title: kubernetes apiserver详解
tags: [apiserver, go-restful]
categories: kubernetes
toc: true
---
本章将从kubernetes源码层次，解析apiserver流程中的关键要点。
<!--more-->
### 启动流程
同kubernetes所有的组件启动代码一样，apiserver使用cobra命令行方式启动

```golang
RunE: func(cmd *cobra.Command, args []string) error {
			// set default options
			completedOptions, err := Complete(s)
			if err != nil {
				return err
			}
			// validate options
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}

			return Run(completedOptions, stopCh)
		},
```
接下来的代码主线：
CreateServerChain  -->  CreateKubeAPIServer --> kubeAPIServerConfig.Complete().New() 
--> m.InstallLegacyAPI  -->  NewLegacyRESTStorage()  --> InstallLegacyAPIGroup  --> 
--> installAPIResources  --> InstallREST -->  installer.Install()  -->  registerResourceHandlers
以上代码主线可以对照源码进行阅读，本文将对其中比较关键的几步进行解析。

### NewLegacyRESTStorage
通过NewLegacyRESTStorage方法创建各个资源的RESTStorage。RESTStorage是一个结构体，具体的定义在vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go下，结构体内主要包含NewFunc返回特定资源信息、NewListFunc返回特定资源列表、CreateStrategy特定资源创建时的策略、UpdateStrategy更新时的策略以及DeleteStrategy删除时的策略等重要方法。
以configmap为例：
```golang
configMapStorage := configmapstore.NewREST(restOptionsGetter)
```
||
```golang
// NewREST returns a RESTStorage object that will work with ConfigMap objects.
func NewREST(optsGetter generic.RESTOptionsGetter) *REST {
	...
	return &REST{store}
}
```
常见的像 pod、event、secret、namespace、endpoints等，统一调用NewREST方法构造相应的资源。待所有资源的store创建完成之后，使用restStorageMap的Map类型将每个资源的路由和对应的store对应起来，方便后续去做路由的统一规划

```golang
restStorageMap := map[string]rest.Storage{
    ...
    "configMaps":                    configMapStorage,
}
apiGroupInfo.VersionedResourcesStorageMap["v1"] = restStorageMap
return restStorage, apiGroupInfo, nil
```
然后再到 installer.Install()-->registerResourceHandlers

### registerResourceHandlers

从restStorageMap取出每个storage，然后执行 registerResourceHandlers

```golang
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService){
    ...
    creater, isCreater := storage.(rest.Creater) //以create为例
    ...
    actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
    ...
    switch action.Verb {
        case "POST": // Create a resource.
            handler = restfulCreateResource(creater, reqScope, admit) //这里只列举一种情况
            route := ws.POST(action.Path).To(handler)  //匹配go-restful设计模式
        ...
    }

}
```
接下来重点分析handler

### handler
handler是apiserver的核心，是所有对apisever的请求的执行体。
handler参数的调用最终都会走到createHandler方法处，位于kubernetes/vendor/k8s.io/apiserver/pkg/endpoints/handlers/crete.go下。最核心的步骤即调用了Create方法
```golang
result, err := finishRequest(timeout, func() (runtime.Object, error) {
			return r.Create(   //r=creater
				ctx,
				name,
				obj,
				rest.AdmissionToValidateObjectFunc(admit, admissionAttributes),
				options,
			)
		})
```
回到creater.Create() --> func (e *Store) Create()
最终调用的是位于kubernetes/staging/src/k8s.io/apiserver/pkg/registry/generic/registry/store.go下的Create方法。
POST对应的是e.Storage.Create(）--> Storage.Create()
||
即store.Storage.Storage.Create(）
然后回到NewREST方法中:
```golang
options := &generic.StoreOptions{RESTOptions: optsGetter}
store.CompleteWithOptions(options)  
```
CompleteWithOptions
```golang
opts, err := options.RESTOptions.GetRESTOptions(e.DefaultQualifiedResource)
e.Storage.Storage, e.DestroyFunc = opts.Decorator(
			opts.StorageConfig,
			e.NewFunc(),
			prefix,
			keyFunc,
			e.NewListFunc,
			attrFunc,
			triggerFunc,
		)
```
||

stare.Storage.Storage = optsGetter.RESTOptions.GetRESTOptions().Decorator

回到optsGetter的构建：

optsGetter --> restOptionsGetter --> c.GenericConfig.RESTOptionsGetter

一步步走下去，最终调用位于createAPIExtensionsConfig函数体中：
```golang
genericConfig.RESTOptionsGetter = &genericoptions.SimpleRestOptionsFactory{Options: etcdOptions}
```

然后回到GetRESTOptions 方法：
func (f *SimpleRestOptionsFactory) GetRESTOptions() 
```golang

ret.Decorator = genericregistry.StorageWithCacher(cacheSize)
return ret
```
如此便十分明确代码逻辑：：
stare.Storage.Storage = genericregistry.StorageWithCacher(cacheSize)

接下来create方法正式登场：

### 数据库操作
ApiServer与数据库的交互主要指的是与etcd的交互。Kubernetes所有的组件不直接与etcd交互，都是通过请求apiserver，apiserver与etcd进行交互完成数据的最终落盘。
接着上面的代码进行分析：

StorageWithCacher --> NewRawStorage --> factory.Create(*config)

factory.Create(*config)代码如下，返回stare.Storage.Storage
```golang
// Create creates a storage backend based on given config.
func Create(c storagebackend.Config) (storage.Interface, DestroyFunc, error) {
	switch c.Type {
	case "etcd2":
		return nil, nil, fmt.Errorf("%v is no longer a supported storage backend", c.Type)
	case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD3:
		return newETCD3Storage(c)
	default:
		return nil, nil, fmt.Errorf("unknown storage type: %s", c.Type)
	}
}
```
通过Create方法判断是创建etcd2或是etcd3的后端etcd版本，目前版本默认的是etcd3。

由newETCD3Storage --> etcd3.New(） --> newStore()
得知最终store的create方法：
```golang
// Create implements storage.Interface.Create.
func (s *store) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
	if version, err := s.versioner.ObjectResourceVersion(obj); err == nil && version != 0 {
		return errors.New("resourceVersion should not be set on objects to be created")
	}
	if err := s.versioner.PrepareObjectForStorage(obj); err != nil {
		return fmt.Errorf("PrepareObjectForStorage failed: %v", err)
	}
	data, err := runtime.Encode(s.codec, obj)        //调用Encode方法序列化；
	if err != nil {
		return err
	}
	key = path.Join(s.pathPrefix, key)              //调用Encode方法序列化；

	opts, err := s.ttlOpts(ctx, int64(ttl))
	if err != nil {
		return err
	}

	newData, err := s.transformer.TransformToStorage(data, authenticatedDataString(key))  //调用TransformToStorage将数据类型进行转换；
	if err != nil {
		return storage.NewInternalError(err.Error())
	}

	txnResp, err := s.client.KV.Txn(ctx).If(                //调用客户端方法进行etcd的写入操作
		notFound(key),
	).Then(
		clientv3.OpPut(key, string(newData), opts...),
	).Commit()
	if err != nil {
		return err
	}
	if !txnResp.Succeeded {
		return storage.NewKeyExistsError(key, 0)
	}

	if out != nil {
		putResp := txnResp.Responses[0].GetResponsePut()
		return decode(s.codec, s.versioner, data, out, putResp.Header.Revision)
	}
	return nil
}
```
至此，完成从路由注册到handler处理以及对应的etcd数据库操作的绑定，完成整个路由后端的操作步骤。

### 总结
本文主要对apiserver中以api开头的资源路由流程进行讲解，还有以apis开头的、权限相关、RBAC启动等相关模块，但是只要理解这部分，对其它模块的了解将不是难事。

### 彩蛋

构建一个扩展的API server考虑两种方式：
- CRDs
- Apiserver-builder

下文摘自https://github.com/kubernetes/sample-apiserver：
> You may use this code if you want to build an Extension API Server to use with API Aggregation, or to build a stand-alone Kubernetes-style API server.
> However, consider two other options:
> CRDs: if you just want to add a resource to your kubernetes cluster, then consider using Custom Resource Definition a.k.a CRDs. They require less coding and rebasing. Read about the differences between Custom Resource Definitions vs Extension API Servers here.
> Apiserver-builder: If you want to build an Extension API server, consider using apiserver-builder instead of this repo. The Apiserver-builder is a complete framework for generating the apiserver, client libraries, and the installation program.

在下一篇文章中将对扩展Apiserver的构建进行讲解，这是众多云计算coder必备的一项技能。