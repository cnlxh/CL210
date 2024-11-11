```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述 OpenStack 服务的后端存储选项。
- 讨论 Swift 和 Ceph 对象存储，⽐较架构注意事项。
- 描述⽤来配置新共享⽂件系统组件的多种⽅法。
- 解读临时存储配置选择的⾏为。

# 实施块存储

红帽 OpenStack 平台提供三种类型的持久存储，即<mark>块存储、对象存储和⽂件存储</mark>

## 块存储选择

块存储分为临时存储和持久存储，<mark>临时存储的生命周期和实例绑定，而持久存储的生命周期是分开的</mark>，OpenStack 中的块存储由块存储服务 (Cinder) 实施，允许⽤⼾以卷的形式访问块存储设备。卷是持久的，也就是说，它们可以附加到⼀个或多个实例并可分离和重新附加，同时其存储的数据会保持完好。

块存储服务使⽤块存储驱动程序来⽀持不同的后端，红帽 OpenStack 平台⽀持多种商⽤存储后端技术，可以通过多种不同的组合来实施。

### LVM 和 iSCSI

基于 Linux 的逻辑卷管理器 (LVM) 将本地物理磁盘作为逻辑卷公开给操作系统。LVM 后端将块存储实施为 LVM 逻辑分区。实施 LVM 的每个计算主机必须具有专⽤于块存储服务的卷组。虽然 LVM 是块存储服务的默认后端，但红帽不⽀持在⽣产环境中使⽤ LVM

### 红帽 Ceph 存储

红帽 Ceph 存储提供 PB 级别的存储，可以配置在⾼弹性和冗余架构中。Ceph 以对象服务 (Swift) 形式提供对象存储 API ⽀持，并可⽤作块存储服务 (Cinder) 和镜像服务 (Glance) 的后端。Ceph 还⽀持通过 NFS 和分布式⽂件系统接⼝ CephFS 访问卷

### 通过 NFS 提供存储

通过利⽤⽹络⽂件系统 (NFS) ⽂件系统协议，⽂件客⼾端可以使⽤远程过程调⽤ (RPC) 并将⽂件⽤作块设备卷来挂载远程⽂件系统和访问内容。通过使⽤共享 NIC 和传统⽹络，NFS 具有成本优势。与 Ceph 或 Gluster 等其他基于云设计的块服务后端相⽐，虽然更为简单，但 NFS 可能⽆法表现出最佳吞吐量或最低延迟。

## 红帽 Ceph 存储架构

Ceph 基于⼀种模块化分布式架构，包含下列元素：

- 对象存储后端，称为 RADOS（可靠的⾃主分布式对象存储）

- 与 RADOS 交互的多种访问⽅式

RADOS 是⼀种⾃我修复、⾃我管理的软件型对象存储。

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter6/AccessMethods-RADOS-300dpi.svg)

**Ceph术语**

1. Ceph集群

2. 节点

3. 池

4. 放置组

### Ceph 存储后端组件

- 监控器 (MON)，维护集群状态的映射，⽤于帮助其他守护进程互相协调。
- 对象存储设备 (OSD)，存储数据并处理数据复制、恢复和重新平衡。
- 管理器 (MGR)，通过基于 Web UI 控制⾯板和 REST API 来跟踪运⾏时指标并公开集群信息。
- 元数据服务器 (MDS)，存储供 CephFS 使⽤的元数据（⽽⾮对象存储或块存储），让客⼾端能够⾼效执⾏ POSIX 命令。

**以下是我们课程中的Ceph组件位置：**

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter6/storage-block-ceph-components.svg)

### 通过 Cephx 进⾏⾝份验证

<mark>Ceph 采⽤了基于共享 secret key 的 cephx ⾝份验证协议⽤于授权集群中客⼾端、应⽤和守护进程之间的通信</mark>

1. Ceph 守护进程使⽤的帐⼾具有与相关守护进程匹配的名称：osd.1 或 mgr.serverc。

2. 使⽤ librados 的客⼾端应⽤帐⼾具有使⽤ client. 前缀的名称，例如client.openstack

3. 部署 Ceph 对象⽹关时，会创建名为 client.rgw.hostname

4. <mark>管理员账号为client.admin，如果没有指定用户名，默认会使用此用户</mark>

### 密钥环⽂件

<mark>为进⾏⾝份验证，客⼾端必须配置有 cephx ⽤⼾名，以及含有该⽤⼾的机密密钥的密钥环⽂件</mark>。cephx ⽤⼾需要这个密钥环⽂件来访问红帽 Ceph 存储集群。你需要将此⽂件复制到需要它的客⼾端系统或应⽤服务器上，在这些客⼾端系统中，librados 使⽤ /etc/ceph/ceph.conf 配置⽂件中的 keyring 参数来查找密钥环⽂件。其默认值为 `/etc/ceph/$cluster.$name.keyring`。例如，对于 client.openstack 帐⼾，其密钥环⽂件为 /etc/ceph/ceph.client.openstack.keyring。

### 身份验证案例

以下 ceph 命令以 client.operator3 进⾏⾝份验证并列出可⽤的池

需要注意的时，你安装了ceph-common就不用进入到容器里执行了

```bash
yum install ceph-common -y
scp xxxx:/etc/ceph/ceph.conf /etc/ceph/
scp xxxx:/etc/ceph/ceph.client.operator3.keyring /etc/ceph/
ceph --id operator3 osd lspools
```

### 查询集群状态

```bash
[root@controller0 /]# ceph -s
  cluster:
    id:     fe8e3db0-d6c3-11e8-a76d-52540001fac8
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum controller0
    mgr: controller0(active)
    mds: cephfs-1/1/1 up  {0=controller0=up:active}
    osd: 6 osds: 6 up, 6 in

  data:
    pools:   7 pools, 416 pgs
    objects: 748 objects, 5655 MB
    usage:   6309 MB used, 107 GB / 113 GB avail
    pgs:     416 active+clean
```

也可以用systemd的查看方法去看看服务是否启动

```bash
[root@controller0 ~]# systemctl list-units ceph\*
UNIT                         LOAD   ACTIVE SUB     DESCRIPTION
ceph-mds@controller0.service loaded active running Ceph MDS
ceph-mgr@controller0.service loaded active running Ceph Manager
ceph-mon@controller0.service loaded active running Ceph Monitor


[root@ceph0 ~]# systemctl list-units ceph\*
UNIT                 LOAD   ACTIVE SUB     DESCRIPTION
ceph-osd@vdb.service loaded active running Ceph OSD
ceph-osd@vdc.service loaded active running Ceph OSD
ceph-osd@vdd.service loaded active running Ceph OSD
```

### Cephx 授权

功能也可以暂时理解为权限

在 cephx 内，每⼀守护进程类型都有⼏种可⽤的功能：

- r 授予读取访问权限。每⼀⽤⼾帐⼾⾄少应在 monitor 上具有读取访问权限，以便能够检索CRUSH map。
- w 授予写⼊访问权限。客⼾端需要写⼊访问权限以在 OSD 上存储和修改对象。对于管理器(MGR)，w 授予启⽤或禁⽤模块的权限。
- x 授予执⾏扩展对象类的权限。这使得客⼾端能够在对象上执⾏额外的操作，如使⽤ rados lock get 设置锁定或使⽤ rbd list 列出 RBD 镜像。
- `*` 授予完整访问权限。
- class-read 和 class-write 是 x 的⼦集。您通常在⽤于 RBD 的池上使⽤它们。

### 查询用户信息

所有用户列表

```bash
[root@controller0 /]# ceph auth list | more
installed auth entries:

mds.controller0
        key: AQCzM89blEMOJBAAiFp/PFV9IRB0sv5tea7srg==
        caps: [mds] allow
        caps: [mon] allow profile mds
        caps: [osd] allow rwx
osd.0
        key: AQBCM89b1fgXGxAA9xG4PyIt+0ZcVvTwFBUOBQ==
        caps: [mgr] allow profile osd
        caps: [mon] allow profile osd
        caps: [osd] allow *
```

特定用户信息

```bash
[root@controller0 /]# ceph auth get client.admin
exported keyring for client.admin
[client.admin]
        key = AQBhMs9bANcVCxAA3ZHYooO3wi1p0hW90kApyw==
        auid = 0
        caps mds = "allow"
        caps mgr = "allow *"
        caps mon = "allow *"
        caps osd = "allow *"
```

只查询keyring

```bash
[root@controller0 /]# ceph auth print-key client.admin
AQBhMs9bANcVCxAA3ZHYooO3wi1p0hW90kApyw==
```

### 创建用户

```bash
[root@controller0 /]# ceph auth get-or-create client.lixiaohui mon 'allow r' osd 'allow rw' > /etc/ceph/ceph.client.lixiaohui.keyring
[root@controller0 /]# ceph auth get client.lixiaohui
exported keyring for client.lixiaohui
[client.lixiaohui]
        key = AQAAdC1n8VBwIRAAOVXXdVBHenAUbmAGtxsPKA==
        caps mon = "allow r"
        caps osd = "allow rw"
```

### Glance与Ceph对接

这里只提到如何修改配置文件，其他信息可以参考ceph官网：`https://docs.ceph.com/en/reef/rbd/rbd-openstack`

```bash
[DEFAULT]
enabled_backends = rbd:rbd
show_image_direct_url = True
[glance_store]
default_backend = rbd
[rbd]
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
[paste_deploy]
flavor = keystone
```

### cinder与Ceph对接

这里只提到如何修改配置文件，其他信息可以参考ceph官网：`https://docs.ceph.com/en/reef/rbd/rbd-openstack`

```bash
[DEFAULT]
enabled_backends = ceph
glance_api_version = 2

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
volume_backend_name = ceph
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = e4355171-b66e-4036-989f-f6af496797a6 # 这里是libvirt创uuidgen的UUID
```

### 创建cinder卷

同时为块存储服务激活了多个后端，则需要创建卷类型并将其绑定到相应的存储后端

比说如，我在配置文件中指定的后端名称为tripleo_ceph

```bash
[root@controller0 ~]# cd /var/lib/config-data/puppet-generated/cinder/etc/cinder
[root@controller0 ~]# grep -v -e ^$ -e ^# cinder.conf

[DEFAULT]
enabled_backends=tripleo_ceph
[tripleo_ceph]
backend_host=hostgroup
volume_backend_name=tripleo_ceph
volume_driver=cinder.volume.drivers.rbd.RBDDriver
rbd_ceph_conf=/etc/ceph/ceph.conf
rbd_user=openstack
rbd_pool=volumes
```

创建一个名为cephtype的类型供用户选择

```bash
(undercloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ openstack volume type create --property volume_backend_name=tripleo_ceph cephtype
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| description | None                                 |
| id          | 4bbce644-64b8-4fb5-9e4b-8b86aa6b4d91 |
| is_public   | True                                 |
| name        | cephtype                             |
| properties  | volume_backend_name='tripleo_ceph'   |
+-------------+--------------------------------------+
```

用我们的cephtype类型创建一个1G的卷

```bash
(overcloud) [stack@director ~]$ openstack volume create --size 1 --type cephtype volume1
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-11-08T02:33:42.000000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 59007d3c-64b1-48d2-8b63-56b837e4e7c4 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volume1                              |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 1                                    |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | cephtype                             |
| updated_at          | None                                 |
| user_id             | 9ce00d8980274974b2285ed73ef80241     |
+---------------------+--------------------------------------+
(overcloud) [stack@director ~]$ openstack volume list
+--------------------------------------+---------+-----------+------+-------------+
| ID                                   | Name    | Status    | Size | Attached to |
+--------------------------------------+---------+-----------+------+-------------+
| 59007d3c-64b1-48d2-8b63-56b837e4e7c4 | volume1 | available |    1 |             |
+--------------------------------------+---------+-----------+------+-------------+
```

看看这个数据有没有跑到ceph里

经过查询，volumes数据池已经有了我们59开头的ID

```bash
[root@controller0 /]# rados -p volumes ls
rbd_directory
rbd_info
rbd_object_map.86bd59895364
rbd_header.86bd59895364
rbd_id.volume-59007d3c-64b1-48d2-8b63-56b837e4e7c4
```

这个大小很小看不出什么，我们试试用一个镜像作为内容提供方，创建一个包含此镜像内容的卷

```bash
(overcloud) [stack@director ~]$ wget http://materials/osp-small.qcow2
--2024-11-08 02:37:34--  http://materials/osp-small.qcow2
Resolving materials (materials)... 172.25.254.254
Connecting to materials (materials)|172.25.254.254|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1632174080 (1.5G)
Saving to: ‘osp-small.qcow2’

100%[================================================================>] 1,632,174,080  302MB/s   in 4.8s

2024-11-08 02:37:39 (328 MB/s) - ‘osp-small.qcow2’ saved [1632174080/1632174080]



(overcloud) [stack@director ~]$ openstack image create --file osp-small.qcow2 image1


(overcloud) [stack@director ~]$ openstack volume create --type cephtype --image image1 --size 20 volume2
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| attachments         | []                                   |
| availability_zone   | nova                                 |
| bootable            | false                                |
| consistencygroup_id | None                                 |
| created_at          | 2024-11-08T02:39:50.000000           |
| description         | None                                 |
| encrypted           | False                                |
| id                  | 88e4c800-aa16-4f89-988c-ed8d8dc72708 |
| migration_status    | None                                 |
| multiattach         | False                                |
| name                | volume2                              |
| properties          |                                      |
| replication_status  | None                                 |
| size                | 20                                   |
| snapshot_id         | None                                 |
| source_volid        | None                                 |
| status              | creating                             |
| type                | cephtype                             |
| updated_at          | None                                 |
| user_id             | 9ce00d8980274974b2285ed73ef80241     |
+---------------------+--------------------------------------+
```

这个创建可能没那么快，我们稍后去ceph看看有没有88e开头的数据

```bash
[root@controller0 /]# rados -p volumes ls
rbd_directory
rbd_children
rbd_info
rbd_object_map.86bd59895364
rbd_header.86bd59895364
rbd_object_map.870e6d5693f8
rbd_id.volume-59007d3c-64b1-48d2-8b63-56b837e4e7c4
rbd_header.870e6d5693f8
rbd_id.volume-88e4c800-aa16-4f89-988c-ed8d8dc72708
```

创建基于Cinder的共享存储

同时将持久块存储卷附加到多个实例。通过卷多重附加，多个实例可以连接到⼀个卷，从⽽能更快速地故障转移到备⽤实例。

```bash
(overcloud) [stack@director ~]$ cinder type-create volume-multi
+--------------------------------------+--------------+-------------+-----------+
| ID                                   | Name         | Description | Is_Public |
+--------------------------------------+--------------+-------------+-----------+
| 2853e96d-d182-4141-8fcf-dc64a3c05b0f | volume-multi | -           | True      |
+--------------------------------------+--------------+-------------+-----------+
(overcloud) [stack@director ~]$ cinder type-key volume-multi set multiattach="<is> True"


(overcloud) [stack@director ~]$ cinder type-show volume-multi
+---------------------------------+--------------------------------------+
| Property                        | Value                                |
+---------------------------------+--------------------------------------+
| description                     | None                                 |
| extra_specs                     | multiattach : <is> True              |
| id                              | 2853e96d-d182-4141-8fcf-dc64a3c05b0f |
| is_public                       | True                                 |
| name                            | volume-multi                         |
| os-volume-type-access:is_public | True                                 |
| qos_specs_id                    | None                                 |
+---------------------------------+--------------------------------------+

(overcloud) [stack@director ~]$ cinder create 2 --name multi-volume1 --volume-type volume-multi

(overcloud) [stack@director ~]$ cinder list
+--------------------------------------+-----------+---------------+------+--------------+----------+-------------+
| ID                                   | Status    | Name          | Size | Volume Type  | Bootable | Attached to |
+--------------------------------------+-----------+---------------+------+--------------+----------+-------------+
| 296432a6-f47c-4988-887c-65eca6ca07bb | available | multi-volume1 | 2    | volume-multi | false    |             |
+--------------------------------------+-----------+---------------+------+--------------+----------+-------------+

(overcloud) [stack@director ~]$ cinder show multi-volume1

| multiattach                    | True                                 |
...
(overcloud) [stack@director ~]$
```

# ⽐较对象存储

## OpenStack 对象存储架构

在 OpenStack Swift 中，有三种主要类型的服务器：

1. **帐户服务器 (swift_account_server)**：管理对象存储的帐户。它负责处理与帐户相关的请求，如创建和删除帐户，以及管理帐户的元数据。

2. **容器服务器 (swift_container_server)**：管理对象存储中的容器（类似于文件夹）。它负责处理与容器相关的请求，如创建、删除和列出容器，以及管理容器的元数据。

3. **对象服务器 (swift_object_server)**：管理对象存储中的实际对象（文件）。它负责处理与对象相关的请求，如上传、下载、删除对象，以及管理对象的元数据

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter6/object-store-architecture.svg)

OpenStack 对象存储系统在帐⼾、容器和对象的分层排列中容纳数据。

1. 帐⼾是容器和对象的整个层次结构开始的地⽅。

2. 容器存储并隔离对象。访问控制列表在容器级别执⾏，⽽不是在对象级别执⾏。

3. 对象存储实际的数据内容及其元数据

OpenStack 对象存储 API 以特殊格式表⽰资源路径：

```text
/v1/{account}/{container}/{object}
```

如果您将⼀个名为 object1 的对象上传到 654321 帐⼾中的 container1 容器，那么文件地址为：

```text
https://myobjectserver.example.com:8080/v1/654321/container1/object1
```

OpenStack 对象存储系统使⽤环数据库来计算特定数据应存储在集群中的物理位置。环使⽤元素之间的关系来维护记录，例如地区、区域、设备、分区和副本。

1. 地区是 OpenStack 对象存储实施的地理隔离。每个地区都有⼀个单独的服务器端点。
2. 区域可以表⽰⼀组设备、服务器，乃⾄整个数据中⼼。
3. 设备是其上具有后端⽂件系统的存储单元，通常挂载到 /srv/node ⽬录下。
4. 分区是对象存储服务运⾏的最⼩单元，⽤于确保数据的完整性和持久性。所有帐⼾、容器和对象数据都放⼊分区中。然后，在集群中分发和复制这些分区。
5. 副本决定了集群中分区的副本数。OpenStack 对象存储系统的默认副本数为 3，这意味着集群中的⼀个对象将有三个副本

## ring

使⽤ swift-ring-builder 构建帐⼾、容器和对象环时，您需要指定分区指数、副本，以及分区重新分配的最⼩时间间隔（⼩时数）

```text
+----------------------------------------------------------------------------+
|                                   Ring 环                                   |
+----------------------------------------------------------------------------+
|  地区 (Region)                                                            |
|  - 地理隔离，单独的服务器端点                                             |
+------------------------------------+---------------------------------------+
                                       |
+------------------------------------+---------------------------------------+
|           区域 (Zone)               |         区域 (Zone)                  |
|  - 一组设备或服务器，可能是独立的故障域    |  - 一组设备或服务器，可能是独立的故障域     |
+-------------------+-----------------+-------------------+-----------------+
                      |                                     |
+-------------------+-----------------+-------------------+-----------------+
|       设备 (Device)                |       设备 (Device)                |
|  - 存储单元，挂载到 /srv/node 目录下   |  - 存储单元，挂载到 /srv/node 目录下    |
|  - 后端文件系统                     |  - 后端文件系统                     |
+-------------------+-----------------+-------------------+-----------------+
                      |                                     |
+-------------------+-----------------+-------------------+-----------------+
|       分区 (Partition)             |       分区 (Partition)             |
|  - 对象存储服务的最小单元             |  - 对象存储服务的最小单元             |
|  - 包含账户、容器和对象数据           |  - 包含账户、容器和对象数据           |
+-------------------+-----------------+-------------------+-----------------+
                      |                                     |
+-------------------+-----------------+-------------------+-----------------+
|       副本 (Replica)               |       副本 (Replica)               |
|  - 分区的多个副本，确保数据可靠性       |  - 分区的多个副本，确保数据可靠性       |
|  - 默认副本数为 3                   |  - 默认副本数为 3                   |
+-------------------+-----------------+-------------------+-----------------+
```

环数据结构的字段包括：

1. 表⽰设备 ID、设备名称、地区、区域、设备权重、存储服务器 IP 地址、存储服务器侦听的端⼝以及其他元数据的设备。

2. 表⽰分区⾄设备分配的设备 ID 列表

## 重新平衡环

对 Swift 对象存储进⾏更改（如添加或删除节点或磁盘）需要您重新平衡对象存储环

这里推荐大家做一下课后练习：`lab storage-object start`

# 管理共享⽂件系统

OpenStack 共享⽂件系统服务由以下进程组成：

**manila-api**
 API 服务器将共享⽂件系统服务功能公开给租⼾⽤⼾。
**manila-scheduler**
调度程序选择哪个存储后端将为所请求的共享创建提供服务。
**manila-share**
此服务与容纳共享的后端存储系统协作。它有两种操作模式：带有或不带共享服务器处理。启⽤共享服务器模式处理后，它可以管理后端共享服务器的⽣命周期

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter6/manila-architecture.svg)

Manila管理的资源有：

1. 共享实例

2. 共享类型，例如nfs、cephfs等

3. 共享网络，共享⽹络将租⼾⽹络与共享所附加的租⼾⼦⽹绑定

4. 共享访问规则，例如rw

## 创建共享类型

命令中的 false 参数表⽰ driver_handles_share_servers 参数的值。若要动态扩展共享节点，请将此值设为 true

```bash
(undercloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ manila type-create cephfstype false
+----------------------+--------------------------------------+
| Property             | Value                                |
+----------------------+--------------------------------------+
| ID                   | e7f377e7-e4ad-4571-91cc-7f5abc0ac8df |
| Name                 | cephfstype                           |
| Visibility           | public                               |
| is_default           | -                                    |
| required_extra_specs | driver_handles_share_servers : False |
| optional_extra_specs |                                      |
| Description          | None                                 |
+----------------------+--------------------------------------+
```

## 创建共享

```bash
(overcloud) [stack@director ~]$ manila create --name host-share --share-type cephfstype cephfs 1
```

看看我们的共享信息

```bash
(overcloud) [stack@director ~]$ manila share-export-location-list host-share
+--------------------------------------+------------------------------------------------------------------------+-----------+
| ID                                   | Path                                                                   | Preferred |
+--------------------------------------+------------------------------------------------------------------------+-----------+
| 9b0b8d8f-be6c-444a-a8f0-a0631a806230 | 172.24.3.1:6789:/volumes/_nogroup/a3e94dcc-7c23-418b-8518-14e557df22fd | False     |
+--------------------------------------+------------------------------------------------------------------------+-----------+
```

## 创建访问规则

这条命令会自动创建ceph用户

```bash
(overcloud) [stack@director ~]$ manila access-allow host-share cephx ceph-user-lxh
```

导出密钥

```bash
[root@controller0 ~]# podman exec -it ceph-mgr-controller0 ceph auth get client.ceph-user-lxh > /etc/ceph/ceph.client.ceph-user-lxh.keyring
[root@controller0 ~]# cat /etc/ceph/ceph.client.ceph-user-lxh.keyring
exported keyring for client.ceph-user-lxh
[client.ceph-user-lxh]
        key = AQD0rC1nE37eDxAAtDykQVfrLM9sgIcH2z7+YQ==
        caps mds = "allow rw path=/volumes/_nogroup/a3e94dcc-7c23-418b-8518-14e557df22fd"
        caps mon = "allow r"
        caps osd = "allow rw pool=manila_data namespace=fsvolumens_a3e94dcc-7c23-418b-8518-14e557df22fd"
```

## 挂载共享

客户端需要拥有以下资源，才可以挂

1. 租户内部网络

2. `http://materials.example.com/ceph.repo`

3. yum install ceph-fuse

```bash
ceph-fuse /mnt/ceph/ \
--id=cloud-user --conf=/home/cloud-user/ceph.conf \
--keyring=/home/cloud-user/cloud-user.keyring \
--client-mountpoint=/volumes/_nogroup/a3e94dcc-7c23-418b-8518-14e557df22fd
```

# 管理临时和持久存储

**临时存储单元和持久存储单元的比较**

| 差异    | 短暂存储                         | 持久存储                         |
| ----- | ---------------------------- | ---------------------------- |
| 存在    | 在创建实例时创建。                    | 在成功执行用户的 API 请求时创建。          |
| 持久    | 实例终止后不保留。                    | 实例终止后持续存在，直到租户用户手动删除该实例。     |
| 速度和延迟 | 使用直连存储进行配置时，提供更低的延迟和更高的响应能力。 | 提供更高的延迟，但具有处理更大 I/O 工作负载的能力。 |
| 最大大小  | 取决于可用的存储容量。                  | 取决于存储容量适用的项目配额。              |
| 大小调节  | 使⽤实例类别的属性执⾏。                 | 根据用户的请求执行。                   |

当租⼾⽤⼾使⽤镜像创建实例时，该实例具有的唯⼀初始存储是临时存储。

默认情况下，计算服务在计算节点上的 /var/lib/nova/instances/UUID/ ⽬录中创建代表临时存储的后端对象。如果红帽 Ceph 存储⽤作 OpenStack 计算服务的后端存储提供商，则后端对象存储在通常名为 vms 的 Ceph 池中。

创建一个持久卷

这样创建的卷，是基于镜像的，卷里会自然包含镜像的所有内容

```bash
(undercloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ openstack image create --file osp-small.qcow2 image1
(overcloud) [stack@director ~]$ openstack volume create --size 10 --image image1 volume1
```

## 了解实例迁移

迁移实例意味着将虚拟机从⼀个计算节点移动到其他计算节点。迁移可以帮助确保实例中运⾏的服务在计算节点发⽣故障时仍然保持可⽤。迁移还有助于执⾏计算节点的计划维护。

迁移有两种基本的类型，即冷迁移和实时迁移

1. <mark>冷迁移：实例将关闭电源</mark>，然后迁移到另⼀个计算节点，这将会停机

2. 热迁移或实时迁移，这个也分为不同的方式：
   
    a. 基于共享存储的实时迁移，在这种类型的迁移中，实例的内存中数据复制到⽬标计算节点。实例的临时存储容纳在源计算节点和⽬标计算节点之间共享的后端存储中，因此实例的临时存储中的数据不需要传输到⽬标计算节点。
    b. 块实时迁移。在这种类型的迁移中，实例的内存中数据以及实例的临时存储中的数据从⼀个计算节点移动到另⼀个计算节点。需要此传输的原因在于，源计算节点和⽬标计算节点不共享相同的后端存储来容纳实例的临时存储。这种类型的迁移需要更⻓的时间才能完成，因为来⾃实例的内存⻚⾯以及临时存储的数据会导致⽹络负载增加。
    c. 由卷提供⽀持的实时迁移。此类迁移涉及使⽤持久卷的实例，⽽⾮使⽤临时磁盘的实例。在这种类型的迁移中，只有实例的内存中数据传输到⽬标计算节点上，因为卷照常保留在存储节点中，不需要与实例⼀起移动。这样可以避免⽹络负载过重并延⻓延迟。


