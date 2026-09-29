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
AWB
   ↓
AE
   ↓
Gamma
   ↓
Color Correction
   ↓
2D/3D NR
   ↓
Sharpen
   ↓
WDR/HDR
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



## 2. manger    
### webUI  (httpd 轻量化)

### mqtt  

### media server  
媒体服务 : 
rtsp : live555 或 自研
web 预览：webrtc 




## 3.hardware/debug/update system  
### OTA   
### log export  
### watchdog  



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