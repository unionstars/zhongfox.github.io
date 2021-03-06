---
layout: post
tags : [docker, linux, kubernetes, network]
title: 深入理解容器网络

---

## 1. Docker Network

libnetwork 使用了 CNM(container network model), CNM 定义了容器虚拟化网络的模型:

* sandbox: 一个沙盒包括一个容器网络栈信息, 可以对容器的接口, 路由, DNS等进行管理, 沙盒实现: 如 linux namespace
* endpoint: 一个端点可以加入一个沙盒和一个网络, 端点实现: 如 veth pair, Open vSwitch内部端口等, 一个端点只能属于一个网络和沙盒
* network: 一组可以直联互通的端点. 实现如linux bridge, Vlan等

libnetwork 内置5中驱动:

* host: 不会为容器创建网络协议栈, 即不创建新的network namespace, 容器进程共享宿主机网络环境
* bridge: 通过nat实现对外通信, 不能解决跨主机容器通信问题
* overlay: 使用VXLAN 方式, 需要配置额外存储, 如Consul, etcd或者zk等
* remote
* null: 容器拥有自己的network namespace, 但并不为容器进行任何网络配置


### 1.1 Bridge 驱动实现

宿主机上启动2个docker容器做测试:

`sudo docker run -it ubuntu /bin/bash`
`sudo docker run -p 4000:3000 ccr.ccs.tencentyun.com/fox-test/node-koa-demo:tag4`

```
$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:96:ae:f2:53
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2464 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5171 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:158607 (158.6 KB)  TX bytes:27187058 (27.1 MB)

eth0      Link encap:Ethernet  HWaddr 52:54:00:e4:66:50
          inet addr:10.135.151.206  Bcast:10.135.191.255  Mask:255.255.192.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2334152 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1977869 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:842043414 (842.0 MB)  TX bytes:234051187 (234.0 MB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

veth4593bc2 Link encap:Ethernet  HWaddr c6:88:13:25:52:ef
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:28 errors:0 dropped:0 overruns:0 frame:0
          TX packets:17 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2625 (2.6 KB)  TX bytes:32428 (32.4 KB)

vethe4094bd Link encap:Ethernet  HWaddr ba:94:26:07:c3:1a
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2399 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5155 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:187115 (187.1 KB)  TX bytes:27123582 (27.1 MB)
```

宿主机新增虚拟网卡docker0, 并配置了静态路由, 指定目标网络是docker网段的流量走docker0

```
$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0 这条让宿主机器可以访问容器内网络, 流量由docker0网桥转发
```


宿主机创建docker0网桥, 通过veth pair 将容器和docker0连接:

```
查看网桥
ubuntu@VM-151-206-ubuntu:~$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024296aef253       no              veth4593bc2
                                                        vethe4094bd
```
>一旦一张虚拟网卡被“插”在网桥上，它就会变成该网桥的“从设备”。从设备会被“剥夺”调用网络协议栈处理数据包的资格，从而“降级”成为网桥上的一个端口。而这个端口唯一的作用，就是接收流入的数据包，然后把这些数据包的“生杀大权”（比如转发或者丢弃），全部交给对应的网桥

在容器内, docker0 的ip将作为容器路由默认网关:

```
root@dabf60697f38:/# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
```

查看宿主机的iptable:

```
~$ sudo iptables-save
# Generated by iptables-save v1.6.0 on Wed Jan 17 21:00:17 2018
*nat
:PREROUTING ACCEPT [491:38712]
:INPUT ACCEPT [477:37797]
:OUTPUT ACCEPT [1867:126055]
:POSTROUTING ACCEPT [1868:126115]
:DOCKER - [0:0]
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE 将宿主机网卡(即不是docke0网卡)出去的流量中源地址是172.17的做SNAT源地址转换
-A POSTROUTING -s 172.17.0.3/32 -d 172.17.0.3/32 -p tcp -m tcp --dport 3000 -j MASQUERADE 
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 4000 -j DNAT --to-destination 172.17.0.3:3000 外部通过宿主机端口访问容器内部, 需要进行目的地址和端口映射

*filter
:INPUT ACCEPT [366284:30610359]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [368968:40992259]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER -d 172.17.0.3/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 3000 -j ACCEPT 允许宿主机网卡(非docker)发往容器内(docker0)
-A DOCKER-ISOLATION -j RETURN
```

docker 直连:

```
root@dabf60697f38:/# curl 172.17.0.3:3000
Hello World tag4root

root@dabf60697f38:/# arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.17.0.3               ether   02:42:ac:11:00:03   C                     eth0
172.17.0.1               ether   02:42:96:ae:f2:53   C                     eth0
```

### 1.2 容器DNS和主机名

以下三个文件在容器启动后被虚拟文件覆盖(init 层):

* `/etc/hosts`
* `/etc/resolv.conf`
* `/etc/hostname`

### 1.3 DIY容器网络

目标: 容器网络和宿主机处于同一网络

1. 启动容器, 网络选择none

   `docker run -itd --name test1 --net=none ubuntu bash`

2. 手动创建网桥, 并启动

   `brctl addbr foxbr0`

   `ip link set foxbr0 up`

4. 将主机eth0的ip地址给网桥foxbr0, eth0 作为接口连到信网桥, eth0 的默认网关设置给网桥

   ```
   ip addr add {ip1/24} dev foxbr0
   ip addr del {ip1/24} dev eth0

   brctl addif foxbr0 eth0
   ip route del default
   ip route add default via {宿主机网络默认网关} dev foxbr0
   ```
5. 找到容器network namespace

   `ln -s /proc/{容器id}/ns/net /var/run/netns/test1ns`

   `ip netns exec test1ns ip link` 在容器网络namespace中执行命令

6. 创建设备对, 一端放在容器里, 一端放入网桥

   ```
   ip link add veth-a type veth peer name veth-b
   brctl addif foxbr0 veth-a        # A端加入网桥
   ip link set veth-a up

   ip link set veth-b netns test1ns # B端加入容器网络
   ```

7. 在容器网络namespace中, 配置veth-b

   ```
   ip netms exec tset1ns ip link set dev veth-b name eth0 # 网卡重命名
   ip netms exec tset1ns ip link set eth0 up
   ip netms exec tset1ns ip link add {ip2/24} dev eth0
   ip netms exec tset1ns ip route add default via {宿主机默认网关}
   ```

---

## 2. 容器跨主通信

技术方案: Overlay Network（覆盖网络）, 在已有的宿主机网络上，通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络

实现方案:

* Flannel
* Calico

---

## 3. Flannel

* UDP
* VXLAN
* host-gw

### 3.1 UDP

* 三层的 Overlay 网络
* 各宿主机增加 Tunnel 设备 flannel0, flannel0 监听端口8285
* 子网（Subnet）: 在由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”, 子网与宿主机的对应关系，保存在 Etcd
* docker0 网桥的地址范围必须是 Flannel 为宿主机分配的子网:  Docker Daemon 启动参数: `dockerd --bip=100.96.1.1/24`
* 性能问题: flanneld 增加了内核态和用户态切换次数


通信过程解析:

* node1-container-1(100.96.1.2, docker0: 100.96.1.1)
* node2-container-2(100.96.2.3, docker0: 100.96.2.1)

1. Flannel 在宿主机上创建出了一系列的路由规则:

   ```
   $ ip route
   default via 10.168.0.1 dev eth0
   100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0 # 其他宿主机容器网段(overlay网段)走flannel0
   100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1 本机容器网段走内部网桥
   10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2 # 宿主机网段走宿主机网卡
   ```

2. node1-container1 -> node2-container2, 匹配路由2, 交由flannel0处理, 从内核态进入用户态, 用户态Flannel 进程读取flannel0, 进行处理

3. node1-flannel0 解析数据包中的目的ip, 并从ectd中获得该ip子网对应的宿主机ip, 封装为UDP数据

4. node1-flannel0把 UDP 包发往 Node 2 的 8285 端口

5. node2-flannel0 从8285 接受数据, 从UDP中解出container-1 发来的原 IP 包, 并把这个 IP 包发送给它所管理的 TUN 设备，即 node2-flanneld 设备, 从用户态向内核态的流动方向

6. Node2 路由:

   ```
   # 在 Node 2 上
   $ ip route
   default via 10.168.0.1 dev eth0
   100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.2.0
   100.96.2.0/24 dev docker0  proto kernel  scope link  src 100.96.2.1 # 本机容器网段走内部网桥
   10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.3 #
   ```

   匹配第二条本机容器网段, 数据走内部网桥docker0


### 3.2 VXLAN

VXLAN: 即 Virtual Extensible LAN（虚拟可扩展局域网）

* VXLAN 本身就是 Linux 内核中的一个模块, 是 Linux 内核本身就支持的一种网络虚似化技术, VXLAN 可以完全在内核态实现上述封装和解封装的工作
* 由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信
* VTEP(VXLAN Tunnel End Point): 虚拟隧道端点, VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端, 它既有 IP 地址，也有MAC 地址
* VXLAN Header VNI: VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值
* FDB (Forwarding Database):
* 性能损失在 20%~30% 左右


通信过程解析:

* node1-container1(container:10.1.15.2, docker0:10.1.15.1/24, flannel.1:10.1.15.0/32)
* node2-container2(container:10.1.16.3, docker0: 10.1.16.1/24, flannel.1: 10.1.16.0/32)


1. Node1 宿主机路由(因node2 加入时创建):

   ```
   $ route -n
   Kernel IP routing table
   Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
   ...
   10.1.16.0       10.1.16.0       255.255.255.0   UG    0      0        0 flannel.1
   ```

   发往 10.1.16.0/24 网段的 IP 包，都需要经过 flannel.1 设备发出，并且，它最后被发往的网关地址是：10.1.16.0 (node2.flannel.1)

2. Node1上的ARP记录node2.flannel.1 mac(因node2 加入时创建, 并不依赖 L3 MISS 事件和 ARP 学习):

   ```
   # 在 Node 1 上
   $ ip neigh show dev flannel.1
   10.1.16.0 lladdr 5e:f8:4f:00:e3:37 PERMANENT
   ```

3. Node 1 上,  flanneld 进程负责维护FDB: 存储了node2.flannel.1.MAC 和 node2.IP 的映射关系

   ```
   # 在 Node 1 上，使用“目的 VTEP 设备”的 MAC 地址进行查询
   $ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
   5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
   ```

4. 原始IP包(node1-container1 -> node2-container2), 首先交由node1.docker0, 然后交由node1.flannel.1, 进入「隧道入口」

   生成内部数据帧: `[目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]`

5. node1.flannel.1 尝试将原始IP包发往node2.flannel.1: 实现是将不同宿主机的flannel.1组成一个二层网络

   生成VXLAN数据: 增加VXLAN Header:  `[VxlanHeader:VNI:1 [目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]]`

6. 使用UDP发送VXLAN数据:

   通过node1 上的FDB, 查询`node2.flannel.1.MAC` -> `node2.IP`

   通过node常规ARP`node2.IP` -> `node2.MAC`

   生成外UDP包: `[目的MAC:node2.MAC, 目的IP:node2.IP [VxlanHeader:VNI:1 [目的MAC:node2.flannel.1.MAC, 目的IP:container2.IP, ...]]]`

7. Node 2 的内核网络栈会发现这个数据帧里有 VXLAN Header，并且 VNI=1。所以 Linux 内核会对它进行拆包，拿到里面的内部数据帧，然后根据 VNI 的值，把它交给 Node2.flannel.1 设备

8. node.2.flannel.1 设备则会进一步拆包，取出“原始 IP 包”

### 3.3 host-gw

* 必须要求集群宿主机之间是二层连通的
* host-gw 其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址
* Flannel 子网和主机的信息，都是保存在 Etcd 当中的。flanneld 只需要 WACTH 这些数据的变化，然后实时更新路由表即可
* 性能损失大约在 10% 左右


通信过程解析:

* node1-container1(container:A.B.1.2/24, cni0:A.B.1.1/24, node.eth0:X.Y.Z.1/24)
* node2-container2(container:A.B.2.2/24, cni0:A.B.2.1/24, node.eth0:X.Y.Z.2/24)
* node 默认网关: X.Y.Z.0

1. 路由配置

   node1.flanneld 自动配置路由:

   ```
   default via X.Y.Z.0 dev eth0                            # 主机默认网关
   A.B.1.0/24 dev cni0 proto Kernel scope link src A.B.1.1 # 发给本机的容器网段IP包, 交给cni0
   A.B.2.0/24 via X.Y.Z.2 dev eth0                         # 发给其他主机的容器网段IP包, 交给对应主机的eth0网卡
   ```

   node2.flanneld 自动配置路由:

   ```
   default via X.Y.Z.0 dev eth0                            # 主机默认网关
   A.B.2.0/24 dev cni0 proto Kernel scope link src A.B.2.1 # 发给本机的容器网段IP包, 交给cni0
   A.B.1.0/24 via X.Y.Z.1 dev eth0                         # 发给其他主机的容器网段IP包, 交给对应主机的eth0网卡
   ```

2. node1-container1 发往 node2-container2

   1) node1-container1发送:  匹配node1 规则3, 数据从eth0, 发往node2.eth0

   2) node2 接受: 匹配node2 规则2, 数据包交给 cni0, 后续会进入container2


---

## Calico

TODO
