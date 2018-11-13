### 术语解释
|术语|解释|
|-----------------|----------------------------------|
|ISP|  Image Signal Processing|
|IPU | Image Processing Units |
|V2505|  盈方微Q3F，Q3-ECO系列ISP模块代号|
|DDK | ISP设备开发套件 |
|ISPC| ISP Control library |
|Felix| ISP Linux驱动名字|
|API| Application Programming Interface|
|3A|ISP自动曝光，自动白平衡，自动对焦的软件控制策略|
|LSH|Sensor Lens Shading Correction|
|IQ|Image Quality|
|CAMIF|Camera Interface，图像Sensor的YUV格式接口|
|MIPI|Mobile Industry Processor Interface，Sensor的图像数据串行传输接口｜
|DVP|Digital Video Port，Sensor图像数据并口传输接口|
|VSYNC|帧同步信号|
|HSYNC|行同步信号|
|CSI|Camera Serial Interface|
|PHY|Port Physical Layer|
|HS|High Speed，MIPI PHY的高速传输数据模式|
|LP|Low Power，MIPI PHY的低功耗模式，用于传输控制信号|
|CRC|Cyclic Redundancy Check|
|PMU|Power Manage Unit|
|HAL|硬件抽象层|


----------
## 1 概述
Sensor作为一种图像输入器件，是将自然光信号转为电信号。由于不同的项目因应用需求不同，接入的Sensor也不同，所以调试与接入Sensor也是日常工作中经常碰到的，那么怎样在InfoTM QSDK中正确的接入新Sensor呢？ 本文的宗旨就是引导工程师正确的接入新的Sensor到QSDK中。

---------
## 2 软件架构

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/qsdk_camera_soft_structure.svg "camera software sketch")

|结构|描述|
|---|----|
|V2500 IPU| Videobox软件结构上的最上层，提供API给应用程序调用，其向下调用DDK的API|
|DDK|为ISP的整个软件开发套件，ISP 3A控制也在这里面，向上提供API给V2500 IPU调用，向下调用Sensor的HAL和ISP驱动来控制Sensor与ISP|
|**hlibcamsensor**|为Sensor的HAL实现，这也是在QSDK中添加新的Sensor主要修改的地方。编译后会生成`libcamsensor.so`的动态库，被DDK所调用|
|Felix.ko|ISP的内核的驱动，被DDK所调用|
|ddk_sensor.ko|Sensor的内核驱动，主要是给Sensor提供电压，时钟和上电时序|

-------------
## 3 系统流程
如下图所示，`videoboxd`运行时，`videoboxd`从**/root/.videobox/**系统目录中读取运行时需要的IPU模块配置文件，从**/root/.ispddk/**系统目录中读取Sensor，ISP的配置文件，通过`/dev/imgfelix0`获取视频数据，通过`/dev/ddk_sensor0`来控制ISP。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/camera_system_follow.svg "camera system follow")

--------------
## 4 添加支持Sensor
QSDK中已经支持了一些Sensor，如果在项目中使用这些Sensor，可按照4.1中的步骤选用这些Sensor，以下为QSDK中默认支持的Sensor。

|Sensor|分辨率|Lane数|帧数|
|---|----|---|---|
|AR330 MIPI|1920x1080|1|30|
|AR330 DVP|1920x1080||30|
|OV4689 MIPI|1920x1080|1|30|
|OV2710 MIPI|1920x1080|1|15|
|IMX322 DVP|1920x1080||30|
|IMX179 MIPI|2624x2624|4|30|
|SC2135 DVP|1920x1080||30|
------------
### 4.1 添加单Sensor
盈方微Q3系列芯片支持多个物理Sensor的同时输入，一般应用场景只需要一个Sensor，所谓单Sensor就是指只有一个物理Sensor作为图像输入。假如有产品项目名为example，要用到单AR330 MIPI Sensor，怎样将其添加到项目example中呢。
#### Step 1: 建立Sensor配置文件目录
建立example项目目录后，在项目目录下建立Sensor配置文件目录。
```
$ mkdir products/example/isp
$ mkdir products/example/isp/json
$ mkdir products/example/sensor0
$ mkdir products/example/sensor0/ar330mipi
```
#### Step 2: 复制相关配置文件
默认支持的Sensor配置文件放在**system/hlibcamsensor/sensor_cfg**目录下。
**有屏**
```
$ cp system/hlibcamsensor/sensor_cfg/json/sensor_ids.json products/example/isp/json/ar330_path.json
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330mipi/* products/example/sensor0/ar330mipi
```
**无屏**
```
$ cp system/hlibcamsensor/sensor_cfg/json/sensor_h1264.json products/example/isp/json/ar330_path.json
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330mipi/* products/example/sensor0/ar330mipi
```
复制配置文件后example的**isp**目录下的结构如下。
```
├── json
│   └── ar330_path.json
├── sensor0
    └── ar330mipi
        ├── sensor0-config.txt
        ├── sensor0-isp-config.txt
        ├── sensor0.itm
        └── sensor0-lsh-config.lsh
```
#### Step 3: 选择Sensor源代码

```bash
$ make menuconfig
```
选择名为AR330_MIPI选项，使AR330 MIPI Sensor的源代码编译到QSDK的镜像文件中，若默认已选上，或选择的是编译所有的Sensor，可以跳过此步。

选择**build source codes types**选项
```bash
QSDK Options  --->
   Libs  --->
      -*- hlibcamsensor  
         select build source codes types  --->                                                                                   
         select bridge sensors  --->                                                                                                 
         select bridge  --->
```
选择**all sensor list**选项

```bash
[ ] BUILD_ALL_SENSORS
[*]   BUILD_SELECTED_SENSOR  
          all sensor list  --->   
```                    
选择**AR330_MIPI**选项

```bash
[ ] IIFDG (NEW)  
[ ] AR330_DVP (NEW)    
[*] AR330_MIPI      
[ ] IMX322_DVP (NEW)                
[ ] IMX179_MIPI (NEW)   
[ ] OV683_MIPI (NEW)
[ ] OV2710_MIPI (NEW)
[ ] OV4689 (NEW)
[ ] OV4688 (NEW)
[ ] P401 (NEW)      
[ ] SC2135_DVP (NEW)
[ ] SC1135_DVP (NEW)  
[ ] STK3855_DVP (NEW)
[ ] DUAL_BRIDGE (NEW)
[ ] XC9080_MIPI (NEW)

```
#### Step 4: 选择配置文件
运行tools/setproduct.sh，选择Sensor配置文件。

```
# product successfully set to example

# please choose sensor0 configuration:

0   ar330mipi
x   none

# your choice: 0

```
选择JSON配置文件。

```
# please choose product json configration:

0   ar330_path.json
x   default

# your choice: 0
```
#### Step 5: 编译生成镜像文件
```bash
$ make
```
----------------
###4.2  添加双Sensor
双Sensor是指同时有两个Sensor作为图像输入，如双目全景的应用产品形态时，需要前后两个物理Sensor时，这时就需要添加两个Sensor。使用Q3F双Sensor时，需要外接一个双Sensor的图像拼接芯片，Q3-ECO有双Sensor的图像拼接功能，但Sensor的接口必需都为DVP 接口。还是以产品项目名为example为例，说明怎样将AR330 MIPI，AR330 DVP Sensor添加到项目example中。
#### Step 1: 建立Sensor配置文件目录
如同单Sensor，建立example项目目录后，需在项目目录下建立Sensor配置文件目录。
```
$ mkdir products/example/isp
$ mkdir products/example/isp/json
$ mkdir products/example/sensor0
$ mkdir products/example/sensor0/ar330mipi
$ mkdir products/example/sensor1
$ mkdir products/example/sensor1/ar330dvp
```
#### Step 2: 复制相关配置文件
默认支持的Sensor配置文件放在**system/hlibcamsensor/sensor_cfg**目录下。
**有屏**
```
$ cp system/hlibcamsensor/sensor_cfg/json/sensor_ids.json products/example/isp/json/ar330_dual.json
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330mipi/* products/example/sensor0/ar330mipi
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330dvp/* products/example/sensor1/ar330dvp
```
**无屏**
```
$ cp system/hlibcamsensor/sensor_cfg/json/sensor_h1264.json products/example/isp/json/ar330_dual.json
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330mipi/* products/example/sensor0/ar330mipi
$ cp system/hlibcamsensor/sensor_cfg/sensors/ar330dvp/* products/example/sensor1/ar330dvp
```
复制配置文件后example的**isp**目录下的结构如下。
```
├── json
│   └── ar330_dual.json
├── sensor0
│   └── ar330mipi
│       ├── sensor0-config.txt
│       ├── sensor0-isp-config.txt
│       ├── sensor0.itm
│       └── sensor0-lsh-config.lsh
└── sensor1
    └── ar330dvp
        ├── sensor1-config.txt
        ├── sensor1-isp-config.txt
        ├── sensor1.itm
        └── sensor1-lsh-config.lsh
```
#### Step 3: 选择Sensor源代码
如同4.1节的步骤3，`make menuconfig`选择名为AR330_MIPI，AR330_DVP的选项，使AR330 MIPI，DVP Sensor的源代码编译到QSDK的镜像文件中，若默认已选上，可以跳过此步。

#### Step 4: 选择配置文件
运行tools/setproduct.sh，选择Sensor配置文件。
```
# product successfully set to example

# please choose sensor0 configuration:

0   ar330mipi
x   none

# your choice: 0

# please choose sensor1 configuration:

0   ar330dvp
x   none

# your choice: 0


```
选择JSON配置文件。
```
# please choose product json configration:

0   ar330_dual.json
x   default

# your choice: 0
```
#### Step 5: 编译生成镜像文件
```bash
$ make
```
-------------------
## 5 添加新Sensor
当需要添加新的Sensor到QSDK中，除了要按第4章来将Sensor的配置文件加入到对应的项目目录中外，还需要实现相应的Sensor API。

---------------
### 5.1 `hlibcamsensor`结构
QSDK将每个Sensor的API的实现代码全部统一放在**hlibcamsensor**的目录中，由**hlibispv2500**和**hlibispv2505**共用一套Sensor API代码，便于维护和添加新Sensor，如下所示为**hlibcamsensor**的代码结构。
```cpp
hlibcamsensor
├── ci_user
│   ├── ci_gasket.c
│   └── sys_userio.c
├── CMakeLists.txt
├── Config.in
├── hlibcamsensor.mk
├── hlibcamsensor.pc
├── include
│   ├── c99
│   ├── ci_internal
│   ├── ci_kernel
│   ├── felixcommon
│   ├── felix_hw_info.h
│   ├── img_defs.h
│   ├── img_errors.h
│   ├── img_pixfmts.h
│   ├── img_sysdefs.h
│   ├── img_systypes.h
│   ├── img_types.h
│   ├── linkedlist.h
│   ├── mc
│   ├── sensorapi
│   ├── sensors
│   ├── sim_image.h
│   ├── sys
│   ├── v2500
│   └── v2505
├── pciuser.c
├── sensorapi.c
├── sensor_cfg
│   ├── json
│   └── sensors
├── sensor_phy.c
└── sensors
    ├── ar330dvp.c
    ├── ar330mipi.c
    ├── bridge
    ├── hm2131Dual.c
    ├── imx179mipi.c
    ├── imx322dvp.c
    ├── ov2710.c
    ├── ov4688.c
    ├── ov4689.c
    ├── p401.c
    ├── sc1135dvp.c
    ├── sc2135dvp.c
    ├── stk3855dual720dvp.c
    ├── v2500
    └── v2505
```
--------------------------
### 5.2 添加Sensor代码

#### Step 1: 创建Sensor的API实现文件
Sensor API的实现代码文件需放在hlibcamsensor/sensors目录下，若添加的Sensor为DVP接口可以从**ar330dvp.c**复制一份并改名为**xxxdvp.c**，MIPI接口则可以从**ar330mipi.c**复制一份改名为**xxxmipi.c**。

#### Step 2: 实现Sensor API函数
创建好Sensor API的实现文件后，需在这个文件里重新实现`SENSOR_FUNCS`结构体里对应的Sensor API。`SENSOR_FUNCS`数据结构定义如下，具体每个结构体成员说明见第8章[Sensor API说明](./main.md#8_Sensor_API说明)。
```cpp
typedef struct _Sensor_Functions_
{
  IMG_RESULT (*GetMode)(SENSOR_HANDLE hHandle, IMG_UINT16 nIndex,
     SENSOR_MODE *psModes);

  IMG_RESULT (*GetState)(SENSOR_HANDLE hHandle,
     SENSOR_STATUS *psStatus);

  IMG_RESULT (*SetMode)(SENSOR_HANDLE hHandle, IMG_UINT16 nMode,
     IMG_UINT8 ui8Flipping);

  IMG_RESULT (*Enable)(SENSOR_HANDLE hHandle);

  IMG_RESULT (*Disable)(SENSOR_HANDLE hHandle);

  IMG_RESULT (*Destroy)(SENSOR_HANDLE hHandle);    

  IMG_RESULT (*GetInfo)(SENSOR_HANDLE hHandle, SENSOR_INFO *psInfo);

  IMG_RESULT (*GetGainRange)(SENSOR_HANDLE hHandle, double *pflMin,
     double *pflMax, IMG_UINT8 *puiContexts);

  IMG_RESULT (*GetCurrentGain)(SENSOR_HANDLE hHandle, double *pflCurrent,
     IMG_UINT8 ui8Context);

  IMG_RESULT (*SetGain)(SENSOR_HANDLE hHandle, double flGain,
     IMG_UINT8 ui8Context);

  IMG_RESULT (*GetExposureRange)(SENSOR_HANDLE hHandle, IMG_UINT32 *pui32Min,
     IMG_UINT32 *pui32Max, IMG_UINT8 *pui8Contexts);

  IMG_RESULT (*GetExposure)(SENSOR_HANDLE hHandle, IMG_UINT32 *pui32Exposure,
     IMG_UINT8 ui8Context);

  IMG_RESULT (*SetExposure)(SENSOR_HANDLE hHandle, IMG_UINT32 ui32Exposure,
     IMG_UINT8 ui8Context);    

  IMG_RESULT (*GetCurrentFocus)(SENSOR_HANDLE hHandle,
     IMG_UINT16 *pui16Current);

  IMG_RESULT (*SetFocus)(SENSOR_HANDLE hHandle, IMG_UINT16 ui16Focus);

  IMG_RESULT (*ConfigureFlash)(SENSOR_HANDLE hHandle, IMG_BOOL bAlwaysOn,
     IMG_INT16 i16FrameDelay, IMG_INT16 i16Frames,
     IMG_UINT16 ui16FlashPulseWidth);    

  IMG_RESULT (*Insert)(SENSOR_HANDLE hHandle);    
#ifdef INFOTM_ISP
  IMG_RESULT (*SetFlipMirror)(SENSOR_HANDLE hHandle, IMG_UINT8 flag);

  IMG_RESULT (*SetFPS)(SENSOR_HANDLE hHandle, double fps);

  IMG_RESULT (*SetGainAndExposure)(SENSOR_HANDLE hHandle, double flGain,
     IMG_UINT32 ui32Exposure, IMG_UINT8 ui8Context);

  IMG_RESULT (*GetFixedFPS)(SENSOR_HANDLE hHandle, int *pfixed);

  IMG_RESULT (*SetResolution)(SENSOR_HANDLE hHandle, int imgW, int imgH);

  IMG_RESULT (*GetSnapRes)(SENSOR_HANDLE hHandle, reslist_t *preslist);

  IMG_RESULT (*Reset)(SENSOR_HANDLE hHandle);

  IMG_UINT8* (*ReadSensorCalibrationData)(SENSOR_HANDLE hHandle,
     int sensor_index, IMG_FLOAT awb_convert_gain,
     IMG_UINT16* otp_calibration_version);

  IMG_UINT8* (*ReadSensorCalibrationVersion)(SENSOR_HANDLE hHandle,
     int sensor_index, IMG_UINT16* otp_calibration_version);

  IMG_RESULT (*UpdateSensorWBGain)(SENSOR_HANDLE hHandle, IMG_FLOAT awb_convert_gain);
#endif //INFOTM_ISP  
}SENSOR_FUNCS;
```
#### Step 3: 添加初始化函数
**sensorapi.c**里需获取新Sensor的初始化函数，所有Sensor的初始化函数被放在`InitialiseSensors[]`的数组中，所以新Sensor的初始化函数需添加到此数组中，如下为已添加的Sensor初始化函数。
```cpp
IMG_RESULT (*InitialiseSensors[])(SENSOR_HANDLE *phHandle, int index) = {
#if !defined(INFOTM_ISP)
  IIFDG_Create,
#endif
#if defined(SENSOR_IIFDG)
  IIFDG_Create,
#endif
#if defined(SENSOR_DG)
  DGCam_Create,
#endif
#if defined(SENSOR_AR330_DVP)
  AR330DVP_Create,
#endif
#if defined(SENSOR_DUAL_BRIDGE)
  DualBridge_Create,
#endif
#if defined(SENSOR_NT99142_DVP)
  NT99142DVP_Create,
#endif
#if defined(SENSOR_AR330_MIPI)
  AR330_Create_MIPI,
#endif
#if defined(SENSOR_IMX322_DVP)
  IMX322_Create_DVP,
#endif
#if defined(SENSOR_IMX179_MIPI)
  IMX179_Create_MIPI,
#endif
#if defined(SENSOR_OV683_MIPI)
  OV683Dual_Create,
#endif
// #if defined(SENSOR_P401)
// P401_Create,
// #endif
// #if defined(SENSOR_OV4688)
// OV4688_Create,
// #endif
#if defined(SENSOR_OV2710)
  OV2710_Create,
#endif
#if defined(SENSOR_OV4689)
  OV4689_Create,
#endif
#if defined(SENSOR_SC2135_DVP)
  SC2135DVP_Create,
#endif
#if defined(SENSOR_SC1135_DVP)
  SC1135DVP_Create,
#endif
#if defined(SENSOR_99141_DVP)
  STK3855DUAL720DVP_Create,
#endif
#if defined(SENSOR_HM2131_MIPI)
  HM2131Dual_Create,
#endif
};
```
Note:
Sensor的初始化函数，主要是对`SENSOR_FUNCS`结构体成员函数指针的赋值，每个成员对应着相应具体Sensor的API实现。以AR330 DVP为例，对应代码如下。
```cpp
...
*phHandle = &psCam->sFuncs;
psCam->sFuncs.GetMode = AR330DVP_GetMode;
psCam->sFuncs.GetState = AR330DVP_GetState;
psCam->sFuncs.SetMode = AR330DVP_SetMode;
psCam->sFuncs.Enable = AR330DVP_Enable;
psCam->sFuncs.Disable = AR330DVP_DisableSensor;
psCam->sFuncs.Destroy = AR330DVP_DestroySensor;    
psCam->sFuncs.GetGainRange = AR330DVP_GetGainRange;
psCam->sFuncs.GetCurrentGain = AR330DVP_GetGain;
psCam->sFuncs.SetGain = AR330DVP_SetGain;
psCam->sFuncs.GetExposureRange = AR330DVP_GetExposureRange;
psCam->sFuncs.GetExposure = AR330DVP_GetExposure;
psCam->sFuncs.SetExposure = AR330DVP_SetExposure;    
psCam->sFuncs.GetInfo = AR330DVP_GetInfo;
psCam->sFuncs.SetFlipMirror = AR330DVP_SetFlipMirror;
psCam->sFuncs.SetGainAndExposure = AR330DVP_SetGainAndExposure;
psCam->sFuncs.SetFPS = AR330DVP_SetFPS;
psCam->sFuncs.GetFixedFPS =  AR330DVP_GetFixedFPS;
psCam->sFuncs.Reset = AR330DVP_Reset;
...
```

#### Step 4: 添加新Sensor名字
**sensorapi.c**里也需获取Sensor名字，并根据Sensor的名字找到对应的初始化函数，Sensor的名字放在`Sensors[]`的数组里面，其定义在**sensor_name.h**中。添加新Sensor里，这两个地方要相应的添加新Sensor的名字，如下为已添加的Sensor名字宏定义。

**Sensor[]数组**
```cpp
const char *Sensors[]={
#if !defined(INFOTM_ISP)
  IIFDG_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_IIFDG)
  IIFDG_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_DG)
  EXTDG_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_AR330_DVP)
  AR330DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_DUAL_BRIDGE)
  DUAL_BRIDGE_INFO_NAME,
#endif
#if defined(SENSOR_NT99142_DVP)
  NT99142DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_AR330_MIPI)
  AR330MIPI_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_IMX322_DVP)
  IMX322DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_IMX179_MIPI)
  IMX179MIPI_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_OV683_MIPI)
  OV683Dual_SENSOR_INFO_NAME,
#endif
// #if defined(SENSOR_P401)
// P401_SENSOR_INFO_NAME,
// #endif
// #if defined(SENSOR_OV4688)
// OV4688_SENSOR_INFO_NAME,
// #endif
#if defined(SENSOR_OV2710)
  OV2710_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_OV4689)
  OV4689_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_SC2135_DVP)
  SC2135DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_SC1135_DVP)
  SC1135DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_99141_DVP)
  STK3855DUAL720DVP_SENSOR_INFO_NAME,
#endif
#if defined(SENSOR_HM2131_MIPI)
  HM2131Dual_SENSOR_INFO_NAME,
#endif
  NULL,
};
```
**Sensor名字定义**
```cpp
#define AR330DVP_SENSOR_INFO_NAME "ar330dvp"
#define AR330MIPI_SENSOR_INFO_NAME "ar330mipi"
#define IMX179MIPI_SENSOR_INFO_NAME "imx179mipi"
#define IMX322DVP_SENSOR_INFO_NAME "imx322dvp"
#define OV2710_SENSOR_INFO_NAME "ov2710"
#define OV4688_SENSOR_INFO_NAME "ov4688"
#define OV4689_SENSOR_INFO_NAME "ov4689"
#define OV683Dual_SENSOR_INFO_NAME "ov683"
#define P401_SENSOR_INFO_NAME "p401"
#define SC1135DVP_SENSOR_INFO_NAME "sc1135dvp"
#define SC2135DVP_SENSOR_INFO_NAME "sc2135dvp"
#define STK3855DUAL720DVP_SENSOR_INFO_NAME "stk3855dvp"
#define DUAL_BRIDGE_INFO_NAME "dual-bridge"
#define NT99142DVP_SENSOR_INFO_NAME "nt99142dvp"
#define HM2131Dual_SENSOR_INFO_NAME "hm2131mipi"
```
Note:
Sensor Item配置文件中的sensorX.name(X为0，1，2 ...)必须与这个名字相同。例如对于AR330 MIPI Sensor在**sensor_name.h**的名字为`ar330mipi`，则其在sensorX.itm配置文件中定义的名字也必需为`ar330mipi`，如下定义。

```bash
sensor0.name ar330mipi
```
#### Step 5: 设置Sensor初始化配置
每个Sensor有很多初始化配置，如30帧的配置，1920x1080的配置，1280x720配置。在QSDK中不同的配置用`mode`来表示，一个`mode`对应代码里的一个Sensor的初始化配置，默认的值为0，也即第0个`mode`。当需要使用非`mode`0配置时，可在Sensor的Item文件中指定，如下所示。
```bash
sensor0.mode  1
```
Note:
Sensor `mode`可设置也可不设置，不设置时默认使用0。下表描述了什么时候需要设置，什么时候不需要设置Sensor `mode`。

|项目目录中Sensor配置文件有没有|mode 0满足要求|需要设置mode|
|--|--|--|
|有|是|不需要|
|有|不|不需要|
|无|是|不需要|
|无|不|需要|

-----------------------
### 5.3 编译代码生成镜像
当修改或添加了**hlibcamsensor**的源文件后，需要做以下步聚编译，才能使当前改动在Videobox中生效。
```bash
$ make hlibcamsensor-rebuild
$ make
```

-------------------------
## 6 DVP Sensor的调试
### 6.1 正确上电
Sensor模组作为一种电气件，若想让其正常工作，需仔细根据Sensor的DataSheet，硬件电路图，正确的提供电压，时钟，上电时序。只有这些正确了后，才能保证能正确的通过I2C来配置Sensor的寄存器，让它出图。一般DVP Sensor需要配置的电气特性如下表，不同Sensor略有不同。

| 需要设置项| 描述 |
| ---- |-----|
|Reset 引脚|Sensor的Reset引脚|
|Power Down引脚|Sensor的Power Down引脚|
|IOVDD|Sensor的IO输出端口电压|
|DVDD|Sensor的Core数字电压|
|AVDD|Sensor的Core模拟电路电压|
|MCLK/EXTCLK|Sensor的外部输入时钟|

QSDK中是通过配置Sensor Item文件来配置其电压大小，时钟频率，上电时序。Item文件的每个项具体含义请参考[附录9.2](./main.md#9.2._Sensor_Item配置说明)，以AR330 Sensor举例，AR330 Sensor的上电时序图如下。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/power_up.svg "power up")

对应的Item文件配置如下。
```bash
sensor0.interface         dvp
sensor0.name              ar330dvp
sensor0.mclk              24000000
sensor0.imager            1
sensor0.i2c               chn.3.addr.32
sensor0.mclk.src          imap-camo
sensor0.bootup.seq.0      pmu.ldo7.2800000.0
sensor0.bootup.seq.1      pmu.ldo8.1800000.0
sensor0.bootup.seq.2      mclk.gpio.114.0
sensor0.bootup.seq.3      reset.gpio.44.20

```
--------------------
### 6.2  出图像调试
调试新Sensor时，出图像是调试的关键一步，出图像了说明Sensor初始化成功，并且整个软件流程也调通了。为了简化出图像的调试流程，需让Sensor先输出Test Pattern的彩条测试图样，这样可以不用实现设置曝光时间和增益函数。否则需要先实现这两个函数才能看到正常的图像输出。若在调试出图像过程中，Sensor没有输出Test Pattern图像，这时候需要用示波器测量Sensor的HSYNC和VSYNC信号，确认Sensor有没有信号输出。如果没有则说明Sensor的配置有问题，或者是Sensor的数据流控是否打开。

Sensor正常出图时，其输出的DVP的时序图如下(主要输出波形有VSYNC(帧同步)， HSYNC/HREF(行同步/行有效)， PCLK(Pixel Clock))。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/DVP_Signal.svg "Signal")

-----------------
### 6.3 错误调试
**I2C报错**

```bash
imapx_i2c_handle_tx_abort: the address sent was not acknowledged by any slave(7-bit mode)
```
启动**videoboxd**，如打印以上的输出信息时，说明Sensor没有正常工作，一般都是硬件问题，需要用万用表和示波器检查如下内容。

- 供电电压是否正确
- 输入的MCLK有没有，引脚是否正确
- Power Down引脚是否正确，电平是否正确
- Reset引脚是否正确，其电平是否正确
- I2C地址是否正确
- 上电时序是否正确

**Failed to acquire a buffer**

```bash
ERROR [CI_API]: IMG_PipelineAcquireBuffer():587 Failed to acquire a buffer (returned -62 - shot id 0)
ERROR [ISPC_PIPELINE]: acquireShot():1238 Failed to acquire buffer with blocking call (returned 1, pCIBuffer=0x(nil))
ERROR [ISPC_CAMERA]: acquireShot()[2017-07-15 16:27:55:299]:1239 Unable to get shot
```
若调试DVP时，遇到以上的错误信息，说明ISP没有获取Sensor的帧数据，这时候很有可能是Sensor没有正常输出信号，或信号时序不对，要用示波器检查如下内容

- Sensor是否有输出
- MCLK频率和电压是否正常
- 检查HSYNC，VSYNC，PCLK的极性

------------
### 6.4 特殊DVP时序的调试
有些Sensor输出的VSYNC, HSYNC开始的延时时间非常的短，几乎同时到达，其信号特点如下图所示。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Special_Signal.svg "Signal")

当遇到这类Sensor时，需要将Sensor一帧内的前几行丢掉以使VSYNC与HSYNC之间有一定的延时时间，这样ISP才能正常的采集到完整的一帧数据，具体解决思路为Sensor的输出图像大小要大于ISP的接收图像大小，在ISP的输入端Crop掉前几行。

假设项目上只需要1080P的图像，则需将Sensor输出图像配置为1920x1088，在ISP输入端配置`IIF`模块为1920x1080大小，具体做法如下。
在`sensor0-isp-config.txt`里配置ISP的`IIF`模块`TL`为0 8， `IIF_CAP_RECT_BR`为1919 1087。
```bash
IIF_DECIMATION               0 0
IIF_CAP_RECT_TL              0 8
IIF_CAP_RECT_BR              1919 1087
```
下图显示出`TL`设置0 0与`TL`设置0 8时ISP输入端图像有效区域的部分(白色区域为有效部分)

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/TL.svg "Top and Left")

------------
### 6.5 DVP Wrapper调试

**Wrapper 功能说明**

DVP Wrapper是针对DVP接口的Sensor，对其输出图像做裁减，主要是为了让InfoTM系列芯片支持更多的DVP接口的Sensor。当DVP Sensor的输出图像数据带**Sync Code**的同步数据时，如Sony IMX322的Sensor，就需要用到DVP Wrapper将**Sync Code**裁减掉。当使用Wrapper后，给后端ISP的输入就是Wrapper的输出；不使用时，ISP的输入为Sensor的直接输出。`Q3F`，`Q3-ECO`才有Wrapper功能。使用Wrapper和不使用Wrapper在`Q3F`，`Q3-ECO`中的数据流程如下。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/DVP_Wraper.svg "camera DVP Wrapper")

**配置DVP Wrapper**

要使用DVP Wrapper，只需在Sensor的Item文件中使能Wrapper并配置相关参数即可，其配置项如下表。

|Item|值|说明|
|--|--|--|
| `sensor0.wraper.use` |  `0` | 是否使用DVP的Wrapper功能。<br><br>`1` - 使用 <br>`0` - 不使用 <br><br>只针对`Q3F`和`Q3-ECO`有效|
| `sensor0.wraper.width`  |  `1920` | DVP Wrapper输出宽度 |
| `sensor0.wraper.height`  | `1080` | DVP Wrapper输出高度 |
| `sensor0.wraper.hdelay`  |  `114` | DVP Wrapper hdelay值，即图像左上角水平方向的起点 |
| `sensor0.wraper.vdelay`  |  `12` | DVP Wrapper vdelay值，即图像左上角垂直方向的起点 |

-----------
## 7 MIPI Sensor调试
### 7.1 MIPI接口描述

MIPI接口指遵守MIPI CSI-2.0协议数据传输接口，分为Master端和Slave端。通常Sensor作为Master端设备主动发送时钟和图像数据，而CSI Controller作为Slave端按照Master的时钟来接收图像数据。MIPI接口的时钟与数据都为差分信号，其中时钟只有一个，数据可以有多个Lanes，应用上可以根据不同的传输数据量多少来配置不同的Lane数，MIPI接口示意图如下。

 ![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/mipi_framework.png "csi interface")

-------------
### 7.2 MIPI Sensor接入
MIPI Sensor的接入和[第6章](./main.md#6_DVP_Sensor的调试)DVP Sensor接入步骤一样，首先要正确的上电，其次就是出图像调试。需要注意的是，MIPI正确上电的Item配置与DVP配置略有不同，以AR330 MIPI Sensor为例Item配置如下，其中加粗的Item为与DVP不同之处。

<pre><code><strong>sensor0.interface        mipi</strong>
sensor0.name             ar330mipi
sensor0.mclk             24000000
<strong>sensor0.imager           0</strong>
sensor0.i2c              chn.3.addr.32
<strong>sensor0.csi              lanes.1.freq.384</strong>
sensor0.mclk.src         imap-clk1
sensor0.bootup.seq.0     pmu.ldo7.2800000.0
sensor0.bootup.seq.1     pmu.ldo8.1800000.0
sensor0.bootup.seq.2     mclk.gpio.114.0
sensor0.bootup.seq.3     reset.gpio.44.20
</code></pre>

--------------
### 7.3 MIPI错误调试
若Sensor与CSI Controller之间数据传输信号同步出现问题时，CSI Controller会发生MIPI CRC或ECC Error，此时QSDK会打印以下信息。

```bash
[ 6.779999] Gasket(0): eType=1, FrameCount=0
[ 6.783333] Gasket MIPI FIFO 1 - Enabled lanes 1
[ 6.786666] Gasket MIPI CRC Error: 0x1
[ 6.789999] Gasket MIPI Header Error: 0x0
[ 6.793333] Gasket MIPI ECC Error: 0x0
[ 6.796666] Gasket MIPI ECC Correted: 0x14
[ 6.799999] FELIX:KRN_CI_PipelineAcquireShot:1091 Isp acquire timeout: 1000
[ 6.803333] FELIX:DEV_CI_Ioctl:390 Failed to acquire a buffer
ERROR [CI_API]: IMG_PipelineAcquireBuffer():587 Failed to acquire a buffer (returned -62 - shot id 0)
```
出现MIPI CRC或ECC Error时，需要检查如下内容。

- MIPI PCB设计是否符合要求
- ISP时钟大小配置是否正确
- CSI时钟是否配置正确
- CSI Controller是否配置正确
- HS-PREPARE是否符合MIPI PHY协议

下面几小节将对上面检查内容做具体说明。

---------
#### 7.3.1 PCB设计要求
MIPI差分信号对硬件布线要求比较高，为了保证MIPI信号的完整性，设计MIPI模块PCB时需注意如下几点。

1. MIPI差分信号阻抗控制在100ohm +/-10%。
1. 每组MIPI信号都需要等长，等间距， 组内P&N等长误差+/- 5mil，最好用GND包裹并尽量多打GND VIA，GND走线至少10mil以上。
1. 信号离GND至少保证2W Airgap，如空间受限无法包地，信号组与组之间保证至少3W Airgap。
1. 每组MIPI信号尽量不要换层，如果换层尽量在换层VIA处就近增加GND VIA。
1. 所有MIPI信号参考GND层，不能跨Reference Plane。
1. 所有MIPI信号远离干扰源，远离高频信号。

-----------------
#### 7.3.2 ISP时钟要求

**ISP时钟大小要求**
ISP时钟与Sensor输出MIPI时钟需满足以下关系。
```
ISP_CLK > MIPI_CLK x 2 x Lanes / PPS
```
Note:
1. `ISP_CLK`为ISP时钟，`MIPI_CLK`为Sensor输出的MIPI时钟，`Lanes`为当前配置的数据通道数，`PPS`为当前每个像素的位宽。例如`MIPI_CLK` = 192M，`Lanes` = 2，`PPS` = 10，此时`ISP_CLK` > 192000000 x 2 x 2 / 10，即`ISP_CLK`大于76.8M。
2. 当条件不满足时，可以提高ISP时钟，或在不改变Sensor输出分辨率和帧率情况下，降低Sensor输出的MIPI时钟。

**调整ISP时钟**
当ISP的时钟不满足上述要求时，可以调整ISP的工作时钟，如调整ISP时钟为200M，需修改启动脚本**/etc/init.d/S902videobox**里的`modprobe Felix.ko`的`clkRate`大小为200000000。
```
modprobe Felix.ko clkRate=200000000
```
**ISP时钟确认**
调整好了ISP时钟后在命令行终端输入如下命令，确认是否修改成功。
```
$ mount –t debugfs none /mnt
$ cat /mnt/clk/clk_summary
```
--------------
#### 7.3.3 CSI Controller时钟配置
**确认Sensor输出MIPI时钟**
用示波器差分探头连接Sensor的CLKN/CLKP，确认Sensor的输出MIPI Clock为多少。
**确认CSI Controller时钟大小**
在命令行终端输入如下命令确认CSI Controller配置时钟是否和Sensor的输出MIPI时钟一致。
```
$ cat sys/class/misc/ddk_sensor/sensor_info
```
以**q3fevb_va**平台为例，输入上述命令后结果如下，加粗的一行为当前设置的CSI Controller时钟大小。
<pre><code>name=ar330mipi
interface=mipi
i2c_addr=0x10
chn=3
lanes=1
<strong>csi_freq=384</strong>
mclk_src=imap-clk1
mclk=24000000
wraper_use=0
wwidth=1920
wheight=1080
whdly=114
wvdly=12
imager=0

</code></pre>

**调整CSI Controller时钟**

当CSI Controller配置时钟和Sensor的输出MIPI时钟不一致时，需修改Item文件来调整CSI Controller时钟大小，调整大小为384M时的Item配置如下。
```
sensor0.csi  lanes.1.freq.384
```
------------------
#### 7.3.4 确认CSI Controller配置
在调试中还需要确认CSI Controller的配置是否正确，CSI Controller端是否报错，查看配置信息的方法与确认CSI Controller时钟方法一样。
```
cat sys/class/misc/ddk_sensor/sensor_info
```
以**q3fevb_va**平台为例，输入上述命令后结果如下，其中`CSI_REG_OFFSET20`，`CSI_REG_OFFSET24`两寄存器值表示CSI Controller是否报错。
```
CSI_REG_OFFSET0 =[0x3130322a]
CSI_REG_OFFSET4 =[0x0]
CSI_REG_OFFSET8 =[0x1]
CSI_REG_OFFSETc =[0x1]
CSI_REG_OFFSET10 =[0x1]
CSI_REG_OFFSET14 =[0x300]
CSI_REG_OFFSET18 =[0x0]
CSI_REG_OFFSET1c =[0x0]
CSI_REG_OFFSET20 =[0x0]
CSI_REG_OFFSET24 =[0x0]
CSI_REG_OFFSET28 =[0x1fffffff]
CSI_REG_OFFSET2c =[0xffffff]
CSI_REG_OFFSET30 =[0x2]
CSI_REG_OFFSET34 =[0x9f1f]
CSI_REG_OFFSET404 =[0x0]
CSI_REG_OFFSET400 =[0x0]
CSI_REG_OFFSET408 =[0x67]
CSI_REG_OFFSET40c =[0x64]
```
如果`CSI_REG_OFFSET20`，`CSI_REG_OFFSET24`两寄存器为0，则没有错误，右为下值，说明Sensor输出的信号给到CSI Controller PHY同步出错了。
```
CSI_REG_OFFSET20 =[0x17011131]
CSI_REG_OFFSET24 =[0x1701]
```

-----------------
#### 7.3.5 Sensor HS-PREPARE配置
MIPI PHY从Low-Power模试进入到High-Speed模式时有一定时序，如下为进入High-Speed模式时序示意图。

 ![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/mipi_hs_seq.png "mipi high mode timing")

MIPI协议规定HS-PREPARE需满足以下约束。
```
40ns + 4UI <= HS-PREPARE <= 85ns + 6UI
HS-PREPARE + HS-ZERO >= 145ns + 10*UI
```
Note:
`ns`为时间单位纳秒，`UI`也为时间单位，具体大小参见[附录9.4](./main.md#9.4_MIPI_UI定义)。

**确认HS-PREPARE时间**
当Sensor输出的HS-PREPARE不符合MIPI协议规定范围时，MIPI CSI Controller会产生CRC错误，这时需要用示波器测量出HS-PREPARE的时间，HS-PREPARE波形图参见下图`BX`与`AX`之间的时间。

 ![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/mipi_hsprepare.png "mipi hs prepare timing")

**调整HS-PREPARE时间**
当通过示波器测量的HS-PREPARE时间不在MIPI协议规定范围时，需要阅读Sensor DataSheet查看HS-PREPARE的寄存器，并调整此寄存器的值，使HS-PREPARE时间在约束的范围内。

-------------
## 8 Sensor API说明
Sensor API作为Sensor的HAL，向上提供API给DDK的Sensor类使用，向下映射到每个Sensor的具体API的实现。添加新的Sensor时，主要工作就是实现新Sensor对应的HAL的每个API。

--------------
### 8.1 Sensor API列表
下表列出了Sensor API，并描述了哪些API可不必实现，哪些建议实现，而哪些必需实现的。所谓必需实现，就是如果不实现会影响到Sensor正常输出图像，所谓建议实现，就是最好实现，不实现也不影响正常输出图像。

|API|实现|
|--|--|
|`GetMode()`|必需实现|
|`GetState()`|必需实现|
|`SetMode()`|必需实现|
|`Enable()`|必需实现|
|`Disable()`|必需实现|
|`Destroy()`|建议实现|
|`GetInfo()`|必需实现|
|`GetGainRange()`|必需实现|
|`GetCurrentGain()`|必需实现|
|`SetGain()`|必需实现|
|`GetExposureRange()`|必需实现|
|`GetExposure()`|必需实现|
|`SetExposure()`|必需实现|
|`SetGainAndExposure()`|必需实现|
|`Reset()`|必需实现|
|`SetFlipMirror()`|建议实现|
|`SetFPS()`|必需实现|
|`ReadSensorCalibrationData()`|可不必实现|
|`ReadSensorCalibrationVersion()`|可不必实现|
|`UpdateSensorWBGain()`|可不必实现|

-------------------------
### 8.2 API 说明

#### 8.2.1 GetMode()

```
IMG_RESULT (*GetMode)(SENSOR_HANDLE hHandle, IMG_UINT16 nIndex, SENSOR_MODE *psModes);
```
获取Sensor `mode`对应的Sensor配置信息，例如Sensor输出图像的宽，高，并将获取到的结果放在`psModes`里。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `nIndex`  - **`IN`** 传入Sensor `mode`的索引，代表是Sensor哪个配置信息。
- `psModes` - **`OUT`** 传出的SENSOR_MODE的结构体指针，用于保存Sensor宽，高等信息。

**返回**

该函数失败返回非0，成功返回0。

**举例**
```
static IMG_RESULT AR330MIPI_GetMode(SENSOR_HANDLE hHandle, IMG_UINT16 nIndex,
        SENSOR_MODE *psModes)
{
    // AR330CAM_STRUCT *psCam = container_of(hHandle, AR330CAM_STRUCT, sFuncs);
    if(nIndex < sizeof(asModes) / sizeof(SENSOR_MODE))
    {
       IMG_MEMCPY(psModes, &(asModes[nIndex]), sizeof(SENSOR_MODE));
       return IMG_SUCCESS;
    }
    return IMG_ERROR_NOT_SUPPORTED;
}
```
-----------------------
#### 8.2.2 GetState()

```
IMG_RESULT (*GetState)(SENSOR_HANDLE hHandle, SENSOR_STATUS *psStatus);
```
获取Sensor的状态信息，并将状态信息保存在结构体`psStatus`中。。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `psStatus`  - **`OUT`** 传出的`SENSOR_STATUS`的结构体指针，用于保存状态信息。

**返回**

该函数失败返回非0，成功返回0。

**举例**
```
static IMG_RESULT AR330MIPI_GetState(SENSOR_HANDLE hHandle, SENSOR_STATUS *psStatus)
{
    AR330CAM_STRUCT *psCam = container_of(hHandle, AR330CAM_STRUCT, sFuncs);
    if (!psCam->psSensorPhy)
    {
        LOG_WARNING("sensor not initialised\n");
        psStatus->eState = SENSOR_STATE_UNINITIALISED;
        psStatus->ui16CurrentMode = 0;
    }
    else
    {
        psStatus->eState = (psCam->bEnabled ? SENSOR_STATE_RUNNING : SENSOR_STATE_IDLE);
        psStatus->ui16CurrentMode = psCam->ui16CurrentMode;
        psStatus->ui8Flipping = psCam->ui8CurrentFlipping;
        psStatus->fCurrentFps = psCam->fCurrentFps;
    }
    return IMG_SUCCESS;
}
```
---------------------------
#### 8.2.2 SetMode()

```
IMG_RESULT (*SetMode)(SENSOR_HANDLE hHandle, IMG_UINT16 nMode, IMG_UINT8 ui8Flipping);
```
指定Sensor的`mode`，对配置文件的解析，指定Sensor的最大，最小曝光时间都在此函数内实现。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `ui16Mode`  - **`IN`** ``mode`的索引，指定初始化配置。
- `ui8Flipping`  - **`IN`** Sensor图像是否做水平垂直翻转　0: 水平垂直不翻转。1: 水平翻转 2: 垂直翻转 3: 水平垂直都翻转。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.3 Enable()

```
IMG_RESULT (*Enable)(SENSOR_HANDLE hHandle);
```
使能Sensor，调用此函数后，Sensor开始输出数据流。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。

**返回**

该函数失败返回非0，成功返回0。

**说明**

* 使能Sensor输出数据要在使能ISP之后，否则，ISP就会报错。

---------------------------
#### 8.2.4 Disable()

```
IMG_RESULT (*Disable)(SENSOR_HANDLE hHandle);
```
不使能Sensor，调用此接口后，Sensor关闭输出数据流。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。

**返回**

该函数失败返回非0，成功返回0。

**举例**

```
static IMG_RESULT AR330MIPI_DisableSensor(SENSOR_HANDLE hHandle)
{
    IMG_UINT16 aui16Regs[2];
    AR330CAM_STRUCT *psCam = container_of(hHandle, AR330CAM_STRUCT, sFuncs);
    IMG_RESULT ret = IMG_SUCCESS;

    if (!psCam->psSensorPhy)
    {
        LOG_ERROR("sensor not initialised\n");
        return IMG_ERROR_NOT_INITIALISED;
    }

    LOG_INFO("Disabling AR330 camera\n");
    psCam->bEnabled = IMG_FALSE;
    // if we are generating a file we don't want to disable!
    aui16Regs[0] = 0x301a; // RESET_REGISTER
    aui16Regs[1] = 0x0058;
    ret = AR330MIPI_I2C_WRITE(psCam->i2c, aui16Regs[0], aui16Regs[1]);
    if (IMG_SUCCESS != ret)
    {
        LOG_ERROR("failed to disable sensor!\n");
    }
    usleep(100000);
    SensorPhyCtrl(psCam->psSensorPhy, IMG_FALSE, 0, 0);
    return ret;
}
```

---------------------------
#### 8.2.5 Destroy()

```
IMG_RESULT (*Destroy)(SENSOR_HANDLE hHandle);
```
消毁逻辑的Sensor，此函数做退出的释放资源，并对Sensor作关闭等操作。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.6 GetInfo()

```
IMG_RESULT (*GetInfo)(SENSOR_HANDLE hHandle, SENSOR_INFO *psInfo);
```
获取当前Sensor的信息，如Bayer格式。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `psInfo`   - **`OUT`** 存放获得的Sensor的相关信息值。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.7 GetGainRange()

```
IMG_RESULT (*GetGainRange)(SENSOR_HANDLE hHandle, double *pflMin, double *pflMax, IMG_UINT8 *puiContexts);
```
获取Sensor支持的Gain的最大值与最小值，并将最大值，最小值分别存在pflMax, pflMin中，Gain的范围可以包括数字Gain和模拟Gain。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `psInfo`   - **`OUT`** 存放获得的Sensor的相关信息值。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.8 GetGainRange()

```
IMG_RESULT (*GetGainRange)(SENSOR_HANDLE hHandle, double *pflMin, double *pflMax, IMG_UINT8 *puiContexts);
```
获取Sensor支持的Gain的最大值与最小值，并将最大值，最小值分别存在pflMax, pflMin中，Gain的范围可以包括数字Gain和模拟Gain。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `psInfo`   - **`OUT`** 存放获得的Sensor的相关信息值。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.9 GetCurrentGain()

```
IMG_RESULT (*GetCurrentGain)(SENSOR_HANDLE hHandle, double *pflCurrent, IMG_UINT8 ui8Context);
```
获取Sensor当前的增益值。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `pflGain`   - **`OUT`** 存放获取的当前增益值。
- `ui8Context`   - **`IN`** 当前使用ISP的哪个`Context`。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.10 SetGain()

```
IMG_RESULT (*SetGain)(SENSOR_HANDLE hHandle, double flGain, IMG_UINT8 ui8Context);
```
设置Sensor当前的增益值。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `flGain`   - **`IN`** 设置的当前增益值。
- `ui8Context`   - **`IN`** 当前使用ISP的哪个`Context`。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.11 GetExposureRange()

```
IMG_RESULT (*GetExposureRange)(SENSOR_HANDLE hHandle, IMG_UINT32 *pui32Min, IMG_UINT32 *pui32Max, IMG_UINT8 *pui8Contexts);
```
获取当前Sensor支持的曝光时间的范围。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `pui32Min`   - **`OUT`** 存放获取的最短曝光时间值。
- `pui32Max`   - **`OUT`** 存放获取的最长曝光时间值。
- `pui8Contexts`   - **`OUT`** 存放可分离曝光时间的个数。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.12 GetExposure()

```
IMG_RESULT (*GetExposure)(SENSOR_HANDLE hHandle, IMG_UINT32 *pui32Exposure, IMG_UINT8 ui8Context);
```
获取当前Sensor曝光时间值。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `pui32Exposure`   - **`OUT`** 存放获取的曝光时间值。
- `ui8Context`   - **`IN`**  当前使用ISP的哪个`Context。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.13 SetExposure()

```
IMG_RESULT (*SetExposure)(SENSOR_HANDLE hHandle, IMG_UINT32 ui32Exposure, IMG_UINT8 ui8Context);
```
设置当前Sensor的曝光时间

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `pui32Exposure`   - **`IN`** 设置的曝光时间值。
- `ui8Context`   - **`IN`**  当前使用ISP的哪个`Context。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.14 SetGainAndExposure()

```
IMG_RESULT (*SetGainAndExposure)(SENSOR_HANDLE hHandle, double flGain, IMG_UINT32 ui32Exposure, IMG_UINT8 ui8Context);
```
设置Sensor当前的增益值和曝光时间，主要给自动曝光设置增益和曝光时间使用。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `flGain`   - **`IN`** 设置的当前增益值。
- `ui8Context`   - **`IN`** 当前使用ISP的哪个`Context。
- `ui32Exposure`   - **`IN`** 设置Sensor当前的曝光时间值。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.15 SetFPS()

```
IMG_RESULT (*SetFPS)(SENSOR_HANDLE hHandle, double fps);
```
设置Sensor的输出帧率

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `fps`   - **`IN`** 设置的帧率大小的值。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.16 Reset()

```
IMG_RESULT (*Reset)(SENSOR_HANDLE hHandle);
```
重启Sensor，调用此API后，图像Sensor重启，完成Disable和Enable的过程。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.17 SetFlipMirror()

```
 IMG_RESULT (*SetFlipMirror)(SENSOR_HANDLE hHandle, IMG_UINT8 flag);
```
设置Sensor的水平垂直翻转。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `flag`   - **`IN`** 水平垂直翻转的值，0: 水平垂直不翻转 1: 水平翻转 2: 垂直翻转 3: 水平垂直都翻转。

**返回**

该函数失败返回非0，成功返回0。

---------------------------
#### 8.2.18 ReadSensorCalibrationData()

```
IMG_UINT8* (*ReadSensorCalibrationData)(SENSOR_HANDLE hHandle, int sensor_index, IMG_FLOAT awb_convert_gain, IMG_UINT16* otp_calibration_version);
```
获取Sensor的OTP数据，OTP数据的版本信息，同时用`awb_convert_gain`和获取的AWB的OTP数据设置AWB的4个通道的增益值。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `sensor_index`   - **`IN`** 当为双Sensor时，表示设置的是哪个Sensor。
- `awb_convert_gain`   - **`IN`** awb增益的整体比例值。
- `otp_calibration_version`   - **`OUT`** 存放OTP数据的版本信息。

**返回**

该函数失败返回`NULL`，成功返回OTP数据首地址。

---------------------------
#### 8.2.19 ReadSensorCalibrationData()

```
IMG_UINT8* (*ReadSensorCalibrationVersion)(SENSOR_HANDLE hHandle, int sensor_index, IMG_UINT16* otp_calibration_version);
```
只获取Sensor的OTP数据，OTP数据的版本信息。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `sensor_index`   - **`IN`** 当为双Sensor时，表示设置的是哪个Sensor。
- `otp_calibration_version`   - **`OUT`** 存放OTP数据的版本信息。

**返回**

该函数失败返回`NULL`，成功返回OTP数据首地址。


---------------------------
#### 8.2.20 UpdateSensorWBGain()

```
IMG_RESULT (*UpdateSensorWBGain)(SENSOR_HANDLE hHandle, IMG_FLOAT awb_convert_gain);
```
用`awb_convert_gain`设置AWB的4个通道的增益值。

**参数**

- `hHandle`   - **`IN`** 结构体`SENSOR_FUNCS`的指针。
- `awb_convert_gain`   - **`IN`** awb增益的整体比例值。

**返回**

该函数失败返回非0，成功返回0。

----------
## 9 附录
### 9.1 配置文件说明
**配置文件作用**

不同的项目，需要的Sensor不同，即使在不同项目中用同一个Sensor，其初始化配置，ISP的配置也不相同。为了解决这个不同项目的兼容性问题，QSDK使用配置文件，下表描述了一个项目需要的配置文件有哪些。

| 对应文件 | 描述 | 是否可缺省 |
| ---- |
| **sensor0-config.txt**| Sensor初始化配置文件 |可以缺省，当没有时QSDK使用Sensor代码里的默认配置 |
| **sensor0-isp-config.txt** | 相应Sensor的ISP 配置文件 | 可以缺省，仅当不使用ISP而使用CAMIF接收Sensor的YUV格式图像数据时不需要这个配置文件|
| **sensor0-lsh-config.lsh** | 相应Sensor的ISP LSH文件 | 可以缺省，仅当不使用ISP而使用CAMIF接收Sensor的YUV格式图像数据时不需要这个配置文件|
| **sensor0.itm** | Sensor的Items配置文件 | 否 |
| **path.json** | Videobox启动时对应的json配置文件 | 可以缺省，当没有时对应默认的配置**path.json**文件 |

Note:
1. QSDK支持多Sensor，也就是ISP同时可以处理2~3个Sensor的数据，当有多Sensor时，`sensor0`表示第一个，`sensor1`表示第二个，以此类推。
2. 当只有一个Sensor时，用`sensor0`/`sensor1`代表都可以。但必须以sensor + 数字 0，1，2来表示一个Sensor。
3. Sensor初始化配置文件，ISP配置文件，LSH文件，需以**sensorX-config.txt**，**sensorX-isp-config.txt**，**sensorX-lsh-config.lsh**来命名，其中'X'表示数字0，1，2。

**配置文件位置**

配置文件位于QSDK源代码顶层目录**products**下，此目录下存放着所有的项目的配置文件，每个**products**对应一个独立的目录，以**q3fevb_va**平台项目单个Sensor为例，其配置文件位于**/products/q3fevb_va/isp**，对应的目录结构如下。

```bash
isp
├── json
│   ├── camif_fsink_path.json
│   ├── camif_ids_path.json
│   └── isplink_path.json
└── sensor0
    ├── ar330dvp
    │   ├── sensor0-config.txt
    │   ├── sensor0-isp-config.txt
    │   ├── sensor0.itm
    │   └── sensor0-lsh-config.lsh
    ├── ar330mipi
    │   ├── sensor0-config.txt
    │   ├── sensor0-isp-config.txt
    │   ├── sensor0.itm
    │   └── sensor0-lsh-config.lsh
    ├── ar330_mipitocamif
    │   ├── sensor0-config.txt
    │   ├── sensor0-isp-config.txt
    │   ├── sensor0.itm
    │   └── sensor0-lsh-config.lsh
    └── gc0308
        └── sensor0.itm

```
Note:
1. **json**目录下存放不同的JSON文件。
2. `sensor0`表示一个Sensor，而`ar330dvp`, `ar330mipi`并不代表是AR330 DVP或MIPI Sensor, 它只是配置名字的代表，表示`ar330dvp`目录下的为`ar330dvp`配置，`ar330mipi`目录下的，为`ar330mipi`配置。
3. 配置目录名字是可以随便取的，只要你知道当前目录下的配置是什么配置就可以了。
4. 目录名**isp**是不能取其它名字，且是小写，在QSDK中添加新的项目时，这此配置文件需事先放到对应的项目目录中。

以**q3evb_v1.1**平台项目为例，双Sensor配置文件
```bash
isp
├── json
│   ├── path_dual.json
│   └── path.json
├── sensor0
│   ├── ar330dvp
│   │   ├── sensor0-config.txt
│   │   ├── sensor0-isp-config.txt
│   │   ├── sensor0.itm
│   │   └── sensor0-lsh-config.lsh
│   ├── ar330mipi
│   │   ├── sensor0-config.txt
│   │   ├── sensor0-isp-config.txt
│   │   ├── sensor0.itm
│   │   └── sensor0-lsh-config.lsh
│   └── ov4689mipi
│       ├── sensor0-config.txt
│       ├── sensor0-isp-config.txt
│       ├── sensor0.itm
│       └── sensor0-lsh-config.lsh
└── sensor1
    ├── ar330dvp
    │   ├── sensor1-config.txt
    │   ├── sensor1-isp-config.txt
    │   ├── sensor1.itm
    │   └── sensor1-lsh-config.lsh
    └── ar330mipi
        ├── sensor1-config.txt
        ├── sensor1-isp-config.txt
        ├── sensor1.itm
        └── sensor1-lsh-config.lsh
```
Note:
1. `sensor0`为第一个Sensor的配置，`sensor1`为第二个Sensor的配置。

### 9.2 Sensor Item配置说明
不同的Sensor的需要的电压，上电时序也不。如Sensor AR330需提供1.8V的IOVDD电压，而OV4689需提供的IOVDD电压为1.2V。而对于不同平台来说，连接Sensor的硬件引脚也不同。为了描述这些特性，QSDK使用Item文件来抽象。若项目中有两个Sensor，则**Sensor0.itm** 为第一个Sensor的Item文件，**Sensor1.itm** 为第二个Sensor的Item文件，具体的Sensor Item的含义如下表。

|Item名称|值|含义|
|--|--|-----------|
| `sensor0.interface`   |`mipi`| Sensor接口类型。   <br><br>`mipi` - MIPI接口   <br>`dvp` - DVP接口   <br>`camif` - CAMIF接口    <br>`camif-csi` - MIPI转CAMIF接口|
| `sensor0.name`   |`ar330mipi` | Sensor名字。<br><br>需与**system/hlibcamsensor/include/sensors/sensor_name.h**定义的名字一致|
| `sensor0.mclk`     |`24000000`| Sensor输入时钟值。|
| `sensor0.imager`   |`0`|  ISP输入接口选择。 <br><br>对于`Q3F` `Q3-ECO`： <br>`0` - MIPI接口 <br>`1` - DVP接口 <br><br>当`sensor0.interface`为`camif-csi`时，imager的第0位用来表示invvsync，帧同步信号反转，第1位用来表示invhref，行同步信号反转|
| `sensor0.i2c`   | `chn.3.addr.32` | Sensor的I2C通道与地址，注意此地址为十进制地址，且地址不需要右移一位 |
| `sensor0.csi`   |`lanes.1.freq.384`|MIPI Sensor的Lanes数, CSI物理PHY的适配频率大小|
| `sensor0.mode` | `0`| 不同的`mode`对应不同的Sensor配置。<br><br>当`sensor0.interface`为`camif-csi`时，`mode`为MIPI传输的数据类型，可选：`Raw10`, `Raw12`, `YUV` |
| `sensor0.mipi_pixclk` |  `100000000` |MIPI PHY Pixel Clock大小。<br><br>当`sensor0.interface`为`camif-csi`时需要根据Sensor输出的Pixel Clock来配置|
| `sensor0.mclk.src` |  `imap-clk1` |Sensor输入Clock源，名字需与**kernel/arch/arm/xxx/clk.c**中定义一致|
| `sensor0.wraper.use` |  `0` | 是否使用DVP的Wrapper功能。<br><br>`1` - 使用 <br>`0` - 不使用 <br><br>只针对`Q3F`和`Q3-ECO`有效|
| `sensor0.wraper.width`  |  `1920` | DVP Wrapper输出宽度 |
| `sensor0.wraper.height`  | `1080` | DVP Wrapper输出高度 |
| `sensor0.wraper.hdelay`  |  `114` | DVP Wrapper hdelay值，即图像左上角水平方向的起点 |
| `sensor0.wraper.vdelay`  |  `12` | DVP Wrapper vdelay值，即图像左上角垂直方向的起点 |
| `sensor0.bootup.seq.0`    | `pmu.ldo7.2800000.0`|上电时序1 <br><br>设置PMU ldo7为2.8v，delay时间为0ms，若平台不用PMU而用GPIO，配置为`pmu.gpio.120.0`的形式，其中`120`为Pin脚ID|
| `sensor0.bootup.seq.1`    | `pmu.ldo8.1800000.0`|上电时序2 <br><br>设置PMU ldo8为1.8v，delay时间为0ms，若平台不用PMU而用GPIO，配置为`pmu.gpio.120.0`的形式，其中`120`为Pin脚ID|
| `sensor0.bootup.seq.2`    | `mclk.gpio.114.0`|上电时序3 <br><br>开启Sensor的输入Clock|
| `sensor0.bootup.seq.3`	 | `reset.gpio.44.20`| 上电时序4 <br><br>设置Sensor的Reset引脚|
| `sensor0.bootup.seq.4`    | `pwd.gpio.144.0` |上电时序5 <br><br>设置Sensor Power Down引脚，这里Pin脚ID为144，delay 0ms <br><br>若该Sensor Power Down脚为低电平有效，则可配置为`pwd.gpio.-144.0` |
| `sensor0.bootup.seq.5`    | `extra.gpio.120.20`|上电时序6 <br><br>额外GPIO，这里Pin脚ID为120，delay 20ms |

Note:
1. Sensor Item的解析是由Sensor的驱动自动解析的，添加新的Sensor时不用关心怎么解析它，只需按照相应的含义配置正确就可以了。
2. Sensor0 Item以sensor0开头，Sensor1 Item以sensor1开头，类推。

### 9.3 Sensor配置文件说明
**sensor0-config.txt**为Sensor的初始化配置文件，当调试Sensor时，需要修改Sensor配置可直接修改此文件即可，目前其格式支持两种形式。

|表示 | 值 |
|-----|
| 属性 | 值 |
| I2C地址 | 值 |

Note:
1. 属性仅支持: Sensor配置输出的图像的宽和高，即`w`和`h`，但可以根据需要自行添加，只需要在Sensor API函数实现自行解析就可。

以**q3fevb_va**平台项目下AR330 MIPI的初始化配置文件为例，Sensor配置文件写法如下。
```bash
w, 1920
h, 1088
//[AR330-1088p 30fps 1lane Fvco = ? MHz]
0x301A, 0x0058, //RESET_REGISTER = 88
0x302A, 0x0005, //VT_PIX_CLK_DIV = 5
0x302C, 0x0004, //VT_SYS_CLK_DIV -------------------[2 - enable 2lanes], [4 - enable 1lane]
0x302E, 0x0001, //PRE_PLL_CLK_DIV = 1
0x3030, 0x0020, //PLL_MULTIPLIER = 32
0x3036, 0x000A, //OP_PIX_CLK_DIV = 10
0x3038, 0x0001, //OP_SYS_CLK_DIV = 1
0x31AC, 0x0A0A, //DATA_FORMAT_BITS = 2570
0x31AE, 0x0201, //SERIAL_FORMAT = 514 ------ [0x201 1 lane] [0x202 2 lanes] [0x204 4 lanes]
0x31B0, 0x003E, //FRAME_PREAMBLE = 62
0x31B2, 0x0018, //LINE_PREAMBLE = 24
0x31B4, 0x4F66, //MIPI_TIMING_0 = 20326
0x31B6, 0x4215, //MIPI_TIMING_1 = 16917
0x31B8, 0x308B, //MIPI_TIMING_2 = 12427
0x31BA, 0x028A, //MIPI_TIMING_3 = 650
0x31BC, 0x8008, //MIPI_TIMING_4 = 32776
0x3002, 0x00EA, //Y_ADDR_START = 234
0x3004, 0x00C6, //X_ADDR_START = 198
0x3006, 0x0529, //Y_ADDR_END = 1313 //21->29 fengwu
0x3008, 0x0845, //X_ADDR_END = 2117
0x300A, 0x0450, //0x4bcFRAME_LENGTH_LINES = 1212
0x300C, 0x0484, //0x840LINE_LENGTH_PCK = 2112//484
0x3012, 0x02D7, //COARSE_INTEGRATION_TIME = 727
0x3014, 0x0000, //FINE_INTEGRATION_TIME = 0
0x30A2, 0x0001, //X_ODD_INC = 1
0x30A6, 0x0001, //Y_ODD_INC = 1
0x3040, 0x0000, //READ_MODE = 0
0x3042, 0x0080, //EXTRA_DELAY = 128
0x30BA, 0x002C, //DIGITAL_CTRL = 44
0x3088, 0x80BA, //SEQ_CTRL_PORT
0x3086, 0x0253, //SEQ_DATA_PORT  
0x31E0, 0x0303,
0x3064, 0x1802,
0x3ED2, 0x0146,
0x3ED4, 0x8F6C,
0x3ED6, 0x66CC,
0x3ED8, 0x8C42,
0x3EDA, 0x88BC,
0x3EDC, 0xAA63,
0x305E, 0x00A0,
```
Note:
1. 文件中每一行对应一个有效的属性与属性值或I2C地址与对应的值，每个列项用`,`分隔。
2. 每行中可以有空格，或注释，不能又其它的字符，注释用`//`表示。
3. `w`，`h`为属性，其它的行为I2C地址与值，每一行对应Sensor的寄存器的地址与值，必须用`0x`的十六进制表示。

### 9.4 MIPI UI定义
`UI`表示传输1 Bit数据需要的时间大小，如下图

 ![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/mipi_ui.png "mipi ui timing")

**举例**
若`MIPI_CLK` = 160M时，每个Lane的Bit速率为2 x `MIPI_CLK` = 320Mbps，此时
```
UI = 1s / 320000000 = 3.124ns
```
Note:
`s`为时间单位秒，`ns`为纳秒，`Mbps`为1秒钟320兆个二进制Bit位。
