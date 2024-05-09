---
title: how to use ks to build a centos template automatically
tags: automation, vsphere, kickstart
date: 2024-05-09 11:27:03
---

### 目标：

通过ks.cfg文件，在vsphere环境自动部署一台 centos 7 虚拟机

### 前提条件：

基础设施虚拟化环境：vSphere 7.0 U2

命令行工具: govc  mkisofs 

操作系统光盘镜像文件:  CentOS-7-x86_64-Minimal-1708.iso



### 操作步骤：

1. 准备 ks.cfg 文件

```ks.cfg
#version=DEVEL

# Use non-interactive command line mode
cmdline
eula --agreed
logging --level=info
selinux --disabled
firewall --disabled
poweroff

# System authorization information
auth --enableshadow --passalgo=sha512
# Use CDROM installation media
cdrom
# Run the Setup Agent on first boot
firstboot --disabled
# Keyboard layouts
keyboard --vckeymap=gb --xlayouts='gb'
# System language
lang en_GB.UTF-8

# Network information
network --onboot=no --device=link --bootproto=static --ip={{ ip }} --netmask={{ netmask }} --gateway={{ gateway }} --nameserver={{ dns_servers }} --hostname={{ hostname.domain }} --noipv6

# Root password
rootpw --iscrypted XYZZZZZZZZZZZZZZZZ
# System services
services --enabled=NetworkManager,sshd,chronyd
# System timezone
timezone Europe/London --isUtc
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
autopart --type=lvm
# Partition clearing information
clearpart --none --initlabel

%packages
@^minimal
@core
chrony

%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end

#Post Install
%post --log=/root/ks-post.log
yum install -y epel-release
yum install -y open-vm-tools
yum install -y perl
%end
```

ks.cfg 文件可以通过 kickstart generator 生成 [url https://access.redhat.com/labs/kickstartconfig/]

2. 创建 kickstart.iso

```bash
mkisofs -V OEMDRV -o kickstart.iso ks.cfg
```

3. 上传操作系统光盘镜像文件

```bash
govc datastore.upload CentOS-7-x86_64-Minimal-1708.iso isos/CentoOS.iso
```

4. 上传 kickstart.iso

```bash
govc datastore.upload kickstart.iso isos/kickstart.iso
```

5. 通过 CentOS iso 创建虚拟机 （名称 MyVM ）(不要开机启动，记得添加硬盘，设置网络)

```bash
govc vm.create -g=centos64Guest -on=false -iso=isos/CentoOS.iso -m 2048 -c 2 -net.adapter vmxnet3 \
 -disk.controller -disk 20GB pvscsi MyVM
```

6. 增加一个光驱 （记下设备号，通常是3001）

```bash
govc device.cdrom.add -vm MyVM
```

7. 关联上 kickstart.iso

```bash
govc device.cdrom.insert -vm MyVM -device 3001 isos/kickstart.iso
```

8. 设置启动顺序，和延迟

```bash
govc device.boot -vm MyVM -delay 1000 -order cdrom,ethernet,disk
```

9. 开机

```bash
govc vm.power -on MyVM
```

10. 等待自动安装结束
11. 获得所有光驱设备号

```bash
govc device.info -json -vm MyVM cdrom-*
```

12. 移除光驱设备

```bash
govc device.remove -vm MyVM 3000
govc device.remove -vm MyVM 3001
```

13. 打开 UUID 选项

```bash
govc vm.change -vm MyVM -e="disk.enableUUID=1"
```

14. 转换成模板

```bash
govc vm.markastemplate MyVM
```











