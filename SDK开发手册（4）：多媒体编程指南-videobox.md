### 术语解释
| 术语 | 解释 |
| --- |---------|
| API | Application Programing Interface，应用程序接口 |
| AWB | Auto White Balance，自动白平衡 |
| AE | Auto Explosure，自动曝光 |
| AF | Auto Focus，自动聚焦 |
| 3A | AWB，AE，AF的统称 |
| IPU | Image Processing Unit，图像处理单元 |
| ISP | Image Signal Processing，图像信号处理。文中ISP对应InfoTM的图像处理硬件V2500或者V2505，反之亦然 |
| GOP | Group of Pictures，一组连续的画面 |
| CBR | Constant Bitrate，恒定码率，编码视频码率基本保持恒定 |
| VBR | Variable Bitrate，变码率，编码视频码率在相对大范围内波动 |
| PIP | Picture in Picture，画中画功能 |
| CSI | Camera Serial Interface，相机串行接口 |
| NAL | Network Abstraction Layer，网络抽象层，编码流的一种打包方式 |
| ROI | Region of Interest，感兴趣区域，用于编码时对特定区域做特别设置 |

## 1 摘要
本文对QSDK的多媒体功能做介绍，包括多媒体框架，各核心组件的功能、参数、接口以及应用方法等，可作为了解QSDK多媒体功能的学习资料，亦可作为多媒体开发人员的参考手册

----
## 2 系统概述
## 2.1 概述
InfoTM的多媒体软件开发平台旨在为应用软件的开发提供便捷。该平台对图像处理、编解码、运动侦测等复杂的功能做封装，开发者可通过JSON配置及调用平台提供的API接口完成需要的多媒体功能

支持功能

* 视频捕获

* H264/H265/MJPEG/JPEG编解码

* 视频输出显示

* 视频图像处理（包括2D/3D去噪、图像增强、图像锐化等）

* 编码视频水印叠加

* 视频侦测分析

* 音频捕获输出

* 音频编解码

----
## 2.2 系统架构

* Applications - 基于QSDK开发的应用软件系统，如图中app-launcher和app-d304

* Middleware - 基于硬件驱动开发的各种组件，对应用层提供相关功能API支持。Videobox主要是负责视频处理，Audiobox则是音频处理，Recorder负责录像功能，Player负责播放功能，SmartRC是自带的智能码率控制模块，RTSP则负责网络视频传输

* Drivers - 提供硬件驱动程序支持，包含多媒体硬件以及其它设备如Wifi，SDIO，USB，I2C等

![](image/arch/qsdk-arch.svg)


----
## 3 视频处理系统
## 3.1 概述
InfoTM视频处理系统的核心组件是Videobox，它是一套视频处理的框架，包含图像处理、编解码、移动侦测、显示等功能

----
## 3.2 Videobox框架

![](image/arch/videobox-arch.svg)

----
## 3.3 Videobox基础
### 3.3.1 简例
我们通过一个简单的示例，初览一下Videobox的使用。下图实现了从摄像头获取图像，经过H264编码，最后保存在文件的应用。其中，`"isp"`用于从摄像头抓取图像，`"vencoder"`用于H264编码，`"filesink"`则将编码后数据写入文件

![](image/path/basic.svg)

实现此功能仅需通过编写JSON文件，假定文件名为`path.json`，其内容如下

```json
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
                   "bind": { "h1264": "frame" }
               }
           }
       },
       "h1264":{
           "ipu":"vencoder",
           "args": {
               "encode_type": "h264"
           }
           "port":{
               "stream":{
               "bind":{ "filesink":"in" }
               }
           }
       },
       "filesink":{
           "ipu":"filesink",
           "args":{
               "data_path":"/tmp/sample.h264"
           }
       }
   }
```
在设备控制端执行下面命令，即可完成上述功能
```bash
vbctrl start path.json
```

----
### 3.3.2 IPU
IPU是图像处理单元（Image Process Unit）的缩写，是Videobox中最重要的概念。不同的IPU提供了不同的多媒体功能，用户可通过创建一系列的IPU，并把它们连接起来，让数据流在IPU之间传输，从而实现需要的应用。上例中`"isp"`、`"h1264"`、`"filesink"`即为三个IPU

----
### 3.3.3 端口（Port）
端口是IPU之间数据传送的接口，分为输入端口和输出端口。在IPU的模型中，红色端口为输出端口，蓝色端口为输入端口。IPU根据自身功能的不同，提供的端口个数也不一样，端口名也不一样。上例中`"port"`即为端口标示，`"out"`和`"stream"`为端口名，`"w"`、`"h"`、`"pixel_format"`、`"bind"`为端口属性

**端口的属性定义**

|属性|类型|描述|备注|
|----|----|----|----|
| `"w"` | int | 端口缓存宽度，单位像素 |见`"h"`|
| `"h"` | int | 端口缓存高度，单位像素 |对IPU `fodetv2`来说，`"w"`，`"h"`分别为区域坐标占用字节数和区域个数|
| `"pixel_format"` | string | 缓存数据格式，如NV12/RGB8888/BPP2/MV等 ||
| `"minfps"` | int | 输出帧率的最小值|仅对IPU `v2500`/`isplus`有效|
| `"maxfps"` | int | 输出帧率的最大值。当`"minfps"`与`"maxfps"`相等时 ，输出固定帧率|仅对IPU `v2500`/`isplus`有效 |
| `"pip_x"` | int | 截图或者PIP时，基于原图的起点x坐标 | 当配置输入端口时，表示裁剪功能；当配置输出端口时，绝大多数是PIP功能，唯一的例外是当配置是IPU  `v2500`/`isplus`的输出端口时，是裁剪功能 |
| `"pip_y"` | int | 截图或者PIP时，基于原图的起点y坐标 | 同`"pip_x"` |
| `"pip_w"` | int | 截图或者PIP时，截取或者叠加的画面宽度 | 同`"pip_x"` |
| `"pip_h"` | int | 截图或者PIP时，截取或者叠加的画面高度 | 同`"pip_x"` |
| `"lock_ratio"` | bool | 输出是否保持宽高比 | 目前仅用于IPU `v2500`/`isplus`的`cap`端口 |
| `"timeout"` | int | 端口缓存的读/写超时退出时间 | |
| `"bind"` | ipu_name：port_name | 绑定两个IPU的端口，此属性只在输出端口配置，而绑定的端口必须是输入端口 | |
| `"embezzle"` | ipu_name：port_name | 端口采用Embezzle模式，此属性只在输出端口配置，而绑定的端口必须是输出端口 | Embezzel模式时，当前端口不分配缓存，而是与绑定端口共享缓存 |

----
### 3.3.4 Path
IPU连接起来完成一个特定应用，这整个链路称为PATH。PATH可以用一个JSON文件来描述，如上例JSON文件就描述了由三个IPU完成的抓图、编码、存文件的Path

----
### 3.3.5 IPU在JSON中的描述
JSON文件中描述一个IPU包含几个部分，以`v2500`为例
```json
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
                "bind": { "h1264": "frame" }
            }
        }
    }
}
```
* `"isp"` - IPU实例名。用户可以自定义，每个IPU必须有，处于JSON文件第一层

* `"ipu"` - IPU的ID。如例中的`"v2500"`，处于第二层，每个IPU必须有，且值不能修改

* `"args"` - IPU参数描述。处于第二层，根据IPU的不同，参数名字、个数可以不一样。例子中配置`"skipframe"`为1，表示丢弃第一帧；`"nbuffers"`为3，表示采用3个缓存输出

* `"port"` - IPU端口描述。处于第二层，用于描述IPU的输出端口，是可选配置。相关属性可参见章节3.3.3

Note:  IPU的详细使用说明，可参见4 IPU列表

----
## 3.4 应用场景示例

本节主要展示一些常见场景及特殊应用示例，供使用者参考

----
### 3.4.1 双路编码

摄像头输入图像，同时编码两路H264视频，一路分辨率为1920x1080，一路分辨率为640x480。常见应用会将低分辨率视频经Wi-Fi传送预览
**模型**

![](image/path/dual-enc.svg)

**JSON描述**
```json
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
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "ispostv2": "in" }
            }
        }
    },
    "ispostv2": {
        "ipu": "ispostv2",
            "args": {
            "lc_grid_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080.bin",
            "lc_geometry_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080_geo.bin",
            "dn_enable": 1
        },
        "port": {
            "uo": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "enc_1080p": "frame" }
            },
            "ss0": {
                "w": 640,
                "h": 480,
                "pixel_format": "nv12",
                "bind": { "enc_vga": "frame" }
            }
        }
    },
    "enc_1080p":{
        "ipu":"vencoder",
        "args": {
            "encode_type": "h264"
        }
        "port":{
            "stream":{
            "bind":{ "filesink":"in" }
            }
        }
    },
    "enc_vga":{
        "ipu":"vencoder",
        "args": {
            "encode_type": "h264"
        }
    },
    "filesink":{
        "ipu":"filesink",
        "args":{
            "data_path":"/tmp/sample.h264"
        }
    }
}
```
**ispostv2参数说明**

* `"lc_grid_file_name1"` - 矫正文件位置，此文件用于镜头矫正（含鱼眼矫正）

* `"lc_geometry_file_name1"` - 矫正区域映射文件位置，可选配置。当矫正区域不是整张画面时，配置此文件，可提高矫正性能

* `"dn_enable"` - 使能3D降噪功能，1 - 使能；0 - 关闭

**获取640x480编码流应用代码示例**

```cpp
#include <qsdk/video.h>

struct fr_buf_info stFrameInfo;
// 从enc - vga的stream端口获取编码后的H264数据缓存
video_get_frame("enc_vga-stream", &stFrameInfo);

// 略

// 释放数据缓存
video_put_frame("enc_vga-stream", &stFrameInfo);
```

----
### 3.4.2 编码加拍照
摄像头输入图像，编码一路1080P视频并保存到文件，同时支持拍照功能
**模型**

![](image/path/enc-jpg.svg)

**JSON描述**
```json
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
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "ispostv2": "in" }
            },
            "cap": {
                "w": 1920,
                "h": 1088,
                "bind": {"h1jpeg":"in"}
            }
        }
    },
    "ispostv2": {
        "ipu": "ispostv2",
            "args": {
            "lc_grid_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080.bin",
            "lc_geometry_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080_geo.bin",
            "dn_enable": 1
        },
        "port": {
            "uo": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "enc_1080p": "frame" }
            }
        }
    },
    "enc_1080p":{
        "ipu":"vencoder",
        "args": {
            "encode_type": "h264"
        }
        "port":{
            "stream":{
            "bind":{ "filesink":"in" }
            }
        }
    },
    "h1jpeg": {
        "ipu": "h1jpeg",
        "args": {
            "mode": "trigger"
        }
    },
    "filesink":{
        "ipu":"filesink",
        "args":{
            "data_path":"/tmp/sample.h264"
        }
    }
}
```
**h1jpeg参数说明**

* `"mode"` - `"h1jpeg"`的工作模式，`"trigger"`时表示只有用户触发拍照时才会开始工作，其它时间处于休眠状态。这样可以节省系统资源

**拍照应用代码示例**

```cpp
#include <qsdk/video.h>

struct fr_buf_info stFrameInfo;

// Trigger "cap" port of isp to capture one frame
camera_snap_one_shot("isp-cap");
// 从h1jpeg的out端口获取编码后的数据缓存
video_get_snap("h1jpeg-out", &stFrameInfo);

// 略

// 释放数据缓存
video_put_snap("h1jpeg-out", &stFrameInfo);
```

----
### 3.4.3 图像添加水印
有些应用场景，如安防，需要在视频中添加日期，此时可以通过水印实现

![](image/path/marker-cn.png)
![](image/path/marker-date.png)

**模型**

![](image/path/marker-enc.svg)


**JSON描述**
```json
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
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "ispostv2": "in" }
            }
        }
    },
    "marker": {
        "ipu": "marker",
        "args": {
            "mode": "nv12"
        },
        "port": {
            "out": {
                "w": 800,
                "h": 64,
                "pixel_format": "nv12",
                "bind": { "ispost": "ov0" }
            }
        }
    },
    "ispostv2": {
        "ipu": "ispostv2",
            "args": {
            "lc_grid_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080.bin",
            "lc_geometry_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080_geo.bin",
            "dn_enable": 1
        },
        "port": {
            "ov0":{
                "pip_x":576,
                "pip_y":960
            },
            "uo": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "enc_1080p": "frame" }
            }
        }
    },
    "enc_1080p":{
        "ipu":"vencoder",
        "args": {
            "encode_type": "h264"
        }
        "port":{
            "stream":{
            "bind":{ "filesink":"in" }
            }
        }
    },
    "filesink":{
        "ipu":"filesink",
        "args":{
            "data_path":"/tmp/sample.h264"
        }
    }
}
```
**marker参数说明**

* `"mode"` - 叠加图层的数据格式。支持类型可参见附录A

**ispostv2参数说明**

* `"pip_x"` - 设置叠加图层位于输入图像的x坐标，以输入图像左上角为原点，单位为像素

* `"pip_y"` - 设置叠加图层位于输入图像的y坐标，以输入图像左上角为原点，单位为像素

**设置水印应用代码示例**

```cpp
#include <qsdk/marker.h>

// 字体属性初始化，包括字库、前景色、背景色
struct font_attr stFontAttr;
memset(&stFontAttr, 0, sizeof(struct font_attr));
sprintf((char *)stFontAttr.ttf_name, "simhei");
stFontAttr.font_color = 0x00ffffff;
stFontAttr.back_color = 0x20000000;

// 应用1：设置中文字符
marker_set_mode("marker", "manual", "%u%Y/%M/%D  %H:%m:%S", &stFontAttr);
marker_set_string("marker", "中文字符测试");

// 应用2：设置日期
marker_set_mode("marker", "auto", "%t%Y/%M/%D  %H:%m:%S", &stFontAttr);

// 应用3：设置UTC时钟
marker_set_mode("marker", "auto", "%u%Y/%M/%D  %H:%m:%S", &stFontAttr);

// 应用4：设置NV12数据
marker_set_picture("marker", "/mnt/sd0/nv12.yuv");
```

----
### 3.4.4 图像裁剪

图像裁剪是指将原始画面做剪裁，取原画面中的一部分作为有效数据，范围外的数据丢弃，常用于画面黑边的处理

Videobox的图像裁剪有两种方式，一种通过配置输出端口实现，对应IPU是`v2500`，另一种是通过配置输入端口实现，如`ispost`/`pp`/`h1jpeg`/`h1264`/`vencoder`等。前者做裁剪时，因`v2500`是视频数据的输出，故后续所有IPU获取的图像均为裁剪过的。而后面一种方式，只影响当前IPU及其后连接的IPU图像

下面分别对两种裁剪方式举例，重点关注两者配置方式的差异：一个配置输出端口，一个配置输入端口

#### **v2500做裁剪**

例子中，裁掉输入图像部分黑边

![](image/path/fish.jpg)
![](image/path/fish-cut.jpg)

**模型**

![](image/path/isp-crop.svg)

**JSON描述**
```json
{
    "isp": {
        "ipu": "v2500",
        "args": {
            "skipframe": 1,
            "nbuffers": 3
        },

        "port": {
            "out": {
                "w": 1080,
                "h": 1080,
                "pip_x": 420,
                "pip_y": 0,
                "pip_w": 1080,
                "pip_h": 1080,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
    },
    "display": { "ipu": "ids"}
}
```
**v2500参数说明**

* `"pip_x"` - 设置裁剪起点x坐标，以原图左上角为原点，单位为像素

* `"pip_y"` - 设置裁剪起点y坐标，以原图左上角为原点，单位为像素

* `"pip_w"` - 设置裁剪图像宽度

* `"pip_h"` - 设置裁剪图像高度

#### **pp做裁剪**

截取原图中一部分，并对截取部分做放大

![](image/path/zoom0.jpg)
![](image/path/zoom1.jpg)

**模型**

![](image/path/pp-crop.svg)

**JSON描述**
```json
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
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "pp": "in" }
            }
        }
    },

    "pp": {
        "ipu": "pp",
        "port": {
            "in": {
                "pip_x":32,
                "pip_y":32,
                "pip_w":640,
                "pip_h":480
            },
            "out": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
    },

    "display": { "ipu": "ids"}
}
```
`"pp"`在上例中主要实现了截图并放大的功能，通过配置输入端口的`"pip_x"`，`"pip_y"`，`"pip_w"`，`"pip_h"`做截图，配置输出端口`"w"`，`"h"`，将图像从640x480放大到1920x1080

----
### 3.4.5 Embezzle功能
Embezzle功能是将一个IPU输出端口的缓存，同时给另一个IPU的输出端口使用，这样可以将两个IPU的图像输出到相同的缓存，实现图像叠加的效果。典型应用场景是画中画功能

![](image/path/pip.jpg)

**模型**

![](image/path/embezzle.svg)

上例中，`"isp"`产生的是1080P大画面，`"pp"`是叠加画面，所以是`"pp"`从`"isp"`的输出端口取得缓存再做叠加，这个顺序非常重要，决定了那个端口是Embezzle的。如果Embezzle的顺序弄反，在这个示例下，小窗口图像将被大画面覆盖

**JSON描述**
```json
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
                "pixel_format": "nv12"
            }
        }
    },

    "dg-frame":{
        "ipu":"dg-frame",
        "port":{
            "out":{
                "w":640,
                "h":480,
                "pixel_format":"nv12",
                "bind":{"pp":"in"}
            }
        }
    },

    "pp": {
        "ipu": "pp",
        "port": {
            "out": {
                "pip_x":1248,
                "pip_y":576,
                "pip_w":640,
                "pip_h":480,
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "display": "osd0" },
                "embezzle":{"isp":"out"}
            }
        }
    },

    "display": { "ipu": "ids"}
}
```
**pp参数说明**

* `"embezzle"` -  `"pp"`的`"out"`端口采用`"embezzle"`模式，并且从`"isp"`的`"out"`端口获取缓存

Note:  例中`"pp"`的输出配置中有`"pip_x"`，`"pip_y"`，`"pip_w"`，`"pip_h"`，在这里表示放在缓存的位置而不是裁剪，`"pp"`的裁剪是放在**输入端口**配置。只有`"isp"`的裁剪是放在**输出端口**配置

----
### 3.4.6 软链接功能
软链接是`ispostv2`支持的功能，搭配`v2500`使用。打开此功能时，`ispostv2`的输入端口`in`与输出端口`uo`共享缓存，而`ispostv2`的输入端口`in`的缓存来源于`v2500`的`out`端口，这就相当于`ispostv2`的`out`端口缓存省掉了。这个功能主要解决某些平台内存紧张的问题。有利必有失，此功能打开后，`ispostv2`的鱼眼矫正及放大功能关闭

**模型**

![](image/path/softlink.svg)

**JSON描述**
```json
{
    "isp": {
        "ipu": "v2500",
        "args": {
            "nbuffers": 3,
            "flip": 0
        },
        "port": {
            "out": {
                "minfps": 25,
                "maxfps": 25,
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "ispost": "in" }
            },
            "his": {
                "bind": {
                    "vam": "in"
                }
            }
        }
    },
    "ispost": {
        "ipu": "ispostv2",
        "args": {
            "linkmode": 1,
            "linkbuffers": 3,
            "dn_enable": 1,
            "dn_target_index": 11,
            "lc_grid_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080.bin",
            "lc_geometry_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1080-1920x1080_geo.bin",
            "buffers": 2
        },
        "port": {
            "dn": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12"
            },
            "uo": {
                "w": 1920,
                "h": 1080,
                "pixel_format": "nv12",
                "bind": { "enc-1080p": "frame" }
            }
        }
    },
    "enc-1080p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h264",
            "buffer_size": 3145728
        }
    },
    "vam": {
        "ipu": "vamovement"
    }
}
```
**ispost参数说明**

* `"linkmode"` - 使能软连接功能

* `"linkbuffers"` - 软链接的缓存数量，必须与`"isp"`的`"out"`端口缓存个数一致

Note:  `"vam"`是运动检测模块，通过直方图信息检测运动区域，详情参见4 IPU列表

----
### 3.4.7 人脸侦测
人脸侦测主要用于判断图像中是否存在人脸，如果存在，则提供人脸位置信息。实现该功能的IPU是`fodetv2`，它支持画面中多个人脸的识别。人脸侦测主要用在安防监控领域，是人脸识别的前置功能
**模型**

![](image/path/fodet.svg)


**JSON描述**
```json
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
        "ipu": "ispostv2",
        "args": {
            "dn_enable":1,
            "dn_target_index":0,
            "lc_grid_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1088-1920x1088.bin",
            "lc_geometry_file_name1": "/root/.ispost/lc_v1_0-0-0-0_hermite32_1920x1088-1920x1088_geo.bin"
        },
        "port": {
            "dn": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12"
            },
            "ss0": {
                "w": 360,
                "h": 240,
                "pixel_format": "nv12",
                "bind": { "fodet": "in", "display":"osd0" }
            }
        }
    },

    "fodet": {
        "ipu": "fodetv2",
        "args": {
            "src_img_wd": 360,
            "src_img_ht": 240,
            "src_img_st": 360,
            "crop_x_ost": 0,
            "crop_y_ost": 0,
            "crop_wd": 360,
            "crop_ht": 240,
            "fd_db_file_name1": "/root/.fodet/fodet_db_haar.bin",
            "idx_start": 4,
            "idx_end": 18,
            "scan_win_wd": 20,
            "scan_win_ht": 20,
            "frame_idt":3
        },
        "port": {
            "ORout": {
                "w": 8,
                "h": 200
            }
        }
    },

    "display": { "ipu":"ids" }
}
```
**应用程序示例**
```cpp
#include <qsdk/event.h>
#include <qsdk/videobox.h>

// 人脸侦测事件处理函数
void EventFodetHandle(char *ps8Event, void *arg)
{
  	int i = 0;
  	struct event_fodet stEventFd;

  	if (!strncmp(ps8Event, EVENT_FODET, strlen(EVENT_FODET)))
  	{
  	    // 获取videobox提供的人脸坐标信息
  	  	memcpy(&stEventFd, (struct event_fodet *)arg, sizeof(struct event_fodet));

				// 根据坐标信息做处理。这里是将所有识别到的坐标打印出来
  	  	for(i = 0; i < stEventFd.num; i++)
  	  	{
            printf("num:%d, x:%d, y:%d, w:%d, h:%d\n", stEventFd.num, stEventFd.x[i], stEventFd.y[i], stEventFd.w[i], stEventFd.h[i]);
  	  	}
  	}
}

int main(void)
{
    // 注册人脸侦测事件的处理函数
    event_register_handler(EVENT_FODET, 0, EventFodetHandle);
    while(1)
    {
        usleep(20000);
    }
    return 0;
}
```

----
### 3.4.8 多路播放
同时播放多路视频流，将每一路的视频合成在一起显示，可以同时看多路画面。此功能可用于NVR回放
**模型**

![](image/path/dec-swc.svg)

例中，四路1080P的解码输出到`swc`，经过`swc`合成之后，成一路1080P输出

**JSON描述**
```json
{
    "dec0":{
        "ipu":"g1264",
        "port":{
            "frame":{
                "w": 1920,
                "h": 1080,
                "bind":{"swc0":"in0"}
            }
        }
     },
    "dec1":{
        "ipu":"g1264",
        "port":{
            "frame":{
                "w": 1920,
                "h": 1080,
                "bind":{"swc0":"in1"}
            }
        }
     },
    "dec2":{
        "ipu":"g1264",
        "port":{
            "frame":{
                "w": 1920,
                "h": 1080,
                "bind":{"swc0":"in2"}
            }
        }
     },
    "dec3":{
        "ipu":"g1264",
        "port":{
            "frame":{
                "w": 1920,
                "h": 1080,
                "bind":{"swc0":"in3"}
            }
        }
     },
    "swc0": {
        "ipu":"swc",
        "args":{
            "w":1920,
            "h":1088
        },
        "port": {
            "out": {
                "pixel_format": "nv12",
                "bind": { "display": "osd0" }
            }
        }
    },

    "display": {
        "ipu": "ids",
        "args": {
            "no_osd1": 1
        }
    }
}
```

----
## 3.5 Videobox配置
Videobox的IPU以及IPU支持的功能，可通过菜单配置

在QSDK开发包的根目录下，输入
```bash
make menuconfig
```
弹出菜单，选择**QSDK Options>Apps>Videobox>Select IPUs**，选择需要支持的IPU
```bash
[ ] Optimize boot speed                          
[*] DGPixel                                      
[*] DGFrame                                      
[*] DGMath                                       
[ ] DGOV                                         
[*] G1264                                        
[ ] G1JDec                                       
[ ] G2                                           
[*] H1JPEG                                       
[ ]   Software scale feature for image scaling   
[ ]   Software crop feature for image croping    
[ ] H1264                                        
[ ] Jenc                                         
[*] FFVdec                                       
[ ] H2                                           
[ ] H2V4                                         
[*] Video Encoder
[ ]   smartrc
[ ]   h1264
      h2 encoder lib (h2v4)  --->
          ( ) h2
          (X) h2v4
          ( ) NONE
[*] IDS
[ ] ISPOSTv1                                     
[*] ISPOSTv2                                     
[ ]   Dynamic create fisheye correction grid data
[ ] FODETv2                                      
[ ] PP                                           
[ ] V2500                                        
[*] V2505                                        
[*]   Hardware AWB method                        
[*] CamV4L2                                      
[*] FileSink                                     
[ ] FileSource                                   
[*] VAMovement                                   
[ ] MVMovement                                   
[*] VAQRScanner      
[*] Marker                                       
[ ]   Using freetype lib to generate maker data  
[*] Softlayer                                    
[ ] Bad encoder(developing)                      
[ ] Two fisheyes stitcher                        
[*] FFPhoto                                      
[ ]   Enable software scale                      
[ ] SoftwareComposer                             
[ ] ISPlus                                       
[ ] Bufsync        
```

IPU功能配置项

|IPU|配置|说明|备注|
|---|
|`H1JPEG`|支持软件裁剪和放大功能|用于裁剪和放大输入的YUV图像||
|`ISPOSTv1`，`ISPOSTv2`|支持动态创建/加载鱼眼矫正数据参数|用于矫正模式的切换||
|`V2505`|使用其它硬件提供的AWB信息，替代`V2505`提供的|用于解决大色块在暗环境下偏色问题|仅适用`Apollo-2`，`Apollo-ECO`必须关掉|
|`Marker`|支持FreeType字库|不使用FreeType时，会采用点阵字库，可减小镜像大小||
|`FFPhoto`|使能软件缩放功能|用于对解码后的图像做缩放|||
|`Video Encoder`|选择H264或H265硬件编码|配置当前硬件编码支持的类型|QSDK V2.3.0启用，`h1264`仅适用`Apollo`，`Apollo-2`，`h2`仅适用`Apollo`，`h2v4`仅适用`Apollo-ECO`|

---
## 4 IPU列表
### 4.1 v2500
**功能描述**
图像处理核心模块，处理从摄像头输入的图像，包含黑度矫正、镜头阴影矫正、2D降噪、坏点矫正、色相差矫正、色彩插值、色彩矫正、伽马矫正、图像锐化以及3A等功能

**模型**

![](image/ipu/v2500.svg)

**端口说明**

* `out` - 输出处理完成的数据，支持数据类型NV12/NV21
* `his` - 输出通路0的直方图信息，可用于移动侦测
* `his1` - 输出通路1的直方图信息，可用于移动侦测
* `cap` - 输出处理完成的数据，支持数据类型NV12/NV21。常用于大分辨率拍照
* `dsc` - 输出处理完成的数据，支持数据类型为RGBA8888，用于图像辅助输出

**参数说明**

* `nbuffers`（可选） -  `out`端口使用`nbuffers`个缓存做输出，缓存越多输出越平稳。缺省值为3，低于3个输出帧率会达不到设定帧率
* `skipframe`（可选） - `v2500`启动后，丢弃`skipframe`个帧再输出。因为一开始AWB及AE还不稳定，画面偏紫，可通过配置此参数丢弃部分画面
* `flip`（可选） - 对输入画面做镜像处理，0 - 不翻转；1 - 水平翻转；2 - 垂直翻转；3 - 水平垂直都翻转。缺省值为0

**使用限制**

* `Apollo`，`Apollo-2`，`Apollo-ECO`
*  端口`his1`只用于`Apollo` ISP同时输出两路视频时通路1的直方图信息。`Apollo-2`, `Apollo-ECO` ISP没有通路1的输出

----
### 4.2 isplus （Beta版）
**功能描述**
含`v2500`功能，同时支持3D降噪

**模型**

![](image/ipu/isplus.svg)

**端口说明**
同v2500

**参数说明**
同v2500

**使用限制**

* `Apollo-2`，`Apollo-ECO`
* 不能和`ispost`/`ispostv2`一起使用
* 输出帧率为设定帧率一半

----
### 4.3 ispostv1
**功能描述**
图像后处理模块，主要支持镜头矫正（含鱼眼矫正）、3D降噪、图像旋转、图层叠加及图像缩放等功能

**模型**

![](image/ipu/ispost.svg)

**端口说明**

* `in` -  图像输入端口，支持数据类型NV12/NV21
* `ol` -  图像叠加层输入端口，支持数据格式NV12/NV21，支持Alpha叠加
* `dn` -  输出矫正完成后图像，支持数据类型NV12/NV21，图像会同时用于3D降噪
* `ss0` - 输出矫正完成后，再降尺寸图像，支持数据类型NV12/NV21
* `ss1` - 同ss0，分辨率可以不同

**参数说明**

* `nbuffers`（可选） - 每个输出端口使用的缓存个数，范围[2，19]，缺省值为3
* `lc_grid_file_name1`（**必填**） - 镜头矫正（含鱼眼矫正）文件路径（含文件名）
* `lc_enable`（可选） - 镜头矫正使能开关。1 - 使能，0 - 关闭。缺省值为1
* `lc_grid_target_index`（可选） - 一个镜头矫正文件中可能包含多组矫正数据，通过配置这个值，选择特定矫正数据。缺省值为0
* `dn_enable`（可选） - 3D降噪功能使能开关，1 - 使能，0 - 关闭。缺省值为0
* `dn_target_index`（可选） - 3D降噪参数选择，取值范围[0，11]，默认值为0。值越大，降噪强度越大，强度过大时运动场景容易有拖影
* `fcefile`（可选） - 多视图模式的JSON配置文件路径（含文件名），用于支持多视图模式的动态切换。例如JSON中可以包含二分屏和全景展开两种视图模式，用户可通过应用接口在两种模式之间切换

**使用限制**

* `Apollo`
* 最大输入分辨率4096x4096
* `ss0`,`ss1`输出分辨率的宽度不可超过1280
* `ov0`输入要求图像宽度8像素对齐，宽度不小于48，高度不小于32

----
### 4.4 ispostv2
**功能描述**
图像后处理模块，主要支持镜头矫正（含鱼眼矫正）、电子防抖、3D降噪、图像旋转、图层叠加及图像缩放等功能

**模型**

![](image/ipu/ispostv2.svg)

**端口说明**

   * `in` - 图像输入端口，支持数据类型NV12/NV21
   * `ov0` - 图像叠加层输入端口0，支持数据格式NV12/NV21
   * `ov1` - 图像叠加层输入端口1，支持数据格式BPP2
   * `ov2` - 图像叠加层输入端口2，支持数据格式BPP2
   * `ov3` - 图像叠加层输入端口3，支持数据格式BPP2
   * `ov4` - 图像叠加层输入端口4，支持数据格式BPP2
   * `ov5` - 图像叠加层输入端口5，支持数据格式BPP2
   * `ov6` - 图像叠加层输入端口6，支持数据格式BPP2
   * `ov7` - 图像叠加层输入端口7，支持数据格式BPP2
   * `uo` - 输出矫正完成后图像，支持数据类型NV12/NV21，图像会同时用于3D降噪
   * `ss0` - 输出矫正完成后，再降尺寸图像，支持数据类型NV12/NV21
   * `ss1` - 同`ss0`，分辨率可以不同

**参数说明**

   * `nbuffers`（可选） - 每个输出端口使用的缓存个数，范围[2,19]，缺省值为3
   * `fisheye`（可选） - 使用鱼眼矫正时，需要设为1，否则为0
   * `lc_grid_file_name1`（**必填**） - 镜头矫正（含鱼眼矫正）文件路径（含文件名）
   * `lc_geometry_file_name1`（可选） - 镜头矫正映射文件路径（含文件名），可用于快速查找矫正点的其实坐标，当`lc_grid_line_buf_enable`使能时，必须配置
   * `lc_enable`（可选） - 镜头矫正使能开关。1 - 使能，0 - 关闭。缺省值为1
   * `lc_grid_target_index`（可选） - 一个镜头矫正文件中可能包含多组矫正数据，通过配置这个值，选择特定矫正数据。缺省值为0
   * `lc_grid_line_buf_enable`（可选） - 行缓存使能开关。1 - 使能，0 - 关闭。使能此功能后，做鱼眼矫正时，若只展开画面一部分，可提升矫正性能。此时， `lc_geometry_file_name1` 必须配置，且要配置正确，否则会发生未知错误。若矫正的是完整图像，可以关闭此开关，同时拿掉  `lc_geometry_file_name1` 指向的文件以减少系统镜像大小
   * `dn_target_index`（可选） - 3D降噪参数选择，取值范围[0，11]，默认值为0。值越大，降噪强度越大，强度过大时运动场景容易有拖影
   * `dn_enable`（可选） - 3D降噪功能使能开关，1 - 使能，0 - 关闭。缺省值为0
   * `linkmode`（可选） - 软连接使能开关。1 - 使能，0 - 关闭。仅在作为`v2500`输出IPU时使用。缺省值为0
   * `linkbuffers`（可选） - 软连接打开时，必须配置此参数，其值等于其输入端口的缓存数
   * `fcefile`（可选） - 多视图模式的JSON配置文件路径（含文件名），用于支持多视图模式的动态切换。例如JSON中可以包含二分屏和全景展开两种视图模式，用户可通过应用接口在两种模式之间切换
   * `vs_enable`（可选） - 使用电子防抖功能时，需要设为1， 否则为0，缺省值为0

**使用限制**

   * `Apollo-2`，`Apollo-ECO`
   * `ss0`,`ss1`输出分辨率的宽度需小于等于1280
   * `ss0`,`ss1`的输出分辨率要小于`uo`或`dn`
   * `ov0`输入要求图像宽度8对齐，宽度不小于48，高度不小于32
   * `ov1`-`ov7`输入图像宽度要求32对齐，宽度不小于160，高度不小于32

Note: 电子防抖功能开启时（设ispostv2输入分辨率的宽W和高H），ispostv2将会多分配2个缓存，分配大小为（2xWxHx1.5）, 配置ispostv2的`dn`和`uo` 输出分辨率时，宽为（W-2x64）和高为（H-2x64），例如：ispostv2需要输出辨率为1920x1080，即`uo`和`dn`输出分辨率为1920x1080，那么ispostv2需要输入分辨率为2048x1208

----
### 4.5 h1
**功能描述**
H264编码模块

**模型**

 ![](image/ipu/h1264.svg)

**端口说明**

   * `frame` - 输入端口，支持数据类型NV12/NV21/RGB565/RGBA8888
   * `stream` - 输出编码后的H264数据
   * `mv` - 输出编码数据的运动向量信息，数据类型MV

**参数说明**

   * `buffer_size`（可选） - 编码输出缓存大小，与`min_buffernum`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值为2.25MB
   * `min_buffernum`（可选） - 编码输出缓存最小张数，必须大于等于3，与`buffer_size`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值为3
   * `framerate`（可选） - 编码目标帧率分子，取值大于0
   * `framebase`（可选） - 编码目标帧率分母，取值大于0，目标帧率计算等于`framerate`/`framebase`。如果前端输入帧率和目标帧率不一致，则编码模块会做帧率转换。目前输入30帧或者25帧时，目标帧率可转到低于30或者25的任意帧率。当输入帧率不是30或者25时，只能做倍数的转换。不配置时目标帧率等于输入帧率
   * `rotate`（可选） - 编码前画面旋转配置。`"90R"` - 向右旋转90度，`"90L"` - 向左旋转90度，缺省值为不旋转
   * `smartrc`（可选） - SmartRC功能的使能开关，1表示使能，缺省值为0。SmartRC详情可参见[码率控制用户手册]()
   * `refmode`（可选） - 使能H264跳帧参考编码开关。0 - 关闭, 1 - 使能。跳帧参考使能后编码后的数据最多可以丢弃一半的帧而不影响解码
   * `rc_mode`（可选） - 编码码率控制模式。0 - CBR；1 - VBR；2 - FixQp，缺省值为0
   * `gop_length`（可选） - 编码GOP长度，是码率控制的基本单元。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `idr_interval`（可选） - 编码时I帧间隔。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `qp_max`（可选） - 编码QP的最大值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_min`（可选） - 编码QP的最小值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_delta`（可选） - 调整I帧QP，取值范围[-12，12]，用于调整I帧的相对质量或是I帧占比。设置`rc_mode`为0或1时有效
   * `bitrate`（可选） - CBR编码目标码率，取值范围[10000，60000000]，单位bps。需设置`rc_mode`才生效
   * `qp_hdr`（可选） - CBR编码起始QP值，取值范围[2，51]。设置为 - 1时，由编码算法自动选择。设置`rc_mode`为0时有效
   * `max_bitrate`（可选） - VBR编码时最大码率，取值范围[10000，60000000]，单位bps。设置`rc_mode`为1时有效
   * `threshold`（可选） - 限制VBR编码时码率下限，下限值为`max_bitrate` * `threshold` / 100，取值范围[1，99]。设置`rc_mode`为1时有效
   * `qp_fix`（可选） - FixQp模式下，采用的QP值，取值范围[2，51]。设置`rc_mode`为2时有效
   * `mv_w`（可选） - 编码图像宽度。在连接`mvmovement`时必须配置
   * `mv_h`（可选） - 编码图像高度。在连接`mvmovement`时必须配置
   * `enable_longterm`（可选）: 使能长期参考帧，0 - 关闭, 1 - 使能。长期参考帧主要用于背景帧的参考。在背景长时间固定的情况下，可以提高压缩率
   * `chroma_qp_offset`（可选）: 调整编码时色度相对于亮度的QP差值，取值范围[-12，12]。为负时增强色度处理，可以降低颜色的拖影和块现象。为正时降低色度处理，绝对值越大，强度越高
   * `cyclic_intra_refresh` （可选） - 消除除第一个I帧外所有的I帧，把这些I帧中的I宏块平均分散到其所在GOP的其他P帧中; 1为开启，0为关闭，默认关闭

**使用限制**

   * `Apollo`，`Apollo-2`
   * 使能`enable_longterm`时，需要额外多一张帧缓存

Note:  对应IPU功能列表中的`H1264`，QSDK V2.3.0及之后版本使用`Video Encoder`选中配置`h1264`

----
### 4.6 h2
**功能描述**
H265编码模块

**模型**

![](image/ipu/h2.svg)

**端口说明**

   * `frame` - 输入端口，支持数据类型NV12/NV21/RGB565/RGBA8888
   * `stream` - 输出编码之后的H265数据

**参数说明**

   * `buffer_size`（可选） - 编码输出缓存大小，与`min_buffernum`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值为3MB
   * `min_buffernum`（可选） - 编码输出缓存最小张数，必须大于等于3，与`buffer_size`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值为3
   * `framerate`（可选） - 编码目标帧率分子，取值大于0
   * `framebase`（可选） - 编码目标帧率分母，取值大于0，目标帧率计算等于`framerate`/`framebase`。如果前端输入帧率和目标帧率不一致，则编码模块会做帧率转换。目前输入30帧或者25帧时，目标帧率可转到低于30或者25的任意帧率。当输入帧率不是30或者25时，只能做倍数的转换。不配置时目标帧率等于输入帧率
   * `rotate`（可选） - 编码前画面旋转配置。`"90R"` - 向右旋转90度，`"90L"` - 向左旋转90度，缺省值为不旋转
   * `smartrc`（可选） - SmartRC功能的使能开关，1表示使能，缺省值为0。SmartRC详情可参见[码率控制用户手册]()
   * `refmode`（可选） - 设置H265时域分级编码模式。0 - 关闭, 1 - 分两级, 2 - 分三级。时域分级编码可以在带宽或解码性能不足时丢弃高层及的帧而不影响解码
   * `rc_mode`（可选） - 编码码率控制模式。0 - CBR；1 - VBR；2 - FixQp，缺省值为0
   * `gop_length`（可选） - 编码GOP长度，是码率控制的基本单元。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `idr_interval`（可选） - 编码时I帧间隔。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `qp_max`（可选） - 编码QP的最大值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_min`（可选） - 编码QP的最小值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_delta`（可选） - 调整I帧QP，取值范围[-12，12]，用于调整I帧的相对质量或是I帧占比。设置`rc_mode`为0或1时有效
   * `bitrate`（可选） - CBR编码目标码率，取值范围[10000，60000000]，单位bps。需设置`rc_mode`才生效
   * `qp_hdr`（可选） - CBR编码起始QP值，取值范围[2，51]。设置为 - 1时，由编码算法自动选择。设置`rc_mode`为0时有效
   * `max_bitrate`（可选） - VBR编码时最大码率，取值范围[10000，60000000]，单位bps。设置`rc_mode`为1时有效
   * `threshold`（可选） - 限制VBR编码时码率下限，下限值为`max_bitrate` * `threshold` / 100，取值范围[1，99]。设置`rc_mode`为1时有效
   * `qp_fix`（可选） - FixQp模式下，采用的QP值，取值范围[2，51]。设置`rc_mode`为2时有效
   * `chroma_qp_offset`（可选）: 调整编码时色度相对于亮度的QP差值，取值范围[-12，12]。为负时增强色度处理，可以降低颜色的拖影和块现象。为正时降低色度处理，绝对值越大，强度越高
   * `cyclic_intra_refresh` （可选） - 消除除第一个I帧外所有的I帧，把这些I帧中的I宏块平均分散到其所在GOP的其他P帧中; 1为开启，0为关闭，默认关闭

**使用限制**

   * `Apollo`，`Apollo-ECO`

Note: 对应IPU功能列表中的`H2`或`H2V4`，QSDK V2.3.0及之后版本使用`Video Encoder`选中配置`h2 encode lib`中的`h2`或者`h2v4`

----
### 4.7 g1
**功能描述**
H264硬件解码模块

**模型**

![](image/ipu/g1264.svg)

**端口说明**

   * `stream` - 输入端口，支持数据类型H264基本流
   * `frame` - 输出端口，支持数据类型NV12

**参数说明**

   * `pp_combine`（可选） - 配置成1，表示将采用`g1`+`pp`联合工作模式
   * `rotate`（可选） - 配置解码后旋转方向，`g1`+`pp`联合工作模式时生效，`“90”` - 画面向右旋转90度；`“180”` - 画面向右旋转180度；`“270”` - 画面向右旋转270度；`“H”` - 画面水平翻转；`“V”` - 画面垂直翻转

**使用限制**

   * `Apollo`

----
### 4.8 g2
**功能描述**
 H265硬件解码模块

**模型**

 ![](image/ipu/g2.svg)

**端口说明**

* `stream` - 输入端口，支持数据类型H265基本流
* `frame` - 输出端口，支持数据类型NV12

**参数说明**
无
**使用限制**

* `Apollo`

----
### 4.9 h1jpeg
**功能描述**
图片编码模块，支持JPEG编码

**模型**

![](image/ipu/h1jpeg.svg)

**端口说明**

   * `in` - 输入端口，支持数据类型NV12
   * `out` - 输出端口，支持数据类型JPEG

**参数说明**

   * `mode`（可选） - 配置成`"trigger"`，表示需要触发编码，不触发时处理休眠状态。配置其它任意值，编码器会连续编码。缺省值:`"trigger"`
   * `rotate`（可选） - 配置图片旋转方向，`"90R"` - 向右旋转90度，`"90L"` - 向左旋转90度，缺省值为不旋转
   * `buffer_size`（可选） - 配置输出端口的缓冲大小，单位字节。未设置时，IPU内部会分配适当的大小
   * `use_backup_buf`（可选） - IPU是否使用备份缓存。1 - 使用，0 - 不使用，缺省值为0。当需要对编码缓存做比较耗时的处理时（比如插值），需要用备份缓存，防止出现图像断层问题
   * `thumbnail_mode`（可选） - 配置缩略图模式，`"intra"`时，表示缩略图放在图片内，`"extra"`时，表示缩略图为独立图片。缺省值为没有缩略图
   * `thumbnail_width`（可选） - 缩略图的宽度，不小于96且是16的倍数，缺省值为128
   * `thumbnail_height`（可选） - 缩略图的高度，不小于32且是2的倍数, 缺省值为96

**使用限制**

   * `Apollo`，`Apollo-2`
   * 支持最大分辨率：8192x8192
   * 支持最小分辨率：96x32
   * 输入数据宽度(stride)必须是16的倍数，高度必须是2的倍数

----
### 4.10 jenc
**功能描述**
图片编码模块，支持JPEG编码

**模型**

![](image/ipu/jenc.svg)

**端口说明**

   * `in` - 输入端口，支持数据类型NV12
   * `out` - 输出端口，支持数据类型JPEG

**参数说明**

   * `mode`（可选） - 配置成`"trigger"`，表示需要触发编码，不触发时处理休眠状态。配置其它任意值，编码器会连续编码。缺省值`"trigger"`
   * `buffer_size`（可选） - 配置输出端口的缓冲大小，单位字节。未设置时，IPU内部会分配适当的大小

**使用限制**

   * `Apollo-ECO`
   * 支持最大分辨率：8176x8176
   * 支持最小分辨率：96x32
   * 输入数据宽度(stride)必须是16的倍数，高度必须是2的倍数

----
### 4.11 ffvdec
**功能描述**
视频软解码模块，采用FFmpeg解码，需要开启对应解码支持库。目前支持H264和H265解码，默认为H264解码

**模型**

![](image/ipu/ffvdec.svg)

**端口说明**

   * `stream` - 输入端口，支持数据类型H264基本流
   * `frame` - 输出端口，支持数据类型NV12/ARGB8888

**参数说明**

   * `h265`（可选） - 解码H265码流时需要配置这个参数为1。默认不配置为H264解码

**使用限制**
无

----
### 4.12 ffphoto
**功能描述**
图片软解码模块，采用FFmpeg解码。目前只支持JPEG解码
**模型**

![](image/ipu/ffphoto.svg)

**端口说明**

   * `stream`  -  输入端口，支持数据类型JPEG数据
   * `frame`  -  输出端口，支持数据类型NV12/ARGB8888

**参数说明**
无
**使用限制**
无

 -  -  -
### 4.13 pp
**功能描述**
用于图像后处理。可支持图像剪裁、旋转、颜色空间转换、图像放大/缩小、图像叠加、去交错等功能

**模型**

![](image/ipu/pp.svg)

**端口说明**

   * `in`  -  输入端口，支持数据类型NV12
   * `ol0`  -  叠加图像输入端口0，支持数据类型RGBA8888
   * `ol1` - 叠加图像输入端口1，支持数据类型RGBA8888
   * `out` - 输出端口，支持数据类型NV12/RGBA8888（`ol0`或者`ol1`使能时必须是RGBA8888）

**参数说明**

   * `rotate`（可选） - 输入画面旋转配置。`“90”` - 画面向右旋转90度；`“180”` - 画面向右旋转180度；`“270”` - 画面向右旋转270度；`“H”` - 画面水平翻转；`“V”` - 画面垂直翻转。需要注意的是，旋转只针对输入图像，叠加图像不受影响

**使用限制**

   * `Apollo`
   * 最大输出宽度 -  3*input_width
   * 最大输出高度: 3*input_height  -  2
   * 最小输出宽度/高度 -  1/70
   * 不支持宽度放大，高度缩小，反之亦然
   * 支持宽高放大不同倍数，或者缩小不同倍数
   * 输入宽高16像素对齐
   * 裁剪起始坐标16像素对齐，裁剪宽高8像素对齐
   * 输出图像宽8像素对齐，高2像素对齐

----
### 4.14 marker
**功能描述**
主要用于在原始图像中添加水印，目前能支持输出NV12/BPP2/RGBA8888三种格式水印，其中BPP2是通过两个比特来索引调色板，这种情况下只支持四种颜色，常用于隐私区域遮挡。NV12和RGBA8888是全色彩水印，叠加内容可以更加丰富，可以是图像，时间信息等

**模型**

 ![](image/ipu/marker.svg)

**端口说明**

   * `out` - 输出端口，支持数据类型NV12/BPP2/RGBA8888

**参数说明**

   * `mode`（可选） - 水印模式。可配置3种值：normal，nv12，bit2。normal时，marker输出到softlayer，nv12和bit2则分别对应NV12和BPP2的输出。默认nv12
   <font color=red>// 接口需要优化 </font>

**使用限制**

NOTE: marker的字库支持有`Freetype mode`、`Iconv mode`和`Grid data mode`三种方式（menuconfig中进行配置）。当配置为选择`Grid data mode`时，由于不用加载外部字库，所以启动速度会更快。但是，由于它是通过在文件中直接定义字符的位图，所以支持的字符是有限制的。目前支持的字符为： `0`，`１`，`２`，`３`，`４`，`５`，`６`，`７`，`８`，`９`，`:`，`/`，` `。

----
### 4.15 ids
**功能描述**
显示模块，主要用于屏幕显示。支持缩放功能

**模型**

 ![](image/ipu/ids.svg)

**端口说明**

   * `osd0` - 输入端口，支持数据类型NV12/NV21/ARGB8888
   * `osd1` - 输入端口，暂不支持

**参数说明**

   * `no_osd1`（可选） - 是否关闭Color key。配置成1 - 关闭，0 - 不关闭。缺省值为关闭

**使用限制**
无

----
### 4.16 bufsync
**功能描述**
缓存同步模块。在v2500同一个输出端口，接两个IPU时，可通过此IPU桥接，保证缓存的同步使用，防止画面断层

**模型**

 ![](image/ipu/bufsync.svg)

**端口说明**

   * `in` - 输入端口，数据类型无限制
   * `out0` - 输出端口0，数据类型与输入一致
   * `out1` - 输出端口1，数据类型与输入一致

**参数说明**
无
**使用限制**
无

----
### 4.17 dg-frame
**功能描述**
用于产生YUV格式帧数据，常用在调试及性能测试中

**模型**

 ![](image/ipu/dg-frame.svg)

**端口说明**

   * `out` - 输出端口，支持数据类型NV12

**参数说明**

   * `nbuffers`（可选） -  输出端口的缓冲数量
   * `data`（可选） - 配置文件路径（含文件名），当mode为picture或者yuvfile时必须填
   * `mode`（可选） - 共三种配置，vacant - 缓冲内容为0；picture - 从文件中读取一帧数据；yuvfile - 读取文件中所有帧。缺省时，由IPU内部产生默认帧

**使用限制**
无

----
### 4.18 fodetv2
**功能描述**
人脸侦测模块，对输入图像做人脸侦测，将找到的人脸坐标，以事件的形式通知到应用

**模型**

 ![](image/ipu/fodetv2.svg)

**端口说明**

   * `in` - 输入端口，支持数据类型NV12
   * `ORout` - 输出端口，输出侦测到的人脸区域坐标[x，y，w，h]，最大支持200个区域。配置端口属性`"w"`和`"h"`时，`"w"`表示坐标占用字节数，这里需要填16，`"h"`表示区域个数

**参数说明**

   * `src_img_wd`（可选）: 输入图像宽度，取值范围[18，320]，单位像素。缺省值320
   * `src_img_ht`（可选）: 输入图像高度，取值范围[18，240]，单位像素。缺省值240
   * `src_img_st`（可选）: 输入图像一行像素的亮度值占用内存字节数
   * `crop_x_ost`（可选）: 输入图像扫描的起始x坐标，原点是图像左上角
   * `crop_y_ost`（可选）: 输入图像扫描的起始y坐标，原点是图像左上角
   * `crop_wd`（可选）: 扫描图像宽度，不超过输入图像宽度
   * `crop_ht`（可选）: 扫描图像高度，不超过输入图像高度
   * `fd_db_file_name1`（可选）: 人脸资料库位置，需包含路径及文件名
   * `idx_start`（可选）: 扫描窗口的起始阶数，影响人脸侦测的最小尺寸。为0时，最小尺寸为18x18，每增加1，宽高扩大1.1倍。缺省值为0
   * `idx_end`（可选）: 扫描窗口的结束阶数，影响人脸侦测的最大尺寸。缺省值为24
   * `scan_win_wd`（可选）: 扫描窗口的宽度，取值范围[18，输入图像宽度]
   * `scan_win_ht`（可选）: 扫描窗口的高度，取值范围[18，输入图像高度]
   * `frame_idt`（可选）: 检测频率。每frame_idt帧检测一次，缺省值为5
   * `mode`（可选）: 分为两种模式，`trigger`模式只在调用`va_fodet_trigger()`接口时才检测人脸，`auto`模式会一直检测人脸。缺省值为`auto`

**使用限制**

   * 侧脸以及脸部有遮挡准确率降低

----
### 4.19 vamovement
**功能描述**
视频分析模块，用于运动检测。此模块通过直方图信息判断运动状况，需配合`v2500`一起使用

**模型**

 ![](image/ipu/vamovement.svg)

**端口说明**

   * `in` - 直方图信息输入，需连接v2500的his端口

**参数说明**

   * `send_blk_info`（可选） - 使能块区域运动汇报。1 - 使能，0 - 关闭，缺省值为0。输入画面平均分割成7x7个区域，使能此功能时，会将每个运动区域的坐标信息发送给应用，关闭此功能时，画面运动时，会发信息给应用，但没有具体运动区域坐标

**使用限制**

   * 必须配合`v2500`一起使用

----
### 4.20 mvmovement（Beta版）
**功能描述**
视频分析模块，用于运动检测。此模块通过运动向量信息判断运动状况，需要配合h1264或者vencoder（"args"中的"encode_type"必须是"h264"）一起使用。支持宏块级（16x16）的运动检测，可用于区域报警

**模型**

 ![](image/ipu/mvmovement.svg)

**端口说明**

   * `in` - 运动向量输入，需连接`h1264`或`vencoder`的`mv`输出端口

**参数说明**
无
**使用限制**

   * `Apollo-2`
   * 必须配合`h1264`或`vencoder`一起使用，配合`vencoder`使用时"encode_type"必须是"h264"

----
### 4.21 vaqrscanner
**功能描述**
视频分析模块，用于二维码识别

**模型**

 ![](image/ipu/vaqrscanner.svg)

**端口说明**

   * `in` - 输入端口，支持数据类型NV12/NV21

**参数说明**
无
**使用限制**
无

----
### 4.22 swc （Beta版）
**功能描述**
主要用于视频合成。最大支持四路视频输入，现用于NVR的演示

**模型**

 ![](image/ipu/swc.svg)

**端口说明**

   * `in0` - 输入端口0，支持数据类型为NV12
   * `in1` - 输入端口1，支持数据类型为NV12
   * `in2` - 输入端口2，支持数据类型为NV12
   * `in3` - 输入端口3，支持数据类型为NV12
   * `out` - 输出端口，支持数据类型NV12

**参数说明**
无
**使用限制**
无

----
### 4.23 softlayer
**功能描述**
主要用于图层叠加，正如其名，它是软件实现的，所以对CPU的要求相对较高。共支持四路图像层的叠加，主要应用于硬件做叠加能力受限时使用。其输出缓存需要通过Embezzle的方式设定，来源于需要叠加的原始图像

**模型**

 ![](image/ipu/softlayer.svg)

**端口说明**

   * `ol0` - 输入端口0，用于输入叠加图像，支持数据类型NV12或RGBA8888
   * `ol1` - 输入端口1，用于输入叠加图像，支持数据类型NV12或RGBA8888
   * `ol2` - 输入端口2，用于输入叠加图像，支持数据类型NV12或RGBA8888
   * `ol3` - 输入端口3，用于输入叠加图像，支持数据类型NV12或RGBA8888
   * `out` - 输出端口，支持数据类型NV12或RGBA8888

**参数说明**
无
**使用限制**
无

----
### 4.24 fileSink
**功能描述**
文件接收模块。用于将收到的数据保存到文件

**模型**

 ![](image/ipu/filesink.svg)

**端口说明**

   * `in` - 输入端口，数据类型无限制

**参数说明**

   * `data_path`（**必填**） - 文件保存位置（含文件名）
   * `max_save_cnt`（可选） - 保存多少笔数据，常用于调试

**使用限制**
无

----
### 4.25 filesource
**功能描述**
文件读取模块，用于从文件中读取数据

**模型**

 ![](image/ipu/filesource.svg)

**端口说明**

   * `out` - 输出端口，数据类型无限制

**参数说明**

   * `data_path`（**必填**） - 文件读取位置（含文件名）
   * `frame_size`（**必填**） - 每次读取大小，单位字节

**使用限制**
无

----
### 4.26 v4l2
**功能描述**
当系统外接摄像头自带ISP或者编码功能时，可通过此IPU，绕过`v2500`，输出YUV/MJPEG/H264数据

**模型**

 ![](image/ipu/v4l2.svg)

**端口说明**

   * `out` - 输出端口，支持数据类型NV12/NV21/YUV422/MJPEG/H264

**参数说明**

   * `driver`（**必填**） - V4L2驱动名字，通常为`imapx_camif`
   * `string`（**必填**） - 外接摄像头名字，需与V4L2摄像头驱动名字相同
   * `pixfmt`（**必填**） - CAMIF输出图像格式，支持数据类型NV12/YUV422/MJPEG/H264

**使用限制**
无

----
### 4.27 vencoder
**功能描述**
视频编码模块，包括H264和H265编码

**模型**

 ![](image/ipu/vencoder.svg)

**端口说明**

   * `frame` - 输入端口，支持数据类型NV12/NV21/RGB565/RGBA8888
   * `stream` - 输出编码后的H264或H265数据
   * `mv` - 输出编码数据的运动向量信息，数据类型MV，只有H264编码支持

**参数说明**

   * `encode_type`（必选） - 编码类型 ，取值为`"h264"`或者`"h265"`，分别表示编码输出为H264数据和H265数据
   * `buffer_size`（可选） - 编码输出缓存大小，与`min_buffernum`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值h264为2.25MB，h265为3M
   * `min_buffernum`（可选） - 编码输出缓存最小张数，必须大于等于3，与`buffer_size`一起决定编码后帧的大小上限，上限为`buffer_size`/`min_buffernum`，超过会被丢弃。缺省值为3
   * `framerate`（可选） - 编码目标帧率分子，取值大于0
   * `framebase`（可选） - 编码目标帧率分母，取值大于0，目标帧率计算等于`framerate`/`framebase`。如果前端输入帧率和目标帧率不一致，则编码模块会做帧率转换。目前输入30帧或者25帧时，目标帧率可转到低于30或者25的任意帧率。当输入帧率不是30或者25时，只能做倍数的转换。不配置时目标帧率等于输入帧率
   * `rotate`（可选） - 编码前画面旋转配置。`"90R"` - 向右旋转90度，`"90L"` - 向左旋转90度，缺省值为不旋转
   * `smartrc`（可选） - SmartRC功能的使能开关，1表示使能，缺省值为0。SmartRC详情可参见[码率控制用户手册]()
   * `refmode`（可选） - `encode_type`为`"h264"`时使能H264跳帧参考编码开关。0 - 关闭, 1 - 使能。跳帧参考使能后编码后的数据最多可以丢弃一半的帧而不影响解码。`encode_type`为`"h265"`时设置H265时域分级编码模式。0 - 关闭, 1 - 分两级, 2 - 分三级。时域分级编码可以在带宽或解码性能不足时丢弃高层及的帧而不影响解码
   * `rc_mode`（可选） - 编码码率控制模式。0 - CBR；1 - VBR；2 - FixQp，缺省值为0
   * `gop_length`（可选） - 编码GOP长度，是码率控制的基本单元。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `idr_interval`（可选） - 编码时I帧间隔。取值范围[1，150]，缺省值为输入帧率。需设置`rc_mode`才生效
   * `qp_max`（可选） - 编码QP的最大值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_min`（可选） - 编码QP的最小值，取值范围[2，51]，设置`rc_mode`为0或1时有效
   * `qp_delta`（可选） - 调整I帧QP，取值范围[-12，12]，用于调整I帧的相对质量或是I帧占比。设置`rc_mode`为0或1时有效
   * `bitrate`（可选） - CBR编码目标码率，取值范围[10000，60000000]，单位bps。需设置`rc_mode`才生效
   * `qp_hdr`（可选） - CBR编码起始QP值，取值范围[2，51]。设置为 - 1时，由编码算法自动选择。设置`rc_mode`为0时有效
   * `max_bitrate`（可选） - VBR编码时最大码率，取值范围[10000，60000000]，单位bps。设置`rc_mode`为1时有效
   * `threshold`（可选） - 限制VBR编码时码率下限，下限值为`max_bitrate` * `threshold` / 100，取值范围[1，99]。设置`rc_mode`为1时有效
   * `qp_fix`（可选） - FixQp模式下，采用的QP值，取值范围[2，51]。设置`rc_mode`为2时有效
   * `mv_w`（可选） - 编码图像宽度。在连接`mvmovement`时必须配置
   * `mv_h`（可选） - 编码图像高度。在连接`mvmovement`时必须配置
   * `enable_longterm`（可选）: `encode_type`为`"h264"`时才生效，使能长期参考帧，0 - 关闭, 1 - 使能。长期参考帧主要用于背景帧的参考。在背景长时间固定的情况下，可以提高压缩率
   * `chroma_qp_offset`（可选）: 调整编码时色度相对于亮度的QP差值，取值范围[-12，12]。为负时增强色度处理，可以降低颜色的拖影和块现象。为正时降低色度处理，绝对值越大，强度越高
   * `cyclic_intra_refresh` （可选） - 消除除第一个I帧外所有的I帧，把这些I帧中的I宏块平均分散到其所在GOP的其他P帧中; 1为开启，0为关闭，默认关闭

**使用限制**

   * `encode_type`必须设定为`h264`或者`h265`
   * 使能`enable_longterm`时，需要额外多一张帧缓存，只有`encode_type`为`"h264"`时才生效
   * 输出端口`mv`只有`encode_type`为`"h264"`时才存在
   * `mv_w``mv_h`只有`encode_type`为`"h264"`时才生效

NOTE: 对应IPU功能列表中的`Video Encoder`，QSDK V2.3.0启用，如果编码格式为H264，需要开启子菜单选项`h1264`。如果编码格式为H265，`Apollo`开发板上需要开启子菜单选项`h2`，`Apollo-ECO`开发板上需要开启子菜单选项`h2v4`。`h1264`与`h2 encode lib`为`Video Encoder`子菜单选项，`h2`，`h2v4`，`NONE`为`h2 encoder lib`子菜单选项，`h1264`仅适用`Apollo`，`Apollo-2`，`h2`仅适用`Apollo`，`h2v4`仅适用`Apollo-ECO`

----
### 4.28 g1jdec
**功能描述**
MJPEG硬件解码模块

**模型**

![](image/ipu/g1jdec.svg)

**端口说明**

   * `stream` - 输入端口，支持数据类型MJPEG/JPEG基本流
   * `frame` - 输出端口，支持数据类型NV12,YUV422SemiPlanar

**参数说明**
无
**使用限制**

   * `Apollo`

----

### 4.29 dehaze

**功能描述**
图像去雾处理模块

**模型**

![](image/ipu/dehaze.svg)

**端口说明**

   * `in` - 输入端口，支持数据类型NV12/NV21图像或视频流
   * `out` - 输出端口，支持数据类型NV12/NV21

**参数说明**

   * `nbuffers`（可选） - 去雾模块输出缓存个数，缺省值为3
   * `flambda1` (可选)  - 损失函数权重系数，类型为小数，值越小去雾效果越明显，缺省值为8.0
   * `tblocksize` (可选)　- 图像灰度值转换子块的大小，值越小图像块效应越小，但计算效率会减小，缺省值为16
   * `gblocksize` (可选)  - 快速导向滤波子块大小，其大小需为图像宽的整数倍，否则输出图像宽右边会被截掉部分像素，缺省值为32
   * `fgsigma`    (可选)  - 快速导向滤波处理后对图像进行高斯模糊的方差大小，类型为小数，缺省值为10.0
   * `stepsize`   (可选)  - 快速导向滤波的步长值，其值为图像子采样的倍数，如2倍的子采样，则步长值为2，缺省值为2
   * `fmaxtrans`  (可选)  - 转换系数的最大值，意义为转换系数做最小限制，防止图像灰度值过度映射而掉失很多图像信息，缺省值为0.86
   * `subwidth`   (可选)  - 子采样图像的宽度，其值应小于图像实际宽度，缺省值为128
   * `subheight`  (可选)  - 子采样图像的高度，其值应小于图像实际高度，缺省值为128
   * `mode`       (可选)　- 图像去雾的模式， `0`：暗通道优先去雾模式；`1`：直方图均衡化去雾模式；`2`：自适应限制直方图去雾模式，缺省值为0
   * `xblock`     (可选)  - 自适应限制直方图去雾模式下，对输入图像水平分块数目，值越小计算耗时增加，缺省值为4
   * `yblock`     (可选)  - 自适应限制直方图去雾模式下，对输入图像垂直分块数目，值越小计算耗时增加，缺省值为4   
   *  `min`       (可选)　- 自适应限制直方图去雾模式下，图像灰度转换的最小值，缺省值为0
   *  `max`　     (可选)　- 自适应限制直方图去雾模式下，图像灰度转换的最大值，缺省值为255
   *  `fclimit`　 (可选)　- 自适应限制直方图去雾模式下，图像直方图限制比例大小，越小图像对比度越高，值缺省值为0.9
   *  `nbin`　    (可选)　- 自适应限制直方图去雾模式下，图像直方图划分区域数目，缺省值为256

**使用限制**

   *  图像输入格式需为`NV12/NV21`，且经过此模块处理后会影响CPU的效率与视频帧率大小


----
## 5 Videobox控制API及数据类型
Videobox API包含图像处理，图像编解码，水印，智能影像等模块，依赖的头文件和库分别是**videobox.h**和**libvideobox.so**

Note: 简洁起见，示例代码内的函数调用均不做错误检查，实际应用中需视情况添加

---
## 5.1 API

---
### 5.1.1 videobox_start()
按照指定的json文件启动videobox

**语法**
```cpp
int videobox_start(const char *js);
```
**参数**

* `js` - `IN`，启动Videobox的JSON文件名（含路径）

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 5.1.2 videobox_stop()
关闭Videobox进程

**语法**
```cpp
int videobox_stop(void);
```
**参数**
无

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 5.1.3 videobox_rebind()
用于将一个IPU的端口绑定到另一个IPU

**语法**
```cpp
int videobox_rebind(const char *p0, const char *p2);
```
**参数**
* `p0` - `IN`，需要绑定的源端口
* `p2` - `IN`，绑定的目标端口

**返回值**

* 0 - 成功
* -1 - 失败

Note:
1. `p0`的IPU必须是Videobox Path链路中最后一个IPU，并且是输入端口
2. `p2`必须是IPU的输出端口
3. 两个端口的数据类型要一致

**示例**
将显示`ids`从1080P切换到480P，此时显示画面将从1080P切换成480P

 ![](image/api/rebind.svg)

```cpp
int s32Ret = 0;

// 将显示端口"display-ids0"，绑定到"ispost0-ss0"
s32Ret = videobox_rebind("display-ids0", "ispost0-ss0");
```

---
### 5.1.4 videobox_repath()
关闭当前Videobox，在用新的JSON启动Videobox

**语法**
```cpp
 int videobox_repath(const char *js)
```
**参数**

* `js` - `IN`，启动Videobox的JSON文件名（含路径）

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 5.1.5 videobox_set_loglevel()
设置Videobox打印信息级别

**语法**
```cpp
EN_VIDEOBOX_RET videobox_set_loglevel(int s32Level)
```
**参数**

* `s32Level` - `IN`，打印信息等级，取值范围[0，5]，值越大打印信息越多

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
无

---
## 5.2 数据类型

暂无

---
## 6 图像处理API及数据类型

---
## 6.1 API

---
### 6.1.1 camera_get_sensors()
获取支持的图像Sensor的名字

**语法**
```cpp
 int camera_get_sensors(const char *cam, cam_list_t *list);
```
**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `list` - `OUT`，支持的Sensor名列表，最大支持13个Sensor。参见[cam_list_t](main.md#6.2.1_cam_list_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
打印当前支持的Sensor列表，假定`v2500`实例名为"isp"
```cpp
cam_list_t stSensorList;
memset((void *)&stSensorList, 0, sizeof(stSensorList));
camera_get_sensors("isp", &stSensorList);
for (i = 0; i < 13; i++)
{
    printf("%s\n", stSensorList.name[i]);
}
```

---
### 6.1.2 camera_get_mirror()
获取画面的镜像状态，如水平，垂直等　

**语法**
```cpp
int camera_get_mirror(const char *cam, enum cam_mirror_state *state);
```
**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `state` - `OUT`，镜像状态。参见[cam_mirror_state](main.md#6.2.2_cam_mirror_state)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.3 camera_set_mirror()
设置图像的镜像状态

**语法**
```cpp
int camera_set_mirror(const char *cam, enum cam_mirror_state state);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `state` - `IN`，镜像状态。参见[cam_mirror_state](main.md#6.2.2_cam_mirror_state)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.4 camera_get_brightness()
获取图像亮度值

**语法**
```cpp
int camera_get_brightness(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，亮度值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.5 camera_set_brightness()
设置图像亮度值

**语法**
```cpp
int camera_set_brightness(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，亮度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.6 camera_get_contrast()
获取图像对比度

**语法**
```cpp
int camera_get_contrast(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，对比度值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.7 camera_set_contrast()
设置图像对比度

**语法**
```cpp
int camera_set_contrast(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，对比度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.8 camera_get_saturation()
设置图像饱和度

**语法**
```cpp
int camera_get_saturation(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，饱和度值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.9 camera_set_saturation()
设置图像饱和度

**语法**
```cpp
int camera_set_saturation(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，饱和度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.10 camera_get_sharpness()
获取图像锐度

**语法**
```cpp
int camera_get_sharpness(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，锐度值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.11 camera_set_sharpness()
设置图像锐度

**语法**
```cpp
int camera_set_sharpness(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，锐度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.12 camera_get_antiflicker()
获取当前抗闪烁频率设置

**语法**
```cpp
int camera_get_antiflicker(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，当前抗闪烁频率值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.13 camera_set_antiflicker()
设置当前抗闪烁频率。用于解决在交流光源下，画面的水波纹现象。频率值需要配置成与交流光源一致
目前只支持50Hz与60Hz光源下的抗闪烁

**语法**
```cpp
int camera_set_antiflicker(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，频率值，取值范围50或60

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.14 camera_get_wb()
获取白平衡场景模式

**语法**
```cpp
int camera_get_wb(const char *cam, enum cam_wb_mode *mode);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `mode` - `OUT`，白平衡模式，参见[cam_wb_mode](main.md#6.2.3_cam_wb_mode)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.15 camera_set_wb()
设置白平衡场景模式

**语法**
```cpp
int camera_set_wb(const char *cam, enum cam_wb_mode mode);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `mode` - `IN`，白平衡模式，参见[cam_wb_mode](main.md#6.2.3_cam_wb_mode)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.16 camera_get_scenario()
获取当前配置是白天/黑夜的场景信息。白天对应正常亮度场景，黑夜对应低亮度场景　

**语法**
```cpp
int camera_get_scenario(const char *cam, enum cam_scenario *scenario);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `scenario` - `OUT`，场景信息，参见[cam_scenario](main.md#6.2.4_cam_scenario)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.17 camera_set_scenario()
获取当前配置是白天/黑夜的场景信息。白天对应正常亮度场景，黑夜对应低亮度场景　

**语法**
```cpp
int camera_set_scenario(const char *cam, enum cam_scenario scenario);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `scenario` - `IN`，场景信息，参见[cam_scenario](main.md#6.2.4_cam_scenario)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.18 camera_get_hue()
获取图像色度

**语法**
```cpp
int camera_get_hue(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，色度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.19 camera_set_hue()
设置图像色度

**语法**
```cpp
int camera_set_hue(const char *cam, int hue);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，色度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.20 camera_get_brightnessvalue()
获取直方图统计亮度值

**语法**
```cpp
int camera_get_brightnessvalue(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，统计亮度值，取值范围[0，255]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.21 camera_get_ISOLimit()
获取Sensor的感光度

**语法**
```cpp
int camera_get_ISOLimit(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，感光度值

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无


---
### 6.1.22 camera_set_ISOLimit()
设置Sensor的感光度，受限于Sensor的最大增益倍数

**语法**
```cpp
int camera_set_ISOLimit(const char *cam, int value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，感光度值，[0，Sensor最大增益倍数]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.23 camera_get_exposure()
获取Sensor的曝光等级

**语法**
```cpp
int camera_get_exposure(const char *cam,int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `OUT`，曝光等级

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.24 camera_set_exposure()
设置Sensor的曝光等级

**语法**
```cpp
int camera_get_exposure(const char *cam,int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，曝光等级，取值范围[-4，4]

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.25 camera_monochrome_is_enabled()
判断当前是否使能黑白模式

**语法**
```cpp
int camera_monochrome_is_enabled(const char *cam);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字

**返回值**

* 0 - 未使能
* 1 - 使能

**示例**
无

---
### 6.1.26 camera_monochrome_enable()
使能/关闭黑白模式

**语法**
```cpp
int camera_monochrome_enable(const char *cam, int enable);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `enable` - `IN`，使能或者关闭黑白模式，0 - 关闭， 1 - 使能

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.27 camera_ae_is_enabled()
判断当前是否使能自动曝光模式

**语法**
```cpp
int camera_ae_is_enabled(const char *cam);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字

**返回值**

* 0 - 未使能
* 1 - 使能

**示例**
无

---
### 6.1.28 camera_ae_enable()
使能/关闭自动曝光模式

**语法**
```cpp
int camera_ae_enable(const char *cam, int enable);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `enable` - `IN`，使能或者关闭自动曝光模式，0 - 关闭， 1 - 使能

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.29 camera_wdr_is_enabled()
判断当前是否使能宽动态模式

**语法**
```cpp
int camera_wdr_is_enabled(const char *cam);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字

**返回值**

* 0 - 未使能
* 1 - 使能

**示例**
无

---
### 6.1.30 camera_wdr_enable()
使能/关闭宽动态模式

**语法**
```cpp
int camera_wdr_enable(const char *cam, int enable);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `enable` - `IN`，使能或者关闭宽动态模式，0 - 关闭， 1 - 使能

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.31 camera_snap_one_shot()
从预览模式进入拍照模式

**语法**
```cpp
int camera_snap_one_shot(const char *cam);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
struct fr_buf_info stFrameInfo;

// 进入拍照模式
camera_snap_one_shot("isp");

// 拍照，并获取照片
video_get_snap("jpeg-out", &stFrameInfo);

// ...

// 释放照片缓存
video_put_snap("jpeg-out", &stFrameInfo);

// 退出拍照模式
camera_snap_exit("isp");
```

---
### 6.1.32 camera_snap_exit()
退出拍照模式，返回预览模式

**语法**
```cpp
 int camera_snap_exit(const char *cam);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_snap_one_shot()](main.md#6.1.31_camera_snap_one_shot%28%29)

---
### 6.1.33 camera_get_sensor_exposure()
获取曝光时间

**语法**
```cpp
int camera_get_sensor_exposure(const char *cam, int *value);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，曝光时间，单位微秒

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.34 camera_set_sensor_exposure()
设置曝光时间

**语法**
```cpp
int camera_set_sensor_exposure(const char *cam, int usec);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `value` - `IN`，曝光时间，单位微秒

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.35 camera_get_dpf_attr()
获取坏点矫正配置属性

**语法**
```cpp
int camera_get_dpf_attr(const char *cam, cam_dpf_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，坏点矫正配置属性，参见[cam_dpf_attr_t](main.md#6.2.7_cam_dpf_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_set_dpf_attr()](main.md#6.1.36_camera_set_dpf_attr%28%29)

---
### 6.1.36 camera_set_dpf_attr()
设置坏点矫正配置属性

**语法**
```cpp
 int camera_set_dpf_attr(const char *cam, cam_dpf_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，坏点矫正配置属性，参见[cam_dpf_attr_t](main.md#6.2.7_cam_dpf_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
cam_dpf_attr_t stDpfAttr;

memset(&stDpfAttr, 0, sizeof(cam_dpf_attr_t));
camera_get_dpf_attr("isp", &stDpfAttr);

stDpfAttr.cmn.mdl_en = 1;
stDpfAttr.threshold = 0;
stDpfAttr.weight = 0.0;
camera_set_dpf_attr("isp", &stDpfAttr);
```

---
### 6.1.37 camera_get_dns_attr()
获取2D降噪配置属性

**语法**
```cpp
int camera_get_dns_attr(const char *cam, cam_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，2D去噪配置属性，参见[cam_dns_attr_t](main.md#6.2.8_cam_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
 参见[camera_set_dns_attr()](main.md#6.1.38_camera_set_dns_attr()%28%29)

---
### 6.1.38 camera_set_dns_attr()
设置2D降噪属性

**语法**
```cpp
int camera_set_dns_attr(const char *cam, cam_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，2D去噪配置属性，参见[cam_dns_attr_t](main.md#6.2.8_cam_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
cam_dns_attr_t stDnsAttr;

memset(&stDnsAttr, 0, sizeof(cam_dns_attr_t));
camera_get_dns_attr("isp", &stDnsAttr);

stDnsAttr.cmn.mdl_en = 1;
stDnsAttr.cmn.mode = CAM_CMN_AUTO;
stDnsAttr.strength = 6.0;
camera_set_dns_attr("isp", &stDnsAttr);
```

---
### 6.1.39 camera_get_sha_dns_attr()
获取锐化模块的降噪属性

实现辅助去噪功能，用于避免在Tone-Mapping之后出现伪彩

**语法**
```cpp
int camera_get_sha_dns_attr(const char *cam, cam_sha_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，锐化模块降噪属性，参见[cam_sha_dns_attr_t](main.md#6.2.9_cam_sha_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_set_sha_dns_attr()](main.md#6.1.40_camera_set_sha_dns_attr%28%29)


---
### 6.1.40 camera_set_sha_dns_attr()
设置锐化模块的降噪属性

实现辅助去噪功能，用于避免在Tone-Mapping之后出现伪彩

**语法**
```cpp
int camera_set_sha_dns_attr(const char *cam, cam_sha_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，锐化模块降噪属性，参见[cam_sha_dns_attr_t](main.md#6.2.9_cam_sha_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
cam_sha_dns_attr_t stShaDnsAttr;

memset(&stShaDnsAttr, 0, sizeof(cam_sha_dns_attr_t));
camera_get_sha_dns_attr("isp", &stShaDnsAttr);

stShaDnsAttr.cmn.mdl_en = 1;
stShaDnsAttr.cmn.mode = CAM_CMN_AUTO;
stShaDnsAttr.strength = 1.1;
stShaDnsAttr.smooth = 1;
camera_set_sha_dns_attr("isp", &stShaDnsAttr);
```
---
### 6.1.41 camera_get_wb_attr()
获取白平衡属性

在不同色温的光源下，白色会偏蓝或偏红。白平衡通过调整R，G，B三个颜色通道的强度，使白色真实呈现

**语法**
```cpp
int camera_get_wb_attr(const char *cam, cam_wb_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，白平衡属性，参见[cam_wb_attr_t](main.md#6.2.10_cam_wb_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_set_wb_attr()](main.md#6.1.42_camera_set_wb_attr%28%29)

---
### 6.1.42 camera_set_wb_attr()
设置白平衡属性

在不同色温的光源下，白色会偏蓝或偏红。白平衡通过调整R，G，B三个颜色通道的强度，使白色真实呈现

**语法**
```cpp
int camera_set_wb_attr(const char *cam, cam_wb_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，白平衡属性，参见[cam_wb_attr_t](main.md#6.2.10_cam_wb_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
cam_wb_attr_t stWbAttr;

memset(&stWbAttr, 0, sizeof(cam_wb_attr_t));
camera_get_wb_stWbAttr("isp", &stWbAttr);

stWbAttr.cmn.mdl_en = 1;
stWbAttr.cmn.mode = CAM_CMN_MANUAL;
stWbAttr.man.rgb.r = 6.5;
stWbAttr.man.rgb.g = 1.0;
stWbAttr.man.rgb.b = 0.1;
camera_set_wb_stWbAttr("isp", &stWbAttr);
```

---
### 6.1.43 camera_get_3d_dns_attr()
获取3D降噪属性

**语法**
```cpp
int camera_get_3d_dns_attr(const char *cam, cam_3d_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，3D降噪属性，参见[cam_3d_dns_attr_t](main.md#6.2.13_cam_3d_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_set_3d_dns_attr()](main.md#6.1.44_camera_set_3d_dns_attr%28%29)

---
### 6.1.44 camera_set_3d_dns_attr()
设置3D降噪属性，作用于YUV颜色空间

**语法**
```cpp
int camera_set_3d_dns_attr(const char *cam, cam_3d_dns_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，3D降噪属性，参见[cam_3d_dns_attr_t](main.md#6.2.13_cam_3d_dns_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
```cpp
cam_3d_dns_attr_t st3DnsAttr;

memset(&st3DnsAttr, 0, sizeof(cam_3d_dns_attr_t));
camera_get_3d_dns_st3DnsAttr("isp", &st3DnsAttr);

st3DnsAttr.cmn.mdl_en = 1;
st3DnsAttr.cmn.mode = CAM_CMN_MANUAL;
st3DnsAttr.y_threshold = 20;
st3DnsAttr.u_threshold = 20;
st3DnsAttr.v_threshold = 20;
st3DnsAttr.weight = 80;
camera_set_3d_dns_st3DnsAttr("isp", &st3DnsAttr);
```

---
### 6.1.45 camera_get_yuv_gamma_attr()
获取伽马属性

**语法**
```cpp
int camera_get_yuv_gamma_attr(const char *cam, cam_yuv_gamma_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `OUT`，3D降噪属性，参见[cam_yuv_gamma_attr_t](main.md#6.2.14_cam_yuv_gamma_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.46 camera_set_yuv_gamma_attr()
设置伽马属性

**语法**
```cpp
int camera_set_yuv_gamma_attr(const char *cam, cam_yuv_gamma_attr_t *attr);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `attr` - `IN`，3D降噪属性，参见[cam_yuv_gamma_attr_t](main.md#6.2.14_cam_yuv_gamma_attr_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.47 camera_get_fps_range()
获取动态帧率范围

**语法**
```cpp
int camera_get_fps_range(const char *cam, cam_fps_range_t *fps_range);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `fps_range` - `OUT`，动态帧率范围，参见[cam_fps_range_t](main.md#6.2.15_cam_fps_range_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.48 camera_set_fps_range()
设置动态帧率范围。设置动态帧率之后，系统会在低光照情况下，延长曝光时间，降低帧率，达到好的图像效果。

**语法**
```cpp
int (const char *cam, const cam_fps_range_t *fps_range);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `fps_range` - `IN`，动态帧率范围，参见[cam_fps_range_t](main.md#6.2.15_cam_fps_range_t)

**返回值**

* 0 - 成功
* -1 - 失败


此功能需配置**/root/.ispddk/sensor0-isp-config.txt**（不同项目配置文件可能不一样）

|参数|说明|
|---|
|`AE_TARGET_MAX_FPS_LOCK_ENABLE`|使能最大帧率锁定。0 - 关闭，1 - 使能|
|`AE_TARGET_LOWLUX_GAIN_ENTER`|低亮环境的增益值，取值范围[0.0，32.0]。要求大于`AE_TARGET_LOWLUX_GAIN_EXIT`，但不超过`CMC_SENSOR_MAX_GAIN_ACM`|
|`AE_TARGET_LOWLUX_GAIN_EXIT`|退出低亮环境的增益值，取值范围[0.0, 32.0]|
|`AE_TARGET_LOWLUX_EXPOSURE_ENTER`|低亮环境的曝光时间，取值范围[0，1000000]|
|`CMC_SENSOR_MAX_GAIN_ACM`|摄像头的最大增益值， 取值范围[1.0，136.0]|


配置参考
```bash
AE_TARGET_MAX_FPS_LOCK_ENABLE 1
AE_TARGET_LOWLUX_GAIN_ENTER 6
AE_TARGET_LOWLUX_GAIN_EXIT 4
AE_TARGET_LOWLUX_EXPOSURE_ENTER 39500
CMC_SENSOR_MAX_GAIN_ACM 12
```

**示例**
无

---
### 6.1.49 camera_get_day_mode()
获取当前是否为白天。其结果只是辅助判断，需要结合亮度传感器综合判断

**语法**
```cpp
int camera_get_day_mode(const char *cam, enum cam_day_mode *mode);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `mode` - `OUT`，当前模式，参见[cam_day_mode](main.md#6.2.16_cam_day_mode)

**返回值**

* 0 - 成功
* -1 - 失败


此功能需配置**/root/.ispddk/sensor0-isp-config.txt**（不同项目配置文件可能不一样）

|参数|说明|
|---|
|`CMC_NIGHT_MODE_DETECT_ENABLE`|使能检测功能开关。0 - 关闭，1 - 使能|
|`CMC_NIGHT_MODE_DETECT_BRIGHTNESS_ENTER`|进入夜晚模式的亮度值，范围[-1.0，1.0]|
|`CMC_NIGHT_MODE_DETECT_GAIN_ENTER`|进入夜晚模式的增益值，范围[1.0，136.0]|
|`CMC_NIGHT_MODE_DETECT_BRIGHTNESS_EXIT`|退出夜晚模式的亮度值，范围[-1.0，1.0]|
|`CMC_NIGHT_MODE_DETECT_GAIN_EXIT`|退出夜晚模式的增益值，范围[1.0，136.0]|
|`CMC_NIGHT_MODE_DETECT_WEIGHTING`|夜晚模式亮度值占的权重，范围[0.0，1.0]|
|`CMC_NIGHT_MODE_DETECT_ANTI_JAMMING_TIME`|夜晚白天切换时抗抖动时间，范围[200，5000] 单位ms|

配置参考
```bash
CMC_NIGHT_MODE_DETECT_ENABLE 1
CMC_NIGHT_MODE_DETECT_BRIGHTNESS_ENTER -0.32
CMC_NIGHT_MODE_DETECT_BRIGHTNESS_EXIT -0.05
CMC_NIGHT_MODE_DETECT_GAIN_ENTER 2.0
CMC_NIGHT_MODE_DETECT_GAIN_EXIT 2.0
CMC_NIGHT_MODE_DETECT_WEIGHTING 0.10
CMC_NIGHT_MODE_DETECT_ANTI_JAMMING_TIME 500
```

**示例**
无

---
### 6.1.50 camera_get_index()
获取当前Sensor的索引，用于多Sensor场景

**语法**
```cpp
int camera_get_index(const char *cam, int *idx)
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `idx` - `OUT`，当前Sensor索引

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.51 camera_set_index()
获取当前Sensor的索引，用于多Sensor场景

当需要对某一个Sensor做调整时，需要先通过此函数设置要调整的Sensor索引

**语法**
```cpp
int camera_set_index(const char *cam, int idx);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `idx` - `IN`，目标Sensor索引

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
假设第一个Sensor的索引是0，第二个Sensor索引是1，如果需要调整索引为1的白平衡
```cpp
cam_wb_attr_t stWbAttr;

// 切换到第二个Sensor
camera_set_index("isp", 1);
memset(&stWbAttr, 0, sizeof(cam_wb_attr_t));
camera_get_wb_stWbAttr("isp", &stWbAttr);

stWbAttr.cmn.mdl_en = 1;
stWbAttr.cmn.mode = CAM_CMN_MANUAL;
stWbAttr.man.rgb.r = 3.5;
stWbAttr.man.rgb.g = 1.0;
stWbAttr.man.rgb.b = 0.1;
camera_set_wb_stWbAttr("isp", &stWbAttr);
```

---
### 6.1.52 camera_save_data()
用于保存Raw RGB或者YUV数据

**语法**
```cpp
int camera_save_data(const char *cam, const cam_save_data_t *save_data);
```

**参数**

* `cam` - `IN`，对应`v2500`IPU实例化的名字
* `save_data` - `IN`，保存的配置参数，参见[cam_save_data_t](main.md#6.2.18_cam_save_data_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
同时保存FLX，YUV，Bayer Raw 三种格式图像
```cpp
cam_save_data_t stSaveData;

memset(&stSaveData, 0, sizeof(cam_save_data_t));

stSaveData.fmt = CAM_DATA_FMT_BAYER_FLX | CAM_DATA_FMT_YUV |CAM_DATA_FMT_BAYER_RAW;
stSaveData.file_path = "/nfs/"
camera_save_data("isp", &stSaveData);
```

---
### 6.1.53 camera_create_and_fire_fcedata()
动态创建并加载矫正数据

**语法**
```cpp
int camera_create_and_fire_fcedata(const char *cam, cam_fcedata_param_t *fcedata)
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `fcedata` - `IN`，矫正数据，参见[cam_fcedata_param_t](main.md#6.2.22_cam_fcedata_param_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
输入鱼眼图像960x960，矫正模式选择垂直向下视角`FISHEYE_MODE_DOWNVIEW`，矫正范围选取经度10度到80度，纬度-45度到45度，展开成600x570
`fisheye_radius`为-5表示，输入图像半径减少5，用于减少画面畸变，矫正效果

![](image/api/fce-downscale.jpg)
![](image/api/fce-downscale-correct.jpg)


```cpp
cam_fcedata_param_t stFceParam;

memset(&stFceParam, 0, sizeof(stFceParam));
stFceParam.common.out_width = 600;
stFceParam.common.out_height = 570;
stFceParam.common.grid_size = 32;
stFceParam.common.fisheye_center_x = 960 / 2;
stFceParam.common.fisheye_center_y = 960 / 2;

stFceParam.fec.fisheye_mode = FISHEYE_MODE_DOWNVIEW;
stFceParam.fec.fisheye_radius = -5;
stFceParam.fec.fisheye_start_phi = -10;
stFceParam.fec.fisheye_end_phi = 80;
stFceParam.fec.fisheye_start_theta = -45;
stFceParam.fec.fisheye_end_theta = 45;

camera_create_and_fire_fcedata(cam, &stFceParam);
```

---
### 6.1.54 camera_load_and_fire_fcedata()
动态加载鱼眼矫正数据

**语法**
```cpp
int camera_load_and_fire_fcedata(const char *cam, cam_fcefile_param_t *fcefile);
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `fcefile` - `IN`，保存的配置参数，参见[cam_fcefile_param_t](main.md#6.2.28_cam_fcefile_param_t)

**返回值**

* 0 - 成功
* -1 - 失败

Note:
1. 不能与[camera_create_and_fire_fcedata()](main.md#6.1.53_camera_create_and_fire_fcedata%28%29)同时使用
2. 只能调用一次
3. 要退出此模式时，需要重启Videobox

**示例**
示例三种模式的动态切换

```cpp
cam_fcefile_param_t stFceFileParam;
cam_fcefile_mode_t *pstModeList;
cam_fcefile_file_t *pstFileList;
cam_fcefile_port_t  stPort;
cam_position_t  stPosition;
int s32ModeIdx = 0;
cam_save_fcedata_param_t stSaveParam;
int i = 0;

memset(&stFceFileParam, 0, sizeof(stFceFileParam));

// 示例三种模式切换
stFceFileParam.mode_number = 3;
pstModeList = stFceFileParam.mode_list;

for (i = 0; i < stFceFileParam.mode_number; i++, pstModeList++)
{
    switch(i)
    {
    // 四分屏输出
    case 0:
        pstModeList->file_number = 4;

        // 输出到[0，0，960，540]
        sprintf(pstModeList->pstFileList[0].file_grid, "/nfs/lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_0.bin");
        pstModeList->pstFileList[0].outport_number = 1;
        stPosition.x = 0;
        stPosition.y = 0;
        stPosition.width = 960;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[0].outport_list[2] = stPort;

        // 输出到[960，0，960，540]
        sprintf(pstModeList->pstFileList[1].file_grid, "/nfs/lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_1.bin");
        pstModeList->pstFileList[1].outport_number = 1;
        stPosition.x = 960;
        stPosition.y = 0;
        stPosition.width = 960;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[1].outport_list[2] = stPort;

        // 输出到[0，540，960，540]
        sprintf(pstModeList->pstFileList[2].file_grid, "/nfs/lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_2.bin");
        pstModeList->pstFileList[2].outport_number = 1;
        stPosition.x = 0;
        stPosition.y = 540;
        stPosition.width = 960;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[2].outport_list[2] = stPort;

        // 输出到[960，540，960，540]
        sprintf(pstModeList->pstFileList[3].file_grid, "/nfs/lc_down_view_0-0_hermite32_1088x1088-960x544_fisheyes_downview_3.bin");
        pstModeList->pstFileList[3].outport_number = 1;
        stPosition.x = 960;
        stPosition.y = 540;
        stPosition.width = 960;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[3].outport_list[2] = stPort;
        break;
    // 二分屏幕输出（上下分屏）    
    case 1:
        pstModeList->file_number = 2;

        // 输出到[0，0，1920，540]
        sprintf(pstModeList->pstFileList[0].file_grid, "/nfs/lc_up_2p360_0-0_hermite32_1088x1088-1920x544_fisheyes_panorama_0.bin");
        pstModeList->pstFileList[0].outport_number = 1;
        stPosition.x = 0;
        stPosition.y = 0;
        stPosition.width = 1920;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[0].outport_list[1] = stPort;

        // 输出到[0，540，1920，540]
        sprintf(pstModeList->pstFileList[1].file_grid, "/nfs/lc_up_2p360_0-0_hermite32_1088x1088-1920x544_fisheyes_panorama_1.bin");
        pstModeList->pstFileList[1].outport_number = 1;
        stPosition.x = 0;
        stPosition.y = 540;
        stPosition.width = 1920;
        stPosition.height = 540;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[1].outport_list[1] = stPort;

        break;
    // 全展开输出
    case 2:
        pstModeList->file_number = 1;

        // 输出到[0，0，1920，1080]
        sprintf(pstModeList->pstFileList[0].file_grid, "/nfs/lc_Barrel_0-0_hermite32_1088x1088-1920x1080_fisheyes_mercator_0.bin");
        pstModeList->pstFileList[0].outport_number = 1;
        stPosition.x = 0;
        stPosition.y = 0;
        stPosition.width = 1920;
        stPosition.height = 1080;
        stPort.type = CAM_FCE_PORT_UO;
        stPort.stPosition = stPosition;
        pstModeList->pstFileList[0].outport_list[1] = stPort;
        break;
    }
}

// 加载动态矫正数据
camera_load_and_fire_fcedata("ispost0", &stFceFileParam);
sleep(2);

// 获取当前的模式
camera_get_fce_mode("ispost0", &s32ModeIdx);

// 切换模式，这里隔两秒切换一次
for (i = 0; i < stFceFileParam.mode_number; i++)
{
    camera_set_fce_mode("ispost0", i);
    sleep(2);
}

// 保存矫正模式配置文件
sprintf(stSaveParam.file_name, "/nfs/fceconfig");
camera_save_fcedata("ispost0", &stSaveParam);
```

---
### 6.1.55 camera_set_fce_mode()
设置鱼眼矫正的模式

**语法**
```cpp
int camera_set_fce_mode(const char *cam, int mode);
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `mode` - `IN`，模式索引

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_load_and_fire_fcedata()](main.md#6.1.54_camera_load_and_fire_fcedata%28%29)

---
### 6.1.56 camera_get_fce_mode()
获取鱼眼矫正的当前模式

**语法**
```cpp
int camera_get_fce_mode(const char *cam, int *mode);
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `mode` - `OUT`，模式索引

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_load_and_fire_fcedata()](main.md#6.1.54_camera_load_and_fire_fcedata%28%29)

---
### 6.1.57 camera_get_fce_totalmodes()
获取鱼眼矫正当前支持的总模式数

**语法**
```cpp
int camera_get_fce_totalmodes(const char *cam, int *count);
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `count` - `OUT`，模式索引

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 6.1.58 camera_save_fcedata()
保存当前的鱼眼矫正配置数据到文件

**语法**
```cpp
int camera_save_fcedata(const char *cam, cam_save_fcedata_param_t *param);
```

**参数**

* `cam` - `IN`，对应`ispostv1`或者`ispostv2`IPU实例化的名字
* `param` - `IN`，保存鱼眼矫正数据的参数，参见[cam_save_fcedata_param_t](main.md#6.2.29_cam_save_fcedata_param_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[camera_load_and_fire_fcedata()](main.md#6.1.54_camera_load_and_fire_fcedata%28%29)

---
## 6.2 数据类型

----
### 6.2.1 cam_list_t
定义Sensor名列表

**定义**
```cpp
typedef struct cam_list_st {
        char name[13][12];
}cam_list_t;
```
**成员**

* `name` - 保存Sensor名的数组列表，最大支持13个Sensor

----
### 6.2.2 cam_mirror_state
定义镜像的状态

**定义**
```cpp
enum cam_mirror_state {
    CAM_MIRROR_NONE,
    CAM_MIRROR_H,
    CAM_MIRROR_V,
    CAM_MIRROR_HV,
};
```
**成员**

* `CAM_MIRROR_NONE` - 无镜像
* `CAM_MIRROR_H` - 水平镜像
* `CAM_MIRROR_V` - 垂直镜像
* `CAM_MIRROR_HV` - 水平和垂直镜像

----
### 6.2.3 cam_wb_mode
定义白平衡场景模式

**定义**
```cpp
enum cam_wb_mode {
    CAM_WB_AUTO,
    CAM_WB_2500K,
    CAM_WB_4000K,
    CAM_WB_5300K,
    CAM_WB_6800K
};
```
**成员**

* `CAM_WB_AUTO` - 自动模式
* `CAM_WB_2500K` - 2500K色温
* `CAM_WB_4000K` - 4000K色温
* `CAM_WB_5300K` - 5300K色温
* `CAM_WB_6800K` - 6800K色温

----
### 6.2.4 cam_scenario
定义白平衡场景模式

**定义**
```cpp
enum cam_scenario {
    CAM_DAY,
    CAM_NIGHT
};
```
**成员**

* `CAM_DAY` - 白天
* `CAM_NIGHT` - 黑夜

----
### 6.2.5 cam_common_t
定义图像处理通用头结构

**定义**
```cpp
typedef struct __cam_common_t
{
    int mdl_en;
    int version;
    cam_common_mode_e mode;
} cam_common_t;
```
**成员**

* `mdl_en` - 模块使能控制，1 - 使能，0 - 关闭
* `version` - 模块版本号
* `mode` - 模式，参见参见[cam_common_mode_e](main.md#6.2.6_cam_common_mode_e)

----
### 6.2.6 cam_common_mode_e
定义图像处理通用头结构

**定义**
```cpp
typedef enum __cam_common_mode_e
{
    CAM_CMN_AUTO,
    CAM_CMN_MANUAL,
    CAM_CMN_MAX
}cam_common_mode_e;
```
**成员**

* `CAM_CMN_AUTO` - 自动模式
* `CAM_CMN_MANUAL` - 手动模式
* `CAM_CMN_MAX` - 模式上限

----
### 6.2.7 cam_dpf_attr_t
定义坏点矫正属性数据结构

**定义**
```cpp
typedef struct __cam_dpf_attr_t
{
    cam_common_t cmn;
    int threshold;
    double weight;
} cam_dpf_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `threshold` - 检测坏点的阀值，范围[0，63]
* `weight` - 平均绝对偏差的权重值，范围[0，255]

---
### 6.2.8 cam_dns_attr_t
定义2D降噪属性数据结构

**定义**
```cpp
typedef struct __cam_dns_attr_t
{
    cam_common_t cmn;
    int is_combine_green;
    double strength;
    double greyscale_threshold;
    double sensor_gain;
    int sensor_bitdepth;
    int sensor_well_depth;
    double sensor_read_noise;
} cam_dns_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `is_combine_green` - 是否使能合并Gr，Gb通道。 1 - 使能，0 - 关闭
* `strength` - 去噪强度，取值范围[0，6]
* `greyscale_threshold` - 像素阈值乘法器，用于调整灰度权重的色差的灵敏度。 该值通常为0.25左右。 取值范围[0.07，14.5]
* `sensor_gain` - Sensor增益， 范围[0,128]
* `sensor_bitdepth` - Sensor位深， 范围[8,16]
* `sensor_well_depth` - Sensor阱深，像素在剪裁之前可以收集的最大电子数。范围[0，65535]
* `sensor_read_noise` - 读取像素值时的噪声标准偏差，范围[0，100]

----
### 6.2.9 cam_sha_dns_attr_t
定义锐化降噪属性数据结构

**定义**
```cpp
typedef struct __cam_sha_dns_attr_t
{
    cam_common_t cmn;
    double strength;
    double smooth;
} cam_sha_dns_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `strength` - 去噪强度，取值范围[0，16]。为1时无效果，小于1时较弱去噪，大于1时较强去噪
* `smooth` - 边缘平滑度，取值范围[0，16]。为1时无效果，小于1时较弱平滑度，大于1时较强平滑度


----
### 6.2.10 cam_wb_attr_t
定义白平衡属性数据结构

**定义**
```cpp
typedef struct __cam_wb_attr_t
{
    cam_common_t cmn;
    cam_wb_man_attr_t man;
} cam_wb_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `man` - 手动白平衡属性，参见[cam_wb_man_attr_t](main.md#6.2.11_cam_wb_man_attr_t)

----
### 6.2.11 cam_wb_man_attr_t
定义白平衡属性数据结构

**定义**
```cpp
typedef struct __cam_wb_man_attr_t
{
    cam_rgb_t rgb;
} cam_wb_man_attr_t;
```
**成员**

* `rgb` - 通用参数，参见[cam_rgb_t](main.md#6.2.12_cam_rgb_t)

----
### 6.2.12 cam_rgb_t
定义白平衡颜色属性数据结构

**定义**
```cpp
typedef struct __cam_rgb_t
{
    double r;
    double g;
    double b;
} cam_rgb_t;
```
**成员**

* `r` - 白平衡的红色值，取值范围[0.5，8]
* `g` - 白平衡的绿色值，取值范围[0.5，8]
* `b` - 白平衡的蓝色值，取值范围[0.5，8]

---
### 6.2.13 cam_3d_dns_attr_t
定义3D降噪属性数据结构

**定义**
```cpp
typedef struct __cam_3d_dns_attr_t
{
    cam_common_t cmn;
    int y_threshold;
    int u_threshold;
    int v_threshold;
    int weight;
} cam_3d_dns_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `y_threshold` - 亮度阈值，偶数，取值范围[2，254]。参考帧与当前帧同一位置像素的Y值差小于`y_threshold`时，则做3D降噪
* `u_threshold` - 色度U阈值，偶数，取值范围[2，254]。参考帧与当前帧同一位置像素的Y值差小于`u_threshold`时，则做3D降噪
* `v_threshold` - 色度V阈值，偶数，取值范围[2，254]。参考帧与当前帧同一位置像素的Y值差小于`v_threshold`时，则做3D降噪
* `weight` - 3D降噪计算时，当前帧占的权重， 取值范围[1，9]。为6时，表示当前帧占60%，参考帧占40%

Note: 如果当前帧占的权重过低，运动场景容易发生拖影

---
### 6.2.14 cam_yuv_gamma_attr_t
定义伽马属性数据结构

**定义**
```cpp
#define CAM_ISP_YUV_GAMMA_COUNT     63

typedef struct __cam_yuv_gamma_attr_t
{
    cam_common_t cmn;
    float curve[CAM_ISP_YUV_GAMMA_COUNT];
} cam_yuv_gamma_attr_t;
```
**成员**

* `cmn` - 通用参数，参见[cam_common_t](main.md#6.2.5_cam_common_t)
* `curve` - 伽马曲线值，取值范围[0，1]

---
### 6.2.15 cam_fps_range_t
定义动态帧率数据结构

**定义**
```cpp
typedef struct __cam_fps_range_t
{
    float min_fps;
    float max_fps;
} cam_fps_range_t;
```
**成员**

* `min_fps` - 最小帧率
* `max_fps` - 最大帧率

---
### 6.2.16 cam_day_mode
定义白天与夜晚模式

**定义**
```cpp
enum cam_day_mode {
    CAM_DAY_MODE_NIGHT,
    CAM_DAY_MODE_DAY
};
```
**成员**

* `CAM_DAY_MODE_NIGHT` - 夜晚模式
* `CAM_DAY_MODE_DAY` - 白天模式

---
### 6.2.17 cam_data_format_e
定义数据存放的数据结构

**定义**
```cpp
typedef enum __cam_data_format_e {
    CAM_DATA_FMT_NONE,
    CAM_DATA_FMT_BAYER_FLX = 1,
    CAM_DATA_FMT_BAYER_RAW = 1 << 1,
    CAM_DATA_FMT_YUV = 1 << 2
} cam_data_format_e;
```
**成员**

* `CAM_DATA_FMT_NONE` - 无
* `CAM_DATA_FMT_BAYER_FLX` - flx格式，Imagination的数据格式，可使用flx工具查看
* `CAM_DATA_FMT_BAYER_RAW` - Bayer raw 16 bit格式，可以使用7YUV工具查看
* `CAM_DATA_FMT_YUV` - NV12格式，可以使用7YUV工具查看

---
### 6.2.18 cam_save_data_t
定义数据存放的数据结构

**定义**
```cpp
typedef struct __cam_save_data_t
{
    cam_data_format_e fmt;
    char file_path[128];
} cam_save_data_t;
```
**成员**

* `fmt` - 保存数据的格式，参见[cam_data_format_e](main.md#6.2.17_cam_data_format_e)
* `file_path` - 文件保存路径（**不**含文件名）

---
### 6.2.19 cam_grid_size_e
定义矫正数据格子大小数据结构

**定义**
```cpp
typedef enum __cam_grid_size_e
{
    CAM_GRID_SIZE_32,
    CAM_GRID_SIZE_16,
    CAM_GRID_SIZE_8,
    CAM_GRID_SIZE_MAX
} cam_grid_size_e;
```
**成员**

* `CAM_GRID_SIZE_32` - 矫正数据单位格子大小为32
* `CAM_GRID_SIZE_16` - 矫正数据单位格子大小为16
* `CAM_GRID_SIZE_8` - 矫正数据单位格子大小为8
* `CAM_GRID_SIZE_MAX` - 支持最大格子配置数

---
### 6.2.20 cam_fcedata_common_t
定义矫正数据通用头数据结构

**定义**
```cpp
typedef struct __cam_fcedata_common_t
{
    int out_width;
    int out_height;
    cam_grid_size_e grid_size;
    int fisheye_center_x;
    int fisheye_center_y;
} cam_fcedata_common_t;
```
**成员**

* `out_width` - 矫正后图像宽度
* `out_height` -  矫正后图像高
* `grid_size` - 矫正数据格子大小，参见[cam_grid_size_e](main.md#6.2.19_cam_grid_size_e)
* `fisheye_center_x` - 圆心X坐标，左上角为原点
* `fisheye_center_y` - 圆心Y坐标，左上角为原点

---
### 6.2.21 cam_fisheye_correction_t
定义矫正数据的数据结构

**定义**
```cpp
typedef struct __cam_fisheye_correction_t
{
    int fisheye_mode;
    int fisheye_gridsize;

    double fisheye_start_theta;
    double fisheye_end_theta;
    double fisheye_start_phi;
    double fisheye_end_phi;

    int fisheye_radius;

    int rect_center_x;
    int rect_center_y;

    int fisheye_flip_mirror;
    double scaling_width;
    double scaling_height;
    double fisheye_rotate_angle;
    double fisheye_rotate_scale;

    double fisheye_heading_angle;
    double fisheye_pitch_angle;
    double fisheye_fov_angle;

    double coefficient_horizontal_curve;
    double coefficient_vertical_curve;

    double fisheye_theta1;
    double fisheye_theta2;
    double fisheye_translation1;
    double fisheye_translation2;
    double fisheye_translation3;
    double fisheye_center_x2;
    double fisheye_center_y2;
    double fisheye_rotate_angle2;

    int debug_info;
} cam_fisheye_correction_t;
```

**成员**

* `fisheye_mode` - 矫正模式，取值范围[0，9]，参见[鱼眼矫正模式介绍](www.todo.com)
* `fisheye_gridsize` - 矫正格子的大小，取值范围8/16/32
* `fisheye_start_theta` - 经度方向起始角度，参建图中定义
* `fisheye_end_theta` - 经度方向结束角度
* `fisheye_start_phi` - 纬度方向起始角度，参考图中定义
* `fisheye_end_phi` - 纬度方向结束角度
* `fisheye_radius` - 球半径
* `rect_center_x` - 输出图像的中心X坐标
* `rect_center_y` - 输出图像的中心Y坐标
* `fisheye_flip_mirror` - 图像镜像设置
* `scaling_width` - 水平方向缩放因子
* `scaling_height` - 垂直方向缩放因子
* `fisheye_rotate_angle` - 旋转角度
* `fisheye_rotate_scale` - 旋转缩放因子
* `fisheye_heading_angle` - 全景视角
* `fisheye_pitch_angle` - 全景俯仰角
* `fisheye_fov_angle` -垂直方向的视场角
* `coefficient_horizontal_curve` - 水平曲线系数
* `coefficient_vertical_curve` - 垂直曲线系数
* `fisheye_theta1` - 水平旋转角度
* `fisheye_theta2` - 光学轴旋转角度
* `fisheye_translation1` - 3D转换参数1
* `fisheye_translation2` - 3D转换参数2
* `fisheye_translation3` - 3D转换参数3
* `fisheye_center_x2` - 镜头光学中心X
* `fisheye_center_y2` - 镜头光学中心Y
* `fisheye_rotate_angle2` - 镜头的旋转角度
* `debug_info` - 调试信息

![](image/api/fce-anger.jpg)
![](image/api/fce-anger-scope.jpg)

---
### 6.2.22 cam_fcedata_param_t
定义矫正数据的数据结构

**定义**
```cpp
typedef struct __cam_fcedata_param_t
{
    cam_fcedata_common_t common;
    cam_fisheye_correction_t fec;
} cam_fcedata_param_t;
```
**成员**

* `common` - 通用数据头，参见[cam_fcedata_common_t](main.md#6.2.20_cam_fcedata_common_t)
* `fec` - 矫正数据结构，参见[cam_fisheye_correction_t](main.md#6.2.21_cam_fisheye_correction_t)

---
### 6.2.23 cam_fce_port_type_e
定义端口类型数据结构

**定义**
```cpp
typedef enum __cam_fce_port_type_e
{
    CAM_FCE_PORT_NONE,
    CAM_FCE_PORT_UO,
    CAM_FCE_PORT_SS0,
    CAM_FCE_PORT_SS1,
    CAM_FCE_PORT_DN,
    CAM_FCE_PORT_MAX
} cam_fce_port_type_e;
```
**成员**

* `CAM_FCE_PORT_NONE` - 无
* `CAM_FCE_PORT_UO` - `uo`输出
* `CAM_FCE_PORT_SS0` - `ss0`输出
* `CAM_FCE_PORT_SS1` - `ss1`输出
* `CAM_FCE_PORT_DN` - `dn`输出
* `CAM_FCE_PORT_MAX` - 最大支持类型

---
### 6.2.24 cam_position_t
定义矫正后图像的位置数据结构

**定义**
```cpp
typedef struct __cam_position_t
{
    int x;
    int y;
    int width;
    int height;
} cam_position_t;

```
**成员**

* `x` - 矫正后图像的输出X位置坐标
* `y` - 矫正后图像的输出Y位置坐标
* `width` - 矫正后图像的输出宽度
* `height` - 矫正后图像的输出高

---
### 6.2.25 cam_fcefile_port_t
定义端口属性数据结构

**定义**
```cpp
typedef struct __cam_fcefile_port_t
{
    cam_fce_port_type_e type;
    cam_position_t pos;
} cam_fcefile_port_t;
```
**成员**

* `type` - 端口类型，参见[cam_fce_port_type_e](main.md#6.2.23_cam_fce_port_type_e)
* `pos` - 输出位置，参见[cam_position_t](main.md#6.2.24_cam_position_t)

---
### 6.2.26 cam_fcefile_file_t
定义视图模式数据结构

**定义**
```cpp
typedef struct __cam_fcefile_file_t
{
    char file_grid[128];
    char file_geo[128];
    int outport_number;
    cam_fcefile_port_t outport_list[CAM_FCE_PORT_MAX - 1];
} cam_fcefile_file_t;
```
**成员**

* `file_grid` - 矫正数据文件路径（含文件名）
* `file_geo` - 矫正数据映射文件路径（含文件名）
* `outport_number` - 端口个数，为0时，API内部会产生默认的端口列表值
* `outport_list` - 端口列表，参见[cam_fcefile_port_t](main.md#6.2.25_cam_fcefile_port_t)

---
### 6.2.27 cam_fcefile_mode_t
定义视图模式数据结构

**定义**
```cpp
typedef struct __cam_fcefile_mode_t
{
    int file_number;
    cam_fcefile_file_t file_list[MAX_CAM_FCEFILE_LIST];
} cam_fcefile_mode_t;
```
**成员**

* `file_number` - 视图文件个数
* `file_list` - 视图文件列表，参见[cam_fcefile_file_t](main.md#6.2.26_cam_fcefile_file_t)

---
### 6.2.28 cam_fcefile_param_t
定义矫正数据文件数据结构

**定义**
```cpp
#define MAX_CAM_FCEFILE_LIST       4

typedef struct __cam_fcefile_param_t
{
    int mode_number;
    cam_fcefile_mode_t mode_list[MAX_CAM_FCEFILE_LIST];
} cam_fcefile_param_t;
```
**成员**

* `mode_number` - 视图模式个数
* `mode_list` - 视图模式列表，参见[cam_fcefile_mode_t](main.md#6.2.27_cam_fcefile_mode_t)

---
### 6.2.29 cam_save_fcedata_param_t
定义鱼眼矫正数据结构，用于保存到文件

**定义**
```cpp
typedef struct __cam_save_fcedata_param_t
{
    char file_name[100];
} cam_save_fcedata_param_t;
```
**成员**

* `file_name` - 保存文件名（含路径）

---
## 7 图像编解码API及数据类型

---
## 7.1 API

----
### 7.1.1 video_set_basicinfo()
设置编码后视频的一些基本信息，如媒体类型MJPEG/H264/H265，Profile以及编码流的格式

**语法**
```cpp
int video_set_basicinfo(const char *channel, struct v_basic_info *info);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `info` - `IN`，视频流基本信息结构体，参见[v_basic_info](main.md#7.2.1_v_basic_info)

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
无

---
### 7.1.2 video_get_basicinfo()
获取编码后视频的一些基本信息，如媒体类型MJPEG/H264/H265，Profile以及编码流的格式

**语法**
```cpp
int video_get_basicinfo(const char *channel, struct v_basic_info *info);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `info` - `OUT`，视频流基本信息结构体，参见[v_basic_info](main.md#7.2.1 v_basic_info)

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
无

---
### 7.1.3 video_set_ratectrl()
设置码率控制信息，如VBR/CBR/FixQp，码率，QP范围等

**语法**
```cpp
int video_set_ratectrl(const char *channel, struct v_rate_ctrl_info *info);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `info` - `IN`，码率控制信息结构体，参见[v_rate_ctrl_info](main.md#7.2.5_v_rate_ctrl_info)

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
设置CBR模式，目标码率3Mbps，假设编码器实例名`"enc_1080p"`
```cpp
struct v_rate_ctrl_info stRateCtrl;
int s32Ret;

stRateCtrl.rc_mode = VENC_CBR_MODE;
stRateCtrl.qp_max = 45;
stRateCtrl.qp_min = 22;
stRateCtrl.qp_delta = -2;
stRateCtrl.qp_hdr = -1;
stRateCtrl.bitrate = 3000000;
stRateCtrl.picture_skip = 0;
s32Ret = video_set_ratectrl("enc_1080p-stream", &stRateCtrl);
```
---
### 7.1.4 video_set_ratectrl()
设置码率控制信息，如VBR/CBR/FixQp，码率，QP范围等

**语法**
```cpp
int video_get_ratectrl(const char *channel, struct v_rate_ctrl_info *info);
```
**参数**
* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `info` - `OUT`，码率控制信息结构体，参见[v_rate_ctrl_info](main.md#7.2.5_v_rate_ctrl_info)

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
获取当前码率控制信息，假设编码器实例名`"enc_1080p"`
```cpp
struct v_rate_ctrl_info stRateCtrl;
int s32Ret;

s32Ret = video_get_ratectrl("enc_1080p-stream", &stRateCtrl);
```

---
### 7.1.5 video_set_roi()
设置图像的感兴趣区域

**语法**
```cpp
int video_set_roi(const char *channel, struct v_roi_info *roi);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `roi` - `IN`，感兴趣区域信息结构体指针，参见[v_roi_info](main.md#7.2.10_v_roi_info)

**返回值**

* 0 - 成功
* 非0 - 失败

Note: 最大支持设置2个感兴趣区域

**示例**
设置1080P画面左上角和右下角，两个960x540区域为感兴趣区域，假设编码器实例名`"enc_1080p"`
```cpp
struct v_roi_info stROIInfo;
int s32Ret;

// 设置左上角为感兴趣区域
stROIInfo.roi[0].enable = 1;
stROIInfo.roi[0].w = 960;
stROIInfo.roi[0].h = 540;
stROIInfo.roi[0].x = 0;
stROIInfo.roi[0].y = 0;
stROIInfo.roi[0].qp_offset = -3;

// 设置右下角为感兴趣区域
stROIInfo.roi[1].enable = 1;
stROIInfo.roi[1].w = 960;
stROIInfo.roi[1].h = 540;
stROIInfo.roi[1].x = 960;
stROIInfo.roi[1].y = 540;
stROIInfo.roi[1].qp_offset = -3;

s32Ret = video_set_roi("enc_1080p-stream", &stROIInfo);
```

---
### 7.1.6 video_get_roi()
获取感兴趣区域信息

**语法**
```cpp
int video_get_roi(const char *channel, struct v_roi_info *roi);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `roi` - `OUT`，感兴趣区域信息结构体指针，参见[v_roi_info](main.md#7.2.10_v_roi_info)

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
获取感兴趣区域参数信息，假设编码器实例名`"enc_1080p"`
```cpp
struct v_roi_info stROIInfo;
int s32Ret;

s32Ret = video_get_roi("enc_1080p-stream", &stROIInfo);
```

---
### 7.1.7 video_get_roi_count()
获取系统支持的最大感兴趣区域个数

**语法**
```cpp
int video_get_roi_count(const char *channel);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 返回支持的最大感兴趣区域个数
* 小于0 - 失败

**示例**
获取感兴趣区域参数信息，假设编码器实例名`"enc_1080p"`
```cpp
int s32Ret;

s32Ret = video_get_roi_count("enc_1080p-stream");
```

---
### 7.1.8 video_disable_channel()
关闭一条数据通道

**语法**
```cpp
int video_disable_channel(const char *channel);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 0 - 成功
* 非0 - 失败

Note: 关闭的端口必须是输出端口

**示例**
以双路编码为例，关闭`"enc-vga"`，此时仅`"enc_1080p"`一路在编码，再使能`"enc-vga"`

![](image/path/dual-enc.svg)

```cpp
int s32Ret;

s32Ret = video_disable_channel("enc_vga-stream");

// ...

s32Ret = video_enable_channel("enc_vga-stream");
```

---
### 7.1.9 video_enable_channel()
使能一条数据通道

**语法**
```cpp
int video_enable_channel(const char *channel);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
参见[video_disable_channel()](main.md#7.1.8_video_disable_channel%28%29)

---
### 7.1.10 video_trigger_key_frame()
触发编码器输出一个I帧。常用于发生错误时，重新开始一个同步点

**语法**
```cpp
int video_trigger_key_frame(const char *chn);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 0 - 成功
* 非0 - 失败

**示例**
触发编码器输出一个I帧，然后获取这个I帧，假设编码器实例名`"enc_1080p"`
```cpp
struct fr_buf_info stFrameInfo;
int s32Ret = 0;

s32Ret = video_trigger_key_frame("enc_1080p-stream");
while (1)
{
    s32Ret = video_get_frame("enc_1080p-stream", &stFrameInfo);
    if (stFrameInfo.priv == VIDEO_FRAME_I)
    {
        break;
    }
    else
    {
        s32Ret = video_put_frame("enc_1080p-stream", &stFrameInfo);
    }
}

// ...
```

---
### 7.1.11 video_get_header()
获取视频序列的SPS，PPS等头信息

**语法**
```cpp
int video_get_header(const char *channel, char *buf, int len);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `buf` - `IN/OUT`，保存视频头信息的缓存起始地址
* `len` - `IN`，保存视频头信息的缓存大小

**返回值**

* 大于0 - 表示视频头的字节数
* 其它 - 失败

**示例**
获取视频头信息，假设编码器实例名`"enc_1080p"`
```cpp
unsigned char u8Header[256];
int s32Len = 0;

s32Len = video_get_header("enc_1080p-stream", u8Header, 128);

// ...
```

---
### 7.1.12 video_set_header()
设置视频序列的SPS，PPS等头信息，此接口仅用于解码器

**语法**
```cpp
int video_set_header(const char *channel, char *buf, int len)
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`g1264`实例化名字叫`"dec_1080p"`，端口名为`frame`，则对应的`channel`值为`"dec_1080p-frame"`
* `buf` - `IN`，视频头信息的缓存起始地址
* `len` - `IN`，视频头信息的长度

**返回值**

* 1 - 成功
* 其它 - 失败

**示例**
无

---
### 7.1.13 video_set_fps()
设置视频通道的帧率

**语法**
```cpp
 int video_set_fps(const char *channel, int fps);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `fps` - `IN`，目标帧率

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

Note: 当前帧率是30或者25时，支持[1, 30]或[1, 25]之间的任意帧率设定。当前帧率是其它帧率时，只支持帧率下降整数倍。不支持帧率放大

---
### 7.1.14 video_get_fps()
获取视频通道的帧率

**语法**
```cpp
 int video_set_fps(const char *channel);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 大于0 - 通道帧率
* -1 - 失败

**示例**
无

---
### 7.1.15 video_set_snap()
设置拍照时的裁剪区域

**语法**
```cpp
 int video_set_snap(const char *channel, struct v_pip_info *pip_info);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`h1jpeg`实例化名字叫`"jpeg"`，端口名为`out`，则对应的`channel`值为`"jpeg-out"`
* `pip_info` - `IN`，裁剪区域数据结构，参见[v_pip_info](main.md#7.2.12_v_pip_info)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
截取1080P输入图像的[0，0，640，320]区域编码

```cpp
struct v_stPipInfo stPipInfo;
struct fr_buf_info stFrameInfo;
int s32Ret = 0;

// 原始图像1920*1080，设置编码（0，0，640，320）区域
stPipInfo.pic_w = 1920;
stPipInfo.pic_h = 1080;
stPipInfo.x = 0;
stPipInfo.y = 0;
stPipInfo.w = 640;
stPipInfo.h = 320;
s32Ret = video_set_snap("jpeg-out", &stPipInfo);

// 获取一张照片
s32Ret = video_get_snap("jpeg-out", &stFrameInfo);

// ...

// 释放缓存
s32Ret = video_put_snap("jpeg-out", &stFrameInfo);
```

---
### 7.1.16 video_get_snap()
拍摄并获取一张照片

**语法**
```cpp
int video_get_snap(const char *channel, struct fr_buf_info *ref);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`h1jpeg`实例化名字叫`"jpeg"`，端口名为`out`，则对应的`channel`值为`"jpeg-out"`
* `ref` - `OUT`，照片存放缓存，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
参见[video_set_snap()](main.md#7.1.15_video_set_snap%28%29)

---
### 7.1.17 video_put_snap()
释放缓存，必须与[video_get_snap()](main.md#7.1.16_video_get_snap%28%29)搭配使用

**语法**
```cpp
int video_put_snap(const char *channel, struct fr_buf_info *ref);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`h1jpeg`实例化名字叫`"jpeg"`，端口名为`out`，则对应的`channel`值为`"jpeg-out"`
* `ref` - `IN`，照片存放缓存，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
参见[video_set_snap()](main.md#7.1.15_video_set_snap%28%29)

---
### 7.1.18 video_set_pip_info()
设置视频通道的裁剪区域

**语法**
```cpp
int video_set_pip_info(const char *channel, struct v_pip_info *pip_info);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `pip_info` - `IN`，裁剪区域数据结构，参见[v_pip_info](main.md#7.2.12_v_pip_info)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
编码器将1080P输入画面裁减到[0，0，1280，720]位置再编码

```cpp
struct v_stPipInfo stPipInfo;
int s32Ret = 0;

stPipInfo.x = 0;
stPipInfo.y = 0;
stPipInfo.w = 1280；
stPipInfo.h = 720;

s32Ret = video_set_pip_info("enc_1080p-stream", &stPipInfo);
```

---
### 7.1.19 video_get_pip_info()
获取视频通道的裁剪参数信息

**语法**
```cpp
 int video_get_pip_info(const char *channel, struct v_pip_info *pip_info)
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `pip_info` - `OUT`，裁剪区域数据结构，参见[v_pip_info](main.md#7.2.12_v_pip_info)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 7.1.20 video_test_frame()
视频通道中检测当前帧是否更新，可与[video_get_frame()](main.md#7.1.21_video_get_frame%28%29)配合使用
通过先检测当前帧是否更新，再调用[video_get_frame()](main.md#7.1.21_video_get_frame%28%29)获取当前帧，达到非阻塞调用的目的

**语法**
```cpp
int video_test_frame(const char *channel, struct fr_buf_info *ref);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`g1264`实例化名字叫`"dec_1080p"`，端口名为`frame`，则对应的`channel`值为`"dec_1080p-frame"`
* `ref` - `IN`，参考帧信息，用于当前帧是否更新的参考，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 大于0 - 当前帧更新
* 0 - 当前帧未更新

**示例**
两路解码，获取解码好的帧数据，不能阻塞，无论那一路准备好，立刻取走，假设两个解码器实例化名字是`"dec1_1080p"`和`"dec2_1080p"`

```cpp
struct fr_buf_info stFrameInfo;
int s32Ret = 0;

while (1)
{
    s32Ret = video_test_frame("dec1_1080p", &stFrameInfo);
    if (s32Ret > 0)
    {
        video_get_frame("dec1_1080p", &stFrameInfo);
        break;		        
    }
    s32Ret = video_test_frame("dec2_1080p", &stFrameInfo);
    if (s32Ret > 0)
    {
        video_get_frame("dec2_1080p", &stFrameInfo);
        break;		        
    }
}

// ...

video_put_frame("dec2_1080p", &stFrameInfo);

```

---
### 7.1.21 video_get_frame()
获取视频通道输出端口的帧数据

**语法**
```cpp
int video_get_frame(const char *channel, struct fr_buf_info *ref);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `ref` - `IN`，参考帧信息，用于当前帧是否更新的参考，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 大于0 - 成功
* 小于0 - 失败

Note: 调用[video_get_frame()](main.md#7.1.21_video_get_frame%28%29)获取缓存之后必须调用[video_put_frame()](main.md#7.1.22_video_put_frame%28%29)归还

**示例**
从编码器输出端口获取编码后数据
```cpp
struct fr_buf_info stFrameInfo;
int s32Ret = 0;

s32Ret = video_get_frame("enc_1080p-stream", &stFrameInfo);

// ...

s32Ret = video_put_frame("enc_1080p-stream", &stFrameInfo);

```

---
### 7.1.22 video_put_frame()
释放获取的帧缓存，必须在调用[video_get_frame()](main.md#7.1.21_video_get_frame%28%29)之后使用

**语法**
```cpp
int video_put_frame(const char *channel, struct fr_buf_info *ref);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `ref` - `IN`，释放缓存的指针，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 大于0 - 成功
* 小于0 - 失败

**示例**
参见[video_get_frame()](main.md#7.1.21_video_get_frame%28%29)

---
### 7.1.23 video_set_frc()
设置编码器的输出帧率，可实现快速摄影，延迟摄影和插帧

**语法**
```cpp
int video_set_frc(const char *channel, struct v_frc_info *frc);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `frc` - `IN`，帧率转换输入信息，参见[v_frc_info](main.md#7.2.14_v_frc_info)

**返回值**

* 1 - 成功
* -1 - 失败

Note: 与[video_set_fps()](main.md#7.1.13_video_set_fps%28%29)区别在于，[video_set_frc()](main.md#7.1.23_video_set_frc%28%29)可以对帧率做放大。无论做放大和缩小，必须是整数倍

**示例**
当前帧率是30帧每秒，将输出帧率翻倍，假设编码器实例名为`"enc_1080p"`
```cpp
struct v_stFrcInfo_info stFrcInfo;
int s32Ret = 0;

stFrcInfo.framerate = 60；
stFrcInfo.framebase = 1；
s32Ret = video_set_stFrcInfo("enc_1080p-stream", &stFrcInfo);
```

---
### 7.1.24 video_get_frc()
获取编码器的输出帧率

**语法**
```cpp
int video_get_frc(const char *channel, struct v_frc_info *frc);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `frc` - `OUT`，帧率转换输入信息，参见[v_frc_info](main.md#7.2.14_v_frc_info)

**返回值**

* 1 - 成功
* -1 - 失败

**示例**
无

---
### 7.1.25 video_set_frame_mode()
设置编码后的视频头的传输模式

**语法**
```cpp
int video_set_frame_mode(const char *channel, enum v_enc_frame_type mode);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `mode` - `IN`，帧模式，参见[v_enc_frame_type](main.md#7.2.15_v_enc_frame_type)

**返回值**

* 1 - 成功
* -1 - 失败

**示例**
在每一个IDR帧的前面都添加视频头信息
```cpp
video_set_frame_mode("enc_1080p-stream", VENC_HEADER_IN_FRMAE_MODE);
```

---
### 7.1.26 video_get_frame_mode()
获取编码后的视频头的传输模式

**语法**
```cpp
enum v_enc_frame_type video_get_frame_mode(const char *channel);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 7.1.27 video_set_resolution()
设置指定端口图像分辨率

**语法**
```cpp
int video_set_resolution(const char *channel, int w, int h);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`h1jpeg`实例化名字叫`"jpeg"`，端口名为`out`，则对应的`channel`值为`"jpeg-out"`
* `w` - `IN`，视频端口图像宽度
* `h` - `IN`，视频端口图像高度

**返回值**

* 0 - 成功
* -1 - 失败

Note: 目前仅用于`v2500`接`h1jpeg`做大分辨率拍照的场景

**示例**
无

---
### 7.1.28 video_get_resolution()
获得指定端口图像分辨率

**语法**
```cpp
int video_get_resolution(const char *channel, int *w, int *h);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `w` - `OUT`，视频端口图像宽度
* `h` - `OUT`，视频端口图像高度

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 7.1.29 video_write_frame()
往指定端口写入数据，常用于往解码器写入视频编码流

**语法**
```cpp
int video_write_frame(const char *channel, void *data, int len);
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`g1264`实例化名字叫`"dec_1080p"`，端口名为`stream`，则对应的`channel`值为`"dec_1080p-stream"`
* `w` - `OUT`，视频端口图像宽度
* `h` - `OUT`，视频端口图像高度

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
解析MP4文件，并通过`g1264`解码视频，假设解码器实例名为`"dec_1080p"`
```cpp
struct demux_t *stDmx;
struct demux_frame_t stFrm;
char ps8Header[128];
char* ps8FilePath="/mnt/sd0/1.mp4";
int s32Ret;

// 初始化demux，用于解析MP4文件
stDmx = demux_init(ps8FilePath);

// 获取视频头信息
s32Ret = demux_get_head(stDmx, ps8Header, 128);

// 将视频头信息设置给解码器
video_set_header("dec_1080p-stream", ps8Header, s32Ret);

// 从MP4文件中获取编码数据
while ((s32Ret = demux_get_frame(stDmx, &stFrm)) == 0)
{
    if (stFrm.codec_id != DEMUX_VIDEO_H264)
    {
        demux_put_frame(stDmx, &stFrm);
        continue;
    }

    // 往g1264的stream端口写数据
    video_write_frame("dec_1080p-stream", stFrm.data, stFrm.data_size);

    // 释放数据缓存
    demux_put_frame(stDmx, &stFrm);
}

// 释放demux
demux_deinit(stDmx);
```

---
### 7.1.30 video_smartrc_start()
开启SmartRc智能码率控制模块

**语法**
```cpp
EN_VIDEOBOX_RET video_smartrc_start(ST_SMARTRC_MONITER_PARAMETER* pstParm)
```
**参数**

* `pstParm` - `IN`，SmartRc智能码率控制模块内部子功能开关，参见[ST_SMARTRC_MONITER_PARAMETER](main.md#7.2.16_ST_SMARTRC_MONITER_PARAMETER)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
无

---
### 7.1.31 video_smartrc_set_mode_params()
设置SmartRc智能码率控制不同场景多个级别的码率控制参数

**语法**
```cpp
EN_VIDEOBOX_RET video_smartrc_set_mode_params(const char *ps8Chn, ST_MODE_RC_SETTINGS *pstSettings)
```
**参数**

* `channel` - `IN`，视频通道名，IPU实例化名字加端口名。假设`vencoder`实例化名字叫`"enc_1080p"`，端口名为`stream`，则对应的`channel`值为`"enc_1080p-stream"`
* `pstSettings` - `IN`，SmartRc智能码率控制参数 ，参见[ST_MODE_RC_SETTINGS](main.md#7.2.19_ST_MODE_RC_SETTINGS)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
无

---
### 7.1.32 video_smartrc_set_params()
设置SmartRc模块通用参数以及不同场景切换的比例参数

**语法**
```cpp
EN_VIDEOBOX_RET video_smartrc_set_params(ST_SMARTRC_RC_PARAMETER *pstSettings)
```
**参数**

* `pstSettings` - `IN`，SmartRc智能码率控制参数 ，参见[ST_SMARTRC_RC_PARAMETER](main.md#7.2.20_ST_SMARTRC_RC_PARAMETER)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
无

---
### 7.1.33 video_smartrc_set_moniters()
设置SmartRc智能码率控制子功能开关

**语法**
```cpp
EN_VIDEOBOX_RET video_smartrc_set_moniters(ST_SMARTRC_MONITER_PARAMETER *pstSettings)
```
**参数**

* `pstSettings` - `IN`，SmartRc智能码率控制参数 ，参见[ST_SMARTRC_MONITER_PARAMETER](main.md#7.2.16_ST_SMARTRC_MONITER_PARAMETER)

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**
无

---
### 7.1.34 video_smartrc_stop()
关闭SmartRc智能码率控制模块

**语法**
```cpp
EN_VIDEOBOX_RET video_smartrc_stop()
```
**参数**

无

**返回值**

* 0 - 成功
* 小于0 - 失败

**示例**

无

---
## 7.2 数据类型

### 7.2.1 v_basic_info
定义视频流基本信息
**定义**
```cpp
struct v_basic_info {
    enum v_media_type   media_type ;
    enum v_enc_profile profile;
    enum v_stream_type stream_type;
};
```
**成员**

* `media_type` - 视频流编码方式枚举变量，参见[v_media_type](main.md#7.2.2_v_media_type)
* `profile` - 视频流的编码Profile枚举变量，参见[v_enc_profile](main.md#7.2.3_v_enc_profile)
* `stream_type` - 视频流类型枚举变量，参见[v_stream_type](main.md#7.2.4_v_stream_type)

----
### 7.2.2 v_media_type
定义编码类型

**定义**
```cpp
enum v_media_type {
    VIDEO_MEDIA_NONE,
    VIDEO_MEDIA_MJPEG,
    VIDEO_MEDIA_H264 ,
    VIDEO_MEDIA_HEVC
};
```
**成员**

* `VIDEO_MEDIA_NONE` - 无编码类型
* `VIDEO_MEDIA_MJPEG` - MJPEG编码
* `VIDEO_MEDIA_H264` - H264编码
* `VIDEO_MEDIA_HEVC` - H265编码

----
### 7.2.3 v_enc_profile
定义编码Profile类型

**定义**
```cpp
enum v_enc_profile {
    VENC_BASELINE,
    VENC_MAIN,
    VENC_HIGH,
};
```
**成员**

* `VENC_BASELINE` - Baseline profile
* `VENC_MAIN` - Main profile
* `VENC_HIGH` - High profile

----
### 7.2.4 v_stream_type
定义编码后流的类型

**定义**
```cpp
enum v_stream_type {
    VENC_BYTESTREAM,
    VENC_NALSTREAM,
};
```
**成员**

* `VENC_BYTESTREAM` - AnnexB流格式，带0x000001的起始码
* `VENC_NALSTREAM` - NAL格式，无0x000001的起始码

----
### 7.2.5 v_rate_ctrl_info
定义码率控制参数信息

**定义**
```cpp
struct v_rate_ctrl_info {
    enum v_rate_ctrl_type rc_mode;
    int gop_length;
    int picture_skip;
    int idr_interval;
    int hrd;
    int hrd_cpbsize;
    int refresh_interval;
    int mbrc;
    int mb_qp_adjustment;
    union {
        struct v_fixqp_info fixqp;
        struct v_vbr_info   vbr;
        struct v_cbr_info   cbr;
    };
};
```
**成员**

* `rc_mode` - 码率控制模式，参见[v_rate_ctrl_type](main.md#7.2.6_v_rate_ctrl_type)
* `gop_length` - GOP长度，码率计算的基本长度，取值范围[1，150]
* `picture_skip` - 码率超过目标码率时，是否允许丢帧。1 - 丢帧，0 - 不丢帧。缺省值为0
* `idr_interval` - 编码时I帧间隔。取值范围[1，150]，缺省值为输入帧率
* `hrd` - HRD功能使能开关。1 - 使能，0 - 关闭。缺省值为0
* `hrd_cpbsize` - 使能HRD开关时，编码缓存区大小，建议大于2秒数据量，否则容易连续掉帧
* `refresh_interval` - 智能P帧间隔，取值范围[0，idr_interval]，适用于`Apollo-2`
* `mbrc` - 宏块级码率控制使能开关，1 - 使能，0 - 关闭。缺省值为0，适用于`Apollo-ECO`
* `mb_qp_adjustment` - 宏块码率控制等级，范围[-8，7]，适用于`Apollo-2`
* `fixqp` - 固定QP配置参数，参见[v_fixqp_info](main.md#7.2.7_v_fixqp_info)
* `vbr` - VBR配置参数，参见[v_vbr_info](main.md#7.2.8_v_vbr_info)
* `cbr` - CBR配置参数，参见[v_cbr_info](main.md#7.2.9_v_cbr_info)

----
### 7.2.6 v_rate_ctrl_type
定义码率控制参数信息

**定义**
```cpp
enum v_rate_ctrl_type {
    VENC_CBR_MODE,
    VENC_VBR_MODE,
    VENC_FIXQP_MODE,
};
```
**成员**

* `VENC_CBR_MODE` - CBR模式
* `VENC_VBR_MODE` - VBR模式
* `VENC_FIXQP_MODE` - FixQp模式

----
### 7.2.7 v_fixqp_info
定义固定QP的数据结构

**定义**
```cpp
struct v_fixqp_info {
    int qp_fix;
};
```
**成员**

* `qp_fix` - 采用FixQp模式时，使用的QP值，范围[2, 51]

----
### 7.2.8 v_vbr_info
定义VBR的数据结构

**定义**
```cpp
struct v_vbr_info {
    int qp_max;
    int qp_min;
    int max_bitrate;
    int threshold;
    int qp_delta;
};
```
**成员**

* `qp_max` - 编码QP的最大值，取值范围[2，51]
* `qp_min` - 编码QP的最小值，取值范围[2，51]
* `max_bitrate` - VBR编码时最大码率，取值范围[10000，60000000]，单位bps
* `threshold` - 限制VBR编码时码率下限，下限值为max_bitrate * threshold / 100，取值范围[1，99]
* `qp_delta` - 调整I帧QP，取值范围[-12，12]，用于调整I帧的相对质量或是I帧占比

----
### 7.2.9 v_cbr_info
定义CBR的数据结构

**定义**
```cpp
struct v_cbr_info {
    int qp_max;
    int qp_min;
    int bitrate;
    int qp_delta;
    int qp_hdr;
};
```
**成员**

* `qp_max` - 编码QP的最大值，取值范围[2，51]
* `qp_min` - 编码QP的最小值，取值范围[2，51]
* `bitrate` - CBR编码目标码率，取值范围[10000，60000000]，单位bps
* `qp_delta` - 调整I帧QP，取值范围[-12，12]，用于调整I帧的相对质量或是I帧占比
* `qp_hdr` - CBR编码起始QP值，取值范围[2，51]。设置为-1时，由编码算法自动选择

---
### 7.2.10 v_roi_info
定义感兴趣区域数据结构

**定义**
```cpp
struct v_roi_info {
    struct v_roi_area roi[2];
};
```
**成员**

* `roi` - 感兴趣区域数据结构，最大支持2个，参见[v_roi_area](main.md#7.2.11_v_roi_area)

---
### 7.2.11 v_roi_area
定义感兴趣区域数据结构

**定义**
```cpp
struct v_roi_area {
    int enable;
    int x, y;
    int w, h;
    int qp_offset;
};
```
**成员**

* `enable` - 感兴趣区域使能，1 - 使能，0 - 关闭，缺省值为0
* `x` - 感兴趣区域x方向起始坐标，图像左上角为原点
* `y` - 感兴趣区域y方向起始坐标，图像左上角为原点
* `w` - 感兴趣区域宽度
* `h` - 感兴趣区域高度
* `qp_offset` - 感兴趣区域QP相对画面的偏移，为负数时提高区域画质，为正时减少区域画质

Note: 感兴趣区域不得超过输入画面大小

---
### 7.2.12 v_pip_info
定义裁剪区域数据结构

**定义**
```cpp
struct v_pip_info{
    char portname[16];
    int x, y;
    int w, h;
    int pic_w, pic_h;
};
```
**成员**

* `portname` - 端口名
* `x` - 裁剪区域x坐标，图像左上角为原点
* `y` - 裁剪区域y坐标，图像左上角为原点
* `w` - 裁剪区域宽度
* `h` - 裁剪区域高度
* `pic_w` - 原始图像宽度
* `pic_h` - 原始图像高度

---
### 7.2.13 fr_buf_info
定义帧缓存数据结构

**定义**
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
    struct fr_buf_info *next, *prev;
};
```
**成员**

* `fr` - 指向内核中的帧数据结构，应用层禁止更改
* `buf` - 指向内核中的帧缓存数据结构，应用层禁止更改
* `virt_addr` - 缓存的虚拟地址，应用层禁止更改
* `phys_addr` - 缓存的物理地址，应用层禁止更改
* `timestamp` - 缓存的时间戳信息
* `size` - 缓存大小，应用层禁止更改
* `map_size` - 缓存映射大小，应用层禁止更改
* `serial` - 缓存更新计数，应用层禁止更改
* `priv` - 私有数据
* `next` - 指向链表中下一个fr_buf_info，应用层禁止更改
* `prev` - 指向链表中上一个fr_buf_info，应用层禁止更改

Note: 此数据结构用于IPU之间或者与用户进行数据交换。用户层可对`virt_addr`指向的缓存做读写，设置时间戳`timestamp`，添加私有数据`priv`，其它数据结构均不能修改

---
### 7.2.14 v_frc_info
定义帧率转换信息数据结构

**定义**
```cpp
struct v_frc_info {
    int framebase;
    int framerate;
}
```

**成员**

* `framebase` - 帧率基准
* `framerate` - 帧率

Note: 实际帧率 = framerate / framebase

---
### 7.2.15 v_enc_frame_type
定义视频头的保存方式

**定义**
```cpp
enum v_enc_frame_type {
    VENC_HEADER_NO_IN_FRAME_MODE = 0,
    VENC_HEADER_IN_FRAME_MODE,
};
```

**成员**

* `VENC_HEADER_NO_IN_FRAME_MODE` - 只在第一个IDR帧前面添加视频头信息
* `VENC_HEADER_IN_FRAME_MODE` - 每个IDR帧前面均添加视频头信息

---
### 7.2.16 ST_SMARTRC_MONITER_PARAMETER
SmartRc子功能开关

**定义**
```cpp
typedef struct v_smartrc_moniter_parameter {
    int s32DynamicROIEnable;
    int s32RBPSMonitorEnable;
    int s32MoveMoniterEnable;
    int s32LightMoniterEnable;
} ST_SMARTRC_MONITER_PARAMETER;
```

**成员**

* `s32DynamicROIEnable` - SmartRc模块子功能动态编码ROI开关
* `s32RBPSMonitorEnable` - SmartRc模块子功能监测实时码率并允许调整编码参数开关
* `s32MoveMoniterEnable` - SmartRc模块子功能监测运动静止场景变化开关
* `s32LightMoniterEnable` - SmartRc模块子功能监测夜视与白天场景变化开关

Note: 动态编码ROI的正常运行需要同时配合运动检测模块上报的运动区域

---
### 7.2.17 EN_SMARTRC_MODE
SmartRc模块定义的四种自动监测场景，用于设置不同场景时码率控制参数

**定义**
```cpp
typedef enum v_smartrc_mode {
    EN_SMARTRC_DAY_MOVE_MODE,
    EN_SMARTRC_DAY_STILL_MODE,
    EN_SMARTRC_NIGHT_MOVE_MODE,
    EN_SMARTRC_NIGHT_STILL_MODE,
    EN_SMARTRC_MAX_MODE,
} EN_SMARTRC_MODE;
```

**成员**

* `EN_SMARTRC_DAY_MOVE_MODE` - 白天运动模式
* `EN_SMARTRC_DAY_STILL_MODE` - 白天静止模式
* `EN_SMARTRC_NIGHT_MOVE_MODE` - 夜视运动模式
* `EN_SMARTRC_NIGHT_STILL_MODE` - 夜视静止模式

---
### 7.2.18 ST_VIDEO_BR_INFO
SmartRc模块定义单个码率级别可控制的码率参数

**定义**
```cpp
typedef struct v_br_info {
    int s32Bitrate;
    int s32QpDelta;
    int s32QpMin;
    int s32QpMax;
} ST_VIDEO_BR_INFO;
```

**成员**

* `s32Bitrate` - 对应等级的码率，单位bps，该值只做为不同码率等级之间的切换比较值，不设定给编码器
* `s32QpDelta` - 参见[v_cbr_info](main.md#7.2.9_v_cbr_info)参数`qp_delta`
* `s32QpMin` - 参见[v_cbr_info](main.md#7.2.9_v_cbr_info)参数`qp_min`
* `s32QpMax` - 参见[v_cbr_info](main.md#7.2.9_v_cbr_info)参数`qp_max`

---
### 7.2.19 ST_MODE_RC_SETTINGS
SmartRc模块定义不同场景下多个码率级别的码率控制参数

**定义**
```cpp
typedef struct v_rate_ctrl_mode_settings {
    EN_SMARTRC_MODE enMode;
    ST_VIDEO_BR_INFO stBrSettings[5];
} ST_MODE_RC_SETTINGS;
```

**成员**

* `enMode` - 场景模式，参见[EN_SMARTRC_MODE](main.md#7.2.17_EN_SMARTRC_MODE)
* `stBrSettings` - 可调码率控制级别的码率参数，参见[ST_VIDEO_BR_INFO](main.md#7.2.18_ST_VIDEO_BR_INFO)

Note: 目前支持最多4路码流，每一路码流支持4种场景模式，每一种场景模式最多支持5个等级

---
### 7.2.20 ST_SMARTRC_RC_PARAMETER
SmartRc模块定义通用参数以及不同场景切换的比例参数

**定义**
```cpp
typedef struct v_smartrc_parameter {
    int s32StreamNum;
    int s32LevelNum;
    int s32MVCheckInterval;
    int s32DayToNightBRPercentage;
    int s32MoveToStillBRPercentage;
    int s32LevelChangeBRPercentThreshold;
} ST_SMARTRC_RC_PARAMETER;
```

**成员**

* `s32StreamNum` - SmartRc控制的码流数
* `s32LevelNum` - 每一个场景下的等级个数
* `s32MVCheckInterval` - 未监测到运动场景时动态场景自动切换为静态场景的时间间隔，单位秒
* `s32DayToNightBRPercentage` - 夜视运动场景相对于白天运动场景的码率百分比
* `s32MoveToStillBRPercentage` - 白天静态场景相对于白天动态场景的码率百分比
* `s32LevelChangeBRPercentThreshold` - 码率实时监测时溢出阈值百分比

Note: `s32LevelChangeBRPercentThreshold`取值必须大于100，等于100表示码率必须完全匹配，为了防止频繁干预码率控制，该值建议大于120

---
## 8 水印API及数据类型

---
## 8.1 API

---
### 8.1.1 marker_disable()
关闭水印显示

**语法**
```cpp
int marker_disable(const char *marker);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
假设`marker`实例化名字`"marker0"`
```cpp
marker_disable("marker0");
```

---
### 8.1.2 marker_enable()
使能水印显示

**语法**
```cpp
int marker_enable(const char *marker);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
假设`marker`实例化名字`"marker0"`
```cpp
marker_enable("marker0");
```

---
### 8.1.3 marker_set_string()
设置要显示的字符串，可以为中文，字母，数字组合

**语法**
```cpp
int marker_set_string(const char *marker, const char *str);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字
* `str` - `IN`，显示的字符串

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
假设`marker`实例化名字`"marker0"`，显示字符串`"InfoTM字符串示例2017"`
```cpp
struct font_attr stFontAttr;

memset(&stFontAttr, 0, sizeof(struct font_attr));
sprintf((char *)stFontAttr.ttf_name, "simhei");
stFontAttr.font_color = 0x00ffffff;
stFontAttr.back_color = 0x20000000;

marker_set_mode("marker0", "manual", "%u%Y/%M/%D  %H:%m:%S", &stFontAttr);
marker_set_string("marker0", "InfoTM字符串示例2017");
```

---
### 8.1.4 marker_set_mode()
设置水印要显示的时间字符格式及字体属性。

**语法**
```cpp
int marker_set_mode(const char *marker, const char *mode, const char *fmt, struct font_attr *attr);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字
* `mode` - `IN`，取值为`"auto"`或者`"manual"`，`"auto"`表示自动生成时间字符串，`"manual"`表示自己通过[marker_set_string()](main.md#8.1.3_marker_set_string%28%29)设置字符串
* `fmt` - `IN`，时间字符格式
* `attr` - `IN`，字体属性

**返回值**

* 0 - 成功
* -1 - 失败

Note: 字符串格式`fmt`必须以`%c`，`%u`，`%U`，`%t`当中的一个开头

|参数|说明|
|---|
|`%c`|不采用后续格式|
|`%u`，`%U`|采用世界时钟格式，`mode`必须是`"auto"`才有效|
|`%t`|解析后续字符串获取格式，`mode`必须是`"auto"`才有效|

当以`%t`开头时

|参数|说明|
|---|
|`%Y`，`%y`|输出年信息|
|`%M`|输出月信息|
|`%D`，`%d`|输出日信息|
|`%H`，`%h`|输出小时信息|
|`%m`|输出分钟信息|
|`%S`，`%s`|输出秒信息|

**示例**
假设`marker`实例化名字`"marker0"`，显示时间信息，假设显示格式是`2017/08/10  09:20:45`
```cpp
struct font_attr stFontAttr;

memset(&stFontAttr, 0, sizeof(struct font_attr));
sprintf((char *)stFontAttr.ttf_name, "simhei");
stFontAttr.font_color = 0x00ffffff;
stFontAttr.back_color = 0x20000000;

marker_set_mode("marker0", "auto", "%t%Y/%M/%D  %H:%m:%S", &stFontAttr);
```

---
### 8.1.5 marker_set_picture()
设置从文件中获取的水印数据，数据格式需与目标画面一致

**语法**
```cpp
int marker_set_picture(const char *marker, char *file_path);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字
* `file_path` - `IN`，文件的路径（含文件名）

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
假设`marker`实例化名字`"marker0"`，`"test.nv12"`为800x64的NV12数据
```cpp
marker_set_picture("marker0", "/mnt/sd0/test.nv12");
```

---
### 8.1.6 marker_get_frame()
获取水印缓存数据结构

**语法**
```cpp
int marker_get_frame(const char *marker, struct fr_buf_info *buf);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字
* `buf` - `IN`，缓存信息指针，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 0 - 成功
* -1 - 失败

Note: 获取之后，必须调用[marker_put_frame()](main.md#8.1.7_marker_put_frame%28%29)释放

**示例**
假设`marker`实例化名字`"marker0"`，获取水印缓存信息
```cpp
struct fr_buf_info stFrameInfo;
struct font_attr stFontAttr;

fr_INITBUF(&stFrameInfo, NULL);
memset(&stFontAttr, 0, sizeof(struct font_attr));
sprintf((char *)stFontAttr.ttf_name, "arial");
stFontAttr.font_color = 0x00ffffff;
stFontAttr.back_color = 0x20000000;
marker_set_mode("marker0", "auto", "%t%Y/%M/%D  %H:%m:%S", &stFontAttr);

// 获取缓存
marker_get_frame("marker0", &stFrameInfo);

// ...

// 释放缓存
marker_put_frame("marker0", &stFrameInfo);
```

---
### 8.1.7 marker_put_frame()
释放水印缓存数据结构

**语法**
```cpp
int marker_put_frame(const char *marker, struct fr_buf_info *buf);
```
**参数**

* `marker` - `IN`，`marker`IPU实例化名字
* `buf` - `IN`，缓存信息指针，参见[fr_buf_info](main.md#7.2.13_fr_buf_info)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
参见[marker_get_frame()](main.md#8.1.6_marker_get_frame%28%29)

---
## 8.2 数据类型

### 8.2.1 display_mode
定义显示模式数据结构

**定义**
```cpp
enum display_mode{
    LEFT = 0,
    MIDDLE,
    RIGHT
};
```
**成员**

* `LEFT` - 左对齐
* `MIDDLE` - 居中
* `RIGHT` - 右对齐

---
### 8.2.2 font_attr
定义字体属性数据结构

**定义**
```cpp
struct font_attr {
    char font_size;
    int back_color;
    int font_color;
    char ttf_name[32];
    enum display_mode mode;
};
```
**成员**

* `font_size` - 字体大小
* `back_color` - 背景色
* `font_color` - 前景色
* `ttf_name` - 字符集名称
* `mode` - 显示方式

## 9 智能影像API及数据类型

Note:
`va_move_`开头的接口，采用直方图做运动检测。而`va_mv_`开头的接口，采用运动向量做运动检测
`va_mv_`开头的接口仅用于`Apollo-2`

---
## 9.1 API

---
### 9.1.1 va_move_get_status()
图像分成7x7个区块，可获取指定区块的运动状态

**语法**
```cpp
int va_move_get_status(const char *va, int row, int column);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `row` - `IN`，区块行坐标，取值范围[0，6]
* `column` - `IN`，区块列坐标，取值范围[0，6]

**返回值**

* 0 - 无运动
* 大于0 - 运动强烈程度
* -1 - 失败

**示例**
无

---
### 9.1.2 va_move_set_sensitivity()
设置运动判断的灵敏度，灵敏度过高，会将轻微扰动当做运动的情况。用户可以根据自己的需求，调整灵敏度

**语法**
```cpp
int va_move_set_sensitivity(const char *va, int level);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `level` - `IN`，算法灵敏度级别，取值范围[0，2]，数值越小灵敏度越低，缺省值为1

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 9.1.3 va_move_get_sensitivity()
获取运动判断的灵敏度

**语法**
```cpp
int va_move_get_sensitivity(const char *va);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字

**返回值**

* 大于等于0 - 当前的灵敏度
* -1 - 失败

**示例**
无

---
### 9.1.4 va_move_set_sample_fps()
设置检测的帧率，即每秒检测多少帧

**语法**
```cpp
int va_move_set_sample_fps(const char *va, int target_fps);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `target_fps` - `IN`，运动检测帧率，缺省值为图像帧率

**返回值**

* 大于0 - 返回设置帧率
* -1 - 失败

**示例**
无

---
### 9.1.5 va_move_get_sample_fps()
获取检测的帧率

**语法**
```cpp
int va_move_get_sample_fps(const char *va);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字

**返回值**

* 大于0 - 返回当前检测帧率
* -1 - 失败

**示例**
无

---
### 9.1.6 va_mv_set_blkinfo()
设置图像的分割模式，用于区域运动检测

**语法**
```cpp
int va_mv_set_blkinfo(const char *va, struct blkInfo_t *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `IN`，分割模式，参见[blkInfo_t](main.md#9.2.1_blkInfo_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 9.1.7 va_mv_get_blkinfo()
获取图像的分割模式

**语法**
```cpp
int va_mv_get_blkinfo(const char *va, struct blkInfo_t *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `OUT`，分割模式，参见[blkInfo_t](main.md#9.2.1_blkInfo_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 9.1.8 va_mv_set_roi()
设置运动检测的感兴趣区域

**语法**
```cpp
int va_mv_set_roi(const char *va, struct roiInfo_t *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `IN`，感兴趣区域信息，参见[roiInfo_t](main.md#9.2.3_roiInfo_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无


---
### 9.1.9 va_mv_get_roi()
获取运动检测的感兴趣区域

**语法**
```cpp
int va_mv_get_roi(const char *va, struct roiInfo_t *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `IN`，感兴趣区域信息，参见[roiInfo_t](main.md#9.2.3_roiInfo_t)

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无

---
### 9.1.10 va_mv_set_sample_fps()
设置检测的帧率，即每秒检测多少帧

**语法**
```cpp
int va_mv_set_sample_fps(const char *va, int target_fps);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `target_fps` - `IN`，运动检测帧率，缺省值为图像帧率

**返回值**

* 大于0 - 返回设置帧率
* -1 - 失败

**示例**
无

---
### 9.1.11 va_mv_get_sample_fps()
获取检测的帧率

**语法**
```cpp
int va_mv_get_sample_fps(const char *va);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字

**返回值**

* 大于0 - 返回当前检测帧率
* -1 - 失败

**示例**
无

---
### 9.1.12 va_mv_set_senlevel()
设置移动侦测算法的灵敏度等级

**语法**
```cpp
int va_mv_set_senlevel(const char *va, int level);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `level` - `IN`，灵敏度等级，取值范围[0，5]，低光照建议[0，2]，正常光照建议[3，5]，数值越低灵敏度越低。缺省值为4

**返回值**

* 大于0 - 返回设置的灵敏度等级
* -1 - 失败

**示例**
无

---
### 9.1.13 va_mv_get_senlevel()
获取移动侦测算法的灵敏度等级

**语法**
```cpp
int va_mv_get_senlevel(const char *va);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字

**返回值**

* 大于0 - 返回当前的灵敏度等级
* -1 - 失败

**示例**
无

---
### 9.1.14 va_mv_set_sensitivity()
设置移动侦测算法的灵敏度参数

**语法**
```cpp
int va_mv_set_sensitivity(const char *va, struct MBThreshold *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `IN`，灵敏度参数数据结构，缺省值为`tdlmv` - 32，`tlmv` - 3，`tcmv` - 7，参见[MBThreshold](main.md#9.2.4_MBThreshold)，

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无


---
### 9.1.15 va_mv_get_sensitivity()
获取移动侦测算法的灵敏度参数

**语法**
```cpp
int va_mv_get_sensitivity(const char *va, struct MBThreshold *st);
```
**参数**

* `va` - `IN`，`vamovement`IPU的实例化名字
* `st` - `OUT`，灵敏度参数数据结构，参见[MBThreshold](main.md#9.2.4_MBThreshold)，

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无


---
### 9.1.16 va_fodet_trigger()
在trigger模式下，触发人脸检测的功能

**语法**
```cpp
EN_VIDEOBOX_RET va_fodet_trigger(char *ps8Chn, ST_FODET_RECT_INFO *pstRet);
```
**参数**

* `ps8Chn` - `IN`，`fodet`IPU的实例化名字
* `pstRet` - `OUT`，检测到的人脸的坐标信息，参见[ST_FODET_RECT_INFO](main.md#9.2.5_ST_FODET_RECT_INFO)，

**返回值**

* 0 - 成功
* -1 - 失败

**示例**
无
---
## 9.2 数据类型

---
### 9.2.1 blkInfo_t
定义图像分割数据结构

**定义**
```cpp
struct blkInfo_t {
    uint32_t blk_x, blk_y;
};
```
**成员**

* `blk_x` - 水平方向分割块数
* `blk_y` - 垂直方向分割块数

---
### 9.2.2 mvRoi_t
定义感兴趣区域数据结构

**定义**
```cpp
struct mvRoi_t {
    uint32_t x, y;
    uint32_t w, h;
    uint32_t mv_mb_num;
};
```

**成员**

* `x` - 感兴趣区域X坐标
* `y` - 感兴趣区域Y坐标
* `w` - 感兴趣区域宽度
* `h` - 感兴趣区域高度
* `mv_mb_num` - 感兴趣区域内当前帧的运动单元数量

---
### 9.2.3 roiInfo_t
定义运动侦测感兴趣区域数据结构

**定义**
```cpp
struct roiInfo_t {
    struct mv_roi_t mv_roi;
    int roi_num;
};
```

**成员**

* `mv_roi` - 感兴趣区域数据结构，参见[mv_roi_t](main.md#9.2.2_mv_roi_t)
* `roi_num` - 感兴趣区域索引，取值范围[0，3]

---
### 9.2.4 MBThreshold
定义运动侦测感兴趣区域数据结构

**定义**
```cpp
struct MBThreshold {
    uint32_t tlmv;
    uint32_t tdlmv;
    uint32_t tcmv;
};
```

**成员**

* `tlmv` - 运动向量的长度的比较阈值，一般取[5，10]
* `tdlmv` - 当前MB的`tlmv`与其参考MB的`tlmv`差值的比较阈值，一般取[0，128]
* `tcmv` - 可信度阈值，作为最终判断是否发生运动的依据

---
### 9.2.5 ST_FODET_RECT_INFO
定义人脸检测检测到人脸坐标的数据结构

**定义**
```cpp
typedef struct v_fodet_rect_info {
    int num;
    int x[MAX_SCAN_NUM];
    int y[MAX_SCAN_NUM];
    int w[MAX_SCAN_NUM];
    int h[MAX_SCAN_NUM];
} ST_FODET_RECT_INFO;
```

**成员**

* `num` - 检测到人脸的个数，一般取[0，10]
* `x` - 检测到人脸位置的X坐标
* `y` - 检测到人脸位置的Y坐标
* `w` - 检测到人脸位置的宽度
* `h` - 检测到人脸位置的高度
----
## 10 Videobox内存计算
Videobox提供了一个Python工具**buildroot/tools/memoryCalculate.py**，通过解析启动JSON文件，计算Videobox需要的内存

**执行命令**
```bash
./memoryCalculate.py path.json
```
**使用限制**

* 支持Python 2.x/Python 3.x

**示例JSON**
```json
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
            },
            "his": {
                "bind": { "vam": "in" }
            },
            "cap": {
                "w": 1920,
                "h": 1088,
                "bind": {"jpeg":"in"}
            }
        }
    },

    "marker0": {
        "ipu": "marker",
        "args": {
            "mode": "nv12"
        },
        "port": {
            "out": {
                "w": 800,
                "h": 64,
                "pixel_format": "nv12",
                "bind": { "ispost0": "ol" }
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
            "ol": {
                "pip_x":576,
                "pip_y":960
            },
            "ss0": {
                "w": 1280,
                "h": 720,
                "pixel_format": "nv12",
                "bind": { "enc720p": "frame", "encrecord": "frame" }
            },
            "dn": {
                "w": 1920,
                "h": 1088,
                "pixel_format": "nv12",
                "bind": { "enc1080p": "frame", "display": "osd0" }
            },
            "ss1":{
                "w": 640,
                "h": 360,
                "pixel_format": "nv12",
                "bind": { "encvga": "frame" }
            }
        }
    },

    "jpeg": {
        "ipu": "h1jpeg",
        "args": {
            "mode": "trigger"
        }
    },

    "enc1080p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "enc720p": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "encvga": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "encrecord": {
        "ipu": "vencoder",
        "args": {
            "encode_type": "h265"
        }
    },
    "display": { "ipu": "ids", "args": { "no_osd1": 1 }},
    "vam": { "ipu": "vamovement"}
}
```
**输出格式**
```bash
-------------------------------------v2500------------------------------------------
       NAME         DDK         HIS         CAP         OUT                   TOTAL
        isp     10204KB       128KB         0KB         0KB                 10332KB

-------------------------------------marker-----------------------------------------
       NAME    INTERNAL         OUT                                           TOTAL
    marker0         9KB       150KB                                           159KB

-------------------------------------h1jpeg-----------------------------------------
       NAME    INTERNAL       SLICE      STREAM   THUMBNAIL                   TOTAL
       jpeg         0KB       510KB      2295KB         0KB                  2805KB

-------------------------------------vamovement-------------------------------------
       NAME    INTERNAL                                                       TOTAL
        vam         0KB                                                         0KB

-------------------------------------ispost-----------------------------------------
       NAME          DN         SS1         SS0                               TOTAL
    ispost0      9180KB      1012KB      4050KB                             14242KB

-------------------------------------h2---------------------------------------------
       NAME         DDK         DPB      STREAM                               TOTAL
   enc1080p        44KB      6502KB      3072KB                              9618KB
    enc720p        39KB      3060KB      3072KB                              6171KB
     encvga        36KB       765KB      3072KB                              3873KB
  encrecord        39KB      3060KB      3072KB                              6171KB

-------------------------------------ids--------------------------------------------
       NAME    INTERNAL                                                       TOTAL
    display         0KB                                                         0KB

------------------------------------------------------------------------------------
                                                                              TOTAL
                                                                            53375KB
```
输出信息按照IPU类型分类显示，除了端口的内存需要，其它内存说明如下

* `INTERNAL` - IPU使用的内存，不包含DDK及端口部分
* `DDK` - 驱动层使用的内存。编解码模块（`h1`/`h2`/`vencoder`/`g1`/`g2`），不包含`DPB`占用内存
* `DPB` - 解码缓冲区内存

## 11 Videobox调试工具
Videobox调试工具提供ISP(`v2500`/`ispostv1`/`ispostv2`)和视频编码模块(`h1`/`h2`/`vencoder`/`jenc`)等视频业务的相关调试信息，可实时反映当前视频系统的运行状态，所记录的信息可供问题定位及分析时使用

**调试信息查看方法**

查询所有调试信息
```bash
vbctrl debug
```

查询ISP调试信息
```bash
vbctrl debug --isp
```

查询视频编码模块调试信息
```bash
vbctrl debug --venc
```

**ISP调试信息描述**

[调试信息]

```
/ # vbctrl debug --isp
-----ISP Base info------------------------------------------------------------------
OutWidth    OutHeight   CapWidth    CapHeight   FPS     ISPClk  
1280        720         1280        720         27      198Mhz  

-----ISP Memory Info------------------------------------------------------------------
ISPUsed     ISPMaxUsed  MemWritePrio    MemReadPrio     
4388KB      4388KB      High            Low             

-----ISP Context Info------------------------------------------------------------------
Context AE      AWB     HWAWB   TNMCurve    DayMode     MirrorMode  3DN     
0       Y       Y       N/A     Dynamic     Day         None        N     
```

[调试信息的分析]

记录当前ISP模块的基本情况

[参数说明]

| `ISP Base Info`参数 | 描述 |
|-------------------------|-------|
| `OutWidth` | `v2500`的`out`端口输出宽度 |
| `OutHeight` | `v2500`的`out`端口输出高度 |
| `CapWidth` | `v2500`的`cap`端口输出宽度 |
| `CapHeight` | `v2500`的`cap`端口输出高度 |
| `FPS` | 当前ISP子系统处理的帧率 |
| `ISPClk` | 当前ISP工作的时钟频率，以MHz为单位 |

| `ISP Memory Info`参数 | 描述 |
|-------------------------|-----------|
| `ISPUsed` | ISP子系统的当前内存使用量，以KB为单位 |
| `ISPMaxUsed` | ISP子系统的最大的内存使用量，以KB为单位 |
| `MemWritePrio` | ISP子系统的DDR写优先级，优先级分为`High`和`Low` |
| `MemReadPrio` | ISP子系统的DDR读优先级，优先级分为`High`和`Low` |

| `ISP Context Info`参数 | 描述 |
|-------------------------|-----------|
| `Context` | ISP子系统的流处理上下文编号，一般情况为0，当使用`Apollo`的dual-sensor时会出现0/1编号，代表2个硬件上下文 |
| `AE` | ISP的AE模块状态，Y - 使能， N - 关闭 |
| `AWB` | ISP的AWB模块状态，Y - 使能， N - 关闭 |
| `HWAWB` | ISP的硬件AWB模块状态，Y - 使能， N - 关闭， N/A - 不存在（`Apollo`/`Apollo-ECO`） |
| `TNMCurve` | ToneMapping Curve调整方式，Dynamic - 动态调整曲线方式, Static - 静态调整曲线方式 |
| `DayMode` | ISP当前场景模式，Day － 白天场景， Night － 夜晚场景 |
| `MirrorMode` | ISP输出图像镜像状态，None - 没有镜像， H - 水平镜像， V - 垂直镜像， HV - 垂直水平镜像 |
| `3DN` | ISP的3D降噪模块的状态，Y - 使能， N - 关闭 |

**VENC调试信息描述**

[调试信息]

```
/ # vbctrl debug --venc
-----Channel Info------------------------------------------------------------------------------------
ChannelName              Type  State   Width Height  FPS   FrameIn         FrameOut        FrameDrop
venc-vga-stream          H265  RUNNING 640   480     29.0  2025            2025            0
venc-1080p-stream        H265  RUNNING 1920  1080    14.0  943             943             0
jpeg-out                 JPEG  IDLE    1920  1088    0.0   0               0               0

-----Rate Control Info--------------------------------------------------------------------------------
ChannelName              RCMode  MinQP   MaxQP   QPDelta RealQP  I-Interval  GOP     Mbrc    ThreshHold(%)   PictureSkip   SetBR(kbps)     BitRate(kbps)
venc-vga-stream          CBR     26      42      6       36      15          15      1       0               Disable       600.0           1454.3
venc-1080p-stream        VBR     26      42      6       37      15          15      1       80              Disable       2048.0          1896.4
```

[调试信息的分析]

统计视频和图片编码模块的运行、配置信息

[参数说明]

| `Channel Info`参数 | 描述 |
|-------------------------|-------|
|`ChannelName`|编码端口名字|
|`Type`|编码端口类型，分为H264，H265，JPEG|
|`State`|编码器运行状态，分为INIT，RUNNING，STOPED，IDLE|
|`Width`|编码端口宽度|
|`Height`|编码端口高度|
|`FPS`|编码端口输出帧率|
|`FrameIn`|编码端口输入图像帧次数|
|`FrameOut`|编码端口输出图像帧次数|
|`FrameDrop`|编码端口丢弃图像帧次数|

| `Rate Control Info` | 描述 |
|-------------------------|-------|
|`ChannelName`|编码端口名字|
|`RCMode`|编码端口码率控制方式，分为VBR，CBR，FIXQP|
|`MinQP`|用户配置最小QP值|
|`MaxQP`|用户配置最大QP值|
|`QPDelta`|I帧与临近P帧的QP差值,绝对值过大容易引起呼吸效应，负值表示I帧QP比P帧小，即I帧采用更精细的量化参数并将占用更多编码比特|
|`RealQP`|编码器运行时QP值，`mbrc`为0时为实际编码QP值，宏块级码率控制开启时无法获取QP平均值，与实际编码QP可能有较大偏差|
|`I-Interval`|用户配置关键帧间隔|
|`GOP`|用户配置GOP的长度|
|`Mbrc`|宏块级码率控制，0是关闭，H265编码1表示开启。H264编码该值为负时，平坦区域比细节区域分配更精细的量化参数，为正时则相反|
|`ThreshHold`|VBR码率下限百分比，VBR时允许码率在用户设置最大码率(MaxBitrate)和对应的最大码率百分比中间(MaxBitrate * 'ThreshHold'/100)调整|
|`PictureSkip`|码率控制开启丢帧开启状态|
|`SetBR`|用户配置码率(CBR)或用户配置最大码率(VBR)，单位kbps|
|`BitRate`|编码器运行时的码率，单位kbps|


Note:
1. `RealQP`开启宏块级码率控制时与实际QP可能有较大差异，H265为当前编码帧最大QP值，H264如果Mbrc为负值，则为当前编码帧最大QP值，如果Mbrc为正值，则为当前编码帧最小QP值
2. `Mbrc`配置，H264中对应配置为`mb_qp_adjustment`，H265中对应配置`mbrc`
3. BitRate统计基于固定时间间隔，查询间隔过短可能是上次的值
4. 码率控制参数详细调节，请参考《QSDK码率配置指南》
