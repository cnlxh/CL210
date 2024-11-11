```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述配置 TLS-e 所需的流程。
- 部署和配置 AIDE。

# 管理端到端安全服务

## 端到端安全服务

红帽 OpenStack 平台上的 TLS-everywhere (TLS-e) 使⽤ TLS/SSL 为内部和外部端点提供安全传输。红帽建议的⽅法使⽤基于 Ansible 的 tripleo-ipa ⽅法。 ansible-tripleo-ipa 软件包已作为依赖项安装，提供相应的 playbook 来⽤于配置 IdM 和部署 TLS-e。提供的 playbook 位于 /usr/share/ansible/tripleo-playbooks ⽬录中。

TLS-e 利⽤红帽 IdM 服务器提供证书颁发机构服务，并管理实施所需的许多证书。

<mark>由于这个TLS涉及到以幂等方式重装undercloud和overcloud集群，此处不做过多阐述，感兴趣可以看看红帽官方文档</mark>

# 使⽤ AIDE 管理基于⽂件的组件安全性

## ⾼级⼊侵检测环境介绍

⾼级⼊侵检测环境 （Advanced Intrusion Detection Environment，AIDE） <mark>是⼀款旨在⽐较并报告⽂件和⽬录内更改的⼯具。第⼀次运⾏时，会提取散列并存储在本地数据库中，以后每次运⾏时进⾏⽐较。检测到差别时将创建⼀个⽇志条⽬，也可配置为执⾏脚本或发送电⼦邮件。</mark>

除了在部署集群时执行AIDE之外，也可以手工执行

## 安装AIDE

```bash
(undercloud) [stack@director ~]$ sudo -i
 [root@director ~]#
 [root@director ~]# yum install aide -y
```

## 初始化AIDE数据库

先看看aide的参数

```bash
 [root@director ~]# aide -h
Aide 0.16

Usage: aide [options] command

Commands:
  -i, --init            Initialize the database
  -C, --check           Check the database
  -u, --update          Check and update the database non-interactively
  -E, --compare         Compare two databases

Miscellaneous:
  -D, --config-check    Test the configuration file
  -v, --version         Show version of AIDE and compilation options
  -h, --help            Show this help message

Options:
  -c [cfgfile]  --config=[cfgfile]      Get config options from [cfgfile]
  -l [REGEX]    --limit=[REGEX]         Limit command to entries matching [REGEX]
  -B "OPTION"   --before="OPTION"       Before configuration file is read define OPTION
  -A "OPTION"   --after="OPTION"        After configuration file is read define OPTION
  -r [reporter] --report=[reporter]     Write report output to [reporter] url
  -V[level]     --verbose=[level]       Set debug message level to [level]

 [root@director ~]# ls /var/lib/aide/
```

初始化一下数据库

```bash
 [root@director ~]# aide --init
Start timestamp: 2023-12-31 11:25:33 -0500 (AIDE 0.16)
AIDE initialized database at /var/lib/aide/aide.db.new.gz

Number of entries:      144181

---------------------------------------------------
The attributes of the (uncompressed) database(s):
---------------------------------------------------

/var/lib/aide/aide.db.new.gz
  MD5      : N3jo/T6T68sdRudTB4XUaA==
  SHA1     : YV7ltgxMp+DzwotyGU1qSKhSRL8=
  RMD160   : 98Du16QkCtcM++4jDq4wkluYO+A=
  TIGER    : IFzLXzFPjGnSwpFDNtUnjS/LXg0Jzpuh
  SHA256   : pMKZF0q6SFTc7pKyvQRGJ3kb9xwTvUen
             9TSnqwO3tJs=
  SHA512   : wjnPYyZnPXjMW0UDqjkLeYjV3iOlhGLO
             on1MrRdPjuBBNw1zOQnRwdtBlJIVrb3H
             wtGIbWJaQxgLfC66gfAVbA==


End timestamp: 2023-12-31 11:26:16 -0500 (run time: 0m 43s)
```

需要注意的是，<mark>默认aide生成的数据库和检测的数据库名称不一致，需要改名才行</mark>，这也是为了防止新生成的覆盖掉

```bash
 [root@director ~]# aide --check
Couldn't open file /var/lib/aide/aide.db.gz for reading
 [root@director ~]# mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

### 强制 AIDE 执⾏检查

```bash
 [root@director ~]# aide --check
Start timestamp: 2023-12-31 11:28:35 -0500 (AIDE 0.16)
AIDE found NO differences between database and filesystem. Looks okay!!
```

### 测试文件修改和新增

新增一个文件，并且修改一个文件，看看它的对比结果

```bash
 [root@director ~]# aide --check
Start timestamp: 2023-12-31 11:35:39 -0500 (AIDE 0.16)
AIDE found differences between database and filesystem!!

Summary:
  Total number of entries:      144181
  Added entries:                0
  Removed entries:              0
  Changed entries:              2

---------------------------------------------------
Changed entries:
---------------------------------------------------

f = .u.    . ... : /var/log/containers/ironic-inspector/dnsmasq.log
f = ...    . A.. : /var/log/journal/f874df04639f474cb0a9881041f4f7d4/system.journal

---------------------------------------------------
Detailed information about changes:
---------------------------------------------------

File: /var/log/containers/ironic-inspector/dnsmasq.log
  Uid      : 42461                            | 995

File: /var/log/journal/f874df04639f474cb0a9881041f4f7d4/system.journal
  ACL      : A: user::rw-                     | A: user::rw-
             A: group::r-x      #effective:r--     | A: group::r-x      #effective:r--
             A: group:adm:r-x   #effective:r--  | A: group:adm:r--
             A: group:wheel:r-x #effective:r- | A: group:wheel:r--
             -                                |
             A: mask::r--                     | A: mask::r--
             A: other::---                    | A: other::---


---------------------------------------------------
The attributes of the (uncompressed) database(s):
---------------------------------------------------

/var/lib/aide/aide.db.gz
  MD5      : N3jo/T6T68sdRudTB4XUaA==
  SHA1     : YV7ltgxMp+DzwotyGU1qSKhSRL8=
  RMD160   : 98Du16QkCtcM++4jDq4wkluYO+A=
  TIGER    : IFzLXzFPjGnSwpFDNtUnjS/LXg0Jzpuh
  SHA256   : pMKZF0q6SFTc7pKyvQRGJ3kb9xwTvUen
             9TSnqwO3tJs=
  SHA512   : wjnPYyZnPXjMW0UDqjkLeYjV3iOlhGLO
             on1MrRdPjuBBNw1zOQnRwdtBlJIVrb3H
             wtGIbWJaQxgLfC66gfAVbA==


End timestamp: 2023-12-31 11:36:19 -0500 (run time: 0m 40s)
```

### 更新数据库

如果确认本次修改是应该的，需要将当前状态更新到数据库

```bash
 [root@director ~]# aide --update
Start timestamp: 2023-12-31 11:40:10 -0500 (AIDE 0.16)
AIDE found differences between database and filesystem!!
New AIDE database written to /var/lib/aide/aide.db.new.gz

Summary:
  Total number of entries:      144181
  Added entries:                0
  Removed entries:              0
  Changed entries:              2

---------------------------------------------------
Changed entries:
---------------------------------------------------

f = .u.    . ... : /var/log/containers/ironic-inspector/dnsmasq.log
f = ...    . A.. : /var/log/journal/f874df04639f474cb0a9881041f4f7d4/system.journal

---------------------------------------------------
Detailed information about changes:
---------------------------------------------------

File: /var/log/containers/ironic-inspector/dnsmasq.log
  Uid      : 42461                            | 995

File: /var/log/journal/f874df04639f474cb0a9881041f4f7d4/system.journal
  ACL      : A: user::rw-                     | A: user::rw-
             A: group::r-x      #effective:r--     | A: group::r-x      #effective:r--
             A: group:adm:r-x   #effective:r--  | A: group:adm:r--
             A: group:wheel:r-x #effective:r- | A: group:wheel:r--
             -                                |
             A: mask::r--                     | A: mask::r--
             A: other::---                    | A: other::---


---------------------------------------------------
The attributes of the (uncompressed) database(s):
---------------------------------------------------

/var/lib/aide/aide.db.gz
  MD5      : N3jo/T6T68sdRudTB4XUaA==
  SHA1     : YV7ltgxMp+DzwotyGU1qSKhSRL8=
  RMD160   : 98Du16QkCtcM++4jDq4wkluYO+A=
  TIGER    : IFzLXzFPjGnSwpFDNtUnjS/LXg0Jzpuh
  SHA256   : pMKZF0q6SFTc7pKyvQRGJ3kb9xwTvUen
             9TSnqwO3tJs=
  SHA512   : wjnPYyZnPXjMW0UDqjkLeYjV3iOlhGLO
             on1MrRdPjuBBNw1zOQnRwdtBlJIVrb3H
             wtGIbWJaQxgLfC66gfAVbA==

/var/lib/aide/aide.db.new.gz
  MD5      : AAGejLjdpBczmchBp8F7Kg==
  SHA1     : XSIuLwPV+oEB/IMu0TgnlHFCV/A=
  RMD160   : Zx3FzhtoIPACjPqITg7ydAl9zZs=
  TIGER    : uAWV1mr+FRWmnfqUGG5/uh7xVnOuxP99
  SHA256   : J7vYv+vO4qSpz2AFo+7o8cub4gequJqc
             qVlKeKHBJ5E=
  SHA512   : Qv1mBiUeyTJFlKn/K/H/FNg+FxFOa3N/
             VXdFD4hTUAM/GRouUSgdcQn3KCCbIP0I
             +0z18+44eSicC6rZtv7bzw==


End timestamp: 2023-12-31 11:40:52 -0500 (run time: 0m 42s)
```

```bash
 [root@director ~]# mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
mv: overwrite '/var/lib/aide/aide.db.gz'? y
```
