
# 前言

在使用树莓派利用SD卡启动系统的过程中，如何配置一张新买的空SD卡也是一个必不可少的技能。本文主要是介绍如何将SD卡分成boot和root两个分区，操作流程主要分为分区、格式化两步

# 操作流程

+ 分区  
查看插入的SD卡情况，可以看出目前插入了一张32G（29G）的SD卡，系统分配给其的名字是/dev/sdb，目前存在1个分区，名字分别为/dev/sdb1，格式是W95 FAT32 (LBA)

    ``` bash  
    hsq@ares:~$ sudo fdisk -l
    Disk /dev/sdb: 29 GiB, 31104958464 bytes, 60751872 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x9c2d2e7a

    Device     Boot Start      End  Sectors Size Id Type
    /dev/sdb1  *     8192 60751871 60743680  29G  c W95 FAT32 (LBA)
    ```

 
    操作分区信息前先卸载掉系统对SD卡的自动挂载  

    ``` bash  
    hsq@ares:~$ umount /media/hsq/kingston
    ```

    然后利用fdisk命令进行分区操作

    ``` bash  
    hsq@ares:~$ sudo fdisk /dev/sdb

    Welcome to fdisk (util-linux 2.27.1).
    Changes will remain in memory only, until you decide to write them.
    Be careful before using the write command.


    Command (m for help): p #这里也可以查看SD卡分区情况
    Disk /dev/sdb: 29 GiB, 31104958464 bytes, 60751872 sectors
    Units: sectors of 1 * 512 = 512 bytes #默认每个扇区0.5KB
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x9c2d2e7a

    Device     Boot Start      End  Sectors Size Id Type
    /dev/sdb1  *     8192 60751871 60743680  29G  c W95 FAT32 (LBA)

    Command (m for help): d#删除当前的分区，有几个分区就要运行几次d
    Selected partition 1
    Partition 1 has been deleted.

    Command (m for help): n #创建一个新的分区作为boot分区
    Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
    Select (default p): #一般用默认的主分区即可

    Using default response p.
    Partition number (1-4, default 1): #分区号默认按序即可
    First sector (2048-60751871, default 2048):  #起始扇区序号默认即可
    Last sector, +sectors or +size{K,M,G,T,P} (2048-60751871, default 60751871): 205824 #终止扇区序号，2000个扇区是1MB，这里与初始扇区差值约为100MB

    Created a new partition 1 of type 'Linux' and of size 99.5 MiB.

    Command (m for help): t #改变分区类型
    Partition type (type L to list all types): L #打印所有的支持类型，很长这里省略
    ······
    ······
    ······
    Partition type (type L to list all types): c #选择类型c，即W95 FAT32 (LBA)作为boot类型

    Changed type of partition 'Linux' to 'W95 FAT32 (LBA)'.

    Command (m for help): n #再创建一个分区作为root分区
    Partition type
    p   primary (1 primary, 0 extended, 3 free)
    e   extended (container for logical partitions)
    Select (default p): #默认

    Using default response p.
    Partition number (2-4, default 2): #默认
    First sector (205825-60751871, default 206848): #默认
    Last sector, +sectors or +size{K,M,G,T,P} (206848-60751871, default 60751871): #默认把剩下所有的空间都分配给root

    Created a new partition 2 of type 'Linux' and of size 28.9 GiB.

    Command (m for help): p #查看一下重新分区后的信息
    Disk /dev/sdb: 29 GiB, 31104958464 bytes, 60751872 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0x9c2d2e7a

    Device     Boot  Start      End  Sectors  Size Id Type
    /dev/sdb1         2048   205824   203777 99.5M  c W95 FAT32 (LBA)
    /dev/sdb2       206848 60751871 60545024 28.9G 83 Linux

    Command (m for help): w #保存分区信息，注意需要之前卸载掉系统对SD卡的自动挂载，否则这里会有一个警告
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Syncing disks.
    ```

+ 格式化  
分区后还要对SD卡的各个分区进行格式化和重命名  

    ``` bash  
    hsq@ares:~$ sudo mkfs.msdos /dev/sdb1 -n boot #将第一个分区sdb1格式化为fat类型，并重命名为boot
    mkfs.fat 3.0.28 (2015-05-16)
    mkfs.fat: warning - lowercase labels might not work properly with DOS or Windows
    hsq@ares:~$ sudo mkfs.ext4 /dev/sdb2 -L root #将第二个分区sdb2格式化为ext4类型，并重命名为root
    mke2fs 1.42.13 (17-May-2015)
    Creating filesystem with 7568128 4k blocks and 1892352 inodes
    Filesystem UUID: 9702d479-9bc1-4c72-9398-70d018b9fbd5
    Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
        4096000

    Allocating group tables: done                            
    Writing inode tables: done                            
    Creating journal (32768 blocks): done
    Writing superblocks and filesystem accounting information: done
    ```

# 问题解决

1. 问题： 用fdisk命令分区结束后w保存时出现警告  
 错误提示：Re-reading the partition table failed.:Device or resource busy  
 原因分析：因为用fdisk命令前没有卸载系统自动挂载的SD  
 解决：umount /media/hsq/kingston

2. 问题：挂载分区虽然成功但是有报警告  
错误提示：Volume was not properly unmounted  
原因分析：多次意外掉电造成的。据说可以把分区文件夹的权限设置为只读而避免这个问题，没验证过  
解决：在ubuntu主机上用以下命令修复。[参考](https://www.tuicool.com/articles/U3A3Ubf)
    ```bash 
    fsck.fat -V /dev/mmcblk0p1
    ```
2. 问题：  
错误提示：   
原因分析：  
解决：
# 参考资料

[1] flash介绍 https://www.cnblogs.com/jxjl/p/7138133.html  
[2] 本文参考的雷同文章 http://www.360doc.com/content/16/0108/20/12144668_526479541.shtml  
[3] 本文参考的雷同文章 https://www.iteye.com/blog/womendu-1229916  
[4] 本文参考的雷同文章，提到了一个专家模式 http://www.roboby.com/making_sd_card_for_linux_boot.html