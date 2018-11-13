### 术语解释
|  术语   |                                           解释                                           |
| ------- | ---------------------------------------------------------------------------------------- |
| API     | Aplication Program Interface，应用程序接口                                               |
| IO复用  | 全称应该是GPIO引脚复用，指的是芯片内置设备是与IO口共用引出管脚（不同的功能对应同一管脚） |
| PLL     | Phase Looked Loop 锁相环                                                                 |
| APLL    | Analogue Phase Locked Loop，模拟锁相环                                                   |
| DPLL    | Digital Phase Looked Loop，数字锁相环                                                    |
| EPLL    | Enhanced Phase Looked Loop，增强锁相环                                                   |
| VPLL    | Video Phase Looked Loop，视频锁相环                                                      |
| OSC     | Oscillator，振荡器                                                                       |
| APB     | Advanced Peripheral Bus，高级外围总线                                                    |
| NCO     | Numerically Controlled Oscillator，数字控制振荡器                                        |
| 占空比  | 在一个脉冲循环内，通电时间相对于总时间所占的比例                                         |
| OHCI    | 开放主机控制接口，USB1.0/1.1                                                             |
| EHCI    | 增强主机控制接口，USB2.0                                                                 |
| OTG     | On-The-Go，主要应用于各种不同的设备或移动设备间的联接，进行数据交换                   |
| USB HCD | USB Host Core Driver， USB主机控制器驱动                                                 |
| UVC     | USB Video Class，是Microsoft与另外几家设备厂商联合推出的为USB视频捕获设备定义的协议标准  |
| USB ETH | USB以太网                                                                                |
| USB HID | USB Human Interface Device，USB人机交互设备，例如键盘、鼠标与游戏杆等                    |


----
## 1 总体概述
本文主要描述了如何使用SDK代码包来调试设备驱动，这些设备包括Clock，GPIO，ADC，PWM，IIC，Power，SDIO，DPMU，Audio，USB。因为`Q3F` `Q3-ECO`（以下这些芯片会统称为`芯片`）上的这些设备非常相似，所以下文中会主要以`Q3F`芯片为例来说明这些设备的驱动调试，有较大区别的地方会特别指出。

----
## 2 Clock
芯片采用一个12MHz的**OSC**晶振作为时钟源，来产生４路**PLL**：**APLL**，**DPLL**，**EPLL**，**VPLL**，这4路**PLL**加上**OSC**和**EXTCLK**组成了各种设备，总线（**BUS**），**APB**和**CPU**的时钟源。**BUS**和**APB**同时也是一些设备的固定时钟源。另外**APLL**是**CPU**的固定时钟源。

----
## 2.1 驱动架构

![clock_driver_frame](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/clock_driver_frame.svg)

**Clock驱动**为**Clock Core**实现各总线和设备Clock硬件配置细节，这样**设备驱动**就可以向**Clock Core**申请和注册该设备对应的Clock配置，然后**Clock Core**通过**clk_ops**系统生成**Clock DebugFS**，以便查看和调试设备的Clock配置。

----
## 2.2 代码文件
芯片的Clock驱动文件如下，本章节会以`Q3F`芯片对应的驱动文件为例

|                   文件名                    |              说明               |
| ------------------------------------------- | ------------------------------- |
| **kernel/drivers/infotm/q3f/clk/clk.c**     | `Q3F`芯片对应的Clock驱动   |
| **kernel/drivers/infotm/apollo3/clk/clk.c** | `Q3-ECO`芯片对应的Clock驱动 |

**Clock DebugFS**是linux内核自带的功能，我们不需要关注相关代码文件，只需要参考下一章节选上相关配置选项就好了。

----
## 2.3 配置选项

在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**Q3F clock driver support**选项
```bash
Device Drivers --->
  |-[*] InfoTM special files and drivers  --->
  |   |-[*]   Infotm Q3F Series Support  --->
  |   |   |-[*]   Q3F clock driver support  --->
```

----
## 2.4 场景功能

----
### 2.4.1 通过API修改Clock
以设置MMC0路控制器时钟频率为50MHz为例
```cpp
void mmc0_set_rate(struct dw_mci *host)
{
        const struct dw_mci_drv_data *drv_data = host->drv_data;
        struct dw_mci *mmc_host = host;

        //获取设备时钟源
        host->ciu_clk = clk_get_sys("imap-mmc.0", "sd-mmc0");

        //先设置设备父时钟源是"epll"，然后设置时钟频率为50MHz，
        //最后读取时钟频率看是否设置成功
        clk_set_parent(mmc_host->ciu_clk,clk_get_sys("epll","epll"));
        clk_set_rate(mmc_host->ciu_clk,50000000);
        mmc_host->bus_hz = clk_get_rate(mmc_host->ciu_clk);
        if(mmc_host->bus_hz != 50000000)
                printk("set freq err!\n");

        //打开设备时钟源
        clk_prepare_enable(host->ciu_clk);

        //关闭和释放设备时钟源
        clk_disable_unprepare(host->ciu_clk);
        clk_put(host->ciu_clk);
}
```

----
### 2.4.2 通过修改默认配置修改Clock
设备驱动除了使用API来配置设备时钟频率，还可以在Clock驱动直接修改。设备时钟频率配置相关结构体`struct bus_and_dev_clk_info`定义如下
```cpp
struct bus_and_dev_clk_info {
        uint32_t dev_id;
        uint32_t nco_en;
        uint32_t clk_src;
        uint32_t nco_value;
        uint32_t clk_divider;
        uint32_t nco_disable;
};
```

**成员**

|     成员      |           描述            |
| ------------- | ------------------------- |
| `dev_id`      | 设备id                    |
| `nco_en`      | **NCO**模式使能，不要修改 |
| `clk_src`     | 时钟源                    |
| `nco_value`   | **NCO**值，不要修改       |
| `clk_divider` | 时钟分频参数              |
| `nco_disable` | **NCO**相关参数，不要修改 |

成员里面我们只用关注`dev_id`，`clk_src`，`clk_divider`（设备时钟频率主要修改的就是这项）这三项，**NCO**相关的成员一般不要修改。假设设备（对应`dev_id`）时钟频率为`dev_clk`，时钟源（对应`clk_src`）频率为`src_clk`，设备时钟频率计算公式是
```bash
dev_clk = src_clk / (clk_divider + 1)
```

现在打开**kernel/drivers/infotm/q3f/clk/clk.c**，设备时钟频率配置对应如下结构体数组
```cpp
static struct bus_and_dev_clk_info dev_clk_info[] = {
        DEV_CLK_INFO(BUS1, 0, EPLL, 0, 3, ENABLE),
        /* default bus3 setting by shaft */
        DEV_CLK_INFO(BUS3, 0, EPLL, 0, 3, DISABLE),
        /* bus4 bus5 do not use in imapx9*/
        DEV_CLK_INFO(BUS4, 0, EPLL, 0, 1, DISABLE),
        DEV_CLK_INFO(BUS6, 0, DEFAULT_SRC, 0, 3, ENABLE),

        DEV_CLK_INFO(DSP_CLK_SRC, 0, DPLL, 0, 2, ENABLE),

        /*Note: why set IDS0 to this value? */
        DEV_CLK_INFO(IDS0_EITF_CLK_SRC, 0, EPLL, 0, 2, DISABLE),
        DEV_CLK_INFO(IDS0_OSD_CLK_SRC, 0, EPLL, 0, 1, DISABLE),

        DEV_CLK_INFO(MIPI_DPHY_CON_CLK_SRC, 1, EPLL, 6, 0, ENABLE),
        DEV_CLK_INFO(SD2_CLK_SRC, 0, DPLL, 0, 23, ENABLE),
        DEV_CLK_INFO(SD1_CLK_SRC, 0, EPLL, 0, 14, DISABLE),
        DEV_CLK_INFO(SD0_CLK_SRC, 0, EPLL, 0, 14, DISABLE),
        DEV_CLK_INFO(UART_CLK_SRC, 0, EPLL, 0, 5, ENABLE),
        DEV_CLK_INFO(SSP_CLK_SRC, 0, EPLL, 0, 3, ENABLE),

        DEV_CLK_INFO(AUDIO_CLK_SRC, 0, DPLL, 0, 24, DISABLE),
        DEV_CLK_INFO(USBREF_CLK_SRC, 1, DPLL, 4, 31, ENABLE),

        DEV_CLK_INFO(CAMO_CLK_SRC, 1, DPLL, 4, 19, ENABLE),
        DEV_CLK_INFO(CLKOUT1_CLK_SRC, 0, DPLL, 0, 0, ENABLE),
        DEV_CLK_INFO(MIPI_DPHY_PIXEL_CLK_SRC, 0, EPLL, 0, 7, DISABLE),
        DEV_CLK_INFO(DDRPHY_CLK_SRC, 0, VPLL, 0, 1, ENABLE),
        DEV_CLK_INFO(CRYPTO_CLK_SRC, 0, DPLL, 0, 9, ENABLE),
        DEV_CLK_INFO(TSC_CLK_SRC, 0, OSC_CLK, 0, 0, ENABLE),
        DEV_CLK_INFO(APB_CLK_SRC, 0, DEFAULT_SRC, 0, 9, ENABLE),
};
```
该结构体数组里面各种总线和设备的时钟频率都配置好了，我们拿`SD1_CLK_SRC`对应的设备**SD1**来举例
```cpp
        DEV_CLK_INFO(SD1_CLK_SRC, 0, EPLL, 0, 14, DISABLE),
```
`clk_src`是`EPLL`，`clk_divider`是14，系统默认的`EPLL`为594MHz（`DPLL`是1536MHz，`APLL`是516MHz，`VPLL`是1200MHz），按照设备时钟频率计算公式算出来**SD1**的时钟频率是594/（14+1）=39.6MHz，通过下面章节的调试方法可以看到这个频率值。

Note: 不建议修改总线的时钟频率，因为这样会很容易导致系统崩溃。

----
## 2.5 API

----
### 2.5.1 clk_get_sys()
```cpp
struct clk *clk_get_sys(const char *dev_id, const char *con_id);
```
这个函数主要根据设备名称获取相应的时钟源。

**参数**

- `dev_id` - **`IN`** 设备id
- `con_id` - **`IN`** 连接id

**返回**

该函数成功返回设备时钟源对应结构体`clk`，失败返回错误码。


----
### 2.5.2 clk_put()
```cpp
void clk_put(struct clk *clk);
```
这个函数用来释放设备时钟源。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`

**返回**

无

----
### 2.5.3 clk_prepare_enable()
```cpp
int clk_prepare_enable(struct clk *clk);
```
这个函数用来打开设备时钟源。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`

**返回**

该函数成功返回0，失败返回错误码。

----
### 2.5.4 clk_disable_unprepare()
```cpp
void clk_disable_unprepare(struct clk *clk);
```
这个函数用来关闭设备时钟源。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`

**返回**

无

----
### 2.5.5 clk_get_parent()
```cpp
struct clk *clk_get_parent(struct clk *clk);
```
这个函数用来获取设备父时钟源。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`

**返回**

该函数成功返回设备父时钟源对应结构体`clk`，失败返回错误码。

----
### 2.5.6 clk_set_parent()
```cpp
int clk_set_parent(struct clk *clk, struct clk *parent);
```
这个函数用来设置父时钟源。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`
- `parent` - **`IN`** 设备父时钟源对应结构体`clk`

**返回**

该函数成功返回0，失败返回错误码。

----
### 2.5.7 clk_get_rate()
```cpp
unsigned long clk_get_rate(struct clk *clk);
```
这个函数用来获取设备时钟频率（单位Hz）。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`

**返回**

该函数成功返回设备时钟频率（单位Hz），失败返回错误码。

----
### 2.5.8 clk_set_rate()
```cpp
int clk_set_rate(struct clk *clk, unsigned long rate);
```
这个函数用来设置设备时钟频率（单位Hz）。

**参数**

- `clk` - **`IN`** 设备时钟源对应结构体`clk`
- `rate` - **`IN`** 设备时钟频率

**返回**

该函数成功返回0，失败返回错误码。

----
## 2.6 调试方法
如果要使用**Clock DebugFS**功能，那么需要在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**DebugFS representation of clock tree**选项
```bash
Device Drivers --->
  |-Common Clock Framework --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*] DebugFS representation of clock tree
```

启动系统后，我们就可以使用**Clock DebugFS**功能来查看所有总线和设备的时钟频率
```
/ # mount -t debugfs none  /sys/kernel/debug
/ # cat /sys/kernel/debug/clk/clk_summary
   clock                        enable_cnt  prepare_cnt  rate
---------------------------------------------------------------------
 osc-clk                        6           6            12000000
    touch screen                2           2            12000000
    vpll                        1           1            1200000000
       ddr-phy                  0           0            400000000
    epll                        6           6            594000000
       mipi-dphy-pix            1           1            74250000
       ssp-ext                  1           1            60328112
       uart                     3           4            99000000
       sd-mmc0                  0           0            39600000
       sd-mmc1                  1           1            39600000
       mipi-dphy-con            0           0            13921872
       ids0-osd                 0           0            297000000
       ids0-eitf                0           0            198000000
       bus4                     0           0            297000000
          venc264-ctrl          0           0            297000000
       bus3                     0           0            148500000
          ispost                0           0            148500000
          fodet                 0           0            148500000
          crypto-dma            0           0            148500000
          hdmi-tx               0           0            148500000
       bus1                     1           1            198000000
          ids0                  0           0            198000000
          mipi-csi              0           0            198000000
          camif                 0           0            198000000
          isp                   1           1            198000000
    dpll                        5           5            1536000000
       apb_output               11          11           85333333
          watch-dog             1           1            85333333
          gpio                  1           1            85333333
          tsc-ctrl              1           1            85333333
          audiocodec            0           0            85333333
          uart-core             2           2            85333333
          spi1                  0           0            85333333
          spi0                  0           0            85333333
          pcm0                  0           0            85333333
          i2c3                  1           1            85333333
          i2c2                  1           1            85333333
          i2c1                  1           1            85333333
          i2c0                  1           1            85333333
          pwm                   1           1            85333333
          cmn-timer1            0           0            85333333
          cmn-timer0            1           1            85333333
          iis                   1           1            85333333
       crypto                   0           0            1536000000
       clk-out1                 0           0            1536000000
       camo                     0           0            24000000
       usb-ref                  0           0            24000000
       audio-clk                1           1            61440000
       sd-mmc2                  1           1            64000000
       dsp                      0           0            512000000
       bus6                     5           5            170666666
          spi-bus               1           1            170666666
          eth                   0           0            170666666
          dma                   1           1            170666666
          sdmmc2                1           1            170666666
          sdmmc1                1           1            170666666
          sdmmc0                0           0            170666666
          usb-otg               0           0            170666666
          usb-ehci              0           0            170666666
          usb-ohci              0           0            170666666
    apll                        4           4            516000000
       apb_pclk                 3           3            64500000
       cpu-clk                  1           1            516000000
       gtm-clk                  1           1            64500000
```
其中我们可以看到`sd-mmc1`设备的时钟频率是39.6MHz。除了前面描述的通过修改设备对应的`clk_divider`参数值来改变设备时钟频率，还有个方法是直接使用**Clock DebugFS**提供的调试功能，比如使用如下调试命令修改`sd-mmc1`设备的时钟频率为19.8MHz。
```
/ # echo sd-mmc1 >/sys/kernel/debug/clk/configure/name
/ # cat /sys/kernel/debug/clk/configure/freq
[ 1723.986666] 39600000
/ # echo 19800000 > /sys/kernel/debug/clk/configure/freq
[ 1801.433333] clk_set_rate sd-mmc1 ,oldrate:39600000, new rate:19800000
[ 1801.436666] clk_set_rate ok 19800000
/ # cat /sys/kernel/debug/clk/configure/freq
[ 1803.823333] 19800000
```

----
## 3 GPIO
芯片的GPIO都支持设备功能模式或者普通GPIO模式的配置，在芯片内部有对应上下拉电阻设计。在GPIO输入模式下，还可以配置中断触发功能。

----
## 3.1 驱动架构
![gpio_driver_frame](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/gpio_driver_frame.svg)

**GPIO驱动**为**GPIO Core**实现各GPIO硬件配置的细节，这样**设备驱动**就可以向**GPIO Core**申请和配置该设备需要用到的GPIO。


----
## 3.2 代码文件
芯片的GPIO驱动文件如下，本章节会以`Q3F`芯片对应的驱动文件为例，该驱动文件实现了GPIO的输入输出配置，以及中断触发功能（GPIO输入模式下）。

|                   文件名                    |              说明               |
| ------------------------------------------- | ------------------------------- |
| **kernel/drivers/infotm/q3f/gpio/gpio-q3f.c**     | `Q3F`芯片对应的GPIO驱动   |
| **kernel/drivers/infotm/apollo3/gpio/gpio-apollo3.c** | `Q3-ECO`芯片对应的GPIO驱动 |

----
## 3.3 配置选项

在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**Infotm gpio chip driver support**选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-[*]   Infotm Q3F Series Support  --->
  |   |   |-[*]   Infotm gpio chip driver support  --->
  |   |   |   |-[*]   Q3F gpio chip driver support
```
----
## 3.4 场景功能

下面把110号GPIO配置成输入或者输出模式。
```cpp
void gpio_set_test(void)
{
        int value = 0;

        //申请110号GPIO资源
        gpio_request(110, "test");

        //配置该GPIO为输入，并且读取该GPIO引脚状态
        gpio_direction_input(110);
        value = gpio_get_value(110);
        printk("value is %s \n", value);

        //配置该GPIO为输出，并且设置该GPIO引脚为低，
        //之后读取引脚状态，看是否配置成功
        gpio_direction_output(110, 0);
        value = gpio_get_value(110);
        printk("value is %s \n", value);

        //设置置该GPIO引脚为高，之后读取引脚状态，
        //看是否配置成功
        gpio_set_value(110, 1)
        value = gpio_get_value(110);
        printk("value is %s \n", value);

        //释放110号GPIO资源
        gpio_free(110);
}
```

----
## 3.5 API

----
### 3.5.1 gpio_request()
```cpp
int gpio_request(unsigned gpio, const char *label);
```
这个函数主要是设备驱动向GPIO Core申请GPIO资源。

**参数**

- `gpio` - **`IN`** GPIO引脚号
- `label` - **`IN`** 设备申请GPIO资源使用的标签

**返回**

该函数成功返回0，失败返回错误码。

----
### 3.5.2 gpio_free()
```cpp
void gpio_free(unsigned gpio);
```
这个函数主要是设备驱动向GPIO Core申请释放GPIO资源。

**参数**

- `gpio` - **`IN`** GPIO引脚号

**返回**

无返回值。

----
### 3.5.3 gpio_direction_input()
```cpp
int gpio_direction_input(unsigned gpio);
```
这个函数是配置GPIO引脚为输入。

**参数**

- `gpio` - **`IN`** GPIO引脚号

**返回**

该函数成功返回0，失败返回错误码。

----
### 3.5.4 gpio_direction_output()
```cpp
int gpio_direction_output(unsigned gpio， int value);
```
这个函数是配置GPIO引脚为输出，并且置GPIO引脚的状态（高或者低电平）。

**参数**

- `gpio` - **`IN`** GPIO引脚号
- `value` - **`IN`** GPIO引脚状态:1-高电平，0-低电平

**返回**

该函数成功返回0，失败返回错误码。

----
### 3.5.5 gpio_set_value()
```cpp
void gpio_set_value(unsigned gpio, int value);
```
这个函数是设置GPIO引脚的状态（高或者低电平）。

**参数**

- `gpio` - **`IN`** GPIO引脚号
- `value` - **`IN`** GPIO引脚状态:1-高电平，0-低电平

**返回**

无返回值。

----
### 3.5.6 gpio_get_value()
```cpp
int gpio_get_value(unsigned gpio);
```
这个函数是获取GPIO引脚的状态（高或者低电平）。

**参数**

- `gpio` - **`IN`** GPIO引脚号

**返回**

GPIO引脚状态:1-高电平，0-低电平。

----
### 3.5.7 gpio_to_irq()
```cpp
int gpio_to_irq(unsigned gpio);
```
这个函数是获取GPIO引脚对应的中断号。

**参数**

- `gpio` - **`IN`** GPIO引脚号

**返回**

GPIO引脚对应的中断号。

----
## 3.6 调试方法

启动系统后，GPIO的调试接口位于**sys/class/gpio**目录下，具体调试方法如下
### Step 1: 导出GPIO引脚号
以引脚号110为例
```
/ # echo 110 > sys/class/gpio/export
```
导出成功后，会生成**sys/class/gpio/gpio110**目录，这个目录里有4个文件节点**direction**，**value**，**edge**，**active_low**可以用来对引脚号为110的GPIO进行控制，节点描述如下

|      属性      |                                              意义                                               |
| -------------- | ----------------------------------------------------------------------------------------------- |
| **direction**  | GPIO引脚的工作模式：`in`-输入，`out`-输出。可读可写                                             |
| **value**      | GPIO引脚的状态：`1`-高电平，`0`-低电平                                                          |
| **edge**       | 引脚中断模式触发方式:`rising`-上升沿，`falling`-下降沿，`both`-上升下降沿，`none`-无                    |
| **active_low** | GPIO逻辑反转：`0`-逻辑正常，`1`-逻辑反转。逻辑反转后，**value**值将会获得与实际电平相反的状态。 |

### Step 2: 配置GPIO
配置110号引脚为输入（高电平）或者输出
```
/ # echo out >/sys/class/gpio/gpio110/direction
/ # echo 1 >/sys/class/gpio/gpio110/value
/ # cat  /sys/class/gpio/gpio110/value
1
/ # echo in >/sys/class/gpio/gpio110/direction
/ # cat  /sys/class/gpio/gpio110/value
1
```

### Step 3: 释放GPIO引脚
110号引脚调试完成后，把该引脚释放
```
/ # echo 110 > /sys/class/gpio/unexport
```

----
## 4 ADC
ADC全称nalog to Digital Converter，即模拟数字信号转换器。芯片采用的是CPU内部集成ADC模块，在SDK系统中，ADC模块被用于基于电平采样实现电池容量计算，传感器数据采集和按键事件侦听。

----
## 4.1 驱动架构

![adc subsystem](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/adc_driver_frame.svg)

ADC驱动架构分为**设备驱动**，**ADC Core**和**ADC驱动**３个部分。**ADC驱动**为**ADC Core**实现ADC控制器硬件功能的细节，这样**设备驱动**可以调用**ADC Core**的标准API来实现各自的需求。

----
## 4.2 代码文件
ADC驱动对应的文件如下

|                      文件名                      |             说明              |
| ------------------------------------------------ | ----------------------------- |
| **kernel/drivers/infotm/common/adc/adc-imapx.c** | ADC驱动相关文件，兼容所有芯片 |
| **kernel/drivers/infotm/common/adc/core.c**      | ADC驱动相关文件，兼容所有芯片 |
| **kernel/drivers/infotm/common/adc/adc-keys.c**  | 使用ADC API实现的按键事件驱动 |
| **kernel/drivers/infotm/common/power/battery.c** | 使用ADC API实现的电池驱动     |


----
## 4.3 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**iMAPx ADC driver**选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   Infotm ADC support  --->
  |   |   |   |-<*>   iMAPx ADC driver
```

----
## 4.4 场景功能

下面以ADC按键事件为例
```cpp
void adc_keys_test(struct platform_device *pdev)
{
        struct adc_request adc_req;
        struct adc_device adc;
        struct adc_keys_drvdata *ddata;

        //申请ADC资源
        adc_req.id = ddata->id;
        adc_req.mode = ADC_DEV_IRQ_SAMPLE;
        adc_req.label = "adc-keys";
        adc = adc_request(&adc_req);

        //设置ADC采样中断触发条件
        adc_set_irq_watermark(adc, irq_watermark);

        //申请ADC中断，中断处理函数是adc_keys_isr
        devm_request_irq(&pdev->dev, adc->irq, adc_keys_isr,
                         0, "adc-keys", ddata);
        //打开ADC控制器
        adc_enable(adc);

        /* adc_keys_isr */

        //关闭ADC控制器
        adc_disable(adc);
        //释放ADC资源
        adc_free(adc);
}

irqreturn_t adc_keys_isr(int irq, void *priv) {
        struct adc_keys_drvdata *ddata = priv;
        int irqstate;
        //进入中断处理函数先关闭ADC控制器
        adc_disable(ddata->adc);

        //获取ADC采样中断状态，如果有采样中断，进入下一步处理
        irqstate = adc_get_irq_state(ddata->adc);
        if(irqstate) {
                //有采样中断，获取ADC采样值，采样值的低32位数据保存在参数`val`，
                //高32位数据保存在参数`val1`，之后上报这个值
                adc_read_raw(ddata->adc, &val, &val1);

                if(!ddata->key_pressed) {
                        key_index = adc_find_keyindex_bysample(ddata, val);
                        if ( key_index == -1) {
                                debounced = true;
                                goto out;
                        }
                        ddata->bdata[key_index].state = true;
                        ddata->key_pressed = true;
                        input_event(ddata->input, EV_KEY, ddata->bdata[key_index].code, 1);
                        input_sync(ddata->input);
                        pr_info("[adc-key] down: %s\n", ddata->bdata[key_index].desc);
                }

        }

        //中断处理函数返回前打开ADC控制器
        adc_enable(ddata->adc);
        return IRQ_HANDLED;
}
```

----
## 4.5 API

----
### 4.5.1 adc_request()
```cpp
struct adc_device *adc_request(struct adc_request *req);
```
这个函数主要是设备驱动向ADC Core申请ADC资源。

**参数**

- `req` - **`IN`** 设备需要申请的ADC资源的信息

**返回**

该函数成功返回申请到的ADC资源（`adc_device`结构体），失败返回空指针。

----
### 4.5.2 adc_free()
```cpp
void adc_free(struct adc_device *adc);
```
这个函数主要是设备驱动向ADC Core释放ADC资源。

**参数**

- `adc` - **`IN`** ADC设备信息

**返回**

无返回值

----
### 4.5.3 adc_enable()
```cpp
int adc_enable(struct adc_device *adc);
```
这个函数是给设备驱动来打开ADC控制器。

**参数**

- `adc` - **`IN`** ADC设备信息

**返回**

该函数成功返回0，失败返回错误码。


----
### 4.5.4 adc_disable()
```cpp
void adc_disable(struct adc_device *adc);
```
这个函数是给设备驱动来关闭ADC控制器。

**参数**

- `adc` - **`IN`** ADC设备信息

**返回**

无返回值。

----
### 4.5.5 adc_read_raw()
```cpp
int adc_read_raw(struct adc_device *adc, int *val, int *val1);
```
这个函数是给设备驱动来获取ADC采样值，采样值的低32位数据保存在参数`val`，高32位数据保存在参数`val1`。

**参数**

- `adc` - **`IN`** ADC设备信息
- `val` - **`OUT`** ADC采样值的低32位数据
- `val1` - **`OUT`** ADC采样值的低32位数据

**返回**

该函数成功返回0，失败返回错误码。

----
### 4.5.6 adc_set_irq_watermark()
```cpp
int adc_set_irq_watermark(struct adc_device *adc, int watermark);
```
这个函数是给设备驱动来设置ADC采样中断触发条件，一般是电压值。

**参数**

- `adc` - **`IN`** ADC设备信息
- `watermark` - **`IN`** ADC采样中断触发条件对应的电压值


**返回**

该函数成功返回0，失败返回错误码。

----
### 4.5.7 adc_get_irq_state()
```cpp
int adc_get_irq_state(struct adc_device *adc);
```
这个函数是设备驱动获取ADC采样中断状态。

**参数**

- `adc` - **`IN`** ADC设备信息

**返回**

0表示没有产生ADC采样中断，1表示有ADC采样中断。


----
## 5 PWM

PWM全称Pulse-Width Modulation，即脉冲宽度调制，用来根据输入时钟，调制出特定占空比，周期和极性的脉冲波形。芯片的PWM模块每一路Timer都支持0-100%的占空比输出，支持极性翻转，被用于背光调节和嗡鸣器。

----
## 5.1 驱动架构
![pwm_driver_frame](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/pwm_driver_frame.svg)

PWM驱动架构分为**设备驱动**，**PWM Core**和**PWM驱动**３个部分。**PWM驱动**为**PWM Core**实现PWM控制器硬件功能的细节，这样**设备驱动**可以调用**PWM Core**的标准API来实现各自的需求。

----
## 5.2 代码文件
PWM驱动对应的文件如下

|                          文件名                           |             说明              |
| --------------------------------------------------------- | ----------------------------- |
| **kernel/drivers/infotm/common/pwm/pwm-imapx.c**          | PWM驱动相关文件，兼容所有芯片 |
| **kernel/drivers/infotm/common/pwm/backlight/imapx_bl.c** | 使用PWM API实现的背光驱动     |
| **kernel/drivers/infotm/common/pwm/imapx_beep.c**         | 使用PWM API实现的嗡鸣器驱动   |


----
## 5.3 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**iMAPx PWM driver**选项

```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   Infotm pwm support  --->
  |   |   |   |-<*>  iMAPx PWM driver
```

----
## 5.4 场景功能

下面场景使用PWM第0路控制器输出波形设置背光
```cpp
void pwm_backlight_test(struct platform_device *pdev)
{
        struct pwm_device *pwm;

        //申请PWM第0路控制器资源
        pwm = pwm_request(0, "lcd-backlight");

        //设置PWM输出波形的极性
        if (polarity==PWM_POLARITY_NORMAL)
                pwm_set_polarity(pwm,PWM_POLARITY_NORMAL);
        else if (polarity==PWM_POLARITY_INVERSED)
                pwm_set_polarity(pwm,PWM_POLARITY_INVERSED);

        //配置PWM输出波形的占空比和周期，duty_cycle是通电时间，
        //period是总时间
        pwm_config(pwm, duty_cycle, period);
        //打开PWM控制器
        pwm_enable(pwm);

        //关闭PWM控制器
        pwm_disable(pwm);
        //释放PWM资源
        pwm_free(pwm);
}
```

----
## 5.5 API

### 5.5.1 pwm_request()
```cpp
struct pwm_device *pwm_request(int pwm_id, const char *label);
```
这个函数主要是设备驱动向PWM Core申请PWM资源。

**参数**

- `pwm_id` - **`IN`** PWM id号
- `label` - **`IN`** PWM设备标签

**返回**

该函数成功返回申请到的PWM资源（`pwm_device`结构体），失败返回空指针。

### 5.5.2 pwm_free()
```cpp
void pwm_free(struct pwm_device *pwm);
```
这个函数主要是设备驱动向PWM Core释放PWM资源。

**参数**

- `pwm` - **`IN`** PWM设备资源


**返回**

无返回值。

### 5.5.3 pwm_config()
```cpp
int pwm_config(struct pwm_device *pwm, int duty_ns, int period_ns);
```
这个函数主要是设备驱动配置PWM输出波形的占空比和周期。

**参数**

- `pwm` - **`IN`** PWM设备资源
- `duty_ns` - **`IN`** PWM输出波形一个周期内的通电时间（单位ns）
- `period_ns` - **`IN`** PWM输出波形一个周期内的总时间（单位ns）

**返回**

该函数成功返回0，失败返回错误码。

### 5.5.4 pwm_set_polarity()
```cpp
int pwm_set_polarity(struct pwm_device *pwm, enum pwm_polarity polarity);
```
这个函数主要是设备驱动配置PWM输出波形的极性，也就是配置高低电平是否翻转。

**参数**

- `pwm` - **`IN`** PWM设备资源
- `polarity` - **`IN`** PWM输出波形的极性：PWM_POLARITY_NORMAL-通电周期为高电平；PWM_POLARITY_INVERSED-通电周期为低电平


**返回**

该函数成功返回0，失败返回错误码。

### 5.5.5 pwm_enable()
```cpp
int pwm_enable(struct pwm_device *pwm);
```
这个函数主要是设备驱动打开PWM控制器。

**参数**

- `pwm` - **`IN`** PWM设备资源


**返回**

该函数成功返回0，失败返回错误码。

### 5.5.6 pwm_disable()
```cpp
void pwm_disable(struct pwm_device *pwm);
```
这个函数主要是设备驱动关闭PWM控制器。

**参数**

- `pwm` - **`IN`** PWM设备资源


**返回**

无返回值。


----
## 6 IIC

IIC全称Inter-Integrated Circuit，即集成电路总线，这种总线类型是由飞利浦半导体公司在八十年代初设计出来的一种简单、双向、二线制、同步串行总线，主要是用来连接整体电路。芯片的IIC总线支持支持标准(100Kb/s)，快速(400Kb/s)，和高速(3.4Mb/s)三种传输模式，SDK默认采用的是快速(400Kb/s)传输模式（兼容标准模式，不兼容快速模式），主要用来控制Gsensor，PMU和CMOS-Sensor等外部器件。

----
## 6.1 驱动架构
![iic_driver_frame](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/iic_driver_frame.svg)

IIC驱动架构分为**设备驱动**，**IIC Core**和**IIC驱动**３个部分。**IIC驱动**为**IIC Core**实现IIC控制器硬件功能的细节，这样**设备驱动**可以调用**IIC Core**的标准API来实现各自的需求。

----
## 6.2 代码文件
IIC驱动对应的文件如下

|                   文件名                   |             说明              |
| ------------------------------------------ | ----------------------------- |
| **kernel/drivers/infotm/common/i2c/i2c.c** | IIC驱动相关文件，兼容所有芯片 |
| **kernel/drivers/infotm/q3f/mfd/ip620x.c** | 使用IIC API实现的PMU（PMU型号为**IP620x**）驱动 |


----
## 6.3 配置选项

在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**iMAPx I2C driver**选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*> Infotm common drivers support   --->
  |   |   |-[*] Infotm I2C support  --->
  |   |   |   |-<*>  iMAPx I2C driver
```


----
## 6.4 场景功能
下面场景是读写PUM单个寄存器，读写寄存器地址`0x01`，要写入的值是`0`
```cpp
void iic_read_write_test(void)
{
        int value = 0;
        ip620x_i2c_read(ip620x_i2c_client, 0x01, &value);
        value &= 0x0;
        ip620x_i2c_write(ip620x_i2c_client, 0x01, value);
}
```
```cpp
static int ip620x_i2c_write(struct i2c_client *client,
                            uint8_t reg, uint16_t val)
{
        int ret;
        uint8_t buf[3];
        struct i2c_msg msg;

        buf[0] = reg;
        buf[1] = (uint8_t)((val & 0xff00) >> 8);
        buf[2] = (uint8_t)(val & 0xff);

        msg.addr = client->addr;
        msg.flags = 0;
        msg.buf = buf;
        msg.len = 3;

        ret = i2c_transfer(client->adapter, &msg, 1);
        if (ret != 1) {
                pr_err("failed to write 0x%x to 0x%x\n", val, reg);
                return -1;
        }

        return 0;
}
```
```cpp
static int ip620x_i2c_read(struct i2c_client *client,
                           uint8_t reg, uint16_t *val)
{
        int ret;
        uint8_t buf[2];
        struct i2c_msg msg[2];

        msg[0].addr = client->addr;
        msg[0].flags = 0;
        msg[0].buf = &reg;
        msg[0].len = 1;

        msg[1].addr = client->addr;
        msg[1].flags = I2C_M_RD;
        msg[1].buf = buf;
        msg[1].len = 2;

        ret = i2c_transfer(client->adapter, msg, 2);
        if (ret != 2) {
                pr_err("failed to read 0x%x\n", reg);
                return -1;
        }

        *val = (buf[0] << 8) | buf[1];

        return 0;
}
```

----
## 6.5 API
IIC驱动在**i2c.c**里面为IIC Core实现的API如下

### 6.5.1 i2c_transfer()
```cpp
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num);
```
这个函数主要是IIC设备驱动来处理一条或者多条IIC信息，可以用来实现对IIC从设备（比如PMU）信息的读写。

**参数**

- `adap` - **`IN`** IIC控制器信息
- `msgs` - **`IN`** | **`OUT`** 要处理的一条或者多条IIC信息，具体信息请参考后面结构体成员描述
- `num` - **`IN`** IIC信息数量

**数据结构**

`msgs`对应的`i2c_msg`结构体如下
```cpp
struct i2c_msg {
        __u16 addr; /* slave address            */
        __u16 flags;
#define I2C_M_TEN       0x0010  /* this is a ten bit chip address */
#define I2C_M_RD        0x0001  /* read data, from slave to master */
#define I2C_M_STOP      0x8000  /* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_NOSTART       0x4000  /* if I2C_FUNC_NOSTART */
#define I2C_M_REV_DIR_ADDR  0x2000  /* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_IGNORE_NAK    0x1000  /* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_NO_RD_ACK     0x0800  /* if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_RECV_LEN      0x0400  /* length will be first received byte */
        __u16 len;      /* msg length               */
        __u8 *buf;      /* pointer to msg data          */
};
```
**成员**

| 成员 | 描述 |
| --- | --- |
| `addr` | IIC设备地址 |
| `flags` | IIC消息标志，参考上述宏定义解释 |
| `len` | 消息长度 |
| `buf` | 消息数据`buf` |

**返回**

该函数成功返回处理的IIC信息数量，失败返回错误码。

----
## 6.6 调试方法

在SDK的根目录下输入`make menuconfig`命令
```bash
make menuconfig
```
进入buildroot配置选项，确保已经选择**i2c-tools**选项
```bash
Target packages --->
  |-Hardware handling --->
  |   |-<*> i2c-tools
```
启动系统后，IIC驱动会作为一个字符设备在/dev/下生成节点
```
/ # ls /dev/i2c-*
/dev/i2c-0  /dev/i2c-1  /dev/i2c-2  /dev/i2c-3
```
可以通过**iic-tool**工具来进行调试
### Step 1: 列举IIC控制器

使用`i2cdetect`命令列举所有IIC控制器
```
/ # i2cdetect -l
i2c-0   i2c             Imapx800 I2C adapter                    I2C adapter
i2c-1   i2c             Imapx800 I2C adapter                    I2C adapter
i2c-2   i2c             Imapx800 I2C adapter                    I2C adapter
i2c-3   i2c             Imapx800 I2C adapter                    I2C adapter
```
### Step 2: 打印PMU所有寄存器值

使用`i2cdump`命令打印PMU所有寄存器值，PMU使用的是`i2c-0`，设备地址是`0x30`
```
/ # i2cdump -y -f 0 0x30
No size specified (using byte-data access)
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f    0123456789abcdef
00: c3 00 7f 00 3f d3 00 0a 00 00 00 00 02 00 00 00    ?.?.??.?....?...
10: 00 f1 ff 4f ff 00 00 00 00 00 12 03 00 00 00 00    .?.O......??....
20: 81 07 7c 06 00 00 00 00 50 00 00 00 8e 00 00 00    ??|?....P...?...
30: 02 00 1f 00 55 00 00 00 00 00 00 00 00 00 00 00    ?.?.U...........
40: 00 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00    ....?...........
50: 3d 00 00 00 00 00 15 30 a8 1b 00 15 30 a8 1b 00    =.....?0??.?0??.
60: 15 30 a8 1b 00 15 30 a8 1b 00 15 30 a8 1b 00 15    ?0??.?0??.?0??.?
70: 30 a8 1b 77 77 fe 88 00 00 00 00 00 00 00 00 00    0??ww??.........
80: 00 45 01 00 00 00 00 00 00 00 00 00 00 00 00 00    .E?.............
90: 09 0b 0b 0b 0b 0b 0c 0c 0c 0c 0c 0c 0c 0c 0d 0d    ????????????????
a0: 00 40 7f 00 00 01 00 00 00 00 00 00 0f 00 00 00    .@?..?......?...
b0: 00 00 00 00 00 00 00 00 ff 55 aa 00 00 00 00 00    .........U?.....
c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
d0: 86 00 01 12 12 12 00 e0 01 be 02 00 00 00 00 00    ?.????.????.....
e0: 1f 00 00 fd 00 00 00 00 00 00 00 00 00 00 00 00    ?..?............
f0: 00 12 36 07 07 00 00 00 00 00 00 00 00 00 00 00    .?6??...........
```
### Step 3: 读写PMU单个寄存器

使用`i2cget`和`i2cset`命令读写PMU单个寄存器，PMU使用的是`i2c-0`，设备地址是`0x30`，读写寄存器地址`0x01`，要写入的值是`0`
```
/ # i2cset -y -f 0 0x30 0x01 0x00
/ # i2cget -y -f 0 0x30 0x01
0x00
```
Note: 使用`i2cset`命令的时候要确保寄存器是可写的。

----
## 7 Power
这里Power主要的含义是给直流供电或者电池供电，其控制逻辑由Power模块进行管理。需要注意的是下文描述的电池电量检测都是基于ADC采样。



----
## 7.1 驱动架构

![power_driver_frame](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/power_driver_frame.svg)

**Battery驱动**为**Power Core**实现Battery硬件配置细节，这样**Power Core**生成Battery对应的**sysfs调试接口**，以便查看和设置Battery。

----
## 7.2 代码文件
Power驱动文件如下

|                      文件名                      |           说明            |
| ------------------------------------------------ | ------------------------- |
| **kernel/drivers/infotm/common/power/battery.c** | 使用ADC API实现的电池驱动 |


----
## 7.3 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**iMAPx battery driver**选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   InfoTM power driver support  --->
  |   |   |   |-<*>  iMAPx battery driver
```


打开**products/q3fevb_va/items.itm**文件，根据**Q3F-EVB**原理图中Battery电路图所示，Power相关选项配置如下

![iic_driver_frame](battery_hardware.png)

```cpp
power.ctrl adc.10.2000
power.full pads.130
power.dcdetect pads.139
power.curve.charge      4169.4150.4130.4101.4067.4013.3950.3916.3857.3784.3720.3666
power.curve.discharge   4000.3902.3785.3725.3681.3615.3554.3496.3441.3373.3323.3300
```
| 名称 | 类型 | 说明 |
| ---- | ---- |---|
| `power.ctrl` | LIST | 电池电压的采样设置:[采样总线方式].[采样总线编号].[采样电阻分压比] |
| `power.full` | PADS | 充电满信号检测管脚, 输入管脚，电池充满电则拉高 |
| `power.dedetect` | PADS | 外接电源信号检测管脚, 输入管脚， 有外接电源则拉高 |
| `power.curve.charge` | ARRAY | 有外接电源情况下，电池电压与电量的对应表，分别对应[100%.90%.80%.70%.60%.50%.40%.30%.20%.15%.5%.0%]这几个电量下对应的电压值 |
| `power.curve.discharge` | ARRAY | 无外接电源情况下，电池电压与电量的对应表，分别对应[100%.90%.80%.70%.60%.50%.40%.30%.20%.15%.5%.0%]这几个电量下对应的电压值 |


----
## 7.4 调试方法
启动系统后，Battery驱动会在sys目录下面生成调试节点，节点目录如下
```
/sys/class/power_supply/battery/
/sys/class/power_supply/ac/
```
其中`battery`目录对应的是电池处于不充电状态，`ac`目录对应的是电池处于充电状态。可以使用`cat`命令分别查看电池电量和充电状态
```
/ # cat /sys/class/power_supply/battery/capacity
99
/ # cat /sys/class/power_supply/battery/voltage_now
5009
/ # cat /sys/class/power_supply/ac/online
1
```

----
## 8 SDIO
SDIO全称Secure Digital Input and Output，即安全数字输入输出，在这种物理接口下定义了３种应用协议，即eMMC,SD,SDIO。我们的产品主要用了后两种协议，分别支持TF卡和SDIO Wi-Fi设备。

----
## 8.1 代码文件

|                          文件名                          |                    说明                    |
| -------------------------------------------------------- | ------------------------------------------ |
| **kernel/drivers/infotm/common/mmc/dw_mmc_280/dw_mmc.c** | `Q3F`和`Q3-ECO`芯片对应的SDIO驱动 |


----
## 8.2 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，先确保已经选择**MMC Host controller select**相关选项，这里**DW_MMC Version2.80**版本的SDIO驱动是给 `Q3F`，`Q3-ECO`芯片用的。
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   [*]   MMC/SD/SDIO Host Controller Drivers  --->
  |   |   |   |-MMC Host controller select (DW_MMC Version2.80)  --->
  |   |   |   |   |-( ) DW_MMC Version2.40
  |   |   |   |   |-(X) DW_MMC Version2.80
```


还需要选择**InfoTM MMC0(eMMC) Driver enable**，**InfoTM MMC1(SD/TF) Driver enable**和**InfoTM MMC2(SDIO) Driver enable**相关选项，这里**InfoTM MMC0(eMMC) Driver enable**选项对应**eMMC**功能，使用MMC0路控制器；**InfoTM MMC1(SD/TF) Driver enable**选项对应**SD/TF**功能，使用MMC1路控制器；**InfoTM MMC2(SDIO) Driver enable**选项对应**SDIO** Wi-Fi功能，使用MMC2路控制器.
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   [*]   MMC/SD/SDIO Host Controller Drivers  --->
  |   |   |   |-MMC Host controller select (DW_MMC Version2.80)  --->
  |   |   |   |-< >   InfoTM MMC0(eMMC) Driver enable
  |   |   |   |-<*>   InfoTM MMC1(SD/TF) Driver enable
  |   |   |   |-<*>   InfoTM MMC2(SDIO) Driver enable
```

Note: 这里**eMMC**，**SD/TF**，**SDIO**选项一定要确认用到相关功能才选上，没有用到一定不能选，否则会导致系统启动报错。

----
## 8.3 场景功能

项目上针对SDIO的时钟调试几乎都是接口时钟调试。SDIO协议规定标准速度模式频率不超过25MHz，高速模式频率上限50MHz，下面以设置MMC2路控制器时钟频率为50MHz为例
```cpp
void mmc2_set_rate(struct dw_mci *host)
{
        const struct dw_mci_drv_data *drv_data = host->drv_data;
        struct dw_mci *mmc_host = host;

        //获取设备时钟源
        host->ciu_clk = clk_get_sys("imap-mmc.2", "sd-mmc2");

        //先设置设备父时钟源是"epll"，然后设置时钟频率为50MHz，
        //最后读取时钟频率看是否设置成功
        clk_set_parent(mmc_host->ciu_clk,clk_get_sys("epll","epll"));
        clk_set_rate(mmc_host->ciu_clk,50000000);
        mmc_host->bus_hz = clk_get_rate(mmc_host->ciu_clk);
        if(mmc_host->bus_hz != 50000000)
                printk("set freq err!\n");

        //打开设备时钟源
        clk_prepare_enable(host->ciu_clk);
}
```

----
## 9 Audio
我们当前使用的Audio是主流音频体系的ALSA框架，由内核提供alsa-driver，由应用程序调用alsa-lib提供的API，即可完成对底层音频硬件的控制。

----
## 9.1 代码文件
Audio驱动在内核中主要分为三大块：**soc-core**、**platform-driver**、**codec-driver**，代码目录如下


|                   文件                    |              说明               |
| ------------------------------------------- | ------------------------------- |
| **kernel/sound/core/**  |   代码与具体的平台无关，是与上层Audio库之间的接口，是Audio驱动的核心模块   |
| **kernel/drivers/infotm/common/sound/configuration/** | Codec设备与CPU之间的交互，通常是IIS传输或者PCM传输   |
|**kernel/drivers/infotm/common/sound/codec/** | Codec的设备驱动，每一个驱动对应着一个或者一种codec设备，不同的Codec驱动存在着差异性 |

----
## 9.2 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**SoC Audio for the Infotm Imapx chips**相关选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-[*]   Infotm SoC Audio  --->
  |   |   |   |-[ ] playback auto open and close speaker
  |   |   |   |-<*> SoC Audio for the Infotm Imapx chips
  |   |   |   |-<*> SoC Audio support for IMAPX - ES8328
  |   |   |   |-<*> SoC Audio support for IMAPX - ES8323
  |   |   |   |-<*> SoC Audio support for IMAPX - ALC5631
  |   |   |   |-<*> SoC Audio support for IMAPX - Acodec
  |   |   |   |-<*> SoC Audio support for IMAPX - Fr1023
  |   |   |   |-<*> Soc Audio support for IMAPX - IP620x
  |   |   |   |-< > Soc Audio support for IMAPX - IP2906
  |   |   |   |-<*> SoC Audio support for IMAPX - Spdif
  |   |   |   |-<*> SoC Audio support for IMAPX - Virtual
  |   |   |   |-<*> SoC Audio transfer for IMAPX - IIS
  |   |   |   |-<*> SoC Audio transfer for IMAPX - PCM
  |   |   |   |-<*> SoC Audio memcpy for IMAPX - DMA pl330
  |   |   |   |-[ ]     auto enable spk
  |   |   |   |-< >     support adc for tqlp9624
```

在具体product目录下配置item文件里面的Codec选项，Codec选项说明如下

| ITEMs | Value | Description |
|---|---|---|
| `codec.model` | `ip6205` | 设置codec模组名称；  <br><br>目前支持`9624tqlp`、`ip6205`、`fr1023`、`es8323`等 |
| `codec.data` | `i2s.0` | 设置codec数据通道；  <br><br>`i2s.0`：使用对应芯片的IIS通道0进行数据传输 |
| `codec.power` | `pmu.ldo1` | 设置codec供电通道;  <br><br>`pmu.ldo1`：供电通道是PMU的LDO1 |
| `sound.speaker` | `pads.160` | 设置控制喇叭开关的IO管脚；  <br><br>`pads.160`：使用对应芯片IO PAD 160 |



----
## 10 DPMU
DPMU全称DDR Power Management Unit，也就是DDR电源管理单位，该功能可以查看每个DDR Port口（对应一路Bus）中的设备访问DDR位宽的使用情况，同时可以控制每一路Bus中单个设备的开关，从而达到监控每个设备的DDR访问带宽。

----
## 10.1 代码文件

|                      文件名                      |             说明              |
| ------------------------------------------------ | ----------------------------- |
| **kernel/drivers/infotm/q3f/dpmu/dpmu.c**     | `Q3F`芯片对应的DPMU驱动   |
| **kernel/drivers/infotm/apollo3/dpmu/dpmu.c** | `Q3-ECO`芯片对应的DPMU驱动 |

----
## 10.2 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保已经选择**Dram PMU func add**选项
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm Q3F Series Support --->
  |   |   |-[*]   Dram PMU func add
```

----
## 10.3 调试方法
启动系统后，DPMU驱动会生成一个可供调试的节点**/dev/dpmu**，调试方法如下

查看最近8次每个Bus DDR平均使用情况
```
/ # echo dpmu bus > /dev/dpmu
[  263.613333] =============************cpu & bus dpmu user anysis start************==============
[  263.616666]  cpu:  W:    0.00MB/s    R:    0.07MB/s
[  263.619999] bus1:  W:    0.00MB/s    R:    0.00MB/s
[  263.623333] bus3:  W:    0.00MB/s    R:    0.00MB/s
[  263.626666] bus4:  W:    0.00MB/s    R:    0.00MB/s
[  263.629999] bus6:  W:    0.00MB/s    R:    0.00MB/s
[  263.633333] =============************cpu & bus dpmu user anysis end************==============
```

查看当前一次每个Bus和Bus挂载设备的DDR使用情况
```
/ # echo dpmu all > /dev/dpmu
[  370.743333] cpu :  W:    0.00MB/s    R:    0.07MB/s
[  370.746666] bus1:  W:    0.00MB/s    R:    0.00MB/s
[  370.749999]  ISP     :W:    0.00MB/s   R:    0.00MB/s
[  370.753333]  CAMIF   :W:    0.00MB/s   R:    0.00MB/s
[  370.756666]  MIPICSI :W:    0.00MB/s   R:    0.00MB/s
[  370.759999]  IDS     :W:    0.00MB/s   R:    0.00MB/s
[  370.763333] bus3:  W:    0.00MB/s    R:    0.00MB/s
[  370.766666]  FODET   :W:    0.00MB/s   R:    0.00MB/s
[  370.769999]  ISPOST  :W:    0.00MB/s   R:    0.00MB/s
[  370.773333] bus4:  W:    0.00MB/s    R:    0.00MB/s
[  370.776666]  VENC_H1 :W:    0.00MB/s   R:    0.00MB/s
[  370.779999] bus6:  W:    0.00MB/s    R:    0.00MB/s
[  370.783333]  SUB_BUS :W:    0.00MB/s   R:    0.00MB/s
[  370.786666]  GDMA    :W:    0.00MB/s   R:    0.00MB/s
[  370.789999]  Ethernet:W:    0.00MB/s   R:    0.00MB/s
[  370.793333]  DSP     :W:    0.00MB/s   R:    0.00MB/s
[  370.796666]  CRYPTO  :W:    0.00MB/s   R:    0.00MB/s
```

查看设备名为`ISP`（该名称可以改为前面Log中出现的任何设备名称）最近8次的DDR的平均使用情况
```
/ # echo dpmu device ISP >/dev/dpmu
[  511.496666] bus1:0:  ISP:W:    0.00MB/s   R:    0.00MB/s
```

----
## 11 USB
USB全称Universal Serial Bus，即通用串行总线。在SDK中，USB控制器驱动(USB HCD)主要使用了OHCI、EHCI两种USB接口规范，OHCI支持全速和低速设备；EHCI支持高速设备。USB主要有**USB-Host**和**USB-Slave**模式，支持U盘，UVC，USB ETH，USB HID等功能。

----
## 11.1 配置选项
在SDK的根目录下输入`make linux-menuconfig`命令
```bash
make linux-menuconfig
```
进入linux内核配置选项，确保如下USB HCD相关配置选项已经选上
```bash
Device Drivers --->
  |-[*] USB support  --->
  |   |-<*>   Support for Host-side USB
  |   |   |-<*>     EHCI HCD (USB 2.0) support
  |   |   |-<*>     OHCI HCD support
```
然后根据不同功能使用不同的配置

**USB-Host和USB-Slave**

USB主要有**USB-Host**和**USB-Slave**模式，有如下三个配置选项，可以任选**HOST ONLY MODE**（**USB-Host**），**DEVICE ONLY MODE**（**USB-Slave**）或者**HOST AND DEVICE MODE**，这里需要注意的是后续配置的**U盘**和**UVC-Host**工作在**USB-Host**模式，**USB外设**(包括HID，ETH，UVC-Slave功能)和**USB FastBoot**工作在**USB-Slave**模式
```bash
Device Drivers --->
  |-InfoTM special files and drivers --->
  |   |-<*>   Infotm common drivers support --->
  |   |   |-<*>   Infotm OTG driver support  --->
  |   |   |   |-<*>   Synopsis DWC_OTG support
  |   |   |   |   |-USB Operation Mode (HOST ONLY MODE)  --->
  |   |   |   |   |   |-( ) HOST ONLY MODE
  |   |   |   |   |   |-( ) DEVICE ONLY MODE
  |   |   |   |   |   |-(X) HOST AND DEVICE MODE
```

**U盘**

U盘功能支持需要选上**CONFIG_USB_STORGE**
```bash
Device Drivers --->
  |-[*] USB support  --->
  |   |-<*>   Support for Host-side USB
  |   |   |-<*>    USB Mass Storage support
```

**UVC-Host**

UVC-Host功能支持需要选上**USB Video Class**，**Cameras/video grabbers support**，**Media USB Adapters**等选项
```bash
Device Drivers --->
  |-<*> Multimedia support  --->
  |   |-[*]   Cameras/video grabbers support
  |   |-[*]   Media Controller API
  |   |-[*]   V4L2 sub-device userspace API
  |   |-[*]   Media USB Adapters  --->
  |   |   |-<*>   USB Video Class (UVC)
  |   |   |-[*]     UVC input events device support (NEW)
  |   |-[*]   V4L platform devices  --->
  |   |-[*]   Memory-to-memory multimedia devices  --->
```

**USB外设**

USB外设功能主要是USB HID，USB ETH和UVC-Slave功能，这些功能支持需要选上**USB Gadget Support**和**Infotm Composite Gadget**
```bash
Device Drivers --->
  |-[*] USB support  --->
  |   |-<*>   USB Gadget Support  --->
  |   |   |-<*>   USB Gadget Drivers (Infotm Composite Gadget)  --->
```

**USB FastBoot**

USB FastBoot功能主要是使用USB数据线连接设备的一种刷机模式，需要选上**USB FFS Inject Early For Fastboot**
```bash
Device Drivers --->
  |-[*] USB support  --->
  |   |-<*>   USB Gadget Support  --->
  |   |   |-<*>   USB FFS Inject Early For Fastboot
```

----
## 11.2 调试方法

系统启动后，可以看到**sys/class/infotm_usb**调试目录，对该目录下的相关调试节点输入不同的调试命令打开不同的功能


**支持U盘等存储设备**
输入以下命令
```
/ # echo 0 > sys/class/infotm_usb/infotm0/enable
[  909.593333] WARN::ep_dequeue:421: bogus device state
[  909.593333]
[  909.596666] infotm_usb gadget: uvc_function_unbind
[  909.603333] WARN::ep_dequeue:421: bogus device state
[  909.603333]
/ # echo mass_storage > sys/class/infotm_usb/infotm0/functions
/ # echo 1 > sys/class/infotm_usb/infotm0/enable
[  925.516666] # infotm_enable enum_ready:1
```

**支持UVC-Slave**
输入以下命令
```
/ # echo 0 > sys/class/infotm_usb/infotm0/enable
[  854.723333] WARN::ep_dequeue:421: bogus device state
[  854.723333]
[  854.729999] WARN::ep_dequeue:421: bogus device state
[  854.729999]
/ # echo uvc > sys/class/infotm_usb/infotm0/functions
/ # echo 1 > sys/class/infotm_usb/infotm0/enable
[  866.246666] # infotm_enable enum_ready:0
[  866.249999] infotm_usb gadget: uvc_function_bind
[  866.256666] infotm_enable: waiting usb gadget connect
```

**支持USB ETH**
输入以下命令
```
/ # echo 0 > sys/class/infotm_usb/infotm0/enable
[  368.849999] WARN::ep_dequeue:421: bogus device state
[  368.849999]
/ # echo rndis > sys/class/infotm_usb/infotm0/functions
/ # echo 1 > sys/class/infotm_usb/infotm0/enable
[  393.456666] # infotm_enable enum_ready:1
[  393.459999] rndis_function_bind_config MAC: 00:00:00:00:00:00
[  393.463333] infotm_usb gadget: using random self ethernet address
[  393.466666] infotm_usb gadget: using random host ethernet address
[  393.479999] usb0: MAC b2:92:b3:94:91:3b
[  393.483333] usb0: HOST MAC 8e:23:1e:c8:00:37
```

**支持USB HID**
输入以下命令
```
/ # echo 0 > sys/class/infotm_usb/infotm0/enable
[  708.306666] WARN::ep_dequeue:421: bogus device state
[  708.306666]
/ # echo hid > sys/class/infotm_usb/infotm0/functions
/ # echo 1 > sys/class/infotm_usb/infotm0/enable
[  721.783333] # infotm_enable enum_ready:1
```

**支持USB FastBoot**
输入以下命令
```
/ # echo 0 > sys/class/infotm_usb/infotm0/enable
[ 1003.909999] WARN::ep_dequeue:421: bogus device state
[ 1003.909999]
/ # echo ffs > sys/class/infotm_usb/infotm0/functions
/ # echo 1 > sys/class/infotm_usb/infotm0/enable
[ 1012.719999] # infotm_enable enum_ready:1
```
