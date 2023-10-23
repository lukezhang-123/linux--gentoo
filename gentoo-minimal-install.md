# 在虚拟机vmware最小化安装minimal gentoo系统


虚拟机配置：
1. 8核16G
2. 硬盘scsi模式，50G
3. 选择linux 5.x 64位系统模式
4. 高级，设置efi启动


安装说明：
1. 安装过程需要`联网`下载依赖的源码包
2. 使用efi启动方式安装


下载安装介质和包

打开下载主页，[https://www.gentoo.org/downloads/](https://www.gentoo.org/downloads/)

下载桌面用amd64模式的安装包

![gentoo download amd64](gentoo-minimal-install/2023-10-23-20-53-15.png)

需要下载两个
1. Minimal Installation CD.iso 是启动到安装环境的iso
2. Stage 3.tar.xz 是系统安装需要的文件


启动iso到live 安装环境

![](gentoo-minimal-install/2023-10-23-20-57-27.png)

![](gentoo-minimal-install/2023-10-23-20-59-50.png)


1. 配置live cd环境的ssh

```
passwd root
/etc/init.d/sshd start

# 测试ssh连接，ssh连接，确认sftp文件可以传，后面传stage3包
ssh root@192.168.180.132
```

2. 配置硬盘，分区准备

```
fdisk /dev/sda
p
g # gpt disklabel
n


+1G # 新建1G分区，下面 t,设置 Partition type， EFI System
t
1
n



p
w

mkfs.vfat -F 32 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt/gentoo
df -h

cd /mnt/gentoo
上传stage3
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner

nano /mnt/gentoo/etc/portage/make.conf
# 改
COMMON_FLAGS="-march=native -O2 -pipe"
# 这几行加下面
GENTOO_MIRRORS="https://mirrors.tuna.tsinghua.edu.cn/gentoo/"
ACCEPT_LICENSE="*"
MAKEOPTS="-j8"
GRUB_PLATFORMS="efi-64"

# ebuild
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
nano /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
sync-uri = rsync://rsync.mirrors.tuna.tsinghua.edu.cn/gentoo-portage/

cp --dereference /etc/resolv.conf /mnt/gentoo/etc/

mount --types proc /proc /mnt/gentoo/proc 
mount --rbind /sys /mnt/gentoo/sys 
mount --make-rslave /mnt/gentoo/sys 
mount --rbind /dev /mnt/gentoo/dev 
mount --make-rslave /mnt/gentoo/dev 
mount --bind /run /mnt/gentoo/run 
mount --make-slave /mnt/gentoo/run 
chmod 1777 /dev/shm /run/shm

chroot /mnt/gentoo /bin/bash 
source /etc/profile 
export PS1="(chroot) ${PS1}"

mkdir /efi
mount /dev/sda1 /efi

emerge-webrsync # 安装ebuild数据库快照
eselect profile list
eselect profile set 22 # [22]  default/linux/amd64/17.1/systemd (stable) *

emerge --ask --verbose --update --deep --newuse @world

emerge --ask app-portage/cpuid2cpuflags
cpuid2cpuflags
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags

ls /usr/share/zoneinfo
ln -sf ../usr/share/zoneinfo/Asia/Shanghai /etc/localtime
nano /etc/locale.gen # 放开注释 en_US.UTF-8 UTF-8
locale-gen
eselect locale list
eselect locale set X  # en_US.utf8
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"

emerge --ask sys-kernel/linux-firmware

# 内核源码
emerge --ask sys-kernel/gentoo-sources
eselect kernel list
eselect kernel set 1  # 执行完才有 /usr/src/linux
ls -l /usr/src/linux  # /usr/src/linux-6.1.57-gentoo/

emerge --ask sys-kernel/genkernel

cd /usr/src/linux
cp /usr/share/genkernel/arch/x86_64/generated-config /usr/src/linux/
cp /usr/src/linux/generated-config /usr/src/linux/.config 
make menuconfig 

make -j8 # 18分钟
make modules_install
make install
ls /boot/

emerge --ask sys-kernel/dracut
ls /lib/modules/ # /lib/modules/6.1.53-gentoo-r1/
dracut --kver=6.1.53-gentoo-r1
ls /boot/initramfs*

exit
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab

enter chroot
emerge --ask net-misc/dhcpcd
systemctl enable dhcpcd

passwd # 新环境root密码

systemd-firstboot --prompt --setup-machine-id  # keymap 232 us, hostname gen
systemctl preset-all

nano /etc/ssh/sshd_config # PermitRootLogin yes
systemctl enable sshd
systemctl enable dhcpcd

emerge --ask sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg

exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot

systemctl start dhcpcd
ip addr add 192.168.180.132/24 dev ens33
ip link set dev ens33 up
```

系统安装完成后

![](gentoo-minimal-install/2023-10-23-21-11-03.png)
