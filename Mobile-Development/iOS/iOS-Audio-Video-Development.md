# iOS 音视频开发知识点

## 1. 音视频基础

### 1.1 音频基础

音频是连续声波的数字化表示，核心参数包括：

| 参数 | 含义 | 常见值 |
| --- | --- | --- |
| 采样率 | 每秒采样次数 | 44.1kHz、48kHz |
| 位深 | 每个采样点的位数 | 16-bit、24-bit |
| 声道数 | 单声道或双声道 | 1、2 |
| 码率 | 每秒数据量 | 64kbps、128kbps |
| 编码格式 | 压缩算法 | AAC、MP3、Opus、PCM |

PCM 是未压缩的原始音频数据，体积较大：

```text
数据量 = 采样率 × 位深 × 声道数 × 时长
48kHz × 16bit × 2ch × 60s ≈ 11MB
```

### 1.2 视频基础

视频是连续图像帧的序列，核心参数包括：

| 参数 | 含义 |
| --- | --- |
| 分辨率 | 每帧像素数量，例如 1920×1080 |
| 帧率 | 每秒帧数，例如 24、30、60fps |
| 码率 | 每秒视频数据量 |
| 编码格式 | H.264、H.265、AV1 等 |
| 色彩格式 | YUV、RGB、NV12、BGRA |

码率、分辨率、帧率共同决定清晰度、流畅度、带宽和功耗。

### 1.3 容器格式与编码格式

容器负责封装音频、视频、字幕、元数据；编码格式负责压缩数据。

| 容器 | 可包含 |
| --- | --- |
| MP4 | H.264/H.265 + AAC |
| MOV | Apple 生态常见 |
| M3U8/TS | HLS 流媒体 |
| FLV | 直播历史常见 |

不要混淆：

- MP4 是容器
- H.264 是视频编码
- AAC 是音频编码

---

## 2. iOS 音视频框架

| 框架 | 作用 |
| --- | --- |
| AVFoundation | 播放、录制、编辑、导出核心框架 |
| Core Media | 时间、SampleBuffer、格式描述 |
| Core Video | PixelBuffer、纹理缓存 |
| Core Audio | 底层音频处理 |
| AudioToolbox | 音频队列、格式转换 |
| VideoToolbox | 硬件编解码 |
| Core Image | 图像滤镜 |
| Metal | 高性能 GPU 渲染 |

典型链路：

```text
采集: AVCaptureSession
  ↓
帧数据: CMSampleBuffer / CVPixelBuffer
  ↓
处理: Core Image / Metal / Accelerate
  ↓
编码: VideoToolbox / AVAssetWriter
  ↓
封装: MP4 / HLS
```

---

## 3. 音频播放

### 3.1 AVAudioPlayer

适合本地短音频或简单播放：

```objc
NSURL *url = [[NSBundle mainBundle] URLForResource:@"sound" withExtension:@"mp3"];
AVAudioPlayer *player = [[AVAudioPlayer alloc] initWithContentsOfURL:url error:nil];
[player prepareToPlay];
[player play];
```

特点：

- 简单易用
- 适合本地文件
- 不适合复杂流媒体

### 3.2 AVPlayer

适合本地和网络媒体播放：

```objc
NSURL *url = [NSURL URLWithString:@"https://example.com/video.mp4"];
AVPlayer *player = [AVPlayer playerWithURL:url];
[player play];
```

监听状态：

```objc
[player.currentItem addObserver:self
                      forKeyPath:@"status"
                         options:NSKeyValueObservingOptionNew
                         context:nil];
```

常见状态：

- `AVPlayerItemStatusReadyToPlay`
- `AVPlayerItemStatusFailed`
- `AVPlayerItemStatusUnknown`

### 3.3 AVAudioSession

音频会话决定 App 如何与系统音频环境协作。

```objc
AVAudioSession *session = [AVAudioSession sharedInstance];
[session setCategory:AVAudioSessionCategoryPlayback error:nil];
[session setActive:YES error:nil];
```

常见 Category：

| Category | 场景 |
| --- | --- |
| `Playback` | 音乐、视频播放、后台播放 |
| `PlayAndRecord` | 通话、语音房 |
| `Record` | 录音 |
| `Ambient` | 与其他 App 混音 |
| `SoloAmbient` | 默认独占，受静音键影响 |

需要处理：

- 来电中断
- Siri 中断
- 耳机插拔
- 蓝牙切换
- 扬声器/听筒切换
- 后台播放

---

## 4. 音频录制与处理

### 4.1 AVAudioRecorder

简单录音：

```objc
NSDictionary *settings = @{
    AVFormatIDKey: @(kAudioFormatMPEG4AAC),
    AVSampleRateKey: @44100,
    AVNumberOfChannelsKey: @2,
    AVEncoderAudioQualityKey: @(AVAudioQualityHigh)
};

AVAudioRecorder *recorder = [[AVAudioRecorder alloc] initWithURL:fileURL
                                                        settings:settings
                                                           error:nil];
[recorder record];
```

### 4.2 AVCaptureSession 采集音频

适合音视频同步录制：

```objc
AVCaptureSession *session = [[AVCaptureSession alloc] init];
AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeAudio];
AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:device error:nil];
[session addInput:input];
```

### 4.3 实时音频处理

实时场景关注：

- 回声消除 AEC
- 自动增益 AGC
- 噪声抑制 NS
- 采样率转换
- 混音
- 音量检测

语音通话通常使用：

```objc
[session setCategory:AVAudioSessionCategoryPlayAndRecord
         withOptions:AVAudioSessionCategoryOptionAllowBluetooth |
                     AVAudioSessionCategoryOptionDefaultToSpeaker
               error:nil];
[session setMode:AVAudioSessionModeVoiceChat error:nil];
```

---

## 5. 视频播放

### 5.1 AVPlayerLayer

```objc
AVPlayer *player = [AVPlayer playerWithURL:url];
AVPlayerLayer *playerLayer = [AVPlayerLayer playerLayerWithPlayer:player];
playerLayer.frame = self.view.bounds;
[self.view.layer addSublayer:playerLayer];
[player play];
```

### 5.2 播放状态监听

常见监听：

- `status`
- `loadedTimeRanges`
- `playbackLikelyToKeepUp`
- `playbackBufferEmpty`
- `timeControlStatus`

周期性进度：

```objc
[player addPeriodicTimeObserverForInterval:CMTimeMake(1, 2)
                                     queue:dispatch_get_main_queue()
                                usingBlock:^(CMTime time) {
    NSLog(@"%f", CMTimeGetSeconds(time));
}];
```

注意释放 time observer，避免泄漏。

### 5.3 HLS

HLS 使用 `.m3u8` 索引和分片传输，适合直播和点播。

优势：

- 自适应码率
- CDN 友好
- iOS 原生支持
- 容错较好

劣势：

- 延迟通常高于 WebRTC
- 首屏依赖分片和关键帧

---

## 6. 视频采集

### 6.1 AVCaptureSession 架构

```text
AVCaptureSession
  ├── Inputs
  │   ├── Camera Input
  │   └── Microphone Input
  └── Outputs
      ├── AVCaptureVideoDataOutput
      ├── AVCaptureAudioDataOutput
      └── AVCaptureMovieFileOutput
```

### 6.2 采集示例

```objc
AVCaptureSession *session = [[AVCaptureSession alloc] init];
session.sessionPreset = AVCaptureSessionPreset1280x720;

AVCaptureDevice *camera = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
AVCaptureDeviceInput *input = [AVCaptureDeviceInput deviceInputWithDevice:camera error:nil];
[session addInput:input];

AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
output.videoSettings = @{
    (id)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_420YpCbCr8BiPlanarFullRange)
};

dispatch_queue_t queue = dispatch_queue_create("video.capture.queue", DISPATCH_QUEUE_SERIAL);
[output setSampleBufferDelegate:self queue:queue];
[session addOutput:output];

[session startRunning];
```

回调：

```objc
- (void)captureOutput:(AVCaptureOutput *)output
didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
       fromConnection:(AVCaptureConnection *)connection {
    CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
}
```

### 6.3 预览

```objc
AVCaptureVideoPreviewLayer *previewLayer = [AVCaptureVideoPreviewLayer layerWithSession:session];
previewLayer.videoGravity = AVLayerVideoGravityResizeAspectFill;
previewLayer.frame = self.view.bounds;
[self.view.layer addSublayer:previewLayer];
```

采集回调不要放主线程，预览使用 `AVCaptureVideoPreviewLayer` 更高效。

---

## 7. 时间戳与 SampleBuffer

### 7.1 PTS、DTS、Duration

| 概念 | 含义 |
| --- | --- |
| PTS | 显示或播放时间 |
| DTS | 解码时间 |
| Duration | 样本持续时间 |

没有 B 帧时，PTS 和 DTS 通常相同；存在 B 帧时，解码顺序和显示顺序不同。

```objc
CMTime pts = CMSampleBufferGetPresentationTimeStamp(sampleBuffer);
CMTime duration = CMSampleBufferGetDuration(sampleBuffer);
Float64 seconds = CMTimeGetSeconds(pts);
```

### 7.2 CMTime

`CMTime` 用分数表示时间，避免浮点误差：

```objc
CMTime frameDuration = CMTimeMake(1, 30);
CMTime timestamp = CMTimeMultiply(frameDuration, 10);
```

不要用 `double` 长时间累加帧时间。

### 7.3 CMSampleBuffer 与 CVPixelBuffer

| 类型 | 作用 |
| --- | --- |
| `CMSampleBuffer` | 音视频样本容器，包含数据、格式、时间戳 |
| `CVPixelBuffer` | 视频像素数据 |
| `CMFormatDescription` | 格式描述 |

跨线程持有 `CMSampleBuffer` 时要注意生命周期，必要时 `CFRetain/CFRelease`。

---

## 8. 编码与解码

### 8.1 H.264/H.265 概念

| 概念 | 含义 |
| --- | --- |
| I 帧 | 关键帧，可独立解码 |
| P 帧 | 依赖前向参考 |
| B 帧 | 双向参考，压缩率高但增加延迟 |
| GOP | 一组帧，通常从关键帧开始 |
| SPS/PPS | H.264 解码参数集 |
| IDR | 解码刷新点 |

低延迟场景通常减少或禁用 B 帧，控制 GOP 长度。

### 8.2 VideoToolbox 编码

VideoToolbox 提供硬件编解码能力。

编码流程：

```text
创建 VTCompressionSession
  ↓
配置编码参数
  ↓
输入 CVPixelBuffer
  ↓
输出编码后的 CMSampleBuffer
```

关键参数：

- 分辨率
- 帧率
- 码率
- Profile
- 关键帧间隔
- 是否实时编码

### 8.3 AVAssetWriter

用于写入媒体文件：

```objc
AVAssetWriter *writer = [[AVAssetWriter alloc] initWithURL:outputURL
                                                  fileType:AVFileTypeMPEG4
                                                     error:nil];
```

常和 `AVCaptureVideoDataOutput`、`AVCaptureAudioDataOutput` 结合，手动写入音视频样本。

---

## 9. 视频编辑与导出

### 9.1 AVMutableComposition

用于组合音视频轨道：

```objc
AVMutableComposition *composition = [AVMutableComposition composition];
AVMutableCompositionTrack *videoTrack =
    [composition addMutableTrackWithMediaType:AVMediaTypeVideo
                             preferredTrackID:kCMPersistentTrackID_Invalid];
```

可实现：

- 裁剪
- 拼接
- 音视频合成
- 音轨替换

### 9.2 AVMutableVideoComposition

用于视频画面处理：

- 旋转
- 缩放
- 转场
- 水印
- 滤镜

### 9.3 AVAssetExportSession

简单导出：

```objc
AVAssetExportSession *exporter =
    [[AVAssetExportSession alloc] initWithAsset:composition
                                     presetName:AVAssetExportPresetHighestQuality];
exporter.outputURL = outputURL;
exporter.outputFileType = AVFileTypeMPEG4;
[exporter exportAsynchronouslyWithCompletionHandler:^{
    NSLog(@"%@", @(exporter.status));
}];
```

限制：

- 自定义编码参数能力有限
- 复杂逐帧处理不灵活
- 高级需求使用 `AVAssetReader + AVAssetWriter`

---

## 10. 渲染与滤镜

### 10.1 Core Image

适合中等复杂度滤镜：

```objc
CIImage *input = [CIImage imageWithCVPixelBuffer:pixelBuffer];
CIFilter *filter = [CIFilter filterWithName:@"CISepiaTone"];
[filter setValue:input forKey:kCIInputImageKey];
CIImage *output = filter.outputImage;
```

### 10.2 Metal

适合高性能实时渲染：

- 美颜
- 滤镜
- 转场
- 视频特效
- 自定义 shader

### 10.3 PixelBuffer 与纹理

通过 `CVMetalTextureCache` 可把 `CVPixelBuffer` 转成 Metal Texture，减少拷贝。

高性能原则：

- 减少 CPU/GPU 数据拷贝
- 复用 `CVPixelBufferPool`
- 避免频繁创建纹理和上下文
- 尽量使用硬件路径

---

## 11. 实时音视频

### 11.1 直播链路

```text
采集
  ↓
前处理
  ↓
编码
  ↓
封包
  ↓
推流
  ↓
服务器转发
  ↓
拉流
  ↓
解码
  ↓
渲染
```

### 11.2 WebRTC

WebRTC 适合低延迟实时通信。

核心能力：

- 音视频采集
- 编解码
- 抖动缓冲
- 拥塞控制
- 回声消除
- NAT 穿透
- 端到端传输

### 11.3 延迟优化

低延迟策略：

- 降低 GOP
- 减少 B 帧
- 控制缓冲区
- 动态码率
- 网络拥塞控制
- 丢弃过期视频帧
- 音频优先

---

## 12. 性能优化

### 12.1 内存优化

音视频帧数据很大，内存容易暴涨。

优化：

- 使用 `CVPixelBufferPool`
- 及时释放 Core Foundation 对象
- 避免队列积压
- 控制分辨率和帧率
- 不在回调中做同步 I/O
- 大对象使用 autoreleasepool

### 12.2 丢帧策略

实时场景中，延迟比完整性更重要。

```objc
videoOutput.alwaysDiscardsLateVideoFrames = YES;
```

策略：

- 预览链路只保留最新帧
- 网络差时降码率
- 处理不过来时丢弃非关键帧
- 音频优先保证连续性

### 12.3 线程模型

推荐：

- 采集回调使用串行后台队列
- 编码使用独立队列
- 渲染回主线程或渲染线程
- 文件写入独立队列
- 避免多个重任务抢同一队列

### 12.4 首帧优化

首帧慢原因：

- DNS/TCP/TLS 慢
- 等待关键帧
- 缓冲策略过保守
- 解码器初始化慢
- 主线程阻塞
- 图片或纹理初始化慢

优化：

- 预连接
- 降低首屏缓冲
- 服务端更快下发关键帧
- 复用播放器
- 异步初始化

---

## 13. 权限与后台

### 13.1 权限

需要配置：

- `NSCameraUsageDescription`
- `NSMicrophoneUsageDescription`
- `NSPhotoLibraryUsageDescription`

请求相机权限：

```objc
[AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo
                          completionHandler:^(BOOL granted) {
    dispatch_async(dispatch_get_main_queue(), ^{
        // update UI
    });
}];
```

### 13.2 后台播放

需要：

- 开启 Background Modes：Audio
- 配置 `AVAudioSessionCategoryPlayback`
- 管理远程控制和锁屏信息

---

## 14. 常见问题

| 问题 | 常见原因 |
| --- | --- |
| 播放失败 | URL、证书、格式、网络、ATS |
| 音画不同步 | 时间戳、缓冲、丢帧策略 |
| 录制黑屏 | 权限、session 配置、线程阻塞 |
| 发热严重 | 分辨率高、帧率高、软编、滤镜重 |
| 内存暴涨 | buffer 积压、未释放、无 pool |
| 首帧慢 | 网络、关键帧、缓冲、解码器初始化 |
| 蓝牙无声 | AudioSession category/mode/options 配置错误 |

---

## 15. 面试高频问题

1. **AVFoundation 中 AVCaptureSession 的作用？**  
   它统一管理采集输入和输出，是音视频采集链路的核心协调者。

2. **CMSampleBuffer 和 CVPixelBuffer 区别？**  
   CMSampleBuffer 是样本容器，包含时间戳和格式信息；CVPixelBuffer 是视频像素数据。

3. **PTS 和 DTS 区别？**  
   PTS 是显示时间，DTS 是解码时间；有 B 帧时两者可能不同。

4. **为什么实时音视频要丢帧？**  
   处理不过来时排队会增加延迟，实时场景更关注低延迟而不是每帧都显示。

5. **硬编码比软编码优势？**  
   硬编码功耗低、性能好、发热少，但参数灵活性可能不如软编码。

6. **HLS 和 WebRTC 区别？**  
   HLS 适合大规模分发和点播直播，延迟较高；WebRTC 适合低延迟实时通信。

---

## 16. 总结

iOS 音视频开发围绕采集、处理、编码、封装、传输、解码、渲染展开。AVFoundation 负责上层能力，Core Media/Core Video 管理底层数据结构，VideoToolbox 提供硬件编解码，Metal/Core Image 负责图像处理。

工程上最关键的是理解时间戳、线程模型、缓冲策略、硬件编解码、内存复用和 AudioSession。音视频问题往往不是单点 API 问题，而是链路中任一环节的时序、性能或资源管理问题。
