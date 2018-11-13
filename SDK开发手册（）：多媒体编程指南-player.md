# QSDK Player编程指南

### 适用产品

| 类别 | 适用对象 |
|---|
| 软件版本 | `QSDK-V2.2.0` |
| 芯片型号 | `Apollo` `Apollo-2` `Apollo-ECO` |

### 修订记录

| 修订说明 | 日期 | 作者 |
|---|
| 初版 | 2017/08/01 | 倪高鹏 |

### 术语解释

| 术语 | 解释 |
|---|
| QSDK | 盈方微Apollo系列芯片软件开发套件 |
| Videobox | QSDK中视频处理模块 |
| Audiobox | QSDK中音频处理模块 |
| IPU | Image Processing Unit，图像处理单元，Videobox中基本处理单元 |
| FFmpeg | FFmpeg是一个开源免费跨平台的视频和音频流处理软件，采用LGPL或GPL许可证。它提供了录制、转换以及流化音视频的完整解决方案 |

----
## 1 关于Player

Player是专门为QSDK开发的提供播放功能的库，需要与Videobox及Audiobox配合使用。Player读取本地录像文件，从容器解析出视频数据和音频数据，然后送到Videobox和Audiobox进行处理。QSDK中，应用层可以通过调用Player库中的接口实现播放功能。

Player是QSDK的可选组件，如果开发者已经有播放功能，可以在配置选项中禁用Player以达到减小镜像的目的。

Player支持的功能：

* 支持的容器： MKV，MP4，MOV
* 支持的视频解码格式： H264，H265
* 支持的音频解码格式： AAC，u-Law，a-Law，ADPCM
* 支持的图片格式： JPEG
* 支持的特殊播放方式： 多倍速快放、多倍速慢放

----
## 2 系统概述
## 2.1 系统架构

如下图所示，Player属于QSDK的中间件，应用层直接调用Player接口来实现播放功能。而Player会创建自己的线程来做音视频的相关处理，Player的线程是直接运行在应用层的进程中。

<p align='center'><img src='image/Player_1.svg' /></p>

----
## 2.2 系统流程

如下图所示，应用层直接调用Player的接口，进行播放、暂停等操作。Player会对码流解封装，抽取音频和视频数据，然后通过IPC接口将音频数据发送给Audiobox处理，视频数据发送给Videobox处理。当播放完成或发生错误时，通过Event Callback的方式通知应用层。

<p align='center'><img src='image/Player_2.svg' /></p>


----
## 2.3 配置选项

在QSDK的根目录下输入`make menuconfig`命令，就可以进到QSDK的配置界面。通过该配置界面可以配置Player、Demux和FFmpeg的相关配置。

**Player配置项**

```
QSDK Options --->
  |-Libs
  |   |-[*] qlibvplay
  |   |   |-[ ] GDB Debug
  |   |   |-[ ] Test Utils
  |   |   |-(1) Player Cache Time
  |   |   |-    Log Level  --->
  |   |   |     |-( )   TRC
  |   |   |     |-( )   DEBUG
  |   |   |     |-(X)   INFO
  |   |   |     |-( )   WARN
  |   |   |     |-( )   ERROR
```

| 配置项	| 描述 |
| --- | --- |
| **GDB Debug** | GDB调试开关。若需要GDB调试，需开启此选项 |
| **Test Utils** | Player调试工具。该工具可以测试Player各项功能，详细介绍见[6 调试工具](main.md#6_调试工具) |
| **Player Cache Time** | Player内部Audio和Video的ES Buffer的最大缓存时间，单位为秒。在码流的AV Interleave比较大时，建议提高该值，否则会出现播放卡顿。默认为1秒，如有需要可以提高到3秒 |
| **Log Level** | Debug Info等级。调试时建议设置到**INFO**，Release时建议设置到**ERROR** |

Note: Player依赖Libdemux，Videobox和FFmpeg，在使用Player前，需要确认一下Libdemux，Videobox和FFmpeg相关配置是否开启

**Demux配置项**

```
QSDK Options --->
  |-Libs
  |   |-[*] libdemux
```

| 配置项	| 描述 |
| --- | --- |
| **libdemux** | 负责解封装的函数库，它会调用FFmpeg来进行解封装，使用Player时，该选项必须开启 |


**Videobox配置项**

```
QSDK Options --->
   |-Apps
   |   |-[*] Videobox
   |   |   |   Select IPUs --->
   |   |   |       |-[*] G1264
   |   |   |       |-[*] G2
   |   |   |       |-[*] FFVdec
   |   |   |       |-[*] FFPhoto
   |   |   |       |-[*]   Enable software scale
```

| 配置项	| 描述 |
| --- | --- |
| **G1264** | H264硬件解码。若码流编码格式为H264，需开启此选项（在`Apollo`开发板上适用） |
| **G2** | H265硬件解码。若码流编码格式为H265，需开启此选项（在`Apollo`开发板上适用） |
| **FFVdec** | H264软件解码。若码流编码格式为H264且不支持硬件解码时，需开启此选项 |
| **FFPhoto** | MJPEG软件解码。若码流格式为JPEG，需开启此选项 |
| **Enable software scale** | **FFPHOTO**支持图像缩放功能。若码流格式为JPEG且需要进行缩放的情况，需开启此选项 |

**FFmpeg配置项**

```
QSDK Options --->
  |-Libs
  |   |-[*] libffmpeg
  |   |   |-[*] decoder enable
  |   |   |-[*]   libfdk-aac
  |   |   |-[*]   g711
  |   |   |-[*]   mjpeg
...
  |   |   |-[*] parser enable
  |   |   |-[*]   aac
  |   |   |-[*]   mjpeg
  |   |   |-[*]   h264
  |   |   |-[*]   hevc
...
  |   |   |-[*] demuxer enable
  |   |   |-[*]   mp4/mov
  |   |   |-[*]   mkv
  |   |   |-[*]   mjpeg
  |   |   |-[*] bsf filter enable
  |   |   |-[*]   h264_mp4toannexb
  |   |   |-[*]   hevc_mp4toannexb
  |   |   |-[*]   aac_adtstoasc
```

| 配置项	| 描述 |
| --- | --- |
| **decoder enable** - **libfdk-aac** | AAC软件解码库。播放带AAC的码流时，需开启 |
| **decoder enable** - **g711** | G711的解码库。播放带g711的码流时，需开启 |
| **decoder enable** - **mjpeg** | MJPEG的软件解码库。播放JPEG图片时，需开启 |
| **parser enable** - **aac** | AAC的解析器。播放带AAC的码流时，需开启 |
| **parser enable** - **mjpeg** | MJPEG的解析器。播放JPEG图片时，需开启 |
| **parser enable** - **h264** | H264的解析器。播放带H264的码流时，需开启 |
| **parser enable** - **hevc** | H265的解析器。播放带H265的码流时，需开启 |
| **demuxer enable** - **mp4/mov** | MP4和MOV的解封装库。播放MP4或MOV时，需开启 |
| **demuxer enable** - **mkv** | MKV的解封装库。播放MKV时，需开启 |
| **demuxer enable** - **mjpeg** | JPEG的解封装库。播放JPEG图片时，需开启 |
| **bsf filter enable** - **h264_mp4toannexb** | H264的过滤器。播放带H264的码流时，需开启 |
| **bsf filter enable** - **hevc_mp4toannexb** | H265的过滤器。播放带H265的码流时，需开启 |
| **bsf filter enable** - **aac_adtstoasc** | AAC的过滤器。播放带AAC的码流时，需开启 |

----
## 3 使用场景
## 3.1 录像播放

录像播放是Player的基本功能。应用可以调用Player的相关接口，实现播放功能。如果需要同时播放多个录像，可以新建多个Player实例来播放。

Note: 在使用Player相关功能之前，需要确保Videobox及Audiobox已经在系统中正常运行
串口输入命令`ps | grep videoboxd | grep -v 'grep'`，如果能查询到Videobox的进程号，表示Videobox正常运行
串口输入命令`ps | grep audiobox | grep -v 'grep'`，如果能查询到Audiobox的进程号，表示Audiobox正常运行

播放的流程如下图所示

<p align='center'><img src='image/Player_3.svg' /></p>

### Step 1: 注册Event

Note: 在注册Event之前，需要确保Eventhub已经在系统中正常运行。串口输入命令`ps | grep eventhub | grep -v 'grep'`，如果能查询到Eventhub的进程号，表示Eventhub正常运行。

```cpp
#include <qsdk/event.h>
#include <qsdk/vplay.h>

static vplay_event_info_t stVplayState;

void player_event_handler(char *pName, void *pArgs)
{
    if ((NULL == pName) || (NULL == pArgs))
    {
        printf("invalid parameter!\n");
        return;
    }

    if (!strcmp(pName, EVENT_VPLAY))
    {
        memcpy(&stVplayState, (vplay_event_info_t *)pArgs, sizeof(vplay_event_info_t));
        printf("vplay event type: %d, buf:%s\n", stVplayState.type, stVplayState.buf);

        switch (stVplayState.type)
        {
            case VPLAY_EVENT_PLAY_FINISH:
                printf("PLAY_STATE_END \n");
                break;
            case VPLAY_EVENT_PLAY_ERROR:
                printf("PLAY_STATE_ERROR \n");
                break;
            default:
                break;
        }
    }
}

// ...

    memset(&stVplayState, 0, sizeof(vplay_event_info_t));
    event_register_handler(EVENT_VPLAY, 0, player_event_handler);

```

以上代码注册了Event Handler到Eventhub中，通过该Handler可以接收到Player发送的Event。相关事件如下表格描述。

| 事件类型 | 描述 |
| --- | --- |
| `VPLAY_EVENT_PLAY_FINISH` | 当前录像播放结束 |
| `VPLAY_EVENT_PLAY_ERROR` | 当前录像播放出现错误 |

### Step 2: 新建实例

```cpp
void init_player_info(VPlayerInfo *pstInfo)
{
    memset(pstInfo, 0, sizeof(VPlayerInfo));

    strcat(pstInfo->video_channel, "dec0-stream");
    strcat(pstInfo->audio_channel, "default");
}

// ...

    vplay_player_info_t stPlayerInfo;
    vplay_player_inst_t *pstPlayer = NULL;

    init_player_info(&stPlayerInfo);
    pstPlayer = vplay_new_player(&stPlayerInfo);

    if (pstPlayer == NULL)
    {
        printf("New player error!\n");
        // 错误处理
    }
```

以上代码创建了一个Player实例。这个实例使用Videobox的`"dec0-stream"`来处理视频，使用Audiobox的`"default"`来处理音频。

因为Videobox输入的是编码后的视频帧数据，所以在播放前，要根据录像文件的视频编码格式在Videobox JSON里面配置相应的解码器。结合下面的Videobox JSON文件可知，播放的视频格式为H264，视频宽高为1920×1080。Videobox解码器实例名为`"dec0"`，Player的`video_channel`要写解码器实例的Port名，即`"dec0-stream"`。解码器的详细设置，可参考[3.5 解码器设置](main.md#3.5_解码器设置)。

```bash
// videobox json
{
    "dec0": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "display": "osd0" }
            }
        }
    },

	"display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```

### Step 3: 添加文件并设置轨道

```cpp
    int s32Ret = 0;
    int s32AudioIndex = 0;
    int s32VideoIndex = 0;
    char pFile[50] = "/mnt/sd0/1080p_264_30fps_2track.mp4";

    // 添加录像文件
    s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_QUEUE_FILE, pFile);
    if (s32Ret < 0)
    {
        printf("Queue file error:%d, file path:%s\n", s32Ret, pFile);
        // 错误处理
        return;
    }

    // 设置音频轨道
    s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_SET_AUDIO_FILTER, &s32AudioIndex);
    if (s32Ret < 0)
    {
        printf("Set audio index error:%d\n", s32Ret);
    }
    // 设置视频轨道
    s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_SET_VIDEO_FILTER, &s32VideoIndex);
    if (s32Ret < 0)
    {
        printf("Set video index error:%d\n", s32Ret);
    }

```

以上代码把录像文件添加到Player的播放列表中，并设置播放的视频轨道和音频轨道。若没有设置轨道，Player的Audio Track和Video Track默认设置为解析到的第一个音频和视频轨道的索引。

### Step 4: 开始播放

```cpp
    int s32PlaySpeed = 1;

    s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_PLAY, &s32PlaySpeed);
    if (s32Ret < 0)
    {
        printf("Play error:%d\n", s32Ret);
        // 错误处理
        return;
    }
```

以上代码开始录像的播放。播放正确开始后，会在终端输出以下信息。其中`VFr`的值一直增加，代表正在播放。

```bash
INFO ->lib/common.cpp:609:[0x13248]VFr:97(0), AFr:51(0),VPTS:3200,APTS:3237,STC:167063897,VQ:22,AQ:11,FPS:33
INFO ->lib/common.cpp:609:[0x13248]VFr:144(0), AFr:75(0),VPTS:4767,APTS:4735,STC:167065399,VQ:23,AQ:12,FPS:31
INFO ->lib/common.cpp:609:[0x13248]VFr:189(0), AFr:97(0),VPTS:6267,APTS:6194,STC:167066902,VQ:23,AQ:13,FPS:29
```
### Step 5: 停止播放并销毁实例

```cpp
    if (vplay_control_player(pstPlayer, VPLAY_PLAYER_STOP, NULL) < 0)
    {
        printf("Stop player error\n");
    }
    printf("stop player ok\n");
    if (vplay_delete_player(pstPlayer) < 0 )
    {
        printf("Delete player error\n");
        return ;
    }
```

当需要时，停止播放并销毁Player实例。

### Step 6: 撤销Event注册

```cpp
    event_unregister_handler(EVENT_VPLAY, player_event_handler);
    // ...
```

Player退出时，需要把当前注册的Event给撤销掉。

----
## 3.2 录像预览

Player支持录像预览功能。预览时，会将录像第一帧图像解码并显示。录像预览与[3.1 录像播放](main.md#3.1_录像播放)流程基本相同，主要差异在于Step 4中设置的控制命令不同，应设置`VPLAY_PLAYER_STEP_DISPLAY`控制命令 ，具体代码如下

```cpp
s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_STEP_DISPLAY, NULL);
if (s32Ret < 0)
{
    printf("Play error:%d\n", s32Ret);
    // 错误处理
    return;
}
```

执行以上代码，如果有如下打印出现，表示录像预览已经成功执行。

```bash
INFO ->lib/speedctrl.cpp: 51:Put one frame, size:5547
INFO ->lib/speedctrl.cpp: 70:Step Display finished
```
----
## 3.3 播放控制

Player支持静音、暂停、继续播放、跳转和获取媒体信息的操作。

**静音**

在录像播放前，可以通过`VPLAY_PLAYER_MUTE_AUDIO`的控制命令来Mute声音。在只需要播放视频的场景里，Player可以设置该控制命令。

```cpp
int s32Mute = 1;

s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_MUTE_AUDIO, &s32Mute);
if (s32Ret < 0)
{
    printf("Mute audio error:%d\n", s32Ret);
}

s32Mute = 0;
s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_MUTE_AUDIO, &s32Mute);
if (s32Ret < 0)
{
    printf("UnMute audio error:%d\n", s32Ret);
}
```

设置`VPLAY_PLAYER_MUTE_AUDIO`的控制命令，会有如下打印

```bash
// 设置mute时，会有如下打印
INFO ->lib/qplayer.cpp:1467:Set Audio mute:1
// 取消mute设置时，会有如下打印
INFO ->lib/qplayer.cpp:1467:Set Audio mute:0
```

Note: 在新建多个Player实例来同时播放多个录像时，需要设置Player为静音。这是由于Audiobox暂时未支持多音轨同时播放。

**暂停**

在录像启播后，可以通过`VPLAY_PLAYER_PAUSE`的控制命令来暂停当前的播放。设置该控制命令后，视频和音频都会暂停播放。

```cpp
s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_PAUSE, NULL);
if (s32Ret < 0)
{
    printf("Player pause error:%d\n", s32Ret);
}
```

设置`VPLAY_PLAYER_PAUSE`的控制命令，会有如下打印

```bash
INFO ->lib/qplayer.cpp:1540:vplay player control ->3
```

**继续播放**

在暂停录像播放后，可以通过`VPLAY_PLAYER_CONTINUE`的控制命令来继续当前的播放。设置该控制命令后，视频和音频都会继续播放。

```cpp
int s32PlaySpeed = 1;

s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_CONTINUE, &s32PlaySpeed);
if (s32Ret < 0)
{
    printf("Player pause error:%d\n", s32Ret);
}
```

设置`VPLAY_PLAYER_CONTINUE`的控制命令，会有如下打印

```bash
INFO ->lib/qplayer.cpp:1540:vplay player control ->4
```

**跳转**

在录像启播后，可以通过`VPLAY_PLAYER_SEEK`的控制命令来跳转并播放。`s32Time`的单位为秒。

```cpp
int s32Time = 60;

s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_SEEK, &s32Time);
if (s32Ret < 0)
{
    printf("Player pause error:%d\n", s32Ret);
}
```

设置`VPLAY_PLAYER_SEEK`的控制命令，会有如下打印

```bash
INFO ->lib/qplayer.cpp:1540:vplay player control ->5
```

**获取媒体信息**

在录像启播后，可以通过`VPLAY_PLAYER_QUERY_MEDIA_INFO`的控制命令来获取媒体信息。

```cpp
    void show_media_info(vplay_media_info_t *pstInfo)
    {
        printf("Show vplay media info\n");
        printf("\tfile        ->%s\n", pstInfo->file.file);
        printf("\ttime_length ->%8lld\n", pstInfo->time_length);
        printf("\ttime_pos    ->%8lld\n", pstInfo->time_pos);
        printf("\ttrack_number->%8d\n", pstInfo->track_number);
        printf("\tvideo_index ->%8d\n", pstInfo->video_index);
        printf("\taudio_index ->%8d\n", pstInfo->audio_index);
        for (int i=0; i < pstInfo->track_number; i++){
            printf("\t###stream_index->%d\n", pstInfo->track[i].stream_index);
            printf("\t\ttype        ->%d\n", pstInfo->track[i].type);
            printf("\t\twidth       ->%d\n", pstInfo->track[i].width);
            printf("\t\theight      ->%d\n", pstInfo->track[i].height);
            printf("\t\tfps         ->%d\n", pstInfo->track[i].fps);
            printf("\t\tsample_rate ->%d\n", pstInfo->track[i].sample_rate);
            printf("\t\tchannels    ->%d\n", pstInfo->track[i].channels);
            printf("\t\tbit_width   ->%d\n", pstInfo->track[i].bit_width);
        }
        printf("\t\n");
    }

    // ...

    vplay_media_info_t stMediaFileInfo = {0};

    memset(&stMediaFileInfo, 0, sizeof(vplay_media_info_t));
    s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_QUERY_MEDIA_INFO, &stMediaFileInfo);
    if (s32Ret < 0)
    {
        printf("Player query error:%d\n", s32Ret);
    }
    else
    {
        printf("video info result->%8lld %8lld %d\n", s32Ret,
                stMediaFileInfo.time_length,
                stMediaFileInfo.time_pos,
                stMediaFileInfo.track_number);
        show_media_info(&stMediaFileInfo);
    }

```

`vplay_media_info_t`里面各个参数含义，参考如下表格

| 媒体信息 | 描述 |
| --- | --- |
| `file` | 当前播放录像的文件名 |
| `time_length` | 当前播放录像的时长 |
| `time_pos` | 录像的播放位置 |
| `track_number` | 录像的轨道数 |
| `video_index` | 当前播放的视频轨道的索引 |
| `audio_index` | 当前播放的音频轨道的索引 |
| `type` | 当前轨道的类型， 详细信息参见[5.7 vplay_codec_type_t](main.md#5.7_vplay_codec_type_t)|
| `width` | 视频的宽度 |
| `height` | 视频的高度 |
| `fps` | 视频的帧率 |
| `sample_rate` | 音频的采样率 |
| `channels` | 音频的声道数 |
| `bit_width` | 音频采样格式，有8位、16位和32位三种格式 |

运行以上代码，会有如下打印

```bash
Show vplay media info
        file        ->1080p_264_30fps_2track.mp4
        time_length ->   31138
        time_pos    ->   31138
        track_number->       3
        video_index ->       1
        audio_index ->       2
        extra_index ->      -1
        ###stream_index->0
                type        ->0
                width       ->1920
                height      ->1080
                fps         ->29
                sample_rate ->0
                channels    ->0
                bit_width   ->0
        ###stream_index->1
                type        ->0
                width       ->320
                height      ->240
                fps         ->29
                sample_rate ->0
                channels    ->0
                bit_width   ->0
        ###stream_index->2
                type        ->8
                width       ->0
                height      ->0
                fps         ->0
                sample_rate ->16000
                channels    ->2
                bit_width   ->16

```

----
## 3.4 倍速播放

倍速播放能使录像文件多倍速快放或多倍速慢放。倍速播放与普通的录像播放的流程基本相同，主要的差异在于录像播放的Step 4设置的控制命令不一样。

**多倍速快放**

通过设置`VPLAY_PLAYER_PLAY`控制命令的`s32PlaySpeed`参数，可以调整播放的速率，正数代表多倍速快放。

```cpp
int s32PlaySpeed = 2;

s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_PLAY, &s32PlaySpeed);
if (s32Ret < 0)
{
    printf("Play error:%d\n", s32Ret);
    // 错误处理
    return;
}
```

以下是设置2倍速快放的`VPLAY_PLAYER_PLAY`控制命令后的打印。如下面打印显示，相邻两行打印`STC`（系统时间）相差1.5秒，而`VPTS`（视频的时间戳）相差3秒，这表明当前的播放正好是2倍速快放。

```bash
INFO ->lib/qplayer.cpp:1004:play with speed ->2
// ...
INFO ->lib/common.cpp:609:[0x13248]VFr:160(0), AFr:0(0),REF:11952,VPTS:11933,APTS:0,STC:7590359,VQ:25,AQ:0,FPS:27
INFO ->lib/common.cpp:609:[0x13248]VFr:202(0), AFr:0(0),REF:14996,VPTS:14933,APTS:0,STC:7591863,VQ:24,AQ:0,FPS:27
INFO ->lib/common.cpp:609:[0x13248]VFr:240(0), AFr:0(0),REF:17924,VPTS:17938,APTS:0,STC:7593366,VQ:25,AQ:0,FPS:25

```

Note: 多倍速只支持2倍速快放。当播放速度大于2倍速时，会由于性能不足而达不到相应的播放速度。

**多倍速慢放**

通过设置`VPLAY_PLAYER_PLAY`控制命令的`s32PlaySpeed`参数，可以调整播放的速率，负数代表多倍速慢放。

```cpp
int s32PlaySpeed = -2;

s32Ret = vplay_control_player(pstPlayer, VPLAY_PLAYER_PLAY, &s32PlaySpeed);
if (s32Ret < 0)
{
    printf("Play error:%d\n", s32Ret);
    // 错误处理
    return;
}
```

以下是设置2倍速慢放的`VPLAY_PLAYER_PLAY`控制命令后的打印。如下面打印显示，相邻两行打印`STC`（系统时间）相差1.5秒，`VPTS`（视频的时间戳）相差0.7秒，这表明当前的播放正好是2倍速慢放。

```bash
INFO ->lib/qplayer.cpp:1004:play with speed ->-2
// ...
INFO ->lib/common.cpp:609:[0x13248]VFr:60(0), AFr:0(0),REF:2056,VPTS:2066,APTS:0,STC:8278795,VQ:21,AQ:0,FPS:13
INFO ->lib/common.cpp:609:[0x13248]VFr:76(0), AFr:0(0),REF:2752,VPTS:2733,APTS:0,STC:8280298,VQ:25,AQ:0,FPS:10
INFO ->lib/common.cpp:609:[0x13248]VFr:97(0), AFr:0(0),REF:3417,VPTS:3433,APTS:0,STC:8281801,VQ:24,AQ:0,FPS:13
```

----
## 3.5 图片播放

Player支持JPEG图片的播放。

图片播放与[3.1 录像播放](main.md#3.1_录像播放)流程基本相同，主要的差异如下

* Step 2中设置的VPlayerInfo的参数不同，应选用`"ffphoto-stream"`作为解码器
* Step 3中不需要设置Audio和Video的轨道信息

```cpp
void init_player_info(VPlayerInfo *pstInfo)
{
    memset(pstInfo, 0, sizeof(VPlayerInfo));

    strcat(pstInfo->video_channel, "ffphoto-stream");
}

// ...

    vplay_player_info_t stPlayerInfo;
    vplay_player_inst_t *pstPlayer = NULL;

    init_player_info(&stPlayerInfo);
    pstPlayer = vplay_new_player(&stPlayerInfo);

    if (pstPlayer == NULL)
    {
        printf("New player error!\n");
        // 错误处理
    }
```

以上代码创建了一个Player实例。这个实例使用Videobox的`"ffphoto-stream"`做为图像解码，IDS作为图像输出显示。图片播放的Videobox的JSON描述如下。

```json
// videobox json
{
    "ffphoto": {
        "ipu": "ffphoto",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
    },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```
----
## 3.6 解码器设置

Player需要正确设置Videobox中解码器，才能正常播放。设置包含以下两方面。

* 不同的解码器都被封装到不同的IPU里，所以需要根据录像文件的不同视频编码格式，选择不同的IPU
* IPU`frame`的宽高，需要根据录像文件的视频宽高来设置

Videobox中解码器主要分为视频解码器和图片解码器，以下是不同解码器的对应表。

IPU与视频解码器的对应列表

| IPU名称 | 编码格式 | 解码方式 | 支持芯片 |
| --- | --- | --- | --- |
| `g1264` | H264 | 硬件解码 | `Apollo` |
| `g2` | H265 | 硬件解码 | `Apollo` |
| `ffvdec` | H264 | 软件解码 | `Apollo` `Apollo-2` `Apollo-ECO` |

IPU与图片解码器的对应列表

| IPU名称 | 编码格式 | 解码方式 | 支持芯片 |
| --- | --- | --- | --- |
| `ffphoto` | JPEG | 软件解码 | `Apollo` `Apollo-2` `Apollo-ECO` |

**示例一**

以下是H264硬件解码的JSON描述，要求录像的视频宽高为1920×1080，且编码格式为H264。

```json
// videobox json
{
    "dec0": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "display": "osd0" }
            }
        }
     },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```

**示例二**

以下是H264软件解码的JSON描述，要求录像的视频宽高为320×240，且编码格式为H264。

```json
// videobox json
{
    "dec0": {
        "ipu": "ffvdec",
        "port": {
            "frame": {
                "w": 320,
                "h": 240,
                "bind": { "display": "osd0" }
            }
        }
     },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```

Note: 软件解码建议用于小分辨率录像的播放，播放大分辨率时容易出现性能不足。

**示例三**

以下是H265硬件解码的JSON描述，要求录像的视频宽高为1920×1080，且编码格式为H265。

```json
// videobox json
{
    "dec0": {
        "ipu": "g2",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "display": "osd0" }
            }
        }
     },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```

**示例四**

以下是JPEG软件解码的JSON描述，要求图片宽高为1920×1080，且编码格式为MJPEG。

```json
// videobox json
{
    "ffphoto": {
        "ipu": "ffphoto",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
     },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 } }
}
```

Note: 图片播放没有暂停、跳转和倍速播放的功能。

----
## 4 API

### 4.1 vplay_new_player()

```cpp
VPlayer* vplay_new_player(VPlayerInfo *pstInfo);
```

创建Player实例。

**参数**

- `pstInfo` - **`IN`** Player初始化参数

**返回**

该函数成功返回Player实例的指针，失败返回NUL。

**示例**

```cpp
int main()
{
    vplay_player_info_t stInfo = {0};
    vplay_player_inst_t *pstInst = NULL;

    strcat(stInfo.audio_channel, "default");
    strcat(stInfo.video_channel, "dec0-stream");
    // 这里可以置空，后边使用VPLAY_PLAYER_QUEUE_FILE继续添加
    strcat(stInfo.media_file, "/mnt/test.mp4");
    pstInst = vplay_new_player(&stInfo);
    if (pstInst == NULL)
    {
      // 错误处理
    }
    // ...
}
```

----
### 4.2 vplay_delete_player()

```cpp
int vplay_delete_player(VPlayer *pstPlayer);
```

销毁Player实例。

**参数**

- `pstPlayer` - **`IN`** Player实例的指针

**返回**

该函数成功返回大于或等于0的整数，失败返回-1。

**示例**

```cpp
int delete(vplay_player_inst_t *pstPlayer)
{
   if (vplay_delete_player(pstPlayer) < 0)
   {
       // 错误处理
   }
   pstPlayer = NULL;
}
```

----
### 4.3 vplay_control_player()

```cpp
int64_t vplay_control_player(vplay_player_inst_t *pstPlayer, vplay_ctrl_action_t enAction, void *pPara);
```

设置多媒体文件播放控制命令。

**参数**

- `pstPlayer` - **`IN`** Player实例
- `enAction` - **`IN`** Player控制命令
- `pPara`   - **`IN`** Player控制命令对应的参数，详细信息如下

**返回**

该函数成功返回0，失败返回负数。

#### 4.3.1 VPLAY_PLAYER_PLAY

**参数**

设置Player开始播放命令。

- `pPara` - **`IN`** int型指针，表示Player播放速度的倍数，正数代表多倍速快放，负数代表多倍速慢放

**示例**

```cpp
int play(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_PLAY;
    int s32PlaySpeed = 1;

    if (vplay_control_player(pstPlayer, enAction, &s32PlaySpeed) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.2 VPLAY_PLAYER_STOP

设置Player停止播放命令。

**参数**

- `para` - 无参数

**示例**

```cpp
int stop(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_STOP;

    if (vplay_control_player(pstPlayer, enAction, NULL) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.3 VPLAY_PLAYER_PAUSE

设置Player暂停播放命令。

**参数**

- `para` - 无参数

**示例**

```cpp
int pause(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_PAUSE;

    if (vplay_control_player(pstPlayer, enAction, NULL) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.4 VPLAY_PLAYER_CONTINUE

设置Player继续播放命令。

**参数**

- `para` - **`IN`** int型指针，表示继续播放时，Player播放速度的倍数，正数代表多倍速快放，负数代表多倍速慢放

**示例**

```cpp
int continue(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_CONTINUE;
    int s32PlaySpeed = 1;

    if (vplay_control_player(pstPlayer, enAction, &s32PlaySpeed) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.5 VPLAY_PLAYER_SEEK

设置Player跳转命令

**参数**

- `para` - **`IN`** int型指针，表示跳转的播放时间，单位为秒

**示例**

```cpp
int seek(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_SEEK;
    int s32TimePosition = 60;

    if (vplay_control_player(pstPlayer, enAction, &s32TimePosition) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.6 VPLAY_PLAYER_QUEUE_FILE

设置Player添加文件的命令。

**参数**

- `para` - **`IN`** 字符串指针，表示文件路径

**示例**

```cpp
int addFileToPlayList(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_QUEUE_FILE;
    char pFile[50] = "/mnt/sd0/1080p_264_30fps_2track.mp4";

    if (vplay_control_player(pstPlayer, enAction, pFile) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.7 VPLAY_PLAYER_QUEUE_CLEAR

设置Player清空播放列表的命令。

**参数**

- `para` - 无参数

**示例**

```cpp
int clearPlayList(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_QUEUE_CLEAR;

    if (vplay_control_player(pstPlayer, enAction, NULL) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.8 VPLAY_PLAYER_REMOVE_FILE

设置Player从播放列表里删除文件的命令。

**参数**

- `para` - **`IN`** 字符串指针，表示待删除的文件路径

**示例**

```cpp
int removeFileFromPlayList(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_REMOVE_FILE;
    char pFile[50] = "/mnt/sd0/1080p_264_30fps_2track.mp4";

    if (vplay_control_player(pstPlayer, enAction, pFile) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.9 VPLAY_PLAYER_QUERY_MEDIA_INFO

设置Player获取媒体信息的命令。

**参数**

- `para` - **`OUT`** vplay_media_info_t结构体指针，用于存放媒体信息

**示例**

```cpp
int getMediaInfo(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_QUERY_MEDIA_INFO;
    vplay_media_info_t pstInfo = {0};

    if (vplay_control_player(pstPlayer, enAction, &pstInfo) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.10 VPLAY_PLAYER_SET_VIDEO_FILTER

设置Player视频轨道的命令。

**参数**

- `para` - **`IN`** int型指针，表示要播放的视频轨道

**示例**

```cpp
int setVideoTrack(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_SET_VIDEO_FILTER;
    int s32VideoTrack = 1;

    if (vplay_control_player(pstPlayer, enAction, &s32VideoTrack) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.11 VPLAY_PLAYER_SET_AUDIO_FILTER

设置Player音频轨道的命令。

**参数**

- `para` - **`IN`** int型指针，表示要播放的音频轨道

**示例**

```cpp
int setAudioTrack(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_SET_AUDIO_FILTER;
    int s32AudioTrack = 1;

    if (vplay_control_player(pstPlayer, enAction, &s32AudioTrack) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.12 VPLAY_PLAYER_STEP_DISPLAY

设置Player视频预览的命令。

**参数**

- `para` - 无参数

**示例**

```cpp
int setDisplay(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_STEP_DISPLAY;

    if (vplay_control_player(pstPlayer, enAction, NULL) < 0)
    {
        // 错误处理
    }
    // ...
}
```

#### 4.3.13 VPLAY_PLAYER_MUTE_AUDIO

设置Player静音的命令。

**参数**

- `para` - **`IN`** int型指针，为1时，表示静音，为0时，表示取消静音

**示例**

```cpp
int setAudioMute(vplay_player_inst_t *pstPlayer)
{
    vplay_ctrl_action_t enAction = VPLAY_PLAYER_MUTE_AUDIO;
    int s32Mute = 1;

    if (vplay_control_player(pstPlayer, enAction, &s32Mute) < 0)
    {
        // 错误处理
    }
    // ...
}
```
----
### 4.4 vplay_query_media_info()

```cpp
int vplay_query_media_info(char *pFileName, vplay_media_info_t *pstInfo);
```

根据文件路径，获取该文件的媒体信息。


**参数**

- `pFileName` - **`IN`** 字符串指针，表示待解析媒体信息的文件路径
- `pstInfo` - **`OUT`** vplay_media_info_t结构体指针，表示媒体信息存储的结构体

**返回**

该函数成功返回1，失败返回-1。

**示例**

```cpp
int queryFileInfo(char *pFileName)
{
    vplay_media_info_t stMediaFileInfo;

    memset(&stMediaFileInfo, 0, sizeof(vplay_media_info_t));
    if (pFileName == NULL)
    {
      printf("File name error!\n");
      return -1;
    }

    if (vplay_query_media_info(pFileName, &stMediaFileInfo) == -1)
    {
        printf("Query file:%s fail!\n", pFileName);
        // 错误处理
    }
    // ...
}
```
----
## 5 数据类型

### 5.1 VPlayer
```cpp
typedef struct {
    void *inst ;
} VPlayer;
```

| 成员名称 | 描述 |
|---|
| `inst` | Player实例的指针 |

----
### 5.2 VPlayerInfo
```cpp
typedef struct {
    char video_channel[VPLAY_CHANNEL_MAX];
    char audio_channel[VPLAY_CHANNEL_MAX];
    char media_file[VPLAY_PATH_MAX];
} VPlayerInfo;
```

| 成员名称 | 描述 |
|---|
| `audio_channel` | 音频通道，留空时，录制的文件不包含音频数据，字符串最大长度为64 |
| `video_channel` | 视频通道，字符串最大长度为64 |
| `media_file` | 待播放的多媒体文件，字符串最大长度为128 |

----
### 5.3 vplay_ctrl_action_t
```cpp
typedef enum {
    VPLAY_PLAYER_PLAY = 0x1,
    VPLAY_PLAYER_STOP = 0x2,
    VPLAY_PLAYER_PAUSE,
    VPLAY_PLAYER_CONTINUE,
    VPLAY_PLAYER_SEEK,
    VPLAY_PLAYER_QUEUE_FILE,
    VPLAY_PLAYER_QUEUE_CLEAR,
    VPLAY_PLAYER_REMOVE_FILE,
    VPLAY_PLAYER_QUERY_MEDIA_INFO,
    VPLAY_PLAYER_SET_VIDEO_FILTER,
    VPLAY_PLAYER_SET_AUDIO_FILTER,
    VPLAY_PLAYER_STEP_DISPLAY,
    VPLAY_PLAYER_MUTE_AUDIO,
} vplay_ctrl_action_t;
```

| 成员名称 | 描述 |
|---|
| `VPLAY_PLAYER_PLAY` | 该命令控制Player播放视频，参数为播放速率 |
| `VPLAY_PLAYER_STOP` | 该命令控制Player停止播放视频，无参数，并重置播放点到开始 |
| `VPLAY_PLAYER_PAUSE` | 该命令控制Player暂停播放视频 |
| `VPLAY_PLAYER_CONTINUE` | 该命令控制Player继续播放视频 |
| `VPLAY_PLAYER_SEEK` | 该命令控制Player跳转到指定位置播放　 |
| `VPLAY_PLAYER_QUEUE_FILE` | 该命令控制Player在播放队列中添加一个文件 |
| `VPLAY_PLAYER_QUEUE_CLEAR` | 该命令控制Player清空播放队列 |
| `VPLAY_PLAYER_REMOVE_FILE` | 该命令控制Player 在播放队列中删除一个文件 |
| `VPLAY_PLAYER_QUERY_MEDIA_INFO` | 该命令控制Player获取媒体信息 |
| `VPLAY_PLAYER_SET_VIDEO_FILTER` | 该命令控制Player设置视频轨道 |
| `VPLAY_PLAYER_SET_AUDIO_FILTER` | 该命令控制Player设置音频轨道 |
| `VPLAY_PLAYER_STEP_DISPLAY` | 该命令控制Player显示预览画面 |
| `VPLAY_PLAYER_MUTE_AUDIO` | 该命令控制Player静音 |

----
### 5.4 vplay_file_info_t
```cpp
typedef struct {
    char file[VPLAY_PATH_MAX];
} vplay_file_info_t;
```

| 成员名称 | 描述 |
|---|
| `file` | 待播放录像的文件路径，最大字符串长度为128 |

----
### 5.5 vplay_media_info_t
```cpp
typedef struct {
    vplay_file_info_t file;
    int64_t time_length;
    int64_t time_pos;
    int track_number;
    int video_index;
    int audio_index;
    int extra_index;
    vplay_track_info_t track[VPLAY_MAX_TRACK_NUMBER];
} vplay_media_info_t;
```

| 成员名称 | 描述 |
|---|
| `file` | 录像文件信息 |
| `time_length` | 录像时长 |
| `time_pos` | 录像当前播放位置 |
| `track_number` | 录像总轨道数 |
| `video_index` | 录像当前播放的视频轨道索引 |
| `audio_index` | 录像当前播放的音频轨道索引 |
| `extra_index` | 不支持 |
| `track` | 详细的轨道信息，最多可获取6个轨道的信息 |


----
### 5.6 vplay_track_info_t
```cpp
typedef struct {
    int stream_index;
    vplay_codec_type_t type;
    int width;
    int height;
    int fps;
    int sample_rate;
    int channels;
    int bit_width;
    int frame_num;
} vplay_track_info_t;
```

| 成员名称 | 描述 |
|---|
| `stream_index` | 轨道索引 |
| `type` | 当前轨道的类型 |
| `width` | 视频的宽度（视频轨道才有该参数） |
| `height` | 视频的高度（视频轨道才有该参数） |
| `fps` | 视频的帧率（视频轨道才有该参数） |
| `sample_rate` | 音频的采样率（音频轨道才有该参数） |
| `channels` | 音频的声道数（音频轨道才有该参数） |
| `bit_width` | Audio Sample格式，有8位、16位和32位三种格式（音频轨道才有该参数） |
| `frame_num` | 总帧数（MKV才有该参数） |

----
### 5.7 vplay_codec_type_t
```cpp
typedef enum {
    VPLAY_CODEC_VIDEO_H264,
    VPLAY_CODEC_VIDEO_H265,
    VPLAY_CODEC_VIDEO_MJPEG,
    VPLAY_CODEC_AUDIO_PCM_ALAW,
    VPLAY_CODEC_AUDIO_PCM_ULAW,
    VPLAY_CODEC_AUDIO_PCM_S8E,
    VPLAY_CODEC_AUDIO_PCM_S16E,
    VPLAY_CODEC_AUDIO_PCM_S32E,
    VPLAY_CODEC_AUDIO_AAC,
    VPLAY_CODEC_AUDIO_G726,
    VPLAY_CODEC_EXTRA_DATA
} vplay_codec_type_t;
```

| 成员名称 | 描述 |
|---|
| `VPLAY_CODEC_VIDEO_H264` | H264编码格式 |
| `VPLAY_CODEC_VIDEO_H265` | H265编码格式 |
| `VPLAY_CODEC_VIDEO_MJPEG` | MJPEG编码格式 |
| `VPLAY_CODEC_AUDIO_PCM_ALAW` | 音频a-law的PCM格式 |
| `VPLAY_CODEC_AUDIO_PCM_ULAW` | 音频u-law的PCM格式 |
| `VPLAY_CODEC_AUDIO_PCM_S8E` | 音频8bit的PCM格式 |
| `VPLAY_CODEC_AUDIO_PCM_S16E` | 音频16bit的PCM格式 |
| `VPLAY_CODEC_AUDIO_PCM_S32E` | 音频32bit的PCM格式 |
| `VPLAY_CODEC_AUDIO_AAC` | 音频AAC格式 |
| `VPLAY_CODEC_AUDIO_G726` | 音频G726格式 |
| `VPLAY_CODEC_EXTRA_DATA` | 不支持 |

----
## 6 调试工具

vplayer是Player的调试工具。可以通过该工具，来测试Player的各项功能。

如需使用，勾选Player的相应配置就可以将该工具编译到镜像里面。详细配置请参考[2.3 配置选项](main.md#2.3_配置选项)。

----
### 6.1 vplayer使用方法

vplayer通过交互式命令来运行，键入不同的命令就可以运行不同的功能。各个命令含义如下

| 命令名称 | 描述 |
| --- | --- |
| `switch x`  | 切换实例，`x`表示实例的索引，取值范围为[`0`-`3`]。同时播放多个录像时，用于多实例间的切换，只播放一个录像，不须键入该命令 |
| `new`  | 新建Player实例 |
| `delete` | 删除Player实例 |
| `photo` | 设置播放图片。播放图片时，需要键入该命令，播放视频时，不需要 |
| `vfilter x` | 设置视频轨道索引，`x`表示索引值 |
| `afilter x` | 设置音频轨道索引，`x`表示索引值|
| `queue x` | 播放列表添加文件，`x`表示文件路径 |
| `amute x` | 播放时mute audio，`x`为`1`时，表示静音，`x`为`0`时，表示取消静音 |
| `play x` | 从头开始播放，`x`表示倍率，正数表示倍速快放，负数表示倍速慢放 |
| `step` | 视频预览 |
| `pause` | 暂停视频播放 |
| `continue x` | 继续视频播放，`x`表示倍率，正数表示倍速快放，负数表示倍速慢放 |
| `seek x` | 视频跳转，`x`表示跳转的位置，单位为秒 |
| `stop` | 停止播放 |
| `exit` | 退出程序 |
| `info` | 查询媒体信息 |
| `query x` | 查询给定文件的媒体信息，`x`表示文件路径 |

----
### 6.2 常用命令

**示例一 录像播放**

录像播放命令如下，`1080p_264_30fps_2track.mp4`有两个视频轨道，如下命令选用第二个视频轨道进行播放。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 新建Player实例</span>
>new
<span class="comment"># 添加录像文件到Player播放列表</span>
>queue 1080p_264_30fps_2track.mp4
<span class="comment"># 设置视频轨道</span>
>vfilter 1
<span class="comment"># 开始播放</span>
>play 1
<span class="comment"># 暂停播放</span>
>pause
<span class="comment"># 继续播放</span>
>continue 1
<span class="comment"># 录像跳转到30秒</span>
>seek 30
<span class="comment"># 获取媒体信息</span>
>info
<span class="comment"># 停止播放</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>

**示例二 录像预览**

录像预览命令如下，键入`step`命令后，就可以看到Player会暂停在录像的第一张画面。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 新建Player实例</span>
>new
<span class="comment"># 添加录像文件到Player播放列表</span>
>queue 1080p_264_30fps_2track.mp4
<span class="comment"># 设置视频轨道</span>
>vfilter 1
<span class="comment"># 预览视频图像</span>
>step
<span class="comment"># 停止预览</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>

**示例三 两倍速快播**

两倍速快播命令如下，键入`play 2`命令以后，就可以看到视频画面快播。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 新建Player实例</span>
>new
<span class="comment"># 添加录像文件到Player播放列表</span>
>queue 1080p_264_30fps_2track.mp4
<span class="comment"># 设置视频轨道</span>
>vfilter 1
<span class="comment"># Player两倍速快播</span>
>play 2
<span class="comment"># 停止播放</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>

**示例四 两倍速慢播**

两倍速慢播命令如下，键入`play -2`命令以后，就可以看到视频画面慢播。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 新建Player实例</span>
>new
<span class="comment"># 添加录像文件到Player播放列表</span>
>queue 1080p_264_30fps_2track.mp4
<span class="comment"># 设置视频轨道</span>
>vfilter 1
<span class="comment"># Player两倍速快播</span>
>play -2
<span class="comment"># 停止播放</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>

**示例五 同时播放四个录像文件**

播放四个录像文件时，会创建多个Player实例，通过`switch`命令切换不同的Player实例。同时播放四个录像文件的命令如下。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 将索引为0的Player实例的指针切换到当前工作台</span>
>switch 0
<span class="comment"># 创建索引为0的Player实例</span>
>new
<span class="comment"># 添加录像文件到当前Player实例的播放列表</span>
>queue 1080p_264_30fps_2track.mp4
<span class="comment"># 使当前Player实例静音</span>
>amute 1
<span class="comment"># 将索引为1的Player实例的指针切换到当前工作台</span>
>switch 1
<span class="comment"># 创建索引为1的Player实例</span>
>new
<span class="comment"># 添加录像文件到当前Player实例的播放列表</span>
>queue 1080p_264_15fps_2track.mp4
<span class="comment"># 使当前Player实例静音</span>
>amute 1
<span class="comment"># 将索引为2的Player实例的指针切换到当前工作台</span>
>switch 2
<span class="comment"># 创建索引为2的Player实例</span>
>new
<span class="comment"># 添加录像文件到当前Player实例的播放列表</span>
>queue 1080p_264_15fps_1track.mkv
<span class="comment"># 使当前Player实例静音</span>
>amute 1
<span class="comment"># 将索引为3的Player实例的指针切换到当前工作台</span>
>switch 3
<span class="comment"># 创建索引为3的Player实例</span>
>new
<span class="comment"># 添加录像文件到当前Player实例的播放列表</span>
>queue 1080p_264_30fps_1track.mp4
<span class="comment"># 使当前Player实例静音</span>
>amute 1
<span class="comment"># 切换到索引为0的Player实例</span>
>switch 0
<span class="comment"># 当前Player实例开始播放</span>
>play 1
<span class="comment"># 切换到索引为1的Player实例</span>
>switch 1
<span class="comment"># 当前Player实例开始播放</span>
>play 1
<span class="comment"># 切换到索引为2的Player实例</span>
>switch 2
<span class="comment"># 当前Player实例开始播放</span>
>play 1
<span class="comment"># 切换到索引为3的Player实例</span>
>switch 3
<span class="comment"># 当前Player实例开始播放</span>
>play 1
<span class="comment"># 切换到索引为0的Player实例</span>
>switch 0
<span class="comment"># 当前Player实例停止播放</span>
>stop
<span class="comment"># 切换到索引为1的Player实例</span>
>switch 1
<span class="comment"># 当前Player实例停止播放</span>
>stop
<span class="comment"># 切换到索引为2的Player实例</span>
>switch 2
<span class="comment"># 当前Player实例停止播放</span>
>stop
<span class="comment"># 切换到索引为3的Player实例</span>
>switch 3
<span class="comment"># 当前Player实例停止播放</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>

相应的JSON文件如下所示。该JSON会将两个`g1264`解码出来的视频图像通过`swc`拼接成一个图像，然后显示到屏幕上。

```bash
{
    "dec0": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "swc00":"in0" }
            }
        }
    },
    "dec1": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "swc00":"in1" }
            }
        }
    },
    "dec2": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "swc00":"in2" }
            }
        }
    },
    "dec3": {
        "ipu": "g1264",
        "port": {
            "frame": {
                "w": 1920,
                "h": 1080,
                "bind": { "swc00":"in3" }
            }
        }
    },
    "swc00": {
        "ipu": "swc",
        "port": {
            "out": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
    },

    "display": { "ipu": "ids", "args": { "no_osd1": 1 }}
}
```

屏幕的显示效果图如下

<p align='center'><img src='image/swc.png' /></p>

**示例六 播放图片**

播放图片的命令如下，当键入`play 1`时，就可以看到图片显示。

<pre><code><span class="comment"># 执行vplayer调试工具</span>
/mnt/sd0 # ./vplayer
<span class="comment"># 设置图片播放</span>
>photo
<span class="comment"># 新建Player实例</span>
>new
<span class="comment"># 添加图片文件到Player播放列表</span>
>queue 1920_1080.jpg
<span class="comment"># 开始播放</span>
>play 1
<span class="comment"># 停止播放</span>
>stop
<span class="comment"># 退出程序</span>
>exit
</code></pre>
