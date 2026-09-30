# IPC    



开源项目  
（1）OpenIPC  


（2）Thingino  



## 1. video stream  
video 典型链路  
```
                         ┌──→ 缩放/CSC → NPU → 检测/跟踪/规则引擎 ──┐
                         │                                        │
Sensor → MIPI CSI → ISP ─┼──→ VPSS/Scaler → OSD/Privacy → VENC ──┼→ RTSP
                         │                       ↑                ├→ GB28181
                         ├──→ JPEG Snapshot      │                ├→ WebRTC
                         │                       │                └→ 本地录像
                         └───────────────────────┘
                              AI metadata/boxes
```
这里只做local 服务，媒体服务独立或与web服务一起

### mipi / vi / isp  
```
sensor
  │
  ├── MIPI CSI-2 / DPHY
  ├── I2C control
  ├── reset
  ├── pwdn
  ├── mclk
  └── power sequence
```   
```
---ISP---
RAW Bayer
   ↓
Black Level Correction
   ↓
Defect Pixel Correction
   ↓
Lens Shading Correction
   ↓
Demosaic
   ↓
AWB（自动白平衡）
   ↓
AE （自动曝光）
   ↓
Gamma
   ↓
Color Correction
   ↓
2D/3D NR
   ↓
Sharpen
   ↓
WDR/HDR(逆光)
   ↓
YUV
```

### VPSS / Scaler
包含
```
Crop 抠图
Resize 放大 / 缩小分辨率
CSC 色彩空间转换，例如 YUV420 ↔ YUV422
Rotate 旋转画面
Mirror 镜像左右翻转
Flip 上下翻转
多路输出
```
```
2688x1520
    │
    ├── CH0 → 2688x1520 → H265 主码流
    │
    ├── CH1 → 640x360   → H264 子码流
    │
    ├── CH2 → 640x640   → AI
    │
    └── CH3 → 1920x1080 → JPEG
```

### 推理 / 识别 / 跟踪  

### 画框 / 编码
#### OSD  



#### venc   
```
Codec:
    H264
    H265

Resolution:
    2688x1520

FPS:
    25

Rate Control:
    CBR
    VBR
    FIXQP

Bitrate:
    2048 / 4096 / 8192 kbps

GOP:
    25 / 50

Profile:
    Main
    High

I-frame interval:
    50
```


## 2. manger    
### webUI  (httpd 轻量化)

### mqtt  

### media server  
rtsp(必须有) : live555 (轻量，低并发) ,sever端等待client连接  

browser player:  
(1) IPC 网页内置 JS 播放器，IPC 开 HTTP+WebSocket，传输 FLV / MJPEG  **推荐,延迟300ms~1s**  
(2) IPC 内置轻量 WebRTC Peer(P2P)    **复杂，延迟低100~300ms**  
(3) MJPEG IPC HTTP 接口持续输出一帧一帧 JPEG 图片，网页`<img>`标签不断刷新。  **码率大，卡顿**  
(4) 网页请求 IPC 的 RTSP，但是网页端用插件  **额外安装插件**  

zlMediaKit : 兼容多种格式   


## 3.hardware/debug/update system  
考虑到IPC的应用场景，不方便随时串口/usb调试，依赖网络的升级/调试/保活需要做好    
### OTA   
优先ab 分区,托底机制 recovery
**网络升级**：（1）局域网自动搜索 （2）联网服务器下发  
### log export  
日志单独分区，或写入TF卡，保留7~30天  
### watchdog  
外围接独立看门狗  



## 4.架构设计
Event System  
```
             ┌ AI
             ├ Motion Detection
             ├ GPIO
             ├ Tamper
Event Bus ←──┼ Storage Error
             ├ Network Error
             └ Temperature
                  │
                  ↓
             Event Manager
                  │
       ┌──────────┼───────────┐
       ↓          ↓           ↓
      MQTT     GB28181      ONVIF
       ↓          ↓           ↓
     Webhook    Alarm       Events
       ↓
     Record
```


## 容易忽略但实用的功能  
###  音频子系统 

全双工对话需要做到    **难度较大，先做半双工**
```
AEC  回声消除
AGC  自动增益
ANR  降噪
```

### 日夜切换
IR-CUT（ICR） : 红外双滤光片切换器  
IR LED : 红外补光灯  
- **白天 / 强光**：切到【红外截止滤光片】
CMOS/CCD 传感器本身能感知红外线，红外光会造成画面偏红、色彩失真；这一片把红外挡住，画面色彩正常彩色。
- **夜晚 / 低照度**：把红外截止片移开，换成【全光谱玻璃】
允许红外光进入传感器，配合红外补光灯（IR-LED），拍出黑白夜视画面
IO 控制，ISP联动

## 拓展  
ONVIF： 
国际开放协议，设备之间互通标准，各种 IPC/NVR 互相对接，偏局域网设备发现、媒体流、能力协商  
```
主要模块  
1. Discovery（WS-Discovery）：局域网自动发现设备，客户端搜 IPC 就是靠这个
2. Device Management：获取设备信息、时间、参数、重启
3. Media / Media2：获取视频 / 音频配置，获取 RTSP 流地址
4. PTZ：云台控制
5. Event：移动侦测、报警事件上报
6. Audio Backchannel：就是前面讲的双向语音对讲，客户端往 IPC 发反向音频流
```    

```
信令：WebService（XML）
媒体流：RTSP + RTP
端口：默认 80（HTTP）、554（RTSP）
```  

GB28181：
中国国家标准《公共安全视频监控联网系统信息传输、交换、控制技术要求》
目标：把各地视频监控统一接入公安 / 政务平台（雪亮工程、平安城市）。

```
信令：SIP（会话初始协议），端口默认 5060
媒体流：RTP（可以 TCP/UDP，不强制 RTSP）
设备要向平台注册、定期保活（心跳），平台知道设备在线状态
```

