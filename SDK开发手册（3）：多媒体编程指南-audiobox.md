### 术语解释
|术语|解释|
|-----------------|----------------------------------|
| Audiobox | SDK中音频处理模块 |
| API | Aplication Program Interface，应用程序接口 |
| AEC | Acoustic Echo Cancellation，回声消除 |

## 1 概述
Audiobox 是SDK的音频子系统，提供音频的录音和放音功能支持，同时支持静音(Mute)控制、音量(Volume)控制、音频流格式(Format)控制、回声消除(AEC)控制等音频通道参数管理功能。另外还提供abctrl工具用来测试Audiobox各项功能

----
## 2 系统架构

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/Audiobox_frame.svg)

Audiobox作为一个独立的进程运行，应用层调用Audiobox提供的API接口实现功能。Audiobox模块包含录音、放音和AEC功能子模块支持相应的功能请求

----
## 3 配置选项
### 3.1 Audiobox配置
在SDK源码目录下执行命令`make menuconfig`进入配置界面，按照如下粗体配置

<pre>
QSDK Options  --->
  |-Apps
  |   |-[*] eventhub
  |   |-[*] common event provider
  |   <strong>|-[*] Audiobox</strong>
  |   |-[*] Videobox
  |   |-[ ] ... ...
  |-Libs
  |-Debug Tools
</pre>

----
### 3.2 AEC功能配置

**加载DSP驱动配置**
AEC功能的实现依赖于DSP库。要使用DSP库，还需确认所使用的芯片平台是否带有DSP模块并确认加载DSP驱动，当前只有`Q3F`内部集成有DSP

在SDK源码目录下执行命令`make linux-menuconfig`进入配置界面，按照如下粗体配置

<pre>
Device Drivers  --->
  |-[ ] ... ...
  |-[*] InfoTM special files and drivers  --->
  |   |---- InfoTM special files and drivers
  |   |-[*]   Infotm Q3F Series Support  --->
  |   |   	  |---- Infotm Q3F Series Support
  |   |   	  |- ... ...
  |   |   	  <strong>|-[*]   Q3F dsp driver support  ---></strong>
  |   |   	  |		|---- Q3F dsp driver support
  |   |   	  |		<strong>|- < M >   ceva tl421 dsp  ---></strong>
  |   |   	  |		|		|---- ceva tl421 dsp
  |   |   	  |		|		|-[ ]   Q3F dsp support Encode function (NEW)
  |   |   	  |		|		|-[ ]   Q3F dsp support Decode function (NEW)
  |   |   	  |		|		<strong>|-[*]   Q3F dsp support AEC function (NEW)</strong>
  |   |   	  |- ... ...
  |   |-<*>   Infotm common drivers support  --->
</pre>

请确认`ceva tl421 dsp`选项为`M`，因为DSP的驱动只能选为模块加载方式，DSP需要在初始化时载入固件镜像，而这个固件镜像必须在系统完成初始化、文件系统挂载完毕后才会被DSP驱动所识别到

**关闭AEC功能的配置**
如果无需支持AEC功能，在SDK源码目录下执行命令`make menuconfig`进入配置界面，按照如下粗体配置
<pre>
QSDK Options  --->
  |-Apps
  |-Libs
  |   |-[ ] ... ...
  |   <strong>|-[ ] hlibdsp</strong>
  |   |-*   hlibvcp7g
  |   |-	  Options  --->
  |   |   	  <strong>|-[ ] enable arm-lib support(not recommanded)</strong>
  |   |-[ ] ... ...
  |-Debug Tools
</pre>

**启用AEC功能(使用DSP库)的配置**
在SDK源码目录下执行命令`make menuconfig`进入配置界面，按照如下粗体配置
<pre>
QSDK Options --->
  |-Apps
  |-Libs
  |   |-[ ] ... ...
  |   <strong>|-[*] hlibdsp</strong>
  |   |-	  Options --->
  |   |   	  <strong>|- DSP Type (ceva-tl421)  ---></strong>
  |   |   	  		|	<strong>|-[X] ceva-tl421</strong>
  |   |   	  |- DDR Size (64MB)  --->
  |   |-*  hlibvcp7g
  |   |-	  Options  --->
  |   |   	  <strong>|-[*] enable dsp-lib support(NEW)</strong>
  |   |   	  <strong>|-[ ] enable arm-lib support(not recommanded)</strong>
  |   |-[ ] ... ...
  |-Debug Tools
</pre>


Note: `enable arm-lib support`选项为内部测试用，启用或关闭AEC功能都不要选中

----
## 4 应用场景
### 4.1 单路录音场景

单路录音场景基本流程如下

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/capture_voice_flow.svg)

**录音场景范例：**
```cpp
/*计算单个音频帧数据长度*/
int stop_capture = 0;

static int get_buffer_size(audio_fmt_t *fmt){
	int bitwidth = ((fmt->bitwidth > 16) ? 32 : fmt->bitwidth);
	return (fmt->channels * (bitwidth >> 3) * fmt->sample_size);
}

void* thread_capture_voice(void* arg){
	char *audiobuf = NULL;
	struct fr_buf_info buf;
	char pcm_name[32] = "default_mic";
	audio_fmt_t real;
	audio_fmt_t fmt = {
		.channels = 2,	.bitwidth = 32,	.samplingrate = 48000,	.sample_size = 1024,
	};
    /*初始化哪些音频通道参数*/
    int	init_format = 1, init_master_volume = 1, init_volume = 1, init_mute = 1, init_aec = 1;
    int	use_reference_mechanism = 1;
    int	master_volume = 100, volume = 100;
    /*Step 1: 申请录音逻辑通道*/
    int	handle = audio_get_channel(pcm_name, &fmt, CHANNEL_BACKGROUND);
    if (0 > handle) {
		ABCTL_ERR("Get channel err: %d\n", handle);
		return NULL;
	}

    /*Step 2: 音频通道参数配置*/
    if(init_format)
    	audio_set_format(pcm_name, fmt);
    if(init_master_volume)
    	audio_set_master_volume(pcm_name, master_volume, DEVICE_IN_ALL);
	if(init_volume)
    	audio_set_volume(handle, volume);
    if(init_mute)
    	audio_set_mute(handle, 1);
	if(init_aec)
    	audio_enable_aec(handle, 1);

	/*Step 3: 录音过程*/
    int ret = audio_get_format(pcm_name, &real); /*获取实际音频流格式信息*/
    int length = get_buffer_size(&real);　/*计算单个音频帧缓冲区大小，拷贝方式需要用到*/
    int fd = open("/mnt/capture_sample.wav", O_CREAT | O_WRONLY | O_TRUNC, 0666);
	if(1 == use_reference_mechanism){			/*采用引用方式录音*/
        while(0 == stop_capture){				/*录音过程未结束*/
            audio_get_frame(handle, &buf);		/*获取音频帧访问引用*/
            write(fd, buf.virt_addr, buf.size);	/*将音频帧写入指定文件*/
            audio_put_frame(handle, &buf);		/*释放音频帧访问引用*/
        }
    }else if(0 == use_reference_mechanism){ 			　/*采用拷贝方式录音*/
    	audiobuf = (char *)malloc(length);				  /*分配本地缓冲区*/
        while(0 == stop_capture){						  /*录音过程未结束*/
            audio_read_frame(handle, &audiobuf, length);　/*先将音频帧拷贝到本地缓冲区*/
            write(fd, audiobuf, length);				　/*再将音频帧从本地缓冲区写入指定文件*/
        }
    }
	close(fd);

    /*Step 4: 释放音频逻辑通道*/
    audio_put_channel(handle);

    return NULL;
}

int main(void){
	pthread_t capture_id = 0;

	stop_capture = 0;
	ret = pthread_create(&capture_id, NULL, thread_capture_voice, NULL);/*启动录音线程*/
	if (ret != 0) {
		AUD_ERR("Create capture thread err\n");
		return -1;
	}
    sleep(20);
    stop_capture = 1;	/*20秒之后停止录音*/
    if(capture_id){
    	pthread_join(capture_id);
    }
    return 0;
}
```

上述范例中，应用创建一个录音线程，线程中实现单路录音场景，`20`秒后停止录音，退出录音线程

#### Step 1: 申请录音逻辑通道
应用程序调用`audio_get_channel()`申请音频逻辑通道用于录音，详见[**5.3 audio_get_channel()**][5.3]

#### Step 2: 音频通道参数配置
申请音频逻辑通道之后，可以在录音开始之前根据需要静态配置音频通道参数，包括静音(Mute)控制、音量(Volume)控制、主音量(master_volume)控制、音频流格式(Format)控制、回声消除(AEC)控制，参照上面范例

也支持录音过程中动态配置上述音频通道参数。需要注意的是

- 音频物理通道级别的参数可以在任何进程中配置，包括`audio_set_format(pcm_name,...)`和`audio_set_master_volume(pcm_name,...)`
- 音频逻辑通道级别的参数只能在申请`handle`的进程内配置，包括`audio_set_volume(handle,...)`、`audio_set_mute(handle,...)`、`audio_enable_aec(handle,...)`，因为`handle`只在申请它的进程内有效

#### Step 3: 录音过程
应用程序从Audiobox读取音频帧数据，写入文件或者其他目标，直到录音停止

Audiobox支持两种方式读取音频帧

- `引用方式` - 应用调用`audio_get_frame()`从Audiobox申请独占音频缓冲区中一个音频帧，阻塞等待直到获取到就绪的音频帧后，调用`write()`直接将此音频帧写入到文件或其他目标，然后调用`audio_put_frame()`释放"引用"之后，Audiobox才可以继续访问此音频帧
- `拷贝方式` - 需要本地先分配`audiobuf`缓冲区，调用`audio_read_frame()`会阻塞等待直到获取到就绪的音频帧并将其拷贝到`audiobuf`，应用进程`write()`调用再将`audiobuf`数据写入到文件或其他目标。

`引用方式`必须保证`audio_put_frame()`与`audio_get_frame()`成对使用，而且`audio_put_frame()`必须尽快调用以释放占用的数据帧，否则延后太久会阻塞Audiobox录音；`拷贝方式`比`引用方式`要多分配一个数据帧的缓冲区，并增加一次数据帧额外拷贝动作，但不会阻塞Audiobox录音

#### Step 4: 释放音频逻辑通道
录音过程结束，应用程序调用`audio_put_channel()`释放音频逻辑通道，`audio_put_channel()`必须和`audio_get_channel()`成对使用

----
### 4.2 单路放音场景

单路放音基本流程如下

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/playback_voice_flow.svg)

**单路放音场景范例：**
```cpp
int stop_playback = 0;

/*计算单个音频帧数据长度*/
static int get_buffer_size(audio_fmt_t *fmt){
	int bitwidth = ((fmt->bitwidth > 16) ? 32 : fmt->bitwidth);
	return (fmt->channels * (bitwidth >> 3) * fmt->sample_size);
}

void* thread_playback_voice(void* arg){
	char *audiobuf = NULL;
    int len = 0;
	struct fr_buf_info buf;
	char pcm_name[32] = "default";
	audio_fmt_t real;
	audio_fmt_t fmt = {
		.channels = 2,	.bitwidth = 32,	.samplingrate = 48000,	.sample_size = 1024,
	};
    /*初始化哪些音频通道参数*/
    int	init_format = 1, init_master_volume = 1, init_volume = 1, init_mute = 1, init_aec = 1;
    int	master_volume = 100, volume = 100;
    /*Step 1: 申请放音逻辑通道*/
    int handle = audio_get_channel(pcm_name, &fmt, CHANNEL_BACKGROUND);
    if (0 > handle) {
		ABCTL_ERR("Get channel err: %d\n", handle);
		return NULL;
	}

    /*Step 2: 音频通道参数配置*/
    if(init_format)
    	audio_set_format(pcm_name, fmt);
    if(init_master_volume)
    	audio_set_master_volume(pcm_name, master_volume, DEVICE_OUT_ALL);
	if(init_volume)
    	audio_set_volume(handle, volume);
    if(init_mute)
    	audio_set_mute(handle, 1);

	/*Step 3: 放音过程*/
    int ret = audio_get_format(pcm_name, &real); /*获取实际音频流格式信息*/
    int length = get_buffer_size(&real);　/*计算单个音频帧缓冲区大小*/
    int fd = open("/mnt/capture_sample.wav", O_CREAT | O_WRONLY | O_TRUNC, 0666);
    while(0 == stop_playback){				/*放音过程未结束*/
        len = read(fd, audiobuf, length);
        if(0 >= len){						/*到达文件末尾*/
        	AUD_ERR("voice file reach end！\n");
        	break;
        }
        audio_write_frame(handle, audiobuf, length);
    }
	close(fd);

    /*Step 4: 释放音频逻辑通道*/
    audio_put_channel(handle);
}

int main(void){
	pthread_t playback_id = 0;

	stop_playback = 0;
	ret = pthread_create(&playback_id, NULL, thread_playback_voice, NULL);/*启动放音线程*/
	if (ret != 0) {
		AUD_ERR("Create playback thread err\n");
		return -1;
	}
    sleep(20);
    stop_playback = 1;	/*20秒之后停止放音*/
    if(playback_id){
    	pthread_join(playback_id);
    }
    return 0;
}
```
上述范例中，应用创建一个放音线程，线程中实现单路放音场景，`20`秒后或文件播放完毕后停止放音，退出放音线程

#### Step 1: 申请放音逻辑通道
应用程序调用`audio_get_channel()`申请音频逻辑通道用于放音，详见[**5.3 audio_get_channel()**][5.3]

#### Step 2: 音频通道参数配置
可以根据需要配置音频通道参数，包括静音(Mute)控制、音量(Volume)控制、主音量(master_volume)控制、音频流格式(Format)控制，比录音场景少了回声消除(AEC)控制，参照上面范例

#### Step 3: 放音过程
应用程序调用`audio_write_frame()`向Audiobox写入音频帧数据，直到放音结束

#### Step 4: 释放音频逻辑通道
放音过程结束，应用程序调用`audio_put_channel()`释放音频逻辑通道，`audio_put_channel()`必须和`audio_get_channel()`成对使用

----
### 4.3 多路放音场景

Audiobox支持音频逻辑通道优先级设置，有前景音(`CHANNEL_FOREGROUND`)和背景音(`CHANNEL_BACKGROUND`)两个优先级，应用调用`audio_get_channel()`申请逻辑通道时可以指定`priority`参数为其中一个。当单路前景音通道与单路背景音通道同时放音时，Audiobox只在扬声器播放前景音通道的音频流。多路前景音或多路背景音的场景没有意义，应该避免出现这样的场景

**多路放音场景流程图：**

![](https://github.com/InfoTM-SDK/Q3FSDK/blob/master/wiki_res/dual_chan_playback_voice_sequence_flow.svg)

**多路放音场景范例：**
```cpp
typedef struct _playback_channel_para {
	pthread_t playback_id;
	char file_name[64];		/*sound file name*/
	int priority;
	int stop_playback;
}PLAY_CHAN_PARA;

/*计算单个音频帧数据长度*/
static int get_buffer_size(audio_fmt_t *fmt){
	int bitwidth = ((fmt->bitwidth > 16) ? 32 : fmt->bitwidth);
	return (fmt->channels * (bitwidth >> 3) * fmt->sample_size);
}

void* thread_playback_voice(void* arg){
	PLAY_CHAN_PARA	*p_chan_para = (PLAY_CHAN_PARA *)arg;
	char *audiobuf = NULL;
    int len = 0;
	struct fr_buf_info buf;
	char pcm_name[32] = "default";
	audio_fmt_t real;
	audio_fmt_t fmt = {
		.channels = 2,	.bitwidth = 32,	.samplingrate = 48000,	.sample_size = 1024,
	};
    /*初始化哪些音频通道参数*/
    int	init_format = 1, init_master_volume = 1, init_volume = 1, init_mute = 1, init_aec = 1;
    int	master_volume = 100, volume = 100;
    /*申请放音逻辑通道*/
    int handle = audio_get_channel(pcm_name, &fmt, p_chan_para->priority);
    if (0 > handle) {
		ABCTL_ERR("Get channel err: %d\n", handle);
		return NULL;
	}

    /*音频通道参数配置*/
    if(init_format)
    	audio_set_format(pcm_name, fmt);
    if(init_master_volume)
    	audio_set_master_volume(pcm_name, master_volume, DEVICE_OUT_ALL);
	if(init_volume)
    	audio_set_volume(handle, volume);
    if(init_mute)
    	audio_set_mute(handle, 1);

	/*放音过程*/
    int ret = audio_get_format(pcm_name, &real); /*获取实际音频流格式信息*/
    int length = get_buffer_size(&real);　/*计算单个音频帧缓冲区大小*/
    int fd = open(p_chan_para->file_name, O_CREAT | O_WRONLY | O_TRUNC, 0666);
    while(0 == p_chan_para->stop_playback){				/*放音过程未结束*/
        len = read(fd, audiobuf, length);
        if(0 >= len){						/*到达文件末尾*/
        	AUD_ERR("voice file reach end！\n");
        	break;
        }
        audio_write_frame(handle, audiobuf, length);
    }
	close(fd);

    /*释放音频逻辑通道*/
    audio_put_channel(handle);
}

int main(void){
	PLAY_CHAN_PARA	bg_sound_chan_para = {0, "/mnt/bg_sound.wav", CHANNEL_BACKGROUND, 0};
    PLAY_CHAN_PARA	alarm_chan_para = {0, "/mnt/alarm.wav", CHANNEL_FOREGROUND, 0};

	/*Step 1: 创建背景音通道，播放背景音文件*/
	ret = pthread_create(&bg_sound_chan_para.playback_id, NULL, thread_playback_voice, &bg_sound_chan_para);
	if (ret != 0) {
		AUD_ERR("Create background playback thread err\n");
		return -1;
	}
	sleep(20);

    /*Step 2: 创建前景音通道，播放报警铃声文件*/
	ret = pthread_create(&alarm_chan_para.playback_id, NULL, thread_playback_voice, &alarm_chan_para);
	if (ret != 0) {
		AUD_ERR("Create foreground playback thread err\n");
		return -1;
	}
	sleep(10);

    /*Step 3: 释放前景音通道，停止播放报警铃声，继续播放背景音文件*/
    alarm_chan_para.stop_playback = 1;
    if(alarm_chan_para.playback_id){
    	pthread_join(alarm_chan_para.playback_id);
    }

	return 0;
}
```

上述范例中，应用先后创建背景音和前景音两个放音线程，前景音通道播放时会覆盖背景音通道播放，前景音通道结束播放后，继续播放背景音通道

#### Step 1: 创建背景音通道，播放背景音文件

创建放音线程，申请背景音通道，扬声器开始播放`/mnt/bg_sound.wav`

#### Step 2: 创建前景音通道，播放报警铃声文件

创建放音线程，申请前景音通道，由于Audiobox优先播放前景音通道，前景音通道抢占`ALSA`，扬声器开始播放`/mnt/alarm.wav`，背景音通道被无视

#### Step 3: 释放前景音通道，停止播放报警铃声，继续播放背景音通道

停止播放报警铃声，释放前景音通道后，扬声器继续播放背景音通道的`/mnt/bg_sound.wav`文件

----
### 4.4 通道音量控制

应用可以调用`audio_set_master_volume()`和`audio_set_volume()`来调整录音或放音逻辑通道的实际工作音量，计算方式为：`real_volume = volume * master_volume / 100;`，这里涉及到两个音量的概念

- `master_volume`是指定的音频物理通道的音量，代表音频物理通道录音或者放音音量的最大值，取值范围[`0`~`100`]，缺省为`100`,可以由`audio_set_master_volume()`设置。举个例子，如果音频设备功率正常或偏小，可以设置`master_volume`为`100`；否则，可以调小`master_volume`避免出现不正常的大音量
- `volume`是指定的音频逻辑通道的音量，可以由`audio_set_volume()`设置，取值范围[`0`~`256`]，缺省为`256`

**调整放音物理通道最大音量**
```cpp
char pcm_name[32] = "default";
out_master_volume = 80;
audio_set_master_volume(pcm_name, out_master_volume, DEVICE_OUT_ALL)
```

**调整录音物理通道最大音量**
```cpp
char pcm_name_mic[32] = "default_mic";
in_master_volume = 80;
audio_set_master_volume(pcm_name_mic, in_master_volume, DEVICE_IN_ALL);
```

**调整录音和放音逻辑通道音量**
```cpp
volume = 128;
audio_set_volume(handle, volume);
```

----
## 5 API
### 5.1 audio_start_service()

```cpp
int audio_start_service(void)
```

启动audiobox进程

**参数**
- `无`

**返回**
该函数成功返回0，失败返回-1

Note: Audiobox进程一般在/etc/init.d的启动脚本中启动，这种情况下应用无须调用`audio_start_service()`

----
### 5.2 audio_stop_service()

```cpp
int audio_stop_service(void);
```

中止audiobox进程

**参数**

- `无`

**返回**

该函数成功返回0，失败返回-1

----
### 5.3 audio_get_channel()

```cpp
int audio_get_channel(const char *channel, audio_fmt_t *fmt, int flag);
```
申请录音或放音音频逻辑通道

**参数**

- `channel` -　字符串格式的音频物理通道名称，指定了上层需要操作的设备类型，SDK目前只支持４个固定类型物理通道，见下表

| `channel`类型 | 描述 |
|---|
| `defualt` | 默认的扬声器设备 |
| `default_mic` | 默认的麦克风设备 |
| `btcodec` | 默认的蓝牙扬声器设备 |
| `btcodecmic` | 默认的蓝牙麦克风设备 |

- `fmt` -　指向音频流格式结构体的指针，详见[**6.2 audio_fmt_t**][6.2]
- `flag` -　通道特性，详细信息如下表

| `flag`位域 | 描述 |
|---|
| `bit0~1` | 放音特性，定义音频逻辑通道优先级，详见[**6.2 channel_priority**][6.2] |
| `bit2` | 未定义 |
| `bit3` | 放音超时退出标记，定义音频逻辑通道超过30分钟无任何音频流输入时，是否自动释放逻辑通道。`1`：释放,`0`:不释放 |
| `bit4~31` | 未定义 |

**返回**

失败返回-1，成功返回非负的逻辑通道handle

Note: 如果`audio_get_channel()`申请的物理通道正在被使用（有其它程序已经注册了逻辑通道）而`fmt`参数与当前物理通道参数配置不同，则本次调用会失败。所以如果一个程序对音频属性没有特别需求，建议把`fmt`设置为`NULL`

----
### 5.4 audio_put_channel()

```cpp
int audio_put_channel(int handle);
```

释放申请的逻辑通道

**参数**

- `handle` - 音频逻辑通道句柄

**返回**

该函数成功返回0，失败返回-1

----
### 5.5 audio_test_frame()

```cpp
int audio_test_frame(int handle, struct fr_buf_info *buf)
```

测试音频通道中的当前帧是否更新

**参数**

- `handle` - 音频逻辑通道句柄
- `buf` - 指向封装音频帧的fr_buf_info结构体的指针，详见[**6.3 fr_buf_info**][6.3]

**返回**

- '大于0' - 当前视频帧已更新
-  '0' - 当前视频帧未更新
- '小于0' - 出错失败

----
### 5.6 audio_get_frame()

```cpp
int audio_get_frame(int handle, struct fr_buf_info *buf);
```

从audiobox中获得音频帧数据，使用完毕必须调用`audio_put_frame()`释放音频帧

**参数**

- `handle` - 音频逻辑通道句柄
- `buf` - 指向封装音频帧的fr_buf_info结构体的指针，详见[**6.3 fr_buf_info**][6.3]

**返回**

该函数成功返回0，失败返回-1

----
### 5.7 audio_put_frame()

```cpp
int audio_put_frame(int handle, struct fr_buf_info *buf);
```

释放由调用`audio_get_frame()`占有的音频帧缓冲区

**参数**

- `handle` - 音频逻辑通道句柄
- `buf` - 指向封装音频帧的fr_buf_info结构体的指针，详见[**6.3 fr_buf_info**][6.3]

**返回**

该函数成功返回0，失败返回-1

----
### 5.8 audio_read_frame()

```cpp
int audio_read_frame(int handle , char *buf, int size);
```

从audiobox中拷贝得到一帧音频数据

**参数**

- `handle` - 音频逻辑通道句柄
- `buf` - 缓存区指针，用来保存从audiobox中获得的音频数据
- `size` - `buf`缓冲区的最大长度，应不小于音频帧数据长度

**返回**

该函数成功返回实际读取的音频帧数据的长度，失败返回-1

----
### 5.9 audio_write_frame()

```cpp
int audio_write_frame(int handle, char *buf, int size);
```

向audiobox中传入一帧音频数据

**参数**

- `handle` - 音频逻辑通道句柄
- `buf` - 缓存空间，保存写入audiobox的音频帧数据
- `size` - 写入的音频帧数据长度

**返回**

该函数成功返回实际写入的音频帧数据的长度，失败返回-1

----
### 5.10 audio_get_format()

```cpp
int audio_get_format(const char *channel, audio_fmt_t *fmt);
```

从audiobox获得当前codec的硬件配置

**参数**

- `channel` - 字符串格式的音频物理通道名称
- `fmt` - 指向音频流格式结构体的指针，详见[**6.2 audio_fmt_t**][6.2]

**返回**

该函数成功返回0，失败返回-1

----
### 5.11 audio_set_format()

```cpp
int audio_set_format(const char *channel, audio_fmt_t *fmt);
```

向audiobox配置当前物理通道

**参数**

- `channel` - 字符串格式的音频物理通道名称
- `fmt` - 指向音频流格式结构体的指针，详见[**6.2 audio_fmt_t**][6.2]

**返回**

该函数成功返回0，失败返回-1

----
### 5.12 audio_get_mute()

```cpp
int audio_get_mute(int handle);
```

获得handle描述的逻辑通道的mute状态

**参数**

- `handle` - 逻辑通道句柄

**返回**

- `1` - 处于mute
- `0` - 不处于mute
- `-1` - 失败

----
### 5.13 audio_set_mute()

```cpp
int audio_set_mute(int handle, int mute);
```
设置音频逻辑通道mute状态

**参数**

- `handle` - 逻辑通道句柄
- `mute` - 需要向audiobox配置的mute状态，`1`：mute，`0`：unmute

**返回**

该函数成功返回0，失败返回-1

----
### 5.14 audio_get_volume()

```cpp
int audio_get_volume(int handle);
```
获取音频逻辑通道当前音量

**参数**

- `handle` - 逻辑通道句柄

**返回**

该函数成功返回音频逻辑通道当前音量值，失败返回-1

----
### 5.15 audio_set_volume()

```cpp
int audio_set_volume(int handle, int volume);
```
调整音频逻辑通道音量

**参数**

- `handle` - 逻辑通道句柄
- `volume` - 指定的逻辑通道新的音量值，取值范围[`0`~`256`]

**返回**

该函数成功返回0，失败返回-1

----
### 5.16 audio_get_master_volume()

```cpp
int audio_get_master_volume(const char *channel)
```
获取音频物理通道当前最大音量值

**参数**

- `channel` - 字符串格式的音频物理通道名称

**返回**

该函数成功返回音频物理通道当前最大音量值，失败返回-1

----
### 5.17 audio_set_master_volume()

```cpp
int audio_set_master_volume(const char *channel, int volume)
```
调整音频物理通道最大音量值

**参数**

- `channel` - 字符串格式的音频物理通道名称
- `volume` - 指定的音频物理通道新的最大音量值，取值范围[`0`~`100`]

**返回**

该函数成功返回0，失败返回-1

----
### 5.18 audio_enable_aec()

```cpp
int audio_enable_aec(int handle, int enable)
```
使能或禁止AEC功能

**参数**

- `handle` - 逻辑通道句柄
- `enable` - 是否使能AEC功能，`1`:使能 `0`:禁止

**返回**

该函数成功返回0，失败返回-1

----
## 6 数据类型
### 6.1 channel_priority

```cpp
enum channel_priority {
	CHANNEL_BACKGROUND,
	CHANNEL_FOREGROUND,
};
```
Audiobox把音频逻辑通道分成前景音(CHANNEL_FOREGROUND)和背景音(CHANNEL_FOREGROUND)两个优先级，如果申请的channel需要在播放音频时将其他所有声音屏蔽，则需要将此channel指定为CHANNEL_FOREGROUND优先级，而将其他所有channel指定为CHANNEL_BACKGROUND优先级；如果没有这种需求，一般选择为CHANNEL_BACKGROUND优先级

| 成员名称 | 描述 |
|---|
| `CHANNEL_BACKGROUND` | 背景音，大多数通道申请时应该选择此属性 |
| `CHANNEL_FOREGROUND` | 前景音，前景音在发声时会覆盖背景音．一般报警铃声等需要申请此属性 |

相关API接口：`audio_get_channel()`

----
### 6.2 audio_fmt_t

```cpp
struct audio_format {
	int channels;
	int bitwidth;
	int samplingrate;
	int sample_size;
};
typedef struct audio_format audio_fmt_t;
```

| 成员名称 | 描述 |
|---|
| `channels` | 音频声道数，`1`：单声道，`2`：双声道 |
| `bitwidth` | 音频通道比特位宽 |
| `samplingrate` | 音频通道采样率 |
| `sample_size` | 音频通道每次需要存取的音频帧的采样个数|

相关API接口：`audio_get_channel()` `audio_get_format()` `audio_set_format()`

----
### 6.3 fr_buf_info

```cpp
struct fr_buf_info {
	void *fr;
	void *buf;

	void *virt_addr;
	uint32_t phys_addr;
	uint64_t timestamp;
	int size;
	int map_size;
	int serial;
	int priv;
    int oldest_flag;//0:disable 1:enable

	/* DO NOT MODIFY */
	struct fr_buf_info *next, *prev;
};
```

| 成员名称 | 描述 |
|---|
| `virt_addr` | 音频帧虚拟内存地址，只读 |
| `timestamp` | 音频帧时间戳 |
| `size` | 音频帧大小 |
| **other member** 　| Audiobox应用编程无需关注 |

相关API接口：`audio_test_frame()` `audio_get_frame()` `audio_put_frame()`

----
## 7 abctrl调试工具
### 7.1 abctrl配置

如需在系统中包含abctrl调试工具，在SDK源码目录下执行命令`make menuconfig`进入配置界面选中`Testing abctrl`项

<pre>
QSDK Options --->
  |-Apps
  |-Libs
  |-Debug Tools
  |   |-[*] InfoTM Update Wrapper (IUW)
  |   <strong>|-[*] Testing menu</strong>
  |   |   |-[*] Testing gpio
  |   |   |-[*] Testing ccf
  |   |   |-[*] Testing delayline
  |   |   |-[*] Testing vbctrl
  |   |   <strong>|-[*] Testing abctrl</strong>
  |   |   |-[ ] ... ...
  |   |-[ ] App Demos (NEW)
</pre>

----
### 7.2 abctrl命令选项
abctrl命令第一个参数为测试子命令，如start/stop/list/record/play/mute/unmute/set等

| 子命令 | 命令选项 | 描述 |
|---|
| `start` |  | Start audiobox service |
| `stop` |  | Stop audiobox service |
| `list` |  | List all available channels with the information for that channel |
| `debug` |  | List debuging information about audio devices and channels |
| `record` |  | Record PCM audio from an input channel |
|         | '-c, --channel' | Specify input channel to record sound from |
|         | '-v, --volume' | Specify logic channel volume, (0~100) |
|         | '-o, --output' | Specify the output file |
|         | '--no-ref' | use audio_read_frame() instead of audio_get_frame() |
|         | '--enable-aec' | enable AEC algorithm to Cancel Echo when recording audio |
|         | '-w, --bit-width' | Specified bit width |
|         | '-s, --sample-rate' | Specified sampling rate |
|         | '-n, --channel-count' | Specified channel count |
|         | '-t, --time' | Specified record time |
| `play` |  |  |
|         | '-c, --channel' | Specify output channel to playback sound to |
|         | '-v, --volume' | Specify logic channel volume, (0~100) |
|         | '-d, --input' | Specify the input file |
|         | '-w, --bit-width' | Specified bit width |
|         | '-s, --sample-rate' | Specified sampling rate |
|         | '-n, --channel-count' | Specified channel count |
| `mute` |  |  |
|         | '-c, --channel' | Specify mute channel |
| `unmute` |  |  |
|         | '-c, --channel' | Specify unmute channel |
| `set` |  |  |
|         | '-c, --channel' | Specified channel name |
|         | '-w, --bit-width' | Specified bit width |
|         | '-s, --sample-rate' | Specified sampling rate |
|         | '-n, --channel-count' | Specified channel count |

在命令行下直接输入`abctrl`，可以打印出全部子命令和参数帮助信息

----
### 7.3 音频实时信息查询

执行命令`abctrl debug`，可以显示当前音频系统的实时运行状态信息，供分析定位音频问题使用

**场景范例：**
创建2个录音进程和3个放音进程，然后执行`abctrl debug`查看状态信息
```bash
/# abctrl play -v 99 -w 32 -s 16000 -n 2 -d /mnt/pcm_16khz_ch2_32b.wav &
/# abctrl record -v 88 -w 32 -s 16000 -t 120 -n 2 -o /mnt/ten_years.wav &
/# abctrl play -v 99 -w 32 -s 16000 -n 2 -d /mnt/pcm_16khz_ch2_32b.wav &
/# abctrl play -v 99 -w 32 -s 16000 -n 2 -d /mnt/pcm_16khz_ch2_32b.wav &
/# abctrl record -v 88 -w 32 -s 16000 -t 120 -n 2 -o /mnt/ten_years.wav &
/# abctrl debug

-----Audio DEV ATTR------------------------------------------------------------------------------------
DevName     DevDir  SampleRate  BitWidth  Channels  SampleSize  FrmTime  FrmRate  PCMMaxFrm
default_mic in      16000     	32        stereo    1024        62ms     16       4
default     out     16000     	32        stereo    1024        62ms     16       4

-----Audio DEV STATUS------------------------------------------------------------------------------------
DevName     DevDir  ChanCnt  bAec    bMute   MasterVol
default_mic in      2        N       N       88
default     out     3        N       N       99

-----Audio CHAN STATUS------------------------------------------------------------------------------------
ChnId   DevName     DevDir  Volume  Priority      Pid       TaskName
0       default_mic in      256     BACKGROUND    638       abctrl
2       default_mic in      256     BACKGROUND    641       abctrl
1       default     out     256     BACKGROUND    637       abctrl
3       default     out     256     BACKGROUND    640       abctrl
4       default     out     256     BACKGROUND    639       abctrl
```

**参数说明：**

|Audio DEV ATTR 参数| 描述|
|----|----|
|DevName| 音频设备名称，定义见[**5.3 audio_get_channel()**][5.3]函数参数`channel` |
|DevDir| 区分录音和放音，`in`:麦克风；`out`:扬声器 |
|SampleRate| 音频通道采样率 |
|BitWidth| 音频通道比特位宽 |
|Channels| 音频声道数，`mono`：单声道，`stereo`：双声道 |
|SampleSize| 音频通道每次需要存取的音频帧的采样个数 |
|FrmTime| 音频帧间隔时间。以 ms 为单位 |
|FrmRate| 每秒音频帧的个数 |
|PCMMaxFrm| PCM硬件设备缓冲区最大帧个数 |

|Audio DEV STATUS 参数| 描述|
|----|----|
|DevName| 音频设备名称，定义见[**5.3 audio_get_channel()**][5.3]函数参数`channel` |
|DevDir| 区分录音和放音，`in`:麦克风；`out`:扬声器 |
|ChanCnt| 基于指定音频设备创建的音频逻辑通道个数 |
|bAec| 音频设备AEC状态，`Y`:Aec开启，`N`:Aec关闭 |
|bMute| 音频设备静音状态，`Y`：静音，`N`：取消静音 |
|MasterVol| 音频设备master音量 |

|Audio CHAN STATUS 参数| 描述|
|----|----|
|ChnId| 音频逻辑通道号 |
|DevName| 音频设备名称，定义见[**5.3 audio_get_channel()**][5.3]函数参数`channel` |
|DevDir| 区分录音和放音，`in`:麦克风；`out`:扬声器 |
|Volume| 音频逻辑通道音量 |
|Priority| 放音专用，音频逻辑通道优先级，`FOREGROUND`:前景音，`BACKGROUND`:背景音 |
|Pid| 申请音频逻辑通道的进程PID |
|TaskName| 申请音频逻辑通道的进程名称 |

----
### 7.4 abctrl示例

**示例一**
```bash
abctrl start
```
启动Audiobox进程

**示例二**
```bash
abctrl stop
```
强制中止Audiobox进程

**示例三**
```bash
abctrl list
```
显示系统支持的音频设备列表，如`EVB`运行上述命令后显示：
```bash
**** List of Hardware Devices Pcminfo ****
Card 0:
playback:
ACCESS:  MMAP_INTERLEAVED RW_INTERLEAVED
FORMAT:  S16_LE S24_LE S32_LE
SUBFORMAT:  STD
SAMPLE_BITS: [16 32]
FRAME_BITS: [32 64]
CHANNELS: 2
RATE: [8000 192000]
PERIOD_TIME: (2666 256000]
PERIOD_SIZE: [512 2048]
PERIOD_BYTES: [4096 8192]
PERIODS: [2 4]
BUFFER_TIME: (5333 1024000]
BUFFER_SIZE: [1024 8192]
BUFFER_BYTES: [4096 65536]
TICK_TIME: ALL
capture:
ACCESS:  MMAP_INTERLEAVED RW_INTERLEAVED
FORMAT:  S16_LE S24_LE S32_LE
SUBFORMAT:  STD
SAMPLE_BITS: [16 32]
FRAME_BITS: [32 64]
CHANNELS: 2
RATE: [8000 192000]
PERIOD_TIME: (2666 256000]
PERIOD_SIZE: [512 2048]
PERIOD_BYTES: [4096 8192]
PERIODS: [2 4]
BUFFER_TIME: (5333 1024000]
BUFFER_SIZE: [1024 8192]
BUFFER_BYTES: [4096 65536]
TICK_TIME: ALL
```

**示例四**
```bash
abctrl record -o /mnt/samples.wav
```
使用本地麦克风录音，命令中只指定了最小参数集，未指定的参数使用缺省值：录音设备`default_mic`，音量`100`，使用引用方式读取音频帧，禁止AEC，`32`位宽采样，采样率`48000`，双声道，录音`10`秒，录音保存到`/mnt/samples.wav`文件。等同于下面命令：
```bash
abctrl record -c default_mic -v 100 -w 32 -s 48000 -n 2　-t 10 -o /mnt/samples.wav
```

**示例五**
```bash
abctrl record -c default_mic -v 100 --no-ref --enable-aec -w 32 -s 48000 -n 2　-t 12 -o /mnt/samples.wav
```
使用本地麦克风录音，命令中指定了所有参数：录音设备`default_mic`，音量`100`，使用`拷贝方式`读取音频帧，使能AEC，`32`位宽采样，采样率`48000`，双声道，录音`12`秒，录音保存到`/mnt/samples.wav`文件

**示例六**
```bash
abctrl record -c btcodecmbtcodecmicic -v 100 -w 16 -s 16000 -n 2　-t 24 -o /mnt/samples.wav
```
使用蓝牙设备麦克风录音：录音设备`btcodecmic`，音量`100`，使用`引用方式`读取音频帧，禁止AEC，`16`位宽采样，采样率`16000`，双声道，录音`24`秒，录音保存到`/mnt/samples.wav`文件

**示例七**
```bash
abctrl play -d /mnt/samples.wav
```
使用本地扬声器播放指定录音文件,命令中只指定了最小参数集，未指定的参数使用缺省值：播放设备`default`，音量`100`，`32`位宽采样，采样率`48000`，双声道，播放`/mnt/samples.wav`文件。等同于下面命令：
```bash
abctrl play -c default -v 100 -w 32 -s 48000 -n 2 -d /mnt/samples.wav
```

**示例八**
```bash
abctrl play -c btcodec -v 100 -w 16 -s 16000 -n 2 -d /mnt/samples.wav
```
使用蓝牙设备扬声器播放指定录音文件：播放设备`btcodec`，音量`100`，`16`位宽采样，采样率`16000`，双声道，播放`/mnt/samples.wav`文件

**示例九**
```bash
abctrl mute　-c default
```
本地扬声器设置静音：指定播放设备为`default`

**示例十**
```bash
abctrl unmute　-c btcodec
```
蓝牙设备扬声器取消静音：指定播放设备为`btcodec`

[5.3]:main.md#5.3_audio_get_channel()
[6.1]:main.md#6.1_channel_priority
[6.2]:main.md#6.2_audio_fmt_t
[6.3]:main.md#6.3_fr_buf_info
