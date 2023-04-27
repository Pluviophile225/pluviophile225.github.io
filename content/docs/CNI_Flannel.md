## Demo的拓扑图

![image-20230426124304649](../assets/image-20230426124304649.png)

master1、worker1、worker2在网段10.0.0.0/24的网段里面，在Rack1里的leaf01下。worker3在网段30.0.0.0/24，在Rack2的leaf02下，两个leaf通过spine来连接。

红色这一块的就可以通过k8sNAT到笔记本，在直接出去。

绿色的是所有设备的网管。

## Documentation

### Configuration

1. A kube-flannel namespace with PodSecurity level set to privileged.

2. A ClusterRole and ClusterRoleBinding for Role Based Acccess Control (RBAC).

3. A service account for flannel to use.

4. A ConfigMap containing both a CNI configuration(cni-conf.json) and a flannel configuration(net-conf.json). 

  The “Network” in the flannel configuration should match the pod network CIDR. The choice of backend is also made here and defaults to VXLAN.

5. A DaemonSet to deploy the flannel pod on each Node. 

  The pod has three containers:

1) An initContainer(install-cni) for deploying the CNI configuration to a location that the kubelet can read: cp  -f  /etc/kube-flannel/cni-conf.json  /etc/cni/net.d/10-flannel.conflist
2) An initContainer(install-cni-plugin) for deploying the flannel binary to a location that the kubelet can use: cp  -f  /flannel  /opt/cni/bin/flannel
3) The flannel container (kube-flannel): /opt/bin/flanneld --ip-masq --kube-subnet-mgr

flannel这个是给kubelet使用的配置文件，会根据这个文件知道要调用flannel的bin文件，一般在opt/flannel/bin下面

```yaml
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
```

第二个配置文件是net-conf，这个是给DaemonSet的主容器用的

```yaml
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

最重要的就是这个DaemonSet了。flaneld跑在DaemonSet上，

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      # CNI在运行的时候，要创建路由、Veth peer的资源，所以一定要共享主机的网络
      hostNetwork: true
      priorityClassName: system-node-critical
      # 同时toleration也要能够忍受控制，也就是control plane的控制
      tolerations:
      - operator: Exists
        effect: NoSchedule
      # 跟api通信使用flannel这个servicecount，就是之前定义的
      serviceAccountName: flannel
      # 接下来就是两个init containers
      initContainers:
      # 第一个就是把init container的文件拷贝到主机目录下，就是flannel的二进制文件copy到主机的路径下去。kubelet在收到创建pod的请求的时候，就会调用这个二进制文件创建相关资源。
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
       # config-map 已经变成了一个配置文件，把这个文件copy到默认路径下去
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        # 跑了这个二进制文件
        - /opt/bin/flanneld
        args:
        # 加了两个参数
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

主容器的配置

```shell
--kube-subnet-mgr : Contact the Kubernetes API for subnet assignment instead of etcd 让flannel与Kubernetes API通信，而不是与etcd通信
--ip-masq=false: setup IP masquerade for traffic destined for outside the flannel network. Flannel assumes that the default policy is ACCEPT in the NAT POSTROUTING chain. pod在和k8s集群内部其他设备的通信的时候不用做nat，如果出去的话，就要做ip masqurade，因为flanel是一个overlay的方案，就要把自己的源地址给改变了。
--iface="": interface to use (IP or name) for inter-host communication. Defaults to the interface for the default route on the machine. This can be specified multiple times to check each option in order. Returns the first match found. 接口选择就是k8s默认网络的接口，ensp0s3
```

flaneld的配置文件

```shell
Network (string): IPv4 network in CIDR format to use for the entire flannel network. (Mandatory if EnableIPv4 is true)
他要跟k8s集群的pod CIDR保持一致
Backend (dictionary): Type of backend to use and specific configurations for that backend. The list of available backends and the keys that can be put into the this dictionary are listed in Backends. Defaults to vxlan backend.
不同的Backend就是不同的overlay的方案

```

### Backend

#### VXLAN

Use in-kernel（是kernel上module实现的） VXLAN to encapsulate the packets.

Type and options:

- `Type` (string): `vxlan`

- `VNI` (number): VXLAN Identifier (VNI) to be used. On Linux, defaults to 1. On Windows should be greater than or equal to 4096.

  VxLan ID，默认是1

- `Port` (number): UDP port to use for sending encapsulated packets. On Linux, defaults to kernel default, currently 8472, but on Windows, must be 4789.

  在linux上是8472，但在windows上，必须是4789

- `GBP` (Boolean): Enable [VXLAN Group Based Policy](https://github.com/torvalds/linux/commit/3511494ce2f3d3b77544c79b87511a4ddb61dc89). Defaults to `false`. GBP is not supported on Windows

- `DirectRouting` (Boolean): Enable direct routes (like `host-gw`) when the hosts are on the same subnet. VXLAN will only be used to encapsulate packets to hosts on different subnets. Defaults to `false`. DirectRouting is not supported on Windows.

  一般会打开，主机如果在同一个网段里面，就没有必要做VxLan封装，这个默认是false的，所以要显示得打开

- `MTU` (number): Desired MTU for the outgoing packets if not defined the MTU of the external interface is used.

- `MacPrefix` (String): Only use on Windows, set to the MAC prefix. Defaults to `0E-2A`.

#### IPIP

Use in-kernel IPIP to encapsulate the packets.

IPIP kind of tunnels is the simplest one. It has the lowest overhead, but can incapsulate only IPv4 unicast traffic, so you will not be able to setup OSPF, RIP or any other multicast-based protocol.

IPIP只能封装IPv4的unicast traffic，pod与pod之间如果要跑主播的流量的话是有问题的，虽然开销要比VxLan小。

Type:

- `Type` (string): `ipip`
- `DirectRouting` (Boolean): Enable direct routes (like `host-gw`) when the hosts are on the same subnet. IPIP will only be used to encapsulate packets to hosts on different subnets. Defaults to `false`.

只有一个DirectRouting，与VxLan里的是一样的。

Note that there may exist two ipip tunnel device `tunl0` and `flannel.ipip`, this is expected and it's not a bug. `tunl0` is automatically created per network namespace by ipip kernel module on modprobe ipip module. It is the namespace default IPIP device with attributes local=any and remote=any. When receiving IPIP protocol packets, kernel will forward them to tunl0 as a fallback device if it can't find an option whose local/remote attribute matches their src/dst ip address more precisely. `flannel.ipip` is created by flannel to achieve one to many ipip network.

部署的flannel会创建一个flannel.ipip的tunnel接口，同时还会有tunl0的接口自动出现。

“tunl0”由 Modprobe IPIP 模块上的 IPIP 内核模块为每个网络命名空间自动创建。它是命名空间默认 IPIP 设备，具有属性 local=any 和 remote=any。当接收 IPIP 协议数据包时，如果内核找不到其本地/远程属性与其 src/dst IP 地址更精确匹配的选项，它会将它们转发到 tunl0 作为回退设备。“flannel.ipip”由flannel创建，以实现一对多的IPIP网络。

#### UDP

Use UDP only for debugging if your network and kernel prevent you from using VXLAN or host-gw.

UDP用于debug的，除非内核不支持VxLan或host-gw（跨网段的话，就不适合使用这个了，没办法保证扩容的机器在同一个网段里）

Type and options:

- `Type` (string): `udp`
- `Port` (number): UDP port to use for sending encapsulated packets. Defaults to 8285.

就只是有个端口了。

VxLan优于IPIP，因为虽然VxLan的开销会大些，但是他能做网络流量的负载。用于区分两个相同的pod之间的不同的流。

### DEMO

ps：因为这个demo需要让所有的节点都起来，我的小电脑撑不住，所以这块笔记的图片均来自视频，后面有时间有设备再把实验跑一遍

```shell
kubectl apply -f kube-flannel_original.yml
kubectl get pods -A
```

首先在master上进行部署

![image-20230426200818295](../assets/image-20230426200818295.png)

接下来我们部署一个test.yaml，这个test中就是有一个网络工具很多的镜像容器，他会跑一个nginx，然后需要创建一个nodeport的服务，然后再创建一个叫client的pod。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: testapp
  name: testappds
spec:
  selector:
    matchLabels:
      app: testapp
  template:
    metadata:
      labels:
        app: testapp
    spec:
      containers:
      - image: wbitt/network-multitool
        imagePullPolicy: IfNotPresent
        name: network-multitool
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: testapp
  name: testappds
spec:
  ports:
  - port: 8080
    nodePort: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: testapp
  type: NodePort
---
apiVersion: v1
kind: Pod
metadata:
  name: client
spec:
  containers:
  - image: wbitt/network-multitool
    imagePullPolicy: IfNotPresent
    name: network-multitool
---
```

apply 一下

```shell
kubectl apply -f test.yaml
kubectl get pods -A -owide
```

![image-20230426201733597](../assets/image-20230426201733597.png)

client落在在worker1，然后我们可以看一下worker1，worker1肯定有两个veth peer了，一个是daemonset，一个是client。两个veth peer分别对应两个pod，创建了一个cni0的一个bridge，还有一个flannel.1的VxLan tunnel接口，.1是因为flannel id默认是1

![image-20230426201925118](../assets/image-20230426201925118.png)

我们首先来看一下flannel.1这个tunnel。他的id是1，本地的封装是用的10.0.0.21这个接口，源端口是随机的，目的端口是8472，nolearning，所有node上的网段都是不一样的，所以没有二层学习的需求，mtu是9100，而enp0s3是9150，减去50个字节的封装，所以是9100。

```sh
ip -d add show flannel.1
```

![image-20230426202218576](../assets/image-20230426202218576.png)

再来看一下cni0，有一个ip，这个ip所有本地pod的网关。

```sh
ip -d add show cni0
```

![image-20230426202639394](../assets/image-20230426202639394.png)

再来看一下client的veth peer的内容

这个要到master上跑，因为我们不知道哪个veth peer是client的。

```sh
kubectl exec -it client -- ip addr show
kubectl exec -it client -- ip neigh
kubectl exec -it client -- ip route
```

![image-20230426205856463](../assets/image-20230426205856463.png)

路由有三条，首先是默认路由，就是走本机的cni0。把pod CIDR的网段也指向了cni0（这个有点多余）。然后最后一条是起点路由。

在没有创建pod之前，就生成了一个flannel.1的VxLan的接口，创建了pod，就会生成veth peer，对端会放到pod里面去，然后本地会创建一个cni0。

创建一个pod的时候，先创建一个veth peer，把一端放到pod里面去，另一端放到cni0下面去。然后再给pod分配ip的时候给了他一个网关，还有一个pod CIDR的静态路由。

接着我们来看一下worker1上的client与test生成的两个的通信，因为他们是同一网段的，所以他们就会找彼此的mac地址，但是两个一开始是不知道，因为我们在上面的neigh中，发现是没有的。然后就会发arp请求去学mac，学了mac之后然后再去发流量。

在主机上，我们就有两个pod的mac地址的。

![image-20230426210924131](../assets/image-20230426210924131.png)

然后我们开始抓client和目的test的包文。

```sh
kubectl exec -it client -- tcpdump -eni eth0
```

接着用client来ping一个包

```sh
kubectl exec -it client -- ping -c 1 10.244.1.8
```

看一下两个的抓包结果，先看test那边的第一个包的结果。

可以看到，一开始不认识，所以发了两个广播来问，谁有9的mac地址，收到了reply后，就有mac地址了，然后就可以封装了，就可以出去了。

![image-20230426211214705](../assets/image-20230426211214705.png)

然后看一下client的抓包

![image-20230426211505914](../assets/image-20230426211505914.png)

当我们发完了以后，再来看看client的neigh，发现就有了neigh，所以.8和.9就可以二层可达了。

![image-20230426211616640](../assets/image-20230426211616640.png)

这个是同一个节点上不同pod之间的通信。本质就是二层交换机bridging的功能。

![image-20230426213220275](../assets/image-20230426213220275.png)

## Flannel-VxLan

我们首先来修改VxLan的配置

修改完配置，在apply之前，我们先来查看一下worker1的ip addr

```yaml
net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "VNI": 4000,
        "Port": 4789,
        "DirectRouting": true 
      }
    }
```

![image-20230426215130164](../assets/image-20230426215130164.png)

接着来apply一下

```sh
kubectl apply -f kube-flannel_vxlan.yml
kubectl get pods -A -owide
```

![image-20230427004715617](../assets/image-20230427004715617.png)

然后我们再来看一下worker1的ip addr

![image-20230427004803212](../assets/image-20230427004803212.png)

![image-20230427004912614](../assets/image-20230427004912614.png)

因为我们没有创建pod，所以没有自动生成cni0和对应veth peer在里面。

然后我们部署一下测试pod

```sh
kubectl apply -f test.yaml
kubectl get pods -A -owide
```

![image-20230427005419881](../assets/image-20230427005419881.png)

用client ping 自己上的test的话，在上一个demo里面已经做给了。

这里主要做的是client 去ping 在worker2 和 worker3的test

### worker1 -> worker2

**不跨网段！！**

首先我们来看看他们相应的路由，先看一下ip addr

```sh
kubectl exec -it client -- ip addr show
```

![image-20230427005658453](../assets/image-20230427005658453.png)

```sh
kubectl exec -it client --ip neigh
kubectl exec -it client --ip route
```

![image-20230427005803527](../assets/image-20230427005803527.png)

接着我们**用client来ping 10.244.2.8**，10.244.2.8在worker2上。首先，发现2.8不是我这个网段的，那么就会去查路由，查路由查到了10.244.1.0/16 via 10.244.1.1 dev eth0，下一跳就是10.244.1.1，但是没有10.244.1.1的map地址，要给这个网关发流量的时候就不行了，就会发arp request，然后在worker1上的cni0就会收到，cni0收到之后就会回reply，然后client的这个pod就会有ip neigh了，有了ip neigh以后就可以封装流量，源ip是自己的这个10.244.1.13，目的ip是10.244.2.8，源mac是自己的mac，目的mac是cni0的mac。包出去了之后了到了worker1上，cni0看到了目的mac是我的mac，然后就剥掉外层，然后看目的ip是10.244.2.8，然后就可以查路由表。

![image-20230427010545976](../assets/image-20230427010545976.png)

然后发现`10.244.2.0/24 via 10.0.0.22 dev enp0s3`

就直接往10.0.0.22的IP地址去了 ，但还没有这个ip neigh

![image-20230427010717517](../assets/image-20230427010717517.png)

现在是没有，但是他可以去学习。因为他们都在10.0.0.0/24 这个网段在自己的直连接口里面，就会发一个arp request去问，worker2收到以后就会回，然后就可以得到10.0.0.22的mac了，再封装一下，源mac是自己，目的mac是收到的reply里面的mac。然后mac收到了发现是enp0s3的mac，就会解，解了以后发现里面是10.244.2.8。查路由表

![image-20230427011115890](../assets/image-20230427011115890.png)

10.244.2.8一看，发现是我们的cni0直连路由，那2.8有ip neigh么，也没有

![image-20230427011205979](../assets/image-20230427011205979.png)

那么就会发arp request，往cni0的接口去发，然后某一个pod收到之后，就会回复，然后就可以学到ip neigh。然后就会封装包，源mac是自己cni0，目的mac是10.244.2.8的mac地址。就扔到了test app。test app收到目的ip，目的mac都是自己，就上升到了icmp了。

然后我们开始发包，并去抓一下包，

```shell
kubectl exec -it client -- ping -c 1 10.244.2.8
```

![image-20230427011655494](../assets/image-20230427011655494.png)

首先看一下worker1上的cni0的抓包

```sh
tcmdump -eni cni0
```

![image-20230427011830361](../assets/image-20230427011830361.png)

最上面是用来获取cni0(10.244.1.1)的mac，获取后拼装成icmp，到了worker1上以后，cni0发现是自己的目的mac，就会把目的mac剥掉，然后目的ip是10.244.2.8，然后查路由表，发现要走10.0.0.22，（这里通过别的流量触发了，已经学到了，不然还要学习一下10.0.0.22的mac地址）然后再封装一层，21是自己（enp0s3），22是目的（10.0.0.22）

![image-20230427012239365](../assets/image-20230427012239365.png)

然后再来看worker2的，包来了之后，enp0s3发现了自己的mac，解了以后，发现是10.244.2.8，发现没有路由。

![image-20230427012848397](../assets/image-20230427012848397.png)

于是要发一个arp request，先问一下10.244.2.8是谁

![image-20230427013032609](../assets/image-20230427013032609.png)

然后10.244.2.8就回了他，然后就用2.8的mac来转发icmp的请求。

如果两个node在同一个网段上，他开启direct routing，其实跟正常的路由转发，没有任何区别，都是查路由，下一跳是直连的，就查IP neigh，如果没有ip neigh就发arp request，学到mac，学到mac就把ip报文再封装一层mac地址，就扔出去了。

### worker2 -> worker3

**跨网段的！！**

接下来我们来分析一下如果我们用client ping 一下到10.244.3.11即不同网段的worker3的地址。

首先查路由表，同样是转给10.244.1.1（eth0），但是没有这个地址的mac地址，于是要本地arp request一下，然后cni0收到了，就将源ip地址和目的ip地址填入，源mac地址填入，目的mac地址就是cni0的mac地址。

![image-20230427014044057](../assets/image-20230427014044057.png)

这样我们cni0收到了这个包，解开以后，我们在worker1上查路由表，发现对应的是这条路由`10.244.3.0/24 via 10.244.3.0 dev flannel.4000 onlink`

![image-20230427014628575](../assets/image-20230427014628575.png)

这个路由下一跳是10.244.3.0，通过dev flannel.4000 onlink 这个接口出去，一看是一个VxLan的接口，我们就要进行VxLan封装，从哪个接口出去，就使用哪个接口的源mac，下一跳是谁就用下一跳的mac。从flannel.4000出去，就用flannel.4000的mac作为源mac。

![image-20230427015115579](../assets/image-20230427015115579.png)

要去10.244.3.0作为下一跳，就要有10.244.3.0的mac地址，然后我们一查可以发现这个IP neigh（这个是flannel给我们添加的）

![image-20230427015335414](../assets/image-20230427015335414.png)

发现有这个mac地址，那么就可以进行封装了，然后就可以使用VxLan封装了，id有了，就是4000，目的端口我们已经指定了是4789，源端口是一个随机的，到了udp就没有了，然后我们从哪个接口出去，就可以使用-d来show一下。

**这个接口是怎么绑定起来的，他为什么只选择enp0s3？**

```sh
ip -d add show flannel.4000
```

![image-20230427015617670](../assets/image-20230427015617670.png)

这样我们就知道了这个源ip，也就是device接口的ip，那么目的IP是什么？不知道，那该如何知道呢。

我们可以来看看neigh的flannel.4000的mac，我们可以通过bridge fdb show来获取mac对应的ip

`FDB`（Forwarding Database entry，即转发表）是 Linux 网桥维护的一个二层转发表，用于保存远端虚拟机/容器的 MAC地址，远端 VTEP IP，以及 VNI 的映射关系，可以通过 `bridge fdb` 命令来对 `FDB` 表进行操作。

```sh
bridge fdb show | grep fa:5e:59:d3:53:15
```

![image-20230427022144477](../assets/image-20230427022144477.png)

这个30.0.0.23到底是谁，是worker3的k8s默认网络接口的ip。就拿这个ip作为VxLan外层封装的目的IP。

接下来来做下tcpdump的验证。

worker1直接抓对应的veth peer的那端口

```sh
tcpdump -eni vethd40b3e16 icmp
```

对于VxLan，我们要抓UDP的流量

```sh
tcpdum -eni enp0s3 udp -vvv
```

`-vvv`就是更加详细一点

在worker3上，首先也是udp的流量

```sh
tcpdump -eni enp0s3 udp -vvv
```

然后我们也找到veth peer对应的那个的地方

```sh
tcpdump -eni veth35d88fcd icmp
```

首先从worker1的veth peer收到了client来的报文，然后就会去3.11，就会查路由

![image-20230427023112524](../assets/image-20230427023112524.png)

然后就会走flannel.4000，从enp0s3出去

![image-20230427023325361](../assets/image-20230427023325361.png)

首先外层，是从10.0.0.21的mac地址发往10.0.0.254（leaf01）的mac地址，做了VxLan的封装。

再来看内层，de:eb:e6:7f:f1:dd这个mac是worker1上flannel的mac，而53:15是worker3上的flannel的mac，这个是worker1的flannel.4000和worker3的flannel.4000之间有个链路，把这两个tunnel的接口连起来，然后内部就是我们原封没有动过的ICMP的payload，这个是一个包去。

然后到了worker3上面，worker3上面收到之后。然后这个对于worker3的enp0s3而言，是我自己的mac地址，就解封装，拿到了对应的ip，发现是VxLan的报文，VxLan的模块就会来处理这个mac，然后看这个ip，发现是cni0的这个路由了，一查在ip neigh中（不在的话，再做一次arp request就可以了），然后就往这个veth peer扔过去。

![image-20230427023958260](../assets/image-20230427023958260.png)

接下来看这个veth peer是怎么响应的。

![image-20230427024333638](../assets/image-20230427024333638.png)

**！！！重要：**

做几次实验可以发现，如果都是ping的话，我们icmp过去的端口都是一样的，而如果使用curl命令的话，我们的端口跟icmp是不同的。

VxLan对于IPIP的优势就在于：VxLan的流量在网路设备上做ECMP的时候其实只能看到最外层的协议栈的封装。假如worker1和worker3之间有很多流量，pod之间，worker1到worker3的流量都是这么封装，VxLan的源IP、目的IP是一样的，ID也是一样的，所以就要看源端口，不同的pod发出来的端口是不同的，同一个pod的不同的流也是不同的，这样的话就可以很好得做哈希，把流量均匀得哈希到不同的链路上。leaf在往spine上转，就要看源端口了，往哪个spine上转合适。就可以把两个相同host的流量，哈希均匀到多个spine上。

而IPIP就没有这个效果，就只看到了是IPIP，不能看到内存的东西，两个主机之间所有的pod之间的流量，都是这样的封装，都会往同一个spine去哈希。

![image-20230427025510891](../assets/image-20230427025510891.png)

## Flannel-IPIP

payload比较小，只是在外层再封装一个ip报文，比较小，只有20个字节。

ipip的配置也非常简单。

会创建tunl0和flannel.ipip的接口，这两个是自动创建出来的。

配置非常简单，如下

```yaml
 net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "ipip",
        "DirectRouting": true 
      }
    }

```

我们修改yml文件

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "ipip",
        "DirectRouting": true 
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate

```

主要就是修改net-config的那块内容。

在应用之前，我们可以先看一下worker1的网络内容

```sh
ip addr show
ip route
```

![image-20230427144950804](../assets/image-20230427144950804.png)

只有默认的三个接口的路由，我们来apply一下刚刚的配置

```sh
kubectl apply -f kube-flannel_ipip.yaml
```

![image-20230427145129309](../assets/image-20230427145129309.png)

然后我们回到worker1，再来看一下网络内容

```sh
ip addr show
```

![image-20230427145217570](../assets/image-20230427145217570.png)

可以看到，我们创建了一个flannel.ipip的tunnel，可以来详细看一下他的内容

```sh
ip -d addr show flannel.ipip
```

![image-20230427145328942](../assets/image-20230427145328942.png)

在worke1上cni0也被创建了，因为我们在worker1上分发了一个pod，coredns的pod，所以cni0就有了。

然后我们用ip route来看一眼路由，看看flannel创建的那些路由。

对于10.244.2.0/24 去了enp0s3 （因为在同一网段）

对于10.244.3.0/24 去了flannel.ipip（在不同网段）

```sh
ip route
```

![image-20230427145515417](../assets/image-20230427145515417.png)

接下来把测试的pod部署起来，部署完后情况如下

![image-20230427150232960](../assets/image-20230427150232960.png)

### worker1 -> worker2

worker1上tcpdump这个veth peer

```sh
tcpdump -eni veth57c643fe
```

![image-20230427150629345](../assets/image-20230427150629345.png)

首先是这个pod不知道自己的网关，所以发一个arp request，来获取网关地址10.244.1.1。学到了网关的mac之后，用网关的mac来封包，然后发送给10.244.2.13了。

这样封装以后包就可以到cni0，也就是worker1的路由上面了，就可以查worker1的路由表。

![image-20230427150930461](../assets/image-20230427150930461.png)

发现要走enp0s3这个口出去

![image-20230427151014203](../assets/image-20230427151014203.png)

我有.22的mac地址，就可以直接拿过来用，如果没有，我就直接用arp request一下就有了，因为他们在同一网段里面，是可以拿到的。然后再做一下封装就可以扔出了，就到了worker2上。

```sh
tcpdump -eni enp0s3 icmp
```

![image-20230427151305347](../assets/image-20230427151305347.png)

到了worker2上，收到了这个mac以后，发现mac是自己的，就负责解封装这个报文。获得ip，然后查相关路由

```sh
tcpdump -eni enp0s3 icmp
```

![image-20230427151509503](../assets/image-20230427151509503.png)

然后就去路由表里看2.13应该发到哪里

```sh
ip route
```

![image-20230427151550588](../assets/image-20230427151550588.png)

然后就去cni0

```sh
ip neigh | grep 10.244.2.13
```

![image-20230427151644464](../assets/image-20230427151644464.png)

这个bridge就通过这个veth peer出去的

![image-20230427151711464](../assets/image-20230427151711464.png)

就发到了worker2上的test 的pod上面，

```sh
tcpdump -eni veth29e3efb4
```

![image-20230427151900304](../assets/image-20230427151900304.png)

他发现mac地址和ip都是自己的，于是就上升到icmp解开封装。这个步骤其实和Flannel-VxLan的worker1->worker2是一样的。

### worker1 -> worker3

接下来我们来测试一下worker1到woker3

```sh
kubectl exec -it testappds-25jt8 -- ping -c 1 10.244.3.16
```

我们从worker1开始看

```sh
tcpdump -eni veth57c643fe
```

![image-20230427152552182](../assets/image-20230427152552182.png)

因为上面我们已经有了worker1网关的mac，所以我们不需要再做arp request了，直接把包转发到了网关。

到了网关以后，worker1就开始查路由

```sh
ip route
```

![image-20230427152716078](../assets/image-20230427152716078.png)

发现3.16走的是`10.244.3.0/24 via 30.0.0.23 dev flannel.ipip onlink`这个路由地址

然后我们来看看这个flannel.ipip的tunnel的内容

```sh
ip -d addr show flannel.ipip
```

![image-20230427152850085](../assets/image-20230427152850085.png)

外层要包装一个ip层，源ip是tunnel的ip，目的ip是下一跳30.0.0.23

可以在物理接口出去的时候看到的

```sh
tcpdump -eni enp0s3 proto 4
```

![image-20230427153026068](../assets/image-20230427153026068.png)

然后mac就是网关的mac，源mac就是这个，目的mac是因为他们要走leaf01，所以用的10.0.0.254这个的mac地址。

包经过leaf01剥掉，然后走到spine，然后走到leaf02，剥掉后查到了要去worker3，最终交给worker3

我们去worker3上dump一下

```sh
tcpdump -eni enp0s3 proto 4
```

![image-20230427153241410](../assets/image-20230427153241410.png)

可以看到，30.0.0.254是leaf02的mac地址，由leaf02发送到了worker3，然后解开后，我们要查30.0.0.23是谁。30.0.0.23是自己，是这个flannel.ipip，就会解封装，然后查内部的10.244.3.16

```sh
ip -d addr show flannel.ipip
```

![image-20230427153356408](../assets/image-20230427153356408.png)

我们就可以看一下ip route，发现要走`10.244.3.0/24 dev cni0 proto kernel scope link src 10.244.3.1`这个路由

```sh
ip route
```

![image-20230427153536599](../assets/image-20230427153536599.png)

然后我们又有这个mac地址，如果没有就发arp request来学习一下

![image-20230427153644595](../assets/image-20230427153644595.png)

就可以直接用cni0的mac作为源mac，这个mac（test的mac）作为目的mac，就往这边发。cni0就会找一下通过哪个veth peer出去

```sh
bridge fdb show | grep aa:c5:62:ca:99:69
```

![image-20230427153946726](../assets/image-20230427153946726.png)

然后我们可以去看一下test pod内抓的包

```sh
tcpdump -eni veth1543573f
```

![image-20230427154057220](../assets/image-20230427154057220.png)

可以看到发过来的这个mac，就是cni0的mac

```sh
ip addr show cni0
```

![image-20230427154125382](../assets/image-20230427154125382.png)

IPIP的封装很简单

```sh
ip -d addr show flannel.ipip
ip route
```

![image-20230427155303151](../assets/image-20230427155303151.png)

local就是源ip，目的ip就是查路由表的下一跳就可以了。

![image-20230427155444522](../assets/image-20230427155444522.png)

另外要注意一下的就是mtu，因为设置的都是9150，而ipip会使用20，所以对于其他而言就是9130。

## Flannel-UDP

不在生成网用，只在debug模式中使用。

除非对于kernel有要求的，可能会用这个。

udp的配置

```yaml
 net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "udp",
        "Port": 8282
      }
    }
```

使用的yaml文件

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "udp",
        "Port": 8282
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.19.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: docker.io/rancher/mirrored-flannelcni-flannel:v0.19.1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

注意，使用UDP的时候要把这个开关打开

```yaml
    securityContext:
      privileged: true
```

```sh
kubectl apply -f kube-flannel_udp.yaml
```

![image-20230427165450482](../assets/image-20230427165450482.png)

接着把test pod给部署起来

```sh
kubectl apply -f test.yaml
kubectl get pods -A -owide
```

![image-20230427165549658](../assets/image-20230427165549658.png)

可以来看一下现在的ip

![image-20230427170307723](../assets/image-20230427170307723.png)

可以来看一下flannel0的接口

```sh
ip -d addr show flannel0
```

![image-20230427170346481](../assets/image-20230427170346481.png)

发现是tun类型，他是可以一端连接到网络协议栈，一端对一个用户程序直接读写的虚拟网卡设备。再来看一下什么时候发送路由

```sh
ip rouet
```

![image-20230427171023938](../assets/image-20230427171023938.png)

把pod CIDR整个集群的网络都往这里走，如果是本地的，就走更加明显的路由，就是10.244.1.0/24 这个是更加精准的路由，所以会优先走这个。所以，如果要查.1.0就走cni0。如果是其他的，比如去worker2或者worker3，都走10.244.0.0/16 dev flannel0。

假如woker1走到worker3 （10.244.1.21 -> 10.244.3.18）

从pod出来，源mac是自己的mac，目的mac是cni0的mac，源ip是10.244.1.21，目的ip就是10.244.3.18。

到了veth peer，给了cni0，因为mac是cni0的mac，就会把mac剥掉，然后就去找路由，路由查到了这个`10.244.0.0/16 dev flannel0`去找3.18，从flannel0扔出去，worker1上的pod(kube-flannel-ds-kr9wv)会把这个包给读出来，读出来之后，一看目的ip是3.18，他就会重新再调用socket api，他知道怎么走。因为他里面有相关的信息，他通过api server可以知道哪个网段分布在哪个node上。3.18就知道他的下一跳在worker3上，就可以直接调用socket api，就把这个包做成一个payload，会被deamonset的这个pod打包成了一个payload，用这个payload调用socket api，告诉socket api要去worker3（30.0.0.23）去目的端口8282。源ip是接口的ip 10.0.0.21。

下面来抓包分析一下。

在worker1首先抓这个pod的veth peer，worker1很简单，第一个目的的mac就是cni0的mac（没有的ip neigh的话就arp request一下），源mac就是对端pod里面的mac，里面的ip是10.244.1.21->10.244.3.18这个包。cni0一看目的mac是自己，就剥掉，一看目的ip，就去查路由表，就走`10.244.0.0/16 dev flannel0`这条路由，就会往flannel0发出去，flannel0是一个tun设备，因为tun设备连接的应用程序是flanneld的pod，就是kube-flannel-ds-xxx，然后他是读取设备的内容，其实就是ip层往里，就会把他们作为一个payload，去查自己里面的表项10.244.3.18，发现他落在在worker3上面，对应的ip就是30.0.0.23，这个flannel的pod就相当于一个普通的应用程序，拿着这个payload去调用网络协议栈的socket api，告诉socket api要去30.0.0.23udp端口8282，payload就是之前IP层往里的封装（10.244.1.21 > 10.244.3.18）的这个内容，然后封装一下给他。

目的ip就是30.0.0.23，源ip就去查路由，从哪个接口出去就用哪个ip，于是就用enp0s3的ip（10.0.0.21）

```sh
tcpdump -eni veth749ee487 icmp
```

![image-20230427173246537](../assets/image-20230427173246537.png)

在enp0s3 udp出去指定端口8282，我们在这个端口上就可以看到上面分析的情况了，这个是外层的内容，从源ip 10.0.0.21.8282到目的ip 30.0.0.23.8282。外层的mac就是enp0s3的mac和leaf01的mac。

```sh
tcpdump -eni enp0s3 udp port 8282
```

![image-20230427174035383](../assets/image-20230427174035383.png)

worker3 监听enp0s3 udp 8282端口。这个包到了worker3上，目的端口是8282，目的ip是自己。他会把这个内容交给监听8282的应用程序。

```sh
tcpdump -eni enp0s3 udp port 8282
```

![image-20230427174315867](../assets/image-20230427174315867.png)

我们来查一下谁在监听这个端口，发现是flanneld在监听这个端口，监听自己的8282端口，用来收包。把收到的这个包解掉之后，把这些payload又写到了worker3上的tun设备里去。协议栈从这个设备里收到了这个包，发现是对10.244.3.18的ping包，于是worker3就去查路由。

```sh
ss -ulpn | grep 8282
```

![image-20230427174423428](../assets/image-20230427174423428.png)

worker3去查路由，一查路由发现，要走cni0这条路由。然后就再查ip neigh（没有的话就去学习一下），然后就找到了veth peer。

![image-20230427174756569](../assets/image-20230427174756569.png)

worker3上抓相关pod的veth peer

```sh
tcpdump -eni veth....  icmp
```

UDP的模式中，报文先从内核协议栈通过tun设备发给了应用程序，做了一次拷贝，用户空间对这个报文做了一些处理，查了一下ip，知道了这个包应该发往哪里，再调用socket api的协议栈，想应用程序发送正常的报文，报文又从用户空间到内核空间。传到的对端，先到内核空间，然后再发给监听这个端口的flanneld这个应用程序，flanneld收到这个报文就知道要发到别的pod的流量，然后又要写入tun设备，又拷贝了一份（用户空间到内核空间）。多了很多用户空间和内核空间的拷贝。这里虽然只做了worker1和worker3，但是worker1和worker2是一样的。

![image-20230427222558665](../assets/image-20230427222558665.png)

pod要去别的节点的pod，先直接发到网关cni0，源mac是自己的mac，目的mac是cni0的mac，源ip是自己的ip，目的ip是另外一个host的一个pod的ip。首先cni0收到了以后，把mac层剥掉，发现目的ip不是本地ip，就直接查路由表，发现路由命中了flannel0，flannel0就会直接往flanneld丢出去了，flannel0是一个tun设备，是一个三层的设备，flannel就可以读到这个设备的内容。会根据内部的数据查一下目的ip的情况，发现这个报文要扔给worker3，就可以直接找到他的ip，就可以通过正常的socket的api的调用，调用这个socket api，去worker3的8282端口，网络协议栈就把拿到的payload，udp封装后就扔给了网关leaf01，就出去了。回来的包发现端口是8282，所以网络协议栈就要看谁在监听这个端口，发现flanneld在监听，网络协议栈就会把这个包扔给flanneld，flanneld就收到这个报文，就知道是我本地的pod，就会写这个内容到flannel0，收到数据后一查发现是pod的ip，就发过去就行了。

## Flannel-Iptables

**kube-proxy and iptables**

- Pod-to-Pod （no NAT）
- Pod-to-Outside
- Pod-to-ClusterIP_Service
- Outside-to-NodePort_Service

首先包进来会走PREROUTING的，然后会做Connection Tracking，因为这个包现在这会是最原始的，所以这个时候做没有问题，然后做mangle PREROUTING，如果对命中的包要做修改，就在这里做。接着会去做nat PREROUTING，这个时候通常的DNAT，像clusterip的service或者nodeport的service都在这里做，做完以后进行判断，这个ip变了以后是本地的还是不是本地的，如果不是本地的，就会做转发，做完转发以后也是一样的mangle FORWARD、filter FORWARD，然后就到POSTROUTING的部分，发出去前要做一个mangle，然后接下来就会做一个nat，这个通常是source nat，假如流量是pod访问internet，网络方案是vxlan等等，这里做一个source nat，然后在这个接口出去。

![image-20230427230512249](../assets/image-20230427230512249.png)

我们首先部署最原始版本的flannel

然后将test的pod进行部署。

```sh
kubectl apply -f kube-flannel-original.yaml
kubectl apply -f test.yaml
```

![image-20230427232858420](../assets/image-20230427232858420.png)

首先我们来看一下用10.244.3.20来ping一下10.244.1.23

然后我们用conntrack - L | grep 10.244.1.23来看一下内容

```sh
conntrack -L | grep 10.244.1.23
```

![image-20230427233445971](../assets/image-20230427233445971.png)

经过worker1的协议栈的时候，有一个conntrack。因为flannel在pod与pod之间直接访问的时候没有做任何的mangle、filter、raw这些东西。

接下来重点来看看带有nat的三种流量

### Pod-to-Outside

这里我们用client来ping一下8.8.8.8

```sh
kubectl exec -it client -- ping -c 1 8.8.8.8
```

现在来看一下

```sh
conntrack -L | grep 8.8.8.8
```

![image-20230428020043143](../assets/image-20230428020043143.png)

source ip是3.20，目的ip是8.8.8.8

但是反向的目的变成了30.0.0.23，因为他做了source nat

这个到底是怎么走过filter的呢

首先我们来看一下nat表，进PREROUTING不会做什么，因为PREROUTING通常是做dnat，FORWARD的时候就是正常转发了，也不会做nat，在做POSTROUTING的时候会有一个source nat。

```sh
iptables -t nat -vnl POSTROUTING
```

![image-20230428020457009](../assets/image-20230428020457009.png)

我们可以再ping一下，

```sh
kubectl exec -it client -- ping -c 1 8.8.8.8
```

![image-20230428020631484](../assets/image-20230428020631484.png)

可以看到数量从25到了26，增加了一个。

MASQUERADE就是根据出接口来做source nat，这个是pod到outside的。默认情况下，overlay方案肯定会做masquerade这个source nat，check的链就是postrouting链。

### Pod-to-ClusterIP_Service

这个是第二种流量

可以看一下现在设置的service

```sh
kubectl describe svc testappds
```

![image-20230428021249252](../assets/image-20230428021249252.png)

我们访问的话就可以直接来curl一下

```sh
kubectl exec -it client -- curl 10.111.222.68:8080
```

那么他到底能命中哪个，我们不得而知，因为会在endpoints里面随机访问的`10.244.1.23:80 , 10.244.2.16:80 , 10.244.3.19:80`

![image-20230428021448873](../assets/image-20230428021448873.png)

首先client的这个pod在worker3上，先发出来的报文，查默认路由，要发到cni0，还没进cni0之前要做PREROUTING，这个报文就被转成了目的ip是endpoints里的。

我们就可以来看一下PREROUTING

```sh
iptables -t nat -vnl PREROUTING
iptables -t nat -vnl KUBE-SERVICES
```

![image-20230428021739123](../assets/image-20230428021739123.png)

可以找到命中的这个svc

```sh
iptables -t nat -vnl KUBE-SVC-QMR6LTDCBEU4U25W
```

第一个是mark-masq，针对外网访问service ip的时候，src不是pod CIDR的，但是目的ip确实是这个的，后面可能会做相应的nat的操作，或者不做nat的操作。

第一条，是到10.244.1.23:80

第二条，是到10.244.2.16:80

第三条，是到10.244.3.19:80

第一条是百分之33的概率到，剩下66%到二三，然后2分配了百分之50，也就是66%的百分之五十。所以三个其实是均匀的

![image-20230428021923704](../assets/image-20230428021923704.png)

再看一层，就可以看到还有一个dnat操作

![image-20230428022303917](../assets/image-20230428022303917.png)

回来的报文

```sh
conntrack -L | grep 10.244.3.20
```

![image-20230428022536943](../assets/image-20230428022536943.png)

### Outside-to-NodePort_Service(Service pod is on the same node)

这个要到spine上去做

```sh
sudo ip vrf exec default curl 30.0.0.23:30080
```

![image-20230428023154184](../assets/image-20230428023154184.png)

```sh
conntrack -L | grep 30080
```

回去的时候就把源ip改成了10.244.3.19，把目的ip改成了10.244.3.1，在同一节点上的时候同时改了两个。

![image-20230428023223375](../assets/image-20230428023223375.png)

我们可以来看一下iptables

修改目的IP就是PREROUTING

```sh
iptables -t nat -vnl PREROUTING
iptables -t nat -vnl KUBE-SERVICES
iptables -t nat -vnl KUBE-NODEPORTS
iptables -t nat -vnl KUBE-EXT-QMR6LTDCBEU4U25W
```

![image-20230428023603029](../assets/image-20230428023603029.png)

最终走到25w这个serevice，这个service就是10.111.222.68这个地方。然后就走到了10.244.3.19，端口就是80，接着往后走。

再看POSTROUTING

```sh
iptables -t nat -vnl POSTROUTING
iptables -t nat -vnl KUBE-POSTROUTING
```

可以看到，我们已经mark了，所以上面这条不命中。我们关注MASQUERADE，我们就要做SNAT，我们过去的流量是通过cni0过去的，所以MASQUERADE，就把dst的ip变成了cni0的IP了。

![image-20230428024049513](../assets/image-20230428024049513.png)

### Outside-to-NodePort_Service(Service pod is on a different node)

我们再来做一个在不同node上的情况。

```sh
sudo ip vrf exec default curl 30.0.0.23:30080
```

可以看到，我们这次到了10.244.2.16这个地方。

我们的流量是先到了worker3上的

![image-20230428024458979](../assets/image-20230428024458979.png)

我们可以先查一下这个表

```sh
conntrack -L | grep 30080
```

可以看到，分别转成了10.244.2.16，10.244.3.0（flannel.1的ip）

![image-20230428024721392](../assets/image-20230428024721392.png)

```sh
iptables -t nat -vnl PREROUTING
iptables -t nat -vnl KUBE-SERVICES
iptables -t nat -vnl KUBE-NODEPORTS
iptables -t nat -vnl KUBE-EXT-QMR6LTDCBEU4U25W
```

![image-20230428025018856](../assets/image-20230428025018856.png)

25W最后走的还是10.111.222.68，目的ip就被随机转成了2.16，端口也转成了80。

因为这里mark了一下，所以经过POSTROUTING的时候就不会RETURN了，就会做SNAT了。

```sh
iptables -t nat -vnl POSTROUTING
iptables -t nat -vnl KUBE-POSTROUTING
```

最终就会用MASQUERADE，SNAT的话，就要看出接口。

![image-20230428025216466](../assets/image-20230428025216466.png)

目的ip是10.244.2.16，可以查一下路由

```sh
ip route
```

发现是通过10.244.2.0的下一跳，通过flannel.1的接口过去的，所以SNAT就要用这个接口的ip，所以就转成了10.244.3.0

![image-20230428025344606](../assets/image-20230428025344606.png)



![image-20230428025854868](../assets/image-20230428025854868.png)

外网设备Node Port提供的service，到了node1，node1做哈希的时候发现流量到了node2，node1就把流量转发给node2。

Packets先到了Node1，src=Client，dst=Node1。命中PREROUTING的KUBE-SERVICE，查到了底，命中了KUBE-NODEPORT。里面根据目的端口30080，就知道命中KUBE-EXT-SERVICE，进去mark一下，再走到KUBE-SERVICE的SERVICE，做DNAT，dst变成了Pod B。POSTROUTING发现了他被MARK了，就根据目的IP，就查找路由的出接口，src = Node1。Node2上就正常做协议栈，然后回给Node1，就做反向NAT。

