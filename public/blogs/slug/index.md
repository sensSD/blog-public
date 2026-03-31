### 这是我的第一篇博客，也是学习QtScrcpy的第一天，以后的学习轨迹都会留在这个标签下，以后用来借鉴。
---
# 原理
关键点|Scrcpy|QtScrcpy
--|:--:|:--:
界面|sdl|qt
视频解码|ffmpeg|ffmpeg
视频渲染|sdl|opengl
跨平台基础设施|自己封装|Qt提供
编程语言|C|C++
编程方式|同步|异步
控制方式|单点触控|单点/多点触控
编译方式|meson+gradle|Qt Creator
---
QtScrcpy启动流程
----
1. 将scrcpy-server推送到手机（adb push）
2. 启动反向代理（adb reverse）
3. pc端监代理的端口
4. 启动scrcpy-server（adb shell app_process）
5. 接收scrcpy-server的连接，并进行通信
---
# 需要学习的技术栈
----
1. ffmpeg
2. opengl
3. adb