# myLFS
基于PC环境的再次LFS(Linux From Scratch)
自从在树莓派LFS了之后(这个过程见我以前博客)，想想原来在pc上LFS的时候是一个个命令执行，这样效率太低了，而且容易出错。既然树莓派上可以用脚本编译软件包，那pc上也可以这样做啊。于是就将树莓派上的LFS脚本修改了一下，在debain9上又LFS了一把，呵呵，可能我LFS上瘾了。
宿主机系统:debain9
LFS版本:网上（http://www.linuxfromscratch.org/lfs/）下载LFS8.1（好在软件包版本和树莓派上的版本基本一致，有少数软件包版本不同）
建议先看一下LFS的文档，可以中文版7.7版本和8.1的英文版对照着看，主要了解一下LFS的大致过程。我的步骤相比标准的LFS过程，最主要的差别是所有软件包的编译在ch5-build.sh和ch6-build.sh两个脚本中进行，如果中间有错误，程序会自动停下来。本文档所有的脚本都可以在https://github.com/breezecloud/myLFS下载，但不包括源程序包，源程序包请自行根据文档下载。

一、宿主机安装必要的软件包
安装debain9：网上下载dvd安装盘，在分区时为LFS单独分一个区，大小20G以上,另外分一个2G交换分区。
在debain系统的终端上执行：（以下同）
sudo su
apt-get update
apt-get install bison gawk m4 texinfo
apt-get install gcc g++ automake autoconf

二、准备LFS用新分区
mkfs -v -t ext4 /dev/<xxx> ;在 LFS 分区上创建 ext4 文件系统，<xxx>是分区的名字，如sda1
export LFS=/mnt/lfs
mkdir -pv $LFS;-p如果目录存在不报错
mount -v -t ext4 /dev/<xxx> $LFS
mkdir –v $LFS/sources
chmod -v a+wt $LFS/sources
mkdir -v $LFS/tools
ln -sv $LFS/tools /  ;在根目录下建立链接
;增加交换文件
mkswap /dev/<yyy> ;新建 swap 分区
/sbin/swapon -v /dev/<zzz>

三、下载软件包和补丁
可以在http://www.linuxfromscratch.org/lfs/view/stable/chapter03/packages.html和http://www.linuxfromscratch.org/lfs/view/stable/chapter03/patches.html下载所有的软件包。建议使用迅雷之类的下载软件一次下载全部源程序包。（注意：SourceForge的网站无法用迅雷下载必须手工下载）。将下载的软件包拷贝到$LFS/sources目录下。

四、添加lfs用户
执行以下命令：
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
passwd lfs
chown -v lfs $LFS/tools
chown -v lfs $LFS/sources
su – lfs
export LFS=/mnt/lfs

五、构建临时文件系统
以lfs用户执行4_4_set_env.sh，执行完之后退出lfs用户重新登录。确认一下环境变量$LFS(/mnt/lfs)正确。
cd $LFS/sources
./4_4_set_env.sh
exit
su - lfs 
cd /lfs/sources
./ch5-build.sh ;执行编译脚本
编译完成的所有工具软件安装在/tools目录下，修改文件属性，以root用户执行：
exit
export LFS=/mnt/lfs
chown -R root:root $LFS/tools

六、安装基本的系统软件
ch5-build.sh执行之后,以下开始已root身份执行命令，执行s6.2.sh 确认一下/mnt/lfs/dev/consul ，/mnt/lfs/dev/null是否生成,执行：
export LFS=/mnt/lfs
cd $LFS/sources
./s6.2.sh
./S6.4_chroot.sh ;进入chroot，需要root用户执行，否则失败。如果机器重启之后要先执行s6.2r.sh再chroot。
;执行s6.5_6.sh（生成相关目录、passwd、group文件）：
cd /sources
./s6.5_6.sh
exec /tools/bin/bash --login +h 
touch /var/log/{btmp,lastlog,wtmp} 
chgrp -v utmp /var/log/lastlog 
chmod -v 664 /var/log/lastlog 
chmod -v 600 /var/log/btmp
./ch6-build.sh;执行编译脚本

七、清除系统及基本系统配置
清除调试信息及/tools目录，此时/tools目录已经不需要可以删除。当然如果你想再次LFS，可以备份/tools目录，这样下次第一阶段（ch5-binuld.sh）就不需要执行了。
logout
export LFS=/mnt/lfs
cd $LFS/sources
./s6.71_chroot.sh ;从现在开始，要在退出后重新进入 chroot 环境就要执行:
s6.4_chroot.sh
s6.71r_chroot.sh
修改s7.2_9.sh脚本，脚本需要在/etc目录下生产了一个fstab，其中<xxx>， <yyy> 和 <fff> 请修改脚本用适当的值替换。例如 sda2，sda5 和 ext4；另外假定根分区（或者是磁盘分区）是 sda2，脚本生成了/boot/grub/grub.cfg配置文件，如果启动文件不是sda2需要修改脚本。
./s7.2_9.sh ;基本系统配置
pc版本的配置（包括启动配置）比树莓派复杂，这里请参考手册的介绍。另外这个版本的系统初始化是基于传统的Sysvinit，还有一个版本的LFS基于systemd的初始化管理，比Sysvinit更新，但也更复杂。建议先从Sysvinit这个版本开始。

八、让LFS可引导
顺利执行脚本s7.2_9.sh之后，基本完成了系统的配置，。然后编译内核执行：
cd $LFS/source
./s8.3.sh ;建议手工执行，也许编译中间出现问题呢？
如果上面脚本胜利执行完成那就要请负责启动电脑的grub出场了。它是一个多重操作系统启动管理器，用来引导不同系统，如windows，linux。
grub-install /dev/sda ;命令将会覆盖已有的引导器。如无需要，请勿运行！！！
（此时也可以使用update-grub生成/boot/grub/grub.cfg配置文件，但发现找不到update-grub命令，update-grub是grub-mkconfig -o /boot/grub/grub.cfg的简化版，可以用grub-mkconfig代替）
但我的建议是既然宿主机debian上已经安装了grub（如果不是用LiveCD这样的方式安装的话），干嘛不配置宿主机的grub来启动我们的LFS呢？当然你需要学习一下如何配置grub以新增加启动的系统。不过明显这样做比较安全，我觉得花这点时间还是值的，如何配置就当思考题读者自己完成吧。
至此已经完成了全部的编译和配置，最后检查以下的配置文件是不是都是正确的吧。
/etc/bashrc
/etc/dircolors
/etc/fstab
/etc/hosts
/etc/inputrc
/etc/profile
/etc/resolv.conf
/etc/vimrc
/root/.bash_profile
/root/.bashrc
终于完成了，先退出chroot：
logout
然后卸载虚拟文件系统：
umount -v $LFS/dev/pts
umount -v $LFS/dev
umount -v $LFS/run
umount -v $LFS/proc
umount -v $LFS/sys
卸载 LFS 文件系统本身：
umount -v $LFS
祈祷一下重启电脑试试是否能成功启动吧。

九、结束语
由于很多步骤都已经写成了脚本，所以建议还是仔细阅读一下文档会对理解LFS的过程很有帮助，脚本只是让你能避免输入错误而无法成功LFS。虽然完成了LFS，但其实离真正理解linux还差很远，不过LFS给了我们一个概念，组成linux这盘大餐的原料是什么？它是怎么烧出来的？如果你想深入了解每个原料的原理可以进一步去出品这个原料的官方网站探索。
最后要遗憾的指出目前完成的操作系统基本是无法在日常工作中使用的，只是具有基本的功能，不过LFS发展到现在也已经有BLFS和ALFS等“变种”，他们可以在LFS的基础上根据你的要求构建出一个为你所用的独一无二的操作系统。所以还是那句话：生命不息，折腾不止。
