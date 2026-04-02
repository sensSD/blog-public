# 创建工程

创建一个QDialog

![](/blogs/QtScrcpy/42b1c5ae530742a2.png)

## 使用QProcess

用于启动外部程序

- 语法

~~~ c++
     QObject *parent;
     QString program = "./path/to/Qt/examples/widgets/analogclock";
     QStringList arguments;
     arguments << "-style" << "fusion";

     QProcess *myProcess = new QProcess(parent);
     myProcess->start(program, arguments);
~~~

- start函数各参数含义

~~~ c++
	// command：启动的程序   arguments：外部程序的参数   mode：进程的模式
    void start(const QString &program, const QStringList &arguments, OpenMode mode = ReadWrite);
~~~

# 代码实现

ui界面一个测试按钮，就不展示了。

在工程文件下创建一个adb文件夹，将pro文件（其实可以是任意文件）复制到该文件夹下，在pro文件中添加代码将其加入工程。

在新文件夹下创建c++类文件AdbProcess。将其加入工程目录

## QtScrcpy.pro
~~~ c++
	# 子工程
	include($$PWD/adb/adb.pri)

	# 包含目录
	INCLUDEPATH += \
   		$$PWD/adb
~~~

添加完后的工程目录如下

![](/blogs/QtScrcpy/e08b6faed1fe7a65.png)

## QDialog.cpp

为了方便后续引入文件，我们可以使用以下代码获取当前应用路径和工作路径
~~~ c++
	qDebug()<<"current applicationDirPath: "<< QCoreApplication::applicationDirPath();
    qDebug()<<"current currentPath: "<<QDir::currentPath();
~~~

- 编写按钮点击事件

~~~ c++
void Dialog::on_TestButton_clicked()
{
	QString program = "notepad"

    QStringList arguments;
    arguments << "devices";

    AdbProcess *myProcess = new AdbProcess(this);

    myProcess->start(program, Q_NULLPTR);
}
~~~

点击按钮后打开笔记本就是成功了。

为了后续方便获取路径，我们写一个方法获取路径。

在AdbProcess.h中定义静态变量存储adb路径，创建对应get函数

## AdbProcess.h

~~~ c++
class AdbProcess : public QProcess
{
    Q_OBJECT

    AdbProcess(QObject *parent = Q_NULLPTR);
    
    /**
     * 获取adb所在路径
     * @brief getAdbPath
     * @return 
     */
    static QString getAdbPath();

private:
    static QString s_adbPath;
};
~~~

## AdbProcess.cpp

~~~ c++
QString AdbProcess::getAdbPath()
{
    if(s_adbPath.isEmpty()) {
        s_adbPath = QString::fromLocal8Bit(qgetenv("QSCRCPY_ADB_PATH"));
        QFileInfo fileInfo(s_adbPath);
        if(s_adbPath.isEmpty() || !fileInfo.isFile()) {
			// 这个查找不到路径的逻辑不太懂，后续再改
            s_adbPath = QCoreApplication::applicationDirPath() + "/adb";
        }
    }

    return s_adbPath;
}
~~~

接下来就可以封装adb了

## AdbProcess.h

~~~ c++
void execute(const QString& serial, const QStringList& args);
~~~

## AdbProcess.cpp

~~~ c++
/**
 * 执行adb
 * @brief AdbProcess::execute
 * @param serial		// 设备名，可以为空
 * @param args		// adb指令
 */
void AdbProcess::execute(const QString &serial, const QStringList &args)
{
    QStringList adbArgs;
    if(!serial.isEmpty()) {
        adbArgs << "-s" << serial;
    }
    adbArgs << args;

    start(getAdbPath(), adbArgs);
}
~~~

封装完adb，接下来可以改写按钮逻辑

~~~ c++
void Dialog::on_TestButton_clicked()
{
    QStringList arguments;
    arguments << "devices";

    AdbProcess *myProcess = new AdbProcess(this);

    myProcess->execute("", arguments);
}
~~~

现在，adb已经初步封装好了。接下来需要排除一些异常情况，枚举出所有异常情况。

## AdbProcess.h

~~~ c++
	enum ADB_EXEC_RESULT {
        AER_SUCCESS_START,          // 启动成功
        AER_ERROR_START,            // 启动失败
        AER_SUCCESS_EXEC,           // 执行成功
        AER_ERROR_EXEC,             // 执行失败
        AER_ERROR_MISSING_BINARY,   // 找不到文件
    };
~~~

## AdbProcess.cpp

~~~ c++
void AdbProcess::initSignals()
{
    // 发生错误
    connect(this, &QProcess::errorOccurred, this, [this](QProcess::ProcessError error){
        if(QProcess::FailedToStart == error) {
            emit adbProcessResult(AER_ERROR_MISSING_BINARY);
        } else {
            emit adbProcessResult(AER_ERROR_START);
        }
        qDebug() << error;
    });

    // 退出状态
    connect(this, QOverload<int, QProcess::ExitStatus>::of(&QProcess::finished),
    [=](int exitCode, QProcess::ExitStatus exitStatus){
        if(QProcess::NormalExit == exitStatus && 0 == exitCode) {
            emit adbProcessResult(AER_SUCCESS_EXEC);
        } else {
            emit adbProcessResult(AER_ERROR_EXEC);
        }
        qDebug() << exitCode << exitStatus;
    });

    // 标准输出
    connect(this, &QProcess::readyReadStandardError, this, [this](){
        qDebug() << readAllStandardError();
    });

    connect(this, &QProcess::readyReadStandardOutput, this, [this](){
        qDebug() << readAllStandardOutput();
    });

    // 启动
    connect(this, &QProcess::started, this, [this](){
        emit adbProcessResult(AER_SUCCESS_START);
    });
}
~~~

在按钮槽中添加connect

## QDialog.cpp

~~~ c++
	connect(myProcess, &AdbProcess::adbProcessResult, this, [this](AdbProcess::ADB_EXEC_RESULT processResult){
        qDebug() << ">>>>>>>>" << processResult;
    });
~~~

今天就写到这了。

---

以下是源代码

## QtScrcpy.pro

~~~ c++
QMAKE_PROJECT_DEPTH = 0

QT       += core gui

greaterThan(QT_MAJOR_VERSION, 4): QT += widgets

CONFIG += c++17

# You can make your code fail to compile if it uses deprecated APIs.
# In order to do so, uncomment the following line.
#DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0

SOURCES += \
    main.cpp \
    dialog.cpp

HEADERS += \
    dialog.h

FORMS += \
    dialog.ui

# 子工程
include($$PWD/adb/adb.pri)

# 包含目录
INCLUDEPATH += \
    $$PWD/adb

# Default rules for deployment.
qnx: target.path = /tmp/$${TARGET}/bin
else: unix:!android: target.path = /opt/$${TARGET}/bin
!isEmpty(target.path): INSTALLS += target
~~~

## AdbProcess.h

~~~ c++
#ifndef ADBPROCESS_H
#define ADBPROCESS_H

#include <QProcess>

class AdbProcess : public QProcess
{
    Q_OBJECT
public:
    enum ADB_EXEC_RESULT {
        AER_SUCCESS_START,          // 启动成功
        AER_ERROR_START,            // 启动失败
        AER_SUCCESS_EXEC,           // 执行成功
        AER_ERROR_EXEC,             // 执行失败
        AER_ERROR_MISSING_BINARY,   // 找不到文件
    };

    AdbProcess(QObject *parent = Q_NULLPTR);

    /** 封装adb
     * 执行adb命令
     * @brief execute
     * @param serial
     * @param args
     */
    void execute(const QString& serial, const QStringList& args);
    
    /**
     * 获取adb所在路径
     * @brief getAdbPath
     * @return 
     */
    static QString getAdbPath();

signals:
    void adbProcessResult(ADB_EXEC_RESULT processResult);

private:

    /**
     * QProcess信号
     * @brief initSignals
     */
    void initSignals();

    static QString s_adbPath;
};

#endif // ADBPROCESS_H
~~~

## AdbProcess.cpp

~~~ c++
#include <QDebug>
#include <QFileInfo>
#include <QDir>
#include <QCoreApplication>

#include "adbprocess.h"

QString AdbProcess::s_adbPath = "";

AdbProcess::AdbProcess(QObject *parent)
    : QProcess(parent)
{
    initSignals();

    getAdbPath();
}

/**
 * 获取adb路径
 * @brief AdbProcess::getAdbPath
 * @return
 */
QString AdbProcess::getAdbPath()
{
    if(s_adbPath.isEmpty()) {
        s_adbPath = QString::fromLocal8Bit(qgetenv("QSCRCPY_ADB_PATH"));
        QFileInfo fileInfo(s_adbPath);
        if(s_adbPath.isEmpty() || !fileInfo.isFile()) {
            s_adbPath = QCoreApplication::applicationDirPath() + "/adb";
        }
    }

    // QDir s_adbPathDir(s_adbPath);
    // qDebug() << s_adbPathDir.absoluteFilePath(s_adbPath);

    return s_adbPath;
}

/**
 * 执行adb
 * @brief AdbProcess::execute
 * @param serial
 * @param args
 */
void AdbProcess::execute(const QString &serial, const QStringList &args)
{
    QStringList adbArgs;
    if(!serial.isEmpty()) {
        adbArgs << "-s" << serial;
    }
    adbArgs << args;

    start(getAdbPath(), adbArgs);
}

/**
 * QProcess信号
 * @brief AdbProcess::initSignals
 */
void AdbProcess::initSignals()
{
    // 发生错误
    connect(this, &QProcess::errorOccurred, this, [this](QProcess::ProcessError error){
        if(QProcess::FailedToStart == error) {
            emit adbProcessResult(AER_ERROR_MISSING_BINARY);
        } else {
            emit adbProcessResult(AER_ERROR_START);
        }
        qDebug() << error;
    });

    // 退出状态
    connect(this, QOverload<int, QProcess::ExitStatus>::of(&QProcess::finished),
    [=](int exitCode, QProcess::ExitStatus exitStatus){
        if(QProcess::NormalExit == exitStatus && 0 == exitCode) {
            emit adbProcessResult(AER_SUCCESS_EXEC);
        } else {
            emit adbProcessResult(AER_ERROR_EXEC);
        }
        qDebug() << exitCode << exitStatus;
    });

    // 标准输出
    connect(this, &QProcess::readyReadStandardError, this, [this](){
        qDebug() << readAllStandardError();
    });

    connect(this, &QProcess::readyReadStandardOutput, this, [this](){
        qDebug() << readAllStandardOutput();
    });

    // 启动
    connect(this, &QProcess::started, this, [this](){
        emit adbProcessResult(AER_SUCCESS_START);
    });
}
~~~

## Dialog.h

~~~ c++
#ifndef DIALOG_H
#define DIALOG_H

#include <QDialog>

QT_BEGIN_NAMESPACE
namespace Ui {
class Dialog;
}
QT_END_NAMESPACE

class Dialog : public QDialog
{
    Q_OBJECT

public:
    Dialog(QWidget *parent = nullptr);

    ~Dialog();

private slots:
    void on_TestButton_clicked();

private:
    Ui::Dialog *ui;
};
#endif // DIALOG_H
~~~

## Dialog.cpp

~~~ c++
#include <QDebug>
#include <QDir>

#include "dialog.h"
#include "ui_dialog.h"
#include "adbprocess.h"

Dialog::Dialog(QWidget *parent)
    : QDialog(parent)
    , ui(new Ui::Dialog)
{
    ui->setupUi(this);
}

Dialog::~Dialog()
{
    delete ui;
}

void Dialog::on_TestButton_clicked()
{
    // 获取当前工作路径
    // qDebug()<<"current applicationDirPath: "<< QCoreApplication::applicationDirPath();
    // qDebug()<<"current currentPath: "<<QDir::currentPath();

    QStringList arguments;
    arguments << "devices";

    AdbProcess *myProcess = new AdbProcess(this);

    connect(myProcess, &AdbProcess::adbProcessResult, this, [this](AdbProcess::ADB_EXEC_RESULT processResult){
        qDebug() << ">>>>>>>>" << processResult;
    });

    /**
    command：启动的程序   arguments：外部程序的参数   mode：进程的模式
    void start(const QString &program, const QStringList &arguments, OpenMode mode = ReadWrite);
    */
    myProcess->execute("", arguments);
}
~~~

## main.cpp

~~~ c++
#include "dialog.h"

#include <QApplication>

int main(int argc, char *argv[])
{
    // 设置环境变量
    qputenv("QSCRCPY_ADB_PATH", "..\\..\\..\\thrid_party\\adb\\win\\adb.exe");
    QApplication a(argc, argv);
    Dialog w;
    w.show();
    return a.exec();
}
~~~