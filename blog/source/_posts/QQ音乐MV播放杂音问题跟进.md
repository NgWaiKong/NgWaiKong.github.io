---
title: QQ音乐MV播放杂音问题跟进
date: 2018/8/04
comments: true
categories: performance
tags: performance
---


# QQ音乐MV播放杂音问题跟进

## 问题背景

QQ音乐Android端播放MV视频《凤凰花开的路口》时带有如电流声一般的杂音，影响用户的正常体验。

## 问题分析

在初步定位中，发现有如下特征：

- Android端杂音问题必现
- iOS、PC端能正常播放《凤凰花开的路口》，没有噪音（各端都是统一用hls格式播放）

对于该问题，定位思路如下：

- 梳理ijkplayer流程
- 找到切入点排查

### （一）ijkplayer流程

![](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/VPf.VLzaJABzmhSeVM5a60btIqrZoSgFY1QyVkiZDUc!/b/dDEBAAAAAAAA&bo=uATVAgAAAAADF1k!&rf=viewer_4&t=5)

结合上图，总结关键步骤（图中内容从左往右，以音频解码播放为例）：
<!-- more -->

1. **播放器初始化：**
   - `stream_open`主要创建读数据线程`read_thread` 
   - 创建存放audio解码前数据的队列`audioq`
   - 创建存放audio解码后数据的队列`sampq`
2. **数据读取：**
   - ①创建`avformat_alloc_context()`
   - ②探测协议类型`avformat_open_input(&ic, is->filename, is->iformat, &ffp->format_opts)`
   - ③探测媒体类型`avformat_find_stream_info`
   - ④获取音视频流`av_find_best_stream`
   - ⑤打开媒体解码器`stream_component_open(ffp, st_index[AVMEDIA_TYPE_AUDIO])`
   - ⑥读取媒体数据，获得AVPacket`av_read_frame(ic, pkt)`
   - ⑦音视频数据分别送入`audioq`中
   - 重复⑥、⑦步骤到数据完毕
3. **音频解码：**
   - 在`audio_thread`中对`audioq`中的数据进行`decoder_decode_frame`解码
   - 解码后的帧`AVFrame`存放到`sampq`中
4. **音频播放：**
   - `aout_thread_n`中，通过调用回调接口`sdl_audio_callback`，对`sampq`中的音频帧数据进行解码成PCM数据
   - 写入PCM数据到提供给AudioTrack播放用的buffer数组，并交由AudioTrack播放

 

### （二）分层切入

在梳理出ijkplayer播放流程后，标记出找到有可能出错的环节，方便进行“分层定位”（图中黄色标记）

- 播放下载文件是否有问题
- 数据读取是否有问题
- 音频解码逻辑是否有问题
- AudioTrack的设置是否有问题

以上环节，根据难易程度逐个验证。

#### 1、播放下载文件是否正常

把Android端播放的ts文件与各端的进行比对，发现两者一样，该环节正常

#### 2、AudioTrack设置是否正常

通过日志检查AudioTrack以下配置参数：

- 采样率
- 位深
- 频道

以上参数设置的值与音频流的相符合，该环节正常

#### 3、音频解码逻辑是否有问题

验证解码逻辑是否有问题，可以通过对PCM数据进行分析来确认。

对ijkplayer源码中`aout_thread_n`进行修改，将PCM数据额外输出到本地，并与正常的PCM数据进行对比。

正常PCM数据频谱图：![PCM正常mp4数据](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/26dAdc6oXkiXMGDlVrXfj0wEFhaERoejypos2eap7Yk!/b/dDEBAAAAAAAA&bo=sgawAQAAAAADR2c!&rf=viewer_4&t=5)

异常PCM数据频谱图：![PCM音频数据缺失](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/7D950BzuoRqgC4rfmeCa*Z0hC7I9BCKv4wjSfxmnmbA!/b/dDIBAAAAAAAA&bo=tAapAQAAAAADR3g!&rf=viewer_4&t=5)

正常PCM数据波形图：![img](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/3EZ57b71xsGbgtL5*Ga0IJt9rp0u6Eh1GLiir1xvWC4!/b/dDIBAAAAAAAA&bo=qwZhAQAAAAADF*8!&rf=viewer_4&t=5) 

异常PCM数据波形图：

![1533385754516](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/PUx6Aq.FP0dt2zFYUVTu77jr6Qc8Vkw2gOVC7Ql65k4!/b/dFYBAAAAAAAA&bo=mwZ0AQAAAAADF9o!&rf=viewer_4&t=5)



- 从频谱图中看出，异常的PCM在人耳十分敏感的频响（1000～8000Hz ）区域内的音频数据严重缺失，导致“杂音问题”。
- 从波形图中看出，异常的与正常的无声区和有声区都吻合，若解封装、解码逻辑出现异常，极大几率是呈现无波动（一条直线的形式）情况。因此可以先大胆**假设解码、解封装逻辑是符合预期的**。

若解码逻辑正常，再结合之前已经验证文件下载正常。可以**推测是数据读取环节出现异常**。

#### 4、数据读取是否有问题

通过对数据读取的各步骤增加日志后，发现在`av_find_best_stream`音频流选择时出现异常：

`ffmpeg -i` 发现，该视频ts分片有2个音频流

![1533377825366](http://m.qpic.cn/psb?/V10Sw18W4gWqgb/2j8uGm1n.7LynyhwwevG7xe8aIHKZmj3s3Qxre6rVt4!/b/dDIBAAAAAAAA&bo=4ASXAAAAAAADF0E!&rf=viewer_4&t=5)

通过强制分别读取两条音频流数据播放，发现：

- 第一条正常播放（PCM数据正常）
- 第二条播放杂音（PCM数据异常）
- Android端选择了第二条进行播放

（通过查看2条流的PCM数据，也验证了在第3步中的假设是正确的。）

### （三）问题定位结论

由上得出结论：**Android端选择了第二条数据有问题的流进行播放。**

## 音频流选择

### 选择方式

在Android使用ffmpeg中的`av_find_best_stream`来选择音频流。

该函数会根据2个主要参数进行选择：

1. 各音频流的在探测媒体类型（`avformat_find_stream_info`）时，**额外解码出来的帧数**（选择多的）
2. 各音频流的**比特率**（选择高的）

``` c
//url:https://ffmpeg.org/doxygen/2.8/libavformat_2utils_8c_source.html
//line:3572
//仅保留相关代码
int av_find_best_stream(AVFormatContext *ic, enum AVMediaType type,
                        int wanted_stream_nb, int related_stream,
                        AVCodec **decoder_ret, int flags)
{
    for (i = 0; i < nb_streams; i++) {
        count = st->codec_info_nb_frames; //音频流探测中解码的帧数
        bitrate = avctx->bit_rate;//音频流的比特率
        multiframe = FFMIN(5, count);
        //先比较解码帧数，再比较音频流比特率，谁大谁选
        if ((best_multiframe >  multiframe) ||
            (best_multiframe == multiframe && best_bitrate >  bitrate) ||
            (best_multiframe == multiframe && best_bitrate == bitrate && best_count >= count))
            continue;
        best_count   = count;
        best_bitrate = bitrate;
        best_multiframe = multiframe;
        ret          = real_stream_index;//最后选择的流index
        best_decoder = decoder;
    }   
    return ret;
}
```
在该视频中，

|                | codec_info_nb_frames | bit_rate |
| -------------- | :------------------: | :------: |
| audio_stream 1 |          38          |  122625  |
| audio_stream 2 |          39          |  126375  |

**第二条流的解码帧数和比特率要比第一条高**，因此ffmpeg选择了第二条流播放

### 对比

分析了以上选择规则后，我们对各平台、框架进行了选择规则的对比

|      |  Android ffmpeg   |                    Android ExoPlayer                     | iOS端播放系统组件 |  Mac QuickTime   | PC端播放系统组件 |
| :--- | :---------------: | :------------------------------------------------------: | :---------------: | :--------------: | ---------------- |
| 规则 | 解码帧数 + 比特率 | 语言匹配程度 + 是否默认音轨 + 声道条数 + 采样率 + 比特率 |    默认第一条     | 所有音频流全播放 | 默认第一条       |

备注：

- ExoPlayer对多音频流的ts分片支持不完善（[issue](https://github.com/google/ExoPlayer/issues/2014)），因此测试时需要调整相关接口。但选择规则依然以上述所示（[DefaultTrackSelector](https://github.com/google/ExoPlayer/blob/release-v2/library/core/src/main/java/com/google/android/exoplayer2/trackselection/DefaultTrackSelector.java)）

- iOS和PC端采用闭源组件，因此测试时使用了**"互换两条音频流顺序"**的方法进行测试。互换后，两端都播放了杂音音频流。 

  `ffmpeg -i INPUT_FILE -map 0:0 -map 0:2 -map 0:1 -c copy  -y OUTPUT_FILE`

- QuickTime同样是闭源，互换音频流后无法明显差别，通过**合成第三条音频流**，来验证是它是对所有音频流全播放。

  `ffmpeg -i INPUT_FILE_1 -i INPUT_FILE_2  -map 0:0 -map 0:1 -map 0:2 -map 1:0 -c copy OUTPUT_FILE`

### 总结

从以上数据看到，iOS和PC端会默认选择第一条流，而在Android端的ffmpeg和ExoPlayer会根据音频流属性来选择数值更好的一条。

- “默认选择第一条”方案能**更容易地把音源问题暴露**。
- “比较音频流属性”方案能**更大几率地选择质量更好的流**来提升用户体验。

但以上2个选择方案都**无法识别**“内容异常”的音频流。

## 解决方案

因此处理该问题，需要从音源上进行修复和规避。

以下是解决方案：

1. 编辑重新上架正常音源
2. 前期Android端增加双音频流的检测上报，帮助后台、编辑进行复查
3. 后续由后台开发工具，分别对存量视频进行双音频流检测和对增量视频保证只转码单音频流

## 参考资料

- https://ffmpeg.org/doxygen/2.8/
- https://github.com/google/ExoPlayer
- https://www.jianshu.com/p/daf0a61cc1e0
- https://www.jianshu.com/p/a6a4bf59cdae
- http://km.oa.com/articles/show/319627
- https://codeday.me/bug/20170711/39603.html

## 感谢

最后感谢evi、aimee、brave、xuezeng、kelvin、harvey、larry、proud各位的帮助