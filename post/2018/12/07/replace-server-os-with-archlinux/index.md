# 替换云服务器操作系统为Archlinux


操作环境：



> -  支持浏览器VNC方式登录的云服务商（e.g. 阿里云、腾讯云）
> -  Ubuntu 16.04 服务器


## 准备工作
对于不支持DHCP的厂商，需要我们自己去手动配置服务器的内网IP地址:
使用`ip addr`命令获取IP地址信息并记录下来，在搞完事情后需要我们根据这些信息手动设置服务器IP。在这里假设我的IP地址`172.18.65.234/18`，其对应的网关地址为`172.18.64.1`。
## 开始安装
### 下载镜像
下载最新archlinux镜像到根目录下：
```bash
root@Ali:~# cd   /
root@Ali:/# wget https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/latest/archlinux-2018.12.01-x86_64.iso
```
### 查看磁盘信息
```bash
root@Ali:/# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    254:0    0  40G  0 disk 
`-vda1 254:1    0  40G  0 part /
```
### 引导iso文件
在Ubuntu里编辑/boot/grub/grub.cfg文件，添加下列内容：
```bash
#timeout设为60,是为了VNC连接时有足够时间选择启动项，若为第一启动项，可不设置
set timeout=60
menuentry 'ArchISO' --class iso {
  set isofile=/archlinux-2018.12.01-x86_64.iso
  loopback loop0 $isofile
  #archisolabel设置archiso文件驻留的文件系统标签。
  #img_dev指明archiso文件驻留的设备
  #img_loop是archiso文件在img_dev里的绝对位置
  linux (loop0)/arch/boot/x86_64/vmlinuz archisolabel=ARCH20181201 img_dev=/dev/vda1 img_loop=$isofile
  initrd (loop0)/arch/boot/x86_64/archiso.img
}
```
然后重启，同时在浏览器里以VNC方式连接到服务器，在GRUB启动菜单里选择ArchISO进入。

### 安装Archlinux

进入Archlinux Live环境后，使用`lsblk`命令，你会发现我们的目标磁盘`/dev/vda1`被挂载到了`/run/archiso/img_dev`目录下。清楚了这一点后，就可以按照ArchWiki的介绍开始安装ArchLinux了，只需将步骤里的`/mnt`换为`/run/archiso/img_dev`即可。*有可能需要`mount -o remount,rw /run/archiso/img_dev`*。

### 联网
安装完成后，重启进入系统（浏览器VNC登录状态），使用`ip link`命令查看设备，使用`ip addr add ip地址 dev 设备`设置IP,使用`ip route add default via 网关 dev 设备`配置网关地址。
```bash
root@arch:~# ip link #查看设备
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:16:3e:0e:e4:6b brd ff:ff:ff:ff:ff:ff
root@arch:~# ip link set ens3 up
root@arch:~# ip addr add 172.18.65.234/18 dev ens3
root@arch:~# ip route add default via 172.18.64.1 dev ens3
root@arch:~# echo 'nameserver 8.8.8.8' > /etc/resolv.conf
```
大功告成！
## Reference
- README.bootparams https://git.archlinux.org/archiso.git/tree/docs/README.bootparams

- GUN GRUB Manual https://www.gnu.org/software/grub/manual/grub/grub.html

- Multiboot USB drive - ArchWiki https://wiki.archlinux.org/index.php/Multiboot_USB_drive#Arch_Linux_monthly_release
