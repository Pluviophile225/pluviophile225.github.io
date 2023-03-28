# New_Config

```go
// NewConfig returns a Config struct with the default values
func NewConfig(codecs serializer.CodecFactory) *Config {
  // 这里是默认的健康检查，后面可以再看一下这个HealthChecker是怎么实现效果的
	defaultHealthChecks := []healthz.HealthChecker{healthz.PingHealthz, healthz.LogHealthz}
 	// 这里生成了id，提供了一下feature的默认值
	var id string
	if feature.DefaultFeatureGate.Enabled(features.APIServerIdentity) {
		id = "kube-apiserver-" + uuid.New().String()
	}
  // 新建一个lifecyclesignals
	lifecycleSignals := newLifecycleSignals()

	return &Config{
    // 用codec填充序列化属性
		Serializer:                  codecs,
    // 默认DefaultBuildHandlerChain，后面会逐个分析这个默认的Chain的filter的
		BuildHandlerChainFunc:       DefaultBuildHandlerChain,
    // 新建一个HandlerChainWaitGroup
		HandlerChainWaitGroup:       new(utilwaitgroup.SafeWaitGroup),
    // 新建一个string类型的sets，里面放DefaultLegacyAPIPrefix
		LegacyAPIGroupPrefixes:      sets.NewString(DefaultLegacyAPIPrefix),
    // 新建一个拒绝访问的PostStartHooks，是string类型的sets，默认为空
		DisabledPostStartHooks:      sets.NewString(),
    // 这里是新建的一个map，对应了PostStartHooks
		PostStartHooks:              map[string]PostStartHookConfigEntry{},
    // 在健康检查中放入默认检查器
		HealthzChecks:               append([]healthz.HealthChecker{}, defaultHealthChecks...),
    // 在就绪检查中放入默认检查器
		ReadyzChecks:                append([]healthz.HealthChecker{}, defaultHealthChecks...),
    // 在生存检查中放入默认检查器
		LivezChecks:                 append([]healthz.HealthChecker{}, defaultHealthChecks...),
    // 
		EnableIndex:                 true,
		EnableDiscovery:             true,
		EnableProfiling:             true,
		EnableMetrics:               true,
    // 最大在运行的请求为400
		MaxRequestsInFlight:         400,
    // 最大值运行的修改请求为200
		MaxMutatingRequestsInFlight: 200,
    // 请求超时时间默认为60秒（除了匹配longrunfunc预测的）
		RequestTimeout:              time.Duration(60) * time.Second,
    // 如果被设置，长时间的请求（比如匹配）会被分配一个随机时间在MinRequestTimeout和2*MinRequestTimeout之间。但是这个是根据请求处理器自己决定是要忽略或者使用这个超时的。
		MinRequestTimeout:           1800,
    // 这个代表的是apiserver可以完成他开始步骤和变成健康的最大时间。从apiserver的开始时间到这段时间走掉，/livez假设没有完成的post-start hooks将会成功完成并且返回true。
		LivezGracePeriod:            time.Duration(0),
    // ShutdownDelayDuration允许阻塞关闭一段时间。比如指向API server的endpoints被所有节点覆盖。在这段时间内，API server保持服务/healthz会返回200，但是/readyz会返回失败。
		ShutdownDelayDuration:       time.Duration(0),
    
		// 1.5MB is the default client request size in bytes
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest size
		// increase the "copy" operations in a json patch may cause.
    // 1.5MB是etcd服务端可以接受的默认客户端请求大小。
    // 请求体可能被编码成了json，然后被转换为porto当需要存储在etcd中，所以我们允许两倍的的最大大小来提升“copy”操作，这个操作在json patch的时候可能会使用。所以是3*1024*1024。
		JSONPatchMaxCopyBytes: int64(3 * 1024 * 1024),
		// 1.5MB is the recommended client request size in byte
		// the etcd server should accept. See
		// https://github.com/etcd-io/etcd/blob/release-3.4/embed/config.go#L56.
		// A request body might be encoded in json, and is converted to
		// proto when persisted in etcd, so we allow 2x as the largest request
		// body size to be accepted and decoded in a write request.
		// If this constant is changed, maxRequestSizeBytes in apiextensions-apiserver/third_party/forked/celopenapi/model/schemas.go
		// should be changed to reflect the new value, if the two haven't
		// been wired together already somehow.
    // 1.5MB是etcd服务端可以接受的默认客户端请求大小。请求体可能被编码成了json，然后被转换为porto当需要存储在etcd中，所以我们允许两倍的的最大大小来提升“copy”操作，这个操作在json patch的时候可能会使用。所以是3*1024*1024。
    // 如果这个常量被修改了，在apiextensions-apiserver/third_party/forked/celopenapi/model/schemas.go里的maxRequestSizeBytes也需要做相应更改。如果他们两个没有被绑定到一起的话
		MaxRequestBodyBytes: int64(3 * 1024 * 1024),

		// Default to treating watch as a long-running operation
		// Generic API servers have no inherent long-running subresources
    // 这个是用来将watch设置为默认的长时间操作
    // Generic API 服务器没有继承long-running的子资源
		LongRunningFunc:           genericfilters.BasicLongRunningRequestCheck(sets.NewString("watch"), sets.NewString()),
    // lifecycleSignals 提供了很多在apiserver的生存周期中发生的信号。
    // 最终会被标为私有的因为它不能被重写。
		lifecycleSignals:          lifecycleSignals,
    //StorageObjectCountTracker是用来追踪每个资源在存储里的所有对象成员，这样我们可以判断来的请求的宽度。
		StorageObjectCountTracker: flowcontrolrequest.NewStorageObjectCountTracker(lifecycleSignals.ShutdownInitiated.Signaled()),
		// 设置APIServerID
		APIServerID:           id,
    // 设置StorageVersionManager，直接使用默认的Manager
		StorageVersionManager: storageversion.NewDefaultManager(),
	}
}
```

