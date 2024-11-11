```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 红帽 OpenStack 管理二：面向云运维人员的第 2 天运维

《红帽 OpenStack 管理二：面向云运维人员的第 2 天运维》（CL210） 专为使用红帽 OpenStack 平台运维和管理功能齐全的混合云和私有云计算环境的云运维人员、服务管理员和自动化工程师而设计。 学生将学习如何管理、监控、排除故障和扩展 Red Hat OpenStack Platform 基础架构。 本课程<mark>重点介绍如何使用 OpenStack Client 命令行</mark>用户界面来配置指标、策略和架构，以支持企业云应用程序和日常运营。

## 课程目标

- 运维和管理 undercloud 和 overcloud 容器化服务、资源、基础架构和工作负载。

- 监控控制平面、网络和计算服务并对其进行故障排除。

- 安全地组织项目和域，包括用户管理和角色策略。

## 观众

- 负责管理私有云的日常运营和自动化的云操作员，以及对大规模 Red Hat OpenStack Platform 私有云或混合云感兴趣或负责维护的基础架构架构师。

## 先决条件

- 红帽认证系统管理员（红帽企业 Linux 中的 RHCSA）认证或同等经验。 <mark>具备红帽 OpenStack 管理一 （CL110） 课程或同等经验</mark>。<mark>本课程原内容不会涉及到怎么安装OpenStack，而是直接上手管理</mark>

## 课堂环境介绍

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter0/cl210-classroom-architecture.svg)

<mark>workstation</mark> 虚拟机 (VM)是<mark>唯⼀装有图形桌⾯</mark>的虚拟机

所有虚拟机共享<mark>外部⽹络 172.25.250.0/24</mark>，其⽹关为 172.25.250.254(workstation)。外部⽹络 DNS 服务同样由 workstation 提供。Overcloud 虚拟机共享⼀个内部⽹络，其含有使⽤不同的 172.24.X.0/24 地址的多个 VLAN。

其他⽤于进⾏实践练习的学员⽤虚拟机包括 lab.example.com DNS 域中的 utility 和 power，以及 overcloud.example.com DNS 域中的controller0、compute0、compute1、computehci0 和 ceph0。Overcloud 虚拟机共享<mark>调配⽹络 172.25.249.0/24</mark>，其包含 director 和 power 节点。Undercloud 使⽤独⽴的专⽤调配⽹络来部署 overcloud 节点。

环境中使⽤ classroom 服务器，作为外部⽹络的 NAT 路由器，它也作为服务器，使⽤<mark>content.example.com 和 materials.example.com</mark> 来提供特定练习的课程内容。workstation VM 也是课堂⽹络的路由器，<mark>必须保持运⾏</mark>才能让所有其他 VM 正常运作。

**教室虚拟机列表**

| 机器名称                              | IP 地址                                                                                                    | 角色                                       |
| --------------------------------- | -------------------------------------------------------------------------------------------------------- | ---------------------------------------- |
| workstation.lab.example.com       | 172.25.250.254 172.25.252.*`N`*                                                                          | 图形化学生工作站                                 |
| director.lab.example.com          | 172.25.250.200 172.25.249.200                                                                            | <mark>独立 undercloud</mark> 节点作为 director |
| power.lab.example.com             | 172.25.250.100 172.25.249.100 172.25.249.101 172.25.249.102 172.25.249.112 172.25.249.103 172.25.249.106 | 处理 overcloud 节点 <mark>IPMI 电源</mark>管理   |
| utility.lab.example.com           | 172.25.250.220 172.24.250.220                                                                            | 提供商网络的<mark>身份管理服务器</mark>和 VLAN         |
| controller0.overcloud.example.com | 172.25.250.1 172.25.249.* `172.24.X.1`                                                                   | 独立的 overcloud controller 节点              |
| compute0.overcloud.example.com    | 172.25.250.2 172.25.249.* `172.24.X.2`                                                                   | 一个 overcloud 计算节点                        |
| compute1.overcloud.example.com    | 172.25.250.12 172.25.249.* `172.24.X.12`                                                                 | 另一个 overcloud 计算节点                       |
| computehci0.overcloud.example.com | 172.25.250.6 172.25.249.* `172.24.X.6`                                                                   | 具有集成存储的 overcloud 计算节点                   |
| ceph0.overcloud.example.com       | 172.25.250.3 172.25.249.* `172.24.X.3`                                                                   | overcloud 块和对象存储服务器节点                    |
| classroom.example.com             | 172.25.254.254                                                                                           | 课堂资料和内容服务器                               |

## 账号密码

- `workstation` 的student用户密码是student

- `director`的stack用户密码是redhat

- 大多数机器的root密码是redhat

- overcloud 节点已经预先配置了`heat-admin`用户，可以无密码的从`workstation`或者`director`上用`heat-admin`访问

| System Credentials                     | Username   | Password           |
| -------------------------------------- | ---------- | ------------------ |
| Unprivileged shell login (as directed) | student    | student            |
| Privileged shell login (as directed)   | root       | redhat             |
| Undercloud node unprivileged access    | stack      | redhat             |
| Undercloud node privileged access      | root       | redhat             |
| Overcloud node unprivileged access     | heat-admin | *passwordless SSH* |
| Overcloud node privileged access       | root       | Use `sudo -i`      |

**Table 3.** 

| Application Credentials                     | Username      | Password   |
| ------------------------------------------- | ------------- | ---------- |
| Red Hat Identity Manager admin              | admin         | RedHat123^ |
| Red Hat OpenStack Platform overcloud admin  | admin         | redhat     |
| Red Hat OpenStack Platform overcloud user   | *as directed* | redhat     |
| Red Hat OpenStack Platform undercloud admin | admin         | redhat     |

## RHOSP16 教室网络拓扑

![](https://gitee.com/cnlxh/cl210/raw/master/images/Chapter0/cl210-classroom-topology.svg)
