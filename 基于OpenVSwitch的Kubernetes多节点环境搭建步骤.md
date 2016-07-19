# 基于OpenVSwitch的Kubernetes多节点环境搭建步骤

一共三台主机，k18s01跑相关的控制器，k18s02和k18s03用来跑容器。

首先我们先建立三台虚拟机，系统使用Fedora 22。主机信息如下：

* 192.168.0.33 k18s01
* 192.168.0.34 k18s02
* 192.168.0.35 k18s03

下面是具体的安装步骤。

首先，在分别在三台主机的/etc/hosts文件中配置好相关的主机名与IP信息：

```
[root@k18s01 ~]# echo "192.168.0.33 k18s01
> 192.168.0.34 k18s02
> 192.168.0.35 k18s03" >> /etc/hosts
```

接着在三台主机上安装相关的软件包：

```
[root@k18s01 ~]# yum -y install --enablerepo=updates-testing kubernetes
[root@k18s01 ~]# yum -y install etcd iptables
```

三台主机上都把防火墙停了：

```
[root@k18s01 ~]# systemctl disable iptables-services firewalld
[root@k18s01 ~]# systemctl stop iptables-services firewalld
Failed to stop iptables-services.service: Unit iptables-services.service not loaded.
```

k18s02和k18s03上开启转发功能：

```
[root@k18s02 ~]# echo "1" > /proc/sys/net/ipv4/ip_forward
```

接着我们修改下k18s的配置，首先三台主机的/etc/kubernetes/config都改为如下配置：

```
[root@k18s01 ~]# cat /etc/kubernetes/config  | grep -v '#'
KUBE_LOGTOSTDERR="--logtostderr=true"

KUBE_LOG_LEVEL="--v=0"

KUBE_ALLOW_PRIV="--allow_privileged=false"

KUBE_MASTER="--master=http://k18s01:8080"
```

生成一个key：
```
[root@k18s01 ~]# openssl genrsa -out /tmp/serviceaccount.key 2048
Generating RSA private key, 2048 bit long modulus
........................................................................................+++
.........+++
e is 65537 (0x10001)
```

接着在k18s01上修改/etc/kubernetes/apiserver如下：

```
[root@k18s01 ~]# cat /etc/kubernetes/apiserver | grep -vE '(#|^$)'
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:4001"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
KUBE_API_ARGS="--service_account_key_file=/tmp/serviceaccount.key"
```

在k18s01上修改/etc/kubernetes/controller-manager如下：

```
[root@k18s01 ~]# cat /etc/kubernetes/controller-manager  | grep -vE '(#|^$)'
KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/tmp/serviceaccount.key"
```

在k18s01上修改etcd的监听端口，将/etc/etcd/etcd.conf中的ETCD_LISTEN_CLIENT_URLS改为”http://0.0.0.0:4001″：

```
[root@k18s01 ~]# cat /etc/etcd/etcd.conf | grep ETCD_LISTEN_CLIENT_URLS
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:4001"
```

在k18s01上启动对应的控制服务：

```
[root@k18s01 ~]# for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
>     systemctl restart $SERVICES
>     systemctl enable $SERVICES
>     systemctl status $SERVICES
> done
```

在k18s02和k18s03上修改/etc/kubernetes/kubelet文件，内容如下：

```
[root@k18s02 ~]# cat /etc/kubernetes/kubelet  | grep -vE '(#|^$)'
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_HOSTNAME="--hostname_override=k18s02"
KUBELET_API_SERVER="--api_servers=http://k18s01:8080"
KUBELET_ARGS=""
```

在k18s02及k18s03上启动相关服务：

```
[root@k18s02 ~]# for SERVICES in kube-proxy kubelet docker; do
>     systemctl restart $SERVICES
>     systemctl enable $SERVICES
>     systemctl status $SERVICES
> done
```

现在我们来设置网络环境。k18s02和k18s03的两个docker0会通过veth接到各自的ovs的bridge上，然后这两个bridge通过vxlan打通。下面具体的设置步骤：
首先先在k18s02和k18s03上安装软件包：

```
[root@k18s02 ~]# yum install -y bridge-utils openvswitch
```

需要改变下docker0的默认网段，目前在k18s02以及k18s03上docker0都是使用了172.17.42.1/16。我们改成k18s02使用172.17.20.1/24，k18s03使用172.17.30.1/24。下面是具体步骤：

```
# k18s02
[root@k18s02 ~]# systemctl stop docker.service
[root@k18s02 ~]# ip l set dev docker0 down
[root@k18s02 ~]# brctl delbr docker0
[root@k18s02 ~]# nohup docker -d --bip=172.17.20.1/24 --fixed-cidr=172.17.20.0/24 &
[root@k18s03 ~]# ip r add 172.17.30.0/24 dev docker0

# k18s03
[root@k18s03 ~]# systemctl stop docker.service
[root@k18s03 ~]# ip l set dev docker0 down
[root@k18s03 ~]# brctl delbr docker0
[root@k18s03 ~]# nohup docker -d --bip=172.17.30.1/24 --fixed-cidr=172.17.30.0/24 &
[root@k18s02 ~]# ip r add 172.17.20.0/24 dev docker0
```

在k18s02和k18s03上启动ovs服务：

```
[root@k18s02 ~]# systemctl start openvswitch.service
```

在k18s02和k18s03上各自建立一个ovs的bridge br-tun：

```
[root@k18s02 ~]# ovs-vsctl add-br br-tun
```

在k18s02和k18s03上建立veth对，连接br-tun和docker0：

```
[root@k18s02 ~]# ip link add veth-docker type veth peer name veth-ovs
[root@k18s02 ~]# ovs-vsctl add-port br-tun veth-ovs
[root@k18s02 ~]# brctl addif docker0 veth-docker
[root@k18s02 ~]# ip l set dev veth-ovs up
[root@k18s02 ~]# ip l set dev veth-docker up
```

接着我们配置下k18s02和k18s03之间的vxlan隧道，k18s02上进行如下配置：

```
[root@k18s02 ~]# ovs-vsctl add-port br-tun port-vxlan -- set Interface port-vxlan type=vxlan options:remote_ip=192.168.0.35
```

k18s03上进行如下配置：

```
[root@k18s03 ~]# ovs-vsctl add-port br-tun port-vxlan -- set Interface port-vxlan type=vxlan options:remote_ip=192.168.0.34
```

现在我们来测试下vxlan隧道是否起作用，我们在k18s03上建立一个veth对，然后将veth的一头放到docker0上，另一头我们配置一个ip。接着我们尝试在k18s02上ping这个地址，如果一切正常的话应该是可以ping通的：

```
# k18s03上的操作
[root@k18s03 ~]# ip link add debug-docker type veth peer name debug-host
[root@k18s03 ~]# brctl addif docker0 debug-docker
[root@k18s03 ~]# ip l set dev debug-docker up
[root@k18s03 ~]# ip l set dev debug-host up
[root@k18s03 ~]# ip a change 172.17.30.99 dev debug-host

# k18s02上的操作
[root@k18s02 ~]# ping 172.17.30.99 -c 4
PING 172.17.30.99 (172.17.30.99) 56(84) bytes of data.
64 bytes from 172.17.30.99: icmp_seq=1 ttl=64 time=1.94 ms
64 bytes from 172.17.30.99: icmp_seq=2 ttl=64 time=0.860 ms
64 bytes from 172.17.30.99: icmp_seq=3 ttl=64 time=0.846 ms
64 bytes from 172.17.30.99: icmp_seq=4 ttl=64 time=0.577 ms

--- 172.17.30.99 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.577/1.056/1.944/0.526 ms
```

可以看到现在到172.17.30.99的数据包会的通过k18s02上的docker0(这是由于我们上面配置了路由)，然后通过veth到br-tun，接着br-tun通过vxlan到k18s03上的br-tun，后者再通过veth到k18s03上的docker0，然后docker0桥内转发数据包给了debug-docker，后者通过veth发给了其peer口，也就是debug-host。如果大家这一步可以ping通的话我们的这个测试用的网络环境基本就可以了。

下面我们在k18s02及k18s03上下载Kubernetes需要的pause镜像。因为这个镜像是托管在google的gcr上的，后者被墙掉了，因此需要用下面的方法绕绕一下：

```
[root@k18s02 ~]# docker pull docker.io/kubernetes/pause
Trying to pull repository docker.io/kubernetes/pause ...
6c4579af347b: Download complete
511136ea3c5a: Download complete
e244e638e26e: Download complete
Status: Downloaded newer image for docker.io/kubernetes/pause:latest
[root@k18s02 ~]# docker tag docker.io/kubernetes/pause gcr.io/google_containers/pause:0.8.0
```

然后再再k18s02及k18s03上面下个nginx的镜像用于下面的测试：

```
[root@k18s02 ~]# docker pull nginx
```

下面来试下用Kubernetes建立个service测试下我们的网络环境是否符合要求。相关的配置文件如下：

```
[root@k18s01 ~]# cat nginx-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-controller
spec:
  replicas: 1
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
[root@k18s01 ~]# cat nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 8000
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
```

这里我们设置replicas为1，这个设置可以让我们观察我们的网络环境是否正常。

现在测试建立下这个rc和service，首先需要建立node。如果发现异常的话可以尝试重新启动下kubelet服务：

```
[root@k18s01 ~]# cat node01.json
{
    "apiVersion": "v1",
    "kind": "Node",
    "metadata": {
        "name": "k18s02",
        "labels":{ "name": "kub-node-label"}
    },
    "spec": {
        "externalID": "k18s02"
    }
}
[root@k18s01 ~]# cat node02.json
{
    "apiVersion": "v1",
    "kind": "Node",
    "metadata": {
        "name": "k18s03",
        "labels":{ "name": "kub-node-label"}
    },
    "spec": {
        "externalID": "k18s03"
    }
}
[root@k18s01 ~]# kubectl create -f ./node01.json
node "k18s02" created
[root@k18s01 ~]# kubectl create -f ./node02.json
node "k18s03" created
[root@k18s01 ~]# kubectl get nodes
NAME      LABELS                          STATUS
k18s01    kubernetes.io/hostname=k18s01   Ready
k18s02    name=kub-node-label             Ready
k18s03    name=kub-node-label             Ready
```

现在建立rc和service：

```
[root@k18s01 ~]# kubectl create -f ./nginx-rc.yaml
replicationcontroller "nginx-controller" created
[root@k18s01 ~]# kubectl get rc
CONTROLLER         CONTAINER(S)   IMAGE(S)   SELECTOR    REPLICAS
nginx-controller   nginx          nginx      app=nginx   1
[root@k18s01 ~]# kubectl get po
NAME                     READY     STATUS    RESTARTS   AGE
nginx-controller-zfd64   1/1       Running   0          25s
[root@k18s01 ~]# kubectl create -f ./nginx-service.yaml
service "nginx-service" created
[root@k18s01 ~]# kubectl get service
NAME            LABELS                                    SELECTOR    IP(S)           PORT(S)
kubernetes      component=apiserver,provider=kubernetes   <none>      10.254.0.1      443/TCP
nginx-service   <none>                                    app=nginx   10.254.85.157   8000/TCP
```

现在在k18s02及k18s03上curl下10.254.85.157:8000，发现都可以获取到nginx的欢迎页面：

```
# k18s02上
[root@k18s02 ~]# curl 10.254.85.157:8000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# k18s03上
[root@k18s03 ~]# curl 10.254.85.157:8000
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

同时观察容器运行情况，可以看到nginx目前是建立在了k18s03上：

```
# k18s02上
[root@k18s02 ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[root@k18s02 ~]#

# k18s03上
[root@k18s03 ~]# docker ps
CONTAINER ID        IMAGE                                  COMMAND                CREATED             STATUS              PORTS               NAMES
9ea0c512a8cb        nginx                                  "nginx -g 'daemon of   4 minutes ago       Up 4 minutes                            k8s_nginx.6420169f_nginx-controller-zfd64_default_eab9db09-40ca-11e5-b237-fa163ee101cd_f4ab7b51  
e4fb6592bfee        gcr.io/google_containers/pause:0.8.0   "/pause"               4 minutes ago       Up 4 minutes                            k8s_POD.ef28e851_nginx-controller-zfd64_default_eab9db09-40ca-11e5-b237-fa163ee101cd_19ed5eed     
[root@k18s03 ~]#
```

可以看到k18s03上负责pod网络的pause也一起起来了，另外由于上面在k18s02上可以访问到我们的nginx，说明我们的ovs网络也起作用了。如果观察iptables规则的话可以发现service的vip的地址都被代理到了kube-proxy。此时我们的一个基于openvswitch的Kubernetes多节点环境就算是搭建好了。
