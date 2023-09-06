## 实习问题：

### 1 Qt信号槽机制的优势和不足

​    优点：类型安全，松散耦合。缺点：同回调函数相比，运行速度较慢。

优点：

类型安全：需要关联的信号槽的签名必须是等同的。即信号的参数类型和参数个数同接受该信号的槽的参数类型和参数个数相同。若信号和槽签名不一致，则编译器会报错。
松散耦合：信号和槽机制减弱了Qt对象的耦合度。激发信号的Qt对象无需知道是哪个对象的那个槽接收它发出的信号，它只需要在适当的时间发送适当的信号即可，而不需要关心是否被接收和哪个对象接收了。Qt保证适当的槽得到了调用，即使关联的对象在运行时删除，程序也不会崩溃。
灵活性：一个信号可以关联多个槽，多个信号也可以关联同一个槽。
缺点：

速度较慢：与回调函数相比，信号和槽机制运行速度比直接调用非虚函数慢10倍左右。
原因：

需要定位接收信号的对象。
安全地遍历所有槽。
编组，解组传递参数。
多线程的时候，信号需要排队等候。（然而，与创建对象的new操作及删除对象的delete操作相比，信号和槽的运行代价只是他们很少的一部分。信号和槽机制导致的这点性能损失，对于实时应用程序是可以忽略的。）

### 一、讲述Qt信号槽机制与优势与不足

优点： ①类型安全。需要关联的信号槽的签名必须是等同的。即信号的参数类型和参数个数同接受该信号的槽的参数类型和参数个数相同。若信号和槽签名不一致，编译器会报错。

          ②松散耦合。信号和槽机制减弱了Qt对象的耦合度。激发信号的Qt对象无需知道是那个对象的那个信号槽接收它发出的信号，它只需在适当的时间发送适当的信号即可，而不需要关心是否被接受和那个对象接受了。Qt就保证了适当的槽得到了调用，即使关联的对象在运行时被删除。程序也不会奔溃。
    
         ③灵活性。一个信号可以关联多个槽，或多个信号关联同一个槽。

不足：速度较慢。与回调函数相比，信号和槽机制运行速度比直接调用非虚函数慢10倍。

        原因：①需要定位接收信号的对象。②安全地遍历所有关联槽。③编组、解组传递参数。④多线程的时候，信号需要排队等待。（然而，与创建对象的new操作及删除对象的delete操作相比，信号和槽的运行代价只是他们很少的一部分。信号和槽机制导致的这点性能损耗，对实时应用程序是可以忽略的。）

### 二、Qt信号和槽的本质是什么 

​        回调函数。信号或是传递值，或是传递动作变化；槽函数响应信号或是接收值，或者根据动作变化来做出对应操作。

### 三、描述QT中的文件流(QTextStream)和数据流(QDataStream)的区别

​       文件流(QTextStream)。操作轻量级数据（int,double,QString）数据写入文本件中以后以文本的方式呈现。

       数据流(QDataStream)。通过数据流可以操作各种数据类型，包括对象，存储到文件中数据为二进制。
    
        文件流，数据流都可以操作磁盘文件，也可以操作内存数据。通过流对象可以将对象打包到内存，进行数据的传输。

### 四、描述QT的TCP通讯流程

服务端：（QTcpServer）

```c++
    ①创建QTcpServer对象

    ②监听list需要的参数是地址和端口号

    ③当有新的客户端连接成功回发送newConnect信号

    ④在newConnection信号槽函数中，调用nextPendingConnection函数获取新连接QTcpSocket对象

    ⑤连接QTcpSocket对象的readRead信号

    ⑥在readRead信号的槽函数使用read接收数据

    ⑦调用write成员函数发送数据
    
    
    Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);
    tcpServer = new QTcpServer;
    tcpServer->listen(QHostAddress("192.168.0.111"),1234);
    connect(tcpServer,SIGNAL(newConnection()),this,SLOT(new_connect()));
}

Widget::~Widget()
{
    delete ui;
}

void Widget::new_connect()
{
    qDebug("--new connect--");
    QTcpSocket* tcpSocket = tcpServer->nextPendingConnection();
    connect(tcpSocket,SIGNAL(readyRead()),this,SLOT(read_data()));
    socketArr.push_back(tcpSocket);

}

void Widget::read_data()
{
    for(int i=0; i<socketArr.size(); i++)
    {
        if(socketArr[i]->bytesAvailable())
        {
            char buf[256] = {};
            socketArr[i]->read(buf,sizeof(buf));
            qDebug("---read:%s---",buf);
        }
    }
}
```


客户端：（QTcpSocket）

```c++
    ①创建QTcpSocket对象

    ②当对象与Server连接成功时会发送connected 信号

    ③调用成员函数connectToHost连接服务器，需要的参数是地址和端口号

    ④connected信号的槽函数开启发送数据

    ⑤使用write发送数据，read接收数据
    
    Widget::Widget(QWidget *parent) :
    QWidget(parent),
    ui(new Ui::Widget)
{
    ui->setupUi(this);
    tcpSocket = new QTcpSocket;
    connect(tcpSocket,SIGNAL(connected()),this,SLOT(connect_success()));
    tcpSocket->connectToHost("172.20.10.3",1234);
}

Widget::~Widget()
{
    delete ui;
}

void Widget::on_send_clicked()
{
    std::string msg = ui->msg->text().toStdString();
    int ret = tcpSocket->write(msg.c_str(),msg.size()+1);
    qDebug("--send:%d--",ret);
}

void Widget::connect_success()
{
    ui->send->setEnabled(true);
}
```

### 五、 描述UDP 之 UdpSocket通讯

​           UDP（User Datagram Protocol即用户数据报协议）是一个轻量级的，不可靠的，面向数据报的无连接协议。在网络质量令人十分不满意的环境下，UDP协议数据包丢失严重。由于UDP的特性：它不属于连接型协议，因而具有资源消耗小，处理速度快的优点，所以通常音频、视频和普通数据在传送时使用UDP较多，因为它们即使偶尔丢失一两个数据包，也不会对接收结果产生太大影响。所以QQ这种对保密要求并不太高的聊天程序就是使用的UDP协议。

        在Qt中提供了QUdpSocket 类来进行UDP数据报（datagrams）的发送和接收。Socket简单地说，就是一个IP地址加一个port端口  。
    
        流程：①创建QUdpSocket套接字对象 ②如果需要接收数据，必须绑定端口 ③发送数据用writeDatagram，接收数据用 readDatagram 。

### 六、多线程使用使用方法

​        方法一：①创建一个类从QThread类派生②在子线程类中重写 run 函数, 将处理操作写入该函数中 ③在主线程中创建子线程对象, 启动子线程, 调用start()函数

        方法二：①将业务处理抽象成一个业务类, 在该类中创建一个业务处理函数②在主线程中创建一QThread类对象 ③在主线程中创建一个业务类对象 ④将业务类对象移动到子线程中 ⑤在主线程中启动子线程 ⑥通过信号槽的方式, 执行业务类中的业务处理函数

多线程使用注意事项: 
* 1. 业务对象, 构造的时候不能指定父对象 
* 2. 子线程中不能处理ui窗口(ui相关的类) 
* 3. 子线程中只能处理一些数据相关的操作, 不能涉及窗口

### 七、多线程下，信号槽分别在什么线程中执行，如何控制

​        可以通过connect的第五个参数进行控制信号槽执行时所在的线程

　　connect有几种连接方式，直接连接和队列连接、自动连接

　　直接连接（Qt::DirectConnection）：信号槽在信号发出者所在的线程中执行

　　队列连接 (Qt::QueuedConnection)：信号在信号发出者所在的线程中执行，槽函数在信号接收者所在的线程中执行

　　自动连接  (Qt::AutoConnection)：多线程时为队列连接函数，单线程时为直接连接函数。





### 八、qt来做界面、信号槽机制、槽连接方式、qt多线程、死锁处理

1.为什么要用qt来做界面
Qt的跨平台性很强，比如同样一套代码写好pro文件可以在windows/linux/Android等直接编译。
2.信号槽机制
在事件的处理方面，信号槽相比回调函数，具有类型安全、松耦合、任意参数的优势，但执行效率会有一点损失。
3.槽连接方式
Direction、queued、blockingqueued、unique、auto
4.qt多线程
两种基本方式，一种是QObject继承，将对象MoveToThread(&QThread)，另一种是QThread继承，并重写run函数。
5.死锁处理
QSemaphore、QMutex、QAtomic等。参考操作系统

### 九、QTL、qt如何显示图片、show()和exec()的区别、qt容器

6.QTL
qt容器，和stl差不多，似乎耗时和内存比stl都更少一点。参考数据结构
7.qt如何显示图片
QLabel
8.show()和exec()的区别
show显示非模态窗口（不影响用户对其他窗口操作），exec显示模态窗口（阻塞其他窗口，必须在当前窗口操作完成后才能访问其他窗口），open半模态（阻塞其他窗口响应，但不影响后续代码执行）
9.qt容器
常见数据结构理解，例如顺序性，重复性，以及增删改查的基本步骤
lambda表达式
常用在绑定槽和并发处，比较实用，捕获输入返回等



## 7. Qt的D指针（`d_ptr`）与Q指针（`q_ptr`）

D指针

PIMPL模式，指向一个包含所有数据的私有数据结构体。

- 私有的结构体可以随意改变，而不需要重新编译整个工程项目
- 隐藏实现细节
- 头文件中没有任何实现细节，可以作为API使用
- 原本在头文件的实现部分转移到乐源文件，所以编译速度有所提高

Q指针

私有的结构体中储存一个指向公有类的Q指针。

总结

- Qt中的一个类常用一个PrivateXXX类来处理内部逻辑，使得内部逻辑与外部接口分开，这个PrivateXXX对象通过D指针来访问；在PrivateXXX中有需要引用Owner的内容，通过Q指针来访问。
- 由于D和Q指针是从基类继承下来的，子类中由于继承导致类型发生变化，需要通过`static_cast`类型转化，所以`DPTR()`与`QPTR()`宏定义实现了转换。



## 8. Qt信号槽（反射机制）相关

### Qt信号槽的调用流程

- MOC查找头文件中的signal与slots，标记出信号槽。将信号槽信息储存到类静态变量staticMetaObject中，并按照声明的顺序进行存放，建立索引。
- connect链接，将信号槽的索引信息放到一个双向链表中，彼此配对。
- emit被调用，调用信号函数，且传递发送信号的对象指针，元对象指针，信号索引，参数列表到active函数。
- active函数在双向链表中找到所有与信号对应的槽索引，根据槽索引找到槽函数，执行槽函数。

### 信号槽的实现：元对象编译器MOC

元对象编译器MOC负责解析signals、slot、emit等标准C++不存在的关键字，以及处理Q_OBJECT、Q_PROPERTY、Q_INVOKABLE等相关的宏，生成moc_xxx.cpp的C++文件（使用黑魔法来变现语法糖）。比如信号函数只要声明、不需要自己写实现，就是在这个moc_xxx.cpp文件中自动生成的。

**moc的本质就是反射器**。

### Qt信号槽的链接方式（connect的第五个参数）

1. `Qt::AutoConnection`： 默认值，使用这个值则连接类型会在信号发送时决定。如果接收者和发送者在同一个线程，则自动使用Qt::DirectConnection类型。如果接收者和发送者不在一个线程，则自动使用Qt::QueuedConnection类型。
2. `Qt::DirectConnection`：槽函数会在信号发送的时候直接被调用，槽函数运行于信号发送者所在线程。效果看上去就像是直接在信号发送位置调用了槽函数。这个在多线程环境下比较危险，可能会造成奔溃。
3. `Qt::QueuedConnection`：槽函数在控制回到接收者所在线程的事件循环时被调用，槽函数运行于信号接收者所在线程。发送信号之后，槽函数不会立刻被调用，等到接收者的当前函数执行完，进入事件循环之后，槽函数才会被调用。多线程环境下一般用这个。
4. `Qt::BlockingQueuedConnection`：槽函数的调用时机与Qt::QueuedConnection一致，不过发送完信号后发送者所在线程会阻塞，直到槽函数运行完。接收者和发送者绝对不能在一个线程，否则程序会死锁。在多线程间需要同步的场合可能需要这个。
5. `Qt::UniqueConnection`：这个flag可以通过按位或（|）与以上四个结合在一起使用。当这个flag设置时，当某个信号和槽已经连接时，再进行重复的连接就会失败。也就是避免了重复连接。



## 9. Qt智能指针相关

Qt的智能指针包括：

- QSharedPointer
- QScopedPointer
- QScopedArrayPointer
- QWeakPointer
- QPointer
- QSharedDataPointer

QSharedPointer

相当于`std::shared_ptr`，内部维持着对拥有的内存资源的引用计数，引用计数下降到0时，这个内存资源就被释放了。

QSharedPointer是线程安全的，多个线程同时修改QSharedPointer对象也不需要加锁，但是QSharedPointer指向的内存区域不一定是线程安全的，所以多个线程同时修改QSharedPointer指向的数据时还要考虑加锁。

QWeakPointer

类似于`std::weak_ptr`。

QScopedPointer

相当于`std::unique_ptr`，内存数据只在一处被使用。

QScopedArrayPointer

类似于QScopedPointer，用于指向的内存数据是一个数组时的场景。

QPointer

QPointer只能用于指向QObject及派生类的对象。当一个QObject或派生类对象被删除后，QPointer能自动将其内部的指针设置为0，这样在使用QPointer之前就可以判断一下是否有效乐。

**QPointer对象超出作用域时，并不会删除它指向的内存对象。**

QSharedPointer

用于实现数据的隐式共享。Qt中大量使用了隐式共享与写时拷贝技术，例如：

```cpp
QString str1 = "abc";
QString str2 = str1;
str2[2] = "X"; 
```

第二行执行完后，str2和str1指向同一片内存数据。第三句执行时，Qt会为str2的内部数据重新分配内存。这样做的好处是可以有效地减少大片数据拷贝的次数，提高程序的运行效率。

Qt中隐式共享和写时拷贝就是利用QSharedDataPointer和QSharedData这两个类实现的。





# 喷浆机械臂控制软件开发

在完成喷浆机械臂的运动学建模，轨迹规划和跟踪控制研究基础上，本章基于Qt环境下开发机械臂控制软件，实现三维可视化，机械臂控制，轨迹规划，通讯模块和人机交互界面等多个功能。搭建喷浆机械臂、喷浆机械臂控制软件和SYMC控制器组成的实验平台，进行隧道模拟拱架下的实车试验。通过实验结果分析，确定关节和喷嘴的跟踪误差符合自动化喷浆机械臂行业标准



### 何磊整体流程：

本研究通过对**喷浆机械臂运动学、动力学、轨迹规划、跟踪控制以及控制软件**等方面的系统研究，旨在提高喷浆机械臂的运行精度和稳定性，总结相关的工作成果如下：

（1）对**六自由度喷浆机械臂进行运动学分析，基于D-H法建立正运动学模型，**通过**各关节的变化量得到末端喷头的实时位置和姿态**。基于正运动学模型，**采用约束关节的解析法进行逆运动学求解，通过末端喷嘴的位姿可求到各关节的目标关节量，**因为该喷浆机械臂采用液压缸直接驱动方式，机械臂运动关节数量最少是符合最优解。

（2）针对隧道的受喷面，**规划出扫面和填充工序的喷嘴运动轨迹规划，完成了基于工人经验的喷浆参数的设置**。在喷嘴运动运动轨迹基础上，对六个关节进行了多项式插值对比实验，验证五次多项式在喷浆机械臂的轨迹规划优势，满足冲击最优算法。

（3）通过拉格朗日法推导机械臂的动力学方程，并利用SimMechanics建立动力学模型，以验证动力学方程的准确性。基于对滑模控制的滑模面、趋近率以及传统指数滑模算法的问题分析，本文提出了一种**新型的自适应模糊滑模控制算法**，通过引入新的滑模面和模糊系统来处理外部干扰和切换函数，从而实现干扰抑制和系统稳定。通过应用Lyapunov稳定性分析方法，证明了该新型滑模控制算法的稳定性。最后，使用机械臂的动力学方程进行了仿真试验，将新型滑模控制算法与传统指数滑模控制算法进行对比。仿真结果表明，**新型滑模控制算法在关节位置和速度跟踪性能方面表现更优，并且能够有效消除控制输入的小范围高频抖动现象，具有更强的抗干扰性和稳定性。**

（4）基于Qt开发了一款人机交互性强的上位机软件，并运用面向对象编程和结构化技术，将**运动学、动力学、轨迹规划和跟踪控制等研究内容集成到机械臂控制软件中**。根据软件架构设计，分别设计和实现了**GUI模块、通讯模块和模式切换模块**。在喷浆机械臂试验平台上进行了现场实验，得到了**六个关节的跟踪曲线和喷嘴在三维坐标系中的误差，**并满足自动化喷浆机械臂行业的要求。实验结果表明，本文提出的喷浆机械臂轨迹规划和新型自适应模糊滑模控制器方法具有有效性和实用性。

### 项目理解：

上位机（Qt自带的UI）只跟控制器通信，控制器再输出电气io控制液压阀再控制机械臂。传感器启动了自己会发送数据，下位机控制电气io控制液压阀。

上位机持续监听关节传感器的数据(告诉上位机关节角度数据），根据遥控器的信息逆解算出各个关节的数据，也就是上位机做关节姿态信息的逆解运算，把运算的信息发给下位机的控制器，下位机做控制（CAN）。

CAN通信协议是将传感器的数据信号编码然后发给上位机。

**臂架控制系统：**

传感器（6个旋转编码器，两个拉线传感器，水平仪）。臂架姿态的精确获取是实现臂架智能运动控制的前提

**末端位置控制：**利用运动学求解得出喷头移动到设定位置所需要的关节动作，并通过算法得出最佳的运动方式，尽量减少关节分步动作带来的臂架动作不连贯。



上位机通信 使用共享资源通信，只有一个进程、几个线程





**怎么尽可能保证控制精度：**用液压系统，但是没法做到很高精度，比如现在期望关节运动到45°，那么关节40°的时候，我们还会控制它上抬，抬到45°±2°算它成功了，误差不能保证。

**如何改善**：1、改善液压系统，2、设计控制算法，比如PID去优化

**运动学**：问逆解，解析法，几何法（贺毅用几何法，何磊用解析）

 

Sendmsg 代码是can，里面有两个线程，mycan receive跟另一个，继承了窗口类的类是GUI





### 5.2.1 上位机软件总架构

Qt环境下开发机械臂控制软件，实现三维可视化，机械臂控制，轨迹规划，通讯模块和人机交互界面等多个功能。

![img](file:///C:\Users\彭凯\AppData\Local\Temp\ksohtml34564\wps1.jpg) 

#### **GUI模块**

在GUI编程中，用到了QWidget类，这个类提供了渲染到屏幕以及处理用户输入事件的基本类。 Qt提供的所有UI元素都是QWidget的子类。 通过继承QWidget重新实现虚拟事件处理程序来完成创建自定义窗口小部件。当更改一个widget(所有用户界面对象的基类）时候，希望通知另一个widget。

在Qt中使用**信号和槽**替代回调技术。特定事件发出信号，有槽函数响应信号从而调用函数。信号的签名必须和槽的签名相匹配，编译器运行时候帮助检查字符串的SIGNAL和SLOT语法类型匹配。发出信号的类不知道和哪个槽函数连接，

所有信号和槽是松耦合。Qt的信号和槽机制可以保证将信号连接到槽后，槽将在正确的事件用信号参数调用槽函数。**使用QObject::connect()函数调用实现信号和槽函数的连接**。信号和槽提高了开发效率，保证界面和数据交互的通讯安全。

#### OpenGL模块

GUI模块使用了Solidworks软件对喷浆机械臂进行了建模，并为其附加了零件材质，保存为IGS格式，并用3ds Max软件进行模型缩放，最后导出为obj格式。通过使用OpenGL类库中的glRotatef和glTranslatef函数对每个臂架的坐标系位置和方向进行了改变，以实现对机械臂运动的控制和显示。通过load函数，我们将obj文件导入Qt的OpenGL中，从而实现了对机械臂的三维模型的控制和显示。

#### **通讯模块**

主要用来**传输传感器的数据、遥控器手柄信号、上下位机**之间的通信。

上位机的通信功能主要为编码发送信息和解码接收信息，接收功能采用VCI_Receive实现，发送功能采用VCI_Transmit实现。

上位机软件**逆解6个关节角度的运动控制信息、三种模式切换信号**通过CAN通信给控制器。

控制器将传感器检测到的6个关节角度，遥控器数据（按钮、摇杆）编码通过CAN线发送给上位机。

 

##### **代码中的实现：**

 **通信模块：sendmsg.cpp的核心包括发送接口函数和接收接口函数**。

上位机的通信功能主要为编码发送信息和解码接收信息，接收功能采用VCI_Receive实现，发送功能采用VCI_Transmit实现。

**发送接口函数**是将**待发送的CAN消息**（关节传感器数据）编码添加到发送队列中，定时将其发送到总线上，以避免消息的丢失或冲突。

接收接口函数是将接收到的数据放入接收队列**中，作为一个缓存区来存储接收到的CAN数据帧，接收队列通常采用循环队列或链式队列**等数据结构实现，使用先进先出的原则进行数据帧的存储和管理。CAN总线可能接收到多个节点的数据帧，因此使用接收队列可以帮助系统缓存并按顺序处理这些数据帧，避免数据的丢失或冲突。

**代码实现过程：**

根据CAN网络的通信原理搭建了CAN网络通信，其中工控机和运动控制之间采用USB转CAN模块。USB转CAN模块兼容周立功API及CANTest软件，上位机软件开发CAN通信采用周立功API。上位机的通信功能主要为编码发送信息和解码接收信息。接收功能采用VCI_Receive实现，发送功能采用VCI_Transmit实现。



##### 代码实现2：

CAN通信线程：文件名canthread。

CANOpen为PCAN包含文件。

run函数中进行CAN信号的接收监听与数据解析，之后触发对应的信号。

发送消息API：由于不在CAN线程下的run函数中执行，所以不在CAN线程中。

CAN线程收到机械臂到达位置里程点确认信息之后，则向机械臂控制器发送臂架矩阵信息，监测传感器数据以判断臂架是否停好，之后执行机器人线程（相机线程canmera_thread。）

轨迹规划：给定起始点与终点坐标，适当插值并设定AUBO轨迹运动模式。

机器人线程：文件名robotthread。开始工作，当机器人线程stop开关为假、喷涂位置点队列不为空且AUBO工作状态正常时依次执行：机械臂运动至位置点——喷涂——机械臂运动至指定点——释放（运动执行需上位机路径规划后）。喷涂若需要控制器控制则发送喷涂指令至控制器，延时等待一段时间后执行下一步。执行完毕之后发送喷涂小区域完毕加一的对应信号。界面中响应对喷涂小区域进行计数的槽函数。



###### 编码解码什么做的

编码传感器数据，遥控器数据，下位机进行数据解码处理

有两个传感器是通过下位机转发因为走的can1，其他走can0所以直接拿（国产两个，大臂和关节6，走同一个can线会掉，所以靠下位机转发）

解码就是下位机要进行编码，把有些数据传上来，上位机发的数据需要在下位机解码给相应接口使用。





##### CAN原理：

CAN报文是指发送单元向接受单元传送数据的帧。

CAN节点通常由三部分组成：CAN收发器、CAN控制器和MCU。

CAN总线通过差分信号进行数据传输，CAN收发器将差分信号转换为TTL电平信号，或者将TTL电平信号转换为差分信号，CAN控制器将TTL电平信号接收并传输给MCU。

**CAN总线以广播的方式从一个节点向另一个节点发送数据**，当一个节点发送数据时，该节点的CPU把将要发送的数据和标识符发送给本节点的CAN芯片，并使其进入准备状态;一旦该CAN芯片收到总线分配，就变为发送报文状态，该CAN芯片将要发送的数据组成规定的报文格式发出。此时，网络中其他的节点都处于接收状态，所有节点都要先对其进行接收，通过检测来判断该报文是否是发给自己的。由于CAN总线是面向内容的编址方案，因此容易构建控制系统对其灵活地进行配置，使其可以在不修改软硬件的情况下向CAN总线中加入新节点。

CAN总线只定义了物理层和数据链路层，而**CANopen是基于CAN总线的应用层协议**。在CANopen应用层，设备之间通过交换通信对象进行通信。CANopen的通信对象类型中有两个对象用于数据传输。它们使用两种不同的数据传输机制来实现。

   **CAN数据链路层采用短帧结构，每帧为8字节**；**CAN信息的每一帧都有CRC校验和其他检错措施**；它有效地降低了数据的错误率，并且CAN节点出现严重错误时，它会自动关闭，因此使得总线上的其他节点不受影响。CAN可以在多主机模式下工作，并且网络上的任何节点都可以随时主动向网络上的其他节点发送信息，而无需考虑主机和从机。当CAN总线上的许多节点同时发送信息时，首先发送高优先级信息，推迟低优先级信息。



##### CAN特点：

1） 多主控制。在总线空闲时，所有单元都可以发送消息（多主控制），而两个以上的单元同时开始发送消息时，根据标识符（Identifier 以下称为 ID）决定优先级。ID 并不是表示发送的目的地址，而是表示访问总线的消息的优先级。两个以上的单元同时开始发送消息时，对各消息ID 的每个位进行逐个仲裁比较。仲裁获胜（被判定为优先级最高）的单元可继续发送消息，仲裁失利的单元则立刻停止发送而进行接收工作。

3） 通信速度较快，通信距离远。最高1Mbps（距离小于40M），最远可达10KM（速率低于5Kbps）。

4） 具有错误检测、错误通知和错误恢复功能。所有单元都可以检测错误（错误检测功能），检测出错误的单元会立即同时通知其他所有单元（错误通知功能），正在发送消息的单元一旦检测出错误，会强制结束当前的发送。强制结束发送的单元会不断反复地重新发送此消息直到成功发送为止（错误恢复功能）。

6） 连接节点多。CAN 总线是可同时连接多个单元的总线。可连接的单元总数理论上是没有限制的。但实际上可连接的单元数受总线上的时间延迟及电气负载的限制。降低通信速度，可连接的单元数增加；提高通信速度，则可连接的单元数减少。



##### can通信协议格式是什么

**格式：**can id 11位，数据8字节，有起始帧，crc校验，控制位等

**报文传输过程中有：**数据帧、远程帧、错误帧、过载帧和帧间隔。

**数据帧：**用于发送单元向接收单元传送数据的帧
遥控帧：用于接收单元向具有相同ID的发送单元请求数据的帧
错误帧：用于当检测出错误时向其他单元通知错误的帧
过载帧：用于接收单元通知其尚未做好接收准备的帧
帧间隔：用于将数据帧和遥控帧与前面的帧分离开来的帧

**数据帧**

- 帧起始：表示数据帧开始的段；
- 仲裁段：表示该帧优先级的段，根据仲裁段ID码长度的不同，分为标准帧（CAN 2.0A）和扩展帧（CAN 2.0B）；
- 控制段：表示数据的字节数及保留位的段；
- 数据段：数据的内容，可发送0~8个字节的数据；
- CRC段：检查帧的传输错误的段；
- ACK段：表示确认正常接收的段；
- 帧结束：表示数据帧结束的段。

 （1）**帧起始：**标识一个数据帧的开始，用于同步，一个显性位，只有在总线空闲期间节点才能发送SOF

（2）**仲裁段（场）**：ID、RTR、IDE、SRR

仲裁段用于写明需要发送到目的CAN节点的地址、确定发送的帧类型（当前发送的是数据帧还是遥控帧），并确定发送的帧格式是标准帧还是扩展帧。

仲裁段在标准格式帧和扩展格式帧中有所不同。标准格式帧的仲裁段由11位标识符和远程发送请求位RTR组成，扩展格式帧的仲裁场由29位标识符和远程发送请求位RTR组成。

（3）**控制段（场）**：主要用于表示数据段有多少个字节

控制段由6个位组成，包括数据长度代码和两个将来作为扩展用的保留位，标准格式和扩展格式的构成有所不同。

数据长度代码指示了数据段中的字节数量。数据长度代码为4个位，在控制段里被发送，数据帧长度允许的字节数为0、1、2、3、4、5、6、7、8，其他数值为非法的。

（4）**数据段：**

数据段由数据帧中的发送数据组成，它可以为0~8字节，每字节包含了8位，首先发送最高有效位MSB，依次发送至最低有效位LSB。

数据段一般由1 ~ 8个字节（Byte）组成，来代表通信协议中相应的含义。每个字节有2个字符，分为高4位和低4位。有的数据需要相邻的2个字节组合才能表示，则需要分为高字节和低字节。

例如 ，通信协议中需要的报文（ID:0000060B ）： 0000060B 57 4e 01 7d 00 6d 11 00 。
第 1 个字节57中的5为高 4 位，7为低 4 位。第 1 、 2字节表示横向距离，而且注明Byte 1 为低字节，Byte 2 为高字节，那么解析时就应该为： 4e57。

（5）**CRC段（场）**：用于进行CRC校验

**循环冗余校验CRC**（Cyclic Redundancy Check）是数据通信领域常用的一种数据传输检错技术。通过在发送端对数据按照某种算法计算出校验码，并将得到的校验码附在数据帧的后面，一起发送到接收端。接收端对收到的数据和校验码按照相同算法进行验证，以此判断接收到的数据是否正确、完整。

标准格式与扩展格式相同
CRC 段是检查帧传输错误的段，由15 个位的CRC值和1 个位的CRC界定符(隐性分隔位)构成

CRC是根据多项式生成的CRC值，CRC的计算范围包括帧起始、仲裁段、控制段、数据段
接收方以同样的方式计算CRC值并进行比较，不一致时利用错误帧请求重新发送


（6）**ACK段（场）**：确定报文被至少一个节点正确接收

ACK段包括ACK槽位、ACK界定符位2个位

发送单元的ACK 段：发送单元在 ACK 段发送2 个位的隐性位
接收单元的ACK 段：接收到正确消息的单元在ACK 槽发送显性位，通知发送单元正常接收结束，这称作“发送ACK”或者“返回ACK”

（7）**帧结束**：7个连续的隐性位（逻辑1），表示帧结束；节点在检测11个连续的隐性位后，认为总线空闲

帧结束是由每一个数据帧和远程帧的标志序列界定的，这个标志序列由7个“隐性”位组成。



**遥控帧格式**

遥控帧的构成如下所示：

- 帧起始（SOF）：表示帧开始的段；
- 仲裁段：表示该帧优先级的段。可请求具有相同 ID 的数据帧；
- 控制段：表示数据的字节数及保留位的段；
- CRC 段：检查帧的传输错误的段；
- ACK 段：表示确认正常接收的段；
- 帧结束：表示遥控帧结束的段。

**数据帧和遥控帧的区别**

数据帧和遥控帧主要有两点区别：

- 遥控帧没有数据帧的数据段；
- 遥控帧RTR位是隐性，RTR位的极性表示了所发送的帧是数据帧（RTR位“显性”）还是远程帧（RTR位“隐性”）。所以，没有数据段的数据帧和遥控帧可通过 RTR 位区别开来。

![img](https://pic3.zhimg.com/80/v2-29d4f8ae25908c4c1462143d657f9fb2_1440w.webp)

**错误帧格式**

错误帧由错误标志（Error Flag）和错误界定符（Error Delimiter）组成。

接收节点发现总线上的报文有错误时，将自动发出活动错误标志，它是6个连续的显性位。其他节点检测到活动错误标志后发送错误认可标志，它由6个连续的隐性位组成。由于各个接收节点发现错误的时间可能不同，所以总线上实际的错误标志可能由6~12个显性位组成。

错误界定符由 8 个位的隐性位构成。当错误标志发生后，每一个CAN 节点监视总线，直至检测到一个显性电平的跳变。此时表示所有的节点已经完成了错误标志的发送，并开始发送8个隐性电平的界定符。



###### CAN总线数据传输规则

对于单个Byte，CAN总线在进行数据传输时，首先传输一个字节的高位（MSB），最后传输该字节的低位（LSB）。
一般情况下，主机厂在定义CAN总线信号的时候，都会明确定义字节的发送顺序，总共有两种顺序：1.首先发送byte0(LSB),然后byte1,byte2,…,最后byte7(MSB)。
2.首先发送byte7(MSB),然后byte6,byte5,…,最后byte0(LSB)。
其中前者发送顺序（先LSB，后MSB）是目前主机厂的主流。
下面以CAN总线报文的发送顺序为首先发送LSB，最后发送MSB的方式为前提，介绍Intel格式和Motorola格式这两种编码方式的不同。

###### Intel格式编码

当一个信号的数据长度不超过1 Byte，并且信号在一个字节内实现时，该信号的高位（S_msb）将被放在该字节的高位，信号的低位（S_lsb）将被放在该字节的低位。
当一个信号的数据长度超过1 Byte或者数据长度不超过1 Byte，但是采用跨字节的方式实现时，该信号的高位（S_msb）将被放在高字节（MSB）的高位，信号的低位（S_lsb）将被放在低字节（LSB）的低位，这样信号的起始位就是低字节的低位。



#### 面试问题：

##### CAN解码打包的格式，解码怎么去解这个码，获取这个数据什么去解开来、信号什么传输的，二进制位编码格式是什么

1. **CAN（Controller Area Network）:**

CAN是一种序列通信协议，常用于汽车和其他工业应用中，使多个设备能够在一个单一的通信线路上互相通信。

2. **CAN消息格式:**

- 标准格式

  : 该格式使用一个11位的标识符。它的结构如下：

  - 起始位：1位
  - 标识符：11位
  - RTR位（远程传输请求）：1位
  - IDE位（标识符扩展）：1位
  - r0位（预留）：1位
  - DLC（数据长度代码）：4位
  - 数据：0-8字节
  - CRC（循环冗余校验）：15位
  - ACK（应答位）：2位
  - 结束位：7位

- **扩展格式**: 该格式使用一个29位的标识符。其结构除了增加了一个18位的标识符扩展外，与标准格式相似。

3. **数据打包:**

在CAN消息中，数据是以字节为单位进行打包的。最多可以发送8字节的数据，每字节都是从最低有效位开始的。数据字段的长度可以从0到8字节。数据长度代码（DLC）字段用于指示数据字段中的字节数。

每个CAN消息包含一个标识符和数据字段。

4. **解码:**

当接收到CAN消息时，控制器会根据其标识符进行解码，它会根据标识符确定消息的来源和目的地，并将数据字段提取到相应的缓冲区。标识符不仅表示消息的优先级，还可能指示数据的来源和类型。CANopen应用程序然后可以读取这些数据并根据预先定义的信号格式进行解码。

  5.**二进制位和编码:**

- **非归零编码（NRZ）**: 在CAN通信中，数据是使用非归零编码进行编码的。这意味着逻辑"1"和逻辑"0"由线上的电压差来表示，并且它们在传输过程中不会返回到零。
- **位填充**: 为了确保接收端能够同步，CAN协议规定当连续发送5个同样的位时（例如，连续5个"1"或连续5个"0"），必须插入一个反向的位。这被称为位填充。这确保了通信中有足够的电平变化，使接收端能够维持同步。

5. **传输:**

CAN通信是基于差分传输的，使用两条线：CAN_H和CAN_L。当线上没有通信时，两条线的电压是相同的。当一台设备发送一个逻辑0（被称为主导位）时，CAN_H线的电压会升高而CAN_L线的电压会降低。反之，发送逻辑1（被称为非主导位）时，线的电压保持不变。

**7.错误检测:**

CAN协议提供了强大的错误检测和错误通知机制。主要的错误检测方法是通过CRC（循环冗余校验）。接收器会计算接收到的消息的CRC，并与消息中的CRC字段进行比较。如果不匹配，该消息就会被标记为错误。

此外，CAN还提供了其他几种错误检测方法，如帧检查、ACK错误和位错误。

**8.仲裁:**

当两个或更多的节点同时尝试发送消息时，会发生所谓的仲裁。CAN仲裁是基于非破坏性的，这意味着具有较低标识符（即较高优先级）的消息不会被破坏，而较高标识符的节点会停止发送并等待。





#### 上下位机操作流程：



1. 下位机程序监控。下位机程序有两部分，

一部分是原车载控制器的，一部分是外挂控制器的；都调成调试模式，工控机监测外挂控制器的变量信息，主要包括操作模式、关节传感器的值、接收的上位机的关节角值、关节停止运动的变量、（在辅助末端操作模式下，监测各个方向的变量）；

另外设置一台笔记本电脑监测原车载控制器的变量，主要是控制关节的几个变量。

   2.上位机软件操作。

自动控制模式操作流程：开启CAN->传感器标定->坐标系标定->选择全自动模式->设置喷浆参数和机械臂的初始关节角(看面板右侧的关节角值设置)->选择参数设置完成->仿真->启动。

辅助末端模式操作流程：开启CAN->传感器标定->坐标系标定->选择XYZ模式->操作遥控器。

3. 在传感器标定时，先运动每个关节，观察关节实际值变化方向和传感器的变化是否相同，相同则在方向栏目填入1，不同则填入-1。

传感器的值是直接通过控制面板读入。关节角实际值推荐调整到零位，这样避免了人工测量。










### 模式切换模块

根据功能需求将模式切换模块划分为单关节模式、半自动模式、全自动模式。

（1）单关节模式：工人通过遥控器单独控制每个关节，每个操纵杆和驱动器之间存在一对一的映射关系，也可以通过图5-8的滑块或输入参数控制每个关节的运动量。

![img](file:///C:\Users\彭凯\AppData\Local\Temp\ksohtml24504\wps8.jpg) 

图5-8 手动模式界面图

（2）半自动模式：也称为半自动牵引模式，首先完成各个关节的传感器标定，操纵员在三维空间遥控喷嘴末端运动，改装遥控器的手柄作用如图5-9所示。操纵员使用遥控器信号传输到上位机，通过逆运动学求解得到各个关节运动量并执行，操纵员只需关注喷嘴末端的运动轨迹。改装遥控器手柄介绍：向前推动左侧手柄，喷嘴沿着隧道中轴线的车头方向移动（X正方向）；向后推动左侧手柄，即喷嘴X负方向运动；向右推动左侧手柄，喷嘴沿着隧道中轴线的垂线右方向运动（Y正方向）；向左推动左侧手柄，即喷嘴Y方向运动；向前推动中间手柄，喷嘴沿着远离地面的垂线方向移动（Z正方向）；向后推动中间手柄，喷嘴沿着Z负方向运动；右侧手柄遥控关节5和关节6，改变喷头方向。通过混凝土喷射任务，为该模式添加了姿态保持，依据第三章的喷嘴姿态计算公式，让喷嘴与导入模型的受喷面法线对齐，当操作员不动关节5和关节6时，此时记住喷嘴所在垂直平面，对喷嘴XYZ的移动时，喷嘴会保持垂直之前的平面。

图5-9 改装遥控器手柄说明

（3）全自动模式：项目部门隧道轮廓模型CAD图或通过激光雷达采集到拱架模型首先精细，将其录入隧道曲线拟合采样模块。机械臂定位是通过激光雷达或全站仪测量后，计算出喷浆机械臂坐标系相对于受喷面坐标系的位置和姿态关系。在全自动喷浆作业前，需要确定这两个坐标系的相对关系，才能准确找到受喷面完成喷嘴的准备工作。本文需要将测量得到车体定位数据输入到上位机软件中，确定受喷浆面坐标与机械臂坐标关系。车体定位示意图如图5-10所示：

![img](file:///C:\Users\彭凯\AppData\Local\Temp\ksohtml24504\wps10.jpg) 

图5-10 机械臂定位示意图

由上图可知，机械臂坐标系原点相对世界坐标系原点偏移为d，机械臂坐标系的X轴相对于世界坐标系X轴的偏转角为θ，受喷面坐标系原点与世界坐标系的距离为L。接下来，定义偏移d的位移矩阵为Tyd，偏转角绕Z轴旋转的旋转矩阵为Rzθ，距离L的位移矩阵为TxL。那么，机械臂坐标系相对于世界坐标系的转换矩阵为TG=Rzθ×Tyd，受喷面坐标系相对于大地坐标系的转换矩阵为TS=TxL，可得机械臂坐标系相对于受喷面坐标系的转换矩阵为TM=(TS)-1TG= TxL-1 RzθTyd。

在完成隧道模型导入和机械臂定位后，就可以进行全自动喷浆模式。喷浆机械臂在全自动模式下整体技术路线，如图5-11所示：

![img](file:///C:\Users\彭凯\AppData\Local\Temp\ksohtml24504\wps11.jpg) 

图5-11 实验技术路线

## 

喷浆机械臂加装角度传感器和拉绳传感的如图5-12所示。关节1、关节2、关节3、关节5和关节6采用角度传感器测量关节转动角度，关节4采用拉绳传感器测出关节伸缩长度。



图5-12 机械臂各关节传感器安装

如图5-13所示，传感器通过CAN将信号传给上位机通讯模块，处理传感器数据并计算出当前机械臂的位置和姿态。轨迹规划模块规划出理论轨迹，通过末端期望轨迹由运动学求解每个关节的运动量，通过闭环的运动控制模块将控制信号发送SYMC控制器，并由A/D转换为比例阀阀口开度，实现机械臂各关节运动，实现末端轨迹跟踪。

![img](file:///C:\Users\彭凯\AppData\Local\Temp\ksohtml24504\wps13.png) 

图5-13 传感器和执行机构框架

喷浆机械臂采用了丹弗斯PVG32比例阀，这是一种液压负载敏感比例阀，其主要功能是通过与负载相关的控制信号来实现对执行器的精确控制。相对于与负载无关的比例控制阀，该比例阀的使用可以有效地优化机械臂的性能，并且通过结合高性能执行技术，进一步提高机械臂的效率。在这个系统中，执行器的流量和压力控制是通过主阀芯位置在PVG工作部分来实现的。使用者可以通过比例阀上的操纵杆或控制器来控制机械臂的运动。

图5-14 PVG32比例阀

PVHC是用于主阀芯的电动执行控制模块，提供100·400Hz的脉宽调制（PWM）。在24v或48v电压下，比例阀阀芯行程和电流大小关系，如图5-15所示： 

### 轨迹规划

本文实验将笔记本电脑插上zlgCAN卡与喷浆机械臂的SYMC控制器的CAN0接口连接，以控制机械臂的运动， 

**根据喷嘴末端的位姿运动轨迹，使用直线插补，计算出每个位姿点的各个关节值**，

由上位机计算出六个关节的逆解数据，采用五次多项式插值方法，将轨迹规划后运动量传递给跟踪控制程序。然后，通过CAN信号传输各个关节阀口的开度值给车载控制器，并使机械臂执行操作，以最终到达轨迹终点。在此过程中，数据采集的时间间隔为100ms。完成全自动控制实验后，我们绘制出六个关节的理论关节轨迹和实际关节轨迹的图 。

### 液压相关

根据上述关节跟踪误差分析，关节5、6采用液压马达驱动并关节质量小，跟踪性能好。

关节2、3、4是由液压缸驱动且质量较大，在运动启停阶段的惯性会使得传感器发生轻微转动和液压油缸的激振力也会让传感器读数发生变化。

特别关节4是伸缩臂，但关节4运动时需要液压油量较大会影响其他关节阀口供油。关节1是液压马达驱动齿轮运动，由于齿轮间隙误差会导致关节1运动有偏差。

在控制软件中，考虑到机械臂加工误差和外部干扰等，每个关节设置的位置精度为0.017rad，实验结果表明，该软件控制的机械臂误差都在0.017rad内，符合喷浆作业对喷浆机械臂的要求。





### 基于Qt的CAN分析仪二次开发

CAN分析仪有上位机，能够满足我们大多数情况下的使用，但当我们想扩展CAN的使用，如对消息进行封装，实现特定的执行功能时，就需要根据库文件进行二次开发。下面是使用`zlg`进行二次开发的一次尝试。

创建CANMsg类，并把ControlCAN.h加入进来：

canmsg.h

```c++
#ifndef CANMSG_H
#define CANMSG_H

#include "ControlCAN.h"
#include <QThread>
#include <QMetaType>

/*can发送类型*/
enum CAN_SEND_TYPE
{
    CAN_SEND_NORMAL = 0,//正常
    CAN_SEND_SIGNAL,//单次
    CAN_SEND_SELF,//自发自收
    CAN_SEND_SELF_SIGNAL//单次自发自收
};

/*can数据类型*/
enum CAN_DATA_TYPE
{
    CAN_DATA_INFO=0,//数据帧
    CAN_DATA_REMOTE//远程帧
};

/*是否扩展帧*/
enum CAN_EXTERN_TYPE
{
    CAN_FRAM_STANDARD=0,//标准帧
    CAN_FRAM_EXTERN//扩展帧
};

Q_DECLARE_METATYPE(PVCI_CAN_OBJ*);

/*这个类主要用来接收和发送can总线数据*/
class CANMsg : public QObject
{
public:
    CANMsg();

    BOOL open(DWORD baudIdx);
    BOOL close();
    void send(VCI_CAN_OBJ info);

protected:
    void run();

private slots:

signals:
    void sendedInfoSignal(PVCI_CAN_OBJ obj);
    void getCanData(PVCI_CAN_OBJ objs,quint32 count);//把接收到的多帧can数据发给解析线程

private:

    DWORD m_Type;
    DWORD m_Idx;
    DWORD m_Chl;
    BOOL m_IsOpen;
    VCI_CAN_OBJ recvObj[10];

};

#endif // CANMSG_H


```

mainwindow.cpp

```c++
#include "mainwindow.h"
#include "ui_mainwindow.h"

#include <QMessageBox>
#include <QDebug>

MainWindow::MainWindow(QWidget *parent) :
    QMainWindow(parent),
    ui(new Ui::MainWindow)
{
    ui->setupUi(this);

    m_pObjCanMgr = new CANMsg();
    qRegisterMetaType<PVCI_CAN_OBJ>("PVCI_CAN_OBJ");

}

MainWindow::~MainWindow()
{
    delete ui;
    delete m_pObjCanMgr;
}

void MainWindow::on_BtnOpen_clicked()
{
    if(ui->BtnOpen->text() == "打开")
    {
        DWORD baudIdx = ui->comboBoxBaud->currentIndex();
        qDebug()<<"baudIdx:"<<baudIdx<<endl;

        if(m_pObjCanMgr->open(baudIdx))
        {
            ui->BtnOpen->setText("关闭");
            ui->comboBoxBaud->setEnabled(false);
            //启动线程
            //if(!m_pObjCanMgr->isRunning())
            //{
            //    m_pObjCanMgr->start();
            //}
            connect(m_pObjCanMgr,SIGNAL(sendedInfoSignal(PVCI_CAN_OBJ)),this,SLOT(onSendCanData(PVCI_CAN_OBJ)));
            connect(m_pObjCanMgr,SIGNAL(getCanData(PVCI_CAN_OBJ,quint32)),this,SLOT(onRecvCanData(PVCI_CAN_OBJ,quint32)));
        }
    }
    else
    {
        //停止线程
        //m_pObjCanMgr->quit();
        m_pObjCanMgr->close();
        ui->BtnOpen->setText("打开");
        ui->comboBoxBaud->setEnabled(true);
        disconnect(m_pObjCanMgr,SIGNAL(sendedInfoSignal(PVCI_CAN_OBJ)),this,SLOT(onSendCanData(PVCI_CAN_OBJ)));
        disconnect(m_pObjCanMgr,SIGNAL(getCanData(PVCI_CAN_OBJ,quint32)),this,SLOT(onRecvCanData(PVCI_CAN_OBJ,quint32)));
    }
}


void MainWindow::on_BtnSend_clicked()
{
    if(ui->BtnOpen->text() == "关闭")
    {
        VCI_CAN_OBJ sendObj;
        memset(&sendObj,0,sizeof(sendObj));

        QString IdStr = ui->lineEditId->text().simplified();

        if(IdStr.isEmpty())
        {
            QMessageBox::information(this,"提示","id不能为空");
            return;
        }

        UINT canID = IdStr.toUInt(nullptr,16);
        qDebug()<< "canID:"<<canID<<endl;
        sendObj.ID = canID;

        //发送类型
        sendObj.SendType = CAN_SEND_NORMAL;
        //数据类型
        sendObj.RemoteFlag = CAN_DATA_INFO;
        //是否扩展帧
        sendObj.ExternFlag = CAN_FRAM_EXTERN;

        QString DataStr = ui->lineEditData->text().simplified();
        //数据长度
        sendObj.DataLen = DataStr.remove(QRegExp("\\s")).size()/2;
        //数据内容
        QByteArray DataByte = QByteArray::fromHex(DataStr.toUtf8());
        memcpy(sendObj.Data,DataByte.data(),sendObj.DataLen);
        m_pObjCanMgr->send(sendObj);
    }
    else
    {
        QMessageBox::information(this,"提示","未打开设备！");
    }
}

void MainWindow::onRecvCanData(PVCI_CAN_OBJ objs,quint32 count)
{
    //qDebug()<< "count"<<count<<endl;

    for(quint32 i = 0;i < count;i++)
    {
        showCanInfo(false,objs+i);
    }
}


void MainWindow::onSendCanData(PVCI_CAN_OBJ obj)
{
    showCanInfo(true,obj);
}

void MainWindow::showCanInfo(bool isSend,PVCI_CAN_OBJ obj)
{
    QString StrPrefix;
    QString StrText;
    QString StrData;

    if(isSend)
    {
        StrPrefix.sprintf("Tx:");
    }
    else
    {
        StrPrefix.sprintf("Rx:");
    }

    StrText.sprintf("id:0x%08x len:%d data:0x",obj[0].ID,obj[0].DataLen);
    for(quint32 j = 0;j < obj[0].DataLen;j++)
    {
        QString StrTmp;
        StrTmp.sprintf("%02x ",obj[0].Data[j]);
        StrData.append(StrTmp);
    }

    StrText.append(StrData);
    StrPrefix.append(StrText);

    ui->textEditInfo->append(StrPrefix);
    qDebug()<<StrPrefix<<endl;
}


void MainWindow::on_pushButtonClear_clicked()
{
    ui->textEditInfo->clear();
}



```



# 什么是can总线

CAN是控制器局域网络（ControllerAreaNetwork，CAN）的简称，是由以研发和生产汽车电子产品著称的德国BOSCH公司开发的，并最终成为国际标准（ISO11898），是国际上应用最广泛的现场总线之一。在北美和西欧，CAN总线协议已经成为汽车计算机控制系统和嵌入式工业控制局域网的标准总线，并且拥有以CAN为底层协议专为大型货车和重工机械车辆设计的J1939协议。

#### CAN总线的特点

（1）多主机方式工作：网络上任意节点可在任意时刻其他节点发送数据，通信方式灵活;

（2）网络上每个节点都有不同的优先级，可以满足实时性的要求;

（3）采用非破坏性仲裁总线结构，当两个节点同时向网络上传送信息时，优先级高的优先传送;

（4）传送方式有点对点、点对多点、点对全局广播三种;

（5）通信距离可达6km;通信速率可达1MB/s;节点数可达110个;

（6）采用的是短帧结构，每帧有8个有效字节;

（7）具有可靠的检错机制，使得数据的出错率极低;

（8）当发送的信息遭到破坏后，可自动重发;

（9）节点在严重错误时，会自动切断与总线联系，以免影响总线上其他操作。



#### CAN总线原理

CAN总线以广播的方式从一个节点向另一个节点发送数据，当一个节点发送数据时，该节点的CPU把将要发送的数据和标识符发送给本节点的CAN芯片，并使其进入准备状态;一旦该CAN芯片收到总线分配，就变为发送报文状态，该CAN芯片将要发送的数据组成规定的报文格式发出。此时，网络中其他的节点都处于接收状态，所有节点都要先对其进行接收，通过检测来判断该报文是否是发给自己的。

由于CAN总线是面向内容的编址方案，因此容易构建控制系统对其灵活地进行配置，使其可以在不修改软硬件的情况下向CAN总线中加入新节点。

#### CAN总线的应用

CAN总线在组网和通信功能上的优点以及其高性价比据定了它在许多领域有广阔的应用前景和发展潜力。这些应用有些共同之处：CAN实际就是在现场起一个总线拓扑的计算机局域网的作用。不管在什么场合，它负担的是任一节点之间的实时通信，但是它具备结构简单、高速、抗干扰、可靠、价位低等优势。CAN总线最初是为汽车的电子控制系统而设计的，目前在欧洲生产的汽车中CAN的应用已非常普遍，不仅如此，这项技术已推广到火车、轮船等交通工具中。

汽车CAN总线节点ECU的硬件设计：汽车CAN总线研发的核心技术就是对带有CAN接口的ECU进行设计，其中ECU的CAN总线模块由CAN控制器和CAN收发器构成。CAN控制器执行完整的CAN协议，完成通讯功能，包括信息缓冲和接收滤波。CAN控制器与物理总线之间需CAN收发器作为接口，它实现CAN控制器与总线之间逻辑电平信号的转换。



#### can总线是数字信号还是模拟信号

can总线是数字信号，与一般的通信总线相比，CAN总线的数据通信具有突出的可靠性、实时性和灵活性。由于其良好的性能及独特的设计，CAN总线越来越受到人们的重视。

#### 模拟信号和数字信号之间的区别

模拟信号指幅度的取值是连续的（幅值可由无限个数值表示）。时间上连续的模拟信号包括连续变化的图像（电视、传真）信号等。时间上离散的模拟信号是一种抽样信号，它是对模拟信号每隔时间T抽样一次所得到的信号，虽然其波形在时间上是不连续的，但其幅度取值是连续的，所以仍是模拟信号。

数字信号指幅度的取值是离散的，幅值表示被限制在有限个数值之内。二进制码就是一种数字信号。二进制码受噪声的影响小，易于有数字电路进行处理，所以得到了广泛的应用。

### 二、

#### **扩展帧与标准帧有什么区别？**

扩展帧与标准帧的区别在于扩展帧拥有更长字节的ID，以便能够扩展更多的CAN通讯设备。

#### **CAN通讯中的优先级？**

标准帧与扩展帧之间，标准帧的优先级会更高，扩展帧的优先级更低，相同帧类型中，往往ID更小的发送机，优先级别更高。

#### **CAN通讯的格式**

第一段：需要发送的通讯设备，先发送一个显性电平0，告诉其他通讯设备，需要开始通讯。

第二段：就是发送仲裁段，其中包括ID帧和数据帧类型，告诉其他通讯设备，需要和哪个通讯设备进行通讯，以及帧的类型，CAN通讯设备的优先级，就是由ID号决定的，往往ID号越小优先级别越高。为标准帧还是扩展帧，由仲裁段最后一位IDE位的电平决定的，IDE为显性则为标准帧，IDE为隐性则为扩展帧。

第三段，为控制段，共6位，四位储存数据段长度的信息，还有两位为保留位。

第四段：为数据段，固定长度为8个字节，先发送高位，后发送低位。

第五段，为CRC，为验证段；

第六段，为ACK为应答段，发送机发送两个隐形电平，接收机发送一个显性电平，告诉发送机，接收完成。

第七段，结束段，发送7个隐形电平。

#### **CAN通讯的终端电阻为多大，作用是什么？**

CAN通讯的终端电阻为120欧姆，在高速CAN通讯的过程中，可能会产生电感现象，对CAN通讯的高低电压产生影响，使得系统无法判别显性或者隐形电平，因此并联一个终端电阻，使得在阻抗高的时候电流可以从终端电阻流过，从而保证CAN通讯的正常运行。

#### **CAN通讯中的优先级？**

标准帧与扩展帧之间，标准帧的优先级会更高，扩展帧的优先级更低，相同帧类型中，往往ID更小的发送机，优先级别更高。

#### **如何定义错误帧？**

错误帧由错误标志段和错误定界段两段组成，接收端发现错误帧，就会发送错误标志段位为6个显性电平，由于不同接收机的发现时间不同，可能会出现6-12个显性电平，错误定界段为8个隐形电平。若错误计数器大于127时，则会发送6个隐形电平。转化为被动错误，当发送计数器计数大于255时，会关闭总线。直到检测出128次11位连续隐性位为止。

**错误帧种类有哪些？**

共5种，位错误、填充错误、CRC错误、格式错误、ACK错误。

**什么是帧间隔？**

帧间隔时间为3个显性电平，在此期间不能发送数据，发送即视为超载；之后为8个隐性电平，为延迟发送期间，如果过了这8个隐性电平时间则进入总线空闲时间。

**什么是busoff?**

Busoff就是发送错误计数器达到255次之后，CAN总线关闭的情况，CAN总线需要经过128次11个隐形电平才能恢复，或者复位。

**CAN通讯在单片机中的设置？**

设置引脚、开启时钟，其中最重要的是过滤器的设置：用于硬件过滤CAN通讯的ID，有两种模式，一种是列表模式，将传输的ID用列表逐一匹配，列表模式可以选择16位或32位的位宽ID，第二种就是掩码模式，掩码模式就是，共11位的代码，只按照条件识别有效的四位即可。

需要选择过滤器，设置过滤器ID和掩码ID，掩码ID位为1时，该位需要与过滤器进行比较。标准帧需要右移5位存储至寄存器的高11位中。




# **qt界面的哪些模块**： 

- `QtCore`：包括了核心的非GUI功能。
- `QtGui`：基本的图形库，用于处理窗口、事件、2D图形绘制等。
- `QtWidgets`：提供了标准的GUI组件，如按钮、标签、菜单等。
- `QtNetwork`：提供了TCP/IP和UDP网络编程的功能。
- `QtSQL`：提供了数据库访问功能。
- `QtMultimedia`：提供了音频、视频、摄像头等多媒体支持。

# **qt的重绘**： 

在Qt中重绘是一个常见的操作，通常通过调用`QWidget::update()`或`QWidget::repaint()`来触发重绘。在`paintEvent(QPaintEvent* event)`中，可以进行自定义绘制。以下是一个简单的例子：

```
cppCopy codevoid MyWidget::paintEvent(QPaintEvent* event) {
  QPainter painter(this);
  // 这里进行绘制操作
  painter.drawLine(10, 10, 100, 100);
}

void MyWidget::someMethod() {
  // 触发重绘
  update();
}
```

# **qt信号槽跨线程用过没 怎么设置**：

 Qt信号和槽机制支持跨线程通信。当你在一个线程中触发信号，并且该信号连接的槽在另一个线程中时，Qt会自动将信号放入接收线程的事件队列中。这是通过将连接类型设置为`Qt::QueuedConnection`或`Qt::AutoConnection`（如果信号和槽在不同线程中，`Qt::AutoConnection`将自动变为队列连接）来实现的。

以下是一个简单的示例：

```
cppCopy code// 假设你有一个在新线程中运行的对象
MyObject* obj = new MyObject;
QThread* thread = new QThread;
obj->moveToThread(thread);
connect(thread, &QThread::finished, obj, &MyObject::deleteLater);
connect(this, SIGNAL(someSignal()), obj, SLOT(someSlot()), Qt::QueuedConnection);
thread->start();
```

这里，当`someSignal()`在主线程中触发时，`someSlot()`将在新线程中调用，通过队列连接进行线程间通信。



### Headers

```c++
/最终将数据写入下位机和从下位机读出数据
//controlcentral   1.com口的参数设置   2.读取传感器数据    3.将数据写入下位机
//configdialog      1.将面板中设置好的通信格式参数送给controlcentral
//dataexchange      1.将传进来的关节角度值按照通信协议打包好为str类型输出
                   //2.将获得的传感器数据解析为关节角度值和水平仪倾角输出

//机械臂绘制
1.读取模型数据
2.改变每个关节坐标原点的位置和姿态

 //传感器函数关系  传感器的正负方向要和关节运动的正负方向相同
 
 //旋转矩阵与欧拉轴，转角，四元数之间的相互转换
                   
Headers
includes.h  
mainwindow.h
mycanthread.h
robotvisualisationwidget.h
sendmsg.h
sensorfuntion.h
trajectory_plan.h
trajectoryDisplayWindow.h （openGLWindow)
    

```

### Sources

```c++
Sources
AutoGenerateTrajectory.cpp 
//(位移关节的初始值、正运动学解、计算在隧道坐标系下的起始点的初始转角>0、将初始位置进行正和逆运动学计算，然后得到第一个点的位置，后将第二个进行偏移w得到第二个点、偏距w，先计算出单边的点，然后通过偏距将另一边点补齐、再增加一个比d更小距离的路径点,并等距w获得对应的路径点)
//做一些改进，采样点的起始位置和喷浆的起始位置不同时，能够正确处理轨迹规划问题
    //1.记录下初始位置的位置向量和方向向量
    //2.设置好轨迹规划的参数
    //3.按照参数取得u，然后计算出key point，再偏置得到另一个key point
     //计算喷浆起始点的位置向量和方向向量
    //计算喷浆起始点位置偏离采样点集的开始或结束地方的距离
 checkjoint.cpp、checksensor.cpp（关节、传感器）
 coordinatesinfo.cpp（坐标信息）
 curvefit.cpp（曲线拟合）

```

###  controlcentral.cpp

```c++
 controlcentral.cpp
      //系统参数初始化，开启系统运行，系统状态的实时反馈
    //整个系统的执行入口
    //void ControlCentral::trajectoryPlanning()
    //调用轨迹规划程序实现运动仿真
    //1.计算出所有的轨迹点
    //2.求出插值点对应的关节角度值
    //3.将robotModel中的关节值更新
    //在自动轨迹规划内设置采样点
     //刷新斜面基准下的新方向增量,傻瓜半自动
    //刷新斜面基准，根据隧道的半自动
    //Gui->setTCP(jointsPoints);最后一根连杆的末端点位置
    //Gui->setPST(pst);末端点姿态
    //更新传感器数据值，并计算与理论值的误差
    //单位是度(and mm)
    //绘制dynamic关节角曲线图
     //绘制机械臂3D图
    //1.每个关节角的位置，2.每个关节角的RPY值
    //如果是仿真则采用仿真的数据，如果是在实时控制机械臂，则采用真实关节值更新三维动画
    /* 发送数据至下位机代码*/
    //设置一个数据开始发送的标志位，启动数据发送
    // 按一定的控制频率将数据发送给下位机
    //void ControlCentral::getP1()
    //input 关节角度  关节角度值来自于传感器的值
    //output 机械臂末端点
    //再利用坐标系转换  得到隧道坐标系下的位置
    //调用逆运动学引擎提供的正向运动学计算末端点的位置
```

#### glwidget.cpp

```c++
glwidget.cpp  //显示OpenGL图形并进行交互。这个类实现了一些基本的OpenGL绘制功能和事件处理，实现图形的渲染和交互操作。
    //void GLWidget::initializeGL() OpenGL初始化设置，包括深度测试、颜色设置、光照设置等。
    //void GLWidget::paintGL()  清空缓冲区，设置视图和光照，绘制3D场景。
    // 鼠标事件处理函数：mousePressEvent、mouseMoveEvent、wheelEvent 函数处理鼠标按下、鼠标移动和滚轮事件，用于进行交互操作，如旋转视图、缩放等。

```

#### gui.cpp

```c++
gui.cpp==看gui.h理解
    //定义了一个名为 gui 的类，它继承自 QObject 类，这意味着这个类可以使用 Qt 的信号和槽机制。这个类似乎与用户界面的操作和控制有关
    //构造函数：
//explicit gui(QObject *parent = nullptr);
这是 gui 类的构造函数，可以传入一个父级对象指针作为参数。

//成员函数：这个类包含了一系列函数，用于获取和设置不同的数据，如机器坐标、传感器值、参数等。这些函数还包括设置一些状态、更新图形、触发信号等操作。

//signals 部分：在这个部分声明了一些信号，这些信号可以在特定条件下被发射（emit）。这些信号与用户界面的事件交互有关，当某些事件发生时，信号会被触发。

//slots 部分：在这个部分声明了一些槽函数，这些函数通常用于响应特定的信号。当信号被触发时，与之相关联的槽函数会被调用。
    //这些槽函数对应于用户界面中的不同操作或按钮点击事件，例如手动模式选择、自动模式选择、XYZ模式选择、参数设置点击等。
    //设置通信格式的槽函数。
    //当机器坐标滑块或文本改变时触发的槽函数。
    //传感器校准的槽函数。
    //设置坐标的槽函数。
    //开启CAN通信的槽函数。
    //设置 传感器 文本的槽函数。
    //设置 点 文本的槽函数。
    // 速度变化的槽函数。

//私有成员变量：在这个类中声明了一些指向其他类的指针，如 MainWindow*、ConfigDialog* 等，这些可能是与用户界面和其他部分相关的类。
    MainWindow* mainWindow;//MainWindow 是应用程序的主窗口，用于显示用户界面。
    ConfigDialog* configDialog;//用于显示和处理配置对话框，允许用户设置一些参数或选项
    Calibration* sensorCalibration;//用于处理传感器的校准操作
    CoordinatesSet* coordinatesSet;//用于管理和处理坐标
    curveFitParameterSet* curfitInterface;//用于管理和处理曲线拟合操作的参数

//总体来说，这个 gui 类似乎用于管理用户界面的各种操作、设置和状态。它通过信号和槽机制实现了用户界面与代码逻辑之间的通信和交互。这个类可能是在Qt框架中用于用户界面设计和操作的一部分。
```

#### inversekinematicsengine.cpp

```c++
inversekinematicsengine.cpp//执行逆向运动学计算
//setTCP(Point newTCP)末端点位置
    //设置机器人的 DH 参数
    //DH 参数是一组数值，用来定义机器人的关节和连杆之间的几何关系，从而建立起机器人的坐标系。每个关节都有自己的 DH 参数，包括以下几个值：
//a： 这是沿着前一个关节 Z 轴的偏移距离。
//alpha： 这是绕前一个关节 Z 轴旋转的角度。
//d： 这是沿着前一个关节 X 轴的偏移距离。
//theta： 这是绕前一个关节 X 轴旋转的角度。
//通过这些参数，可以构建机器人每个关节的坐标系，并通过变换矩阵来描述关节之间的变换关系。DH 参数的使用使得机器人的正向运动学（从关节角度计算末端执行器位置）和逆向运动学（从末端执行器位置计算关节角度）的计算变得相对简单，减少了运动学分析的复杂性。
    //设置末端点的姿态（ApproachVector）
    //当前计算点的前一点的关节角度值
    //根据输入的目标关节角度数组，将其进行处理后赋值给类成员变量 TargetJoint
    //设置当前计算点的前一点的机器坐标值
```

#### joystick.cpp

```c++
joystick.cpp
    // openJoy()打开手柄，开启对手柄的检测功能
    // 轮询检测手柄按键函数
paintTheroyTrajectory(QVector<Point> t)//理论曲线接口
paintpracticeTrajectory(vector<double> p)//实际曲线接口
    
main.cpp
    #include "mainwindow.h"：应用程序的主窗口类的声明。

#include <QApplication>： Qt 框架的应用程序类的头文件。

#include "inversekinematicsengine.h"：逆向运动学引擎类的声明。

#include "trajectoryinterpolator.h"：轨迹插值器类的声明。

#include "AutoGenerateTrajectory.h"：自动生成轨迹的类的声明。

#include "robotmodel.h"：机器人模型类的声明。

#include "gui.h"：图形用户界面（GUI）类的声明。

#include "controlcentral.h"：控制中心类的声明。

int main(int argc, char *argv[])：这是主函数的定义，它接受命令行参数 argc 和 argv[]。

QApplication a(argc, argv);：创建了一个 QApplication 类的实例 a，并传递命令行参数给它，用于初始化 Qt 应用程序。
```

#### mainwindow.cpp

```c++
mainwindow.cpp //初始化主窗口以及主窗口内各个部件的状态、样式、文本
//MainWindow(QWidget *parent): QMainWindow(parent), ui(new Ui::MainWindow)  调用 ui 对象的 setupUi 函数，用于将界面的设计布局应用到当前主窗口对象。
    //滑动条和数值框的值后面应该根据传感器读取的数值进行初始化
    //手动输入值的初始化,这里的值应该设置为传感器获得的数据
    //on_SettingFinish_clicked()自动化轨迹生成参数设置好后，选中设置完成，才能进行仿真和启动
    //如果从手动模式切换到自动控制模式时，参数设置完成按钮已经被按下，参数不能被读入，重新选中才能读入参数
    void MainWindow::on_Simulator_clicked()
{
    //调用函数实现仿真运动
    if(selectSimulator)
    {
        qDebug() << "simulator" << endl;
        selectSimulator = false;
        selectStart = true;
        ui->Simulator->setChecked(true);
        ui->Start->setChecked(false);
        ui->Stop->setChecked(false);
        ui->Reset->setChecked(false);
        ui->Close->setChecked(false);
        emit SimulatorButtonClicked();
    }
}
void MainWindow::on_Start_clicked()
{//读取保存好的关节组数据文件，发送到下位机
    
//包含两个 MainWindow 类的槽函数，分别是 on_Simulator_clicked 和 on_Start_clicked，用于响应名为 Simulator 和 Start 的按钮的点击事件
```

#### mycanthread.cpp

```c++
mycanthread.cpp、mycanthread_recev.cpp(看代码更好)
    //connect(clock, SIGNAL(timeout()), this, SLOT(testConnectStatus())); 连接 clock 的超时信号到 MyCanThread 类的 testConnectStatus() 槽函数。
// ReceiveCANThread();//接收CAN数据
   
    
    #include "mycanthread.h"
#include <QDebug>

MyCanThread::MyCanThread():QThread()
{
    stopped = false;//线程是否停止
    devtype=3;//USBCAN1
    devind=0;//
    res=0;//
    canind=0;//CAN
    reftype=0;//
    bool ok;
    clock = new QTimer;
    clock->setInterval(80);
    VCI_ERR_INFO vei;
    VCI_CAN_OBJ preceive[100];
    VCI_CAN_OBJ psend;
    int baud=0x10000000;//
    openSuccess = false;
    ConnectError = true;
    send_flag = false;
    receData = "38 00 00 00 00 00 00 37";
    clock->start();
    //连接 clock 的超时信号到 MyCanThread 类的 testConnectStatus() 槽函数。
    connect(clock, SIGNAL(timeout()), this, SLOT(testConnectStatus()));
}

void MyCanThread::run()
{
    while(!stopped)
    {
        ReceiveCANThread();//接收CAN数据
    }
    stopped = false;
}

void MyCanThread::testConnectStatus()
{
//    qDebug()<< "探测CAN连接状态";
    VCI_CAN_OBJ psend;
    //
    ULONG Tr;
    psend.ID=id;    //帧 ID
    psend.SendType=0;   //0 时为正常发送（发送失败会自动重发，重发最长时间为 1.5-3 秒）
    psend.RemoteFlag=0; //是否是远程帧。=0 时为为数据帧，=1 时为远程帧（数据段空）
    psend.ExternFlag=0;     //是否是扩展帧。=0 时为标准帧（11 位 ID），=1 时为扩展帧（29 位 ID）
    psend.DataLen=8;    //数据的有效字节长度，最大为8个字节
    VCI_ERR_INFO vei;
//    for (int i=0; i < 8; i++)
//    {
//        psend.Data[i]='00';
//    }

//    Tr=VCI_Transmit(devtype,devind,canind,&psend,1);    //发送数据
//    if(Tr != 1){
//        ConnectError = true;
//        qDebug()<< "发送失败";
//    }else{
//        qDebug()<< "连接正常";
//    }
    VCI_CAN_STATUS vcs;
    //DWORD dwRel;
//    if(VCI_ReadCANStatus(devtype,devind,canind,&vcs) == ERR_CMDFAILED)
//    {
//        qDebug()<< "连接失败";
//    }else{
//        qDebug()<< "连接正常";
//    }
//    if(VCI_ReadErrInfo(devtype,devind,canind,&vei) == ERR_NO_STARTUP)
//    {
//        qDebug()<< "连接失败";
//    }else{
//        qDebug()<< "连接正常";
//    }
}

void MyCanThread::stop()
{
    stopped = true;
}

void MyCanThread::ReceiveCANThread()
{
    VCI_ERR_INFO vei;
    VCI_CAN_OBJ preceive[1000];
    ULONG res = 2;
    //int count;

    res=VCI_GetReceiveNum(devtype,devind,canind);//获取缓冲区中接收但尚未被读取的帧数

    if(res<=0)//未接收到数据或操作失败
    {
//        qDebug() << "未收到下位机数据";
//        ConnectError = true;
        if(VCI_ReadErrInfo(devtype,devind,canind,&vei)!=STATUS_ERR)
        {
            //qDebug()<<QStringLiteral("3-7  1:")<<QString::number(vei.ErrCode,16);
        }
    }else{
        res=VCI_Receive(devtype,devind,canind,preceive,1000,100);//200ms     接收数据

//        qDebug()<<"Frame ID"<<res;
            if(res==4294967295)//4294967295=0xFFFFFFFF
            {
                if(VCI_ReadErrInfo(devtype,devind,canind,&vei)!=STATUS_ERR)
                {
                    qDebug()<<"Read Data failed"<<"Error Data:"<<QString::number(vei.ErrCode,16);
                }
            }else{
                //qDebug() << "接收成功";
                emit my_signal(preceive, res);
            }
    }

}
// 在 CAN 通信中传输数据
void MyCanThread::TransmitCANThread(unsigned int id,unsigned char *ch)
{
    VCI_CAN_OBJ psend;
    //
    ULONG Tr;
    psend.ID=id;    //帧 ID
    psend.SendType=0;   //0 时为正常发送（发送失败会自动重发，重发最长时间为 1.5-3 秒）
    psend.RemoteFlag=0; //是否是远程帧。=0 时为为数据帧，=1 时为远程帧（数据段空）
    psend.ExternFlag=0;     //是否是扩展帧。=0 时为标准帧（11 位 ID），=1 时为扩展帧（29 位 ID）
    psend.DataLen=8;    //数据的有效字节长度，最大为8个字节
    for (int i=0; i < 8; i++)
    {
        psend.Data[i]=ch[i];
    }

    Tr=VCI_Transmit(devtype,devind,canind,&psend,1);    //发送数据
    if(Tr==1){
//        qDebug()<< "发送成功";
    }
}

bool MyCanThread::getOpenStatus()
{
    return this->openSuccess;
}

bool MyCanThread::getConnectStatus()
{
    return this->ConnectError;
}

void MyCanThread::OpenCANThread()
{
//    bool ok;
    VCI_ERR_INFO vei;
//    VCI_CAN_OBJ preceive[100];
//    VCI_CAN_OBJ psend;
    int baud=0x01000000;    //参数有关数据缓冲区地址首指针

//    设置设备参数 canind = 0
//        if(VCI_SetReference(devtype,devind,canind,reftype,&baud)==STATUS_ERR){
//            qDebug("set reference error");
//            VCI_CloseDevice(devtype,devind);
//            return;
//        }

   //打开设备
    if(VCI_OpenDevice(devtype,devind,res)==STATUS_ERR)//
    {
        if(VCI_ReadErrInfo(devtype,devind,canind,&vei)!=STATUS_ERR)
        {
        qDebug()<<"Open failed"<<QString::number(vei.ErrCode,16);
        }else
            qDebug()<<"error";
        return;
    }else{
            qDebug()<<"open successed";
    }


    //初始化
    VCI_INIT_CONFIG init_config;
    init_config.Mode=0;//正常模式，1为只听模式
    init_config.Filter=1;//滤波方式
    //can
    //波特率是250Kbps
    init_config.Timing0=0x01;//定时器0
    init_config.Timing1=0x1c;//定时器1
    //接受的地址
    init_config.AccCode=0x10000000;//验收码
    init_config.AccMask=0xFFFFFFFF;//屏蔽码
    //
    if(VCI_InitCAN(devtype,devind,canind,&init_config)==STATUS_ERR){
        qDebug("Init Error");
        VCI_CloseDevice(devtype,devind);
        return;
    }else
        qDebug()<<"Init successed";


    //读取设备信息
    VCI_BOARD_INFO vbi;
    if(VCI_ReadBoardInfo(devtype,devind,&vbi)!=STATUS_ERR){
        qDebug()<<QStringLiteral("Count_Channel:")<<vbi.can_Num;
        qDebug()<<QStringLiteral("version_hardware:")<<vbi.hw_Version;
        qDebug()<<QStringLiteral("version_APIlib:")<<vbi.in_Version;
        qDebug()<<QStringLiteral("Num_Interrupt:")<<vbi.irq_Num;
        qDebug()<<QStringLiteral("can_Channel:")<<vbi.can_Num;
    }
    //清除缓冲区
    VCI_ClearBuffer(devtype,devind,canind);

    //启动设备
    if(VCI_StartCAN(devtype,devind,canind)==STATUS_ERR){
        qDebug()<<"start fail";
        VCI_CloseDevice(devtype,devind);
        return;
    }else{
        openSuccess = true;
        ConnectError = false;
        qDebug()<<"start successed";
    }
}

void MyCanThread::CloseCANThread()
{
    VCI_CloseDevice(devtype,devind);
    openSuccess = false;
    ConnectError = true;
    qDebug()<<"closed";
}


//        qDebug()<< "id="<< id << endl;

//        //判断ID=16#14的接收缓存，如果是38 22 22 22 22 22 22 37 则发送数据
//        if (id == 20){
//            if (QString::number(preceive[i].Data[0],16) == "38"
//                    and QString::number(preceive[i].Data[1],16) == "22"
//                    and QString::number(preceive[i].Data[2],16) == "22"
//                    and QString::number(preceive[i].Data[3],16) == "22"
//                    and QString::number(preceive[i].Data[4],16) == "22"
//                    and QString::number(preceive[i].Data[5],16) == "22"
//                    and QString::number(preceive[i].Data[6],16) == "22"
//                    and QString::number(preceive[i].Data[7],16) == "37"){
//                send_flag = true;
//                //发送一个信号，开启发送线程
//                emit startSendThread();
//            }
//        }

//        if(id == 20)
//        {
//            qDebug()<< "接收到下位机要求新一组数据的信息";
////            QString s="0x";
////            qDebug()<<QStringLiteral("FtameID:")<<s.append(QString::number(preceive[i].ID,16));
////            qDebug()<<QStringLiteral("FrameData:")<<QString::number(preceive[i].Data[0],16)<<QString::number(preceive[i].Data[1],16)
////                    <<QString::number(preceive[i].Data[2],16)<<QString::number(preceive[i].Data[3],16)
////                    <<QString::number(preceive[i].Data[4],16)<<QString::number(preceive[i].Data[5],16)
////                    <<QString::number(preceive[i].Data[6],16)<<QString::number(preceive[i].Data[7],16);
////            qDebug()<<QStringLiteral("FrameLength:")<<preceive[i].DataLen;
//        }
  }

}
```



#### pwmAndVelocity() .cpp 

用于计算电机的脉宽调制（PWM）信号和速度的函数

#### robotvisualisationwidget.cpp

```c++
robotvisualisationwidget.cpp //机器人的可视化状态
    //绘制机械臂 位置
    // jointsPostures = rot;姿态
    // 更新关节的位置和姿态
    // glRotatef(GLfloat(-(coords.f_1)*180/PI), 0.0, 1.0, 0.0);OpenGL中的旋转函数，根据角度进行旋转操作。实现机器人可视化中的姿态变换，以在三维空间中正确显示机器人的朝向
    //  glTranslatef(340.6f, 0.0, -45.25f);OpenGL中的平移函数，用于在三维空间中平移物体或模型。控制机器人可视化中的位置调整，以便在三维空间中正确放置机器人的位置
    //绘制末段点
    //绘制世界坐标系的XYZ轴
```

#### sendmsg.cpp

```c++
sendmsg.cpp  通信==用于发送 CAN 消息到下位机
   
#include "sendmsg.h"
#include <QCoreApplication>
#include <iostream>
//初始化发送和接收线程，并建立线程之间的通信连接。这样，在实例化 sendMsg 对象时，就可以在合适的时候发送和接收 CAN 消息
sendMsg::sendMsg()
{
    qRegisterMetaType<VCI_CAN_OBJ>("VCI_CAN_OBJ");
    //注册can结构体， 以便在信号和槽之间传递 CAN 数据
    qRegisterMetaType<PVCI_CAN_OBJ>("PVCI_CAN_OBJ");//注册can结构体
    qRegisterMetaType<CAN_SEND_FRAME_STRUCT>("CAN_SEND_FRAME_STRUCT");
    // 连接的 CAN 设备和通道
    deviceTye = 3;
    device = 0;
    chanel = 0;

    makedata = new makeData;
    sendThreadCAN1 = new MyCanThread;
    //处理发送 CAN 消息的线程。然后将该线程移到一个独立线程 sendThread 中执行
    sendThreadCAN1->moveToThread(&sendThread);
    processMsg = new MyCanThread_recev;
    //处理接收 CAN 消息的线程。然后将该线程移到一个独立线程 processThread 中执行
    processMsg->moveToThread(&processThread); 
 
    connect(sendThreadCAN1,SIGNAL(my_signal(PVCI_CAN_OBJ,int)),processMsg,SLOT(ReceiveCANThread(PVCI_CAN_OBJ,int)));
    //使用 connect 函数将 sendThreadCAN1 的信号连接到 processMsg 的槽，用于将接收到的 CAN 消息传递给消息处理线程。
    connect(processMsg, SIGNAL(startSendThread()), this, SLOT(transmitSendInfo()));
    //使用 connect 函数将 processMsg 的信号连接到 this 的槽 transmitSendInfo()，以便在消息处理线程接收到信息后触发一个发送信号。
}


void sendMsg::setJoints(QVector<int> joints)
{
    this->joints = joints;
}

void sendMsg::setPWM(QVector<int> pwm)
{
    this->pwm = pwm;
}

// 此处对于模式切换有改动，主要适应于半自动和手动模式，如果使用全自动模式请使用
// 根据所选的运动模式，发送控制模式的命令到下位机。根据不同的模式，发送相应的数据帧，以告知下位机切换到对应的控制模式。
void sendMsg::sendSelectMode(KinematicsMode mode)
{
    unsigned char data_from_text[8];
    unsigned int start_id = 400;//0x190
    QString start_char;
    //当控制模式为自动模式时，代码向 start_char 字符串中追加了一串表示 CAN 数据帧内容的固定十六进制数字。这个字符串可能在后续的 CAN 通信中被使用，可能是作为控制信息发送给下位机。
    if(mode == KinematicsMode::Automation)
    {
        std::cout << "Automation" << std::endl;
        start_char.append("37 01 01 01 01 01 01 38");
    }
    else if(makedata->getchangemode() == true)
    {
        std::cout << "XYZ" << std::endl;
        start_char.append("37 02 02 02 02 02 02 38");
    }
    else if(makedata->getchangemode() == false)
    {
        std::cout << "manual" << std::endl;
        start_char.append("37 03 03 03 03 03 03 38");
    }

     //告诉下位机 机械臂的控制模式
     for(int i = 0; i < 8; i++)
     {
         data_from_text[i] = hex_str_to_int((unsigned char *)start_char.section(' ',i,i).trimmed().toStdString().c_str());
     }
     \
     sendThreadCAN1->TransmitCANThread(start_id,(unsigned char *)data_from_text);
     QThread::msleep(5);
     sendThreadCAN1->TransmitCANThread(start_id,(unsigned char *)data_from_text);
          QThread::msleep(5);
sendThreadCAN1->TransmitCANThread(start_id,(unsigned char *)data_from_text);
}

void sendMsg::sensorCalibration(){
     makedata->savepos();
}

void sendMsg::sendEndFlag()
{
}
// 启动发送线程和处理线程，用于处理发送和接收 CAN 消息
void sendMsg::start()
{
    sendThreadCAN1->OpenCANThread();//打开 CAN 设备
    sendThreadCAN1->start();//启动 CAN 发送线程
    sendThread.start();//启动 sendThread 和 processThread 线程
    processThread.start();
}
void sendMsg::stopSendAndRecv()
{
    sendThreadCAN1->stop();   //停止从 CAN 总线接收数据
}
void sendMsg::close()//关闭和终止与 CAN 通信相关的线程
{
    sendThread.quit();//停止发送线程
    processThread.quit();//停止接收线程
    sendThreadCAN1->stop();//停止 CAN 接收线程
    sendThreadCAN1->CloseCANThread();//关闭 CAN 设备
}
//发送一个信号，表示开始发送消息。该信号会被连接到相应的槽函数，以实现数据的发送
void sendMsg::transmitSendInfo()
{
    emit start_sendMessage();
}
bool sendMsg::getOpenStatus()//CAN 设备是否已打开
{
    return sendThreadCAN1->getOpenStatus();
}
bool sendMsg::getConnectStatus()// CAN 设备是否已连接
{
    return sendThreadCAN1->getConnectStatus();
}





重点：




//发送 CAN 消息
void sendMsg::sendMessage()
{
    //采用事件触发机制，而不是盲等
    //如果上位机接收到下位机的信息，需要发送下一组关节角数据，则开启线程，启动数据发送

    //1.写明ID地址    2.制造数据
    //下位机发送了启动信号,则改变send_flag = true;while send_flag =true {发送数据,改变send_flag=flase,等待下位机通知再次发送} //停止发送按钮也可以直接改变send_flag=false
    QString transmit_str = QStringLiteral("ID:");
    unsigned int id = 0, start_id=16;

    unsigned char data_from_text[8];
    QStringList list1;
    QString start_char;
    start_char.append("38 33 33 33 33 33 33 37");
    //当没有发送完计算出的所有关节角组，一直等待发送

    //将关节角和 PWM 数据发送到 CAN 总线上，以便与下位机进行通信和控制。每个数据包的格式都经过了转换，然后通过 sendThreadCAN1 对象的方法进行发送
    // PWM=脉冲宽度调制数据，用于控制电机的速度或位置
    
     //先发送一帧数据给下位机，告诉下位机要开始接收数据，38 33 33 33 33 33 33 37
     for(int i = 0; i < 8; i++)
     {
         data_from_text[i] = hex_str_to_int((unsigned char *)start_char.section(' ',i,i).trimmed().toStdString().c_str());
     }
     sendThreadCAN1->TransmitCANThread(start_id,(unsigned char *)data_from_text);
    //将 data_from_text 数组中的数据发送到 CAN 总线上，使用 ID start_id

     unsigned int pwm_ID = 10;//构造 PWM 数据的 ID
     for (int i=0;i < 6;i++) {
        id = static_cast<unsigned int>(i + 1);        //ID: 1~6
        pwm_ID++;
        sendData = makedata->rawData2sendData(joints[i]);//将关节角数据转换为发送给下位机的格式
        sendData_pwm = makedata->rawData2sendData(pwm[i]);//将 PWM 数据转换为发送给下位机的格式
        //qDebug() << "sendData " << sendData << endl;
        //makedata->receData2realData(sendData);
        //qDebug() << "realJoint " << makedata->getRealValue();

        //发送关节角
        for(int j = 0; j < 8; j++)
        {
            data_from_text[j] = hex_str_to_int((unsigned char *)sendData.section(' ',j,j).trimmed().toStdString().c_str());
        }
        sendThreadCAN1->TransmitCANThread(id,(unsigned char *)data_from_text);//将关节角数据发送到 CAN 总线上，使用相应的 ID

        //发送PWM
        for (int j = 0; j < 8; j++) {
            data_from_text[j] = hex_str_to_int((unsigned char *)sendData_pwm.section(' ',j,j).trimmed().toStdString().c_str());
        }
        sendThreadCAN1->TransmitCANThread(pwm_ID,(unsigned char *)data_from_text);

        sendData.clear();
        sendData_pwm.clear();
    }


}
// 发送 PWM 数据到下位机的代码块
void sendMsg::sendMessages(){
     //作为新的发送器
    QString sendData_pwm;


    unsigned char data_from_text[8];

     unsigned int pwm_ID = 0x191;
    sendData_pwm = makedata->rawData6sendData(pwm);
//    qDebug() << "sendData_pwm:" << sendData_pwm<<endl;
//从 pwm 数组中获取 PWM 数据，并将其转换为一种适合发送给下位机的格式

        //发送PWM
        for (int j = 0; j < 8; j++) {
            data_from_text[j] = hex_str_to_int((unsigned char *)sendData_pwm.section(' ',j,j).trimmed().toStdString().c_str());
        }
        sendThreadCAN1->TransmitCANThread(pwm_ID,(unsigned char *)data_from_text);
//将转换后的 PWM 数据以指定的 ID (pwm_ID) 发送到 CAN 总线上
        sendData.clear();
        sendData_pwm.clear();
}

//发送 XYZ 模式的消息。它包括创建用于发送数据的缓冲区、将 PWM 数据转换为适合发送的格式，以及通过 sendThreadCAN1->TransmitCANThread 方法将数据发送到 CAN 总线上。
void sendMsg::sendXYZmess(){
    QString sendData_pwm;


    unsigned char data_from_text[8];

     unsigned int pwm_ID = 0x191;
    sendData_pwm = makedata->rawData6sendData(pwm);
//    qDebug() << "sendData_pwm:" << sendData_pwm<<endl;


        //发送PWM
        for (int j = 0; j < 8; j++) {
            data_from_text[j] = hex_str_to_int((unsigned char *)sendData_pwm.section(' ',j,j).trimmed().toStdString().c_str());
        }
        sendThreadCAN1->TransmitCANThread(pwm_ID,(unsigned char *)data_from_text);

        sendData.clear();
        sendData_pwm.clear();
}


//发送自动模式的消息。与 sendXYZmess() 函数类似，它也包括创建缓冲区、转换 PWM 数据并发送数据
void sendMsg::sendAUTOmess(){
    QString sendData_pwm;


    unsigned char data_from_text[8];

     unsigned int pwm_ID = 0x191;
    sendData_pwm = makedata->rawData6sendData(pwm);
//    qDebug() << "sendData_pwm:" << sendData_pwm<<endl;


        //发送PWM
        for (int j = 0; j < 8; j++) {
            data_from_text[j] = hex_str_to_int((unsigned char *)sendData_pwm.section(' ',j,j).trimmed().toStdString().c_str());
        }
        sendThreadCAN1->TransmitCANThread(pwm_ID,(unsigned char *)data_from_text);

        sendData.clear();
        sendData_pwm.clear();
}
//根据十六进制表示将字符转换为整数
int sendMsg::hex_str_to_int(unsigned char *ch)
```

#### timingtransmitter.cpp

```c++
timingtransmitter.cpp 定时发送控制信号
#include "timingtransmitter.h"
#include "QDebug"
#include "math.h"
#include <QDateTime>
#define PI 3.14159265358979323846
    //TimingTransmitter 类是一个继承自 QThread 的类，用于实现定时发送控制信号
TimingTransmitter::TimingTransmitter():QThread()
{
    alarm = new QTimer;
    alarm->setInterval(25);//定时器的间隔为 25 毫秒
    alarm->start();
    connect(alarm,SIGNAL(timeout()),this,SLOT(onTransmitter()));
    //连接定时器的 timeout 信号到 onTransmitter 槽函数，实现定时触发
}
//设置控制模式，即切换控制模式。根据不同的模式，定时器的触发行为可能会改变
void TimingTransmitter::modeselect(KinematicsMode kinematicsmode){
    this->kinematicsmode = kinematicsmode;
}


//onTransmitter 槽函数是定时触发的主要处理逻辑。根据当前的控制模式，它会发射不同的信号，即 start_XYZ 或 start_auto，从而触发相关的控制操作
void TimingTransmitter::onTransmitter(){

    if((this->kinematicsmode == KinematicsMode::manual)) {
//        qDebug() << "定时器发送成功！" << "manual" << endl;
    }
    else if(this->kinematicsmode == KinematicsMode::Automation){
//        qDebug() << "定时器发送成功！" << "Automation" << endl;
        AUTOcontrol();
    }else {
//        qDebug() << "定时器发送成功！" << "XYZ" << endl;
        XYZcontrol();
    }
}

//在控制模式为 XYZ 时执行的操作。它会通过 emit start_XYZ() 发射一个信号，可能是用来触发某些与 XYZ 控制相关的操作
void TimingTransmitter::XYZcontrol(){
        emit start_XYZ();
}


//控制模式为 Automation 时执行的操作。它会通过 emit start_auto() 发射一个信号，可能是用来触发某些与自动化控制
void TimingTransmitter::AUTOcontrol(){
            emit start_auto();
```

#### sensorFunction.cpp

```C++ 
sensorFunction.cpp //对传感器数据和关节数据进行处理
//设置传感器数据、关节数据和一个参数 k
//getJointValue(int sensor)：函数用于返回一个传感器值作为关节数
    //fixFun()：修正操作，根据传感器数据计算 tmpJoint 并更新 offset 的值  == 根据基准值调整传感器数据
```

#### trajectory_plan.cpp

```c++
trajectory_plan.cpp、trajectoryDisplayWindow.cpp、trajectoryinterpolator.cpp
    // 直线路径插补相关函数
    // 将姿态插补和直线路径插补封装在一起
    // 窗口类，用于显示轨迹（或路径）的信息
    
    
    //生成机械臂运动的轨迹点和姿态，可以根据提供的点、姿态和参数进行固定步长插值或时间插值。这些插值方法可以用于控制机械臂在特定路径上的运动
    //轨迹插值器。这个类的作用是计算和生成机械臂运动的轨迹点和姿态，以便控制机械臂在特定路径上进行运动。它可以使用固定步长插值或时间插值的方法来生成轨迹点和姿态，以实现机械臂的运动规划和控制。
trajectoryinterpolator.h
  //class trajectoryinterpolator : public QObject
    //继承自 QObject，具有信号和槽的特性，用于轨迹的插值计算和规划
    成员函数：
void setStepLength(double step);：设置固定步长插值中的步长。
void setStartPoint(Point p);：设置轨迹的起始点。
void setFinishPoint(Point p);：设置轨迹的结束点。
void setStartPosture(ApproachVector posture);：设置轨迹起始点的姿态。
void setFinishPosture(ApproachVector posture);：设置轨迹结束点的姿态。
void setTimeInterPara(double v, double r, double w, double n_segment);：设置时间插值所需的参数，包括机械臂喷头移动速度、过渡圆弧半径、轨迹宽度和弧线段数量。
void stepInterpolator();：进行固定步长插值，用于生成轨迹点，但不能控制喷头的移动速度。
void timeInterpolator();：进行时间插值，计算生成插值点，可以控制喷头的移动速度。
void calculate();：根据设置的参数计算轨迹和姿态。
QVector<Point> getTrajectory();：获取生成的轨迹点。
QVector<ApproachVector> getPosture();：获取生成的轨迹姿态。
    
```

#### trajectoryDisplayWindow.cpp

```c++

trajectoryDisplayWindow.cpp
    //承自 QGLWidget 的自定义 OpenGL 窗口类，用于在 GUI 中显示 3D 图形
OpenGLWindow :: OpenGLWindow(QWidget *parent)
       :QGLWidget(parent)
{
    m_scloe = -30;//缩放参数
    m_rotx = 5;//绕 X、Y、Z 轴的旋转角度，用于控制模型的旋转效果
    m_roty = 5;
    m_rotz = 5;
    m_count = 0;//跟踪绘制次数
    m_isize = 0;//控制渲染的某些元素的大小
    m_anglev = 8;//控制渲染中的角度

}

OpenGLWindow::~OpenGLWindow()
{
}

void OpenGLWindow::initializeGL()
{
    glClearColor(1, 1, 1, 1.0);//GL背景色=白色
    glShadeModel(GL_SMOOTH);//着色模式==平滑着色模
    glEnable(GL_DEPTH);//启用深度测试。深度测试是OpenGL中用来控制绘制顺序的一个重要机制，确保远处的对象不会遮挡近处的对象。
}

//用于在OpenGL窗口大小改变时重新配置OpenGL的视口设置和投影矩阵配置，以适应新的窗口尺寸
void OpenGLWindow::resizeGL(int w, int h)
{
    if (h == 0)//防止height为0
    {
        h = 1;
    }
    glViewport(0, 0, w, h);//重置当前的视口 ==指定了渲染窗口中可以用来显示图形的区域

    glMatrixMode(GL_PROJECTION);//选择投影矩阵  ==选择投影矩阵模式，以便设置透视投影矩阵
    glLoadIdentity();//重置投影矩阵
    gluPerspective(45.0, w / h, 0.1, 1000.0);//建立透视投影矩阵 ==四个参数：视场角、宽高比、近裁剪面和远裁剪面

    glMatrixMode(GL_MODELVIEW);//选择模型观察矩阵
    glLoadIdentity();//重置模型观察矩阵

}
//OpenGL窗口中绘制3D场景，包括坐标轴、网格、圆柱体等元素。
//这些元素的绘制是通过OpenGL的绘制函数和变换函数来实现的，以达到在3D空间中可视化显示场景的效果
void OpenGLWindow::paintGL()
{
    //qDebug() << "paintGL with t_vetorx.size() = " << t_vetorx.size();
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);//清除屏幕和深度缓存
    glLoadIdentity();//重置当前的模型观察矩阵。在glLoadIdentity()调用之后，函数返回之前，添加代码来创建基本的形
                     //目前所做的就是将屏幕清除成前面所决定的颜色，清除深度缓存并且重置场景，仍然没绘制任何东西。
                     //glPolygonMode(GL_FRONT_AND_BACK ,GL_LINE );
                     //前后面，填充方式（点point、线line、FILL）

    //OpenGL 的平移、旋转函数显示窗口 
    glTranslatef(-1, -4,m_scloe);
    glRotatef(m_rotx, 1.0, 0.0, 0.0);
    glRotatef(m_roty-40, 0.0, 1, 0.0);
    glRotatef(m_rotz-2, 0.0, 0.0, 1);
    GLUquadricObj *objCylinder = gluNewQuadric();//创建一个新的二次曲面对象，这里用于绘制圆柱体

    int pt = 12;//xyz轴的长度
    int num = 12;//控制网格疏密

    //Y轴
    glColor3f(0, 1, 0);
    glLineWidth(5);
    glBegin(GL_LINES);//开始绘制线段。
        glVertex3f(0,0,0);// 绘制线段的顶点坐标
        glVertex3f(pt,0,0);
    glEnd();              // 结束绘制线段

    //网格
    //Y
    glPushMatrix();

    glColor3f(0.8, 0.8, 0.8);
    glTranslatef(-m_isize, -m_isize, -m_isize);
    GLGrid(0, 0, 0, pt, 0, pt, num);//绘制网格
    glPopMatrix();

    glPushMatrix();
    glColor3f(0, 1, 0);
    glTranslatef(pt, -m_isize, -m_isize);
    glRotatef(90, 0, 1, 0.0);
    gluCylinder(objCylinder, 0.5, 0.0, 1, 100, 1);//绘制圆柱体
    glPopMatrix();

    //Z
    glPushMatrix();
    glTranslated(-m_isize, m_isize, -m_isize);
    glRotatef(-90, 1.0, 0.0, 0.0);
    glColor3f(0.8, 0.8, 0.8);
    GLGrid(0, 0, 0, pt, 0, pt, num);
    glPopMatrix();

    glPushMatrix();
    glColor3f(0, 0, 1);
    glTranslatef(-m_isize, pt, -m_isize);
    glRotatef(-90, 1, 0, 0);
    gluCylinder(objCylinder, 0.5, 0.0, 1, 100, 1);
    glRasterPos3d(5.0,5.0,5.0);//设置光栅位置，用于在场景中绘制文字
    glPopMatrix();

    //X
    glPushMatrix();                           //push,pop栈使用
    glTranslatef(-m_isize, -m_isize, -m_isize);
    glRotatef(90, 0.0, 0.0, 1.0);              //网格所在空间位置
    glColor3f(0.8, 0.8, 0.8);                  //网格颜色
    GLGrid(0, 0, 0, pt, 0, pt, num);
    glPopMatrix();

    glPushMatrix();
    glColor3f(1, 0, 0);                         //箭头颜色
    glTranslatef(-m_isize, -m_isize, pt);
    glRotatef(90, 0, 0, 1);
    gluCylinder(objCylinder, 0.5, 0.0, 1, 100, 1);//小箭头
    glPopMatrix();

    LabelXYZ();

/*
 * 调用时只需将下两个接口进行调用，出入具体的嵌套接口即可
*/
    updateTheroyTraj();
    updatePracticeTraj();
}

//绘制坐标轴标签的函数 LabelXYZ()，它绘制了X、Y和Z轴上的字母标签
void OpenGLWindow::LabelXYZ()
{
    /*********************X,Y,Z字母***************************************/
        GLfloat size2 = 2.0;
        glLineWidth(size2);
        //通过一系列的数组定义（a1、b1、c1、a2、b2、c2、a3、b3、c3）定义了每个轴上的标签的位置
        double a1[3]= {6.5,6.75,7};
        vector<double> n1_vetorx(a1,a1+3);
        double b1[3]= {13.0,13.0,13.0};
        vector<double> n1_vetory(b1,b1+3);
        double c1[3]= {-0.75,-0.5,-0.25};
        vector<double> n1_vetorz(c1,c1+3);
        //调用OpenGL的绘制函数来绘制每个轴上的标签
        if (n1_vetorx.size() >= 2) {                              //X
            glBegin(GL_LINE_STRIP);//开始绘制线段，依次添加顶点坐标
            glTranslatef(-m_isize, -m_isize, -m_isize);
            glColor3f(1, 0, 0);//置颜色为红色

     //Y
            glBegin(GL_LINE_STRIP);
            glTranslatef(-m_isize, -m_isize, -m_isize);//对每个轴上的标签进行平移，以适应场景中的位置
            glColor3f(0, 1, 0);

            glVertex3f(n2_vetorx[0], n2_vetorz[0], n2_vetory[0]);
            glEnd();
            glFlush();
        }

        double a3[3]= {0.0,0.0,0.5};
        vector<double> n3_vetorx(a3,a3+3);
        double b3[3]= {0.0,0.0,0.0};
        vector<double> n3_vetory(b3,b3+3);
        double c3[3]= {12.75,13.00,13.25};
        vector<double> n3_vetorz(c3,c3+3);
        if (n3_vetorx.size() >= 2) {                              //Z
            glBegin(GL_LINE_STRIP);
            glTranslatef(-m_isize, -m_isize, -m_isize);
            glColor3f(0, 0, 1);

            glVertex3f(n3_vetorx[2], n3_vetorz[0], n3_vetory[0]);
            glVertex3f(n3_vetorx[0], n3_vetorz[0], n3_vetory[0]);
            glVertex3f(n3_vetorx[2], n3_vetorz[2], n3_vetory[0]);
            glVertex3f(n3_vetorx[0], n3_vetorz[2], n3_vetory[0]);


            glEnd();
            glFlush();
        }
}

//用于处理鼠标滚轮事件和鼠标按下事件
//用户通过鼠标滚轮来控制场景的缩放效果
void OpenGLWindow::wheelEvent(QWheelEvent *event)//处理鼠标滚轮事件
    update();//实现场景的缩放效果。调用 update() 来重新绘制场景
}

void OpenGLWindow::mousePressEvent(QMouseEvent *event)//处理鼠标按下事件。当用户按下鼠标按钮时，会触发这个函数
void OpenGLWindow::mouseMoveEvent(QMouseEvent *event)


//绘制OpenGL中的网格
void OpenGLWindow::GLGrid(float pt1x, float pt1y, float pt1z, float pt2x, float pt2y, float pt2z, int num)

{
    const float _xLen = (pt2x - pt1x) / num;
    const float _yLen = (pt2y - pt1y) / num;
    const float _zLen = (pt2z - pt1z) / num;
    glLineWidth(0.01f);
    glLineStipple(1, 0x0303);//线条样式

    glBegin(GL_LINES);
    glEnable(GL_LINE_SMOOTH);

    int xi = 0;
    int yi = 0;
    int zi = 0;
    //绘制平行于X的直线
    for (zi = 0; zi <= num; zi++) {
        float z = _zLen * zi + pt1z;
        for (yi = 0; yi <= num; yi++) {
            float y = _yLen * yi + pt1y;
            glVertex3f(pt1x, y, z);//指定每个线段的两个端点坐标，从而构成直线段
            glVertex3f(pt2x, y, z);

        }
    }
    //绘制平行于Y的直线
    for (zi = 0; zi <= num; zi++) {
        float z = _zLen * zi + pt1z;
        for (xi = 0; xi <= num; xi++) {
            float x = _xLen * xi + pt1x;
            glVertex3f(x, pt1y, z);
            glVertex3f(x, pt2y, z);
        }
    }
    //绘制平行于Z的直线
    for (yi = 0; yi <= num; yi++) {
        float y = _yLen * yi + pt1y;
        for (xi = 0; xi <= num; xi++) {
            float x = _xLen * xi + pt1x;
            glVertex3f(x, y, pt1z);
            glVertex3f(x, y, pt2z);
        }
    }
    glEnd();
}


//用于绘制理论轨迹
void OpenGLWindow::updateTheroyTraj()
//绘制实际轨迹
void OpenGLWindow::updatePracticeTraj()

```























####  `OpenGLWindow` 的类

这段代码展示了一个名为 `OpenGLWindow` 的类，该类继承自 `QGLWidget` 并用于创建OpenGL窗口。它用于显示3D图形并处理相关的OpenGL绘图和事件。

```c++
using namespace std;  // 使用标准C++命名空间

class OpenGLWindow : public QGLWidget
{
    Q_OBJECT  // 使用Qt元对象宏，启用Qt的信号和槽机制
public:
    explicit OpenGLWindow(QWidget *parent = nullptr);  // 构造函数
    ~OpenGLWindow();  // 析构函数
    void GLGrid(float pt1x, float pt1y, float pt1z, float pt2x, float pt2y, float pt2z, int num);  // 绘制网格的函数

private:
    void LabelXYZ();  // 标注坐标轴的函数

protected:
    void initializeGL();  // 初始化OpenGL上下文
    void resizeGL(int w, int h);  // 在窗口大小变化时调整OpenGL视口
    void paintGL();  // 绘制OpenGL场景
    void mousePressEvent(QMouseEvent *event);  // 处理鼠标按下事件
    void mouseMoveEvent(QMouseEvent *event);  // 处理鼠标移动事件
    void wheelEvent(QWheelEvent *event);  // 处理鼠标滚轮事件

private:
    void updateTheroyTraj();  // 更新理论轨迹
    void updatePracticeTraj();  // 更新实际轨迹

    // 一些私有成员变量，用于存储不同数据
};
这个类似乎定义了一个用于展示3D图形的OpenGL窗口，其中包含了许多用于初始化、绘制、处理事件等的函数。这个类可能会在Qt框架中使用，用于创建一个3D可视化的窗口，并通过继承自 QGLWidget 来实现与OpenGL的交互。
```

