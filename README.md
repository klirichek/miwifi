小米路由核心系统支持包
===

现有的核心包支持小米路由（型号R1D，即带1T硬盘版本）。

核心包含有小米路由器的内核源码（内核版本2.6.36），主SoC厂商Broadcom的官方工具链
和一个在原ROM基础上叠加了 Debian 核心系统的混合系统 (ROOT FS) 


概览
---

编译、TFTP部署、NFS文件系统部署视频演示： http://v.youku.com/v_show/id_XNzQ4OTU2Njg4.html

启动及核心功能视频演示： http://v.youku.com/v_show/id_XNzQ4OTkzNDgw.html

小米路由的无线部分，2.5G和5G用的芯片和华硕 ASUS RT-AC56U 一致，处理器部分也和
RT-AC56U 相似，因此核心包的内核是基于 RT-AC56U 的官方开源版本开发，后期发现其
实博通(Broadcom) 的这个系列的WiFi芯片给的Linux驱动都一样，业界惯例，都没有源码，
只有预编译好的二进制文件


WiFi驱动编译后是名为 wl.ko 的模块，位于：kernel/linux-2.6.36/drivers/net/wl/ 下，
这个与博通(Broadcom)官方SDK一致，应该也与小米拿到的一致，最多有些小Bug的fix

BCM4709 带的以太网，其 Linux 驱动是有源码的，位于：

  kernel/linux-2.6.36/drivers/net/et/ 下，这个也与博通(Broadcom)官方SDK一致


更多信息，可参考这个页面： [小米路由核心分析](http://wiki.jackslab.org/%E5%B0%8F%E7%B1%B3%E8%B7%AF%E7%94%B1%E6%94%B9%E7%A7%81%E6%9C%89%E4%BA%91)

整个系统除支持原ROM的核心功能外，自带 Debian 核心环境，可用apt-get install 直接
安装你想要的软件包，相当方便

核心包编译出的 rootfs 可直接部署在路由内硬盘的第四个分区上，亦可部署在 NFS server
上


尝试使用指南
===

初次尝试建议使用 tftp 加载内核，挂载 NFS 文件系统，内核和rootfs都在PC上，不会改变原路由的结构

待得测试、调试稳定后，再将内核裁减后部署到flash.os分区上（可在miwifi目录下执行 'make nonfs' 即可生成裁减后的内核），NFS文件系统打包后，部署到 /dev/sda4 上，详细参考本页的“注意事项”

以下在 Ubuntu 12.04 验证通过 


源码下载编译
---

    $ sudo apt-get install lzma
    $ git clone git://github.com/comcat/miwifi.git
    $ cd miwifi/
    $ make

vmlinuz 为可直接被 CFE 加载的内核镜像（内含ramfs）

jarvis-rootfs.tgz 为 rootfs 包。用户 'root' 的默认密码为 'admin'，WiFi 的SSID 为 Jarvis 和 Jarvis_5G，密码为 'qwer1234'


配置网络
---

我们在PC上部署 tftp server 和 NFS server，因此需要一根网线连接路由WAN口边上的LAN口到PC网口连接的HUB上，保证PC和路由在局域网内能通信

小米路由启动时，CFE (bootloader) 会检测 nvram 中的 flag_tftp_bootup 这个参数，如果其值为 on，其会首先尝试tftp加载 192.168.1.2:/vmlinuz 这个路径。因此我们要为 PC 配一个 192.168.1.2 的IP： 

    $ sudo ifconfig eth0:0 192.168.1.2

另，路由启动后，其为LAN口配的地址为 192.168.31.1 ，为保证PC能访问其 web 配置界面，也为 PC 配一个 192.168.31.2 的IP：

    $ sudo ifconfig eth0:1 192.168.31.2


部署内核和文件系统
---

x. 安装配置 tftp 服务器 

    $ sudo apt-get install tftpd tftp
    $ cat /etc/xinetd.d/tftp
    service tftp
    {
        socket_type = dgram
        protocol = udp
        wait = yes
        user = root
        server = /usr/sbin/in.tftpd
        server_args = -s /tftpboot
        disable = no
        per_source = 11
        cps = 100 2
        flags = IPv4
    }
     
    $ sudo mkdir /tftpboot
    $ sudo chmod 777 /tftpboot
    $ sudo /etc/init.d/xinetd restart


到此 tftp Server 安装成功，把编译好的内核 vmlinuz 拷到 /tftpboot 目录下： 

    $ cp /path/to/miwifi/vmlinuz /tftpboot/


用 tftp 测试 tftp server 是否正常： 

    $ tftp
    > connect 192.168.1.2
    > get vmlinuz           # 获取tftp server 上的根目录下的 vmlinuz 文件


x. 安装配置 nfs 服务器

    $ sudo apt-get install nfs-kernel-server
     
    Config the nfs directory:
     
    $ sudo mkdir -p /tftpboot/rootfs
    $ cat /etc/exports
    /tftpboot/rootfs    *(async,rw,insecure,insecure_locks,no_root_squash)
     
    $ sudo /etc/init.d/nfs-kernel-server restart


验证 NFS Server: 

    $ sudo mount -t nfs 192.168.31.2:/tftpboot/rootfs /mnt


设置nvram变量
---

PC 上 SSH 登录已经打开SSH服务的小米路由：

    comcat@jackslab:$ ssh root@192.168.31.1
    root@192.168.31.1's password:
     
     
    BusyBox v1.19.4 (2014-04-26 02:37:48 CST) built-in shell (ash)
    Enter 'help' for a list of built-in commands.
     
     -----------------------------------------------------
        Welcome to XiaoQiang!
     -----------------------------------------------------
    root@XiaoQiang:/# nvram get flag_tftp_bootup
    off
    root@XiaoQiang:/# nvram set flag_tftp_bootup=on
     
    root@XiaoQiang:/# nvram set rootfs=nfs              # 指示内核从NFS启动
    root@XiaoQiang:/# nvram commit
     
    root@XiaoQiang:/# reboot


加载内核启动
---

    CFE version v1.0.4
    BSP: 6.37.14.34 (r415984) based on BBP 1.0.37 for BCM947XX (32bit,SP,)
    Build Date: Wed Apr 30 18:03:21 CST 2014 (szy@shenzhiyong-ct)
      
    ...........
      
    Device eth0:  hwaddr 8C-BE-BE-20-B7-48, ipaddr 192.168.1.1, mask 255.255.255.0
            gateway not set, nameserver not set
    ********** flag_tftp_bootup=on **********
    tftp network: ifconfig eth0 -addr=192.168.1.1 -mask=255.255.255.0 -gw=192.168.1.1
    Device eth0:  hwaddr 8C-BE-BE-20-B7-48, ipaddr 192.168.1.1, mask 255.255.255.0
            gateway 192.168.1.1, nameserver not set
    kernel: boot -raw -z -addr=0x8000 -max=0x800000 192.168.1.2:vmlinuz
    Loader:raw Filesys:tftp Dev:eth0 File:192.168.1.2:vmlinuz Options:(null)
    Loading: ......... 6057024 bytes read
    Entry at 0x00008000
    Closing network.
    Starting program at 0x00008000
    Linux version 2.6.36.4jackslab (comcat@jackslab) (gcc version 4.5.3 (Buildroot 2012.02)
    ) #1 SMP PREEMPT Wed Jul 30 19:44:09 CST 2014
    CPU: ARMv7 Processor [413fc090] revision 0 (ARMv7), cr=10c53c7f
    CPU: VIPT nonaliasing data cache, VIPT nonaliasing instruction cache
    Machine: Northstar Prototype
    ......
    ......
    ......
    ......


刷机指南
===

TFTP + NFS 调试系统完成后，系统稳定的话，就可以将内核部署在 /dev/mtd1 (os) 或者 /dev/mtd2 (os1) 上

指示 CFE 启动时是加载 /dev/mtd1 (flash.os1) 上的内核还是 /dev/mtd2 (flash.os2) 上的内核，可以通过 nvram 变量 flag_last_success 来完成

文件系统则部署在 /dev/sda4 上，内核启动时检测 nvram 中 rootfs 这个参数，如其值为 'sata' 则挂载 /dev/sda4 作为文件系统，如果你想用 u 盘的话，记得分四个区，把系统解压到 /dev/sda4 上，或者干脆修改内核目录下 usr/ramfs/init 脚本


以下在 Ubuntu 12.04 验证通过 


源码下载编译
---

    $ sudo apt-get install lzma
    $ git clone git://github.com/comcat/miwifi.git
    $ cd miwifi/
    $ make nonfs

vmlinuz.trx 为可直接被 CFE 从flash加载的内核镜像（内含ramfs），大小小于 3MB，可以放入 mtd1/mtd2，这两个分区的大小为 3MB

jarvis-rootfs.tgz 为 rootfs 包。用户 'root' 的默认密码为 'admin'，WiFi 的SSID 为 Jarvis 和 Jarvis_5G，密码为 'qwer1234'


部署内核和文件系统
---

如果是在原官方系统里部署，可以先将内核和文件系统 scp 拷贝的 /userdisk 下，这个目录默认挂载的是 /dev/sda4 是1T硬盘里最大的分区，用于用户下载等各种数据的存储

    $ scp vmlinuz.trx root@192.168.31.1:/userdisk/
    root@192.168.31.1's password:
    vmlinuz
     
    $ scp jarvis-rootfs.tgz root@192.168.31.1:/userdisk/
    root@192.168.31.1's password:
    jarvis-rootfs.tgz             


PC 上 SSH 登录已经打开SSH服务的小米路由后，部署文件系统： 

    root@XiaoQiang:/# mount | grep userdisk
    /dev/sda4 on /userdisk type ext4 (rw,noatime,barrier=1,data=ordered)
     
    root@XiaoQiang:/# tar zxpf /userdisk/jarvis-rootfs.tgz -C /userdisk/


部署内核，此例中我们把自己的内核部署在 /dev/mtd2 上： 

    root@XiaoQiang:/# cat /proc/mtd
    dev:    size   erasesize  name
    mtd0: 00040000 00010000 "boot"
    mtd1: 00300000 00010000 "os"
    mtd2: 00300000 00010000 "os1"
    mtd3: 00890000 00010000 "squashfs"
    mtd4: 00010000 00010000 "crash"
    mtd5: 00100000 00010000 "overlay"
    mtd6: 00010000 00010000 "board_data"
    mtd7: 00010000 00010000 "nvram"
    mtd8: 00fe0000 00010000 "firmware"
    root@XiaoQiang:/# mtd write /userdisk/vmlinuz.trx os1
    Unlocking os1 ...
     
    Writing from /userdisk/vmlinuz.trx to os1 ...     


设置nvram变量
---

nvram 变量 flag_last_success 为 0，从 flash.os1 启动；为 1，则从 flash.os2 启动 

    root@XiaoQiang:/# nvram set flag_tftp_bootup=off          # 先把 tftp 启动关了
     
    root@XiaoQiang:/# nvram get flag_last_success
    0
    root@XiaoQiang:/# nvram set flag_last_success=1           # 从 flash.os2 (/dev/mtd2) 启动
    root@XiaoQiang:/# nvram set rootfs=sata
    root@XiaoQiang:/# nvram commit


注意事项
===

核心包自带的文件系统（ROOTFS）的目的，主要是为了验证内核的各个子系统和驱动是否工
作正常，主要是SATA硬盘、WiFi、Ethernet、Flash等，故而系统启动后，自动启动的服务
极少，保证WiFi能联外网，局域网能工作，Web控制界面能登陆即可。其中 fw3 服务启动
时加载原有规则老失败，这应该是个Bug，现有的解决方案是直接写 iptables 的规则，绕
过去了，这个以后应该会解决。

初次使用的朋友，建议先按上面的步骤用 tftp 和 nfs 加载一下，看是否能工作，然后再
自己调整系统服务和安装自己要用的软件包，反正所有的操作都在 nfs 上，不会改变原有
的系统，待系统稳定后再把NFS的目录打个tar包部署在 /dev/sda4 上不迟

现在核心包默认使用NFS方式，这个造成了编译后生成的 vmlinuz 内核镜像超过了 3MB，如
果您尝试稳定后，打算直接把内核部署到flash.os/flash.os2上，可将
 kernel/linux-2.6.36/usr/ramfs/usr/bin/busybox 删除，再将内核配置里的NFS关闭，即
可将内核镜像的大小控制在2.7MB左右

当然您要是让内核从 /dev/sda4 启动，记得设置 nvram： 

    # nvram set flag_tftp_bootup=off
    # nvram set rootfs=sata
    # nvram commit
