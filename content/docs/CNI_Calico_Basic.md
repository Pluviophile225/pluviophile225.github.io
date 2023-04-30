## Demo拓扑图

![image-20230428124933432](../assets/image-20230428124933432.png)

## Calico基础结构

![image-20230428130522440](../assets/image-20230428130522440.png)

我们先来看pod上的内容，flannel的叫kube-flannel，这里叫calico/node，一共有三块东西，felix、BIRD、confd。提供了两个plugin，（二进制执行文件）给kubelet来调用。在pod里面，有个叫calixx与eth0接入，这就是一个veth peer，他里面没有bridge，就直接插在了网络协议栈上面。BIRD在跑BGP的时候，IPIP  mod的时候会用到，用来生成pod CIDR的路由。felix是一些policy，confd是一些与配置相关的。

上面的Typha，因为calico是支持大的集群的，如果calico/node都要与apiserver沟通的话，apiserver的压力就太大了。Typha就可以集中管理calico/node与apiserver的通信。

Calico/node

- Felix: Programs routes and ACLs, and anything else required on the host to provide desired connectivity for the endpoints on that host. Runs on each machine that hosts endpoints. Runs as an agent daemon.对路由和 ACL 进行编程，以及主机上为该主机上的端点提供所需连接所需的任何其他内容。在托管终结点的每台计算机上运行。作为代理守护程序运行。
- BIRD: Gets routes from Felix and distributes to BGP peers on the network for inter-host routing. Runs on each node that hosts a Felix agent. Open source, internet routing daemon.从 Felix 获取路由，并分发到网络上的 BGP 对等方以进行主机间路由。在托管 Felix 代理的每个节点上运行。开源的互联网路由守护程序。
- confd: Watches Calico datastore for changes to BGP configuration and global defaults such as AS number, logging levels, and IPAM information. Open source, lightweight configuration management tool. dynamically generates BIRD configuration files based on the updates to data in the datastore. When the configuration file changes, confd triggers BIRD to load the new files.监视 Calico 数据存储，了解 BGP 配置和全局默认值（如 AS 编号、日志记录级别和 IPAM 信息）的更改。开源、轻量级配置管理工具。根据数据存储中的数据更新动态生成 BIRD 配置文件。当配置文件更改时，confd 会触发 BIRD 加载新文件。

## Install

现在有一个kubernetes的空集群，但是还没有装任何cni

![image-20230428132534828](../assets/image-20230428132534828.png)

在calico的yaml中，我们需要着重在意下面内容

![image-20230428133134751](../assets/image-20230428133134751.png)

首先第一个是自定义的kubedeam的CIDR，第二个是block的size，这里用24只是为了好看ip，第三个因为我们有很多个出口，所以要设置他默认使用哪个出口。

```yaml
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
- name: CALICO_IPV4POOL_BLOCK_SIZE
  value: "24"
- name: IP_AUTODETECTION_METHOD
  value: "interface=enp0s3"
```

然后我们可以来部署一下这个内容

```sh
kubectl apply -f calico-original.yaml
```

![image-20230428175337920](../assets/image-20230428175337920.png)

现在来部署一下自己的test pod

```sh
kubectl apply -f test.yaml
```

![image-20230428181621881](../assets/image-20230428181621881.png)

这里重点关注一下worker3上的两个pod通信是如何完成的。

我们可以看一下worker3上的ip addr show

![image-20230428181956786](../assets/image-20230428181956786.png)

tunl0就是系统默认创建的，是ipip的，给他分了一个ip。

再来看一眼路由

```sh
ip route
```

默认路由是enp0s3

blackhole 掉 10.244.73.0/24 proto bird，扔掉这个流量

10.244.73.1 dev cali5cf2070224c scope link

10.244.73.2 dev claia7a3e029a7b scope link

因为calico没有bridge，所以没有办法根据bridge学习到veth peer的对端地址，所以这里需要把他们显示得写出来。这个是felix做的事情。

master1 、 worker1 、 worker2都走

10.244.204.0/24 via 10.0.0.21 dev tunl0 proto bird onlink

10.244.65.0/24  via 10.0.0.11 dev tunl0 proto bird onlink

10.244.42.0/24  via 10.0.0.22 dev tunl0 proto bird onlink

然后下面的30.0.0.0/24 自己的直连路由

![image-20230428182857261](../assets/image-20230428182857261.png)

- Installs the calico/node container on each host using a DaemonSet(calico-node), calico/node container runs three internal daemons: 一个container里面三个daemon

  - Felix, the Calico daemon that runs on every node and provides endpoints.在每个节点上，提供endpoints，生成去往本地pod的路由
  - BIRD, the BGP daemon that distributes routing information to other nodes.分发节点信息
  - confd, a daemon that watches the Calico datastore for config changes and updates BIRD’s config files.来监听calico的配置

- Installs the Calico CNI binaries and network config on each host using a DaemonSet(calico-node/initContainers_install-cni)

  把calico cni的二进制文件放到host上去。

- Runs calico/kube-controllers as a deployment(calico-kube-controllers)

  会跑起来一个controller

- The calico-config ConfigMap, which contains parameters for configuring the install.

  用来控制安装的参数

- ServiceAccount and RBAC

接着来看一下同一个node上不同pod之间流量是如何完成的。

首先来看一下client的内容。

可以看到了很奇怪的是，ip route出口的地址是169.254.1.1，非常奇怪的一个地址，然后eth0的ip地址是32位的，与flannel是不一样的。

这里没有用到这个ip（169.254.1.1）做数据的转发，只是用这个ip来解析mac地址，因为enth0是个broadcast，是一个32位的广播网。会匹配默认路由，问这个ip（169.254.1.1）的mac，主机协议栈收到了这个arp request，有这么一条路由`169.254.0.0/16 dev enp0s3 scope link metric 1000`存在，如果打开了arp 代理的话，就会用这个接口的mac地址来回这个arp request。

![image-20230428190024393](../assets/image-20230428190024393.png)

下面来看一下 arp proxy

```sh
sysctl -a | grep -i cali | grep proxy_arp
```

![image-20230428190844882](../assets/image-20230428190844882.png)

现在我们用7.1来ping一下7.2

```sh
kubectl exex -it testappds-gz6ls -- ping -c 1 10.244.73.2
```

注意，224c就是client，9a7b就是test pod。

首先是发了一个arp request，cf:81就是client的eth0地址，发的是ff:ff:ff:ff..广播地址，问的是169.254.1.1，那么是谁回的呢，用收到这个arp request的接口，来回复这个包，并且告诉他mac地址在哪里（ee:ee:ee:ee:ee:ee:ee:ee）。然后发的包就用这个包来封装。然后到了主机协议栈，发现了mac是（ee:ee:ee:ee:ee:ee:ee:ee）是自己的，就去查路由，发现就10.244.73.2要从路由表的那个接口出去，于是就从这个接口出去了。

![image-20230428191335299](../assets/image-20230428191335299.png)

扔出去了以后，用出接口的mac（8对e）来封装这个包。

![image-20230428192033298](../assets/image-20230428192033298.png)

这样以后，我们就有了新的ip neigh了

![image-20230428192235803](../assets/image-20230428192235803.png)

来比较一下flannel与calico，flannel首先是有bridge，然后pod都在同一个网段里，主机协议栈的veth peer这一段都在bridge里面，bridge是网络协议栈的一块，所以流量之间的arp是可以直接互相到达的，回的arp reply都没有问题。

Calico不同的是，Calico也有veth peer，但是veth peer的pod的ip是32位的ip，所以每个pod都是一个网段，那不同的网段之间通信要走网关。走的网关是`default via 169.254.1.1 dev eth0`，网关其实在流量转发过程中并没有ip在里面，所以是什么ip都无所谓，但这里为了避免冲突，用了这个ip。但是要去问这个ip的mac地址，该怎么办呢，就使用了proxy arp的这个特征，在主机协议栈上calico这个veth peer上打开了proxy arp，然后同时主机上也有这么一条16位的路由，然后就会回这个arp request，用自己的calic的mac去回。

我们来分析一下下面这张图的流量流转情况。首先这张图是有关在worker2上的两个pod内容的分析。在node上的有个pod叫client，他有一个veth peer对，一端在Client里面的eth0（ip 122.168.42.1/32 mac ca:16:92:db:cc:5d），另一端直接接在了网络协议栈上（mac ee:ee:ee:ee:ee:ee:ee:ee）。另一个pod叫test，他也有一个veth peer对，一端在pod内部叫eth0（ip 122.168.42.2/32 mac 82:64:68:8c:97:db），另一端直接接在在网络协议栈上（mac ee:ee:ee:ee:ee:ee:ee:ee）。网络协议栈上有一个出口叫enp0s3（ip 10.0.0.22/24 mac 08:00:10:00:00:22）

![image-20230428202420507](../assets/image-20230428202420507.png)

接着我们来使用这个client的pod向这个test的pod发起ping请求

```sh
kubectl exec -it client --ping -c 1 122.168.42.2
```

发起这个请求的时候，我们要在client上面查一下本地的路由

```sh
ip route
```

可以发现只有一个路由，就是169.254.1.1 这个图里没有表示出来，但是就是这个区域内的值。于是我们有了源ip、源mac，目的ip，但是没有目的mac。于是我们需要发起一个arp request来查询这个内容。下面这个就是展开来的图。

![image-20230428203942304](../assets/image-20230428203942304.png)

我们可以看到，是由eth0发起的广播，而calia7a.... 就是client在网络协议栈的那段，收到了这个广播以后，就会把自己的mac地址告诉他。（前提是这个要开启proxy_arp）于是获得了mac地址，就可以发送请求了。

![image-20230428203329968](../assets/image-20230428203329968.png)

当我们有了源ip 122.168.42.1 ， 源mac ca:16:92:db:cc:5d ， 目的ip 122.168.42.2，目的mac ee:ee:ee:ee:ee:ee:ee:ee。就可以封装包发出来啦。我们这个包发到了网络协议栈后，15:calica7a3e...发现了这个mac地址就是自己的mac地址以后，他就会负责解封装，然后发现内部是要发到122.168.42.2的，于是就可以查路由表了。

```sh
ip route
```

这里没有显示路由表的内容，但是很明显的，我们是一定会走16:cali18a94c...这个路由的，但是他可能一开始也不知道自己对端的mac地址，这个时候就又要去发一个arp请求。

![image-20230428205224427](../assets/image-20230428205224427.png)

可以看到，使用的mac地址也是ee:ee:ee:ee:ee:ee:ee:ee，但是要注意，这个mac地址用的是16:cali18a..这个veth peer，然后发送广播，对端的test收到了这个请求，就把自己的mac地址发过来了。于是源ip、目的ip，就是刚刚的，我们没有做改变，修改源mac和目的mac，向test的pod发过去。而test的pod发现这个是自己的mac地址，就会解封装了，然后获得相应的内容了。

![image-20230428204638327](../assets/image-20230428204638327.png)

最后从test里，向原来的发回请求就好了。注意，走完了一次以后，就发现，我们这个pod中已经把10.0.0.22和169.254.1.1给学习好了。

![image-20230428205853261](../assets/image-20230428205853261.png)

## Calico - IPIP

修改配置文件，将always改成CrossSubnet

![image-20230428210537378](../assets/image-20230428210537378.png)

```yaml
# Enable IPIP
- name: CALICO_IPV4POOL_IPIP
  value: "CrossSubnet"
```

ipip的backend也要用bird

在上面install的例子里面，这里设置的是always，所以哪怕worker1和worker2在同一个网段里面，他们也会走tunnel。这里设置了CrossSubnet就是说，只有在不同的网段里，我们才会使用使用ipip来实现tunnel的通信，如果是同一网段，就直接发就好了

比方说，我们修改了以后，再去worker1上看看情况

```sh
ip route show proto bird
```

![image-20230428211749923](../assets/image-20230428211749923.png)

去往master1和worker2上的只要走物理接口就可以了，然后去往worker3的要走ipip的这个tunnel

我们用worker1上的client，分别发给worker2和worker3。

![image-20230428211918707](../assets/image-20230428211918707.png)

worker1上的client的ip route、ip neigh、ip addr如下

![image-20230428212243975](../assets/image-20230428212243975.png)

### worker1 -> worker2

```sh
kubectl exec -it client ping -c 1 10.244.42.2
```

首先我们监控worker1上的这个client pod对应的veth peer端口。中间省略了arp获取8对e mac的过程。封包出去，然后到了我们worker1的enp0s3口。

```sh
tcpdump -eni calia7a3e029a7b icmp
```

![image-20230428214611277](../assets/image-20230428214611277.png)

我们可以来看一下enp0s3的接发包的过程。

```sh
tcpdump -eni enp0s3 icmp
```

![image-20230428214750115](../assets/image-20230428214750115.png)

收到这个icmp的报文内容以后，就去查10.244.42.2的路由

```sh
ip route
```

![image-20230428214846469](../assets/image-20230428214846469.png)

在路由`10.244.42.0/24 via 10.0.0.22 dev enp0s3 proto bird`可以看到走这条路由，可以看到是从enp0s3出去，去往下一跳10.0.0.22，然后就要查他的ip neigh

```sh
ip nei | grep 10.0.0.22
```

![image-20230428215224908](../assets/image-20230428215224908.png)

可以看到他的ip neigh，出接口就是enp0s3的mac，目的mac就是这里查到的mac，包扔出去了以后。

可以到worker2上查查内容了，因为是同一网段，所以enp0s3就可以收到了。收到了由enp0s3解开封包，发现目的ip是10.244.42.2，于是再去查路由。

![image-20230428215335932](../assets/image-20230428215335932.png)

可以想象到，这个查的一定就是本地的一个cali..... 的veth peer的一端，然后就往这个接口扔出了，但是扔出去前就要查42.2的ip neigh，如果没有的话，会由这个发一个广播，然后42.2对应的这个pod会回复自己的mac，于是源mac就是8对e，目的mac就是收到的这个mac。最终由需要的pod来收到这个内容。

![image-20230428215649130](../assets/image-20230428215649130.png)

下面的图的内容表述的是一个意思。

![image-20230428220919752](../assets/image-20230428220919752.png)

![image-20230428220931274](../assets/image-20230428220931274.png)

![image-20230428220944375](../assets/image-20230428220944375.png)

### worker1 -> worker3

```sh
kubectl exec -it -- ping -c 1 10.244.73.2
```

首先看一下从pod出发的流量，这个与之前都是一样的，就是找mac地址（广播）然后在协议栈那块的veth peer端给他了地址，然后包装mac地址，封装之前的ip报文后发出去，到了mac端解开后得到如下内容

```sh
tcpdump -eni calia7a3e029a7b icmp
```

![image-20230429134645210](../assets/image-20230429134645210.png)

然后就去查路由，找73.2的路由

```sh
ip route
```

![image-20230429134723613](../assets/image-20230429134723613.png)

发现命中了tunl0这条路由，再来看一下tunl0是什么tunl

```sh
ip -d add show tunl0
```

![image-20230429134807679](../assets/image-20230429134807679.png)

目的ip地址是知道的就是30.0.0.23，就要看出接口是谁了。发现出去的口是10.0.0.21于是就做了封装

```sh
ip route get 30.0.0.23
```

![image-20230429134909654](../assets/image-20230429134909654.png)

可以来看一下封装以后的样子，

```sh
tcpdump -eni enp0s3 proto 4
```

mac地址就很简单了，因为出口是10.0.0.21，就直接用他的mac地址，然后出去看一下条，ip neigh是有下一跳的，就是有交换机leaf01的mac地址的。

![image-20230429135330894](../assets/image-20230429135330894.png)

然后走完leaf01，spine，leaf02，最终到达了worker3上

```sh
tcpdump -eni enp0s3 proto 4
```

worker3收到了，leaf02的mac到了worker3的mac，worker3收到了以后发现ip是自己，就剥掉，然后发现内层是一个ipip的tunl。

![image-20230429135709669](../assets/image-20230429135709669.png)

tunl0的内容如下

```sh
ip -d addr show tunl0
```

![image-20230429135902515](../assets/image-20230429135902515.png)

所以任意的ip都会使用这个tunl0的，进来了以后就送到了tunl0，然后去查里面的ip，然后再看一下里面的ip是73.2，就送往对应的pod里面去。封装的时候，就会去看看73.2的ip neigh了。

```sh
tcpdump -eni calia.... icmp
```

![image-20230429140052283](../assets/image-20230429140052283.png)

![image-20230429141707175](../assets/image-20230429141707175.png)

![image-20230429141717243](../assets/image-20230429141717243.png)

![image-20230429141730018](../assets/image-20230429141730018.png)

## Calico-VxLan

首先我们要去修改配置

- 把calico_backend改成vxlan

  ```yaml
  calico_backend:"vxlan"
  ```

- 修改CrossSubnet

  ```yaml
  - name: CALICO_IPV4POOL_IPIP
    value: "Never"
  - name: CALICO_IPV4POOL_VXLAN
    value: "CrossSubnet"
  - name: CALICO_IPV6POOL_VXLAN
    value: "Never"
  ```

- bird ready检测给注释掉

  ```yaml
  # - - bird-live
  # - - bird-ready
  ```

修改好配置后，我们来apply一下这个yaml，然后apply一下要运行test pod

```sh
kubectl apply -f calico-vxlan.yaml
kubectl apply -f test.yaml
```

![image-20230429142635463](../assets/image-20230429142635463.png)

然后可以来看一下在ip addr中多了什么东西

tunl0不用关注，这个是之前的ipip的。有两个cali接口，说明有两个pod。最重要的是有一个vxlan.calico

![image-20230429142744430](../assets/image-20230429142744430.png)

我们来详细看一下vxlan.calico

```sh
ip -d addr show vxlan.calico
```

可以看到这个vxlan使用的id是4096，local的ip就是10.0.0.21（绑定了enp0s3这个出口），然后源端口是随机的，目的端口是4789

![image-20230429142929205](../assets/image-20230429142929205.png)

ip route看一下啊

因为我们使用了CrossSubnet，所以在woker2和master1，直接走的物理接口，去往不同网段的就会走这个tunl。

```sh
ip route
```

![image-20230429143300736](../assets/image-20230429143300736.png)

可以看到，这个10.244.73.0应该就是在worker3上的vxlan.calico的这个tunl的ip了，然后我们可以用ip neigh来看一下这个ip，主要可以看到他的mac地址。

```sh
ip nei | grep 10.244.73.0
```

![image-20230429143428935](../assets/image-20230429143428935.png)

也可以看一下这个mac地址从哪里来看

```sh
bridge fdb show | grep 66:0a:4a:c5:63:05
```

![image-20230429143538612](../assets/image-20230429143538612.png)

接着我们就来ping了。

因为这次部署，client跑到了worker3上，所以我们用worker1上的test和worker2上的test跑在同一网段的，用worker3上的client与worker1或worker2上test跑不同网段的。

### worker1 -> worker2

首先来看一下worker1上这个pod的内容

```sh
kubectl exec -it testappds-c8tdf -- ip route
kubectl exec -it testappds-c8tdf -- ip addr
kubectl exec -it testappds-c8tdf -- ip nei
```

tunl0是ipip的时候创建的。这里可以不用关注

![image-20230429143834877](../assets/image-20230429143834877.png)

我们来ping worker2上的内容

```sh
kubectl exec -it testappds-c8tdf -- ping -c 1 10.244.42.3
```

这里就不重复讲arp的内容了。

首先看一下出去的test pod。

```sh
tcpdump -eni cali9a7186820ea icmp
```

自己的mac，到达对端veth peer的mac（8对e），然后是10.244.204.2 到达 10.244.42.3

![image-20230429144209276](../assets/image-20230429144209276.png)

包到了worker1的路由上，去掉mac，查路由，查到路由`10.244.42.0/24 via 10.0.0.22 dev enp0s3 proto 80 onlink`找到了这条路由，直接从enp0s3扔出去。自己的mac就拿enp0s3封装，对端的mac就是ip neigh的这个10.0.0.22这个mac

于是到enp0s3来查一下，ip就不动了，封装一下源mac和目的mac就可以出去了

```sh
tcpdump -eni enp0s3 icmp
```

![image-20230429144538500](../assets/image-20230429144538500.png)

到达了worker2上，首先剥掉mac层， 因为mac地址就是自己的。

```sh
tcpdump -eni enp0s3 icmp
```

接着就去查路由表是42.3

![image-20230429144627780](../assets/image-20230429144627780.png)

查到了`10.244.42.3 dev cali9fff596671e scope link`，然后就走这个出去，然后我们就要去查10.244.42.3的mac地址，就可以通过ip neigh查找这个mac地址，源mac用8对e，目的mac就用这个。然后最后发过来。

![image-20230429144805595](../assets/image-20230429144805595.png)

![image-20230429153532844](../assets/image-20230429153532844.png)

![image-20230429153542450](../assets/image-20230429153542450.png)

![image-20230429153555236](../assets/image-20230429153555236.png)

### worker1 -> worker3

接着我们来看要ping worker3上的client的过程

```sh
kubectl exec -it testappds-c8tdf -- ping -c 1 10.244.73.2
```

首先是从worker1的veth peer的对端出去，

```sh
tcpdump -eni cali9... icmp
```

这一步与上面是相同的。

![image-20230429152036383](../assets/image-20230429152036383.png)

包到了worker1上以后，就要去查路由表

```sh
ip route
```

![image-20230429152111474](../assets/image-20230429152111474.png)

发现命中了`10.244.73.0/24 via 10.277.73.0 dev vxlan.calico onlink`这条路由，下一跳就是10.244.73.0，出接口是vxlan的这个接口

```sh
ip -d addr show vxlan.calico
```

![image-20230429152224924](../assets/image-20230429152224924.png)

就按照vxlan的封装来，内层的ip是10.244.204.2 > 10.244.73.2，源mac就是vxlan.calico的mac，目的mac就是下一跳的mac。里层payload就封装完了，就开始封装vxlan部分，id是4096，udp端口就是4789。外层的ip就是locol的ip，目的的ip就是告诉我对端关联的mac的那个ip。

![image-20230429152505077](../assets/image-20230429152505077.png)

看一下worker1上的出口

```sh
tcpdump -eni enp0s3 udp port 4789
```

![image-20230429152718684](../assets/image-20230429152718684.png)

最里面的是icmp，外部有一层mac，这个mac是vxlan的mac，前面这些是payload，再外边就是vxlan的封装，源ip是vxlan的local的ip，目的ip是查出来的关联ip，封装就完成了，最外层的mac就是enp0s3出去的mac了。

然后在worker3上接收了相关的内容。

```sh
tcpdump -eni enp0s3 udp port 4789
```

![image-20230429153239058](../assets/image-20230429153239058.png)

然后就去找4789，交给vxlan处理了，看到内层的mac是vxlan.calico接口的mac，就会往这个接口送，做完了就会查最里层的ip，就从calia.....这个接口出去了，就找他的目的mac，然后发到这个里面就行了。

![image-20230429153612659](../assets/image-20230429153612659.png)

![image-20230429153627318](../assets/image-20230429153627318.png)

![image-20230429153639285](../assets/image-20230429153639285.png)

## Calico-default-ibgp-full-mesh

首先修改配置

```yaml
- name: CALICO_IPV4POOL_IPIP
  value: "Never"
- name: CALICO_IPV4POOL_VXLAN
  value: "Never"
- name: CALICO_IPV6POOL_VXLAN
  value: "Never"
```

apply这个yaml

```sh
kubectl apply -f 4.0 calico-bgp.yaml
```

### 同一个网段的pod之间的通信

可以看到这个是由pod testapp-s2mx5向pod testapp-dqwjn发起请求。

![image-20230430185640122](../assets/image-20230430185640122.png)

首先，同样的是由veth peer的eth0端往对端cali....这端发送icmp请求，其中ip就是目的ip和源ip，即两个pod的ip，然后源mac地址就是自己的mac地址，而目的mac就是veth peer对端的cali...的mac地址，这个如果没有就发arp request，广播得到。然后将内容发到cali...后，发现是自己的mac地址，就解开mac封装，拿到目的ip和源ip，查路由表，找到下一跳的地方，从下一跳发出去。这个下一跳是通过ibgp学习得到的。接着icmp包到了worker2的enp0s3上以后，发现是自己的mac地址，就会解开mac封装，拿到了ip地址以后，就去查路由表，发现要从cali...这个veth peer出去，然后这个cali...就会去找对端的eth0，如果没有的话，就要发一个arp request广播来找，收到了的话，就封装到mac里面发出去。

![image-20230430190444491](../assets/image-20230430190444491.png)

接下来是回发的内容。与上面的过程是一样的。

![image-20230430204848962](../assets/image-20230430204848962.png)

### 不同网段的pod之间的通信

这个是worker1去ping worker3的整个过程。

![image-20230430204925389](../assets/image-20230430204925389.png)

这个的过程和同网段的pod之间的通信是很像的，只是最后一步从worker1的enp0s3口出去，但是到达了leaf交换机上以后，发现没有这个相关的路由，因为学习ibgp的内容的是worker1、worker2、worker3、master1，而leaf1和leaf2还有spine，没有学习到相关的内容。

![image-20230430205014175](../assets/image-20230430205014175.png)

## Calico-bgp-as-per-rack

具体的配置如下

```yaml
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
# 配置master1内容
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: master1.k8s.com
spec:
  bgp:
    asNumber: 65001
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
# 配置worker1的内容
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: worker1.k8s.com
spec:
  bgp:
    asNumber: 65001
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
# 配置worker2的内容
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: worker2.k8s.com
spec:
  bgp:
    asNumber: 65001
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
#配置worker3的内容
apiVersion: projectcalico.org/v3
kind: Node
metadata:
  name: worker3.k8s.com
spec:
  bgp:
    asNumber: 65002
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

这个配置和下面是示例有点不同，但是意思是相同的。

配置好以后，整个架构如下。首先spine交换的asn number是65000，ebgp有两个交换机，swp1和swp2。然后交换机swp1的ip是10.0.0.1，配置了swp1的ibgp有三个（上面的配置里只有两个的）10.0.0.21、10.0.0.22、10.0.0.23这三个，然后每个主机都配置好，对于swp1的ebgp就是swp2。对于swp2，ibgp就是40.0.0.24，ebgp就是swp1。

![image-20230430213640270](../assets/image-20230430213640270.png)

因为同一网段内的通信与上面是完全相同的，所以这里就只要分析不同网段里的内容。图片上的内容写错了，第二个Host应该是worker4，因为上面的架构中，只有worker4是在不同的网段内的。

在这里我们可以看到，这里与上面的默认配置最主要去区别，就是下面这个的区别。

![image-20230430215004917](../assets/image-20230430215004917.png)

bpf1、bpf2、bpf3都接在了10.0.0.1上面。

![image-20230430214549085](../assets/image-20230430214549085.png)

下面来分析一下整个发送的流程。

首先内容从pod里发出来，从eth0口出来，发送广播来获取cali12...这个的mac地址，拿到mac地址以后就可以进行mac的封装了，然后发送到了cali12...，接着查路由表找到出口，是从enp0s3出去，目的mac就是交换机leaf1的mac，然后因为是使用as per rack的学习方式，所以，交换机有学习到整个的内容，然后交换机走到spine，再走到leaf2，最后走到bpf4上面，bpf4上面发现mac地址是自己的，于是解封装，发现发往的地址在cali62...口上，于是再查他的ip neigh，如果没有的话，就arp 广播一下，找到了mac地址，就发过去了，然后进行处理。

![image-20230430215113909](../assets/image-20230430215113909.png)

下面是回复的过程，这个过程是一样的。

![image-20230430215429437](../assets/image-20230430215429437.png)

## Calico-bgp-as-per-computer

这个与per rack唯一不同的就是asn的分布。然后这个我们要使用ebgp，在每个node上面也分配asn number。

下面可以看一下整个过程的拓扑

![image-20230430221712220](../assets/image-20230430221712220.png)

修改之前的配置，具体修改以后如下

```yaml
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

配置好后，可以来看一下现在的样子

```sh
calicoctl get node -o wide
```

![image-20230430222229548](../assets/image-20230430222229548.png)

交换机刚刚配置的是ibgp，所以需要对leaf1做一下修改。

可以来看一下leaf1的配置

```sh
net show configuration bgp
```

![image-20230501001235479](../assets/image-20230501001235479.png)

把原来的配置删除，增加新的配置

```sh
net add bgp neighbor servers remote-as external
net del bgp neighbor servers remote-as internal
net commit
```

可以来查看bgp的summary

```sh
net show bgp summary
```

![image-20230501001455541](../assets/image-20230501001455541.png)

可以来看一下worker2学到的路由

```sh
ip route show proto bird
```

![image-20230501001648148](../assets/image-20230501001648148.png)

可以看到配置改变了以后的样子

![image-20230501001843875](../assets/image-20230501001843875.png)

![image-20230501002143493](../assets/image-20230501002143493.png)

![image-20230501002155780](../assets/image-20230501002155780.png)

![image-20230501002202730](../assets/image-20230501002202730.png)

![image-20230501002211201](../assets/image-20230501002211201.png)
