# kubectl配置

- 下载kubectl

- 将下载下来的`kubectl`拷贝到master节点

- ```sh
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin
  ```

  添加使用权限、并且放到bin目录中，这一步以后使用kubectl version看看是否安装成功

- 接下来配置一下kubeconfig，将配置保存到`/.kube/config`

  配置如下

  ```yaml
  apiVersion: v1
  kind: Config
  clusters:
  - name: "mycluster"
    cluster:
      server: "https://xxx"
      certificate-authority-data: "xxx"
  - name: "mycluster2"
    cluster:
      server: "https://xxx"
      certificate-authority-data: "xxx"
      
  users:
  - name: "mycluster"
    user:
      token: "xxxx"
  
  contexts:
  - name: "mycluster"
  	context:
  		user: "mycluster"
  		cluster: "mycluster"
  - name: "mycluster2"
    context:
      user: "mycluster"
      cluster: "mycluster2"
  
  current-context: "mycluster"
  ```

- 创建一个配置文件

  ```sh
  mkdir kubectl
  cd kubectl
  vi config
  ```

  再把上面的内容贴近去

- 将配置文件加入到环境变量中

  ```sh
  vi ~/.bash_profile
  
  #加入这么一行
  KUBECONFIG=/home/xxx/kubectl/config
  #并在path中加入:$KUBECONFIG
  PATH=$PATH:$HOME/.local/bin:$HOME/bin:$KUBECONFIG
  export KUBECONFIG
  
  source ~/.bash_profilekubelet作用
  ```

## kubectl命令

```sh
kubectl get pods -n myweb
```

查看命名空间为`myweb`的所有pod

```sh
kubectl get node --show-labels
```

显示node和node对应的标签

```sh
kubectl label nodes <node-name> <label-key>=<label-value>
```

为某个节点打上标签



## kubelet

- Node管理
- pod管理
- 容器健康检查
- 容器监控
- 资源清理
- 和容器运行时交互（docker、rkt、Virtlet等等）

kubelet会暴露端口`10250`来和apiserver交互

GET

/pods

/stats/summary

/metrics

/healthz

如何使用kubelet来调用呢？

```sh
docker exec -it kubelet curl -k https://localhost:10250/pods --header "Authorization:Bearer kubeconfig-user-mtx....."
```

这个请求头部分就是在我们的kubeconfig的token里面，就知道是哪个用户在使用了。

## kube-proxy

外部通过NodePort、ClusterIP等方式访问服务。

kube-proxy运行在每个Node上面，负责Pod网络代理，维护网络规则和四层负载均衡工作

```sh
kubectl describe svc mygo -n myweb
```

可以通过这个命令，看到这个服务对应的ip，使用的Endpoints等。

## kube-controller-manager

kube-controller-manager负责节点管理、pod复制和endpoint创建。

监控集群中各种资源的状态使之和定义的状态保持一致。



# Helm3

helm是k8s的包管理器，好比linux里的yum、apt等

helm就是用来解决资源整合问题

从github上下载下来，解压缩helm，然后放入/user/local/bin目录下面即可

输入`helm version`可以看到这个和kubelet使用同样的kubeconfig

```sh
helm list -n myweb
```

查看helm列表

```sh
helm uninstall my
```

卸载helm安装的内容

## helm打包自己的项目

首先创建一个项目

```sh
helm create <项目名>
```

然后可以看到有很多的配置内容，我们使用命令

```sh
helm install abc <项目名> --dry-run --debug
helm template my mygin
```

就可以看到helm模版内容解析出来，不会执行的。

## helm模版

`values.yaml`

比方说我们在values.yaml中加入下面内容

```yaml
container:
	command: "[\"/app/myserver\"]"
```

然后在deployment.yaml中需要使用的地方输入`{{ .Values.container.command }}`

下面这种方式是为了判断是否为空，为空就直接不显示

```yaml
{{ if .Values.container.command }}
command: {{ .Values.container.command }}
{{ end }}
```

或者像下面这样，（如果有，就设置一下，不然就不要了，这个横杠是去掉多余的空行）

```yaml
{{- with .Values.container.command }}
command: {{ . }}
{{- end }}
```

如果有多行的，就需要这么写，比方说我们针对volumeMounts

```yaml
{{- with .Values.container.volumeMounts }}
volumeMounts: {{- toYaml . | nindent 12 }}
{{- end }}
```

n表示换行，indent 12表示打了12个空格

# Ingress

- 提供外部可访问的URL
- 七层负载均衡
- SSL等

以nginx-ingress为例

1. 与Kubernetes API交互，动态获取Ingress规则变化
2. 根据规则生成Nginx配置，写入运行nginx服务的pod里
3. 控制器获取配置，生成nginx.conf文件
4. 变化后reload

## 使用helm安装nginx-ingress

- 加入repo

  ```sh
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  ```

- 安装内容

  ```sh
  helm install my-nginx ingress-nginx/ingress-nginx
  ```

  如果没有科学上网，可能没法下载下来，就需要用别的方法来

  到hub.docker.com里找，找ingress-nginx-controller

  然后例如下面的方式

  ```sh
  docker pull giantswarm/ingress-nginx-controller:v0.40.2
  # 修改tag
  docker tag giantswarm/ingress-nginx-controller:v0.40.2 k8s.gcr.io/ingress-nginx/controller:v0.40.2
  # 删除之前的tag
  docker rmi giantswarm/ingress-nginx-controller:v0.40.2
  ```

  接着下载chart到本地

  ```sh
  helm fetch ingress-nginx/ingress-nginx
  ```

  然后解压缩，修改values中的内容，主要是把验证的信息给注释掉

  最后进行安装

  ```sh
  helm install my-nginx ingress-nginx -n my-nginx-ingress
  ```

- 更新命令

  ```sh
  helm upgrade my-nginx ingress-nginx -n my-nginx-ingress
  ```

DaemonSet用于在每个Kubernetes节点中将守护进程的副本作为后台进程运行。

作为nginx-ingress需要使用这种方式。

去values.yaml中做修改

- hostNetwork: true

- hostPort:

  ​	enabled: true

- Kind: DaemonSet

- Service:

  ​	type:  NodePort

## 发布一个负载均衡服务

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
	name: ingress-myserviceb
	annotations:
		kubernetes.io/ingress.class: "nginx"
spec:
	rules:
		- host: xxx.xxx.com
			http:
				paths:
					- path: /
						backend:
							serviceName: web1
							servicePort: 80
```

## 设置基本的路径重新写

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
	name: ingress-myserviceb
	annotations:
		kubernetes.io/ingress.class: "nginx"
		nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
	rules:
		- host: xxx.xxx.com
			http:
				paths:
					- path: /v1/(.*)
						backend:
							serviceName: web1
							servicePort: 80
```

设置Tcp反向代理

直接在values.yaml中，修改tcp的内容就可以了。

```yaml
tcp: {
	32379: "myweb/etcd1:2379"
}
```

# kubeadm

## kubeadm安装

首先修改repo

```sh
cat << EOF > /etc/yum.repo.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors,aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
	https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum makecache
```

安装

```sh
sudo yum -y install kubelet-1.18.6 kubeadm-1.18.6 kubectl-1.18.6

# 执行下面这个命令看一下列表
rpm -aq kubelet kubectl kubeadm

# 设置kubelet为开机启动
sudo systemctl enable kubelet
```

## 初始化集群

首先要设置docker配置

开启docker服务

```sh
systemctl enable docker.service
```

使用systemd作为docker的cgroup driver

```sh
sudo vi /etc/docker/daemon.json

# 加入以下内容
{
	"exec-opts":["native.cgroupdriver=systemd"]
}
```

重启docker，查看是否使用systemd作为cgroup的driver

```sh
systemctl daemon-reload && systemctl restart docker

# 执行这个命令，看看出来的值是否是systemd
docker info | grep Cgroup
```

关闭swap

```sh
# 暂时关闭swap
swapoff -a

# 永久关闭swap
sudo vi /etc/fstab
#注释掉 UUID=xxxx
```

然后在我们的master节点上执行初始化集群

```sh
sudo kubeadm init -kubernetes-version=v1.18.6 --image-repository registry.aliyuncs.com/google_containers
```

执行完后会给我们如何加入这个集群的方式

关于token

```sh
# 查看token列表
sudo kubeadm token list
# 重新生成token
sudo kubeadm token create --print-join-command
# 其他节点加入
sudo kubeadm join 192.168.0.53:6443 --token yd38.....\
--discovery-token-ca-cert-hash sha256:2ef....
```

配置一下kube

```sh
# 1、首先新建一个目录
mkdir -p $HOME/.kube
# 2、复制相关文件进来
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
# 3、设置相关权限
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

这样以后，我们就可以执行kubectl了，直接使用`kubectl cluster-info`查看集群相关信息

## 安装网络组件

centos 7有点问题，需要设置一下

```sh
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

在master上执行如下程序

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 验证一下，发现有点问题
kubectl get pods --all-namespaces
```

这是因为我们使用的网段不同，在master节点上，之前没有设置过网段，所以需要设置一下k8s的网段

```sh
sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml

# 在command节点 加入 
- --allocate-node-cidrs=true
- --cluster-cidr=10.244.0.0/16

# 然后执行 
systemctl restart kubelet
```

## 加入子节点

如果kubeadm有问题可以用`kubeadm reset`来解决，主要针对之前执行过没有删除干净的情况。

```sh
# 1、执行iptables开启
sudo sysctl net.bridge.bridge-nf-call-iptables=1

# 2、加入子节点
sudo kubeadm join 192.168.0.53:6443 --token yd38.....\
--discovery-token-ca-cert-hash sha256:2ef....
```

## 部署rancher作为管理系统

1、下载docker并运行起来

```sh
sudo docker run -d --privileged --restart=unless-stopped -p 8080:80 -p 8443:443 -v /home/shenyi/rancher:/var/lib/rancher/ rancher/rancher:stable
```

2、去master节点上做个设置

```sh
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user kubernetes-admin
```

最后的用户名要和kubeconfig中的一样

然后按照rancher端的curl命令直接敲进来

```sh
curl --insecure -sfL https://192.168.0.106:8443/v3/import/pclwnv429cf4fznp74vfrblsr9w9f4fdvv6gckmtzh6k4tslls66g5.yaml | kubectl apply -f -
```

3、修改master节点上的kube-scheduler.yaml和kube-controller-manager.yaml

把--port=0删掉

然后`systemctl restart kubelet`

# kubernetes的授权和认证机制

配置文件在`~/.kube/config`

kubernets-admin拥有全部权限

主要会分为User Count和Service Count

User Count用于外部访问，Service Count用于内部的之间的通信

使用默认方式，使用证书

装好了openssl

```sh
sudo yum install openssl openssl-devel
```

创建一个文件夹叫ua/cl

进入到文件夹中后，使用下面的命令生成一个密钥文件

```sh
openssl genrsa -out client,key 2048
```

接着生成一个证书请求文件csr(/CN指定了用户名cl)

```sh
opensll req -new -key client.key -out client.csr -subj "/CN=shenyi"
```

接着根据k8s的CA证书生成我们用户的客户端证书

```sh
sudo openssl x509 -req -in client.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out client.crt -days 365
```

可以使用下面的命令看一下我们的endpoints到底是啥

```sh
kubectl get endpoints
```

接着使用curl来请求这个内容

```sh
curl https://192.168.0.53:6443/api
```

这样是肯定请求不了的

所以要带上证书的请求来，其实可以用`--insecure` 代替 `--cacert /etc/kubernetes/pki/ca.crt`从而忽略服务端证书验证

```sh
curl --cert ./client.crt --key ./client.key --cacert /etc/kubernetes/pki/ca.crt -s https://192.168.0.53:6443/api
```

假如我们忘记了之前设置的`/CN=cl`，忘记了CN后面跟的内容，我们可以使用下面的命令

```sh
openssl x509 -noout -subject -in client.crt
```

接下来将client.crt加入到~/.kube/config，执行kubectl命令时切换成我们的用户

```sh
kubectl config --kubeconfig=/home/shenyi/.kube/config set-credentials shenyi --client-certificate=/home/shenyi/ua/shenyi/client.crt --client-key=/home/shenyi/ua/shenyi/client.key
```

这一步把用户设置进去了

接下来我们要创建一个上下文

```sh
kubectl config --kubeconfig=/home/shenyi/.kube/config set-context user_context --cluster=kubernetes --user=shenyi
```

指定当前上下文

```sh
kubectl config --kubeconfig=/home/shenyi/.kube/config use-context user_context
```

## 角色与角色绑定

把角色切回管理员

```sh
kubectl config use-context kubernetes-admin@kubernetes
```

创建一个可以查看pod的角色

```sh
vim mypod_role.yaml
```

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: mypod
rules:
- apiGroups: ["*"]
  resources: ["pods"]
  verbs: ["get","watch","list"]
```

关于资源，可以用这个命令查看

```sh
kubectl api-resources -o wide
```

执行一下这个yaml

```sh
kubectl apply -f mypod_role.yaml
# 然后可以查看这个角色
kubectl get role -n default
kubectl describe role mypod
```

- 第一种方式，命令行的方式来绑定

  ```sh
  kubectl create rolebinding mypodbinding -n default --role mypod --user shenyi
  ```

  删除

  ```sh
  kubectl delete rolebinding mypodbinding
  ```

- 第二种方式，配置文件mypod_rolebinding.yaml

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    creationTimestamp: null
    name: mypodrolebinding
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: mypod
  subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: shenyi
  ```

### ClusterRole

需要管理多个namespace，就要使用到clusterrole，绑定既可以使用RoleBinding，也可以使用ClusterRoleBinding。

创建一个clusterrole的yaml文件

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mypod-cluster
rules:
- apiGroups: ["*"]
  resources: ["pods"]
  verbs: ["get","watch","list"]
```

如果使用rolebinding，也是需要说明命名空间的(针对某个命名空间)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mypodrolebinding-cluster
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mypod-cluster
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: shenyi
```

删除命令

```sh
kubectl delete rolebinding mypodrolebinding-cluster -n kube-system
```

下面创建一个clusterrolebinding的yaml文件，这样子绑定的话，就不需要说明命名空间，对于所有命名空间都有效。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mypod-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mypod-cluster
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: shenyi
```

### 配置token的方式请求api

生成一个token

```sh
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
# 把刚刚生成的token贴下来
kubectl config set-credentials shenyi --token=xxxx 
```

然后看`./kube/config`里面找到使用了用户名下面有了token

然后使用下面的命令来访问

```sh
curl -H "Authorization: Bearer xxxx" https://192.168.0.53:6443/api/v1/namespaces/default/pods --insecure
```

但是这里的api-server不支持这种方式，所以我们要修改一下api-server的启动参数

首先创建一个文件

```sh
sudo vi /etc/kubernetes/pki/token_auth
```

填入之前生成的token内容,shenyi,1001

然后修改api-server启动参数

```sh
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

加入`--token-auth-file=/etc/kubernetes/pki/token_auth`

## ServiceAccount

### 创建

ServiceAccount主要用于pod与api-server之间的通信

查看serviceaccount

```sh
kubectl get sa
```

用命令创建一个sa

```sh
kubectl create serviceaccount mysa
```

每个namespace都会有一个默认的default账号，且sa局限在自己所属的namespace中。而UserAccount是可以跨ns的。k8s会在secrets中保存token。

第二种方式创建：

```sh
kubectl create serviceaccount mysa -o yaml --dry-run=client
kubectl create serviceaccount mysa -o yaml --dry-run=client > mysa.yaml
# 应用一下这个yaml
kubectl apply -f mysa.yaml
```

### 赋予权限、外部访问API

```sh
kubectl create clusterrolebinding mysa-crb --clusterrole=mypod-cluster --serviceaccount=default:mysa
```

装一个工具叫jq，是一个轻量级的json处理命令

```sh
sudo yum install jq -y
```

获取需要的内容`-M` 去掉颜色 `-r`去掉引号

```sh
kubectl get sa mysa -o json | jq -Mr'.secrets[0].name'
```

得到了一个secret的内容

然后使用下面的命令

```sh
kubectl get sercret mysa-token-6tggr(上面命令获取的) -o json | jq -Mr '.data.token' | base64 -d
```

合起来获取这个变量

```sh
mysatoken=$(kubectl get secret $(kubectl get sa mysa -o json | jq -Mr '.secrets[0].name') -o json | jq -Mr '.data.token' | base64 -d)
```

接下来请求方式就是

```sh
curl -H "Authorization: Bearer $mysatoken" --insecure https://192.168.0.53:6443/api/v1/namesapces/default/pods
```

### 在pod里访问k8s API（token的方式）

创建一个nginx的容器，进入pod。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginxtest
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80

```

查看几个比较重要的环境变量

```sh
# 对应的是内网地址
echo $KUBERNETES_SERVICE_HOST
# 端口
echo $KUBERNETES_PORT_443_TCP_PORT
# TOKEN
TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`
# APISERVER
APISERVER="https://$KUBERNETES_SERICE_HOST:$KUBERNETES_PORT_443_TCP_PORT"
```

发起请求

```sh
curl --header "Authorization: Bearer $TOKEN" --insecure -s $APISERVER/api
# 但是下面这个没有权限
curl --header "Authorization: Bearer $TOKEN" --insecure -s $APISERVER/api/v1/namespaces/default/pods 
```

下面给我们这个pod使用之前创建的mysa角色，需要在之前的yaml中做修改

```yaml
 spec:
      serviceAccountName: mysa
      containers:
        - name: nginxtest
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

### 在pod里访问k8s API（token + 证书的方式）

```sh
curl --header "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt $APISERVER/api/v1/namespaces/default/pods
```

# Pod Deployment

pod中有pause容器

它的作用：

- 扮演Pid=1的，回收僵尸进程
- 基于Linux的namespace的共享

创建一个pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myngx
spec:
  containers:
  - name: ngx
    image: "nginx:1.18-alpine"
```

### 创建一个多容器的pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myngx
spec:
  containers:
  - name: ngx
    image: "nginx:1.18-alpine"
  - name: "alpine"
    command: ["sh","-c","echo this is second && sleep 3600"]
    image: "apline:3.12"
```

### 配置数据卷

```sh
apiVersion: v1
kind: Pod
metadata:
  name: myngx
spec:
  containers:
  - name: ngx
    image: "nginx:1.18-alpine"
    volumeMounts:
    - name: mydata
      mountPath: /data
  - name: "alpine"
    command: ["sh","-c","echo this is second && sleep 360000"]
    image: "apline:3.12"
  volumes:
  - name: mydata
    hostPath:
      path: /home/shenyi/data
      type: Directory
```

进入到容器

```sh
kubectl exex -it myngx -c ngx -- sh
```

注意有多个容器要选择进入哪个容器

### 创建deployment

Deployment通过副本集管理和创建Pod。

`emptyDir`常用于临时空间、多容器共享

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: sharedata
              mountPath: /data
        - name: apline
          image: apline:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","echo this is alpine && sleep 360000"]
          volumeMounts:
            - name: sharedata
              mountPath: /data
      volumes:
        - name: sharedata
          emptyDir: {}
```

在容器中安装curl

```sh
apk add curl
```

### init容器

init容器是一种特殊容器，总是运行到完成，每个都必须在下一个启动之前完成。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: sharedata
              mountPath: /data
      initContainers:
        - name: apline
          image: apline:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","wait for db && sleep 3600"]
          volumeMounts:
            - name: sharedata
              mountPath: /data
      volumes:
        - name: sharedata
          emptyDir: {}
```

# ConfigMap

### ConfigMap创建、环境变量引用

- 容器entrypoint的命令行参数
- 容器的环境变量
- 映射成文件
- 编写代码在pod中运行，使用Kubernetes API来读取ConfigMap

创建一个cm

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mycm
data:
  # 每一个键对应一个简单的值，以字符串的形式体现
  username: "shenyi"
  userage: "19"
  user.info: |
    name=shenyi
    age=19
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          env:
            - name: TEST
              value: testvalue
            - name: USERNAME
              valueFrom:
                configMapKeyRef:
                  name: mycm   # ConfigMap的名称
                  key: username   # 需要取值的键
```

### ConfigMap映射成文件

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: cmdata
              mountPath: /data
          env:
            - name: TEST
              value: testvalue
            - name: USERNAME
              valueFrom:
                configMapKeyRef:
                  name: mycm   # ConfigMap的名称
                  key: username   # 需要取值的键
     volumes:
       - name: cmdata
       configMap:
         name: mycm
         items:
           - key: user.info
             path: user.txt
```

### ConfigMap全部映射文件和subpath

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: cmdata
              mountPath: /data/uesr.txt
              subPath: user.info
          env:
            - name: TEST
              value: testvalue
            - name: USERNAME
              valueFrom:
                configMapKeyRef:
                  name: mycm   # ConfigMap的名称
                  key: username   # 需要取值的键
     volumes:
       - name: cmdata
       configMap:
         defaultMode: 0655
         name: mycm
         # items:
         #  - key: user.info
         #    path: user.txt
```

### ConfigMap:用程序读（体外）

体外：kubernetes外部

开启api代理

```sh
kubectl proxy --address='0.0.0.0' --accept-hosts='^*$' --port=8009
```

下载客户端库

```sh
go get k8s.io/client-go@v0.18.6
```

初始化客户端

```go
func getClient() *kubernetes.Clientset{
	config:=&rest.Config{
		Host:"http://124.70.204.12:8009”,    //IP自行改掉
 	}
	c,err:=kubernetes.NewForConfig(config)
	if err!=nil{
		log.Fatal(err)
	}
	return c
}
```

```go
func main(){
  cm , err := getClient().CoreV1().ConfigMaps("default").Get(context.Background(),"mycm",v1.GetOptions{})
  if err != nil{
    log.Fatal(err)
  }
  fmt.Println(cm.Data)
}
```

### ConfigMap:用程序读（体内）

```go
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"log"
	"os"
)
var api_server string
var token string
func init() {
	api_server=fmt.Sprintf("https://%s:%s",
		os.Getenv("KUBERNETES_SERVICE_HOST"),os.Getenv("KUBERNETES_PORT_443_TCP_PORT"))
     f,err:=os.Open("/var/run/secrets/kubernetes.io/serviceaccount/token")
     if err!=nil{
     	log.Fatal(err)
	 }
     b,_:=ioutil.ReadAll(f)
     token=string(b)
}
func getClient() *kubernetes.Clientset{
	config:=&rest.Config{
		//Host:"http://124.70.204.12:8009",
		Host:api_server,
		BearerToken:token,
		TLSClientConfig:rest.TLSClientConfig{CAFile:"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"},
 	}
	c,err:=kubernetes.NewForConfig(config)
	if err!=nil{
		log.Fatal(err)
	}

	return c
}
func main() {
       cm,err:=getClient().CoreV1().ConfigMaps("default").
       	Get(context.Background(),"mycm",v1.GetOptions{})
       if err!=nil{
       	log.Fatal(err)
	   }
       for k,v:=range cm.Data{
       	   fmt.Printf("key=%s,value=%s\n",k,v)
	   }
	  select {}
}
```

创建一个账号

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cmuser
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cmrole
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cmclusterrolebinding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cmrole
subjects:
  - kind: ServiceAccount
    name: cmuser
    namespace: default
```

交叉编译

```sh
set GOOS=linux
set GOARCH=amd64
go build -o cmtest cm.go
chmod +x cmtest
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cdmtest
spec:
  selector:
    matchLabels:
      app: cmtest
  replicas: 1
  template:
    metadata:
      labels:
        app: cmtest
    spec:
      serviceAccount: cmuser
      nodeName: jtthink2
      containers:
        - name: cmtest
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/cmtest"]
          volumeMounts:
            - name: app
              mountPath: /app
      volumes:
        - name: app
          hostPath:
            path: /home/shenyi/goapi
            type: Directory
```

### ConfigMap:调用API监控cm的变化

```go
package main

import (
	"k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"log"
)
//var api_server string
//var token string
//func init() {
//	api_server=fmt.Sprintf("https://%s:%s",
//		os.Getenv("KUBERNETES_SERVICE_HOST"),os.Getenv("KUBERNETES_PORT_443_TCP_PORT"))
//    f,err:=os.Open("/var/run/secrets/kubernetes.io/serviceaccount/token")
//    if err!=nil{
//    	log.Fatal(err)
//	 }
//    b,_:=ioutil.ReadAll(f)
//    token=string(b)
//}
func getClient() *kubernetes.Clientset{
	config:=&rest.Config{
		Host:"http://124.70.204.12:8009",
		//Host:api_server,
		//BearerToken:token,
		//TLSClientConfig:rest.TLSClientConfig{CAFile:"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"},
 	}
	c,err:=kubernetes.NewForConfig(config)
	if err!=nil{
		log.Fatal(err)
	}

	return c
}
type CmHandler struct{}
func(this *CmHandler) OnAdd(obj interface{}){}
func(this *CmHandler) OnUpdate(oldObj, newObj interface{}){
     if newObj.(*v1.ConfigMap).Name=="mycm"{
     	log.Println("mycm发生了变化")
	 }
}
func(this *CmHandler)	OnDelete(obj interface{}){}

func main() {
       //cm,err:=getClient().CoreV1().ConfigMaps("default").
       //	Get(context.Background(),"mycm",v1.GetOptions{})
       //if err!=nil{
       //	log.Fatal(err)
	   //}
       //for k,v:=range cm.Data{
       //	   fmt.Printf("key=%s,value=%s\n",k,v)
	   //}

	fact:=informers.NewSharedInformerFactory(getClient(), 0)

	cmInformer:=fact.Core().V1().ConfigMaps()
	cmInformer.Informer().AddEventHandler(&CmHandler{})

	fact.Start(wait.NeverStop)
	select {}

}
```

# Secret

Secret用来保存敏感信息，例如密码、OAuth令牌和SSH密钥。

创建一个secret，填写的内容要用base64加密才能apply

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: "c2hlbnlpCg=="
  pass: "MTIzCg=="
```

如果要填写明文，就要写StringData

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
StringData:
  user: "zhangsan"
  pass: "123456"
```

反解base64加密

```sh 
echo -n xxxx | base64 -d
```

## 命令获取secret内容，挂载文件

```sh
kubectl get secret mysecret -o yaml
kubectl get secret mysecret -o json
```

假如我们要获取整个yaml中的kind是什么就可以使用下面命令

```sh
kubectl get secret mysecret -o jsonpath={.kind}

# 同样的，要获取data的话
kubectl get secret mysecret -o jsonpath={.data}

# 获取名称的明文
kubectl get secret mysecret -o jsonpath={.data.user} | base64 -d
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: cmdata
              mountPath: /data/uesr.txt
              subPath: user.info
            - name: users
              mountPath: /users
          env:
            - name: TEST
              value: testvalue
            - name: USERNAME
              valueFrom:
                configMapKeyRef:
                  name: mycm   # ConfigMap的名称
                  key: username   # 需要取值的键
            - name: USER
              valueFrom:
                secretKeyRef:
                  name: mysecret  
                  key: user
     volumes:
       - name: users
       secret:
         defaultMode: 0655
         secretName: mysecret
       - name: cmdata
       configMap:
         defaultMode: 0655
         name: mycm
         # items:
         #  - key: user.info
         #    path: user.txt
```

## secret进行basic-auth认证

1、手工配置

安装Apache的Web服务器内置的工具

```sh
sudo yum -y install httpd-tools
```

创建一个密码文件

```sh
htppasswd -c auth shenyi
```

然后输入密码`123456`

然后会生成一个文件auth，cat一下可以看到明文的账号加加密的密码

如果还要再添加一个用户，注意，这里不用-C，否则会覆盖之前的内容

```sh
htpasswd auth lisi
```

然后输入用户的密码

然后在default命名空间创建一个bauth的configmap，配置键为`auth`，值为刚刚生成的内容。

使用的镜像是nginx:1.18-alpine

它的默认配置文件中`/etc/nginx/nginx.conf`

其中主配置文件引用了`/etc/nginx/conf.d/default.conf`文件

然后再创建一个configmap叫`nginxconf`

创建键值对，键`ngx`

值为

```yaml
server {
    listen       80;
    server_name  localhost;
    location / {
       auth_basic      "test auth";
       auth_basic_user_file /etc/nginx/basicauth;
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nginxconf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: ngx
            - name: basicauth
              mountPath: /etc/nginx/basicauth
              subPath: auth
      volumes:
        - name: nginxconf
          configMap:
            defaultMode: 0655
            name: nginxconf
        - name: basicauth
          configMap:
            defaultMode: 0655
            name: bauth
```

2、使用secret挂载

用文件的方式导入

```sh
kubectl create secret generic secret-basic-auth --from-file=auth
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nginxconf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: ngx
            - name: basicauth
              mountPath: /etc/nginx/basicauth
              subPath: auth
      volumes:
        - name: nginxconf
          configMap:
            defaultMode: 0644
            name: nginxconf
        - name: basicauth
          secret:
            defaultMode: 0644
            name: secret-basic-auth
```

## 拉取私有镜像、创建Docker Secret

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myalpine
spec:
  selector:
    matchLabels:
      app: myalpine
  replicas: 1
  template:
    metadata:
      labels:
        app: myalpine
    spec:
      containers:
        - name: alpine
          image: shenyisyn/myalpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","echo this is alpine && sleep 36000"]
      imagePullSecrets:
        - name: dockerreg
```

然后创建一个docker的secret，下面的内容需要修改为自己的

```sh
kubectl create secret docker-registry dockerreg \
  --docker-server=https://index.docker.io/v1/\
  --docker-username=shenyisyn \
  --docker-password=xxxxx \
  --docker-email=65480539@qq.com
```

使用下面的命令可以看到明文

```sh
kubectl get secret dockerreg -o jsonpath={.data.*} | base64 -d
```

# Sevice

## 创建一个基本的service

先创建一个configmap

```yaml
apiVersion: v1
data:
  h1: this is h1
  h2: this is h2
kind: ConfigMap
metadata:
  name: html
```

然后创建pod和service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx1
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: htmldata
              mountPath: /usr/share/nginx/html/index.html
              subPath: h1
          ports:
            - containerPort: 80
      volumes:
        - name: htmldata
          configMap:
             defaultMode: 0644
             name: html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:  #service通过selector和pod建立关联
    app: nginx
```

## Service负载均衡多个Pod

可以看到，这个service是通过标签来匹配的。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx1
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: htmldata
              mountPath: /usr/share/nginx/html/index.html
              subPath: h1
          ports:
            - containerPort: 80
      volumes:
        - name: htmldata
          configMap:
             defaultMode: 0644
             name: html
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx2
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx2
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: htmldata
              mountPath: /usr/share/nginx/html/index.html
              subPath: h2
          ports:
            - containerPort: 80
      volumes:
        - name: htmldata
          configMap:
            defaultMode: 0644
            name: html
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:  #service通过selector和pod建立关联
    app: nginx

```

## 宿主机访问k8s的Service的基本方法

安装一个工具

```sh
sudo yum install bind-utils -y
```

然后执行

```sh
nslookup nginx-svc
```

`/etc/resolv.conf`用于设置DNS服务器IP地址、DNS域名和设置主机的域名搜索顺序

我们执行下面命令

```sh
kubectl get svc -n kube-system
```

获得kube-system的clusterIp

然后在`/etc/resolv.conf`中加入

`nameserver`刚刚kube-system的clusterip地址

然后执行下面命令

```sh
nslookup nginx-svc.default.svc.cluster.local
```

这样操作比较麻烦，我们可以在`/etc/resolv.conf`里面加入`search default.svc.cluster.local svc.cluster.local`

然后就可以直接使用curl来访问service了

## 无头Service入门

Headless -> 即没有ip

创建的时候就是将clusterip设置为none

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  clusterIP: None
  ports:
    - port: 80
      targetPort: 80
  selector:  #service通过selector和pod建立关联
    app: nginx
```

这个时候再使用`nslookup`的话会出来两个ip，其实这两个就是对应两个pod的ip地址。

- 有些服务需要自己来决定使用哪个IP
- StatefulSet状态下，pod之间的互相访问

## kube-proxy，修改为ipvs模式

可以看一下kube-proxy

```sh
sudo netstat -ntlp | grep kube-proxy
```

kube-proxy监听10249和10256端口：对外提供/metrics和/healthz的访问

kube-proxy的配置在kube-system的configmap里面

有两个主要模式iptables和IPVS

iptables是默认的，基于linux内核，成熟稳定，当我们的service和endpoint比较少，比如千百个，iptables性能较高。

ipvs使用优化的查找算法，而不是从列表中找规则，在大规模场景下要改成ipvs

修改方法：

1、在kube-system的ns里面，找到kube-proxy的configmap，设置mode为ipvs。

2、然后删掉工作负载，等deployment重启两个pod，然后看pod的日志，是否切换为ipvs。

# PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce # 读写方式
  persistentVolumeReclaimPolicy: Delete
  local:
    path: /home/shenyi/data
  nodeAffinity: # 如果有local，要使用节点亲和性，保证有这个path。
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: pv
              operator: In
              values:
                - local
```

查看当前k8s节点的node

```sh
kubectl get node --show-labels=true
```

给node2打一个自定义标签

```sh
kubectl label nodes node2 pv=local
```

删除标签

```sh
kubectl label nodes node2 pv-
```

查看这个pv

```sh
kubectl get pv
```

## 创建PVC、初步绑定PV、POD挂载

PVC描述需要的存储标准，然后从现有PV中匹配或者动态创建新的资源，最后将两者进行绑定。

PV提供者，提供存储方式和容量。PVC是消费者，消费容量（它不需要关注用什么技术实现，绑定即可）

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce # 读写方式
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/shenyi/data
  nodeAffinity: # 如果有local，要使用节点亲和性，保证有这个path。
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: pv
              operator: In
              values:
                - local
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ngx-pvc
spec:
 accessModes:
   - ReadWriteOnce
 storageClassName: ""
 resources:
   requests:
     storage: 1Gi
```

绑定：

- spec关键字段要匹配
- storageClassName字段必须一致

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx1
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: mydata
              mountPath: /data
          ports:
            - containerPort: 80
      volumes:
        - name: mydata
          persistentVolumeClaim:
            claimName: ngx-pvc
```

## StorageClass

理解为创建PV的模版

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: Local
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

这里的`volumeBindingMode`有两种，`WaitForFirstConsumer`是有pod创建的时候才开始绑定，`Immediate`是StorageClass一创建就绑定好PV和PVC。

# POD自动伸缩（HPA）

- 基于CPU利用率自动扩缩ReplicationController、Deployment、ReplicaSet和StatefulSet的pod数量。
- 基于其他应用程序提供的自定义度量指标来执行自动扩缩。
- Pod自动扩缩不适用于无法扩缩的对象，比如DaemonSet。

安装metrics-server

```sh
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

```sh
# 可以查看CPU使用率等等
kubectl top node
```

## 限制Pod资源、创建HPA

```yaml
resources:
  request:
    cpu: "200m"
    memory: "256Mi"
  limits:
  	cpu: "400m"
  	memory: "512Mi"
```

requests来设置各容器需要的最小资源

limits用于限制运行时容器占用的资源

```go
// stress.go
package main

import (
	"encoding/json"
	"github.com/gin-gonic/gin"

)

func main() {
	test:=map[string]string{
		"str":"requests来设置各容器需要的最小资源",
	}
  r:=gin.New()
  r.GET("/", func(context *gin.Context) {
  	ret:=0
  	for i:=0;i<=1000000;i++{
  		t:=map[string]string{}
  		b,_:=json.Marshal(test)
  		_=json.Unmarshal(b,t)
  		ret++
	}
	  context.JSON(200,gin.H{"message":ret})
  })
  r.Run(":8080")
}

```

```yaml
# ngx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web1
spec:
  selector:
    matchLabels:
      app: myweb
  replicas: 1
  template:
    metadata:
      labels:
        app: myweb
    spec:
      nodeName: jtthink2
      containers:
        - name: web1test
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/stress"]
          volumeMounts:
            - name: app
              mountPath: /app
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "400m"
              memory: "512Mi"
          ports:
            - containerPort: 8080
      volumes:
        - name: app
          hostPath:
            path: /home/shenyi/goapi

---
apiVersion: v1
kind: Service
metadata:
  name: web1
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:  #service通过selector和pod建立关联
    app: myweb
```

创建一个自动扩容，超过百分之20开始扩容，最小是1，最大是5

```sh
kubectl autoscale deployment web1 --min=1 --max=5 --cpu-percent=20
```

然后执行`kubectl get hpa`可以看到扩容的内容

接着装一个压测工具

```sh
sudo yum -y install httpd-tools
```

使用下面的命令

```sh
ab -n 10000 -c 10 http://web1/
```

然后可以看到hpa开始扩容了，变成了5个，因为我们设置了最大的是5个

## yaml的方式创建HPA

有好几个API版本

- autoscaling/v1 只支持通过cpu伸缩
- autoscaling/v2beta1 支持通过cpu、内存和自定义数据来进行伸缩
- autoscaling/v2beta2 beta1的进一步（一般用它）

可以使用命令来查看支持的版本

```sh
kubectl api-versions | grep autoscaling
```

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: web1hpa
  namespace: default
spec:
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web1
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization #AverageValue 这个是使用量
          averageUtilization: 50   #使用率
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 50   #使用率
```

```sh
kubectl api-resources # 可以获取kubernetes中所有的资源内容
```

# CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: proxies.extensions.jtthink.com
spec:
  # 分组名，在REST API中也会用到的，格式是: /apis/分组名/CRD版本
  group: extensions.jtthink.com
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
                name:
                  type: string
                age:
                  type: integer
  # 范围是属于namespace的 ,可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 复数名
    plural: proxies
    # 单数名
    singular: proxy
    # 类型名
    kind: MyRoute
    # kind的简称，就像service的简称是svc
    shortNames:
      - mr
```

然后就可以创一个crd的对象了

```yaml
apiVersion: extensions.jtthink.com/v1
kind: MyRoute
metadata:
  name: mygw
spec:
  name: "shenyi"
  age: 19
```

## 控制器

控制器会监视资源的创建/更新/删除事件，并出发Reconcile函数作为响应。整个调整过程被称作“Reconcile Loop”或者“Sync Loop”

# 调度器

粗暴的pod调度流程

- 通过某个方法发布pod（一堆配置）
- ControllerManager会把pod加入待调度队列
- scheduler开始干活，通过机制决定到底要调度哪个node。然后写入etcd
- 被选上节点里的kubelet得到消息，然后干活（pull image这些，正式启动pod）

过滤、打分

## NodeAffinity

实现了节点选择器和节点亲和性。

```sh
# 现实节点标签
kubectl get nodes --show-labels
```

给某个节点加上标签

```sh
kubectl label nodes node1 app1=ngx
```

然后在yaml中加入

```yaml
spec:
  nodeSelector:
    app1:ngx
```

这样就能调度到node1上面了。

## 节点亲和性

这部分内容可以用到的时候去看文档

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
            - key: app
              operator: In
              values:
                - ngx
```

## 污点和容忍度

Taint（污点）排斥一类特定的pod

查看污点

```sh
kubectl describe node node1 | gerp Taints
```

- NoSchedule: 一定不能被调度
- PreferNoSchedule: 尽量不要调度
- NoExecute: 不仅不会调度，还会驱逐Node上已有的Pod

在节点上打污点

```sh
kubectl taint node node1 name=shenyi:NoSchedule
```

删除污点

```sh
kubectl taint node node1 name:NoSchedule-
```

给需要的Pod添加容忍度

```yaml
tolerations:
  - key: "name"
    operator: "Equal"
    value: "shenyi"
    effect: "NoSchedule"
```

## Pod亲和性

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - ngx1
          topologyKey: name
```

# k8s从1.18升级到1.20

首先删除rancher，到有rancher的那台主机

```sh
docker stop rancher的容器ID && docker rm rancher的容器ID
```

每台机器执行reset

```sh
sudo kubeadm reset
sudo rm /etc/cni/net.d -fr
# 清空iptables
sudo iptables -F
```

卸载之前的kubeadm

```sh
sudo yum -y remove kubelet-1.18.6 kubeadm-1.18.6 kubectl-1.18.6
```

安装1.20

```sh
sudo yum -y install kubelet-1.20.2 kubeadm-1.20.2 kubectl-1.20.2
# 允许数据包转发
echo 1 > /proc/sys/net/ipv4/ip_forward
modprobe br_netfilter
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables

# 设置kubelet为开机启动
systemctl daemon-reload
systemctl enable kubelet
```

初始化集群（主机执行）

```sh
kubeadm init --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --kubernetes-version=1.20.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12
```

执行完后获得的最后一个东西，复制到需要加入的机器，执行一下

接着来设置配置文件（在主机上）

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chowd $(id -u):$(id -g) $HOME/.kube/config
```

给工作节点打一个标签

```sh
kuebctl label node node2 node-role.kubernetes.io/node=node
```

## 快速安装flannel

去官网下载flanneld-v0.13.1-rc1-amd64.docker

然后执行指令

```sh
docker load < flanneld-v0.13.1-rc1-amd64.docker
```

然后apply flannel的yaml文件

```sh
# 来到子节点机器，执行
sudo ifconfig cni0 down

sudo ip link delete cni0


# 然后删除之前的POD 就可以了
kubectl delete pod coredns-54d67798b7-f7r6s coredns-54d67798b7-g2tr2 -n kube-system


# pod名称请自行根据自己的POD名称 修改
```



## 1.20导入rancher

安装rancher

```sh
docker run -itd --privileged  -p 9443:443 \
-v /home/shenyi/rancher:/var/lib/rancher \
--restart=unless-stopped  -e CATTLE_AGENT_IMAGE="registry.cn-hangzhou.aliyuncs.com/rancher/rancher-agent:v2.5.5"  registry.cn-hangzhou.aliyuncs.com/rancher/rancher:v2.5.5


# 确保/home/shenyi/rancher  这个目录已经存在
```

然后导入

```sh
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user kubernetes-admin
```

然后再执行rancher网页里面，不受信任的那个命令行即可。

