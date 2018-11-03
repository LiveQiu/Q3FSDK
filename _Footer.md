Q3F EVB开发套件标准版使用说明书
1.概述
Q3F EVB开发套件标准版是一款基于InfoTM Apollo2处理器的音视频整体解决方案。用户可以方便地使用开发套件进行二次开发。开发平台为用户提供了强大的处理性能和调试资源，形成完整的软硬体解决方案。

2.硬件规格说明
开发套件规格
开发套件主要规格。
功能	描述
CPU	Apollo2（Q3F）
Cortex A5 550MHz ARM，16KB L1 I-cache，16KB L1 D-cache，
H.264 1080p@30fps+VGA@30fps+JPEG
DDR	CPU内置64MB 16bit DDR2，up to 533MHz
Flash	SPI Nor Flash 32MB
DSP	CPU内置32bit 256MHz，16KB I-cache，16KB D-cache
Camera Sensor	MIPI OV4689
LCD	srgb 3.0inch，16:9，FY30002，320x240
WiFi	AP6212
接口	SD卡槽、以太、喇叭、麦克、USB 2.0 OTG、按键
OS	Linux + InfoTM QSDK
	

其中芯片主控Q3F处理器是开发套件的核心，详细数据见芯片手册。

3.功能说明
开机启动
快速开机
开机过程，指示灯（上电、网络、录像等指示）
UI启动成功，Camera采集图像显示成功
快速联网
开机后联网，二维码扫描联网/声波联网
插入SD卡（自动覆盖、格式化、保存录像、图片等）
远程P2P连接（尚云、TUTK）
P2P程序（远程音视频直播720P、语音对讲（回声消除）、远程文件回放）
本地控制
UI操作（物理按键操作MENU、OK、UP、DOWN、POWER）
录像（H264 1080P 30fps、720P 30fps）
拍照（JPEG 400万）
音量调节
图像设置（分辨率、曝光、锐度、白平衡、色彩、ISO）
数字水印
通过USB连接到PC或手机
UVC摄像头功能
英语评测功能（待集成）
智能语音平台功能（待集成）

其中TUTK、DuerOS、英语评测功能时间点再评估。

4.调试功能说明
串口调试
配置和编译
烧录升级
产测