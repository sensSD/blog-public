### QtScrcpy启动流程<br>
	1.将scrcpy-server推送到手机（adb push）<br>
	2.启动反向代理（adb reverse）<br>
	3.pc端监代理的端口<br>
	4.启动scrcpy-server（adb shell app_process）<br>
	5.接收scrcpy-server的连接，并进行通信<br>

首先创建好server文件，编写启动服务器函数

# server.h::startServer

~~~ c++
/**
  * 开始服务器
  * @brief startServer
  * @param serial 设备号
  * @param localPort 端口号
  * @param maxSize 最大分辨率
  * @param bitRate 比特率
  * @return
  */
bool startServer(const QString& serial, quint16 localPort, quint16 maxSize, quint32 bitRate);
~~~

# server.cpp::startServer

初始化需要的参数，然后进入启动阶段

~~~ c++
bool server::startServer(const QString &serial, quint16 localPort, quint16 maxSize, quint32 bitRate)
{
  m_localPort = localPort;
  m_maxSize = maxSize;
  m_bitRate = bitRate;
  m_serial = serial;

  m_serverStartStep = SSS_PUSH;
  return startServerStep(); 
}
~~~

# server.h::startServerStep

~~~ c++
/**
  * 按步骤启动服务器
  * @brief startServerStep
  * @return
  */
bool startServerStep();
~~~

# server.cpp::startServerStep

按启动步骤打开服务

~~~ c++
bool Server::startServerStep()
{
  bool stepSeccess = false;

  switch (m_serverStartStep)
  {
    case SSS_NULL:
      break;
    case SSS_PUSH:
      stepSeccess = pushServer();
      break;
    case SSS_ENABLE_REVERSE:
      stepSeccess = enableReverse();
      break;
    case SSS_EXEC_SERVER:
      // 启动服务器前，需要先监听端口进行通信
      m_serverSocket.setMaxPendingConnections(MAX_CONNECTIONS);
      if(!m_serverSocket.listen(QHostAddress::LocalHost, m_localPort)) {
        qCritical(QString("Could not listen on port %1").arg(m_localPort).toStdString().c_str());
        m_serverStartStep = SSS_NULL;
        // 连接失败，删除代理接口和上传的服务器
        disableReverse();
        removeServer();
        emit serverStartResult(false);
        return false;
      }
      stepSeccess = executeServer();
      break;
    case SSS_RUNNING:
      break;
  }

  if (!stepSeccess) {
    emit serverStartResult(false);
  }

  return stepSeccess;
}
~~~

# server.h::onWorkProcessResult

创建槽函数，当adb执行后触发该函数

~~~ c++
private slots:
    /**
     * 工作进程结果
     * @brief onWorkProcessResult
     * @param processResult
     */
    void onWorkProcessResult(AdbEnums::ADB_EXEC_RESULT processResult);
~~~

# server.cpp::onWorkProcessResult

~~~ c++
void Server::onWorkProcessResult(AdbEnums::ADB_EXEC_RESULT processResult)
{
  if(sender() == &m_workProcess) {
    switch (m_serverStartStep)
    {
      case SSS_NULL:
        break;
      case SSS_PUSH:
        // 成功状态只有AER_SUCCESS_EXEC和AER_SUCCESS_START
        if(AdbEnums::AER_SUCCESS_EXEC == processResult) {
          m_serverStartStep = SSS_ENABLE_REVERSE;
          m_serverCopiedToDevice = true;
          startServerStep();
        } else if (AdbEnums::AER_SUCCESS_START != processResult) {
          qCritical("adb push failed");
          m_serverStartStep = SSS_NULL;
          emit serverStartResult(false);
        }
        break;
      case SSS_ENABLE_REVERSE:
        if (AdbEnums::AER_SUCCESS_EXEC == processResult) {
          m_enableReverse = true;
          m_serverStartStep = SSS_EXEC_SERVER;
          startServerStep();
        } else if(AdbEnums::AER_SUCCESS_START != processResult) {
          qCritical("adb reverse failed");
          m_serverStartStep = SSS_NULL;
          // 启动代理失败，则把设备上的服务器删除，不留痕迹
          removeServer();
          emit serverStartResult(false);
        }
        break;
      
      default:
        break;
    }
  }

  if(sender() == &m_serverProcess) {
    if(SSS_EXEC_SERVER == m_serverStartStep) {
      // 启动服务器为阻塞命令，只会抛出start信号
      if(AER_SUCCESS_START == processResult) {
        m_serverStartStep = SSS_RUNNING;
        emit serverStartResult(true);
      } else if(AER_ERROR_START == processResult) {
        m_serverStartStep = SSS_NULL;
        disableReverse();
        qCritical("adb start-server failed");
        removeServer();
        emit serverStartResult(false);
      }
    }
  }
}
~~~

# server.cpp::server

在构造函数中绑定连接

~~~ c++
Server::Server(QObject *parent)
  : QObject{parent}
{
  connect(&m_workProcess, &AdbProcess::adbProcessResult, this, &Server::onWorkProcessResult);
  connect(&m_serverProcess, &AdbProcess::adbProcessResult, this, &Server::onWorkProcessResult);
  connect(&m_serverSocket, &QTcpServer::newConnection, this, [this]{
    m_deviceSocket = m_serverSocket.nextPendingConnection();

    // 连接成功时，scrcpy-server会返回devices name，size
    QString deviceName;
    QSize size;
    if(m_deviceSocket && m_deviceSocket->isValid() && readInfo(deviceName, size)) {
      qDebug("Device connected");
      // 连接到设备时，反向代理就可以关闭了
      disableReverse();
      // 此时删除服务会在安卓端留下标记，当进程执行结束时，会自动删除
      removeServer();
      emit connectToResult(true, deviceName, size);
    } else {
      stopServer();
      emit connectToResult(false, deviceName, size);
    }
  });
}
~~~

# server.h::pushServer

~~~ c++
/**
  * 上传服务器
  * @brief pushServer
  * @return
  */
bool pushServer();
~~~

# server.cpp::pushServer

~~~ c++
bool Server::pushServer()
{
  // getServerPath逻辑与getAdbPath类似
  m_workProcess.push(m_serial, getServerPath(), DEVICE_SERVER_PATH);
  return true;
}
~~~

# server.h::enableReverse

~~~ C++
/**
  * 启用反向代理
  * @brief enableReverse
  * @return
  */
bool enableReverse();
~~~

# server.cpp::enableReverse

~~~ c++
bool Server::enableReverse()
{
  m_workProcess.reverse(m_serial, SOCKET_NAME, m_localPort);
  return true;
}
~~~

# server.h::executeServer

~~~ c++
/**
  * @brief 执行服务器
  * 
  * @return true 
  * @return false 
  */
bool executeServer();
~~~

# server.cpp::executeServer

封装adb：adb shell CLASSPATH=/data/local/tmp/scrcpy-server.jar app_process / com.genymobile.scrcpy.Server maxsize bitrate false

~~~ c++
bool Server::executeServer()
{
  QStringList args;
  args << "shell"
       << QString("CLASSPATH=%1").arg(DEVICE_SERVER_PATH)
       << "app_process"
       << "/"
       << "com.genymobile.scrcpy.Server"
       << QString::number(m_maxSize)
       << QString::number(m_bitRate)
       << "false"
       << "";

  m_serverProcess.execute(m_serial, args);
       
  return true;
}
~~~

# server.h::readInfo

~~~ c++
/**
  * @brief 
  * 读取设备信息
  * @param deviceName 
  * @param size 
  * @return true 
  * @return false 
  */
bool readInfo(QString& deviceName, QSize& size);
~~~

# server.cpp::readInfo

~~~ c++
bool Server::readInfo(QString & deviceName, QSize & size)
{
  // 传来的数据格式
  // abk001-----------------------0x0438 0x02d0
  //               64b            2b w   2b h
  unsigned char buffer[DEVICE_NAME_FIELD_LENGTH + 4];
  // 数据不完全时等待读取
  if(m_deviceSocket->bytesAvailable() <= DEVICE_NAME_FIELD_LENGTH + 4) {
    m_deviceSocket->waitForReadyRead(300);
  }

  qint64 len = m_deviceSocket->read((char*)buffer, sizeof(buffer));
  if(len < DEVICE_NAME_FIELD_LENGTH + 4) {
    qInfo("Could not read device info");
    return false;
  }

  // 截取设备名称
  buffer[DEVICE_NAME_FIELD_LENGTH - 1] = '\0';
  deviceName = QString::fromUtf8((char*)buffer);
  // 高8位左移8位后与低8位按位或运算，得到设备宽度
  // 计算过程：buffer[66] << 8 | buffer[67] = 0x0200 | 0xD0 = 0x02D0 = 720
  size.setWidth((buffer[DEVICE_NAME_FIELD_LENGTH] << 8 | buffer[DEVICE_NAME_FIELD_LENGTH + 1]));
  // 同宽度计算
  size.setHeight((buffer[DEVICE_NAME_FIELD_LENGTH + 2] << 8 | buffer[DEVICE_NAME_FIELD_LENGTH + 3]));

  return true;
}
~~~

# server.h::removeServer

~~~ c++
/**
  * 删除服务器
  * @brief removeServer
  * @return
  */
bool removeServer();
~~~

# server.cpp::removeServer

~~~ c++
bool Server::removeServer()
{
  if(!m_serverCopiedToDevice) {
    return true;
  }
  m_serverCopiedToDevice = false;
  
  AdbProcess* adb = new AdbProcess();
  if(!adb) {
    return false;
  }
  
  connect(adb, &AdbProcess::adbProcessResult, this, [this](AdbEnums::ADB_EXEC_RESULT processResult) {
    if(AdbEnums::AER_SUCCESS_START != processResult) {
      sender()->deleteLater();
    }
  });
  
  adb->removePath(m_serial, DEVICE_SERVER_PATH);

  return true;
}
~~~

# server.h::disableReverse

~~~ c++
/**
  * @brief 关闭反向代理
  * 
  * @return true 
  * @return false 
  */
bool disableReverse();
~~~

# server.cpp::disableReverse

~~~ c++
bool Server::disableReverse()
{
  if(!m_enableReverse) {
    return true;
  }
  
  AdbProcess* adb = new AdbProcess();
  if(!adb) {
    return false;
  }
  
  connect(adb, &AdbProcess::adbProcessResult, this, [this](AdbEnums::ADB_EXEC_RESULT processResult) {
    if(AdbEnums::AER_SUCCESS_START != processResult) {
      sender()->deleteLater();
    }
  });
  
  adb->removeReverse(m_serial, SOCKET_NAME);
  m_enableReverse = false;

  return true;
}
~~~

# server.h::stopServer

~~~ c++
/**
  * 停止服务器
  * @brief stopServer
  */
void stopServer();
~~~

# server.cpp::stopServer

~~~ c++
void Server::stopServer() {
  if(m_deviceSocket) {
    m_deviceSocket->close();
  }

  m_serverProcess.kill();
  disableReverse();
  removeServer();
  m_serverSocket.close();
}
~~~

# 一些定义和成员变量

- server.h

~~~ c++
private:
  QString m_serial;
  quint16 m_localPort;
  quint16 m_maxSize;
  quint32 m_bitRate;
  static QString m_serverPath;
  bool m_serverCopiedToDevice;
  bool m_enableReverse;

  SERVER_START_STEP m_serverStartStep = SSS_NULL;

  AdbProcess m_workProcess;
  AdbProcess m_serverProcess;

  QTcpServer m_serverSocket;
  QPointer<QTcpSocket> m_deviceSocket = nullptr;
~~~

- server.cpp

~~~ c++
#define DEVICE_SERVER_PATH "/data/local/tmp/scrcpy-server.jar"
#define SOCKET_NAME "scrcpy"
#define DEVICE_NAME_FIELD_LENGTH 64
#define MAX_CONNECTIONS 1
~~~

