## Linux host

Linux host(Network protocols stack) is a Multi-Function Router/Switch/Firewall

物理网卡就是物理接口

网络名词空间一个个单独隔离出来的终端设备，分配在不同的网络名称空间

tun/tap设备连接到网络协议栈（类路由器交换机）

可以在上面创建bridge（连接同一网段的设备）

veth peer把本地的一些属于不同namespace的设备连接到网络协议栈

iptables 实现防火墙相关的功能

![image-20230424110543675](../assets/image-20230424110543675.png)

* 深色的小圆圈NIC就是物理网卡
* tap和tun设备是虚拟网卡，tab是二层是，tun是三层的
* 两端带有小点的线是veth peer接口
* 程序通过调用socket api这一层发出去的（目的ip、目的端口、协议）
  通过ARP实现，网关有没有，下一跳有没有什么的
* 虚拟机连接过来，要虚拟网卡，常见的是给一个tap设备。tap设备是直接放在bridge上的。
* pod是容器的镜像文件，不需要驱动，有一个veth peer就可以了。
* pod F和pod H通过一个bridge连接，他们在同一网段里，bridge作为他们的网关
* pod c通过veth peer 直接连接到协议栈，它没有跟他在同一网段的设备。走的是纯路由的功能
* pod v直接使用网卡的功能，为了性能的考虑。
* openvpn连接的tun设备，vpn工作的流程：会给网络协议栈中生成一条路由，通常是默认路由，用来指向公司的防火墙设备。在网络协议栈里的默认路由的下一条就是tun设备，出来以后就会被openvpn读到。application（比如打开网站）发起请求，网络协议栈收到请求，查路由，发现下一条是tun设备，就向tun设备扔出请求，openvpn就知道要把包发到什么地方去，又会把发给tun设备的数据作为payload，又重新再调用一次socket的api，这次调用的socket api的目的ip就是公司的防火墙ip。封装一次之后，原来可能是http的payload，发给tun设备后，就成了ip的包，因为tun是三层的虚拟网卡设备，ip的包又被openapi当初payload调用socket api，这个目的ip就变成了公司防火墙的ip，这个时候再去查下一条的时候，就会匹配到明细的路由。回包的逻辑：目的端口是一个特殊的端口，是openvpn监听的端口，网络协议栈直接给openvpn，openvpn看到是自己的包，就解封装，解掉上层的tcp、udp包，发现是内部的，就会通过tun扔进来，网络协议栈看到网络IP是自己的，看到端口是原来的application，就会把内容再发给那个application，这样实现了交互。

## 结构

![image-20230424130319584](../assets/image-20230424130319584.png)

worker1、worker2在192.168.210.0网段

worker3在192.168.230.0网段

## 实验

1. 实验一，两个pod在同一个主机，通过bridge连起来
2. 实验二，两个pod直接插到网络协议栈里面
3. 实验三，连接到主机上的pod通过bridge如何访问外网的
4. 实验四，client pod 和 service pod在同一个主机上，是如何通过cluster ip来访问它
5. 实验五，通过外网的设备访问我们的node pod service
6. 三四五讲iptables如何实现service
7. 六、七 IPIP模块 和 VxLan模块如何互通的
8. 八通过静态路由
9. 在主机上部署bird（BGP），与交换机学习路由

### Hand CNI - 1

（container-to-container）

![image-20230424152323009](../assets/image-20230424152323009.png)

- 创建一个bridge
- 创建两个namespace
- 创建两对veth peer

首先创建两个namespace

![image-20230424151425145](../assets/image-20230424151425145.png)

```shell
ip netns list
ip netns add netns1
ip netns add netns3
ip netns list
```

一开始只有LOOPBACK

```shell
ip netns exec netns1 ip -d link show
```

![image-20230424151818277](../assets/image-20230424151818277.png)

```shell
ip netns exec netns3 ip -d link show
```

![image-20230424151845926](../assets/image-20230424151845926.png)

接着把loopback0 up起来

```shell
ip netns exec netns1 ip link set lo up
ip netns exec netns3 ip link set lo up
ip netns exec netns1 ip -d link show
ip netns exec netns3 ip -d link show
```

![image-20230424152038816](../assets/image-20230424152038816.png)

创建veth peer

```shell
ip link add dev veth01 type veth peer name veth1
ip link add dev veth03 type veth peer name veth3
ip -d link show
```

![image-20230424152635697](../assets/image-20230424152635697.png)

接下来我们将veth peer放入namespace，并且重命名

```shell
ip link set dev veth01 netns netns1
ip link set dev veth03 netns netns3

ip netns exec netns1 ip link set dev veth01 name eth0
ip netns exec netns3 ip link set dev veth03 name eth0

ip -d link show
```

![image-20230424153027252](../assets/image-20230424153027252.png)

```shell
ip netns exec netns1 ip -d link show
ip netns exec netns3 ip -d link show
```

![image-20230424153119397](../assets/image-20230424153119397.png)

接下来是给两个namespace的eth0配置ip

现在有ip了，但是还没有up，这里没有up是因为他的对端veth1没有up

```shell
ip netns exec netns1 ip addr add 192.168.10.1/24 dev eth0
ip netns exec netns3 ip addr add 192.168.10.3/24 dev eth0

ip netns exec netns1 ip link set dev eth0 up
ip netns exec netns3 ip link set dev eth0 up

ip netns exec netns1 ip -d addr show 
ip netns exec netns3 ip -d addr show 
```

![image-20230424153441738](../assets/image-20230424153441738.png)

接下来，我们创建bridge，并把对端的veth1和veth3放入bridge，再将他们和bridge自己up起来

```shell
ip link add dev br0 type bridge

ip link set dev veth1 master br0
ip link set dev veth3 master br0

ip link set dev veth1 up
ip link set dev veth3 up

ip link set br0 up

ip -d link show
brctl show
```

![image-20230424153942887](../assets/image-20230424153942887.png)

最大传输单元MTU（Maximum Transmission Unit，MTU），是指网络能够传输的最大数据包大小，以字节为单位。

接下来设置一下mtu

```shell
ip link set br0 mtu 9150
ip link set dev veth1 mtu 9150
ip link set dev veth3 mtu 9150
ip -d link show

ip netns exec netns1 ip link set dev eth0 mtu 9150
ip netns exec netns3 ip link set dev eth0 mtu 9150
ip netns exec netns1 ip -d addr show
ip netns exec netns3 ip -d addr show
```

![image-20230424154310861](../assets/image-20230424154310861.png)

![image-20230424154250185](../assets/image-20230424154250185.png)

开始测试

首先看一下ip的addr和整个路由表

```shell
ip netns exec netns1 ip addr show
ip netns exec netns1 ip neigh
ip netns exec netns1 ip route
```

![image-20230424155005628](../assets/image-20230424155005628.png)

```shell
ip netns exec netns3 ip addr show
ip netns exec netns3 ip neigh
ip netns exec netns3 ip route
```

![image-20230424155105572](../assets/image-20230424155105572.png)

因为只是刚刚接起来，所以只有地址相关的信息，而路由和neigh都没有。具体的情况可以看上面的图。

接下来可以ping一下彼此，然后看看路由表和neigh会如何变化的

```shell
ip netns exec netns1 ping 192.168.10.3 -c 3
ip netns exec netns3 ping 192.168.10.1 -c 3

ip netns exec netns1 ip neigh
ip netns exec netns3 ip neigh
```

![image-20230424155350076](../assets/image-20230424155350076.png)

双方彼此ping三次

然后查看neigh

![image-20230424155445254](../assets/image-20230424155445254.png)

如果遇到问题，可能会用到的命令如下：

```shell
#A#5.2 Fix issue caused by iptables/docker or /proc/sys/net/ipv4/ip_forward

iptables -F
iptables -I FORWARD -j ACCEPT
iptables -S

echo 1 > /proc/sys/net/ipv4/ip_forward

iptables -vnL
```

### Hand CNI - 2

（container-to-container）

![image-20230424225018132](../assets/image-20230424225018132.png)

- 没有使用bridge
- 配置路由
- 这个时候一般会用32位的ip
- veth在主机协议栈这边也可以配ip，为了让pod做路由下一跳的解析，pod要去别的网段，就要去问一下10.254的mac地址，就可以发给网络协议栈了。

> 下面有三个option

- option1

  网关ip直接用主机的这个，直接给veth1和veth3配一个ip

  在namespace里面，把网关指定为配置的这个ip

- option2

  这个使用的是Calico的实现

  首先要配一个路由，在namespace中指向169.254.1.1这个ip

  而169.254.1.1在主机上没有ip，但是要确保在主机上有一条路由169.254.0.0/16，通过enp0s3出去

  然后再启用veth上的proxy_arp，让他可以去回复pod发过来的arp的request

- option3

  这个使用的是Cilium的实现

  创建另外一对veth peer，向host和net

  把ip放到host上去，然后up起来

  然后再将这个ip指定为网关

**option1、2、3的对比**

option1的话，是给在网络协议栈上的veth1和veth3给他们一个IP地址，然后将两个namespace添加默认路由为这个地址

option2的话，给两个namespcae添加一个地址，但是这个地址在主机上是没有ip的，所以要将他绑给主机上的enp0s3再出去，这种方式要开启proxy_arp

option3的话再创建一个veth peer，一端叫veth-host，另一端叫veth-net，这两端都放在协议栈上，veth-host有一个ip，这个ip就是各个pod的网关ip。linux主机在默认的情况下，如果收到一个arp，需要解析这个ip，在这个主机的一个接口上面，但是收到arp请求的接口，不是这个有IP的接口，比如说pod（192.168.10.1）发了一个arp请求去解析10.254的这个ip的mac，主机协议栈就会回一个arp request，回的mac地址就是收到这个arp的接口的mac地址

首先创建两个namespace

```shell
ip netns add netns1
ip netns add netns3
ip netns list
ip netns exec netns1 ip -d link show
ip netns exec netns3 ip -d link show
```

![image-20230424232248203](../assets/image-20230424232248203.png)

![image-20230424232329948](../assets/image-20230424232329948.png)

接着将他们的loopback都up起来

```shell
ip netns exec netns1 ip link set lo up
ip netns exec netns3 ip link set lo up
ip netns exec netns1 ip -d link show
ip netns exec netns3 ip -d link show
```

![image-20230424232454826](../assets/image-20230424232454826.png)

然后给他们添加veth peer

```shell
ip link add dev veth01 type veth peer name veth1
ip link add dev veth03 type veth peer name veth3
ip -d link show
```

![image-20230424233005127](../assets/image-20230424233005127.png)

接下来将veth peer分配给指定的namespace，并把另一端的名称都改为eth0

```shell
ip netns exec netns1 ip link set dev veth01 name eth0
ip netns exec netns3 ip link set dev veth03 name eth0

ip -d link show

ip netns exec netns1 ip -d link show
ip netns exec netns3 ip -d link show
```

![image-20230425002043124](../assets/image-20230425002043124.png)

![image-20230425002118947](../assets/image-20230425002118947.png)

接下来是给两个namespace分配ip地址

```shell
ip netns exec netns1 ip addr add 192.168.10.1/32 dev eth0
ip netns exec netns3 ip addr add 192.168.10.3/32 dev eth0
```

![image-20230425002353457](../assets/image-20230425002353457.png)

再将veth peer两端都up起来

```shell
ip netns exec netns1 ip link set dev eth0 up
ip netns exec netns3 ip link set dev eth0 up

ip link set veth1 up
ip link set veth3 up
```

![image-20230425002442026](../assets/image-20230425002442026.png)

接着我们使用option1，我们在veth peer的协议栈端，配置ip，并且将网关设置为这个ip

```shell
ip addr add 192.168.10.254/32 dev veth1
ip addr add 192.168.10.254/32 dev veth3
```

![image-20230425002928904](../assets/image-20230425002928904.png)

添加路由

```shell
ip netns exec netns1 ip route add default via 192.168.10.254 dev eth0 onlink
ip netns exec netns3 ip route add default via 192.168.10.254 dev eth0 onlink
```

![image-20230425003024764](../assets/image-20230425003024764.png)

配置完成后，我们可以来看一下路由

可以看到，这个是netns1的路由情况

![image-20230425003156332](../assets/image-20230425003156332.png)

下面是netns3的路由情况

![image-20230425003231594](../assets/image-20230425003231594.png)

除了在pod上面添加了默认路由，也要在主机上添加路由指向pod。因为这里都是32位的ip地址，ip neigh不在同一网段，不会自动生成ip neigh，要添加路由告诉他是通过哪个接口过去的。

在主机ip里添加指向pod的路由

```shell
ip route add 192.168.10.1/32 dev veth1
ip route add 192.168.10.3/32 dev veth3
ip route
```

![image-20230425003541680](../assets/image-20230425003541680.png)

测试如下

接下来我们可以互相ping了

```shell
ip netns exec netns1 ping 192.168.10.3 -c 3
ip netns exec netns3 ping 192.168.10.1 -c 3
```

![image-20230425003745641](../assets/image-20230425003745641.png)

ping 完以后我们可以再来看一次他们的neigh

```shell
ip netns exec netns1 ip neigh
ip netns exec netns3 ip neigh
```

![image-20230425003830351](../assets/image-20230425003830351.png)

> 路由和bridge的最大的区别就是ip地址的划分，要添加默认路由了。主机上也要知道pod的ip，calico会直接生成一条这个路由，会生成这么一个路由。

### Hand CNI - 3

（container-to-outside）

![image-20230425010850727](../assets/image-20230425010850727.png)

当有一个具有公网 IP 地址的 Linux 路由器连接到一个局域网中，该路由器需要配置网络地址转换 (NAT) 规则，以便局域网中的计算机可以访问互联网。以下是该命令的具体流程：

1. `iptables` 命令将新规则插入到 `nat` 表格的 `POSTROUTING` 链的开头，因此该规则将会首先被检查。
2. 当一个数据包经过该路由器的网络接口时，Linux 内核会按照预定义的网络包过滤规则查找该数据包需要执行的动作。
3. 如果该数据包的源 IP 地址属于 `192.168.10.0/24` 子网并且该数据包不是从 `br0` 接口发出的，该数据包将会匹配到 `iptables` 新插入的规则。
4. 对于匹配到的数据包，该规则会将该数据包的源 IP 地址修改为该路由器所使用的出口接口的 IP 地址。
5. 修改后的数据包将继续被发送到其目的地。
6. 当返回的数据包到达该路由器时，内核会使用 `iptables` 的另一个规则，将目的地址修改为源地址，并将数据包转发回正确的计算机。此时，该数据包的目的 IP 地址会是 `192.168.10.0/24` 子网中的某个计算机的 IP 地址。

通过这种方式，`iptables` 可以实现网络地址转换，使得局域网内的计算机可以通过路由器的公网 IP 地址访问互联网，同时保护内网的安全性。

pod 连接到了一个bridge，出enp0s9访问leaf03的时候就要经过这个net才行

在出enp0s9，访问leaf03的192.168.210.254/24的时候要出net

首先要给bridge添加ip地址 并用netns1 ping看看能不能ping通

```shell
ip addr add 192.168.10.254/24 dev br0

ip netns exec netns1 ping 192.168.10.254 -c 3
ip netns exec netns3 ping 192.168.10.254 -c 3
```

![image-20230425014810780](../assets/image-20230425014810780.png)

接着要给两个pod分别添加默认路由，因为Hand CNI - 1 的时候是同一个网段里面的，不需要往外走，所以不用加默认路由，可以先看一下添加默认路由前netns1的路由情况

![image-20230425014924812](../assets/image-20230425014924812.png)

这个是添加默认路由之前的。（这个路由感觉就是因为设置了24位有效才有的吧？）

然后我们添加默认路由，把bridge添加为默认路由

```shell
ip netns exec netns1 ip route add default via 192.168.10.254 dev eth0
ip netns exec netns3 ip route add default via 192.168.10.254 dev eth0

ip netns exec netns1 ip route
ip netns exec netns3 ip route
```

然后再来看一下路由，可以看到，这里添加了一个路由

![image-20230425015256089](../assets/image-20230425015256089.png)

但是这样还是不能出去，我们可以ping了试一下（肯定不能啊，那边都没连起来）

其实包是能出去到leaf上去的，因为主机有192.168.210.254这一段的IP，但是交换机回不了。

netns1一直在ping

![image-20230425015615366](../assets/image-20230425015615366.png)

然后打开交换机，可以看一下，首先看交换机的所有接口

```shell
net show interface
```

![image-20230425015840410](../assets/image-20230425015840410.png)

```shell
sudo tcpdump -eni vlan210
```

然后，我们与交换机相通的是192.168.210.254/24这个口，即vlan210，可以看到，在交换机上，一直在接收请求。

![image-20230425020031926](../assets/image-20230425020031926.png)

可以看下ip 的route，发现是没有这个路由的

![image-20230425020130415](../assets/image-20230425020130415.png)

所以需要在流量出去的时候让他做一下nat，可以看一下现在，是没有任何一条nat的，看到的这些大部分都是默认的

```shell
iptables -vnL -t nat
```

![image-20230425020410162](../assets/image-20230425020410162.png)

我们可以配一条nat的rule，源地址是192.168.10.0/24这个的网段的，如果出接口不是br0，就要通过enp0s9出去了，就要做MASQUEREAD。可以在Chain POSTROUTING里看到加了这个MASQUEREAD，这样的话出去的时候就要做nat了

```shell
iptables -I POSTROUTING -t nat  -s 192.168.10.0/24 ! -o br0 -j MASQUERADE
iptables -vnL -t nat
```

![image-20230425020737643](../assets/image-20230425020737643.png)

我们再来ping一下，是可以通的

![image-20230425021507568](../assets/image-20230425021507568.png)

再到交换机来看一下，可以看到用了192.168.210.21这个ip

![image-20230425022038140](../assets/image-20230425022038140.png)

可以看到，其实走的就是enp0s9的ip

![image-20230425022140912](../assets/image-20230425022140912.png)

刚刚过程的整个流程如下

就是找到了对应规则，然后把包交给出口的地点，这里对应的就是enp0s9

![image-20230425022331594](../assets/image-20230425022331594.png)

接下来我们来过滤一下ICMP的内容，这里就有详细写了，源src是192.168.10.1.目的是192.168.210.254，回来的源是192.168.210.254，回来的目的是192.168.210.21

```shell
conntrack -L | grep icmp
```

![image-20230425022523254](../assets/image-20230425022523254.png)

### Hand CNI - 4

（container[client]-to-ClusterIP-container[server]）

![image-20230425132948487](../assets/image-20230425132948487.png)

- 在同一个节点上面有一个client pod 和 一个service pod
- client pod 访问 其他的pod的时候通常是通过clusterip
- 这种访问的情况也要借助IP tables的规则

完成这个实验前要完成Hand CNI - 1，这个实验是直接在这个机器上跑的。

首先，要在192.168.10.3:80端口跑起来一个http server，作为server端，跑起来以后，我们可以看看他是不是监听了。

```shell
ip netns exec netns3 python3  -m http.server --bind 192.168.10.3 80 &
sleep 10
ip netns exec netns3 ss -tlnp
```

![image-20230425134302613](../assets/image-20230425134302613.png)

然后，我们给bridge增加ip

```shell
ip addr add 192.168.10.254/24 dev br0
```

我们先在主机上去curl一下

```shell
curl 192.168.10.3:80
```

![image-20230425134510070](../assets/image-20230425134510070.png)

然后再在client（netns1）上去curl一下，都是可以跑通的

```shell
ip netns exec netns1 curl 192.168.10.3:80
```

![image-20230425134625648](../assets/image-20230425134625648.png)

接下来，我们要用pod去curl我们假装出来的service ip和端口的话，是根本不可能的，因为我们根本没有这个ip和端口，就可能直接出去了。而cluster ip是自己私有的。所以我们要做一个iptable的规则，让这个cluster ip 和 端口（11.96.10.3:8080）DNAT成我们netns3的ip 和 端口。如果Linux主机也想访问，要把类型设置为OUTPUT类型。

```shell
iptables -I OUTPUT      -t nat -d 11.96.10.3 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.10.3:80  
iptables -I PREROUTING  -t nat -d 11.96.10.3 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.10.3:80 

iptables -S -t nat
```

![image-20230425135616712](../assets/image-20230425135616712.png)

接下来我们来ping一下试试看，发现是不行的，其实是因为我们没有添加默认路由，要跟Hand CNI - 3一样，把默认路由添上，我们才会走bridge这个方式

```shell
ip netns exec netns1 curl 11.96.10.3:8080
curl 11.96.10.3:8080

iptables -vnL -t nat
```

![image-20230425140105281](../assets/image-20230425140105281.png)

添加默认路由

```shell
ip netns exec netns1 ip route add default via 192.168.10.254 dev eth0
ip netns exec netns3 ip route add default via 192.168.10.254 dev eth0
```

然后就可以curl 通了

![image-20230425140401568](../assets/image-20230425140401568.png)

![image-20230425140420578](../assets/image-20230425140420578.png)

另外，如果要看主机通过什么出去了可以使用这个命令

```shell
ip route get xxxx
```

整个的流程如下

![image-20230425140843224](../assets/image-20230425140843224.png)

### Hand CNI - 5

(outside-to-NodePort-container[service])

![image-20230425144022262](../assets/image-20230425144022262.png)

kubernetes外部访问服务，我们一般是使用NodePort这种service来做的。同样，我们也要做DNAT，把访问我们enp0s9的这个某一个端口，把他DNAT我们service的一个端口。

流向其实没有很大的区别，不过是从NIC反过来访问的。

![image-20230425144304611](../assets/image-20230425144304611.png)

其实这个的本质就是我们在IP table里面新建一个对应的规则就可以了。我们可以先到交换机上，来curl一下，这肯定是失败的，根本没有相关端口的定义。

![image-20230425145733937](../assets/image-20230425145733937.png)

接下来，我们创建相关的IP table

```shell
iptables -I PREROUTING  -t nat -d 192.168.210.21 -p tcp -m tcp --dport 32080 -j DNAT --to-destination 192.168.10.3:80 
iptables -I OUTPUT      -t nat -d 192.168.210.21 -p tcp -m tcp --dport 32080 -j DNAT --to-destination 192.168.10.3:80  

iptables -vnL -t nat
```

![image-20230425145850360](../assets/image-20230425145850360.png)

接着我们再用交换机来curl一下

就可以访问到相关的内容了

![image-20230425145931515](../assets/image-20230425145931515.png)

这样配置后，主机自己也可以访问的

![image-20230425150123102](../assets/image-20230425150123102.png)

### Hand CNI - 6

（container-to-container）

![image-20230425150912005](../assets/image-20230425150912005.png)

在每个节点上面，创建一个tunl的接口

再把需要发往其他节点上的pod的流量，走tunl

这里是用IPIP实现的

worker1 、 worker2在同一个网段，worker3在另外一个网段，worker1、worker2直接路由就可以了，去worker3的话就必须要走tunl了。

首先分别在worker1、worker2、worker3上创建namespace，把它的loopback0给up起来，创建出来一个veth peer

```shell
ip netns add netns1
ip netns list

ip netns exec netns1 ip -d link show
ip netns exec netns1 ip link set lo up
ip netns exec netns1 ip -d link show

ip link add dev veth01 type veth peer name veth1
ip -d link show
```

![image-20230425152231116](../assets/image-20230425152231116.png)

接着把veth peer一端放到namespace里去，并且把两端都up起来。

```shell
ip link set dev veth01 netns netns1

ip netns exec netns1 ip link set dev veth01 name eth0

ip link set veth1 up
ip netns exec netns1 ip link set dev eth0 up

ip -d link show
ip netns exec netns1 ip -d link show
```

![image-20230425152459973](../assets/image-20230425152459973.png)

接着给worker1、worker2、worker3分别配上ip

```shell
ip netns exec netns1 ip addr add 192.168.10.1/24 dev eth0
```

![image-20230425152817590](../assets/image-20230425152817590.png)

接下来创建bridge，把veth peer一端放到bridge里去，并且把这个bridge给up起来。

```shell
ip link add dev br0 type bridge

ip link set dev veth1 master br0

ip link set br0 up

ip -d link show
brctl show
```

![image-20230425153055298](../assets/image-20230425153055298.png)

接下来给worker1、worker2、worker3分别配上他们bridge的IP地址，配完后可以ping一下，然后将这个添加到netns1的默认路由里面，可以再看一下路由表

```shell
ip addr add 192.168.20.254/24 dev br0

ip netns exec netns1 ping 192.168.20.254 -c 3

ip netns exec netns1 ip route add default via 192.168.20.254 dev eth0

ip netns exec netns1 ip route
```

![image-20230425153546788](../assets/image-20230425153546788.png)

接着，为每个worker添加tunl，注意配置的是ipip的tunl，他的ip就使用我们物理接口enp0s9，这个接口，下面以worker2为例

```shell
ip tunnel list
ip link add dev ipip0 type ipip local 192.168.210.22
ip tunnel list
```

![image-20230425153935900](../assets/image-20230425153935900.png)

tunl0是默认的一个device，locol = any，remote = any。如果没有人接收，他就会负责接收。

接着把这个tunl ipip0给up起来，并且给他一个ip

显示出来可以看到，remote是any，local是enp0s9的ip

```shell
ip link set dev ipip0 up
ip addr add 192.168.20.2/32 dev ipip0

ip -d addr show ipip0
```

![image-20230425154544749](../assets/image-20230425154544749.png)

接下来我们要配置路由，如果是同一个网段的话，我们只要配置对端网段的ip就可以了，不需要做封装。如果要跨网段，我们要通过ipip0的tunl出去

```shell
ip route add 192.168.10.0/24 via 192.168.210.21 dev enp0s9
ip route add 192.168.30.0/24 via 192.168.230.23 dev ipip0 onlink
ip route
```

![image-20230425155219587](../assets/image-20230425155219587.png)

然后就可以互通了，先来测一下worker1和worker2

这个还是不带ipip封装的

![image-20230425155604243](../assets/image-20230425155604243.png)

![image-20230425155621864](../assets/image-20230425155621864.png)

接着我们用worker2来ping一下worker3

![image-20230425162056504](../assets/image-20230425162056504.png)

![image-20230425162119742](../assets/image-20230425162119742.png)

ipip是一个三层的设备，是没有mac地址的。

VxLan是一个二层的设备，有mac地址。

### Hand CNI - 7

（container-to-container）

![image-20230425181407197](../assets/image-20230425181407197.png)

这个实验是在Hand CNI - 6的基础上做的

只是把用ipip实现的tunl换成了使用VxLan实现的tunl

首先要把Hand CNI - 6中创建的tunl所对应的路由给去除

这样才能让路由走我们新写的这个方式

```shell
ip route del 192.168.30.0/24 

ip route
```

接下来开始建VxLan tunnel，这个id需要自己指定，最好在两端用的是一样的id，端口使用4789，no learning是不学习mac地址。

然后我们给这个VxLan一个IP地址，并给他手工配一个MAC地址，如果不指定就是随机生成。这里只是为了后面好看信息。

```shell
ip link add vxlan100 type vxlan \
		id 100 \
		dstport 4789 \
		local 192.168.210.22 \
		dev enp0s9 \
		nolearning
```

![image-20230425182602943](../assets/image-20230425182602943.png)

创建成功后，我们修改ip和mac地址

```shell
ip addr add 192.168.20.4/32 dev vxlan100
ip link set dev vxlan100 address 00:00:ff:ff:20:04
ip link set dev vxlan100 up
ip -d addr show vxlan100
```

![image-20230425182657976](../assets/image-20230425182657976.png)

接着，我们要在worker2上，添加到worker3的路由

```shell
ip route add 192.168.30.0/24 via 192.168.30.4 dev vxlan100 onlink
ip route show
```

![image-20230425183246314](../assets/image-20230425183246314.png)

这样以后，我们如果要去30.1，我们可以看到下一跳是192.168.30.0/24 通过VxLan封装进去，VxLan本地是192.168.210.22这个ip。在VxLan中id是100，包从本地的pod过来的时候，外层的mac是已经被剥掉的，用的目的mac是br0mac，源mac是自己的mac。剥掉mac之后再来查路由，发现要去30.1，30.1查路由发现是走vxlan100这条路由，要构建vxlan封装，对一个IP包要进行vxlan封装，首先要封装它的mac，给一个内存的mac，源mac就是vxlan100的mac，那目的mac是什么，目的mac就是下一跳的mac，下一跳是192.168.30.4，但是我们没有30.4的这个mac，我们是不知道的，所以我们要手工添加这个ip neigh。

```shell
ip neigh add 192.168.30.4 dev vxlan100 lladdr 00:00:ff:ff:30:04
ip neigh show 
```

![image-20230425184811979](../assets/image-20230425184811979.png)

那么这样以后我们vxLan的内层的二层就已经有了，源MAC是自己的MAC，而目的MAC就是30.4的MAC，内层的包已经ok了，payload就ok了。vxlan的封装id有了，端口有了，VxLan的封装就结束了，对于外层的IP的封装，可以看到出接口是enp0s9，目的ip是多少，目的ip不知道，目的ip就要看ip neigh是从哪学过来的。就需要加一条bridge。是谁告诉我mac和ip的呢，是从dst 192.168.230.23得到的，这个ip能解封装mac是这样的一个报文。

```shell
bridge fdb add 00:00:ff:ff:30:04 dev vxlan100 dst 192.168.230.23
bridge fdb show dev vxlan100
```

![image-20230425185727687](../assets/image-20230425185727687.png)

这样配置完了以后，我们就可以做测试了

在worker3中，我们可以抓udp，也可以抓4789这个端口。

![image-20230425190136701](../assets/image-20230425190136701.png)

![image-20230425190055097](../assets/image-20230425190055097.png)

VxLan跟ipip区别不大，就是封装的时候会多一些， VxLan有一个别的优势，worker2到worker3的路径哈希是一样，如果流量很大的话，就不会哈希到网络上的不同路径，就只会哈希到一个span上了。而在VxLan中，不同的流可以对应不同的哈希，一个pod跟不同pod用的是不同的端口，这样属于不同的流，哈希出来的结果就会到不同的span上去，就可以均匀负载分担。

### OpenVPN

![image-20230425202833794](../assets/image-20230425202833794.png)

自己的IP是10.23.130.195，网关是10.23.130.1，然后vpn的IP是a.b.c.d

然后就可以拨vpn，拨通以后呢，本来的默认网关是10.23.130.1，现在的默认网关做了修改，有了一个优先级更加高的默认网关，指向了VPN。可以看到拨通vpn之前，默认网关是10.23.130.1，然后接口呢就是195这个IP。然后拨完vpn之后，默认路由变成了tunl设备的下一跳了。优先级会比原来的高。现在建立一个tun0 10.18.130.65，然后我们去往internet就是通过10.18.130.65这个tun0出去了。会有一个默认路由指向tun设备。

假如我们要从Chrome访问公司的某个网站，肯定有IP了，socket api进行封装，把这个包往tunl设备扔出去，tunl设备收到这个报文之后，openvpn程序就能读到，openvpn会把整个包作为一个payload，再次调用socket api，因为openvpn知道要往哪个ip去，调用socket api的时候就有一个具体的目的ip，这个目的ip可以匹配到一个明细路由，再从这个接口出去。协议栈就会把去往abcd的无线的网口扔出去了，从无线接口出去的包，达到发到internet的目的，出去的包外层的源ip是无线接口的ip，目的ip是abcd这个vpnset的这个ip。内层的ip是tunl设备的ip，这个tunl设备的ip也是vpn站点分配的ip，目的ip就是要访问的网站的ip。

![image-20230425203111805](../assets/image-20230425203111805.png)

### Hand CNI - 8

（container-to-container）

![image-20230425210325476](../assets/image-20230425210325476.png)

ipip和VxLan是overlay的方案，这两个都有封装和解封装的消耗。有时候如果我们能够和网络设备跑一些协议，让网络设备学到pod CIDR的内容的话，就可以使用这种方式。calico和cilium都是支持的。

静态路由的实验方案如下：

- 有三个节点，每个节点上都一个pod
- 有三个bridge，通过bridge接入进来，bridge就是他们的网关
- overlay的话如果在同一个网段就可以指向对端的IP地址就可以，如果在不同网段的就要指向tunl接口
- 现在对于静态的而言，我们需要在节点上配置静态路由，也要在leaf3上配置静态路由。

首先设置节点worker1、worker2出去的时候的静态路由

```sh
ip route add 192.168.30.0/24 via 192.168.210.254
```

![image-20230425212532989](../assets/image-20230425212532989.png)

再设置worker3回来的方式

```sh
ip route add 192.168.10.0/24 via 192.168.230.254
ip route add 192.168.20.0/24 via 192.168.230.254
```

但是这个时候leaf3的时候没有任何路由，所以我们要对这个静态路由也做设置

可以看到这个是设置之前的路由

![image-20230425213127894](../assets/image-20230425213127894.png)

接下来我们要设置一下他的静态路由

```sh
sudo ip route add 192.168.10.0/24 via 192.168.210.21
sudo ip route add 192.168.20.0/24 via 192.168.210.22
sudo ip route add 192.168.30.0/24 via 192.168.230.23
ip route
```

![image-20230425213257521](../assets/image-20230425213257521.png)

设置完后可以开始ping了，

![image-20230425213719571](../assets/image-20230425213719571.png)

![image-20230425213739840](../assets/image-20230425213739840.png)

可以说，这种方式其实是最好的方式，但是我们现在只有三个节点，非常方便设置，但是当我们的节点开始很多的时候，涉及的内容太多了，没有办法使用手工来解决。所以我们需要有第九个实验。

### Hand CNI - 9

![image-20230425214139035](../assets/image-20230425214139035.png)

Hand CNI - 9 与 Hand CNI - 8 两个的结构是一样的，本质上都是要写静态路由，而8是手写这个静态路由，而9是使用学习的方法来写。

首先安装`bird`

```sh
apt install bird
```

将worker2中`/etc/bird/bird.conf`修改为

```sh
# This is a minimal configuration file, which allows the bird daemon to start
# but will not cause anything else to happen.
#
# Please refer to the documentation in the bird-doc package or BIRD User's
# Guide on http://bird.network.cz/ for more information on configuring BIRD and
# adding routing protocols.

# Change this into your BIRD router ID. It's a world-wide unique identification
# of your router, usually one of router's IPv4 addresses.
router id 192.168.210.22;

# The Kernel protocol is not a real routing protocol. Instead of communicating
# with other routers in the network, it performs synchronization of BIRD's
# routing tables with the OS kernel.
protocol kernel {
        scan time 60;
        import all;
        export all;   # Actually insert routes learned via BIRD into the kernel routing table
}

# The Device protocol is not a real routing protocol. It doesn't generate any
# routes and it only serves as a module for getting information about network
# interfaces from the kernel.
protocol device {
        scan time 60;
}

protocol direct {
        interface "br0";
}

protocol bgp upstream {
   import all;
   export where source = RTS_DEVICE;

   local as 65002;
   source address 192.168.210.22;
   neighbor 192.168.210.254 as 65000;
}
```

然后重启bird，查看bird状态

```sh
systemctl restart bird
systemctl status bird
```

![image-20230425215254924](../assets/image-20230425215254924.png)

三个bird启动后，就可以自动学习设置了。

![image-20230425220836225](../assets/image-20230425220836225.png)

calico使用的就是bird，cilium也推荐可以使用这个。

```sh
# Check the route learned via BIRD/BGP on each node
birdc show protocols
birdc show route

# Check the route learned via BGP on leaf3
ip route
```

## 总结一下

同一个节点上的pod之间通信，通常就是bridge和routing

node之间通常有overlay，像ipip和VxLan。underlay通常用BGP，不会用静态的方式。

pod的内部，用service，cluster ip这种形式，是使用iptables的rule来做dnat

外部的设备访问k8s内部的service，是通过nodeport，用iptables的rule来做

pod 访问外网，MASQUEREAD的rule

### CNI

#### IPAM（IP address + Subnetmast，routes）

- Node level
- Cluster level

#### Connect Pod to others

- Pod to Pod on the same node
- Pod to Pod on the different nodes(Tunnel,Routing) mustn't have NAT
- Pod to Outside
- Pod to Service(ClusterIP)
- Outside to Service(NodePort)



