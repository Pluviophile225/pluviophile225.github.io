## Manager创建client

```go
package main

import (
	"context"
	"fmt"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"time"
)

func K8sRestConfig() *rest.Config {
	config, err := clientcmd.BuildConfigFromFlags("", "./resources/config")
	if err != nil {
		log.Fatal(err)
	}
	return config
}

func main() {
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger: logf.Log.WithName("test"),
		})
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		time.Sleep(time.Second * 3)
		pod := &v1.Pod{}
		err = mgr.GetClient().Get(context.TODO(),
			types.NamespacedName{Namespace: "default", Name: "gateway"}, pod)
		if err != nil {
			log.Fatal(err)
		}
		fmt.Println(pod)
	}()

	mgr.Start(context.Background())
}
```

## Manager的Get

get会走缓存，也可能不能缓存，这里想看看能不能饶过缓存读取资源。

具体的代码如下

当我们执行的时候，是这么走的

可以看到，会在这边进行判断，需不需要走缓存，如果是`isUncached`的话，就直接从Client中读取，否则从缓存中读取。

```go
// Get retrieves an obj for a given object key from the Kubernetes Cluster.
func (d *delegatingReader) Get(ctx context.Context, key ObjectKey, obj Object) error {
	if isUncached, err := d.shouldBypassCache(obj); err != nil {
		return err
	} else if isUncached {
		return d.ClientReader.Get(ctx, key, obj)
	}
	return d.CacheReader.Get(ctx, key, obj)
}
```

然后我们点进去shouldBypassCache函数，看看到底根据什么来判断需不需要走缓存。可以看到，这个是根据GVK来判断是不是在缓存中的。

```go
func (d *delegatingReader) shouldBypassCache(obj runtime.Object) (bool, error) {
	gvk, err := apiutil.GVKForObject(obj, d.scheme)
	if err != nil {
		return false, err
	}
	// TODO: this is producing unsafe guesses that don't actually work,
	// but it matches ~99% of the cases out there.
	if meta.IsListType(obj) {
		gvk.Kind = strings.TrimSuffix(gvk.Kind, "List")
	}
	if _, isUncached := d.uncachedGVKs[gvk]; isUncached {
		return true, nil
	}
	if !d.cacheUnstructured {
		_, isUnstructured := obj.(*unstructured.Unstructured)
		_, isUnstructuredList := obj.(*unstructured.UnstructuredList)
		return isUnstructured || isUnstructuredList, nil
	}
	return false, nil
}
```

修改下面一段代码，就是manager生成的时候的代码，这样以后走的就不是缓存了。

```go
mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger: logf.Log.WithName("test"),
			NewClient: func(cache cache.Cache, config *rest.Config, options client.Options, uncachedObjects ...client.Object) (client.Client, error) {
				return cluster.DefaultNewClient(cache, config, options, &v1.Pod{})
			},
		})
```

判断方式可以这样，就是我们下载下来controller-runtime的代码，然后在之前的`func (d *delegatingReader) Get(ctx context.Context, key ObjectKey, obj Object)`函数中打印内容，这样就可以判断出来是不是走的Cache了。

## Scheme

Scheme其实就是两个map，一个是根据GVK获取对象，一个是根据对象获取GVK。这个在初始化的时候就完成构建了。

现在只把corev1注册进来

```go
func main(){
  s := runtime.NewScheme()
  v1.AddToScheme(s)
  fmt.Println(s.AllKnownTypes)
}
```

把所有的注册进来

```go
func main(){
  s := runtime.NewScheme()
  scheme.AddToScheme(s)
  fmt.Println(s.AllKnownTypes)
}
```

那么上面的Get的逻辑是什么呢？

```go
// setOptionsDefaults set default values for Options fields.
func setOptionsDefaults(options Options) Options {
	// Use the Kubernetes client-go scheme if none is specified
	if options.Scheme == nil {
		options.Scheme = scheme.Scheme
	}
```

可以看到，如果没有设置Scheme，那么就会默认使用全部的Scheme，于是都有cache了。

## Manager中的Informer

使用Manager中的informer

需要注意的是，这里的GetInformer得到的Informer是在controller-runtime中的，与之前的是不同的，但是他们实现了同样的接口，所以可以转换成cache.SharedIndexInformer。然后来获取相关内容。

```go
func main() {
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger:    logf.Log.WithName("test"),
			Namespace: "default",
		})
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		time.Sleep(time.Second * 3)
		podInformer, _ := mgr.GetCache().GetInformer(context.Background(), &v1.Pod{})
		fmt.Println(podInformer.(cache.SharedIndexInformer).GetIndexer().ListKeys())

	}()

	mgr.Start(context.Background())
}
```

## Controller

`test.go`

```go
package main

import (
	"context"
	v1 "k8s.io/api/core/v1"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/source"
	"sigs.k8s.io/controller-runtime/pkg/tests/lib"
)

func K8sRestConfig() *rest.Config {
	config, err := clientcmd.BuildConfigFromFlags("", "./resources/config")
	if err != nil {
		log.Fatal(err)
	}
	return config
}

func main() {
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger:    logf.Log.WithName("test"),
			Namespace: "default",
		})
	if err != nil {
		log.Fatal(err)
	}
  // 这里就是开始手工创建controller，参数分别为，第一个是名字，第二个是manager，第三个是Options，主要填入Reconciler
	ctl, err := controller.New("abc", mgr, controller.Options{
		Reconciler: &lib.Ctl{},
	})
  // 设置监控资源
	src := &source.Kind{
		Type: &v1.Pod{},
	}
  // 放入的Queen
	hdler := &handler.EnqueueRequestForObject{}

	ctl.Watch(src, hdler)

	mgr.Start(context.Background())
}

```

`ctl.go`

```go
package lib

import (
	"context"
	"fmt"
	controllerruntime "sigs.k8s.io/controller-runtime"
)

type Ctl struct {
}
// 所谓的Reconciler就是实现了Reconcile这个借口，每次Reconcile的时候调用一下
func (*Ctl) Reconcile(ctx context.Context, req controllerruntime.Request) (controllerruntime.Result, error) {

	fmt.Println(req.NamespacedName)
	return controllerruntime.Result{}, nil
}
```

## 手工触发reconcile

这里的逻辑就是实现gin的一个网页，然后访问一次这个网页，就触发一次reconcile，其实就是把事件加入到队列中。然后给这个gin程序实现start接口，可以直接放入manager的add函数中。

```go
package lib

import (
	"context"
	"github.com/gin-gonic/gin"
	v1 "k8s.io/api/core/v1"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	cc "sigs.k8s.io/controller-runtime/pkg/internal/controller"
	"sigs.k8s.io/controller-runtime/pkg/manager"
)

type MyWeb struct {
	e   handler.EventHandler
	ctl controller.Controller
}

func NewMyWeb(e handler.EventHandler, ctl controller.Controller) *MyWeb {
	return &MyWeb{e: e, ctl: ctl}
}

func (m MyWeb) Start(ctx context.Context) error {
	//TODO implement me
	r := gin.New()
	r.GET("/add", func(c *gin.Context) {
		pod := &v1.Pod{}
		pod.Name = "mytestpod"
		pod.Namespace = "abc"
		m.e.Create(event.CreateEvent{Object: pod}, m.ctl.(*cc.Controller).Queue)
	})

	return r.Run(":8081")
}

var _ manager.Runnable = &MyWeb{}

```

## 其他资源变更触发Reconcile

```go
package lib

import (
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/util/workqueue"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"sigs.k8s.io/controller-runtime/pkg/source"
)

func AddCmWatch(ctl controller.Controller) error {
	src := &source.Kind{
		Type: &v1.ConfigMap{},
	}

	e := handler.Funcs{
		CreateFunc: func(event event.CreateEvent, limitingInterface workqueue.RateLimitingInterface) {
			limitingInterface.Add(reconcile.Request{
				types.NamespacedName{
					Name: "abc", Namespace: "bbb",
				},
			})
		},
	}

	return ctl.Watch(src, e)
}

```

## 工作队列

这个工作队列符合的特性

- Fair：先到先处理
- Stingy：同一事件在处理之前多次被添加，只处理一次
- 支持多消费者和生产者，支持运行时重新入队
- 关闭通知

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/util/workqueue"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"time"
)

func newItem(name, ns string) reconcile.Request {
	return reconcile.Request{
		NamespacedName: types.NamespacedName{
			Name:      name,
			Namespace: ns,
		}}
}

func main() {
	que := workqueue.New()
	go func() {
		for {
			item, _ := que.Get()
			fmt.Println(item.(reconcile.Request).NamespacedName)
			time.Sleep(time.Millisecond * 20)
		}
	}()
	for {
		que.Add(newItem("abc", "default"))
		que.Add(newItem("abc", "default"))
		que.Add(newItem("abc", "default"))
		time.Sleep(time.Second * 1)
	}
}

```

运行结果为一行`default/abc`

整个的逻辑是，有一个dirty的map，键存的是item的内容，所以后面的add发现dirty里面有了，就不会塞入了。

而Get的作用是将第一个item放入处理队列。如果只有一个的话，就会取出来，然后判断长度为0，就进行等待。进入等待状态。手动处理数据，最后执行`que.Done`就会唤醒之前的协程，继续往下走。

```go
package main

import (
	"fmt"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/util/workqueue"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"time"
)

func newItem(name, ns string) reconcile.Request {
	return reconcile.Request{
		NamespacedName: types.NamespacedName{
			Name:      name,
			Namespace: ns,
		}}
}

func main() {
	que := workqueue.New()
	go func() {
		for {
			item, _ := que.Get()
			fmt.Println(item.(reconcile.Request).NamespacedName)
			time.Sleep(time.Millisecond * 20)
      que.Done(item)
		}
	}()
	for {
		que.Add(newItem("abc", "default"))
		que.Add(newItem("abc", "default"))
		que.Add(newItem("abc", "default"))
		time.Sleep(time.Second * 1)
	}
}

```

如果对队列不做限制，那么加入的过程可能很快，这样负载就太高了。所以需要用限速队列来延迟加入一下。

限速队列 通过一些限速算法（令牌桶 go官方的）让数据延迟一定的时间后再加入队列

延迟队列 实现上面的延迟xx时间后加入

```go
package main

import (
	"fmt"
	"golang.org/x/time/rate"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/util/workqueue"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"strconv"
	"time"
)

func newItem(name, ns string) reconcile.Request {
	return reconcile.Request{
		NamespacedName: types.NamespacedName{
			Name:      name,
			Namespace: ns,
		}}
}

func main() {
	que := workqueue.NewRateLimitingQueue(&workqueue.BucketRateLimiter{
		Limiter: rate.NewLimiter(1, 1),
	})
	go func() {
		for {
			item, _ := que.Get()
			fmt.Println(item.(reconcile.Request).NamespacedName)
			time.Sleep(time.Millisecond * 20)
		}
	}()

	for i := 0; i < 10; i++ {
		que.AddRateLimited(newItem("abc"+strconv.Itoa(i), "default"))
	}

}

```

这样以后，每秒钟加入一个到队列

## golang限流器

```go
	limter := rate.NewLimiter(1, 3)
	for i := 0; ; i++ {
		err := limter.Wait(context.Background())
		if err != nil {
			log.Fatalln(err)
		}
		fmt.Println(i)
	}
```

一开始直接打印0、1、2

然后每过一秒打印一个数字

或者用下面这种方式实现，也是一样的

```go
	limter := rate.NewLimiter(1, 3)
	for i := 0; ; i++ {
		r := limter.Reserve()
		time.Sleep(r.Delay())
		fmt.Println(i)
	}
```

## 控制器的并发数设置

如果没有设置MaxConcurrentReconciles的话，那么是单线程的，reconcile需要等待。

业务里面，如果reconcile比较耗时间，就需要设置这个参数。一般3-5就可以了。

```go
func main() {
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger:    logf.Log.WithName("test"),
			Namespace: "default",
		})
	if err != nil {
		log.Fatal(err)
	}
	ctl, err := controller.New("abc", mgr, controller.Options{
		Reconciler:              &lib.Ctl{},
		MaxConcurrentReconciles: 2,
	})
	src := &source.Kind{
		Type: &v1.Pod{},
	}
	hdler := &handler.EnqueueRequestForObject{}

	ctl.Watch(src, hdler)
	mgr.Add(lib.NewMyWeb(hdler, ctl))
	lib.AddCmWatch(ctl)

	mgr.Start(context.Background())
}
```

## Own资源监听

```go
package main

import (
	"context"
	v1 "k8s.io/api/core/v1"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/source"
	"sigs.k8s.io/controller-runtime/pkg/tests/lib"
)

func K8sRestConfig() *rest.Config {
	config, err := clientcmd.BuildConfigFromFlags("", "./resources/config")
	if err != nil {
		log.Fatal(err)
	}
	return config
}

func main() {
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
			Logger:    logf.Log.WithName("test"),
			Namespace: "default",
		})
	if err != nil {
		log.Fatal(err)
	}
	ctl, err := controller.New("abc", mgr, controller.Options{
		Reconciler:              &lib.Ctl{},
		MaxConcurrentReconciles: 2,
	})
	src := &source.Kind{
		Type: &v1.Pod{},
	}
	//hdler := &handler.EnqueueRequestForObject{}
	ownObj := &v1.Pod{}
	hdler := &handler.EnqueueRequestForOwner{
		OwnerType: ownObj,
	}
	ctl.Watch(src, hdler)
	mgr.Add(lib.NewMyWeb(hdler, ctl, ownObj, mgr.GetScheme()))
	lib.AddCmWatch(ctl)

	mgr.Start(context.Background())
}



// myweb.go
package lib

import (
	"context"
	"github.com/gin-gonic/gin"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	cc "sigs.k8s.io/controller-runtime/pkg/internal/controller"
	"sigs.k8s.io/controller-runtime/pkg/manager"
)

type MyWeb struct {
	e      handler.EventHandler
	ctl    controller.Controller
	ownObj runtime.Object
	scheme *runtime.Scheme
}

func NewMyWeb(e handler.EventHandler, ctl controller.Controller, ownObj runtime.Object, scheme *runtime.Scheme) *MyWeb {
	return &MyWeb{e: e, ctl: ctl, ownObj: ownObj, scheme: scheme}
}

func (m MyWeb) Start(ctx context.Context) error {
	//TODO implement me
	r := gin.New()
	r.GET("/add", func(c *gin.Context) {
		pod := &v1.Pod{}
		pod.Name = "mytestpod"
		pod.Namespace = "mytest"
		controllerutil.SetOwnerReference(m.ownObj.(metav1.Object), pod, m.scheme)
		m.e.Create(event.CreateEvent{Object: pod}, m.ctl.(*cc.Controller).Queue)
	})

	return r.Run(":8081")
}

var _ manager.Runnable = &MyWeb{}

```

