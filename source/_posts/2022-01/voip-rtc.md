---
title: VoIP和RTC
date: 2022-01-21 14:14:50
tag:
- voip
- rtc
categories: 
- rtc

---

# 术语
## VoIP
VoIP: `Voice over Internet Protocol` 
基于IP的语音传输。只是概念，不对应具体协议栈、有不同实现。
别名：IP电话、互联网电话、宽带电话、宽带电话服务。
VoIP可用于包括VoIP电话、智能手机、个人计算机在内的诸多互联网接入设备，通过蜂窝网络、Wi-Fi进行通话及发送短信。

VoIP的实现迭代: 
(1)DSL/cable调制解调器
(2)Wifi/3G
(3)LTE
(4)RCS

## V.VoIP
`Video over IP`
一些V.VoIP的基本要素包括信令、媒体引擎、会话描述协议（SDP）、实时传输协议/实时控制协议（RTP/RTCP）、
网络地址转换（NAT）、安全协议、服务质量（QoS），以及其余电话组件。
V.VoIP实质上是封装了全部这一切，再加上用户界面，包括拨号、通信录/联系人列表和呼叫历史记录（未接/已接/已拨电话），
以提供一个完整的V.VoIP客户端。

## RTC
RTC: `Real-Time Communications` 
RTC也就是实时交流。一般是指音\视频。日常生活中的例子包括：微信通话、视频会议、直播PK连麦、在线教育。

上述两种概念，都看重"端到端延时"指标，大致定义如下:
{% img /images/2022-01/voip_transfer.png 800 1200 transfer %}
从这个流程，可以看出相比于普通文本传输，主要是多了和音视频相关的"采集"、"处理"、"编码"等操作。
其中的"网络"部分，由于VoIP是从电话网络演化而来，主要侧重于原来的电话网到IP网络；
RTC则主要侧重于IP网络。

## SIP协议
`Session initialization Protocol`
一种信令协议。
VoIP的具体实现的其中一部分的协议，主要做控制层（信令），基于文本。（RFC 2543, RFC 3261~3265）大致开始于1999年。

## WebRTC
谷歌收购GIPS引擎，与GTalk通信库合并，2011年纳入Chrome体系并开源，命名为WebRTC。
2012年获各大浏览器厂商支持，纳入W3C标准。
{% img /images/2022-01/web_rtc_sip.png 800 1200 web_rtc_sip.png %}

WebRTC的架构大致如下，主要是给前端提供了封装好的API，屏蔽底层硬件。：
{% img /images/2022-01/webrtc.png 800 1200 webrtc %}

绝大部分浏览器的新版本都支持webRTC:
{% img /images/2022-01/webrtc_support.png 800 1200 webrtc_support.png %}

## RTMP
RTMP: 是Real Time Messaging Protocol的简称，
基于 *TCP* 协议，用来进⾏实时数据通信的⽹络协议，
推流到CDN即用的是RTMP;

## SRS
`Simple Realtime Server`
互联网直播服务器集群的开源框架;
https://github.com/ossrs/srs/wiki/v2_CN_Home

# 发展历史
## 1996～2004：H323
电话线上网时代；
### H.323标准
ITU（国际电信联盟）推出，
基于传统 PSTN 架构（Public Switched Telephone Network，公共交换电话网）。
(MCU+RTP)
H.323标准的VOIP网络中：
用户呼叫021号码：先送到上海、再到目的地； 
在SIP中：直接查，无短途长途之分。

场景：PC语音会议；
优点：达到了可通话的质量；
缺点：MCU是中心化的，拓展性有上限。有合流成本。

## 2001～2007: P2P/Mesh
IETF, SIP(UDP + HTTP) 软交换

Skype: 超级节点+P2P/Mesh通信， 通信质量接近PSTN网络;
GTalk: 谷歌山寨的Skype

瑞典GIPS公司：开发出3A算法；
GIPS被谷歌收购，GIPS与 GTalk 的libjingle合并成为早期WebRTC一部分。

场景：社交IM的1v1通信；
优点：轻量级、架构拓展性强； 
缺点：跨运营商无法保证Qos，去中心化后无法监控、管控；

## 2003～2012: SFU
MMO RPG游戏兴起；

场景：游戏频道语音沟通、直播聊天室。
优点：架构简洁、实时性强，无需中心化合流聚合；
缺点：客户端下行压力大，互动人数有最大上限；

## SFU vs MCU
SFU : Selective Forwarding Unit, 上行一路流，下行N路流，服务端仅转发。客户端压力大。
MCU: MultiPoint Control Unit, 中心化，上行下行都一路流，服务端负责合流。服务端压力大。

{% img /images/2022-01/sfu_mcu.png 800 1200 sfu_mcu.png %}

# 基础评估指标
## 性能相关：
- 功耗
- cpu/gpu/内存占用
- 引发降频时间(手机)
## 屏幕分享相关
- 清晰度&分辨率
- 色彩准确度
- 相对静止场景最低码率
- 相对运动场景流畅度（文档滚动、局部动画）
基础端到端延时(<1s)
## 音频相关
- 频宽
- 基础音质（POLQA评分>2.5）
- 基础端到端延时
- 3A(增益控制、噪声一直、回声抵消)
## 视频相关
- 基础画质
- 基础流畅度（帧率、帧间隔）
- 基础端到端延时
- 音画同步

# 直播过程
## 对于主播：
1。申请开始直播:http; // 返回ok
2。获取推流目标:信令服务器 -> 创建房间-> 获取当前位置合适的MCU节点；
3。KTP推流到MCU节点；（UDP）
4。MCU存到SRS(经过RTMP转码);

## 对于用户,拉数据:
用户 -(flv over http)-> CDN -> Router -> SRS




# 参考资料
https://segmentfault.com/a/1190000040760567








