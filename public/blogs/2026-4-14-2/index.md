  那么在前面那么多准备工作完成的情况下，接下来就是真正的解码过程了。但是先不急着写代码，由于解码的过程还是比较繁琐的，所以最开始先梳理一遍解码的整体流程。

# 一、解码整体流程

1 初始化解码器/Frames和Socket<br>
2 采用专门线程不断读取设备（socket/h264流）数据<br>
3 通过 FFmpeg 相关函数，将 H.264 数据解码为 YUV 视频帧（AVFrame）<br>
4 通知外部有新的一帧解码完成，供渲染或其它处理使用

# 二、设计主要的类和对象

- Decoder：负责管理解码主流程（线程、FFmpeg相关对象的创建、释放、解码循环）
- Frames：用于线程安全地管理解码过程中的两帧数据（解码帧、渲染帧），方便解码线程和UI线程之间的数据同步。
- DeviceSocket：用于从设备端（例如安卓手机）获取 H.264 裸流数据
- FFmpeg 相关函数：真正解码H.264的工具

# 三、详细流程代码分解

相关成员变量

<b>Decoder.h</b>

```cpp
private:
  QPointer<DeviceSocket> m_deviceSocket; // 接收h264数据
  bool m_quit = false;                   // 退出标记
  Frames *m_frames;                      // 解码出的帧
```

<b>Frames.h</b>

```cpp
private:
  // 解码出来的一帧数据（yuv）
  // 保存正在解码的yuv
  AVFrame *m_decodingFrame = nullptr;
  // 保存正在渲染的yuv
  AVFrame *m_renderingframe = nullptr;
  // 保证AVFrame的多线程安全
  QMutex m_mutex;
  bool m_renderingFrameConsumed = true;
```

## 1. 相关对象创建和初始化

### 1.1 静态初始化和反初始化

```cpp
bool Decoder::init() {
  if (avformat_network_init()) { // 初始化FFmpeg的网络模块（例如，访问rtsp/http/udp等协议）
      return false;
  }
  return true;
}

void Decoder::deInit() {
  avformat_network_deinit(); // 反初始化网络模块
}
```

- ```avformat_network_init()/deinit()```：如果解码的数据来源于网络（比如socket），需要初始化/释放ffmpeg的网络环境，一般只需要调用一次。

## 2. 启动解码线程

```cpp
bool Decoder::startDecode() {
  if (!m_deviceSocket) {
    return false;
  }
  m_quit = false;
  start();  // QThread 的 start，会自动调用 run() （多线程解码）
  return true;
}
```

- 使用线程解码，解码不会卡住主线程

## 3. 解码主流程（关键）

```cpp
void Decoder::run() {
  struct ResourceGuard { ... ~ResourceGuard() { ... } } resources;
```

- 这里用 ResourceGuard 结构体专门负责资源释放，防止内存泄漏

### 3.1 创建解码用缓冲区

```cpp
resources.decoderBuffer = (unsigned char *)av_malloc(BUFSIZE);
```

```av_malloc```：FFmpeg 的专用分配内存函数

### 3.2 初始化 FFmpeg 自定义数据读取IO

```cpp
resources.avioCtx = avio_alloc_context(
  resources.decoderBuffer, BUFSIZE, 0,
  this, readPacket, NULL, NULL
);
```

- ```avio_alloc_context```：FFmpeg提供的创建自定义IO的入口，可以让FFmpeg通过指定的回调readPacket读取数据
  
  - ```resources.decoderBuffer```：内部使用的缓冲
  - ```BUFSIZE```：缓冲区长度
  - ```readPacket```：读取数据的自定义函数（从socket读取h264流）

### 3.3 格式上下文初始化

```cpp
resources.formatCtx = avformat_alloc_context(); // 分配封装格式上下文
// 指定自定义的io
resources.formatCtx->pb = resources.avioCtx;

if (avformat_open_input(&resources.formatCtx, NULL, NULL, NULL) < 0) {
  // 打开输入失败
  return;
}
resources.isFormatCtxOpen = true;
```

- ```avformat_alloc_context```：分配AVFormatContext
- ```avformat_open_input```：让ffmpeg根据自定义io解析输入流格式（如获得文件流中的各种流信息，自动识别视频/音频等），这里传NULL表示用自定义IO（通常是文件名）

### 3.4 查找解码器

```cpp
const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
```

### 3.5 解码器上下文初始化

```cpp
resources.codecCtx = avcodec_alloc_context3(codec); // 分配解码器上下文
if (avcodec_open2(resources.codecCtx, codec, NULL) < 0) {
  // 打开解码器失败
  return;
}
resources.isCodecCtxOpen = true;
```

- ```avcodec_alloc_context3```：创建AVCodecContext（解码器实例）
- ```avcodec_open2```：用指定解码器初始化解码器上下文

### 3.6 创建解码数据包

```cpp
resources.packet = av_packet_alloc();
resources.packet->data = nullptr;
resources.packet->size = 0;
```

- ```av_packet_alloc```：分配一个AVPacket，用于存储一帧未解码的压缩数据（如h264一帧）

## 4. 解码主循环

```cpp
while (!m_quit && !av_read_frame(resources.formatCtx, resources.packet)) {
    // ...
}
```

- ```av_read_frame```：从流中读取下一帧数据（如H.264一帧），存到 packet 中

### 4.1 数据解码

```cpp
AVFrame *decodingFrame = m_frames->decodingFrame();
if ((ret = avcodec_send_packet(resources.codecCtx, resources.packet)) < 0) {
  // 送数据给解码器
  return;
}
if (decodingFrame) {
  ret = avcodec_receive_frame(resources.codecCtx, decodingFrame); // 获取解码后的帧(yuv)
}
```

- ```avcodec_send_packet```：将一个压缩帧（例如 packet，H.264）送入解码器
- ```avcodec_receive_frame```：从解码器中获取解码得到的一帧原始图像（YUV，保存在 AVFrame 里）

### 4.2 Frame帧管理

成功后调用：
```cpp 
pushFrame();
```

- 内部通过``Frames::offerDecodedFrames()``进行帧数据swap，然后通过信号```onNewFrame```通知外界有新帧

### 4.3 其他流程

- 循环解码直到数据流结束（socket关闭、用户请求停止等）

# 四、Frames对象的作用

- **解码线程和渲染线程之间需要安全地交换帧数据**，这个过程由```Frames```类负责
- ```m_decodingFrame```、```m_renderingframe```为两个缓冲帧，通过```swap()```实现安全切换
- 通过互斥锁（QMutex）来保证线程同时访问数据不会出错
- 标记```m_renderingFrameConsumed```表示一帧数据是否已被渲染线程消费

## 五、DeviceSocket的作用

- recvData函数内部会调用```m_deviceSocket->subThreadRecvData```从真实的设备/Socket抓取新的一批h264数据，作为 FFmpeg 解码器的数据输入。

## 六、总结流程

1 ```Decoder::startDecode()``` 创建解码线程
2 ```run()```中按顺序初始化 FFmpeg 解码相关对象
3 在子线程中不停读取 socket 数据（通过 FFmpeg 读取回调 ```readPacket```）
4 不断送数据到 H.264 解码器，并获取 YUV 帧
5 解码出来的帧通过```Frames```进行缓冲和线程同步
6 通知外界解码完成，可以进行下一步处理

## 七、核心 FFmpeg 函数和用途汇总

| 函数名 | 作用简述 |
| :--- | :--- |
| avformat_network_init | 初始化网络功能（socket/rtsp等） |
| av_malloc | 分配FFmpeg专用的内存 |
| avio_alloc_context | 分配带自定义IO（回调）的IO上下文 |
| avformat_alloc_context | 分配封装格式上下文AVFormatContext |
| avformat_open_input | 打开输入流（文件/网络/自定义流） |
| avcodec_find_decoder | 查找指定编解码器（如H264） |
| avcodec_alloc_context3 | 分配编解码器上下文 |
| avcodec_open2 | 初始化编解码器上下文 |
| av_packet_alloc | 分配未解码数据数据包（AVPacket） |
| av_read_frame | 读取下一帧压缩数据包 |
| avcodec_send_packet | 向解码器送压缩包（输入流） |
| avcodec_receive_frame | 从解码器取出解码结果帧 |
| av_frame_alloc | 分配解码后的帧数据（YUV等） |
| av_frame_free | 释放帧资源 |
| av_packet_free | 释放数据包资源 |
| avcodec_free_context | 释放编解码器上下文资源 |
| avformat_close_input | 关闭输入流并释放封装上下文 |
| avio_context_free | 释放IO上下文 |

## 八、总结

1.用FFmpeg解码，需要先注册/查找需要的解码器、创建相关对象（上下文和数据结构），一步步初始化好<br>
2.解码流程：不断读取一帧压缩数据 -> 送到解码器 -> 取出解码帧 -> 可渲染<br>
3.涉及多线程同步，就要引入帧缓存类（Frames），防止数据冲突<br>
4.每个类、结构体在解码线程内分配，用完及时释放，防止内存泄漏

# 代码实现

### Frames.h

```cpp
#pragma once

#include <QMutex>
#include <QObject>
#include <QWaitCondition>

typedef struct AVFrame AVFrame;

class Frames : public QObject {
  Q_OBJECT

public:
  explicit Frames(QObject *parent = nullptr);
  virtual ~Frames();

public:
  /**
   * @brief 初始化Frames
   *
   * @return true
   * @return false
   */
  bool init();

  /**
   * @brief 清除Frames
   *
   */
  void deInit();

  /**
   * @brief 锁定Frames
   *
   */
  void lock();

  /**
   * @brief 解锁Frames
   *
   */
  void unLock();

  /**
   * @brief 获取解码帧
   *
   * @return AVFrame*
   */
  AVFrame *decodingFrame();

  /**
   * @brief 提供已解码的帧
   * 
   * @return true 
   * @return false 
   */
  bool offerDecodedFrames();

  /**
   * @brief 消费渲染帧
   * 
   * @return const AVFrame*
   */
  const AVFrame *consumeRenderingFrame();

  /**
   * @brief 交换解码帧和渲染帧
   *
   */
  void swap();

  void stop();

private:
  // 解码出来的一帧数据（yuv）
  // 保存正在解码的yuv
  AVFrame *m_decodingFrame = nullptr;
  // 保存正在渲染的yuv
  AVFrame *m_renderingframe = nullptr;
  // 保证AVFrame的多线程安全
  QMutex m_mutex;
  bool m_renderingFrameConsumed = true;
};
```

### Frames.cpp

```cpp
#include "Frames.h"
#include <qassert.h>

extern "C" {
#include "libavformat/avformat.h"
#include "libavutil/avutil.h"
}

Frames::Frames(QObject *parent) : QObject{parent} {
}

bool Frames::init() {
  m_decodingFrame = av_frame_alloc();
  if (!m_decodingFrame) {
    goto error;
  }

  m_renderingframe = av_frame_alloc();
  if (!m_renderingframe) {
    goto error;
  }

error:
  m_renderingFrameConsumed = true;
  return true;
}

Frames::~Frames() {
  deInit();
}

void Frames::deInit() {
  if (m_decodingFrame) {
    av_frame_free(&m_decodingFrame);
    m_decodingFrame = nullptr;
  }
  if (m_renderingframe) {
    av_frame_free(&m_renderingframe);
    m_renderingframe = nullptr;
  }
}

void Frames::lock() {
  m_mutex.lock();
}

void Frames::unLock() {
  m_mutex.unlock();
}

AVFrame* Frames::decodingFrame() {
  return m_decodingFrame;
}

bool Frames::offerDecodedFrames() {
  m_mutex.lock();
  swap();
  bool previousFrameConsumed = m_renderingFrameConsumed;
  m_renderingFrameConsumed = false;
  m_mutex.unlock();
  
  return previousFrameConsumed;
}

const AVFrame* Frames::consumeRenderingFrame() {
  Q_ASSERT(!m_renderingFrameConsumed);
  m_renderingFrameConsumed = true;

  return m_renderingframe;
}

void Frames::swap() {
  AVFrame *temp = m_decodingFrame;
  m_decodingFrame = m_renderingframe;
  m_renderingframe = temp;
}

void Frames::stop() {
}
```

### Decoder.h

```cpp
#pragma once

#include <QMutex>
#include <QPointer>
#include <QThread>
#include <qtmetamacros.h>

extern "C" {
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
}

class Frames;
class DeviceSocket;
class Decoder : public QThread {
  Q_OBJECT

public:
  explicit Decoder(QThread *parent = nullptr);
  virtual ~Decoder();

public:
  /**
   * @brief 初始化解码器
   *
   * @return true
   * @return false
   */
  static bool init();

  /**
   * @brief 清理解码器
   *
   */
  static void deInit();

  /**
   * @brief 保存解码帧
   *
   * @param frames
   */
  void setFrames(Frames *frames);

  /**
   * @brief 设置socket
   *
   * @param deviceSocket
   */
  void setDeviceSocket(DeviceSocket *deviceSocket);

  /**
   * @brief 接收数据
   *
   * @param buf
   * @param bufSize
   * @return qint32
   */
  qint32 recvData(quint8 *buf, qint32 bufSize);

  /**
   * @brief 开始解码
   *
   * @return true
   * @return false
   */
  bool startDecode();

  /**
   * @brief 停止解码
   *
   */
  void stopDecode();

signals:
  /**
   * @brief 新的帧到达
   *
   */
  void onNewFrame();

  /**
   * @brief 解码停止
   *
   */
  void onDecodeStop();

protected:
  /**
   * @brief 运行解码线程
   *
   */
  void run();

  /**
   * @brief 推送解码帧
   *
   */
  void pushFrame();

private:
  QPointer<DeviceSocket> m_deviceSocket; // 接收h264数据
  bool m_quit = false;                   // 退出标记
  Frames *m_frames;                      // 解码出的帧
};
```

### Decoder.cpp

```cpp
#include <qdebug.h>
#include <qlogging.h>
#include <qtypes.h>

#include "Decoder.h"
#include "DeviceSocket.h"
#include "Frames.h"
#include "libavutil/mem.h"

#define BUFSIZE 0x10000

Decoder::Decoder(QThread *parent) : QThread{parent} {
}

Decoder::~Decoder() {
  stopDecode();
}

bool Decoder::init() {
  // 初始化
  if (avformat_network_init()) {
    return false;
  }
  return true;
}

void Decoder::deInit() {
  avformat_network_deinit();
}

void Decoder::setFrames(Frames *frames) {
  m_frames = frames;
}

void Decoder::setDeviceSocket(DeviceSocket *deviceSocket) {
  m_deviceSocket = deviceSocket;
}

// 参数： Decoder对象，解码数据缓存，解码数据缓存大小
static qint32 readPacket(void *opaque, quint8 *buf, qint32 bufSize) {
  Decoder *decoder = (Decoder *)opaque;
  if (decoder) {
    return decoder->recvData(buf, bufSize);
  }
  return 0;
}

qint32 Decoder::recvData(quint8 *buf, qint32 bufSize) {
  if (!buf) {
    return 0;
  }
  if (m_deviceSocket) {
    // 从deviceSocket获取h264数据
    qint32 len = m_deviceSocket->subThreadRecvData(buf, bufSize);
    qDebug() << "recvData len:" << len;
    if (len == -1) {
      return AVERROR(errno);
    }
    if (len == 0) {
      return AVERROR_EOF;
    }
    return len;
  }
  return AVERROR_EOF;
}

bool Decoder::startDecode() {
  if (!m_deviceSocket) {
    return false;
  }

  m_quit = false;
  // 启动解码线程
  start();

  return true;
}

void Decoder::stopDecode() {
  m_quit = true;
  if (m_frames) {
    m_frames->stop();
  }
  // 等待解码线程退出
  wait();
}

void Decoder::run() {
  struct ResourceGuard {
    unsigned char *decoderBuffer = nullptr;
    AVIOContext *avioCtx = nullptr;
    AVFormatContext *formatCtx = nullptr;
    AVCodecContext *codecCtx = nullptr;
    AVPacket *packet = nullptr;
    bool isFormatCtxOpen = false;
    bool isCodecCtxOpen = false;

    ~ResourceGuard() {
      if (isCodecCtxOpen && codecCtx) {
        avcodec_free_context(&codecCtx);
      }
      if (isFormatCtxOpen && formatCtx) {
        avformat_close_input(&formatCtx);
      }
      if (avioCtx) {
        av_freep(&avioCtx->buffer);
        avio_context_free(&avioCtx);
      }
      if (packet) {
        av_packet_free(&packet);
      }
      if (decoderBuffer) {
        av_free(decoderBuffer);
      }
    }
  } resources;

  // 申请解码缓冲区
  resources.decoderBuffer = (unsigned char *)av_malloc(BUFSIZE);
  if (!resources.decoderBuffer) {
    qCritical("Could not allocate decoder buffer");
    return;
  }

  // 初始化io上下文
  resources.avioCtx = avio_alloc_context(resources.decoderBuffer, BUFSIZE, 0,
                                         this, readPacket, NULL, NULL);
  if (!resources.avioCtx) {
    qCritical("Could not allocate avio context");
    av_free(resources.decoderBuffer);
    return;
  }

  // 初始化封装上下文
  resources.formatCtx = avformat_alloc_context();
  if (!resources.formatCtx) {
    qCritical("Could not allocate format context");
    return;
  }

  // 为封装上下文指定io上下文
  resources.formatCtx->pb = resources.avioCtx;
  // 打开封装上下文
  if (avformat_open_input(&resources.formatCtx, NULL, NULL, NULL) < 0) {
    qCritical("Could not open input");
    return;
  }
  resources.isFormatCtxOpen = true;

  // 初始化解码器
  const AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_H264);
  if (!codec) {
    qCritical("Could not find H.264 codec");
    return;
  }

  // 初始化解码器上下文
  resources.codecCtx = avcodec_alloc_context3(codec);
  if (!resources.codecCtx) {
    qCritical("Could not allocate codec context");
    return;
  }

  // 打开解码器上下文
  if (avcodec_open2(resources.codecCtx, codec, NULL) < 0) {
    qCritical("Could not open H.264 codec");
    return;
  }
  resources.isCodecCtxOpen = true;

  // 解码数据包：保存解码前的一帧h264数据
  resources.packet = av_packet_alloc();
  if (!resources.packet) {
    qCritical("Could not allocate packet");
    return;
  }

  // 初始化解码数据包
  resources.packet->data = nullptr;
  resources.packet->size = 0;

  // 从封装上下文中读取一帧解码前的数据，保存到AVPacket中
  while (!m_quit && !av_read_frame(resources.formatCtx, resources.packet)) {
    // 获取AVFrame用来保存解码出来的yuv数据
    AVFrame *decodingFrame = m_frames->decodingFrame();
    // 解码
    int ret;
    // 解码h264
    if ((ret = avcodec_send_packet(resources.codecCtx, resources.packet)) < 0) {
      qCritical("Could not send video packet: %d", ret);
      return;
    }
    // 取出yuv
    if (decodingFrame) {
      ret = avcodec_receive_frame(resources.codecCtx, decodingFrame);
    }
    if (!ret) {
      // 成功解码出一帧
      pushFrame();
    } else if (ret != AVERROR(EAGAIN)) {
      qCritical("Could not receive video frame: %d", ret);
      av_packet_unref(resources.packet);
      return;
    }

    av_packet_unref(resources.packet);

    if (resources.avioCtx->eof_reached) {
      break;
    }
  }
  qDebug() << "End of frames";

  emit onDecodeStop();
}

void Decoder::pushFrame() {
  bool previousFrameConsumed = m_frames->offerDecodedFrames();
  if (!previousFrameConsumed) {
    return;
  }

  emit onNewFrame();
}
```