## CRD的初步设计和代码生成器的初步使用

[代码生成器](gtihub.com/kubernetes/code-generator)

常用的几个生成功能

deepcopy-gen： CustomResources  实现runtime.Object 接口对应的 DeepCopy 方法

client-gen: clientsets 相关代码

informer-gen： 创建 informers( watch 对应 CR变化产生的事件)

lister-gen：对 GET/List 请求提供只读的缓存层

创建目录`pkg/apis/dbconfig/v1`，里面的文件包含`doc.go`、`register.go`、`types.go`

`types.go`

```go
package v1
import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// DbConfig
type DbConfig struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec DbConfigSpec `json:"spec"`
	// +optional
	Status DbConfigStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// DbConfigs
type DbConfigList struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ListMeta `json:"metadata,omitempty"`

	Items []DbConfig `json:"items"`
}

type DbConfigSpec struct {
	Replicas  int `json:"replicas,omitempty"`
	Dsn string `json:"dsn,omitempty"`
}

type DbConfigStatus struct {
	Replicas int
}
```

`register.go`

```go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: "api.jtthink.com", Version: "v1"}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group qualified GroupResource
func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme   = SchemeBuilder.AddToScheme
)

// Adds the list of known types to Scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&DbConfig{},
		&DbConfigList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}

```

注意，现在的`code-generator`需要设置注释头

在同级目录创建`boilerplate.go.txt`

```go
/*
Copyright The Kubernetes Authors.
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
```

使用下面命令自动生成

```sh
$GOPATH/src/k8s.io/code-generator/generate-groups.sh all  github.com/shenyisyn/dbcore/pkg/client github.com/shenyisyn/dbcore/pkg/apis dbconfig:v1 --go-header-file=boilerplate.go.txt
```

## CRD的初步编写、自动生成client的使用

安装依赖库

```sh
go get k8s.io/client-go@v0.20.2
```

创建文件夹，放入`initConfig.go`

```go
package k8sconfig

import (
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"os"
)
//全局变量

const NSFile="/var/run/secrets/kubernetes.io/serviceaccount/namespace"

//POD里  体内
func K8sRestConfigInPod() *rest.Config{
	config,err:=rest.InClusterConfig()
	if err!=nil{
		log.Fatal(err)
	}
	return config
}
// 获取 config对象
func K8sRestConfig() *rest.Config{
	if os.Getenv("release")=="1"{ //自定义环境
		log.Println("run in cluster")
		return K8sRestConfigInPod()
	}
	log.Println("run outside cluster")
	config, err := clientcmd.BuildConfigFromFlags("","./resources/config" )
	if err!=nil{
	   log.Fatal(err)
	}
	config.Insecure=true
	return config
}

```

创建`crd.yaml`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: dbconfigs.api.jtthink.com
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: api.jtthink.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 是否有效
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                dsn:
                  type: string
  # 范围是属于namespace的 ,可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 复数名
    plural: dbconfigs
    # 单数名
    singular: dbconfig
    # 类型名
    kind: DbConfig
    listKind: DbConfigList
    # kind的简称，就像service的简称是svc
    shortNames:
      - dc
```

`main.go`

```go
func main(){
  k8sConfig := k8sconfig.K8sRestConfig()
  
  client,err := clientv1.NewForConfig(k8sConfig)
  if err != nil{
    log.Fatal(err)
  }
  
  dcList,_ := client.DbConfigs("default").List(contest.Backgournd,metav1.ListOption{})
  fmt.Println(dcList)
}
```

## Controller

```go
package controllers

import (
	"context"
	"fmt"
	v1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"

	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

type DbConfigController struct {
	client.Client
}

func NewDbConfigController() *DbConfigController {
	return &DbConfigController{}
}
func(r *DbConfigController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error)    {
	config:=&v1.DbConfig{}
	 err:= r.Get(ctx,req.NamespacedName,config)
	if err!=nil{
		return reconcile.Result{}, err
	}
	fmt.Println(config)
	return reconcile.Result{},err
}
func (r *DbConfigController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}
```

`initRuntime`

```go
package k8sconfig

import (
	v1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	"github.com/shenyisyn/dbcore/pkg/controllers"
	"os"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	"sigs.k8s.io/controller-runtime/pkg/manager"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"

)

func InitManager() {
	logf.SetLogger(zap.New())
	mgr, err := manager.New(K8sRestConfig(), manager.Options{
		Logger:  logf.Log.WithName("dbcore"),
	})
	if err != nil {
		mgr.GetLogger().Error(err, "unable to set up manager")
		os.Exit(1)
	}
	if err=v1.SchemeBuilder.AddToScheme(mgr.GetScheme());err!=nil{
		mgr.GetLogger().Error(err, "unable add scheme")
		os.Exit(1)
	}
	if err=builder.ControllerManagedBy(mgr).
		For(&v1.DbConfig{}).
		Complete(controllers.NewDbConfigController());err!=nil{
		mgr.GetLogger().Error(err, "unable to create manager")
		os.Exit(1)
	}
	if err=mgr.Start(signals.SetupSignalHandler());err!=nil{
		mgr.GetLogger().Error(err, "unable to start manager")
		os.Exit(1)
	}

}
```

## CRD的基本验证、默认值、支持status字段

`crd.yaml`

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: dbconfigs.api.jtthink.com
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: api.jtthink.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 是否有效
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 20
                dsn:
                  type: string
              required:
                 - replicas
                 - dsn
            status:
               type: object
               properties:
                  replicas:
                    type: string
      subresources:
          status: {}

  # 范围是属于namespace的 ,可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 复数名
    plural: dbconfigs
    # 单数名
    singular: dbconfig
    # 类型名
    kind: DbConfig
    listKind: DbConfigList
    # kind的简称，就像service的简称是svc
    shortNames:
      - dc
```

## CRD的字段打印、扩容和伸缩属性设置

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: dbconfigs.api.jtthink.com
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: api.jtthink.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 是否有效
      served: true
      storage: true
      additionalPrinterColumns:
        - name: replicas
          type: string
          jsonPath: .spec.status.replicas
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 20
                dsn:
                  type: string
              required:
                 - replicas
                 - dsn
            status:
               type: object
               properties:
                  replicas:
                    type: string
      subresources:
          status: {}
          scale:
            # specReplicasPath 定义定制资源中对应 scale.spec.replicas 的 JSON 路径
            specReplicasPath: .spec.replicas
            # statusReplicasPath 定义定制资源中对应 scale.status.replicas 的 JSON 路径
            statusReplicasPath: .status.replicas



  # 范围是属于namespace的 ,可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 复数名
    plural: dbconfigs
    # 单数名
    singular: dbconfig
    # 类型名
    kind: DbConfig
    listKind: DbConfigList
    # kind的简称，就像service的简称是svc
    shortNames:
      - dc
```

## 创建deployment的代码封装

模版

```go
package builders
const deptpl=`
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Name }}
  namespace: {{ .Namespace}}
spec:
  selector:
    matchLabels:
      app: {{ .Namespace}}_{{ .Name }}
  replicas: 1
  template:
    metadata:
      labels:
        app: {{ .Namespace}}_{{ .Name }}
        version: v1
    spec:
      containers:
        - name: {{ .Namespace}}_{{ .Name }}_container
          image: docker.io/shenyisyn/dbcore:v1
          imagePullPolicy: IfNotPresent
          ports:
             - containerPort: 8081
             - containerPort: 8090


`
```

`deploybuilder.go`

```go
package builders

import (
	"bytes"
	v1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/util/yaml"
	"text/template"
)

type DeployBuilder struct {
	Name string
	NameSpace string
	innerDeploy *v1.Deployment

}
func NewDeployBuilder(name,ns string ) (*DeployBuilder,error) {
	dep:=&v1.Deployment{}
	dep.Name,dep.Namespace=name,ns
	tpl,err:=template.New("deploy").Parse(deptpl)
	if err!=nil{ return nil ,err}
	var doc bytes.Buffer
	err=tpl.Execute(&doc,dep)
	if err!=nil{ return nil ,err}
	err=yaml.Unmarshal(doc.Bytes(),dep)
	if err!=nil{
		return nil ,err
	}
	return &DeployBuilder{Name: name,NameSpace: ns,innerDeploy:dep,
		},nil
}
func(this *DeployBuilder) Replicas(r int) *DeployBuilder{
	 *this.innerDeploy.Spec.Replicas=int32(r)
	 return this
}
func(this *DeployBuilder) Build() *v1.Deployment {
	return this.innerDeploy
}

```

## 提交yaml创建deployment初步

`DbConfigController`

```go
package controllers

import (

	"context"
	v1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	"github.com/shenyisyn/dbcore/pkg/builders"

	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

type DbConfigController struct {
		client.Client
}
func NewDbConfigController() *DbConfigController {
	return &DbConfigController{}
}
//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
func (r *DbConfigController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	config:=&v1.DbConfig{}
	err:=r.Get(ctx,req.NamespacedName,config)
	if err!=nil{
		return reconcile.Result{}, err
	}
	builder,err:=builders.NewDeployBuilder(req.Namespace,req.Name,r.Client)
	if err!=nil{
		return reconcile.Result{}, err
	}
	err=builder.Build()
	if err!=nil{
		return reconcile.Result{}, err
	}

	return reconcile.Result{}, err
}
//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
func (r *DbConfigController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}

```

`deploybuilder.go`

```go
package builders

import (
	"bytes"
	"context"
	"fmt"
	v1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/yaml"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"text/template"
)
//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
type DeployBuilder struct {
	deploy *v1.Deployment
	 client.Client
}
//构建 deploy 创建器
func NewDeployBuilder(ns,name string ,c client.Client) (*DeployBuilder,error) {
	fmt.Println("aaaa",ns,name)
	deploy:=&v1.Deployment{}
	err:=c.Get(context.Background(),types.NamespacedName{
		Name: name,Namespace: ns,
	},deploy)
	if err!=nil{
		deploy.Name,deploy.Namespace=name,ns

		tpl,err:=template.New("deploy").Parse(deptpl)
		var tplRet bytes.Buffer
		if err!=nil{return nil,err}
		err=tpl.Execute(&tplRet,deploy)
		if err!=nil{return nil,err}
		// 解析  deploy模板----仅仅是模板

		err=yaml.Unmarshal(tplRet.Bytes(),deploy)
		if err!=nil{return nil,err}
	}
	return &DeployBuilder{deploy:deploy,Client:c},nil
}
//设置 副本
func(this *DeployBuilder) Replicas( r int ) *DeployBuilder{
	  *this.deploy.Spec.Replicas=int32(r)
	  return this
}
//构建出  deployment
func(this *DeployBuilder) Build()  error{
	if this.deploy.CreationTimestamp.IsZero(){
		return this.Create(context.Background(),this.deploy)
	}
//	return this.deploy
return nil
}

```

## 提交yaml修改deployment

```go
package builders

import (
	"bytes"
	"context"
	configv1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	appv1 "k8s.io/api/apps/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/yaml"

	"sigs.k8s.io/controller-runtime/pkg/client"
	"text/template"
)
//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
type DeployBuilder struct {
	deploy *appv1.Deployment
	config   *configv1.DbConfig  //新增属性 。保存 config 对象
	client.Client

}
//目前软件的 命名规则
func deployName(name string ) string {
	return "dbcore-"+name
}

//构建 deploy 创建器
func NewDeployBuilder(config *configv1.DbConfig,client client.Client) (*DeployBuilder,error) {
	    deploy:=&appv1.Deployment{}
		err:=client.Get(context.Background(),types.NamespacedName{
			Namespace: config.Namespace,Name: deployName(config.Name),  //这里做了改动
		},deploy)
	if err!=nil{  //没取到
		deploy.Name,deploy.Namespace=config.Name,config.Namespace
		tpl,err:=template.New("deploy").Parse(deptpl)
		var tplRet bytes.Buffer
		if err!=nil{return nil,err}
		err=tpl.Execute(&tplRet,deploy)
		if err!=nil{return nil,err}
		// 解析  deploy模板----仅仅是模板


		err=yaml.Unmarshal(tplRet.Bytes(),deploy)
		if err!=nil{return nil,err}

	}
	return &DeployBuilder{deploy:deploy,Client:client,config:config },nil
}

func(this *DeployBuilder) apply() *DeployBuilder{
	// 同步副本
	*this.deploy.Spec.Replicas=int32(this.config.Spec.Replicas)
	return this
}
//构建出  deployment  ..有可能是新建， 有可能是update
func(this *DeployBuilder) Build(ctx context.Context)  error{

		if this.deploy.CreationTimestamp.IsZero(){
			this.apply() //同步  所需要的属性 如 副本数
			err:=this.Create(ctx,this.deploy)
			if err!=nil{
				return err
			}
		}else{
			patch:=client.MergeFrom(this.deploy.DeepCopy())
			this.apply() //同步  所需要的属性 如 副本数
			err:=this.Patch(ctx,this.deploy,patch)
			if err!=nil{
				return err
			}
		}
		//更新 是课后作业
	 return nil

}

```

## 级联删除

```go
package builders

import (
	"bytes"
	"context"
	configv1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	appv1 "k8s.io/api/apps/v1"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/yaml"

	"sigs.k8s.io/controller-runtime/pkg/client"
	"text/template"
)

type DeployBuilder struct {
	deploy *appv1.Deployment
	config   *configv1.DbConfig  //新增属性 。保存 config 对象
	client.Client

}
//目前软件的 命名规则
func deployName(name string ) string {
	return "dbcore-"+name
}

//构建 deploy 创建器
func NewDeployBuilder(config *configv1.DbConfig,client client.Client) (*DeployBuilder,error) {
	    deploy:=&appv1.Deployment{}
		err:=client.Get(context.Background(),types.NamespacedName{
			Namespace: config.Namespace,Name: deployName(config.Name),  //这里做了改动
		},deploy)
	if err!=nil{  //没取到
		deploy.Name,deploy.Namespace=config.Name,config.Namespace
		tpl,err:=template.New("deploy").Parse(deptpl)
		var tplRet bytes.Buffer
		if err!=nil{return nil,err}
		err=tpl.Execute(&tplRet,deploy)
		if err!=nil{return nil,err}
		// 解析  deploy模板----仅仅是模板


		err=yaml.Unmarshal(tplRet.Bytes(),deploy)
		if err!=nil{return nil,err}

	}
	return &DeployBuilder{deploy:deploy,Client:client,config:config },nil
}
//同步属性
func(this *DeployBuilder) apply() *DeployBuilder{
	// 同步副本
	*this.deploy.Spec.Replicas=int32(this.config.Spec.Replicas)
	return this
}
//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
func(this *DeployBuilder) setOwner() *DeployBuilder{
	  this.deploy.OwnerReferences=append(this.deploy.OwnerReferences,
	  	v1.OwnerReference{
	  	   APIVersion: this.config.APIVersion,
	  	   Kind:this.config.Kind,
	  	   Name: this.config.Name,
			UID:this.config.UID,
		})
	  return this
}
//构建出  deployment  ..有可能是新建， 有可能是update
func(this *DeployBuilder) Build(ctx context.Context)  error{
		if this.deploy.CreationTimestamp.IsZero(){
			this.apply().setOwner() //同步  所需要的属性 如 副本数 , 并且设置OwnerReferences

			err:=this.Create(ctx,this.deploy)
			if err!=nil{
				return err
			}
		}else{
			patch:=client.MergeFrom(this.deploy.DeepCopy())
			this.apply() //同步  所需要的属性 如 副本数
			err:=this.Patch(ctx,this.deploy,patch)
			if err!=nil{
				return err
			}
		}
		//更新 是课后作业
	 return nil

}
```

```sh
kubectl delete dc mydb1 -cascade=foreground # 前台删除
```

## 重新拉起被手工删除的资源

```go
package controllers

import (
	"context"
	v1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	"github.com/shenyisyn/dbcore/pkg/builders"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/util/workqueue"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

type DbConfigController struct {
		client.Client
}
func NewDbConfigController() *DbConfigController {
	return &DbConfigController{}
}
// 注意，这块代码要在manager中配置
func (r *DbConfigController) OnDelete(event event.DeleteEvent,
	limitingInterface workqueue.RateLimitingInterface){
		 for _,ref:=range event.Object.GetOwnerReferences(){

		 	if ref.Kind=="DbConfig" && ref.APIVersion=="api.jtthink.com/v1"{
				limitingInterface.Add(
					reconcile.Request{
						types.NamespacedName{
							Name:ref.Name,Namespace: event.Object.GetNamespace(),
					},
				})
			 }
		 }
}

func (r *DbConfigController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	config:=&v1.DbConfig{}
	err:=r.Get(ctx,req.NamespacedName,config)
	if err!=nil{
		return reconcile.Result{}, err
	}
	builder,err:=builders.NewDeployBuilder(config,r.Client)
	if err!=nil{return reconcile.Result{}, err}
	err=builder.Build(ctx) // 创建出 deploy
	if err!=nil{return reconcile.Result{}, err}
	return reconcile.Result{}, err
}

func (r *DbConfigController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}

```

## 监控deployment的副本变化，显示在控制台

修改crd，使得监控的内容是ready

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: dbconfigs.api.jtthink.com
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: api.jtthink.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 是否有效
      served: true
      storage: true
      additionalPrinterColumns:
        - name: Ready
          type: string
          jsonPath: .status.ready
        - name: Age
          type: date
          jsonPath: .metadata.creationTimestamp
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 20
                dsn:
                  type: string
              required:
                 - replicas
                 - dsn
            status:
               type: object
               properties:
                  replicas:
                    type: integer
                  ready:
                    type: string
      subresources:
          status: {}
          scale:
            # specReplicasPath 定义定制资源中对应 scale.spec.replicas 的 JSON 路径
            specReplicasPath: .spec.replicas
            # statusReplicasPath 定义定制资源中对应 scale.status.replicas 的 JSON 路径
            statusReplicasPath: .status.replicas



  # 范围是属于namespace的 ,可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 复数名
    plural: dbconfigs
    # 单数名
    singular: dbconfig
    # 类型名
    kind: DbConfig
    listKind: DbConfigList
    # kind的简称，就像service的简称是svc
    shortNames:
      - dc
```

修改模版，放一个initcontainer，这个的效果就是0/1过了15秒变成1/1

```go
package builders
const deptpl=`
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dbcore-{{ .Name }}
  namespace: {{ .Namespace}}
spec:
  selector:
    matchLabels:
      app: dbcore-{{ .Namespace}}-{{ .Name }}
  replicas: 1
  template:
    metadata:
      labels:
        app: dbcore-{{ .Namespace}}-{{ .Name }}
        version: v1
    spec:
      initContainers:
        - name: init-test
          image: busybox:1.28
          command: ['sh', '-c', 'echo sleeping && sleep 15']
      containers:
        - name: dbcore-{{ .Namespace}}-{{ .Name }}-container
          image: docker.io/shenyisyn/dbcore:v1
          imagePullPolicy: IfNotPresent
          ports:
             - containerPort: 8081
             - containerPort: 8090


`
```

修改crd的types.go

```go
package v1
import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:openapi-gen=true
// DbConfig
type DbConfig struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec DbConfigSpec `json:"spec"`
	// +optional
	Status DbConfigStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// DbConfigList
type DbConfigList struct {
	metav1.TypeMeta `json:",inline"`
	// +optional
	metav1.ListMeta `json:"metadata,omitempty"`

	Items []DbConfig `json:"items"`
}

type DbConfigSpec struct {
	Replicas  int `json:"replicas,omitempty"`
	Dsn string `json:"dsn,omitempty"`
}

type DbConfigStatus struct {
	Replicas int32 `json:"replicas,omitempty"`
	Ready string `json:"ready,omitempty"`  //新增属性。 用来显示 当前副本数情况
}
```

`deploybuilder.go`

```go
package builders

import (
	"bytes"
	"context"
	"fmt"
	configv1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	appv1 "k8s.io/api/apps/v1"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/yaml"

	"sigs.k8s.io/controller-runtime/pkg/client"
	"text/template"
)

type DeployBuilder struct {
	deploy *appv1.Deployment
	config   *configv1.DbConfig  //新增属性 。保存 config 对象
	client.Client

}
//目前软件的 命名规则
func deployName(name string ) string {
	return "dbcore-"+name
}

//构建 deploy 创建器
func NewDeployBuilder(config *configv1.DbConfig,client client.Client) (*DeployBuilder,error) {
	    deploy:=&appv1.Deployment{}
		err:=client.Get(context.Background(),types.NamespacedName{
			Namespace: config.Namespace,Name: deployName(config.Name),  //这里做了改动
		},deploy)
	if err!=nil{  //没取到
		deploy.Name,deploy.Namespace=config.Name,config.Namespace
		tpl,err:=template.New("deploy").Parse(deptpl)
		var tplRet bytes.Buffer
		if err!=nil{return nil,err}
		err=tpl.Execute(&tplRet,deploy)
		if err!=nil{return nil,err}
		// 解析  deploy模板----仅仅是模板


		err=yaml.Unmarshal(tplRet.Bytes(),deploy)
		if err!=nil{return nil,err}

	}
	return &DeployBuilder{deploy:deploy,Client:client,config:config },nil
}
//同步属性
func(this *DeployBuilder) apply() *DeployBuilder{
	// 同步副本
	*this.deploy.Spec.Replicas=int32(this.config.Spec.Replicas)
	return this
}

func(this *DeployBuilder) setOwner() *DeployBuilder{
	  this.deploy.OwnerReferences=append(this.deploy.OwnerReferences,
	  	v1.OwnerReference{
	  	   APIVersion: this.config.APIVersion,
	  	   Kind:this.config.Kind,
	  	   Name: this.config.Name,
			UID:this.config.UID,
		})
	  return this
}
//构建出  deployment  ..有可能是新建， 有可能是update
func(this *DeployBuilder) Build(ctx context.Context)  error{
		if this.deploy.CreationTimestamp.IsZero(){
			this.apply().setOwner() //同步  所需要的属性 如 副本数 , 并且设置OwnerReferences

			err:=this.Create(ctx,this.deploy)
			if err!=nil{
				return err
			}
		}else{
			patch:=client.MergeFrom(this.deploy.DeepCopy())
			this.apply() //同步  所需要的属性 如 副本数
			err:=this.Patch(ctx,this.deploy,patch)
			if err!=nil{
				return err
			}
			//查看状态
			replicas:=this.deploy.Status.ReadyReplicas //获取当前deployment的ready状态副本数
			this.config.Status.Ready=fmt.Sprintf("%d/%d",replicas,this.config.Spec.Replicas)
			this.config.Status.Replicas=replicas

			err=this.Client.Status().Update(ctx,this.config)
			if err!=nil{
				return err
			}
		}
		//更新 是课后作业
	 return nil

}


```

`initManager.go`

```go
package k8sconfig

import (
	v1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	"github.com/shenyisyn/dbcore/pkg/controllers"
	appv1 "k8s.io/api/apps/v1"
	"log"
	"os"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/source"

	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
)

//初始化 控制器管理器
func InitManager()  {
	logf.SetLogger(zap.New())
	mgr, err := manager.New(K8sRestConfig(),
		manager.Options{
		Logger: logf.Log.WithName("dbcore"),
		})
	if err != nil {
		log.Fatal("创建管理器失败:",err.Error())
	}
	err=v1.SchemeBuilder.AddToScheme(mgr.GetScheme())
	if err != nil {
		mgr.GetLogger().Error(err, "unable add schema")
		os.Exit(1)
	}
	//初始化控制器对象
	dbconfigController:=controllers.NewDbConfigController()
	if err=builder.ControllerManagedBy(mgr).
		For(&v1.DbConfig{}).
		//监听 deployment
		Watches(&source.Kind{Type:&appv1.Deployment{}},
			handler.Funcs{
			   DeleteFunc: dbconfigController.OnDelete,
			   UpdateFunc:dbconfigController.OnUpdate,
			},
	    ).
		Complete(dbconfigController);err!=nil{
		mgr.GetLogger().Error(err, "unable to create manager")
		os.Exit(1)
	}
	if err=mgr.Start(signals.SetupSignalHandler());err!=nil{
		mgr.GetLogger().Error(err, "unable to start manager")
	}
}

```

## 参数交互设计和ConfigMap创建

再写一个configmapbuilder

```go
package builders

import (
	"bytes"
	"context"
	configv1 "github.com/shenyisyn/dbcore/pkg/apis/dbconfig/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/yaml"
	"text/template"

	"sigs.k8s.io/controller-runtime/pkg/client"
)

//本课程来自 程序员在囧途(www.jtthink.com) 咨询群：98514334
type ConfigMapBuilder struct {
	cm *corev1.ConfigMap
	config   *configv1.DbConfig  //新增属性 。保存 config 对象
	client.Client

}
//构建 cm 创建器
func NewConfigMapBuilder(config *configv1.DbConfig,client client.Client) (*ConfigMapBuilder,error) {
	cm:=&corev1.ConfigMap{}
	err:=client.Get(context.Background(),types.NamespacedName{
		Namespace: config.Namespace,Name: deployName(config.Name),  //这里做了改动
	},cm)
	if err!=nil{  //没取到

		cm.Name,cm.Namespace=config.Name,config.Namespace

		tpl,err:=template.New("configmap").Parse(cmtpl)
		var tplRet bytes.Buffer
		if err!=nil{return nil,err}

		err=tpl.Execute(&tplRet,cm)
		if err!=nil{
			return nil,err}
		err=yaml.Unmarshal(tplRet.Bytes(),cm)
		if err!=nil{return nil,err}


	}
	return &ConfigMapBuilder{cm:cm,Client:client,config:config },nil
}
//同步属性
func(this *ConfigMapBuilder) setOwner() *ConfigMapBuilder{
	this.cm.OwnerReferences=append(this.cm.OwnerReferences,
		metav1.OwnerReference{
			APIVersion: this.config.APIVersion,
			Kind:this.config.Kind,
			Name: this.config.Name,
			UID:this.config.UID,
		})
	return this
}
func(this *ConfigMapBuilder) apply() *ConfigMapBuilder{
	 //下节课来写

	return this
}
//构建出  ConfigMap  ..有可能是新建， 有可能是update
func(this *ConfigMapBuilder) Build(ctx context.Context)  error{
	if this.cm.CreationTimestamp.IsZero(){
		this.apply().setOwner() //同步  所需要的属性 如 副本数 , 并且设置OwnerReferences
		err:=this.Create(ctx,this.cm)
		if err!=nil{
			return err
		}
	}else{
		patch:=client.MergeFrom(this.cm.DeepCopy())
		this.apply() //同步  所需要的属性 如 副本数
		err:=this.Patch(ctx,this.cm,patch)
		if err!=nil{
			return err
		}

	}
	//更新 是课后作业
	return nil

}
```

模版内容

```go
package builders
// configmap对应模板
const cmtpl=`
apiVersion: v1
kind: ConfigMap
metadata:
 name: dbcore-{{ .Name }}
 namespace: {{ .Namespace }}
data:
 app.yaml: |
  dbConfig:
   dsn: ""
   maxOpenConn: 20
   maxLifeTime: 1800
   maxIdleConn: 5
  appConfig:
   rpcPort: 8081
   httpPort: 8090

  apis:
   - name: test
     sql: "select * from test"
`
//  DBCORE deploy 模板
const deptpl=`
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dbcore-{{ .Name }}
  namespace: {{ .Namespace}}
spec:
  selector:
    matchLabels:
      app: dbcore-{{ .Namespace}}-{{ .Name }}
  replicas: 1
  template:
    metadata:
      labels:
        app: dbcore-{{ .Namespace}}-{{ .Name }}
        version: v1
    spec:
      initContainers:
        - name: init-test
          image: busybox:1.28
          command: ['sh', '-c', 'echo sleeping && sleep 15']
      containers:
        - name: dbcore-{{ .Namespace}}-{{ .Name }}-container
          image: docker.io/shenyisyn/dbcore:v1
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: configdata
              mountPath: /src
          ports:
             - containerPort: 8081
             - containerPort: 8090
	  volumes:
		 - name: configdata
		   configMap:
			 defaultMode: 0644
			 name: dbcore-{{ .Name }}


`

```



