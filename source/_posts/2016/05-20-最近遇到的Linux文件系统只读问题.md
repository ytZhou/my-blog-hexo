---
title: 最近遇到的Linux文件系统只读问题
date: 2016-05-20 11:40:20
categories:
- Linux
tags:
- Linux
---
最近两次遇到Linux系统中磁盘出现问题而导致不能写入和更改磁盘数据。

## 问题的出现
- 问题一： 某天早上导师说实验室网站挂掉了，让我去看看。从YII输出的错误信息看应该是后端MySQL连接不上，只好登录服务去查看MySQL进程是否存在。
- 问题二： 今天由于实验室自己的Linux主机ubuntu出现问题，通过另外一块盘备份的时候出现不能写入，在该盘挂载的目录下执行命令都会提示Input/Output error。ubuntu forums上的相关问题

## 问题的解决
- 对于第一个问题登录主机之后发现MySQL进程存在。然后试着通过从shell登录MySQL，结果提示Can’t connect to local MySQL server through socket /xx/xx.sock不能创建，网上说是需要修改这儿sock的权限+R即可，但是修改之后依旧不可以，ls之后发现已经是rwx了。这就让我非常纳闷了。后面发觉整个Linux文件系统不能写入东西了，然后more /proc/mounts之后发现分区的确变成ro只读状态了。由于之前备份过数据，只好冒着一定的危险，选择万能的重启策略了。最好，重启之后手动开启了Apache和Mysql服务就完成了问题的解决。
- 由于有了之前的经验，今天遇到这个问题网上查了资料说是可能是磁盘损坏导致。通过dmesg命令发现，这一条log：
[ 139.666511] EXT4-fs (sdb5): mounting ext3 file system using the ext4 subsystem
[ 139.724006] EXT4-fs (sdb5): warning: mounting unchecked fs, running e2fsck is recommended
[ 139.725427] EXT4-fs (sdb5): mounted filesystem with ordered data mode. Opts: (null)
[ 200.126359] EXT4-fs error (device sdb5): ext4_lookup:1441: inode #110643744: comm rm: deleted inode referenced: 110668315

既然如此提示，那就通过e2fsck来修复sdb5这个分区，但是修复的时候没选-a自动修复参数，默然是-y交互方式，导致修复的时候一直按y按了好长时间 囧。

## 小记
当磁盘出现问题的时候一般会发出信号，使得文件系统变成只读模式，此时不能修改和写入数据。这个时候比较快的策略应该是dmesg查看kernel的log，然后通过mount或者/proc/mounts看已经挂载分区的状态。如果是重要数据那么选择备份之后，卸载文件系统尝试通过e2fsck/fsck命令修复分区。
涉及命令：mount/umount/df/fdisk/lsof(这个命令很强大，非常有用)/fsck/e2fsck(-a自动修复，-y交互模式)

