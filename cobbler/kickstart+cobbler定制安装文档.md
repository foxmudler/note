#kickstart+cobbler无人值守安装

**一、简介**

**1.1 什么是PXE**

PXE(Pre-boot Execution Environment，预启动执行环境)是由Intel公司开发的最新技术，工作于Client/Server的网络模式，支持工作站通过网络从远端服务器下载映像，并由此支持通过网络启动操作系统，在启动过程中，终端要求服务器分配IP地址，再用TFTP（trivial file transfer protocol）或MTFTP(multicast trivial file transfer protocol)协议下载一个启动软件包到本机内存中执行，由这个启动软件包完成终端基本软件设置，从而引导预先安装在服务器中的终端操作系统。

严格来说，PXE 并不是一种安装方式，而是一种引导方式。进行 PXE 安装的必要条件是在要安装的计算机中必须包含一个 PXE 支持的网卡（NIC），即网卡中必须要有 PXE Client。PXE 协议可以使计算机通过网络启动。此协议分为 Client端和 Server 端，而PXE Client则在网卡的 ROM 中。当计算机引导时，BIOS 把 PXE Client 调入内存中执行，然后由 PXE Client 将放置在远端的文件通过网络下载到本地运行。运行 PXE 协议需要设置 DHCP 服务器和 TFTP 服务器。DHCP 服务器会给 PXE Client（将要安装系统的主机）分配一个 IP 地址，由于是给 PXE Client 分配 IP 地址，所以在配置 DHCP 服务器时需要增加相应的 PXE 设置。此外，在 PXE Client 的 ROM 中，已经存在了 TFTP Client，那么它就可以通过 TFTP 协议到 TFTP Server 上下载所需的文件了。

**PXE的工作过程：**

    1. PXE Client 从自己的PXE网卡启动，向本网络中的DHCP服务器索取IP；
    2. DHCP 服务器返回分配给客户机的IP 以及PXE文件的放置位置(该文件一般是放在一台TFTP服务器上) ；
    3. PXE Client 向本网络中的TFTP服务器索取pxelinux.0 文件；
    4. PXE Client 取得pxelinux.0 文件后之执行该文件；
    5. 根据pxelinux.0 的执行结果，通过TFTP服务器加载内核和文件系统 ；
    6. 进入安装画面, 此时可以通过选择HTTP、FTP、NFS 方式之一进行安装；

详细工作流程，请参考下面这幅图：

![img](http://images.cnitblog.com/blog/370046/201406/152331542808644.jpg)

**1.2 什么是Kickstart**

Kickstart是一种无人值守的安装方式。它的工作原理是在安装过程中记录典型的需要人工干预填写的各种参数，并生成一个名为ks.cfg的文件。如果在安装过程中（不只局限于生成Kickstart安装文件的机器）出现要填写参数的情况，安装程序首先会去查找Kickstart生成的文件，如果找到合适的参数，就采用所找到的参数；如果没有找到合适的参数，便需要安装者手工干预了。所以，如果Kickstart文件涵盖了安装过程中可能出现的所有需要填写的参数，那么安装者完全可以只告诉安装程序从何处取ks.cfg文件，然后就去忙自己的事情。等安装完毕，安装程序会根据ks.cfg中的设置重启系统，并结束安装。

PXE+Kickstart 无人值守安装操作系统完整过程如下：

![img](http://images.cnitblog.com/blog/370046/201406/152331551083531.jpg)
    1、PXE Client向DHCP发送请求 
    PXE Client从自己的PXE网卡启动，通过PXE BootROM(自启动芯片)会以UDP(简单用户数据报协议)发送一个广播请求，向本网络中的DHCP服务器索取IP。
    
    2、DHCP服务器提供信息 
    DHCP服务器收到客户端的请求，验证是否来至合法的PXE Client的请求，验证通过它将给客户端一个“提供”响应，这个“提供”响应中包含了为客户端分配的IP地址、pxelinux启动程序(TFTP)位置，以及配置文件所在位置。
    
    3、PXE客户端请求下载启动文件 
    客户端收到服务器的“回应”后，会回应一个帧，以请求传送启动所需文件。这些启动文件包括：pxelinux.0、pxelinux.cfg/default、vmlinuz、initrd.img等文件。
    
    4、Boot Server响应客户端请求并传送文件 
    当服务器收到客户端的请求后，他们之间之后将有更多的信息在客户端与服务器之间作应答, 用以决定启动参数。BootROM由TFTP通讯协议从Boot Server下载启动安装程序所必须的文件(pxelinux.0、pxelinux.cfg/default)。default文件下载完成后，会根据该文件中定义的引导顺序，启动Linux安装程序的引导内核。
    
    5、请求下载自动应答文件 
    客户端通过pxelinux.cfg/default文件成功的引导Linux安装内核后，安装程序首先必须确定你通过什么安装介质来安装linux，如果是通过网络安装(NFS, FTP, HTTP)，则会在这个时候初始化网络，并定位安装源位置。接着会读取default文件中指定的自动应答文件ks.cfg所在位置，根据该位置请求下载该文件。
    这里有个问题，在第2步和第5步初始化2次网络了，这是由于PXE获取的是安装用的内核以及安装程序等，而安装程序要获取的是安装系统所需的二进制包以及配置文件。因此PXE模块和安装程序是相对独立的，PXE的网络配置并不能传递给安装程序，从而进行两次获取IP地址过程，但IP地址在DHCP的租期内是一样的。
    
    6、客户端安装操作系统 
    将ks.cfg文件下载回来后，通过该文件找到OS Server，并按照该文件的配置请求下载安装过程需要的软件包。 
    OS Server和客户端建立连接后，将开始传输软件包，客户端将开始安装操作系统。安装完成后，将提示重新引导计算机。

**二、系统环境**

    实验环境:VMware Workstation 12
    系统平台:CentOS release 6.7 (最小化安装)
    网络模式:NAT模式（共享主机的IP地址）
    DHCP / TFTP IP：10.0.0.10
    HTTP / FTP / NFS IP：10.0.0.10
    防火墙已关闭/iptables: Firewall is not running.
    SELINUX=disabled

**PXE-Lickstart安装环境准备**

    1、ks文件
    2、DHCP服务器
    3、TFTP服务器
    4、HTTP服务器
    5、网卡启动
    6、YUM仓库安装源

环境准备
    [root@kickstart ~]# cat /etc/redhat-release 
    CentOS release 6.6 (Final)
    [root@kickstart ~]# uname -r
    2.6.32-504.el6.x86_64
    [root@kickstart ~]# uname -m
    x86_64
    [root@kickstart ~]# getenforce
     Permissive
    [root@kickstart ~]# /etc/init.d/iptables stop
    [root@kickstart ~]# ifconfig eth0|awk -F "[ :]+" 'NR==2 {print $4}'
    10.0.0.205
    [root@kickstart ~]# hostname
    cobbler
    [root@cobbler ~]#wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo 
    #####下载阿里云的epel源
    [root@kickstart /]  wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-6.repo
    [root@kickstart /]  yum clean all
    [root@kickstart /]  yum makecache

**安装步骤**

	[root@kickstart /] mount /dev/cdrom /mnt/
	[root@kickstart /]# yum -y install httpd createrepo
	[root@kickstart yum.repos.d]# mkdir /var/www/html/CentOS-6.7-x86_64
	[root@kickstart yum.repos.d]# cp -a /mnt/* /var/www/html/CentOS-6.7-x86_64/
	[root@kickstart ~]# yum -y install tftp-server dhcp xinetd
	[root@kickstart ~]# vim /etc/xinetd.d/tftp
	disable                 = no
	[root@kickstart ~]# cd /usr/share/doc/dhcp-4.1.1/ 
	[root@kickstart dhcp-4.1.1]# cat dhcpd.conf.sample > /etc/dhcp/dhcpd.conf
	[root@kickstart dhcp-4.1.1]# vim /etc/dhcp/dhcpd.conf
	#This declaration allows BOOTP clients to get dynamic addresses,
	# which we don't really recommend.
	   
	subnet 10.0.0.0 netmask 255.255.255.0 {
	  range dynamic-bootp 10.0.0.110 10.0.0.199;
	#  option broadcast-address 10.254.239.31;
	#  option routers rtr-239-32-1.example.org;
	   option subnet-mask 255.255.255.0;
	   next-server 10.0.0.205;
	   filename "pxelinux.0";

**启动相关服务**
    [root@kickstart dhcp-4.1.1]# /etc/init.d/dhcpd start  
    [root@kickstart dhcp-4.1.1]# /etc/init.d/httpd start
    [root@kickstart dhcp-4.1.1]# /etc/init.d/xinetd start

**将准备好的.cfg文件上传致以下目录中**
    [root@kickstart dhcp-4.1.1]# cd /var/www/html/CentOS-6.7-x86_64/
    [root@kickstart CentOS-6.7-x86_64]# rz
    [root@kickstart CentOS-6.7-x86_64]# ll | grep "CentOS-6.7-x86_64.cfg"
    -rw-r--r-- 1 root root   1094 May 28 07:01 CentOS-6.7-x86_64.cfg

**一定要记得上传之后要测试以下文件是否能被正常访问到**
    [root@demo CentOS-6.7-x86_64]# curl  -I --head http://10.0.0.10/CentOS-6.7-x86_64/CentOS-6.7-x86_64.cfg    
    HTTP/1.1 200 OK
    Date: Fri, 27 May 2016 23:12:41 GMT
    Server: Apache/2.2.15 (CentOS)
    Last-Modified: Fri, 27 May 2016 23:01:38 GMT
    ETag: "4c1375-446-533dae3a611fd"
    Accept-Ranges: bytes
    Content-Length: 1094
    Connection: close
    Content-Type: text/plain; charset=UTF-8

**创建索引文件**
    [root@kickstart yum.repos.d]# createrepo -pdo /var/www/html/CentOS-6.7-x86_64/ /var/www/html/CentOS-6.7-x86_64/

**创建kickstart启动引导文件**
    [root@kickstart yum.repos.d]# yum -y install syslinux
    [root@kickstart yum.repos.d]# cp  /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot/
    [root@kickstart yum.repos.d]# mkdir /var/lib/tftpboot/pxelinux.cfg
    [root@kickstart yum.repos.d]# cp /mnt/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
    [root@kickstart yum.repos.d]# cd /var/lib/tftpboot/pxelinux.cfg/
    [root@kickstart tftpboot]#  vim default
      label linux
      menu label ^Install kickstart system
      menu default
      kernel vmlinuz
      append initrd=initrd.img ks=http://10.0.0.10/CentOS-6.7-x86_64/CentOS-6.7-x86_64.cfg
**创建组信息**
    [root@kickstart pxelinux.cfg]# createrepo -g `ls /var/www/html/CentOS-6.7-x86_64/repodata/*-comps.xml` /var/www/html/CentOS-6.7-x86_64/

#引入cobbler

cobbler 是一款自动化操作系统的实现，与PXE无人值守安装的区别在于它可以同时部署多个版本的系统，而PXE只能选择一种系统。可以理解cobbler是kickstart的升级版，其优点就是比较容易配置，而且还自带web界面，比较容易管理。

cobbler是 一个Linux服务器安装的服务，可以通过网络启动（PXE）的方式来快速安装、重装物理服务器和虚拟机，同时还可以管理DHCP、DNS等。

cobbler内置了一个轻量级的配置管理系统，但它也支持和其他配置管理系统集成，如puppet，暂时还不支持saltstack。 



Cobbler提供以下服务集成：
* PXE服务支持
* DHCP服务管理
* DNS服务管理
* 电源管理
* Kickstart服务支持
* yum仓库管理

##cobbler的安装

    [root@kickstart ~]#  yum -y install cobbler cobbler-web dhcp tftp-server pykickstart httpd

    [root@kickstart ~]# rpm -ql cobbler     #查看安装了什么文件
##下面列出重要部分来注释
    /etc/cobbler                  #配置文件目录
    /etc/cobbler/settings         #cobbler主配置文件，这个文件是YAML格式，cobbler是Python写的程序
    /etc/cobbler/dhcp.template    #DHCP服务的配置模板
    /etc/cobbler/tftpd.template   #tftp服务的配置模板
    /etc/cobbler/rsync.template   #rsync服务的配置模板
    /etc/cobbler/iso              # iso模板配置文件目录
    /etc/cobbler/pxe              # pxe模板文件目录
    /etc/cobbler/power            # 电源的配置文件目录
    /etc/cobbler/users.conf       # Web服务授权配置文件
    /etc/cobbler/users.digest     # 用于web访问的用户名密码配置文件
    /etc/cobbler/dnsmasq.template # DNS服务的配置模板
    /etc/cobbler/modules.conf     # Cobbler模块配置文件
    
    /var/lib/cobbler              # Cobbler数据目录
    /var/lib/cobbler/config       # 配置文件
    /var/lib/cobbler/kickstarts   # 默认存放kickstart文件
    /var/lib/cobbler/loaders      # 存放的各种引导程序
    
    /var/www/cobbler              # 系统安装镜像目录
    /var/www/cobbler/ks_mirror    # 导入的系统镜像列表
    /var/www/cobbler/images       # 导入的系统镜像启动文件
    /var/www/cobbler/repo_mirror  # yum源存储目录
    
    /var/log/cobbler              # 日志目录
    /var/log/cobbler/install.log  # 客户端系统安装日志
    /var/log/cobbler/cobbler.log  # cobbler日志


##cobbler命令说明
     cobbler check：		检查cobbler配置
     cobbler list：		列出所有的cobbler元素
     cobbler report：	列出元素的详细信息
     cobbler distro: 	 查看导入的发行版系统信息
     cobbler system： 	查看添加的系统信息
     cobbler profile：	查看配置信息
     cobbler sync：	 	同步cobbler配置
     cobbler reposync：	同步yum仓库


    [root@kickstart ~]# service cobblerd start
    [root@kickstart ~]# cobbler check			#报错如以下信息
    Traceback (most recent call last):
      File "/usr/bin/cobbler", line 36, in <module>
        sys.exit(app.main())
      File "/usr/lib/python2.6/site-packages/cobbler/cli.py", line 657, in main
        rc = cli.run(sys.argv)
      File "/usr/lib/python2.6/site-packages/cobbler/cli.py", line 270, in run
        self.token         = self.remote.login("", self.shared_secret)
      File "/usr/lib64/python2.6/xmlrpclib.py", line 1199, in __call__
        return self.__send(self.__name, args)
      File "/usr/lib64/python2.6/xmlrpclib.py", line 1489, in __request
        verbose=self.__verbose
      File "/usr/lib64/python2.6/xmlrpclib.py", line 1253, in request
        return self._parse_response(h.getfile(), sock)
      File "/usr/lib64/python2.6/xmlrpclib.py", line 1392, in _parse_response
        return u.close()
      File "/usr/lib64/python2.6/xmlrpclib.py", line 838, in close
        raise Fault(**self._stack[0])
    xmlrpclib.Fault: <Fault 1: "<class 'cobbler.cexceptions.CX'>:'login failed'">

######此时只需重启cobbler服务即可
    [root@kickstart ~]# service cobblerd restart
    [root@kickstart ~]# cobbler check           
    The following are potential configuration items that you may want to fix:
    
    1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
    2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
    3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
    4 : change 'disable' to 'no' in /etc/xinetd.d/rsync
    5 : file /etc/xinetd.d/rsync does not exist
    6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
    7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
    8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
    
    Restart cobblerd and then run 'cobbler sync' to apply changes.

######检测之后又有如下报错信息。需要把所有错误全部修正之后才可以继续部署


######错误修正方法：
    [root@kickstart ~]# vim /etc/cobbler/settings  #修改cobbler配置文件
    next_server: 10.0.0.204		#指定tftp所在服务器
    server: 10.0.0.204	#主机的IP地址
    manage_dhcp: 1		#让cobbler管理DHCP
    pxe_just_once: 1	#防止循环装系统，适用于服务器第一启动项是PXE启动


######设置 新装系统的默认root密码123456。
######下面的命令提示来源于提示8. random-phrase-here为干扰码。可以自行设定。

    [root@kickstart ~]#  openssl passwd -1 -salt 'test' '123456'      
    $1$test$at615QShYKduQlx5z9Zm7/
    [root@kickstart ~]# vim /etc/cobbler/settings
    default_password_crypted: "$1$test$at615QShYKduQlx5z9Zm7/"
    
    [root@kickstart ~]# vim /etc/xinetd.d/tftp 
    disable = no
    
    [root@kickstart ~]# vim /etc/xinetd.d/rsync 
    disable = no
    
    [root@kickstart ~]# cobbler get-loaders               #自动从官网下载
######查看下载的内容
    [root@kickstart ~]# cd /var/lib/cobbler/loaders/
    [root@kickstart loaders]# ll
    total 1128
    -rw-r--r-- 1 root root    631 May 16 03:41 COPYING.elilo
    -rw-r--r-- 1 root root  18007 May 16 03:41 COPYING.syslinux
    -rw-r--r-- 1 root root    626 May 16 03:41 COPYING.yaboot
    -rw-r--r-- 1 root root 356493 May 16 03:41 elilo-ia64.efi
    -rw-r--r-- 1 root root 243679 May 16 03:41 grub-x86_64.efi
    -rw-r--r-- 1 root root 237224 May 16 03:41 grub-x86.efi
    -rw-r--r-- 1 root root  54964 May 16 03:41 menu.c32
    -rw-r--r-- 1 root root  16794 May 16 03:41 pxelinux.0
    -rw-r--r-- 1 root root   1054 May 16 03:41 README
    -rw-r--r-- 1 root root 198236 May 16 03:41 yaboot



    [root@kickstart loaders]# /etc/init.d/xinetd restart
    Stopping xinetd:                                           [  OK  ]
    Starting xinetd:                                           [  OK  ]
    [root@kickstart loaders]# /etc/init.d/cobblerd restart
    Stopping cobbler daemon:                                   [  OK  ]
    Starting cobbler daemon:                                   [  OK  ]
    [root@kickstart ~]# cobbler check	
    [root@kickstart loaders]# vim /etc/cobbler/dhcp.template 
    subnet 10.0.0.0 netmask 255.255.255.0 {
         option routers             10.0.0.2;
         option domain-name-servers 223.5.5.5;
         option subnet-mask         255.255.255.0;
         range dynamic-bootp        10.0.0.100 10.0.0.200;
         default-lease-time         21600;
         max-lease-time             43200;
         next-server                $next_server;
         class "pxeclients" {
              match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
              if option pxe-system-type = 00:02 {
                      filename "ia64/elilo.efi";
              } else if option pxe-system-type = 00:06 {
                      filename "grub/grub-x86.efi";
              } else if option pxe-system-type = 00:07 {
                      filename "grub/grub-x86_64.efi";
              } else {
                      filename "pxelinux.0";
              }
         }
    
    }

##同步最新cobbler配置，它会根据配置自动修改dhcp等服务
    [root@linux-node1 loaders]# cobbler sync         #同步所有配置，下面是具体过程
    task started: 2016-05-17_064201_sync
    task started (id=Sync, time=Tue May 17 06:42:01 2016)
    running pre-sync triggers
    cleaning trees
    removing: /var/lib/tftpboot/pxelinux.cfg/default
    removing: /var/lib/tftpboot/grub/images
    copying bootloaders
    trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
    copying: /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
    trying hardlink /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
    trying hardlink /var/lib/cobbler/loaders/yaboot -> /var/lib/tftpboot/yaboot
    trying hardlink /usr/share/syslinux/memdisk -> /var/lib/tftpboot/memdisk
    trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
    trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
    copying distros to tftpboot
    copying images
    generating PXE configuration files
    generating PXE menu structure
    rendering DHCP files
    generating /etc/dhcp/dhcpd.conf
    rendering TFTPD files
    generating /etc/xinetd.d/tftp
    cleaning link caches
    rendering Rsync files
    running post-sync triggers
    running python triggers from /var/lib/cobbler/triggers/sync/post/*
    running python trigger cobbler.modules.sync_post_restart_services
    running: dhcpd -t -q
    received on stdout: 
    received on stderr: 
    running: service dhcpd restart
    received on stdout: Shutting down dhcpd: [  OK  ]
    Starting dhcpd: [  OK  ]
    
    received on stderr: 
    running shell triggers from /var/lib/cobbler/triggers/sync/post/*
    running python triggers from /var/lib/cobbler/triggers/change/*
    running python trigger cobbler.modules.scm_track
    running shell triggers from /var/lib/cobbler/triggers/change/*
    *** TASK COMPLETE ***
##挂载centOS7系统镜像
    [root@linux-node1 ~]# mount /dev/cdrom /mnt       
    mount: block device /dev/sr0 is write-protected, mounting read-only
##导入系统镜像
    [root@kickstart ~]# cobbler import --path=/mnt/ --name=CentOS-6.7-x86_64 --arch=x86_64
    task started: 2016-05-17_064931_import
    task started (id=Media import, time=Tue May 17 06:49:31 2016)
    Found a candidate signature: breed=redhat, version=rhel6
    Found a matching signature: breed=redhat, version=rhel6
    Adding distros from path /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64:
    creating new distro: CentOS-6.7-x86_64
    trying symlink: /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64 -> /var/www/cobbler/links/CentOS-6.7-x86_64
    creating new profile: CentOS-6.7-x86_64
    associating repos
    checking for rsync repo(s)
    checking for rhn repo(s)
    checking for yum repo(s)
    starting descent into /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64 for CentOS-6.7-x86_64
    processing repo at : /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64
    need to process repo/comps: /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64
    looking for /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64/repodata/*comps*.xml
    Keeping repodata as-is :/var/www/cobbler/ks_mirror/CentOS-6.7-x86_64/repodata
    *** TASK COMPLETE ***
     --path    镜像路径
     --name   为安装源定义一个名字
     --arch   指定安装源是32位、64位、ia64，目前支持的选项有:x86 |x86_64|ia64
    安装源的唯一标示就是根据name参数来定义，本例导入成功后，
    安装源的唯一标示就是： centOS-7.1-x86_64，如果重复，系统会提示导入失败。
    
    导入的时候时间会有点长----------------等待中----------------------------经计算需要4分钟。

##查看镜像
    [root@linux-node1 ~]# cobbler distro list
    CentOS-6.7-x86_64
######镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror 下的CentOS-6.7-x86_64目录下。因此/var/www/cobbler目录必须具有足够空间来容纳安装文件。
    [root@linux-node1 ~]# cd /var/www/cobbler/ks_mirror/
    [root@linux-node1 ks_mirror]# ls
    CentOS-7.1-x86_64  config
    [root@linux-node1 ks_mirror]# ll CentOS-6.7-x86_64/
    总用量 312
    -rw-r--r-- 1 root root     16 4月   1 2015 CentOS_BuildTag
    drwxr-xr-x 3 root root   4096 3月  28 2015 EFI
    -rw-r--r-- 1 root root    215 3月  28 2015 EULA
    -rw-r--r-- 1 root root  18009 3月  28 2015 GPL
    drwxr-xr-x 3 root root   4096 3月  28 2015 images
    drwxr-xr-x 2 root root   4096 3月  28 2015 isolinux
    drwxr-xr-x 2 root root   4096 3月  28 2015 LiveOS
    drwxr-xr-x 2 root root 258048 4月   1 2015 Packages
    drwxr-xr-x 2 root root   4096 4月   1 2015 repodata
    -rw-r--r-- 1 root root   1690 3月  28 2015 RPM-GPG-KEY-CentOS-7
    -rw-r--r-- 1 root root   1690 3月  28 2015 RPM-GPG-KEY-CentOS-Testing-7
    -r--r--r-- 1 root root   2883 4月   1 2015 TRANS.TBL

##指定ks.cfg文件及调整内核参数
    cobbler的ks.cfg文件存放位置
    [root@linux-node1 ~]# cd /var/lib/cobbler/kickstarts/
    [root@linux-node1 kickstarts]# ls
    default.ks    esxi5-ks.cfg      legacy.ks     sample_autoyast.xml  sample_esx4.ks   sample_esxi5.ks  sample_old.seed
    esxi4-ks.cfg  install_profiles  pxerescue.ks  sample_end.ks（默认使用的ks文件）        sample_esxi4.ks  sample.ks        sample.seed
##上传准备好的ks文件
    [root@linux-node1 kickstarts]# rz -E
    rz waiting to receive.
    [root@linux-node1 kickstarts]# ll -lrt
    -rw-r--r-- 1 root root 1450 11月 28 12:09 CentOS-6.7-x86_64.cfg
    [root@linux-node1 kickstarts]# cat CentOS-6.7-x86_64.cfg
    #Kickstart Configurator by Jason Zhao
    #platform=x86, AMD64, or Intel EM64T
    #System  language
    lang en_US
    #System keyboard
    keyboard us
    #Sytem timezone
    timezone Asia/Shanghai
    #Root password
    rootpw --iscrypted $default_password_crypted
    #rootpw --iscrypted $1$ops-node$7hqdpgEmIE7Z0RbtQkxW20
    #Use text mode install
    text
    #Install OS instead of upgrade
    install
    #Use NFS installation Media
    url --url=$tree
    #url --url=http://192.168.56.11/CentOS-6.7-x86_64
    #System bootloader configuration
    bootloader --location=mbr
    #Clear the Master Boot Record
    zerombr
    #Partition clearing information
    clearpart --all --initlabel 
    #Disk partitioning information
    part /boot --fstype xfs --size 1024 --ondisk sda
    part swap --size 16384 --ondisk sda
    part / --fstype xfs --size 1 --grow --ondisk sda
    #System authorization infomation
    auth  --useshadow  --enablemd5 
    #Network information
    $SNIPPET('network_config')
    #network --bootproto=dhcp --device=eth0 --onboot=on
    # Reboot after installation
    reboot
    #Firewall configuration
    firewall --disabled 
    #SELinux configuration
    selinux --disabled
    #Do not configure XWindows
    skipx
    %pre
    $SNIPPET('log_ks_pre')
    $SNIPPET('kickstart_start')
    $SNIPPET('pre_install_network_config')
    # Enable installation monitoring
    $SNIPPET('pre_anamon')
    %end
    #Package install information
    %packages
    @ base
    @ core
    sysstat
    iptraf
    ntp
    lrzsz
    ncurses-devel
    openssl-devel
    zlib-devel
    OpenIPMI-tools
    mysql
    nmap
    screen
    %end
    %post
    systemctl disable postfix.service
    %end

 在第一次导入系统镜像后，cobbler会给镜像指定一个默认的kickstart自动安装文件在/var/lib/cobbler/kickstarts下的sample_end.ks。
##查看镜像信息
    [root@kickstart kickstarts]# cobbler list
    distros:
       CentOS-6.7-x86_64
    
    profiles:
       CentOS-6.7-x86_64
    
    systems:
    
    repos:
    
    images:
    
    mgmtclasses:
    
    packages:
    
    files:

##查看安装镜像文件信息

    [root@kickstart kickstarts]#  cobbler distro report --name=CentOS-6.7-x86_64                                                            
    Name                           : CentOS-6.7-x86_64
    Architecture                   : x86_64
    TFTP Boot Files                : {}
    Breed                          : redhat
    Comment                        : 
    Fetchable Files                : {}
    Initrd                         : /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64/images/pxeboot/initrd.img
    Kernel                         : /var/www/cobbler/ks_mirror/CentOS-6.7-x86_64/images/pxeboot/vmlinuz
    Kernel Options                 : {}
    Kernel Options (Post Install)  : {}
    Kickstart Metadata             : {'tree': 'http://@@http_server@@/cblr/links/CentOS-6.7-x86_64'}
    Management Classes             : []
    OS Version                     : rhel6
    Owners                         : ['admin']
    Red Hat Management Key         : <<inherit>>
    Red Hat Management Server      : <<inherit>>
    Template Files                 : {}

##查看所有的profile设置

    [root@kickstart kickstarts]# cobbler profile report --name=CentOS-6.7-x86_64
    Name                           : CentOS-6.7-x86_64
    TFTP Boot Files                : {}
    Comment                        : 
    DHCP Tag                       : default
    Distribution                   : CentOS-6.7-x86_64
    Enable gPXE?                   : 0
    Enable PXE Menu?               : 1
    Fetchable Files                : {}
    Kernel Options                 : {}
    Kernel Options (Post Install)  : {}
    Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-6.7-x86_64.cfg
    Kickstart Metadata             : {}
    Management Classes             : []
    Management Parameters          : <<inherit>>
    Name Servers                   : []
    Name Servers Search Path       : []
    Owners                         : ['admin']
    Parent Profile                 : 
    Internal proxy                 : 
    Red Hat Management Key         : <<inherit>>
    Red Hat Management Server      : <<inherit>>
    Repos                          : []
    Server Override                : <<inherit>>
    Template Files                 : {}
    Virt Auto Boot                 : 1
    Virt Bridge                    : xenbr0
    Virt CPUs                      : 1
    Virt Disk Driver Type          : raw
    Virt File Size(GB)             : 5
    Virt Path                      : 
    Virt RAM (MB)                  : 512
    Virt Type                      : kvm

##编辑profile，修改关联的ks 文件

    [root@linux-node1 ~]# cobbler profile edit --name=CentOS-6.7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-6.7-x86_64.cfg
##修改安装系统的内核参数。
######在centos7系统有一个地方变了，就是网卡名变为eno16777736这种形式，但是为了运维标准化，我们需要将它变成我们常用的eth0，因此使用下面的参数。但是注意是centOS7才需要下面的步骤，centOS6不需要！！

    [root@linux-node1 ~]#  cobbler profile edit --name=CentOS-7.1-x86_64 --kopts='net.ifnames=0 biosdevname=0'
    [root@linux-node1 ~]# cobbler profile report CentOS-7.1-x86_64
    Name                           : CentOS-7.1-x86_64
    TFTP Boot Files                : {}
    Comment                        : 
    DHCP Tag                       : default
    Distribution                   : CentOS-7.1-x86_64
    Enable gPXE?                   : 0
    Enable PXE Menu?               : 1
    Fetchable Files                : {}
    Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}
    Kernel Options (Post Install)  : {}
    Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-7.1-x86_64.cfg
    Kickstart Metadata             : {}
    Management Classes             : []
    Management Parameters          : <<inherit>>
    Name Servers                   : []
    Name Servers Search Path       : []
    Owners                         : ['admin']
    Parent Profile                 : 
    Internal proxy                 : 
    Red Hat Management Key         : <<inherit>>
    Red Hat Management Server      : <<inherit>>
    Repos                          : []
    Server Override                : <<inherit>>
    Template Files                 : {}
    Virt Auto Boot                 : 1
    Virt Bridge                    : xenbr0
    Virt CPUs                      : 1
    Virt Disk Driver Type          : raw
    Virt File Size(GB)             : 5
    Virt Path                      : 
    Virt RAM (MB)                  : 512
    Virt Type                      : kvm
    #每次修改完以后要同步一次


##添加节点：
    cobbler system  add --name=node1 \
    --mac=00:50:56:36:A6:97 \
    --profile=CentOS-6.7-x86_64 \
    --ip-address=10.0.0.114 \
    --subnet=255.255.255.0 \
    --gateway=10.0.0.2 \
    --interface=eth0 \
    --static=1	\
    --hostname=node1.example.com \
    --name-servers="223.5.5.5 114.114.114.114" \
    --kickstart=/var/lib/cobbler/kickstarts/CentOS-6.7-x86_64.cfg





（要再需要重装的系统上去执行！！！！）
##自动重装系统
    [root@bogon ~]# yum -y install cobbler  koan
##查看可以重装的系统类型
    [root@bogon ~]#koan --server=192.168.56.11 --list=profiles
    - looking for Cobbler at http://192.168.56.11:80/cobbler_api
##指定重装系统类型。
    [root@bogon ~]# koan --replace-self --server=192.168.56.11 --profile=CentOS-6-x86_64
##重启系统即可重装系统
    [root@bogon ~]# reboot 


##修改cobbler的提示界面

    [root@linux-node1 ~]# vim
    /etc/cobbler/pxe/pxedefault.template 
    MENU TITLE Cobbler | http://www.qinnianzx.com
##修改配置必须同步
	[root@linux-node1 ~]# cobbler sync


##WEB页面访问地址
    http://10.0.0.204/cobbler_web
    默认登陆账号密码都是cobbler
    etc/cobbler/users.conf 	    #web服务授权配置文件
    /etc/cobbler/users.digest   #用于web访问的用户名密码配置文件
    
    [root@kickstart ~]# cat /etc/cobbler/users.digest 
    cobbler:Cobbler:a2d6bae81669d707b72c0bd9806e01f3
    #设置cobbler			 web用户登录密码

##在cobbler组添加cobbler用户，提示输入2次密码确定

    [root@kickstart ~]#  htdigest /etc/cobbler/users.digest "Cobbler" cobbler
    Changing password for user cobbler in realm Cobbler
    New password: 
    Re-type new password: 
    [root@kickstart ~]# cobbler sync     同步
##重启服务httpd

    [root@kickstart ~]#/etc/init.d/httpd restart
    [root@kickstart ~]# /etc/init.d/cobblerd restart
    这样密码就修改为123456了


##构建私有yum仓库
		以openstack源为例
1.添加repo
    cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum

2.同步repo
    [root@linux-node1 ~]# cobbler reposync

3.添加repo到对应的profile
     cobbler profile edit --name=Centos-7-x86_64  --repos=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ 

4.修改kickstart文件。添加。（些到%post %end中间）
    %post
    systemctl disable postfix.service
    
    $yum_config_stanza
    %end

5.添加定时任务，定期同步repo


