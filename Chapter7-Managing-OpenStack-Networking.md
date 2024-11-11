```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

# 描述⽹络协议类型

## ML2简介

**Modular Layer 2 (ML2)** 是 OpenStack 网络（Neutron）的一个插件框架，它允许 OpenStack Networking <mark>同时使用多种复杂数据中心中的第二层网络技术</mark>。ML2 框架区分两种类型的驱动程序：**类型驱动程序** 和 **机制驱动程序**。

- **类型驱动程序**：定义如何实现一个 OpenStack 网络。例如，VXLAN 是一个类型驱动程序，负责维护类型特定的网络状态，并验证提供者网络的信息。

- **机制驱动程序**：定义访问特定类型 OpenStack 网络的机制。例如，Open vSwitch 机制驱动程序负责确保由类型驱动程序建立的信息正确应用。

ML2 允许多个机制和类型驱动程序同时使用，从而可以访问同一虚拟网络的不同端口。这使得 ML2 框架能够灵活地支持新的第二层网络技术，减少了初始和持续维护的工作量

## 介绍⽹络类型

OpenStack ⽹络服务是在虚拟环境中提供⽹络即服务 (NaaS) 的 SDN ⽹络项⽬。它实施传统⽹络功能，如⼦⽹、桥接和 VLAN，以及开放虚拟⽹络 (OVN) 等更近期的技术。开放虚拟⽹络是红帽 OpenStack 平台的默认 SDN。

### 扁平⽹络

<mark>要将实例直接连接到外部⽹络，请使⽤扁平提供商⽹络。</mark>

### VLAN

虚拟局域⽹（VLAN）功能通过包含基于数据包的可选 VLAN 标识符（标签）来增加进⼀步的隔离和以太⽹段分区。这些标识符在称为“DOT1Q”的 IEEE 802.1Q 标准中定义。以太⽹帧标头中放⼊了
⼀个 32 位字段，其包含标签协议标识符 (TPID)。此 32 位字段的值是 0x8100。它包括 VLAN 标识符字段，指⽰该帧所属的虚拟段。<mark>排除两个保留值后，VLAN 标识符剩余 12 位⼤⼩，允许单个 LAN ⽹段上最多存在 4094 个唯⼀ VLAN</mark>。

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter7/VLAN_tagging_br-eth1.png)

### GENEVE

VLAN也就是4000多个，远远不够，而VXLAN的24为标头解决了这个问题，但是与其他的如STT和NVGRE等方案不怎么兼容，而GENEVE通过⽀持 VXLAN、NVGRE 和 STT 的所有功能来解决这些限
制。GENEVE 只封装数据格式，这意味着它不包含控制平⾯的任何规范，每个 GENEVE 隧道都具有唯⼀的 VNI，GENEVE 端⼝接⼝可在 overcloud 节点上找到

```bash
[root@controller0 ~]# ovs-vsctl show
da52b817-899b-459f-b486-b529fcbd9275
    Bridge br-prov2
        fail_mode: standalone
        Port eth4
            Interface eth4
        Port br-prov2
            Interface br-prov2
                type: internal
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
        Port ovn-8e0b99-0
            Interface ovn-8e0b99-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="172.24.2.2"}
        Port o-hm0
            Interface o-hm0
                type: internal
        Port ovn-3f678a-0
            Interface ovn-3f678a-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="172.24.2.6"}
        Port ovn-b8bd46-0
            Interface ovn-b8bd46-0
                type: geneve
                options: {csum="true", key=flow, remote_ip="172.24.2.12"}
```

### 查询网络VLAN范围定义

这里可以看到外部网络名称以及存储ID范围等信息，我们可以用101-104的VLAN ID来创建flat

```bash
[root@controller0 ml2]# pwd
/var/lib/config-data/puppet-generated/neutron/etc/neutron/plugins/ml2
[root@controller0 ml2]# grep ^network ml2_conf.ini
network_vlan_ranges=datacentre:1:1000,vlanprovider1:101:104,vlanprovider2:101:104,storage:30:30
```

# 通过dashboard创建内部和外部网络并附加到实例

**创建演示**


