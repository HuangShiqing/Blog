# 前言
gstreamer是非常优秀的构建流媒体应用的多媒体框架，我们可以用它来获取树莓派的摄像头图像然后进一步作模型推理
# 安装
``` bash
sudo apt-get install libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio
sudo apt-get install gstreamer-plugins-base1.0-dev
```
# 概念简介
[基本概念的官方资料](https://gstreamer.freedesktop.org/documentation/tutorials/table-of-concepts.html?gi-language=c)
## element
![](./picture/GStreamer/1.png)
基本单位，用来产生、处理、消亡数据流。本质是一种特殊的GObject
>The elements are GStreamer's basic construction blocks  

+ 创建一个元素  
source = gst_element_factory_make ("videotestsrc", "source");  
```c
GstElement * = gst_element_factory_make (const gchar * factoryname,const gchar * name)  
Parameters:  
    factoryname – a named factory to instantiate  
    name ( [allow-none] ) – name of new element, or NULL to automatically create a unique name
```
| 可选的factoryname | 介绍 | 官方介绍 |
| :----:| :----:| :----: |
| videotestsrc | 产生测试视频 |  |
|v4l2src|获取linux的v4l2摄像头||
| videoconvert | 前后原本不通则会通，原本通则该元素对性能影响极小 |  |
| appsink | 数据进来后让用户可以自己处理 |  |
| autovideosink | 自动选择最佳的显示出来 | displays on a window the images it receives |
|xvimagesink|||
| uridecodebin | 内部自动产生必要的组合元素处理uri | internally instantiate all the necessary elements (sources, demuxers and decoders) to turn a URI into raw audio and/or video streams |
| playbin | 包含了所有的 | a special element which acts as a source and as a sink, and is a whole pipeline. Internally, it creates and connects all the necessary elements to play your media, so you do not have to worry about it |
| tee | 多通，将输入复制多份输出 | it sends through its source pads everything it receives through its sink pad. an input stream can be replicated any number of times |
| filesrc |  |  |
|  |  |  |
|  |  |  |
|  |  |  |

## pad
![](./picture/GStreamer/2.png)  
元素间以pad作为数据出入口，数据出口叫做src pad（作为下一个element的源），数据入口叫做sink pad（作为上一个element的入口）。要想连接前后元素必须保证前元素的出口pad支持的类型（cap）和后元素的入口pad支持的类型（cap）有交集
>In order for two elements to be linked together, they must share a common subset of Capabilities

| pad类型 | 描述 | 代表元素 |
| :----:| :----:| :----: |
| Always Pads（static_pad） | 最常见的一般类型的pad |  |
| Sometimes Pads | 产生于有数据流入元素之后的pad | oggdemux |
| Request Pad | 需要手动指定的pad | tee |
|  |  |  |


+ 获取static pad  
queue_audio_pad = gst_element_get_static_pad (audio_queue, "sink");
+ 获取request pad  
tee_video_pad = gst_element_get_request_pad (tee, "src_%u");



## Capabilities
[cap介绍](https://www.cnblogs.com/xuyh/p/4564494.html)  
pad所能通过的数据类型，或者相当于pad的一种属性。对于某个pad想要其输出特定属性的数据流（比如320*240的video/x-raw），但是其属性没办法直接设置，只能通过在该pad后面用函数gst_caps_new_simple和gst_element_link_filtered连接一个所需属性的cap作为滤波器
>Pads can support multiple Capabilities (for example, a video sink can support video in different types of RGB or YUV formats) and Capabilities can be specified as ranges (for example, an audio sink can support samples rates from 1 to 48000 samples per second).

| 常见的cap |  |  |
| :----:| :----:| :----: |
| video/x-raw |  |  |
| video/x-raw-yuv |  |  |
| video/x-h264 |  |  |
|  |  |  |
|  |  |  |
+ 生成特定属性的cap作为滤波器  
  GstCaps * caps = gst_caps_new_simple ("video/x-raw",
      "format", G_TYPE_STRING, "RGB",
      "width", G_TYPE_INT, 320,
      "height", G_TYPE_INT, 240,
      "framerate", GST_TYPE_FRACTION, 25, 1,
      NULL);
## bin
用来包含其他元素的元素
>which is the element used to contain other elements
+ 在bin中加入多个元素  
gst_bin_add_many (GST_BIN (data.pipeline), data.source, data.convert, data.resample, data.sink, NULL);
## pipeline
特殊的bin，所有的元素必须都放到一个pipeline中才能使用
>A pipeline is a particular type of bin.  

>All elements in GStreamer must typically be contained inside a pipeline before they can be used, because it takes care of some clocking and messaging functions
+ 创建一个新的pipeline  
pipeline = gst_pipeline_new ("test-pipeline");
+ 将元素加入到pipeline中，以NULL结尾  
gst_bin_add_many (GST_BIN (pipeline), source, sink, NULL);

## link
+ 连接元素，内部实现就是调用了gst_pad_link  
if (gst_element_link (source, sink) != TRUE)
+ 连接pad  
if (gst_pad_link (tee_audio_pad, queue_audio_pad) != GST_PAD_LINK_OK)  
request pad在被连接后需要用函数gst_object_unref (queue_audio_pad)手动释放
+ 在两个元素之间连接一个滤波cap指定两者直接的数据类型  
link_ok = gst_element_link_filtered (element1, element2, caps);
## propertie
element有自己的属性，用来控制element的行为或者查询element状态
>can be modified to change the element's behavior (writable properties) or inquired to find out about the element's internal state (readable properties).
+ 设置属性，以NULL结尾  
g_object_set (source, "pattern", 0, NULL);
+ 获取属性，以NULL结尾  
 g_object_get (source, "name", &name, NULL);
## GStreamer States
元素或者pipeline的状态，设置成PLAYING后数据开始流动。状态之间只能相邻移动，但是用户使用时不用在意，gst_element_set_state会自动处理
>You can only move between adjacent ones

| 可能的状态 | 官方说明 |  |
| :----:| :----:| :----: |
| NULL | the NULL state or initial state of an element. | 该状态将会回收所有被该element占用的资源 |
| READY | the element is ready to go to PAUSED. |  |
| PAUSED | the element is PAUSED, it is ready to accept and process data. Sink elements however only accept one buffer and then block. |
| PLAYING | the element is PLAYING, the clock is running and the data is flowing. |

+ 启动pipeline  
ret = gst_element_set_state (pipeline, GST_STATE_PLAYING);
## bus
让element可以传递GstMessage给程序以及线程间传递消息
| 常见的message | 官方说明 |  |
| :----:| :----:| :----: |
|GST_MESSAGE_ERROR|error||
|GST_MESSAGE_EOS|End-Of-Stream|播放完毕|
|GST_MESSAGE_STATE_CHANGED|element state-changed messages||
||||
|GST_MESSAGE_STREAM_STATUS|The application can use this information to detect failures in streaming threads and/or to adjust streaming thread priorities|用来查看流和线程的启动等信息|

+ 获取pipeline的bus  
bus = gst_element_get_bus (pipeline);
+ gst_bus_timed_pop_filtered()
## signal
## GstBuffer
## GSource
# 工具gst-launch-1.0
```bash
gst-launch-1.0 videotestsrc ! autovideosink  
```
[官方说明](https://gstreamer.freedesktop.org/documentation/tutorials/basic/gstreamer-tools.html)已经很详细了，不赘述
# 功能代码片段
## 在两个元素之间利用cap_filter设定通过数据的类型和属性
```c
GstCaps * caps = gst_caps_new_simple ("video/x-raw",
    "format", G_TYPE_STRING, "RGB",
    "width", G_TYPE_INT, 320,
    "height", G_TYPE_INT, 240,
    "framerate", GST_TYPE_FRACTION, 25, 1,
    NULL);
gboolean link_ok = gst_element_link_filtered (element1, element2, caps);
gst_caps_unref (caps);
if (!link_ok) {
    g_warning ("Failed to link element1 and element2!");}
```
## appsink利用new_sample的回调获取输入图像
```c
GstSample* gstSample = gst_app_sink_pull_sample(mAppSink);
if( !gstSample )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_app_sink_pull_sample() returned NULL...\n");
    return;
}
GstBuffer* gstBuffer = gst_sample_get_buffer(gstSample);
if( !gstBuffer )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_sample_get_buffer() returned NULL...\n");
    return;
}
// retrieve
GstMapInfo map; 
if(	!gst_buffer_map(gstBuffer, &map, GST_MAP_READ) ) 
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_buffer_map() failed...\n");
    return;
}
//gst_util_dump_mem(map.data, map.size); 
//获取的图像指针和图像尺寸大小
void* gstData = map.data; //GST_BUFFER_DATA(gstBuffer);
const uint32_t gstSize = map.size; //GST_BUFFER_SIZE(gstBuffer);
if( !gstData )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_buffer had NULL data pointer...\n");
    return;
}
// retrieve caps
GstCaps* gstCaps = gst_sample_get_caps(gstSample);
if( !gstCaps )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_buffer had NULL caps...\n");
    return;
}
GstStructure* gstCapsStruct = gst_caps_get_structure(gstCaps, 0);
if( !gstCapsStruct )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_caps had NULL structure...\n");
    return;
}
// get width & height of the buffer
int width  = 0;
int height = 0;
if( !gst_structure_get_int(gstCapsStruct, "width", &width) ||
    !gst_structure_get_int(gstCapsStruct, "height", &height) )
{
    g_printerr(LOG_GSTREAMER "gstCamera -- gst_caps missing width/height...\n");
    return;
}
if( width < 1 || height < 1 )
    return;
mWidth  = width;//图像宽度
mHeight = height;//图像高度
mDepth  = (gstSize * 8) / (width * height);//图像深度
mSize   = gstSize;//图像数据的尺寸，rgb图像的话=h*w*3
```
# 参考链接
1. [Pads相关](https://www.cnblogs.com/xuyh/p/4564494.html)
1. [How do I fix a “cannot open display” error when opening an X program after ssh'ing with X11 forwarding enabled?](https://superuser.com/questions/310197/how-do-i-fix-a-cannot-open-display-error-when-opening-an-x-program-after-sshi)
1. [Gdk-WARNING **: locale not supported by C library解决方法](https://www.cnblogs.com/ylan2009/articles/2484737.html) 
1. [Source of Raspistill](https://www.raspberrypi.org/forums/viewtopic.php?t=61072)  
1. [usb和csi相机对比](https://blog.csdn.net/ZhaoDongyu_AK47/article/details/103981905)  
