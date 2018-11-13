### 术语解释
|术语|解释|
|-----------------|----------------------------------|
| Videobox | SDK中视频处理模块 |
| Audiobox | SDK中音频处理模块 |
| IPU | Image Process Unit，图像处理单元，Videobox中基本处理单元 |
| FFmpeg | FFmpeg是一个开源免费跨平台的视频和音频流处理软件，采用LGPL或GPL许可证。它提供了录制、转换以及流化音视频的完整解决方案 |

----
## 1 关于Recorder

Recorder是专门为SDK开发的提供录像功能的库，需要与Videobox及Audiobox配合使用。Recorder分别从Videobox和Audiobox获取视频和音频帧，然后封装到Container中，并保存到本地储存设备。SDK中，应用层可以通过调用Recorder库实现录像功能。

Recorder是SDK的可选组件，如果开发者已经封装音视频的流程，可以在配置选项中禁用Recorder以达到减小镜像的目的。

Recorder支持的功能：

* 支持的容器： MKV，MP4，MOV
* 支持的视频编码格式： H264，H265
* 支持的音频编码格式： AAC，u-Law，a-Law，ADPCM
* 支持的特殊录制： 缩时录影、 延时录影和两路视频录制

----
## 2 系统概述
## 2.1 系统架构

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Recorder_1.svg)

应用层直接调用Recorder接口，Recorder通过创建自己的线程而直接运行在上层应用程序进程中。

----
## 2.2 系统流程

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Recorder_2.svg)

应用层直接调用Recorder的接口，进行录制、设置参数等操作。Recorder会通过IPC接口从Videobox中获取编码后的视频数据，然后通过IPC接口从Audiobox中获取PCM数据进行编码，最后使用FFmpeg的接口将Audio和Video封装进Container中。当录制完成或发生错误时，会通过Event Callback的方式通知上层。

----
## 2.3 配置选项

**Recorder配置项**

```
QSDK Options --->
  |-Libs
  |   |-[*] qlibvplay
  |   |   |-[ ] GDB Debug
  |   |   |-[ ] Fragmented File
  |   |   |-(3) Recorder Cache Time
  |   |   |-    Log Level  --->
  |   |   |     |-( )   TRC
  |   |   |     |-( )   DEBUG
  |   |   |     |-(X)   INFO
  |   |   |     |-( )   WARN
  |   |   |     |-( )   ERROR
```

| 配置项 | 描述 |
| --- | --- |
| **GDB Debug** | 开启后可以用GDB进行调试 |
| **Fragmented File** | Fragmented的录像文件相较于一般的录像文件，头文件信息可以分段保存。录制过程中如果拔出SD卡，Fragmented的录像文件对于已录制的内容都是可以播放的，普通MP4文件会整个文件损坏。另外该选项只对MP4和MOV有效。 |
| **Recorder Cache Time** | Recorder内部Audio和Video的ES Buffer的最大缓存时间，单位为秒。在码流的AV Interleave比较大时，建议提高该值，否则会出现播放卡顿。默认为3秒，需要时可以提高到5秒。 |
| **Log Level** | Debug Info等级，调试时建议调到INFO，Release时建议调到ERROR。 |

Note: Recorder依赖Videobox和FFmpeg，在使用Recorder前，需要确认Videobox和FFmpeg相关配置是否开启。

**Videobox配置项**
```
QSDK Options --->
   |-Apps
   |   |-[*] Videobox
   |   |   |   Select IPUs --->
   |   |   |       |-[ ] H1264
   |   |   |       |-[ ] H2
   |   |   |       |-[ ] H2V4
   |   |   |       |-[*] Video Encoder
   |   |   |       |-[ ]   smatrrc
   |   |   |       |-[ ]   h1264
   |   |   |       |-      h2 encoder lib (h2v4)  --->
   |   |   |       |-          ( ) h2
   |   |   |       |-          (X) h2v4
   |   |   |       |-          ( ) NONE
```

| 配置项        | 描述 |
| --- | --- |
| **H1264** | 如果编码格式为H264，需要开启此选项，使用`Video Encoder`选中`h1264` |
| **H2V4** | 如果编码格式为H265，需要开启此选项（在`Q3-ECO`开发板上适用）使用`Video Encoder`选中`h2v4` |
| **Video Encoder** | 编码模块，默认需要开启。如果编码格式为H264，需要开启子菜单选项`h1264`，如果编码格式为H265，`Q3-ECO`开发板上需要开启子菜单选项`h2v4`，`h1264`与`h2 encode lib`为`Video Encoder`子菜单选项，`h2`，`h2v4`，`NONE`为`h2 encoder lib`子菜单选项 |

**FFmpeg配置项**
```
QSDK Options --->
  |-Libs
  |   |-[*] libffmpeg
  |   |   |-[*] encoder enable
  |   |   |-[*]   libfdk-aac
  |   |   |-[*]   g711
  |   |   |-[*]   g726
...
  |   |   |-[*] muxer enable
  |   |   |-[*]   mp4/mov
  |   |   |-[*]   mkv
...
  |   |   |-[*] bsf filter enable
  |   |   |-[*]   h264_mp4toannexb
  |   |   |-[*]   hevc_mp4toannexb
  |   |   |-[*]   aac_adtstoasc
```

| 配置项 | 描述 |
| --- | --- |
| **libfdk-aac** | 如果音频编码格式为AAC，需要开启此选项 |
| **g711** | 如果音频编码格式为G711，需要开启此选项 |
| **g726** | 如果音频编码格式为G726，需要开启此选项 |
| **mp4/mov** | 如果容器格式为MP4/MOV，需要开启此选项 |
| **h264_mp4toannexb** | 如果视频格式为H264，需要开启此选项 |
| **hevc_mp4toannexb** | 如果视频格式为H265，需要开启此选项 |
| **aac_adtstoasc** | 如果音频编码格式为AAC，需要开启此选项 |

----
## 3 使用场景
## 3.1 单轨道视频录像
单轨道视频录像是Recorder的基本功能，它可以将实时的视频内容和实时的音频内容录制到文件中，录像文件中将会包含一路视频轨道和一路音频轨道。如果需要录制多个录像文件，可以新建多个Recorder实例来录制。

Note: 在使用Recorder功能之前，需要确保Videobox及Audiobox已经在系统中正常运行。

录制过程如下图所示。

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Recorder_3.svg)

### Step 1: 注册Event

Note: 在注册Event之前，需要确保Eventhub已经在系统中正常运行。

```cpp
#include <qsdk/event.h>

static vplay_event_info_t vplayState;

void recorder_event_handler(char *name, void *args)
{
    if ((NULL == name) || (NULL == args)) {
        printf("invalid parameter!\n");
        return;
    }

    if (!strcmp(name, EVENT_VPLAY)) {
        memcpy(&vplayState, (vplay_event_info_t *)args, sizeof(vplay_event_info_t));
        printf("vplay event type: %d, buf:%s\n", vplayState.type, vplayState.buf);

        switch (vplayState.type) {
            // Recorder Event
            case VPLAY_EVENT_NEWFILE_START:
                printf("Start new file recorder!\n");
                break;
            case VPLAY_EVENT_NEWFILE_FINISH:
                vplayState.type = VPLAY_EVENT_NONE;
                printf("Add file:%s\n", (char *)vplayState.buf);
                break;
            case VPLAY_EVENT_VIDEO_ERROR:
            case VPLAY_EVENT_AUDIO_ERROR:
            case VPLAY_EVENT_EXTRA_ERROR:
            case VPLAY_EVENT_INTERNAL_ERROR:
                printf("Fatal error, need stop recorder, type = %d!\n", vplayState.type);
                break;
            case VPLAY_EVENT_DISKIO_ERROR:
                printf("Fatal error, need stop recorder, type = %d!\n", vplayState.type);
                break;
            default:
                break;
        }
    }
}

// ...

    memset(&vplayState, 0, sizeof(vplay_event_info_t));
    event_register_handler(EVENT_VPLAY, 0, recorder_event_handler);

```

以上代码，注册了Event Handler到Eventhub中，通过该Handler可以接收到Recorder发送的Event。相关事件如下表格描述。

| 事件类型 | 描述 |
| --- | --- |
| `VPLAY_EVENT_NEWFILE_START` | 开始录制一个新的文件 |
| `VPLAY_EVENT_NEWFILE_FINISH` | 结束当前文件的录制，并返回文件名 |
| `VPLAY_EVENT_VIDEO_ERROR` | 视频录制错误 |
| `VPLAY_EVENT_AUDIO_ERROR` | 音频录制错误 |
| `VPLAY_EVENT_EXTRA_ERROR` | 不使用 |
| `VPLAY_EVENT_INTERNAL_ERROR` | 内部错误 |
| `VPLAY_EVENT_DISKIO_ERROR` | IO操作出现错误 |

### Step 2: 新建实例
<pre><code>#include &lt;qsdk/vplay.h&gt;;

// ...

void init_recorder_default(VRecorderInfo *info) {
    char temp[] = "recorde_stream";

    memset(info, 0, sizeof(VRecorderInfo));
    snprintf(info->video_channel, VPLAY_CHANNEL_MAX, <strong>"enc1080p-stream"</strong>);
    snprintf(info->videoextra_channel, VPLAY_CHANNEL_MAX, "");
    snprintf(info->audio_channel, VPLAY_CHANNEL_MAX, "default_mic");

    // audio格式
    info->audio_format.type = AUDIO_CODEC_TYPE_AAC;
    info->audio_format.sample_rate = 16000;
    info->audio_format.sample_fmt = AUDIO_SAMPLE_FMT_S16;
    info->audio_format.channels = 2;

    // 视频分段时长，时间单位为秒
    info->time_segment = 180;

    // 容器类型
    strcat(info->av_format, "mkv");
    strcat(info->suffix, temp);
    // 录像文件保存路径
    strcat(info->dir_prefix, "/mnt/sd0/");
}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
</code></pre>

以上代码，创建了一个Recorder实例。这个实例使用Videobox的`"enc1080p"`做为视频源，使用Audiobox的`"default_mic"`做为音频源。采用16K双声道AAC做为音频编码格式。输出的录像文件为**/mnt/sd0/recorde_stream.mkv**。

因为Videobox输出的已经是编码后的视频帧了，所以Recorder没有设置视频编码格式的选项。结合下面的Videobox JSON文件可知，视频编码格式采用的是H.265。Videobox的编码器实例名为`"enc1080p"`，Recorder的`video_channel`要写编码器实例的Port名，即`"enc1080p-stream"`。


```json
// videobox json
{
    "isp": {
        "ipu": "v2500",
        "args": {
            "skipframe": 1,
            "nbuffers": 3
        },
        "port": {
            "out": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "display": "osd0", "enc1080p": "frame" }
            }
        }
    },
    "enc1080p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "display": { "ipu": "ids", "args": { "no_osd1": 1 }}
}
```

### Step 3: 开始录像
```cpp
    if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
        printf("start recorder error\n");
    }
    // 进行其它操作
```

录像正确开始后，会在终端输出以下信息。其中`VFr`的值一直增加，代表正在录制。

```bash
INFO ->lib/common.cpp:595:[0x84950]VFr:43(0), AFr:0(0),VPTS:1396,APTS:0,STC:105861149,VQ:0,AQ:0,FPS:28,VRate:84454Bps,A-VRate:86563Bps
INFO ->lib/common.cpp:595:[0x84950]VFr:88(0), AFr:0(0),VPTS:2892,APTS:0,STC:105862650,VQ:0,AQ:0,FPS:29,VRate:77116Bps,A-VRate:81618Bps
INFO ->lib/common.cpp:595:[0x84950]VFr:133(0), AFr:0(0),VPTS:4388,APTS:0,STC:105864153,VQ:0,AQ:0,FPS:29,VRate:77481Bps,A-VRate:80197Bps

```
### Step 4: 停止录像

```cpp
    if (vplay_control_recorder(recorder, VPLAY_RECORDER_STOP, NULL) < 0) {
        printf("stop recorder error\n");
    }
    printf("stop recorder ok\n");
    if (vplay_delete_recorder(recorder) < 0) {
        printf("delete recorder error\n");
        return;
    }
```

当需要时，停止录像并销毁Recorder实例。

### Step 5: 撤销Event注册

```cpp
    event_unregister_handler(EVENT_VPLAY, recorder_event_handler);
    // ...
```
Recorder退出时，需要把当前注册的Event给撤销掉。

----

## 3.2 双轨道视频录像

Recorder支持双轨道视频录像，它会将两路视频和一路音频打包进同一个录像文件。两路视频的优点是：在没有硬件解码的时候，可以用低分辨率视频进行流畅播放；在有硬件解码或CPU比较强的时候，用大分辨率视频进行播放。

在行车记录仪产品中，使用双轨道视频录像时，会将1080p视频和320×240分辨率的视频同时录制到一个录像文件中。本地回放时，选择320×240的视频来进行播放。因为这两颗芯片没有硬件解码，选择小分辨率视频轨道来回放会比较流畅。同时，PC回放录像文件时，不会看到320×240的小分辨率视频。

```cpp
#include <qsdk/vplay.h>

// ...

void init_recorder_default(VRecorderInfo *info) {

    // ...
    // 设置第一路视频
    snprintf(info->video_channel, VPLAY_CHANNEL_MAX, "enc1080p-stream" );
    // 设置第二路视频
    snprintf(info->videoextra_channel, VPLAY_CHANNEL_MAX, "encss0-stream" );
    // ...
    // 其余代码同3.1

}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
    if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
        printf("start recorder error\n");
    }
    // ...
    // 其余代码同3.1
```

```json
// videobox json
{
    "isp": {
        "ipu": "v2500",
        "args": {
            "skipframe": 1,
            "nbuffers": 3
        },
        "port": {
            "out": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "ispost0": "in" }
            }
        }
    },

    "ispost0": {
        "ipu": "ispost",
        "args": {
            "dn_enable":1,
            "dn_target_index":0,
            "lc_grid_file_name1":"/root/.ispost/lc_v1_hermite32_1920x1088_scup_0~30.bin"
        },
        "port": {
            "ss0": {
                "w": 320,
                "h": 240,
                "pixel_format": "nv12",
                "bind": { "encss0": "frame" }
            },
            "dn": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "enc1080p": "frame", "display": "osd0" }
            }
        }
    },
    "enc1080p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "encss0": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "display": { "ipu": "ids", "args": { "no_osd1": 1 }}
}
```

调用`vplay_new_recorder()`时，会有如下打印，检查`videoextra_channel`的打印就可以确认第二路视频通道是否正确设置。
<pre><code>show recorder info start ->
         <strong>video_channel         ->enc1080p-stream<-</strong>
         <strong>videoextra_channel    ->encss0-stream<-</strong>
         audio_channel    ->default_mic<-
         enable gps       ->0<-
         keep temp        ->0<-
         audio type       ->3<-
         audio sample_rate->16000<-
         audio sample_fmt ->1<-
         audio channels   ->2<-
         audio effect     ->0<-
         time_segment     ->180<-
         time_backward    ->0<-
         time_format      ->
         suffix           ->recorde_stream
         av_format        ->mkv
         dir_prefix       ->/mnt/sd0/
</code></pre>

另外可以从Log里面看到，会有两次Dump Header的打印，这表明在录制两路视频。

```bash
origin stream header -> 91

          0  0  0  1 40  1  C  1 FF FF
          1  0  0  3  0  0 80  0  0  3
          0  0  3  0 B4 97  2 40  0  0
          0  1 42  1  1  1  0  0  3  0
          0 80  0  0  3  0  0  3  0 B4
         A0  3 C0 80 10 E5 96 5E 4B 2B
         64 B9 9B  2  0  0  3  0  2  0
          0  3  0 3C 7F 16 25  0  0  0
          1 44  1 C0 90 80 80 F0 30 51
         49
        header over

origin stream header -> 91

          0  0  0  1 40  1  C  1 FF FF
          1  0  0  3  0  0 80  0  0  3
          0  0  3  0 B4 97  2 40  0  0
          0  1 42  1  1  1  0  0  3  0
          0 80  0  0  3  0  0  3  0 B4
         A0  3 C0 80 10 E5 96 5E 4B 2B
         64 B9 9B  2  0  0  3  0  2  0
          0  3  0 3C 7F 16 25  0  0  0
          1 44  1 C0 90 80 80 F0 30 51
         49
        header over
```

----
## 3.3 录制参数设置

录制参数的设置包括设定录像分段、设定录像文件名命名规则、设定容器、设定音频编码参数和设置用户数据信息。

----

### 3.3.1 设定录像分段时长

Recorder支持设定录像分段时间。并且在录像时，必须设置录像分段时间。设定录像分段时长只需在创建Recorder实例时，设置相应的参数到`time_segment`即可。另外为了分段的录像文件名不重复，需要设定录像文件名命名规则（对应`info`中`suffix`和`time_format`参数），该规则的设置在[**3.3.2**](main.md#3.3.2_设定录像文件名命名规则)中详细介绍。


```cpp
#include <qsdk/vplay.h>

void init_recorder_default(VRecorderInfo *info) {

    // ...
    char temp[] = "%ts_%l_F";

    // 录像分段时长，单位为秒
    info->time_segment = 180;
    // 文件名命名规则
    strcat(info->suffix, temp);
    // 时间格式
    strcpy(info->time_format, "%Y_%m%d_%H%M%S___");
    // 其余代码同3.1

}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
    // ...
    // 其余代码同3.1
```

调用`vplay_new_recorder()`时，会有如下打印，检查`time_segment`的打印就可以确认视频分段时长设置是否准确。另外也可以等待三分钟后（设置为180秒的话），将生成录像文件放到PC播放来确认影片时长。
<pre><code>show recorder info start ->
         video_channel         ->enc1080p-stream<-
         videoextra_channel    -><-
         audio_channel    ->default_mic<-
         enable gps       ->0<-
         keep temp        ->0<-
         audio type       ->3<-
         audio sample_rate->16000<-
         audio sample_fmt ->1<-
         audio channels   ->2<-
         audio effect     ->0<-
         <strong>time_segment     ->180<-</strong>
         time_backward    ->0<-
         time_format      ->%Y_%m%d_%H%M%S___
         suffix           ->%ts_%l_F
         av_format        ->mkv
         dir_prefix       ->/mnt/sd0/
</code></pre>

----

### 3.3.2 设定录像文件名命名规则

Recorder在长时间录像时，为了避免新生成的文件覆盖以前录制的文件，需要设置录像文件名命名规则来保证文件名不会重复。文件名命名规则包括两个部分，时间格式设置（对应于`VRecorderInfo`的`time_format`参数）和文件前缀设置（对应于`VRecorderInfo`的`suffix`参数）。`time_format`和`suffix`都是通过通配符来设定文件名命名规则。另外文件名的后缀跟设置的容器有关系，所以不需要特别设置。

**`suffix`相关通配符定义**

| 参数	| 描述 |
| --- | --- |
| `%ts` | start time (local time zone ) |
| `%te` | end time (local time zone ) |
| `%us` | start time(utc time) |
| `%ue` | stop time (utc time) |
| `%c`  | recorded video count, start from 1 |
| `%l` | media duration |

Note: `%ts`，`%te`，`%us`和`%ue`会根据`time_format`的格式显示时间

**示例一**

```
void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    // 文件名命名规则
    strcat(info->suffix, "recorder_%c_%l");
    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
}
```
`%c`表示录制的文件个数，`%l`表示影片时长，则依次生成的文件名为**recorder_0_0180.mkv**，**recorder_1_0180.mkv**，**recorder_2_0180.mkv**...

**示例二**

```
void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    // 文件名命名规则
    strcat(info->suffix, "%ts_%l_F");
    // 时间格式
    strcpy(info->time_format, "%Y_%m%d_%H%M%S___");

    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
}
```
`%ts`代表录制的起始时间，`%l`表示影片时长，则依次生成的文件名为**2007_0620_200530____0180_F.mkv**，**2007_0620_200830____0180_F.mkv**，**2007_0620_201130____0180_F.mkv**...

`%ts`的显示规则由`time_format`来设置，`time_format`会在后面具体讲解。

**示例三**

```
void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    // 文件名命名规则
    strcat(info->suffix, "%us_%l_F");
    // 时间格式
    strcpy(info->time_format, "%Y_%m%d_%H%M%S___");

    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
}
```
`%us`表示录制起始的UTC的时间，`%l`表示影片时长，则依次生成的文件名为**2007_0620_120830____0180_F.mkv**，**2007_0620_121130____0180_F.mkv**，**2007_0620_121430____0180_F.mkv**...

`%us`表示UTC的时间，如果`%ts`表示北京时间的话，正常`%us`会与`%us`相差8小时（前提是系统正确设置日期时间）。`%us`的显示规则，由`time_format`决定，`time_format`会在后面具体讲解。

**`time_format`通配符定义**

| 参数	| 描述 |
| --- | --- |
| `%Y`  | years(four numbers), likes 2017 |
| `%H`  | hour |
| `%M`  | minute |
| `%S`  | second |
| `%y`  | years(two numbers), if it is 2017, will set 17 |
| `%m`  | month |
| `%d`  | day |
| `%s`  | total seconds, from 19700101|

**示例一**

```
void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    // 文件名命名规则
    strcat(info->suffix, "%us_%l_F");
    // 时间格式
    strcpy(info->time_format, "%y_%m%d_%H%M%S___");

    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
}
```
`%y`只显示年份的后两位，则依次生成的文件名为**07_0620_120830____0180_F.mkv**，**07_0620_121130____0180_F.mkv**，**07_0620_121430____0180_F.mkv**...

**示例二**

```
void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    // 文件名命名规则
    strcat(info->suffix, "%us_%l_F");
    // 时间格式
    strcpy(info->time_format, "%s___");

    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
}
```
`%s`表示1970/01/01到现在的秒数，则依次生成的文件名为**1483258655____0180_F.mp4**，**1483258835____0180_F.mp4**，**1483259015____0180_F.mp4**...

----

### 3.3.3 设定容器

Recorder通过`av_format`设置容器的类型，可以设置的字符串为`“mov”`，`“mp4”`和`“mkv”`。如果是编码格式为H264，一般选择MP4和MOV容器；如果编码格式为H265，一般选择MKV容器。

各个容器对比如下：

| 容器 | 版权 | 浏览器支持 | 支持Fragment | 支持编码格式 |
| --- | --- | --- | --- | --- |
| MKV | 开放标准，无版权问题　| 安卓和iOS等移动设备上不是默认支持的 | 不支持 | H264&H265 |
| MP4 | ISO/IEC制定的标准，无版权问题　| 大部分移动设备都支持 | 支持 | H264&H265 |
| MOV | Apple制定的标准，无版权问题　| 大部分移动设备都支持 | 支持 | H264&H265 |

```cpp
#include <qsdk/vplay.h>

// ...

void init_recorder_default(VRecorderInfo *info) {
    // ...
    // 容器类型
    strcat(info->av_format, "mkv");
    // ...
    // 其余代码同3.1
}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
    // ...
    // 其余代码同3.1
```

调用`vplay_new_recorder()`时，会有如下打印，检查`av_format`的打印就可以确认时间分片设置是否准确。另外也可以等待几分钟后，确认生成的文件名是否符合。
<pre><code>show recorder info start ->
         video_channel         ->enc1080p-stream<-
         videoextra_channel    -><-
         audio_channel    ->default_mic<-
         enable gps       ->0<-
         keep temp        ->0<-
         audio type       ->3<-
         audio sample_rate->16000<-
         audio sample_fmt ->1<-
         audio channels   ->2<-
         audio effect     ->0<-
         time_segment     ->180<-
         time_backward    ->0<-
         time_format      ->
         suffix           ->recorde_stream
         <strong>av_format        ->mkv</strong>
         dir_prefix       ->/mnt/sd0/
</code></pre>

----

### 3.3.4 设定音频编码参数

Recorder通过`audio_format`来设置Audio的相关参数，包括音频编码格式，采样率，采样格式以及声道数。
支持的audio格式包括：

| 音频编码格式 | 容器 | 采样率 | 采样格式 | 声道数 |
| --- | --- | --- | --- | --- |
| G711 a-law | MKV | 8k, 16k | 16bit | 2, 1 |
| G726 | MKV, MP4 & MOV | 16k | 16bit | 2, 1 |
| AAC | MKV, MP4 & MOV | 8k, 16k, 32k | 16bit | 2, 1 |


```cpp
#include <qsdk/vplay.h>

// ...

void init_recorder_default(VRecorderInfo *info) {
    // ...
    // audio格式
    info->audio_format.type = AUDIO_CODEC_TYPE_AAC;
    info->audio_format.sample_rate = 16000;
    info->audio_format.sample_fmt = AUDIO_SAMPLE_FMT_S16;
    info->audio_format.channels = 2;
    // ...
    // 其余代码同3.1
}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
    // 其余代码同3.1
```
`audio_format.type`代表Audio编码格式。详见[audio_codec_type_t](main.md##5.5_audio_codec_type_t)。

调用`vplay_new_recorder()`时，会有如下打印，检查`audio type`、`audio sample_rate`、`audio sample_fmt`和`audio channels`的打印就可以确认Audio设置是否正确。
<pre><code>show recorder info start ->
         video_channel         ->enc1080p-stream<-
         videoextra_channel    -><-
         audio_channel    ->default_mic<-
         enable gps       ->0<-
         keep temp        ->0<-
         <strong>audio type       ->3<-</strong>
         <strong>audio sample_rate->16000<-</strong>
         <strong>audio sample_fmt ->1<-</strong>
         <strong>audio channels   ->2<-</strong>
         audio effect     ->0<-
         time_segment     ->180<-
         time_backward    ->0<-
         time_format      ->
         suffix           ->recorder_stream
         av_format        ->mkv
         dir_prefix       ->/mnt/sd0/
</code></pre>

----

### 3.3.5 设置用户数据信息

Recorder可以通过`vplay_control_recorder()`接口，设置用户数据信息到码流里面（只对MP4起作用）。通过该接口可以设置相机标定信息到码流中。设置的数据长度应不大于1MB。设定用户数据有两种方式：设置文件路径和设置字符串。

**设置文件路径**

```cpp
#include <qsdk/vplay.h>

// ...

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);

if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

vplay_udta_info_t udta;
char *p = "/mnt/sd0/user.data";
int len = strlen(p);

if (access(p, R_OK) == 0) {
    // 该流程是会将文件（/mnt/sd0/user.data）里面的内容写到码流里面，这里的size需设置为0
    printf("udta file->%s<-\n", p);
    udta.buf = p;
    udta.size = 0;
}

// 设置用户数据信息
vplay_control_recorder(recorder, VPLAY_RECORDER_SET_UDTA, &udta);

if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
// 其余代码同3.1
```

Note: 设置文件路径时，需要把Size设置为0。

**设置字符串**

```cpp
#include <qsdk/vplay.h>

// ...

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}
vplay_udta_info_t udta;
char *p = "Infotm set user data";
int len = strlen(p);

// 该流程是将字符串"Infotm set user data"写到码流里面，这里的size需设置为字符串长度
printf("udta data->%s<-\n", p);
udta.buf = p;
udta.size = len;

// 设置用户数据信息
vplay_control_recorder(recorder, VPLAY_RECORDER_SET_UDTA, &udta);

if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
// 其余代码同3.1
```

Note: 设置字符串时，需要把Size设置为字符串长度。

用二进制编辑打开，可以在文件末尾搜索到相应的二进制字符串。另外，想取到用户数据信息，解析`meta`标签里面的`user`标签即可。

<img src="https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Recorder_4.png" width="80%"/>

----
## 3.4 动态控制

**Mute操作**

Recorder可以通过设置`VPLAY_RECORDER_MUTE`控制命令，来使声音静音。在有些只需要录制视频的场景，可以使用该接口。

```cpp
if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
sleep(10);
// mute audio
if (vplay_control_recorder(recorder, VPLAY_RECORDER_MUTE, NULL) < 0) {
    printf("set recorder mute error\n");
}
sleep(10);
// unmute audio
if (vplay_control_recorder(recorder, VPLAY_RECORDER_UNMUTE, NULL) < 0) {
    printf("set recorder mute error\n");
}
sleep(10);
if (vplay_control_recorder(recorder, VPLAY_RECORDER_STOP, NULL) < 0) {
    printf("stop recorder error\n");
}
// ...
// 其余代码同3.1
```
mute audio 时会有如下打印：

```bash
INFO ->lib/apicalleraudio.cpp:432:mute audio ->1
```
unmute audio时会有如下打印：

```bash
INFO ->lib/apicalleraudio.cpp:432:mute audio ->0
```

按以上代码操作录制出来的影片，它的0-10秒的播放是有声音的，10-20秒播放是没有声音的，20-30秒播放是有声音的。

**Start/Stop操作**

Recorder通过设置`VPLAY_RECORDER_START`控制命令可以开始录制，通过设置`VPLAY_RECORDER_STOP`控制命令可以结束录制。

```cpp
init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
sleep(20);
if (vplay_control_recorder(recorder, VPLAY_RECORDER_STOP, NULL) < 0) {
    printf("stop recorder error\n");
}
// 其余代码同3.1
```

Recorder设置`VPLAY_RECORDER_START`控制命令时，会有如下打印

```bash
INFO ->lib/qrecorder.cpp:294:start QRecorder
INFO ->lib/qrecorder.cpp:332:start muxer api thread
INFO ->lib/qrecorder.cpp:672:recorder start finish ->2
```

Recorder设置`VPLAY_RECORDER_STOP`控制命令时，会有如下打印

```bash
INFO ->lib/qrecorder.cpp:289:SetWorkingState ->3
INFO ->lib/qrecorder.cpp:376:pause QRecorder
...
INFO ->lib/qrecorder.cpp:677:recorder stop finish ->3
```

----
## 3.5 微速摄影（Fast-Motion）

微速摄影，就是让摄影机按比标准速度慢得多的时间间隔连续拍摄，以求在较短的时间内表达摄影对象在相当长时间内的变化过程。Recorder可以通过控制命令来设置微速摄影的倍速。

`VPLAY_RECORDER_FAST_MOTION`和`VPLAY_RECORDER_FAST_MOTION_EX`都可以做到微速摄影。

| 微速摄影模式 | 实现方式 | 优点 | 缺点 | 码流帧率 |
| --- | --- | --- | --- | --- |
| `VPLAY_RECORDER_FAST_MOTION_EX` | 只保存从Videobox拿到的I帧 | 可以与其它的Videobox的Video Channel共用，比如P2P的Video Channel | 倍速只能为I帧间隔整数倍，并且码率比较大 | 与Videobox的Video Channel一样 |
| `VPLAY_RECORDER_FAST_MOTION` | Videobox内部采取丢帧策略 | 编码出的码流与正常码流没有差异 | 需要独自占用一路编码 | 与Videobox的Video Channel一样 |


<pre><code>VRecorder *recorder = NULL;
VRecorderInfo recorderInfo;

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

int rate = 10;
if (vplay_control_recorder(recorder, <strong>VPLAY_RECORDER_FAST_MOTION</strong>, &rate) < 0) {
    printf("set recorder mute error\n");
}
if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
sleep(30);
// 其余代码同3.1
</code></pre>
<pre><code>VRecorder *recorder = NULL;
VRecorderInfo recorderInfo;

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

int IFrameInterval = video_get_fps(recorderInfo.video_channel);
int rate = 10 * IFrameInterval;
if (vplay_control_recorder(recorder, <strong>VPLAY_RECORDER_FAST_MOTION_EX</strong>, &rate) < 0) {
    printf("set recorder mute error\n");
}
if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
// 其余代码同3.1
</code></pre>

设置`VPLAY_RECORDER_FAST_MOTION`或`VPLAY_RECORDER_FAST_MOTION_EX`控制命令时，会有如下打印，其中-10代表10倍的慢速录影。

```
INFO ->lib/mediamuxer.cpp:2020:fast rate -> -10
```
设置`VPLAY_RECORDER_FAST_MOTION_EX`控制命令时，会额外有如下打印
```
INFO ->lib/qrecorder.cpp:707:fast motion mode by player ->10
```

----
## 3.6 高速摄影（Slow-Motion）

高速摄影是一种使用非常快的快门速度来捕捉图像的技术，最常用于那些通常情况下人眼无法看到的场景的拍摄。SDK中，高速摄影不会改变Sensor采集的实际帧率，而是通过修改录像时间截达到慢速播放的目的。Recorder可以通过控制命令来设置高速摄影的倍速。

```cpp
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

int rate = 2;
if (vplay_control_recorder(recorder, VPLAY_RECORDER_SLOW_MOTION, &rate) < 0) {
    printf("set recorder mute error\n");
}
if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
    printf("start recorder error\n");
}
// 其余代码同3.1
```

设置`VPLAY_RECORDER_SLOW_MOTION`控制命令时，会有如下打印，其中2代表倍速
```
INFO ->lib/qrecorder.cpp:667:slow motion mode
slow rate -> 2
```
也可以播放录制的视频，来确认是否有成功设置。如果播放看到慢放的效果，就是设置已经生效。

----
## 3.7 预录像

Recorder支持预录像功能，开启预录像只需要在创建Recorder实例时，将预录像时长设置到`time_backward`变量中即可。预录像通过在Recorder Buffer里缓存相应时长的Audio与Video数据，使得在启动录像时，能够录制启动录像之前一小段时间的音视频。

预录像一般用在开启移动侦测的场景。例如设置预录像时长为30s，那么在移动侦测触发录像时，会把触发之前的30s也录下来。下面介绍一个预录像和移动侦测相结合的例子。

### Step 1: 注册事件
```cpp
#include <qsdk/vplay.h>

static vplay_event_info_t vplayState;
static time_t stopRecorder;
static int detectMv = 0;

void recorder_event_handler(char *name, void *args)
{
    if ((NULL == name) || (NULL == args)) {
        printf("invalid parameter!\n");
        return;
    }

    if (!strcmp(name, EVENT_VPLAY)) {
        // 代码同3.1
    }
    else if (!strcmp(name, EVENT_VAMOVE)) {
        time(&stopRecorder);
        detectMv = 1;
        printf("Detect movement, start recorder %lld !\n", stopRecorder);
    }
}

// ...

    event_register_handler(EVENT_VPLAY, 0, recorder_event_handler);
    event_register_handler(EVENT_VAMOVE, 0, recorder_event_handler);

```

注册移动侦测相关的事件，用来监控是否有物体移动。当有物体移动时，就会有CallBack上来，上层记录出现移动物体的时间。该时间会在后续判断是否停止录像时用到。

### Step 2: 设定预录像时间

```cpp
#include <qsdk/vplay.h>

// ...

void init_recorder_default(VRecorderInfo *info) {

    // ...
    info->time_backward = 30;// 预录像时长，单位为秒
    // 其余代码同3.1

}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
    sleep(30);// 等待Recorder Buffer缓存足够的数据
```

预录像是从创建Recorder后才开始缓存数据，缓存中会保存最新的30秒的音视频数据（如果预录像时长设置为30秒的话）。
调用`vplay_new_recorder()`时，会有如下打印，检查`time_backward`的打印就可以确认定预录像时长设置是否准确。
<pre><code>show recorder info start ->
         video_channel         ->enc1080p-stream<-
         videoextra_channel    -><-
         audio_channel    ->default_mic<-
         enable gps       ->0<-
         keep temp        ->0<-
         audio type       ->3<-
         audio sample_rate->16000<-
         audio sample_fmt ->1<-
         audio channels   ->2<-
         audio effect     ->0<-
         time_segment     ->180<-
         <strong>time_backward    ->30<-</strong>
         time_format      ->
         suffix           ->recorde_stream
         av_format        ->mkv
         dir_prefix       ->/mnt/sd0/
</code></pre>

### Step 3: 检测物体移动

```cpp
    time_t now;
    int beginRecorder = 0;

    while(1) {
        if (detectMv)
        {
            // 检测到物体移动，开始录像
            if (beginRecorder == 0)
            {
                if (vplay_control_recorder(recorder, VPLAY_RECORDER_START, NULL) < 0) {
                    printf("start recorder error ->%d\n", counter);
                }
                beginRecorder = 1;
            }
            detectMv = 0;
            while (detectMv == 0)
            {
                time(&now);
                // 等待30秒，看是否还有移动物体出现
                if (now - stopRecorder > 30)
                {
                    // 30秒内，没有移动物体出现，就关闭录像
                    if (beginRecorder == 1)
                    {
                        if (vplay_control_recorder(recorder, VPLAY_RECORDER_STOP, NULL) < 0) {
                            printf("stop recorder error ->%d\n", counter);
                        }
                        beginRecorder= 0;
                    }
                    break;
                }
                sleep(1);
            }
        }
        // 重新监视是否出现移动物体
        sleep(1);
    }

```

当检测到有物体移动时（`detectMv`被设置为`1`），就会开始文件的录制。观看录制的影片，可以看到在物体移动之前的30秒音视频内容。检测到物体移动后，如果30秒内都没有出现物体移动，就停止当前录制

另外移动侦测分`EVENT_VAMOVE`和`EVENT_MVMOVE`，主要是检测移动物体算法有差异。

```cpp
// Videobox的JSON
{
    "isp": {
        "ipu": "v2500",
        "args": {
            "skipframe": 1,
            "nbuffers": 3
        },
        "port": {
            "out": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "display": "osd0", "enc1080p":"frame" }
            },
            "his": {
                "bind": { "vam": "in" }
            }
        }
    },
    "enc1080p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h264"
        }
    },
    "display": { "ipu": "ids", "args": { "no_osd1": 1 }},
    "vam": { "ipu": "vamovement"}
}
```

----
## 4 API
### 4.1 vplay_new_recorder()
```cpp
VRecorder *vplay_new_recorder(VRecorderInfo *info);
```
新建Recorder实例。在进行录像操作时，要获取该实例，通过该实例来进行Recorder的相关操作。

**参数**

- `info` - Recorder初始化参数

**返回**

成功返回非NULL指针，失败则返回NULL

**示例**

```cpp
#include <qsdk/vplay.h>

// ...

void init_recorder_default(VRecorderInfo *info) {

    memset(info, 0, sizeof(VRecorderInfo));

    char temp[] = "%ts_%l_F";
    snprintf(info->video_channel, VPLAY_CHANNEL_MAX, "enc1080p-stream");
    snprintf(info->videoextra_channel, VPLAY_CHANNEL_MAX, "");
    snprintf(info->audio_channel, VPLAY_CHANNEL_MAX, "default_mic");

    // audio格式
    info->audio_format.type = AUDIO_CODEC_TYPE_AAC;
    info->audio_format.sample_rate = 16000;
    info->audio_format.sample_fmt = AUDIO_SAMPLE_FMT_S16;
    info->audio_format.channels = 2;

    // 视频分段时长
    info->time_segment = 180;
    // 缓存时长
    info->time_backward = 0;

    // 时间格式
    strcpy(info->time_format, "%Y_%m%d_%H%M%S___");
    // 容器类型
    strcat(info->av_format, "mkv");
    // 文件名命名规则
    strcat(info->suffix, temp);
    // 录像文件保存路径
    strcat(info->dir_prefix, "/mnt/sd0/");

}

// ...

    VRecorder *recorder = NULL;
    VRecorderInfo recorderInfo;

    init_recorder_default(&recorderInfo);
    recorder = vplay_new_recorder(&recorderInfo);
    if (recorder == NULL) {
        printf("new recorder error\n");
        return;
    }
```

----
### 4.2 vplay_delete_recorder()
```cpp
int vplay_delete_recorder(VRecorder *rec);
```
销毁Recorder实例。在录制结束后，需要销毁该实例。

**参数**

- `rec` - Recorder实例

**返回**

成功返回1，失败返回-1

**示例**

```cpp
#include <qsdk/vplay.h>

// ...

VRecorder *recorder = NULL;
VRecorderInfo recorderInfo;

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}
// ...
if (vplay_delete_recorder(recorder) < 0) {
    printf("delete recorder error\n");
    return;
}
```

----
### 4.3 vplay_control_recorder()
```cpp
int vplay_control_recorder(vplay_recorder_inst_t *rec, vplay_ctrl_action_t type, void *para);
```
设置多媒体文件录制的控制命令。

**参数**

- `rec` - Recorder实例
- `type` - 控制行为
- `para` - 不同控制行为对应所需的参数，详细信息如下

**返回**

成功返回大于或等于0的值，失败返回小于0的值

#### 4.3.1 VPLAY_RECORDER_START

Recorder开始录像。

**参数**

- `para` - 无参数

**示例**

```cpp
int recorde(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_START;

    if (vplay_control_recorder(rec, action, NULL) < 0) {
        // 错误处理
    }
}
```

#### 4.3.2 VPLAY_RECORDER_STOP

Recorder停止录像。

**参数**

- `para` - 无参数

**示例**

```cpp
int stop(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_STOP;

    if (vplay_control_recorder(rec, action, NULL) < 0) {
        // 错误处理
    }
}
```

#### 4.3.3 VPLAY_RECORDER_MUTE

Recorder使录像静音。

**参数**

- `para` - 无参数

**示例**

```cpp
int muteAudio(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_MUTE;

    if (vplay_control_recorder(rec, action, NULL) < 0) {
        // 错误处理
    }
}
```

#### 4.3.4 VPLAY_RECORDER_UNMUTE

Recorder取消录像静音。

**参数**

- `para` - 无参数

**示例**

```cpp
int unmuteAudio(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_UNMUTE;

    if (vplay_control_recorder(rec, action, NULL) < 0) {
        // 错误处理
    }
}
```

#### 4.3.5 VPLAY_RECORDER_SET_UDTA

Recorder设置用户数据信息。

**参数**

- `para` - vplay_udta_info_t，详见[数据描述](main.md#vplay_udta_info_t)

**示例**

```cpp
int setUdata(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_SET_UDTA;
    vplay_udta_info_t udta = {0};

    // 1: 设置文件路径
    char file[] = "/mnt/sd0/udta.bin";
    // 文件路径赋值
    udta.buf = file;
    udta.size = 0;
    if (vplay_control_recorder(rec, action, &udta) < 0) {
        // 错误处理
    }

    // 2: 设置字符串，字符串内容为"/mnt/sd0/udta.bin"
    char mem_buf[] = "I am the memory buffer";
    udta.buf = mem_buf;
    udta.size = strlen(mem_buf);// point to buffer
    if (vplay_control_recorder(rec, action, &udta) < 0) {
        // 错误处理
    }
}
```

#### 4.3.6 VPLAY_RECORDER_FAST_MOTION

设置微速摄影录制。

**参数**

- `para` - int型指针，表示FastMotion的倍数

**示例**

```cpp
int setFastMotion(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_FAST_MOTION;
    int rate = 8;

    if (vplay_control_recorder(rec, action, &rate) < 0) {
        // 错误处理
    }
}
```

#### 4.3.7 VPLAY_RECORDER_FAST_MOTION_EX

设置微速摄影录制，通过Player丢帧实现。

**参数**

- `para` - int型指针，表示FastMotion的倍数

**示例**

```cpp
int setFastMotionEx(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_FAST_MOTION_EX;
    int rate = 8;

    if (vplay_control_recorder(rec, action, &rate) < 0) {
        // 错误处理
    }
}
```

#### 4.3.8 VPLAY_RECORDER_SLOW_MOTION

设置高速摄影录制。

**参数**

- `para` - int型指针，表示SlowMotion的倍数

**示例**

```cpp
int setSlowMotion(vplay_recorder_inst_t *rec)
{
    vplay_ctrl_action_t action = VPLAY_RECORDER_SLOW_MOTION;
    int rate = 120;

    if (vplay_control_recorder(rec, action, &rate) < 0) {
        // 错误处理
    }
}
```

----
### 4.4 vplay_set_cache_mode()
```cpp
void vplay_set_cache_mode(vplay_recorder_inst_t *rec, int mode);
```
在录像开始时，获取Videobox已经缓存的视频帧数据。

**参数**

- `rec` - Recorder实例
- `mode` - 获取缓存视频帧选项，`0`代表不获取，`１`代表获取，不设置的话，默认为`0`

**返回**

无。

**示例**

```cpp
#include <qsdk/vplay.h>

init_recorder_default(&recorderInfo);
recorder = vplay_new_recorder(&recorderInfo);
if (recorder == NULL) {
    printf("new recorder error\n");
    return;
}

// APP start recorder for the first time
// record from the oldest frame in ring buffer
vplay_set_cache_mode(recorder, 1);
vplay_start_recorder(recorder);

// ...

// APP stop recorer and then start it again
// record from the latest frame in ring buffer
vplay_stop_recorder(recorder);
vplay_set_cache_mode(recorder, 0);
vplay_start_recorder(recorder);
// ...
```
----
## 5 数据类型

### 5.1 vplay_udta_info_t
```cpp
typedef struct {
    char *buf;
    int size;
} vplay_udta_info_t;
```

| 成员名称 | 描述 |
| --- | --- |
| `buf` | 内存`buf`指针或者文件名`buf`指针　|
| `size` | 当`size`为0时，`buf`指向文件名；当`size` > 0时，`buf`指向内存数据，最大为1MB |

相关接口：`vplay_control_recorder()`的`VPLAY_RECORDER_SET_UDTA`控制命令

----

### 5.2 vplay_ctrl_action_t
```cpp
typedef enum{
    // ...

    VPLAY_RECORDER_START = 0x1000,
    VPLAY_RECORDER_STOP,
    VPLAY_RECORDER_MUTE,
    VPLAY_RECORDER_UNMUTE,
    VPLAY_RECORDER_SET_UDTA,
    VPLAY_RECORDER_FAST_MOTION,
    VPLAY_RECORDER_SLOW_MOTION,
    // do fast motion by drop frames, rate should be n times of I frame interval
    VPLAY_RECORDER_FAST_MOTION_EX
} vplay_ctrl_action_t;
```

| 成员 | 描述 |
| --- | --- |
| `VPLAY_RECORDER_START` | Recorder开始录像　|
| `VPLAY_RECORDER_STOP` | Recorder停止录像 |
| `VPLAY_RECORDER_MUTE` | Recorder设置录像静音　|
| `VPLAY_RECORDER_UNMUTE` | Recorder设置录像静音取消　|
| `VPLAY_RECORDER_SET_UDTA` | Recorder设置用户数据信息　|
| `VPLAY_RECORDER_FAST_MOTION` | Recorder设置为快速录像，速率为原始数据源的ｎ倍　|
| `VPLAY_RECORDER_SLOW_MOTION` | Recorder设置为慢速录像，速率为原始数据源的ｎ分之一　|
| `VPLAY_RECORDER_FAST_MOTION_EX` | Recorder设置为快速录像，速率为原始数据源的ｎ倍（倍数只能为I帧间隔的整数倍）|

相关接口：`vplay_control_recorder()`

----

### 5.3 vplay_recorder_info_t
```cpp
typedef struct {
    char audio_channel[VPLAY_CHANNEL_MAX];
    char video_channel[VPLAY_CHANNEL_MAX];
    char videoextra_channel[VPLAY_CHANNEL_MAX];
    int enable_gps;
    int keep_temp;
    audio_info_t audio_format;
    int time_segment;
    int time_backward;
    char time_format[STRING_PARA_MAX_LEN];
    char suffix[STRING_PARA_MAX_LEN];
    char av_format[STRING_PARA_MAX_LEN];
    char dir_prefix[STRING_PARA_MAX_LEN];
    int fixed_fps;
} vplay_recorder_info_t;
```

| 成员名称 | 描述 |
| --- | --- |
| `audio_channel` | 音频通道，表示Audiobox视频数据通道，录制时从该通道获取编码后的视频数据；留空时，录制的文件不包含音频数据，字符串最大长度为64Byte（`default_mic`代表默认的麦克风设备；`btcodecmic`代表默认的蓝牙麦克风设备）|
| `video_channel` | 视频通道，表示Videobox视频数据通道，录制时从该通道获取编码后的视频数据，字符串最大长度为64Byte|
| `videoextra_channel` | 第二路视频通道，表示Videobox视频数据通道，字符串最大长度为64Byte|
| `enable_gps` | 不支持 |
| `keep_temp` | 是否保留临时文件，不保留临时文件时，临时文件命名为**.rec_temp.mp4**，每次都会自动删除；当保留临时文件时，按照命名规则命名文件，其中结束时间和时间长度都清空|
| `audio_format` | 容器内的音频的格式，字符串最大长度为64Byte|
| `time_segment` | 每个视频文件录制的分段时长，单位为秒 |
| `time_backward` | 回录模式下最长回录时间， 单位为秒 |
| `time_format` | 文件命名中包含时间信息时，定义时间的格式 |
| `suffix` | 多媒体文件命名规则，字符串最大长度为64Byte |
| `av_format` | 多媒体文件格式，暂时支持: MKV，MOV，MP4，字符串最大长度为64Byte |
| `dir_prefix` | 多媒体文件及临时信息存放目录，字符串最大长度为64Byte |
| `fixed_fps` | 多媒体文件帧率控制，0代表自动识别帧率，当设置SLOW/FAST motion时，需要指定该数字 |

相关接口：`vplay_new_recorder()`

----

### 5.4 audio_info_t
```cpp
typedef struct {
    audio_codec_type_t type;
    int sample_rate;
    int channels;
    sample_format_t sample_fmt;
    int effect;
} audio_info_t;
```

| 成员名称 | 描述 |
| --- | --- |
| `type` | 内存Buffer指针或者文件名Buffer指针。参见[**5.5 audio_codec_type_t**](main.md#5.5_audio_codec_type_t) |
| `sample_rate` | 音频采样率 |
| `channels` | 音频声道数量 |
| `sample_fmt` | 音频采样率，参见[**5.6 sample_format_t**](main.md#5.6_sample_format_t) |
| `effect` | 音效（暂时无用） |

相关接口：`vplay_new_recorder()`接口参数`vplay_recorder_info_t.audio_format`

----

### 5.5 audio_codec_type_t
```cpp
typedef enum  {
    AUDIO_CODEC_TYPE_NONE = -1,
    AUDIO_CODEC_TYPE_PCM = 0,
    AUDIO_CODEC_TYPE_G711U,         // not support now
    AUDIO_CODEC_TYPE_G711A,         // only work in mkv
    AUDIO_CODEC_TYPE_AAC,           // only work in mp4
    AUDIO_CODEC_TYPE_ADPCM_G726,    // not support now
    AUDIO_CODEC_TYPE_MP3,           // not support now
    AUDIO_CODEC_TYPE_SPEEX,         // not support now
    AUDIO_CODEC_TYPE_MAX
} audio_codec_type_t;
```

| 成员名称 | 描述 |
| --- | --- |
| `AUDIO_CODEC_TYPE_PCM` |  PCM音频 |
| `AUDIO_CODEC_TYPE_G711U` |  G711U 音频，容器MKV支持 |
| `AUDIO_CODEC_TYPE_G711A` |  G711A 音频，容器MKV支持 |
| `AUDIO_CODEC_TYPE_AAC` |  AAC音频，容器MP4支持 |
| `AUDIO_CODEC_TYPE_ADPCM_G726` |  G726音频，暂无容器支持 |
| `AUDIO_CODEC_TYPE_MP3` |  MP3音频，暂不支持 |
| `AUDIO_CODEC_TYPE_SPEEX` |  SPEEX音频，暂不支持 |

相关接口：`vplay_new_recorder()`接口参数`vplay_recorder_info_t.audio_format.audio_codec_type_t`

----

### 5.6 sample_format_t
```cpp
typedef enum {
    AUDIO_SAMPLE_FMT_NONE = -1,
    AUDIO_SAMPLE_FMT_S8,
    AUDIO_SAMPLE_FMT_S16,
    AUDIO_SAMPLE_FMT_S24,
    AUDIO_SAMPLE_FMT_S32,
    // float
    AUDIO_SAMPLE_FMT_FLTP,
    // double
    AUDIO_SAMPLE_FMT_DBLP,
    AUDIO_SAMPLE_FMT_MAX

} sample_format_t;
```

| 成员名称 | 描述 |
| --- | --- |
|`AUDIO_SAMPLE_FMT_S8`|8Bit的音频采样格式|
|`AUDIO_SAMPLE_FMT_S16`|16Bit的音频采样格式|
|`AUDIO_SAMPLE_FMT_S24`|24Bit的音频采样格式|
|`AUDIO_SAMPLE_FMT_FLTP`|float类型的音频采样格式|
|`AUDIO_SAMPLE_FMT_DBLP`|double类型的音频采样格式|

相关接口：`vplay_new_recorder()`接口参数`vplay_recorder_info_t.audio_format.sample_fmt`

----
## 6 调试工具

vrec是Recorder的调试程序，可以设置vrec不同的启动参数，来测试Recorder的各项功能。

编译qlibvplay的时候会自动生成vrec，生成目录为**QSDK/output/build/qlibvplay-1.0.0/**。如需使用，请手动拷贝到板子上运行。

----

### 6.1 vrec使用方法

| 命令选项 | 描述 |
| --- | --- |
| `-v`, `--video_channel` | 对应`VRecorderInfo.video_channel`，Videobox的Video Channel信息 |
| `-V` | 对应`VRecorderInfo.videoextra_channel`，Videobox的第二路Video Channel信息 |
| `-a`, `--audio_channel` | 对应`VRecorderInfo.audio_channel`，Audiobox的Audio Channel信息 |
| `-t`, `--audio_type` | 对应`VRecorderInfo.audio_format.type`，表示音频格式信息，`0`: pcm `1`: g711u `2`: g711a `3`: aac `4`: g726 `5`: mp3 |
| `-r`, `--audio_sample_rate` | 对应`VRecorderInfo.audio_format.sample_rate`，表示音频采样率 |
| `-f`, `--audio_sample_fmt` | 对应`VRecorderInfo.audio_format.sample_fmt`，表示音频采样格式，0:8 1:16 2:24 3:32 |
| `-c`, `--audio_channels` | 对应`VRecorderInfo.audio_format.channels`，表示声道数 |
| `-s`, `--time_segment` | 对应`VRecorderInfo.time_segment`，表示录像分段时长 |
| `-b`, `--time_backward` | 对应`VRecorderInfo.time_backward`，表示预录像时长 |
| `-S`, `--suffix` | 对应`VRecorderInfo.suffix`，表示文件命名规则中文件前缀命名规则 |
| `-F`, `--av_format` | 对应`VRecorderInfo.av_format`，表示容器类型 |
| `-d`, `--dir_prefix` | 对应`VRecorderInfo.dir_prefix`，表示录像保存路径 |
| `-m`, `--mute` | 控制Recorder进行静音动作 |
| `-k`, `--keep_temp` | 对应`VRecorderInfo.keep_temp`，表示是否保存临时录像文件 |
| `--slowmotion` | 设置高速摄影倍速 |
| `--fastmotion` | 设置微速摄影倍数，只支持`VPLAY_RECORDER_FAST_MOTION`模式 |

----

### 6.2 常用命令

**示例一**
```bash
vrec --help
```
显示帮助信息。

**示例二**
```bash
vrec -v enc720p-stream -b 0 -s 100
```

单路MKV录制。默认容器为`mkv`，视频通道为`enc720p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒。

**示例三**
```bash
vrec -v enc720p-stream -b 0 -s 100 -t 3 -F mp4 -d /mnt/sd0/
```

单路MP4录制。视频通道为`enc720p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒，设置音频为AAC，容器为`mp4`，录像文件保存位置为`/mnt/sd0/`。

**示例四**
```bash
vrec -v enc720p-stream -b 0 -s 100 -t 3 -F mp4 -d /mnt/sd0/ -S %ts_%l_F
```

设置特定文件前缀命名规则。视频通道为`enc720p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒，设置音频为AAC，容器为`mp4`，录像文件保存位置为`/mnt/sd0/`，文件前缀命名规则为`%ts_%l_F`。

**示例五**
```bash
vrec -v enc720p-stream -b 0 -s 100 -t 3 -F mp4 -d /mnt/sd0/ --slowmotion 2
```

2倍速高速摄影录制。视频通道为`enc720p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒，设置音频为AAC，容器为`mp4`，录像文件保存位置为`/mnt/sd0/`，高速摄影倍数设置为`2`。

**示例六**
```bash
vrec -v enc720p-stream -b 0 -s 100 -t 3 -F mp4 -d /mnt/sd0/ --fastmotion 2
```

2倍速微速摄影录制。视频通道为`enc720p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒，设置音频为AAC，容器为`mp4`，录像文件保存位置为`/mnt/sd0/`，微速摄影倍数设置为`2`。

**示例七**
```bash
vrec -v enc720p-stream -V enc1080p-stream -b 0 -s 100 -t 3 -F mp4 -d /mnt/sd0/
```

双路录制。第一路视频通道为`enc720p-stream`，第二路视频通道为`enc1080p-stream`，预录像时长为`0`秒，录像分段时长为`100`秒，设置音频为AAC，容器为`mp4`，录像文件保存位置为`/mnt/sd0/`。
