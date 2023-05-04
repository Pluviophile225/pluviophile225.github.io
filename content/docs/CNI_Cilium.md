## Demo架构

![image-20230502124008416](../assets/image-20230502124008416.png)

## Cilium ebpf

用于XDP ingress、Traffic Control ingress/egress、Socket send/recv

下面这张图是原本的执行路径

![image-20230503124055236](../assets/image-20230503124055236.png)

现在我们使用XDP ingress，可以直接走到需要去的地方，像下面这张图这样。

![image-20230503124139883](../assets/image-20230503124139883.png)

对于TC而言，我们也可以使用TC的ingress来选择他要直接达到的egress。

![image-20230503150928395](../assets/image-20230503150928395.png)

具体的来分析一下kubeproxy和ebpf。看右边的ebpf，在主机的veth收到的流量，可以直接redirect到我们的物理网卡，走过tc的ingress会被match住直接到对应tc的egress中。进来的流量也是。

![image-20230503151214855](../assets/image-20230503151214855.png)

安装cilium步骤如下：

1. 首先添加cilium的仓库

   ```sh
   helm repo add cilium https://helm.cilium.io/
   helm repo update
   ```

2. 接着来生成cilium yaml文件

   ```sh
   helm  template  cilium cilium/cilium --version 1.12.3  --namespace kube-system  \
   	--set kubeProxyReplacement=strict 	\
   	--set bpf.masquerade=true 	\
   	--set k8sServiceHost=10.0.0.11 	\
   	--set k8sServicePort=6443   \
   	--set ipam.operator.clusterPoolIPv4PodCIDR=10.244.0.0/16    >   cilium-template.yaml
   ```

3. 接着来apply一下这个配置

   ```sh
   kubectl kubectl apply -f cilium-template.yaml
   ```

   然后可以来看一下apply以后的效果

   ![image-20230503154136688](../assets/image-20230503154136688.png)

   可以进到里面看一下

   ```sh
   kubectl exec -it -nkube-system cilium-2lh7b -- cilium status
   ```

   ![image-20230503154356078](../assets/image-20230503154356078.png)

   更详细可以这么看

   ```sh
   kubectl exec -it -nkube-system cilium-2lh7b -- cilium status --verbose
   ```

   也可以用log来看一下

   ![image-20230503154629186](../assets/image-20230503154629186.png)

   添加了配置后可以再来看一下这个status，就开始采用bpf了。

   ![image-20230503155142404](../assets/image-20230503155142404.png)

## Cilium host-routing

针对pods在同一个节点上是如何转发流量的。

在节点3上面有client和一个test pod，可以先来看一下client的情况。首先来看一下ip addr。

```sh
kubectl exec -it client -- ip addr
```

![image-20230503191300784](../assets/image-20230503191300784.png)

因为现在没有流量，所以肯定还没有ip neigh

```sh
kubectl exec -it client -- ip nei
```

可以再来看一下路由，2.246也是通过eth0出去。

```sh
kubectl exec -it client -- ip route
```

![image-20230503191417285](../assets/image-20230503191417285.png)

到主机上，看一下ip addr

```sh
ip addr
```

可以看到，生成了一个cilium_host 和 cilium_net

![image-20230503191523002](../assets/image-20230503191523002.png)

接下来来抓一下包，

```sh
kubectl exec -it client -- tmpdump -eni eth0
```

出去的这个包，从client的pod veth的这端发出去以后，就能在主机那端收到了。

![image-20230503201209714](../assets/image-20230503201209714.png)

```sh
tcpdump -eni lxce681d2a8bfbd
```

icmp的request就过来了，这会通过网络协议栈上的veth peer，发到另一个接口去。

![image-20230503201333125](../assets/image-20230503201333125.png)

```sh
tcpdump -eni lxc3d5e17e9704d
```

可以看到，这里没有icmp request。所以其实，他把这个包，直接redirect到了eth0那边了。

![image-20230503201703123](../assets/image-20230503201703123.png)

可以看一下eth0的内容，

```sh
kubectl exec -it testappds-5pdk7 -- tcpdump -eni eth0
```

可以看到，直接从tc 的ingress，跑到了这边的eth0来了，返回去也是同样的逻辑。

![image-20230503202313468](../assets/image-20230503202313468.png)

整个的逻辑就像这个样子。

![image-20230503202900601](../assets/image-20230503202900601.png)

整体流程如下，首先是整个的架构，

![image-20230503231333373](../assets/image-20230503231333373.png)

然后发起arp 请求，广播发起，用proxy arp来发，获取mac地址

![image-20230503231407324](../assets/image-20230503231407324.png)

然后用icmp，但是icmp就直接redirect到pod2上面了

![image-20230503231510561](../assets/image-20230503231510561.png)

![image-20230503231525676](../assets/image-20230503231525676.png)

注意，看表项的话，直接看cilium的endpoint就可以了

```sh
kubectl exec -it -n kube-system cilium-chstw -- cilium endpoint list
kubectl exec -it -n kube-system cilium-chstw -- cilium bpf endpoint list
```

这些内容都存在bpf的maps里面。

![image-20230504004742106](../assets/image-20230504004742106.png)

## Cilium vxlan

overlay networking

cilium没有办法实现类似calico里面的crosssubnet和flannel里面的directrouting。就是没法处理同一网段的不走这个vxlan，直接使用enp0s3接口

使用的是8472的端口

pod与pod之间，在不同的节点上流量是如何转发的

下面来使用client来访问worker3

![image-20230504005104266](../assets/image-20230504005104266.png)

先抓worker3上pod对应的veth peer

```sh
tcpdump -eni lxce681d2a8bfbd
```

![image-20230504005420791](../assets/image-20230504005420791.png)

做了封装之后就扔出去，就丢到了vxlan tunnel里去。mac地址都没有改变，

![image-20230504005535841](../assets/image-20230504005535841.png)

接着到了worker1上面

worker3发过来，就到了worker1上，解封装，然后直接扔到了对应veth端的eth0上去了。

![image-20230504005848283](../assets/image-20230504005848283.png)

到了eth0后，就发现，他已经把这个mac地址给修改了。

![image-20230504010104833](../assets/image-20230504010104833.png)

整体的过程如下

首先是这个过程整个的结构

![image-20230504010158283](../assets/image-20230504010158283.png)

首先要做一个arp request，来获取veth peer对端的mac地址。

![image-20230504010223478](../assets/image-20230504010223478.png)

接着从pod的veth peer端发出，发到了网络协议栈的veth peer的对端，走的是ebpf_redirect_neigh，然后直接从vxlan这个口里出去，注意，这里的id是vxlan的identity id，在bpf endpoint里面可以看到的那个id，然后交给enp0s3出去，对端的enp0s3收到了这个进行解封装，然后被ebpf劫持，直接走到ebpf_redirect_peer里去，查到要去vxlan里面，注意，一直到这一步，mac地址都没有做改变，一直到最后的时候，使用ebpf将mac地址进行修改了。

![image-20230504010329394](../assets/image-20230504010329394.png)

下面是回包

![image-20230504010633384](../assets/image-20230504010633384.png)

整个pod发出流量的过程如下图，这个是直接触发了bpf_redirect_neigh()直接出去了。

![image-20230504010822943](../assets/image-20230504010822943.png)

在入方向的过程，是从物理接口，直接被bpf_redirect_peer()直接到了pod的veth接口上去了。

![image-20230504010911646](../assets/image-20230504010911646.png)

## Native Routing

![image-20230504011648978](../assets/image-20230504011648978.png)

配置的地方要将tunnel配置为disabled，要告诉这个ipv4-native-routing-cidr是多少。

先安装bird

```sh
apt get install bird
```

配置文件

```conf
router id 10.0.0.11;
protocol kernel {
        scan time 60;
        import none;
        export all;  
}
protocol device {
        scan time 60;
}
protocol direct {
        disabled;
}
protocol static {
        route 10.244.0.0/24 via "cilium_host";
}
protocol bgp upstream {
      description "BGP uplink to Leaf";
      local 10.0.0.11 as 65011;
      neighbor 10.0.0.254 as 65001;
      import filter {accept;};
      export filter {accept;};
}
```

重启bird

```sh
systemctl restart bird
```

对每一个worker都配置好自己的内容

接着来看一下不同节点上不同pod之间的流量是如何转发的

首先，整个的架构是这个样子的

![image-20230504013651140](../assets/image-20230504013651140.png)

从client pod的eth0发出请求，发到了对端的veth peer上，然后直接将内容交给enp0s3，修改mac地址发到另一边的enp0s3上，然后收到以后会直接发到testpod的eth0上面去。

![image-20230504013703277](../assets/image-20230504013703277.png)

回包也是如此。

![image-20230504013959288](../assets/image-20230504013959288.png)

## Cilium sock-lb

在做socket调用的时候感知到目的IP和端口，会被ebpf的程序随机得dnat。

下面来看一个例子，client去访问NodeA和NodeB的Service。

![image-20230504194501857](../assets/image-20230504194501857.png)

client如果要做一些http的get，在tcp做connect的时候就能查到SVC，把目的ip和端口直接改成后端的ip和端口。

![image-20230504194630438](../assets/image-20230504194630438.png)

![image-20230504194833089](../assets/image-20230504194833089.png)

整个的过程如下

![image-20230504210209152](../assets/image-20230504210209152.png)

![image-20230504210218930](../assets/image-20230504210218930.png)

![image-20230504210228661](../assets/image-20230504210228661.png)

![image-20230504210239421](../assets/image-20230504210239421.png)

## Cilium dsr

如果哈希成了主机B上的pod，B操作完了以后，要返回给A，然后再还给client。

Client往主机A上发的时候，不仅将192.168.0.1:31000转换为1.1.1.1:80，也会把源IP10.100.1.1:6000转换成192.168.0.1:60000。这样以后B接收到了就会返给A，然后A收到了就会反向返给（也要同时改目的和源）Client。

![image-20230504210737931](../assets/image-20230504210737931.png)

这样的话，相当于主机A在这个过程中做了一个中转，就会导致一些浪费。

然后我们有一个模式叫NodePort externalTrafficPolicy = local。简单来说，就是如果访问的不在本地，那么就直接丢弃。

![image-20230504211242758](../assets/image-20230504211242758.png)

接下来看**DSR**是怎么做的

收到这样的请求之后，我们会去查service，就会做dnat，如果endpoint在远端的话，同样会做dnat，会把目的ip和端口转换成1.1.1.1:80，源头ip和port不变，返回的目的ip和port是知道的，但是不知道源ip和port。在A发往B的时候，会Append svc addr 到ip的hdr。就会把192.168.0.1:31000放在头部里面，当成一个option。

![image-20230504211336122](../assets/image-20230504211336122.png)

然后Node B出来的时候，就会把这个option放入这个内容里，这样回复的内容client也能认识。

![image-20230504211658465](../assets/image-20230504211658465.png)

![image-20230504213559888](../assets/image-20230504213559888-1683207361472-1.png)

![image-20230504213614188](../assets/image-20230504213614188.png)

![image-20230504213624835](../assets/image-20230504213624835.png)

![image-20230504213635100](../assets/image-20230504213635100.png)