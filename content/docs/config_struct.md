# Config

```go
type Config struct {
	// SecureServing is required to serve https
	SecureServing *SecureServingInfo

	// Authentication is the configuration for authentication
	Authentication AuthenticationInfo

	// Authorization is the configuration for authorization
	Authorization AuthorizationInfo

	// LoopbackClientConfig is a config for a privileged loopback connection to the API server
	// This is required for proper functioning of the PostStartHooks on a GenericAPIServer
	// TODO: move into SecureServing(WithLoopback) as soon as insecure serving is gone
  // LoopbackClientConfig 是为了一个与API server有特权的回环链接的配置
  // 这对GenericAPIServer的PostStartHooks允许来说是必要的
	LoopbackClientConfig *restclient.Config

	// EgressSelector provides a lookup mechanism for dialing outbound connections.
	// It does so based on a EgressSelectorConfiguration which was read at startup.
  // EgressSelector为呼叫外部链接提供了一个查找机制。
  // 这个是根据一开始读取的EgressSelectorConfigurationn建立的。
	EgressSelector *egressselector.EgressSelector

	// RuleResolver is required to get the list of rules that apply to a given user
	// in a given namespace
  // RuleResolver被需要来获取应用到指定命名空间的一个用户的规则列表
	RuleResolver authorizer.RuleResolver
	// AdmissionControl performs deep inspection of a given request (including content)
	// to set values and determine whether its allowed
  // AdmissionControl对给定请求做一个深度检查来设置值，并且决定是否被允许
	AdmissionControl      admission.Interface
	CorsAllowedOriginList []string
	HSTSDirectives        []string
	// FlowControl, if not nil, gives priority and fairness to request handling
  // FlowControl，如果不是空的，就给请求处理优先级和公平性
	FlowControl utilflowcontrol.Interface

	EnableIndex     bool
	EnableProfiling bool
	EnableDiscovery bool
	// Requires generic profiling enabled
	EnableContentionProfiling bool
	EnableMetrics             bool

	DisabledPostStartHooks sets.String
	// done values in this values for this map are ignored.
  // 忽略此映射中的done值
	PostStartHooks map[string]PostStartHookConfigEntry

	// Version will enable the /version endpoint if non-nil
  // 如果不为空，Version就会使用/version endpoint
	Version *version.Info
	// AuditBackend is where audit events are sent to.
  // AuditBackend是审计events被发送到的地方
	AuditBackend audit.Backend
	// AuditPolicyRuleEvaluator makes the decision of whether and how to audit log a request.
  // AuditPolicyRuleEvaluator决定是否以及如何审计一个请求
	AuditPolicyRuleEvaluator audit.PolicyRuleEvaluator
	// ExternalAddress is the host name to use for external (public internet) facing URLs (e.g. Swagger)
	// Will default to a value based on secure serving info and available ipv4 IPs.
  // ExternalAddress是对外面对URLs的域名
	ExternalAddress string

	// TracerProvider can provide a tracer, which records spans for distributed tracing.
  // TracerProvider提供了一个追踪者，为相关的链路记录下span内容
	TracerProvider *trace.TracerProvider

	//===========================================================================
	// Fields you probably don't care about changing
	//===========================================================================

	// BuildHandlerChainFunc allows you to build custom handler chains by decorating the apiHandler.
  // BuildHandlerChainFunc允许用户通过装饰apiHandler自己建一个handler chains
	BuildHandlerChainFunc func(apiHandler http.Handler, c *Config) (secure http.Handler)
	// HandlerChainWaitGroup allows you to wait for all chain handlers exit after the server shutdown.
  // HandlerChainWaitGroup允许你在服务器关闭后等所有chain handlers退出
	HandlerChainWaitGroup *utilwaitgroup.SafeWaitGroup
	// DiscoveryAddresses is used to build the IPs pass to discovery. If nil, the ExternalAddress is
	// always reported
  // DiscoveryAddresses被用来构建传给discovery的IP。如果他为空，那么ExternalAddress总是被记录的
	DiscoveryAddresses discovery.Addresses
	// The default set of healthz checks. There might be more added via AddHealthChecks dynamically.
  // 默认的健康检查。可能会动态添加一些通过AddHealthChecks方法
	HealthzChecks []healthz.HealthChecker
	// The default set of livez checks. There might be more added via AddHealthChecks dynamically.
  // 默认的生存检查。
	LivezChecks []healthz.HealthChecker
	// The default set of readyz-only checks. There might be more added via AddReadyzChecks dynamically.
  // 默认的就绪检查
	ReadyzChecks []healthz.HealthChecker
	// LegacyAPIGroupPrefixes is used to set up URL parsing for authorization and for validating requests
	// to InstallLegacyAPIGroup. New API servers don't generally have legacy groups at all.
  // LegacyAPIGroupPrefixes被用来构建InstallLegacyAPIGroup方法请求的验证的URL转换
  // 新的API服务端，是没有一点legacy群的。
	LegacyAPIGroupPrefixes sets.String
	// RequestInfoResolver is used to assign attributes (used by admission and authorization) based on a request URL.
	// Use-cases that are like kubelets may need to customize this.
  // RequestInfoResolver是用来基于请求URL指定属性的（被admission和authorization使用的）。
  // 像kubelets这种用户使用的可能需要定制自己的这个属性
	RequestInfoResolver apirequest.RequestInfoResolver
	// Serializer is required and provides the interface for serializing and converting objects to and from the wire
	// The default (api.Codecs) usually works fine.
  // Serializer是必须的，提供接口，来序列化和转化对象的
  // 默认的（api.Codecs）就可以工作得很好了。
	Serializer runtime.NegotiatedSerializer
	// OpenAPIConfig will be used in generating OpenAPI spec. This is nil by default. Use DefaultOpenAPIConfig for "working" defaults.
  // OpenAPIConfig是在生产OpenAPI spce的时候使用的。默认是空，会对默认值用DefaultOpenAPIConfig。
	OpenAPIConfig *openapicommon.Config
	// OpenAPIV3Config will be used in generating OpenAPI V3 spec. This is nil by default. Use DefaultOpenAPIV3Config for "working" defaults.
  // OpenAPIV3Config是在生产OpenAPI V3 spce的时候使用的。默认是空，会对默认值用DefaultOpenAPIV3Config。
	OpenAPIV3Config *openapicommon.Config
	// SkipOpenAPIInstallation avoids installing the OpenAPI handler if set to true.
  // 如果SkipOpenAPIInstallation被设置为true，就不会去安装OpenApI handler。
	SkipOpenAPIInstallation bool

	// RESTOptionsGetter is used to construct RESTStorage types via the generic registry.
  // RESTOptionsGetter通过generic registry来构建RESTStorage 类型。
	RESTOptionsGetter genericregistry.RESTOptionsGetter

	// If specified, all requests except those which match the LongRunningFunc predicate will timeout
	// after this duration.
  // 如果被设置，所有的请求（除了那些匹配LongRunningFunc 预测的）如果超过这个时间就算超时。
	RequestTimeout time.Duration
	// If specified, long running requests such as watch will be allocated a random timeout between this value, and
	// twice this value.  Note that it is up to the request handlers to ignore or honor this timeout. In seconds.
  // 如果被设置，长时间的请求（比如匹配）会被分配一个随机时间在MinRequestTimeout和2*MinRequestTimeout之间。但是这个是根据请求处理器自己决定是要忽略或者使用这个超时的。
	MinRequestTimeout int

	// This represents the maximum amount of time it should take for apiserver to complete its startup
	// sequence and become healthy. From apiserver's start time to when this amount of time has
	// elapsed, /livez will assume that unfinished post-start hooks will complete successfully and
	// therefore return true.
  // 这个代表的是apiserver可以完成他开始步骤和变成健康的最大时间。从apiserver的开始时间到这段时间走掉，/livez假设没有完成的post-start hooks将会成功完成并且返回true。
	LivezGracePeriod time.Duration
	// ShutdownDelayDuration allows to block shutdown for some time, e.g. until endpoints pointing to this API server
	// have converged on all node. During this time, the API server keeps serving, /healthz will return 200,
	// but /readyz will return failure.
  // ShutdownDelayDuration允许阻塞关闭一段时间。比如指向API server的endpoints被所有节点覆盖。在这段时间内，API server保持服务/healthz会返回200，但是/readyz会返回失败。
	ShutdownDelayDuration time.Duration

	// The limit on the total size increase all "copy" operations in a json
	// patch may cause.
	// This affects all places that applies json patch in the binary.
  // 在json patch上所有copy操作上的总大小的限制。这影响所有二进制中json patch
	JSONPatchMaxCopyBytes int64
	// The limit on the request size that would be accepted and decoded in a write request
	// 0 means no limit.
  // 这个限制的是请求大小，在写请求中被接受和解码的大小。0代表没有限制。
	MaxRequestBodyBytes int64
	// MaxRequestsInFlight is the maximum number of parallel non-long-running requests. Every further
	// request has to wait. Applies only to non-mutating requests.
  // MaxRequestsInFlight是并行的非长请求的最大数量。更多的请求需要等待。仅仅作用在non-mutating请求上。
	MaxRequestsInFlight int
	// MaxMutatingRequestsInFlight is the maximum number of parallel mutating requests. Every further
	// request has to wait.
  // MaxMutatingRequestsInFlight是并行执行的mutating请求的最大数量，更多的请求需要等待。
	MaxMutatingRequestsInFlight int
	// Predicate which is true for paths of long-running http requests
  // 预测哪个路径对long-running http请求是true的
	LongRunningFunc apirequest.LongRunningRequestCheck

	// GoawayChance is the probability that send a GOAWAY to HTTP/2 clients. When client received
	// GOAWAY, the in-flight requests will not be affected and new requests will use
	// a new TCP connection to triggering re-balancing to another server behind the load balance.
	// Default to 0, means never send GOAWAY. Max is 0.02 to prevent break the apiserver.
  // GowayChance是向HTTP/2客户端发送GOWAY请求的可能性。当客户端收到GOWAY，正在处理的请求不会被影响且新的请求可以使用一个新的TCP连接来通过负载均衡追踪和重新平衡到另一个server。
  // 默认是0，意味着从不发送GOWAY，最大是0.02来防止中断apiserver。
	GoawayChance float64

	// MergedResourceConfig indicates which groupVersion enabled and its resources enabled/disabled.
	// This is composed of genericapiserver defaultAPIResourceConfig and those parsed from flags.
	// If not specify any in flags, then genericapiserver will only enable defaultAPIResourceConfig.
  // MergeResourceConfig表明哪个groupVersion是可使用的还有它的资源是可用的还是不可用的。这个由genericapiserver defaultAPIResourceConfig组成并且这些从flags中得到。如果没有在flags中表明，genericapiserver仅仅使用defaultAPIResourceConfig。
	MergedResourceConfig *serverstore.ResourceConfig

	// lifecycleSignals provides access to the various signals
	// that happen during lifecycle of the apiserver.
	// it's intentionally marked private as it should never be overridden.
  // lifecycleSignals 提供了很多在apiserver的生存周期中发生的信号。
  // 最终会被标为私有的因为它不能被重写。
	lifecycleSignals lifecycleSignals

	// StorageObjectCountTracker is used to keep track of the total number of objects
	// in the storage per resource, so we can estimate width of incoming requests.
  // StorageObjectCountTracker是用来追踪每个资源在存储里的所有对象成员，这样我们可以判断来的请求的宽度。
	StorageObjectCountTracker flowcontrolrequest.StorageObjectCountTracker

	// ShutdownSendRetryAfter dictates when to initiate shutdown of the HTTP
	// Server during the graceful termination of the apiserver. If true, we wait
	// for non longrunning requests in flight to be drained and then initiate a
	// shutdown of the HTTP Server. If false, we initiate a shutdown of the HTTP
	// Server as soon as ShutdownDelayDuration has elapsed.
	// If enabled, after ShutdownDelayDuration elapses, any incoming request is
	// rejected with a 429 status code and a 'Retry-After' response.
  // ShutdownSendRetryAfter表示在apiserver优雅关闭期间表示什么时候去关闭HTTP服务器。如果是true，等待non longrunning请求运行被drained（？还没查什么意思）然后关闭HTTP服务器。如果是false，我们在ShutdownDelayDuration时间一过以后就立马关闭HTTP服务器。如果被应用了，在ShutdownDelayDuration时间过了以后，任何来的请求都会被用429状态代码拒绝还返回一个“Retry-After”响应。
	ShutdownSendRetryAfter bool

	//===========================================================================
	// values below here are targets for removal
	//===========================================================================

	// PublicAddress is the IP address where members of the cluster (kubelet,
	// kube-proxy, services, etc.) can reach the GenericAPIServer.
	// If nil or 0.0.0.0, the host's default interface will be used.
  // PublicAddress是集群成员可以访问到GenericAPIServer的IP地址
  // 如果是空或0.0.0.0，host的默认接口可以被使用。
	PublicAddress net.IP

	// EquivalentResourceRegistry provides information about resources equivalent to a given resource,
	// and the kind associated with a given resource. As resources are installed, they are registered here.
  // EquivalentResourceRegistry 提供了与给定资源相同的资源，以及与资源相关的种类，当资源被安装，他们在这里被注册。
	EquivalentResourceRegistry runtime.EquivalentResourceRegistry

	// APIServerID is the ID of this API server
  // APIServerID是这个API server的ID
	APIServerID string

	// StorageVersionManager holds the storage versions of the API resources installed by this server.
  // StorageVersionManager掌握被这个server安装的API资源的storage版本
	StorageVersionManager storageversion.Manager
}
```

## SecureServingInfo

```go
// SecureServing is required to serve https
```

SecureServing需要来服务https

```go
type SecureServingInfo struct {
	// Listener is the secure server network listener.
  // Listener是安全服务器网络listener
	Listener net.Listener

	// Cert is the main server cert which is used if SNI does not match. Cert must be non-nil and is
	// allowed to be in SNICerts.
  // 如果SNI不匹配的话，Cert是最重要的服务器cert。
  // Cert必须是非空的，而且需要被SNICerts包含。
	Cert dynamiccertificates.CertKeyContentProvider

	// SNICerts are the TLS certificates used for SNI.
  // SNICerts是被用来服务SNI的TLS认证证书
	SNICerts []dynamiccertificates.SNICertKeyContentProvider

	// ClientCA is the certificate bundle for all the signers that you'll recognize for incoming client certificates
  // ClientCA是准备为接下来到来的客户端认证承认的。起的是签发者的任务
	ClientCA dynamiccertificates.CAContentProvider

	// MinTLSVersion optionally overrides the minimum TLS version supported.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
  // MinTLSVersion选择重写可支持的最小TLS版本。
  // 值是从tls package contants中得到的
	MinTLSVersion uint16

	// CipherSuites optionally overrides the list of allowed cipher suites for the server.
	// Values are from tls package constants (https://golang.org/pkg/crypto/tls/#pkg-constants).
  // CipherSuits选择重写服务器支持的可允许的cipher列表
	CipherSuites []uint16

	// HTTP2MaxStreamsPerConnection is the limit that the api server imposes on each client.
	// A value of zero means to use the default provided by golang's HTTP/2 support.
  // HTTP2MaxStreamsPerConnection是在每个客户端上暴露的的限制。
	HTTP2MaxStreamsPerConnection int

	// DisableHTTP2 indicates that http2 should not be enabled.
  // DisableHTTP2表明http2不被允许。
	DisableHTTP2 bool
}
```

## AuthenticationInfo

```go
// Authentication is the configuration for authentication
```

Authentication是用来配置身份认证的。

```go
type AuthenticationInfo struct {
	// APIAudiences is a list of identifier that the API identifies as. This is
	// used by some authenticators to validate audience bound credentials.
  // APIAudiences是作为API身份的一个身份列表。一些身份验证器用它来验证身份。
	APIAudiences authenticator.Audiences
	// Authenticator determines which subject is making the request
  // 身份认证器决定哪个对象在做请求
	Authenticator authenticator.Request
}
```

## AuthorizationInfo

```go
// Authorization is the configuration for authorization
```

Authorization是用来配置鉴权的

```go
type AuthorizationInfo struct {
	// Authorizer determines whether the subject is allowed to make the request based only
	// on the RequestURI
  // Authorizer决定是否这个对象被允许根据RequestURI来做出响应
	Authorizer authorizer.Authorizer
}
```



