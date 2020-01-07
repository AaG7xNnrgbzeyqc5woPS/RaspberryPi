
http://www.tyrantek.com/archives/508/

---------------------------


原文: 使用dump和restore来制作SD卡的备份img文件   
 http://www.tyrantek.com/archives/508/  
 请看原文,有图片,更好的排版,以下仅仅做为备份  
 
 --------------------------------------

随着树莓派的不断使用，系统已不再是最初的系统。出于对SD卡可靠性的担忧，我需要备份整个SD卡。同时手上另有一张空白的SD卡，如何复制SD卡并正常启动，是摆在我面前的难题。

一开始我使用dd来备份，但dd属于底层的设备块拷贝工具，大量的空白数据既占空间，备份速度又慢。更可恶的是，不同的SD卡真实大小不一样，同样是8G的SD卡，dd恢复回去后系统不能启动。只有从小镜像恢复到大SD卡才能成功。

经过漫长的有效数据备份探索，总算实现只备份有效数据。整个过程不需要任何付费软件，全部由Linux命令工具完成。

例中在一台独立的ubuntu系统中进行备份，通过插入树莓派的SD卡来读取数据。

# 生成空白镜像

使用下面的命令生成一个空白镜像文件，count为你需要的大小。

sudo dd if=/dev/zero of=raspberrypi.img bs=1MB count=3000   

> 注：也许你需要在树莓派上使用df -h命令，预先需要估计下img文件需要的大小。

# 镜像分区

parted可对img进行分区，如果没有这个命令，使用sudo apt-get install parted来安装。
首先需要给镜像添加Label，并标记为msdos格式。

sudo parted raspberrypi.img mklabel msdos

这里仅标签，并未进行实际的格式化。标签后才能进一步分区。
然后键入下面的命令进入parted的交互分区界面。

sudo parted raspberrypi.img

键入p可以查看分区情况，如下图所示：

parted1.png

键入面的命令分区：

mkpart primary fat32 1 32M  
mkpart primary ext4 32M -1  

> 注意：为了避免出现“The resulting partition is not properly aligned for best performance.” 这样的错误，第一个分区的start起点为1。

之后再次键入p查看分区情况，如下图所示：

parted2.png  

这时候可以看到两个分区已建立好，由于未进行格式化，所以“File System”字段还是空的。  

键入q推出parted界面。  

# 格式化镜像中的分区  

格式化工具不支持在img中格式化，所以需要先挂载镜像。

在 Linux 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。

## 连接loop设备

sudo losetup --show -f raspberrypi.img

命令会返回链接后的设备路径，本例中为/dev/loop0。
## 连接设备后还需映射分区：
sudo kpartx -va /dev/loop0

如果找不到kpartx，使用命令sudo apt-get install kpartx来安装。
kpartx运行成功将打印映射后的信息：
add map loop0p1 (252:0): 0 61440 linear /dev/loop0 2048
add map loop0p2 (252:1): 0 5795840 linear /dev/loop0 63488
现在你可以在/dev/mapper目录下找到两个分区的符号。

## 总算可以格式化了，运行下面的命令：

sudo mkfs.vfat /dev/mapper/loop0p1
sudo mkfs.ext4 /dev/mapper/loop0p2

# 向分区中写入数据

首先需要把等待备份的SD卡用USB 读卡器连接到 Ubuntu，然后依次挂载SD卡的两个分区。

sudo mkdir /mnt/usb1
sudo mkdir /mnt/usb2
sudo mount -t vfat /dev/sdb1 /mnt/usb1
sudo mount -t ext4 /dev/sdb2 /mnt/usb2

自行替换上述命令中的SD卡设备号。

创建一个用来挂载img镜像中分区的目录/mnt/image:

sudo mkdir /mnt/image

## 挂载img中的fat分区

sudo mount -t vfat /dev/mapper/loop0p1 /mnt/image

用cp命令拷贝SD卡的fat分区数据到镜像的fat分区：

sudo cp -rfp /mnt/usb1/* /mnt/image

参数中：r递归拷贝，f强制读取，p不拷贝连接符号。
拷贝完成，卸载img的fat分区：

sudo umount /mnt/image

## 再加载img的ext4分区：

sudo mount -t ext4 /dev/mapper/loop0p2 /mnt/image

却换到mnt/image目录，ext4文件系统需要使用dump和restore来完整的导出有用数据：

cd /mnt/image
sudo dump -0uaf - /mnt/usb2 | sudo restore -rf -

如果找不到dump，使用命令sudo apt-get install dump来安装。

 *dump的参数：*

    0完全备份；
    u当成功备份后信息写入/var/lib/dumpdates
    a自动大小
    f后跟-表明写到标准输入
    /mnt/usb2指明从哪里dump数据

*restore参数:*

    r表明重建文件系统
    目标系统必须为mke2fs类型
    f后跟-表明从标准输入读取数据

# 清理

img制作完成，需要卸载镜像

sudo umount /mnt/image
sudo kpartx -d /dev/loop0
sudo losetup -d /dev/loop0

然后卸载掉SD卡：

sudo umount /mnt/usb1
sudo umount /mnt/usb2

现在raspberrypi.img中已包含了完整的SD数据备份，可以使用dd恢复到任何大于3000M的SD卡中。
拓展

整个备份过程以制作img文件为目标，但其实可以直接进行SD卡与SD卡的复制。
用parted来分区目标SD卡、并格式化，加载格式化后的分区，按照数据拷贝流程进行数据迁移。

# 备注: 
  dd 命令中注意, M 和 MB 和 MiB的差别. M=MiB=1024KiB, M=1000KiB
  做好img后注意使用 parted检查下
  
  
