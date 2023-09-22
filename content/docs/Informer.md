## DeltaFIFO

这是个增量队列，队列满足先进先出FIFO（first in first out）

对于DeltaFIFO，只要实习了DeltaFIFOOptions就可以，不一定非要要求是pod

下面是自己实现的一个虚拟pod，只要完成一个接口即可。

```go
// 虚拟pod
type pod struct {
	Name  string
	Value int
}

func newPod(name string, v1 int) pod {
	return pod{Name: name, Value: v1}
}

func podKeyFunc(obj interface{}) (string, error) {
	return obj.(pod).Name, nil
```

接下来来模仿list watch机制，加入一些pod，注意这里面所有的pod都是自己定义的这个。

```go
func main() {
	df := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{KeyFunction: podKeyFunc})
	pod1 := newPod("pod1", 1)
	pod2 := newPod("pod2", 2)
	pod3 := newPod("pod3", 3)
	df.Add(pod1)
	df.Add(pod2)
	df.Add(pod3)

	fmt.Println(df.List())
}
```

![image-20230921114110389](../assets/image-20230921114110389.png)

然后我来使用pop一下，因为是队列，所以肯定有pop功能

```go
	df.Pop(func(obj interface{}, isInInitialList bool) error {
		for _, delta := range obj.(cache.Deltas) {
			fmt.Println(delta.Type, ":", delta.Object.(pod).Name)
		}
		return nil
	})
```

![image-20230921115205880](../assets/image-20230921115205880.png)

接着我们加入更新和删除的事件，然后再执行代码

```go
func main() {
	df := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{KeyFunction: podKeyFunc})
	pod1 := newPod("pod1", 1)
	pod2 := newPod("pod2", 2)
	pod3 := newPod("pod3", 3)
	df.Add(pod1)
	df.Add(pod2)
	df.Add(pod3)
	pod1.Value = 1.1
	df.Update(pod1)
	df.Delete(pod1)

	df.Pop(func(obj interface{}, isInInitialList bool) error {
		for _, delta := range obj.(cache.Deltas) {
			fmt.Println(delta.Type, ":", delta.Object.(pod).Name)
		}
		return nil
	})
}

```

![image-20230921115639500](../assets/image-20230921115639500.png)

会把pod1相关的内容都放在一起。然后我们就可以在这个里面进行switch类型进行回掉操作了。

```go
func main() {
	df := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{KeyFunction: podKeyFunc})
	pod1 := newPod("pod1", 1)
	pod2 := newPod("pod2", 2)
	pod3 := newPod("pod3", 3)
	df.Add(pod1)
	df.Add(pod2)
	df.Add(pod3)
	pod1.Value = 1.1
	df.Update(pod1)
	df.Delete(pod1)

	df.Pop(func(obj interface{}, isInInitialList bool) error {
		for _, delta := range obj.(cache.Deltas) {
			fmt.Println(delta.Type, ":", delta.Object.(pod).Name)
			switch delta.Type {
			case cache.Added:
				fmt.Println("执行新增回调")
			case cache.Updated:
				fmt.Println("执行更新回调")
			case cache.Deleted:
				fmt.Println("执行删除回调")
			}
		}
		return nil
	})
}

```

## Reflector

List的代码

```go
func main() {

	client := lib.InitClient()

	podLW := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())
	list, err := podLW.List(metav1.ListOptions{})
	if err != nil {
		log.Fatal(err)
	}
	podList := list.(*v1.PodList)
	for _, pod := range podList.Items {
		fmt.Println(pod.Name)
	}
}

```

watch的代码

```go
func main() {

	client := lib.InitClient()

	podLW := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())

	watcher, err := podLW.Watch(metav1.ListOptions{})

	if err != nil {
		log.Fatal(err)
	}

	for {
		select {
		case v, ok := <-watcher.ResultChan():
			if ok {
				fmt.Println(v.Type, ":", v.Object.(*v1.Pod).Name)
			}
		}
	}

}
```

创建Reflector并且进行队列取值

```go
package main

import (
	"fmt"

	"github.com/shenyisyn/informer/lib"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/client-go/tools/cache"
)

func main() {

	client := lib.InitClient()

	podLW := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())

	df := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{KeyFunction: cache.MetaNamespaceKeyFunc})
	rf := cache.NewReflector(podLW, &v1.Pod{}, df, 0)
	ch := make(chan struct{})

	go func() {
		rf.Run(ch)
	}()

	for {
		df.Pop(func(obj interface{}, isInInitialList bool) error {
			for _, delta := range obj.(cache.Deltas) {
				fmt.Println(delta.Type, ":", delta.Object.(*v1.Pod).Name)
			}
			return nil
		})
	}
}

```

接着是indexer，不设置indexer会丢失delete事件的

```go
func main() {

	client := lib.InitClient()

	podLW := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())
	store := cache.NewStore(cache.MetaNamespaceKeyFunc)
	df := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{
		KeyFunction:  cache.MetaNamespaceKeyFunc,
		KnownObjects: store,
	})
	rf := cache.NewReflector(podLW, &v1.Pod{}, df, 0)
	ch := make(chan struct{})

	go func() {
		rf.Run(ch)
	}()

	for {
		df.Pop(func(obj interface{}, isInInitialList bool) error {
			for _, delta := range obj.(cache.Deltas) {
				fmt.Println(delta.Type, ":", delta.Object.(*v1.Pod).Name)
				switch delta.Type {
				case cache.Sync, cache.Added:
					store.Add(delta.Object)
				case cache.Updated:
					store.Update(delta.Object)
				case cache.Deleted:
					store.Delete(delta.Object)
				}
			}
			return nil
		})
	}
}
```

简单的informer实现

```go
package watchdog

import (
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/tools/cache"
)

type WatchDog struct {
	lw      *cache.ListWatch
	objType runtime.Object
	h       cache.ResourceEventHandler

	reflector *cache.Reflector
	fifo      *cache.DeltaFIFO
	store     cache.Store
}

func NewWatchDog(lw *cache.ListWatch, objType runtime.Object, h cache.ResourceEventHandler) *WatchDog {

	store := cache.NewStore(cache.MetaNamespaceKeyFunc)

	fifo := cache.NewDeltaFIFOWithOptions(cache.DeltaFIFOOptions{
		KeyFunction:  cache.MetaNamespaceKeyFunc,
		KnownObjects: store,
	})

	reflector := cache.NewReflector(lw, objType, fifo, 0)

	return &WatchDog{lw: lw, objType: objType, h: h,
		reflector: reflector, fifo: fifo, store: store}
}

func (wd *WatchDog) Run() {
	ch := make(chan struct{})
	go func() {
		wd.reflector.Run(ch)
	}()

	for {
		wd.fifo.Pop(func(obj interface{}, isInInitialList bool) error {
			for _, delta := range obj.(cache.Deltas) {
				switch delta.Type {
				case cache.Sync:
					wd.store.Add(delta.Object)
					wd.h.OnAdd(delta.Object, true)
				case cache.Added:
					wd.store.Add(delta.Object)
					wd.h.OnAdd(delta.Object, false)
				case cache.Updated:
					if old, exists, err := wd.store.Get(delta.Object); err == nil && exists {
						wd.store.Update(delta.Object)
						wd.h.OnUpdate(old, delta.Object)
					}
				case cache.Deleted:
					wd.store.Delete(delta.Object)
					wd.h.OnDelete(delta.Object)
				}
			}
			return nil
		})
	}
}

```

测试

```go
package main

import (
	"fmt"

	"github.com/shenyisyn/informer/lib"
	"github.com/shenyisyn/informer/watchdog"
	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/client-go/tools/cache"
)

type PodHandler struct{}

// OnAdd implements cache.ResourceEventHandler.
func (*PodHandler) OnAdd(obj interface{}, isInInitialList bool) {
	fmt.Println("OnAdd:", obj.(*v1.Pod).Name)
}

// OnDelete implements cache.ResourceEventHandler.
func (*PodHandler) OnDelete(obj interface{}) {
	fmt.Println("OnDelete:", obj.(*v1.Pod).Name)
}

// OnUpdate implements cache.ResourceEventHandler.
func (*PodHandler) OnUpdate(oldObj interface{}, newObj interface{}) {
	fmt.Println("OnUpdate:", newObj.(*v1.Pod).Name)
}

var _ cache.ResourceEventHandler = &PodHandler{}

func main() {

	client := lib.InitClient()

	podLW := cache.NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())

	wd := watchdog.NewWatchDog(podLW, &v1.Pod{}, &PodHandler{})
	wd.Run()
}

```

## SharedInformer

sharedinformer主要用于处理针对一个事件，但是有很多个处理方式的时候的。

### ProcessorListener

```go
package cache

import (
	"fmt"
	"time"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

type PodHandler struct {
}

func (p PodHandler) OnAdd(obj interface{}, isInit bool) {
	fmt.Println("OnAdd:", obj.(metav1.Object).GetName())
}

func (p PodHandler) OnUpdate(obj interface{}, newObj interface{}) {
	fmt.Println("OnUpdate")
}

func (p PodHandler) OnDelete(obj interface{}) {
	fmt.Println("OnDelete")
}

func Test() {
	lis := newProcessListener(&PodHandler{}, 0, 0, time.Now(), 0, func() bool { return false })
	go func() {
		// 模拟外部插入
		count := 0
		for {
			pod := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: fmt.Sprintf("pod%d", count)}}
			time.Sleep(time.Second * 1)
			lis.addCh <- addNotification{
				newObj: pod,
			}
			count++
		}
	}()

	go func() {
		// 拿到值
		lis.pop()
	}()

	// 统一调用回调
	lis.run()
}

```

### 模拟简单的sharedinformer

sharedinformer的作用可以理解为，线程安全得运行每个handler。就是在上面的processorlistener的基础上加上了锁相关的内容。

```go
package cache

import (
	"fmt"
	"time"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/util/wait"
)

type PodHandler struct {
	msg string
}

func (p PodHandler) OnAdd(obj interface{}, isInit bool) {
	fmt.Println("OnAdd:"+p.msg, obj.(metav1.Object).GetName())
}

func (p PodHandler) OnUpdate(obj interface{}, newObj interface{}) {
	fmt.Println("OnUpdate" + p.msg)
}

func (p PodHandler) OnDelete(obj interface{}) {
	fmt.Println("OnDelete" + p.msg)
}

type MySharedInformer struct {
	processor *sharedProcessor
}

func NewMySharedInformer() *MySharedInformer {
	return &MySharedInformer{processor: &sharedProcessor{}}
}
func (msi *MySharedInformer) addEventHandler(handler ResourceEventHandler) {
	lis := newProcessListener(handler, 0, 0, time.Now(), 0, func() bool { return false })
	msi.processor.addListener(lis)
	go func() {
		// 模拟外部插入
		count := 0
		for {
			pod := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: fmt.Sprintf("pod%d", count)}}
			time.Sleep(time.Second * 1)
			lis.addCh <- addNotification{
				newObj: pod,
			}
			count++
		}
	}()

}

func (msi *MySharedInformer) start(ch <-chan struct{}) {
	msi.processor.run(ch)
}

func Test() {
	msi := NewMySharedInformer()
	msi.addEventHandler(&PodHandler{})
	msi.addEventHandler(&PodHandler{msg: "second"})
	msi.start(wait.NeverStop)
	// lis := newProcessListener(&PodHandler{}, 0, 0, time.Now(), 0, func() bool { return false })
	// go func() {
	// 	// 模拟外部插入
	// 	count := 0
	// 	for {
	// 		pod := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: fmt.Sprintf("pod%d", count)}}
	// 		time.Sleep(time.Second * 1)
	// 		lis.addCh <- addNotification{
	// 			newObj: pod,
	// 		}
	// 		count++
	// 	}
	// }()

	// go func() {
	// 	// 拿到值
	// 	lis.pop()
	// }()

	// // 统一调用回调
	// lis.run()
}
```

### 加入ListWatch手工运行

```go
package cache

import (
	"fmt"
	"time"

	"github.com/shenyisyn/informer/lib"
	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/fields"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
)

type PodHandler struct {
	msg string
}

func (p PodHandler) OnAdd(obj interface{}, isInit bool) {
	fmt.Println("OnAdd:"+p.msg, obj.(metav1.Object).GetName())
}

func (p PodHandler) OnUpdate(obj interface{}, newObj interface{}) {
	fmt.Println("OnUpdate" + p.msg)
}

func (p PodHandler) OnDelete(obj interface{}) {
	fmt.Println("OnDelete" + p.msg)
}

type MySharedInformer struct {
	processor *sharedProcessor
  // processor就是用来处理回调的，这里的sharedprocessor可以一个事件交个多个处理方式来处理，主要加了锁的逻辑
	reflector *Reflector
  // reflector是用来list和watch的，他需要通过配置监听的内容是什么，将这个内容放入到fifo中
	fifo      *DeltaFIFO
  // DeltaFIFO是一个队列，作用就是来存储变化的所有的类型和具体的内容的，使用Pop的方法可以来获取每个变化的内容。这个队列高级一些，会将同一个资源的内容统一处理。
	store     Store
  // store就是个indexer，就是为了存储一下的。是为了缓存，方便找到的作用
}

func NewMySharedInformer(lw *ListWatch, objType runtime.Object) *MySharedInformer {
	store := NewStore(MetaNamespaceKeyFunc)
	fifo := NewDeltaFIFOWithOptions(DeltaFIFOOptions{
		KeyFunction:  MetaNamespaceKeyFunc,
		KnownObjects: store,
	})
	reflector := NewReflector(lw, objType, fifo, 0)
	return &MySharedInformer{processor: &sharedProcessor{}, reflector: reflector, fifo: fifo, store: store}
}
func (msi *MySharedInformer) addEventHandler(handler ResourceEventHandler) {
	lis := newProcessListener(handler, 0, 0, time.Now(), 1024, func() bool { return false })
	msi.processor.addListener(lis)

}

func (msi *MySharedInformer) start(ch <-chan struct{}) {
	go func() {
		for {
			msi.fifo.Pop(func(obj interface{}, isInInitialList bool) error {
				for _, delta := range obj.(Deltas) {
					switch delta.Type {
					case Sync, Added:
						msi.store.Add(delta.Object)
						msi.processor.distribute(addNotification{newObj: delta.Object}, false)
					case Updated:
						if old, exists, err := msi.store.Get(delta.Object); err != nil && exists {
							msi.store.Update(delta.Object)
							msi.processor.distribute(updateNotification{newObj: delta.Object, oldObj: old}, false)
						}
					case Deleted:
						msi.store.Delete(delta.Object)
						msi.processor.distribute(deleteNotification{oldObj: delta.Object}, false)
					}
				}
				return nil
			})
		}
	}()
	go func() {
		msi.reflector.Run(ch)
	}()
	msi.processor.run(ch)
}

func Test() {
	client := lib.InitClient()
	podLw := NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())
	msi := NewMySharedInformer(podLw, &v1.Pod{})
	msi.addEventHandler(&PodHandler{})
	msi.start(wait.NeverStop)
	// lis := newProcessListener(&PodHandler{}, 0, 0, time.Now(), 0, func() bool { return false })
	// go func() {
	// 	// 模拟外部插入
	// 	count := 0
	// 	for {
	// 		pod := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: fmt.Sprintf("pod%d", count)}}
	// 		time.Sleep(time.Second * 1)
	// 		lis.addCh <- addNotification{
	// 			newObj: pod,
	// 		}
	// 		count++
	// 	}
	// }()

	// go func() {
	// 	// 拿到值
	// 	lis.pop()
	// }()

	// // 统一调用回调
	// lis.run()
}

```

### 索引

可以看到下面这个代码中，使用的Indexers是按照Namespace来构建的索引

```go
func Test() {
	indexers := Indexers{"namespace": MetaNamespaceIndexFunc}
	pod1 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "pod1", Labels: map[string]string{
		"app": "l1",
	}, Namespace: "ns1"}}
	pod2 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "pod2", Labels: map[string]string{
		"app": "l2",
	}, Namespace: "ns2"}}

	myindex := NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)
	myindex.Add(pod1)
	myindex.Add(pod2)

	objList, _ := myindex.IndexKeys("namespace", "ns1")
	for _, obj := range objList {
		fmt.Println(myindex.GetByKey(obj))
	}
}
```

如果我们做一些修改，要使得按照Label来构建索引，就可以这么写

```go
func LabelsIndexFunc(obj interface{}) ([]string, error) {
	meta, err := meta.Accessor(obj)
	if err != nil {
		return []string{""}, fmt.Errorf("object has no meta: %v", err)
	}
	return []string{meta.GetLabels()["app"]}, nil
}
func Test() {
	indexers := Indexers{"namespace": LabelsIndexFunc}
	pod1 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "pod1", Labels: map[string]string{
		"app": "l1",
	}, Namespace: "ns1"}}
	pod2 := &v1.Pod{ObjectMeta: metav1.ObjectMeta{Name: "pod2", Labels: map[string]string{
		"app": "l2",
	}, Namespace: "ns2"}}

	myindex := NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)
	myindex.Add(pod1)
	myindex.Add(pod2)

	objList, _ := myindex.IndexKeys("namespace", "ns1")
	for _, obj := range objList {
		fmt.Println(myindex.GetByKey(obj))
	}
}
```

### 索引集成到informer中

整个逻辑如下，运行结果如下图

```go
func LabelsIndexFunc(obj interface{}) ([]string, error) {
	meta, err := meta.Accessor(obj)
	if err != nil {
		return []string{""}, fmt.Errorf("object has no meta: %v", err)
	}
	return []string{meta.GetLabels()["app"]}, nil
}
func Test() {

	indexers := Indexers{"app": LabelsIndexFunc}
	myindex := NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers)
	go func() {
		r := gin.New()
		r.GET("/", func(c *gin.Context) {
			ret, _ := myindex.IndexKeys("app", "mysqltest")
			c.JSON(200, ret)
		})
		r.Run(":4000")
	}()
	client := lib.InitClient()
	podLw := NewListWatchFromClient(client.CoreV1().RESTClient(), "pods", "default", fields.Everything())
	msi := NewMySharedInformer(podLw, &v1.Pod{}, myindex)
	msi.addEventHandler(&PodHandler{})
	msi.start(wait.NeverStop)
}

```

那么对于这个index，是什么时候插入到呢？

其实就是在

```go
func (msi *MySharedInformer) start(ch <-chan struct{}) {
	go func() {
		for {
			msi.fifo.Pop(func(obj interface{}, isInInitialList bool) error {
				for _, delta := range obj.(Deltas) {
					switch delta.Type {
					case Sync, Added:
						msi.store.Add(delta.Object)
						msi.processor.distribute(addNotification{newObj: delta.Object}, false)
					case Updated:
						if old, exists, err := msi.store.Get(delta.Object); err != nil && exists {
							msi.store.Update(delta.Object)
							msi.processor.distribute(updateNotification{newObj: delta.Object, oldObj: old}, false)
						}
					case Deleted:
						msi.store.Delete(delta.Object)
						msi.processor.distribute(deleteNotification{oldObj: delta.Object}, false)
					}
				}
				return nil
			})
		}
	}()
	go func() {
		msi.reflector.Run(ch)
	}()
	msi.processor.run(ch)
}
```

这里里面的sync事件插入的。这里的store.add其实就是加入到了索引中了。

![image-20230922151440126](../assets/image-20230922151440126.png)

![image-20230922150803955](../assets/image-20230922150803955.png)

## SharedInformerFactory

```go
type MyFacotry struct {
	client    *kubernetes.Clientset
	informers map[reflect.Type]SharedIndexInformer
}

func (this *MyFacotry) PodInformer() SharedIndexInformer {
	if informer, ok := this.informers[reflect.TypeOf(&v1.Pod{})]; ok {
		return informer
	}
	podLW := NewListWatchFromClient(this.client.RESTClient(), "pods",
		"default", fields.Everything())
	indexers := Indexers{NamespaceIndex: MetaNamespaceIndexFunc}
	informer := NewSharedIndexInformer(podLW, &v1.Pod{}, 0, indexers)
	this.informers[reflect.TypeOf(&v1.Pod{})] = informer
	return informer
}

func NewMyFactory(client *kubernetes.Clientset) *MyFacotry {
	return &MyFacotry{client: client, informers: make(map[reflect.Type]SharedIndexInformer)}
}

func (this *MyFacotry) Start() {
	ch := wait.NeverStop
	for _, i := range this.informers {
		go func(informer SharedIndexInformer) {
			informer.Run(ch)
		}(i)
	}
}

func Test() {
	client := lib.InitClient()
	fact := NewMyFactory(client)
	fact.PodInformer().AddEventHandler(ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			fmt.Println(obj.(*v1.Pod).Name)
		},
	})

	fact.Start()

	r := gin.New()
	r.GET("/", func(ctx *gin.Context) {
		ctx.JSON(200, fact.PodInformer().GetIndexer().List())
	})
	r.Run(":4000")
}

```

