# 编译主线U-Boot

参考[lanseyujie](https://github.com/lanseyujie/tn3399_v3)的教程

首先编译[ATF](https://github.com/ARM-software/arm-trusted-firmware/tags)

下载源码并解压，cd进入

执行make CROSS_COMPILE=aarch64-linux-gnu- PLAT=rk3399编译

编译完成后设置环境变量：export BL31=path_to_your_bl31.elf

接着编译U-Boot

下载主线U-Boot源码，解压并打上项目中提供的patch

执行make tn3399-v3-rk3399_defconfig && make CROSS_COMPILE=aarch64-linux-gnu- -j32完成编译

源码目录下的u-boot-rockchip.bin就是所需镜像

将其烧录到img镜像或者TF卡的32k偏移处即可引导系统。**注意使用dd烧录到img镜像时，要加上conv=notrunc选项**

# 编译内核

下载内核源码，添加项目提供的dts，正常编译即可

主线内核从kernel.org下载

BSP内核推荐[mrfixit2001](https://github.com/mrfixit2001/rockchip-kernel)维护的版本

# 构建发行版

## 对相同SoC的系统镜像进行移植

举例：

下载rockpro64的[Manjaro-ARM](https://github.com/manjaro-arm/rockpro64-images)系统镜像，使用losetup挂载img到临时目录

替换dtb为TN3399_V3

进行自定义，例如使用neovim代替nano，卸载ntfs-3g而使用ntfs3，卸载rockpro64的U-Boot包

最后将TN3399_V3的U-Boot刻录到镜像上

大概操作：

```
# 查找第一个未使用的loop设备，一般是/dev/loop0
sudo losetup -f
# 将/dev/loop0和img关联起来
sudo losetup path_to_your_img /dev/loop0
# 根据img更新/dev/loop的分区
sudo partprobe /dev/loop0
# 挂载分区到临时目录
sudo mount /dev/loop0p2 /mnt && sudo mount /dev/loop0p1 /mnt/boot
# chroot到临时目录，主机需要安装systemd-container qemu-user-static binfmt
sudo systemd-nspawn -D /mnt -M tmp
# 对rootfs做些自定义，例如换源，添加预装软件等
# 退出systemd-nspawn，方式是按住 Ctrl，接着连按]三次
# 卸载分区
sudo umount -R /mnt
# 删除loop设备
sudo losetup -d /dev/loop0
# 给img烧录U-Boot，不同SoC厂商的启动偏移地址不一样
sudo dd if=path_to_uboot of=path_to_your_img bs=1k seek=32 conv=notrunc
```

## 从零构建系统镜像

要使用Ubuntu可以从[这里](http://mirrors.ustc.edu.cn/ubuntu-cdimage/ubuntu-base/releases/)下载rootfs，解压到临时目录，chroot后对其进行自定义：换源、添加用户、修改主机名、语系、时区，预装安装软件等

然后将编译的内核镜像和模块放到rootfs中，配置一种引导方式，如Extlinux或者U-Boot Script，整个rootfs就可以通过U-Boot引导并正常使用

可使用dd命令创建空白镜像，对镜像进行分区，放入rootfs，最后刻录U-Boot，一个系统镜像就制作完成

x86主机chroot到arm的rootfs需要安装qemu-user-static。chroot的方法有chroot和systemd-nspawn两种，推荐第二种

大概操作：

```
# 创建空白文件，大小自己决定
sudo dd if=/dev/zero of=~/myimg.img bs=1M count=2048 status=progress
# 给文件创建gpt分区表
sudo parted ~/myimg.img mklabel gpt
# 给文件分区，也可以使用fdisk parted等分区工具，第一个分区起始位置需要靠后，给U-Boot留下空间
sudo cfdisk ~/myimg.img
# 参考上面的操作，使用losetup挂载img,下载Ubuntu Ports的rootfs复制到img里，做些自定义，比较重要的有passwd sudo timezone locales hosts hostname fstab的配置
```

# WIFI

内核需要搭配固件才能驱动AP6255，主线内核和BSP内核所使用的固件不同，但都应该将其放置到rootfs的/lib/firmware/brcm下