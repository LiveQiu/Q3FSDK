# 串口调试
硬件设备包括开发板、配套电源、普通串口线或者是usb转串口线一根、PC一台。

串口线连接方式：绿线接RX，白线接TX，黑线接GND,开发板上有对应的RX/TX/GND标注。

根据系统的区别，常用的串口调试工具也不一样，下文主要以minicom（ubuntu）和securecrt（window）为例来介绍调试工具的配置和使用。

## minicom
在 ubuntu 系统中我们推荐使用 minicom 作为调试工具。目前我们使用的 minicom 大多是 minicom version 2.7 或以上版本。进入操作系统后打开终端,输入以下命令安装minicom 工具：

`$ sudo apt-get install minicom`

具体配置流程如下：

1. 在安装完minicom工具后，在终端输入$sudo minicom -s进入minicom的配置界面。

2. 通过方向键的上下键来选定配置端口信息(Serial port setup)，
      
```      
            +-----[configuration]------+
            | Filenames and paths      |
            | File transfer protocols  |
            | Serial port setup        |
            | Modem and dialing        |
            | Screen and keyboard      |
            | Save setup as dfl        |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+
```

3. 选定后回车进入该项配置，分别配置USB串口的设备节点、波特率、bit位等，波特率设置为115200Bd。

```
    +-----------------------------------------------------------------------+
    | A -    Serial Device      : /dev/ttyUSB0                              |
    | B - Lockfile Location     : /var/lock                                 |
    | C -   Callin Program      :                                           |
    | D -  Callout Program      :                                           |
    | E -    Bps/Par/Bits       : 115200 8N1                                |
    | F - Hardware Flow Control : Yes                                       |
    | G - Software Flow Control : No                                        |
    |                                                                       |
    |    Change which setting?                                              |
    +-----------------------------------------------------------------------+
            | Screen and keyboard      |
            | Save setup as dfl        |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+
```

Serial Device根据实际情况配置，可以通过命令$ls /dev/ttyUSB*来查看连接的USB串口线的节点信息。

4. 配置完成后回车以保存配置。
为避免每次进入串口都需要重新配置，可选择在配置完成后选择Save setup as.. 并输入配置名（例如USB0）。保存完成后下次进入minicom时直接输入$sudo minicom USB0 即可。

```
            +-----[configuration]------+                                     
            | Filenames and paths      |                          +-----------------------------------------+
            | File transfer protocols  |                          |Give name to save this configuration?    |
            | Serial port setup        |                          |>USB0                                    |
            | Modem and dialing        |                          +-----------------------------------------+
            | Screen and keyboard      |
            | Save setup as USB0       |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+
```

5. 最后选择Exit,进入串口。

## securecrt
在Windows中我们一般使用securecrt作为我们的调试工具,可以直接从网上搜索并下载该工具。该软件是绿色版,下载完成后直接解压即可使用。

Windows下的具体配置流程如下：

1. 将开发板通过串口线连接至PC。

2. 打开PC端设备管理器，在端口一栏中找到其他设备中的带叹号设备，如图所示。（如果驱动已正常安装可直接跳至第7步）

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fsecurecrt1.JPG)

3. 通过查找串口线上的芯片，如这次串口线上的芯片为ft232，在网上找到对应的驱动，并下载解压。

4. 右键点击USB Serial Port，选择“更新驱动程序软件”，点击“浏览计算机以查找驱动程序软件”，并选择之前解压得到的文件夹，如图所示。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fsecurecrt2.JPG)

5. 点击下一步后，等待安装结束；

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fsecurecrt3.JPG)

6. 重新拔插串口线（或选择扫描设备硬件改动），刷新设备管理器，如图所示，该串口已正常工作；

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fsecurecrt4.JPG)

7. 成功安装securecrt后运行软件，在串口设置界面添加串口的参数设置，具体参数如图所示：

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fsecurecrt5.JPG)

8. 至此，串口已能正常使用了。

# 调试方法
SDK提供了多种调试方法包括IUS Card、Boot Card、TFTP等，下面简单介绍一下各种调试方式的搭建或操作方法。

## IUS Card
该种调试方法是将编译完成的所有镜像文件制作到一张SD卡中，然后通过该SD卡完成镜像烧录至设备端。

具体的制作方法请查看有关烧录卡制作的部分。

该种调试方法优缺点说明如下。

优点：操作简单，linux电脑和windows电脑都可操作。

缺点：需要烧录，等待时间长。

## Boot Card
该种调试方法是将编译完成的系统制作在SD卡内，设备在插入该SD卡的条件下启动时，将会直接运行SD卡内的系统内容。目前该调试方法仅适用于linux系统，具体操作命令如下：

`. tools/gendisk.sh /dev/sdb`

该种调试方法优缺点说明如下。

优点：插上SD卡启动就可以调试，无需烧录，方便快捷。

缺点：Windows系统不支持该调试方式。

## TFTP
TFTP（简单文件传输协议）是TCP/IP协议族中的一个用来在客户机与服务器之间进行简单文件传输的协议，提供不复杂、开销不大的文件传输服务，它基于UDP协议而实现。

在使用该调试方法前，我们需要去搭建TFTP服务器。具体搭建流程如下：

* 在ubuntu中安装tftp-hpa、tftpd-hpa、xinetd。
* 创建文件夹/tftpboot  （这个是服务器的文件交换目录，将来客户机获取服务器文件时就是从这个文件夹中获取的，目录可自己定义），并且修改这个文件夹的权限为777
* 修改tftp配置文件，如果没有就创建。并添加以下内容

```
service tftp
{
disable = no
socket_type = dgram
protocol   = udp
wait       = yes
user       = root
server     = /usr/sbin/in.tftpd
server_args = -s /tftpboot //此处文件目录就是文件交换目录
source     = 11
cps        = 100 2
flags       = IPv4
}
```

* 修改inetd.conf文件，在最后添加下面内容：

`tftp dgram udp wait nobody /usr/sbin/tcpd /usr/sbin/in.tftpd   /tftpboot	 //此处文件目录就是上文件交换目录`

* 修改tftpd-hpa文件，添加以下内容：

```
#RUN_DAEMON="no"
#OPTIONS="-s /home/zyp/tftpboot -c -p -U tftpd"
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/tftpboot"      //此处文件目录就是文件交换目录 
TFTP_ADDRESS="0.0.0.0:69"
TFTP_OPTIONS="-l -c -s"
```

* 重启服务，命令如下：

```
# service tftpd-hpa restart
# sudo /etc/init.d/xinetd reload
# sudo /etc/init.d/xinetd restart
```

至此TFTP的搭建流程就完成了，该调试方法的优缺点说明如下。

优点：设备没有tf卡时，调试方便。

缺点：需要以太网，并且纯手动操作，要记住一堆命令。

# 上电启动过程的串口打印消息
设备上电后，我们可以通过串口在PC终端打印出来的debug信息来了解设备是否正常运行信息。以下是系统启动过程中的基本打印信息及其含义：

|Debug消息	|说明|
|--------------------|---------------------|
spl: boot normal 	|uboot0 正常启动打印
spl: b kernel	|uboot0 启动 kernel 后跳转到kernel
U-Boot 2009.08 (12月 12 2016 - 15:17:31)	|uboot1 正常启动信息
Kernel command line: console=ttyAMA3, 115200 mem=62M mode=ius	|升级模式 kernel cmdline
upgrade start	|开始烧录或升级镜像
upgrade sucess time is 75s	|烧录或升级完成
Kernel command line: console=ttyAMA3, 115200 mem=62M rootfstype=squashfs root=/dev/spiblock1 rw 	|正常从spi norfalsh 启动 kernel cmdline
VFS: Mounted root (squashfs filesystem) readonly on device 251:1	|root 文件系统挂载成功
Starting logging: OK	|执行 init.d 的启动脚本
Videobox started	|videobox 启动完成