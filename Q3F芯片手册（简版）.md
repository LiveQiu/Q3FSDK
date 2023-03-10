# 产品概述 
## 概述
Q3F影像处理器芯片集成了高性能低功耗的ARM Cortex-A5 处理器（单核）。采用H.264视频编码技术，最高支持1080p@30帧的视频录制。内置高性能ISP，提供更高质量的图像和视频处理能力；内嵌一个32比特高性能定点DSP核，可支持多种音频处理算法；超低功耗的特性使终端产品可应用于多种环境并同时保证高质量的视频录制。丰富的外设接口和内置DDR存储器可有效降低方案的整体成本和功耗。

Q3F芯片主要应用于720p以及1080p的IP摄像机产品中，同时也可应用在行车记录仪、运动DV、智能门铃、故事机以及航拍器等产品中。

![Q3F block diagram](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Q3Fblockdiagram.JPG)

## 特点
### CPU内核
* ARM Cortex-A5 单核处理器, 最高工作频率550MHz
* 16KB一级指令缓存，16KB一级数据缓存
### DSP内核
* 32比特高性能定点DSP内核，最高工作频率256MHz
* 16KB指令缓存、16KB数据缓存、16KB程序存储空间和48KB数据存储空间
* 与CPU采用IPC交互机制
* 全空间访问
### 存储系统
* 96KB系统引导ROM
* SPI Nor/Nand-Flash接口(1/2/4线)
* 支持eMMC5.0接口
* DRAM 存储器接口
    * 内嵌512Mb 16比特 DDR2 
    * 最高时钟频率支持533MHz 
### 引导系统
* 96KB系统引导ROM
* 104KB 片上RAM，包括8KB独立RAM和96KB与编码模块共享的复用RAM
* 支持多种引导模式
    * 片外SPI Nor-Flash存储空间引导
    * 片外SPI Nand-Flash存储空间引导
    * 片外SD/eMMC 存储空间引导
    * USB 主机引导
* 支持系统镜像从外部SPI Nor/Nand-Flash存储器、SD/eMMC接口、USB主机或设备加载
### 视频编码器
* 支持的标准
    * H.264 BP/MP/HP Level 5.1
    * JPEG Baseline (DCT顺序)
    * MVC 扩展 Stereo High
* 多码流实时编码能力
    * 1080p@30fps+VGA@30fps+JPEG 抓拍 1080p@1fps
    * 编码能力取决于编码器的工作频率
* H.264/MVC 特点
    * 输入数据格式 YCbCr注1、RGB
    * 输出数据格式 H.264/MVC
    * 支持的图像大小 144 x 96 到1920 x 1080 步幅4个像素
    * 比特率 最小: 10kbps 最大: 40 Mbps（1080p @60 fps）
    * 运动检测
    * 感兴趣的区域
    * 帧内的区域
    * 循环帧内刷新 
* JPEG 特点
    * 输入数据格式  YCbCr注1、RGB
    * 输出数据格式 JFIF 文件格式1.02、Non-progressive JPEG
    * 支持图像大小 96 x 32 到8192 x 8192（64M）步幅4个像素
    * 缩略图插入 支持RGB 8比特, RGB 24比特和JPEG压缩的缩略图
注1: 编码器内部只使用4:2:0格式的图像

* 预处理特性
    * RGB 和YCbCr 4:2:0颜色空间转换
    * YCbCr 4:2:2到YCbCr 4:2:0 颜色空间转换
    * 裁剪 从8192 x 8192到任何支持的编码大小
    * 选择 90/270度选择
    * 图像缩小
### 图像与视频子系统
* 视频输入接口
    * 支持BT.656并行接口，时钟频率最高 150MHz
    * 支持BT.601并行接口，时钟频率最高 150MHz
    * 支持4xLane MIPI接口（4Gbps）
    * 支持与 SONY、 Aptina、 OmniVision、 Panasonic 等主流高清 CMOS 对接
    * 兼容多种 sensor 电平
    * 提供可编程 sensor 时钟输出
* ISP
    * 支持黑电平校正
    * 支持镜头阴影校正
    * 支持2D降躁和独立3D降躁
    * 支持坏点校正
    * 支持色差校正
    * 支持3A功能
    * 支持闪烁检测
    * 支持Gamma矫正
    * 支持图像增强和缩放
    * 支持去雾
    * 支持全局和区域直方图
    * 支持Tone mapping
* ISP后处理
    * 支持用户定义的镜头畸变校正（包括鱼眼）
    * 支持视频稳定局部运动矢量评估 (12可编程区域)
    * 视频稳定图像配准
    * 卷帘快门补偿
    * 时域噪声滤波
    * 图像旋转（90/270度）和翻转(水平和垂直)
    * 支持图层叠加功能
    * 两个独立的输出缩小模块（12bit缩放因子）
    * 和ISP有直接数据访问通道，可降低功耗和带宽
    * 和视频编码器有直接数据访问通道，可降低功耗和带宽
    * 支持人脸/运动物体检测
### 显示子系统 
* 支持BT656/601接口
* 支持BT1120接口
* 支持sRGB接口
* 支持RGB接口
* 最高输出时钟频率75MHz
* 支持两层叠加功能
* 支持消抖功能
* 内嵌灵活配置的缩放模块，缩放功能可在OSD前也可在OSD后
### 安全引擎 
* 硬件实现AES/DES/3DES加解密算法
* 支持DMA数据传输方式
* 硬件实现SHA加密算法(CPU数据传输模式)
* 内部集成512bit OTP 存储空间和硬件随机数发生器
### Connectivity 
* 一个十兆/百兆速率Ethernet MAC
    * 兼容标准IEEE 802.3-2002
    * 支持全双工和半双工模式
    * 10/100Mbps自适应
    * RMII PHY接口
* 三个SD/SDHC/SDIO/eMMC接口
    * 兼容SD3.0标准
    * 兼容SDIO3.0标准
    * 兼容eMMC5.0标准
* 一个USB OTG 接口
    * 和USB Host复用PHY
    * 支持USB2.0标准
    * 支持UHCI1.1标准
    * 支持OTG1.0规范
* 一个USB 2.0 Host接口
    * 兼容EHCI 1.0和OHCI 1.0协议
    * 和USB OTG复用PHY
* 音频接口
    * 支持PCM和IIS接口
    * PCM接口支持主从模式
    * IIS接口只支持主模式
* 四个 IIC接口
    * 支持100Kbps/400Kbps/3.4Mbps
    * 支持7比特和10比特地址模式
    * 支持批量传输
* 两路一线SSP接口
    * 支持主从模式
    * 兼容SPI、Microwire和 TI多种格式
    * SSP0 和多线SPI复用接口
* 多线SPI 接口
    * 和SSP0复用接口
    * 只支持主模式
    * 支持1、2和4线模式
    * 接口协议支持Motorola SPI
* 三路 UART 接口
    * 三路独立uart接口，包括一个4线和两个2线接口
    * 兼容IRDA1.0 和RS232
* 红外接口
    * 兼容NEC
    * 兼容SAA3010 RC-5 
* 两通道SAR ADC
### 通用输入输出接口
* 支持灵活可配的GPIO
* 部分GPIO可作为外部中断输入
### 系统设备
* 六通道DMA 控制器
    * 6个独立传输通道
    * 支持指令集机制以实现灵活编程配置DMA传输
* 看门狗时钟
    * 支持喂狗超时系统重启复位
* PWM 时钟
    * 五个PWM 时钟
    * 支持PWM 输出
    * 第一个PWM时钟支持死区信号输出
* 全局时钟
    * 两个32比特全局时钟
* RTC (实时时钟) 
* PLL
    * 四个PLL，以实现不同特征频率的需求
    * PLL最高输出频率（APLL:550MHz, DPLL:1536MHz, EPLL:1188MHz, VPLL:1056MHz）
    * 支持多个时钟分频电路
### 封装与工艺
* TFBGA10x11
* TSMC 40LP
