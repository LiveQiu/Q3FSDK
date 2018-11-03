# Q3F EVB开发套件标准版使用说明书

# 概述
Q3F EVB开发套件标准版是一款基于InfoTM Q3F（Q3420P）处理器的音视频整体解决方案。用户可以方便地使用开发套件进行二次开发。开发平台为用户提供了强大的处理性能和调试资源，形成完整的软硬体解决方案。

# 硬件规格说明
开发套件规格
开发套件主要规格。

|功能	|描述|
|:-------------: |:---------------------------------------------------------|
|CPU	|Cortex A5 550MHz ARM，16KB L1 I-cache，16KB L1 D-cache，<br>H.264 1080p@30fps+VGA@30fps+JPEG|
|DDR	|CPU内置64MB（or 128MB） 16bit DDR2，up to 533MHz|
|Flash	|SPI Nor Flash 32MB|
|DSP	|CPU内置32bit， up to 256MHz，16KB I-cache，16KB D-cache|
|Camera Sensor	|MIPI OV4689|
|LCD	|srgb 3.0inch，16:9，FY30002，320x240|
|WiFi	|AP6212|
|接口	|SD卡槽、以太、喇叭、麦克、USB 2.0 OTG、按键|
|OS	|Linux + InfoTM QSDK|
	
其中芯片主控Q3F（Q3420P）处理器是开发套件的核心，详细数据见[芯片手册](https://github.com/InfoTM-SDK/Q3FSDK/wiki/Q3420P%E8%8A%AF%E7%89%87%E6%89%8B%E5%86%8C%EF%BC%88%E7%AE%80%E7%89%88%EF%BC%89)。

开发套件如下

![Q3F evb](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fevb.JPG)

开发套件的架构示意图

![Q3F evb block](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/Q3Fevbblockdiagram.JPG)

# EVB功能说明
- 开机启动
    + 设备上电，快速开机
    + 开机过程，指示灯（上电、网络状态等指示）
    + UI启动成功，Camera采集图像显示成功

- 快速联网
    + 开机后首次联网，二维码扫描联网/声波联网
    + 其他联网方法，通过UI进行配置或手机APP UI进行配置

- 插入SD卡
    + SD卡格式化
    + 视频、图片文件保存
    + 文件自动覆盖

- 远程P2P连接
    + 远程音视频直播720P
    + 语音对讲（回声消除）
    + 远程视频文件回放

- 本地控制
    + UI操作（物理按键操作MENU、OK、UP、DOWN）
    + 录像（H264 1080P 30fps、720P 30fps）
    + 拍照（JPEG 400万）
    + 图像设置（分辨率、曝光、锐度、白平衡、色彩、ISO）
    + 数字水印功能
    + 其他产品所需功能

# 基于开发套件和SDK开发和调试
* 开发环境搭建
* SDK配置和编译
* 开发套件固件手动烧录升级
* 调试接口
* 产测