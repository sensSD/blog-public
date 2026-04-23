# 前情提要

&emsp;在开始编写代码前，需要先了解什么是YUV，以及怎么通过opengl将其渲染到电脑上。

## 什么是YUV

&emsp;YUV是指亮度参量和色度参量分开表示的像素格式，其中**Y表示明亮度**（Luminance或Luma），也就是灰度值；**而“U”和“V”表示的则是色度**（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。

总结就是：

- Y: 表示明亮度
- U和V: 色度

&emsp;从安卓设备上返回的为YUV420格式，对于常见的格式有：

- 4:4:4 表示完全取样
- 4:2:2 表示2:1的水平取样，垂直完全采样
- 4:2:0 表示2:1的水平取样，垂直2：1采样
- 4:1:1 表示4:1的水平取样，垂直完全采样

大多数视频采用的420格式，因为人眼对色度相对不敏感，所以通过降低色度采样率来减少数据量，同时保持视觉质量

## opengl基础

### 为什么选择OpenGL而不是CPU转换？

- CPU转换YUV→RGB的问题：
  - 对于1080p视频，每秒需要处理30帧，CPU计算量巨大
  - 占用CPU资源，导致投屏延迟增加
  
- OpenGL GPU加速的优势：
  - 着色器并行处理每个像素的色彩转换
  - CPU只需准备数据，GPU完成转换+显示，延迟极低
  - QtScrcpy项目中正是因为要实现"超低延迟"投屏才采用这个方案

### OpenGL的三个核心概念

#### 1. VBO（顶点缓冲对象）
- 作用：存储顶点数据（位置、纹理坐标等）到GPU显存
- 优点：一次上传数据到GPU，可多次使用，比每次传输CPU内存快得多
- 代码中的体现：`m_vbo`存储了矩形的4个顶点坐标和对应的纹理坐标

#### 2. 着色器（Shader）
- **顶点着色器**：处理每个顶点，处理坐标变换
- **片段着色器**：处理每个像素，这里正是YUV→RGB转换的地方
- 代码中：`s_vertShader` 和 `s_fragShader`

#### 3. 纹理（Texture）
- 作用：存储图像数据到GPU
- 在这个项目中：用3个纹理分别存储Y、U、V三个分量
- 好处：GPU可以并行采样三个纹理，效率高

### OpenGL渲染流程

准备阶段（initializeGL）：
&emsp;<b>1.</b> 初始化OpenGL函数指针
&emsp;<b>2.</b> 创建VBO，上传顶点数据
&emsp;<b>3.</b> 编译着色器，链接程序
&emsp;<b>4.</b> 设置初始状态

每帧渲染阶段（paintGL）：
&emsp;<b>1.</b> 绑定着色器程序
&emsp;<b>2.</b> 检查帧尺寸是否改变
&emsp;<b>3.</b> 若改变，则重新创建纹理
&emsp;<b>4.</b> 绑定三个纹理到纹理单元
&emsp;<b>5.</b> 执行渲染命令
&emsp;<b>6.</b> 释放着色器程序

结果：GPU执行着色器→YUV→RGB转换→输出到屏幕

# 代码逻辑

## 数据结构设计

### 坐标数据

```cpp
static const GLfloat coordinate[] = {
    // 顶点坐标（xyz），范围[-1, 1]，中心为(0,0)
    -1.0f, -1.0f, 0.0f,  // 左下
     1.0f, -1.0f, 0.0f,  // 右下
    -1.0f,  1.0f, 0.0f,  // 左上
     1.0f,  1.0f, 0.0f,  // 右上

    // 纹理坐标（uv），范围[0, 1]
    0.0f, 1.0f,   // 左下
    1.0f, 1.0f,   // 右下
    0.0f, 0.0f,   // 左上
    1.0f, 0.0f    // 右上
};
```

### 着色器程序

#### 顶点着色器

```cpp
// 顶点着色器源码
static const QString s_vertShader = R"(
  attribute vec3 vertexIn;    // xyz顶点坐标
  attribute vec2 textureIn;   // xy纹理坐标

  varying vec2 textureOut;    // 传递给片段着色器的纹理坐标

  void main(void)
  {
    gl_Position = vec4(vertexIn, 1.0);
    textureOut = textureIn;
  }
)";
```

- 因为做的是显示全屏2D图像，不需要做转换

#### 片段着色器

```cpp
// 片段着色器源码
static QString s_fragShader = R"(
  varying vec2 textureOut;        // 由顶点着色器传递过来的纹理坐标

  uniform sampler2D textureY;     // uniform 纹理单元，利用纹理单元可以使用多个纹理
  uniform sampler2D textureU;     // sampler2D是2D采样器
  uniform sampler2D textureV;     // 声明yuv三个纹理单元

  void main(void)
  {
    vec3 yuv;
    vec3 rgb;

    // SDL2 BT.709色彩标准定义的转换矩阵
    const vec3 Rcoeff = vec3(1.1644,  0.000,  1.7927);
    const vec3 Gcoeff = vec3(1.1644, -0.2132, -0.5329);
    const vec3 Bcoeff = vec3(1.1644,  2.1124,  0.000);

    // 根据指定的纹理textureY和坐标textureOut来采样
    yuv.x = texture2D(textureY, textureOut).r;
    yuv.y = texture2D(textureU, textureOut).r - 0.5;
    yuv.z = texture2D(textureV, textureOut).r - 0.5;

    // 采样完转为rgb
    // 减少一些亮度
    yuv.x = yuv.x - 0.0625;
    rgb.r = dot(yuv, Rcoeff);
    rgb.g = dot(yuv, Gcoeff);
    rgb.b = dot(yuv, Bcoeff);
    // 输出颜色值
    gl_FragColor = vec4(rgb, 1.0);
  }
)";
```

#### 成员变量

| 变量 | 作用 | 为什么需要 |
| :--- | :--- | :--- |
| `m_frameSize` | 存储视频帧分辨率 | 创建纹理时需要知道尺寸 |
| `m_needUpdate` | 标记是否需要重建纹理 | 当分辨率改变时，旧纹理失效 |
| `m_textureInitialized` | 标记纹理是否已初始化 | 避免未初始化时就尝试使用 |
| `m_vbo` | 顶点缓冲对象 | 存储顶点和纹理坐标到GPU |
| `m_shaderProgram` | 着色器程序 | 每次渲染都要使用它 |
| `m_texture[3]` | Y、U、V三个纹理ID | 分别存储三个分量数据 |


---

## 初始化流程（initializeGL）

这个函数在窗口第一次显示时自动调用，只调用一次。

### 第1步：初始化OpenGL函数指针

`initializeOpenGLFunctions();`

- 必须在任何其他GL操作前调用

### 第2步：禁用深度测试

`glDisable(GL_DEPTH_TEST);`

- 我们显示的是2D图像，不需要深度比较

### 第3步：创建并配置VBO

```cpp
m_vbo.create();
m_vbo.bind();
m_vbo.allocate(coordinate, sizeof(coordinate));
```

&emsp;<b>1.</b> `create()` → 在GPU显存中创建一块缓冲区
&emsp;<b>2.</b> `bind()` → 将这块缓冲区设为当前操作目标
&emsp;<b>3.</b> `allocate()` → 将coordinate数组从CPU内存复制到GPU显存

### 第4步：初始化着色器

`initShader();`

这个函数做的事情较多，后面详细讲解。

### 第5步：设置背景色

```cpp
glClearColor(0.0, 0.0, 0.0, 0.0);  
glClear(GL_COLOR_BUFFER_BIT);   
```

- 效果：初始背景为黑色，给视频播放做好准备

---

## 着色器初始化（initShader）

### 第1步：处理OpenGL ES兼容性

```cpp
if (QCoreApplication::testAttribute(Qt::AA_UseOpenGLES)) {
    s_fragShader.prepend(R"(
        precision mediump int;
        precision mediump float;
    )");
}
```

- OpenGLES（移动设备用）需要明确指定浮点精度
- mediump → 中等精度，平衡性能和精度

### 第2步：编译着色器

```cpp
m_shaderProgram.addShaderFromSourceCode(QOpenGLShader::Vertex, s_vertShader);
m_shaderProgram.addShaderFromSourceCode(QOpenGLShader::Fragment, s_fragShader);
```

- 第一行：编译顶点着色器
- 第二行：编译片段着色器

### 第3步：链接着色器程序

```cpp
m_shaderProgram.link();
m_shaderProgram.bind();
```

- `link()` → 将两个着色器合并成一个可执行程序
- `bind()` → 设为当前使用的着色器程序

### 第4步：配置顶点属性指针

```cpp
// 告诉GPU：vertexIn属性从VBO的哪里读取数据
m_shaderProgram.setAttributeBuffer("vertexIn", 
    GL_FLOAT,           // 数据类型为float
    0,                  // 起始偏移：0字节
    3,                  // 每个顶点有3个float（xyz）
    3 * sizeof(float)   // 步幅：相邻两个顶点间隔12字节
);
m_shaderProgram.enableAttributeArray("vertexIn");  // 启用这个属性
```

### 第5步：配置纹理坐标指针

```cpp
m_shaderProgram.setAttributeBuffer("textureIn",
    GL_FLOAT,
    12 * sizeof(float),  // 起始偏移：纹理坐标在VBO的第49字节处
    2,                   // 每个纹理坐标有2个float（uv）
    2 * sizeof(float)    // 步幅：相邻纹理坐标间隔8字节
);
m_shaderProgram.enableAttributeArray("textureIn");
```

### 第6步：绑定纹理单元

```cpp
m_shaderProgram.setUniformValue("textureY", 0);
m_shaderProgram.setUniformValue("textureU", 1);
m_shaderProgram.setUniformValue("textureV", 2);
```

- 作用：告诉着色器中的uniform变量应该从哪个纹理单元读取数据

## 纹理管理

### 为什么用3个纹理而不是1个？

**选项1：用1个纹理存储所有YUV数据**
- 缺点：要在着色器中手动计算每个分量的位置
- 缺点：UV分量的采样率不同（Y是W×H，U/V是W/2×H/2），难以处理

**选项2：用3个纹理分别存储Y、U、V**（本项目采用）
- 优点：结构清晰，每个纹理一致性强
- 优点：GPU可以并行采样三个纹理
- 优点：着色器代码简洁

### 创建纹理（initTextures）

#### Y分量纹理（完整分辨率）

```cpp
glGenTextures(1, &m_texture[0]);         // 申请1个纹理ID
glBindTexture(GL_TEXTURE_2D, m_texture[0]);  // 将其设为当前操作目标

// 设置缩放过滤策略
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

// 设置边界处理
glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

// 分配显存空间（尺寸为frameSize）
glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE, 
    m_frameSize.width(), m_frameSize.height(),   // 尺寸
    0, GL_LUMINANCE, GL_UNSIGNED_BYTE, nullptr); // GL_LUMINANCE只使用一个通道（灰度）
```

- <b>参数详解</b>

| 参数 | 含义 |
| :--- | :--- |
| `GL_TEXTURE_MIN_FILTER = GL_LINEAR` | 纹理缩小时用线性插值（平滑） |
| `GL_TEXTURE_MAG_FILTER = GL_LINEAR` | 纹理放大时用线性插值（平滑） |
| `GL_CLAMP_TO_EDGE` | 超出纹理边界时重复边界像素 |
| `GL_LUMINANCE` | 只使用一个通道，适合灰度数据 |
| `GL_UNSIGNED_BYTE` | 像素值范围[0, 255] |

#### U/V分量纹理（1/4分辨率）

```cpp
// 完全相同的配置，只是尺寸减半
glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE,
    m_frameSize.width() / 2, m_frameSize.height() / 2,  // ← 关键：减半
    0, GL_LUMINANCE, GL_UNSIGNED_BYTE, nullptr);
```

#### 销毁纹理（deInitTextures）

```cpp
void QYUVOpenGLWidget::deInitTextures() {
    if (QOpenGLFunctions::isInitialized(QOpenGLFunctions::d_ptr)) {
        glDeleteTextures(3, m_texture);  // 删除GPU中的纹理，释放显存
    }
    memset(m_texture, 0, sizeof(m_texture));  // 清空数组
    m_textureInitialized = false;              // 标记为未初始化
}
```

#### 更新纹理数据（updateTexture）

这是性能的关键！

```cpp
void QYUVOpenGLWidget::updateTexture(GLuint texture, quint32 textureType, 
                                     quint8* pixels, quint32 stride) {
    if (!pixels) return;  // 数据指针为空，直接返回
    
    // 根据纹理类型计算尺寸
    QSize size = 0 == textureType ? m_frameSize : m_frameSize / 2;

    makeCurrent();                    // 切换到正确的GL上下文
    glBindTexture(GL_TEXTURE_2D, texture);  // 绑定要更新的纹理
    
    // 这一行是关键！设置像素数据的行步幅
    glPixelStorei(GL_UNPACK_ROW_LENGTH, static_cast<GLint>(stride));
    
    // 增量更新纹理（只修改部分内容，而不是全量重传）
    glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, 
        size.width(), size.height(),
        GL_LUMINANCE, GL_UNSIGNED_BYTE, pixels);
        
    doneCurrent();  // 释放GL上下文
}
```

为什么这样设计效率高？
  <b>1.</b> 增量更新而不是全量重建

  - `glTexImage2D` → 重新分配显存并传输所有数据 → 慢
  - `glTexSubImage2D` → 只更新部分数据 → 快

  <b>2.</b> GL_UNPACK_ROW_LENGTH的作用

  - 安卓视频数据可能有行填充（stride可能 > width）
  - 设置stride后，GPU自动跳过填充字节，只读取有效数据

#### 每帧渲染（paintGL）

```cpp
void QYUVOpenGLWidget::paintGL() {
    m_shaderProgram.bind();  // 使用着色器程序
    
    // 检查帧尺寸是否改变
    if (m_needUpdate) {
        deInitTextures();      // 删除旧纹理
        initTextures();        // 创建新纹理
        m_needUpdate = false;  // 复位标志
    }
    
    if (m_textureInitialized) {
        // 绑定三个纹理到对应的纹理单元
        glActiveTexture(GL_TEXTURE0);
        glBindTexture(GL_TEXTURE_2D, m_texture[0]);  // Y
        
        glActiveTexture(GL_TEXTURE1);
        glBindTexture(GL_TEXTURE_2D, m_texture[1]);  // U
        
        glActiveTexture(GL_TEXTURE2);
        glBindTexture(GL_TEXTURE_2D, m_texture[2]);  // V
        
        // 绘制矩形（2个三角形，共4个顶点）
        glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
    }
    
    m_shaderProgram.release();  // 释放着色器程序
}
```

执行过程：

```markdown
1. GPU执行顶点着色器
   ├─ 对4个顶点分别执行
   └─ 输出：齐次坐标 + 纹理坐标

2. 光栅化（Rasterization）
   ├─ 将2个三角形转换为像素
   └─ 为每个像素计算纹理坐标插值

3. GPU执行片段着色器
   ├─ 对每个像素执行
   ├─ 采样YUV数据
   ├─ 进行YUV→RGB转换
   └─ 输出RGB颜色

4. 混合和显示
   └─ 将RGB像素写入帧缓冲
```

## 流程图

[![看不清可以点开这个](https://mermaid.ink/img/pako:eNptlFtP2zAYhv9K5OsWBGnWw8UmSqEcWjoYMDGXi2yxSjSaViHVgFKpDA3KOJVxGBtjA6ZRNE1tQQyVQ_drsJP-i7l2qsK0XDix_bzv5--z4zR4lVAQ8IGYLicnhOFAVBPo0wFxbh_fXA-OjYxGkkgLhp6rSgwZ44LT-VjwQ1VTDVWeVGdRMDTOJX421UmFB7iwgld3uc5cL-HjBZvpZEzANq8dXZpvr8zbLbx4jktX1tnRqD9ikwFGdkH8_tCqVjmJP6ya11dmfhEf5MnhEtkpk7UizpWbqi6m6obm7a5V2qxt_SHrP8yDrLV8jj-dmqcr-HrDJrsZGYRW8Y9ZLf7XH599JdmCzfM2yFQ9UEdTLHXBKVjlBbJzYRUW8cZ3m-1hVC9MyqpmMMi82cO5S1x56NbLuL40D0r2Sjh_UvuStU7myf4F2S0_yXCur87N0fk5oR8qqJfWfhhNGym6Cupd286S0jz5WOA2doR-5h1iG3UP5oWn3g_gEIPD0LzZxMXPY60jraN3leW7yk-7HGs7-F1jC_li6ErnhPD9ZMLMYwCmkopsoHsheS52jmzLbKcBpohAWkFz-zQ2GdDlNx26Ls9M8XLZWIRhTyEurpJcnlTOybctOsUnp1Iv-cGNgn_2OQo4UX8GoX3UGgg_xkPQXF4ixd_N4aZkiBHPID3_VvXXUNBPW7J2bBNIU4CD_jKqAnyGnkIOEEd6XK53QbqORIExgeIoCnz0U5H11_XlZKgmKWsvEol4Q6YnUrGJRoeXLqDKNKUmQWMhvTOR0gzgk5gB8KXBNPA52zxSi7vdLbrcoiS629sktwPMsHGxhY556NvzyCO2eV0ZB5hlUdtbRK9X8kpu0e2SqFZyOQBSVCOhh_k1wG6DzF8kaboY?type=png)](https://mermaid-live.nodejs.cn/edit#pako:eNptlFtP2zAYhv9K5OsWBGnWw8UmSqEcWjoYMDGXi2yxSjSaViHVgFKpDA3KOJVxGBtjA6ZRNE1tQQyVQ_drsJP-i7l2qsK0XDix_bzv5--z4zR4lVAQ8IGYLicnhOFAVBPo0wFxbh_fXA-OjYxGkkgLhp6rSgwZ44LT-VjwQ1VTDVWeVGdRMDTOJX421UmFB7iwgld3uc5cL-HjBZvpZEzANq8dXZpvr8zbLbx4jktX1tnRqD9ikwFGdkH8_tCqVjmJP6ya11dmfhEf5MnhEtkpk7UizpWbqi6m6obm7a5V2qxt_SHrP8yDrLV8jj-dmqcr-HrDJrsZGYRW8Y9ZLf7XH599JdmCzfM2yFQ9UEdTLHXBKVjlBbJzYRUW8cZ3m-1hVC9MyqpmMMi82cO5S1x56NbLuL40D0r2Sjh_UvuStU7myf4F2S0_yXCur87N0fk5oR8qqJfWfhhNGym6Cupd286S0jz5WOA2doR-5h1iG3UP5oWn3g_gEIPD0LzZxMXPY60jraN3leW7yk-7HGs7-F1jC_li6ErnhPD9ZMLMYwCmkopsoHsheS52jmzLbKcBpohAWkFz-zQ2GdDlNx26Ls9M8XLZWIRhTyEurpJcnlTOybctOsUnp1Iv-cGNgn_2OQo4UX8GoX3UGgg_xkPQXF4ixd_N4aZkiBHPID3_VvXXUNBPW7J2bBNIU4CD_jKqAnyGnkIOEEd6XK53QbqORIExgeIoCnz0U5H11_XlZKgmKWsvEol4Q6YnUrGJRoeXLqDKNKUmQWMhvTOR0gzgk5gB8KXBNPA52zxSi7vdLbrcoiS629sktwPMsHGxhY556NvzyCO2eV0ZB5hlUdtbRK9X8kpu0e2SqFZyOQBSVCOhh_k1wG6DzF8kaboY)

# 源代码

## QYUVOpenGLWidget.h

```cpp
#pragma once

#include <QMutex>
#include <QPointer>
#include <QThread>

extern "C" {
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
}

class Frames;
class DeviceSocket;
class Decoder : public QThread {
  Q_OBJECT

 public:
  explicit Decoder(QThread* parent = nullptr);
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
  void setFrames(Frames* frames);

  /**
   * @brief 设置socket
   *
   * @param deviceSocket
   */
  void setDeviceSocket(DeviceSocket* deviceSocket);

  /**
   * @brief 接收数据
   *
   * @param buf
   * @param bufSize
   * @return qint32
   */
  qint32 recvData(quint8* buf, qint32 bufSize);

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
  QPointer<DeviceSocket> m_deviceSocket;  // 接收h264数据
  bool m_quit = false;                    // 退出标记
  Frames* m_frames;                       // 解码出的帧
};
```

## QYUVOpenGLWidget.cpp

```cpp
#include "QYUVOpenGLWidget.h"

#include <QCoreApplication>
#include <QDebug>
#include <QOpenGLTexture>
#include <QSurfaceFormat>

// 顶点坐标和纹理坐标数据
static const GLfloat coordinate[] = {
    // 顶点坐标，存储4个xyz坐标
    // 坐标范围为[-1,1],中心点为 0,0
    // 二维图像z始终为0
    // GL_TRIANGLE_STRIP的绘制方式：
    // 使用前3个坐标绘制一个三角形，使用后三个坐标绘制一个三角形，正好为一个矩形
    // x     y     z
    -1.0f, -1.0f, 0.0f, 1.0f, -1.0f, 0.0f, -1.0f, 1.0f, 0.0f, 1.0f, 1.0f, 0.0f,

    // 纹理坐标，存储4个xy坐标
    // 坐标范围为[0,1],左下角为 0,0
    0.0f, 1.0f, 1.0f, 1.0f, 0.0f, 0.0f, 1.0f, 0.0f};

// 顶点着色器源码
static const QString s_vertShader = R"(
  attribute vec3 vertexIn;    // xyz顶点坐标
  attribute vec2 textureIn;   // xy纹理坐标

  varying vec2 textureOut;    // 传递给片段着色器的纹理坐标

  void main(void)
  {
    gl_Position = vec4(vertexIn, 1.0);
    textureOut = textureIn;
  }
)";

// 片段着色器源码
static QString s_fragShader = R"(
  varying vec2 textureOut;        // 由顶点着色器传递过来的纹理坐标

  uniform sampler2D textureY;     // uniform 纹理单元，利用纹理单元可以使用多个纹理
  uniform sampler2D textureU;     // sampler2D是2D采样器
  uniform sampler2D textureV;     // 声明yuv三个纹理单元

  void main(void)
  {
    vec3 yuv;
    vec3 rgb;

    // SDL2 BT709_SHADER_CONSTANTS
    const vec3 Rcoeff = vec3(1.1644,  0.000,  1.7927);
    const vec3 Gcoeff = vec3(1.1644, -0.2132, -0.5329);
    const vec3 Bcoeff = vec3(1.1644,  2.1124,  0.000);

    // 根据指定的纹理textureY和坐标textureOut来采样
    yuv.x = texture2D(textureY, textureOut).r;
    yuv.y = texture2D(textureU, textureOut).r - 0.5;
    yuv.z = texture2D(textureV, textureOut).r - 0.5;

    // 采样完转为rgb
    // 减少一些亮度
    yuv.x = yuv.x - 0.0625;
    rgb.r = dot(yuv, Rcoeff);
    rgb.g = dot(yuv, Gcoeff);
    rgb.b = dot(yuv, Bcoeff);
    // 输出颜色值
    gl_FragColor = vec4(rgb, 1.0);
  }
)";

QYUVOpenGLWidget::QYUVOpenGLWidget(QWidget* parent) : QOpenGLWidget(parent) {
}

QYUVOpenGLWidget::~QYUVOpenGLWidget() {
  makeCurrent();
  m_vbo.destroy();
  deInitTextures();
  doneCurrent();
}

QSize QYUVOpenGLWidget::minimumSizeHint() const {
  return QSize(50, 50);
}

QSize QYUVOpenGLWidget::sizeHint() const {
  return size();
}

void QYUVOpenGLWidget::setFrameSize(const QSize& size) {
  if (size != m_frameSize) {
    m_frameSize = size;
    m_needUpdate = true;
    repaint();  // 触发重绘
  }
}

const QSize& QYUVOpenGLWidget::frameSize() const {
  return m_frameSize;
}

void QYUVOpenGLWidget::updateTextures(quint8* dataY, quint8* dataU, quint8* dataV,
                                      quint32 lineSizeY, quint32 lineSizeU, quint32 lineSizeV) {
  if (m_textureInitialized) {
    updateTexture(m_texture[0], 0, dataY, lineSizeY);
    updateTexture(m_texture[1], 1, dataU, lineSizeU);
    updateTexture(m_texture[2], 2, dataV, lineSizeV);
    update();  // 更新ui
  }
}

void QYUVOpenGLWidget::initializeGL() {
  initializeOpenGLFunctions();

  // 关闭深度测试
  glDisable(GL_DEPTH_TEST);

  // 创建并绑定顶点缓冲对象
  m_vbo.create();
  m_vbo.bind();
  // 顶点数组复制到缓冲对象中
  m_vbo.allocate(coordinate, sizeof(coordinate));

  // 初始化着色器
  initShader();

  // 设置背景清理色为黑色
  glClearColor(0.0, 0.0, 0.0, 0.0);
  // 清理颜色背景
  glClear(GL_COLOR_BUFFER_BIT);
}

void QYUVOpenGLWidget::resizeGL(int w, int h) {
  glViewport(0, 0, w, h);
  repaint();
}

void QYUVOpenGLWidget::paintGL() {
  m_shaderProgram.bind();

  if (m_needUpdate) {
    deInitTextures();
    initTextures();
    m_needUpdate = false;
  }

  if (m_textureInitialized) {
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_2D, m_texture[0]);

    glActiveTexture(GL_TEXTURE1);
    glBindTexture(GL_TEXTURE_2D, m_texture[1]);

    glActiveTexture(GL_TEXTURE2);
    glBindTexture(GL_TEXTURE_2D, m_texture[2]);

    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);
  }

  m_shaderProgram.release();
}

void QYUVOpenGLWidget::initShader() {
  if (QCoreApplication::testAttribute(Qt::AA_UseOpenGLES)) {
    s_fragShader.prepend(R"(
      precision mediump int;
      precision mediump float;
      )");
  }

  // 编译并链接着色器程序
  m_shaderProgram.addShaderFromSourceCode(QOpenGLShader::Vertex, s_vertShader);
  m_shaderProgram.addShaderFromSourceCode(QOpenGLShader::Fragment, s_fragShader);
  m_shaderProgram.link();
  m_shaderProgram.bind();

  // 指定顶点坐标在vbo中的访问方式
  // 参数：顶点坐标在shader中的参数名称，顶点坐标为float，起始偏移，顶点坐标类型为vec3，步幅
  m_shaderProgram.setAttributeBuffer("vertexIn", GL_FLOAT, 0, 3, 3 * sizeof(float));
  // 启用顶点坐标属性数组
  m_shaderProgram.enableAttributeArray("vertexIn");

  // 指定纹理坐标在vbo中的访问方式
  m_shaderProgram.setAttributeBuffer("textureIn", GL_FLOAT, 12 * sizeof(float), 2,
                                     2 * sizeof(float));
  m_shaderProgram.enableAttributeArray("textureIn");

  // 设置纹理单元与shader中uniform变量的对应关系
  m_shaderProgram.setUniformValue("textureY", 0);
  m_shaderProgram.setUniformValue("textureU", 1);
  m_shaderProgram.setUniformValue("textureV", 2);
}

void QYUVOpenGLWidget::initTextures() {
  // YUV三个分量的大小：Y是全尺寸，U/V是半尺寸
  const QSize sizes[] = {m_frameSize, m_frameSize / 2, m_frameSize / 2};

  for (int i = 0; i < 3; ++i) {
    glGenTextures(1, &m_texture[i]);
    glBindTexture(GL_TEXTURE_2D, m_texture[i]);

    // 设置纹理参数
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameterf(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

    // 创建纹理
    glTexImage2D(GL_TEXTURE_2D, 0, GL_LUMINANCE, sizes[i].width(), sizes[i].height(), 0,
                 GL_LUMINANCE, GL_UNSIGNED_BYTE, nullptr);
  }

  m_textureInitialized = true;
}

void QYUVOpenGLWidget::deInitTextures() {
  if (QOpenGLFunctions::isInitialized(QOpenGLFunctions::d_ptr)) {
    glDeleteTextures(3, m_texture);
  }

  memset(m_texture, 0, sizeof(m_texture));
  m_textureInitialized = false;
}

void QYUVOpenGLWidget::updateTexture(GLuint texture, quint32 textureType, quint8* pixels,
                                     quint32 stride) {
  if (!pixels) return;

  QSize size = 0 == textureType ? m_frameSize : m_frameSize / 2;

  makeCurrent();
  glBindTexture(GL_TEXTURE_2D, texture);
  glPixelStorei(GL_UNPACK_ROW_LENGTH, static_cast<GLint>(stride));
  glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, size.width(), size.height(), GL_LUMINANCE,
                  GL_UNSIGNED_BYTE, pixels);
  doneCurrent();
}
```
