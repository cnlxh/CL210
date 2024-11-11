```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述适⽤于 OpenStack ⾝份服务的红帽⾝份管理后端的安装和架构
- 管理实施⽤⼾授权以访问 OpenStack 服务的⽤⼾令牌
- 管理项⽬配额、域、层次结构和组
- ⾃定义⽤⼾⻆⾊。

# 管理集成式 IdM 后端配置

## ⾝份服务概述

OpenStack <mark>⾝份服务 (Keystone) </mark>利⽤内部服务功能<mark>提供⾝份验证、基于⻆⾊的授权、策略管理和令牌处理</mark>。这些功能归类为⾝份、策略、令牌、资源、⻆⾊分配和⽬录。

⾝份服务 API 在可配置的端点上提供，这些端点按照公共和内部流量划分。多个控制器节点可通过虚拟机 IP (VIP) 地址使⽤Pacemaker 来提供 API 冗余访问

<mark>⽤⼾从 OpenStack 组件请求服务前要先通过⾝份验证，且必须分配有⻆⾊，才能参与到项⽬中</mark>。可以使⽤⾝份服务 v3 中引⼊的组来管理⽤⼾，组可按照与单个⽤⼾相同的⽅式分配⻆⾊并附加
到项⽬。项⽬（以前称为租⼾）是所拥有的资源的集合，如⽹络、镜像、服务器和安全组。它们按照组织的开发需求进⾏构造。项⽬可以代表客⼾、帐⼾或任何组织单元。在⾝份服务 v3 中，项⽬可以包含⼦项⽬，⼦项⽬从⽗项⽬中继承项⽬⻆⾊分配和配额。

RHOSP ⽀持多种⾝份验证⽅法。最基本的⾝份验证⽅法根据存储在本地数据库中的帐⼾信息验证⽤⼾名和密码。在⼤型企业环境中，RHOSP 可以针对外部⾝份验证后端验证⽤⼾凭据，例如 Active Directory (AD)、红帽 IdM 或通⽤ LDAP 服务器。

在openstack中，有domain、project、group、user的概念，具体概念如下

```text
                   +--------------------+
                   |    OpenStack       |
                   +--------------------+
                               |
          +-----------------------------------------+
          |                                         |
  +------------------+                      +------------------+
  |      Domain 1    |                      |      Domain 2    |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |     Project 1    |                      |     Project 3    |
  +------------------+                      +------------------+
  |     Project 2    |                      |     Project 4    |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |     Group A      |                      |     Group C      |
  +------------------+                      +------------------+
  |     Group B      |                      |     Group D      |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |      User A      |                      |      User C      |
  +------------------+                      +------------------+
  |      User B      |                      |      User D      |
  +------------------+                      +------------------+
```

## 安装红帽 IDM

红帽 IdM 将 LDAP、Kerberos、DNS 和 PKI 协议与富管理框架相结合，为企业环境中的 Linux 客⼾端提供⾝份服务。IdM 是红帽调优上游开源 FreeIPA 项⽬并提供⽀持的发⾏版本，该项⽬构建⾃⼴受欢迎的 Linux 管理组件，如 PAM、BIND DNS、MIT Kerberos 和 Dogtag PKI，并与核⼼ 389 LDAP Directory Server 集成。

```bash
dnf install freeipa-server-dns -y
ipa-server-install --setup-dns --auto-reverse --mkhomedir
```

## 集成IDM

1. IDM服务器必须先创建出用于集成的用户和组

2. keystone服务必须具有IDM的CA证书

3. 修改keystone配置，以便于支持特定的域名

4. 在keystone启动多域支持

先在workstation上打一下lab，摧毁现有Example域的配置

```bash
 lab identity-domain start
```

### 创建用户

kinit的密码是RedHat123^

```bash
[root@utility ~]# kinit admin
Password for admin@LAB.EXAMPLE.NET:
[root@utility ~]# echo lxhpassword | ipa user-add lxh --first=OpenStack --last=lixiaohui --password
----------------
Added user "lxh"
----------------
  User login: lxh
  First name: OpenStack
  Last name: lixiaohui
  Full name: OpenStack lixiaohui
  Display name: OpenStack lixiaohui
  Initials: Ol
  Home directory: /home/lxh
  GECOS: OpenStack lixiaohui
  Login shell: /bin/sh
  Principal name: lxh@LAB.EXAMPLE.NET
  Principal alias: lxh@LAB.EXAMPLE.NET
  User password expiration: 20240331084046Z
  Email address: lxh@lab.example.net
  UID: 365000043
  GID: 365000043
  Password: True
  Member of groups: ipausers
  Kerberos keys available: True
```

创建OpenStack组，并将用户加入到组

```bash
[root@utility ~]# ipa group-add openstack
-----------------------
Added group "openstack"
-----------------------
  Group name: openstack
  GID: 365000044
[root@utility ~]# ipa group-add-member openstack --users=lxh
  Group name: openstack
  GID: 365000044
  Member users: lxh
-------------------------
Number of members added 1
-------------------------
```

```bash
[root@utility ~]# ipa group-show openstack --all
  dn: cn=openstack,cn=groups,cn=accounts,dc=lab,dc=example,dc=net
  Group name: openstack
  GID: 365000044
  Member users: lxh
  ipauniqueid: 9a3ff6c4-a881-11ee-b76d-52540000fadc
  objectclass: top, groupofnames, nestedgroup, ipausergroup, ipaobject, posixgroup
```

### 安装CA证书

登录到keystone所在的controller0，然后把CA证书拷贝过来

```bash
[root@controller0 ~]# scp root@utility:/etc/ipa/ca.crt /root/
The authenticity of host 'utility (172.25.250.220)' can't be established.
ECDSA key fingerprint is SHA256:1H687jfusVXYAUzAuByFfx1U/lB4VS+6h04wRhXhmZU.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'utility,172.25.250.220' (ECDSA) to the list of known hosts.
Password:
ca.crt                                                                                                                      100% 1651     2.6MB/s   00:00
[root@controller0 ~]#
```

将证书转为PEM格式

```bash
[root@controller0 ~]# openssl x509 -in ca.crt -out idm_ca.pem -outform PEM
```

将证书拷贝到根信任列表并信任此CA

```bash
[root@controller0 ~]# cp idm_ca.pem /etc/pki/ca-trust/source/anchors/
[root@controller0 ~]# update-ca-trust extract
```

允许通过 LDAP 进行身份验证时使用 nsswitch（名字服务切换)

```bash
[root@controller0 ~]# setsebool -P authlogin_nsswitch_use_ldap=on
```

### keystone 容器中启⽤多域配置

这里的42425是容器中的keystone用户

```bash
[root@controller0 ~]# mkdir /var/lib/config-data/puppet-generated/keystone/etc/keystone/domains/
[root@controller0 ~]# podman exec -it keystone /bin/bash
()[root@controller0 /]# grep 42425 /etc/passwd
keystone:x:42425:42425::/var/lib/keystone:/usr/sbin/nologin
()[root@controller0 /]# exit
exit
[root@controller0 ~]# chown -R 42425:42425 /var/lib/config-data/puppet-generated/keystone/etc/keystone/domains/
```

开启多域支持，指定多域配置文件路径，将指定sql来存储和管理角色分配

```bash
[root@controller0 ~]# cd /var/lib/config-data/puppet-generated/keystone/etc/keystone/
[root@controller0 keystone]# crudini --set keystone.conf identity domain_specific_drivers_enabled true
[root@controller0 keystone]# crudini --set keystone.conf identity domain_config_dir /etc/keystone/domains
[root@controller0 keystone]# crudini --set keystone.conf assignment driver sql
```

创建出特定域的配置文件

不用担心，考试中，会提供keystone的配置文件样例，我们只需要注意这里的user和password和证书路径就行

这些域⾝份验证配置⽂件的⽂件名<mark>必须符合 keystone.DOMAIN.conf 格式，其中DOMAIN 是域的名称</mark>。

```bash
cat > domains/keystone.Example.conf <<-EOF
[ldap]
url =  ldaps://utility.lab.example.com
user = uid=lxh,cn=users,cn=accounts,dc=lab,dc=example,dc=net
password = lxhpassword
user_tree_dn = cn=users,cn=accounts,dc=lab,dc=example,dc=net
user_objectclass = inetUser
user_id_attribute = uid
user_name_attribute = uid
user_mail_attribute = mail
user_pass_attribute =
user_allow_create = False
user_allow_update = False
user_allow_delete = False
group_objectclass = groupofnames
group_tree_dn = cn=groups,cn=accounts,dc=lab,dc=example,dc=net
group_id_attribute = cn
group_name_attribute = cn
group_allow_create = False
group_allow_update = False
group_allow_delete = False
use_tls = False
query_scope = sub
chase_referrals = false
tls_cacertfile = /etc/pki/ca-trust/source/anchors/idm_ca.pem
[identity]
driver = ldap
EOF
```

```bash
[root@controller0 keystone]# chown 42425:42425 -R domains/
```

重启容器生效以上配置

推荐使用systemctl管理，非要使用podman管理也不是不行，只是有时候可能会导致systemctl服务异常，建议用systemctl

```bash
[root@controller0 ~]# systemctl restart tripleo_keystone
```

### dashboard中启用多域名支持

确保文件中包含以下两行

```bash
[root@controller0 keystone]# cd /var/lib/config-data/puppet-generated/horizon/etc/openstack-dashboard/
[root@controller0 openstack-dashboard]# vim local_settings
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
```

重启容器生效以上配置

```bash
[root@controller0 ~]# systemctl restart tripleo_horizon
```

### 创建测试域

这里的域名要注意和配置文件中的名称保持一致，包括大小写什么的，而且如果无法userlist可以试试用podman restart keystone

```bash
(undercloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ openstack domain list
+----------------------------------+------------+---------+--------------------+
| ID                               | Name       | Enabled | Description        |
+----------------------------------+------------+---------+--------------------+
| 1e9c34fd067a4c88a14959ce575aeb45 | heat_stack | True    |                    |
| default                          | Default    | True    | The default domain |
+----------------------------------+------------+---------+--------------------+
(overcloud) [stack@director ~]$ openstack domain create Example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| enabled     | True                             |
| id          | d8de20f9ca8242739cdb99860857fbae |
| name        | Example                          |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+
(overcloud) [stack@director ~]$ openstack user list --domain Example
+------------------------------------------------------------------+------------+
| ID                                                               | Name       |
+------------------------------------------------------------------+------------+
| d4c1be0784dd81f091a22446f65cec8dd7639c183ddad16c6854caea38ebe003 | admin      |
| 16d15e2d8d61c94e12bd6df5c28475c3f44c330680b06ad6df8d26b5da34f767 | architect1 |
| f8d38b17e3cf6f14d320a0074beea8c65c2074627276f1454cf18f72c9f901e2 | architect2 |
| dcad4d79d957b682545e36f7b53b0b85d413e1d11d3a5a0ab804987c73ce4d7c | architect3 |
| f149ccc0b2de39ffd35f969c44b4885630da0f2ddb2526ac29b1ffbd7b63a5c8 | architect4 |
| 4df7bbabc1a5409e35639b291dea723a1ce5dd622425d802658692aed3dacd7b | architect5 |
| 8b3d00e1d6ba05645fa66eb668dfa6d0e322465f48f47e15779a251eed515b38 | architect6 |
| 947cd30e90148b30e9b29f036aea0acbc73ddb13df406ed880c8df9749edb970 | architect7 |
| 2b8a3d49fcc2e9bb241867325e3ff8ab679be7600d699a365852a1f836eee877 | architect8 |
| 9497f55a848562c77011277466735c81dc0b970fac052868d77cd6304f3e66d0 | architect9 |
| 553c86846df02194d961883e015175341c2bd97f243b9b3707b2150e27f2d8d8 | developer1 |
| 7700fc93c6193081ca6a1d03945ca08b92ad5205174696e2c8936d3bdbf62175 | developer2 |
| 357fd08835ce3fd72f452baa16ff30ab50d3df89b39515fda406808c6ee704bb | developer3 |
| 7a01e68ab1a5269dd3ecc0310028298f3c05ef3cf039cb0790891b9269c13309 | developer4 |
| 38e30324089fce5f29c34c83b0dab8148249c67e57f50fc0920411a852b7842c | developer5 |
| 9de510cb622ba78e3f7b48aa3749d7a6828118d013c288b8d2224f45f988b398 | developer6 |
| 27d496574c7f91fed6f8daa1f75bb5a8a134e0ce673648545e82014171d22db5 | developer7 |
| c40fe1a170b384d315197d33a86fbda4eecc445afd4e98306ca6ae5c2c632db6 | developer8 |
| 2e8616cf247f30920cb702506e6a48335dd2d263d88854dadf978c57b98504c9 | developer9 |
| d5efdbab2b9e7068339f96e0d25048f93f28c4a9eba7083d0a0509a3038eba0c | operator1  |
| 996f64edb812e8efc67d83ee481b4d1a5831a90596ed618fb3786ba2077ebc22 | operator2  |
| e7ebe54acbbf4a1e00f34af7162ec818b620fe33f4a01bb1b33e74bb9364e29c | operator3  |
| a7a724871b2886b80cb027283a9715bacf36793a016ce5b81ddcab3e8b8d2b5c | operator4  |
| f0b04b7f34d50a40a778570222525f1d06da80603149536c27300d195f7f440b | operator5  |
| 5f2dc40ae6f977f9baa49127c7744bbecdc81530836fd1b57c3e95e5807fbb6f | operator6  |
| b9a5b749bedbb1adcd99ad66dc8e55b663ff3964d1dd8fe2a78b51d2318c12a6 | operator7  |
| f9c24fcd3f354bb3f089dd5c97708ef1725fd92d6e0a4a0744b69e3c1c8a3c4c | operator8  |
| ce687226b82563cfe349a15c15f28b0eee955370600739da5a97358d48b52290 | operator9  |
| 0650858bdef9aa4523e96c7224d50f33add555dc3e9c7ee827d86dd377ce249e | lxh        |
+------------------------------------------------------------------+------------+
```

# 管理⾝份服务令牌

⾝份服务通过由插件配置指定的⾝份验证流程确认⽤⼾的⾝份，然后为该⽤⼾提供代表⽤⼾⾝份的令牌。典型的⽤⼾令牌有作⽤域，即它会列出可以使⽤它的资源和访问权限。令牌寿命有限，允许⽤⼾⽆需进⼀步⾝份验证即可执⾏服务请求，直到令牌到期或被撤销。有作⽤域令牌列出⽤⼾权限和特权，如当前项⽬相关的⻆⾊中所定义。当⽤⼾向 OpenStack 服务发出请求时，服务会根据⽤⼾提供的⻆⾊验证所请求的资源访问权限，然后允许或拒绝所请求的服务。

任何⽤⼾都可以使⽤ openstack token issue 命令请求当前的有作⽤域令牌，其输出中显⽰⽤⼾ ID、项⽬和新的令牌到期时间。此令牌类型实际上是三种授权作⽤域类型中的⼀个：<mark>⽆作⽤域、项⽬作⽤域和域作⽤域。</mark>

## Fernet 密钥

OpenStack 中有许多类型的令牌提供程序，各⾃都⽐其前代有所进步。Fernet （发⾳为 fer'net）是最新的提供程序，解决了 UUID、PKI 和 PKIZ 提供程序中的缺陷，如令牌⼤⼩、安全性和性能问题。

令牌应定期更换，以最⼤程度地降低创建假冒 Fernet 令牌的可能性。Fernet 令牌提供程序使⽤⼀种轮转⽅法来启⽤新的对称密钥，⽽不破坏解密使⽤旧密钥创建的 Fernet 令牌的能⼒。

Fernet 采⽤数字形式，根据轮转中每个密钥的索引来命名其密钥，默认放在keystone的配置文件位置：/etc/keystone/fernet-keys，Fernet 在其轮转过程中使⽤三种类型的密钥：

1. 主密钥
   主密钥视为当前密钥。⼀个⾝份服务节点上只能有⼀个主密钥，<mark>主密钥⽂件名始终具有最⾼的索引编号。主密钥⽤于加密和解密 Fernet 令牌。</mark>

2. 辅助密钥
   辅助密钥是之前作为主密钥⽽已被替换（轮出）的密钥。<mark>辅助仅⽤于解密 Fernet 令牌</mark>；具体⽽⾔，解密原先⽤它加密的剩余 Fernet 令牌。辅助密钥的⽂件名称带有低于最⾼编号的索引，<mark>但索引值绝不会是 0</mark>

3. 暂存密钥
   <mark>暂存密钥是新添加的密钥，将在下⼀次密钥轮转时成为下⼀个主密钥，暂存密钥的⽂件名是0</mark>

## 轮转 Fernet 密钥

默认情况下，overcloud 的 Fernet 密钥管理由 Director 使⽤ Mistral 来执⾏。管理员应该使⽤ Mistral来启动 Fernet 密钥轮转。

```bash
[root@controller0 fernet-keys]# pwd
/var/lib/config-data/puppet-generated/keystone/etc/keystone/fernet-keys
[root@controller0 fernet-keys]# ls
0  1
[root@controller0 fernet-keys]# cat 0
BFdmfu__yYFapx1vxxw6B7tlYvmbMT-aWv6gdEZJd94=
[root@controller0 fernet-keys]# cat 1
p-g9V2EsPONhrSEkS3-x-gyGQGTTSqHSTTXP8ukwix8=
[root@controller0 fernet-keys]#
```

这里要注意，从官方复制粘贴，<mark>需要自己加单引号才行</mark>

```bash
(undercloud) [stack@director ~]$ source stackrc
(undercloud) [stack@director ~]$ openstack workflow execution create tripleo.fernet_keys.v1.rotate_fernet_keys '{"container": "overcloud"}'
+--------------------+-------------------------------------------+
| Field              | Value                                     |
+--------------------+-------------------------------------------+
| ID                 | c20542eb-462e-4b6f-a298-284f52aadbe8      |
| Workflow ID        | e4fbacec-a504-45e3-9d35-6268573c79d5      |
| Workflow name      | tripleo.fernet_keys.v1.rotate_fernet_keys |
| Workflow namespace |                                           |
| Description        |                                           |
| Task Execution ID  | <none>                                    |
| Root Execution ID  | <none>                                    |
| State              | RUNNING                                   |
| State info         | None                                      |
| Created at         | 2024-01-01 13:35:15                       |
| Updated at         | 2024-01-01 13:35:15                       |
+--------------------+-------------------------------------------+
```

轮转可能需要一会儿，你可以用下面的看看是否成功

```bash
(undercloud) [stack@director ~]$ openstack workflow execution show c20542eb-462e-4b6f-a298-284f52aadbe8
+--------------------+-------------------------------------------+
| Field              | Value                                     |
+--------------------+-------------------------------------------+
| ID                 | c20542eb-462e-4b6f-a298-284f52aadbe8      |
| Workflow ID        | e4fbacec-a504-45e3-9d35-6268573c79d5      |
| Workflow name      | tripleo.fernet_keys.v1.rotate_fernet_keys |
| Workflow namespace |                                           |
| Description        |                                           |
| Task Execution ID  | <none>                                    |
| Root Execution ID  | <none>                                    |
| State              | SUCCESS                                   |
| State info         | None                                      |
| Created at         | 2024-01-01 13:35:15                       |
| Updated at         | 2024-01-01 13:35:31                       |
+--------------------+-------------------------------------------+
```

看看默认密钥存在哪儿了

看上去我们的密钥后端程序是fernet，并且存在/etc/keystone/fernet-keys

```bash
[root@controller0 keystone]# pwd
/var/lib/config-data/puppet-generated/keystone/etc/keystone
[root@controller0 keystone]# crudini --get keystone.conf token provider
fernet
[root@controller0 keystone]# crudini --get keystone.conf catalog driver
sql
[root@controller0 keystone]# crudini --get keystone.conf fernet_tokens key_repository
/etc/keystone/fernet-keys
```

再去看看我们的密钥情况

```bash
(undercloud) [stack@director ~]$ ssh root@controller0
[root@controller0 ~]# cd /var/lib/config-data/puppet-generated/keystone/etc/keystone/fernet-keys/
[root@controller0 fernet-keys]# ls
0  1  2
[root@controller0 fernet-keys]# cat 0
JM-2KPqMR5qSB1bIyNh94Kltf_pUJpqHt0VXm7qYwSk=
[root@controller0 fernet-keys]# cat 1
p-g9V2EsPONhrSEkS3-x-gyGQGTTSqHSTTXP8ukwix8=
[root@controller0 fernet-keys]# cat 2
BFdmfu__yYFapx1vxxw6B7tlYvmbMT-aWv6gdEZJd94=
```

设定过期时间的间隔

看上去我的间隔是3600秒

```bash
[root@controller0 fernet-keys]# crudini --get ../keystone.conf token expiration
3600
```

设定最多有几个密钥

```bash
[root@controller0 fernet-keys]# crudini --get ../keystone.conf fernet_tokens max_active_keys
5
```

# 管理项⽬组织

## 管理域和项⽬

随着⾝份服务 v3 中域的引⼊，<mark>⽤⼾不再包含在项⽬中，⽽是包含在域中</mark>。借助此设计更改，⽤⼾现在可以从属于多个项⽬

让我们重温一下域和项目之类的关系

```text
                   +--------------------+
                   |    OpenStack       |
                   +--------------------+
                               |
          +-----------------------------------------+
          |                                         |
  +------------------+                      +------------------+
  |      Domain 1    |                      |      Domain 2    |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |     Project 1    |                      |     Project 3    |
  +------------------+                      +------------------+
  |     Project 2    |                      |     Project 4    |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |     Group A      |                      |     Group C      |
  +------------------+                      +------------------+
  |     Group B      |                      |     Group D      |
  +------------------+                      +------------------+
          |                                         |
  +------------------+                      +------------------+
  |      User A      |                      |      User C      |
  +------------------+                      +------------------+
  |      User B      |                      |      User D      |
  +------------------+                      +------------------+
```

### 创建域

```bash
(overcloud) [stack@director ~]$ openstack domain create lxh-domain
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| enabled     | True                             |
| id          | 750c06bbb1ca4179a41fc866eb48d13c |
| name        | lxh-domain                       |
| options     | {}                               |
| tags        | []                               |
+-------------+----------------------------------+
```

### 创建普通项目

在lxh-domain中创建一个项目

```bash
(overcloud) [stack@director ~]$ openstack project create lxh-project1 --domain lxh-domain
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | 750c06bbb1ca4179a41fc866eb48d13c |
| enabled     | True                             |
| id          | 733e55768ff44a7ab5d5dd8368cc365a |
| is_domain   | False                            |
| name        | lxh-project1                     |
| options     | {}                               |
| parent_id   | 750c06bbb1ca4179a41fc866eb48d13c |
| tags        | []                               |
+-------------+----------------------------------+
```

### 创建嵌套项目

一个project相当于一个集团公司的事业群，这是一个超级大部门，在这个超级大部门中，会存在很多小的子公司或者小部门，此时可以用嵌套项目来完成

在lxh-project1中，创建一个lxh-sub-project1

```bash
(overcloud) [stack@director ~]$ openstack project create lxh-sub-project1 --parent lxh-project1 --domain lxh-domain
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | 750c06bbb1ca4179a41fc866eb48d13c |
| enabled     | True                             |
| id          | d7ef9bad79964f1a8c4143cb5d71ef45 |
| is_domain   | False                            |
| name        | lxh-sub-project1                 |
| options     | {}                               |
| parent_id   | 733e55768ff44a7ab5d5dd8368cc365a |
| tags        | []                               |
+-------------+----------------------------------+
```

### 在项目中创建用户

```bash
(overcloud) [stack@director ~]$ openstack user create --project lxh-sub-project1 --domain lxh-domain lxhuser --password lixiaohui
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| default_project_id  | d7ef9bad79964f1a8c4143cb5d71ef45 |
| domain_id           | 750c06bbb1ca4179a41fc866eb48d13c |
| enabled             | True                             |
| id                  | 21462c6e77e54b52ae115200d1587e80 |
| name                | lxhuser                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
(overcloud) [stack@director ~]$ openstack user list --domain lxh-domain
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 21462c6e77e54b52ae115200d1587e80 | lxhuser |
+----------------------------------+---------+
```

## 管理角色和权限

在 RHOSP 环境中，⽤⼾必须使⽤项⽬范围进⾏⾝份验证。因此，必须授予每个⽤⼾⾄少⼀个项⽬的访问权限。⾝份服务使⽤基于⻆⾊的访问控制 (RBAC)。管理员可以通过将⻆⾊分配给⽤⼾或组，
授予其对域和项⽬的访问权限。

### 授予用户管理员角色

```bash
(overcloud) [stack@director ~]$ openstack role add --user lxhuser --user-domain lxh-domain --project lxh-project1 --project-domain lxh-domain admin
```

```bash
(overcloud) [stack@director ~]$ openstack role assignment list --user lxhuser --user-domain lxh-domain --names
+-------+---------------------+-------+-------------------------+------------+--------+-----------+
| Role  | User                | Group | Project                 | Domain     | System | Inherited |
+-------+---------------------+-------+-------------------------+------------+--------+-----------+
| admin | lxhuser@lxh-domain |       | lxh-project1@lxh-domain |            |        | False     |
```

### 角色继承

启用了角色继承，组 lxhgroup 在 lxh-domain 域中获得的 admin 角色及其权限将自动继承到该域下的所有项目中。组成员 lxhuser 因此在所有项目中均具有 admin 权限，而不需要为每个项目单独添加。

```bash
(overcloud) [stack@director ~]$ openstack group create lxhgroup --domain lxh-domain
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description |                                  |
| domain_id   | 750c06bbb1ca4179a41fc866eb48d13c |
| id          | b0e3aa43023d47bba6ae40e88892d73d |
| name        | lxhgroup                         |
+-------------+----------------------------------+

(overcloud) [stack@director ~]$ openstack group add user --group-domain lxh-domain --user-domain lxh-domain lxhgroup lxhuser
```

看看人加进去没有

```bash
(overcloud) [stack@director ~]$ openstack group contains user lxhgroup lxhuser --group-domain lxh-domain --user-domain lxh-domain
lxhuser in group lxhgroup
```

我们为组 lxhgroup 在 lxh-domain 域中赋予 admin 角色，并使其继承到该域下的所有项目

```bash
(overcloud) [stack@director ~]$ openstack role add --group lxhgroup --group-domain lxh-domain --project lxh-project1 --project-domain lxh-domain --inherited admin

(overcloud) [stack@director ~]$ openstack role assignment list --group lxhgroup --group-domain lxh-domain --names
+-------+------+---------------------+-------------------------+------------+--------+-----------+
| Role  | User | Group               | Project                 | Domain     | System | Inherited |
+-------+------+---------------------+-------------------------+------------+--------+-----------+
| admin |      | lxhgroup@lxh-domain | lxh-project1@lxh-domain |            |        | True      |
```

# ⾃定义⽤⼾⻆⾊

除了标准的member和admin之类的，还可以自己创建角色，并关联到用户

自定义的角色没有权限，所以需要自己写策略，策略的规则部分可能更为复杂，并⽀持布尔运算符。此⽰例显⽰了多个条件的连接词，比如and 和 or

例子：

```text
"<target>": "role:reader and system_scope:all"
"<target>": "is_admin:True or project_id:%(project_id)s"
```

我们创建一个lxh-role的角色，然后再创建一个lxhuser2，并使其关联我们的角色，然后我们制定一些策略到我们的角色上，看看有没有效果

**创建角色和用户**

```bash
(overcloud) [stack@director ~]$ openstack role create role1
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | bb2314902df4413bbf4bf55aa55cbb5a |
| name        | role1                            |
| options     | {}                               |
+-------------+----------------------------------+
(overcloud) [stack@director ~]$ openstack user create user1 --domain lxh-domain --password lxhpassword
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | 3cbf4e2f874b4662b689b69480362ae7 |
| enabled             | True                             |
| id                  | 89bbca2e8b3f43638182a8976ddab838 |
| name                | user1                            |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

**关联角色到用户**

```bash
(overcloud) [stack@director ~]$ openstack role add --user user1 --user-domain lxh-domain --project lxh-project1 --project-domain lxh-domain role1
(overcloud) [stack@director ~]$ openstack role assignment list --user user1 --user-domain lxh-domain --names
+-------+------------------+-------+-------------------------+--------+--------+-----------+
| Role  | User             | Group | Project                 | Domain | System | Inherited |
+-------+------------------+-------+-------------------------+--------+--------+-----------+
| role1 | user1@lxh-domain |       | lxh-project1@lxh-domain |        |        | False     |
+-------+------------------+-------+-------------------------+--------+--------+-----------+
```

此时角色是空的，什么权限都没有，我们用这个角色来获取一下主机聚合

用超级用户可以获取

```bash
(overcloud) [stack@director ~]$ source overcloudrc
(overcloud) [stack@director ~]$ openstack hypervisor list
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
| ID                                   | Hypervisor Hostname               | Hypervisor Type | Host IP     | State |
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
| 4ae32d90-761d-4920-8b20-598f21c9d12e | compute1.overcloud.example.com    | QEMU            | 172.24.1.12 | up    |
| a54675fc-0ba3-4872-8b76-4fb094b41d30 | computehci0.overcloud.example.com | QEMU            | 172.24.1.6  | up    |
| b506a957-3a44-4db6-b4bb-93e786b26356 | compute0.overcloud.example.com    | QEMU            | 172.24.1.2  | up    |
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
```

用我们自己的账号无法获取

先制作一个自己的rc文件，然后再get

```bash
(overcloud) [stack@director ~]$ cp overcloudrc user1
(overcloud) [stack@director ~]$ vim user1
# Clear any old environment that may conflict.
for key in $( set | awk '{FS="="}  /^OS_/ {print $1}' ); do unset $key ; done
export NOVA_VERSION=1.1
export COMPUTE_API_VERSION=1.1
export OS_USERNAME=user1
export OS_PROJECT_NAME=lxh-project1
export OS_USER_DOMAIN_NAME=lxh-domain
export OS_PROJECT_DOMAIN_NAME=lxh-domain
export OS_NO_CACHE=True
export OS_CLOUDNAME=overcloud
export no_proxy=,172.25.249.50,172.25.250.50
export PYTHONWARNINGS='ignore:Certificate has no, ignore:A true SSLContext object is not available'
export OS_AUTH_TYPE=password
export OS_PASSWORD=lxhpassword
export OS_AUTH_URL=http://172.25.250.50:5000
export OS_IDENTITY_API_VERSION=3
export OS_COMPUTE_API_VERSION=2.latest
export OS_IMAGE_API_VERSION=2
export OS_VOLUME_API_VERSION=3
export OS_REGION_NAME=regionOne

# Add OS_CLOUDNAME to PS1
if [ -z "${CLOUDPROMPT_ENABLED:-}" ]; then
    export PS1=${PS1:-""}
    export PS1=\${OS_CLOUDNAME:+"(\$OS_CLOUDNAME)"}\ $PS1
    export CLOUDPROMPT_ENABLED=1
fi
```

可以看到，我们没什么权限

```bash
(overcloud) [stack@director ~]$ source user1
(overcloud) [stack@director ~]$ openstack user list
You are not authorized to perform the requested action: identity:list_users. (HTTP 403) (Request-ID: req-0c02ed30-5d0b-4ce4-8338-d8b55454d5c2)
(overcloud) [stack@director ~]$ openstack hypervisor list
Policy doesn't allow os_compute_api:os-hypervisors to be performed. (HTTP 403) (Request-ID: req-c441468a-18d0-4eb9-9c8a-f2beb08d568b)
```

返回到管理员，分配策略到角色

先登录nova获取一些nova中的聚合主机策略怎么写，发现他需要admin_api的规则

```bash
[root@controller0 ~]# podman exec -it nova_api /bin/bash
()[root@controller0 /]# oslopolicy-policy-generator --namespace nova > nova.txt
()[root@controller0 /]# grep -i hyper nova.txt
"os_compute_api:os-hypervisors": "rule:admin_api"
```

再登录keystone看看用户列表策略怎么写，只能具有reader角色的才行

```bash
[root@controller0 ~]# podman exec -it keystone /bin/bash
()[root@controller0 /]# oslopolicy-policy-generator --namespace keystone > keystone.txt
()[root@controller0 /]# grep -i list_users keystone.txt
"identity:list_users_in_group": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.group.domain_id)s)"
"identity:list_users": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s)"
```

根据我们查到的内容，制作自己的nova和keystone策略，策略要放到具体服务的配置文件下，名称为policy.json

nova的部分，我们同时关联了我们的角色，创建了文件，可以验证一下，避免出问题

```bash
[root@controller0 ~]# cd /var/lib/config-data/puppet-generated/
[root@controller0 puppet-generated]# cd nova/etc/nova/

[root@controller0 nova]# cat > policy.json <<-'EOF'
{
"os_compute_api:os-hypervisors": "rule:admin_api or role:role1"
}
EOF

[root@controller0 nova]# cat policy.json
{
"os_compute_api:os-hypervisors": "rule:admin_api or role:role1"
}

[root@controller0 nova]# cat policy.json | python -m json.tool
{
    "os_compute_api:os-hypervisors": "rule:admin_api or role:role1"
}
```

keystone的部分，我们同时关联了我们的角色，创建了文件，可以验证一下，避免出问题

```bash
[root@controller0 ~]# cd /var/lib/config-data/puppet-generated/
[root@controller0 puppet-generated]# cd keystone/etc/keystone/
[root@controller0 keystone]# ls
credential-keys  domains  domains-prebuild.bak  fernet-keys  keystone.conf

[root@controller0 keystone]# cat > policy.json <<-'EOF'
{
"identity:list_users": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s) or role:role1"
}
EOF

[root@controller0 keystone]# cat policy.json
{
"identity:list_users": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s) or role:role1"
}

[root@controller0 keystone]# cat policy.json | python -m json.tool
{
    "identity:list_users": "(role:reader and system_scope:all) or (role:reader and domain_id:%(target.domain_id)s) or role:role1"
}
```

重启容器生效

```bash
[root@controller0 ~]# systemctl restart tripleo_keystone
[root@controller0 ~]# systemctl restart tripleo_nova_api
```

再次回到director看看是否有了权限

```bash
(overcloud) [stack@director ~]$ source user1
(overcloud) [stack@director ~]$ openstack user list
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 89bbca2e8b3f43638182a8976ddab838 | user1 |
+----------------------------------+-------+
(overcloud) [stack@director ~]$ openstack hypervisor list
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
| ID                                   | Hypervisor Hostname               | Hypervisor Type | Host IP     | State |
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
| 4ae32d90-761d-4920-8b20-598f21c9d12e | compute1.overcloud.example.com    | QEMU            | 172.24.1.12 | up    |
| a54675fc-0ba3-4872-8b76-4fb094b41d30 | computehci0.overcloud.example.com | QEMU            | 172.24.1.6  | up    |
| b506a957-3a44-4db6-b4bb-93e786b26356 | compute0.overcloud.example.com    | QEMU            | 172.24.1.2  | up    |
+--------------------------------------+-----------------------------------+-----------------+-------------+-------+
```
