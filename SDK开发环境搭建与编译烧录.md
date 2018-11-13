# 开发环境搭建
SDK需要在Linux系统上进行编译，Linux系统种类繁多，我们推荐用户使用以下系统执行SDK的编译：Ubuntu 16.04。开发者需要基本的使用ubuntu系统开发的经验。

在编译前需要安装编译所需的工具和库文件。在准备好基本的PC开发系统后，更新系统软件库：

```
sudo apt-get update
sudo apt-get upgrade
```

之后采用apt-get install来安装以下需要的工具和库文件。

`sudo apt-get install make automake autoconf gcc g++ python curl lzop perl build-essential libncurses5 libssl-dev libncursesw5-dev libncurses5-dev lsb ia32-libs`

在完成了依赖工具和库文件的安装后，就可以开始SDK的开发与编译了。

# SDK配置和编译
开发人员拿到SDK package源码包后，可以通过以下命令来对源码包进行解压操作：

`tar zxvf SDK_XXXX.tgz`

## SDK结构
解压后的SDK基本的目录结构说明如下：

```
QSDK release 目录结构如下：
    |-- tools                       # 存放单核媒体处理平台的目录
    |   |-- gendisk.sh              # 组件源代码
    |   |-- mkburn.sh               # 组件源代码
    |   |-- qsdk.init               # 组件源代码
    |   |-- qsdk.clean              # SDK清理脚本
    |   |-- qsdk.env                # QSDK环境变量脚本
    |   |-- ....
    |-- busybox                     # busybox源代码
    |-- kernel                      # kernal源代码
    |-- toolchain                   # 交叉编译链
    |-- uboot                       # uboot源代码
    |   |-- uboot0                  # uboot0 源代码
    |   |-- uboot1                  # uboot1 源代码
    |-- qsdk                        # qsdk
    |   |-- ramdisk                 # qsdk生成ramdisk所需的部分文件
    |   |-- include                 # qsdk编译所需头文件
    |   |-- bin                     # qsdk 所需的预编译二进制文件
    |   |-- etc                     # 部分二进制程序所需的配置文件
    |   |-- usr                     # qsdk各种库和二进制文件
    |   |   |-- bin                 # 二进制应用程序
    |   |   |-- lib                 # 动态库和静态库
    |-- private                     # 编译好的内核驱动等文件
    |   |-- Felix.ko                # 编译好的第三方私有内核驱动
    |-- products                    # uboot,kernel,busybox,item,编译好的镜像、工具、drv驱动等
    |   |-- q3fevb_va               # producst
    |   |   |-- isp                 # isp setting
    |   |   |-- items.itm           # item setting
    |   |   |-- configs             # 内核
    |   |   |   |-- linux-defconfig   # 内核配置参数
    |   |   |   |-- busybox-defconfig # system镜像busybox配置参数
    |   |   |   |-- bblite-defconfig  # ramdisk 镜像中busybox配置参数
    |   |   |-- ispost              # ispost配置目录
    |   |   |-- burn.ixl            # 系统烧写镜像参数文件
    |   |   |-- ota.ixl             # ota镜像制作参数文件
    |   |   |-- system              # system rootfs overlay目录
    |   |   |-- root                # randisk rootfs overlay目录
    |-- packages                    # qsdk发布的部分软件包，方便用户调试camera，wifi等设备
    |   |-- hlibcamsensor           # camsensor driver dir
    |   |-- qlibwifi                 # Wi-Fi drvier dir
    |-- samples                     # 存放单核媒体处理平台的目录
    |   |-- demo                    # 组件源代码
    |-- output                      # 该目录为使用中的临时生成目录，默认不存在
    |   |-- host                    # 交叉编译链等主机工具
    |   |-- staging                 # 编译所需的头文件,库，及二进制文件，所有的编译过程中的头文件查找，库链接都和该文件夹相关
    |   |-- images                  # 目标系统所需的镜像文件，包含内核等各种
    |   |-- system                  # system根文件系统目录，最终制作系统正常工作的工作镜像
    |   |-- root                    # ramdisk根文件系统目录，最终用来制作系统升级和OTA升级的Recovery镜像
    |   |-- product                 # 特定product的软连接
```

## 配置和编译
开发人员在编译过程中需要根据产品类型/需求对SDK进行配置，详细的流程描述如下。

### 产品相关配置
1. 选择产品型号，在SDK的源码根目录下输入以下命令：

`./tools/setproduct.sh`

SDK将详细列出该版本支持的产品类型，开发人员通过输入对应产品型号的序列号进行产品型号选择操作。例如：

```
abc@Dell-OptiPlex-390:~/workspace/q3f-sdk-v1.1$./tools/setproduct.sh
sensor_num1:-1,sensor_num2:-1

#please choose a product from list below:

0  :apollo3_evb
1  :apollo3_evb_ipc
2  :apollo3_evb_cardv
3  :apollo3_evb_fpga

#your choice: 0
```

产品配置成功后，会有以下提示信息：

```
#
# configuration written to /home/abc/workspace/q3f-sdk-v1.1/.config
#

#product successfully set to apollo3_evb
```

2. 选择产品使用的数据配置json文件，在完成产品型号的配置后，SDK的编译脚本会自动提示用户选择产品json文件。用户根据实际的使用情况，选择对应的json文件即可，示例如下：

```
Please choose product json configuration:
  json:
     0   :path_dual.json
     1   :path.json
     x   :default
your choice: x
```

3. 配置Camera Sensor，在完成jason文件选择后，SDK编译程序将自动提示用户进行Camera Sensor的配置。用户可根据SDK支持的Sensor类型，结合实际设备的硬件信息，依次配置每个Camera Sensor的型号参数。示例如下：

```
#Please choose sensor0 configuration:

     0   :ar0330dvp
     1   :ar0330mipi
     2   :ov4689mipi
     x   :none sensor0

#your choice: 2

#Please choose sensor1 configuration:

     0   :ar0330dvp
     1   :ar0330mipi
     2   :ov4689mipi
     x   :none sensor0

#your choice: x

#choose configuration successfully to apollo3_evb
```
在Sensor的配置中选择“x”，表示设备上不存在第二个sensor。

在上面的示例中，我们按照开发套件上的摄像头硬件情况，选择了一个摄像头，型号为ov4689mipi。


以上已经完成了产品的所有配置工作。所有的工作可以使用参数配置的方法一次性完成。


### 详细模块配置

在产品配置的环节，已经生成了产品的.config文件，可以跳过本步骤，直接进入编译环节。

如果开发者希望手动的修改一些差异的配置，可以通过对代码模块的配置来达成。在基于Q3F SDK的开发过程，主要的配置涉及linux内核配置和SDK的配置。其对应的配置命令分别为：

```
make linux-menuconfig
make menuconfig
make busybox-menuconfig
```

Menuconfig是一种菜单式的配置方式，其是一种基于命令行的、交互的、询问式的配置方法。Linux下的开发者不会陌生。

menuconfig中涉及的配置项较多，开发人员在不了解其具体功能前请不要做随意更改。若需要修改，请先查询[配置相关详细文档]()。

### 编译镜像

开始执行编译操作。在SDK代码的根目录下执行make命令后，即可开始编译操作。

若需要确保编译镜像无历史遗留文件或过期的改动，可先执行make clean操作。

编译完成后编译窗口会显示如下信息：

```
...
Generating spi burning output/images/spi_burn ...
spiblk init uboot 49152
spiblk init item 16384
spiblk init ramdisk 4194304
spiblk init kernel 2662400
spiblk init system 12800000
generated!
```

整个编译时间较长，第一次完整编译在半个小时左右。

### 镜像说明

在SDK完成了整体编译后，会自动将我们烧录和debug需要使用的镜像文件和升级包拷贝至output/images目录下，目录中的主要文件在下表中做了详细说明。

|镜像文件	|说明|
|--------------|-------------------------|
|items.itm|	配置文件|
|ramdisk.img|	ramdisk镜像，内有烧录程序，在烧录过程中使用|
|system.img|	系统镜像，可选为ext4或squash格式，启动后挂载成root|
|uboot0.isi|	bootloader镜像|
|uImage|	Linux内核|
|burn.ius|	烧录镜像。通过InfoTM升级包制做工具(IUW)将上述各镜像打包，就生成了burn.ius，用于制做烧录卡|
|ota.ius|	OTA镜像，用于做OTA升级|
|vmlinux|	未压缩的内核，ELF文件，即编译出来的最原始的文件。用于kernel-debug，可产生system.map符号表|

### 常用编译命令

在SDK的开发过程中我们为了方便调试，提供了以下的编译命令来节省整体编译的时间。

|编译命令	|说明|
|--------------|-------------------------|
make package	|单独编译某package
make package-rebuild	|修改某package代码后，增量编译该package
make package-dirclean; make package	|修改某package的编译脚本后（如：Makefile.am, configure.ac, package.mk 等）
make uboot-rebuild	|更改uboot后，增量编译uboot
make linux-rebuild	|更改kernel后，增量编译kernel
make menuconfig	|配置QSDK
make linux-menuconfig	|配置内核
make busybox-menuconfig	|配置busybox
make bblite-menuconfig	|配置小busybox(ramdisk用)

# 开发套件固件手动烧录升级
在SDK整体编译完成之后，目标目录下会生成具体的镜像文件，接下来的工作就是将镜像文件烧录至产品设备中。

## Linux下制作烧录卡
烧录卡制作工具包含在了SDK源码中，工具目录如下：../tools/mkburn.sh，用户在制作烧录卡时只需要在SDK代码的根目录下运行以下命令即可。

`. tools/mkburn.sh /dev/sdb`

sdb是SD卡在PC系统上的设备节点，可以通过ls /dev/sd*来查看SD卡真实的节点名称。

## 镜像烧录升级
在烧录卡制作完成后，便可以开始设备的烧录工作了，具体流程如下：

1. 烧录前先将设备关机；
2. 插入制作完成的烧录卡；
3. 将设备上电开机；
4. 设备启动后便会进入烧录流程，通过串口可查看到烧录进度。
5. 当串口中输出以下log时表示系统烧录完成。

`Remove ius card to enter the new system.`

6. 拔出烧录卡后，设备自动重启进入新烧录的系统。

# [调试接口](https://github.com/InfoTM-SDK/Q3FSDK/wiki/SDK%E8%B0%83%E8%AF%95%E6%8E%A5%E5%8F%A3)
