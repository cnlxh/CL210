```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 使⽤建议的诊断和故障排除⼯具及技术。

- 对 OpenStack ⽹络、镜像和卷服务进⾏诊断和故障排除

# 诊断 OpenStack 问题

以下⽅法可以很好地解决OpenStack 服务的问题：

1. 验证服务：确保所需的服务已启动并正在运⾏

2. 调试 OpenStack 客⼾端：在调试模式下运⾏ OpenStack 客⼾端以查找错误。

3. 查看⽇志：查看⽇志⽂件中的踪迹或错误。

## 常规故障排除⼯具

openstack安装在Linux中，所以Linux的命令可以帮助排错，例如ps、top、tail、systemctl、podman等等

举个例子，看看某个容器现在什么状态

```bash
[root@controller0 ~]# podman ps | grep nova_api
8b838d246975  director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-api:16.1-55                kolla_start           2 years ago   Up 44 hours ago           nova_api_cron
0505334574cf  director.ctlplane.overcloud.example.com:8787/gls-dle-dev-osp16-osp16_containers-openstack-nova-api:16.1-55                kolla_start           2 years ago   Up 2 hours ago            nova_api


[root@controller0 ~]# systemctl status tripleo_nova_api
● tripleo_nova_api.service - nova_api container
   Loaded: loaded (/etc/systemd/system/tripleo_nova_api.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-11-11 03:29:57 UTC; 2h 6min ago
```

## 使⽤ OpenStack 客⼾端进⾏调试

OpenStack 命令⾏客⼾端⽀持 --debug 选项。此选项将⽇志记录级别设为 DEBUG，⽽⾮默认的INFO 级别。此调试信息包括 OpenStack CLI 发送的 API 请求的详细信息，以及从 API 端点发回的响
应正⽂的详细信息。

```bash
(overcloud) [stack@director ~]$ openstack user list --debug
```

以上命令会输出很多api调用信息，但是是在屏幕上，我们可以放到文件里

```bash
(overcloud) [stack@director ~]$ openstack user list --debug --log-file lxh-debug.log
```

## ⽹络故障排除⼯具

1. ip命令查询ip信息

2. route命令查询路由信息

3. netstat或ss命令查询

## OpenStack配额管理

有时候报错也不一定是真的错误，可能是配额满了

显示项目的全局限额，包括绝对限额和速率限额:

```bash
(overcloud) [stack@director ~]$  openstack limits show --absolute
+--------------------------+-------+
| Name                     | Value |
+--------------------------+-------+
| maxTotalInstances        |    10 |
| maxTotalCores            |    20 |
| maxTotalRAMSize          | 51200 |
| maxServerMeta            |   128 |
| maxTotalKeypairs         |   100 |
| maxServerGroups          |    10 |
| maxServerGroupMembers    |    10 |
| totalRAMUsed             |  3072 |
| totalCoresUsed           |     6 |
| totalInstancesUsed       |     3 |
| totalServerGroupsUsed    |     0 |
| maxTotalVolumes          |    10 |
| maxTotalSnapshots        |    10 |
| maxTotalVolumeGigabytes  |  1000 |
| maxTotalBackups          |    10 |
| maxTotalBackupGigabytes  |  1000 |
| totalVolumesUsed         |     1 |
| totalGigabytesUsed       |     5 |
| totalSnapshotsUsed       |     0 |
| totalBackupsUsed         |     0 |
| totalBackupGigabytesUsed |     0 |
+--------------------------+-------+
```

显示项目或用户的资源配额和使用情况，更侧重于具体资源的使用情况和管理。

```bash
(overcloud) [stack@director ~]$ openstack quota show
```

## 查询容器化信息

看看他们都什么状态

```bash
[root@controller0 ~]# podman ps --format="table {{.Names}} {{.Status}}"
Names                              Status
openstack-manila-share-podman-0    Up 45 hours ago
openstack-cinder-volume-podman-0   Up 45 hours ago
ovn-dbs-bundle-podman-0            Up 45 hours ago
haproxy-bundle-podman-0            Up 45 hours ago
redis-bundle-podman-0              Up 45 hours ago
rabbitmq-bundle-podman-0           Up 45 hours ago
galera-bundle-podman-0             Up 45 hours ago
ceph-mgr-controller0               Up 45 hours ago
ceph-mds-controller0               Up 45 hours ago
ceph-mon-controller0               Up 45 hours ago
octavia_worker                     Up 45 hours ago
octavia_housekeeping               Up 45 hours ago
octavia_health_manager             Up 45 hours ago
nova_api_cron                      Up 45 hours ago
```

拿个容器看看详细配置，例如映射了哪些文件之类的

```bash
[root@controller0 ~]# podman inspect nova_api
```

顺便看看他们的日志

```bash
[root@controller0 ~]# podman logs nova_api
```

日志在物理机的以下位置

```text
[root@controller0 ~]# ls /var/log/containers/
[root@controller0 ~]# ls /var/log/containers/stdouts/
```

建议做一下练习，感受一下常规的问题排除：

```text
lab troubleshoot-diagnostic start
```

# 常⻅核⼼问题故障排除

1. 问题：您已创建了实例，但⽆法为其分配浮动 IP 地址。

这种问题，一般是因为路由器没有关联外部网关，这样就导致无法关联实例

2. 问题：您⽆法使⽤ SSH 登录实例。

这一般是因为安全组没有放心ssh流量或者ssh密钥不对

3. 问题：您已验证所有 OpenStack ⽹络组件的配置，但计算节点之间的通信或与 OpenStack 外的系统的通信仍然失败

这种就有可能是物理网络故障，需要联合网络工程师一起联合调试

4. 问题：镜像无法删除

可能是镜像受保护，解除保护再看看



以上只是列举了常见问题，具体问题，还是要从日志入手
