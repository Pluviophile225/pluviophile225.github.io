## 简单的反代功能

第三方库

```markdown
# Go
# fasthttp
# nginx caddy openr
```

安装fasthttp

```sh
go get github.com/yeqown/fasthttp-reverse-proxy/v2
```

```go
func ProxyHandler(ctx *fasthttp.RequestCtx){
  jtthink.ServeHTTP(ctx)
  ctx.Response.Header.Add("myname","shenyi")
}
var jtthink=proxy.NewReverseProxy("www.jtthink.com")

func main(){
  
  fasthttp.ListenAndServe(":80",ProxyHandler)
}
```

## 初步集成k8s runtime库、配置文件的设计

装库

```sh
go get sigs.k8s.io/controller-runtime@v0.9.6
```

创建一个`app.yaml`的配置文件

```yaml
server:
  port: 80

ingress:
  rules:
    - http:
        paths:
          - path: /
            pathType: 
            backend:
              service:
                name: test
                port:
                  number: 80

```

接着创建一个`pkg/sysinit/config.go`的配置文件解析类来匹配这个yaml

```go
type Server struct{
  Port int
}
type SysConfigStruct struct{
  Server Server
  Ingress v1.IngressSpec
}

var SysConfig = new(SysConfigStruct)

func InitConfig(){
  config,err := ioutil.ReadFile("./app.yaml")
  if err != nil{
    log.Fatal(err)
  }
  err = yaml.Unmarshal(config,SysConfig)
  if err != nil{
    log.Fatal(err)
  }
  
}
```

## 路由解析（1）结合mux初步完成路由解析

安装库

```sh
go get -u github.com/gorilla/mux
```

创建`/sysinit/rules.go`

```go
type ProxyHandler struct{
  Proxy *proxy.ReverseProxy
}

func(this *ProxyHandler) ServeHTTP(http.ResponseWriter,*http.Request){}

var MyRouter *mux Router

func init(){
  MyRouter=mux.NewRouter()
}

func ParseRule(){
  for _,rule := range SysConfig.Ingress.Rules{
    for _, path := range rule.HTTP.paths{
      rProxy := proxy.NewReverseProxy(
        fmt.Sprintf("%s:%d",path.Backend.Service.Name,path.Backend.Service.Port.Number))
      MyRouter.NewRoute().Path(path.Path).
      Methods("GET","POST","PUT","DELETE","OPTIONS").
      Handler(&ProxyHandler{Proxy:rProxy})
    }
  }
}

func GetRoute(req fasthttp.Request)*proxy.ReverseProxy{
  match:=&mux.RouteMatch{}
  httpReq:=&http.Request{URL:&url.URL{Path:string(req.URI().Path())},Method:string(req.Header.Method)}
  if MyRouter.Match(httpReq,match){
    return match.Handler.(*ProxyHandler).Proxy
  }
  return nil
}
```

```go
func ProxyHandler(ctx *fasthttp.RequestCtx){
  if getProxy:=sysinit.GetRoute(ctx.Request);getProxy!=nil{
    getProxy.ServeHTTP(ctx)
  }else{
    ctx.Response.SetStatusCode(404)
    ctx.Response.SetBodyString("404...")
  }
  
 
}
var jtthink=proxy.NewReverseProxy("www.jtthink.com")

func main(){
  
  fasthttp.ListenAndServe(":80",ProxyHandler)
}
```

## 路由解析（2）支持前缀匹配

```go
type ProxyHandler struct{
  Proxy *proxy.ReverseProxy
}

func(this *ProxyHandler) ServeHTTP(http.ResponseWriter,*http.Request){}

var MyRouter *mux Router

func init(){
  MyRouter=mux.NewRouter()
}

func ParseRule(){
  for _,rule := range SysConfig.Ingress.Rules{
    for _, path := range rule.HTTP.paths{
      rProxy := proxy.NewReverseProxy(
        fmt.Sprintf("%s:%d",path.Backend.Service.Name,path.Backend.Service.Port.Number))
      if path.PathType != nil && *path.PathType == v1.PathTypeExact{
        MyRouter.NewRoute().Path(path.Path).
      	Methods("GET","POST","PUT","DELETE","OPTIONS").
      	Handler(&ProxyHandler{Proxy:rProxy})
      }else{
        MyRouter.NewRoute().PathPrefix(path.Path).
      	Methods("GET","POST","PUT","DELETE","OPTIONS").
      	Handler(&ProxyHandler{Proxy:rProxy})
      }
      
      
    }
  }
}

func GetRoute(req fasthttp.Request)*proxy.ReverseProxy{
  match:=&mux.RouteMatch{}
  httpReq:=&http.Request{URL:&url.URL{Path:string(req.URI().Path())},Method:string(req.Header.Method)}
  if MyRouter.Match(httpReq,match){
    return match.Handler.(*ProxyHandler).Proxy
  }
  return nil
}
```

## 路由解析（3）支持Host匹配

ingress可以多配置

```yaml
server:
  port: 80

ingress:
  - apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-myservice
      annotations:
        kubernetes.io/ingress.class: "nginx"
    spec:
      rules:
        - host: xxx.com
          http:
            paths:
              - path: /
                pathType: 
                backend:
                  service:
                    name: test
                    port:
                      number: 80
```

其实就是加了Host选项，注意这里的代码有更新。拿到了课件再做修改。

```go
package sysinit

import (
	"github.com/gorilla/mux"
	"net/http"
)
//本课程来自程序员在囧途(www.jtthink.com)咨询群：98514334
//route构建器， build 方法必须要执行
var MyRouter *mux.Router

func init() {
	MyRouter=mux.NewRouter()
}
type RouteBuilder struct {
	route *mux.Route
}
func NewRouteBuilder() *RouteBuilder {
	return &RouteBuilder{route: MyRouter.NewRoute()}
}
//本课程来自程序员在囧途(www.jtthink.com)咨询群：98514334
func(this *RouteBuilder) SetPath(path string ,exact bool ) *RouteBuilder{
	if exact{
		 this.route.Path(path)
	}else{
		 this.route.PathPrefix(path)
	}
   return this
}
//第二个参数是故意的，方便调用时 传入 条件，省的外面写 if else
func(this *RouteBuilder) SetHost(host string, set bool ) *RouteBuilder{
	if set{
		this.route.Host(host)
	}
	 return this
}
func(this *RouteBuilder) Build(handler http.Handler)  {
	 this.route.
		Methods("GET","POST","PUT","DELETE","OPTIONS").
		Handler(handler)
}
//本课程来自程序员在囧途(www.jtthink.com)咨询群：98514334
```

```go
package sysinit

import (
	"fmt"
	"github.com/gorilla/mux"
	"github.com/valyala/fasthttp"
	"github.com/yeqown/fasthttp-reverse-proxy/v2"
	v1 "k8s.io/api/networking/v1"
	"net/http"
	"net/url"
)
type ProxyHandler struct {
	Proxy *proxy.ReverseProxy  // proxy对象。 保存proxy
}
//空函数没啥用
func(this *ProxyHandler) ServeHTTP(http.ResponseWriter, *http.Request){}


//解析配置文件中的rules， 初始化 路由
func ParseRule()  {
	//现在要循环 遍历
	for _,ingress:=range SysConfig.Ingress{
		for _,rule:= range   ingress.Spec.Rules{
			for _,path:=range rule.HTTP.Paths {
				//构建 反代对象
				rProxy:=proxy.NewReverseProxy(
					fmt.Sprintf("%s:%d",path.Backend.Service.Name,path.Backend.Service.Port.Number))
				//本课程来自程序员在囧途(www.jtthink.com)咨询群：98514334
				routeBud:=NewRouteBuilder()

			    routeBud.
					SetPath(path.Path,path.PathType!=nil && *path.PathType==v1.PathTypeExact).
					SetHost(rule.Host,rule.Host!="").
					Build(&ProxyHandler{Proxy: rProxy})



			}
		}
	}

}

// 获取路由   （先匹配 请求path ，如果匹配到 ，会返回 对应的proxy 对象)
func GetRoute(req fasthttp.Request)*proxy.ReverseProxy{
	match:=&mux.RouteMatch{}
	httpReq:=&http.Request{
		URL:&url.URL{Path:  string(req.URI().Path())},
		Method:string(req.Header.Method()),
		Host: string(req.Header.Host()),
	}
	if MyRouter.Match(httpReq,match){
		return match.Handler.(*ProxyHandler).Proxy
	}
	return  nil
}
//本课程来自程序员在囧途(www.jtthink.com)咨询群：98514334
```

## 路由解析（4）支持路径重写

```yaml
server:
  port: 80

ingress:
 - apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
    name: ingress-myservicea
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$1
      kubernetes.io/ingress.class: "nginx"
   spec:
    rules:
       - host: aabb.com
         http:
          paths:
            - path: /jtthink/{(.*)}
              backend:
                service:
                  name: www.jtthink.com
                  port:
                    number: 80
            - path: /baidu/{(.*)}
              backend:
                service:
                  name: www.baidu.com
                  port:
                    number: 80
```

过滤器`commcon.go`

```go
package filters

import (
	"github.com/valyala/fasthttp"
	"reflect"
)
//所有过滤器 的接口
type ProxyFileter interface {
	SetValue(values ...string)
	Do(ctx *fasthttp.RequestCtx)
}
type ProxyFilters []ProxyFileter
func(this ProxyFilters) Do(ctx *fasthttp.RequestCtx){
	for _,filter:=range this {
		filter.Do(ctx)
	}
}
var FileterList=map[string]ProxyFileter{}

//注册过滤器
func registerFilter(key string ,filter ProxyFileter)  {
	FileterList[key]=filter
}
func init() {

}
//检查注解是否 和预设的 过滤器 匹配
func CheckAnnotations(annos map[string]string,exts ...string  ) []ProxyFileter{
	fileters:=[]ProxyFileter{}
	for anno_key,anno_value:=range annos{
		for filter_key,filterReflect:=range FileterList{
			if anno_key==filter_key{
				t:=reflect.TypeOf(filterReflect)
				if t.Kind()==reflect.Ptr{
					t=t.Elem()
				}
				filter:=reflect.New(t).Interface().(ProxyFileter)
				params:=[]string{anno_value}
				params=append(params,exts...)
				filter.SetValue(params...)
				fileters=append(fileters,filter)
			}
		}
	}
	return fileters
}
```

`rewrite.go`

```go
package filters

import (
	"github.com/valyala/fasthttp"
	"log"
	"regexp"
	"strings"
)
const RewriteAnnotation="nginx.ingress.kubernetes.io/rewrite-target"

func init() {
	registerFilter(RewriteAnnotation,(*RewriteFilter)(nil) )
}
type RewriteFilter struct {
    pathValue string
    target string
}
//可变参数。第1个是 rewrie-target:的值 如 /$1  第2个是 原始的path 值 ，如/jtthink/{(.*)}
func(this *RewriteFilter) SetValue(values ...string){
	if len(values)!=2{
		return
	}
	this.target=values[0]
	this.pathValue=values[1]
	this.pathValue=strings.Replace(this.pathValue,"{","",-1)
	this.pathValue=strings.Replace(this.pathValue,"}","",-1)
}
func(this *RewriteFilter) Do(ctx *fasthttp.RequestCtx){
	getUrl:=ctx.Request.URI().String() //获取 请求URL  譬如  /jtthink/users
	reg,err:=regexp.Compile(this.pathValue)
	if err!=nil{
		log.Println(err)
		return
	}

	getUrl=reg.ReplaceAllString(getUrl,this.target)

	ctx.Request.SetRequestURI(getUrl)
	if err!=nil{
		log.Println(err)
		return
	}

```

## 增加请求头过滤器

```go
package filters

import (
	"github.com/valyala/fasthttp"
	"strings"
)
const AddRequestHeaderAnnotation=AnnotationPrefix+"/add-request-header"

func init() {
	registerFilter(AddRequestHeaderAnnotation,(*AddRequestHeaderFilter)(nil) )
}
type AddRequestHeaderFilter struct {
    pathValue string
    target string  //注解  值
    path string
}
func(this *AddRequestHeaderFilter) SetPath(value  string){}
//可变参数。第1个是 注解值:的值 如 /$1
func(this *AddRequestHeaderFilter) SetValue(values ...string){
	this.target=values[0]
}
func(this *AddRequestHeaderFilter) Do(ctx *fasthttp.RequestCtx){
	 kvList:=strings.Split(this.target,";")
	 for _,kv:=range kvList{
	 	k_v:=strings.Split(kv,"=")
	 	if len(k_v)==2{
			ctx.Request.Header.Add(k_v[0],k_v[1])
		}
	 }

}
```

## 添加响应头过滤器

```go
package filters

import (
	"github.com/valyala/fasthttp"
	"strings"
)
const AddReponseHeaderAnnotation=AnnotationPrefix+"/add-response-header"

func init() {
	//注册响应 过滤器
	registerReponseFilter(AddReponseHeaderAnnotation,(*AddResponseHeaderFilter)(nil) )
}
type AddResponseHeaderFilter struct {
    pathValue string
    target string  //注解  值
    path string
}
func(this *AddResponseHeaderFilter) SetPath(value  string){}
//可变参数。第1个是 注解值:的值 如 /$1
func(this *AddResponseHeaderFilter) SetValue(values ...string){
	this.target=values[0]
}
func(this *AddResponseHeaderFilter) Do(ctx *fasthttp.RequestCtx){
	 kvList:=strings.Split(this.target,";")
	 for _,kv:=range kvList{
	 	k_v:=strings.Split(kv,"=")
	 	if len(k_v)==2{
			ctx.Response.Header.Add(k_v[0],k_v[1])
		}
	 }

}
```

## 创建最基本的控制器

```go
package k8sconfig

import (
	"context"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

//@Controller
type JtProxyController struct {
	client.Client
}

func NewJtProxyController() *JtProxyController {
	return &JtProxyController{}
}
func (r *JtProxyController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {

	return reconcile.Result{}, nil
}
func (r *JtProxyController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}
```

## 发布Ingress、自定义控制器接收

```go
package k8sconfig

import (
	"context"
	"fmt"
	v1 "k8s.io/api/networking/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

//@Controller
type JtProxyController struct {
	client.Client
}

func NewJtProxyController() *JtProxyController {
	return &JtProxyController{}
}
func (r *JtProxyController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	obj:=&v1.Ingress{}
	err:=r.Get(ctx,req.NamespacedName,obj)
	if err!=nil{
		return reconcile.Result{}, err
	}
	if v,ok:=obj.Annotations["kubernetes.io/ingress.class"];ok && v=="jtthink"{
		fmt.Println(obj)
	}


	return reconcile.Result{}, nil
}
func (r *JtProxyController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}

```

## 准备工作、网关和控制器同时启动

```go
package main

import (
	"fmt"
	"github.com/valyala/fasthttp"
	"jtproxy/pkg/filters"
	"jtproxy/pkg/k8sconfig"
	"jtproxy/pkg/sysinit"
	"log"
	"os"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
)
func ProxyHandler(ctx *fasthttp.RequestCtx){
	//代表匹配到了 path
	if getProxy:=sysinit.GetRoute(ctx.Request);getProxy!=nil{
		filters.ProxyFilters(getProxy.RequestFilters).Do(ctx) //过滤
		getProxy.Proxy.ServeHTTP(ctx) //反代
		filters.ProxyFilters(getProxy.ResponseFilters).Do(ctx) //过滤
	}else{
		ctx.Response.SetStatusCode(404)
		ctx.Response.SetBodyString("404...")
	}


}
func main() {
	logf.SetLogger(zap.New())
	var mylog=logf.Log.WithName("jtproxy")
	mgr, err := manager.New(k8sconfig.K8sRestConfig(), manager.Options{})
	if err != nil {
		mylog.Error(err, "unable to set up manager")
		os.Exit(1)
	}
	//++
	err=k8sconfig.SchemeBuilder.AddToScheme(mgr.GetScheme())
	if err != nil {
		mylog.Error(err, "unable add schema")
		os.Exit(1)
	}

	err=builder.ControllerManagedBy(mgr).
		//For(&v1.Ingress{}).Complete(k8sconfig.NewJtProxyController())
		For(&k8sconfig.Route{}).Complete(k8sconfig.NewJtProxyController())
	if err != nil {
		mylog.Error(err, "unable to create manager")
		os.Exit(1)
	}
	sysinit.InitConfig()  //初始化  业务系统配置
	errCh:=make(chan error)
	go func() {
		//启动控制器管理器
		if err=mgr.Start(signals.SetupSignalHandler());err!=nil{
			errCh<-err
		}
	}()
	go func() {
		 // 启动网关
		 if err=fasthttp.ListenAndServe(fmt.Sprintf(":%d", sysinit.SysConfig.Server.Port),ProxyHandler);err!=nil{
			 errCh<-err
		 }
	}()
	getError:=<-errCh
	log.Println(getError.Error())

}
```

## 发布自己的ingress、控制器监听和持久化配置

`main.go`

```go
package main

import (
	"fmt"
	"github.com/valyala/fasthttp"
	"jtproxy/pkg/filters"
	"jtproxy/pkg/k8sconfig"
	"jtproxy/pkg/sysinit"
	v1 "k8s.io/api/networking/v1"
	"log"
	"os"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
)
func ProxyHandler(ctx *fasthttp.RequestCtx){
	//代表匹配到了 path
	if getProxy:=sysinit.GetRoute(ctx.Request);getProxy!=nil{
		filters.ProxyFilters(getProxy.RequestFilters).Do(ctx) //过滤
		getProxy.Proxy.ServeHTTP(ctx) //反代
		filters.ProxyFilters(getProxy.ResponseFilters).Do(ctx) //过滤
	}else{
		ctx.Response.SetStatusCode(404)
		ctx.Response.SetBodyString("404...")
	}


}
func main() {
	logf.SetLogger(zap.New())
	var mylog=logf.Log.WithName("jtproxy")
	mgr, err := manager.New(k8sconfig.K8sRestConfig(), manager.Options{})
	if err != nil {
		mylog.Error(err, "unable to set up manager")
		os.Exit(1)
	}
	//++
	err=k8sconfig.SchemeBuilder.AddToScheme(mgr.GetScheme())
	if err != nil {
		mylog.Error(err, "unable add schema")
		os.Exit(1)
	}

	err=builder.ControllerManagedBy(mgr).
		For(&v1.Ingress{}).Complete(k8sconfig.NewJtProxyController())
	//For(&k8sconfig.Route{}).Complete(k8sconfig.NewJtProxyController())
	if err != nil {
		mylog.Error(err, "unable to create manager")
		os.Exit(1)
	}
	sysinit.InitConfig()  //初始化  业务系统配置
	errCh:=make(chan error)
	go func() {
		//启动控制器管理器
		if err=mgr.Start(signals.SetupSignalHandler());err!=nil{
			errCh<-err
		}
	}()
	go func() {
		 // 启动网关
		 if err=fasthttp.ListenAndServe(fmt.Sprintf(":%d", sysinit.SysConfig.Server.Port),ProxyHandler);err!=nil{
			 errCh<-err
		 }
	}()
	getError:=<-errCh
	log.Println(getError.Error())
}
```

`config.go`

```go
package sysinit

import (
	"io/ioutil"
	"k8s.io/api/networking/v1"
	"log"
	"os"
	"sigs.k8s.io/yaml"
)

type Server struct {
	Port int   //代表是代理启动端口
}
type SysConfigStruct struct {
	Server Server
	Ingress []v1.Ingress
}
var SysConfig =new(SysConfigStruct)
func InitConfig()  {
	config,err:=ioutil.ReadFile("./app.yaml")
	if err!=nil{
		log.Fatal(err)
	}
	err=yaml.Unmarshal(config,SysConfig)
	if err!=nil{
		log.Fatal(err)
	}
   ParseRule()

}
//更新 ingress对象 配置 ,更新内存配置 和 配置持久化
func ApplyConfig(ingress *v1.Ingress) error   {
  isEdit:=false
  for index,config:=range SysConfig.Ingress{
  	   if config.Name==ingress.Name && config.Namespace==ingress.Namespace{
		   SysConfig.Ingress[index]=*ingress
		   isEdit=true
		   break
	   }
  }
  if !isEdit{
	  SysConfig.Ingress=append(SysConfig.Ingress,*ingress)
  }
  b,_:=yaml.Marshal(SysConfig)
  appYamlFile,err:=os.OpenFile("./app.yaml",os.O_CREATE|os.O_TRUNC|os.O_WRONLY,0644)
	if err != nil {
		return err
	}
	defer appYamlFile.Close()
   _,err=appYamlFile.Write(b)
	return err
}
```

`controller.go`

```go
func (r *JtProxyController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	ingress:=&v1.Ingress{}
	err:=r.Get(ctx,req.NamespacedName,ingress)
	if err!=nil{
		return reconcile.Result{}, err
	}
	if v,ok:=ingress.Annotations["kubernetes.io/ingress.class"];ok && v=="jtthink"{
		 err=sysinit.ApplyConfig(ingress)
		 if err!=nil{
			 return reconcile.Result{}, err
		 }
	}
	return reconcile.Result{}, nil
}
```

## 发布Ingress出发配置重载

`config.go`

```go
package sysinit

import (
	"io/ioutil"

	"github.com/gorilla/mux"
	v1 "k8s.io/api/networking/v1"

	"os"

	"sigs.k8s.io/yaml"
)

type Server struct {
	Port int //代表是代理启动端口
}
type SysConfigStruct struct {
	Server  Server
	Ingress []v1.Ingress
}

var SysConfig = new(SysConfigStruct)

func InitConfig() error {
	config, err := ioutil.ReadFile("./app.yaml")
	if err != nil {
		return err
	}
	err = yaml.Unmarshal(config, SysConfig)
	if err != nil {
		return err
	}
	ParseRule()
	return nil

}

// 更新 ingress对象 配置 ,更新内存配置 和 配置持久化
func ApplyConfig(ingress *v1.Ingress) error {
	isEdit := false
	for index, config := range SysConfig.Ingress {
		if config.Name == ingress.Name && config.Namespace == ingress.Namespace {
			SysConfig.Ingress[index] = *ingress
			isEdit = true
			break
		}
	}
	if !isEdit {
		SysConfig.Ingress = append(SysConfig.Ingress, *ingress)
	}
	b, _ := yaml.Marshal(SysConfig)
	appYamlFile, err := os.OpenFile("./app.yaml", os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0644)
	if err != nil {
		return err
	}
	defer appYamlFile.Close()
	_, err = appYamlFile.Write(b)
	if err != nil {
		return err
	}

	return ReloadConfig()
}

// 重载配置
func ReloadConfig() error {
	MyRouter = mux.NewRouter()

	return InitConfig()
}

```

```go
package main

import (
	"fmt"
	"github.com/valyala/fasthttp"
	"jtproxy/pkg/filters"
	"jtproxy/pkg/k8sconfig"
	"jtproxy/pkg/sysinit"
	v1 "k8s.io/api/networking/v1"
	"log"
	"os"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/manager/signals"
)
func ProxyHandler(ctx *fasthttp.RequestCtx){
	//代表匹配到了 path
	if getProxy:=sysinit.GetRoute(ctx.Request);getProxy!=nil{
		filters.ProxyFilters(getProxy.RequestFilters).Do(ctx) //过滤
		getProxy.Proxy.ServeHTTP(ctx) //反代
		filters.ProxyFilters(getProxy.ResponseFilters).Do(ctx) //过滤
	}else{
		ctx.Response.SetStatusCode(404)
		ctx.Response.SetBodyString("404...")
	}


}

func main() {
	logf.SetLogger(zap.New())
	var mylog=logf.Log.WithName("jtproxy")
	mgr, err := manager.New(k8sconfig.K8sRestConfig(), manager.Options{})
	if err != nil {
		mylog.Error(err, "unable to set up manager")
		os.Exit(1)
	}
	//++
	err=k8sconfig.SchemeBuilder.AddToScheme(mgr.GetScheme())
	if err != nil {
		mylog.Error(err, "unable add schema")
		os.Exit(1)
	}

	err=builder.ControllerManagedBy(mgr).
		For(&v1.Ingress{}).
		Complete(k8sconfig.NewJtProxyController())
	//For(&k8sconfig.Route{}).Complete(k8sconfig.NewJtProxyController())
	if err != nil {
		mylog.Error(err, "unable to create manager")
		os.Exit(1)
	}
	if err=sysinit.InitConfig();err!=nil{//初始化  业务系统配置
		mylog.Error(err, "unable to load sysconfig")
		os.Exit(1)
	}
	errCh:=make(chan error)
	go func() {
		//启动控制器管理器
		if err=mgr.Start(signals.SetupSignalHandler());err!=nil{
			errCh<-err
		}
	}()
	go func() {
		 // 启动网关
		 if err=fasthttp.ListenAndServe(fmt.Sprintf(":%d", sysinit.SysConfig.Server.Port),ProxyHandler);err!=nil{
			 errCh<-err
		 }

	}()
	getError:=<-errCh
	log.Println(getError.Error())
}

```

## 删除ingress触发网关配置重载

```go
	proxyCtl:=k8sconfig.NewJtProxyController()
	err=builder.ControllerManagedBy(mgr).
			For(&v1.Ingress{}).
		    Watches(&source.Kind{
									Type: &v1.Ingress{},
								},
								handler.Funcs{DeleteFunc: proxyCtl.OnDelete}).
		    Complete(proxyCtl)
```

```go
package k8sconfig

import (
	"context"
	"jtproxy/pkg/sysinit"
	v1 "k8s.io/api/networking/v1"
	"k8s.io/client-go/util/workqueue"
	"log"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

//@Controller
type JtProxyController struct {
	client.Client
}

func NewJtProxyController() *JtProxyController {
	return &JtProxyController{}
}
func (r *JtProxyController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	ingress:=&v1.Ingress{}

	err:=r.Get(ctx,req.NamespacedName,ingress)
	if err!=nil{
		return reconcile.Result{}, err
	}
	if r.IsJtProxy(ingress.Annotations){
		 err=sysinit.ApplyConfig(ingress)
		 if err!=nil{
			 return reconcile.Result{}, err
		 }
	}
	return reconcile.Result{}, nil
}
func (r *JtProxyController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}
//判断是否 是否 我们所需要处理的ingress
func (r *JtProxyController) IsJtProxy(annotations map[string]string) bool {
	 if   v,ok:=annotations["kubernetes.io/ingress.class"];ok && v=="jtthink"{
		 return true
	}
	return false
}
func (r *JtProxyController) OnDelete(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface){
	if r.IsJtProxy(event.Object.GetAnnotations()){
		if err:=sysinit.DeleteConfig(event.Object.GetName(),event.Object.GetNamespace());err!=nil{
			log.Println(err)
		}
	}
}
```

## 手工简单部署打包镜像

```dockerfile
FROM golang:1.15-alpine3.12 as builder
RUN mkdir /src
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
ADD . /src
WORKDIR /src
RUN  GOPROXY=https://goproxy.cn go build -o jtproxy test.go  && chmod +x jtproxy


FROM alpine:3.12
ENV ZONEINFO=/app/zoneinfo.zip
RUN mkdir /app
WORKDIR /app

COPY --from=builder /usr/local/go/lib/time/zoneinfo.zip /app

COPY --from=builder /src/jtproxy /app

ENTRYPOINT  ["./jtproxy"]
EXPOSE 80
```

## 开发时测试部署、体内访问apiserver

`init.go`

```go
package k8sconfig

import (
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"os"
)


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

`rbac.yaml`

```go
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jtproxy-sa
  namespace: jtthink-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jtproxy-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jtproxy-ClusterRoleBinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jtproxy-clusterrole
subjects:
  - kind: ServiceAccount
    name: jtproxy-sa
    namespace: jtthink-system
```

`deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jtproxy-controller
  namespace: jtthink-system
spec:
  selector:
    matchLabels:
      app: jtproxy-controller
  replicas: 1
  template:
    metadata:
      labels:
        app: jtproxy-controller
    spec:
      nodeName: jtthink2
      serviceAccountName: jtproxy-sa
      containers:
        - name: jtproxy
          image: golang:1.15-alpine3.12
          imagePullPolicy: IfNotPresent
          env:
            - name: "GOPROXY"
              value: "https://goproxy.cn"
            - name: "release"
              value: "1"
          workingDir: "/app"
          command: ["go","run","test.go"]
          volumeMounts:
            - name: app
              mountPath: /app
            - name: gopath
              mountPath: /go
          ports:
            - containerPort: 80
      volumes:
        - name: app
          hostPath:
             path: /home/shenyi/jtingress
        - name: gopath
          hostPath:
            path: /home/shenyi/gopath
---
apiVersion: v1
kind: Service
metadata:
  name: jtproxy-svc
  namespace: jtthink-system
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 31180
  selector:
    app: jtproxy-controller
---

```

## 为ingress显示ip address

`controller.go`

```go
package k8sconfig

import (
	"context"
	"jtproxy/pkg/sysinit"
	corev1 "k8s.io/api/core/v1"
	v1 "k8s.io/api/networking/v1"
	"k8s.io/client-go/util/workqueue"
	"log"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

//@Controller
type JtProxyController struct {
	client.Client
}

func NewJtProxyController() *JtProxyController {
	return &JtProxyController{}
}
func (r *JtProxyController) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error) {
	ingress:=&v1.Ingress{}

	err:=r.Get(ctx,req.NamespacedName,ingress)
	if err!=nil{
		return reconcile.Result{}, err
	}
	if r.IsJtProxy(ingress.Annotations){

		err=sysinit.ApplyConfig(ingress)
		if err!=nil{
			return reconcile.Result{}, err
		}
		//本课时新增 ,修改IP
		if ingress.Status.LoadBalancer.Ingress==nil{
			ingress.Status.LoadBalancer.Ingress=[]corev1.LoadBalancerIngress{
				{IP: ServiceIP},
			}
			err = r.Status().Update(ctx, ingress)
			if err!=nil{
				return reconcile.Result{}, err
			}
		}

	}
	return reconcile.Result{}, nil
}
func (r *JtProxyController) InjectClient(c client.Client) error {
	r.Client = c
	return nil
}
//判断是否 是否 我们所需要处理的ingress
func (r *JtProxyController) IsJtProxy(annotations map[string]string) bool {
	 if   v,ok:=annotations["kubernetes.io/ingress.class"];ok && v=="jtthink"{
		 return true
	}
	return false
}
func (r *JtProxyController) OnDelete(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface){
	if r.IsJtProxy(event.Object.GetAnnotations()){
		if err:=sysinit.DeleteConfig(event.Object.GetName(),event.Object.GetNamespace());err!=nil{
			log.Println(err)
		}
	}
}
```

`initConfig.go`

```go
package k8sconfig

import (
	"context"
	"io/ioutil"
	v1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"os"
)
//全局变量
var ServiceIP string
const JtProxySvcName="jtproxy-svc" //默认的固定的 Svc名称
const NSFile="/var/run/secrets/kubernetes.io/serviceaccount/namespace"
//用来获取 svc的ip
func init() {
	if os.Getenv("release")!="1"{ //本机情况
		ServiceIP="127.0.0.1"  //写着玩的。
		return
	}
	config:=K8sRestConfig()
	//pod里会有这样一个文件
	//  namespace 在这里 /var/run/secrets/kubernetes.io/serviceaccount/namespace
	ns,_:=ioutil.ReadFile(NSFile) //取到了 命名空间
	client,err:=kubernetes.NewForConfig(config) //clientset
	if err!=nil{
		log.Fatal(err)
	}
	svc,err:=client.CoreV1().
		Services(string(ns)).
		Get(context.Background(),JtProxySvcName,v1.GetOptions{})
	if err!=nil{
		log.Fatal(err)
	}
	ServiceIP=svc.Spec.ClusterIP // 获取到控制器svc对应的clusterip

}
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



