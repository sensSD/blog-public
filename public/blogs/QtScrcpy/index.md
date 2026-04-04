在AI时代，vibe coding已经成为趋势。那我也应该抛弃没有AI插件的QtCreator，改为使用VsCode继续我的项目开发。

其实说这么多，就是习惯了AI帮我写代码了。

# CMake

在VC中编写Qt程序需要下载Qt官方的插件，我这里下载的是Qt Extension，下载完后只要在根目录下写个CMake文件，插件就会自动帮你构建好项目，然后就可以优雅地在VC上写代码了。

- 注：不会写CMake的话，可以直接把.pro文件丢给AI，它会告诉你怎么写。
- 注：我得学一下CMake怎么编写。

#adb的路径问题

我优化了查找adb路径的逻辑，现在大部分环境下应该都能正常运行adb指令了。

## AdbProcess.cpp:: getAdbPath()
~~~ c++
QString AdbProcess::getAdbPath()
{
    if(s_adbPath.isEmpty()) {
        s_adbPath = QCoreApplication::applicationDirPath();
        QFileInfo fileInfo(s_adbPath);
        
        if(s_adbPath.isEmpty() || !fileInfo.isFile()) {
            QDir dir = fileInfo.dir();

			// 核心逻辑，当前路径不为根目录时，cd到上一层
            while(dir.dirName() != "QtScrcpy" && !dir.isRoot()) {
                dir.cdUp();
            }

            s_adbPath = dir.path() + "/thrid_party/adb/win/adb.exe";
        }
    }
    
    // qDebug() << s_adbPath;

    return s_adbPath;
}
~~~