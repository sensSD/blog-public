day2的博客不小心被覆盖了，辛辛苦苦写的消失了还是很心疼，但是没事，这篇博客的内容才是重点。

# AdbProcess.h

## push()

~~~ c++
	/**
     * 上传文件
     * @brief push
     * @param serial 设备号
     * @param local 本地文件路径
     * @param remote 上传到连接设备的路径
     */
    void push(const QString& serial, const QString& local, const QString& remote);
~~~

## removePath()

~~~ c++
	/**
     * 删除文件
     * @brief removePath
     * @param serial 设备号
     * @param remote 要删除的连接设备的路径
     */
    void removePath(const QString& serial, const QString& remote);
~~~

## reverse()

~~~ c++
    /**
     * 端口映射
     * @brief reverse
     * @param serial 设备号
     * @param deviceSocketName 设备套接字名称
     * @param localPort 电脑端映射的设备端口号
     */
    void reverse(const QString& serial, const QString& deviceSocketName, quint16 localPort);
~~~

## removeReverse()

~~~ c++
    /**
     * 删除端口映射
     * @brief removeReverse
     * @param serial 设备号
     * @param deviceSocketName 设备套接字名称
     */
    void removeReverse(const QString& serial, const QString& deviceSocketName);
~~~

## getDevicesSerialFromStdOut()

~~~ c++
    /**
     * 获取设备列表
     * @brief getDevicesSerialFromStdOut
     * @return QStringList
     */
    QStringList getDevicesSerialFromStdOut();
~~~

## getDeviceIpFromStdOut()

~~~ c++
    /**
     * 获取设备IP
     * @brief getDeviceIpFromStdOut
     * @return QString
     */
    QString getDeviceIpFromStdOut();
~~~

还有两个获取标准输出和错误的，没什么技术点，就不写了。

# AdbProcess.cpp

## push()

封装的adb命令：```adb <本地文件/文件夹路径> <设备目标路径>```

~~~ c++
void AdbProcess::push(const QString &serial, const QString &local, const QString &remote)
{
    QStringList args;
    args << "push"
         << local 
         << remote;

    execute(serial, args);
}
~~~

## removePath()

封装的adb命令：```adb shell rm <remote>```

~~~ c++
void AdbProcess::removePath(const QString &serial, const QString &remote)
{
    QStringList args;
    args << "shell" 
         << "rm" 
         << remote;

    execute(serial, args);
}
~~~

## reverse()

封装的adb命令：```adb reverse localabstract:<deviceSocketName> tcp:<localPort>```

~~~ c++
void AdbProcess::reverse(const QString &serial, const QString &deviceSocketName, quint16 localPort)
{
    QStringList args;
    args << "reverse" 
         << QString("localabstract:%1").arg(deviceSocketName) 
         << QString("tcp:%1").arg(localPort);

    execute(serial, args);
}
~~~

## removeReverse()

封装的adb命令：```adb reverse --remove localabstract:<deviceSocketName>```

~~~ c++
void AdbProcess::removeReverse(const QString &serial, const QString &deviceSocketName)
{
    QStringList args;
    args << "reverse" 
         << "--remove" 
         << QString("localabstract:%1").arg(deviceSocketName);

    execute(serial, args);
}
~~~

## getDevicesSerialFromStdOut()

主要技术点：QRegularExpression()，qt6的正则表达式函数

~~~ c++
QStringList AdbProcess::getDevicesSerialFromStdOut()
{
    QStringList serials;
    QStringList devicesInfoList = m_standardOutput.split(QRegularExpression("\r\n|\n"), Qt::SkipEmptyParts);
    for(const auto& deviceInfo : devicesInfoList) {
        QStringList deviceInfos = deviceInfo.split(QRegularExpression("\t"), Qt::SkipEmptyParts);
        if(2 == deviceInfos.size() && 0 == deviceInfos[1].compare("device")) {
            serials << deviceInfos[0];
        }
    }

    return serials;
}
~~~

## getDeviceIpFromStdOut()

~~~ c++
QString AdbProcess::getDeviceIpFromStdOut()
{
    QRegularExpression ipRegex(R"(inet\s+(\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}))");
    QRegularExpressionMatch match = ipRegex.match(m_standardOutput);
    
    if (match.hasMatch()) {
        return match.captured(1);
    }

    return "";
}
~~~

- 需要学习qt的正则表达式及相关函数

写完发现其实技术点也不多，大多是重复内容。正则表达式一直没能记下来，今天写完这部分代码觉得不得不学了，知识真是越学越多啊...

最后写点感想，从qtcreator转战vscode后，在ai辅助下效率提升太多了，报错也能通过它快速解决问题，就连正则都是问ai写的。爽快的同时也感叹在现在ai有多重要，希望以后我能成为那个真正使用ai，不被它淘汰的人吧。