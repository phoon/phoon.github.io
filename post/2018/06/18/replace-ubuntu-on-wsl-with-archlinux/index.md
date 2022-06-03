# Replace Ubuntu on WSL with Archlinux


## 契机  
Windows10的WSL[Windows Subsystem for Linux]在2017秋季更新后修复了大量的BUG。不过对于WSL,之前我一直处于围观状态，嗯，作为一位狂热Linuxer，怎么能随便投身Windows呢呵呵。

在最初的时候，WSL还只有Ubuntu，到现在应用商店里提供了有Fedora、Debian、SUSE和KALI。呃...等等，怎么能没有我们优秀的Archlinux呢？作为Archlinuxer第一个不服。

<!--more-->


 恰巧前几天逛论坛时看到了几篇关于在WSL上安装其他发行版本的文章，于是我也打开了久违的Windows(据说优秀的人都有两块硬盘的......嗯)，琢磨着把Archlinux给捣鼓进去。于是看了看网上的教程，然后结合自己的骚操作，作文以记之。

## 搞事情
1.首先呢，安装WSL(我装的Ubuntu18.04)对吧(这你肯定会的吧，不然你就是不优秀)。
2.将Ubuntu的默认用户改为root，打开PowerShell，键入:
```powershell
ubuntu1804 config --default-user root
```
3.安装Archlinux

- 3.1下载Archlinux根镜像airootfs.sfs：

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/latest/arch/x86_64/airootfs.sfs
```
这是一个squashfs格式的文件，我们需要用到squashfs-tools来解压它。

- 3.2 解压根镜像：

```bash
sudo apt install squashfs-tools
unsquashfs airootfs.sfs
```
解压得到squashfs-root文件夹

- 3.3替换ubuntu

关闭Bash窗口，进入Windows文件资源管理器，找到`C:\Users\{用户名}\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu*\LocalState\rootfs`文件夹，重命名备份为`rootfs-root`，将`rootfs-old/root/squashfs-root`文件夹复制到`rootfs-root`的同级目录，并将其重命名为`rootfs`。

- 3.4善后

再次打开我们的Ubuntu1804，就可以发现他已经变身成为Archlinux了(掌声、、、咳咳、、、低调)。
现在，我们需要初始化pacman的密钥：

```bash
pacman-key --init
```
这样就可以建立新密钥并生成系统主密钥，然后，验证咱们的主密钥：
```bash
pacman-key --populate archlinux
```
然后我就兴奋的用pacman搞事情了，然后...
我就遇到了一系列的：
```bash
error:*!@×@$*^%@&*% (unable to lock database)
```
这时候，我们只需要把旧锁给删掉就可以正常搞！事！情！了
```bash
rm /var/lib/pacman/db.lck
```
新建一个用户，并加入sudo:
```bash
useradd -m -g users -G wheel peven
nano /etc/sudoers          #添加
  peven ALL=(ALL) ALL
```
关闭Bash窗口，再次打开PowerShell，将默认用户改为peven:
```powershell
ubuntu1804 config --default-user peven
```
没生效或者出现问题，重启大法好。
## Duang~ Duang~ Duang~

```zsh
peven@ArchWSL ~ % neofetch
                   -`                    peven@ArchWSL
                  .o+`                   -------------
                 `ooo/                   OS: Arch Linux on Windows 10 x86_64
                `+oooo:                  Kernel: 4.4.0-17134-Microsoft
               `+oooooo:                 Uptime: 1 min
               -+oooooo+:                Shell: zsh 5.5.1
             `/:-:++oooo+:               Terminal: /dev/tty1
            `/++++/+++++++:              CPU: Intel i5-8250U (8) @ 1.800GHz
           `/++++++++++++++:             Memory: 1776MiB / 8089MiB
          `/+++ooooooooooooo/`
         ./ooosssso++osssssso+`
        .oossssso-````/ossssss+`
       -osssssso.      :ssssssso.
      :osssssss/        osssso+++.
     /ossssssss/        +ssssooo/-
   `/ossssso+/:-        -:/+osssso+-
  `+sso+:-`                 `.-/+oso:
 `++:.                           `-/+/
 .`                                 `/
```
完了你问体验？呵呵，回Windows是不可能回的，这辈子都不可能的。
## Referrence
`Install on WSL`: https://wiki.archlinux.org/index.php/Install_on_WSL

`Install from existing Linux`: https://wiki.archlinux.org/index.php/Install_from_existing_Linux


