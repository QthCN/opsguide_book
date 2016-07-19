# 构建集群管理平台OGP

## 简介

本书的目的之一就是从头开始开发一个自动化的运维平台，而对于运维平台来说，集群管理功能是其中的重中之重。因此从本章开始，笔者会开始介绍我们的OGP(Operation Guide Platform)平台。本章及以后的每一章都会一步一步的增加我们OGP平台的功能。而本章则是专注于实现OGP最基本的功能：应用部署。在本章结束后，读者会看到OGP具有以下的功能：

* 从OGP控制台中可以指定应用部署在哪台主机
* OGP会自动保证应用运行在用户指定的主机上
* 从OGP控制台可以进行应用的下线、升级及版本回退
* 外部应用（如jenkins）可以通过客户端注册应用信息

现在先让笔者来介绍下目前流行的一些集群管理平台是如何部署应用的，我们可以以这些平台作为参考，然后开始设计并实现我们的OGP。

## 参考

### Kubernetes

我们首先来看下Kubernetes是怎么实现应用部署的。在Kubernetes中，一个应用一般属于一个Pod。例如其官网中的例子：

```bash
$ kubectl run hello-node --image=gcr.io/PROJECT_ID/hello-node:v1 --port=8080
deployment "hello-node" created
```

这里通过kubectl的run命令，我们部署了一个叫做hello-node的应用，其使用了hello-node:v1这个镜像。在执行了这个命令后Kubernetes中会有如下的组件间的调用：

* kubectl发送HTTP请求给api server，告知其需要部署这个Pod
* api server接到这个请求后，在etcd中创建一个目录，记录下这个信息
* scheduler会watch着api server，而api server则将这个watch请求转为watch etcd。此时当api server在etcd中创建了Pod信息后，scheduler就会感知到这个变化
* scheduler感知变化后，运行调度算法，将这个Pod与某个物理主机绑定起来，并调用api server的接口发送HTTP请求给api server，api server在etcd中记录相关信息
* kubelet也会像scheduler一样watch着api server，进而watch到了Pod绑定信息的变化。当这个Pod绑定的节点上的kubelet watch到这个绑定操作后，其就会负责在对应的主机上启动这个Pod，也就是启动对应的容器
* 除此之外，kubelet会每隔一段世界和api server发送心跳，而controller manager会watch诸如kubelet这类组件的心跳信息，并在特定条件下触发相应操作

这里对于发布应用来说，最重要的就是前五步。这里我们来通过代码看一下这五步在代码中是如何实现的。在看代码之前我们先要讲一下watch这个操作的实现。

#### 通过HTTP协议实现WATCH操作

Etcd、ZooKeeper这类分布式锁服务都提供了一个叫做『Watch』的功能，例如在Etcd上创建一个/target的文件，并给其set一个值100。然后我们就可以让我们的应用监听这个/target路径，如果这个/target包含的内容变更了，那么Etcd就会给我们的监听应用发送一个消息，此时应用就能感知到这种变动并进行相应的操作。

这种Watch操作在Kubernetes非常的重要。比如我们上面所讲的应用部署的步骤，scheduler和kubelet其实就是依赖这种『Watch』来感知到其监听的路径变化，进而触发操作的。比如scheduler监听/pods/tobind这个路径，当kubectl发送请求后api server会在这个路径中写入要被创建的Pod的信息，scheduler就能感知到有新的Pod要处理。同理kubelet也会监听某个路径，例如/{主机名}/pods，当scheduler通过api server在某个主机的/{主机名}/pods中写入某个Pod信息后，对应的主机就能watch到这个变化，然后启动对应的Pod。但在Kubernetes中，为了安全、不依赖Etcd等原因，只有api server可以和Etcd交互，其它组件如scheduler、controller manager、kubelet等都只能和api server通过RESTful的HTTP API交互，我们都知道HTTP只提供了GET、POST、PUT等动作，并没有类似『Watch』这类监听服务端某个信息并能感知变化的动作。那么在Kubernetes中api server是如何提供这种基于HTTP的被『Watch』的能力的呢？答案是通过HTTP Streaming。

对于一个普通的HTTP响应，其返回报文中会包含一个报文长度字段Content-Length表明数据报文有多长。但如果我们要实现『Watch』，我们就需要用流的方式提供数据。也就是当我们向server端发送一个HTTP请求后，HTTP响应报文不应该包含长度字段，而是应该一段一段的返回数据，并且我们的client要能知道这些数据什么时候终止，如果没有到达终止条件那么我们的client就能不停的从这个socket中read数据。这类流式响应报文中会有一个Transfer-Encoding: chunked头部字段，表明返回报文会分为一段一段，并且每一段的开头会包含一个本段的长度信息，例如这个Wikipedia中的例子：

```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25
This is the data in the first chunk

1C
and this is the second one

3
con
8
sequence
0
```

可以看到，这里并没有出现Content-Length，取而代之的是ransfer-Encoding以及一段一段的分段数据。最后的结束标示则是一个0长度字段，表明没有数据了。笔者并没有研究过基于HTTP的断点续传如何实现，但笔者认为通过这种流式的HTTP响应或许可以实现断点续传。

通过这种方式，Kubernetes就可以让Kubelet等组件watch事件了。当Kubelet这类组件向api server发送请求后，api server就返回一个流式响应，每当etcd上出现变动时就响应一段数据给Kubelet等组件。

当然除了上面这种方式外，要实现『Watch』还可以基于websocket等其他方法，但在Kubernetes中使用的就是上面这种方法。

#### 代码实现

现在我们来看下Kubernetes中实现上面几步的代码。这是本书第一次提供其它系统的代码层面实现，由于一些读者可能并不会编写代码，因此笔者在这里需要说明一下笔者在这里给出『代码』的动机。我们都知道在以后『开源』的软件会慢慢的蚕食目前商业软件的份额，但『开源』软件的一个问题是它缺乏文档。为了保证这类软件运行稳定，像过去那样找对应的熟悉相关软件的工程师来运维是非常不现实的，因为第一开源软件没有像商业软件那样的培训，第二开源软件种类繁多，一个人很难系统的熟悉这些软件。因此对于一个公司来说，如果它希望自己采纳开源软件，并且能用的好，用的稳定，必须有对这类软件从源码级别到生产部署级别都很熟悉的人来维护。这也符合我们前面说的『DEVOPS』的理念。因此笔者在这里需要『强迫』大家看一下这些产品的代码，读者目前并不需要会这些代码所使用的语言，只需要能用心看一下这些代码并培养一个『这不就是把大白话写在电脑里』的感觉即可。不过由于篇幅问题，笔者只会列出关键的代码。读者在阅读代码的时候如果没有思路，可以grep下笔者这里给出的『关键代码』，然后顺着函数名逆向的推导调用链一般就能找到方向。现在我们来看下kubernetes的代码。

首先我们看下kubectl的代码，其向api server发送请求的代码如下:

首先在这里注册run命令对应的逻辑代码入口：

```go
func NewCmdRun(f *cmdutil.Factory, cmdIn io.Reader, cmdOut, cmdErr io.Writer) *cobra.Command {
	cmd := &cobra.Command{
		Use: "run NAME --image=image [--env=\"key=value\"] [--port=port] [--replicas=replicas] [--dry-run=bool] [--overrides=inline-json] [--command] -- [COMMAND] [args...]",
		// run-container is deprecated
		Aliases: []string{"run-container"},
		Short:   "Run a particular image on the cluster.",
		Long:    run_long,
		Example: run_example,
		Run: func(cmd *cobra.Command, args []string) {
			argsLenAtDash := cmd.ArgsLenAtDash()
			err := Run(f, cmdIn, cmdOut, cmdErr, cmd, args, argsLenAtDash)
			cmdutil.CheckErr(err)
		},
	}
	cmdutil.AddPrinterFlags(cmd)
	addRunFlags(cmd)
	cmdutil.AddApplyAnnotationFlags(cmd)
	cmdutil.AddRecordFlag(cmd)
	cmdutil.AddInclude3rdPartyFlags(cmd)
	return cmd
}
```

这里Run:结构体成员指定了具体的执行函数。其实现中的关键部分为：

```go
obj, _, mapper, mapping, err := createGeneratedObject(f, cmd, generator, names, params, cmdutil.GetFlagString(cmd, "overrides"), namespace)
```

如果去查看createGeneratedObject，我们可以看到如下的代码：

```go
func (m *Helper) createResource(c RESTClient, resource, namespace string, obj runtime.Object) (runtime.Object, error) {
	return c.Post().NamespaceIfScoped(namespace, m.NamespaceScoped).Resource(resource).Body(obj).Do().Get()
}
```

可以从这里的参数知道，这里的c是一个RESTClient的client，更确切的说是一个连接到api server的Client。从方法调用我们知道，这里主要是向api server发送了一个POST请求，也就是含有『创建』这个语义。而后面的resource和body大家感兴趣的话可以去看下createGeneratedObject的代码。总之目前这里的代码告诉了我们，当我们运行了run命令后，kubectl会的调用注册的func函数，而这个函数则会调用RESTClient发送一个Post请求给我们的api server。对于感兴趣知道这个Post请求包含了什么的读者，可以在这里加上一些输出信息。

接着我们看下api server端的代码。按照笔者上面给出的步骤我们知道api server端至少要做两件事，一件是提供对应的RESTful API，还有一件是在etcd中set对应的信息。我们先来看RESTful API的注册，进行URL到具体处理函数的关键代码是：

```go
// Install v1 API.
m.initV1ResourcesStorage(c)
```

在api server中，storage一般指的就是后端的存储，也就是etcd。因此其实从这个角度来说，api server就是一个提供RESTful到etcd存取操作的mapping服务。initV1ResourcesStorage中关键的代码为：

```go
podStorage := podetcd.NewStorage(
		restOptions("pods"),
		kubeletclient.ConnectionInfoGetter(nodeStorage.Node),
		m.ProxyTransport,
	)
...
m.v1ResourcesStorage = map[string]rest.Storage{
		"pods":             podStorage.Pod,
		"pods/attach":      podStorage.Attach,
		"pods/status":      podStorage.Status,
		"pods/log":         podStorage.Log,
		"pods/exec":        podStorage.Exec,
		"pods/portforward": podStorage.PortForward,
		"pods/proxy":       podStorage.Proxy,
		"pods/binding":     podStorage.Binding,
		"bindings":         podStorage.Binding,

		"podTemplates": podTemplateStorage,

		"replicationControllers":        controllerStorage.Controller,
		"replicationControllers/status": controllerStorage.Status,

		"services":        serviceRest.Service,
		"services/proxy":  serviceRest.Proxy,
		"services/status": serviceStatusStorage,

		"endpoints": endpointsStorage,

		"nodes":        nodeStorage.Node,
		"nodes/status": nodeStorage.Status,
		"nodes/proxy":  nodeStorage.Proxy,

		"events": eventStorage,

		"limitRanges":                   limitRangeStorage,
		"resourceQuotas":                resourceQuotaStorage,
		"resourceQuotas/status":         resourceQuotaStatusStorage,
		"namespaces":                    namespaceStorage,
		"namespaces/status":             namespaceStatusStorage,
		"namespaces/finalize":           namespaceFinalizeStorage,
		"secrets":                       secretStorage,
		"serviceAccounts":               serviceAccountStorage,
		"persistentVolumes":             persistentVolumeStorage,
		"persistentVolumes/status":      persistentVolumeStatusStorage,
		"persistentVolumeClaims":        persistentVolumeClaimStorage,
		"persistentVolumeClaims/status": persistentVolumeClaimStatusStorage,
		"configMaps":                    configMapStorage,

		"componentStatuses": componentstatus.NewStorage(func() map[string]apiserver.Server { return m.getServersToValidate(c) }),
	}
```

这里和Pods相关的handle是由这里的podetcd.NewStorage给出的。NewStorage的代码比较短，这里笔者都列出来：
```go
// NewStorage returns a RESTStorage object that will work against pods.
func NewStorage(opts generic.RESTOptions, k client.ConnectionInfoGetter, proxyTransport http.RoundTripper) PodStorage {
	prefix := "/pods"

	newListFunc := func() runtime.Object { return &api.PodList{} }
	storageInterface := opts.Decorator(
		opts.Storage, cachesize.GetWatchCacheSizeByResource(cachesize.Pods), &api.Pod{}, prefix, pod.Strategy, newListFunc)

	store := &registry.Store{
		NewFunc:     func() runtime.Object { return &api.Pod{} },
		NewListFunc: newListFunc,
		KeyRootFunc: func(ctx api.Context) string {
			return registry.NamespaceKeyRootFunc(ctx, prefix)
		},
		KeyFunc: func(ctx api.Context, name string) (string, error) {
			return registry.NamespaceKeyFunc(ctx, prefix, name)
		},
		ObjectNameFunc: func(obj runtime.Object) (string, error) {
			return obj.(*api.Pod).Name, nil
		},
		PredicateFunc:           pod.MatchPod,
		QualifiedResource:       api.Resource("pods"),
		DeleteCollectionWorkers: opts.DeleteCollectionWorkers,

		CreateStrategy:      pod.Strategy,
		UpdateStrategy:      pod.Strategy,
		DeleteStrategy:      pod.Strategy,
		ReturnDeletedObject: true,

		Storage: storageInterface,
	}

	statusStore := *store
	statusStore.UpdateStrategy = pod.StatusStrategy

	return PodStorage{
		Pod:         &REST{store, proxyTransport},
		Binding:     &BindingREST{store: store},
		Status:      &StatusREST{store: &statusStore},
		Log:         &podrest.LogREST{Store: store, KubeletConn: k},
		Proxy:       &podrest.ProxyREST{Store: store, ProxyTransport: proxyTransport},
		Exec:        &podrest.ExecREST{Store: store, KubeletConn: k},
		Attach:      &podrest.AttachREST{Store: store, KubeletConn: k},
		PortForward: &podrest.PortForwardREST{Store: store, KubeletConn: k},
	}
}
```

我们上面说了api server主要就是映射URL到etcd的存取，而在api server中存取操作就是由这里的registry.Store中的Storage这个storageInterface类型熟悉进行的。后者本质上提供了Interface接口，接口的说明如下：

```go
// Interface offers a common interface for object marshaling/unmarshling operations and
// hides all the storage-related operations behind it.
type Interface interface {
	// Returns list of servers addresses of the underyling database.
	// TODO: This method is used only in a single place. Consider refactoring and getting rid
	// of this method from the interface.
	Backends(ctx context.Context) []string

	// Returns Versioner associated with this interface.
	Versioner() Versioner

	// Create adds a new object at a key unless it already exists. 'ttl' is time-to-live
	// in seconds (0 means forever). If no error is returned and out is not nil, out will be
	// set to the read value from database.
	Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error

	// Delete removes the specified key and returns the value that existed at that spot.
	// If key didn't exist, it will return NotFound storage error.
	Delete(ctx context.Context, key string, out runtime.Object, preconditions *Preconditions) error

	// Watch begins watching the specified key. Events are decoded into API objects,
	// and any items passing 'filter' are sent down to returned watch.Interface.
	// resourceVersion may be used to specify what version to begin watching,
	// which should be the current resourceVersion, and no longer rv+1
	// (e.g. reconnecting without missing any updates).
	Watch(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error)

	// WatchList begins watching the specified key's items. Items are decoded into API
	// objects and any item passing 'filter' are sent down to returned watch.Interface.
	// resourceVersion may be used to specify what version to begin watching,
	// which should be the current resourceVersion, and no longer rv+1
	// (e.g. reconnecting without missing any updates).
	WatchList(ctx context.Context, key string, resourceVersion string, filter FilterFunc) (watch.Interface, error)

	// Get unmarshals json found at key into objPtr. On a not found error, will either
	// return a zero object of the requested type, or an error, depending on ignoreNotFound.
	// Treats empty responses and nil response nodes exactly like a not found error.
	Get(ctx context.Context, key string, objPtr runtime.Object, ignoreNotFound bool) error

	// GetToList unmarshals json found at key and opaque it into *List api object
	// (an object that satisfies the runtime.IsList definition).
	GetToList(ctx context.Context, key string, filter FilterFunc, listObj runtime.Object) error

	// List unmarshalls jsons found at directory defined by key and opaque them
	// into *List api object (an object that satisfies runtime.IsList definition).
	// The returned contents may be delayed, but it is guaranteed that they will
	// be have at least 'resourceVersion'.
	List(ctx context.Context, key string, resourceVersion string, filter FilterFunc, listObj runtime.Object) error

	// GuaranteedUpdate keeps calling 'tryUpdate()' to update key 'key' (of type 'ptrToType')
	// retrying the update until success if there is index conflict.
	// Note that object passed to tryUpdate may change across invocations of tryUpdate() if
	// other writers are simultaneously updating it, so tryUpdate() needs to take into account
	// the current contents of the object when deciding how the update object should look.
	// If the key doesn't exist, it will return NotFound storage error if ignoreNotFound=false
	// or zero value in 'ptrToType' parameter otherwise.
	// If the object to update has the same value as previous, it won't do any update
	// but will return the object in 'ptrToType' parameter.
	//
	// Example:
	//
	// s := /* implementation of Interface */
	// err := s.GuaranteedUpdate(
	//     "myKey", &MyType{}, true,
	//     func(input runtime.Object, res ResponseMeta) (runtime.Object, *uint64, error) {
	//       // Before each incovation of the user defined function, "input" is reset to
	//       // current contents for "myKey" in database.
	//       curr := input.(*MyType)  // Guaranteed to succeed.
	//
	//       // Make the modification
	//       curr.Counter++
	//
	//       // Return the modified object - return an error to stop iterating. Return
	//       // a uint64 to alter the TTL on the object, or nil to keep it the same value.
	//       return cur, nil, nil
	//    }
	// })
	GuaranteedUpdate(ctx context.Context, key string, ptrToType runtime.Object, ignoreNotFound bool, precondtions *Preconditions, tryUpdate UpdateFunc) error

	// Codec provides access to the underlying codec being used by the implementation.
	Codec() runtime.Codec
}
```

从注释可以看出，通过这个Interface， api server尽可能的屏蔽后端存储的操作细节，这也是为了做到不依赖etcd。这里后端的通用实现的关键代码如下：

```go
// Create creates a storage backend based on given config.
func Create(c storagebackend.Config, codec runtime.Codec) (storage.Interface, error) {
	switch c.Type {
	case storagebackend.StorageTypeUnset, storagebackend.StorageTypeETCD2:
		return newETCD2Storage(c, codec)
	case storagebackend.StorageTypeETCD3:
		// TODO: We have the following features to implement:
		// - Support secure connection by using key, cert, and CA files.
		// - Honor "https" scheme to support secure connection in gRPC.
		// - Support non-quorum read.
		return newETCD3Storage(c, codec)
	default:
		return nil, fmt.Errorf("unknown storage type: %s", c.Type)
	}
}
```
这里会选择是采用etcdv2还是evcdv3，如果采用的是etcdv2，那么对应的Set操作的代码为：

```go
// Implements storage.Interface.
func (h *etcdHelper) Create(ctx context.Context, key string, obj, out runtime.Object, ttl uint64) error {
	trace := util.NewTrace("etcdHelper::Create " + getTypeName(obj))
	defer trace.LogIfLong(250 * time.Millisecond)
	if ctx == nil {
		glog.Errorf("Context is nil")
	}
	key = h.prefixEtcdKey(key)
	data, err := runtime.Encode(h.codec, obj)
	trace.Step("Object encoded")
	if err != nil {
		return err
	}
	if version, err := h.versioner.ObjectResourceVersion(obj); err == nil && version != 0 {
		return errors.New("resourceVersion may not be set on objects to be created")
	}
	trace.Step("Version checked")

	startTime := time.Now()
	opts := etcd.SetOptions{
		TTL:       time.Duration(ttl) * time.Second,
		PrevExist: etcd.PrevNoExist,
	}
	response, err := h.etcdKeysAPI.Set(ctx, key, string(data), &opts)
	trace.Step("Object created")
	metrics.RecordEtcdRequestLatency("create", getTypeName(obj), startTime)
	if err != nil {
		return toStorageErr(err, key, 0)
	}
	if out != nil {
		if _, err := conversion.EnforcePtr(out); err != nil {
			panic("unable to convert output object to pointer")
		}
		_, _, err = h.extractObj(response, err, out, false, false)
	}
	return err
}
```

这里通过h.etcdKeysAPI.set向etcd发送请求，对应的实现为：
```go
func (k *httpKeysAPI) Set(ctx context.Context, key, val string, opts *SetOptions) (*Response, error) {
	act := &setAction{
		Prefix: k.prefix,
		Key:    key,
		Value:  val,
	}

	if opts != nil {
		act.PrevValue = opts.PrevValue
		act.PrevIndex = opts.PrevIndex
		act.PrevExist = opts.PrevExist
		act.TTL = opts.TTL
		act.Refresh = opts.Refresh
		act.Dir = opts.Dir
	}

	resp, body, err := k.client.Do(ctx, act)
	if err != nil {
		return nil, err
	}

	return unmarshalHTTPResponse(resp.StatusCode, resp.Header, body)
}
```
从代码可以看出，这里主要就是发送了一个HTTP请求给etcd。那么通用的Create函数是如何获取不同请求的请求路径和请求报文体的呢？这个还是要回到我们上面说的registry.Store，其成员属性给出了不同资源的不同请求信息。例如这里的NewFunc就给出了Pod资源的请求信息：

```go
NewFunc:     func() runtime.Object { return &api.Pod{} },
...
// Pod is a collection of containers, used as either input (create, update) or as output (list, get).
type Pod struct {
	unversioned.TypeMeta `json:",inline"`
	ObjectMeta           `json:"metadata,omitempty"`

	// Spec defines the behavior of a pod.
	Spec PodSpec `json:"spec,omitempty"`

	// Status represents the current information about a pod. This data may not be up
	// to date.
	Status PodStatus `json:"status,omitempty"`
}
```

我们可以参照着API文档看到这些字段对应的含义。

最后我们看下Kubelet和scheduler是如何『Watch』api server的。关键的代码如下：

```go
func makePodSourceConfig(kc *KubeletConfig) *config.PodConfig {
	// source of all configuration
	cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kc.Recorder)

	// define file config source
	if kc.ConfigFile != "" {
		glog.Infof("Adding manifest file: %v", kc.ConfigFile)
		config.NewSourceFile(kc.ConfigFile, kc.NodeName, kc.FileCheckFrequency, cfg.Channel(kubetypes.FileSource))
	}

	// define url config source
	if kc.ManifestURL != "" {
		glog.Infof("Adding manifest url %q with HTTP header %v", kc.ManifestURL, kc.ManifestURLHeader)
		config.NewSourceURL(kc.ManifestURL, kc.ManifestURLHeader, kc.NodeName, kc.HTTPCheckFrequency, cfg.Channel(kubetypes.HTTPSource))
	}
	if kc.KubeClient != nil {
		glog.Infof("Watching apiserver")
		config.NewSourceApiserver(kc.KubeClient, kc.NodeName, cfg.Channel(kubetypes.ApiserverSource))
	}
	return cfg
}
```

从文档和这类的代码可以看到，kubelet主要会监视三个地方的变化，一个是配置文件中指定目录下的配置信息的变化，一个是外部的HTTP接口获取信息的变化，还有一个就是kc.KubeClient锁代表的api server中的变化。通过这三种方法我们都可以让kubelet运行Pod。这里主要来看下NewSourceApiserver。

```go
// Run starts a watch and handles watch events. Will restart the watch if it is closed.
// Run starts a goroutine and returns immediately.
func (r *Reflector) Run() {
	glog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedType, r.resyncPeriod, r.name)
	go wait.Until(func() {
		if err := r.ListAndWatch(wait.NeverStop); err != nil {
			utilruntime.HandleError(err)
		}
	}, r.period, wait.NeverStop)
}
```

从函数名可以看出这里的代码会进行『Watch』操作。而『Watch』的实现则是：

```go
watchFunc := func(options api.ListOptions) (watch.Interface, error) {
    return c.Get().
        Prefix("watch").
        Namespace(namespace).
        Resource(resource).
        VersionedParams(&options, api.ParameterCodec).
        FieldsSelectorParam(fieldSelector).
        Watch()
}
```

从函数名我们可以大致猜出这里就是发送了一个HTTP请求给/watch路径进行Watch操作。scheduler的实现也是类似，感兴趣的读者可以自行去看下相应的实现。

## 实现

### 整体框架

现在读者已经了解了应用部署在目前流行的Kubernetes中是如何实现的，那就开始来实现我们的OGP吧。目前笔者设计的OGP的架构是这样支持应用部署的：

* 组件上分为ogp-controller、ogp-docker-agent、ogp-cli和portal
  * portal：集群控制台。用户可以通过portal对集群进行管理操作，例如应用部署就可以通过portal完成。portal的实现基于Flask框架和eventlet这个协程库，通过Flask这个框架对外提供WEB页面，同时通过eventlet增强并发能力。这两个库后面会详细介绍。
  * ogp-cli：命令行管理工具。用户可以通过ogp-cli对ogp-controller发送命令。
  * ogp-docker-agent：docker状态维护程序。每台物理主机上都会运行一个ogp-docker-agent(后面会简称DA)。DA会的定期的和ogp-controller同步其所运行的物理主机上的容器状态，同时其也会接收ogp-controller发来的请求。
  * ogp-controller：集群控制器。集群的核心组件，所有的请求都会发送给ogp-controller，其会在内存中维护整个集群的状态，并和DA保持容器信息的状态维护。
* 用户通过git推送代码到gitlab，jenkins检测到代码被推送到gitlab后会触发预定义的操作。这里的操作主要有：
  * 运行代码中的Dockerfile进行build image的操作
  * 将build好的image推送到私有的docker仓库中
  * 调用ogp-cli，发送信息给ogp-controll注册这个image信息。在OGP中我们规定应用的名称就是镜像的名称，而应用的版本则是镜像的版本，这个版本目前我们使用jenkins的执行编号
  * 待ogp-cli注册镜像到ogp-controller中后，我们在portal中就能看到这个镜像了。当然在portal这个镜像的名字叫做应用，并且会带有一个版本号。我们可以在portal中点击发布，将这个特定版本的应用绑定到某台主机上，这里假设绑定到主机A
  * ogp-controller收到这个请求后，会在内存中创建这种绑定关系，接着ogp-controller会发送消息给主机A上的DA，通知其立刻进行一次容器状态同步。当DA开始同步后，其会发现其所在的主机A上应该会运行一个应用，于是DA会检测主机A上当前运行了哪些容器，将这个信息和ogp-controller发来的信息做比较，待发现信息不一致后DA会执行这个应用的启动操作。启动的方法就是调用docker的HTTP接口先从仓库中拉取镜像，然后启动容器
  * DA创建应用成功后会主动通知ogp-controller再次进行同步，或者每隔一段时间DA也会和ogp-controller进行同步，此时ogp-controller就能知道应用已经运行在主机A上了。用户此时从portal就能看到应用状态的更新

ogp-cli、ogp-docker-agent、ogp-cli和portal就是我们本章要实现的四个组件。他们之间的调用关系目前就是上面讲的这样。可以看到目前他们的功能还比较薄弱，但没关系，我们后面基本上每一章都会给OGP新增一些功能，通过这种迭代式的开发我们最终会得到一个功能强大的集群管理平台。现在我们来讲下这四个组件代码层面的架构和实现，首先我们来讲下OGP中的组件之间是如何调用的。

### Boost.asio与异步通信

如果读者开发过网络应用，那么肯定听说过socket。linux下的socket编程可以在《UNIX网络编程》中找到详细的教程。如果没有听说过那也没关系，笔者这里会列一个简单的例子。

假设我们现在有两个进程C和S，C是一个类似于客户端进程的东西，它会发送一个只包含字符串的请求给S，然后S会在这个字符串后面添加一个字符串并将这个新的字符串返回给C。我们来看下如何使用socket实现这个例子：

```python
(python3.5)➜  cs cat c.py
import socket

if __name__ == "__main__":
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('localhost', 9090))
    sock.send('hello'.encode('utf-8'))
    print(sock.recv(1024))
    sock.close()
(python3.5)➜  cs cat s.py
import socket

if __name__ == '__main__':
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.bind(('localhost', 9090))
    sock.listen(5)
    while True:
        connection, address = sock.accept()
        try:
            connection.settimeout(5)
            buf = connection.recv(1024)
            connection.send(buf + " world".encode('utf-8'))
        except socket.timeout:
            raise
        connection.close()
(python3.5)➜  cs
```

接着我们先运行s.py，然后运行c.py，我们可以看到如下输出：
```python
(python3.5)➜  cs python c.py
b'hello world'
```

就这样我们通过socket的方式让这两个进程进行了通信，当然我们可以在不同的主机上运行着两个进程，只需要改变下这里监听的地址即可。

虽然这里通过socket实现了我们的网络通信，但在我们的OGP集群环境中通过这种socket通信方式会存在问题。原因在于我们的集群中会有成百上千的主机，这些主机上的DA都会和ogp-controller同步信息和发送心跳请求，这样ogp-controller就需要处理大量的请求。假设每个请求都会消耗一些时间，那么按照我们上面的代码，所有的请求在server端都需要排队，因为server是单进程的，只能一个请求一个请求的处理，因此集群的性能会出现很大问题，甚至可能会出现心跳包来不及处理让集群误以为某个DA进程宕了的情况。因此最容易想到的方法就是让我们的控制器运行在多线程模式下，增强其并发能力。但对于集群环境来说多线程也不是一个很好的选择，因为加入我们的集群中有2000台主机，那么至少我们的ogp-controller需要建立2000个线程，一个控制器运行这么多线程并保持长连接是非常消耗资源的，并且效率比较低下。那么是否可以通过短连接的方式呢？例如每次DA同步数据的时候都进行对ogp-controller的连接并进行消息交互，然后断开连接。这样也不可行，因为短连接的建立消耗的资源也是一个很重的负担。因此为了解决这个问题我们需要换个思路。假设我们DA和控制器之间每3秒进行一次心跳同步，且一次同步花费0.001ms（主要是网络开销，但其实在IO复用中这类网络开销是可以忽略的，这里主要是为了举例），那么如果运行2000个线程，则这2000个线程中的每个线程其实在(3000-0.001)/3000这近乎百分之百的时间里都是在等待的，这白白的等待浪费了线程间切换的时间片。因此我们可以让一个线程服务这2000个DA，此时实际的工作时间可以近似为2000*0.001/3约百分之七十。但问题是按照我们上面的方法无法通过一个线程建立和多个进程的长连接，解决的方法就是通过非阻塞IO。

非阻塞IO的思路很简单。这里以server端为例。当server端通过listen创建了套接字后，它会把这个套接字放到一个poll中，然后通过内核提供的机制监听这个poll。内核会在这个poll中的任何一个socket上有数据的时候激活这个监听的server线程，此时由于我们只存放了监听套接字，那么现在一定是有client连接上来了，于是server去处理这个请求，得到两个套接字：本机的监听套接字及连接上来的client的对端套接字。接着server把这两个套接字都放到poll中，然后进行睡眠监听，等待内核发现有消息的时候唤醒它，就通过这个方式在OGP中我们的poll会最终存在2000多个套接字，当DA没有消息发过来的时候我们的控制器会处于睡眠中或者是在执行其它工作，当有DA发来心跳信息的时候控制器进程就会被唤醒然后执行对应的心跳响应代码。就这样我们一个线程就能处理这么多的请求。

可以看到这种非阻塞IO的前提是需要内核的支持，目前linux、windows等主流OS都支持这种机制，例如提供了select、poll、epoll等接口供程序调用。在一些资料里这种调用方式也被称为IO复用。

当我们使用了非阻塞IO的时候，我们的代码逻辑就不一定是顺序执行了。因为当我们使用了非阻塞IO后，我们的read、send等方法会立刻返回，而不是像阻塞IO一样读取或发送了一定数据后才返回。这样我们需要再次调用socket接口去查询我们的请求状态，但这样做就失去了一部分非阻塞IO的作用。为了解决这个问题，就需要使用异步IO。下面的伪代码说明了异步IO和一般的同步IO的区别（注意下面的代码是伪代码，并不能实际执行）：

```python
(python3.5)➜  cs cat c.py
import socket

if __name__ == "__main__":
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('localhost', 9090))
    sock.send('hello'.encode('utf-8'))
    print(sock.recv(1024))
    sock.close
(python3.5)➜  cs cat ac.py
import socket

def handle_receive(data):
    print(data)

def handle_send(data):
    sock.async_recv(1024, handle_receive)

if __name__ == "__main__":
    loop = io_loop()
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(('localhost', 9090))
    sock.async_send('hello'.encode('utf-8'), handle_send)
    loop.run()
    sock.close()
```
在c.py这个同步IO中，sock.send这里会阻塞，接着在sock.recv处也会阻塞。而在ac.py这个异步IO的例子里，我们的asnyc_send会立刻返回，然后当数据发送完成后内核会唤醒进程，然后进程会执行我们绑定的回调函数handle_send，后者调用async_recv接收数据，同样的这个函数会立刻返回，进程会再次睡眠，知道内核通知数据接收完毕，此时会开始调用handle_receive这个回调函数输出结果。可以看到，在异步IO中，实际的代码逻辑会依赖于我们注册的回调函数去实现，另外由于回调函数的回调顺序和我们调用异步函数的顺序可能不一致，因此大家在编写这方面代码的时候需要注意这一点。

在我们的OGP平台中，ogp-controller和ogp-docker-agent这两个组件之间会存在大量的通信，因此这两个组件我们会使用异步IO的方式进行消息交互。而ogp-cli和portal与ogp-controller之间的通信比较简单，因此对于ogp-cli和portal来说他们只需要发送阻塞IO给ogp-controller即可，因为这样代码会清晰许多。当然ogp-controller端则是非阻塞IO进行处理。

由于网络编程需要处理很多底层细节问题，因此在OGP的异步IO中，我们使用Boost下的asio这个网络通信库作为我们异步IO的通信框架。Boost是C++中一个非常有名的软件库集合，有兴趣的读者可以去了解一下，Boost中的很多库已经成为了C++新标准中的标准库。除了asio外类似libevent这类库也可以作为异步IO框架使用。

我们来看下在asio中如何进行异步IO，下面这个例子来自于boost官方文档中的example：

```cpp
//
// async_tcp_echo_server.cpp
// ~~~~~~~~~~~~~~~~~~~~~~~~~
//
// Copyright (c) 2003-2013 Christopher M. Kohlhoff (chris at kohlhoff dot com)
//
// Distributed under the Boost Software License, Version 1.0. (See accompanying
// file LICENSE_1_0.txt or copy at http://www.boost.org/LICENSE_1_0.txt)
//

#include <cstdlib>
#include <iostream>
#include <boost/bind.hpp>
#include <boost/asio.hpp>

using boost::asio::ip::tcp;

class session
{
public:
  session(boost::asio::io_service& io_service)
    : socket_(io_service)
  {
  }

  tcp::socket& socket()
  {
    return socket_;
  }

  void start()
  {
    socket_.async_read_some(boost::asio::buffer(data_, max_length),
        boost::bind(&session::handle_read, this,
          boost::asio::placeholders::error,
          boost::asio::placeholders::bytes_transferred));
  }

private:
  void handle_read(const boost::system::error_code& error,
      size_t bytes_transferred)
  {
    if (!error)
    {
      boost::asio::async_write(socket_,
          boost::asio::buffer(data_, bytes_transferred),
          boost::bind(&session::handle_write, this,
            boost::asio::placeholders::error));
    }
    else
    {
      delete this;
    }
  }

  void handle_write(const boost::system::error_code& error)
  {
    if (!error)
    {
      socket_.async_read_some(boost::asio::buffer(data_, max_length),
          boost::bind(&session::handle_read, this,
            boost::asio::placeholders::error,
            boost::asio::placeholders::bytes_transferred));
    }
    else
    {
      delete this;
    }
  }

  tcp::socket socket_;
  enum { max_length = 1024 };
  char data_[max_length];
};

class server
{
public:
  server(boost::asio::io_service& io_service, short port)
    : io_service_(io_service),
      acceptor_(io_service, tcp::endpoint(tcp::v4(), port))
  {
    start_accept();
  }

private:
  void start_accept()
  {
    session* new_session = new session(io_service_);
    acceptor_.async_accept(new_session->socket(),
        boost::bind(&server::handle_accept, this, new_session,
          boost::asio::placeholders::error));
  }

  void handle_accept(session* new_session,
      const boost::system::error_code& error)
  {
    if (!error)
    {
      new_session->start();
    }
    else
    {
      delete new_session;
    }

    start_accept();
  }

  boost::asio::io_service& io_service_;
  tcp::acceptor acceptor_;
};

int main(int argc, char* argv[])
{
  try
  {
    if (argc != 2)
    {
      std::cerr << "Usage: async_tcp_echo_server <port>\n";
      return 1;
    }

    boost::asio::io_service io_service;

    using namespace std; // For atoi.
    server s(io_service, atoi(argv[1]));

    io_service.run();
  }
  catch (std::exception& e)
  {
    std::cerr << "Exception: " << e.what() << "\n";
  }

  return 0;
}
```

按照笔者之前举例的异步IO的例子，这里asio的代码就可以按照异步函数+回调函数的方法来解读：

* 首先是异步接受方法async_accept，其会有一个回调函数server::handle_accept
* 当有请求连接过来后handle_accept就会被调用，其会执行异步读取操作async_read_some，并且其有一个回调函数在读取操作完成是调用，回调函数为session::handle_read。另外这里会继续将server端的socket放到poll中等待其它连接
* handle_read被调用后，代码处理对方传来的数据，然后向对方返回数据，此时调用异步写入函数async_write，其注册了写入操作完成时的回调函数session::handle_write
* handle_write会继续将socket放入poll中继续读取对方传来的数据，并重复这个过程

当然asio能提供的功能比上面列出的多的多，除了这里列出的功能外，在我们的OGP中主要还使用了下面这些功能：

* streambuf，用于处理读取和写入的buffer
* strand，用于保证某个session中的顺序操作
* 多线程，我们上面的asio例子只是单个线程监听io_server底层对应的poll，但实际上内核支持多线程同时监听

### JSON与protobuf

熟悉javascript或者python的人都一定会熟悉JSON，原因是它的格式是在是太简单了，并且这两种语言自带的字典类型都非常好的支持这种格式。读者如果没有听说过JSON的话，XML一定有听说过。例如下面就是同一段信息通过JSON和XML表示的样子：

```
// XML
<id>
	<firstname>bingo</firstname>
	<lastname>tree</lastname>
</id>

// JSON
"id": {
	"firstname": "bingo",
	"lastname": "tree"
}
```

在实际的使用中这两种格式其实都用的挺多，除此之外比较常见的格式还有YAML。不过这里读者主要以JSON举例。

我们可以看到JSON的语法格式非常直观，对于完全不熟悉的人来说只要花上10分钟就能看懂甚至写出满足要求的JSON字符串。但在我们的OGP平台中，组件与组件之间的异步通信走的消息格式并没有使用JSON，而是使用的protobuf。那么protobuf是什么？为什么我们要使用这种格式而不使用JSON呢？

protobuf是google内部使用并开源的一种消息格式化标准，读者可以认为它做的事情和JSON、XML是一样的，就是把一段信息变成某个符合特定规范的格式，然后在网络中传递这个格式到对方，对方按照规范可以解析并还原信息。正如我们上面的例子，我们的代码中有一个id类，它有两个属性firstname和lastname，在网络中传输的时候我们一般是把它dump成JSON的文本格式进行传输，而如果使用protobuf的话我们就可以把这个类dump成protobuf的二进制格式进行传输。功能上其实和JSON做的事情差不多。那为什么在OGP中我们选用protobuf而不是使用JSON呢？原因主要在于性能。protobuf可以提供更小的数据包，并提供更加快速的序列化、反序列化速度。对于我们的OGP来说，由于控制器和DA等组件之间要传递数目巨大的数据包，因此protobuf所能提供的这个优点尤其重要。读者如果感兴趣可以使用python官方的JSON模块去序列化、反序列化一个10M的文件，然后感受一下其速度以及内存开销，相信读者一定会被内存开销吓到的。

protobuf的使用方法大家可以参考下其官方文档，这里列一下其文档中的一个例子。我们这里以GO语言序列化数据，并用C++反序列化数据。按照我们上面说的，双方必须对消息格式有一个统一的规范，在protobuf中这种规范通过消息文件来制定：

```
message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
}

// Our address book file is just one of these.
message AddressBook {
  repeated Person people = 1;
}
```

这就是一个消息文件，message关键字用于定义消息，这里最外层的消息就是AddressBook，其嵌套了Person的数组，而Person消息则可以包含PhoneNumer消息以及name、id、email等属性。

将这个消息文件分别编译为go和C++代码的可调用文件的方法为：
```
protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto
protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto
```
此时就能分别获得addressbook.pb.go和addressbook.pb.h、addressbook.pb.cc文件，通过这些文件就能读取和写入我们的AddressBook消息了。下面是相关的代码：

```go
book := &pb.AddressBook{}
// ...

// Write the new address book back to disk.
out, err := proto.Marshal(book)
if err != nil {
        log.Fatalln("Failed to encode address book:", err)
}
if err := ioutil.WriteFile(fname, out, 0644); err != nil {
        log.Fatalln("Failed to write address book:", err)
}
```

```cpp
tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }
```

从这里也能看到，通过protobuf我们可以做到跨语言的交互。

使用protobuf对于C++来说还有另外一个好处，那就是我们可以用protobuf生成的addressbook.pb.h作为我们的model对象，这可以省去我们不少时间和代码。熟悉ORM的读者一定会对这种方式感到非常熟悉。

### Flask与WSGI

由于我们的OGP会的很时髦的提供一个portal，因此我们需要一个写这个portal的框架。对于portal来说用C++编写绝对不是一个好主意，因此这里我们使用Python。目前国内的很多公司都使用Python进行运维自动化系统的开发，并且Python确实非常适合这类系统。原因主要是Python本身语法较为简单，有非常丰富的类库，并且这类运维自动化系统对性能要求并不是很高。当然目前还有一个趋势就是Go语言的流行，例如本书主要使用的docker就是Go编写的。不过和Python相比，对于portal来说Go并没有什么优势。

Python中用于提供WEB的框架有很多，例如Django、Tornado、Flask、Web.py等等。光笔者使用过的就有Django、Tornado和Flask，但就笔者个人的感觉而言，这些框架其实大同小异。基本都是URL映射加上template的形式。在OGP中笔者选用Flask作为我们的WEB框架。下面来简单介绍一下Flask以及其使用方法。

Flask是一个基于Werkzeug和Jinja 2上的一个框架，一个最简单的例子如下：

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

我们可以把上面的代码保存为hello.py文件，然后执行python hello.py并用浏览器访问5000端口就可以看到页面上输出Hello world了。相比较于Java的Servlet来说如果抛开语言层面的不同，其实Flask提供的WEB相关的功能和Servlet是几乎一样的。在上面这个例子里我们通过@app.route("/")将根地址映射到了hello函数，当用户访问根地址后，请求会从浏览器流向到Flask，Flask收到请求后就会分析请求的目的地址，然后发现目的地址是"/"，于是查看映射关系并找到对应的处理函数hello，然后调用hello并获取到返回的内容，再将这个内容传递给浏览器。

如果这里的返回内容不是一个字符串而是一个html的模板，那么Flask会自动的解析这个模板，填充相关内容，然后将这个html页面返回给浏览器让浏览器展现相关页面。例如：

```python
from flask import render_template

@app.route('/hello/')
@app.route('/hello/<name>')
def hello(name=None):
    return render_template('hello.html', name=name)
```

这里hello.html的内容为：

```
<!doctype html>
<title>Hello from Flask</title>
{% if name %}
  <h1>Hello {{ name }}!</h1>
{% else %}
  <h1>Hello, World!</h1>
{% endif %}
```

此时用户如果访问的地址是/hello，那么就会显示Hello,world!。如果用户访问的是/hello/bingotree，那么就会显示Hello bingotree页面。可以看到这里的html模板也具有一些逻辑操作的能力，这点和JSP这类模板是一样的。

虽然通过上面的这种形式我们可以写出一个WEB服务，但当代码量达到一定程度的时候这种方式就不合适了，因为如果这些代码都写在一个大文件里那么维护起来的代价是很大的。因此我们可以通过blueprint的方式按照业务类型和逻辑进行拆分。比如官网的这个例子，我们将一些简单的页面单独放到一个文件中做映射：
```python
from flask import Blueprint, render_template, abort
from jinja2 import TemplateNotFound

simple_page = Blueprint('simple_page', __name__,
                        template_folder='templates')

@simple_page.route('/', defaults={'page': 'index'})
@simple_page.route('/<page>')
def show(page):
    try:
        return render_template('pages/%s.html' % page)
    except TemplateNotFound:
        abort(404)
```
然后在主文件中注册这个simple_page页面：
```python
from flask import Flask
from bingotree.simple_page import simple_page

app = Flask(__name__)
app.register_blueprint(simple_page)
```
如果我们还有一个登陆页面，那么我们可以用类似simple_page的方法建立一个login_page文件，将相关的操作放到这个文件中，并在主文件中进行注册。

现在我们再来看下WSGI。Flask目前是个百分百兼容WSGI协议的一个WEB框架，而在下一节读者会看到我们使用了eventlet来增强我们WEB服务的并发能力，eventlet也提供了一个WSGI的WEB服务，并且我们会把Flask的app做为一个WSGI对象传递给eventlet，让其包裹住Flask的WSGI app并提供服务。那么这个WSGI到底是什么呢？

WSGI是在PEP 333中提出的一个规范，简单的理解的话这个规范做的事情就是定义了一个函数其返回值及参数信息，这个和基于接口契约编程是一个道理。如果Flask提供的WEB服务能通过一个函数开放出来，例如函数flask_run，那么实现了WSGI的服务器就可以直接调用这个函数，因为这个服务器知道应该传递什么参数给你这个函数，并且也知道你这个函数会的返回什么东西给服务器。此时当HTTP请求到达WSGI服务器后，服务器按照WSGI规范将这个请求的请求信息解析为具体的参数，然后将这些参数传递给flask_run。由于flask_run知道传递来的参数是满足WSGI规范的，自然也就知道这些参数的含义，因此就能进行处理。从这一点上来说Flask既可以做WSGI服务器，也能提供满足WSGI要求的函数出来给其它的WSGI服务器使用。

那么WSGI的好处是什么呢？一个好处就是上面讲的我用Flask写的逻辑代码可以放在所有WSGI服务器上运行，而一个WEB的WSGI服务器其实在这里就相当于中间件，它本身要处理很多非业务相关的事情，例如内存管理等等。这样对于我来说我的选择就有很多，可以根据实际需要选择对应的WSGI服务器。另一个好处是WSGI能提供middleware的支持，这里的middleware和JBoss、WAS、WebSphere这类中间件有些区别，这里的middleware笔者更倾向于认为是一种请求链处理的中间环节。假设说我们有三个处理函数wsgi_a,wsgi_b和wsgi_c，在WSGI规范中我们可以把他们串联起来，例如请求先到达wsgi_a，然后wsgi_a处理后再传递给wsgi_b，接着再传递给wsgi_c，最后由wsgi_c返回具体的页面给WEB服务器并最终返回给浏览器。之所以能实现这种串联的原因是这里wsgi_a、wsgi_b和wsgi_c都基于WSGI标准，因此它们所接收的参数和返回值都是标准的。这类函数的作用有很多，例如可以提供cache，提供认证，提供审计功能等等。并且这类函数可以非常容易的复用，甚至是直接用在不同的项目中，因此在github里大家可以找到很多middleware的项目专门就是用来做WSGI middleware的。对于middleware的『串联』方法非常简单，例如：

```python
bing = SomeWsgiApplication()
ot = Middleware1(bing)
ree = Middleware2(ot)
```

此时我们运行ree后，请求会先交由Middleware2的逻辑处理，例如这里Middleware2会做个cache，然后请求会交给Middleware1处理，例如这里Middleware1会进行认证，最后这个请求会交给SomeWsgiApplication处理。

### Bootstrap

对于运维人员来说，开发自动化系统的前端页面是一个很头疼的问题。在以前笔者见到的很多系统，前端页面都很『简陋』，有时候笔者甚至认为不要用css直接用基础的html元素做出来的也比那些页面好看。但这种情况随着前端技术的更新迭代已经有了很大的改善，尤其是Bootstrap出现以后，对于一个没有太多前端技术的人来说要开发一个运维管控类系统的前端页面已经变的非常容易了。

Bootstrap是Twitter推出的一个用于前端开发的开源工具包，它提供了非常丰富的前端组件供用户使用，并且这些组件功能足以满足运维管控类系统的需求。相比较于用户自己重头写一个导航栏或者漫无目标的满互联网去找各种各样风格各异的导航来来说，用户可以直接在Bootstrap的文档中直接找到导航栏对应的控件，并且文档中有详细的例子教大家如何使用。另外Bootstrap只需要用户在文件中依赖两个js（其中一个是jQuery）和一个css文件即可，使用起来非常方便。同时Bootstrap提供了非常多的定制化主题功能，用户甚至可以直接在其官网通过鼠标调整控件的风格并下载具体的主题。在OGP中笔者就使用了这种功能。这一点其实类似于Wordpress，在没有Wordpress的时候用户想要搭建一个个人博客需要自己去开发博客的前端和后台，后来有了Wordpress甚至不会写代码的人也可以拥有自己的博客了，而Bootstrap对于运维人员来说就类似于Wordpress对于那些想要拥有自己博客的人一样，简直是带来了『光明』。

当然使用Bootstrap的一个问题是『同质化』。有兴趣的同学可以看下近几年一些介绍运维系统的PPT，大家会发现他们的UI很类似，因为大家都用的Bootstrap，所以看上去就大同小异了。

### 协程与eventlet

使用Python来提供WEB服务的一个问题是Python的并发性实在太差。Python是支持多线程的，并且一般这些线程对应的就是操作系统的线程，但问题是Python的解释器有一个GIL（全局解释性锁），当Python中的线程要运行的时候（也可以认为是对应的操作系统要运行的时候），这个线程必须要获取到这个GIL才行。因此如果某个线程虽然被操作系统分配到了时间片，但是却没有能够获取到GIL，那么这个线程还是得放入到等待队列中等待下次被调度。因此Python实际上是不支持这种通常意义上的多线程的。一般在编写需要多线程工作的应用时，Python的选择一般是使用多进程来满足这类需求，或者就是不使用Python而是改用其它语言。但由于Python提供前端WEB页面的框架实在是非常好用，因此必须有一个办法去绕过这个问题，此时就可以使用协程。

协程的原理很简单，打个比方就能讲明白了：假设说有十个人去食堂打饭，这个食堂比较穷，只有一个打饭的窗口，并且也只有一个打饭阿姨，那么打饭就只能一个一个排队来打咯。这十个人胃口很大，每个人都要点5个菜，但这十个人又有个毛病就是做事情都犹豫不决，所以点菜的时候就会站在那里，每点一个菜后都会想下一个菜点啥，因此后面的人等的很着急呀。这样一直站着也不是个事情吧，所以打菜的阿姨看到某个人犹豫5秒后就开始吼一声，会让他排到队伍最后去，先让别人打菜，等轮到他的时候他也差不多想好吃啥了。这确实是个不错的方法，但也有一个缺点，那就是打菜的阿姨会的等每个人5秒钟，如果那个人在5秒内没有做出决定吃啥，其实这5秒就是浪费了。一个人点一个菜就是浪费5秒，十个人每个人点5个菜可就浪费的多啦（菜都凉了要）。那咋办呢？这个时候阿姨发话了：大家都是学生，学生就要自觉，我以后也不主动让你们排到最后去了，如果你们觉得自己会犹豫不决，就自己主动点直接点一个菜就站后面去，等下次排到的时候也差不多想好吃啥了。这个方法果然有效，大家点了菜后想的第一件事情不是下一个菜吃啥，而是自己会不会犹豫，如果会犹豫那直接排到队伍后面去，如果不会的话就直接接着点菜就行了。这样一来整个队伍没有任何时间是浪费的，效率自然就高了。

这个例子里的排队阿姨的那声吼就是我们的CPU中断，用于切换上下文。每个打饭的学生就是一个task。而每个人自己决定自己要不要让出窗口的这种行为，其实就是我们协程的核心思想。

在用线程的时候，其实虽然CPU把时间给了你，你也不一定有活干，比如你要等IO、等信号啥的，这些时间CPU给了你你也没用呀。

在用协程的时候，CPU就不来分配时间了，时间由你们自己决定，你觉得干这件事情很耗时，要等IO啥的，你就干一会歇一会，等到等IO的时候就主动让出CPU，让别人上去干活，别人也是讲道理的，干一会也会把时间让给你。协程就是使用了这种思想，让编程者控制各个任务的运行顺序，从而最大可能的发挥CPU的性能。

协程的实现在网络上有一些代码，主要是通过把当前的寄存器暂存起来，然后以后回来的时候再恢复这些寄存器来实现的。我们可以把它想象成即便是单个线程，也有多个栈来存放线程上下文切换是的寄存器。而且这种切换是线程主动进行的，不是由时钟中断产生的。C的实现可以参考：http://www.cnblogs.com/sniperHW/archive/2012/06/19/2554574.html

Python中的协程一般是通过C模块的扩展来实现的，不过Python中有一个yield关键字，可以实现类似协程的效果，有兴趣的读者可以去了解下，顺便可以了解下Python中的生成器，它和yield的关系非常密切。

在OGP的portal中，我们通过Flask提供WEB框架，通过Bootstrap提供UI，而性能方面我们则通过eventlet来让我们的Flask运行在协程模式下。这种模式下的好处是当一个请求的处理逻辑中需要涉及到其它IO操作时，这个进程的当前协程就可以把时间片让出来给其它协程使用。通过这种方式我们可以极大的增强我们portal的并发处理能力，并能保证这样的并发能力对于一个运维管理平台来说是足够的了。

eventlet是一个基于greenlet的协程库，其实准确的说greenlet是一个C扩展的协程库，而eventlet在这个库上增加了很多功能，例如提供了绿化能力、提供了WSGI服务器等。eventlet基于greenlet以及我们前面异步通信中讲的select、epoll等功能，提供了一个功能强大的无IO阻塞框架。由于这个概念在很多框架中都有，例如tornado以及Python 3.5自带的标准异步操作，所以笔者在这里详细介绍下eventlet是如何实现这种无IO阻塞框架的。

首先我们看一个eventlet的简单例子：
```Python
import eventlet
from eventlet.green import urllib2

def print_html_return(url):
    body = urllib2.urlopen(url).read()
    print('html body len is {0}'.format(len(body)))

gt = eventlet.spawn(print_html_return, 'http://www.taobao.com')
gt2 = eventlet.spawn(print_html_return, 'http://www.tmall.com')
gt3 = eventlet.spawn(print_html_return, 'http://www.alibaba.com')

gt.wait()
gt2.wait()
gt3.wait()
```
在这个例子里，我们会的去访问三个url，获取到返回的html body的长度。和普通的方法不同的是，这里我们使用了eventlet的urllib2模块，这个模块的被eventlet给『绿化』过，这个模块被『绿化』后其read方法就是非阻塞的方法了，代码不会的等待网址返回body而卡在那里，而是会的交出自己的时间片，通过协程（可以参考小秦的这个文章）调用其他准备好了可以执行的代码。

这里的关键函数是spawn，其会启动一个协程，其实现为：
```python
def spawn(func, *args, **kwargs):
    """Create a greenthread to run ``func(*args, **kwargs)``.  Returns a
    :class:`GreenThread` object which you can use to get the results of the
    call.

    Execution control returns immediately to the caller; the created greenthread
    is merely scheduled to be run at the next available opportunity.
    Use :func:`spawn_after` to  arrange for greenthreads to be spawned
    after a finite delay.
    """
    hub = hubs.get_hub()
    g = GreenThread(hub.greenlet)
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    return g
```
可以看到，这里首先要获取一个hub。hub是啥呢？简单的说，hub就是一个管理协程的工具，hub的实现其实是个loop的循环，循环的上半部分代码是执行可以被执行的任务，循环的下半部分代码是使用epoll这类方法去监听可以使用的fd。hub的代码我们下面会看到详细的分析，先看这里的get_hub：
```python
def get_hub():
    """Get the current event hub singleton object.

    .. note :: |internal|
    """
    try:
        hub = _threadlocal.hub
    except AttributeError:
        try:
            _threadlocal.Hub
        except AttributeError:
            use_hub()
        hub = _threadlocal.hub = _threadlocal.Hub()
    return hub

...

_threadlocal = threading.local()
```

这里的意思就是先看看我们有没有已经建立了hub了，如果没有的话看看threadlocal.Hub这个类有没有，如果这个类也没有的话就先调用use_hub获取一个Hub类。由于我们才刚开始调用spawn，所以我们当然啥都没有，于是就进入到use_hub里了：
```python
def use_hub(mod=None):
    """Use the module *mod*, containing a class called Hub, as the
    event hub. Usually not required; the default hub is usually fine.

    Mod can be an actual module, a string, or None.  If *mod* is a module,
    it uses it directly.   If *mod* is a string and contains either '.' or ':'
    use_hub tries to import the hub using the 'package.subpackage.module:Class'
    convention, otherwise use_hub looks for a matching setuptools entry point
    in the 'eventlet.hubs' group to load or finally tries to import
    `eventlet.hubs.mod` and use that as the hub module.  If *mod* is None,
    use_hub uses the default hub.  Only call use_hub during application
    initialization,  because it resets the hub's state and any existing
    timers or listeners will never be resumed.
    """
    if mod is None:
        mod = os.environ.get('EVENTLET_HUB', None)
    if mod is None:
        mod = get_default_hub()
    if hasattr(_threadlocal, 'hub'):
        del _threadlocal.hub
    if isinstance(mod, six.string_types):
        assert mod.strip(), "Need to specify a hub"
        if '.' in mod or ':' in mod:
            modulename, _, classname = mod.strip().partition(':')
            mod = __import__(modulename, globals(), locals(), [classname])
            if classname:
                mod = getattr(mod, classname)
        else:
            found = False
            if pkg_resources is not None:
                for entry in pkg_resources.iter_entry_points(
                        group='eventlet.hubs', name=mod):
                    mod, found = entry.load(), True
                    break
            if not found:
                mod = __import__(
                    'eventlet.hubs.' + mod, globals(), locals(), ['Hub'])
    if hasattr(mod, 'Hub'):
        _threadlocal.Hub = mod.Hub
    else:
        _threadlocal.Hub = mod
```
代码不长，功能很简单，就是import我们需要的Hub模块。在这里我们会的通过get_default_hub通过import来确定能用的hub，最终import eventlet.hubs.epolls这个模块。

ok，Hub类有了，然后就是建立Hub对象了。epolls的构造方法为：
```python
class Hub(poll.Hub):
    def __init__(self, clock=time.time):
        BaseHub.__init__(self, clock)
        self.poll = epoll()
        try:
            # modify is required by select.epoll
            self.modify = self.poll.modify
        except AttributeError:
            self.modify = self.poll.register
```

可以看到他有一个poll的属性，这个属性就是以后用来epoll用的，下面会看到他的用处。另外这里父类的初始化方法也是很重要的：

```python
class BaseHub(object):
    """ Base hub class for easing the implementation of subclasses that are
    specific to a particular underlying event architecture. """

    SYSTEM_EXCEPTIONS = (KeyboardInterrupt, SystemExit)

    READ = READ
    WRITE = WRITE

    def __init__(self, clock=time.time):
        self.listeners = {READ: {}, WRITE: {}}
        self.secondaries = {READ: {}, WRITE: {}}
        self.closed = []

        self.clock = clock
        self.greenlet = greenlet.greenlet(self.run)
        self.stopping = False
        self.running = False
        self.timers = []
        self.next_timers = []
        self.lclass = FdListener
        self.timers_canceled = 0
        self.debug_exceptions = True
        self.debug_blocking = False
        self.debug_blocking_resolution = 1
```

这里的self.greenlet = greenlet.greenlet(self.run)就是协程调用的核心了，可以看到我们的hub就是一个协程，执行的代码是self.run。剧透下，当其他的通过类似spawn生成的方法阻塞然后释放时间片的时候，就会将时间片返回给hub，也就是返回给hub的self.run方法。因此这个self.run就是我们的核心调度器。

继续看我们的代码。获取了hub后，执行了下面的代码：
```python
g = GreenThread(hub.greenlet)
```
其实GreenThread就是greenlet的一个封装：
```python
class GreenThread(greenlet.greenlet):
    """The GreenThread class is a type of Greenlet which has the additional
    property of being able to retrieve the return value of the main function.
    Do not construct GreenThread objects directly; call :func:`spawn` to get one.
    """

    def __init__(self, parent):
        greenlet.greenlet.__init__(self, self.main, parent)
        self._exit_event = event.Event()
        self._resolving_links = False
```
可以看到，g = GreenThread(hub.greenlet)就是建立一个新的协程，也就是新建立一个greenlet，greenlet的父方法是我们的hub，也就是我们上面说的run。而自己这个协程运行的方法则是self.main：
```python
def main(self, function, args, kwargs):
    try:
        result = function(*args, **kwargs)
    except:
        self._exit_event.send_exception(*sys.exc_info())
        self._resolve_links()
        raise
    else:
        self._exit_event.send(result)
        self._resolve_links()
```
可以看到，如果这个协程的方法执行完毕，那么会发送一个result（或异常）给event，这个时候就能返回__main__了。
接着看代码：
```python
hub.schedule_call_global(0, g.switch, func, args, kwargs)
```
这里的实现是：
```python
def schedule_call_global(self, seconds, cb, *args, **kw):
    """Schedule a callable to be called after 'seconds' seconds have
    elapsed. The timer will NOT be canceled if the current greenlet has
    exited before the timer fires.
        seconds: The number of seconds to wait.
        cb: The callable to call after the given time.
        *args: Arguments to pass to the callable when called.
        **kw: Keyword arguments to pass to the callable when called.
    """
    t = timer.Timer(seconds, cb, *args, **kw)
    self.add_timer(t)
    return t
```
这里生成了一个timer对象，然后通过add_time将spawn建立的协程加到了hub的next_timer中：
```python
def add_timer(self, timer):
    scheduled_time = self.clock() + timer.seconds
    self.next_timers.append((scheduled_time, timer))
    return scheduled_time
```
在这之后，就回到了我们例子的main中了。在执行了余下的两个spawn后，可以想象我们目前有的东西有一个hub，这个hub有一个next_timer对象，next_timer对象有三个timer，且timer中有g.switch这个greenlet的执行切入口方法。但这个时候我们还是没有开始执行我们定义的print_html_return，而且代码还是在我们的main中，hub的run方法也没有开始呢，因此我们继续往下看，看看wait方法的含义：
```python
def wait(self):
    """ Returns the result of the main function of this GreenThread.  If the
    result is a normal return value, :meth:`wait` returns it.  If it raised
    an exception, :meth:`wait` will raise the same exception (though the
    stack trace will unavoidably contain some frames from within the
    greenthread module)."""
    return self._exit_event.wait()
```
这里的_exit_event的wait方法是：
```python
def wait(self):
    """Wait until another coroutine calls :meth:`send`.
    Returns the value the other coroutine passed to
    :meth:`send`.

    >>> from eventlet import event
    >>> import eventlet
    >>> evt = event.Event()
    >>> def wait_on():
    ...    retval = evt.wait()
    ...    print("waited for {0}".format(retval))
    >>> _ = eventlet.spawn(wait_on)
    >>> evt.send('result')
    >>> eventlet.sleep(0)
    waited for result

    Returns immediately if the event has already
    occured.

    >>> evt.wait()
    'result'
    """
    current = greenlet.getcurrent()
    if self._result is NOT_USED:
        self._waiters.add(current)
        try:
            return hubs.get_hub().switch()
        finally:
            self._waiters.discard(current)
    if self._exc is not None:
        current.throw(*self._exc)
    return self._result
```
可以看到，g调用wait后，会将自己加入到自己的event对象的_waiters中，然后调用hub的switch，切换到hub中执行hub的run方法。从这里开始我们的hub开始真正运行起来。看下hub的switch吧：
```python
def switch(self):
    cur = greenlet.getcurrent()
    assert cur is not self.greenlet, 'Cannot switch to MAINLOOP from MAINLOOP'
    switch_out = getattr(cur, 'switch_out', None)
    if switch_out is not None:
        try:
            switch_out()
        except:
            self.squelch_generic_exception(sys.exc_info())
    self.ensure_greenlet()
    try:
        if self.greenlet.parent is not cur:
            cur.parent = self.greenlet
    except ValueError:
        pass  # gets raised if there is a greenlet parent cycle
    clear_sys_exc_info()
    return self.greenlet.switch()
```
做了一些检查后，最后的self.greenlet.switch()互换出了我们hub的run，来看下我们的run：
```python
def run(self, *a, **kw):
    """Run the runloop until abort is called.
    """
    # accept and discard variable arguments because they will be
    # supplied if other greenlets have run and exited before the
    # hub's greenlet gets a chance to run
    if self.running:
        raise RuntimeError("Already running!")
    try:
        self.running = True
        self.stopping = False
        while not self.stopping:
            while self.closed:
                # We ditch all of these first.
                self.close_one()
            self.prepare_timers()
            if self.debug_blocking:
                self.block_detect_pre()
            self.fire_timers(self.clock())
            if self.debug_blocking:
                self.block_detect_post()
            self.prepare_timers()
            wakeup_when = self.sleep_until()
            if wakeup_when is None:
                sleep_time = self.default_sleep()
            else:
                sleep_time = wakeup_when - self.clock()
            if sleep_time > 0:
                self.wait(sleep_time)
            else:
                self.wait(0)
        else:
            self.timers_canceled = 0
            del self.timers[:]
            del self.next_timers[:]
    finally:
        self.running = False
        self.stopping = False
```
代码很短，所以很美。来看下重要部分的实现：
```python
self.prepare_timers()
```
timer其实就是我们之前spawn的时候注册的timer，prepare_timers的实现是：
```python
def prepare_timers(self):
    heappush = heapq.heappush
    t = self.timers
    for item in self.next_timers:
        if item[1].called:
            self.timers_canceled -= 1
        else:
            heappush(t, item)
    del self.next_timers[:]
```
这里会遍历我们的next_timers列表，将没有called的加入到一个heap中。然后清除我们的next_timers列表。继续看代码：
```python
self.fire_timers(self.clock())
...
def fire_timers(self, when):
    t = self.timers
    heappop = heapq.heappop

    while t:
        next = t[0]

        exp = next[0]
        timer = next[1]

        if when < exp:
            break

        heappop(t)

        try:
            if timer.called:
                self.timers_canceled -= 1
            else:
                timer()
        except self.SYSTEM_EXCEPTIONS:
            raise
        except:
            self.squelch_timer_exception(timer, sys.exc_info())
            clear_sys_exc_info()
```
这里会检查时间有时间上可以执行的timer，当有的时候调用timer()方法，其实现是：
```python
def __call__(self, *args):
    if not self.called:
        self.called = True
        cb, args, kw = self.tpl
        try:
            cb(*args, **kw)
        finally:
            try:
                del self.tpl
            except AttributeError:
                pass
```
可以看到，当调用timer的时候其实就是调用我们的print_html_return了。不过可能有人会问，如果只是这样的话，我们的异步呢？我们的epoll呢？所以为了代码能先继续看下去，这里先告诉大家print_html_return中的urllib2.urlopen(url).read()在请求发出开始等待接收数据的时候，会的将自己的socket的读的fd注册到hub中（因为他被『绿化』了），然后交出自己的时间片给hub。因此在这里我们可以知道，我们的三个timer都执行过了，并且都由于read请求而没有执行完，将自己的fd注册到了hub上，同时现在的时间片是交给了hub。因此我们先继续看代码：
```python
if wakeup_when is None:
    sleep_time = self.default_sleep()
else:
    sleep_time = wakeup_when - self.clock()
if sleep_time > 0:
    self.wait(sleep_time)
else:
    self.wait(0)
```
wait是个很关键的方法，不同的hub其实最大的不同之处就是wait的实现不通。来看下poll hub的wait：
```python
def wait(self, seconds=None):
    readers = self.listeners[READ]
    writers = self.listeners[WRITE]

    if not readers and not writers:
        if seconds:
            sleep(seconds)
        return
    try:
        presult = self.do_poll(seconds)
    except (IOError, select.error) as e:
        if get_errno(e) == errno.EINTR:
            return
        raise
    SYSTEM_EXCEPTIONS = self.SYSTEM_EXCEPTIONS

    if self.debug_blocking:
        self.block_detect_pre()

    # Accumulate the listeners to call back to prior to
    # triggering any of them. This is to keep the set
    # of callbacks in sync with the events we've just
    # polled for. It prevents one handler from invalidating
    # another.
    callbacks = set()
    for fileno, event in presult:
        if event & READ_MASK:
            callbacks.add((readers.get(fileno, noop), fileno))
        if event & WRITE_MASK:
            callbacks.add((writers.get(fileno, noop), fileno))
        if event & select.POLLNVAL:
            self.remove_descriptor(fileno)
            continue
        if event & EXC_MASK:
            callbacks.add((readers.get(fileno, noop), fileno))
            callbacks.add((writers.get(fileno, noop), fileno))

    for listener, fileno in callbacks:
        try:
            listener.cb(fileno)
        except SYSTEM_EXCEPTIONS:
            raise
        except:
            self.squelch_exception(fileno, sys.exc_info())
            clear_sys_exc_info()

    if self.debug_blocking:
        self.block_detect_post()
```
print_html_return中的urllib2.urlopen(url).read()在请求发出开始等待接收数据的时候，会的将自己的socket的读的fd注册到hub中，因此下面两行能拿到我们在等待接收数据的fd：
```python
readers = self.listeners[READ]
writers = self.listeners[WRITE]
```
然后就是do_poll啦，do_poll在epoll hub的实现就是普通的epoll：
```python
def do_poll(self, seconds):
    return self.poll.poll(seconds)
```
self.poll在上面提到过，就是一个python的epoll对象。在拿到我们可以使用的fd后，会的执行：
```python
for listener, fileno in callbacks:
    try:
        listener.cb(fileno)
    except SYSTEM_EXCEPTIONS:
        raise
    except:
        self.squelch_exception(fileno, sys.exc_info())
        clear_sys_exc_info()
```
不难猜测，我们的urllib2.urlopen(url).read()在交出时间片之前会注册回调函数以供回调。

hub的基本流程基本已经清楚了，整理下就是：
* 通过spawn建立一个GreenThread，GreenThread将自己要执行的func包装在self.main中通过timer加入到next_timer中，然后通过event调用hub的run，同时等待event的返回。self.main在func执行后会做event的send操作，让代码转到__main__中。
* hub的run是一个loop，loop显示查看next_timer中有没有要执行的协程，有的话就执行。所有要执行的timer都执行完后执行epoll之类的网络操作，用于处理spawn产生协程中func的需要等待fd的操作。

接下来我们来看『绿化』，『绿化』的目的是将python中原生的各种涉及网络fd的方法改为非阻塞的方法，自动的将fd加入epoll之类的调用中，而且过程对使用者透明。『绿化』后，一个阻塞的fd调用会注册到hub上，然后该方法立马返回，将时间片交给hub，hub执行了所有的timer后统一在loop的下半部执行epoll，处理等待数据的fd，从而实现提高网络调用的性能。

来看下urllib2.urlopen(url).read()：
```python
from eventlet import patcher
from eventlet.green import ftplib
from eventlet.green import httplib
from eventlet.green import socket
from eventlet.green import time
from eventlet.green import urllib

patcher.inject(
    'urllib2',
    globals(),
    ('httplib', httplib),
    ('socket', socket),
    ('time', time),
    ('urllib', urllib))

FTPHandler.ftp_open = patcher.patch_function(FTPHandler.ftp_open, ('ftplib', ftplib))

del patcher
```
可以看到import httplib2的时候会执行pather的inject方法，用于替换python内置的模块为green的模块，inject的实现核心是：
```python
for name, mod in additional_modules:
    sys.modules[name] = mod
```
那么『绿化』的模块是怎么和hub做到联动的呢，我们来看下『绿化』的socket的recv方法：
```python
def recv(self, buflen, flags=0):
    fd = self.fd
    if self.act_non_blocking:
        return fd.recv(buflen, flags)
    while True:
        try:
            return fd.recv(buflen, flags)
        except socket.error as e:
            if get_errno(e) in SOCKET_BLOCKING:
                pass
            elif get_errno(e) in SOCKET_CLOSED:
                return ''
            else:
                raise
        try:
            self._trampoline(
                fd,
                read=True,
                timeout=self.gettimeout(),
                timeout_exc=socket.timeout("timed out"))
        except IOClosed as e:
            # Perhaps we should return '' instead?
            raise EOFError()
```
可以看到，如果get_errno(e) in SOCKET_BLOCKING，那么就会的执行_trampoline，其会调用trampoline，后者的实现是：
```python
def trampoline(fd, read=None, write=None, timeout=None,
               timeout_exc=timeout.Timeout,
               mark_as_closed=None):
    """Suspend the current coroutine until the given socket object or file
    descriptor is ready to *read*, ready to *write*, or the specified
    *timeout* elapses, depending on arguments specified.

    To wait for *fd* to be ready to read, pass *read* ``=True``; ready to
    write, pass *write* ``=True``. To specify a timeout, pass the *timeout*
    argument in seconds.

    If the specified *timeout* elapses before the socket is ready to read or
    write, *timeout_exc* will be raised instead of ``trampoline()``
    returning normally.

    .. note :: |internal|
    """
    t = None
    hub = get_hub()
    current = greenlet.getcurrent()
    assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
    assert not (
        read and write), 'not allowed to trampoline for reading and writing'
    try:
        fileno = fd.fileno()
    except AttributeError:
        fileno = fd
    if timeout is not None:
        def _timeout(exc):
            # This is only useful to insert debugging
            current.throw(exc)
        t = hub.schedule_call_global(timeout, _timeout, timeout_exc)
    try:
        if read:
            listener = hub.add(hub.READ, fileno, current.switch, current.throw, mark_as_closed)
        elif write:
            listener = hub.add(hub.WRITE, fileno, current.switch, current.throw, mark_as_closed)
        try:
            return hub.switch()
        finally:
            hub.remove(listener)
    finally:
        if t is not None:
            t.cancel()
```
现在就很清楚了，这里会将我们的fd注册到hub中，然后按照上面代码中分析的那样进行调度。当fd可用的时候，current.switch就会从当前代码继续下去，进入到recv后while一次回到上面就可以正常接收数据了。

现在读者应该已经对eventlet是如何基于协程和IO复用来提供强大的非阻塞IO框架有一个基本的了解了，如果笔者了解了这里的东西，那么对于tornado等框架中的非阻塞IO也会很容易理解其实现。现在我们看下在OGP中使用eventlet提供Flask的WSGI服务的代码：

```python
import eventlet
from eventlet import wsgi
from flask import Flask, render_template, session, Blueprint
...
app = Flask(__name__, template_folder="../templates")
# 选择绿化的模块
eventlet.patcher.monkey_patch(os=False, select=True, socket=True,
                                  thread=False, time=True,
                                  psycopg=False, MySQLdb=False)
wsgi.server(eventlet.listen(('', int(CONF.server.port))), app)
```
是不是很简单？

### 代码

现在来看下OGP的代码。首先我们需要把异步的网络操作和我们的逻辑代码分隔开来，
