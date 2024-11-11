```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 说明启动流程并讨论计算计划和计算放置服务。
- 说明红帽超融合基础架构。 
- 讨论迁移流程，迁移实例，以及启⽤和禁⽤计算节点。

# 描述实例启动流程

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter8/launch-process-flow.svg)

## 调度流程

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter8/launch-scheduling.svg)

# 介绍红帽超融合基础架构

## 超融合节点概念

本章节介绍另⼀个预定义⻆⾊，名为 ComputeHCI。它⽤于部署超融合计算节点。超融合是⼀种节点配置，它将普通的虚拟机监控程序计算⻆⾊与同⼀计算节点上的本地 Ceph OSD 相结合。红帽超融合基础架构始终使⽤ Ceph 作为存储组件。超融合节点上的 Ceph 组件与 Ceph 存储⻆⾊存储节点上安装的组件相同。超融合存储的好处是通过更有效的资源管理降低成本并提⾼操作灵活性。远程分⽀和边缘⽹络是超融合节点的主要⽤例场景。

超融合基础架构允许使⽤标准化私有云构建块，从⽽更轻松地将计算和存储功能作为⼀个单元进⾏管理，以此节省成本和空间。使⽤超融合基础架构可提⾼数据中⼼与远程⽹络之间的软件可移植性，并且可以更好地综合利⽤⽹络边缘的⽹络功能虚拟化部署。

## 主机聚合

主机聚合⽤于在可⽤区域内为节点创建进⼀步的划分。创建和配置聚合需要管理员权限。主机聚合⽤于分配元数据键值对到节点组。节点可以是多个主机聚合的成员。相同键值元数据可以分配到许多聚合，并且可以根据需要使⽤任意数量的键值对来配置聚合，以定义主机选择⽅案。

比如:

1. 创建amd CPU的主机集合

2. 创建具有英伟达显卡的主机集合

### 定向调度案例

我们来做一个实际案例，定向调度到computehci节点上

#### 创建主机聚合

```bash
(overcloud) [stack@director ~]$ openstack aggregate create hci-aggregate
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| availability_zone | None                                 |
| created_at        | 2024-11-08T09:01:24.525602           |
| deleted           | False                                |
| deleted_at        | None                                 |
| hosts             | None                                 |
| id                | 1                                    |
| name              | hci-aggregate                        |
| properties        | None                                 |
| updated_at        | None                                 |
| uuid              | 7f62c2be-335f-43be-ab00-e2bcd7db2f47 |
+-------------------+--------------------------------------+
```

将主机添加进来

```bash
(overcloud) [stack@director ~]$ openstack aggregate add host hci-aggregate computehci0.overcloud.example.com
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| availability_zone | None                                 |
| created_at        | 2024-11-08T09:01:24.000000           |
| deleted           | False                                |
| deleted_at        | None                                 |
| hosts             | computehci0.overcloud.example.com    |
| id                | 1                                    |
| name              | hci-aggregate                        |
| properties        |                                      |
| updated_at        | None                                 |
| uuid              | 7f62c2be-335f-43be-ab00-e2bcd7db2f47 |
+-------------------+--------------------------------------+
```

给这个聚合打一个标签

这个标签是用于稍后的flavor对接用的

```bash
(overcloud) [stack@director ~]$ openstack aggregate set --property computehci=true hci-aggregate
(overcloud) [stack@director ~]$ openstack aggregate show hci-aggregate
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| availability_zone | None                                 |
| created_at        | 2024-11-08T09:01:24.000000           |
| deleted           | False                                |
| deleted_at        | None                                 |
| hosts             | computehci0.overcloud.example.com    |
| id                | 1                                    |
| name              | hci-aggregate                        |
| properties        | computehci='true'                    |
| updated_at        | None                                 |
| uuid              | 7f62c2be-335f-43be-ab00-e2bcd7db2f47 |
+-------------------+--------------------------------------+
```

我们创建一个flavor

```bash
(overcloud) [stack@director ~]$ openstack flavor create --ram 1024 --disk 10 --vcpus 2 --public default-hci
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| description                | None                                 |
| disk                       | 10                                   |
| extra_specs                | {}                                   |
| id                         | cdc5d01c-f022-404a-bfc3-6ebd8fee2616 |
| name                       | default-hci                          |
| os-flavor-access:is_public | True                                 |
| properties                 |                                      |
| ram                        | 1024                                 |
| rxtx_factor                | 1.0                                  |
| swap                       | 0                                    |
| vcpus                      | 2                                    |
+----------------------------+--------------------------------------+
```

我们给他配置同样的标签

```bash
(overcloud) [stack@director ~]$ openstack flavor set default-hci --property computehci=true
(overcloud) [stack@director ~]$ openstack flavor show default-hci
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| OS-FLV-DISABLED:disabled   | False                                |
| OS-FLV-EXT-DATA:ephemeral  | 0                                    |
| access_project_ids         | None                                 |
| description                | None                                 |
| disk                       | 10                                   |
| extra_specs                | {'computehci': 'true'}               |
| id                         | cdc5d01c-f022-404a-bfc3-6ebd8fee2616 |
| name                       | default-hci                          |
| os-flavor-access:is_public | True                                 |
| properties                 | computehci='true'                    |
| ram                        | 1024                                 |
| rxtx_factor                | 1.0                                  |
| swap                       | 0                                    |
| vcpus                      | 2                                    |
+----------------------------+--------------------------------------+
```

接下来，你创建镜像、网络等资源后，创建实例，实例就会到hci节点上

# 管理计算节点

## 迁移和撤离简介

**迁移：**

1. 在冷迁移中，如果实例处于运⾏状态，管理员将关闭该实例。将镜像的副本从源计算节点发送到⽬标计算节点。该实例将重新分配给数据库中的⽬标主机并重新启动。

2. 实时迁移将实例从⼀个主机移动到另⼀个主机⽽不关闭电源。实例不知道它正被移动。

**撤离**

撤离通常在计算节点出现故障或进⼊关闭模式时发⽣。可以撤离正在运⾏的计算节点，但它不被视为撤离过程的正常使⽤情况。实例在另⼀个计算节点上重建。实例保留其名称、UUID、⽹络地址和任何其他分配的资源。如果实例磁盘不在共享存储或块存储卷上，则撤离过程具有破坏性。如果没有指定⽬标计算节点，OpenStack 会选择哪些计算节点将会接收撤离的实例。

## 实例迁移

使⽤ openstack server migrate 命令结合 --live 选项，可以在运⾏时迁移实例，如果实例已经停止，就不用--live，管理员都可使⽤ --target 强制 OpenStack 使⽤特定的⽬标计算节点

--shared-migration 和 --block-migration 选项⽤于管理磁盘转移。在默认情况下，--shared-migration 选项由 OpenStack 实施，红帽建议后端启用ceph来完成共享存储迁移。

### 为迁移做准备

1. 创建了一个名为small的镜像
2. 创建了一个名为key1的密钥对
3. 创建了一个名为network1的内部网络
4. 创建了一个名为20d2c1m的flavor
5. 创建了一个名为server1的实例

### 测试实时迁移

先看看实例在哪个机器上，从下面来看，是在compute0上

```bash
(overcloud) [stack@director ~]$ openstack server show server1 | grep host
| OS-EXT-SRV-ATTR:host                | compute0.overcloud.example.com                                                     |
| OS-EXT-SRV-ATTR:hostname            | server1                                                                            |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute0.overcloud.example.com                                                     |
| hostId                              | 62bcd5ada19c938217ef38b8f43cf2403e609633192f19cbb43645b6                           |
| host_status                         | UP                                                                                 |
```

来把他迁移到compute1上面去

这里提示我们，以后要用--live-migration，我们也记一下

```bash
(overcloud) [stack@director ~]$ openstack server migrate --shared-migration --live compute1.overcloud.example.com server1
The --live option has been deprecated. Please use the --live-migration option instead.
```

稍等一会儿，看看迁过去了没有

没问题，迁移成功

```bash
(overcloud) [stack@director ~]$ openstack server show server1 | grep host
| OS-EXT-SRV-ATTR:host                | compute1.overcloud.example.com                                                     |
| OS-EXT-SRV-ATTR:hostname            | server1                                                                            |
```

### 测试块迁移

这是非实时迁移访问，适用于没有共享存储

```bash
(overcloud) [stack@director ~]$ openstack server migrate --host compute0.overcloud.example.com server1
```

这种冷迁移需要手工确认迁移

```bash
(overcloud) [stack@director ~]$ openstack server show server1 | grep status
| host_status                         | UP                                                                                 |
| status                              | VERIFY_RESIZE                                                                      |
```

需要注意的是，我发现新版本的openstack中要用`openstack server migrate confirm`

```bash
(overcloud) [stack@director ~]$ openstack server resize confirm server1
```

没问题，已经是在目标机器上active了

```bash
(overcloud) [stack@director ~]$ openstack server show server1 | grep -E  'host|status'
| OS-EXT-SRV-ATTR:host                | compute0.overcloud.example.com                                                     |
| OS-EXT-SRV-ATTR:hostname            | server1                                                                            |
| OS-EXT-SRV-ATTR:hypervisor_hostname | compute0.overcloud.example.com                                                     |
| hostId                              | 62bcd5ada19c938217ef38b8f43cf2403e609633192f19cbb43645b6                           |
| host_status                         | UP                                                                                 |
| status                              | ACTIVE                                                                             |
```

看看主机列表

```bash
(overcloud) [stack@director ~]$ openstack server list --all-projects
+--------------------------------------+----------------------------------------------+--------+--------------------------+----------------------------------------+--------+
| ID                                   | Name                                         | Status | Networks                 | Image                                  | Flavor |
+--------------------------------------+----------------------------------------------+--------+--------------------------+----------------------------------------+--------+
| abbb84b3-66a2-4893-a641-2d11d8657fb6 | server1                                      | ACTIVE | network1=10.0.0.165      | small                                  |        |
| bd61939e-3346-4e76-8b6e-01f618cab414 | amphora-cc55ec37-735f-41fa-a5d1-e8b3b65a2981 | ACTIVE | lb-mgmt-net=172.23.0.125 | octavia-amphora-16.1-20200812.3.x86_64 |        |
+--------------------------------------+----------------------------------------------+--------+--------------------------+----------------------------------------+--------+
```

## 撤离案例

必须先禁⽤该节点，不然不能撤离的

```bash
(overcloud) [stack@director ~]$ openstack compute service list -c Host -c Binary -c Status -c State
+----------------+-----------------------------------+----------+-------+
| Binary         | Host                              | Status   | State |
+----------------+-----------------------------------+----------+-------+
| nova-conductor | controller0.overcloud.example.com | enabled  | up    |
| nova-scheduler | controller0.overcloud.example.com | enabled  | up    |
| nova-compute   | compute1.overcloud.example.com    | enabled  | up    |
| nova-compute   | computehci0.overcloud.example.com | enabled  | up    |
| nova-compute   | compute0.overcloud.example.com    | enabled | up    |
+----------------+-----------------------------------+----------+-------+
```

来把compute0禁用掉

```bash
(overcloud) [stack@director ~]$ openstack compute service set --disable compute0.overcloud.example.com nova-compute
```

再看的时候，就是disabled了

```bash
(overcloud) [stack@director ~]$ openstack compute service list -c Host -c Binary -c Status -c State
+----------------+-----------------------------------+----------+-------+
| Binary         | Host                              | Status   | State |
+----------------+-----------------------------------+----------+-------+
| nova-conductor | controller0.overcloud.example.com | enabled  | up    |
| nova-scheduler | controller0.overcloud.example.com | enabled  | up    |
| nova-compute   | compute1.overcloud.example.com    | enabled  | up    |
| nova-compute   | computehci0.overcloud.example.com | enabled  | up    |
| nova-compute   | compute0.overcloud.example.com    | disabled | up    |
+----------------+-----------------------------------+----------+-------+
```

强制发生撤离

这个撤离在dashboard上也可以看到进度

```bash
(overcloud) [stack@director ~]$ nova host-evacuate-live compute0.overcloud.example.com
+--------------------------------------+-------------------------+---------------+
| Server UUID                          | Live Migration Accepted | Error Message |
+--------------------------------------+-------------------------+---------------+
| abbb84b3-66a2-4893-a641-2d11d8657fb6 | True                    |               |
+--------------------------------------+-------------------------+---------------+
```
