# QEMU网络操作相关说明及常用命令

QEMU提供了4种网络模式（除了第四种，其它都是通过-net参数配置的。默认的参数是-net nic -net user）：

* 基于brige
* 基于NAT
* QEMU内置的用户模式网络
* 直接分配网络设备的网络

查看支持的网卡种类：

```
[root@RHEL65 kvm_demo]# qemu-system-x86_64 -net nic,model=?
qemu: Supported NIC models: ne2k_pci,i82551,i82557b,i82559er,rtl8139,e1000,pcnet,virtio
```

基本-net参数：

* -net nic：必须的参数
* vlan=n：指定VLAN，默认是0
* macaddr=XXX：指定mac地址
* model=XXX：指定虚拟的型号
* name=XXX：指定网卡名字，在QEMU monitor中有用
* addr=XXX：指定PCI相关的ADDR
* vectors=XXX：在virtio中有用

客户机中通过下面的一些命令查看网卡信息：

* lspci | grep Eth
* ethtool -i eth1
* ifconfig
* (qemu) info network

下面详细看看各种模式：

1.网桥模式：

通过-net tap来配置，相关参数有：

* vlan=n：指定vlan，默认是0
* nam=XXX：QEMU monitor中看到的名字
* ifname=XXX：在宿主机中添加的TAP虚拟设备的名字
* script=fileXXX：在KVM启动的时候，在宿主机中自动执行的脚本
* downscript=fileXXX：在KVM关闭的时候，在宿主机中自动执行的脚本

如果要使用这种模式，需要安装下面两个包：

```
yum install bridge-utils tunctl
```

查看tun的模块是否加载：

```
[root@RHEL65 kvm_demo]# lsmod | grep tun
tun                    17095  2 vhost_net
```

宿主机的一个简单配置：

```
[root@RHEL65 Desktop]# brctl addbr br0
[root@RHEL65 Desktop]# brctl addif br0 eth0
[root@RHEL65 Desktop]# brctl stp br0 on   #打开STP协议，否则可能造成环路
[root@RHEL65 Desktop]# ifconfig eth0 0
[root@RHEL65 Desktop]# dhclient br0  #动态给br0配置ip、route等，也可以手动配置。这里可以想象成eth0已经没用了，功能全部由br0这个网桥设备（或者可以假想成一个虚拟网卡）来代替。宿主机的应用要上网的话，可能要通过一个网卡，既然eth0没用了，那么就得走br0了，于是默认GATEWAY的出口interface啥的都会的走br0，br0收到后应该会的forwarding，然后从eth0出去。所以个人感觉宿主机现在是有一块br0的网卡，当要和外界通信的时候，肯定是send_packet(br0,XXX)，把数据从br0这个虚拟网卡发出去。而br0连接在了虚拟交换机上，eth0也在里边，于是从eth0转发出去了，就好像直接从br0转发出去了一样。同时，由于br0连在了虚拟交换机上，所以虚拟交换机上的KVM应该可以ping通它。那如果我想要和外网通信，并且想用eth0应该咋办呢？很简单，把eth0配成默认的gateway的出口，并给个IP就行了。
[root@RHEL65 Desktop]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.19.0    *               255.255.255.0   U     0      0        0 br0
192.168.122.0   *               255.255.255.0   U     0      0        0 virbr0
default         promote.cache-d 0.0.0.0         UG    0      0        0 br0
```

准备启动脚本，功能是在KVM启动的时候把它的虚拟网卡加入到br0中去：

```
[root@RHEL65 kvm_demo]# cat qemu-ifup
#!/bin/bash
switch=br0

if [ -n "$1" ]; then
#tunctl -u ${whoami} -t $1
ip link set $1 up
sleep 1
brctl addif ${switch} $1
exit 0
else
echo "Error: no interface specified"
exit 1
fi
```

准备结束脚本，一般不用做这个，应为QEMU会自动做：

```
[root@RHEL65 kvm_demo]# cat qemu-ifdown
#!/bin/bash
switch=br0

if [ -n "$1" ];then
tunctl -d $1
brctl delif ${switch} $1
ip link set $1 down
exit 0
else
echo "Error: no interface specified"
exit 1
fi
```

一定要赋予可执行权限：

```
[root@RHEL65 kvm_demo]# chmod a+x qemu-if*
```

先看一下此时的br0的状态：

```
[root@RHEL65 kvm_demo]# brctl show
bridge name bridge id STP enabled interfaces
br0 8000.000c296b6bd5 yes eth0
pan0 8000.000000000000 no
virbr0 8000.5254008cec79 yes virbr0-nic
```

启动一个KVM：

```
[root@RHEL65 kvm_demo]# qemu-system-x86_64 rhel6u5.bf.img -smp 2 -m 1024 -net nic -net tap,ifname=tap1,script=/root/kvm_demo/qemu-ifup,downscript=no -vnc 192.168.19.193:1 -enable-kvm
```

再次查看br0状态，发现tap1出来了，ifconfig也可以看到tap1了：

```
[root@RHEL65 ~]# brctl show
bridge name bridge id STP enabled interfaces
br0 8000.000c296b6bd5 yes eth0
tap1
pan0 8000.000000000000 no
virbr0 8000.5254008cec79 yes virbr0-nic
```

这个时候在KVM里可以看到有一个eth0的网卡，给他配个IP后就可以ping通内网了（当然也能ping通插在br0上的br0网卡了）。
当把KVM关了的时候，直接就会删除tap1这个设备，br0中的那个tap1也会被删除。

2.NAT模式

补充一个小技巧：如果外网要访问内网的IP，可以在防火墙上做些设置，把NAT服务器的某个port映射到内网IP上。这样对某个端口的访问就可以映射到对内网的某个端口的访问了。

其原理也很简单，比如我们的KMV网络都连在一个br1上，这个br1上没有任何的物理网口，但是br1这个虚拟网卡插在上面。我们把这个br1的虚拟网卡配置一个IP，作为默认的网关，于是KVM虚机的网络就能PING通这个br1了，但是他们的IP从哪里来呢？由于br1是个虚拟网卡，有IP，所以我们跑dhcp的服务dnsmasq（虽然名字里有dns，但功能貌似是dhcp）在这个br1的IP上，KVM里的网卡通过DHCP请求从br1虚拟交换机上的br1虚拟网卡上运行的dnsmasq服务就能获得IP了。另外dnsmasq也能做NAT，所以我KVM的默认路由设成br1，那么我就能通过这个br1和外界沟通了。当然在iptables上还有很多要设置的。

所以在nat这里启动脚本就比较复杂了，要判断dnsmasq是否运行，没有运行的话要运行，然后把tap设备加入到相应的br里，然后设置iptables等。另外KVM里获取IP就不用手工配置了，通过dhclient来获取。启动命令还是类似的：


```
[root@RHEL65 kvm_demo]# qemu-system-x86_64 rhel6u5.bf.img -smp 2 -m 1024 -net nic -net tap,ifname=tap1,script=/root/kvm_demo/qemu-ifup-NAT,downscript=/root/kvm_demo/qemu-ifdown-NAT -vnc 192.168.19.193:1 -enable-kvm
```

我们可以看到新建立的natbr0已经起来了，并且只有tap1在里边：

```
[root@RHEL65 ~]# brctl show
bridge name bridge id STP enabled interfaces
br0 8000.000c296b6bd5 yes eth0
natbr0 8000.a2db142023d1 yes tap1
pan0 8000.000000000000 no
virbr0 8000.5254008cec79 yes virbr0-nic
natbr0这个虚拟网卡的ip是：
natbr0    Link encap:Ethernet  HWaddr A2:DB:14:20:23:D1  
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          inet6 addr: fe80::5075:a3ff:feb6:85d3/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 b)  TX bytes:468 (468.0 b)
```

tap1所在的KVM的网卡eth0通过二层协议，通过广播可以获取自己的IP以及其它信息（通过dhclient eth0命令）。

对于这里的KVM，NAT就是GATEWAY所在的主机（也就是虚拟网卡natbr0），如果natbr0上面运行了DNS服务，那么可以把KVM的DNS配成natbr0，否则配成natbr0所在的宿主机的DNS即可，后者通过NAT是可以访问的。

3.QEMU内部的用户模式网络

默认就是-net nic -net user这个配置。该网络完全由QEMU自己实现，不依赖于bridge-utils/dnsmasq等工具，QEMU自己实现了一整套TCP/IP协议栈。

在新版本的QEMU中，使用-netdev参数替代上面的参数了。功能上有所加强，但上面的这些都能实现。
