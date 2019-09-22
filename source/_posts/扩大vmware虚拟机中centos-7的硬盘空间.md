---
layout: "post"
title: "扩大VMware虚拟机中CentOS 7的硬盘空间"
date: "2018-07-31T16:37:12.149Z"
tags:
  - VMWare
  - CentOS7
  - 扩容
categories:
  - 收藏
  - 环境配置
---

## 查看挂载点信息：

```
[root@localhost]# df -h
文件系统 容量 已用 可用 已用% 挂载点
/dev/mapper/centos-root 18G 15G 2.9G 84% /
 devtmpfs 485M 0 485M 0% /dev
 tmpfs 494M 84K 494M 1% /dev/shm
 tmpfs 494M 7.1M 487M 2% /run
 tmpfs 494M 0 494M 0% /sys/fs/cgroup
 /dev/sda1 497M 119M 379M 24% /boot
 /dev/sr0 3.9G 3.9G 0 100% /run/扩大VMware虚拟机中CentOS 7的硬盘空间/zoubf/CentOS
```

<!-- more -->

## 扩展 VMWare 硬盘空间

关闭 Vmware 的 Linux 系统，这样，才能在 VMWare 菜单中设置需要增加到的磁盘大小

如果这个选项是灰色的，说明此虚拟机建有快照，把快照全部删除再试试

## 对新增加的硬盘进行分区、格式化

增加了空间的硬盘是 /dev/sda

分区：

```
[root@localhost]# fdisk /dev/sda
p　　　　　　　查看已分区数量（我看到有两个 /dev/sda1 /dev/sda2）
n　　　　　　　新增加一个分区
p　　　　　　　分区类型我们选择为主分区
 　　　　　　分区号选3（因为1,2已经用过了，见上）
 回车　　　　　　默认（起始扇区）
 回车　　　　　　默认（结束扇区）
t　　　　　　　修改分区类型
 　　　　　　 选分区3
 8e　　　　　　修改为LVM（8e就是LVM）
w　　　　　　写分区表
q　　　　　　完成，退出fdisk命令
```

使用`partprobe`命令 或者重启机器

格式化分区

```
mkfs.ext3 /dev/sda3
```

## 添加新 LVM 到已有的 LVM 组，实现扩容

```
lvm　　　　　　　　　　　　　　　　　　    进入lvm管理
lvm>pvcreate /dev/sda3　　　这是初始化刚才的分区，必须的
lvm>vgextend centos /dev/sda3  将初始化过的分区加入到虚拟卷组centos (卷和卷组的命令可以通过  vgdisplay )
lvm>vgdisplay -v
lvm>lvextend -l+21513 /dev/mapper/centos-root　　扩展已有卷的容量（21513 是通过vgdisplay查看的free的大小）
lvm>pvdisplay   查看卷容量，这时你会看到一个很大的卷了
lvm>quit    　退出

```

## 以上只是卷扩容了，下面是文件系统的真正扩容，输入以下命令：

CentOS 7 下面 由于使用的是 XFS，所以要用

```
xfs_growfs /dev/mapper/centos-root
```

CentOS 6 下面 要用

```
resize2fs /dev/mapper/centos-root
```

## 查看新的磁盘空间

```
df -h
```
