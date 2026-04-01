# adb介绍
Android 调试桥(adb)是一个通用命令行工具，其允许您与模拟器实例或连接的Android 设备进行通信。它可为各种设备操作提供便利，如安装和调试应用，并提供对Unixshell（可用来在模拟器或连接的设备上运行各种命令）的访问。该工具作为一个客户端-服务器程序，包括三个组件：
- adb客户端，该组件发送命令。adb客户端在开发计算机上运行。我们可以通过从命令行运行adb客户端来发送命令。
- adb服务器，该组件管理客户端和后台程序之间的通信。adb服务器在开发计算机上作为后台进程运行。
- adbd服务器，该组件在设备上运行命令。adbd服务器在每个模拟器或设备实例上作为后台进程运行。

以下是下载地址：
- [windows](https://dl.google.com/android/repository/platform-tools-latest-windows.zip)
- [linux](https://dl.google.com/android/repository/platform-tools-latest-linux.zip)
- [mac](https://dl.google.com/android/repository/platform-tools-latest-windows.zip)
---
本次需要的文件有以下三个
![](/blogs/QtScrcpy/068b905c88482f9c.png)
# adb的工作方式
启动一个adb客户端时，此客户端首先检查是否有已运行的adb服务器进程。如果没有，它将启动服务器进程。当服务器启动时，它与本地TCP端口5037绑定，并侦听从adb客户端发送的命令一所有adb客户端均使用端口5037与adb服务器通信。
### 连接方式1：通过socket连接，adbd监听5555端口，adb接收
### 连接方式2：USB连接
---
基本用法
---
### 命令语法
adb基本命令如下  
~~~powershell
adb [-d|-e|-s <serialNumber>]<command>
~~~
如果只有一个设备/模拟器连接时，可以省略掉[-d-e|-s<serialNumber>]这一部分，直接使用adb <command>。

---
### 为命令指定目标设备
如果有多个设备/模拟器连接，则需要为命令指定目标设备。
参数|含义
--|:--
-d|指定当前唯一通过 USB 连接的 Android 设备为命令目标
-e|指定当前唯一运行的模拟器为命令目标
-s <serialNumber>|指定相应 serialNumber 号的设备/模拟器为命令目标
在多个设备/模拟器连接的情况下较常用的是 -s <serialNumber> 参数，serialNumber 可以通过 adb devices 命令获取。如：
~~~powershell
$ adb devices

List of devices attached
cf264b8f	device
emulator-5554	device
10.129.164.6:5555	device
~~~
输出里的 cf264b8f、emulator-5554 和 10.129.164.6:5555 即为 serialNumber。

比如这时想指定 cf264b8f 这个设备来运行 adb 命令获取屏幕分辨率：

```adb -s cf264b8f shell wm size```
adb常用命令
---
- 查看已连接的模拟器/设备的列表
    
    `adb devices`
- 将命令发送至特定设备
 
    `adb -s serial_number command `

    如果在多个设备可用时您未指定目标模拟器/设备实例就发出命令，那么 adb 将生成一个错误。

- 安装应用

    `adb install path_to_apk`

- 设置端口转发

    使用 reserve 命令设置任意端口转发 — 将对模拟器/设备实例上特定端口的请求转发到主机的其他端口。下面向您介绍如何设置模拟器/设备端口 6100 到主机端口 7100 的转发：
    
    `adb reserve tcp:6100 tcp:7100`
    
    也可以使用 adb 设置传输到指定的UNIX域套接字的转发，如下所示：
    
    `adb reserve localabstract:logd tcp:7100`
    
    [unix域套接字介绍](https://www.cnblogs.com/pengdonglin137/p/3849012.html)
    
- 将文件复制到设备/从设备复制文件
 
   使用 adb 命令 pull 和 push 将文件复制到模拟器/设备实例或从其中复制文件。与 install 命令不同（其仅将 APK 文件复制到特定位置），pull 和 push 命令允许您将任意目录和文件复制到模拟器/设备实例中的任意位置。

    要从模拟器或设备复制文件或目录（及其子目录），使用
    
    `adb pull remote local`
    
    要将文件文件或目录（及其子目录）复制到模拟器或设备，使用
    
    `adb push local remote`
    
    在上述命令中，local 和 remote 指的是开发计算机（本地）和模拟器/设备实例（远程）上目标文件/目录的路径。例如：

    `adb push foo.txt /sdcard/foo.txt`
    
- 停止 adb 服务器

	`adb kill-server`

- 发出 shell 命令

	`adb [-d|-e|-s serial_number] shell shell_command`

	或者，使用如下命令进入模拟器/设备实例上的远程 shell：

	`adb [-d|-e|-s serial_number] shell`

	退出使用 CTRL + D 或输入exit

通过 WLAN 连接到设备
---
一般情况下，通过 USB 使用 adb。不过，也可以按照下面的说明通过 WLAN 使用它。

- 将 Android 设备和 adb 主计算机连接到这两者都可以访问的常用 WLAN 网络。请注意，并非所有访问点均适用；您可能需要使用已正确配置防火墙的访问点以支持 adb 的访问点。

- 使用 USB 电缆将设备连接到主计算机。
- 设置目标设备以侦听端口 5555 上的 TCP/IP 连接。

    `$ adb tcpip 5555`

- 从目标设备断开 USB 电缆连接。
- 查找 Android 设备的 IP 地址。例如，在 Nexus 设备上，您可以通过访问 Settings > About tablet（或 About phone) > Status > IP address 查找 IP 地址。或者，在 Android Wear 设备上，您可以通过访问 Settings > Wi-Fi Settings > Advanced > IP address 查找 IP 地址。
- 连接至设备，通过 IP 地址识别此设备。

    `$ adb connect device_ip_address`

- 请确认您的主计算机已连接至目标设备：

    `$ adb devices`

    List of devices attached

    device_ip_address:5555 device

