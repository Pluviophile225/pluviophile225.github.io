# Kubernetes API 基本概念

![image-20230304151526319](../assets/image-20230304151526319.png)

## Resource

系统中实体。api server以yaml或json格式在HTTP协议下接受或发送。

- Resource可以单独对外API Server外部暴露，通过一个URL定位到一个具体的Resource；
- 也可以通过集合的形式对外暴露一组同类型的Resource。

Resource可以理解为Restful中的资源概念，例如用户数据库中的一个用户就是一个resource；一个resource逻辑上是一个有id的实体；一个resource和一个定位它的url一一对应

对Resource（同kind下的resource实体集合）命名使用“小写复数”形式；例如“pods”，“deployments”是resource

## API Group

代表了一组一同暴露的资源，这组资源划分为不同的version暴露出去

为group取名字的时候，推荐使用域名形式，并且全部小写，只有Kubernetes自己可以使用空域名定义group，例如“apps”，“events”，“node”，“policy”，它们都是group却没有域名

Group和Version共同构成了apiVersion这一概念，格式为group/version，例如“policy.k8s.io/v1”。一个Group可以有多个version，而version下面又会有多个resource。

例如“apps/v1”，"apps/v1beta1"，"apps/v1beta2"

## Kind

每一个Kind都代表一个“Schema”，Schema就是用于定义事物可以有什么属性的文档，参考xml Schema

例如可以说“狮子”是一个kind，“大象”是另一个kind，它们分别定义了两种动物的不同属性集合 Kind使用驼峰式命名并且单数形式。例如“ReplicaSet”，“StatefulSet”，“Pod”

> Kind与Resource的关系

动物园里的“一只只动物构成的群体”形成了“Resource”

“狮子”，“大象”等动物类型等同于它们的“Kind”

![image-20230304153302471](../assets/image-20230304153302471.png)

> Kind的三种类型（Type）

![image-20230304153436423](../assets/image-20230304153436423.png)

Object：该type下kind的实例代表存储的实体

![image-20230304153610955](../assets/image-20230304153610955.png)

List：该type下kind的实例代表一组实体

List类型的Kind名字是以List结尾，它代表一个类型，该类型的实例是由多个resource实例构成的集合。List描述这类集合具有的属性（毕竟List是Schema）

一般来说，每个kind都有一个endpoint可以返回其所有resource，这个集合就是一个List；

例如：PodList，ServiceList，NodeList

Simple Kind：该type下kind的实例一般都是那些临时用一用，虚拟的，不会单独实际存储的实体

![image-20230304154424186](../assets/image-20230304154424186.png)

## CRD

Custom Resource Definition，按照之前介绍的概念，叫Custom Object Definition会更合理一点。它的产物是Custom Object，是一个Schema，规定了一类实体可以有什么属性。

CRD自身也是一个API Object（Kind里的Object分类），其GVK为：apiextension.k8s.io/v1beta1/CustomResourceDefinition；

这个API Object由Extension API Server负责处理的

## Custom Resource

它是CRD产物，根据CRD确定自身有什么属性，为这些属性赋值来定义出一个该实例。

一个CR实例可以直接类比一个Pod实例。可以像使用“Pod”这个Object一样去使用一个CR。



# TLS证书签发（简单实现版）

用Golang实现一个CA-Certificate Authority，可以签发X509证书

![image-20230306100603399](../assets/image-20230306100603399.png)

![image-20230306100927597](../assets/image-20230306100927597.png)

初始化工程

![image-20230306101240248](../assets/image-20230306101240248.png)

## Http Server

在`httpserver`中的请求包装如下

```go
func Run() {
	if running {
		return
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/", rootHandler)
	mux.HandleFunc("/csr-template", getCsrTemplateHandler)
	mux.HandleFunc("/csr", signCsrHandler)
	server = &http.Server{
		Addr:    ":8111",
		Handler: mux,
	}

	running = true
	if server.ListenAndServe() != nil {
		running = false
		log.Print("can't start http server @ 8111")
	}
	running = false
}
```

可以看到访问`127.0.0.1:8111/csr-template`可以得到请求方的机构信息，再使用这些机构信息来向`127.0.0.1:8111/csr`来获取这个`csr`

`getCsrTemplateHandler`

> 这个函数就用来返回机构信息的模板

```go
func getCsrTemplateHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != "GET" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

	csr := ca.CertificateSigningRequest{
		SubjectCountry:            []string{"China"},
		SubjectOrganization:       []string{"Qinghua"},
		SubjectOrganizationalUnit: []string{"ComputerScience"},
		SubjectProvince:           []string{"Beijing"},
		SubjectLocality:           []string{"北京"},

		SubjectCommonName: "www.tsinghua.edu.cn",
		EmailAddresses:    []string{"ex@example.com"},
	}

	csrBytes, err := json.Marshal(csr)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusOK)
	w.Header().Set("content-type", "application/json")
	w.Write(csrBytes)
}
```

`signCsrHandler`

> 用来签发

```go
func signCsrHandler(w http.ResponseWriter, r *http.Request) {
	if r.Method != "POST" {
		w.WriteHeader(http.StatusMethodNotAllowed)
		return
	}

	reqBody, err := ioutil.ReadAll(r.Body)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	csr := &ca.CertificateSigningRequest{}
    // 这里将请求的json格式的body转换为csr中的属性
	err = json.Unmarshal(reqBody, csr)
	if err != nil {
		w.WriteHeader(http.StatusBadRequest)
		return
	}

	var sync chan int = make(chan int, 1)
    //开一个线程来签发csr
	go signCsrRoutine(w, csr, sync)

	<-sync
}
```

```go
func signCsrRoutine(w http.ResponseWriter, csr *ca.CertificateSigningRequest, sync chan<- int) {
	defer close(sync)
	theCert, err := ca.CA.SignX509(csr)
    // 使用X509的方式进行签发csr

	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, "error happen: %v", err)
		return
	}

	w.WriteHeader(http.StatusAccepted)
	w.Header().Add("Content-Type", "application/json")
	jsonByte, _ := json.Marshal(theCert)
	w.Write(jsonByte)

}
```

## CA

`CSR - CertificateSigningRequest`

```go
package ca

import (
	"crypto/x509"
	"encoding/asn1"
	"net"
	"net/url"
)

type CertificateSigningRequest struct {
	Version int

	SubjectCountry            []string
	SubjectOrganization       []string
	SubjectOrganizationalUnit []string
	SubjectLocality           []string
	SubjectProvince           []string
	SubjectStreetAddress      []string
	SubjectPostalCode         []string
	SubjectSerialNumber       string
	SubjectCommonName         string
	SubjectExtraNames         []DistinguishedName

	PublicKeyAlg       x509.PublicKeyAlgorithm
	SignatureAlgorithm x509.SignatureAlgorithm

	DNSNames       []string
	EmailAddresses []string
	IPAddresses    []net.IP
	URIs           []url.URL
	SANs           []SubjectAlternativeName
	Extensions     []Extension
}

type DistinguishedName struct {
	Type  asn1.ObjectIdentifier
	Value interface{}
}

type SubjectAlternativeName struct {
	Type  string
	Value string
}

type Extension struct {
	ID       asn1.ObjectIdentifier
	Critical bool
	Value    []byte
}

type Certificate struct {
	ID string `json:"certificateId"`
}

```

`本地的存储路径`

```go
const (
	rsaPrivateKeyLocation string = rootCAFolder + "/root.private.key"
	//rsaPrivateKeyPassword string = "123456"
	rootCALocation string = rootCAFolder + "/root.crt"

	localKeyLocation        string = localCAFolder + "/local.private.key"
	localCertLocation       string = localCAFolder + "/local.crt"
	localPrivateKeyPassword string = "123456"

	rootCAFolder   string = "cert/rootCA"
	clientCAFolder string = "cert/clientCert"
	localCAFolder  string = "cert/localCert"
)
```

`CA的结构体`

```go
type CertificateAuthority struct {
	RootCA     cx509.Certificate
	PrivateKey *rsa.PrivateKey
}
```

`加载证书和私钥信息`

```go
/*
从磁盘加载根证书和私钥信息
*/
func (ca *CertificateAuthority) load() {
	//如果没有配置根证书，我们自签一个
	if !checkFileExist(rootCALocation) || !checkFileExist(rsaPrivateKeyLocation) {
		if err := ca.makeRootCA(); err != nil {
			log.Print("can't create self-signed root CA")
			return
		}
		//我们需要同时签发本地server的certificate，用于后续的mTLS
		os.Remove(localCertLocation)
		os.Remove(localKeyLocation)
	}

	//加载 rootCA 的 private key
	bytes, err := ioutil.ReadFile(rsaPrivateKeyLocation)
	if err != nil {
		panic("can't load ca private key")
	}
	pemBlocks, _ := pem.Decode(bytes)
	if pemBlocks.Type != "ENCRYPTED PRIVATE KEY" {
		panic("ca private key type should be ENCRYPTED")
	}
	data, err := pkcs8.ParsePKCS8PrivateKeyRSA(pemBlocks.Bytes) //need package pkcs8 to parse
	if err != nil {
		panic("can't parse private key bytes via pkcs8")
	}
	ca.PrivateKey = data
	//加载 rootCA
	rootCABytes, err := ioutil.ReadFile(rootCALocation)
	if err != nil {
		panic("can't load root ca")
	}
	pemBlocks, _ = pem.Decode(rootCABytes)
	rootCA, err := cx509.ParseCertificate(pemBlocks.Bytes)
	if err != nil {
		panic("can't parse root ca")
	}
	ca.RootCA = *rootCA

	//我们检查是否需要生成本地server的certificate
	if !checkFileExist(localCertLocation) || !checkFileExist(localKeyLocation) {
		if err := ca.signLocalCert(); err != nil {
			log.Print("can't create local certificate")
			return
		}
	}
}
```

`自签一个证书`用做根证书

```go
/*
CA 做一个自签名证书，作为自己的根证书，当配置没有在cert\rootCA下提供根证书和私钥时，我们就自己做一个
*/
func (ca *CertificateAuthority) makeRootCA() error {
	privateKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		log.Print("error happens when generate private key to create root CA")
		return err
	}

	mathRand.Seed(time.Now().UnixNano())
	rootCertificateTemplate := cx509.Certificate{
		Version:      1,
		SerialNumber: big.NewInt((int64)(mathRand.Int())),
		Subject: pkix.Name{
			Country:            []string{"CN"},
			Organization:       []string{"Fudan"},
			OrganizationalUnit: []string{"Mathematics"},
			Locality:           []string{"上海"},
			Province:           []string{"Shanghai"},
			StreetAddress:      []string{"Handan Road #200"},
			PostalCode:         []string{"200201"},
			CommonName:         "Fudan CA",
		},

		EmailAddresses: []string{"jacky01.zhang@outlook.com"},
		DNSNames:       []string{"localhost"},

		NotBefore:             time.Now(),
		NotAfter:              time.Now().AddDate(10, 0, 0),
		IsCA:                  true,
		BasicConstraintsValid: true,
	}
	buf, err := cx509.CreateCertificate(rand.Reader, &rootCertificateTemplate, &rootCertificateTemplate, &privateKey.PublicKey, privateKey)
	if err != nil {
		log.Print("sign the root ca fail")
		return err
	}
	err = saveToPEM(buf, rootCAFolder, "root.crt", "CERTIFICATE")
	if err != nil {
		log.Print("persistent the root ca fail")
		return err
	}

	buf, err = pkcs8.MarshalPrivateKey(privateKey, nil, nil)
	if err != nil {
		log.Print("marshal ca private key fail")
		return err
	}
	err = saveToPEM(buf, rootCAFolder, "root.private.key", "ENCRYPTED PRIVATE KEY")
	if err != nil {
		log.Print("persistent the root ca private key fail")
		return err
	}

	return nil
}
```

用根证书签署一个证书签发请求CSR

```go
/*
用根证书签署一个证书签发请求CSR。CSR是以我自己的Struct表达的
*/
func (ca *CertificateAuthority) SignX509(csr *CertificateSigningRequest) (*Certificate, error) {
	// 生成一个私钥
	csrPrivateKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		log.Print("error happens when generate private key to sign CSR")
		return nil, err
	}

	cx509CSR := csr.toCX509CSR(csrPrivateKey)
    // 转换为cx509的格式

	mathRand.Seed(time.Now().UnixNano())
    //填入CertificateTemplat
	cx509CertificateTemplate := cx509.Certificate{
		Version:            cx509CSR.Version,
		SerialNumber:       big.NewInt((int64)(mathRand.Int())),
		Signature:          cx509CSR.Signature,
		SignatureAlgorithm: cx509CSR.SignatureAlgorithm,
		PublicKey:          cx509CSR.PublicKey,
		PublicKeyAlgorithm: cx509CSR.PublicKeyAlgorithm,
		Subject:            cx509CSR.Subject,

		URIs:           cx509CSR.URIs,
		DNSNames:       cx509CSR.DNSNames,
		EmailAddresses: cx509CSR.EmailAddresses,
		IPAddresses:    cx509CSR.IPAddresses,

		Extensions: cx509CSR.Extensions,

		NotBefore:             time.Now(),
		NotAfter:              time.Now().AddDate(1, 0, 0),
		BasicConstraintsValid: true,
	}

	buf, err := cx509.CreateCertificate(rand.Reader, &cx509CertificateTemplate, &ca.RootCA, cx509CSR.PublicKey, ca.PrivateKey)
    // 用根证书签发一个证书
	if err != nil {
		log.Print("sign the x509 csr fail")
		return nil, err
	}
	_, err = cx509.ParseCertificate(buf) //just to verify the generated byte[] is a qualified certification
	if err != nil {
		log.Print("verify the cx509 certificate fail")
		return nil, err
	}

	var fileNamePrefix string = time.Now().Format("2006-01-02_15-04-05")
	err = saveToPEM(buf, clientCAFolder, fileNamePrefix+".crt", "CERTIFICATE")
	if err != nil {
		log.Print("persistent generated certificate file fail")
		return nil, err
	}
	err = saveToPEM(cx509.MarshalPKCS1PrivateKey(csrPrivateKey), clientCAFolder, fileNamePrefix+".key", "PRIVATE KEY")
	if err != nil {
		log.Print("persistent generated private key file fail")
		return nil, err
	}
    // 保存.key 和 .crt一个是密钥，一个是证书

	res := &Certificate{ID: fileNamePrefix}
	return res, err
}
```



# gRPC + mTLS

简单版的遗留问题：

1、证书要返回给请求方、私钥要返回给请求方

2、请求方和签发机构之间的交互的安全性   要验证请求方、签发机构，确保它们的安全性

解决方案：

1、可以使用传统方式，请求向HTTP Server请求文件下载；但是使用gRPC会更加快一点

2、mTLS是首选，证书验证完成了“登录”；证书中信息可以用来鉴权；顺便把传输加密也做了

![image-20230307112837729](../assets/image-20230307112837729.png)

步骤：

1. 定义gRPC消息和服务，并生成golang代码
2. 实现定义的服务接口，用gRPC服务器取代http1.1服务器
3. 为gRPC添加mTLS，进行双向TLS验证
4. 准备mTLS证书，并测试

> mTLS所用证书从哪里来 - dapr项目的做法

![image-20230307123735712](../assets/image-20230307123735712.png)

![image-20230307123925229](../assets/image-20230307123925229.png)

## gRPC

写`.proto`文件

```protobuf
syntax = "proto3";
import "google/protobuf/empty.proto";

option go_package = "/;grpc";

package grpc;

message CertificateSigningRequest {
  repeated string SubjectCountry = 1;
  repeated string SubjectOrganization  = 2;
  repeated string SubjectOrganizationalUnit = 3;
  repeated string SubjectLocality = 4;
  repeated string SubjectProvince = 5;
  repeated string SubjectStreetAddress = 6;
  repeated string SubjectPostalCode = 7;
  string SubjectSerialNumber = 8;
  string SubjectCommonName = 9;
  repeated string DNSNames = 10;
  repeated string EmailAddresses = 11;
  repeated string  IPAddresses = 12;
}

message SignResponse {
  string CertificateId = 1;
}

message FileIdentifer {
  string Id = 1;
}

message FileStream {
  bytes contents = 1;
}

service CertificateService {
  rpc CsrTemplate(google.protobuf.Empty) returns (CertificateSigningRequest){}
  rpc SignCsr(CertificateSigningRequest) returns (SignResponse){}
  rpc GetCert(FileIdentifer) returns (FileStream) {}
  rpc GetKey(FileIdentifer) returns (FileStream) {}
}
```

执行命令行

```shell
protoc --go_out=. service.proto --go-grpc_out=. service.proto
```

根据生成的代码将这些`service`实现了

```go
package server

import (
	"context"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"
	"log"
	"myca/pkg/ca"
	mygrpc "myca/pkg/grpc"
	"net"
	"strconv"
	"strings"
)

type ServerError struct {
	msg string
}

func (e *ServerError) Error() string {
	return e.msg
}

type certificateServiceServer struct {
	mygrpc.UnimplementedCertificateServiceServer
}

/*
return a csr template, to easy requester's work
*/
func (s *certificateServiceServer) CsrTemplate(context.Context, *emptypb.Empty) (*mygrpc.CertificateSigningRequest, error) {
	csr := &mygrpc.CertificateSigningRequest{
		SubjectCountry:            []string{"China"},
		SubjectOrganization:       []string{"Qinghua"},
		SubjectOrganizationalUnit: []string{"ComputerScience"},
		SubjectProvince:           []string{"Beijing"},
		SubjectLocality:           []string{"北京"},

		SubjectCommonName: "tsinghua.edu.cn",
		EmailAddresses:    []string{"ex@example.com"},
		DNSNames:          []string{"localhost"},
		IPAddresses:       []string{"0.0.0.0", "127.0.0.1"},
	}

	return csr, nil
}

/*
Sing a certificate signing request
*/
func (s *certificateServiceServer) SignCsr(ctx context.Context, csrReq *mygrpc.CertificateSigningRequest) (*mygrpc.SignResponse, error) {
	csr := &ca.CertificateSigningRequest{}

	csr.DNSNames = csrReq.DNSNames
	csr.EmailAddresses = csrReq.EmailAddresses
	csr.SubjectCommonName = csrReq.SubjectCommonName

	csr.SubjectCountry = csrReq.SubjectCountry
	csr.SubjectLocality = csrReq.SubjectLocality
	csr.SubjectOrganization = csrReq.SubjectOrganization
	csr.SubjectOrganizationalUnit = csrReq.SubjectOrganizationalUnit
	csr.SubjectPostalCode = csrReq.SubjectPostalCode
	csr.SubjectProvince = csrReq.SubjectProvince
	csr.SubjectSerialNumber = csrReq.SubjectSerialNumber
	csr.SubjectStreetAddress = csrReq.SubjectStreetAddress

	for _, ipStr := range csrReq.IPAddresses {
		ips := strings.Split(ipStr, ".")
		if len(ips) != 4 {
			continue
		}
		var ip net.IP
		for _, ele := range ips {
			v, err := strconv.ParseUint(ele, 10, 8)
			if err != nil {
				continue
			}
			ip = append(ip, byte(v))
		}
		if len(ip) != 4 {
			continue
		}
		csr.IPAddresses = append(csr.IPAddresses, ip)
	}

	theCert, err := ca.CA.SignX509(csr)

	if err != nil {
		return nil, status.Error(codes.Internal, "singing csr fail")
	}

	result := &mygrpc.SignResponse{CertificateId: theCert.ID}
	return result, nil
}

/*
return the generated certificate
*/
func (s *certificateServiceServer) GetCert(ctx context.Context, in *mygrpc.FileIdentifer) (*mygrpc.FileStream, error) {
	contents, err := ca.CA.GetCertFile(in.Id)
	if err != nil {
		log.Printf("can't find the expected client certificate file %v", err)
		return nil, &ServerError{msg: "can't get the expected file"}
	}
	return &mygrpc.FileStream{Contents: contents}, nil
}

/*
return the generated private key
*/
func (s *certificateServiceServer) GetKey(ctx context.Context, in *mygrpc.FileIdentifer) (*mygrpc.FileStream, error) {
	contents, err := ca.CA.GetKeyFile(in.Id)
	if err != nil {
		log.Printf("can't find the expected client private key file %v", err)
		return nil, &ServerError{msg: "can't get the expected file"}
	}
	return &mygrpc.FileStream{Contents: contents}, nil
}
```

用cobra命令`cobra-cli add caserver`命令加入新的cobra指令行

```go
/*
Copyright © 2022 NAME HERE <EMAIL ADDRESS>
*/
package cmd

import (
	"github.com/spf13/cobra"

	grpcserver "myca/pkg/grpc/server"
	"myca/pkg/httpserver"
	"myca/pkg/util"
)

// caserverCmd represents the caserver command
var caserverCmd = &cobra.Command{
	Use:   "caserver",
	Short: "start CA web server",
	Long:  `A CA web server can response to certificate signing request`,
	Run: func(cmd *cobra.Command, args []string) {
		startServer()
	},
}

var useGRPC *bool
var useMTLS *bool

func init() {
	rootCmd.AddCommand(caserverCmd)

	// Here you will define your flags and configuration settings.

	// Cobra supports Persistent Flags which will work for this command
	// and all subcommands, e.g.:
	// caserverCmd.PersistentFlags().String("foo", "", "A help for foo")

	// Cobra supports local flags which will only run when this command
	// is called directly, e.g.:
	// caserverCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
	useGRPC = caserverCmd.Flags().Bool("grpc", true, "enable the gRPC instead of http1.1")
	useMTLS = caserverCmd.Flags().Bool("mtls", true, "enable the mtls for gRPC, no effect when don't use gRPC")
}

/*
start the http server
*/
// 从输入参数读入grpc和mtls两个参数的值
func startServer() {
	if *useGRPC {
		grpcserver.Run(*useMTLS, util.Shutdown())
	} else {
		httpserver.Run()
	}
}

```

## mTLS

```go
func createTLSCredentials() (credentials.TransportCredentials, error) {
	caPEMFile, err := ioutil.ReadFile("cert/rootCA/root.crt") //assume both grpc server and client's certificate are signed by same CA
	if err != nil {
		return nil, err
	}

	caPool := cx509.NewCertPool()
	if !caPool.AppendCertsFromPEM(caPEMFile) {
		return nil, &ServerError{msg: "load local cert fail"}
	}

	localCert, err := tls.LoadX509KeyPair("cert/localCert/local.crt", "cert/localCert/local.private.key")
	if err != nil {
		log.Print("load local certificate and key file fail")
		return nil, err
	}

	config := &tls.Config{
		Certificates: []tls.Certificate{localCert},
		ClientAuth:   tls.RequireAndVerifyClientCert, //means mTLS, will check client's certificate
		ClientCAs:    caPool,
	}

	return credentials.NewTLS(config), nil
}

```

# 部署到集群

整个流程如下

![image-20230307192617031](../assets/image-20230307192617031.png)

## 写好Dockerfile

```dockerfile
#### image 1, for building the myca
FROM golang:1.20.1 as builder
LABEL maintainer="chenlong"

RUN mkdir -p /go/src/myca
WORKDIR /go/src/myca
COPY . /go/src/myca
RUN go env -w GOPROXY=https://goproxy.cn,direct && GOOS=linux go build -a -o myca .

#### image 2, the ca image which can be pulled and ran
FROM alpine:latest
LABEL maintainer="chenlong"

WORKDIR /
COPY --from=builder /go/src/myca/myca .
RUN mkdir cert && mkdir cert/clientCert && mkdir cert/localCert && mkdir cert/rootCA
#line below is needed in alpine, otherwise exe can't run
RUN mkdir /lib64 && ln -s /lib/libc.musl-x86_64.so.1 /lib64/ld-linux-x86-64.so.2

EXPOSE 8112

CMD ["../myca","caserver","--grpc=false","--mtls=false"]
```

运行build指令

```shell
docker build -t myca:1.0 .
```

接着push到docker hub中

```shell
docker tag myca:1.0 chenlong152/myca:1.0
docker push chenlong152/myca:1.0
```

## 写yaml文件

```yaml
apiVersion: v1
kind: Service
metadata:
    name: myca-service
    labels:
        app: myca-service
spec:
    type: NodePort
    ports:
    - port: 8112
      protocol: TCP
      targetPort: 8112
    selector:
        run: myca
---
apiVersion: apps/v1
kind: Deployment
metadata:
    name: myca-deployment
spec:
  selector:
      matchLabels:
          run: myca
  replicas: 1
  template:
      metadata:
          labels:
              run: myca
      spec:
          containers:
          - name: myca
            image: docker.io/chenlong152/myca:1.0
            imagePullPolicy: Always
            ports:
            - containerPort: 8112
```

