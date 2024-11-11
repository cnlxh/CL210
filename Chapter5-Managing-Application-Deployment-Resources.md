```text
作者：李晓辉

联系方式：

1. 微信：Lxh_Chat

2. 邮箱：939958092@qq.com 
```

# 课程目标

- 描述红帽 OpenStack 平台中使⽤的常⻅镜像格式。
- 使⽤ diskimage-builder 构建镜像。
- 使⽤ Virt-customize ⾃定义镜像。
- 使⽤ Cloud-init 在部署期间⾃定义启动的实例。

# ⽐较镜像格式

## 常⻅镜像格式

| 格式    | 描述                                                                                                   |
| ----- | ---------------------------------------------------------------------------------------------------- |
| raw   | 一种非结构化磁盘映像格式，它是原始磁盘的精确副本。这些映像具有扩展名，例如 、 或 .`.img``.raw``.bin`                                        |
| QCOW2 | QEMU Copy-On-Write v2 格式。QEMU 仿真器支持此格式，并且可以动态扩展空间。这是 Linux KVM 最常用的格式。                               |
| ISO   | CD 或 DVD 光盘数据内容的存档格式。                                                                                |
| AKI   | Amazon 内核映像。通常不与 Linux KVM 或 Red Hat OpenStack Platform 一起使用。                                        |
| AMI   | Amazon 系统映像。通常不与 Linux KVM 或 Red Hat OpenStack Platform 一起使用。                                        |
| ARI   | Amazon ramdisk 映像。通常不与 Linux KVM 或 Red Hat OpenStack Platform 一起使用。                                  |
| VDI   | 一个 VirtualBox 磁盘映像，最初用于 VirtualBox 托管的虚拟机管理程序。由 QEMU 仿真器支持。                                          |
| VHD   | 虚拟硬盘格式，由 Microsoft 的 Hyper-V 使用，最初由 Windows Virtual PC 使用。VMware、Xen、VirtualBox 等也支持这种格式，但不是普遍流行的格式。 |
| VMDK  | 虚拟机磁盘格式，最初是为 VMware 创建的，但现在是一种普遍支持的常见开放格式。                                                           |

RAW和QCOW2是最常见的格式，以下是一些主要区别：

**RAW 和 QCOW2 图像格式的比较**

| 属性    | 生                                                                                        | QCOW2                                                                                                                                    |
| ----- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 映像大小  | RAW 图像是源磁盘的位等效副本，<mark>包括空白空间</mark>。在支持稀疏存储的文件系统或后端中，RAW 图像使用的实际空间通常比它们报告的空间少。          | QCOW2 是虚拟磁盘映像的空间优化表示形式。不要与压缩或稀疏存储混淆，优化是文件结构内部的。通常，<mark>QCOW2 文件比来自同一源的 RAW 映像小</mark>。QCOW2 还支持zlib压缩。QCOW2 文件已经过优化，因此可能不使用稀疏文件系统或存储功能。 |
| 性能    | <mark>RAW 格式性能高于 QCOW2 性能</mark>，因为磁盘空间是在实例部署时分配的，从而避免了与按需空间分配相关的延迟。RAW 图像在使用过程中不需要额外处理。 | <mark>QCOW2 格式性能低于 RAW 格式性能</mark>，因为图像特征需要动态处理。当请求的空间超过当前优化分配时，在扩展空间时会产生延迟。                                                             |
| 加密    | 不适⽤                                                                                      | 可选加密。 使用 256 位 AES （CRC） 密码或 LUKS 加密。                                                                                                    |
| 快照    | 不适⽤                                                                                      | QCOW2 支持多个快照，每个快照都是映像在某个时间点的只读记录。QCOW2 快照存储在内部。                                                                                          |
| 写入时复制 | 不适⽤                                                                                      | QCOW2 文件更改将写入单独的叠加文件，保留原始映像文件，并在修改之前将要更改的块复制到叠加层。多个叠加文件可以共享同一个基础映像。使用多个文件允许从同一个基础映像运行多个实例。                                               |

## 创建并了解qcow2镜像文件

可以看到，虚拟大小为1G，实际大小只有193k

```bash
(undercloud) [stack@director ~]$ qemu-img create -f qcow2 test.qcow2 1G
Formatting 'test.qcow2', fmt=qcow2 size=1073741824 cluster_size=65536 lazy_refcounts=off refcount_bits=16

(undercloud) [stack@director ~]$ ls -lh test.qcow2
-rw-r--r--. 1 stack stack 193K Jan  1 11:43 test.qcow2

(undercloud) [stack@director ~]$ qemu-img info test.qcow2
image: test.qcow2
file format: qcow2
virtual size: 1 GiB (1073741824 bytes)
disk size: 196 KiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

## 创建并了解raw镜像文件

可以看到虚拟大小和实际大小都是1G

```bash
(undercloud) [stack@director ~]$ qemu-img create -f raw test.raw 1G
Formatting 'test.raw', fmt=raw size=1073741824
(undercloud) [stack@director ~]$
(undercloud) [stack@director ~]$ ls -lh test.raw
-rw-r--r--. 1 stack stack 1.0G Jan  1 11:45 test.raw
(undercloud) [stack@director ~]$ qemu-img info test.raw
image: test.raw
file format: raw
virtual size: 1 GiB (1073741824 bytes)
disk size: 4 KiB
```

## 镜像文件的格式转换

```bash
(undercloud) [stack@director ~]$ qemu-img convert -f qcow2 test.qcow2 -O raw test.qcow2 test-convert.raw
(undercloud) [stack@director ~]$ qemu-img info test-convert.raw
image: test-convert.raw
file format: raw
virtual size: 2 GiB (2147483648 bytes)
disk size: 4 KiB
(undercloud) [stack@director ~]$ ls -lh test-convert.raw
-rw-r--r--. 1 stack stack 2.0G Jan  1 11:48 test-convert.raw
```

# 构建⾃定义镜像

您可以使⽤以下三种⽅法中的任何⼀种来⽣成⾃定义镜像：

1. diskimage-builder

2. Guestfish 或 Virt-customize

3. Cloud-init

## Diskimage-builder

diskimage-builder 在 chroot 环境中绑定 /proc、/sys 和 /dev 挂载。镜像构建过程⽣成最⼩的系统，它们包含实现其 OpenStack 相关⽬的需要的所有组件。镜像可以是⽂件系统镜像那样的简单镜像，也可以经过⾃定义提供完整的磁盘镜像。

元素⽤于指定放⼊镜像的内容以及所需的任何修改。镜像必须⾄少使⽤⼀种基础分发元素，给定的分发具有多个元素。例如，分发元素可以是 rhel，其他元素则⽤于修改 rhel 基础镜像。基于多个元素调⽤脚本并应⽤到镜像。

每⼀元素可以使⽤ element-deps 和 element-provides 定义或影响依赖项。

1. **定义依赖**：`element-deps` 文件是一个纯文本文件，其中列出了所有该元素所依赖的其他元素。在构建镜像时，Diskimage-builder 会首先构建这些依赖元素，然后再构建该元素本身。例如，如果一个元素依赖于 `ubuntu` 元素，那么 `ubuntu` 元素会在构建该元素之前被构建

2. **定义提供的功能**：`element-provides` 文件是另一个纯文本文件，其中列出了该元素提供的功能或服务。这意味着如果某个元素已经提供了某些功能，那么在构建镜像时，Diskimage-builder 不会再次构建这些功能，从而避免重复工作。

安装diskimage-builder

```bash
[root@director ~]# yum install diskimage-builder
```

列出包含的元素

```bash
[root@director ~]# ls /usr/share/diskimage-builder/elements
apt-conf                     debian-minimal       dracut-ramdisk         install-static        pip-and-virtualenv         simple-init
apt-preferences              debian-systemd       dracut-regenerate      install-types         pip-cache                  source-repositories
apt-sources                  debian-upstart       dynamic-login          ironic-agent          pkg-map                    stable-interface-names
baremetal                    debootstrap          element-manifest       iscsi-boot            posix                      svc-map
base                         deploy-baremetal     enable-serial-console  iso                   proliant-tools             sysctl
block-device-efi             deploy-kexec         ensure-venv            journal-to-console    pypi                       sysprep
block-device-gpt             deploy-targetcli     epel                   local-config          python-brickclient         uboot
block-device-mbr             deploy-tgtadm        fedora                 lvm                   python-stow-versions       ubuntu
bootloader                   devuser              fedora-minimal         manifests             ramdisk                    ubuntu-common
cache-url                    dhcp-all-interfaces  gentoo                 mellanox              ramdisk-base               ubuntu-minimal
centos                       dib-init-system      growroot               modprobe              rax-nova-agent             ubuntu-signed
centos7                      dib-python           grub2                  modprobe-blacklist    redhat-common              ubuntu-systemd-container
centos-minimal               dib-run-parts        hpdsa                  no-final-image        rhel                       vm
cleanup-kernel-initrd        disable-nouveau      hwburnin               oat-client            rhel7                      yum
cloud-init                   disable-selinux      hwdiscovery            openssh-server        rhel-common                yum-minimal
cloud-init-datasources       dkms                 ibft-interfaces        openstack-ci-mirrors  rpm-distro                 zipl
cloud-init-disable-resizefs  docker               ilo                    opensuse              runtime-ssh-host-keys      zypper
cloud-init-nocloud           dpkg                 __init__.py            opensuse-minimal      select-boot-kernel-initrd  zypper-minimal
debian                       dracut-network       install-bin            package-installs      selinux-permissive
```

### 基础元素

/usr/share/diskimage-builder/elements/base 是 Diskimage-builder 中的一个重要目录，它包含了基础元素（base element）的定义和脚本。基础元素通常包括一些最低限度的配置和软件包，是构建其他自定义镜像的基础。

### Diskimage-builder 阶段⼦⽬录说明

diskimage-builder 镜像偶⻅过程分为⼏个阶段。阶段⼦⽬录位于 element ⽬录下；它们不⼀定默认存在，所以请根据需要创建。它们包含具有两位数字前缀并按数字顺序执⾏的脚本。其约定
是将数据⽂件存储在 element ⽬录中，但仅将可执⾏脚本存储在阶段⼦⽬录中。如果某个脚本不可执⾏，则它不会运⾏。阶段⼦⽬录按照下表中列出的顺序处理：

- **root.d**：
  
  - **作用**：包含在 `chroot` 环境中执行的脚本。这些脚本在构建过程中会在目标文件系统的根目录中运行，通常用于安装和配置基本软件包和设置，所以你有什么自定义的项目需要添加，可以放这里。
  
  - **执行顺序**：相对较早执行，以便确保基础环境的配置。

- **extra-data.d**：
  
  - **作用**：包含下载和处理额外数据的脚本。这些数据可以是镜像中需要包含的特定文件或配置。
  
  - **执行顺序**：通常在安装和配置的中期执行，以确保额外的数据在需要时可用。

- **pre-install.d**：
  
  - **作用**：在安装任何软件包之前运行的脚本。这些脚本通常用于准备系统环境，例如更新包列表或设置环境变量。
  
  - **执行顺序**：在 `install.d` 之前。

- **install.d**：
  
  - **作用**：执行安装操作的脚本，用于安装必要的软件包和工具。
  
  - **执行顺序**：在 `pre-install.d` 之后，`post-install.d` 之前。

- **post-install.d**：
  
  - **作用**：在软件包安装完成后运行的脚本，用于配置已安装的软件或执行其他需要在安装后进行的操作。
  
  - **执行顺序**：在 `install.d` 之后

- **block-device.d**：
  
  - **作用**：包含用于配置块设备（如磁盘分区、文件系统等）的脚本。通常用于准备磁盘分区和文件系统结构。
  
  - **执行顺序**：根据需要，通常在配置阶段或安装阶段之后。

- **finalize.d**：
  
  - **作用**：执行最终定制操作的脚本，用于做出最后的调整和优化，例如设置引导加载程序。
  
  - **执行顺序**：在所有配置和安装操作完成后执行。

- **cleanup.d**：
  
  - **作用**：在所有操作完成后运行的脚本，用于清理构建过程中产生的临时文件和数据，以减小镜像大小并优化性能。
  
  - **执行顺序**：最后执行，以确保所有的临时文件和数据被清除。

### 基本的Diskimage-builder 变量

| 变量                  | 描述                                   |
| ------------------- | ------------------------------------ |
| `DIB_LOCAL_IMAGE`   | 要从中构建的基础映像                           |
| `DIB_YUM_REPO_CONF` | 在映像构建期间要复制到chroot环境中的客户端 Yum 存储库配置文件 |
| `ELEMENTS_PATH`     | 元素的工作副本的路径                           |

### Diskimage-builder 选项

```bash
[root@director ~]# disk-image-create vm rhel -n -p python-django-compressor -a amd64 -o web.img 2>&1 | tee diskimage-build.log
```

论点    描述
vm    定义要使用的元素。 为虚拟机磁盘映像提供合理的默认值。
rhel    指定映像将为 Red Hat Enterprise Linux。
-n    跳过 base 元素的默认包含，如果您不想安装 cloud-init 和软件包更新，这可能是可取的。
-p    指定要安装的软件包;在此示例中，安装了 python-django-compressor 包。
-a    指定映像的体系结构。
-o    指定输出图像名称。

| 参数     | 描述                                                 |
| ------ | -------------------------------------------------- |
| `vm`   | 定义要使用的元素。 为虚拟机磁盘映像提供合理的默认值。                        |
| `rhel` | 指定映像将为 Red Hat Enterprise Linux。                   |
| `-n`   | 跳过 base 元素的默认包含，如果您不想安装 cloud-init 和软件包更新，这可能是可取的。 |
| `-p`   | 指定要安装的软件包;在此示例中，安装了 python-django-compressor 包。    |
| `-a`   | 指定映像的体系结构。                                         |
| `-o`   | 指定输出映像名称。                                          |

### 构建案例

我们基于osp-small.qcow2来构建新的镜像
先下载一下这个镜像

```bash
cd
wget http://materials/osp-small.qcow2
```

准备基础元素目录和定制目录

```bash
cp -a /usr/share/diskimage-builder/elements/ /root/
mkdir -p elements/rhel/post-install.d
```

定制在rhel中安装并配置httpd服务

```bash
cd elements/rhel/post-install.d

cat > 00-install-httpd <<-EOF
#!/bin/bash
yum makecache
yum install httpd -y
EOF

cat > 01-create-index <<-EOF
#!/bin/bash
echo hello lixiaohui > /var/www/html/index.html
EOF

cat > 02-enable-httpd <<-EOF
#!/bin/bash
systemctl enable httpd
EOF
```

授予执行权限

```bash
cd
chmod +x elements/rhel/post-install.d/*
```

根据以上信息，定制构建变量

```bash
export DIB_LOCAL_IMAGE=/root/osp-small.qcow2
export DIB_YUM_REPO_CONF=/etc/yum.repos.d/rhel-dvd.repo
export ELEMENTS_PATH=/root/elements
export DIB_NO_TMPFS=1
```

开始构建镜像

这个构建可能需要较长时间，请耐心等待

```bash
disk-image-create vm rhel -t qcow2 -a amd64 -p httpd -o httpd.qcow2
```

输出

```textile
2024-11-11 06:36:12.495 | Converting image using qemu-img convert
2024-11-11 06:37:48.987 | Image file httpd.qcow2 created...
2024-11-11 06:37:49.169 | Build completed successfully
```

## Guestfish和 Virt-customize

Guestfish 和 Virt-customize 都使⽤ libguestfs API 来执⾏其功能。Libguestfs 需要可以处理各种不同格式的后端，默认情况下使⽤ libvirt

### Guestfish

下例使⽤ -i 选项来⾃动挂载分区，使⽤ -a 选项来添加磁盘镜像，并且使⽤ --network 选项来启⽤⽹络访问

```bash
yum install libguestfs-tools -y
[root@workstation ~]# guestfish -i --network -a osp-small.qcow2
libguestfs: error: could not create appliance through libvirt.

Try running qemu directly without libvirt using this environment variable:
export LIBGUESTFS_BACKEND=direct
```

我们发现，报告没有libvirt，因为我们是虚拟机，没有是正常的，导出这个direct的变量重新运行即可

```bash
[root@workstation ~]# export LIBGUESTFS_BACKEND=direct
[root@workstation ~]# guestfish -i --network -a osp-small.qcow2

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: ‘help’ for help on commands
      ‘man’ to read the manual
      ‘quit’ to quit the shell

Operating system: Red Hat Enterprise Linux 8.1 (Ootpa)
/dev/sda1 mounted on /

><fs>
```

不要忘了最后更新selinux标签

```bash
><fs> command 'yum install httpd -y'
><fs> write '/var/www/html/index.html' 'hello lixiaohui'
><fs> command 'systemctl enable httpd'
><fs> command 'useradd lixiaohui'
><fs> command 'systemctl enable httpd'
><fs> selinux-relabel /etc/selinux/targeted/contexts/files/file_contexts /
><fs> exit
```

<mark>尝试创建一个实例看看是否成功</mark>

### Virt-customize

Virt-customize 是⼀款⾼级⼯具，它也使⽤ libguestfs API，但通过利⽤简单的选项执⾏任务来简化镜像构建，下例中使⽤ virt-customize 命令及 -a 选项来添加磁盘，安装⼀个软件包，设置 root 密码，再还原 SELinux 上下⽂

```bash
[root@workstation ~]# wget -O virt-test.qcow2 http://materials/osp-small.qcow2
[root@workstation ~]# virt-customize -a virt-test.qcow2 --install httpd --root-password password:lxhpassword --selinux-relabel
[   0.0] Examining the guest ...
[  20.9] Setting a random seed
[  21.0] Installing packages: httpd
[  77.1] Setting passwords
[  88.5] SELinux relabelling
[ 246.6] Finishing off
```

## Cloud-init

为了避免镜像的激增，您可以只维护⼏个通⽤基础镜像，然后在部署期间执⾏实例相关的⾃定义。Cloud-init 是⼀组 Python 脚本和实⽤程序，可以执⾏实例的早期初始化。其任务可能包括注⼊SSH 密钥、配置⽹络设置和设置主机名。Cloud-init 需要⽤于配置实例的信息称为⽤⼾数据。

### Cloud-init 服务

1. **cloud-init-local.service**
   
   - **功能**：此服务在网络可用之前，但在根分区挂载为可读写后首次运行。主要用于向实例提供网络配置信息。
   
   - **执行命令**：`cloud-init init --local`

2. **cloud-init.service**
   
   - **功能**：这是引导过程中的第二个服务，需要网络已配置并可用。此阶段处理所有用户数据，执行配置文件中 `cloud_init_modules` 下列出的所有模块。
   
   - **执行命令**：`cloud-init init`

3. **cloud-config.service**
   
   - **功能**：在 `cloud-init.service` 之后运行，执行配置文件中 `cloud_config_modules` 下列出的所有模块。
   
   - **执行命令**：`cloud-init modules --mode=config`

4. **cloud-final.service**
   
   - **功能**：这是最后一个运行的服务，依赖于 `cloud-config.service` 和 `rc-local.service`，尽可能晚执行。此服务执行配置文件中 `cloud_final_modules` 下列出的所有模块。
   
   - **执行命令**：`cloud-init modules --mode=final`

如果有兴趣，可以看看模块的内容：

```textile
/usr/lib/python3.6/site-packages/cloudinit/config/
```


