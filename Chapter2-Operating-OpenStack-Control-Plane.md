```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 识别在控制器节点上运⾏的共享服务。
- 管理消息和消息代理。
- 备份和恢复控制平⾯。

# 识别控制平⾯服务和操作

## 共享服务概述

- MariaDB

- Redis

- memcached

- Pacemaker

- Ceph MON and MDS

- NoVNC and Spice

## Ceph MON 和 MDS

Ceph 存储集群需要监控服务，Ceph MON 守护程序则提供此功能。它在各个控制器节点上运⾏，并提供 Ceph 集群配置信息的可靠存储。为了管理 Ceph ⽂件系统命名空间并⽀持 CephFS，Ceph 元数据服务器 (MDS) 也在控制器节点上运⾏。

## NoVNC 和 SPICE

OpenStack 包括并⽀持两种协议，⽤于提供远程虚拟控制台访问，分别是虚拟⽹络计算机 (VNC) 和独⽴计算环境简单协议 (SPICE)。

- VNC 是默认、稳定且成熟的协议，在 OpenStack 中使⽤ noVNC 开源客⼾端实施，noVNC 由 JavaScript 库和使⽤ HTML5 和 WebSockets 在桌⾯和移动浏览器中运⾏的应⽤程序组成。

- 管理员可以选择使⽤ SPICE，⽽不使⽤ VNC。SPICE 是红帽⽀持的开源项⽬，旨在提供⽐现有 VNC客⼾端更⾼级的功能。 虽然 SPICE ⽐ VNC 更有优势，但很多 SPICE-HTML5 浏览器插件不⽀持增强功能。要使⽤ SPICE 增强功能，如多监视器、USB 设备直通、⾳频和 3D 图形，红帽建议使⽤<mark>独⽴的 SPICE 客⼾端</mark>从专⽤管理⽹络访问 OpenStack 实例。

<mark>用VNC访问实例控制台的流程</mark>

每个计算节点运⾏⼀个 vncserver 进程，在内部 API ⽹络上侦听端⼝号为 5900 及以上的⼀个或多个端⼝，具体取决于该计算节点上部署的实例数量。每个控制器节点运⾏⼀个 novncproxy 进程，在同⼀内部 API ⽹络上侦听端⼝ 6080。其余的服务从属于计算服务 (Nova)，其组件位于控制器和计算节点上

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter2/controlplane-services-vnc-flow.svg)

1. 浏览器通过 NoVNC 插件连接到 nova-api（controller，内部 API，端⼝ 8774），以请求打开⼀个控制台来连接正在运⾏的特定服务器实例。OpenStack dashboard 连接使⽤ haproxy（controller，外部，端⼝ 80）

2. Nova-api 传递 get_vnc_console 请求到 nova-compute（compute，内部API，AMQP）

3. nova-compute 传递 get_vnc_console 请求到 libvirt (compute) 以查找虚拟机属性

4. libvirt 从虚拟机 UUID ⽣成令牌，并将令牌返回到 nova-compute（compute，内部API，AMQP）

5. nova-compute 将⽣成的令牌和 connect_info 对象返回到 nova-api（controller，内部 API，AMQP）

6. nova-api 传递 authorize_console 请求到 nova-consoleauth（compute，内部API，AMQP），这将 connect_info 对象及令牌缓存为索引，供以后发⽣实际连接请求时使⽤

7. nova-api 也将 nova-novncproxy URL 和实例特定令牌返回到浏览器（workstation，外部）

8. 浏览器（workstation，外部）通过 dashboard haproxy（controller，外部，端⼝6080）代理连接到该 URL，以访问 nova-novncproxy（controller，内部 API，端⼝6080）

9. nova-novncproxy 从该 URL 解析令牌和实例 ID，并将它传递到 nova-consoleauth（controller，内部 API，AMQP）

10. 利⽤这个令牌，nova-consoleauth 从缓存检索 connect_info 对象并将它返回到 nova-novncproxy（controller，内部 API）

11. nova-novncproxy 通过为所请求虚拟机指定的端⼝直接连接到 vncserver（compute，内部 API，5900+），并创建反向代理配置

12. nova-novncproxy 通过 dashboard haproxy 将控制台图形发回到⽤⼾的浏览器（workstation，外部）

# 描述组件通信

OpenStack 服务的后端基于两种功能，即：实现持久性的数据库，以及⽀持各项服务不同组件之间通信的消息代理。任何⽀持 AMQP 的消息代理解决⽅案都可⽤作后端。本次课程中RabbitMQ作为在其 OpenStack 产品中使⽤的消息代理

消息代理在创建者和使用者应用程序之间发送和接收消息。 在内部，RabbitMQ 使用交换和队列以及它们之间的绑定来执行此通信。 当应用程序生成要发送到一个或多个使用者应用程序的消息时，它会将消息放置在一个或多个队列绑定到的交换中。 交换将消息路由到相关队列。 使用者订阅队列以接收创建者的消息。 路由基于传输消息中包含的路由密钥。

以下是RabbitMQ的一些基本术语

| Term               | Description                         |
| ------------------ | ----------------------------------- |
| Binding            | 用于从 exchange 路由到队列的 key 或 filter 参数 |
| Consumer           | 使用接收到的消息的应用程序                       |
| Exchange           | 通过消息元数据将创建者发布的消息路由到队列               |
| Publisher/Producer | 发布消息的应用程序                           |
| Queue              | 存储消息，直到应用程序使用为止                     |
| Routing key        | ⽣产者指定的消息元数据，供交换⽤于消息路由               |
| Vhost              | RabbitMQ 实例的逻辑划分，用于隔离应用程序、队列和交换     |

## 消息代理交换概念

交换与队列的交互基于消息中包含的路由密钥与相关交换上队列关联的绑定密钥之间的匹配关系。根据这两个元素的使⽤情况，RabbitMQ 中提供多种类型的交换:

**MQ基本概念图示**

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter2/message-broker-work-queue.svg)

**Exchange类型：直接**

taskC_create 消息路由到名为 taskC_queue 的队列

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter2/message-broker-binding.svg)

**Exchange类型：主题订阅**

- 通过在路由模式中使⽤通配符，消息可以同时发送到⼀个或多个队列

- 只有计算任务路由到名为compute_tasks 的队列，但包括计算在内的所有任务都路由到名为 all_tasks 的队列。

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter2/message-broker-topic.svg)

## 使⽤消息队列实施 RPC

当进程需要回复请求时，它会发布⼀条消息，其中包含供接收者⽤于响应的命名队列。请求进程然后订阅该队列并继续轮询响应，直到收到响应。服务器端的主题消费者⼯作者处理原始请求，再实例化发布者以创建响应

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter2/message-broker-rpc.svg)

<mark>综上所述，基本上所有的openstack核心组件，都通过rabbitmq发送和接收消息，在排除一些故障时，可以考虑查看rabbitmq中的消息或相关的组件日志</mark>

## rabbitmq实操

使⽤ report 命令可显⽰ RabbitMQ 守护进程的当前状态摘要，包括交换和队列的数量及类型

```bash
[root@controller0 ~]# podman exec -it rabbitmq-bundle-podman-0 /bin/bash
()[root@controller0 /]# rabbitmqctl report
Reporting server status of node rabbit@controller0 ...

Status of node rabbit@controller0 ...
[{pid,677},
 {running_applications,
     [{rabbitmq_management,"RabbitMQ Management Console","3.7.23"},
      {rabbitmq_web_dispatch,"RabbitMQ Web Dispatcher","3.7.23"},
      {amqp_client,"RabbitMQ AMQP Client","3.7.23"},
      {rabbitmq_management_agent,"RabbitMQ Management Agent","3.7.23"},
      {rabbit,"RabbitMQ","3.7.23"},
      {mnesia,"MNESIA  CXC 138 12","4.15.6"},
      {rabbit_common,
          "Modules shared by rabbitmq-server and rabbitmq-erlang-client",
          "3.7.23"},
```

使⽤ list_users 命令来列出 RabbitMQ ⽤⼾

```bash
()[root@controller0 /]# rabbitmqctl list_users
Listing users ...
user    tags
guest   [administrator]
```

使⽤ add_user命令来创建RabbitMQ ⽤⼾

```bash
()[root@controller0 /]# rabbitmqctl add_user lixiaohui lxhpassword
Adding user "lixiaohui" ...
()[root@controller0 /]# rabbitmqctl list_users
Listing users ...
user    tags
lixiaohui       []
guest   [administrator]
```

使⽤ set_permissions命令来设置RabbitMQ ⽤⼾权限

```bash
()[root@controller0 /]# rabbitmqctl set_permissions lixiaohui '.*' '.*' '.*'
Setting permissions for user "lixiaohui" in vhost "/" ...
()[root@controller0 /]# rabbitmqctl list_permissions
Listing permissions for vhost "/" ...
user    configure       write   read
lixiaohui       .*      .*      .*
guest   .*      .*      .*
```

使⽤ set_user_tags命令来设置⽤⼾标签

```bash
()[root@controller0 /]# rabbitmqctl set_user_tags lixiaohui administrator
Setting tags for user "lixiaohui" to [administrator] ...
()[root@controller0 /]# rabbitmqctl list_users
Listing users ...
user    tags
lixiaohui       [administrator]
guest   [administrator]
```

使⽤ list_exchanges 显⽰ RabbitMQ 守护进程上默认配置的交换

```bash
()[root@controller0 /]# rabbitmqctl list_exchanges
Listing exchanges for vhost / ...
name    type
heat-engine-listener_fanout     fanout
manila-share_fanout     fanout
```

使⽤ list_queues 命令可列出可⽤的队列及其属性

```bash
()[root@controller0 /]# rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name    messages
compute.compute1.overcloud.example.com  0
compute.computehci0.overcloud.example.com       0
manila-scheduler.hostgroup      0
```

使⽤ list_consumers 命令可列出所有的消费者，以及它们所订阅的队列

```bash
()[root@controller0 /]# rabbitmqctl list_consumers
Listing consumers in vhost / ...
queue_name      channel_pid     consumer_tag    ack_required    prefetch_count  arguments
compute.compute1.overcloud.example.com  <rabbit@controller0.1.1237.0>   2       true    0       []
```

## 跟踪 RabbitMQ 消息

RabbitMQ 有⼀个称为 Firehose Tracer 的内置功能，可⽤来跟踪所有消息。启⽤之后，进⼊系统的所有消息都将复制到 amq.rabbitmq.trace 交换，跟踪会增加系统负载，因此请确保在调查完成后禁⽤跟踪。

**使⽤ rabbitmqctl trace_on 和 rabbitmqctl trace_off 命令可以启⽤和禁⽤跟踪**

---

# 备份和恢复控制平⾯

## 备份恢复工具介绍

红帽企业 Linux 提供了⼀个恢复和系统迁移实⽤程序，称为 Relax-and-Recover (ReaR)。

ReaR 采⽤ Bash 来编写，⽀持通过不同⽹络传输⽅式分发救援镜像或备份⽂件存储。ReaR 会⽣成可引导镜像，并使⽤此镜像从备份中恢复。它提供了⼀种有效的⽅法，来备份和恢复红帽 OpenStack平台控制平⾯节点。 

**ReaR ⽀持下列引导介质格式：**

- ISO

- USB

- eSATA

- PXE

**ReaR 可以使⽤下列协议来传输⽂件：**

- HTTP/HTTPS
- SSH/SCP
- FTP/SFTP
- NFS
- CIFS (SMB)

## 使⽤ Ansible ⾃动备份和恢复控制平⾯

Ansible 是⽤于创建和恢复由 ReaR ⽣成的镜像的主要⽅法。

tripleo-ansible 软件包提供必要的⽂件和 Ansible Playbook，以运⾏ backup-and-restore ⻆⾊。这些⽂件位于各个控制平⾯节点的 /usr/share/ansible/roles/backup-and-restore⽬录中。

```bash
(undercloud) [stack@director ~]$ tree /usr/share/ansible/roles/backup-and-restore/
/usr/share/ansible/roles/backup-and-restore/
├── backup
│   └── tasks
│       ├── db_backup.yml
│       ├── main.yml
│       ├── service_manager_pause.yml
│       └── service_manager_unpause.yml
├── defaults
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   └── default
│       ├── Dockerfile
│       ├── molecule.yml
│       ├── playbook.yml
│       └── prepare.yml
├── setup_nfs
│   └── tasks
│       └── main.yml
├── setup_rear
│   └── tasks
│       └── main.yml
├── tasks
│   ├── main.yml
│   ├── pacemaker_backup.yml
│   ├── setup_nfs.yml
│   └── setup_rear.yml
├── templates
│   ├── exports.j2
│   ├── local.conf.j2
│   └── rescue.conf.j2
└── vars
    └── redhat.yml

13 directories, 20 files
```

看看都有哪些变量

```bash
(undercloud) [stack@director ~]$ cd /usr/share/ansible/roles/backup-and-restore/
(undercloud) [stack@director backup-and-restore]$ cat vars/redhat.yml
---
tripleo_backup_and_restore_rear_packages:
  - rear
  - syslinux
  - genisoimage
  - nfs-utils
tripleo_backup_and_restore_nfs_packages:
  - nfs-utils
```

再看看优先级较低的默认变量

```bash
(undercloud) [stack@director backup-and-restore]$ cat defaults/main.yml | grep -v -e ^$ -e ^#
---
tripleo_container_cli: "{{ container_cli | default('docker') }}"
tripleo_backup_and_restore_service_manager: true
tripleo_backup_and_restore_mysql_container: mysql
tripleo_backup_and_restore_debug: false
tripleo_backup_and_restore_nfs_server: 192.168.24.1
tripleo_backup_and_restore_nfs_storage_folder: /ctl_plane_backups
tripleo_backup_and_restore_nfs_clients_nets: ['192.168.24.0/24', '10.0.0.0/24', '172.16.0.0/24']
tripleo_backup_and_restore_rear_simulate: false
tripleo_backup_and_restore_using_uefi_bootloader: 0
tripleo_backup_and_restore_exclude_paths_common: ['/data/*', '/tmp/*', '{{ tripleo_backup_and_restore_nfs_storage_folder }}/*']
tripleo_backup_and_restore_exclude_paths_controller_non_bootrapnode: true
tripleo_backup_and_restore_exclude_paths_controller: ['/var/lib/mysql/*']
tripleo_backup_and_restore_exclude_paths_compute: ['/var/lib/nova/instances/*']
tripleo_backup_and_restore_hiera_config_file: "/etc/puppet/hiera.yaml"
tripleo_backup_and_restore_local_config:
  ISO_DEFAULT: '"automatic"'
  USING_UEFI_BOOTLOADER: 0
  OUTPUT: ISO
  BACKUP: NETFS
  BACKUP_PROG_COMPRESS_OPTIONS: '( --gzip)'
  BACKUP_PROG_COMPRESS_SUFFIX: '".gz"'
tripleo_backup_and_restore_rescue_config: {}
tripleo_backup_and_restore_output_url: "nfs://{{ tripleo_backup_and_restore_nfs_server }}/ctl_plane_backups"
tripleo_backup_and_restore_backup_url: "nfs://{{ tripleo_backup_and_restore_nfs_server }}/ctl_plane_backups"
```

看看都有哪些tasks

1. 先按照部署NFS服务器

2. 再按照配置rear工具

3. 暂停服务

4. 备份数据库

5. 备份packmaker

6. 用rear备份

7. 恢复服务

从步骤看来，我们需要确定NFS服务器的地址和存储位置，并设置将备份放到NFS上的步骤

```bash
(undercloud) [stack@director backup-and-restore]$ cat tasks/main.yml
---
- name: Gather variables for each operating system
  include_vars: "{{ item }}"
  with_first_found:
    - skip: true
      files:
        - "{{ ansible_distribution | lower }}-{{ ansible_distribution_version | lower }}.yml"
        - "{{ ansible_distribution | lower }}-{{ ansible_distribution_major_version | lower }}.yml"
        - "{{ ansible_os_family | lower }}-{{ ansible_distribution_major_version | lower }}.yml"
        - "{{ ansible_distribution | lower }}.yml"
        - "{{ ansible_os_family | lower }}-{{ ansible_distribution_version.split('.')[0] }}.yml"
        - "{{ ansible_os_family | lower }}.yml"
  tags:
    - always

- name: Setup NFS server
  import_tasks: ../setup_nfs/tasks/main.yml

- name: Setup ReaR
  import_tasks: ../setup_rear/tasks/main.yml

- name: Service management
  import_tasks: ../backup/tasks/service_manager_pause.yml
  when:
    - tripleo_backup_and_restore_service_manager

- name: Backup the database
  import_tasks: ../backup/tasks/db_backup.yml

- name: Backup pacemaker configuration
  import_tasks: pacemaker_backup.yml

- name: Create recovery images with ReaR
  import_tasks: ../backup/tasks/main.yml

- name: Service management
  import_tasks: ../backup/tasks/service_manager_unpause.yml
  when:
    - tripleo_backup_and_restore_service_manager
```

## 备份和恢复控制平⾯

<mark>lab controlplane-backup start</mark>

### 将 utility 服务器配置为备份节点

```bash
cat > nfs-inventory.ini <<-'EOF'
[backup_node]
backup ansible_host=172.25.250.1 ansible_user=root
EOF
```

### 创建控制平⾯的静态清单⽂件

```bash
(undercloud) [stack@director ~]$ tripleo-ansible-inventory --ansible_ssh_user heat-admin --static-yaml-inventory tripleo-inventory.yaml
```

### 在 director 服务器上安装和配置 ReaR

```bash
ansible-playbook -v -i ~/tripleo-inventory.yaml \
--extra="ansible_ssh_common_args='-o StrictHostKeyChecking=no'" \
--become --become-user root --tags bar_setup_rear \
~/bar_rear_setup-undercloud.yaml
```

### 在 controller0 服务器上安装和配置 ReaR

```bash
ansible-playbook -v -i ~/tripleo-inventory.yaml \
-e tripleo_backup_and_restore_exclude_paths_controller_non_bootrapnode=false \
--extra="ansible_ssh_common_args='-o StrictHostKeyChecking=no'" \
--become --become-user root --tags bar_setup_rear \
~/bar_rear_setup-controller.yaml
```

### 创建 director 服务器的备份

```bash
ansible-playbook -v -i ~/tripleo-inventory.yaml \
--extra="ansible_ssh_common_args='-o StrictHostKeyChecking=no'" \
--become --become-user root --tags bar_create_recover_image \
~/bar_rear_create_restore_images-undercloud.yaml
```

此操作可能需要很⻓时间，可以在运行期间，去utility看看文件

```bash
[root@utility ~]# exportfs -rav
exporting 172.25.250.0/24:/ctl_plane_backups
[root@utility ~]# ls /ctl_plane_backups/director/
backup.tar.gz  director.lab.example.com.iso  README  rear-director.log  VERSION
```
