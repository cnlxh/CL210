```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述 OpenStack 云级计算中编排的架构和实施。
- 创建编排模板，以将单⼀服务器、多个资源和⼤  规模应⽤作为堆栈进⾏部署。
- 创建 Ansible Playbook，以部署单⼀服务器、多个资源和应⽤。

# 管理云级应⽤部署

手工去部署大规模的服务时，较为繁琐，且不利于重复执行，本章将探讨红帽 OpenStack 平台 Heat 编排服务和红帽 Ansible ⾃动化平台，来这个来实现自动化部署大规模服务

云级⾃动化需要全⾯、灵活的⼯具，它们要能识别所有必要的 OpenStack 资源类型，包括定义OpenStack 资源之间关系和依赖项的资源类型。云应⽤⾃动化通常由两个层⾯构成：

1. 调配，创建和配置应⽤所需的底层基础架构。

2. 部署，在调配的基础架构上安装和配置应⽤。

## Heat 编排服务

编排是⼀个核⼼ OpenStack 组件，与⾝份服务和其他 OpenStack 服务具有良好的集成。Heat 编排的主要优势包括：

1. 了解 OpenStack 资源类型，包括实例、浮动 IP 地址、⽹络、⼦⽹、卷、卷附件和警报等

2. 具有资源组，能够创建任何数量的配置资源，即使包含复杂的嵌套堆栈也⼀样

3. 管理⾃动扩展组，以根据定义的指标⾃动横向扩展实例和连接的资源

4. 尽可能⾃动并⾏部署资源

5. 管理资源依赖项，以便以正确的顺序创建资源

6. 允许对运⾏中的堆栈进⾏堆栈增强和配置调整

7. 通过⼀个命令执⾏堆栈删除，按照依赖顺序删除资源

## 红帽 Ansible ⾃动化平台

与编排模板类似，Ansible Playbook 使⽤采⽤ YAML 编写的简单声明性语法。 通常，Ansible ⽤于配置清单⽂件中定义的调配基础架构。

Ansible 定义要执⾏的任务，可以组合到 playbook 和⻆⾊中。Playbook 针对清单中的⼀个或多个主机运⾏⼀系列任务。Ansible ⾃动化平台的主要优势包括：

1. 在数个或数百个主机上部署相同的配置

2. 使⽤已调配的基础架构

3. ⽀持通过⻆⾊实现重复利⽤，在不同主机上重复使⽤相同的软件配置

4. 通过 OpenStack Ansible 模块提供 OpenStack 调配功能

## 编排服务架构

堆栈模板和环境⽂件的输⼊参数传递到编排引擎。编排引擎解读堆栈模板和环境⽂件并部署堆栈，编排引擎⽣成的事件⽤于跟踪堆栈部署状态。

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter10/orchestration-service-arch.svg)

模板写好之后，可以用下方的方法看看有没有YAML语法错误

```bash
cat xxx.yaml | python -m json.tool
```

或者用--dry-run来模拟测试一下创建

```bash
 openstack stack create --dry-run
```

使⽤ openstack orchestration template validate 来验证模板资源结构。 验证可以捕获缺失或循环的依赖项。

# 编写 Heat 编排模板

随着每次发布 OpenStack 主要更新，编排的功能和语法得到增强，会新增版本

```bash
(overcloud) [stack@director ~]$ openstack orchestration template version list
+--------------------------------------+------+------------------------------+
| Version                              | Type | Aliases                      |
+--------------------------------------+------+------------------------------+
| AWSTemplateFormatVersion.2010-09-09  | cfn  |                              |
| HeatTemplateFormatVersion.2012-12-12 | cfn  |                              |
| heat_template_version.2013-05-23     | hot  |                              |
| heat_template_version.2014-10-16     | hot  |                              |
| heat_template_version.2015-04-30     | hot  |                              |
| heat_template_version.2015-10-15     | hot  |                              |
| heat_template_version.2016-04-08     | hot  |                              |
| heat_template_version.2016-10-14     | hot  | heat_template_version.newton |
| heat_template_version.2017-02-24     | hot  | heat_template_version.ocata  |
| heat_template_version.2017-09-01     | hot  | heat_template_version.pike   |
| heat_template_version.2018-03-02     | hot  | heat_template_version.queens |
| heat_template_version.2018-08-31     | hot  | heat_template_version.rocky  |
+--------------------------------------+------+------------------------------+
```

根据以上信息，写个模板开头

```yaml
heat_template_version: heat_template_version.2018-08-31
 description: >
   line1 of multiline
   line2 of multiline
```

后续模板中，一般会包含以下几个主要组成部分：

1. 定义输入参数

2. 定义资源

3. 定义函数

4. 定义等待条件

一般来说，在已经部署好的实例上，安装软件一般都以下三种常见方法：

1. ⾃定义 Glance 镜像

2. 使⽤ user_data 脚本和 cloud-init 来配置镜像中预安装的软件，这个需要你修改yaml中的user_data部分的代码

3. 使⽤ OS::Heat::SoftwareDeployment 资源类型

虽然徒手写heat代码具有较大的困难，但是一般来说，可以从官方摘录然后改改就行，此处我们采用课程中的lab来熟悉格式

```bash
lab automating-hot start
```

# 使⽤ Ansible 部署应⽤

## OpenStack 模块简介

最新的 OpenStack 模块使⽤前缀 os_ 作为标准。没有此前缀的较旧模块正在逐渐弃⽤，带有 _info 后缀的模块从服务或组件收集信息。例如，os_network_info 模块可检索⼀个或多个⽹络的信息

模块列表：

```textile
https://docs.ansible.com/ansible/2.9/modules/list_of_cloud_modules.html#openstack
```

模块执行之前，需要先通过openstack平台的身份验证，一般可以有这么几种方法：

1. source xxxrc

2. ⾝份验证值也可以来⾃ /etc/ansible/openstack.yaml、/etc/openstack/clouds.yaml 或 ~/.config/openstack/clouds.yaml

3. playbook变量或模块的auth参数

## 运行openstack模块

先身份认证

```bash
[root@workstation ~]# scp stack@director:~/overcloudrc .
overcloudrc                                                                                                                 100%  961   448.8KB/s   00:00
[root@workstation ~]# source overcloudrc
(overcloud) [root@workstation ~]#
```

看看模块列表

```bash
(overcloud) [root@workstation ~]# ansible-doc -l | grep ^os_
os_auth                                                       Retrieve an auth token
os_client_config                                              Get OpenStack Client config
os_coe_cluster                                                Add/Remove COE cluster from OpenStack Cloud
os_coe_cluster_template                                       Add/Remove COE cluster template from OpenStack Cloud
os_flavor_info                                                Retrieve information about one or more flavors
```

运行模块的格式如下：

```bash
ansible -m os_xxxxx -a 'xxx=xxx xxx=xxx' HOSTNAME
```

playbook的模块书写样例获取方式如下：

ansible-doc os_server

```yaml
EXAMPLES:

- name: Create a new instance and attaches to a network and passes metadata to the instance
  os_server:
       state: present
       auth:
         auth_url: https://identity.example.com
         username: admin
         password: admin
         project_name: admin
       name: vm1
       image: 4f905f38-e52a-43d2-b6ec-754a13ffb529
       key_name: ansible_key
       timeout: 200
       flavor: 4
       nics:
         - net-id: 34605f38-e52a-25d2-b6ec-754a13ffb723
         - net-name: another_network
       meta:
         hostname: test1
         group: uge_master

# Create a new instance in HP Cloud AE1 region availability zone az2 and
# automatically assigns a floating IP
- name: launch a compute instance
  hosts: localhost
  tasks:
    - name: launch an instance
      os_server:
        state: present
        auth:
          auth_url: https://identity.example.com
          username: username
          password: Equality7-2521
          project_name: username-project1
        name: vm1
        region_name: region-b.geo-1
        availability_zone: az2
        image: 9302692b-b787-4b52-a3a6-daebb79cb498
        key_name: test
        timeout: 200
        flavor: 101
        security_groups: default
        auto_ip: yes
```

同样作为复习rhce的ansible语法和熟悉openstack模块，我们也可以通过做课程练习的方式来熟悉：

```bash
lab automating-ansible start
```

和

```bash
lab automating-review start
```
