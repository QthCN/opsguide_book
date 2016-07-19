* 前言
* 第一部分 简介
  * [传统运维的主要工作及挑战](https://github.com/QthCN/opsguide_book/blob/master/%E4%BC%A0%E7%BB%9F%E8%BF%90%E7%BB%B4%E7%9A%84%E4%B8%BB%E8%A6%81%E5%B7%A5%E4%BD%9C%E5%8F%8A%E6%8C%91%E6%88%98.md)
* 第二部分 基础
  * 虚拟化及OpenStack - ok
    * 虚拟化原理 - ok
    * 虚拟化产品 - ok
    * KVM - ok
    * libvirt - ok
    * OpenStack简介 - ok
    * OpenStack架构 - ok
      * 功能的划分方式 - ok
      * 组件的解耦方式 - ok
      * 接口的调用方式 - ok
        * RESTful API - ok
        * 消息 - ok
      * 请求的执行时序 - ok
    * OpenStack源码及社区 - ok
  * 容器 - ok
    * namespace - ok
      * namespace简介及例子 - ok
      * namespace在内核中的实现 - ok
    * cgroup - ok
      * cgroup简介及例子 - ok
    * 联合文件系统和chroot - ok
    * docker - ok
    * kubernetes - ok
    * 容器与虚拟化 - ok
  * 公有云 - ok
  * 持续集成和持续部署 - ok
    * 持续集成 - ok
    * 测试驱动开发 - ok
    * 持续部署 - ok
    * gitlab和jenkins - ok
    * 基于docker的持续部署 - ok
  * DevOps - ok
  * 构建集群管理平台OGP
    * 简介
    * 参考
      * Kubernetes - ok
        * 通过HTTP协议实现WATCH操作 - ok
        * 代码实现 - ok
    * 实现
      * 整体框架 - ok
      * Boost.asio与异步通信 - ok
      * JSON与protobuf - ok
      * Flask与WSGI - ok
      * Bootstrap - ok
      * 协程与eventlet - ok
      * 代码
* 第三部分 稳定
  * 服务发现
    * 什么是服务发现 - ok
    * 参考 - ok
      * Consul - ok
      * Registrator - ok
      * Etcd - ok
      * SkyDNS - ok
      * SmartStack - ok
      * Kubernetes - ok
      * 多播与名字指针 - ok
    * 防火墙问题 - ok
    * 实现
      * 整体框架
      * reload时零报文丢失的HAProxy
        * HAProxy简介
        * tc命令
        * iptables命令
      * dnsmasq
      * 配置文件的读取与生成
        * 词法、语法解析器(yacc,bison,ply,ANTLR...)
        * ply实现一个跨库查询语言
      * 代码
  * 网络
    * 网桥和交换机
      * bridge
      * OVS和OVN
    * Neutron
    * Docker中的网络
    * Kubernetes中的网络
    * SDN和OpenFlow
    * 内核中的网络
      * 网络报文在内核中的流动
  * 集中式配置
  * 高可用
    * 通过代理访问的高可用
      * Keepalived和vip
    * 直接访问的高可用
      * Paxos
        * Paxos
        * Paxos Lease
        * Multi Paxos
      * Raft
    * 参考
      * Etcd中的Raft
    * 实现
  * 持久存储
  * 调度
  * 日志及监控
  * 展现
  * 配置管理(自动化平台的运维)
  * 安全
  * 编排
  * 持续集成及持续部署
* 第四部分 智能
  * 故障自动修复
  * 预测
* 附录
  * OGP规范
  * 书籍推荐
  * QEMU & virsh的常用操作命令
  * QEMU网络操作相关说明及常用命令
  * 基于OpenVSwitch的Kubernetes多节点环境搭建步骤
