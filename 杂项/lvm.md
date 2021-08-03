# 扩展LVM
pvcreate /dev/sde
vgextend nlas /dev/sde
lvextend -l +100FREE /dev/nlas/root


# 新增硬盘，创建/data，并挂在到新分区

首先查看磁盘以及分区，
```
# fdisk -l
```
以下为命令输出内容
```
...

磁盘 /dev/sdb：8001.5 GB, 8001506074624 字节，15627941552 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：262144 字节 / 524288 字节


磁盘 /dev/sda：480.1 GB, 480070426624 字节，937637552 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：262144 字节 / 262144 字节
磁盘标签类型：gpt
Disk identifier: 45848BB0-5815-4C2E-AA05-39BA98F1B50A


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648      2508799      1G  Microsoft basic
 3      2508800    937635839  445.9G  Linux LVM

...

```

可以看到新增的硬盘是/dev/sdb。使用fdisk为新硬盘分区
```
# fdisk /dev/sdb
命令(输入 m 获取帮助): g                                                     // 创建一个空的gpt分区表
命令(输入 m 获取帮助): n                                                     // 创建一个新的分区
(1-128 1)                                                                   // 输入分区编号默认为1 直接回车就好
(2048-15627941518 2048)                                                     // 选择分区的开始扇区号，默认2048,直接回车
Last sector, +sectors or +size{K,M,G,T,P} (2048-15627941518 15627941518)    // 选择分区结束扇区号，默认15627941518， 直接回车
命令(输入 m 获取帮助): t                                                     // 修改修改分区格式
已选择分区 1                                                                 // t 命令的输出
分区类型(输入 L 列出所有类型): 31                                            // 设置分区类型为 Linux LVM
命令(输入 m 获取帮助): w                                                     // 保存更改并退出
```

此时再使用```fdisk -l```命令查看磁盘分区输出如下结果
```
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

磁盘 /dev/sdb：8001.5 GB, 8001506074624 字节，15627941552 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：262144 字节 / 524288 字节
磁盘标签类型：gpt
Disk identifier: 63A52C5E-E16F-4654-A9FF-B72F76A557E3


#         Start          End    Size  Type            Name
 1         2048  15627941518    7.3T  Linux LVM
WARNING: fdisk GPT support is currently new, and therefore in an experimental phase. Use at your own discretion.

磁盘 /dev/sda：480.1 GB, 480070426624 字节，937637552 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 4096 字节
I/O 大小(最小/最佳)：262144 字节 / 262144 字节
磁盘标签类型：gpt
Disk identifier: 45848BB0-5815-4C2E-AA05-39BA98F1B50A


#         Start          End    Size  Type            Name
 1         2048       411647    200M  EFI System      EFI System Partition
 2       411648      2508799      1G  Microsoft basic
 3      2508800    937635839  445.9G  Linux LVM

...

```
可以看到，/dev/sdb上多了一个编号为1，大小7.3T，类型为Linux LVM的分区

下面可以创建pv
```
# pvcreate /dev/sdb1 # 将/dev/sdb1创建为pv
  Physical volume "/dev/sdb1" successfully created.
```
查看现有pv
```
# pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sda3  centos lvm2 a--  445.90g  4.00m
  /dev/sdb1  data   lvm2 a--   <7.28t <1.28t
```
使用 /dev/sdb1 创建新的vg
```
# vgcreate data /dev/sdb1
  Volume group "data" successfully created
```
查看vg
```
# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   3   0 wz--n- 445.90g  4.00m
  data     1   1   0 wz--n-  <7.28t <7.28t
```
在data卷组上创建大小为6t,名称为data的lv
```
# lvcreate +L 6T -n data data
  Logical volume "data" created.
```
查看lv
```
# lvs
  LV   VG     Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home centos -wi-ao---- <364.59g
  root centos -wi-ao----   50.00g
  swap centos -wi-ao----   31.31g
  data data   -wi-ao----    6.00t
```
格式化data逻辑卷
```
# mkfs.xfs /daev/mapper/data-data
meta-data=/dev/mapper/data-data  isize=512    agcount=32, agsize=50331584 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1610610688, imaxpct=5
         =                       sunit=64     swidth=128 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=521728, version=2
         =                       sectsz=512   sunit=64 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
挂在data分区
```
# mkdir /data  // 首先创建挂载点 /data
# vim /etc/fstab  // 编辑 /etc/fstab文件，以便开机自动挂在data分区，在文件最后新增下面一行内容
...

/dev/mapper/centos-swap swapswap    defaults0 0
```
使用```reboot```命令重启计算机后，查看分区情况
```
# df -h
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                  32G     0   32G    0% /dev
tmpfs                     32G     0   32G    0% /dev/shm
tmpfs                     32G   11M   32G    1% /run
tmpfs                     32G     0   32G    0% /sys/fs/cgroup
/dev/mapper/centos-root   50G  4.2G   46G    9% /
/dev/sda2               1016M  221M  796M   22% /boot
/dev/sda1                200M   12M  189M    6% /boot/efi
/dev/mapper/data-data    6.0T   34M  6.0T    1% /data
/dev/mapper/centos-home  365G   38M  365G    1% /home
tmpfs                    6.3G   12K  6.3G    1% /run/user/42
tmpfs                    6.3G     0  6.3G    0% /run/user/0
```
从上面的结果看到。已经将/dev/mapper/data-data 挂载到 /data上
