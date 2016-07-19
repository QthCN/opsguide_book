# QEMU & virsh的常用操作命令

## QEMU的快照操作

进入qemu monitor，运行savevm即可(或者可以加一个快照的名字，如savevm aaa)。之后可以通过qemu-img info XXX.img或qemu-img snapshot -l XXX.img查看快照信息。
如果要还原，可以在qemu monitor中list snapshots查看下具体的snapshot，然后通过loadvm aaa还原某个快照。

## virsh的快照操作

注意，–live参数在RHEL企业版不支持，只有RHEL的虚拟化virtualization版本支持。如果没有这个参数，那么做快照的期间虚拟机是处于暂停状态的，做完后自动恢复正常状态。

* snapshot-create：从XML创建快照
* snapshot-create-as：从参数创建快照
* snapshot-current：获取当前的（最新的）快照
* snapshot-delete：删除一个快照
* snapshot-dumpxml：dump快照的xml
* snapshot-list：列出domain的快照
* snapshot-revert：跳转到某个快照

测试（这个命令就类似qemu monitor中的savevm，snapshot会保存在qcow2的image中。最后的state表明虚拟机的状态，running表明是在线的时候做的）：

```
[root@COMPUTE01 images]# virsh snapshot-create-as KVMTEST snap01
Domain snapshot snap01 created
[root@COMPUTE01 images]# virsh snapshot-list KVMTEST
 Name                 Creation Time             State
------------------------------------------------------------
 snap01               2014-05-06 23:46:57 -0700 running
```

恢复snapshot（这个命令就类似于qemu中的loadvm，就是先把vm启动起来，然后在qemu monitor中调用loadvm snap02）：

```
[root@COMPUTE01 images]# virsh snapshot-revert KVMTEST snap02 --force
```

只生成xml文件而不进行快照操作：

```
[root@COMPUTE01 images]# virsh snapshot-create-as KVMTEST snap02 --print-xml
```

查看某一快照的xml信息（通过这个可以看到某一个快照的硬件配置等情况）：

```
[root@COMPUTE01 images]# virsh snapshot-dumpxml KVMTEST snap02
```

## qemu-img的常用操作

创建特定格式的image：

```
[root@COMPUTE01 ~]# qemu-img create -f qcow2 rhel6u5.img 5G
```

检测image的完整性（只针对qcow2和vdi格式）：

```
[root@COMPUTE01 ~]# qemu-img check -f qcow2 rhel6u5.img
No errors were found on the image.
Image end offset: 262144
```

backing file：

在创建image的时候可以指定-o参数，然后加这个backing file为源image（base image）。然后生成的image（称之为overlay）只会保存和这个backing_file不同的数据，而不会对backing_file做修改（copy-on-write）：

```
[root@COMPUTE01 ~]# qemu-img create -f qcow2 -o backing_file=rhel6u5.img rhel6u5_02.img 5G
Formatting 'rhel6u5_02.img', fmt=qcow2 size=5368709120 backing_file='rhel6u5.img' encryption=off cluster_size=65536
```

之后对rhel6u5_02.img的修改只会的保存在rhel6u5_02.img中，而不会写入到rhel6u5.img。除非使用commit命令，才会把相关的修改写入rhel6u5_02.img中。
另外可以通过rebase命令重新指定某个image的backing file，原理就是把overlay的那个盘中的老的base image和新的base image的差异信息写入overlay的盘，使得其就好像是从新的base image中生成的一样。

对于base image，不建议去使用，原因是为了防止不一致的情况出现。

那么backing file到底有什么用呢？（可以看下http://www.linux-kvm.com/content/how-you-can-use-qemukvm-base-images-be-more-productive-part-1）

commit修改到base盘：
将增量的盘的差异数据写入base盘，相关的信息见backing file：

```
[root@COMPUTE01 ~]# qemu-img commit -f qcow2 rhel6u5_02.img
Image committed.
```

关于commit的内容是比较复杂的，具体的可以看“http://itxx.sinaapp.com/blog/content/130”这篇文章。比如我overlay可以下推到base file，base file也可以合并到overlay，难点有几个，比如涉及到image的snapshot chain。

查看磁盘信息：

一般用于看磁盘的真正大小(注意，由于snapshot的存在，一个image实际占用的大小可能远大于分配的大小。比如我建立的image的大小是100M，但我做了很多的快照，可能实际占用的空间会占用500M，但实际可用空间还是100M)：

```
[root@COMPUTE01 ~]# qemu-img info -f qcow2 rhel6u5.img
image: rhel6u5.img
file format: qcow2
virtual size: 5.0G (5368709120 bytes)
disk size: 136K
cluster_size: 65536
```

convert格式：

```
[root@COMPUTE01 ~]# qemu-img convert -f raw -O qcow2 centos1.img centos1qcow2.img
```

resize：

注意，如果要调小，需要在vm中把文件系统先调小。如果调大，调完后需要在文件系统中调大文件系统。raw都支持，qcow2需要RHEL 6.1后支持：

```
[root@COMPUTE01 ~]# qemu-img resize rhel6u5.img 6G
Image resized.
[root@COMPUTE01 ~]# qemu-img resize rhel6u5.img - 1G
Image resized.
[root@COMPUTE01 ~]# qemu-img resize rhel6u5.img + 1G
Image resized.
```

snapshot：

* -l：查看snapshot的个数
* -c：创建一个snapshot
* -a：恢复到某个snapshot
* -d：删除一个snapshot

```
[root@COMPUTE01 ~]# qemu-img snapshot -l rhel6u5.img
[root@COMPUTE01 ~]# qemu-img snapshot -c snapshot01 rhel6u5.img #好像有点问题，不推荐用这个，建议用savevm或virsh的snapshot命令
[root@COMPUTE01 ~]# qemu-img snapshot -l rhel6u5.img
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         snapshot01                0 2014-05-06 19:44:09   00:00:00.000
[root@COMPUTE01 ~]# qemu-img snapshot -a snapshot01 rhel6u5.img
[root@COMPUTE01 ~]# qemu-img snapshot -d snapshot01 rhel6u5.img
[root@COMPUTE01 ~]# qemu-img snapshot -l rhel6u5.img
```

## vnc

如果要启用VNC，那么编译qemu-kvm的时候就要注意相关的开关。然后启动的时候用下面的命令：

```
[root@RHEL65 kvm_demo]# qemu-system-x86_64 -m 512 -smp 1 -hda /root/kvm_demo/rhel6u5.img -vnc 192.168.19.105:1
```

这里的端口号是:后面的数字+5900，也就是5901

## CPU

一个vCPU就是一个QEMU的线程，可以通过ps -efL | grep qemu查看。
可以在启动qemu的时候加上对应的参数来设置CPU个数，如：
```
[root@RHEL65 libexec]# qemu-system-x86_64 -smp 2,sockets=2,cores=2,threads=2
```

之后在vm里通过cat /proc/cpuinfo或在QEMU monitor中运行info cpus可以查看vm的CPU信息。

推荐对多个使用单个CPU的VM进行过载，但不推荐vCPU个数大于CPU个数。在负载较重的生产中尽量还是不要过载了。

CPU模型：就是CPU的类型，可以通过qemu-system-x86_64 -cpu ?查看。

配合taskset工具，可以设置CPU的亲和性。

如果要支持嵌套虚拟化，需要加上-cpu host或-cpu qemu64,+vmx(qemu64可以换成各种model)

## 内存

通过-m指定内存的大小，单位是MB
注意，除非特别指定，否则KVM不会在启动的时候立马分配所有内存。

通过-mem-path可以挂载hugepage，通过-mem-prealloc可以让kvm在启动的时候就分配指定数量的内存

内存过载：

一般有三个技术可以实现：

* 通过交换分区
* 通过virio_balloon驱动
* 通过内存页共享

内存过载的话一般第一个比较成熟，但效率也最低。由于KVM是一个进程，因此只有在这个进程请求更多内存的时候内核才会分配内存给它。所以宿主机只要有足够的交换空间，那么是可以过载的。但物理内存+交换空间的大小只和需要大于内置给所有客户虚拟机的内存只和。

## 存储配置

主要参数（如果直接指定file而不加参数，就是-hda）：

* -hda：第一个IDE设备，可以是个文件，也可以是真是的物理硬盘（如-hda /dev/sdb），在KVM里看到的是/dev/hda或/dev/sda
* -hdb：第二个IDE设备
* -hdc:第三个IDE设备
* -hdd：第四个IDE舍必
* -fda：第一个软盘，在KVM里是/dev/fd0
* -fdb：第二个软盘
* -cdrom：CD-ROM，在KVM里是/dev/cdrom，也可以直接将宿主机的cdrom拿来用，命令是-cdrom /dev/cdrom，不过不能和-hdc合用，因为KVM中的-hdc就是cdrom
* -mtblock:Flash存储器的模拟
* -sd：SD卡的模拟
* -pflash：并行Flash存储器的模拟

较新版本的QEMU可以用-drive详细的定义一个驱动器，几个常见的参数有：

* file=XXX
* if=XXX：XXX可以是ide/scsi/virtio等，指明驱动器类型
* snapshot=on/off：是否启用snapshot。如果设置为on，那么KVM不会的将对这个drive的修改写入对应文件中去，而是写入到临时文件中，除非在qemu monitor中强制执行commit
* cache=none/off/writeback/writethrough：none和off一样，读写都不走cache，writeback和writethrough读都走cache，写的话writeback写入cache就ok，而writethrough要写入块设备才ok，所以writethrough很慢。
* readonly=on/off：是否只读

指定启动顺序：

-boot参数。在QEMU中，a、b表示第一和第二个软驱，c表示第一个硬盘，d表示CDROM，n表示网络启动。默认就是abcdn的顺序。用法是：

* -boot order=c：每次重启都从c启动
* -boot once=c：第一次是从c启动，之后都是默认顺序
* -boot order=dc,menu=on：默认启动是d和c的顺序，并且打开交互式启动菜单选项

一个例子：

```
[root@RHEL65 kvm_demo]# qemu-system-x86_64 -m 1024 -smp 2 -drive file=/root/kvm_demo/rhel6u5.bf.img,cache=writeback,if=ide  -vnc 192.168.19.105:1 -enable-kvm
```

## qemu-img

QEMU的磁盘管理工具，主要用法有：

* qemu-img check XXX：检测image是否有问题，支持qcow2/qed/vdi格式
* qemu-img info XXX：查看image信息

例如：

```
[root@RHEL65 kvm_demo]# qemu-img info rhel6u5.img
image: rhel6u5.img
file format: raw
virtual size: 7.8G (8388608000 bytes)
disk size: 7.8G
```

创建磁盘：

```
[root@RHEL65 kvm_demo]# qemu-img create -f qcow2 rhel6u5.img 8G
```

还可以使用它做调整image大小、查看image中的快照、转换image格式、对image进行加密等等很多的操作。

可以用qemu-img -h查看支持的image格式，一般推荐是qcow2

这里来看一个东西：后端镜像文件，比如我已经有一个安装好所有东西的image A，然后我想给5个客户用，那么他们的image B/C/D/E/F都可以以image A为source，通过qemu-img建立的image B/C/D/E/F的初始化大小为0，如果客户B修改了A中的东西，那么image B就会的仅仅记录修改的东西，而不影响A：

```
[root@RHEL65 kvm_demo]# ll
total 11955228
drwxr-xr-x 50 root root      20480 Mar 24 07:53 qemu-kvm.git
-rw-r--r--  1 root root 8388608000 Mar 25 04:59 rhel6u5.img
-rwxrw-rw-  1 root root 3853516800 Jan 26 02:48 rhel-server-6.5-x86_64-dvd.iso
[root@RHEL65 kvm_demo]# qemu-img create -f qcow2 -o backing_file=/root/kvm_demo/rhel6u5.img,size=10G rhel6u5.bf.img
Formatting 'rhel6u5.bf.img', fmt=qcow2 size=10737418240 backing_file='/root/kvm_demo/rhel6u5.img' encryption=off cluster_size=65536 lazy_refcounts=off
[root@RHEL65 kvm_demo]# ll
total 11955424
drwxr-xr-x 50 root root      20480 Mar 24 07:53 qemu-kvm.git
-rw-r--r--  1 root root     197120 Mar 26 05:54 rhel6u5.bf.img #############是以rhel6u5.img为源的一个image
-rw-r--r--  1 root root 8388608000 Mar 25 04:59 rhel6u5.img
-rwxrw-rw-  1 root root 3853516800 Jan 26 02:48 rhel-server-6.5-x86_64-dvd.iso
```
