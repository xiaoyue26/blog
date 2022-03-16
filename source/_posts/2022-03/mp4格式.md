---
title: mp4格式
date: 2022-03-13 17:45:02
tag:
- mp4
- h264
- 直播
- 点播
categories: 
- 音视频

---

# WHAT: MP4
一种封装格式(容器格式)，封装视频和音频、还有海报、字幕和元数据等；

常见的封装格式: MKV,AVI,MP4

MP4文件格式又被称为 MPEG-4 Part 14，出自 MPEG-4 标准第 14 部分。

# MP4格式
协议分为两阶段:
(1)`MPEG-4 Part 12`: 定义ISO基础媒体文件格式，存储基于时间的媒体内容；
(2)`MPEG-4 Part 14`: 在`MPEG-4 Part 12`基础上拓展，实际定义了MP4；
实际格式:
```
[
    Box
    ,Box:[Box,Box:[Box]]
]
```
多个box组成，box可嵌套；

## Box
不同的Box类型:
(1)`ftyp`: `File Type Box`; 描述MP4规范与版本；最好用什么版本来解析、兼容哪些版本的解析；
(2)`moov`: `Movie Box`; 媒体的`metadata`信息；1个；
    next_track_ID: 下一个轨道的id;(如果要新增)
    volume: 播放音量;
单轨属性:
    layer: 视频轨道的叠加顺序，数字越小越靠近观看者;
    alternate_group：track的分组ID，同个分组里的track，只能有一个track处于播放状态;
    width、height：视频的宽高 
(3)`mdat`: Media Data Box；实际媒体数据；多个；
(4)...其他box类型；


### moov
`ffmpeg`默认情况下生成`moov`是在`mdat`写完成之后再写入，所以moov是在mdat的后面，使用faststart参数可以将moov移到mdat前面。
(moov在前面的话，首帧渲染速度会更快一点)
```shell script
ffmpeg -i out.flv -c copy -movflags faststart out2.mp4
```

### 媒体数据结构划分
MP4媒体数据 -> `chunk` -> `sample` -> 帧

##### H264编码中的帧: 
I帧: 关键帧；存的是完整的JPEG数据；
P帧: 前向预测编码帧；存的是与之前帧的差别；（参考别的帧）
B帧: 双向预测内插编码帧；双向差别，存的是和前帧、后帧的差别；（所以压缩率比P帧高，cpu占用高）

对于B帧，视频帧的解码顺序和渲染顺序不一致。（先解前后，才解中间；而渲染是顺序来的）

// H.265是一种视频压缩标准，仅需H.264的一半带宽即可播放相同质量的视频。


##### H264流格式
1。`Annex-B`
用`0x00000001`分割；
2。`RTP`
长度+数据的格式；

# fmp4
mp4: 点播
fmp4: 直播

时长: mp4固定,fmp4不固定（边生成边播）；
元数据: `mp4`->`moov box`;
       `fmp4`->`moof box`,和`mdat`通常结对出现；（流式）


# mp4相关工具
在线解析mp4的工具: https://gpac.github.io/mp4box.js/test/filereader.html
mp4dump、mp4edit、mp4encrypt等工具: http://www.bento4.com/
https://github.com/gpac/gpac/wiki/MP4Box
gpac：开源的多媒体工具包，包括用于MP4打包的mp4box等。https://github.com/gpac/gpac
mp4v2：提供了API来创建和修改mp4文件。https://code.google.com/archive/p/mp4v2/



# 参考
https://segmentfault.com/a/1190000039270533