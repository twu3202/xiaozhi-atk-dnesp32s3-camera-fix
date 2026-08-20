# 小智 ESP32 — 正点原子 DNESP32S3 摄像头花屏修复 (OV5640)

修复上游 [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) **v2.2.6** 在
**正点原子 DNESP32S3**(ESP32-S3, 16MB Flash / 8MB PSRAM)板上的摄像头问题。

> **TL;DR** — 上游 `atk-dnesp32s3` 板型的摄像头走 `esp_video`(DVP **单缓冲**)。
> 设备一联网,WiFi 抢占 PSRAM 带宽把 DVP DMA 冲断 → **严重花屏 / 绿屏 / 断纹**。
> 改用 legacy `esp32-camera` 组件 + 双缓冲 + QVGA 后画面稳定。
> 出厂测试固件能跑实时视频,说明**硬件没问题**,是框架在这块板上不稳。

修复后状态:**OV5640 识别 ✅ 颜色正常 ✅ 画面无花屏 ✅ ES8388 音频 ✅ 服务器端实时打断 ✅**

## 修了三个问题

| # | 症状 | 根因 | 修法 |
|---|---|---|---|
| 1 | 联网后**花屏/绿屏/断纹** | `esp_video` DVP 单缓冲,WiFi 争用 PSRAM 带宽时 DMA 被冲断(同类问题见上游 issue #1530) | 换 legacy `Esp32Camera`,`fb_count=2` 双缓冲,`grab_mode=GRAB_LATEST` |
| 2 | 肤色**偏蓝** | OV5640 (esp_cam_sensor 1.5.2) 的 DVP YUV422 输出 U/V(Cb/Cr) 顺序与标准 YUYV 相反 | 整帧 `memcpy` 出来后,在**私有副本**上原地交换每个 2 像素宏块内的两个色度字节 |
| 3 | VGA 分辨率下 `EV-EOF-OVF` | 640×480 时 DMA→PSRAM 拷贝溢出 | 降到 **QVGA (320×240)**,数据量只有 1/4;对云端 VLM 识别足够 |

问题 2 的修法有个细节值得说:**先整帧 `memcpy` 再改色度**,而不是边读边换字节。
逐字节慢读会拉长"读取 DMA 缓冲区"的时间窗,期间 DVP 可能正在改写同一块内存 → 随机区域帧撕裂。
拷到私有副本后再慢慢处理就没有争用了。

## 关键配置

板载摄像头由**外部晶振**供时钟,PWDN/RESET 走 XL9555 IO 扩展 —— 这几项配错了就识别不到:

```c
config.pin_xclk      = -1;          // 板载外部晶振供 XCLK,ESP 不产生时钟
config.xclk_freq_hz  = 24000000;    // 外部晶振频率,供 OV5640 算 PLL
config.pin_pwdn      = -1;          // 由 XL9555 控制,不用 ESP GPIO
config.pin_reset     = -1;          // 同上
config.sccb_i2c_port = 1;           // 主 I2C 是 0,摄像头 SCCB 独占 1
config.pixel_format  = PIXFORMAT_RGB565;
config.frame_size    = FRAMESIZE_QVGA;
config.fb_count      = 2;
config.fb_location   = CAMERA_FB_IN_PSRAM;
config.grab_mode     = CAMERA_GRAB_LATEST;
```

初始化前需手动经 XL9555 拉低 PWDN、拉低再释放 RESET(各留 50ms)。

`sdkconfig` 追加项:

```
CONFIG_BOARD_TYPE_ATK_DNESP32S3=y
CONFIG_OV5640_SUPPORT=y          # esp32-camera 支持 OV5640
CONFIG_USE_SERVER_AEC=y          # 服务器端 AEC → 说话即打断
```

## 使用

```bash
git clone https://github.com/78/xiaozhi-esp32.git
cd xiaozhi-esp32
git checkout v2.2.6
git apply /path/to/atk-dnesp32s3-ov5640-camera-fix.patch
idf.py -DBOARD_NAME=atk-dnesp32s3 -DBOARD_TYPE=atk-dnesp32s3 build
```

补丁改动三个文件:

- `main/boards/atk-dnesp32s3/atk_dnesp32s3.cc` — 换用 `Esp32Camera`(问题 1、3)
- `main/boards/atk-dnesp32s3/config.json` — `CONFIG_CAMERA_OV2640_*` → `OV5640_*`(**本板实际是 OV5640,上游写的是 OV2640**)
- `main/boards/common/esp_video.cc` — DVP 也用双缓冲 + YUV422 色度修正

> 第三个文件的改动,在换用 `esp32-camera` 之后**对本板已不再生效**,保留是给**仍想留在 `esp_video` 框架**上的人参考 —— 单缓冲改双缓冲 + 那个 memcpy 顺序,是可以独立套用的。

## 验证环境

- 板:正点原子 DNESP32S3(ESP32-S3 QFN56 rev v0.2,16MB Flash / 8MB PSRAM)
- 摄像头:OV5640(DVP,SCCB 地址 0x3c);音频:ES8388 双工;屏:2.4″ SPI LCD
- 摄像头引脚(`config.h`):D0–7 = GPIO4/5/6/7/15/16/17/18,VSYNC=47,HREF=48,PCLK=45,SIOD=39,SIOC=38,XCLK=NC(外部晶振)
- ESP-IDF **v5.5.2**,上游 xiaozhi-esp32 **v2.2.6**

## 说明与已知限制

- 这是**针对单一板型的修复**,只在上面这一块板上验证过,针对 v2.2.6。
- 视觉推理全部在官方 **xiaozhi.me 云端**跑,设备只负责拍照上传(`self.camera.take_photo`)。
- 服务器端 AEC 实时打断**能用但不够灵敏**(云端判定延迟,属正常)。
- QVGA 画质对云端 VLM 识别足够;要更高分辨率得先解决 DMA→PSRAM 的溢出。
- **本仓库只提供源码补丁,不提供编译好的固件** —— 完整镜像会带上设备自身的 NVS(WiFi 配置、设备绑定状态),且内含 Espressif ESP-SR 唤醒词模型等有各自再分发条款的二进制。请自行编译。

## 致谢

- [78/xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) — 上游小智固件
- [espressif/esp32-camera](https://github.com/espressif/esp32-camera) — legacy 摄像头驱动
- 正点原子(ALIENTEK)— DNESP32S3 开发板与官方示例参数

补丁基于上游 v2.2.6,遵循上游许可证。
