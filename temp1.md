对CloudEndure迁移后的EC2进行实例类型优化

越来越多的 Amazon Web Services (AWS) 客户开始使用工具将其本地服务器迁移到 AWS，其中最为著名的工具为CloudEndure。AWS在2019年1月收购了CloudEndure这家以色列初创公司，CloudEndure 具有一些其他公司无法提供的独特功能，包括能够将服务器从任何源平台迁移到任何目标平台。机器转换流程将为您完成所有管理程序和操作系统的配置更改。CloudEndure 是一种数据块级复制，几乎可支持所有应用程序。它使用的是连续复制，这意味着源机器上的任何更改都能在不到一秒钟的时间内复制到转储区域。这样一来，即可最大程度地减少切换期间的停机时间。

CloudEndure 是一款出色的工具，可帮助您迁移工作负载。但是如果您使用CloudEndure迁移大量服务器 – 尤其面对老旧操作系统的 CentOS/Redhat/Ubuntu等Linux迁移时，则会遇到一些挑战，例如：
1）CloudEndure调用Test Mode (测试模式)后，Target Instances (目标实例) 的EC2无法正常启动。
2）EC2实例开机后，Screenshot显示启动正常。AWS 控制台EC2实例中自检显示网卡错误，SSH无法登录。
3）自检网卡错误发生在新的EC2 Nitro实例（如C5/M5等），换成旧版本的EC2（如C4/M4等）则可以正常登录。
4）其他奇怪的问题。

**以上的问题，可以通过在源服务器****进行****Linux Kernel****升级****和在Amazon EC2 实例上安装和启用最新版的 ENA 驱动程序进行解决****，****但是这两步操作都需要****源服务器****Linux重启后才会生效****，这对源服务应用系统的正常运行造成了影响。**
**有没有办法不中断源服务器的运行，****同时能够解决以上问题？ 答案是****在****Target Instances (目标实例)进行****L****inux Kernel****升级和安装最新版ENA 驱动程序 进行解决。**

在开始之前，请确保您已满足以下条件：
    * 体验 CloudEndure 服务，安装代理以及通过控制台进行使用。了解有关如何使用 CloudEndure 的更多信息。
    * 已创建 CloudEndure 账户并购买了许可证，在CloudEndure 项目并配置了 AWS 凭证。
    * 在源机器上安装了 CloudEndure 代理，并完成了数据复制。
    * 尝试过CloudEndure 的Test Mode (测试模式)。Test Mode 是指在目标区域生成目标服务器的过程。

操作步骤：
1）请检查Linux kernel版本号，请参考CE Docs； 

2）使用旧版本的EC2（如C4/M4等）替换EC2 Nitro实例（如C5/M5等）。具体操作步骤为，进入CE控制台，在主界面Machines 一栏中点击服务器，进入 Blueprint（蓝图）界面，设置目标实例的配置信息
“Machine Type”中选择目标实例的类型，可以尝试的顺序为：T2 -&gt; C4 -&gt;M4；

3）本项目的源服务器为多年前安装CentOS，内网使用没有做过升级，其内核版本为“Linux  2.6.32-431.20.3.el6.x86_64”。经多次尝试，只有C4实例能够正常启动，且运行SSH登录。

4）登录后，运行命令如下：
//升级kernal脚本
cd /etc/yum.repos.d
ls -A | xargs -i mv {} {}.bak
wget -O ./CentOS6-Base-163.repo http://mirrors.163.com/.help/CentOS6-Base-163.repo
yum clean all
nohup yum -y update &amp;
tail -50f nohup.out


//恢复yum文件后缀名称
find ./ -name "\*.repo.bak" | awk -F "." '{print $2}' | xargs -i -t mv ./{}.repo.bak  ./{}.repo

//重启之后 看kernel版本和升级网卡
reboot
//查看Linux内核版本命令 ：
uname -a
//Linux  2.6.32-431.20.3.el6.x86_64 #1 SMP Thu Jun 19 21:14:45 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux ---&gt; 2.6.32-754

//升级网卡驱动
sudo -i 
yum -y install kernel-devel-$(uname -r) gcc git patch rpm-build wget
wget https://github.com/amzn/amzn-drivers/archive/master.zip
unzip master.zip
cd amzn-drivers-master/kernel/linux/ena
make
cp ena.ko /lib/modules/$(uname -r)/  
insmod ena.ko  
depmod
echo 'add_drivers+=" ena "' &gt;&gt; /etc/dracut.conf.d/ena.conf 
dracut -f -v      
lsinitrd /boot/initramfs-2.6.32-754.31.1.el6.x86_64.img | grep ena.ko 


//安装 DKMS：配置动态内核模块支持 (DKMS) 计划，以确保未来的内核升级期间包括驱动程序。
yum reinstall http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum -y install dkms

VER=$( grep ^VERSION /root/amzn-drivers-master/kernel/linux/rpm/Makefile | cut -d' ' -f2 )   # Detect current version
sudo cp -a /root/amzn-drivers-master /usr/src/amzn-drivers-${VER}   # Copy source into the source directory.


cat &gt; /usr/src/amzn-drivers-${VER}/dkms.conf &lt;&lt;EOM                  # Generate the dkms config file
yum -y reinstall http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum -y install dkms 

VER=$( grep ^VERSION /root/amzn-drivers-master/kernel/linux/rpm/Makefile | cut -d' ' -f2 )   
sudo cp -a /root/amzn-drivers-master /usr/src/amzn-drivers-${VER}   
cat &gt; /usr/src/amzn-drivers-${VER}/dkms.conf &lt;&lt;EOM                 
PACKAGE_NAME="ena"
PACKAGE_VERSION="$VER"
CLEAN="make -C kernel/linux/ena clean"
MAKE="make -C kernel/linux/ena/ BUILD_KERNEL=\\${kernelver}"
BUILT_MODULE_NAME[0]="ena"
BUILT_MODULE_LOCATION="kernel/linux/ena"
DEST_MODULE_LOCATION[0]="/updates"
DEST_MODULE_NAME[0]="ena"
AUTOINSTALL="yes"
EOM


dkms add -m amzn-drivers -v $VER
dkms build -m amzn-drivers -v $VER
dkms install -m amzn-drivers -v $VER


//使用 modinfo 命令确认存在 ENA 模块。
modinfo ena

//将 net.ifnames=0 附加到 /boot/grup/menu.lst 文件的内核行中
vi /boot/grub/menu.lst
add line as : net.ifnames=0 


//运行 "poweroff" 以从 SSH 终端停止实例，或使用 AWS 命令行界面 (AWS CLI) 或 Amazon EC2 控制台停止实例。
//启用实例级别的增强网络支持。以下示例从 AWS CLI 修改实例属性：
poweroff
aws ec2 modify-instance-attribute --instance-id i-096b9dcfa6e344e36 --ena-support --region cn-northwest-1


5）把EC2实例更改为Nitro实例M5，启动EC2。 网卡自检通过，可以正常的SSH登录。

6）好处：Nitro支撑的C5 实例提供了 EC2 产品系列中最佳的价格/计算性能比。与C4实例相比，其性价比提高了49% 。

为了方便调用，可以通过以下命令行在Target Instances (目标实例) 进行Linux Kernel升级和安装最新版ENA 驱动程序，如下：



CE更多问题补充：
EC2克隆后无法启动；
EC2 超过2TB的无法启动；



