### 术语解释
|术语|解释|
|-----------------|----------------------------------|
| Items | item description， 文本格式的设备资源配置描述 |
| SPI Flash | SPI interface Flash， SPI接口的串行Flash芯片 |
| eMMC | Embedded Multi Media Card， SDIO接口的存储介质 |
| iRom | Internal Rom Code， 固化的启动程序 |
| MTD | Memory Technology Device， 内存技术设备 |
| FTL | Flash Translation Layer， Flash物理块到逻辑块转换层用于文件系统挂载 |

----

## 1 背景概述
QSDK支持从SPI Flash和eMMC设备进行启动，开发过程中可以根据实际需求选择合适的启动设备。启动设备的选择主要涉及Items的配置、U-Boot的配置、Kernel驱动配置。

----

###1.1 SPI Flash和eMMC启动特点

选择SPI Flash或者eMMC作为启动设备在系统启动速度、可用存储空间、以及软硬件处理方面都有所不同。如果系统对启动速度要求快、设备容量要求大那么请选eMMC作为启动设备。其他情况下可以选SPI Flash 作为启动设备。

| 启动设备 | SPI Flash | eMMC |
|---|
| 启动速度 | 12-15MB/s| 25MB/s |
| 容量大小 | 16MB、32MB | 4GB、8GB |
| PCB占用面积 | 小 | 大 |

Note: Q3F Q3-ECO SPI Flash平均速度12-15MB/s



----

## 2 SPI Flash启动（基于FTL）
FTL把Flash的MTD设备映设为块设备。FTL屏蔽了不同存储硬件之间的差异，对文件系统成提供了统一的操作接口。通过FTL可以使用常用的Flash文件系统（如Squashfs），也可以使用块设备文件系统（如Fat、Ext4），从而更有效的利用存储设备。在没有特殊需求的情况下，SPI Flash推荐使用FTL启动方式。

----

###2.1 Items配置启动设备为SPI Flash
在Item配置文件选择从SPI Flash启动，具体的配置文件在产品目录下面。PRODUCT_NAME请替换成使用的产品名字。在**QSDK/product/PRODUCT_NAME/items.itm**文件中， `board.disk` 键值配置成 `flash`

```bash
board.disk	flash
```

----

###2.2 Items分区配置
在**QSDK/product/PRODUCT_NAME/items.itm**中，找到分区配置部分进行配置。

**16MB容量的SPI Flash分区推荐配置**
```bash
part0					 uboot.48.boot
part1					 item.16.boot
part2					 ramdisk.4096.boot
part3					 kernel.4096.boot
part5					 system.19968.fs
part6					 config.512.fs
```

**32MB容量SPI Flash配置**

32MB容量的SPI Flash可以适当的增加`part5`的大小，配置如下

<pre>
part0					 uboot.48.boot
part1					 item.16.boot
part2					 ramdisk.4096.boot
part3					 kernel.4096.boot
part5					 system.<strong>23488</strong>.fs
part6					 config.512.fs
</pre>

----

###2.3 Burn烧录配置
Burn.ixl是分区烧录支持文件，在**QSDK/product/PRODUCT_NAME/burn.ixl**文件配置如下

```bash
i run   0x08000200  0x08000000   ../../output/images/uboot0.isi
r flash 0x0                      ../../output/images/uboot0.isi
r flash 0x1                      ../../output/images/items.itm
r flash 0x2                      ../../output/images/ramdisk.img
r flash 0x3                      ../../output/images/uImage
r flash 0x5                      ../../output/images/rootfs.squashfs
```

在Burn.ixl文件中，数字编号**0x0，0x1，0x2，0x3, 0x5**和Items文件中的**part0，part1，part2，part3，part5**的数字编号对应。

----

###2.4 U-Boot0配置
U-Boot0选择启动设备为SPI Flash。进入QSDK根目录下面并输入命令：`make menuconfig`，依次选择**BootLoader** > **U-Boot** > **Boot Device** > **SPI Flash**配置如下

```bash
/home/worker/QSDK$ make menuconfig
Target options  --->
Build options  --->
Toolchain  --->
System configuration  --->
Kernel  --->
Target packages  --->
Filesystem images  --->
Host utilities  --->
Bootloader  --->
   [*] U-Boot
   [ ]   Local source (NEW)
   (${TOPDIR}/uboot) Local path of uboot source (NEW)
   Boot Device (SPI Flash)  --->
	   (X) SPI Flash
	   ( ) eMMC
	   ( ) mmc
```

----

###2.5 Kernel配置
####2.5.1 SPI驱动配置
进入QSDK根目录下并输入命令`make linux-menuconfig`，配置SPI驱动，使用的CPU不同，选择配置的驱动也不一样。

| CPU类型 | 选择配置 |
|---|
| `Q3F`、`Q3-ECO` | **iMAPx 4-Wire SPI  master driver** |

```bash
Device Drivers  --->
	[*]InfoTM special files and drivers  --->
		<*>Infotm common drivers support  --->
			[*]Infotm SPI support  --->
				< >iMAPx SPI(PL022) master driver
				<*> iMAPx 4-Wire SPI  master driver
```

----


####2.5.2 SPI Flash Translation Layer 配置
Flash Translation Layer配置如下
```bash
Device Drivers  --->
	[*]InfoTM special files and drivers  --->
		<*>Infotm common drivers support  --->
			 Infotm norfalsh ftl support  --->
			   [*]Infotm Norflash ftl SUPPORT
```

----

####2.5.3 MTD以及块驱动配置
SPI Flash使用了MTD设备驱动，配置如下
```bash
<*>Memory Technology Device (MTD) support  --->
[*]Block devices  --->
```

----

####2.5.4 SPI设备驱动配置
SPI Flash存储设备驱动配置
```bash
Device Drivers  --->
    <*>Memory Technology Device (MTD) support  --->
	   Self-contained MTD device drivers  --->
	      <*>Support most SPI Flash chips (AT26DF, M25P, W25X, ...)
		  [*]Use FAST_READ OPCode allowing SPI CLK >= 50MHz
```

----

####2.5.5 文件系统配置
SPI Flash使用了Squashfs类型的文件系统，文件系统配置如下
```bash
File systems  --->
  [*]Enable POSIX file locking API 
  [*]Miscellaneous filesystems  --->
	 <*>SquashFS 4.0 - Squashed file system support
	 [*]Include support for ZLIB compressed file systems
	 [*]Include support for XZ compressed file system
```

----

####2.5.6 Squashfs生成镜像的配置
进入QSDK根目录下面并输入命令：`make menuconfig`，选择 **Filesystem images** > ** squashfs target filesystem**。 **SquashFS version**版本选**4.x**， **Compression algorithm** 选择**xz**。

```bash
Filesystem images  --->
   [*]squashfs target filesystem
       SquashFS version (4.x)  --->
	   Compression algorithm (xz)  --->
```

编译系统之后在** QSDK/output/images/**目录下面生成一个烧写的镜像文件**rootfs.squash**

----

## 3 SPI Flash启动（基于MTD）
在安防产品中使用MTD设备启动，在QSDK中使用MTD启动需要结合U-Boot1使用。

----

### 3.1 QSDK加入U-Boot1的支持
进入QSDK根目录下并输入命令`make menuconfig`配置U-Boot1，根据使用的CPU类型选择正确的配置。

| CPU类型 | 选择配置 |
|---|
| `Q3F`   | **q3f** |
| `Q3-ECO` | **apollo3** |

```bash
/home/worker/QSDK$ make menuconfig
Target options  --->
Build options  --->
Toolchain  --->
System configuration  --->
Kernel  --->
Target packages  --->
Filesystem images  --->
Host utilities  --->
Bootloader  --->
   [*] UBoot1
   	( ) apollo3
	   ( ) q3f
	   ( ) q3
```
----

###3.2 Items配置启动设备为SPI Flash
在Item配置文件选择从SPI Flash启动，具体的配置文件在产品目录下面。PRODUCT_NAME请替换成使用的产品名字。在**QSDK/product/PRODUCT_NAME/items.itm**文件中， `board.disk` 键值配置成 `flash`

```bash
board.disk	flash
```

----

###3.3 Items分区配置
基于MTD的SPI Flash启动设备通过U-Boot1传递带有`mtdparts`关键字的命令行参数给内核创建MTD分区。以Q3F为例，在**QSDK/bootloader/uboot1/include/configs/imap_q3f.h**中定义的Bootargs参数如下

```bash
bootargs=console=ttyAMA3,115200 mem=64M coherent_pool=2M rootfstype=squashfs root=/dev/mtdblock5 rw mtdparts=nor_flash:48k@0(u-boot0)ro,16k@0xc000(items)ro,64k@0x10000(env),256k@0x20000(uboot1),1792k@0x60000(kernel0),6400k@0x220000(system),7808k@0x860000(apps)
```
其中和创建分区有关的部分是以`mtdparts`为关键字语句

```bash
mtdparts=nor_flash:48k@0(u-boot0)ro,16k@0xc000(items)ro,64k@0x10000(env),256k@0x20000(uboot1),1792k@0x60000(kernel0),6400k@0x220000(system),7808k@0x860000(apps)
```
**Mtdparts**配置格式以及解释如下
```bash
mtdparts=<mtd-id>:<size>[@offset][<name>][ro]
```
解释

| 参数名字  | 参数解析 |
|-----|
| **mtd-id** | 整个Flash在MTD系统中的名字 |
| **size**   | 分区大小 |
| **offset** | 分区的偏移量 |
| **name** | 分区的名字 |
| **ro**  |  分区是只读分区 |

在本例中**Mtdparts**各个部分解释说明

| 参数名字 |　参数解析 |
|-----|
| **nor_flash** | Flash在MTD系统中的名字，定义在**QSDK/kernel/driver/mtd/device/m25p80.c** |

| |分区名字 |分区大小(Byte)|分区偏移量(Byte)|是否只读|
|-----|
|**48k@0(u-boot0)ro** | `u-boot0`| `48k` | `0` | 是 |
|**16k@0xc000(items)ro** | `items`| `16k` | `0xc000` | 是 |
|**64k@0x10000(env)**|`env`|`64k`|`0x10000`| 否 |
|**256k@0x20000(uboot1)**|`uboot1`|`256k`|`0x20000`|否|
|**1792k@0x60000(kernel0)**|`kernel0`|`1792k`|`0x60000`|否|
|**6400k@0x220000(system)**|`system`|`6400k`|`0x220000`|否|
|**7808k@0x860000(apps)**|`apps`|`7808k`|`0x860000`|否|

----

###3.4 Burn烧录配置
Burn.ixl是烧录配置文件。在**QSDK/product/PRODUCT_NAME/burn.ixl**文件配置如下

```bash
i run   0x08000200 0x08000000    ../../output/images/uboot0.isi
r flash 0x0                      ../../output/images/uboot0.isi
r flash 0x1                      ../../output/images/items.itm
r flash 0x2                      ../../output/images/ramdisk.img
r flash 0x3                      ../../output/images/uImage
r flash 0x5                      ../../output/images/rootfs.squashfs
r flash 0x10                     ../../output/images/uboot1.isi
```
在Burn.ixl文件中，数字编号**0x0，0x1，0x2，0x3, 0x5，0x10**和Items文件中的**part0，part1，part2，part3，part5　part16**的数字编号对应。

----

###3.5 Kernel添加MTD支持

进入QSDK根目录下并输入命令`make linux-menuconfig`，配置如下

```bash
Device Drivers  --->
    <*>Memory Technology Device (MTD) support  --->
	< >Parallel port support  --->
	[*]Block devices  --->
```
----

###3.6 设备驱动配置
配置SPI Flash控制器驱动，确保**Infotm Norflash ftl SUPPORT**不选，配置如下
```bash
Device Drivers  --->
   [*]InfoTM special files and drivers  --->
     <*>Infotm common drivers support  --->
	    Infotm norfalsh ftl support  --->
			[ ]Infotm Norflash ftl SUPPORT
	    [*]Infotm SPI support  --->
	       <*>iMAPx SPI(PL022) master driver
		   <*>iMAPx 4-Wire SPI  master driver****
```

配置SPI设备驱动如下

```bash
Device Drivers  --->
    <*>Memory Technology Device (MTD) support  --->
	   Self-contained MTD device drivers  --->
	      <*>Support most SPI Flash chips (AT26DF, M25P, W25X, ...)
		  [*]Use FAST_READ OPCode allowing SPI CLK >= 50MHz
```

----


###3.7 文件系统配置

SPI Flash使用了Squashfs类型的文件系统，文件系统配置如下
```bash
File systems  --->
  [*]Enable POSIX file locking API
  [*]Miscellaneous filesystems  --->
	 <*>SquashFS 4.0 - Squashed file system support
	 [*]Include support for ZLIB compressed file systems
	 [*]Include support for XZ compressed file system
```

----

###3.8 内核命令行解析
内核解析U-Boot1传入的命令行参数创建MTD分区，那么在内核中需要加入命令行解析功能，配置如下

```bash
Device Drivers  --->
    <*>Memory Technology Device (MTD) support  --->
    <*>Command line partition table parsing
```

----

###3.9 Squashfs生成镜像的配置
进入QSDK根目录下面并输入命令：`make menuconfig`，选择 **Filesystem images** > ** squashfs target filesystem**。 **SquashFS version**版本选**4.x**， **Compression algorithm** 选择**xz**。

```bash
Filesystem images  --->
   [*]squashfs target filesystem
       SquashFS version (4.x)  --->
	   Compression algorithm (xz)  --->
```

编译系统之后在** QSDK/output/images/**目录下面生成一个烧写的镜像文件**rootfs.squash**

----

## 4 eMMC启动
###4.1 Items配置启动设备为eMMC
在**QSDK/product/PRODUCT_NAME/items.itm**文件中， `board.disk` 键值配置成 `emmc`

```
board.disk	emmc
```

----

###4.2 Items分区配置
Item文件的配置方法和SPI Flash启动配置一样，在**items.item**中分区配置如下
```
part0					uboot.48.boot
part1					item.16.boot
part2					ramdisk.4096.boot
part3					kernel.4096.boot
part5					system.89000.fs
```

----

###4.3 Burn烧录配置
在**QSDK/product/PRODUCT_NAME/burn.ixl**文件，eMMC的文件系统可以有Ext4或者Squashfs两种文件系统。两个文件系统特性如表所示

| 文件系统 | 特点 | 选用时机 |
|---|
| **Squashfs** | 压缩的只读文件系统 | 仅需挂载只读文件系统 |
| **Ext4** | 非压缩可读写文件系统 | 挂载可读写文件系统或者只读文件系统 |


选择使用Ext4文件系统对应**rootfs.ext4**
```bash
i run   0x08000200 0x08000000     ../../output/images/uboot0.isi
r flash 0x0                       ../../output/images/uboot0.isi
r flash 0x1                       ../../output/images/items.itm
r flash 0x2                       ../../output/images/ramdisk.img
r flash 0x3                       ../../output/images/uImage
r flash 0x5                       ../../output/images/rootfs.ext4
```
选择使用**Squashfs**文件系统对应**rootfs.squashfs**
```bash
i run   0x08000200 0x08000000     ../../output/images/uboot0.isi
r flash 0x0                       ../../output/images/uboot0.isi
r flash 0x1                       ../../output/images/items.itm
r flash 0x2                       ../../output/images/ramdisk.img
r flash 0x3                       ../../output/images/uImage
r flash 0x5                       ../../output/images/rootfs.squashfs
```

----

###4.4 U-Boot0配置
在QSDK根目录下面输入命令`make menuconfig`，依次选择**BootLoader** > **U-Boot** > **Boot Device**为eMMC，如下所示

```bash
/home/worker/QSDK$make menuconfig
Target options  --->
Build options  --->
Toolchain  --->
System configuration  --->
Kernel  --->
Target packages  --->
Filesystem images  --->
Host utilities  --->
Bootloader  --->
   [*] U-Boot
   [ ]   Local source (NEW)
   (${TOPDIR}/uboot) Local path of uboot source (NEW)
   Boot Device (eMMC)  --->
	   ( ) SPI Flash
	   (X) eMMC
	   ( ) mmc
```

----

###4.5 Kernel配置
####4.5.1 eMMC驱动配置
在内核中配置eMMC控制器驱动，根据平台使用的CPU不同，配置选择的控制器驱动也不一样。

| CPU 类型 | 选择配置 |
|---|
| `Q3F`、`Q3-ECO` | **DW_MMC Version2.80** |

```bash
Device Drivers  --->
	[*]InfoTM special files and drivers  --->
		<*>Infotm common drivers support  --->
			[*]MMC/SD/SDIO Host Controller Drivers  --->
				--- MMC/SD/SDIO Host Controller Drivers
				 ( ) DW_MMC Version2.40
				 (X) DW_MMC Version2.80
```

----

####4.5.2 块设备驱动配置
eMMC使用了块设备驱动，配置如下

```bash
 <*>Memory Technology Device (MTD) support  --->
 [*]Block devices  --->
```
eMMC的设备驱动需要配上
```bash
 -*- MMC/SD/SDIO card support  --->
     ***MMC/SD/SDIO Card Drivers***
	 -*-MMC block device driver
```
----

####4.5.3 文件系统配置
eMMC使用Ext4文件系统时，配置如下

```bash
File systems  --->
[*] Enable POSIX file locking API
 <*>The Extended 4 (ext4) filesystem
```
eMMC使用Squashfs文件系统时，配置如下
```bash
File systems  --->
  [*]Enable POSIX file locking API
  [*]Miscellaneous filesystems  --->
     <*>SquashFS 4.0 - Squashed file system support
     [*]Include support for ZLIB compressed file systems
     [*]Include support for XZ compressed file system
```
----

####4.5.4 Ext4生成镜像配置
进入QSDK根目录输入`make menuconfig`，选择 **Filesystem images** > ** ext2/3/4 target filesystem** > **ext2/3/4 variant** > **ext4** 配置生成镜像Ext4格式
```bash
Filesystem images  --->
[*] ext2/3/4 target filesystem
	ext2/3/4 variant (ext4)  --->
	   ( ) ext2 (rev0)
	   ( ) ext2 (rev1)
	   ( ) ext3
	   (X) ext4
```
编译系统之后在**QSDK/output/images/**目录下面生成一个烧写的镜像文件**rootfs.ext4**

----

####4.5.5 Squashfs生成镜像的配置
进入QSDK根目录下面并输入命令：`make menuconfig`，选择 **Filesystem images** > ** squashfs target filesystem**。 **SquashFS version**版本选**4.x**， **Compression algorithm** 选择**xz**。

```bash
Filesystem images  --->
   [*]squashfs target filesystem
       SquashFS version (4.x)  --->
	   Compression algorithm (xz)  --->
```
编译系统之后在**QSDK/output/images/**目录下面生成一个烧写的镜像文件**rootfs.squashfs**

----

##5 分区配置
在**QSDK/product/PRODUCT_NAME/items.itm**配置文件中找到分区配置部分

```bash
part0					uboot.48.boot
part1					item.16.boot
part2					ramdisk.4096.boot
part3					kernel.4096.boot
#part4                   kernel.2560.boot
part5					system.19968.fs
```

| 分区编号 | 分区名字　| 分区作用 | 大小可配 |  是否必选  |
|---|
| **part0** | `uboot` |存放`uboot`| 否 | 是 |
| **part1** | `item` |存放`item` | 否| 是 |
| **part2**| `ramdisk`|存放`ramdisk` | 是 | 否 |
| **part3** | `kernel` |存放`kernel`| 是 | 是 |
|**#part4** | `kernel` |存放备用`kernel`| 是　| 否|
| **part5** | `system` |存放`system` | 是 | 是 |

Note: 其中**#part4**被注释掉了，在正常使用过程中不需要该分区。

启动设备是SPI Flash时，在Item中还可以有`normal`类型的分区，所以在QSDK中一共有3种分区类型。

| 分区类型 | 分区作用 | 何时选用 |
|---|
| 物理分区`boot` | 存放Boot、Items、Kernel、Ramdisk等镜像 | 直接存放数据时 |
| 只读文件系统分区`normal`  | 挂载只读文件系统 | 当前只能用于SPI Flash，挂载只读文件系统 |
| FTL逻辑分区`fs` | FTL只读或者读写文件系统 | 挂载FTL转换的只读或者读写文件系统（绝大部分情况推荐使用） |

那么本例中Items分区配置可以表述为

| 分区编号 | 分区名字　| 分区作用 | 大小可配 |  是否必选  |　分区类型　|
|---|
| **part0** | `uboot` |存放`uboot`| 否 | 是 |物理分区`boot` |
| **part1** | `item` |存放`item` | 否| 是 | 物理分区`boot` |
| **part2**| `ramdisk`|存放`ramdisk` | 是 | 否 | 物理分区`boot` |
| **part3** | `kernel` |存放`kernel`| 是 | 是 | 物理分区`boot` |
|**#part4** | `kernel` |存放备用`kernel`| 是　| 否| 物理分区`boot` |
| **part5** | `system` |存放`system` | 是 | 是 | FTL逻辑分区`fs` |


----

###5.1 添加物理分区

一些应用中需要直接操作物理设备，系统通过操作物理分区来直接控制物理设备，启动设备不同添加配置物理分区的方法也不相同。

----

5.1.1　基于FTL的SPI Flash启动

在Items的配置文件中添加**物理分区8**。分区编号是**part8**，分区名字**mypart**，分区大小**64KB**，分区类型物理分区，方法如下

<pre>
part0     uboot.48.boot
part1     item.16.boot
part2     ramdisk.4096.boot
part3     kernel.4096.boot
part5     system.19968.fs
part6     id.512.fs
part7     config.512.fs
<strong>part8     mypart.64.boot</strong>  #新添加的物理分区8，大小是64KB。
</pre>

Note: part16固定用作U-Boot1分区，part分区个数不超过17个

新添加的分区通过设备节点和偏移量的方法进行访问。在本例中新添加的分区**part8**在物理分区的尾部，其偏移量根据启动设备的不同偏移量计算也不同。

SPI_OFFSET = part0(**48KB**) **+** part1(**16KB**) **+** RSV(**16KB**) **+** part2(**4096KB**) **+** part3(**4096KB**)

eMMC_OFFSET = EMMC_IMAGE_OFFSET(**2048KB**) **+** part0(**48KB**) **+** part1(**16KB**) **+** RSV(**16KB**) **+** part2(**4096KB**) **+** part3(**4096KB**)

| 启动设备 | 访问节点 | 偏移量 |
|---|
| SPI Flash | **/dev/spiblock0** | SPI_OFFSET |
| eMMC | **/dev/mmcblk0**  | eMMC_OFFSET |

----

5.1.2　基于MTD的SPI Flash启动

以Q3F为例，在**QSDK/bootloader/uboot1/include/configs/imap_q3f.h**中定义的Bootargs参数中和创建分区有关的部分

```bash
mtdparts=nor_flash:48k@0(u-boot0)ro,16k@0xc000(items)ro,64k@0x10000(env),256k@0x20000(uboot1),1792k@0x60000(kernel0),6400k@0x220000(system),7808k@0x860000(apps)
```

在不超过Flash容量的前提下添加一个物理分区**tstpart**，大小是**128KB**

```bash
mtdparts=nor_flash:48k@0(u-boot0)ro,16k@0xc000(items)ro,64k@0x10000(env),256k@0x20000(uboot1),1792k@0x60000(kernel0),6400k@0x220000(system),7808k@0x860000(apps)，128k@0x860128(tstpart)
```

在本例中MTD分区的设备节点是 **/dev/mtd7**，偏移量是０，大小是128KB

| 启动设备 | 访问节点 | 偏移量 | 分区大小|
|---|
| SPI Flash | **/dev/mtd7** | 0 | 128KB |


----

###5.2 添加FTL逻辑分区
为了使用常见的块设备文件系统如Ext4或者是Flash文件系统如Squashfs，需要FTL分区的支持。在Items的配置文件中添加**FTL逻辑分区9**，分区名字**ftlpart**，分区大小是**128KB**，分区类型**FTL逻辑分区**。

<pre>
part0     uboot.48.boot
part1     item.16.boot
part2     ramdisk.4096.boot
part3     kernel.4096.boot
part5     system.19968.fs
part6     id.512.fs
part7     config.512.fs
part8     mypart.64.boot   #新添加的物理分区8，大小是64KB
<strong>part9     ftlpart.128.fs   #新添加的FTL逻辑分区9，大小是128KB</strong>
</pre>

Note: eMMC最大的**fs**分区是四个，Items不能有大于四个的**fs**分区。

FTL逻辑分区通过挂载文件系统进行访问。根据启动设备不同，文件系统的挂载设备节点也不一样。

| 启动设备 | 文件系统挂载节点 |
|---|
| SPI Flash | **/dev/spiblock4** |
| eMMC | **/dev/mmcblk0p4**  |


-----

###5.3 SPI Flash添加Normal只读文件系统分区
当启动设备是SPI Flash时，还可以有Normal只读文件系统分区，在Items配置文件中添加一个Normal只读文件系统分区

<pre>
part0     uboot.48.boot
part1     item.16.boot
part2     ramdisk.4096.boot
part3     kernel.4096.boot
part5     system.19968.fs
part6     id.512.fs
part7     config.512.fs
part8     mypart.64.boot      #新添加的物理分区8，大小是64KB
part9     ftlpart.128.fs      #新添加的FTL逻辑分区9，大小是128KB
<strong>part10    mynormal.64.normal  #新添加的Normal只读文件系统分区，大小是64KB </strong>
</pre>


**Normal分区的访问**

Normal分区通过挂载只读文件系统进行访问即把文件系统挂载到**/dev/spiblock1**访问。**所有的Normal分区访问节点相同**，添加多个Normal分区不会生成更多的访问节点。所以添加两个以及两个以上的Normal分区没有意义，**推荐只添加一个容量够大的Normal分区**。


**分区和设备节点关系**

只有SPI Flash允许配置Normal分区，eMMC是不允许配置Normal分区的，有Normal分区时设备的访问节点如下

|  分区类型  | SPI Flash启动 |
|---|
| 物理分区`boot`  | **/dev/spiblock0** |
| 只读文件系统分区`normal` | **/dev/spiblock1** |
| FTL逻辑分区`fs`    |  **/dev/spiblock2 ... /dev/spiblock63** |

不配置Normal分区时，分区和设备节点对应关系如下

|  分区类型  | SPI Flash启动 | eMMC启动 |
|---|
| 物理分区`boot`  | **/dev/spiblock0** | **/dev/mmcblk0** |
| FTL逻辑分区`fs`    |  **/dev/spiblock1 ... /dev/spiblock63** | **/dev/mmcblock0p1 ... /dev/mmcblock0p4** |


** 配置Normal分区时命令行参数修改　**

系统中既有Normal分区又有FTL逻辑分区时，需要修改命令行启动参数，以Q3F为例，在**QSDK/bootloader/apollo2/inc/image.h**文件中修改命令行参数中**root**分区为**/dev/spiblock2**

```cpp
#define CMDLINE_SPI_FLASH "console=ttyAMA3,115200 lpj=%d mem=%dM rootfstype=%s root=/dev/spiblock2 rw"
```

Note: 系统中Normal和FTL逻辑分区同时使用时需要修改命令行参数，不同时使用则不需要修改命令行参数


----

###5.4 新分区烧录配置
在烧录配置文件**QSDK/product/PRODUCT_NAME/burn.ixl**为新加的分区做烧录支持，新添加的**物理分区part8**以及**FTL逻辑分区part9**烧录支持，配置Burn.ixl文件内容

<pre>
i run   0x08000200 0x08000000  ../../output/images/uboot0.isi
r flash 0x0                    ../../output/images/uboot0.isi
r flash 0x1                    ../../output/images/items.itm
r flash 0x2                    ../../output/images/ramdisk.img
r flash 0x3                    ../../output/images/uImage
r flash 0x5                    ../../output/images/rootfs.squashfs
<strong>r flash 0x8                    ../../output/images/myrawdata      #分区8烧录支持
r flash 0x9                    ../../output/images/myfs.img       #分区9烧录支持
r flash 0xa                    ../../output/images/mynormal.img   #分区10烧录支持</strong>
</pre>

