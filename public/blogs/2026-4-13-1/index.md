由于要从deviceSocket获取视频数据进行解码，而解码是耗时的步骤，为了提高渲染效率，我们需要改造deviceSocket让其在单独线程运行

- 先创建DeviceSocket文件

在写代码前，我们先复习下生产者-消费者模型

# 生产者-消费者

有两个进程A和B，它们共享一个**固定大小的缓冲区**，A进程产生数据放入缓冲区，B进程从缓冲区中取出数据进行计算，那么这里其实就是一个生产者和消费者的模式，A相当于生产者，B相当于消费者

![](https://pic2.zhimg.com/v2-cb47f33c3a2e5e3091013b945cf661c3_r.jpg)

## Qt中的生产者-消费者

- #### QWaitCondition
  **[csdn链接 - QWaitCondition的使用](https://blog.csdn.net/haokan123456789/article/details/136108511)**<br>
  用来同步线程的条件变量，类中的所有函数都是**线程安全**的。
  QWaitCondition允许线程告诉其他线程某种条件已经满足。一个或多个线程可以阻止等待QWaitCondition使用wakeOne()或wakeAll()设置条件。使用wakeOne()唤醒一个随机选择的线程，或使用wakeAll()唤醒所有线程。
  一般与QMutex一起使用。
- #### QMutex、QMutexLocker
  **[csdn链接 - QMutex、QMutexLocker的使用](https://blog.csdn.net/m0_46577050/article/details/145915100)**<br>
  互斥锁（QMutex）在使用时需要在**进入和结束的时候使用对应的函数锁定和解锁**。在简单的程序中还好，但是在结构复杂的程序中因为需要手动锁定和解锁，很容易忽略细节而出现问题，于是为了应对这种情况QMutexLocker便诞生了（为了简化简化互斥锁的锁定和解锁）。
  QMutexLocker通常创建为局部变量，QMutexLocker在创建时传入一个并未锁定（若是锁定可用relock重新锁定或unlock解锁）的QMutex指针变量，并且会将QMutex变量锁定，在释放时会将QMutex变量解锁。（**QMutexLocker创建时将传入的QMutex锁定，释放时将传入的QMutex解锁**）

# 代码实现

## DeviceSocket.h

成员变量

~~~ c++
private:
  // 锁
  QMutex m_mutex;
  // 线程控制
  QWaitCondition m_recvDataCond;

  // 标志
  bool m_recvData = false;
  bool m_quit = false;

  // 数据缓存
  quint8* m_buffer;
  // 缓冲区和实际数据大小可能不同
  qint32 m_bufferSize = 0;
  qint32 m_dataSize = 0;
~~~

## DeviceSocket.h::subThreadRecvData

~~~ c++
  /**
   * @brief 生产者线程调用，接收数据
   * 
   * @param buf 
   * @param bufSize 
   * @return qint32 
   */
  qint32 subThreadRecvData(quint8 *buf, qint32 bufSize);
~~~

## DeviceSocket.cpp::subThreadRecvData

- Q_ASSERT：当条件不满足时，终止运行程序

~~~ c++
qint32 DeviceSocket::subThreadRecvData(quint8 *buf, qint32 bufSize) {
  // 保证在子线程中调用
  Q_ASSERT(QCoreApplication::instance()->thread() != QThread::currentThread());
  if (m_quit) {
    return 0;
  }
  
  // 对当前进程上锁
  QMutexLocker locker(&m_mutex);
  
  // 初始化
  m_buffer = buf;
  m_bufferSize = bufSize;
  m_dataSize = 0;
  
  // 有空位时进入缓冲区
  while(!m_recvData) {
    m_recvDataCond.wait(&m_mutex);
  }
  
  m_recvData = false;
  return m_dataSize;
}
~~~

## DeviceSocket.h::onReadyRead

~~~ c++
protected slots:
  /**
   * @brief 消费者线程调用，接收数据事件的槽函数
   * 
   */
  void onReadyRead();
~~~

## DeviceSocket.h::onReadyRead

~~~ c++
void DeviceSocket::onReadyRead() {
  QMutexLocker locker(&m_mutex);
  // 当缓冲区有数据时
  if (m_buffer && 0 < bytesAvailable()) {
    // 取可用字节数和缓冲区大小的最小值，确保不会尝试读取超过缓冲区容量的数据
    qint64 readSize = qMin(bytesAvailable(), (qint64)m_bufferSize);
    m_dataSize = read((char*)m_buffer, m_bufferSize);
    m_buffer = nullptr;
    m_bufferSize = 0;
    m_recvData = true;
    m_recvDataCond.wakeOne();
  }
}
~~~

## DeviceSocket.h::quitNotify

~~~ c++
protected slots:
  /**
   * @brief 退出DeviceSocket
   * 
   */
  void quitNotify();
~~~

## DeviceSocket.cpp::quitNotify

~~~ c++
void DeviceSocket::quitNotify() {
  // 设置退出标志m_quit为true
  m_quit = true;
  // 在互斥锁保护下清空缓冲区相关变量
  QMutexLocker locker(&m_mutex);
  if(m_buffer) {
    m_buffer = nullptr;
    m_bufferSize = 0;
    m_recvData = true;
    m_dataSize = 0;
    // 唤醒等待条件变量的线程，确保其他线程能及时响应退出请求
    m_recvDataCond.wakeOne();
  }
}
~~~

至此，我们解决了多进程的处理问题，剩下在不同线程间传递数据的问题还没解决。为此，需要继承QEvent编写发送事件

## QScrspyEvent.h

~~~ c++
#pragma once

#include "QScrcpyEventEnums.h"
#include <QEvent>

using namespace QScrcpyEventEnums;

class QScrcpyEvent : public QEvent {
public:
    QScrcpyEvent(QScrcpyEventEnums::Type type) : QEvent(static_cast<QEvent::Type>(type)) {}
};

class DeviceSocketEvent : public QScrcpyEvent {
public:
    DeviceSocketEvent() : QScrcpyEvent(QScrcpyEventEnums::DeviceSocket) {}
};
~~~

现在就可以在DeviceSocket中添加事件处理了

## DeviceSocket.h::event

~~~ c++
protected:
  /**
   * @brief 发送事件与主线程通信
   * 
   * @param event 
   * @return true 
   * @return false 
   */
  bool event(QEvent *event) override;
~~~

## DeviceSocket.cpp::event

~~~ c++
bool DeviceSocket::event(QEvent *event) {
  if (event->type() == QScrcpyEventEnums::DeviceSocket) {
    onReadyRead();
    return true;
  }
  return QTcpSocket::event(event);
}
~~~

在subThreadRecvData中添加两行代码

~~~ c++
qint32 DeviceSocket::subThreadRecvData(quint8 *buf, qint32 bufSize) {
  // 保证在子线程中调用
  Q_ASSERT(QCoreApplication::instance()->thread() != QThread::currentThread());
  if (m_quit) {
    return 0;
  }
  
  QMutexLocker locker(&m_mutex);
  
  m_buffer = buf;
  m_bufferSize = bufSize;
  m_dataSize = 0;

  // 发送事件
  DeviceSocketEvent* getDataEvent = new DeviceSocketEvent();
  QCoreApplication::postEvent(this, getDataEvent);
  
  while(!m_recvData) {
    m_recvDataCond.wait(&m_mutex);
  }
  
  m_recvData = false;
  return m_dataSize;
}
~~~

- 这两段代码实现了一个跨线程的异步数据读取机制，用于解决DeviceSocket在不同线程间的数据传递问题。

## DeviceSocket.cpp::构造函数

~~~ c++
DeviceSocket::DeviceSocket(QObject *parent) : QTcpSocket{parent} {
  connect(this, &DeviceSocket::readyRead, this, &DeviceSocket::onReadyRead);
  connect(this, &DeviceSocket::disconnected, this, &DeviceSocket::quitNotify);
  connect(this, &DeviceSocket::aboutToClose, this, &DeviceSocket::quitNotify);
}
~~~

- 需要学习的点：对多线程处理流程还是不太熟悉，在了解了消费者-生产者模型的基础上看代码还是迷迷糊糊。对异步也需要更多的学习