# How to get Kubernetes API host and port

```shell
[root@master ~]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.199.100:6443
KubeDNS is running at https://192.168.199.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://192.168.199.100:6443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

```shell
[root@master ~]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.199.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

既然kubectl在`$HOME/.kube`下找到的`config`文件，为什么不直接从这里读取API地址？

因为有可能会有配置合并，因为可能有很多个配置文件会`merge`以后得到一个配置。

我们可以这样设置`KUBE_API`地址方便下面使用

```shell
$ KUBE_API=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}')
```

# How to call Kubernetes API using curl

任何HTTP客户端都可以，(curl,httpie,wget,postman)

# Authenticating API server to client

:one:首先尝试查询`API's /version`

```shell
[root@master ~]# curl $KUBE_API/version
curl: (60) Peer's Certificate issuer is not recognized.
More details here: http://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
```

:sob:没有办法访问呢

```shell
[root@master crt]# cat /root/.kube/config 
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJek1ESXhPREUyTVRRek9Wb1hEVE16TURJeE5URTJNVFF6T1Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTHlJCjFXTUVSMVlqNFpWYUJOS1ZyNXZJSzF5WVRmQTBSZ2VrMlFTdi9mNTlHVlB2TVppYmVKSkVFc3g1SStEdFBDME8KWjZKMU5FYjZYM0RFc1UzWW4rbkwydk1hS0JjWWN4aXhLUHBoRnVoRm9uWG1QM0ZNc2JrZkVYOXVuRmpGb1Z2NwoyRFdTaGxVZmVZR3ZjdkxldURuZ1ZacllzUUhRSUxVZFNwbE1oMVRYOEF2UWtER0U2SWZoQVhFWjZGdDVxSkw0CklHWlc5d0VheTdJdDIvN0c1eFlXK3J1N3duSTFBT3o4bVNEbDZrWVRLN092OWRLdmxNcTVuSUJrRWNJRmZOS1UKU2R2K0o4Uk9JK1lwdUgxeEliVmNDM3JvWDFFRjg2MERPVnY3MUpkNUNKY25na25ydkhJeGM3b2R2ZEpsajZhVwpOalFJckkzdC96UHU3VnpOY25jQ0F3RUFBYU5DTUVBd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZIRlNsSEQvNDc4c2hPY1Zycmo5ZHFUOThnL3pNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFDaDFSdUVtbzBCRjNuWVB3Zi80QUxCSjZZQ3JlWC84VW4vYnFTQmMza3E2SFV1T1JLTwp3bVY0SmNDSy9FTnR2REdhZGZmRVZpYW9zWUgybkdWc2NXY1JId1BwU0VSMk1JU01VZmlNQU84Vng1RjZaciszClB2bXhhT0RuSVMvT1pRb003amJ5RGZRRTdhZk1Qa09EWk5KWHl3OExxWllIRWJQV1cwMUthOWZCbEt0enVPV3AKbjRJYVJMbFlpTTFHdmdscnNETEhSd0REc1czbENpZzhZRm9qaWlnUy9COUxQUjJDSHBQSDVlTE84SlBUNy8zKwo2eStDYnMyNEtOZnAxQmxPdUZRTEVscnovVjJRK3pFLzR4UERmam44SldRN3RjMDIwL3ZUaUI5MGZ5TEIvRkxnCjExRkxIR1hRUy9MNG9IbUJJWWRRKzJ2M3Ruc1hjeTkvcUJKawotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://192.168.199.100:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURFekNDQWZ1Z0F3SUJBZ0lJQmxaMDVPaDM5MEF3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TXpBeU1UZ3hOakUwTXpsYUZ3MHlOREF5TVRneE5qRTBOREZhTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXF2YlN5TnNyYTlpdHdBd08KSkh1SmRFb2lObFd6Ylk4Rkp4Z2xmajFKb0IwcFh4U1R1cTNjcW12bmkrM3A1TE4rT0wwMzFUbU1SNDVoL20xKwpFUysxWE5xZTdpTU9UdmZ0V1hLdDRMcTFLbjlhS1o5bmU0TUdTQ0FqZ29FN1I4ZjNzRU9Pb3UwZkpQVVFtUElNCjlPMm1WQytUSzZ3MGlTWGZyZ1dvRGNzV0Q5RnAvRHpmNkY3TVZmMTliNjJlL1U0WnF4d2dLc2gxL2pjUVhDRXUKMFBqaXdSWG5SZmUxUzJxQXBmVW1GdU5aZGhuUHUrQlpCUXpzTlBhV0o5SjloVTMxWmxObVF3OGIySGNoUkt2UwpqN3JNTWZrbnBDOFV0RW5KaTNXbXBNMzJldzNVZTdFR1NoeWVRVTNQWFJOSEZvSGlnM3Q5a3hzSG1VVTBhcEg0CldCY3JZUUlEQVFBQm8wZ3dSakFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0h3WURWUjBqQkJnd0ZvQVVjVktVY1AvanZ5eUU1eFd1dVAxMnBQM3lEL013RFFZSktvWklodmNOQVFFTApCUUFEZ2dFQkFMd2l4aDFOL2NqVWdGSGJhLzhPWVJlbUxreFVXTHZqRThvZGFPSUJHRitRd0Uwb3JwcnlHeXoyCi8ybDFMdGg2cnhVSE1XVnI1VXFPWlV1WVV3SnBJTWVLVElYRnAydnoyaHdKcVNvSTQ1NUJJS2lVZUx4YVhLdGQKWjZSQUR0YWJyT25rR2c0dUZ5R09NMVNid0VzVnJ6cVF5RDI0SDFmUDUxNmJIb29kSERNNTRhdkYxNjUya25MVApjTXNGY045aFdLU0ZBT0hDdGJYUGJPTUNOL2t3T2VROWVFUUNNN2VjbzBHU3pkdEQ2OS80SFJsSnBlSnYxN0NuCnZORS9RWFAzdThsdXlQMUQzQUIzUEh3ZGUrM2kzMndYYU0rRHpoZDlQc0RPdzllN0VZYW9BRDd0a0VxWjFVa28KeEJkUnljSzZobHRsMUlkRTRUSzZQaTRZK1gweFN3TT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb2dJQkFBS0NBUUVBcXZiU3lOc3JhOWl0d0F3T0pIdUpkRW9pTmxXemJZOEZKeGdsZmoxSm9CMHBYeFNUCnVxM2NxbXZuaSszcDVMTitPTDAzMVRtTVI0NWgvbTErRVMrMVhOcWU3aU1PVHZmdFdYS3Q0THExS245YUtaOW4KZTRNR1NDQWpnb0U3UjhmM3NFT09vdTBmSlBVUW1QSU05TzJtVkMrVEs2dzBpU1hmcmdXb0Rjc1dEOUZwL0R6Zgo2RjdNVmYxOWI2MmUvVTRacXh3Z0tzaDEvamNRWENFdTBQaml3UlhuUmZlMVMycUFwZlVtRnVOWmRoblB1K0JaCkJRenNOUGFXSjlKOWhVMzFabE5tUXc4YjJIY2hSS3ZTajdyTU1ma25wQzhVdEVuSmkzV21wTTMyZXczVWU3RUcKU2h5ZVFVM1BYUk5IRm9IaWczdDlreHNIbVVVMGFwSDRXQmNyWVFJREFRQUJBb0lCQUFWaVlLRVN4ZnRQaDZsVQp0OTFPUnJYeTM4RDJVZ0JSVU1nNmFuUGZXa0pBcU56bHVRRllHR3NGbXZVOU9QQ0s5cDZ5MXQ5UVFLckFRVFhTCkhQWk5tbGlpU2Y4Vis0MWhJWWgvcEJvL3h4VGZqZWRocmRDbC83eWx4bmlGdVdnNVZBT3BIUVRra3VhSEVVNi8KME1pbDgyY1RXSDgzblMvMGtXYlpwc0ZJZEJsclljbkQ2NlJIWlN1aTgyY2hFckdNSHFOUEtUOVBLbWwyMEdvbgoxcEFFNDFqQkR3Z1g2ZmNOTzY3Y0ZMb01wVlZXYTVDMzYyTVQza2Y5L21oNjFjaEg1SE53M0VhWjg1c3JmZkJVCktXbUpHTC9UbDJ5UTduai8xN2huWkZVLy9vMTJZMW5DNDNtMUlGQVJIdldqMEJmMnN1OTFFZG1UY280eVkxVmoKcmEvekFBRUNnWUVBMjRzajFUdWQ1T29kTHU1NUV6MXVwcjdaRHE1bXBuZXhudysrU2k1SmY0aHVKQmlRMUtreQoyUmRWRHR1M0ZGRmFiVDU4SjVsOVlaQ3l2dVhGSUVQMThjTldjRTVma29ld1pqNldxMnc3TG9rRkJCOGhyZkU1ClpaM0RxRVU0bXhzTzJlV1psWHRVdG92UVZkbXk4K0pzdlBIKzlsR1ZhTnc0RGlYMW5mMGpmV0VDZ1lFQXgxcVAKY05vRk1IakNxSXFTbFZqVGhOcXNySGRlbnRQRHVKNjJKeUpCMHFWQmxtSWZMaWhGTlBEakhtaWxacUpNTEFYagpyeEJkd3JpU2JVYmI5NmpUZFZUdkdyeFk3OS9GbWM1QmxvNVpvSVd2eEQ2UEFrNHdEWktFd1VVL2pVMkYwdUM2Cmd4Y2o3d0M1SVZ0WHJRZVV5WVNXS2ljUkE3NXNOeW54VGYrVWJnRUNnWUJSK3FydXZNeEE1b3J2TTIxU21lWHYKcmVRdmIwQTFlUXlDY01hRnZMTUZSRlNjZGUvZStTOWJrVExaMFlHVHZLMGZqZTJlZTlvdHpISnloaW9OMmxMRQpiRVNpdXlGRS9oWUlsK1o3TEhjTThXMUdGTG5tMGVTMDVTeGljVGFwOUhpZk5QVWN0R2oxb1UreVB4QnJzV2taClJPUUg1bjc4SVA5dGlROG1aNWdSQVFLQmdIY3g1WXdUUDRFUTQwckV1QXBGOXdwN2VUMFJqbWltczJLaXVzVEIKVGR2MTVUWldhdEE5VWN2cXI5R1J2anVVbExqSnVLNEd1aGpnSk9UanRrZnBFSzRaMzNENzVxMWQvWmNONU5keApPNU9uKzBUNkpxVzVQREFSU0FFTE40bDBMYXk5bzZjWDRldFlZbGpZZFo3R1pxYnErS0l4ZzVIYWZIZXJRMVZnCm1FNEJBb0dBR0UzcVNhL3poWnVWTjR4bEFxN2R6VmFmYTNsb1pSbTlPQjFXQlFDaE5ZZW5XSnQ3NDN5Y1JELzYKRlg1V3J2ZitlNzluZjhrckRudXNPSWtGL0pweE5HdlJuL1JNbGdNQldHU0lvNHExRjhQVlJ4bSt4Tk9GVHQ5dApDMWw4dTI4QjBHMW1PSW1rUXpPVjdzRnhOUzVOZ29haW1GTVFFOFk2Nm9CdXA0YU8zdlk9Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```

:two:接着把证书设为环境变量

```shell
export clientcert=$(grep client-cert ~/.kube/config |cut -d" " -f 6)
echo $clientcert
```

```shell
export clientkey=$(grep client-key-data ~/.kube/config |cut -d" " -f 6)
echo $clientkey
```

```shell
export certauth=$(grep certificate-authority-data ~/.kube/config |cut -d" " -f 6)
echo $certauth
```

加密这些变量

```shell
echo $clientcert | base64 -d > ./client.pem
echo $clientkey | base64 -d > ./client-key.pem
echo $certauth | base64 -d > ./ca.pem
```

:three:通过这些变量访问

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem $KUBE_API/version
{
  "major": "1",
  "minor": "20",
  "gitVersion": "v1.20.6",
  "gitCommit": "8a62859e515889f07e3e3be6a1080413f17cf2c3",
  "gitTreeState": "clean",
  "buildDate": "2021-04-15T03:19:55Z",
  "goVersion": "go1.15.10",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

Yay! 🎉

# Authenticating Client to API Server using Certificates

像上面那步一样设置就可以了

```shell
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem $KUBE_API/apis/apps/v1/deployments
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "194785"
  },
  "items": [...]
}
```

😍

# Authenticating Client to API Server using Service Account Tokens

:one:创建一个Token

```shell
# Kubernetes <1.24
$ JWT_TOKEN_DEFAULT_DEFAULT=$(kubectl get secrets \
    $(kubectl get serviceaccounts/default -o jsonpath='{.secrets[0].name}') \
    -o jsonpath='{.data.token}' | base64 --decode)

# Kubernetes 1.24+
$ JWT_TOKEN_DEFAULT_DEFAULT=$(kubectl create token default)
```

:two:接着直接用这个头来访问

```shell
[root@master crt]# curl --cacert ./ca.pem $KUBE_API/apis/apps/v1 --header "Authorization: Bearer $JWT_TOKEN_DEFAULT_DEFAULT"
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "apps/v1",
  "resources": [...]
}
```

:+1:很Nice，但是就好奇了，他有多大的权限呢？

小试一下，又不行了:cry:

```shell
[root@master crt]# curl --cacert ./ca.pem $KUBE_API/apis/apps/v1/namespaces/default/deployments --header "Authorization: Bearer $JWT_TOKEN_DEFAULT_DEFAULT"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:serviceaccount:default:default\" cannot list resource \"deployments\" in API group \"apps\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

:three:那我们使用更加强大的`kube-system`来试试呗

```shell
# Kubernetes <1.24
$ JWT_TOKEN_KUBESYSTEM_DEFAULT=$(kubectl -n kube-system get secrets \
    $(kubectl -n kube-system get serviceaccounts/default -o jsonpath='{.secrets[0].name}') \
    -o jsonpath='{.data.token}' | base64 --decode)

# Kubernetes 1.24+
$ JWT_TOKEN_KUBESYSTEM_DEFAULT=$(kubectl -n kube-system create token default)
```

```shell
[root@master crt]# curl --cacert ./ca.pem $KUBE_API/apis/apps/v1/namespaces/default/deployments --header "Authorization: Bearer $JWT_TOKEN_KUBESYSTEM_DEFAULT"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "deployments.apps is forbidden: User \"system:serviceaccount:kube-system:default\" cannot list resource \"deployments\" in API group \"apps\" in the namespace \"default\"",
  "reason": "Forbidden",
  "details": {
    "group": "apps",
    "kind": "deployments"
  },
  "code": 403
}
```

我这边还是不行，但是博客上的那个是可以的，但我觉得问题不大，其实就是一个权限的问题，其实可以调整权限的吧。

# Bonus: How to call Kubernetes API from inside a Pod

这个就不照着做了，其实是一样的。

```shell
$ kubectl run -it --image curlimages/curl --restart=Never mypod -- sh
$ env | grep KUBERNETES
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
```

```shell
$ curl https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/apis/apps/v1 \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

# Creating, Reading, Watching, Updating, Patching, and Deleting objects

嗯。  :yum:很重要的一个知识点咧。

```tex
GET    /<resourcePlural>            -  Retrieve a list of type <resourceName>.


POST   /<resourcePlural>            -  Create a new resource from the JSON
                                       object provided by the client.

GET    /<resourcePlural>/<name>     -  Retrieves a single resource with the
                                       given name.

DELETE /<resourcePlural>/<name>     -  Delete the single resource with the
                                       given name.

DELETE /<resourcePlural>            -  Deletes a list of type <resourceName>.


PUT    /<resourcePlural>/<name>     -  Update or create the resource with the given
                                       name with the JSON object provided by client.

PATCH  /<resourcePlural>/<name>     -  Selectively modify the specified fields of
                                       the resource.

GET    /<resourcePlural>?watch=true -  Receive a stream of JSON objects 
                                       corresponding to changes made to any
                                       resource of the given kind over time.
```

## Create

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem $KUBE_API/apis/apps/v1/namespaces/default/deployments \
  -X POST \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "365d"]
'
```

## Get all objects in the default namesapce

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem $KUBE_API/apis/apps/v1/namespaces/default/deployments
```

## Get an object by a name and a namespace

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem  $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep
```

## Watch

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem  $KUBE_API/apis/apps/v1/namespaces/default/deployments?watch=true
```

## Update

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem  $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
  -X PUT \
  -H 'Content-Type: application/yaml' \
  -d '---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "730d"]  # <-- Making it sleep twice longer
'
```

## Delete

```shell
[root@master crt]# curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem  $KUBE_API/apis/apps/v1/namespaces/default/deployments/sleep \
- X DELETE
```

# How to call Kubernetes API using kubectl

```shell
$ kubectl config current-context
cluster1

$ kubectl proxy --port=8080 &
```

```shell
$ curl localhost:8080/apis/apps/v1/deployments
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "resourceVersion": "660883"
  },
  "items": [...]
}
```

```shell
# Sends HTTP GET request
$ kubectl get --raw /api/v1/namespaces/default/pods

# Sends HTTP POST request
$ kubectl create --raw /api/v1/namespaces/default/pods -f file.yaml

# Sends HTTP PUT request
$ kubectl replace --raw /api/v1/namespaces/default/pods/mypod -f file.json

# Sends HTTP DELETE request
$ kubectl delete --raw /api/v1/namespaces/default/pods
```

# Bonus: Kubernetes API calls equivalent to a kubectl command

```shell
$ kubectl scale deployment sleep --replicas=2 -v 6
I0116 ... loader.go:372] Config loaded from file:  /home/vagrant/.kube/config
I0116 ... cert_rotation.go:137] Starting client certificate rotation controller
I0116 ... round_trippers.go:454] GET https://192.168.58.2:8443/apis/apps/v1/namespaces/default/deployments/sleep 200 OK in 14 milliseconds
I0116 ... round_trippers.go:454] PATCH https://192.168.58.2:8443/apis/apps/v1/namespaces/default/deployments/sleep/scale 200 OK in 12 milliseconds
deployment.apps/sleep scaled
```

> Wanna see the actual request and response bodies? Increase the log verbosity up to **8**.









