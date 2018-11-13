# Wi-Fi模块调试指南

### 适用产品

| 类别 | 适用对象 |
|---|
| 软件版本 | `QSDK-V2.2.0` |
| 芯片型号 | `Apollo` `Apollo-2` `Apollo-ECO` |

### 修订记录

| 修订说明 | 日期 | 作者 |
|---|
| 初版 | 2017/07/27 | Bob Yang |

### 术语解释

| 术语 | 解释 |
|---|
| SDK  | Software Development Kit，软件开发工具包 |
| API  | Aplication Program Interface，应用程序接口 |
| wpa_supplicant | 一个开源的Wi-Fi管理和控制程序，主要在Wi-Fi STA模式下使用 |
| hostapd | 一个开源的Wi-Fi管理和控制程序，主要在Wi-Fi AP模式下使用 |
| STA 模式 | Station，类似于无线终端，STA本身并不接受无线的接入,它可以连接到 AP，一般无线网卡即工作在该模式 |
| AP 模式 | Access Point，提供无线接入服务，允许其它无线设备接入，提供数据访问，一般的无线路由/网桥工作在该模式下 |

----
## 1 概述

本篇主要说明一个新的Wi-Fi模块如何加到`QSDK-V2.2.0`（下文简称QSDK）代码包里面，以使Wi-Fi应用层API可以正常运行。主要分成两个部分：添加Wi-Fi模块内核驱动程序和添加HAL层代码。

----
## 2 总体描述
## 2.1 系统架构

![](wifi_sys_frame.svg)

应用程序调用Wi-Fi应用层API与HAL层通讯，Wi-Fi HAL层与Wi-Fi内核驱动程序通讯。

----
## 2.2 系统流程

我们以应用程序d304main，Wi-Fi模块AP6212为例（全文同）

![](wifi_sys_flow.svg)

* **wifi**作为一个服务进程存在于系统中，它对应的就是Wi-Fi的HAL层。可以在系统启动时，在启动脚本（**/etc/init.d/S904tutk**）中启动wifi程序。
* **/wifi/bcmdhd.ko**是AP6212的内核驱动程序，**wifi**进程会加载该驱动。加载成功之后**wifi**进程通过**wpa_supplicant**或者**hostapd**与该驱动通讯。
* **libwifi.a**给应用程序**d304main**提供API来控制**wifi**进程，它对应的就是Wi-Fi应用层API。**libwifi.a**使用IPC与**wifi**进程通讯。

----
## 3 添加Wi-Fi模块
## 3.1 代码文件结构
Wi-Fi内核驱动程序的代码文件结构，文件路径是**kernel/drivers/infotm/common/net/**

```bash
├──bcmdhd6212                    #AP6210 6212 6214 6236模块驱动代码目录
├──bcmdhd6212_prealloc           #AP6210 6212 6214 6236模块静态内存分配驱动代码目录
├──rtl8188EUS_wifi               #rtl8188模块驱动代码目录
├──rtl8189ES_wifi                #rtl8189模块驱动代码目录

```
Wi-Fi HAL层代码文件结构，文件路径是**system/qlibwifi/**
```bash
├──bcm6210                       #bcm6210 HAL层API代码目录
│   ├──firmware                  #bcm6210 firmware
│   ├──bcm6210.c                 #bcm6210 HAL层API代码文件
├──bcm6212                       #bcm6212 HAL层API代码目录
│   ├──firmware                  #bcm6212 firmware
│   ├──bcm6212.c                 #bcm6212 HAL层API代码文件
├──bcm6214                       #bcm6214 HAL层API代码目录
│   ├──firmware                  #bcm6214 firmware
│   ├──bcm6214.c                 #bcm6214 HAL层API代码文件
├──bcm6236                       #bcm6236 HAL层API代码目录
│   ├──firmware                  #bcm6236 firmware
│   ├──bcm6236.c                 #bcm6236 HAL层API代码文件
├──rtl8188                       #rtl8188 HAL层API代码目录
│   ├──firmware                  #rtl8188 firmware
│   ├──rtl8188.c                 #rtl8188 HAL层API代码文件
├──rtl8189                       #rtl8189 HAL层API代码目录
│   ├──firmware                  #rtl8189 firmware
│   ├──rtl8189.c                 #rtl8189 HAL层API代码文件
├──wifi.c                        #该文件主要负责Wi-Fi模块驱动加载，和外部无线设备连接等主要流程处理
├──lib_wifi.c                    #该文件主要负责和wpa_supplicant或者hostapd进程通讯，以及解析各种命令
├──wifi_manager.c                #Wi-Fi应用层API接口文件

```

----
## 3.2 添加源码和配置选项
### 3.2.1 添加Wi-Fi驱动源码文件
新Wi-Fi模块驱动文件目录添加在**kernel/drivers/infotm/common/net/**目录下，AP6212模块添加了**bcmdhd6212**和**bcmdhd6212_prealloc**两个目录。然后修改该目录下的**Makefile**和**Kconfig**文件。

**Makefile**添加该模块相关驱动
```bash
obj-$(CONFIG_BCMDHD_6212)    += bcmdhd6212/
obj-$(CONFIG_DHD_USE_STATIC_BUF)    += bcmdhd6212_prealloc/
```
**Kconfig**添加该模块
```bash
if INFOTM_NET
source "drivers/infotm/common/net/bcmdhd6212/Kconfig"
endif
```

Note: AP6212模块驱动源码目录是**bcmdhd6212**，系统启动后可能会不断的加载卸载该驱动，这样系统的物理连续内存会不足，最后会导致加载驱动失败。增加目录**bcmdhd6212_prealloc**使用静态内存方案来解决这个问题，这样只会在系统启动的时候申请一次内存，之后模块驱动加载和卸载不会影响到这些内存。所有的Wi-Fi模块在调试的时候可以先只增加一个驱动源码目录，到了量产阶段都需要增加相应的静态内存方案。

----
### 3.2.2 配置Wi-Fi内核模块
在QSDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，以模块（m）的形式配置Wi-Fi模块驱动
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   Infotm Network support  --->
  |   |   |   |-<M>   Broadcom FullMAC wireless cards support
  |   |   |   |-(/etc/firmware/fw_bcmdhd.bin) Firmware path
  |   |   |   |-(/etc/firmware/nvram.txt) NVRAM path
  |   |   |   |-(/etc/firmware/config.txt) Config path
  |   |   |   |-[*]   use static buf
```
* **Broadcom FullMAC wireless cards support**选项是AP6212模块的驱动（该驱动兼容AP6210，AP6212，AP6214，AP6236），注意这里选项前面选“M”。
* **Firmware path**，**NVRAM path**，**Config path**这些固件选项使用默认值即可。
* **use static buf**选项表示该Wi-Fi模块驱动使用静态内存，也就是只会在系统启动的时候申请一次内存，之后模块驱动加载和卸载不会影响到这些内存。

等最后系统全部编译完成之后会生成**output/build/linux-local/drivers/infotm/common/net/bcmdhd6212/bcmdhd.ko**文件，同时这个ko文件会自动拷贝到系统如下路径**output/system/lib/modules/3.10.0-infotm+/kernel/drivers/infotm/common/net/bcmdhd6212/bcmdhd.ko**，应用程序调用`wifi_start_service()`时，`wifi_start_service()`内部会调用**modprobe**程序加载这个驱动。

----
### 3.2.3 添加Wi-Fi模块固件
如果Wi-Fi模块有固件，那么需要把固件放在该模块对应的firmware目录下。AP6212的固件都放在**system/qlibwifi/bcm6212/firmware**目录下
```bash
├── config.txt
├── fw_bcm43438a0_apsta.bin
├── fw_bcm43438a0.bin
├── fw_bcm43438a1_apsta.bin
├── fw_bcm43438a1.bin
├── nvram_ap6212a.txt
└── nvram_ap6212.txt
```
等最后系统全部编译完成之后这些固件都会拷贝到系统如下路径**output/system/wifi/firmware/**，应用程序调用`wifi_start_service()`时，`wifi_start_service()`内部会用到这些文件。

----
### 3.2.4 修改config.in文件
在**/system/qlibwifi/config.in**文件添加该Wi-Fi模块

```bash
config BR2_WIFI_HARDWARE_MODULE_B1
        bool "bcm6212"
        help
          BCM6212 hardware module name.

config BR2_WIFI_HARDWARE_MODULE
        string
        default "bcm6210"        if BR2_WIFI_HARDWARE_MODULE_B
        default "bcm6212"        if BR2_WIFI_HARDWARE_MODULE_B1
```
这样在QSDK代码根目录下输入`make menuconfig`命令（参考3.2.5章节），就可以在**Wi-Fi Module**选项里面看到**bcm6212**。

Note: **config.in**里面定义的模块名字 `bcm6212` 必须和HAL层API代码实现目录和文件名（**system/qlibwifi/bcm6212/bcm6212.c**）保持一致。

----
### 3.2.5 配置Wi-Fi HAL选项
在QSDK代码根目录下输入`make menuconfig`命令
```bash
make menuconfig
```
进入到QSDK配置选项，然后按照如下配置:
```bash
QSDK Options --->
  |-Libs --->
  |   |-[*] qlibwifi
  |   |   |-Wi-Fi Module (bcm6212)  --->
```
* **qlibwifi**选项对应的就是**libwifi.a**。
* **Wi-Fi Module**选项就是Wi-Fi的HAL层要选择的Wi-Fi模块，我们这里选择**bcm6212**。

----
## 3.3. 配置Item
在具体product目录下配置item文件里面的Wi-Fi模块选项，比如我们打开**products/qiwo_304/items.itm**文件。

AP6212接口是`sdio`，模块的名字是`bcm6212`，那么在item文件里面如下配置
```bash
wifi.model sdio.bcmdhd6212
```
AP6212模块需要配置上电使能，GPIO122（item文件里面对应`pads.122`）输出高那么Wi-Fi使能模块，GPIO122输出为低那么关闭模块，item文件里面如下配置
```bash
wifi.power pads.122   #请根据具体电路进行配置
```

----
## 4 HAL 数据类型

```cpp
struct wifi_hal_t {
        int stop_dhcp;
        char driver_module_name[16];
        char driver_module_path[64];
        char iface_name[16];
        char ap_iface_name[16];
        char supplicant_path[32];
        char hostapd_path[32];
        char apdns_path[32];
        int (*load_driver)(struct wifi_hal_t *wifi_hal, int mode);
        int (*unload_driver)(struct wifi_hal_t *wifi_hal, int mode);
        int (*is_driver_loaded)(struct wifi_hal_t *wifi_hal, int mode);
};
```

成员
----

| 成员 | 描述 |
| --- | --- |
| `stop_dhcp` | 内部函数 |
| `driver_module_name` |  Wi-Fi 模块驱动程序（ko）的名字 |
| `driver_module_path` | Wi-Fi 模块驱动程序（ko）的路径 |
| `iface_name` | STA 模式节点名称，如 wlan0 |
| `ap_iface_name` | AP 模式节点名称，如 wlan0 |
| `supplicant_path` | wpa_supplicant 程序路径 |
| `hostapd_path` | hostapd 程序路径 |
| `apdns_path` | AP 模式 DNS 分配程序路径，系统自带是 dnsmasq |
| `load_driver` | Wi-Fi 模块驱动加载函数 |
| `unload_driver` | Wi-Fi 模块驱动卸载函数 |
| `is_driver_loaded` | 判断 Wi-Fi 模块驱动是否加载成功函数 |

----
## 5 HAL API

AP6212模块的HAL API实现对应**system/qlibwifi/bcm6212/bcm6212.c**文件，主要的初始化函数以及API实现如下

----
### 5.1 wifi_hal_init()
```cpp
void wifi_hal_init(struct wifi_hal_t *wifi_hal)
{
        snprintf(wifi_hal->driver_module_name, sizeof(wifi_hal->driver_module_name), "bcmdhd");
        snprintf(wifi_hal->driver_module_path, sizeof(wifi_hal->driver_module_path), "/wifi/bcmdhd.ko");
        snprintf(wifi_hal->iface_name, sizeof(wifi_hal->iface_name), "wlan0");
        snprintf(wifi_hal->ap_iface_name, sizeof(wifi_hal->ap_iface_name), "wlan0");
        snprintf(wifi_hal->supplicant_path, sizeof(wifi_hal->supplicant_path), "wpa_supplicant");
        snprintf(wifi_hal->hostapd_path, sizeof(wifi_hal->hostapd_path), "hostapd");
        snprintf(wifi_hal->apdns_path, sizeof(wifi_hal->apdns_path), "dnsmasq");
        wifi_hal->load_driver = bcm6212_load_driver;
        wifi_hal->unload_driver = bcm6212_unload_driver;
        wifi_hal->is_driver_loaded = bcm6212_driver_loaded;

        return;
}

```
这个函数主要用来初始化`wifi_hal_t`结构体里面的成员，初始化每个成员的注意点如下

* `driver_module_name`是Wi-Fi 模块驱动文件的名字（不带ko），这里设置为`"bcmdhd"`。加载Wi-Fi模块驱动后通过`lsmod`命令可以查看到。
```bash
/ # lsmod
Module                  Size  Used by    Tainted: G
bcmdhd                396264  0
hx280enc_h1             3708  0
```
* `driver_module_path`是 Wi-Fi 模块驱动文件的路径，这里设置为`"/wifi/bcmdhd.ko"`，使用`insmod()`加载Wi-Fi模块驱动的时候会用到这个文件。由于系统里面主要使用的是**modprobe**程序，而且这里设置的路径目录下是没有ko文件的，所以该成员设置没有用。
* `iface_name`是STA模式节点名称，这里设置为`"wlan0"`。加载Wi-Fi模块驱动后通过`ifconfig -a`命令可以查看到。
```bash
/ # ifconfig -a
wlan0     Link encap:Ethernet  HWaddr AC:83:F3:35:9E:E6
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:468 (468.0 B)
```
* `ap_iface_name`是AP模式节点名称，这里设置为`"wlan0"`。和`iface_name`一样，加载Wi-Fi模块驱动后通过`ifconfig -a`命令可以查看到。
* `supplicant_path`是**wpa_supplicant**程序路径，这里使用系统默认的设置`"wpa_supplicant"`。
* `hostapd_path`是**hostapd**程序路径，这里使用系统默认的设置`"hostapd"`。
* `apdns_path`是 AP 模式 DNS 分配程序路径，这里使用系统默认的设置`"dnsmasq"`。
* `load_driver`是模块驱动加载函数，这里设置为`"bcm6212_load_driver"`，下面有详细描述如何实现。
* `unload_driver`是模块驱动卸载函数，这里设置为`"bcm6212_unload_driver"`，下面有详细描述如何实现。
* `is_driver_loaded`是判断模块驱动是否加载成功函数，这里设置为`"bcm6212_driver_loaded"`，下面有详细描述如何实现。

----
### 5.2 is_driver_loaded()
```cpp
int (*is_driver_loaded)(struct wifi_hal_t *wifi_hal, int mode);
```
这个函数主要判断模块驱动是否加载成功。

**参数**

- `wifi_hal` - **`IN`** Wi-Fi模块对应的`wifi_hal_t`数据结构
- `mode` - **`IN`** Wi-Fi模块的工作模式，只能是AP或者STA

**返回**

该函数成功返回1，失败返回0。

**举例**
```cpp
static int bcm6212_driver_loaded(struct wifi_hal_t *wifi_hal, int mode)
{
        FILE *proc;
        char line[sizeof(wifi_hal->driver_module_name)+10];

        if ((proc = fopen(MODULE_FILE, "r")) == NULL) {
                printf("Could not open %s: %s", MODULE_FILE, strerror(errno));
                return 0;
        }
        while ((fgets(line, sizeof(line), proc)) != NULL) {
                if (strncmp(line, wifi_hal->driver_module_name, strlen(wifi_hal->driver_module_name)) == 0) {
                        fclose(proc);
                        return 1;
                }
        }
        fclose(proc);
        return 0;
}
```
这个函数主要功能就是通过读取**/proc/modules**文件，查看里面是否有前面设置的`driver_module_name`字符串（这里是`bcmdhd`），从而判断模块驱动是否加载成功。

----
### 5.3 load_driver()
```cpp
int (*load_driver)(struct wifi_hal_t *wifi_hal, int mode);
```
这个函数主要功能是加载Wi-Fi模块驱动。

**参数**

- `wifi_hal` - **`IN`** Wi-Fi模块对应的`wifi_hal_t`数据结构
- `mode` - **`IN`** Wi-Fi模块的工作模式，只能是AP或者STA

**返回**

该函数成功返回0，失败返回-1。

**举例**
```cpp
static int bcm6212_load_driver(struct wifi_hal_t *wifi_hal, int mode)
{
        char bcm6212_module_arg[256];
        char cmd[256];

        if (mode == AP) {
                sprintf(bcm6212_module_arg, "firmware_path=/wifi/firmware/fw_bcm43438a0_apsta.bin nvram_path=/wifi/firmware/nvram_ap6212.txt iface_name=wlan0");

        } else {
                sprintf(bcm6212_module_arg, "firmware_path=/wifi/firmware/fw_bcm43438a0.bin nvram_path=/wifi/firmware/nvram_ap6212.txt iface_name=wlan0");
        }

        if (access(wifi_hal->driver_module_path, F_OK) == 0) {
                if (insmod(wifi_hal->driver_module_path, bcm6212_module_arg) < 0) {
                        printf("%s called,insmod driver failed",__func__);
                        return -1;
                }
        } else {
                sprintf(cmd, "modprobe %s %s", wifi_hal->driver_module_name, bcm6212_module_arg);
                system(cmd);
        }

        return 0;
}
```
这个函数主要使用modprobe程序来加载Wi-Fi模块驱动，`bcm6212_module_arg`是加载驱动的参数，包括固件路径和节点名称。如果Wi-Fi模块没有固件，那就不需要设置这些参数。

Note: 由于`driver_module_path`目录没有驱动ko文件，所以不会使用`insmod()`来加载驱动。

----
### 5.4 unload_driver()
```cpp
int (*unload_driver)(struct wifi_hal_t *wifi_hal, int mode);
```
这个函数主要功能是卸载Wi-Fi模块驱动。

**参数**

- `wifi_hal` - **`IN`** Wi-Fi模块对应的`wifi_hal_t`数据结构
- `mode` - **`IN`** Wi-Fi模块的工作模式，只能是AP或者STA

**返回**

该函数成功返回0，失败返回-1。

**举例**
```cpp
static int bcm6212_unload_driver(struct wifi_hal_t *wifi_hal, int mode)
{
        int count = 20;

        if (rmmod(wifi_hal->driver_module_name) == 0) {
                while (count-- > 0) {
                        if (!wifi_hal->is_driver_loaded(wifi_hal, mode))
                                break;
                        usleep(500000);
                }
                usleep(500000);
                if (count)
                        return 0;
        }

        return -1;
}
```
这个函数使用rmmod程序来卸载相应的Wi-Fi模块驱动，这里卸载的`driver_module_name`是`bcmdhd`。

----
## 6 验证改动

到目前为止，我们已经成功添加Wi-Fi模块驱动和添加HAL层代码，现在我们需要验证这些改动是否有效。修改**/system/qlibwifi/Makefile.am**，把`wifi_test`程序编译进系统
```cpp
bin_PROGRAMS=wifi ap_state wifi_test

#noinst_PROGRAMS=wifi_test
```
`wifi_test`程序是一个Wi-Fi应用层API的测试程序，该程序通过调用Wi-Fi应用层API来间接调用Wi-Fi HAL层API。所以我们可以用它来调试HAL层API。

QSDK全部重新编译之后的系统启动后，按照如下步骤验证改动

### Step 1: 确认wifi程序启动

使用`ps`命令确认下wifi进程在后台运行
```bash
/ # ps
PID   USER     COMMAND
  500 root     cepd
  517 root     audiobox
  519 root     wifi
```
如果没有运行，那么先启动该程序
```bash
/ # wifi &
```
### Step 2: 启动wifi_test程序

启动`wifi_test`程序，程序执行成功会出现一个`%`号
```bash
/ # wifi_test
start to test, now enter your command, press 'q' to exit
have not found wifi process please run wifi
%
```

### Step 3: 启动Wi-Fi STA模式

在`%`号之后输入字符串`sta_enable`，然后回车即可
```bash
% sta_enable
```
如果按照前面文档步骤添加Wi-Fi模块驱动和添加HAL层代码都成功了，那么会出现如下Log
```bash
sta enabled

scan result 1 cuduantui 8c:a6:df:e9:bb:c6 WPA-PSK-CCMP 6
scan result 2 infotm-w 68:8f:84:f1:60:a0 WPA-PSK-TKIP 6
scan result 3 infotm-ic 24:1f:a0:9c:5d:b7 WPA2-PSK-CCMP 13
scan result 4 CMCC_Auto 0c:82:68:ba:98:08 WPA-PSK-CCMP 6
scan result 5 infotm-bsp 00:6b:8e:f6:21:d8 WPA-PSK-CCMP 13
scan result 6 q3f 08:10:79:67:8f:f7 WPA-PSK-CCMP 6
scan result 7 BSP_GFW 54:36:9b:0d:48:68 WPA-PSK-CCMP 13
```
* `sta enabled`是表示Wi-Fi模块正常工作的一个标志，这个字符串很可能会被其他Log刷掉，请自行搜索该字符串。
* 而Log中成功扫描到附近可用的无线网络是另外一个Wi-Fi模块正常工作的标志。

### Step 4: 退出wifi_test程序

在`%`号之后输入`q`，然后回车可以退出测试程序：
```bash
% q
your command is q
/ #
```
