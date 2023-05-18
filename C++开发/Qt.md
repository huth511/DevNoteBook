## Qt编译安装

### 流程

- 下载：`https://mirrors.aliyun.com/qt/official_releases/qt/5.15/5.15.2/single/qt-everywhere-src-5.15.2.tar.xz?spm=a2c6h.25603864.0.0.1396238euHTqrq`
- 解压：`tar xJvf qt-everywhere-src-5.15.2.tar.xz`
- 配置：`./configure -prefix /opt/Qt5.15 -skip qtconnectivity -skip qt3d -skip qtandroidextras -skip qtgamepad -skip qtdoc -skip qtwinextras -skip qtvirtualkeyboard -skip qtx11extras -skip qtquick3d -skip qtquickcontrols -skip qtquickcontrols2 -skip qtquicktimeline -skip qtmacextras -skip qtscript -skip qtpurchasing -skip qtsensors -skip qtspeech -skip qtwebchannel -skip qtwayland -skip qtactiveqt -skip qtremoteobjects -skip qtgraphicaleffects -skip qtserialbus -skip qtserialport -skip qtwebview -skip qttools -no-feature-dbus -no-feature-d3d12 -qt-zlib -qt-freetype -platform linux-g++`
  - module即目录下的qt文件夹
  - 显示所有的feature：`./configure -list-features`
    - 禁用feature：`-no-feature-XXX`

### 使用docker安装

- 启动

  ```sh
  docker run -d --privileged=true --name qt_dev --entrypoint /sbin/init -v "D:\GraphScope\qemu-aarch64-static:/usr/bin/qemu-aarch64-static" carlonluca/qt-dev:5.15.2
  
  docker run -d --privileged=true --name tess_worker1 --entrypoint /sbin/init -v "D:\GraphScope\qemu-aarch64-static:/usr/bin/qemu-aarch64-static" -v "F:\Tess_docker_arm\share":"/home/huth/share" -p 22226:22 -p 57790:7788 tess_dev_env:qt5.15.4
  
  docker run -d --privileged=true --name tess_worker4 --entrypoint /sbin/init  -v "D:\GraphScope\qemu-aarch64-static:/usr/bin/qemu-aarch64-static" -v "F:\Tess_docker_arm\share":"/home/huth/share" -p 22228:22 -p 57792:7788 tess_dev_env:qt5.15.2_2
  ```

## 跨线程调用

- signal/slot中注意事项
  - 自定义类型需要：
    - 继承QObject
    - 并添加宏Q_Object
    - 并添加`qRegisterMetaType`
      - 注册位置：在第一次使用此类链接跨线程的signal/slot之前，一般在当前类的构造函数中进行注册；
      - 注册方法：在当前类的顶部包含：#include <QMetaType>，构造函数中加入代码：`qRegisterMetaType<MyClass>("Myclass")；`
      - Myclass的引用类型需单独注册：`qRegisterMetaType<MyClass>("Myclass&")；`

## 一些注意

### QMetaType问题

问题描述：

```cpp
// 有以下使用
QVariant::fromValue<GraphicsBaseItem::ShapeType>(GraphicsBaseItem::RECTANGLE);

// GraphicsBaseItem是自定义类型，必须要经过以下宏处理
Q_DECLARE_METATYPE(GraphicsBaseItem);
/*  该宏里会调用qRegisterMetaType<T>()方法，该方法需要T拥有：
	default constructor,
	copy constructor,	[1]
	public destructor
*/

// 但是GraphicsBaseItem继承了QGraphicsItem
class GraphicsBaseItem : public QGraphicsItem {}
// 并且QGraphicsItem里删除了拷贝，拷贝赋值函数	<1>
Q_DISABLE_COPY(QGraphicsItem)

// 所以会报错，也无法通过添加拷贝构造函数等来解决
```

解决：
`Q_DECLARE_METATYPE(GraphicsBaseItem::ShapeType)`，它只需要ShapeType被声明就可以了

### 继承QObject和非Qt类

*目前来看：要让其具有Qt特性，QObject要放在第一位继承*

### QMetaObject::invokeMethod(...)

```cpp
QMetaObject::invokeMethod(
    mpTessClient, 					// 调用对象
    "doOnNewestBatch", 				// 调用方法名
    Qt::QueuedConnection, 			// 连接方式
    QGenericReturnArgument(), 		// 返回参数
    Q_ARG(int, mPartition), 		// 方法输入参数0
    Q_ARG(uint64_t, batch_num)		// 方法输入参数1
    // ... ...
);
```

必须是qt元系统能识别的函数，比如应该设为信号或槽，或用`Q_INVOKABLE`声明

## QMake使用

### 添加编译参数

```makefile
QMAKE_CXXFLAGS += -ggdb  -Wno-unused-parameter -Wunused-variable
```

