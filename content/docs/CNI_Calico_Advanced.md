## Calico-network-policy

![image-20230501161831735](../assets/image-20230501161831735.png)

一般来说，有client来访问前端。通常来说，前端不知道源是谁，但很清楚，只提供tcp 80服务，这个是ingress的。通常我们要访问后端的存储，我们只允许他访问backend的，访问tcp 3306的服务。对于这种无差别的pod，对于有状态的集群，可能会互相访问。后端的backend db，只允许前端的frontend web来访问。对egress限制去任何地方，即deny any。

![image-20230501162125813](../assets/image-20230501162125813.png)

我们挨个执行下面这些yaml

![image-20230501162645848](../assets/image-20230501162645848.png)

采用bgp模式，选用ebgp。

我们来使用test.yaml

```sh
kubectl apply -f test.yaml
```

```yaml
# 前端是一个daemonset
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: frontend
  name: frontendweb
spec:
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - image: wbitt/network-multitool
        imagePullPolicy: IfNotPresent
        name: network-multitool
---
# 将他的服务端口暴露出来
apiVersion: v1
kind: Service
metadata:
  labels:
    app: frontend
  name: frontendweb
spec:
  ports:
  - port: 8080
    nodePort: 30080
    protocol: TCP
    targetPort: 80
  selector:
    app: frontend
  type: NodePort
---
# 创建一个client的pod
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
spec:
  containers:
  - image: wbitt/network-multitool
    imagePullPolicy: IfNotPresent
    name: network-multitool
---
# 使用deployment来创建一个后端的mysql数据库
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend
  name: backenddb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - image: mysql
        imagePullPolicy: IfNotPresent
        name: mysql
---
# 暴露相关的接口
apiVersion: v1
kind: Service
metadata:
  labels:
    app: backend
  name: backenddb
spec:
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: backend
  type: ClusterIP
---
```

下面的policy的设置

这里就设置了哪些端口可以到达，哪些端口不能到达。

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-web
  namespace: default
spec:
# 应用在app 叫 frontend的pod上
  selector: app == 'frontend'
  ingress:
  - action: Allow
    protocol: TCP
    # 只能访问80端口
    destination:
      ports:
        - 80
  egress:    
  - action: Allow
    protocol: TCP
    # 只能去访问destination的app叫backend，3306端口
    destination:
      selector: app == 'backend'
      ports:
        - 3306
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-mysql
  namespace: default
spec:
  selector: app == 'backend'
  ingress:
  - action: Allow
    protocol: TCP
    source:
      selector: app == 'frontend'
    destination:
      ports:
        - 3306
  egress:    
  - action: Deny
---
```

## Calico-ebpf

ebpf - extended Berkeley Packet Filter

运行在内核的虚拟机的东西，允许很小的程序被加载到内核中，可以attach到一些hook上面，当一些事件发生的时候一些程序就会被运行了。

1.Tracing programs can be attached to a significant proportion of the functions in the kernel. Tracing programs are useful for collecting statistics and deep-dive debugging of the kernel.

2.Traffic Control (tc) programs can be attached at ingress and egress to a given network device. The kernel executes the programs once for each packet. Since the hooks are for packet processing, the kernel allows the programs to modify or extend the packet, drop the packet, mark it for queueing, or redirect the packet to another interface. Calico’s eBPF dataplane is based on this type of hook; we use tc programs to load balance Kubernetes services, to implement network policy, and, to create a fast-path for traffic of established connections.

3.Several types of socket programs hook into various operations on sockets, allowing the eBPF program to, for example, change the destination IP of a newly-created socket, or force a socket to bind to the “correct” source IP address. Calico uses such programs to do connect-time load balancing of Kubernetes Services; this reduces overhead because there is no [DNAT](https://projectcalico.docs.tigera.io/about/about-networking) on the packet processing path.

4.XDP, or “eXpress Data Path”, is actually the name of an eBPF hook. Each network device has an XDP ingress hook that is triggered once for each incoming packet before the kernel allocates a socket buffer for the packet. The downside of XDP is that it requires network device driver support to get good performance. 

左边这个就是ebpf的程序，编译成了foo.o的文件，然后被BPF的loader加载内核里面去，在加载到内核之前，要去verifier一下，保证不会crash掉kernel，要和内核的模块一样快，要保证稳定的API，变成了native的一些执行码，这个是可以访问BPF的maps，根据使用的场景来，就可以来访问这个bpf maps。接着从eth0来了一个包，假如这个程序就是去match这个包，就会做一些操作，直接到协议栈里、丢掉或者redirect到别的地方。

![image-20230501190057795](../assets/image-20230501190057795.png)

接下来看一下一般的hook点会在什么地方。

在进到net filter的橘色框之前就要做一个TC的ingress，出去前有一个TC的engress，假如ip是pod的ip。

![image-20230501190514153](../assets/image-20230501190514153.png)

就可以直接跳过这些过程，直接到达TC的engress，这样就少了很多过程，相当于是一个redirect的功能。

![image-20230501190924836](../assets/image-20230501190924836.png)

还有通过直接转发的，从XDP收到，直接redirect到另外一个网卡上去。

![image-20230501191050716](../assets/image-20230501191050716.png)

Calico主要使用eBPF的redirect还有socket层面的来实现load balance。会hook在一些socket的操作上。

| **Factor**                            | **Standard  Linux Dataplane**                           | **eBPF**  **dataplane**                                      |
| ------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| Throughput                            | Designed  for 10GBit+                                   | Designed  for 40GBit+                                        |
| First  packet latency                 | Low  (kube-proxy  service latency is bigger factor)     | Lower                                                        |
| Subsequent  packet latency            | Low                                                     | Lower                                                        |
| Preserves  source IP within cluster   | Yes                                                     | Yes                                                          |
| Preserves  external source IP         | Only  with externalTrafficPolicy:  Local                | Yes                                                          |
| Direct  Server Return                 | Not  supported                                          | Supported  (requires compatible underlying network)          |
| Connection  tracking                  | Linux  kernel’s conntrack  table (size can be adjusted) | BPF  map (fixed size)                                        |
| Policy  rules                         | Mapped  to iptables rules                               | Mapped  to BPF instructions                                  |
| Policy  selectors                     | Mapped  to IP sets                                      | Mapped  to BPF maps                                          |
| Kubernetes  services                  | kube-proxy iptables or IPVS mode                        | BPF  program and maps                                        |
| IPIP                                  | Supported                                               | Supported  (no performance advantage due to kernel limitations) |
| VXLAN                                 | Supported                                               | Supported                                                    |
| Wireguard                             | Supported  (IPv4 and IPv6)                              | Supported  (IPv4)                                            |
| Other  routing                        | Supported                                               | Supported                                                    |
| Supports  third party CNI plugins     | Yes  (compatible plugins only)                          | Yes  (compatible plugins only)                               |
| Compatible  with other iptables rules | Yes  (can write rules above or below other rules)       | Partial;  iptables bypassed for workload traffic             |
| Host  endpoint policy                 | Supported                                               | Supported                                                    |
| Enterprise  version                   | Available                                               | Available                                                    |
| XDP  DoS Protection                   | Supported                                               | Supported                                                    |
| IPv6                                  | Supported                                               | Not  supported (yet)                                         |

### 已经运行起来的k8s

- There is a working Kubernetes cluster(with **kube**-proxy) running in Calico BGP mode
- Configure Calico to talk directly to the API server
  - Create kubernetes-service-endpoint ConfigMap in kube-system namespace
  - Restart the Calico pods
- Disable kube-proxy
- Enable eBPF mode
- Try out DSR mode

首先创建一个ConfigMap

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kubernetes-services-endpoint
  namespace: kube-system
data:
  KUBERNETES_SERVICE_HOST: "10.0.0.11"
  KUBERNETES_SERVICE_PORT: "6443"
---
```

kubernetes上，提供对外服务的apiserver地址。

然后我们来apply一下这个config

```sh
kubectl apply -f kubernetes-service_configmap.yaml
```

接着等待`60s`左右来等他起来。然后将两个pod删掉

```sh
kubectl delete pod -n kube-system -l k8s-app=calico-node
kubectl delete pod -n kube-system -l k8s-app=calico-kube-controllers
```

接着我们来patch一下kube-proxy的内容，disable相关内容

```sh
kubectl patch ds -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"nodeSelector":{"non-calico": "true"}}}}}'
```

接下来来enbale ebpf的模式

```sh
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfEnabled": true}}'
```

这个是socket program做的dnat的转换的，不是由ip table来进行prerouting，postrouting来做改变的。

dsr的优势是让报文到达目的后，源ip不会被snat掉，不会做改变。

![image-20230501194859992](../assets/image-20230501194859992.png)

下面打开dsr的特性

```sh
calicoctl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
```

接着来抓包看一下这个的过程。

首先到leaf1上（因为leaf1是外部访问）

leaf 访问worker2，然后转发到了worker1上。

```sh
sudo ip vrf exec default curl 10.0.0.21:30080
```

worker2在给worker1转发报文的时候，什么都没有做，只是把mac修改了一下。

worker1收到了之后，就开始回，回给了leaf1。worker1回的syn ack。用的是自己的mac，ip是worker2的ip。

![image-20230501202329661](../assets/image-20230501202329661.png)

### 尚未运行k8s

- Create a suitable Kubernetes cluster (without kube-proxy)
  kubeadm init --pod-network-cidr=122.168.0.0/16 --skip-phases=addon/kube-proxy
- Create kubernetes-service-endpoint ConfigMap in tigera-operator namespace
- Install the Tigera Operator
  - Apply tigera-operator.yaml (Add the above ConfigMap here)
  - Apply custom-resources.yaml (with DIY config)
- Apply Node and BGPPeer configuration
- Try out DSR mode

1. 首先是kubeadm init，要使用自己的cidr，但是要跳过kube-proxy阶段

   ```sh
   kubeadm init --pod-network-cidr=10.244.0.0/16 --skip-phases=addon/kube-proxy
   ```

   这样初始出来的kubernetes集群就没有kube-proxy了

   这样以后就可以获得所有的node了

   ![image-20230502003102164](../assets/image-20230502003102164.png)

2. 接下来要配置一下configmap

   这个配置，放在了下一步，因为这里面还没有这个namespace，不不然会报错的。

3. 运行tigera-operator

   这个用apply好像会报错，所以用create来实现

   ```sh
   kubectl create -f tigra-operator.yaml
   ```

   然后可以看到相应的operator跑起来了。

   ![image-20230502003606706](../assets/image-20230502003606706.png)

4. 配置一下

   这个是installation的文件，配置ippools，还要修改cidr

   ```yaml
   # This section includes base Calico installation configuration.
   # For more information, see: https://projectcalico.docs.tigera.io/master/reference/installation/api#operator.tigera.io/v1.Installation
   apiVersion: operator.tigera.io/v1
   kind: Installation
   metadata:
     name: default
   spec:
     # Configures Calico networking.
     calicoNetwork:
       nodeAddressAutodetectionV4:
         interface: "enp0s3"
       linuxDataplane: BPF
       # Note: The ipPools section cannot be modified post-install.
       ipPools:
       - blockSize: 24
         cidr: 10.244.0.0/16
         encapsulation: None
         natOutgoing: Disabled
         nodeSelector: all()
   ---
   ```

   ```s
   kubectl create -f installation.yaml
   ```

   ![image-20230502004005820](../assets/image-20230502004005820.png)

5. 然后使用Node-BGPPeer_BGP-AS-per-server.yaml

   ```sh
   apiVersion: projectcalico.org/v3
   kind: BGPConfiguration
   metadata:
     name: default
   spec:
     logSeverityScreen: Info
     nodeToNodeMeshEnabled: false
     serviceClusterIPs:
     - cidr: 10.96.0.0/12
   ---
   apiVersion: projectcalico.org/v3
   kind: Node
   metadata:
     name: master1.k8s.com
   spec:
     bgp:
       asNumber: 65011
       ipv4Address: 10.0.0.11/24
   ---
   apiVersion: projectcalico.org/v3
   kind: BGPPeer
   metadata:
     name: master1.peer-tor
   spec:
     node: master1.k8s.com
     peerIP: 10.0.0.254
     asNumber: 65001
   ---
   apiVersion: projectcalico.org/v3
   kind: Node
   metadata:
     name: worker1.k8s.com
   spec:
     bgp:
       asNumber: 65021
       ipv4Address: 10.0.0.21/24
   ---
   apiVersion: projectcalico.org/v3
   kind: BGPPeer
   metadata:
     name: worker1.peer-tor
   spec:
     node: worker1.k8s.com
     peerIP: 10.0.0.254
     asNumber: 65001
   ---
   apiVersion: projectcalico.org/v3
   kind: Node
   metadata:
     name: worker2.k8s.com
   spec:
     bgp:
       asNumber: 65022
       ipv4Address: 10.0.0.22/24
   ---
   apiVersion: projectcalico.org/v3
   kind: BGPPeer
   metadata:
     name: worker2.peer-tor
   spec:
     node: worker2.k8s.com
     peerIP: 10.0.0.254
     asNumber: 65001
   ---
   apiVersion: projectcalico.org/v3
   kind: Node
   metadata:
     name: worker3.k8s.com
   spec:
     bgp:
       asNumber: 65023
       ipv4Address: 30.0.0.23/24
   ---
   apiVersion: projectcalico.org/v3
   kind: BGPPeer
   metadata:
     name: worker3.peer-tor
   spec:
     node: worker3.k8s.com
     peerIP: 30.0.0.254
     asNumber: 65002
   ---
   
   ```

6. enable一下DSR

   ```sh
   calicoctl patch felixconfiguration default --patch='{"spec": {"bpfExternalServiceMode": "DSR"}}'
   
   ```

7. 部署测试的pod

## Calico-VPP

iptables、IPVS、ebpf的hook都是在linux内核去实现包的转发，VPP的dataplane是由用户空间实现。

VPP - Vector Packet Processor

用户空间的网络协议栈的再一次实现。通过加载plugin这种形式来实现。

逻辑架构。以前的包是通过主机的网络协议栈作为重要的转发点，所有的pod都连到主机的网络协议栈上去。

现在由于主机的网络协议栈比较臃肿，或者很多不需要的东西，也要走一遍。现在是由VPP来在用户空间进行转发。

我们的pods、主机的网络协议栈，都通过tun的tap或者tun设备来通过与这个VPP交互。tun设备有tun和tap设备，tun是三层的，tap是二层的。一端接入到网络协议栈，一端接入到应用程序。tap设备里就是二层的帧，tun设备里面就三层的ip包。协议栈往tap或者tun设备的里发数据，应用程序就能收到。VPP就可以理解成处理这些数据的应用程序。VPP收到这些流量以后，终归要转出去的，使用uplink interface来发出去，这个是个物理接口。

如果一个pod被分配到一个主机上，kubelet就会创建一个网络名词空间给这个pod，然后来调用这个cni插件，二进制文件，给这个网络名词空间创建接口。Calico会继承出来这个ip地址和路由，把这个信息传给VPP这个应用程序，VPP就会创建一个tun设备，tun设备的接口就放到指定的网络名词空间里去了。也会把calico传来的地址和路由信息配上，就完成了流量引入到VPP中。

![image-20230502004521350](../assets/image-20230502004521350.png)

通常我们要来指定这个uplink的driver，如果不指定，就会随机来选一个。

![image-20230502005835726](../assets/image-20230502005835726.png)

VPP启动以后，根据配置把接口（enp0s3）拿过来，用配置的接口驱动来接管这个接口了。放到一个namespace中，或者完全挪走，不管用哪种方式，在主机的网络协议栈里就会消失掉。

接着配置这个接口（host-enp0s3），这个配置肯定要跟原来的（address、routes）一样的。

然后要创建一个tap接口（enp0s3），同样还叫原来的名字，在VPP和网络协议栈中。

然后这个tap接口就会给同样的配置，包括了mac地址等等，只不过他变成了一tap设备。

这个tap接口，也要有同样的地址、路由等。

在VPP里面，他的名字就叫tap0。

VPP同样会创建一个tun设备给每一个pod。

下面这个是在实验中部署完这个VPP以后呈现出来的一个数据。可以先看uplink，原来在主机上叫enp0s3，被vpp管理以后就叫host-enp0s3，ip地址和mac没有变化，接入主机协议栈呢，就会创建一个同样名字的tap接口，速率会有改变，对应的VPP端的tap0会与他接入，ip地址是与主机tap0接口相同，但是mac地址不同。每一个pod，都会创建一个tun接口。

![image-20230502010507516](../assets/image-20230502010507516.png)
