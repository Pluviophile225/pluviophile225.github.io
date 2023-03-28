## k8s源码准备

### Debug

#### Go-Delve

> 开源项目，为Go语言做debug

```shell
go install github.com/go-delve/delve/cmd/dlv@latest
```

#### 启动集群

1、编译参数

> -w -s,保留文件名，行号；
>
> -gcflags="all=-N -l" 禁止优化和内联

![image-20230221160322143](../assets/image-20230221160322143.png)

> 修改build_binaries()函数

![image-20230222131715168](../assets/image-20230222131715168.png)

2、启动本地集群

用`make clean`清除未编译的可执行程序。通过`hack/local-up-cluster.sh`脚本启动本地集群

![image-20230221161202237](../assets/image-20230221161202237.png)

![image-20230221162014324](../assets/image-20230221162014324.png)

可以看到起来了这些组件

3、重启API Server

![image-20230221162943255](../assets/image-20230221162943255.png)

```text
sudo dlv --headless exec /home/kubernetes/go/src/k8s.io/kubernetes/_output/local/bin/linux/amd64/kube-apiserver --listen=:12345 --api-version=2 --log --log-output=debugger,gdbwire,lldbout,debuglineerr,rpc,dap,fncall,minidump --log-dest=/home/kubernetes/delve-log/log -- --authorization-mode=Node,RBAC  --cloud-provider= --cloud-config=   --v=3 --vmodule= --audit-policy-file=/tmp/kube-audit-policy-file --audit-log-path=/tmp/kube-apiserver-audit.log --authorization-webhook-config-file= --authentication-token-webhook-config-file= --cert-dir=/var/run/kubernetes --egress-selector-config-file=/tmp/kube_egress_selector_configuration.yaml --client-ca-file=/var/run/kubernetes/client-ca.crt --kubelet-client-certificate=/var/run/kubernetes/client-kube-apiserver.crt --kubelet-client-key=/var/run/kubernetes/client-kube-apiserver.key --service-account-key-file=/tmp/kube-serviceaccount.key --service-account-lookup=true --service-account-issuer=https://kubernetes.default.svc --service-account-jwks-uri=https://kubernetes.default.svc/openid/v1/jwks --service-account-signing-key-file=/tmp/kube-serviceaccount.key --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,Priority,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction --disable-admission-plugins= --admission-control-config-file= --bind-address=0.0.0.0 --secure-port=6443 --tls-cert-file=/var/run/kubernetes/serving-kube-apiserver.crt --tls-private-key-file=/var/run/kubernetes/serving-kube-apiserver.key --storage-backend=etcd3 --storage-media-type=application/vnd.kubernetes.protobuf --etcd-servers=http://127.0.0.1:2379 --service-cluster-ip-range=10.0.0.0/24 --feature-gates=AllAlpha=false --external-hostname=localhost --requestheader-username-headers=X-Remote-User --requestheader-group-headers=X-Remote-Group --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-client-ca-file=/var/run/kubernetes/request-header-ca.crt --requestheader-allowed-names=system:auth-proxy --proxy-client-cert-file=/var/run/kubernetes/client-auth-proxy.crt --proxy-client-key-file=/var/run/kubernetes/client-auth-proxy.key --cors-allowed-origins="/127.0.0.1(:[0-9]+)?$,/localhost(:[0-9]+)?$"
```

这些内容需要拷贝下来一会会用到的。

 接着关掉API Server

![image-20230221202102287](../assets/image-20230221202102287.png)

验证一下结果

![image-20230221202127972](../assets/image-20230221202127972.png)

![image-20230222130811029](../assets/image-20230222130811029.png)

再加上log的目标地址等等

4、连接Debug Server

![image-20230222135449730](../assets/image-20230222135449730.png)

#### 命令行中调试

![image-20230222135705964](../assets/image-20230222135705964.png)

#### VS Code中调试

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Connect to server",
            "type": "go",
            "request": "attach",
            "mode": "remote",
            "port":12345,
            "host":"192.168.1.108"
        }
    ]
}
```

然后打断点进行远程调试

#### Postman请求Server

注意：使用的时候要先加入这个代码

>  export KUBECONFIG=/var/run/kubernetes/admin.kubeconfig

除此之外：如果使用Centos

```markdown
** 
#重新声明环境变量
ll /etc/kubernetes/admin.conf                                               #查看文件是否存在,如果不存在执行下面的步骤
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile      #重新写入环境变量
source ~/.bash_profile
```

1、建立Service Account

![image-20230222171823777](../assets/image-20230222171823777.png)

![image-20230222171859257](../assets/image-20230222171859257.png)

2、建立Secret（>=1.24）

新建一个secret

![image-20230222172445176](../assets/image-20230222172445176.png)

创建这个secret

![image-20230222172658679](../assets/image-20230222172658679.png)

![image-20230222172732090](../assets/image-20230222172732090.png)

![image-20230222172826048](../assets/image-20230222172826048.png)

3、建立ClusterRole

先查看有哪些权限

![image-20230222172934310](../assets/image-20230222172934310.png)

然后创建一个rolebinding来使得forpostman拥有权限

![image-20230222173419088](../assets/image-20230222173419088.png)

4、获取Secret中Token

```shell
cluster/kubectl.sh get secret postman-sa-secret -o jsonpath="{.data['ca\.crt']}" | base64 -d > /tmp/ca.crt
```

![image-20230222174152383](../assets/image-20230222174152383.png)

5、提取Secret中证书

把`ca.crt`下载下来

6、下载证书并上传到Postman

查看secret的token并放入postman中

![image-20230222195957412](../assets/image-20230222195957412.png)

![image-20230222200353320](../assets/image-20230222200353320.png)

在postman中添加证书

## API-Server

### 源码重要文件夹

> ../cmd/kube-apiserver

API Server可执行程序的主入口，基于cobra，主要负责接受命令行参数，把api server启动起来。也是代码学习的入口

> ../pkg

大部分k8s的源码所在地，除了被抽离为单独组件的部分。例如api server的代码，proxy组件的代码，kubelet组件的代码等等

> ../plugin

Kubernetes内建的plugin实现，包含admission和auth两个部分

> ../vender/k8s.io 和 ../staging/src/k8s.io

Vendor机制是老一代依赖包管理机制，module是新一代，不过vendor目录存在的话还是会被优先使用；staging中包含正在被单独抽离的组件，软引用到vendor下

> ../pkg/api 和 ../pkg/apis

Api文件夹下包含和OpenAPI相关的模型定义等内容，用于根据OpenAPI规范形成符合其规定的API 而apis是包含内建API Groups和API Objects的，和scheme相关的代码大部分在这里

### API Server启动过程

**API Object**

是Kubernetes内部管理的基本元素，是k8s在etcd中信息存储单元。

例如Deployment，Pod，Service，都是API Object。代码内部也常用“API”来称呼它们

**API Group**

一组API Object组成的一个具有共有性质的对象集合。

例如apps这个group，它由Deployment，ReplicaSet，StatefulSet等等API Object组成的

**Legacy API Object**

绝大多数的API Object都被归在某个API Group下面，特别是新版本中引入的一定会遵从这一原则。但在Kubernetes项目初始阶段所引入的API Object没有显示定义在API Group下面，例如Pod，Event，Node等等。在代码中有时也称它们为“core” API Object

#### Cobra简介

> AppName Cmd Arg --Flag = xxx

1. 定义一个命令对象

   ```go
   cmd = cobra.Command{
       Use: "..."
       Short: "..."
       ... ...
   }
   ```

   对应于命令模板里的"Command"

2. 为命令指定参数"Flag"

   不同于Argument，Flag是控制程序行为的既定参数，它们可以通过cmd的Flags()方法获取和设置

   `cmd.Flags().BoolVar(&p,......)`

   `cmd.Flags().StringVar(&p,......)`

3. 编写flag和Arg校验逻辑

   校验都可以通过cmd的方法进行，例如`cmd.MarkFlagsRequiredTogether("a","b")`

   `cmd.ExactArgs(10)`或在`cmd.Args`方法中校验

4. 编写命令响应代码

   cmd的Run或RunE属性都是方法，它们是相应命令，执行操作，这里是代码的主体

   RunE会把出现的错误返回，Run不会

5. 执行

   到这里就非常简单，直接调用cmd.Execute()来触发命令的逻辑，也就是上面的Run或RunE。框架会负责把用户输入转化到Arg和Flag中，校验，然后交给Run方法

**Cobra在kubernetes中的使用**

主函数：

```go
func main() {
	command := app.NewAPIServerCommand()
	code := cli.Run(command)
	os.Exit(code)
}
```

可以看到cli的run主要就是调用了Cobra的execute函数

![image-20230223162910084](../assets/image-20230223162910084.png)

```go
func NewAPIServerCommand() *cobra.Command {
	s := options.NewServerRunOptions() //用户在flag上的设置可以最终通过s获取
	cmd := &cobra.Command{
		Use: "kube-apiserver",
		Long: `The Kubernetes API server validates and configures data
for the api objects which include pods, services, replicationcontrollers, and
others. The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,

		// stop printing usage when the command errors
		SilenceUsage: true,
		PersistentPreRunE: func(*cobra.Command, []string) error {
			// silence client-go warnings.
			// kube-apiserver loopback clients should not log self-issued warnings.
			rest.SetDefaultWarningHandler(rest.NoWarnings{})
			return nil
		},
		RunE: func(cmd *cobra.Command, args []string) error {
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()

			// Activate logging as soon as possible, after that
			// show flags with the final logging configuration.
			if err := s.Logs.ValidateAndApply(utilfeature.DefaultFeatureGate); err != nil {
				return err
			}
			cliflag.PrintFlags(fs)

			// set default options
			completedOptions, err := Complete(s) // 填默认值，给flag填充默认值（用户不可能全部输入）
			if err != nil {
				return err
			}

			// validate options 验证option，看看有没有冲突
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}
			// SetupSignalHandler返回管道，
			return Run(completedOptions, genericapiserver.SetupSignalHandler())
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}
	// 将option的结构体与cmd的flags进行联合，使得在cmd的输入的flags可以起作用
	fs := cmd.Flags()
	namedFlagSets := s.Flags()
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name(), logs.SkipLoggingConfigurationFlags())
	options.AddCustomGlobalFlags(namedFlagSets.FlagSet("generic"))
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
	cliflag.SetUsageAndHelpFunc(cmd, namedFlagSets, cols)

	return cmd
}
```

命令行参数->配置参数->Server

![image-20230223190156701](../assets/image-20230223190156701.png)

#### Server Chain

构建过程从右到左；请求流向从左到右

> Aggregation Server - Master - Extension Server - Not Found Handler

Aggregation Server

负责转发请求到Master或Custom API Server

Master

Kube API Server，负责Build-in的API Object相关的处理

Extension Server

Customer Resource的处理由它完成。包括CR和CRD

Not Found Handler

找不到对应的API Object的时候，返回404

> 在command的cobra里的，在RunE中的run方法

```go
// Run runs the specified APIServer.  This should never exit.
func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
	// To help debugging, immediately log version
	klog.Infof("Version: %+v", version.Get())

	klog.InfoS("Golang settings", "GOGC", os.Getenv("GOGC"), "GOMAXPROCS", os.Getenv("GOMAXPROCS"), "GOTRACEBACK", os.Getenv("GOTRACEBACK"))
	
    // 开始构建Server Chain
	server, err := CreateServerChain(completeOptions, stopCh)
	if err != nil {
		return err
	}

	prepared, err := server.PrepareRun()
	if err != nil {
		return err
	}

	return prepared.Run(stopCh)
}
```

```go
// CreateServerChain creates the apiservers connected via delegation.
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
	// 这个是基于completedOptions做出来的
    kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions)
	if err != nil {
		return nil, err
	}

	// If additional API servers are added, they should be gated.
    // 通过这个方法来做自己的Config
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(kubeAPIServerConfig.ExtraConfig.ProxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig, kubeAPIServerConfig.GenericConfig.TracerProvider))
	if err != nil {
		return nil, err
	}

    // 形成链条的最后一环
	notFoundHandler := notfoundhandler.New(kubeAPIServerConfig.GenericConfig.Serializer, genericapifilters.NoMuxAndDiscoveryIncompleteKey)
	// 创建ExtensionServer
    apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
	if err != nil {
		return nil, err
	}
	
    // 创建MasterServer
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
    // GenericAPIServer
	if err != nil {
		return nil, err
	}
	
    // 最后一环是aggregationServer
	// aggregator comes last in the chain
	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, kubeAPIServerConfig.ExtraConfig.ProxyTransport, pluginInitializer)
	if err != nil {
		return nil, err
	}
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
	if err != nil {
		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
		return nil, err
	}
	
    // 返回链条的头 aggregationServer
	return aggregatorServer, nil
}
```

```go
// CreateKubeAPIServer creates and wires a workable kube-apiserver
func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	return kubeAPIServer, nil
}
```

![image-20230227154642054](../assets/image-20230227154642054.png)

#### 在Master中装载“API”

![image-20230223192322615](../assets/image-20230223192322615.png)

```markdown
* completedConfig.New()方法
	* 1、去找Master所支持的build-in的api group，这些api group会包含不同的api Object
	* 2、每个Api gourp会形成StorageProvider
		* 提供处理与etcd交互，还负责响应Restful的请求
	* 3、InstallLegacyAPIs()的方法
		* 把一些“元老”装进来
	* 4、InstallAPIs()
		* 针对每个Api Group，把每个group里面所有的api objects都装进来
```

首先是CreateKubeAPISever()方法

```go
func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	return kubeAPIServer, nil
}
// 可以看到它主要就针对Config调用Complete（）的方法，在返回的结果上调用New方法
// Complete完了以后得到了completedConfig，然后调用它的New方法
```

这个方法new出来的是什么呢？

![image-20230223194356661](../assets/image-20230223194356661.png)

这个Instance就是MasterAPI的一个代名词

---

这里就是completedConfig的New方法了，主要处理“元老”API Object和新的一些API Object们

```go
func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
	if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
	}

	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
	if err != nil {
		return nil, err
	}

	if c.ExtraConfig.EnableLogsSupport {
		routes.Logs{}.Install(s.Handler.GoRestfulContainer)
	}

	// Metadata and keys are expected to only change across restarts at present,
	// so we just marshal immediately and serve the cached JSON bytes.
	md, err := serviceaccount.NewOpenIDMetadata(
		c.ExtraConfig.ServiceAccountIssuerURL,
		c.ExtraConfig.ServiceAccountJWKSURI,
		c.GenericConfig.ExternalAddress,
		c.ExtraConfig.ServiceAccountPublicKeys,
	)
	if err != nil {
		// If there was an error, skip installing the endpoints and log the
		// error, but continue on. We don't return the error because the
		// metadata responses require additional, backwards incompatible
		// validation of command-line options.
		msg := fmt.Sprintf("Could not construct pre-rendered responses for"+
			" ServiceAccountIssuerDiscovery endpoints. Endpoints will not be"+
			" enabled. Error: %v", err)
		if c.ExtraConfig.ServiceAccountIssuerURL != "" {
			// The user likely expects this feature to be enabled if issuer URL is
			// set and the feature gate is enabled. In the future, if there is no
			// longer a feature gate and issuer URL is not set, the user may not
			// expect this feature to be enabled. We log the former case as an Error
			// and the latter case as an Info.
			klog.Error(msg)
		} else {
			klog.Info(msg)
		}
	} else {
		routes.NewOpenIDMetadataServer(md.ConfigJSON, md.PublicKeysetJSON).
			Install(s.Handler.GoRestfulContainer)
	}

	m := &Instance{
		GenericAPIServer:          s,
		ClusterAuthenticationInfo: c.ExtraConfig.ClusterAuthenticationInfo,
	}

	// install legacy rest storage
	// 这里就是将一些“元老”的API Object进行注册安装
	if err := m.InstallLegacyAPI(&c, c.GenericConfig.RESTOptionsGetter); err != nil {
		return nil, err
	}

	// The order here is preserved in discovery.
	// If resources with identical names exist in more than one of these groups (e.g. "deployments.apps"" and "deployments.extensions"),
	// the order of this list determines which group an unqualified resource name (e.g. "deployments") should prefer.
	// This priority order is used for local discovery, but it ends up aggregated in `k8s.io/kubernetes/cmd/kube-apiserver/app/aggregator.go
	// with specific priorities.
	// TODO: describe the priority all the way down in the RESTStorageProviders and plumb it back through the various discovery
	// handlers that we have.
	restStorageProviders := []RESTStorageProvider{
		apiserverinternalrest.StorageProvider{},
		authenticationrest.RESTStorageProvider{Authenticator: c.GenericConfig.Authentication.Authenticator, APIAudiences: c.GenericConfig.Authentication.APIAudiences},
		authorizationrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer, RuleResolver: c.GenericConfig.RuleResolver},
		autoscalingrest.RESTStorageProvider{},
		batchrest.RESTStorageProvider{},
		certificatesrest.RESTStorageProvider{},
		coordinationrest.RESTStorageProvider{},
		discoveryrest.StorageProvider{},
		networkingrest.RESTStorageProvider{},
		noderest.RESTStorageProvider{},
		policyrest.RESTStorageProvider{},
		rbacrest.RESTStorageProvider{Authorizer: c.GenericConfig.Authorization.Authorizer},
		schedulingrest.RESTStorageProvider{},
		storagerest.RESTStorageProvider{},
		flowcontrolrest.RESTStorageProvider{InformerFactory: c.GenericConfig.SharedInformerFactory},
		// keep apps after extensions so legacy clients resolve the extensions versions of shared resource names.
		// See https://github.com/kubernetes/kubernetes/issues/42392
		appsrest.StorageProvider{},
		admissionregistrationrest.RESTStorageProvider{},
		eventsrest.RESTStorageProvider{TTL: c.ExtraConfig.EventTTL},
	}
    // 这个就是安装新的一些API Object
	if err := m.InstallAPIs(c.ExtraConfig.APIResourceConfigSource, c.GenericConfig.RESTOptionsGetter, restStorageProviders...); err != nil {
		return nil, err
	}

	m.GenericAPIServer.AddPostStartHookOrDie("start-cluster-authentication-info-controller", func(hookContext genericapiserver.PostStartHookContext) error {
		kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
		if err != nil {
			return err
		}
		controller := clusterauthenticationtrust.NewClusterAuthenticationTrustController(m.ClusterAuthenticationInfo, kubeClient)

		// generate a context  from stopCh. This is to avoid modifying files which are relying on apiserver
		// TODO: See if we can pass ctx to the current method
		ctx, cancel := context.WithCancel(context.Background())
		go func() {
			select {
			case <-hookContext.StopCh:
				cancel() // stopCh closed, so cancel our context
			case <-ctx.Done():
			}
		}()

		// prime values and start listeners
		if m.ClusterAuthenticationInfo.ClientCA != nil {
			m.ClusterAuthenticationInfo.ClientCA.AddListener(controller)
			if controller, ok := m.ClusterAuthenticationInfo.ClientCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(ctx); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(ctx, 1)
			}
		}
		if m.ClusterAuthenticationInfo.RequestHeaderCA != nil {
			m.ClusterAuthenticationInfo.RequestHeaderCA.AddListener(controller)
			if controller, ok := m.ClusterAuthenticationInfo.RequestHeaderCA.(dynamiccertificates.ControllerRunner); ok {
				// runonce to be sure that we have a value.
				if err := controller.RunOnce(ctx); err != nil {
					runtime.HandleError(err)
				}
				go controller.Run(ctx, 1)
			}
		}

		go controller.Run(ctx, 1)
		return nil
	})

	if utilfeature.DefaultFeatureGate.Enabled(apiserverfeatures.APIServerIdentity) {
		m.GenericAPIServer.AddPostStartHookOrDie("start-kube-apiserver-identity-lease-controller", func(hookContext genericapiserver.PostStartHookContext) error {
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			controller := lease.NewController(
				clock.RealClock{},
				kubeClient,
				m.GenericAPIServer.APIServerID,
				int32(c.ExtraConfig.IdentityLeaseDurationSeconds),
				nil,
				time.Duration(c.ExtraConfig.IdentityLeaseRenewIntervalSeconds)*time.Second,
				metav1.NamespaceSystem,
				labelAPIServerHeartbeat)
			go controller.Run(wait.NeverStop)
			return nil
		})
		m.GenericAPIServer.AddPostStartHookOrDie("start-kube-apiserver-identity-lease-garbage-collector", func(hookContext genericapiserver.PostStartHookContext) error {
			kubeClient, err := kubernetes.NewForConfig(hookContext.LoopbackClientConfig)
			if err != nil {
				return err
			}
			go apiserverleasegc.NewAPIServerLeaseGC(
				kubeClient,
				time.Duration(c.ExtraConfig.IdentityLeaseDurationSeconds)*time.Second,
				metav1.NamespaceSystem,
				KubeAPIServerIdentityLeaseLabelSelector,
			).Run(wait.NeverStop)
			return nil
		})
	}

	return m, nil
}
```

---

New方法中对LegacyAPI的创建如下

```go
// InstallLegacyAPI will install the legacy APIs for the restStorageProviders if they are enabled.
func (m *Instance) InstallLegacyAPI(c *completedConfig, restOptionsGetter generic.RESTOptionsGetter) error {
	legacyRESTStorageProvider := corerest.LegacyRESTStorageProvider{
		StorageFactory:              c.ExtraConfig.StorageFactory,
		ProxyTransport:              c.ExtraConfig.ProxyTransport,
		KubeletClientConfig:         c.ExtraConfig.KubeletClientConfig,
		EventTTL:                    c.ExtraConfig.EventTTL,
		ServiceIPRange:              c.ExtraConfig.ServiceIPRange,
		SecondaryServiceIPRange:     c.ExtraConfig.SecondaryServiceIPRange,
		ServiceNodePortRange:        c.ExtraConfig.ServiceNodePortRange,
		LoopbackClientConfig:        c.GenericConfig.LoopbackClientConfig,
		ServiceAccountIssuer:        c.ExtraConfig.ServiceAccountIssuer,
		ExtendExpiration:            c.ExtraConfig.ExtendExpiration,
		ServiceAccountMaxExpiration: c.ExtraConfig.ServiceAccountMaxExpiration,
		APIAudiences:                c.GenericConfig.Authentication.APIAudiences,
	}
    // 这里面比较重要的就是这个apiGroupInfo了，Storage负责与etcd交互的同时也负责处理restful请求
	legacyRESTStorage, apiGroupInfo, err := legacyRESTStorageProvider.NewLegacyRESTStorage(c.ExtraConfig.APIResourceConfigSource, restOptionsGetter)
	if err != nil {
		return fmt.Errorf("error building core storage: %v", err)
	}
	if len(apiGroupInfo.VersionedResourcesStorageMap) == 0 { // if all core storage is disabled, return.
		return nil
	}

	controllerName := "bootstrap-controller"
	coreClient := corev1client.NewForConfigOrDie(c.GenericConfig.LoopbackClientConfig)
	bootstrapController, err := c.NewBootstrapController(legacyRESTStorage, coreClient, coreClient, coreClient, coreClient.RESTClient())
	if err != nil {
		return fmt.Errorf("error creating bootstrap controller: %v", err)
	}
	m.GenericAPIServer.AddPostStartHookOrDie(controllerName, bootstrapController.PostStartHook)
	m.GenericAPIServer.AddPreShutdownHookOrDie(controllerName, bootstrapController.PreShutdownHook)

	if err := m.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}
```

---

New方法中对其他API的创建

```go
// InstallAPIs will install the APIs for the restStorageProviders if they are enabled.
func (m *Instance) InstallAPIs(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter, restStorageProviders ...RESTStorageProvider) error {
	apiGroupsInfo := []*genericapiserver.APIGroupInfo{}

	// used later in the loop to filter the served resource by those that have expired.
	resourceExpirationEvaluator, err := genericapiserver.NewResourceExpirationEvaluator(*m.GenericAPIServer.Version)
	if err != nil {
		return err
	}

	for _, restStorageBuilder := range restStorageProviders {
		groupName := restStorageBuilder.GroupName()
		apiGroupInfo, err := restStorageBuilder.NewRESTStorage(apiResourceConfigSource, restOptionsGetter)
		if err != nil {
			return fmt.Errorf("problem initializing API group %q : %v", groupName, err)
		}
		if len(apiGroupInfo.VersionedResourcesStorageMap) == 0 {
			// If we have no storage for any resource configured, this API group is effectively disabled.
			// This can happen when an entire API group, version, or development-stage (alpha, beta, GA) is disabled.
			klog.Infof("API group %q is not enabled, skipping.", groupName)
			continue
		}

		// Remove resources that serving kinds that are removed.
		// We do this here so that we don't accidentally serve versions without resources or openapi information that for kinds we don't serve.
		// This is a spot above the construction of individual storage handlers so that no sig accidentally forgets to check.
		resourceExpirationEvaluator.RemoveDeletedKinds(groupName, apiGroupInfo.Scheme, apiGroupInfo.VersionedResourcesStorageMap)
		if len(apiGroupInfo.VersionedResourcesStorageMap) == 0 {
			klog.V(1).Infof("Removing API group %v because it is time to stop serving it because it has no versions per APILifecycle.", groupName)
			continue
		}

		klog.V(1).Infof("Enabling API group %q.", groupName)

		if postHookProvider, ok := restStorageBuilder.(genericapiserver.PostStartHookProvider); ok {
			name, hook, err := postHookProvider.PostStartHook()
			if err != nil {
				klog.Fatalf("Error building PostStartHook: %v", err)
			}
			m.GenericAPIServer.AddPostStartHookOrDie(name, hook)
		}
		// 可以看到，将所有的GroupInfo放一起了
		apiGroupsInfo = append(apiGroupsInfo, &apiGroupInfo)
	}
	// 统一进行安装api group
	if err := m.GenericAPIServer.InstallAPIGroups(apiGroupsInfo...); err != nil {
		return fmt.Errorf("error in registering group versions: %v", err)
	}
	return nil
}
```

#### 构造并填充Scheme

![image-20230223195940390](../assets/image-20230223195940390.png)

```markdown
* Scheme相当于Windows的注册表里面会存储当前API Server所知道的所有API Group
```

什么时候填充的呢？

```markdown
** 在server.go里面会导入很多包，然后其中有一个叫Controlplane的包，它其中有一个叫import_known_version
它也会导入很多包，然后有一个叫install，里面会install所有api group里面内容。
```

在Server.go中的导入里找到这个controlplane

![image-20230223200454567](../assets/image-20230223200454567.png)

![image-20230223200631836](../assets/image-20230223200631836.png)

点进去可以找到这个import_known_version.go的内容

![image-20230223200658571](../assets/image-20230223200658571.png)

这个文件本身没有什么东西，它只负责导入，每个导入都是install的方法。都是API Group的install。其中core就是LegacyAPIGroup的内容。我们打开apps/install的这个包

```go
/*
Copyright 2016 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

// Package install installs the apps API group, making it available as
// an option to all of the API encoding/decoding machinery.
package install

import (
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/kubernetes/pkg/api/legacyscheme"
	"k8s.io/kubernetes/pkg/apis/apps"
	"k8s.io/kubernetes/pkg/apis/apps/v1"
	"k8s.io/kubernetes/pkg/apis/apps/v1beta1"
	"k8s.io/kubernetes/pkg/apis/apps/v1beta2"
)
// 这个包在被导入的时候，init方法马上会执行
func init() {
	Install(legacyscheme.Scheme)
}

// 这个AddToScheme将这个包所有的API Object都注册进Scheme
// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
	utilruntime.Must(apps.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1beta2.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta2.SchemeGroupVersion, v1beta1.SchemeGroupVersion))
}
```

### Scheme

**Version**

每个API Group都会有多个version，每一个version会包含多个kind（一个kind会出现在多个version下）这些version又称为External Version，它们面向API Server外部；Internal Version是API Server在内部程序中处理数据时API Object的实际类型

**GVK**

Group，Version，Kind。这三元组唯一确定了一个Kind。程序中，GVK可以理解为一个字符串，三者拼接的结果（程序中是一个含三字段的结构体）

**Type**

代码中常见“Type”，例如gvk2Type字段。这里的Type是一个API Object结构体类型（元数据），是程序中处理一个Object时使用的数据结构

Scheme是一个结构体，内含处理外部Version之间转换，GVK和Go Type之间转换用的数据和方法。

> GVK与Go Type之间转换

Scheme内部有两张map，分别对应gvk到type和type到gvk；使得二者可以互相找到

> API Object的默认值

API Object有诸多属性，使用者在操作一个object时，不太可能给出所有属性；另外在object从一个Version转换到另一个Version时也可能需要为不存在对应关系的字段填值

> 内外部Version之间Convert

所有外部Version都会被转化为内部Version，转换函数是记录在scheme之内的

![image-20230223214835373](../assets/image-20230223214835373.png)

![image-20230223215634456](../assets/image-20230223215634456.png)



#### 设计模式Builder及其在Kubernetes API Server中的应用

##### Builder设计模式

![image-20230226195323376](../assets/image-20230226195323376.png)

```markdown
** Director: 负责构建，组装的逻辑，调换不同的Director，可以用相同的部件组件出产品
Builder: 建造者的抽象
ConcreteBuilder: 它是部件构造过程的封装，只要换个Builder实现，同样的Director就会生产出改良的产品 
Product: 建造产物
```

##### Build - Kubernetes API Server的实现

![image-20230226201737364](../assets/image-20230226201737364.png)

```markdown
** 
Scheme是最终的产品，中间是API Group Register，实际上是一个go的文件，每个API Group都会有这样一个文件，这个文件起到了Director的作用。同时有个SchemeBuilder，起到的是Builder的角色，Installer的作用是触发Scheme的构建，会生成一个空实例，通过触发来把这个空实例完全填满，承载当前Api Server所有的Group以及Group里面的Object。
一定会有的是addKnownTypes，他是一个func，他的作用是把我当前API Group里面所知道的API Object都加入到某一个Scheme当中，所以这个函数的输入参数是一个Scheme实例，他通过调用Scheme的方法AddKnownTypes加入到Scheme中。看情况，还有addConvertFuncs和addDefaultFuncs去填充默认值，addDefaultFuncs去把空的属性填充默认值，addConvertFuncs将不同版本进行转换，将转换函数存入。这三个func会被放入SchemeBuilder中，在里面是一个切片，形成切片的一条条记录，在API Group Register中会引用SchemeBuilder里面的AddToScheme方法，这两个方法是等同的。以后就可以在API Group Register中直接使用这个方法了。AddToScheme这个方法的作用是把切片中的func方法都拿过来，逐个去调用。调用完的结果是将内容都放到Scheme当中去。
整个过程是这样的，Installer会在API Server启动过程中触发出来，install方法会被调用，install方法会拿到一个scheme的实例，把这个实例传给AddToScheme方法，这个就会逐个加入func的调用，每个func的调用都是以Scheme为输入参数的，这三个函数(addConvertFuncs、addDefaultFuncs、addKnownTypes)会被最终调用，会得到Scheme，会把Group里面所有的Type注册到Scheme中。
```

`Register`这个`Directior`

```go
/*
Copyright 2016 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package apps

import (
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/kubernetes/pkg/apis/autoscaling"
)

var (
	// SchemeBuilder stores functions to add things to a scheme.
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	// AddToScheme applies all stored functions t oa scheme.
    // 会把AddToScheme函数拿出来 作为自己的导出变量，便于其他地方来使用AddToScheme
	AddToScheme = SchemeBuilder.AddToScheme
)

// GroupName is the group name use in this package
const GroupName = "apps"

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: runtime.APIVersionInternal}

// Kind takes an unqualified kind and returns a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

//会把当前这个api Group知道的所有api Object都传入这个scheme中
// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	// TODO this will get cleaned up with the scheme types are fixed
	scheme.AddKnownTypes(SchemeGroupVersion,
		&DaemonSet{},
		&DaemonSetList{},
		&Deployment{},
		&DeploymentList{},
		&DeploymentRollback{},
		&autoscaling.Scale{},
		&StatefulSet{},
		&StatefulSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
		&ReplicaSet{},
		&ReplicaSetList{},
	)
	return nil
}

```

其中的`AddToScheme`方法

```go
func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
    // 循环会调用SchemeBuilder里面的所有元素
    // SchemeBuilder实际上是所有func的切片
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}
```

最后来看下`install`

```go
/*
Copyright 2016 The Kubernetes Authors.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

// Package install installs the apps API group, making it available as
// an option to all of the API encoding/decoding machinery.
package install

import (
	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/kubernetes/pkg/api/legacyscheme"
	"k8s.io/kubernetes/pkg/apis/apps"
	"k8s.io/kubernetes/pkg/apis/apps/v1"
	"k8s.io/kubernetes/pkg/apis/apps/v1beta1"
	"k8s.io/kubernetes/pkg/apis/apps/v1beta2"
)

func init() {
	Install(legacyscheme.Scheme)
}

// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
    // 这里的apps、v1beta1等等这些其实就是使用了不同的Director
    // 以scheme为参数调用AddToScheme，可以将之前的方法都调用一下
	utilruntime.Must(apps.AddToScheme(scheme))
	utilruntime.Must(v1beta1.AddToScheme(scheme))
	utilruntime.Must(v1beta2.AddToScheme(scheme))
	utilruntime.Must(v1.AddToScheme(scheme))
	utilruntime.Must(scheme.SetVersionPriority(v1.SchemeGroupVersion, v1beta2.SchemeGroupVersion, v1beta1.SchemeGroupVersion))
}

```

`SchemeBuilder`其实就是函数的切片

```go
type SchemeBuilder []func(*Scheme) error

// AddToScheme applies all the stored functions to the scheme. A non-nil error
// indicates that one function failed and the attempt was abandoned.
func (sb *SchemeBuilder) AddToScheme(s *Scheme) error {
	for _, f := range *sb {
		if err := f(s); err != nil {
			return err
		}
	}
	return nil
}

// Register adds a scheme setup function to the list.
func (sb *SchemeBuilder) Register(funcs ...func(*Scheme) error) {
	for _, f := range funcs {
		*sb = append(*sb, f)
	}
}

// NewSchemeBuilder calls Register for you.
func NewSchemeBuilder(funcs ...func(*Scheme) error) SchemeBuilder {
	var sb SchemeBuilder
	sb.Register(funcs...)
	return sb
}
```

#### External Version的注册

![image-20230227124430401](../assets/image-20230227124430401.png)

```markdown
** 在pkg/apis/apps里面的install.go和register.go
在staging/src/k8s.io/api/apps/v1/register.go和types.go
在右边这个文件夹里，有相应的SchemeBuilder、AddToScheme方法等等。
在v1/zz_generated.conversion.go中有一些自动生成的conversion相关的func方法的内容注册进来
```

```go
package v1

import (
	appsv1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// GroupName is the group name use in this package
const GroupName = "apps"

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version: "v1"}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

//在这里将二者做了一个关联
var (
	localSchemeBuilder = &appsv1.SchemeBuilder
	AddToScheme        = localSchemeBuilder.AddToScheme
)

// 在init方法中，register一下addDefaultingFuncs
// 在同目录的default.go里面做了一些手写默认值的方法
func init() {
	// We only register manually written functions here. The registration of the
	// generated functions takes place in the generated files. The separation
	// makes the code compile even when the generated files are missing.
	localSchemeBuilder.Register(addDefaultingFuncs)
}
// 两件事情：1、将localSchemeBuilder和staging文件夹里做了一个关联
//         2、注册一下一些addDefaultingFuncs方法
```

**可以看到，一些default和convert方法的注册是在pkg/v1/register.go里面完成注册的，而对于一些类型的注册的，是在右边staging中的库中完成注册的。合起来一起完成任务的。**

#### Converter和Defaulter的代码生成

> 代码生成：1、有Internal和External之间的数据转换 2、go语言没有继承机制，代码冗余必然会发生

```markdown
** 
每个新出的version都要与Internal Version进行转换，如果没有Generate的过程每出一个版本就要手写。
go没有类的继承，所以没法像java一样整出一个父类来继承。
```

![image-20230227130156276](../assets/image-20230227130156276.png)

1. 在doc.go里面添加注释

   ```go
   /*
   Copyright 2016 The Kubernetes Authors.
   
   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at
   
       http://www.apache.org/licenses/LICENSE-2.0
   
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
   */
   
   在package v1上面有很多注释
   在生成convert的时候要将这两个package类型进行转换
   default也是一样的
   但是有一个input值的概念
   这样就会生成zz_generated.conversion.go和zz_generated.defaults.go
   // +k8s:conversion-gen=k8s.io/kubernetes/pkg/apis/apps
   // +k8s:conversion-gen-external-types=k8s.io/api/apps/v1
   // +k8s:defaulter-gen=TypeMeta
   // +k8s:defaulter-gen-input=k8s.io/api/apps/v1
   
   package v1 // import "k8s.io/kubernetes/pkg/apis/apps/v1"
   ```

2. 这样就会生成zz_generated.conversion.go和zz_generated.defaults.go

   ![image-20230227131430626](../assets/image-20230227131430626.png)

3. 手写补充一些转换的代码（比如这里放在了defaults.go中）

![image-20230227131503273](../assets/image-20230227131503273.png)

### Generic Server

#### Go http 和 Go-restful

##### 用go http包做HTTP服务

- http.server
- Mux:请求的路由，把url映射到handler
- Handler func（函数签名有要求）

![image-20230227140857821](../assets/image-20230227140857821.png)

```markdown
** 
1、mux将路由与方法绑定
2、server实例处理端口号
3、handler就是传入mux
```

##### 用go-restful库做HTTP服务

- http.Server
- Container:相当于WS聚合地；内含一个mux，所以实现了http.Handler；http.Server生成时作为Mux传入
- Route：url、http method和handler的三元组
- WebService：一组route构成一个WS，一个WS内的route具有相同的rootPath（可以理解为url前半部分一样）

![image-20230227142811000](../assets/image-20230227142811000.png)

#### Generic Server的定位

> 暴露http服务所需的基础设施

各个内部Server做的事情是一致的：对外提供restful服务来操作API Object。所以大框架上大家是一致的：需要去实现Http Restful服务，大家都需要http server，那么这可以集中提供：

各个内部Server会互相连接，形成处理链，这同样需要有实体来负责。

> 统一各种处理机制

对于同一个事项，不同的内部Server应该采取同样的方式，这在开源项目中是比较难以保证的。例如API Resource的对外暴露形式；登录鉴权机制

> 避免重复

大量的实现都是可以被复用的

每个内部Server都是构建在Generic Server之上，把自己的“内容”填入Generic Server。

每个Generic Server最重要的输出是一个叫Director的东西，它本质上是一个`mux`和一个`go container`的组合，所有的`http request`最终都是被这些`Director`处理的。

![image-20230227135416725](../assets/image-20230227135416725.png)

```markdown
** 
Aggregation可以分辨出来属于Custom还是内部的，然后进行流转转发。
Aggregation是聚合的作用，直接面对客户端的，Master处理Kubernetes内建的Object，Extension Server处理Customer Resource，最后一个Not Found Handler是返回404.
这四个是一个Web Appliaction，是四个模块。
Custom Server是用户写的，Aggregation是通过Proxy代理来进行路由的。
```

#### Server的装配 - 基础设施

> 装配入口：kubernetes\staging\src\k8s.io\apiserver\pkg\server\config.go -> New(...)

![image-20230227143335206](../assets/image-20230227143335206.png)



```go
// c就是config,大部分属性是从CompletedConfig里面直接copy过来的
s := &GenericAPIServer{
		discoveryAddresses:         c.DiscoveryAddresses,
		LoopbackClientConfig:       c.LoopbackClientConfig,
		legacyAPIGroupPrefixes:     c.LegacyAPIGroupPrefixes,
		admissionControl:           c.AdmissionControl,
		Serializer:                 c.Serializer,
		AuditBackend:               c.AuditBackend,
		Authorizer:                 c.Authorization.Authorizer,
		delegationTarget:           delegationTarget,
		EquivalentResourceRegistry: c.EquivalentResourceRegistry,
		HandlerChainWaitGroup:      c.HandlerChainWaitGroup,
		Handler:                    apiServerHandler,
   //非常重要，构建了http request的路由，连接了url和响应函数；同时包含了一个request需要经过的预处理函数

		listedPathProvider: apiServerHandler,

		minRequestTimeout:     time.Duration(c.MinRequestTimeout) * time.Second,
		ShutdownTimeout:       c.RequestTimeout,
		ShutdownDelayDuration: c.ShutdownDelayDuration,
		SecureServingInfo:     c.SecureServing,
		ExternalAddress:       c.ExternalAddress,

		openAPIConfig:           c.OpenAPIConfig,
		openAPIV3Config:         c.OpenAPIV3Config,
		skipOpenAPIInstallation: c.SkipOpenAPIInstallation,
		
    	// 这些Hook集合在New方法接下来的代码中填充，包括自己定义的，和delegationTarget上的
		postStartHooks:         map[string]postStartHookEntry{},
		preShutdownHooks:       map[string]preShutdownHookEntry{},
		disabledPostStartHooks: c.DisabledPostStartHooks,

		healthzChecks:    c.HealthzChecks,
		livezChecks:      c.LivezChecks,
		readyzChecks:     c.ReadyzChecks,
		livezGracePeriod: c.LivezGracePeriod,

		DiscoveryGroupManager: discovery.NewRootAPIsHandler(c.DiscoveryAddresses, c.Serializer),

		maxRequestBodyBytes: c.MaxRequestBodyBytes,
		livezClock:          clock.RealClock{},

		lifecycleSignals:       c.lifecycleSignals,
		ShutdownSendRetryAfter: c.ShutdownSendRetryAfter,

		APIServerID:           c.APIServerID,
		StorageVersionManager: c.StorageVersionManager,

		Version: c.Version,

		muxAndDiscoveryCompleteSignals: map[string]<-chan struct{}{},
	}
```

##### APIServerHandler的构建

```go
apiServerHandler := NewAPIServerHandler(name, c.Serializer, handlerChainBuilder, delegationTarget.UnprotectedHandler())
```

```go
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
    // 上来先构建了一个Mux nonGoRestfulMux所有非go-restful的请求由它来handle
    
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
	}

	gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(s, panicReason, httpWriter)
	})
	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
		serviceErrorHandler(s, serviceErr, request, response)
	})

    // 最后形成一个director的东西，包含了两个mux，起了个名字
	director := director{
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}

    // 最后把Director再包装一下
	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
    // 这么多重复信息？可能会有用，但是感觉有很多重复的信息
}
```

其中`APIServerHandler`中有一个ServeHTTP方法

```go
// ServeHTTP makes it an http.Handler
func (a *APIServerHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	a.FullHandlerChain.ServeHTTP(w, r)
}
```

最终是由`handlerChainBuilder(director)`来调用ServeHTTP方法，利用装饰器模式把这个director又包装了一下，进到`director`前把登录鉴权解决了。

接着看一下`handlerChainBuilder`

```go
handlerChainBuilder := func(handler http.Handler) http.Handler {
		return c.BuildHandlerChainFunc(handler, c.Config)
	}
```

只是调用了一下`config`上的`BuildHandlerChainFunc`

> kubernetes\staging\src\k8s.io\apiserver\pkg\server\config.go

![image-20230227150108849](../assets/image-20230227150108849.png)

具体的实现如下：

![image-20230227150146273](../assets/image-20230227150146273.png)

这些其实就是登录鉴权的处理，利用了装饰器模式，把`request`的处理包含在了这个`request`的外层。

#### Server的装配 - Server链条的形成

![image-20230227194418897](../assets/image-20230227194418897.png)

```markdown
**
1、装配入口：kubernetes\staging\src\k8s.io\apiserver\pkg\server\config.go->New(...)
中有`apiServerHandler` := NewAPIServerHandler(name,c.Serializer,handlerChainBuilder,delegationTarget.UnprotectedHandler())
最后一个参数在NewAPIServerHandler中把下个Server的Director被设置为NotFoundHandler
2、最后返回的`&APIServerHandler`中会有`ServeHTTP`方法，请求会被`ServeHTTP`处理。
3、在`ServeHTTP`中，请求会转给FullHandlerChain进行处理
4、发现它的`FullHandlerChain`就是自己的director包装了一下，`FullHandlerChian`就是将一些检验、鉴权等等都提前做了处理。归根结底，交给了`director`的`ServerHTTP`进行处理，如果能处理就进行处理，不能处理交给`nonGoRestfulMux`进行处理，其实也就是下一个`Server`。
```

#### OpenAPI与Generic Server

> OpenAPI的作用

```markdown
** OpenAPI是由Swagger发展而来的一个规范，一种形式化描述Restful Service的“语言”，便于消费者理解和使用一个Service。通过OpenAPI规范，可以描述一个服务：
* 提供哪些restful服务
* 各服务接收的输入以及输出对象格式
* 支持的操作，如post，get等
```

> OpenAPI的地方

![image-20230228153104491](../assets/image-20230228153104491.png)

可以看到，在目录的`api/openapi-spec`下面

这份文档比较重要，可以找到所有操作的`url`的方式、参数等等

![image-20230228153145679](../assets/image-20230228153145679.png)

找到`swagger.json`文件

```markdown
** 
swagger.json文件定义了kubernetes对外提供的restful service，客户端可以依照该规定来向api server发http请求。
在这里，可以看到绝大多数当前版本内建的API Object，并且每个外部版本+API Object的组合拥有一套swagger中的一套定义。
```

![image-20230228153924902](../assets/image-20230228153924902.png)

> Definition

![image-20230228160217906](../assets/image-20230228160217906.png)

描述了每个API Object可以有怎么样的属性。

> Path

![image-20230228160508131](../assets/image-20230228160508131.png)

决定了向API Server发请求的时候需要向哪一个url发送请求。在`url`下支持的方法。

可以通过这个找到，当前系统中所有API Object对应的url。

##### 这份文档`api/openapi-spec/swagger.json`是怎么生成的

在各个API Object的`doc.go`中

比如在`kubernetes\staging\src\k8s.io\api\apps\v1\doc.go`

![image-20230228161200727](../assets/image-20230228161200727.png)

在这个注解中，就标明了这个需不需要生成`openapi`内容

接着在`kubernetes/staging/src/k8s.io/api/apps/v1/types.go`目录下定义这个API Object的所有属性

![image-20230228161836252](../assets/image-20230228161836252.png)

最后通过这两个文件生成了`kubernetes\pkg\generated\openapi\zz_generated.openapi.go`这个自动生成文件

![image-20230228162146381](../assets/image-20230228162146381.png)

详细解释一下这个`zz_generated.openapi.go`这个文件

> 文件位置

`kubernetes\pkg\generated\openapi`

![image-20230228162520711](../assets/image-20230228162520711.png)

> GetOpenAPIDefinition

![image-20230228162627709](../assets/image-20230228162627709.png)

`把一个字符串所代表的一个Object映射成这个API Object在这个swagger里面的schema`

```markdown
**
这个文件中，可以找到当前kubernetes版本中所有内建的api object的open api definition，并且每个外部版本+API Object的组会会有一个Definiton。这里的key和swagger.json中的definition id有一对一映射关系，
可以查看kubernetes/staging/src/k8s.io/apiserver/pkg/endpoints/openapi/openapi.go文件中的friendlyName函数
```

> 函数调用

![image-20230228162824781](../assets/image-20230228162824781.png)

把传入的`ref`拿过来，去返回一个schema。这是OpenAPI的definition，程序会为这个Schema生成swagger.json里面的内容。

```markdown
**
这些Generated code的作用：为每一个以字符串（就是上面那个key）标识的api object，生成其openapi definition，definition的主体是一个json schema，该schema定义了这个api object的json表示中可以有哪些属性，属性类型信息等等。
```

##### 生成OpenAPI Spec

`kubernetes/cmd/kube-apiserver/app/server.go`中的`buildGenericConfig()`方法

这个方法会在前序阶段中调用，会build那些config。

生成的`GetOpenAPIDefinitons`会被交给`GenericServer`的`OpenAPIConfig`中去

![image-20230228164239577](../assets/image-20230228164239577.png)

![image-20230228165035197](../assets/image-20230228165035197.png)

#### Server的装配 - API Resource的装载

![image-20230228195022227](../assets/image-20230228195022227.png)

##### registerResourceHandlers

输入：

- a *APIInstaller

  `vendor/k8s.io/apiserver/pkg/endpoints/groupversion.go`

  ![image-20230228195829744](../assets/image-20230228195829744.png)

- path string

- storage rest.Storage

  `vendor/k8s.io/apiserver/pkg/registry/generic/registry/store.go`

  ![image-20230228195841407](../assets/image-20230228195841407.png)

  ![image-20230228200005206](../assets/image-20230228200005206.png)

- ws *restful.WebService

1. 获取被注册object的group与version；确定是不是subresource

2. 确定其区分不区分namespace

3. 根据传入的storage对象实现的接口，确定支持的各种操作（verbs）

   ![image-20230228201708759](../assets/image-20230228201708759.png)

4. 创建ListOptions，CreateOptions，PatchOptions，UpdateOptions以及其它各种options

5. 生成apiResource，备返回（记录当前这次注册的源信息）

6. 创建actions list，每个resource的每个verb一条记录

   这个是有**namespace**的情况

   ![image-20230228202155851](../assets/image-20230228202155851.png)

   这个是没有**namespace**的情况

   ![image-20230228202541589](../assets/image-20230228202541589.png)

7. 决定放入etcd时使用的version；以及从etcd取出时可以转化为的version

   计算向etcd写入（encode）使用的version和读出（decode）时可转换为的version

   ![image-20230228202633345](../assets/image-20230228202633345.png)

8. 生成ResourceInfo，备返回

9. 根据Serializer，得出支持的MediaTypes，从而设置webservice的response中属性

10. 把以上各个环节得到的信息，放入reqScope（内部的一个属性）中

11. 视情况去计算reqScope的FieldManager属性

12. 逐个处理Actions list中的action，基于reqScope等属性，为他们生成route并注册到webservice中去

    ![image-20230228202912661](../assets/image-20230228202912661.png)

13. 更新apiResource，备返回

#### HTTP Server的启动

![image-20230228203650600](../assets/image-20230228203650600.png)

```markdown
**
注册openapi相关的controller到post start hook中。controller的作用是周期性得启动去检查某些状态是否达成，如果未达成就会做些操作让实际状态向预期状态更加近一点。
这个controller的作用是，当api object发生变化的时候所对应的open api spec可能不同，会调整open api spec。
接着触发GenericAPIServer的PrepareRun，制作openapi、server healthz、Livez、readyz等endpoints。
preparedGenericAPIServer的run方法编织出server生命周期中的状态流转（利用channel机制）；关闭开始前调用pre shutdown hooks。
NonBlockingRun触发启动http server，启动后触发post start hooks的执行。
SecureServingInfo.Serve() HTTP2.0设置，TLS设置，启动Server。
```

##### Server内生命周期状态流转preparedGenericAPIServer.Run()

![image-20230301132209771](../assets/image-20230301132209771.png)

```markdown
**
`All Mux and Discovery Complete` 触发 `(MuxAndDiscoveryComplete)`
Mux就是所有的API Object都已经向Mux里面注册了，Discovery就是那些APIS这些endpoints注册也都完成了，就会触发这个状态的达成。
`stopCh` 由外界发出，立刻触发`shutdownInitiatedCh`(ShutdownInitiated)
`stopCh`接着会触发`delayedStopCh`(AfterShutdownDelayDuration)先会睡眠一段时间，如果还满足`HandlerChainWaitGroup.Wait()`,那么就会触发`drainedCh`(InFlightRequestsDrained)。
`stopCh`也会触发`RunPreShutdownHooks()`会触发`preShutdownHooksHasStoppedCh`
```

##### NonBlockingRun

![image-20230301140751715](../assets/image-20230301140751715.png)

### Master API Server

![image-20230302112725880](../assets/image-20230302112725880.png)

#### Master Config的填写

![image-20230302112908260](../assets/image-20230302112908260.png)

```markdown
**
1、先去build一些general的config
	让generic Server build那些config
	SecureServing是有关证书的内容
	Features是提供打开关闭feature能力的
	APIEnablement是指object可以被关闭的
	EgressSelector在某些情况下需要找链路
	Traces就是系统出现问题时候用的，如果打开放到generic Config里去
	OpenAPIConfig 准备OpenAPI的spec等
	StorageFactory，响应restful请求，与etcd交互
	处理Authentication（登陆过程）（service account、openid）、audit
2、准备ClusterAuthenticationInfo
	ClientCA - 设置Client证书和签发机构；
	aggregated api server 和 aggregator 之间通信时，哪些Header中有用户信息
3、注册admissionPostStartHook
	启动一个go routine，在server结束时清理一些cache信息
4、准备OpenID验证机构的信息
	客户端向API Server发一个请求，可以带上OpenID机构提供的身份信息，API Server可以据此识别出用户。这里配置API Server所使用的OpenID机构信息
```

![image-20230302122807103](../assets/image-20230302122807103.png)

```markdown
**
* 用户请求首先到aggregator，包含了所有信息、证书
* aggregator就会做登陆操作和鉴权操作（RBAC）
* 交给目标Customer API Server，中forward请求时会应用证书，用户名字、所属group等都交给他（塞到http header上去）
* ConfigMap里面会存这些信息的默认值，将客户化信息和默认信息合起来
* 向aggregator发request进行验证是否有权限
```

#### 准备Config

![image-20230302130740966](../assets/image-20230302130740966.png)

```markdown
**
设置Service可使用的IP Range
	系统中定义的Service可以使用的IP地址段以及API Server自己的IP
DiscoveryAddress的设置
	当Client希望同API Server通信时，通过这个对象可以找到最优的API Server地址
设置NodePort Service可以port的范围
	不是哪个port都给它们用
一些杂项设置
	Endpoint调谐周期
	Endpoint记录的生存时常
	Endpoint的调谐器
	Service的修复周期
```

#### 创建Master Instance

![image-20230302131208985](../assets/image-20230302131208985.png)

### Extension API Server

专门处理Custome Resource Definition的API Server

> CRD - 可以定义其它API Object的API Object 

![image-20230302134109172](../assets/image-20230302134109172.png)

> 对Server实现的影响

GenericServer所提供的installAPIGroup方法能够装载“静态”API Object，从而使得Server能处理对它们的请求；但这个装载方法只能覆盖CRD，而Customer Resource是可能被“动态”创建出来的，所以需要其他的方式把它们装载到Server中，从而响应针对它们的请求

#### 准备Config

![image-20230302134545626](../assets/image-20230302134545626.png)

```markdown
** 
1、复制Master API Server的Config
	形成自己的generic Config。大部分都是直接复用Master的，只需清楚PostStartHooks和RESTOptionsGetter
2、用Option中信息和Extension自身信息设置Admission
	制作流程和Master中是一样的：
	*options.ServerRunOptions.Admission.ApplyTo()，不同的是参数用Extension Server相关的。结果主要作用在Generic Config
3、制作RESTOptionsGetter
	复制的时候这个信息被擦掉了，用Extension自己的ETCDOptions做一个新的，放入Generic Config
4、用Option中信息和Extension自身的信息设置APIEngagement
	制作的流程和Master中是一样的：
	*options.ServerRunOptions.APIEngagement.ApplyTo(),主要作用在Generic Config
5、制作Extension Config并返回
	生成Extension Config结构体实例，填入之前做出的Generic Config，除此之外，另一个字段“CRDRESTOptionsGetter”是基于前序etcdOptions做出来的
```

#### 创建Extension Instance

![image-20230302154438738](../assets/image-20230302154438738.png)

##### 单独看一下Custom Resource的API Handler

![image-20230302155344156](../assets/image-20230302155344156.png)

### Aggregation Server

#### APIService - Aggregator的核心API Object

![image-20230302165036376](../assets/image-20230302165036376.png)

> 在APIServiceSpec的结构体中的Service *ServiceReference
>
> 如果这个属性是空，就代表是本地master或者extension所提供的。
>
> 如果属性非空，那么就要构造http client去转发给custom apiserver

#### 准备Config

![image-20230302165247300](../assets/image-20230302165247300.png)

#### 创建Aggregation Server

![image-20230302165622531](../assets/image-20230302165622531.png)

![image-20230302171548220](../assets/image-20230302171548220.png)

```markdown
** 
* New 一个 Generic server
* 制作一个Informer
	作为后续监控API Service Object的变更之用
* 安装API Group下Object到Generic Server
	Group “apiregistration.k8s.io” 下只有“APIService”这个API Object和它的子object Status；准备好API Group Information后，调用Generic Server的InstallAPIGroup方法
* 制作 Group Discovery Request的handler
	用于响应对/apis的请求，和对以/apis/开头但没有命中其它handler的请求
	返回结果是当前所有Aggregated Server支持的API Group集合
* 制作监控 APIService的controller
	每一个APIService在ETCD中创建出来以后，都需要被Aggreator考虑进来从而能代理它所提供的服务。最后加入post start hook
	该Controller最终会调用APIAggreator.AddAPIService()和RemoveAPIService()，这两个方法里会注册、取消注册针对特定url的响应函数，这样就可以去响应针对aggregated server支持的API Object的请求了。
* 制作监控客户端证书变化的controller
	当发生证书变更时，所有APIService会被重新入队上一步建出的controller去重建一下。将本Controller加入post start hook
* 制作检查server可用性的controller
	APIService中会指定一个Custom Server的Service，我们需要检验该Service是否可用，使用这个controller。最后也加入post start hook
* 启动goroutine监控并处理Storage Version的更新
	每10分钟检查一次是否有更新需求
```

#### PrepareRun - 启动OpenAPI的Controller

> 在post start hook中去启动OpenAPI Controller

```go
	if s.openAPIConfig != nil {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapi-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIAggregationController.Run(context.StopCh)
			return nil
		})
	}

	if s.openAPIV3Config != nil && utilfeature.DefaultFeatureGate.Enabled(features.OpenAPIV3) {
		s.GenericAPIServer.AddPostStartHookOrDie("apiservice-openapiv3-controller", func(context genericapiserver.PostStartHookContext) error {
			go s.openAPIV3AggregationController.Run(context.StopCh)
			return nil
		})
	}
```

> 制作一个OpenAPI Controller，观测各个Aggregated Server所支持的API的变化，然后更新Aggregator自己的OpenAPI Spec

```go
	// delay OpenAPI setup until the delegate had a chance to setup their OpenAPI handlers
	if s.openAPIConfig != nil {
		specDownloader := openapiaggregator.NewDownloader()
		openAPIAggregator, err := openapiaggregator.BuildAndRegisterAggregator(
			&specDownloader,
			s.GenericAPIServer.NextDelegate(),
			s.GenericAPIServer.Handler.GoRestfulContainer.RegisteredWebServices(),
			s.openAPIConfig,
			s.GenericAPIServer.Handler.NonGoRestfulMux)
		if err != nil {
			return preparedAPIAggregator{}, err
		}
		s.openAPIAggregationController = openapicontroller.NewAggregationController(&specDownloader, openAPIAggregator)
	}

	if s.openAPIV3Config != nil && utilfeature.DefaultFeatureGate.Enabled(features.OpenAPIV3) {
		specDownloaderV3 := openapiv3aggregator.NewDownloader()
		openAPIV3Aggregator, err := openapiv3aggregator.BuildAndRegisterAggregator(
			specDownloaderV3,
			s.GenericAPIServer.NextDelegate(),
			s.GenericAPIServer.Handler.NonGoRestfulMux)
		if err != nil {
			return preparedAPIAggregator{}, err
		}
		s.openAPIV3AggregationController = openapiv3controller.NewAggregationController(openAPIV3Aggregator)
	}

```

#### APIAggregator.AddAPIService()方法

![image-20230302200014988](../assets/image-20230302200014988.png)

### Admission机制

![image-20230303140825687](../assets/image-20230303140825687.png)

> API Server内建的各种Admission Plugin提供这些代码，使用者只可以启用或禁用某个Admission（无法影响一个Admission内部的逻辑）
>
> WebHook是使用者可以客制化的

#### Admission Plugin的装配

![image-20230303141550316](../assets/image-20230303141550316.png)

![image-20230303142002261](../assets/image-20230303142002261.png)

![image-20230303142355777](../assets/image-20230303142355777.png)

> 以Update为例

![image-20230303142657046](../assets/image-20230303142657046.png)

![image-20230303142818830](../assets/image-20230303142818830.png)

### Http Req处理过程和Default Filters

一个Http请求会由一个HttpHandler对象来处理，该对象具有方法ServeHttp

通过装饰器模式，我们在一个handler外围不断包裹针对不同方面的处理逻辑，从而形成请求响应的全部流程

![image-20230303143417219](../assets/image-20230303143417219.png)

![image-20230303153235151](../assets/image-20230303153235151.png)

#### Default Filters

![image-20230303153734638](../assets/image-20230303153734638.png)

```markdown
**
WithAuditID 会给Audit打一个id方便后续使用
WithPanicRecovery 加一下当发生panic的一些恢复机制
WithMuxAndDiscoveryComplete 保护机制，启动过程比较久，这个时候有request到达，会触发404，会做很多相应的事情，向来的请求返回一些信息告诉他还没完全准备好
WithRequestInfo 把请求的一些信息都拿出来
WithLatencyTrackers
WithTracing 加一些trace的机制
WithHTTPLoging 加一些log
WithRetryAfter 当前API Server正在Shut Down，有请求来了，可以通过这个告诉请求者等几秒钟后再试，这样再试了就达到了其他API Server Instance
WithHSTS 安全相关的
WithCacheControl 禁HTTP上的cache Control
WithAuditAnnotations 在audit上加一些annotations
WithProbabilisticGoway 满足一些条件会把request踢掉
WithWaitGroup 所有的request都会被放在go的waitgroup中去 如果要关机，就会等这些内容处理完了再关机，等是有时限的
WithRequestDeadline request最长不能超过多久
WithTimeoutForNonLongRunningRequests
WithCORS 不同的域名是否可以共享同样的resource
WithAuthentication 登陆的过程
WithAudit 登录完了以后看登录有没有成功
WithImpersonation 扮演某个人（角色）来请求API Server
WithMaxInFlightLimit 流量控制
WithAuthorization 鉴权
```

#### 编解码 - Http Payload与Go结构实例之间的转换

![image-20230303210154332](../assets/image-20230303210154332.png)

#### Serializer的加载和使用

![image-20230303210727525](../assets/image-20230303210727525.png)

1. 做一个reqScope

   `vendor\k8s.io\apiserver\pkg\endpoints\installer.go`

   ![image-20230303210957797](../assets/image-20230303210957797.png)

2. 在制作req handler的时候，使用以上reqScope

   ![image-20230303211057914](../assets/image-20230303211057914.png)

3. 在handler内使用Serializer得到encoder/decoder进行编解码

   `vendor\k8s.io\apiserver\pkg\endpoints\handlers\create.go`的`CreateHandler`方法

   ![image-20230303211308645](../assets/image-20230303211308645.png)

#### 对Request进行响应的业务逻辑部分 - Store

每一个API Object都会有一个REST结构体，负责最终处理针对本Object的Request；而这个REST结构体大多时候通过内嵌`genericregistry.Store`结构体来直接复用其属性和方法，特别是内建API Object，所以这个Store结构体包含了大多数处理逻辑

上述REST结构体一般定义在API Object相应的`storage/storage.go`文件中；例如group“apps”的`deployment`如下图所示；该文件内还会有NewREST方法来构造并返回REST实例，包括子Object的

![image-20230303212427450](../assets/image-20230303212427450.png)

![image-20230303212441265](../assets/image-20230303212441265.png)

Store是如何装载，最终为Request Handler所用，是非常类似Serizlizer和Admit的，也是在installAPIResources方法中。

![image-20230303212736556](../assets/image-20230303212736556.png)

##### 以Create 和 Update 为例

![image-20230303213226611](../assets/image-20230303213226611.png)

![image-20230303213239821](../assets/image-20230303213239821.png)

##### Request Handler使用Store来响应Http Request

通过对Serializer和Admission的剖析，我们获知Http请求最终都是在诸如createHandler方法中构造出来的。例如Create（HTTP Post），就是在文件`vendor/k8s.io/apiserver/pkg/endpoints/handlers/create.go`中的CreateHandler；

而Update（HTTP Put或Patch），就是在`/vendor/k8s.io/apiserver/pkg/endpoints/handlers/update.go`的UpdateHandler方法中

![image-20230303213345495](../assets/image-20230303213345495.png)

### 鉴权与登录

> 用户种类和用户信息

```markdown
**
- Service Account：集群内管理，主要的目的是集群内程序与API Server连通之用
- Normal User：由集群外提供，对集群来说它更像一个抽象的概念，不负责保存和维护

** 用户都包含如下信息，登录过程获取这些信息放入Request，鉴权过程使用之：
- Name：用户名
- UID：唯一ID
- Groups：所例属的group
- Extra fields：一些额外的信息，因不同的登录验证策略而不同
```

API Server支持的种类

```markdown
* X509
* Service Account Token
* OpenID Token
* WebHook Token
* Request Header
* Static Token File
* Bootstrap Token
```

一个http请求到来时，先要经过登录过程。Server的Filter中一个就是做登录验证的工作。

一个request只要被一个策略明确“Approve”或明确“Reject”，那么登录过程就结束了。

系统默认有一个“Anonymous”策略：如果以上策略均不Approve也不拒绝，那么这个Request将被赋予Anonymous身份交由Server处理。

#### 鉴权模式

> RBAC - Role Based Access Control

目前默认的鉴权模式，通过role/clusterrole，然后用RoleBinding/ClusterRoleBinding把他们赋予user或group

> Attribute Based Access Control - ABAC

根据各种属性（user的，group的，目标resource的，甚至是环境变量）来决定一个request是否能做某件事

> Node

主要对由Kubelet过来的请求鉴权

> Webhook

API Server把鉴权的工作交给一个web服务，server向该服务发送SubjectAccessReview对象，该服务做出判断并把结果附在SAR上传回

![image-20230304132234126](../assets/image-20230304132234126.png)

#### 触发登录和鉴权 - 路径 1 Delegate机制

![image-20230304132435111](../assets/image-20230304132435111.png)

`delegationTarget.UnprotectedHandler()`是未经过Filter包裹的。

#### 路径2 Proxy机制

![image-20230304132809610](../assets/image-20230304132809610.png)

#### 登录验证器的实现和加载

![image-20230304133118801](../assets/image-20230304133118801.png)

#### 鉴权器的实现和加载

![image-20230304133334912](../assets/image-20230304133334912.png)