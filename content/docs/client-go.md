# Client-go介绍

- RESTClient：最基础的客户端，主要是对HTTP请求进行了封装，支持Json和Protobuf格式的数据。
- DiscoveryClient：发现客户端，负责发现APIServer支持的资源组、资源版本和资源信息的。
- ClientSet：负责操作Kubernetes内置的资源对象，例如：Pod、Service等。
- DynamicClient：动态客户端，可以对任意的Kubernetes资源对象进行通用操作，包括CRD。

![image-20230308134707881](../assets/image-20230308134707881.png)

# RESTClient

RESTClient是所有客户端的父类，它是最基础的客户端。

它提供了RESTful对应的方法的封装，如Get()、Put()、Post()、Delete()等。通过这些封装的方法与Kubernetes APIServer进行交互。

因为它是所有客户端的父类，所以它可以操作Kubernetes内置的所有资源对象以及CRD。

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
)

// 获取kube-system 这个namespace下的Pod列表

func main() {
	/*
		1.k8s 的配置文件
		2.保证开发机可以通过这个文件连接到k8s集群
	*/

	// 1. 加载配置文件，生成 config 对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2.配置 API 路径
	config.APIPath = "api" //pods /api/v1/pods

	// 3.配置分组版本
	config.GroupVersion = &corev1.SchemeGroupVersion // 无名资源组，group:"" ,version:"v1"

	// 4.配置数据的编解码工具
	config.NegotiatedSerializer = scheme.Codecs

	// 5.实例化 RESTClient 对象
	restClient, err := rest.RESTClientFor(config)
	if err != nil {
		panic(err.Error())
	}

	// 6.定义接收返回值的变量
	result := &corev1.PodList{}

	// 7.与 APIServer 交互
	err = restClient.
		Get().                                                         // Get请求方式
		Namespace("kube-system").                                      // 指定命名空间
		Resource("pods").                                              // 指定需要查询的资源，传递资源名称
		VersionedParams(&metav1.ListOptions{}, scheme.ParameterCodec). // 参数及参数的序列化工具
		Do(context.TODO()).                                            //触发请求
		Into(result)                                                   // 写入返回结果
	if err != nil {
		panic(err.Error())
	}

	for _, item := range result.Items {
		fmt.Printf("namespace: %v, name: %v\n", item.Namespace, item.Name)
	}

}

```

在`RESTClient`中真正发请求的部分

```go
func (r *Request) Do(ctx context.Context) Result {
	var result Result
	err := r.request(ctx, func(req *http.Request, resp *http.Response) {
		result = r.transformResponse(resp, req)
	})
	if err != nil {
		return Result{err: err}
	}
	if result.err == nil || len(result.body) > 0 {
		metrics.ResponseSize.Observe(ctx, r.verb, r.URL().Host, float64(len(result.body)))
	}
	return result
}

```

```markdown
* Get定义了请求方式，返回了一个Request结构体对象。这个 Request 结构体对象，就是构建访问 APIServer 请求用的。
* 依次执行了 Namespace 、Resource 、 VersionedParams ， 构建与 APIServer 交互的参数。
* Do 方法通过 request 发起请求，然后通过 transformResponse 解析请求返回，并绑定到对应资源对象的结构体对象上，这里的话，就表示是 corev1.PodList 的对象
* Do 里面的 Request 方法，先是检查了有没有可用的 Client ，在这里就开始调用 net/http 包的功能
```

# ClientSet

RESTClient虽然可用操作Kubernetes的所有资源对象，但是使用起来确实比较复杂，需要配置的参数过于频繁，因此，为了更优雅的更方便的与Kubernetes APIServer进行交互，需要进一步的封装。

ClientSet是基于RESTClient的封装，同时ClientSet是使用预生成的API对象与APIServer进行交互，这样做方便我们进行二次开发。

ClientSet是一组资源对象客户端的集合，例如负责操作Pods、Services等资源的CoreV1Client，负责操作Deployments、DaemonSets等资源的AppsV1Client等。通过这些资源对象提供的操作方法，即可对Kubernetes内置的资源对象进行Create、Update、Get、List、Delete等操作。

```go
package main

import (
	"context"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {

	// 1. 加载配置文件，生成config对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2. 实例化ClientSet
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	pods, err := clientset.CoreV1(). // 返回CoreV1Client实例
		Pods("kube-system"). // 查询 pod 列表
		List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err.Error())
	}

	for _, item := range pods.Items {
		fmt.Printf("namespace: %v, name: %v\n", item.Namespace, item.Name)
	}

}

```

首先是`CoreV1`的实例

```go
// CoreV1 retrieves the CoreV1Client
func (c *Clientset) CoreV1() corev1.CoreV1Interface {
	return c.coreV1
}
```

```go
func (c *CoreV1Client) Pods(namespace string) PodInterface {
	return newPods(c, namespace)
}
```

可以看到`pod`使用的方法是`newPods`

接着看`pods`这个方法，可以看到他传递的是`namespace`参数

```go
// newPods returns a Pods
func newPods(c *CoreV1Client, namespace string) *pods {
	return &pods{
		client: c.RESTClient(),
		ns:     namespace,
	}
}
```

可以看到接着是返回一个`interface`

```go
type PodsGetter interface {
	Pods(namespace string) PodInterface
}

// PodInterface has methods to work with Pod resources.
type PodInterface interface {
	Create(ctx context.Context, pod *v1.Pod, opts metav1.CreateOptions) (*v1.Pod, error)
	Update(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
	UpdateStatus(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
	Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
	DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
	Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Pod, error)
	List(ctx context.Context, opts metav1.ListOptions) (*v1.PodList, error)
	Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
	Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.Pod, err error)
	Apply(ctx context.Context, pod *corev1.PodApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Pod, err error)
	ApplyStatus(ctx context.Context, pod *corev1.PodApplyConfiguration, opts metav1.ApplyOptions) (result *v1.Pod, err error)
	UpdateEphemeralContainers(ctx context.Context, podName string, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)

	PodExpansion
}
```

然后使用了`List`这个接口

可以看到，clientset最后其实也是RESTClient的封装调用。

```go
// List takes label and field selectors, and returns the list of Pods that match those selectors.
func (c *pods) List(ctx context.Context, opts metav1.ListOptions) (result *v1.PodList, err error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	result = &v1.PodList{}
	err = c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Do(ctx).
		Into(result)
	return
}
```

```markdown
** 简单梳理为
* CoreV1 返回CoreV1Client实例对象
* Pods 调用了newPods函数，该函数返回的是PodInterface对象
* PodInterface对象实现了 Pods 资源相关的全部方法，同时在 newPods 里面还将 RESTClient 实例对象赋值给了对应的Client属性
* List 内使用 RESTClient 与 K8S APIServer 进行了交互
```

# DynamicClient

DynamicClient是一种动态客户端，通过动态指定资源组、资源版本和资源等信息，来操作任意的Kubernetes资源对象的一种客户端，即不仅仅是操作Kubernetes内置的资源对象，还可以操作CRD，这也是与ClientSet最明显的区别。

使用ClientSet的时候，程序会将所用的版本与类型紧密耦合。而DynamicClient使用嵌套的map[string]interface{}结构存储Kubernetes APIServer的返回值，使用反射机制，在运行的时候，进行数据绑定，这种方式更加灵活，但是却无法获取强数据类型的检查和验证。

- Object.runtime：Kubernetes中的所有资源对象，都实现了这个接口，其中包含DeepCopyObject和GetObjectKind的方法，分别用于对象深拷贝和获取对象的具体资源类型。
- Unstructured：包含map[string]interface{}类型字段，在处理无法预知结构的数据时，将数据值存入interface{}中，待运行时利用反射判断，该结构体提供了大量的工具方法，便于处理非结构化数据。

```go
package main

import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 1. 加载配置文件，生成 config 对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2. 实例化客户端对象，这里是实例化动态客户端对象
	dynamicClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// 3. 配置我们需要调用的 GVR
	gvr := schema.GroupVersionResource{
		Group:    "",
		Version:  "v1",
		Resource: "pods",
	}

	// 4. 发送请求且得到返回结果
	unStructData, err := dynamicClient.Resource(gvr).Namespace("kube-system").List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err.Error())
	}

	// 5. unStructData 转换为结构化的数据
	podList := &corev1.PodList{}
	err = runtime.DefaultUnstructuredConverter.FromUnstructured(
		unStructData.UnstructuredContent(),
		podList,
	)

	for _, item := range podList.Items {
		fmt.Printf("namespace： %v,name: %v\n", item.Namespace, item.Name)
	}
}
```

配置使用的`GVR`

可以看到动态客户端的实例对象

动态客户端其实底层就是RESTClient

```go
// NewForConfigAndClient creates a new dynamic client for the given config and http client.
// Note the http client provided takes precedence over the configured transport values.
func NewForConfigAndClient(inConfig *rest.Config, h *http.Client) (*DynamicClient, error) {
	config := ConfigFor(inConfig)
	// for serializing the options
	config.GroupVersion = &schema.GroupVersion{}
	config.APIPath = "/if-you-see-this-search-for-the-break"

	restClient, err := rest.RESTClientForConfigAndClient(config, h)
	if err != nil {
		return nil, err
	}
	return &DynamicClient{client: restClient}, nil
}
```

再去点一下，跳到dynamicClient的结构体

```go
type DynamicClient struct {
	client rest.Interface
}
```

找到`Resource`方法

```go
func (c *DynamicClient) Resource(resource schema.GroupVersionResource) NamespaceableResourceInterface {
	return &dynamicResourceClient{client: c, resource: resource}
}
```

`Resource`通过传递过来的GVR，得到动态资源客户端

`Namespace`中返回`ResourceInterface`中有很多方法的实现

```go
func (c *dynamicResourceClient) Namespace(ns string) ResourceInterface {
	ret := *c
	ret.namespace = ns
	return &ret
}
```

```go
func (c *dynamicResourceClient) List(ctx context.Context, opts metav1.ListOptions) (*unstructured.UnstructuredList, error) {
	if err := validateNamespaceWithOptionalName(c.namespace); err != nil {
		return nil, err
	}
	result := c.client.client.Get().AbsPath(c.makeURLSegments("")...).SpecificallyVersionedParams(&opts, dynamicParameterCodec, versionV1).Do(ctx)
	if err := result.Error(); err != nil {
		return nil, err
	}
	retBytes, err := result.Raw()
	if err != nil {
		return nil, err
	}
	uncastObj, err := runtime.Decode(unstructured.UnstructuredJSONScheme, retBytes)
	if err != nil {
		return nil, err
	}
	if list, ok := uncastObj.(*unstructured.UnstructuredList); ok {
		return list, nil
	}

	list, err := uncastObj.(*unstructured.Unstructured).ToList()
	if err != nil {
		return nil, err
	}
	return list, nil
}
```

```markdown
** 过程简介
* Resource方法，Resource方法基于 gvr 生成了一个针对于资源的客户端，也可以称为动态资源客户端。dynamicResourceClient
* Namespace，指定一个可操作的命名空间，同时namespace是动态客户端的方法
* List，也是动态客户端的方法，通过restClient调用k8s apiServer返回了pod的数据，这个返回的数据格式是二进制的json格式，然后通过一系列的解析方法转换为unstructed.UnstructuredList的数据
```

# DiscoveryClient

DiscoveryClient针对GVR。用于查看当前Kubernetes集群支持的那些资源组、资源版本、资源信息。

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {

	// 1. 加载配置文件，生成 config 对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2. 实例化客户端
	discoveryClient, err := discovery.NewDiscoveryClientForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// 3. 发送请求，获取GVR数据
	_, apiResources, err := discoveryClient.ServerGroupsAndResources()
	if err != nil {
		panic(err.Error())
	}

	for _, list := range apiResources {
		gv, err := schema.ParseGroupVersion(list.GroupVersion)
		if err != nil {
			panic(err.Error())
		}
		for _, resource := range list.APIResources {
			fmt.Printf("name:%v, group:%v, version:%v\n", resource.Name, gv.Group, gv.Version)
		}
	}

}
```

首先看`ServerGroupsAndResources`

```go
func ServerGroupsAndResources(d DiscoveryInterface) ([]*metav1.APIGroup, []*metav1.APIResourceList, error) {
	var sgs *metav1.APIGroupList
	var resources []*metav1.APIResourceList
	var err error

	// If the passed discovery object implements the wider AggregatedDiscoveryInterface,
	// then attempt to retrieve aggregated discovery with both groups and the resources.
	if ad, ok := d.(AggregatedDiscoveryInterface); ok {
		var resourcesByGV map[schema.GroupVersion]*metav1.APIResourceList
		sgs, resourcesByGV, err = ad.GroupsAndMaybeResources()
		for _, resourceList := range resourcesByGV {
			resources = append(resources, resourceList)
		}
	} else {
		sgs, err = d.ServerGroups()
	}

	if sgs == nil {
		return nil, nil, err
	}
	resultGroups := []*metav1.APIGroup{}
	for i := range sgs.Groups {
		resultGroups = append(resultGroups, &sgs.Groups[i])
	}
	if resources != nil {
		return resultGroups, resources, nil
	}

	groupVersionResources, failedGroups := fetchGroupVersionResources(d, sgs)

	// order results by group/version discovery order
	result := []*metav1.APIResourceList{}
	for _, apiGroup := range sgs.Groups {
		for _, version := range apiGroup.Versions {
			gv := schema.GroupVersion{Group: apiGroup.Name, Version: version.Version}
			if resources, ok := groupVersionResources[gv]; ok {
				result = append(result, resources)
			}
		}
	}

	if len(failedGroups) == 0 {
		return resultGroups, result, nil
	}

	return resultGroups, result, &ErrGroupDiscoveryFailed{Groups: failedGroups}
}
```

首先是获取所有的`gv`，然后将获取的`gv`传入函数`fetchGroupVersionResources`

其中最核心的是`ServerResourcesForGroupVersion`

```go
func fetchGroupVersionResources(d DiscoveryInterface, apiGroups *metav1.APIGroupList) (map[schema.GroupVersion]*metav1.APIResourceList, map[schema.GroupVersion]error) {
	groupVersionResources := make(map[schema.GroupVersion]*metav1.APIResourceList)
	failedGroups := make(map[schema.GroupVersion]error)

	wg := &sync.WaitGroup{}
	resultLock := &sync.Mutex{}
	for _, apiGroup := range apiGroups.Groups {
		for _, version := range apiGroup.Versions {
			groupVersion := schema.GroupVersion{Group: apiGroup.Name, Version: version.Version}
			wg.Add(1)
			go func() {
				defer wg.Done()
				defer utilruntime.HandleCrash()

				apiResourceList, err := d.ServerResourcesForGroupVersion(groupVersion.String())

				// lock to record results
				resultLock.Lock()
				defer resultLock.Unlock()

				if err != nil {
					// TODO: maybe restrict this to NotFound errors
					failedGroups[groupVersion] = err
				}
				if apiResourceList != nil {
					// even in case of error, some fallback might have been returned
					groupVersionResources[groupVersion] = apiResourceList
				}
			}()
		}
	}
	wg.Wait()

	return groupVersionResources, failedGroups
}
```

```markdown
** 
* 1. ServerGroups负责获取GV数据，然后调用fetchGroupVersionResources方法，本质还是RESTClient，且给这个方法传递 GV 参数。
* 2. 然后通过调用ServerResourcesForGroupVersion方法获取 GV 对应的 Resource 数据，也就是资源数据
* 3. 同时返回一个map[gv]resourceList的数据格式，最后处理map -> slice，然后返回 GVR slice。
```

## 将GVR缓存到本地

GVR数据其实是很少变动的，因此我们可以将GVR的数据，缓存在本地，来减少Client与APIServer交互，因为交互存在网络损耗。

在discovery/cached中，有另外两个客户端是来实现将GVR数据缓存到本地文件中和内存中的，分别是CachedDiscoveryClient 和 menCacheClient。

平常管理k8s集群的kubectl命令也是用这种方式来使用GVR与APIServer交互的，它的缓存文件默认是在~/.kube/cache

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/discovery/cached/disk"
	"k8s.io/client-go/tools/clientcmd"
	"time"
)

func main() {

	// 1. 加载配置文件，生成config对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2. 实例化客户端，本客户端负责将 GVR 数据，缓存到本地文件中的。
	cacheDiscoveryClient, err := disk.NewCachedDiscoveryClientForConfig(config, "../cache/discovery", "../cache/http", time.Minute*60)
	if err != nil {
		panic(err.Error())
	}
	_, apiResources, err := cacheDiscoveryClient.ServerGroupsAndResources()
	// 1. 先从缓存文件中找 GVR 数据，有则直接返回，没有则需要调用 APIServer
	// 2. 调用 APIServer 获取GVR数据
	// 3. 将获取到的 GVR 数据缓存到本地，然后返回个客户端
	for _, list := range apiResources {
		gv, err := schema.ParseGroupVersion(list.GroupVersion)
		if err != nil {
			panic(err.Error())
		}
		for _, resource := range list.APIResources {
			fmt.Printf("name:%v, group:%v, version:%v\n", resource.Name, gv.Group, gv.Version)
		}
	}

}
```

# Informer机制

![image-20230310113419637](../assets/image-20230310113419637.png)

Informer负责与Kubernetes APIServer进行Watch操作，可以是Kubernetes内置资源对象，也可以是CRD。

Informer是一个带有本地缓存以及索引机制的核心工具包，当请求为 查询 操作的时候，会优先从本地缓存内去查找数据，而 创建、更新、删除，这类操作，则会根据事件通知写入到队列DeltaFIFO中，同时对应的事件处理过后，更新本地缓存，使本地缓存与ETCD的数据保持一致。

Informer抽象出来的这个缓存层，将查询相关操作的压力接收了下来，这样就不必每次去调用APIServer的接口，减轻了APIServer的数据交互压力。

- Reflector：使用List-Watch来保证本地缓存数据的准确性、顺序性和一致性，List对应资源的全量列表数据，Watch负责变化部分的数据，Watch指定的Kubernetes的资源，当watch的资源发生变化时，触发变更的时间，比如Added，Updated和Deleted事件，并将资源对象的变化事件存放到本地队列DeltaFIFO中。
- DeltaFIFO：是一个增量队列，记录了资源变化的过程，Reflector就当对于队列的生产者。这个组件可以拆分为两部分来理解，FIFO就是一个队列，拥有队列基本方法，例如ADD，UPDATE，DELETE，LIST，POP，CLOSE等，Delta是一个资源对象存储，保存存储对象的消费类型，比如Added，Updated，Deleted等。
- Indexer：用来存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存到Indexer，Indexer与Etcd中数据完全保持一致，从而client-go可以本地读取，减少Kubernetes APIServer的数据交互压力。

## List-Watch

List-Watch机制是Kubernetes中的异步消息通知机制，通过它能有效得确保消息的实时性、顺序性和可靠性。

List-Watch，分为两个部分。

- List负责调用资源对应的Kubernetes APIServer的RESTful API获取全局数据列表，并同步到本地缓存中。
- Watch负责监听资源的变化，并调用相应事件的处理函数进行处理，同时更新本地缓存，使本地缓存与Etcd中数据，保持一致。

List是基于HTTP中的短链接实现。Watch则是基于HTTP长链接实现，Watch使用长链接的方式，极大得减轻了Kubernetes APIServer的访问压力。

```go
package main

import (
	"context"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 1.加载配置，生成 config 对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2.实例化客户端，clientSet
	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// 3.调用监听的方法
	w, err := clientSet.AppsV1().Deployments("default").Watch(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err.Error())
	}

	for {
		select {
		case e, _ := <-w.ResultChan():
			fmt.Println(e.Type, e.Object)
			// e.Type：表示事件变化的类型，Added，Deleted
			// e.Object: 表示变化后的数据
		}
	}

}
```

![image-20230310133010044](../assets/image-20230310133010044.png)

## Reflector

Reflector是Client-go中，用来监听指定资源的组件，当资源发生变化的时候，例如增加、更新、删除等操作的时候，会以事件的形式存入本地队列，然后有对应的方法处理。

在实例化Reflector的过程中，有一个ListWatcher的接口对象，这个结构对象有两个方法，分别是List和Watch，这两个方法就实现了List-Watch功能。

Reflector的核心逻辑可以简单归为三部分：

- List：调用List方法获取资源全部列表数据，转换为资源对象列表，然后保存到本地缓存中。
- 定时同步：定时器定时触发同步机制，定时更新缓存数据，在Reflector的结构体对象中，是可以配置定时同步的周期时间的。
- Watch：监听资源的变化，并且调用对应的事件处理函数来进行处理。

Reflector组件对于数据同步更新，都是基于ResourceVersion来进行的，每个资源对象都会有ResourceVersion这个属性，当数据发送变化的时候，ResourceVersion也将会以递增的形式更新，这样就确保了变更事件的顺序性。

Watch 资源对象的时候，ResourceVersion可以根据我们的需求进行配置。

- ResourceVersion未设置的时候，从最新版本开始监听。
  - 为了建立初始状态，Watch从起始资源版本中存在的所有资源实例的合成“添加”事件开始。以下所有监视事件都针对在Watch开始的资源版本之后发生的所有更改。
- ResourceVersion设置为“0”的时候，则表示从任意版本开始监听。
  - 以这种方式初始化的监视可能返回任意陈旧的数据。首选可用的最新资源版本，但不是必需的。允许任何起始资源版本。由于分区或过时的缓存，Watch可能从客户端之前观察到的更旧的资源版本开始，特别是在高可用性配置中。不能容忍这种明显倒带的，客户不应该用这种语义启动Watch。
  - 为了建立初始状态，Watch从起始资源版本中存在的所有资源实例的合成“添加”事件开始。以下所有监视事件都针对在Watch开始的资源版本之后发生的所有更改。
- ResourceVersion从指定版本开始监听。
  - 监视事件适用于提供的资源版本之后的所有更改。Watch不会以所提供资源版本的合成“添加”事件启动。由于客户端提供了资源版本，因此假定客户端已经具有起始资源版本的初始状态。

ResourceVersion不仅是在Reflector中有重要的应用，Update机制或Patch机制也是会基于ResourceVersion来比较两个资源对象，确定是否有变化的。

当Watch资源断开的时候，Reflector会重新进行List-Watch以确保数据的可靠性。

同时Watch使用HTTP的长链接的形式进行资源的监听，保证了数据实时性的同时，还减轻Kubernetes APIServer的访问压了。

## DeltaFIFO

DeltaFIFO是一个增量的本地队列，记录了资源对象的变化过程。

它的生产者是Reflector组件。将监听的对象，同步到DeltaFIFO中。

FIFO就是一个先入先出的本地队列，Delta则是资源对象的变化，例如增加、删除、修改等。

FIFO负责接收Reflector传递过来的事件，并将其按照顺序存储，然后等待事件的处理函数进行处理，同时若出现多个相同的事件，则只会被处理一次。

FIFO既然是一个队列那么肯定有队列相关的操作方法，在这里就是通过Queue这个接口对象，来实现队列所需的方法的，同时还根据需求拓展了一些其他的方法。例如：Pop、AddIfNotPresent等。

Delta有两个属性，分别是Type和Object。

- Type就表示这个事件的类型，就比如说Added表示增加，Updated表示更新等等。
- Object是一个interface类型的，它就表示一个具体Kubernetes资源对象，例如：Pod、Service等。

## Indexer

Indexer是Informer中LocalStore部分。

Indexer本身是一个存储，同时在存储的基础上扩展了索引的功能。

Reflector通过DeltaFIFO一系列的操作后，然后更新存储到Indexer中。

Indexer中的数据，与Etcd中的数据是完全一致的，当client-go需要获取数据的时候，则无须每次都从APIServer中获取，从而减轻了APIServer的请求压力。

- IndexFunc：索引器函数，用于计算一个资源对象的索引值列表，可用根据需求定义其他的，比如根据Label标签、Annotation等属性来生成索引值列表。
- Index：存储数据，要查找某个命名空间下面的Pod，那就要让Pod按照其命名空间进行索引，对应的Index类型就是map[namespace]sets.pod。
- Indexers：存储索引器，key为索引器名称，value为索引器的实现函数，例如：map["namespace"]MetaNamespaceIndexFunc。
- Indices：存储缓存器，key为索引器名称，value为缓存的数据，例如：map["namespace"]map[namespace]sets.pod。

Indexers与Indices容易混淆，Indexers是存储索引（生成索引键）的，Indices里面是存储真正的数据（对象键）。

```go
package main

import (
	"fmt"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/tools/cache"
)

/*
- IndexFunc：索引器函数，用于计算一个资源对象的索引值列表，可用根据需求定义其他的，比如根据Label标签、Annotation等属性来生成索引值列表。
- Index：存储数据，要查找某个命名空间下面的Pod，那就要让Pod按照其命名空间进行索引，对应的Index类型就是map[namespace]sets.pod。
- Indexers：存储索引器，key为索引器名称，value为索引器的实现函数，例如：map["namespace"]MetaNamespaceIndexFunc。
- Indices：存储缓存器，key为索引器名称，value为缓存的数据，例如：map["namespace"]map[namespace]sets.pod。
*/

/*
	1. 实现两个索引器函数，分别基于Namespace、NodeName，资源对象Pod
*/

func NamespaceIndexFunc(obj interface{}) (result []string, err error) {
	pod, ok := obj.(*v1.Pod)
	if !ok {
		return nil, fmt.Errorf("类型错误,%v", err)
	}

	result = []string{pod.Namespace}

	return
}

func NodeNameIndexFunc(obj interface{}) (result []string, err error) {
	pod, ok := obj.(*v1.Pod)
	if !ok {
		return nil, fmt.Errorf("类型错误,%v", err)
	}

	result = []string{pod.Spec.NodeName}

	return
}

func main() {
	// 实例一个 Indexer 对象
	index := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{
		"namespace": NamespaceIndexFunc,
		"nodeName":  NodeNameIndexFunc,
	})

	//模拟数据
	pod1 := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "index-pod-1",
			Namespace: "default",
		},
		Spec: v1.PodSpec{
			NodeName: "node1",
		},
	}
	pod2 := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "index-pod-2",
			Namespace: "default",
		},
		Spec: v1.PodSpec{
			NodeName: "node2",
		},
	}
	pod3 := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "index-pod-3",
			Namespace: "kube-system",
		},
		Spec: v1.PodSpec{
			NodeName: "node3",
		},
	}

	// 将数据写入到index中
	_ = index.Add(pod1)
	_ = index.Add(pod2)
	_ = index.Add(pod3)

	// 通过索引器函数查询一下数据
	pods, err := index.ByIndex("namespace", "default")
	if err != nil {
		panic(err.Error())
	}

	for _, pod := range pods {
		fmt.Println(pod.(*v1.Pod).Name)
	}

	fmt.Println("-----------------nodeName IndexFunc--------------------")

	pods, err = index.ByIndex("nodeName", "node1")
	if err != nil {
		panic(err.Error())
	}

	for _, pod := range pods {
		fmt.Println(pod.(*v1.Pod).Name)
	}
}

```

**ThreadSafeMap**

> k8s.io/client-go/tools/cache/thread_safe_store.go

这是一个并发安全的存储，Indexer就是在ThreadSafeMap基础上进行封装的，实现了索引相关的功能。

**Cache**

> k8s.io/client-go/tools/cache/store.go

Cache是对ThreadSafeStore的一个再次封装，很多操作都是直接调用的ThreadSafeStore的操作实现的。

##  SharedInformer

Informer会将资源缓存在本地以供后续使用。

Kubernetes中运行了很多控制器，有很多资源需要管理，难免会出现一个资源受到多个控制器管理。

可以通过SharedInformer来创建一份供多个控制器共享的缓存。这样就不用重复缓存资源，同时也减少了系统的内存开销。

使用了SharedInformer之后，不管有多少个控制器同时读取事件，SharedInformer只会调用一个Watch API来watch上游的API Server，大大降低了API Server的负载。实际上kube-controller-manager就是这么工作的。

SharedInformer是一个接口，包含添加事件，当有资源变化时，会回调通知使用者，启动函数及获取是否全局对象是否已经同步到本地存储中。

SharedInformer一般是使用SharedInformerFactory来管理控制器需要的资源对象的Informer实例，使用map的结构进行存储资源对象的Informer。

**SharedIndexInformer**

SharedIndexInformer在ShanredInformer基础上扩展了添加和获取Indexers的能力。

**SharedInformerFactory**

SharedInformerFactroy为所有已知的GVR提供共享的Informer。

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// 使用 ClientSet 来演示 sharedInformer
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	clientSet, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// 初始化 SharedInformerFactory
	sharedInformerFactory := informers.NewSharedInformerFactory(clientSet, 0)

	// 查询 pod 数据，那么就生成 PodInformer 对象
	podInformer := sharedInformerFactory.Core().V1().Pods()

	// 生成一个 indexer ，便于数据的查询
	indexer := podInformer.Lister()

	// 启动 informer
	sharedInformerFactory.Start(nil)

	// 等待数据同步完成
	sharedInformerFactory.WaitForCacheSync(nil)

	// 利用 indexer 获取数据
	pods, err := indexer.List(labels.Everything())
	if err != nil {
		panic(err.Error())
	}

	for _, items := range pods {
		fmt.Printf("namespace:%v,name:%v\n", items.GetNamespace(), items.GetName())
	}
}
```

# Client-Go调用封装

`client/client.go`

```go
package client

import (
	"k8s.io/client-go/discovery"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

type Clients struct {
	clientSet       kubernetes.Interface
	discoveryClient discovery.DiscoveryClient
}

func NewClients() (clients Clients) {

	// 1. 加载配置,生成配置文件对象
	config, err := clientcmd.BuildConfigFromFlags("", "../kubeconfig")
	if err != nil {
		panic(err.Error())
	}

	// 2. 实例化各种客户端
	clients.clientSet, err = kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	return
}

func (c *Clients) ClientSet() kubernetes.Interface {
	return c.clientSet
}

```

`informer/informer.go`

```go
package informer

import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/informers"
	"myclient/pkg/client"
	"time"
)

var sharedInformerFactory informers.SharedInformerFactory

func NewSharedInformerFactory(stopCh <-chan struct{}) (err error) {
	var (
		clients client.Clients
	)
	// 1. 加载客户端
	clients = client.NewClients()

	// 2. 实例化 sharedInformerFactory
	sharedInformerFactory = informers.NewSharedInformerFactory(clients.ClientSet(), time.Second*60)

	// 3. 启动 informer
	gvrs := []schema.GroupVersionResource{
		{Group: "", Version: "v1", Resource: "pods"},
		{Group: "", Version: "v1", Resource: "services"},
		{Group: "", Version: "v1", Resource: "namespaces"},

		{Group: "apps", Version: "v1", Resource: "deployments"},
		{Group: "apps", Version: "v1", Resource: "statefulsets"},
		{Group: "apps", Version: "v1", Resource: "daemonsets"},
	}

	for _, v := range gvrs {
		// 创建 informer
		_, err = sharedInformerFactory.ForResource(v)
		if err != nil {
			return
		}
	}

	// 启动所有创建的 informer
	sharedInformerFactory.Start(stopCh)

	// 等待所有 informer 全量同步数据完成
	sharedInformerFactory.WaitForCacheSync(stopCh)

	return
}

func Get() informers.SharedInformerFactory {
	return sharedInformerFactory
}

```

`main.go`

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/labels"
	"myclient/pkg/informer"
)

func main() {
	stopCh := make(chan struct{})

	err := informer.NewSharedInformerFactory(stopCh)
	if err != nil {
		panic(err)
	}

	items, err := informer.Get().Core().V1().Pods().Lister().List(labels.Everything())
	if err != nil {
		panic(err)
	}

	for _, v := range items {
		fmt.Printf("namespace:%v,name:%v\n", v.Namespace, v.Name)
	}

}
```

# Gin + Client-Go 与 Kubernetes APIServer交互

```go
package main

import (
	"github.com/gin-gonic/gin"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/proxy"
	"k8s.io/client-go/rest"
	"myclient/pkg/client"
	"myclient/pkg/informer"
	"net/http"
)

func main() {
	stopCh := make(chan struct{})

	// 进行了 client-go 的初始化操作
	err := informer.Setup(stopCh)
	if err != nil {
		panic(err.Error())
	}

	// 启动一个 web服务
	// 1. 实例化 Gin
	r := gin.Default()

	// 2. 写路由
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"code":    20000,
			"message": "pong",
		})
	})

	r.GET("/pod/list", func(c *gin.Context) {

		items, err := informer.Get().Core().V1().Pods().Lister().List(labels.Everything())
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code":    40000,
				"message": err.Error(),
			})
			return
		}

		c.JSON(http.StatusOK, gin.H{
			"code":    20000,
			"message": "success",
			"data":    items,
		})
	})

	// 动态获取资源数据
	r.GET("/:resource/:group/:version", func(c *gin.Context) {

		resource := c.Param("resource")
		group := c.Param("group")
		version := c.Param("version")

		gvr := schema.GroupVersionResource{
			Group:    group,
			Version:  version,
			Resource: resource,
		}

		// 通过 gvr 获取到对应资源对象的 informer

		i, err := informer.Get().ForResource(gvr)
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code":    40000,
				"message": err.Error(),
			})
			return
		}

		items, err := i.Lister().List(labels.Everything())
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code":    40000,
				"message": err.Error(),
			})
			return
		}

		c.JSON(http.StatusOK, gin.H{
			"code":    20000,
			"message": "success",
			"data":    items,
		})
	})

	// 转发请求到 k8s apiserver
	r.Any("/apis/*action", func(c *gin.Context) {

		// transport http.client 的一个封装，也是 client-go 封装的
		t, err := rest.TransportFor(client.GetConfig())
		if err != nil {
			panic(err.Error())
		}

		s := *c.Request.URL
		s.Host = "192.168.199.100:6443" // k8s 的 apiserver 地址
		s.Scheme = "https"

		httpProxy := proxy.NewUpgradeAwareHandler(&s, t, true, false, nil)
		httpProxy.UpgradeTransport = proxy.NewUpgradeRequestRoundTripper(t, t)
		httpProxy.ServeHTTP(c.Writer, c.Request)

	})

	// 3. 执行指定方法
	_ = r.Run(":8888")

}
```

