# 示例代码规划

## 1 示例代码结构
## 1.1 示例代码目录
示例代码请放在**QSDK/system/app-demos**目录，目录结构
```bash
app-demos
├─audiobox
│  ├─001
│  │      demo_main.cpp
│  │
│  ├─002
│  │      demo_main.cpp
│  │
│  └─003
│         demo_main.cpp
│
├─camera
│  ├─001
│  │      demo_main.cpp
│  │      path.json
│  │
│  ├─002
│  │      demo_main.cpp
│  │      path.json
│  │
│  └─003
│         demo_main.cpp
│         path.json
│
├─display
├─player
├─recorder
├─uvc
├─va
│  ├─001
│  └─002
├─video
└─wi-fi
```

Note: 示例目录都用`001`，`002`，`003`......编号，模块内示例编号自行分配。示例代码文件名都叫`demo_main.cpp`，Videobox的启动JSON文件都叫`path.json`

## 1.2 示例代码编译
示例代码默认不编译，输入**make menuconfig**选择以下配置，如下所示。
```
QSDK Options --->
   |-Debug Tools -->
   |   |-[*] app demos
   |   |   |
```
选中之后编译会在目录**/system/usr/bin/**下生成所有demo的可执行文件。在第一次全部编译之后可以部分编译，输入命令`make app-demos-rebuild`。

**示例编号** - 填写示例代码的编号
**描述** - 描述示例代码功能
**使用说明** - 如何使用示例代码？能看到什么现象？
**备注** - 示例代码的适用范围，芯片`Apollo` `Apollo-2` `Apollo-ECO`，硬件/软件要求等

## 2 示例代码-BSP
## 2.1 UVC
**示例列表**

* `[001]` 使用UVC协议传输H264图像
* `[002]` 使用UVC协议传输MJPEG图像
* `[003]` 使用UVC协议设置MJPEG分辨率
* `[004]` 使用UVC协议设置H264分辨率
* `[005]` 使用UVC协议设置曝光时间、曝光模式等参数
* `[006]` 使用UVC协议设置亮度、增益、色度、对比度、锐度等图像参数

### 2.1.1 示例编号(001)
**描述**
使用UVC协议传输H264图像
**使用说明**
1.确定videoboxd等应用程序正常运行，**将path.json**以及**demo_uvc_001**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装VLC应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_001**
4.在UBUNTU上执行**cvlc -v v4l2:///dev/video0:chroma="H264":width=640:height=480:fps=15 **
5.在VLC界面上可以看到H264视频
**备注**
支持`apollo2`

### 2.1.2 示例编号(002)
**描述**
使用UVC协议传输MJPEG图像
**使用说明**
1.确定videoboxd等应用程序正常运行，将**path.json**以及**demo_uvc_002**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装VLC应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_002**
4.在UBUNTU上执行**cvlc -v v4l2:///dev/video0:chroma="MJPG":width=640:height=480:fps=15 **
5.在VLC界面上可以看到MJPG视频
**备注**
支持`apollo2`

### 2.1.3 示例编号(003)
**描述**
使用UVC协议设置MJPEG分辨率
**使用说明**
1.确定videoboxd等应用程序正常运行，将**path.json**以及**demo_uvc_003**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装VLC应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_003**
4.在UBUNTU上执行**cvlc -v v4l2:///dev/video0:chroma="MJPG":width=640:height=480:fps=15 **
5.在VLC界面上可以看到MJPG视频
6.在UBUTNU上退出**cvlc**，执行**cvlc -v v4l2:///dev/video0:chroma="MJPG":width=1280:height=720:fps=15**
7.可以观察到VLC界面上MJPG视频与步骤5相比有分辨率变化
**备注**
支持`apollo2`

### 2.1.4 示例编号(004)
**描述**
使用UVC协议设置H264分辨率
**使用说明**
1.确定videoboxd等应用程序正常运行，将**path.json**以及**demo_uvc_004**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装VLC应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_004**
4.在UBUNTU上执行**cvlc -v v4l2:///dev/video0:chroma="H264":width=640:height=480:fps=15 **
5.在VLC界面上可以看到H264视频
6.在UBUTNU上退出**cvlc**，执行**cvlc -v v4l2:///dev/video0:chroma="H264":width=1280:height=720:fps=15**
7.可以观察到VLC界面上H264视频与步骤5相比有分辨率变化
**备注**
支持`apollo2`

### 2.1.5 示例编号(005)
**描述**
使用UVC协议设置曝光时间、曝光模式等参数
**使用说明**
1.确定videoboxd等应用程序正常运行，将**path.json**以及**demo_uvc_005**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装GUVCVIEW应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_005**
4.在UBUNTU上执行**./guvcview/guvcview -d /dev/video0 --resolution=640x480 -u h264**
5.在GUVCVIEW界面上调整曝光时间，曝光模式等参数，可以设置成功
**备注**
支持`apollo2`

### 2.1.6 示例编号(006)
**描述**
使用UVC协议设置亮度、增益、色度、对比度、锐度等图像参数
**使用说明**
1.确定videoboxd等应用程序正常运行，将**path.json**以及**demo_uvc_006**可执行文件放到同一个目录
2.以UBUNTU环境为例，PC端需要安装GUVCVIEW应用程序
3.USB连接到PC端，待枚举成功后，在串口执行**demo_uvc_006**
4.在UBUNTU上执行**./guvcview/guvcview -d /dev/video0 --resolution=640x480 -u h264**
5.在GUVCVIEW界面上调整参亮度、增益、色度、对比度、锐度等参数，可以设置成功
**备注**
支持`apollo2`

## 2.2 Wi-Fi
**示例列表**

* `[001]` STA模式打开-扫描附近AP-获取扫描到的AP信息-连接其中一个AP-获取Wi-Fi连接信息-断开连接-断开连接后重新打开STA模式-自动重连AP-正常退出
* `[002]` STA模式打开-扫描附近AP-获取扫描到的AP信息-连接其中一个AP-断开连接同时删除之前的所有连接信息-重新打开STA模式-不会自动重连AP-正常退出
* `[003]` AP模式打开-手机端Wi-Fi使用系统默认设置连接该AP-获取Wi-Fi连接信息-AP模式关闭-正常退出
* `[004]` 设置新的AP模式配置-确认该设置成功了-AP模式打开-手机端Wi-Fi使用新设置连接该AP-获取Wi-Fi连接信息-AP模式关闭-正常退出

**备注**

需要注意的是如果是制作IUS烧录卡，那么在编译之前需要修改如下三个文件（其中`XXX`表示产品型号）：
```
products/XXX/items.itm
products/XXX/system/etc/fstab
products/XXX/system/etc/init.d/S900initdir
```
**products/XXX/items.itm**文件改动如下(`-`号表示删除该行，`+`号表示增加该行，下同)：
```cpp
-part7                                  config.512.fs
+part7                                  config.512.fs.fat
```
**products/XXX/system/etc/fstab**文件改动如下：
```cpp
 tmpfs           /tmp           tmpfs    defaults          0      0
+tmpfs           /config           tmpfs    defaults          0      0
 sysfs          /sys           sysfs    defaults          0      0
```
**products/XXX/system/etc/init.d/S900initdir**整个文件代码如下：
```bash
#! /bin/sh
case "$1" in
        start|"")
		fsck.fat -a /dev/spiblock2
        if [ ! -d /config ];then
                 mkdir -p /config
        fi
		ln -s /config /mnt/
		mount -a
        if [ ! -d /config/wifi ];then
                 mkdir -p /config/wifi
                 cp /wifi/*.conf /config/wifi
        fi
        if [ ! -d /config/bt ];then
                mkdir -p /config/bt
        fi
                ;;
        stop)
                ;;
        *)
                echo "Usage: initdir {start|stop}" >&2
                exit 1
                ;;
esac
```

### 2.2.1 示例编号(001)

**描述**

STA模式打开-扫描附近AP-获取扫描到的AP信息-连接其中一个AP-获取Wi-Fi连接信息-断开连接-断开连接后重新打开STA模式-自动重连AP-正常退出

**使用说明**

### Step 1: 确认wifi程序启动

使用`ps`命令确认下wifi进程在后台运行
```
/ # ps
PID   USER     COMMAND
  500 root     cepd
  517 root     audiobox
  519 root     wifi
```
如果没有运行，那么先启动该程序
```
/ # wifi &
```

### Step 2: 启动demo_wi-fi_001程序

启动`demo_wi-fi_001`程序，程序执行成功会出现如下提示和一个`%`号
```
/ # demo_wi-fi_001
Please enter your command number:
            1 sta_enable
            2 sta_disable
            3 sta_scan
            4 sta_connect
            5 sta_disconnect
            6 get_wifi_state
            7 get_net_info
            8 exit
%
```
在`%`号之后输入字符串`1`，回车
```
% 1
your command is 1
% wifi add cmd sta enable
```
在`%`号之后输入字符串`3`，回车
```
% 3
your command is 3
% wifi add cmd sta scan
```
在`%`号之后输入字符串`4`，回车，按照提示输入`SSID`和`PSK`，回车
```
% 4
your command is 4
Please enter SSID and PSK for sta_connect:
Demo 12345678
```
在`%`号之后分别输入字符串`6`，回车，`7`，回车
```
% 6
your command is 6
s32GetState is 0x4
% 7
your command is 7
hwaddr is ac:83:f3:35:9e:e6
ipaddr is 192.168.43.183
gateway is 192.168.43.1
mask is 255.255.255.0
dns1 is 192.168.43.1
dns2 is 0.0.0.0
server is 0.0.0.0
lease is 0.0.0.0
%
```
在`%`号之后输入字符串`5`，回车，按照提示输入`SSID`，回车
```
% 5
your command is 5
Please enter SSID for sta_disconnect:
Demo
```
在`%`号之后输入字符串`2`，回车，
```
% 2
your command is 2
% wifi add cmd sta disable
```
在`%`号之后输入字符串`1`，回车
```
% 1
your command is 1
% wifi add cmd sta enable
```
在`%`号之后分别输入字符串`6`，回车，`7`，回车
```
% 6
your command is 6
s32GetState is 0x4
% 7
your command is 7
hwaddr is ac:83:f3:35:9e:e6
ipaddr is 192.168.43.183
gateway is 192.168.43.1
mask is 255.255.255.0
dns1 is 192.168.43.1
dns2 is 0.0.0.0
server is 0.0.0.0
lease is 0.0.0.0
%
```
在`%`号之后输入字符串`8`，回车，退出
```
% 8
your command is 8
```

**备注**

支持`Apollo`，`Apollo-2`和`Apollo-ECO`

### 2.2.2 示例编号(002)

**描述**

STA模式打开-扫描附近AP-获取扫描到的AP信息-连接其中一个AP-断开连接同时删除之前的所有连接信息-重新打开STA模式-不会自动重连AP-正常退出

**使用说明**

### Step 1: 确认wifi程序启动

使用`ps`命令确认下wifi进程在后台运行
```
/ # ps
PID   USER     COMMAND
  500 root     cepd
  517 root     audiobox
  519 root     wifi
```
如果没有运行，那么先启动该程序
```
/ # wifi &
```

### Step 2: 启动demo_wi-fi_002程序

启动`demo_wi-fi_002`程序，程序执行成功会出现如下提示和一个`%`号
```
/ # demo_wi-fi_002
Please enter your command number:
            1 sta_enable
            2 sta_disable
            3 sta_scan
            4 sta_connect
            5 sta_forget
            6 get_wifi_state
            7 get_net_info
            8 exit
%
```
在`%`号之后输入字符串`1`，回车
```
your command is 1
% wifi add cmd sta enable
```
在`%`号之后输入字符串`3`，回车
```
% 3
your command is 3
% wifi add cmd sta scan
```
在`%`号之后输入字符串`4`，回车，按照提示输入`SSID`和`PSK`，回车
```
% 4
your command is 4
Please enter SSID and PSK for sta_connect:
Demo 12345678
```
在`%`号之后分别输入字符串`6`，回车，`7`，回车
```
% 6
your command is 6
s32GetState is 0x4
% 7
your command is 7
hwaddr is ac:83:f3:35:9e:e6
ipaddr is 192.168.43.183
gateway is 192.168.43.1
mask is 255.255.255.0
dns1 is 192.168.43.1
dns2 is 0.0.0.0
server is 0.0.0.0
lease is 0.0.0.0
%
```
在`%`号之后输入字符串`5`，回车，按照提示输入`SSID`，回车
```
% 5
% 5
your command is 5
% wifi add cmd sta forget
wifi s[   66.783333] CFG80211-ERROR) wl_cfg80211_disconnect : ervice cmd wake 4
Reason 3
```
在`%`号之后输入字符串`2`，回车，
```
% 2
your command is 2
% wifi add cmd sta disable
```
在`%`号之后输入字符串`1`，回车
```
% 1
your command is 1
% wifi add cmd sta enable
```
在`%`号之后分别输入字符串`6`，回车，`7`，回车
```
% 6
your command is 6
s32GetState is 0x3
% 7
your command is 7
%
```
在`%`号之后输入字符串`8`，回车，退出
```
% 8
your command is 8
```

**备注**

支持`Apollo`，`Apollo-2`和`Apollo-ECO`

### 2.2.3 示例编号(003)

**描述**

AP模式打开-手机端Wi-Fi使用系统默认设置连接该AP-获取Wi-Fi连接信息-AP模式关闭-正常退出

**使用说明**

### Step 1: 确认wifi程序启动

使用`ps`命令确认下wifi进程在后台运行
```
/ # ps
PID   USER     COMMAND
  500 root     cepd
  517 root     audiobox
  519 root     wifi
```
如果没有运行，那么先启动该程序
```
/ # wifi &
```

### Step 2: 启动demo_wi-fi_003程序

启动`demo_wi-fi_003`程序，程序执行成功会出现如下提示和一个`%`号
```
/ # demo_wi-fi_003
Please enter your command number:
            1 ap_enable
            2 ap_disable
            3 get_wifi_state
            4 get_net_info
            5 exit
%
```
在`%`号之后输入字符串`1`，回车
```
% 1
your command is 1
% wifi add cmd ap enable
```
在手机端使用Wi-Fi连接`SSID`为**infotm_debug**的AP，PSK为**12345678**，然后在`%`号之后输入字符串`3`，回车，输入`4`，回车
```
% 3
your command is 3
s32GetState is 0x20
% 4
your command is 4
hwaddr is ac:83:f3:35:9e:e6
ipaddr is 192.168.43.183
gateway is 192.168.43.1
mask is 255.255.255.0
dns1 is 192.168.43.1
dns2 is 0.0.0.0
server is 0.0.0.0
lease is 0.0.0.0
%
```
在`%`号之后输入字符串`2`，回车
```
% 2
your command is 2
% wifi add cmd ap disable
```
在`%`号之后输入字符串`5`，回车，退出
```
% 5
your command is 5
```

**备注**

支持`Apollo`，`Apollo-2`和`Apollo-ECO`

### 2.2.1 示例编号(004)

**描述**

设置新的AP模式配置-确认该设置成功了-AP模式打开-手机端Wi-Fi使用新设置连接该AP-获取Wi-Fi连接信息-AP模式关闭-正常退出

**使用说明**

### Step 1: 确认wifi程序启动

使用`ps`命令确认下wifi进程在后台运行
```
/ # ps
PID   USER     COMMAND
  500 root     cepd
  517 root     audiobox
  519 root     wifi
```
如果没有运行，那么先启动该程序
```
/ # wifi &
```

### Step 2: 启动demo_wi-fi_004程序

启动`demo_wi-fi_004`程序，程序执行成功会出现如下提示和一个`%`号
```
/ # demo_wi-fi_004
Please enter your command number:
            1 ap_enable
            2 ap_disable
            3 ap_set_config
            4 ap_get_config
            5 get_wifi_state
            6 get_net_info
            7 exit
```
在`%`号之后输入字符串`3`，回车，输入`4`，回车（确认`SSID`为**ap_test**，PSK为**11111111**的AP设置成功）
```
% 3
your command is 3
% 4
your command is 4
network_id is 0
status is 0
list_type is 0
level is 0
channel is 1
ssid is ap_test
bssid is
key_management is wpa2-psk
pairwise_ciphers is
psk is 11111111
```
在`%`号之后输入字符串`1`，回车
```
% 1
your command is 1
% wifi add cmd ap enable
```
在手机端使用Wi-Fi连接`SSID`为**ap_test**的AP，PSK为**11111111**，然后在`%`号之后输入字符串`5`，回车，输入`6`，回车
```
% 5
your command is 6
s32GetState is 0x20
% 6
your command is 6
hwaddr is ac:83:f3:35:9e:e6
ipaddr is 192.168.43.183
gateway is 192.168.43.1
mask is 255.255.255.0
dns1 is 192.168.43.1
dns2 is 0.0.0.0
server is 0.0.0.0
lease is 0.0.0.0
%
```
在`%`号之后输入字符串`2`，回车
```
% 2
your command is 2
% wifi add cmd ap disable
```
在`%`号之后输入字符串`7`，回车，退出
```
% 7
your command is 7
```

**备注**

支持`Apollo`，`Apollo-2`和`Apollo-ECO`

## 2.3 AudioBox
**示例列表**

* `[001]` 从麦克风中录音PCM数据，不编码，直接保存到WAV格式的本地文件
* `[002]` 从WAV格式的文件中读入PCM数据，直接输出扬声器播放
* `[003]` 从麦克风中录音PCM数据，编码成G711数据，保存到WAV格式的本地文件
* `[004]` 从WAV格式的文件中读入G711A数据，并将解码后的PCM数据，直接输出扬声器播放
* `[005]` 从麦克风中录音PCM数据，不编码，但做声音的处理(目前只有AEC回声消除)，保存到WAV本地文件

### 2.3.1 示例编号(001)
**描述**
从麦克风中录音PCM数据，不编码，直接保存到WAV格式的本地文件
**使用说明**
1. 确定Audiobox服务程序存在。如不存在，使用命令**audiobox&**启动
2. 将SD卡挂载到**/mnt/sd0**目录下
3. 执行**demo_audiobox_001**开始录制
4. **/mnt/sd0**目录下将会出现录音文件**record_pcm.wav**
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 2.3.2 示例编号(002)
**描述**
从WAV格式的文件中读入PCM数据，直接输出扬声器播放
**使用说明**
1. 确定Audiobox服务程序存在。如不存在，使用命令**audiobox&**启动
2. 将SD卡挂载到**/mnt/sd0**目录下
3. 将示例编号001中生成的音频文件**record_pcm.wav**重命名为**play_pcm.wav**
4. 将音频文件**play_pcm.wav**拷贝到**/mnt/sd0**目录下
5. 执行**demo_audiobox_002**开始播放
6. 示例编号001中录制的声音将会播放出来
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 2.3.3 示例编号(003)
**描述**
从麦克风中录音PCM数据，编码成G711A数据，保存到WAV格式的本地文件
**使用说明**
1. 确定Audiobox服务程序存在。如不存在，使用命令**audiobox&**启动
2. 将SD卡挂载到**/mnt/sd0**目录下
3. 执行**demo_audiobox_003**开始录制
4. **/mnt/sd0**目录下将会出现录音文件**record_g711a.wav**
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 2.3.4 示例编号(004)
**描述**
从WAV格式的文件中读入G711A数据，并将解码后的PCM数据，直接输出扬声器播放
**使用说明**
1. 确定Audiobox服务程序存在。如不存在，使用命令**audiobox&**启动
2. 将SD卡挂载到**/mnt/sd0**目录下
3. 将示例编号003中生成的音频文件**record_g711a.wav**重命名为**play_g711a.wav**
4. 将音频文件**play_g711a.wav**拷贝到**/mnt/sd0**目录下
5. 执行**demo_audiobox_004**开始播放
6. 示例编号003中录制的声音将会播放出来
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 2.3.5 示例编号(005)
**描述**
从麦克风中录音PCM数据，不编码，但做声音的处理(目前只有AEC回声消除)，保存到WAV本地文件
**使用说明**
1. 确定Audiobox服务程序存在。如不存在，使用命令**audiobox&**启动
2. 将SD卡挂载到**/mnt/sd0**目录下
3. 将示例编号004中的音频文件**play_g711a.wav**拷贝到**/mnt/sd0**目录下
4. 执行**demo_audiobox_004**开始播放作为回音源
5. 同时执行**demo_audiobox_005**开始录制
6. **/mnt/sd0**目录下将会出现去除回音的录音文件**record_aec_pcm.wav**
**备注**
1. 支持`Apollo-2`
2. 需要在编译阶段打开AEC的相关选项

```
make menuconfig 
QSDK Options --->
   |- Libs--->-->
   |   |-[*] hlibdsp
   |   |   |-options --->
   |   |   |-->DSP Type (ceva-tl421)--->
   |   |   |-->DDR Size (64MB)--->
   |   |-*- hlibvcp7g
   |   |   |-[*] enable dsp-lib support
   
-----------
make linux-menuconfig
Device Drivers  --->
   |- [*] InfoTM special files and drivers  --->
   |- [*] Infotm Q3F Series Support  --->
   |  |-[*] Q3F dsp driver support  --->
   |  |  |-<M> ceva tl421 dsp  --->
   |  |  |  |- [*]Q3F dsp support AEC function
   
```

----
## 3 示例代码-MID
多媒体示例代码由JSON和应用程序两部分构成。应用程序中用示例JSON启动Videobox，再通过调用QSDK API完成指定功能。

## 3.1 Camera
**示例列表**

* `[001]` ISP输出到ISPOSTV2，经MARKER叠加一路日期水印，到IDS输出
* `[002]` ISP输出到ISPOSTV2，叠加多路水印，包含日期和遮挡色块，再到IDS输出
* `[003]` ISP做大分辨率拍照功能
* `[004]` ISP输出数据经ISPOST，到IDS显示
* `[005]` ISP先做裁剪，再输出数据经ISPOST，到IDS显示
* `[006]` ISP先做镜像，再输出数据经ISPOST，到IDS显示
* `[007]` YUV输出到ISPOSTV2，到IDS显示
* `[008]` ISP调整色度，对比度，饱和度，亮度，锐度，伽马，开关AE，AWB
* `[009]` ISP调整帧率，黑白模式
* `[010]` ISP示例1p/2p/4p切换
* `[011]` ISP示例机内全景展开
* `[012]` ISP配置动态帧率

### 3.1.1 示例编号(001)
**描述**

ISP输出到ISPOSTV2，经MARKER叠加一路日期水印，到IDS输出

**使用说明**

1. 将**path.json**以及**demo_camera_001**可执行文件放到同一个目录
2. 进入**demo_camera_001**可执行文件的目录后，执行**./demo_camera_001**
3. 看到预览的视频

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`
2. 水印会在整个视频区域随机位置显示

### 3.1.2 示例编号(002)
**描述**

ISP输出到ISPOSTV2，叠加多路水印，包含日期和遮挡色块，再到IDS输出

**使用说明**

1. 将**path.json**以及**demo_camera_002**可执行文件放到同一个目录
2. 进入**demo_camera_002**可执行文件的目录后，执行**./demo_camera_002**
3. 看到预览的视频

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`
2. 7个视频遮挡会同时交替改变大小

### 3.1.3 示例编号(003)
**描述**

ISP做大分辨率拍照功能

**使用说明**

1. 将**path.json**以及**demo_camera_003**可执行文件放到同一个目录
2. 进入**demo_camera_003**可执行文件的目录后，执行**./demo_camera_003**
3. 看到预览的视频

**备注**

1. 包含大分辨率连拍及大小分辨率交替拍

### 3.1.4 示例编号(004)
**描述**

ISP输出数据经ISPOST，到IDS显示

**使用说明**

1. 将**path.json**以及**lc_hermite32_1920x1080_1920x1080.bin**同**demo_camera_004**可执行文件放到同一个目录
2. 进入**demo_camera_004**可执行文件的目录后，执行**./demo_camera_004**
3. 看到预览的视频

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.5 示例编号(005)
**描述**

ISP先做裁剪变换，再输出数据经ISPOST，到IDS显示

**使用说明**

1. 将**path.json**和**lc_hermite32_1920x1080_1920x1080.bin**同**demo_camera_005**可执行文件放到同一个目录
2. 将**path_crop.json**和**lc_hermite32_1280x1280_1280x1280.bin**同**demo_camera_005**可执行文件放到同一个目录
3. 进入**demo_camera_005**可执行文件的目录后，执行**./demo_camera_005**
4. 在预览的视频中先显示1080p的原图，2s后显示crop后720p图像,crop区域:起点(200,200),宽高(1280,720)

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`
2. 修改crop 区域，需要修改path_crop.json中isp out的输出参数

|isp out port 的参数|参数描述|
|--------------------------------|---------------|
|w|isp scaler输出的width, w 小于等于 pip_w |
|h|isp scaler输出的hight, h 小于等于 pip_h|
|pip_x|isp crop 区域的坐标x轴的值|
|pip_y| isp crop 区域的坐标y轴的值|
|pip_w|isp crop 区域的width|
|pip_h|isp crop 区域的hight|


### 3.1.6 示例编号(006)
**描述**

ISP先做镜像变换，再输出数据经ISPOST，到IDS显示

**使用说明**

1. 将**path.json**和**lc_hermite32_1920x1080_1920x1080.bin**同**demo_camera_006**可执行文件放到同一个目录
2. 进入**demo_camera_006**可执行文件的目录后，执行**./demo_camera_006**
3. 在预览的视频上每隔2s出现四种方向上的镜像变化，变化顺序是: 水平－>垂直－>水平垂直－>原始

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.7 示例编号(007)
**描述**
YUV输出到ISPOSTV2，到IDS显示
**使用说明**
1. 将**path.json**同**demo_camera_007**可执行文件放到同一个目录
2. 进入**demo_camera_007**可执行文件的目录后，执行**./demo_camera_007**
3. 可以在显示屏上预览到图像
4. 输入`q`或`ctrl+c`退出demo程序
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.8 示例编号(008)
**描述**
ISP调整色度，对比度，饱和度，亮度，锐度，伽马，开关AE，AWB
**使用说明**
1. 将源代码中的函数**AdjustHue**注释去掉，并重新编译
2. 将**path.json**同**demo_camera_008**可执行文件放到同一个目录
3. 进入**demo_camera_008**可执行文件的目录后，执行**./demo_camera_008**
4. 可以预览到图像中的绿色偏黄，红色偏紫
5. 输入`q`或`ctrl+c`退出demo程序
6. 下列函数重复1，2，3，5步骤
    //s32Ret = AdjustWb(FCAM); // 图像偏蓝
    //s32Ret = EnableAe(FCAM); // 手电筒照摄像头，图像变白，白圈不缩小
    //s32Ret = AdjustYuvGamma(FCAM); // 图像变白
    //s32Ret = AdjustSharpness(FCAM); // 图像中黑色线缆边缘有白晕
    //s32Ret = AdjustBrightness(FCAM); // 图像变亮
    //s32Ret = AdjustSaturation(FCAM); // 图像中的绿色变的更绿
    //s32Ret = AdjustContrast(FCAM); // 图像中亮的区域更亮
    //s32Ret = AdjustHue(FCAM); // 图像中的绿色偏黄，红色偏紫
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.9 示例编号(009)
**描述**
ISP调整帧率，黑白模式
**使用说明**
1. 将源代码中的函数**AdjustFps**注释去掉，并重新编译
2. 将**path.json**同**demo_camera_009**可执行文件放到同一个目录
3. 进入**demo_camera_009**可执行文件的目录后，执行**./demo_camera_009**
4. 帧率控制在10~25的范围内
5. 输入`q`或`ctrl+c`退出demo程序
6. 下列函数重复1，2，3，5步骤
    //s32Ret = EnableMonochrome(FCAM); // 图像是黑白的
**备注**
1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.10 示例编号(010)

**描述**

使用DG load一张yuv图像，经过ISPOST，到IDS上显示。4s后，图像经过4p切换，显示到IDS上

**使用说明**

1. 将**path.json**，**FEIN_1088x1088_mode7_NV12.yuv**，**lc_Barrel_0-0_hermite32_1088x1088-1920x1080_fisheyes_mercator_0.bin**
    **lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_0.bin**，**lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_1.bin**
    **lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_2.bin**，**lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_3.bin**
    **lc_up_2p360_0-0_hermite32_1088x1088-1920x544_fisheyes_panorama_0.bin**，**lc_up_2p360_0-0_hermite32_1088x1088-1920x544_fisheyes_panorama_1.bin**共7个bin文件
    和一个yuv文件同**demo_camera_010**可执行文件放到同一个目录
2. 进入**demo_camera_010**可执行文件的目录后，执行**./demo_camera_010**
3. 在预览的上显示展开的图像，4s后显示4p切换后展开的图像
4. 关闭宏定义SHOW_4P。编译生成可执行文件执行**./demo_camera_010**, 将**./demo_camera_010**放在以上7个bin文件和yuv文件相同的目录
5. 进入**demo_camera_010**可执行文件的目录后，执行**./demo_camera_010**
6. 在预览的上显示展开的图像，4s后显示2p切换后展开的图像

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.11 示例编号(011)

**描述**

使用DG load一张yuv图像，经过ISPOST，全景展开到IDS上显示

**使用说明**

1. 将**path.json**，**fisheye_1080x1080.yuv**，**lc_v1_0-0-0-0_hermite32_1080x1080-1920x1088_fisheye_360.bin**同
    **demo_camera_011**可执行文件放到同一个目录
2. 进入**demo_camera_011**可执行文件的目录后，执行**./demo_camera_011**
3. 在预览的上显示全景展开的图像

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`

### 3.1.12示例编号(012)

**描述**

ISP输出数据经ISPOST，到IDS显示，根据场景不同，摄像头输出的帧率不同

**使用说明**

1. 将**path.json**，**lc_hermite32_1920x1080_1920x1080.bin**同**demo_camera_012**可执行文件放到同一个目录
2. 进入**demo_camera_012**可执行文件的目录后，执行**./demo_camera_012**
3. 在预览的上显示图像，根据场景的不同，摄像头输出的帧率不同。可以通过命令**cat /proc/fr_swap**查看

**备注**

1. 支持`Apollo-2`和`Apollo-ECO`


## 3.2 Video
**示例列表**

* `[001]` 1080P输入做JPEG保存（H1JPEG）
* `[002]` 1080P输入做JPEG保存（JENC）
* `[003]` 1080P输入做H264编码，并保存到文件
* `[004]` 720P+480P输入，经H264编码，并保存到两个文件
* `[005]` YUV输入编码H264
* `[006]` ROI编码H264
* `[007]` H1旋转后编码H264
* `[008]` H1264编码SmartP
* `[009]` ISP输入30帧，编码自定义帧率编码
* `[010]` 1080P输入做H265编码，并保存到文件
* `[011]` 720P+480P输入，H265编码，并保存到两个文件
* `[012]` YUV输入编码H265
* `[013]` ROI编码H265
* `[014]` H2旋转后编码H264
* `[015]` 编码时动态设置CBR  VBR  FixQp
* `[016]` Trigger I功能演示

### 3.2.1 示例编号 001
**描述**

1080P输入做JPEG保存(H1JPEG)

**使用说明**

1. 将**path.json**同**demo_video_001**可执行文件放到同一目录
2. 进入**demo_video_001**可执行文件的目录后，执行**./demo_video_001**
3. 将会在**/mnt/sd0/**目录下生成1080p照片

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.2 示例编号 002
**描述**

1080P输入做JPEG保存(JENC)

**使用说明**

1. 将**path.json**同**demo_video_002**可执行文件放到同一目录
2. 进入**demo_video_002**可执行文件的目录后，执行**./demo_video_002**
3. 将会在**/mnt/sd0/**目录下生成1080p照片

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.3 示例编号 003
**描述**

1080P输入做H264编码，并保存到文件

**使用说明**

1. 将**path.json**同**demo_video_003**可执行文件放到同一目录
2. 进入**demo_video_003**可执行文件的目录后，执行**./demo_video_003 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的1080P H264文件，文件名包含当前通道名和宽高信息

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.4 示例编号 004
**描述**

720P+480P输入，经H264编码，并保存到两个文件

**使用说明**

1. 将**path.json**同**demo_video_004**可执行文件放到同一目录
2. 进入**demo_video_004**可执行文件的目录后，执行**./demo_video_004 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成两个编码后的H264文件，文件名包含当前通道名，通道号和宽高信息

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.5 示例编号 005
**描述**

YUV输入编码H264

**使用说明**

1. 将**path.json**同**demo_video_005**可执行文件放到同一目录
2. 进入**demo_video_005**可执行文件的目录后，执行**./demo_video_005 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录
3. 需要**path.json**中“data”指定YUV文件，并且YUV格式为NV12，宽高需要和**path.json**中"w":1920,"h":1080一致

### 3.2.6 示例编号 006
**描述**

ROI编码H264

**使用说明**

1. 将**path.json**同**demo_video_006**可执行文件放到同一目录
2. 进入**demo_video_006**可执行文件的目录后，执行**./demo_video_006 x**，`x`为码率控制模式参数，0（CBR）1（VBR）
3. 看到输出打印`video_set_roi enable sucess`和`video_set_roi disable sucess`一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H264文件在101~200张Frame之间左上和右下部分编码Qp值小于其他区域

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.7 示例编号 007
**描述**

H1旋转后编码H264

**使用说明**

1. 将**path.json**同**demo_video_007**可执行文件放到同一目录
2. 进入**demo_video_007**可执行文件的目录后，执行**./demo_video_007 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H264文件，其画面是旋转90度的，宽高与文件名宽高相反

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.8 示例编号 008
**描述**

H1264编码SmartP

**使用说明**

1. 将**path.json**同**demo_video_008**可执行文件放到同一目录
2. 进入**demo_video_008**可执行文件的目录后，执行**./demo_video_008 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H264文件，I帧全为长期参考帧，I之后的第一个P帧和30(refresh_interval)整数倍的P帧只有一个参考帧，其余P帧参考帧有两个

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.9 示例编号 009
**描述**

ISP输入30帧，编码自定义帧率编码

**使用说明**

1. 将**path.json**同**demo_video_009**可执行文件放到同一目录
2. 进入**demo_video_009**可执行文件的目录后，执行**./demo_video_009 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 运行过程中首先会看到周期性输出打印`-----Fps----->>x<<-----`，x值维持在30左右
4. 一段时间后会有输出打印，`video_set_fps sucess s32TargetFps:27`，此时会看到周期性输出打印`-----Fps----->>x<<-----`，x值维持在27左右
5. 继续运行会有输出打印，`video_set_fps sucess s32TargetFps:17`，此时会看到周期性输出打印`-----Fps----->>x<<-----`，x值维持在17左右
6. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.10 示例编号 010
**描述**

1080P输入做H265编码，并保存到文件

**使用说明**

1. 将**path.json**同**demo_video_010**可执行文件放到同一目录
2. 进入**demo_video_010**可执行文件的目录后，执行**./demo_video_010 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H265文件，文件名包含当前通道名和宽高信息

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.11 示例编号 011
**描述**

720P+480P输入，经H265编码，并保存到两个文件

**使用说明**

1. 将**path.json**同**demo_video_011**可执行文件放到同一目录
2. 进入**demo_video_011**可执行文件的目录后，执行**./demo_video_011 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成两个编码后的H265文件，文件名包含当前通道名，通道号和宽高信息

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.12 示例编号 012
**描述**

YUV输入编码H265

**使用说明**

1. 将**path.json**同**demo_video_012**可执行文件放到同一目录
2. 进入**demo_video_012**可执行文件的目录后，执行**./demo_video_012 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H265文件，文件名包含当前通道名和宽高信息

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录
3. 需要**path.json**中“data”指定YUV文件，并且YUV格式为NV12，宽高需要和**path.json**中"w":1920,"h":1080一致

### 3.2.13 示例编号 013
**描述**

ROI编码H265

**使用说明**

1. 将**path.json**同**demo_video_013**可执行文件放到同一目录
2. 进入**demo_video_013**可执行文件的目录后，执行**./demo_video_013 x**，`x`为码率控制模式参数，0（CBR）1（VBR）
3. 看到输出打印`video_set_roi enable sucess`和`video_set_roi disable sucess`一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H265文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H265文件在101~200张Frame之间左上和右下部分编码Qp值小于其他区域

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.14 示例编号 014
**描述**

H2旋转后编码H265

**使用说明**

1. 将**path.json**同**demo_video_014**可执行文件放到同一目录
2. 进入**demo_video_014**可执行文件的目录后，执行**./demo_video_014 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H265文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H265文件，其画面是旋转90度的，宽高与文件名宽高相反

**备注**

1. 支持`Apollo-ECO`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.15 示例编号 015
**描述**

编码时动态设置CBR  VBR  FixQp

**使用说明**

1. 将**path.json**同**demo_video_015**可执行文件放到同一目录
2. 进入**demo_video_015**可执行文件的目录后，执行**./demo_video_015 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 运行过程中可以看到输出打印`video_set_ratectrl 1 Sucess` `video_set_ratectrl 2 Sucess` `video_set_ratectrl 0 Sucess`
3. 一段时间后键盘手动输入`q`退出当前Demo程序
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H264文件，查看第101，201和301张Frame，其编码QP与前几个Frame会有较大差异

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

### 3.2.16 示例编号 016
**描述**

Trigger I功能演示

**使用说明**

1. 将**path.json**同**demo_video_016**可执行文件放到同一目录
2. 进入**demo_video_016**可执行文件的目录后，执行**./demo_video_016 x**，`x`为码率控制模式参数，0（CBR）1（VBR）2（FixQP）
3. 运行一段时间后会有输出打印`video_trigger_key_frame sucess`
4. 输入`q`退出后将会在**/mnt/sd0/**目录下生成编码后的H264文件，文件名包含当前通道名和宽高信息
5. 查看编码之后的H264文件，第101张附近Frame会是I帧，而之前都是周期性30才出现I帧

**备注**

1. 支持`Apollo-2`平台
2. 需要创建**/mnt/sd0**目录

## 3.3 Recorder
**示例列表**

* `[001]` H.264/H.265视频录制
* `[002]` 定时自动分段录制
* `[003]` 双路码流打包到同一个Container
* `[004]` 微动（Fast-motion）视频录制
* `[005]` 慢动作（视频录制）视频录制
* `[006]` UDAT设定
* `[007]` 不同Container录制`mkv`，`mov`，`mp4`
* `[008]` 动态分段录制

### 3.3.1 示例编号001
**描述**

H.264/H.265视频录制

**使用说明**

1. 将**video_enc_h2.json**和**video_enc_h1.json**同一目录
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_001 x**，`x`为编码器选择的参数，0（H264）1（H265）
3. 运行过程中可以看到类似输出打印`INFO ->lib/common.cpp:595:[0x62da0]VFr:47(0), AFr:22(0),VPTS:2858,APTS:2766,STC:36307679,VQ:0,AQ:0,FPS:15,VRate:79881Bps,A-VRate:69729Bps`
4. 一段时间后当前Demo程序会自动退出，大约有30秒
5. **/mnt/sd0**目录会生成一个MP4文件，拷贝到PC上可以正常播放

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.2 示例编号002
**描述**

定时自动分段录制

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_002 x**，`x`为录像分段时长
3. 运行过程中可以看到输出分段时长的打印`time_segment     ->30`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成多个MP4文件，除去最后一个生成的MP4文件，其它的MP4文件时长应该都为录像分段时长

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.3 示例编号003
**描述**

双路码流打包到同一个Container

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_003**
3. 运行过程中可以看到两次类似输出打印`origin stream header -> 91`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成一个MP4文件，拷贝到PC上可以正常播放

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.4 示例编号004
**描述**

微动（Fast-motion）视频录制

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_004**
3. 运行过程中可以看到类似输出打印`INFO ->lib/mediamuxer.cpp:2086:fast rate -> -2`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成一个MP4文件，拷贝到PC上播放时会有快播的效果

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.5 示例编号005
**描述**

慢动作（视频录制）视频录制

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_005**
3. 运行过程中可以看到类似输出打印`INFO ->lib/qrecorder.cpp:679:slow motion  mode`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成一个MP4文件，拷贝到PC上播放时会有慢放的效果

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.6 示例编号006
**描述**

UDAT设定

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_006**
3. 运行过程中可以看到类似输出打印`INFO ->lib/mediamuxer.cpp:2022:load udta data from memory->0xbe80ca98 0`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成一个MP4文件，二进制方式打开，在文件末尾可以找到相应的字符串

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.7 示例编号007
**描述**

不同Container录制`mkv`，`mov`，`mp4`

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_007 x**，`x`为容器选择的参数，0（MP4）1（MOV）2（MKV）
3. 运行过程中可以看到类似输出打印`av_format        ->mp4`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成相应容器的文件

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

### 3.3.8 示例编号008
**描述**

动态分段录制

**使用说明**

1. 将**video_enc_h2.json**拷贝到开放板的目录里
2. 进入**video_enc_h2.json**的目录后，执行**demo_recorder_008**
3. 运行过程中可以看到类似输出打印`INFO ->lib/mediamuxer.cpp:2096:Start new segment!`
4. 一段时间后当前Demo程序会自动退出，大约有30多秒
5. **/mnt/sd0**目录会生成3个MP4文件，前面两个MP4文件时长大约为12秒，最后一个大约为6秒

**备注**

1. 支持`Apollo`平台
2. 需要创建**/mnt/sd0**目录

## 3.4 Player
**示例列表**

* `[001]` 播放本机录制的视频
* `[002]` 同时播放多路视频
* `[003]` 播放多轨道视频时选择播放第二轨的视频

### 3.4.1 示例编号001
**描述**

播放本机录制的视频

**使用说明**

1. 将**player_1080p_g1.json**和**1080p_264_30fps_2track.mp4**同一目录
2. 进入**player_1080p_g1.json**的目录后，执行**demo_player_001**
3. 看到屏幕上显示视频画面

**备注**

1. 支持`Apollo`平台

### 3.4.2 示例编号002
**描述**

同时播放多路视频

**使用说明**

1. 将**player_1080p_g1.json**和**1080p_264_30fps_2track.mp4**同一目录
2. 进入**player_1080p_g1.json**的目录后，执行**demo_player_002**
3. 看到屏幕上显示两个视频画面都在播放

**备注**

1. 支持`Apollo`平台
2. Videobox编译时，需要把**SoftwareComposer**勾选上

### 3.4.3 示例编号003
**描述**

播放多轨道视频时选择播放第二轨的视频

**使用说明**

1. 将**player_1080p_g1.json**和**1080p_264_30fps_2track.mp4**同一目录
2. 进入**player_1080p_g1.json**的目录后，执行**demo_player_003**
3. 看到屏幕上显示视频画面

**备注**

1. 支持`Apollo`平台

## 3.5 VA
**示例列表**

* `[001]` 通过直方图做运动检测，IDS显示７x７区域运动信息
* `[002]` 通过运动向量做运动检测，IDS显示4 x 4运动位置框图
* `[003]` 人脸侦测模块，IDS显示侦测到人脸框图

### 3.5.1 示例编号 001
**描述**

通过直方图做运动检测，IDS显示７x７区域运动信息

**使用说明**

1. 将**path.json**同**demo_va_001**可执行文件放到同一目录
2. 进入**demo_va_001**可执行文件的目录后，执行**./demo_va_001**
3. 在摄像头前晃动手指，会在显示屏幕上看到手指晃动的区域有红色方框
4. 当手指停止晃动，红色的方框消失
5. 输入`ctrl+c`退出demo程序

**备注**

1. 支持`Apollo`，`Apollo-2`和`Apollo-ECO`平台

### 3.5.2 示例编号 002
**描述**

通过运动向量做运动检测，IDS显示4 x 4区域运动信息

**使用说明**

1. 将**path.json**同**demo_va_002**可执行文件放到同一目录
2. 进入**demo_va_002**可执行文件的目录后，执行**./demo_va_002**
3. 在摄像头前晃动手指，会在显示屏幕上看到手指晃动的区域有红色方框
4. 当手指停止晃动，红色的方框消失
5. 输入`ctrl+c`退出demo程序

**备注**

1. 支持`Apollo`和`Apollo-2`平台

### 3.5.3 示例编号 003
**描述**

人脸侦测模块，IDS显示侦测到人脸框图

**使用说明**

1. 将**path.json**同**demo_va_003**可执行文件放到同一目录
2. 进入**demo_va_003**可执行文件的目录后，执行**./demo_va_003**
3. 将人脸正对摄像头，会在显示屏幕上看到，人脸区域会有一个红色方框
4. 当人脸消失时，红色的方框会消失

**备注**

1. 支持`Apollo-2`平台
2. 摄像头要方向向上

## 3.6 Display
**示例列表**

* `[001]` IDS上显示视频预览的画面
* `[002]` IDS上显示OSD的ColorKey方式混合后的画面

### 3.6.1 示例编号001
**描述**

IDS使用OSD的0层(YUV数据)显示视频画面

**使用说明**

1. 将**path.json**同**demo_display_001**可执行文件放到同一目录
2. 进入**demo_display_001**可执行文件的目录后，执行**./demo_display_001**
3. LCD屏幕显示视频画面


**备注**

1. 支持`Apollo-2`和`Apollo-3`平台

### 3.6.2 示例编号002
**描述**

IDS使用OSD的0层(YUV数据)和1层(RGB数据)进行ColorKey方式混合后，显示带有边框的视频画面

**使用说明**

1. 将**path.json**同**demo_display_002**可执行文件放到同一目录
2. 进入**demo_display_002**可执行文件的目录后，执行**./demo_display_002**
3. LCD屏幕显示带有红色粗边框的视频画面

**备注**

1. 支持`Apollo-2`和`Apollo-3`平台

----
## 4 示例代码-SYS
