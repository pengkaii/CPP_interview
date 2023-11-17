# WebServer:

## 面试问题

### 1、项⽬介绍

#### 介绍⼀下你的项⽬

**这个项⽬**是我在学习计算机⽹络和Linux socket编程过程中独⽴开发的轻量级Web服务器，服务器的⽹络模型是主从reactor加线程池的模式，IO处理使⽤了非阻塞IO和IO多路复⽤技术，**服务器具备处理多个客户端的http请求**。

**项⽬中的⼯作**可以分为两部分，

⼀部分是服务器⽹络框架（主从reactor+线程池模式）、异步日志系统，有限状态机等⼀些基本系统的搭建，

另⼀部分是为了**提⾼服务器性能所做的⼀些优化**，⽐如缓存机制的LFU算法的搭建。

最后还对⽹络框架进行了压力测试，使⽤webbench创建1000个进程对服务器进行60s并发请求，测试结果表明，对于短连接的QPS为2万，对于长连接的QPS为5.2万。

通过这个项⽬，我熟悉了Linux环境下的编程，并在后期优化的过程中了解了⼀些服务端性能调优的⽅法。（收获）

**服务器编程基本框架：**

模块功能： I/O处理单元 、处理客户连接，读写网络数据 。

逻辑单元： 业务进程或线程。

网络存储单元： 数据库、文件或缓存

 请求队列：各单元之间的通信方式



##### 主从Reactor原理

主从Reactor多线程模型中，mainReactor只处理连接事件，读写事件交给subReactor来处理。

**mainReactor 由主线程运行，他作用如下：通过epoll监听listenfd的可读事件，当可读事件发生后，调用accept函数获取clientfd，然后随机取出一个subReactor，将cliednfd的读写事件注册到这个subReactor的epoll上即可。mainReactor只负责建立连接事件，用一个线程来处理就好,不进行业务处理，也不关心已连接套接字的IO事件。subReactor通常有多个，处理读写事件的subReactor个数一般和CPU数量相等，每个subReactor由一个线程来运行。其注册clientfd的读写事件，当发生IO事件后，需要进行业务处理,业务逻辑还是由线程池来处理。**



**事件处理过程：**

基本概念
Reactor：把IO事件分配给对应的handler处理
Acceptor：处理客户端连接事件
Handler：处理非阻塞的任务

梳理下基于主从Reactor多线程模型的事件处理过程：
Reactor主线程对象通过epoll监听连接事件，通过Acceptor处理连接事件，当Acceptor处理连接事件后，主reactor将连接分配给子Reactor，子Reactor将连接加入到连接队列进行监听，并创建handler进行各种事件处理。当有新事件发生时，子reactor就会对用对应的handler处理，handler读取数据后，分发给后面的worker线程处理，worker线程池分配独立的worker线程进行处理并返回结果，handler收到结果后再讲结果返回给客户端。



**其他并发模型：**

- **单Reactor多线程**进行处理，其主线程(I/O处理单元)只负责监听文件描述符上是否有事件发生，有的话立即通知工作线程(逻辑单元 )，将socket可读写事件放入请求队列，交给工作线程处理，即读写数据、接受新连接及处理客户请求均在工作线程中完成。通常由同步I/O实现（epoll_wait）
- **proactor模式**中，主线程和内核负责处理读写数据、接受新连接等I/O操作，工作线程仅负责业务逻辑，如处理客户请求。通常由异步I/O实现(aio_read/aio_write)。

##### 为什么用主从reactor+线程池

将IO操作和业务处理解耦,subreactor负责读写数据，由线程池进⾏业务处理

**主从reactor就是将reactor分为主reactor和从reactor，主reactor中只负责连接的建⽴和分配，读取请求、业务处理、返回响应等耗时的操作均在从reactor中处理，能够有效地应对⾼并发的场合。**

① 分工明确 , 互不干扰 :主反应器 ( MainReactor ) 在主线程中运行 , 只负责客户端的连接 ;多个子反应器 ( SubReactor ) 分别在对应的子线程中运行 , 负责每个客户端连接的数据交互 , 与业务逻辑调度 ; 这里的子反应器和对应的子线程有多个 ;
② 主线程与子线程交互简单 :主线程中 , 主反应器将接受者 ( Acceptor ) 建立好的连接交给子线程中的子反应器 , 除此之外 , 主线程与子线程没有其它的数据交互 ;



**监听多少个端⼝**（HTTP80端口），**怎么监听**（用epoll）

你刚才讲的那些性能调优的⼿段？

从**程序本身的优化和系统参数的优化**两⽅⾯去讲



##### 网络框架优点、代码规范

**优点：**

**I/O任务和计算任务解耦**，避免计算密集型连接占⽤subReactor导致无法响应其他连接，可扩展性好）还了解哪些其他⽹络框架？Proactor

**项⽬代码规范**（从模块解耦、命名空间、命名规范）

- **模块解耦：**
- 在实际编码中，可以将功能相关的代码组织在独立的模块中，并通过接口或抽象类来定义模块之间的通信方式。通过将模块解耦，可以减少代码之间的依赖关系，提高代码的灵活性和可维护性。
- **命名空间：**
- 使用命名空间可以避免名称冲突，提供更好的代码组织和管理。在实际编码中，我们应该为每个模块、库或项目定义适当的命名空间，以便将其内部的标识符隔离开来，避免与其他代码发生冲突。
- **命名规范：**
- 良好的命名规范有助于代码的可读性和可维护性。在实际编码中，我们应该遵循一致的命名规范，给变量、函数、类、文件等标识符取有意义的名称，避免使用不清晰或含糊的命名。

```c++
// 使用驼峰命名法
int userCount;
void getUserInfo();

// 使用帕斯卡命名法
class UserManager {
    // ...
};
```





##### 优化、难点

**优化（减少系统调用）：**

- 减少系统调⽤（线程池、缓存机制；零拷贝；读写锁）

- **程序本身：**
     **减少程序等待 IO 的事件**：非阻塞 IO + IO 多路复用。
      **设计⾼性能⽹络框架**，同步 IO（主从 reactor + 线程池）。同步I/O：内核向应用程序通知就绪事件，由应用程序自身来完成I/O的读写操作

- **【减少系统调⽤】**

  1、避免频繁申请/释放内存：线程池、缓存机制。
  2、对于文件发送，使⽤零拷贝函数 sendFile() 来发送，避免拷贝数据到用户态。
  ​ 3、尽量减少锁的使用，如果需要，尽量减小临界区（⽇志系统和线程池）。

- **系统参数调优：**
   1、最大文件描述符数（⽤户级和系统级），
    2、tcp 连接的参数（半连接/连接队列的⻓度、tcp syncookies）。



**难点：**

项⽬中我主要的⼯作可以分为两部分：

1. ⼀部分是服务器⽹络框架、异步⽇志系统等⼀些基本系统的搭建，这部分的难点主要就是技术的理解和选型，以及将⼀些开源的框架调整后应⽤到我的项⽬中去。
2. 另⼀部分是为了**提⾼服务器性能所做的⼀些优化**，⽐如**缓存机制**等⼀些额外系统的搭建。这部分的难点主要是找出服务器性能瓶颈，然后结合⾃⼰的想法去突破这个瓶颈，提⾼服务器的性能。

**解决：**

​      ⼀⽅⾯是对不同的技术理解不够深刻，难以选出最合适的技术框架。**这部分的话我主要是反复阅读作者在GitHub提供的⼀些技术⽂档，同时也去搜索⼀些技术对⽐的⽂章去看，如果没有任何相关的资料我会尝试去联系作者。**

​      另⼀⽅⾯是编程期间遇到的困难，在代码编写的过程中由于⼯程能⼒不⾜，程序总会出现⼀些bug。**这部分的话我⾸先是通过⽇志去定位bug，然后推断bug出现的原因并尝试修复，如果是⾃⼰⽬前⽔平⽆法修复的bug，我会先到⽹上去查找有没有同类型问题的解决⽅法，然后向同学或者直接到StackOverflow等⼀些国外知名论坛上求助。**



#### 设计模式（单例模式）

**设计模式（单例模式）：**

- 单例模式：在**线程池**中都有使⽤到。用于确保一个类只有一个实例，并提供一个全局访问点来访问该实例。
- 单例模式可以分为懒汉式和饿汉式，两者之间的区别在于**创建实例的时间不同**：
- **饿汉式（线程安全）**：指系统⼀运⾏，就初始化创建实例，当需要时，直接调⽤即可，这个单例实例在整个程序的生命周期内一直存在。（本身就线程安全，没有多线程的问题）
- **懒汉式（不安全）**：指系统运⾏中，实例并不存在，只有当需要使⽤该实例时，才会去创建并使⽤实例，表示在需要时才创建实例。（这种⽅式要**考虑线程安全**）。
- 在懒汉模式中，需要使用互斥锁来确保在多线程环境下只创建一个实例。首次访问时会检查实例是否为空，如果为空，则使用互斥锁保证只有一个线程创建实例。

```c++
饿汉模式
class Singleton {
private:
    static Singleton* instance;  // 静态成员变量，用于保存唯一实例
    Singleton() {}  // 私有构造函数，防止外部实例化
public:
    static Singleton* getInstance() {
        if (instance == nullptr) {
            instance = new Singleton();
        }
        return instance;
    }
};
Singleton* Singleton::instance = nullptr;  // 初始化静态成员变量
int main() {
    Singleton* obj1 = Singleton::getInstance();
    Singleton* obj2 = Singleton::getInstance();
    // obj1 和 obj2 指向相同的实例
    if (obj1 == obj2) {
        std::cout << "Same instance" << std::endl;
    }
    return 0;
}
```



```c++
懒汉模式：
#include <iostream>
#include <mutex>
class Singleton {
private:
    static Singleton* instance;  // 静态成员变量，用于保存唯一实例
    static std::mutex mutex;     // 用于多线程同步
    Singleton() {}  // 私有构造函数，防止外部实例化
public:
    static Singleton* getInstance() {
        if (instance == nullptr) {
            std::lock_guard<std::mutex> lock(mutex); // 加锁，确保只创建一个实例
            if (instance == nullptr) {
                instance = new Singleton();
            }
        }
        return instance;
    }
};
Singleton* Singleton::instance = nullptr;  // 初始化静态成员变量
std::mutex Singleton::mutex;  // 初始化互斥锁
int main() {
    Singleton* obj1 = Singleton::getInstance();
    Singleton* obj2 = Singleton::getInstance();
    // obj1 和 obj2 指向相同的实例
    if (obj1 == obj2) {
        std::cout << "Same instance" << std::endl;
    }
    return 0;
}

```



**服务器申请了域名吗**

没有申请域名，但阿⾥云服务器上⾃带⼀个公⽹ip，**可以直接通过公⽹ip来访问服务器**。



但是本地测试的话，我⼀般使⽤本地回环ip，127.0.0.1来进⾏访问。能 ping 通 127.0.0.1 说明本机的⽹卡和IP协议安装都没有问题。



#### 项⽬⾯向对象体现、noncopyable、enable_shared

**c++⾯向对象特性有封装、继承、多态。**
**封装**：我在项⽬中将各个模块使⽤类进⾏封装，⽐如连接⽤ httpconnection/ftpconnection 类来封装，⽇志就⽤ log 类来封装，将类的属性私有化，⽐如请求的解析状态，并且对外的接⼝设置为公有，⽐如连接的重置，不对外暴露⾃身的私有⽅法，⽐如读写的回调函数等。还有⼀个就是，项⽬中每个模块都使⽤了各⾃的命名空间进⾏封装，避免了命名冲突或者名字污染。

**继承**：项⽬中的继承⽤得⽐较少，主要是对⼯具类的继承，项⽬中多个地⽅使⽤到 noncopyable 和enable_shared_from_this，保证了代码的复⽤性。实际上对共有功能可以设计⼀个基类来继承，⽐如我项⽬中的connection⽬前有 httpconnection 和 tcpconnection 两种，可以通过继承 connection 基类来减少重复代码，因为我当时做的时候只考虑到 http连接，ftp是后⾯加上去的，所以就没⽤这样设计，后⾯可以优化⼀下。

**多态**：我项⽬中的多态主要⽤了静态多态，动态多态没有涉及。静态多态在⽇志系统中对流输⼊运算符进⾏了重载，以及在⽇志系统和内存池中都有各种函数模板的泛型编程。实际上刚刚说的 httpconnection 和ftpconnection 从 connection 派⽣出来后是可以使⽤动态多态的。



**noncopyable :**

用于**阻止对象拷贝**的辅助类模板   ==  将拷贝构造函数和拷贝赋值运算符声明为私有（或者删除）来阻止对象的拷贝。由于它是一个空类，所以不会占用额外的空间 



**enable_shared_from_this:**

在类内部获取指向自身的 **std::shared_ptr**

在 C++ 中，如果我们通过**一个裸指针**创建了一个`std::shared_ptr`，并将这个指针交给多个`std::shared_ptr` 进行管理，那么就会产生潜在的问题：裸指针和 `shared_ptr` 会各自维护自己的引用计数，可能导致资源的提前释放。

`enable_shared_from_this` 解决了这个问题，它要求类继承自 `std::enable_shared_from_this` 并通过调用 `shared_from_this()` 成员函数来**获得指向自身的 `std::shared_ptr`**。这样，可以确保在**类的生命周期内**，只有一个**有效的 `std::shared_ptr` 来管理该对象**，避免资源提前释放。

```c++
#include <memory>
class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    //    使用了 C++ 中的智能指针 std::shared_ptr 和 std::enable_shared_from_this 来创建共享的对象实例。std::enable_shared_from_this 是一个基类，用于解决在类内部获得 shared_ptr 的问题，以避免可能导致的循环引用问题。
    std::shared_ptr<MyClass> getShared() {
        return shared_from_this(); // 获取指向自身的 shared_ptr
    }
};
int main() {
    std::shared_ptr<MyClass> ptr1 = std::make_shared<MyClass>();
    std::shared_ptr<MyClass> ptr2 = ptr1->getShared();
    // 现在 ptr1 和 ptr2 共享同一个 MyClass 对象
    return 0;
}
使用 enable_shared_from_this 需要注意以下几点：
一个对象只能被其管理的 std::shared_ptr 赋值，不能通过裸指针直接创建 shared_ptr。
使用 shared_from_this() 的前提是，该对象已经被某个 std::shared_ptr 管理，否则调用 shared_from_this() 会导致未定义行为。  
```

## 2、项目细节

#### 2、线程池

##### 工作原理、作用

该项目使用固定线程数量的线程池（大小固定的数组）并发处理用户请求，**主线程负责读写**，**工作线程**（线程池中的线程）**负责处理逻辑**（HTTP请求报文的解析等等）。**所有线程是分离状态，当某个线程运行结束时，它的资源会被系统自动回收**，通过**锁和信号量来实现线程同步**，保证操作的原子性。

**线程池工作原理：**

- **线程创建**：当线程池初始化时，它会预先创建一定数量的核心线程（这是可配置的）。
- **任务队列**：线程池通常有一个任务队列，用于存储待执行的任务。
- **任务分发**：当提交一个新任务时，线程池会首先检查有无空闲的核心线程。如果有，就直接将任务交给这个线程执行；如果没有，并且当前线程数未达到上限，则会创建一个新线程来处理这个任务；如果达到了上限，则将任务放入任务队列中等待。
- **线程终止**：如果一个线程长时间处于空闲状态，并且当前的线程数量超过核心线程数，线程池可能会决定关闭这个线程。这也是为了资源优化。

**作用：**

- **降低资源消耗**：重复利用线程，减少线程创建和销毁的开销。
- **提高响应速度**：任务到来时，可以不需要等待线程创建就直接执行。
- **提高线程的可管理性**：线程是稀缺资源，通过线程池可以进行统一分配、调优和监控。

好处：

- 创建/销毁线程伴随着系统开销（用户内核转换等），过于频繁的创建/销毁线程，会很大程度上影响处理效率；线程并发数量过多，抢占系统资源从而导致阻塞 ；线程池是为了避免创建和销毁线程所产生的开销，避免活动的线程消耗的系统资源：

  - **提高响应速度**，任务到达时，无需等待线程即可立即执行；
  - **提高线程的可管理性**：线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。

- 阻塞队列会对当前线程产生阻塞，比如一个线程从一个空的阻塞队列中取元素，此时线程会被阻塞直到阻塞队列中有了元素。**当队列中有元素后，被阻塞的线程会自动被唤醒**；使用阻塞队列的原因：

  - 线程池创建线程需要获取locker，影响并发效率，阻塞队列可以很好的缓冲。

  - 另外一方面，如果新任务的到达速率超过了线程池的处理速率，那么新到来的请求将累加起来，这样的话将耗尽资源



```c++
pthread_create(&thread, nullptr, ThreadEntry, this);
```

pthread_create第四个参数是指向处理子线程工作的函数的指针，但是如果线程处理函数是类的成员函数时，this指针会被默认传进pthread_create这和规定的参数类型void*矛盾，所以在项目中线程处理函数要是静态成员函数，第四个参数就是this指针。

```c++
class MyClass {
public:
    static void *ThreadEntry(void *arg) {
        MyClass *thisPtr = static_cast<MyClass*>(arg);
        thisPtr->DoWork();
        return nullptr;
    }
    void StartThread() {
        pthread_create(&thread, nullptr, ThreadEntry, this);
    }
    void WaitForThreadToExit() {
        pthread_join(thread, nullptr);
    }
private:
    void DoWork() {
        // 实际工作函数
        std::cout << "Working in thread!" << std::endl;
    }
    pthread_t thread;
};
```





**线程池的7个核心参数：**

核心参数有:

- 存储线程的数组锁
- 条件变量
- 任务队列(一个function类的队列)

**主要通过线程池数组来存储已经创建的线程，为了后续主线程回收时使用。通过锁和条件变量来实现了生产者消费者模式。主要通过任务队列实现主线程创建任务，放入任务队列，其余线程为消费者，不断获取任务队列中的任务，然后执行。在该项目中，通过使用分离线程，优化了需要主线程来回收的过程。**

线程池的核心参数有 核心线程数、最大线程数、非核心线程的存活时间、时间单位、阻塞工作队列、拒绝策略、ThreadFactory(线程工厂)。

核心线程数：可理解为线程池可维护的最小线程数量，核心线程创建后不会被回收，但大于核心数量的线程，在超过存活时间后会被回收。

最大线程数：线程池可容纳(允许创建)的最大线程数量。(核心线程数+非核心线程数)

非核心线程的存活时间：非核心线程的空闲时间超过该存活时间，就会被回收。

时间单位：存活时间的单位。

阻塞工作队列：是用来存储等待执行任务的队列。

拒绝策略：线程池的核心线程和非核心线程耗尽，工作队列也满了的时候，如果有新提交的任务，则直接采用拒绝策略。默认的拒绝策略是：ThreadPoolExecutor.AbortPolicy，丢弃任务并抛出。一共4个拒绝策略，还有三个分别是 丢弃任务不抛出异常、丢弃队列中最末尾的任务（也就是最早进入队列的最旧的任务）、由原调用线程执行该任务（谁调用，谁执行).

ThreadFactory(线程工厂)：主要用于创建线程，默认的工厂是defaultThreadFactory。

**常见线程池有哪些以及使用场景？**
常见线程池：

FixedThreadPool（线程数固定的线程池）使用的队列是无界队列，LinkedBlockingQueue。使用场景：**适用于CPU密集型任务**，确保CPU在长期被工作线程使用情况下，尽可能少的分配线程，即适用于执行长期的任务。

CachedThreadPool（线程数根据任务动态调整的线程池）使用的队列是，同步队列，队列容量未0；使用场景：用于并发执行大量短期的小任务。

SingleThreadPool(单一线程的线程池，只有一个线程）使用的队列是：无界队列，使用场景：适用于串行执行任务的场景，将任务按顺序执行。

ScheduledThreadPool（能执行定时、周期性任务的线程池）使用的队列是：DelayedWorkQueue基于堆栈结构的延迟队列，基于数组实现，初始容量为16的队列，按照延迟时间排序。使用场景：周期性执行，且需要限制线程数量的场景。



 








##### 线程池开关、来任务处理

**线程池怎么开，关：**

- **开启**：通常线程池在创建时就会初始化一定数量的核心线程。
- **关闭**：关闭线程池时，通常有两种方式：
  - `shutdown()`: 将线程池的状态设置为关闭，并不再接收新任务，但会继续处理队列中的任务。
  - `shutdownNow()`: 尝试停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待的任务列表。

**有和无任务的处理：**

- **没有任务**：当线程池中的线程没有任务可执行时，它们可能会进入等待状态。如果一个线程长时间没有任务执行，并且当前的线程数量超过核心线程数，线程池可能会决定结束这个线程。
- **有任务**：当有新任务提交给线程池时，线程池会按照上面描述的“任务分发”逻辑来处理。

###### **线程池任务分配**

- 提交一个线程任务时，线程池会分配一个空闲线程，用于执行线程任务。

- 如果线程池中不存在空闲线程，那么就判断“当前存活的线程数量”是否小于核心线程数量，如果小于核心线程数，则创建一个核心线程去处理提交的线程任务。

- 如果线程池的线程数量大于核心线程数，则线程池会判断工作队列是否已满，若工作队列未满，则将该任务放入工作队列，等待线程池从工作队列取出别执行。

- 如果工作队列已满，则判断线程池的当前线程数是否达到最大线程数，如果未达到最大线程数，则创建一个非核心线程来执行提交的任务；**若达到线程池的最大线程数量，还有新任务需要提交，则直接执行拒绝策略。**

- 综上：线程池的执行顺序是：核心线程、工作队列、非核心线程。

###### **工作线程处理完一任务后**

（1） 当处理完任务后如果请求队列为空时，则这个线程重新回到阻塞等待的状态

（2） 当处理完任务后如果请求队列不为空时，那么这个线程将处于与其他线程竞争资源的状态，谁获得锁谁就获得了处理事件的资格。



##### **实现细节**

主线程为异步线程，负责监听文件描述符，接收socket新连接，当监听的socket发生了读写事件，将任务插入请求队列。**线程池的工作线程**从请求队列取出一个任务，完成数据的读写处理，执行任务逻辑处理。

 **具体做法：**通过epoll_wait 发现这个connfd上有可读事件了(EPOLLIN),主线程就将这个HTTP请求报文读进这个连接socket的读缓存中，之后将任务对象(指针)插入线程池的请求队列中;

- **1、线程池初始化：**

  线程池是一组资源的集合,这组资源**在服务器启动之初就被完全创建好并初始化**,这称为静态资源。

  **先是初始化一个线程池数组，然后遍历数组进行pthread_create，同时会注册子线程的work函数，work会让子线程阻塞在工作队列上 (因为work会循环的从工作队列中取出任务并处理，为保证线程安全，取出任务的时候采用信号量以及互斥锁的措施。一开始工作队列为空，所以this->condition.wait条件变量唤醒等待函数会一直阻塞着)。创建完线程之后再detech一下做子线程分离就初始化好了**。 消息队列的大小由机器硬件来决定，本实验环境选取`max_request = 10000`。

- 2、每一个工作线程会在一个无限循环中等待新的任务。当主线程添加一个新的任务到任务队列时，它会通过条件变量通知一个等待的线程（如果有的话）。工作线程被唤醒后，会取出任务队列中的任务并执行它。当析构函数被调用时（例如程序退出时），它会通知所有线程停止执行并等待它们完成当前的任务。

- > 设置成脱离态的目的：为了在使用线程的时候，避免线程的资源得不到正确的释放，从而导致了内存泄漏的问题。所以要确保进程为可分离的的状态，否则要进行线程等待已回收他的资源。

  > 线程池的实现还需要依靠互斥锁机制以及条件变量机制来实现线程同步，保证操作的原子性

- 3、操作工作队列一定**要加互斥锁**（`locker`），因为它被所有线程共享(与最大请求数做个判断，允许)。

###### 线程池参数

- **设计参数：**

  **核心线程数**：始终在池中保持的线程数量。

  **最大线程数**：池中允许的最大线程数量。

  **线程空闲时间**：如果线程的空闲时间超过这个值，并且当前线程数大于核心线程数，那么线程可能会被终止。

  **任务队列**：用于存储等待执行的任务。这可以是一个有界队列或无界队列。

1. **线程池管理器：**
   - 管理一组工作线程。
   - 维护一个任务队列，通常是线程安全的队列。
   - 提供接口供外部提交任务到任务队列。
   - 通过条件变量来通知工作线程有新任务到来。
2. **工作线程 ：**
   - 循环等待任务队列中的任务。
   - 执行取出的任务。
   - 在没有任务时，线程可能进入休眠状态等待新任务的通知。
3. **任务 (Task)**：
   - 一个函数对象（std::function），它可以被线程执行。
4. **同步机制**：
   - 使用互斥量（Mutex）来保护对共享资源（如任务队列）的访问。
   - 使用条件变量来实现线程之间的通知机制。
5. **优雅地关闭线程池 (Graceful Shutdown)**：
   - 停止接受新任务。
   - 完成所有已经存在的任务。
   - 通知所有工作线程退出。
   - 等待所有工作线程结束。

**选择子线程方法：**

主线程**选择哪个子线程**来为新任务服务方式：

1. 随机算法和轮流选取算法。
2. 主进程和所有地子进程通过一个共享的工作队列(list 单链表)来同步，子进程都睡眠在该工作队列上。

```c++
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <iostream>
// 线程池类
class ThreadPool {
public:
    // 构造函数，启动线程池
    ThreadPool(size_t numThreads) {
        start(numThreads);
    }
    // 析构函数，停止所有线程
    ~ThreadPool() {
        stop();
    }
    // 将任务加入队列
    void enqueue(std::function<void()> task) {
        {
            // 锁定任务队列
            std::unique_lock<std::mutex> lock(queueMutex);
            // 添加任务到队列
            tasks.emplace(std::move(task));
        }
        // 任务添加后，通知一个等待的线程
        condition.notify_one();
    }
private:
    // 工作线程的容器
    std::vector<std::thread> workers;
    // 任务队列
    std::queue<std::function<void()>> tasks;
    // 用于同步的互斥锁
    std::mutex queueMutex;
    // 条件变量，用于通知有任务到来
    std::condition_variable condition;
    // 停止标志
    bool stopping = false;
    // 启动指定数量的工作线程
    void start(size_t numThreads) {
        for (size_t i = 0; i < numThreads; ++i) {
            // 创建新线程并添加到工作队列
            workers.emplace_back([this] {
                // 每个工作线程的主循环
                while (true) {
                    std::function<void()> task;
                    {
                // 锁定互斥锁以访问任务队列
                        std::unique_lock<std::mutex> lock(this->queueMutex);
                // 等待条件变量通知，或停止标志被设置或任务队列非空
                        this->condition.wait(lock, [this] {
                            return this->stopping || !this->tasks.empty();
                        });

                // 如果收到停止信号并且任务队列为空，则退出线程
                        if (this->stopping && this->tasks.empty())
                            break;

               // 从任务队列中取出任务
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }

              // 执行取出的任务
                    task();
                }
            });
        }
    }
    // 停止所有线程
    void stop() noexcept {
        {
            // 确保互斥锁被锁定
            std::unique_lock<std::mutex> lock(queueMutex);
            // 设置停止标志
            stopping = true;
        }
        // 唤醒所有等待线程
        condition.notify_all();
        // 等待所有工作线程完成当前任务并退出
        for (std::thread &worker : workers)
            worker.join();
    }
};
// 程序的主函数
int main() {
    // 创建拥有4个工作线程的线程池
    ThreadPool pool(4);
    // 将8个任务加入线程池
    for (int i = 0; i < 8; ++i) {
        pool.enqueue([i] {
            // 打印当前执行的任务编号
            std::cout << "executing task " << i << std::endl;
        });
    }
    // 主线程会在此阻塞，直到线程池中的所有任务都已完成处理，
    // 随后触发ThreadPool的析构函数，它会等待所有线程结束后再完全退出
    return 0;
}

```

######  线程池代码

```C++
#include<pthread.h>
#include<queue>
#include<iostream>
#include<string.h>
#include<exception>
#include<unistd.h>

const int NUMBER = 2;

using callback = void(*)(void* arg);


class Task{
private:
public:
    callback fun;// 任务函数指针
    void* arg;   // 任务函数参数
public:
    Task(){
        fun = nullptr;
        arg = nullptr;
    }
    Task(callback f, void* arg){
        fun = f;
        this->arg = arg;
    }
};

class Task_queue
{
private:
    pthread_mutex_t m_mutex;
    std::queue<Task> m_queue;
public:
    Task_queue();
    ~Task_queue();

    //添加任务
    void add_task(Task& task);
    void add_task(callback f, void* arg);

    //取出任务
    Task& get_task();

    int task_size(){
        return m_queue.size();
    }
};

class Thread_pool{
private:
    pthread_mutex_t m_mutex_pool;   // 线程池互斥锁
    pthread_cond_t m_not_empty;     //条件变量，用来唤醒等待的线程，假设队列是无限的
    pthread_t* thread_compose;      // 线程数组
    pthread_t m_manger_thread;
    Task_queue* m_task_queue;
    int m_min_num;                  //最小线程数
    int m_max_num;                  //最大线程数
    int m_busy_num;                 //工作的线程数
    int m_live_num;                 //存活的线程数一
    int m_destory_num;              //待销毁的线程数
    bool shutdown = false;          //是否销毁
    
    //在使用pthread_create时候，第三个参数的函数必须是静态的
    //要在静态函数中使用类的成员(成员函数和变量)只能通过两种方式
    //1.通过类的静态对象，单例模式中使用类的全局唯一实例来访问类成员
    //2.类对象作为参数传递给该静态函数中
    //管理线程的任务函数，形参是线程池类型的参数
    static void* manager(void* arg);
    //工作线程的任务函数
    static void* worker(void* arg);
    //销毁线程池
    void destory_thread();
public:
    Thread_pool(int min, int max);
    ~Thread_pool();

    void add_task(Task task); // 添加任务到线程池
    int get_busy_num();  // 获取忙碌线程数
    int get_live_num();  // 获取存活线程数
};


Task_queue::Task_queue()
{
    pthread_mutex_init(&m_mutex, nullptr);// 初始化任务队列互斥锁
}


Task_queue::~Task_queue()
{
    pthread_mutex_destroy(&m_mutex);// 销毁任务队列互斥锁
}


void Task_queue::add_task(Task& task){
    pthread_mutex_lock(&m_mutex);
    m_queue.push(task);
    pthread_mutex_unlock(&m_mutex);
}

void Task_queue::add_task(callback f, void* arg){
    pthread_mutex_lock(&m_mutex);
    Task t(f, arg);
    m_queue.push(t);
    pthread_mutex_unlock(&m_mutex);
}


Task& Task_queue::get_task(){
    Task t;
    pthread_mutex_lock(&m_mutex);
    if(m_queue.size() > 0){
        t = m_queue.front();
        m_queue.pop();
    }
    pthread_mutex_unlock(&m_mutex);
    return t;
}

Thread_pool::Thread_pool(int min, int max){
    //实例化任务队列
    m_task_queue = new Task_queue;
    //do while的好处，但是c++有析构，所以不太用的上
    do
    {
        //初始化线程池
        m_min_num = min;
        m_max_num = max;
        m_busy_num = 0;
        m_live_num = min;

        //根据最大上限给 线程数组 分配内存
        thread_compose = new pthread_t[m_max_num];
        if(thread_compose == nullptr){
            std::cout<<"create thread array fail~\n";
            break;
        }
        //初始化线程数组
        memset(&thread_compose, 0, sizeof(thread_compose) * m_max_num);

        //初始化互斥锁和条件变量
        if(pthread_mutex_init(&m_mutex_pool, nullptr) != 0){
            throw::std::exception();
        }
        if(pthread_cond_init(&m_not_empty, nullptr) != 0){
            throw::std::exception();
        }

        //创建管理者线程
        pthread_create(&m_manger_thread, nullptr, manager, this);

        //创建工作线程
        for(int i = 0; i < m_min_num; i++){
            //传this指针，才能访问类内成员函数
            pthread_create(&thread_compose[i], nullptr, worker, this);
        }
    } while (0);
    
}

Thread_pool::~Thread_pool(){
    shutdown = 1;
    //销毁管理者线程
    pthread_join(m_manger_thread, NULL);
    //唤醒所有消费者线程
    for (int i = 0; i < m_live_num; ++i)
    {   //signal是唤醒某一个，broadcast是唤醒所有，但是还是要抢一把锁，所以都一样
        pthread_cond_signal(&m_not_empty);
    }

    if(m_task_queue){
        delete m_task_queue;
    }
    if(thread_compose){
        delete [] thread_compose;
    }
    pthread_mutex_destroy(&m_mutex_pool);
    pthread_cond_destroy(&m_not_empty);
}

void Thread_pool::add_task(Task task){
    if(shutdown)
    {   
        std::cout<<"the thread pool will destory!\n";
        return;
    }
    //添加任务不需要加锁，因为任务队列中有锁
    m_task_queue->add_task(task);
    //添加完任务就要唤醒工作线程取任务
    pthread_cond_signal(&m_not_empty);
}

int Thread_pool::get_busy_num(){
    int busy_num = 0;
    pthread_mutex_lock(&m_mutex_pool);
    busy_num = m_busy_num;
    pthread_mutex_unlock(&m_mutex_pool);
    return busy_num;
}

int Thread_pool::get_live_num(){
    int live_num = 0;
    pthread_mutex_lock(&m_mutex_pool);
    live_num = m_live_num;
    pthread_mutex_unlock(&m_mutex_pool);
    return live_num;
}

//管理线程任务函数
//主要任务：不断检测工作线程数量，存活线程的数量，然后再决定增加还是删除
void* Thread_pool::manager(void* arg){
    Thread_pool* pool = static_cast<Thread_pool*>(arg);
    while(pool->shutdown){
        //每五秒检测一次
        sleep(5);

        //取出线程数量
        pthread_mutex_lock(&pool->m_mutex_pool);
        int queue_size = pool->m_task_queue->task_size();
        int live_size = pool->m_live_num;
        int busy_size = pool->m_busy_num;
        pthread_mutex_unlock(&pool->m_mutex_pool);

        //创建线程，最多创建两个
        if(queue_size > live_size && live_size <pool->m_max_num){
            pthread_mutex_lock(&pool->m_mutex_pool);
            int count = 0;
            for(int i = 0; i < pool->m_max_num && count < NUMBER 
                && pool->m_live_num < pool->m_max_num; i++){
                    //之前memset了
                    if(pool->thread_compose[i] == 0){
                        pthread_create(&pool->thread_compose[i], nullptr, worker, pool);
                        count++;
                        pool->m_live_num++;
                    }
            }
            pthread_mutex_unlock(&pool->m_mutex_pool);
        }

        //销毁多余的线程
        //判断条件：忙线程*2 < 存活的线程 && 存活的线程数 > 最小线程数
        if(2*busy_size < live_size && live_size > pool->m_min_num){
            pthread_mutex_lock(&pool->m_mutex_pool);
            pool->m_destory_num = NUMBER;
            pthread_mutex_unlock(&pool->m_mutex_pool);
            //让工作线程自杀——唤醒但没事儿干就自杀，定义在worker里面
            //没事儿干的线程被阻塞了，阻塞在m_not_empty()条件变量上
            //唤醒后的线程就自己退出了，代码在worker里面
            for (int i = 0; i < NUMBER; ++i)
            {
                pthread_cond_signal(&pool->m_not_empty);
            }
        }
    }
    return nullptr;
}

//工作线程任务函数
void* Thread_pool::worker(void* arg){
    Thread_pool* pool = static_cast<Thread_pool*>(arg);
    //工作线程一直不停工作
    while(true){
        //当线程访问任务队列的时候加锁
        pthread_mutex_lock(&pool->m_mutex_pool);
        //判断工作队列是否为空，为空阻塞线程
        while(pool->m_task_queue->task_size() == 0 && pool->shutdown == 1){
            std::cout<<"thread "<<std::to_string(pthread_self())<<"waiting...\n";
            //将调用线程放入条件变量的等待队列中
            pthread_cond_wait(&pool->m_not_empty, &pool->m_mutex_pool);
            
            //解除阻塞之后判断是否要销毁线程
            //由于管理线程中满足线程销毁条件了，就通过条件变量唤醒线程
            //然后由于唤醒的线程是空闲的即任务队列中没东西，如果有东西就不会空闲，
            //不会满足manager线程中销毁线程的条件
            if(pool->m_destory_num > 0){
                pool->m_destory_num--;
                if(pool->m_live_num > pool->m_min_num){
                    pool->m_live_num--;
                    //先解锁再销毁
                    pthread_mutex_unlock(&pool->m_mutex_pool);
                    pool->destory_thread();
                }
            } 
        }

        if(pool->shutdown){
            pthread_mutex_unlock(&pool->m_mutex_pool);
            pool->destory_thread();
        }
        
        //取任务
        Task task = pool->m_task_queue->get_task();
        pool->m_busy_num++;
        pthread_mutex_unlock(&pool->m_mutex_pool);
        //执行任务
        std::cout<<"thread "<<std::to_string(pthread_self())<<" start working.....\n"; 
        task.fun(task.arg);
        delete task.arg;
        task.arg = nullptr;

        //任务处理结束
        std::cout<<"thread "<<std::to_string(pthread_self())<<" end work.....\n"; 

        pthread_mutex_lock(&pool->m_mutex_pool); 
        pool->m_busy_num--;
        pthread_mutex_unlock(&pool->m_mutex_pool);

    }
    return nullptr;
}


void Thread_pool::destory_thread(){
    pthread_t tid = pthread_self();
    for(int i = 0; i < m_max_num; i++){
        if(thread_compose[i] == tid){
            std::cout << "threadExit() function: thread " 
                << std::to_string(pthread_self()) << " exiting..." << std::endl;
        }
        thread_compose[i] = 0;
        break;
    }
    pthread_exit(nullptr);
}
```



```c++
C++实现的简易线程池，包含线程数量，启动标志位，线程列表以及条件变量。
构造函数 主要是声明未启动和线程数量的。
start函数 为启动线程池，将num个线程绑定threadfunc⾃定义函数并执⾏，加⼊线程列表
stop 是暂时停⽌线程，并由条件变量通知所有线程。
析构函数 是停⽌，阻塞所有线程并将其从线程列表剔除后删除，清空线程列表。
#pragma once
#include <vector>
#include <queue>
#include <memory>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <stdexcept>
class ThreadPool {
public:
  ThreadPool(size_t threads);
  ~ThreadPool();
  // 函数模板的声明。F是一个函数类型或函数对象的类型，Args...是一组可变数量的参数类型。
  // 右值引用类型的参数f，用于接收要执行的函数或函数对象
 // 一组右值引用类型的可变数量参数，用于接收要传递给函数的参数
  template<typename F, typename... Args>
  auto enqueue(F&& f, Args&&... args)
  -> std::future<typename std::result_of<F(Args...)>::type>; 
  // enqueue异步执行函数，并返回一个std::future对象
  // result_of 的作⽤与decltype 相同，不过针对的是函数返回值，通过std::result_of模板来推断函数F的返回类型
private:
  std::vector<std::thread> _workers;
  std::queue<std::function<void()>> _tasks;
  std::mutex _queueMutex;
  std::condition_variable _condition;
  bool _stop;
};
inline ThreadPool::ThreadPool(size_t threads) : _stop(false) {
 // 设置线程任务
    for(size_t i = 0; i < threads; ++i) {
       // 每个线程需要做的事情很简单：
       // 1. 从任务队列中获取任务（需要保护临界区）
       // 2. 执⾏任务
       _workers.emplace_back([this] {
           while(true) {
               std::function<void()> task;
           {
               std::unique_lock<std::mutex> lock(this->_queueMutex);
               // 等待唤醒，条件是停⽌或者任务队列中有任务
               this->_condition.wait(lock,
                                       [this]{ return this->_stop || !this-
>_tasks.empty(); });
               if(this->_stop && this->_tasks.empty())
                     return;
               task = std::move(this->_tasks.front());
               this->_tasks.pop();
           }
           task();
        }
    });
  }
}
// 将传入的函数对象 f 和一组可变数量的参数 args 包装成一个任务，并将任务放入任务队列中等待执行。通过返回一个 std::future 对象，可以在未来获取函数执行的结果
// 通过调用 enqueue 函数，可以将需要执行的函数和参数添加到线程池中，线程池会在后台启动工作线程来执行这些任务，并可以通过 std::future 获取函数执行的结果
template<typename F, typename... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
     -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;
    // 将需要执⾏的任务函数打包（bind），转换为参数列表为空的函数对象
    // 将函数对象 f 和参数 args 绑定在一起，创建一个可以无参数调用的函数对象 task。std::packaged_task 是一个模板类，用于包装可调用对象，并返回一个可以获取调用结果的 std::future 对象。
    auto task = std::make_shared<std::packaged_task<return_type()>> (
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
 );
    //获取一个 std::future 对象，用于获取函数执行的结果
 std::future<return_type> res = task->get_future();
 {
      std::unique_lock<std::mutex> lock(_queueMutex);
      if(_stop)//则不再接受新的任务，并抛出异常。
            throw std::runtime_error("enqueue on stopped ThreadPool");
      // 最妙的地⽅，利⽤ lambda函数 包装线程函数，使其符合 function<void()> 的形式
      // 并且返回值可以通过 future 获取
     // 通过 lambda 函数将任务函数 task 包装成一个可以无参数调用的函数对象，并将其放入任务队列 _tasks。
       _tasks.emplace([task]() {
             (*task)();
       });
 }
 _condition.notify_all();//唤醒所有工作线程，通知它们有新的任务可执行。
 return res;//任务的 std::future 对象
}
inline ThreadPool::~ThreadPool() {
   {
      std::unique_lock<std::mutex> lock(_queueMutex);
      _stop = true;
   }
   _condition.notify_all(); // 唤醒所有线程，清空任务
   for(std::thread& worker : _workers) {
      worker.join();
   }
}

```









###### 线程数量、线程越多好吗

**线程数量确定：**

默认4个线程数，服务器是4核CPU的。调整线程池中的线程数量的最主要的目的是为了充分并合理地使用 CPU 和内存等资源，从而最大限度地提高程序的性能线程池中的线程数的限制因素是CPU中央处理器的处理器的数量**N**。

对于**CPU密集型任务**：

- 线程数通常设置为与CPU核心数相同的数量。
- 由于任务主要消耗CPU时间，过多的线程会导致上下文切换增多，反而可能降低性能。
- 一个核心同时只能执行一个线程的任务，所以超过核心数的线程在同一时间内并不会得到执行。

对于**I/O密集型任务**：

- 线程数往往设置得比CPU核心数多。
- 这是因为I/O密集型任务中，线程大部分时间在等待I/O操作完成（如读写文件、网络数据传输等），在这个等待过程中CPU可以执行其他任务。
- 设置更多的线程可以提高程序的吞吐量，确保CPU在等待I/O操作的同时，仍然可以处理其他任务。

不过，线程数量的最优值还受限于其他因素，如系统的线程开销、程序的设计、内存大小、I/O系统的性能等。线程池的大小设置过大，不仅会浪费资源（内存和上下文切换的开销），而且还可能因为线程竞争导致性能下降。

**多线程中线程越多越好吗：**

不是。线程越多，CPU有限的情况下，线程来回切换开销大。

- （1）假设现有8个CPU、8个线程，每个线程占用一个CPU，同一时间段内，若8个线程都运行往前跑，相比较5/6/7个线程，8个线程的效率高。
-  （2）但若此时有9个线程，只有8个CPU，9个线程同时运行，则此时牵扯到线程切换，而线程切换是需要消耗时间的。 
- 所以随着线程数越多，效率越来越高，但到一个峰值，再增加线程数量时，就会出现问题。线程太多要来回的切换，最终可能线程切换所用时间比执行时间业务所用时间还大。 
- （3） 随着线程数越多,由于线程执行的时序的问题，程序可能会崩溃或产生二义性。











###### 1000个客户端访问，如何响应每一个

**项⽬中使⽤了I/O多路复⽤技术，每个线程中管理⼀定数量的连接，只有线程池中的连接有请求，epoll就会返回请求的连接链表，管理该连接的线程获取活动链表，然后依次处理各个连接的请求。如果该线程没有任务，就会等待主reactor分配任务。这样就能达到服务器⾼并发的要求，同⼀时刻，每个线程都在处理自己所管理连接的请求。**

**具体实现：**

本项目是通过对**子线程循环调用**来解决高并发的问题的。

我们通过子线程的run调用函数进行while循环，让每一个线程池中的线程永远都不会停止，访问请求被封装到请求队列(list)中，如果没有任务，线程就一直阻塞等待；有任务，线程就抢占式进行处理，直到请求队列为空，表示任务全部处理完成。

首先在创建线程的同时就调用了pthread_detach将线程进行分离，不用单独对工作线程进行回收，资源自动回收。

###### 一客户占用线程久，影响别的客户请求吗

**影响：**

会影响这个大请求的所在线程的所有请求，因为每个eventLoop都是依次处理它通过epoll获得的活动事件，也就是活动连接。如果该eventloop处理的连接占用时间过长的话，**该线程后续的请求只能在请求队列中等待被处理，从⽽影响接下来的客户请求**。

**解决：**

1. **主** **reactor** **的⻆度**：可以记录⼀下每个子reactor的阻塞连接数，主reactor根据每个reactor的当前负载来分发请求，达到负载均衡的效果。

2. **从** **reactor** **的角度**：

-  **超时时间**：为每个连接分配⼀个时间⽚，类似于操作系统的进程调度，当当前连接的时间片**⽤完以后，将其重新加⼊请求队列**，响应其他连接的请求，还可以为每个连接设置⼀个优先级，这样可以优先响应重要的连接。

- **关闭时间**：为了避免部分连接长时间占⽤服务器资源，可以给每个连接设置⼀个最⼤响应时间，当⼀个连接的最⼤响应时间用完后，服务器可以**主动将这个连接断开，让其重新连接**。



解决、IO，计算耗时问题

**⼀连接请求耗时长，发⽣IO耗时还是计算耗时？**

IO耗时的话，则会阻塞同⼀subreactor中的所有线程，如何是计算耗时的话，因为本身也设计了计算线程池，对服务器本身并没有太多影响

**⼀连接请求资源⼤，在发送响应报⽂时会造成响应其他连接的请求吗？优化：**

（断点续传，设置响应报⽂的⼤⼩上限，当响应报⽂超出上限时，可以记录已经发送的位置，之后可以选择继续由该线程进⾏发送，也可以转交给其他线程进⾏发送）对服务器本身并没有太多影响）

两种高效的并发模式：并发其实适合于**``I/O`密集型**而不适合于计算密集型，比如经常读写文件，访问数据库等，由于I/O操作的速度远没有CPU计算速度快，所以让程序阻塞于I/O操作将浪费大量的CPU时间。

##### **虚假唤醒定义、解决**

**虚假唤醒定义：**

当一定的条件触发时会唤醒很多在阻塞态的线程，但只有部分的线程唤醒是有用的，其余线程的唤醒是多余的。
比如说卖货，如果本来没有货物，突然进了一件货物，这时所有的顾客都被通知了，但是只能一个人买，所以其他人都是无用的通知。

**解决虚假唤醒：**

导致虚假唤醒的原因主要就是一个线程直接在if代码块中被唤醒了，这时它已经跳过了if判断，执行下面的程序。我们只需要将if判断改为while，这样线程就会被重复判断而不再会跳出判断代码块，从而不会产生虚假唤醒这种情况了。

**例子：使用if判断包装wait方法会出现虚假唤醒**

举个例子，我们现在有一个生产者-消费者队列和三个线程。

**1）** 1号线程从队列中获取了一个元素，此时队列变为空。

**2）** 2号线程也想从队列中获取一个元素，但此时队列为空，2号线程便只能进入阻塞(cond.wait())，等待队列非空。

**3）** 这时，3号线程将一个元素入队，并调用cond.notify()唤醒条件变量。

**4）** 处于等待状态的2号线程接收到3号线程的唤醒信号，便准备解除阻塞状态，执行接下来的任务(获取队列中的元素)。

**5）** 然而可能出现这样的情况：当2号线程准备获得队列的锁，去获取队列中的元素时，此时1号线程刚好执行完之前的元素操作，返回再去请求队列中的元素，1号线程便获得队列的锁，检查到队列非空，就获取到了3号线程刚刚入队的元素，然后释放队列锁。

**6）** 等到2号线程获得队列锁，判断发现队列仍为空，1号线程“偷走了”这个元素，所以对于2号线程而言，这次唤醒就是“虚假”的，它需要再次等待队列非空。

这一问题：在wait成功之后，资源就一定可以被使用么？答案是否定的，如果同时有两个或者两个以上的线程正在等待此资源，wait返回后，资源可能已经被使用了。

> 有可能多个线程都在等待这个资源可用的信号，信号发出后只有一个资源可用，但是有A，B两个线程都在等待，B比较速度快，获得互斥锁，然后加锁，消耗资源，然后解锁，之后A获得互斥锁，但A回去发现资源已经被使用了，它便有两个选择，一个是去访问不存在的资源，**另一个就是继续等待，那么继续等待下去的条件就是使用while，要不然使用if**的话pthread_cond_wait返回后，就会顺序执行下去。
>
> 所以，在这种情况下，应该使用while而不是if:

```c
while(resource == FALSE)pthread_cond_wait(&cond, &mutex);
```

不做处理可能会报错



##### 惊群问题定义、解决、场景

**定义：**

多个线程（或者进程）同时等待一个事件的到来并准备处理事件，当事件到达时，把所有等待该事件的线程（或进程）均唤醒，但是**只能有一个线程最终可以获得事件的处理权**，其他所有线程又**重新陷入睡眠等待**下次事件到来。这种线程被频繁唤醒却又没有真正处理事件导致CPU无谓浪费称为计算机中的“惊群问题”。

**解决：**

1. 对于**线程池中的惊群问题**，我们需要分情况看待，有时候**业务需求就是需要唤醒所有线程**，那么这时候使用notify_all()唤醒所有线程就不能称为”惊群问题“，因为CPU并没有无谓消耗。而对于只需要唤醒一个线程的情况，我们需要使用**notify_one()函数**代替notify_all()只唤醒一个线程，从而避免惊群问题。

2. 对于**epll函数调用的惊群问题解决**办法可以参考Nginx的解决办法，多个进程将listenfd加入到epoll之前，首先尝试获取一个全局的accept_mutex互斥锁，只有获得该锁的进程才可以把listenfd加入到epoll中，当网络连接事件到来时，只有epoll中含有listenfd的线程才会被唤醒并处理网络连接事件。从而解决了epoll调用中的惊群问题。

**场景：**

1. **Linux2.6内核版本之前系统API中的accept调用**
   在Linux2.6内核版本之前，当多个线程中的accept函数同时监听同一个listenfd的时候，如果此listenfd变成可读，则系统会唤醒所有使用accept函数等待listenfd的所有线程（或进程），但是最终只有一个线程可以accept调用返回成功，其他线程的accept函数调用返回EAGAIN错误，线程回到等待状态，这就是accept函数产生的惊群问题。但是在Linux2.6版本之后，内核解决了accept函数的惊群问题，**当内核收到一个连接之后，只会唤醒等待队列上的第一个线程（或进程），从而避免了惊群问题**。
2. **epoll函数中的惊群问题**
   如果我们使用多线程epoll对同一个fd进行监控的时候，当fd事件到来时，内核会把所有epoll线程唤醒，因此产生惊群问题。为何内核不能像解决accept问题那样解决epoll的惊群问题呢？内核可以解决accept调用中的惊群问题，是因为内核清楚的知道accept调用只可能一个线程调用成功，其他线程必然失败。而对于**epoll调用而言，内核不清楚到底有几个线程需要对该事件进行处理**，所以只能将所有线程全部唤醒。
3. **线程池中的惊群问题**
   在实际应用程序开发中，为了避免线程的频繁创建销毁，我们一般建立线程池去并发处理，而线程池最经典的模型就是生产者-消费者模型，包含一个任务队列，当队列不为空的时候，线程池中的线程从任务队列中取出任务进行处理。一般使用条件变量进行处理，当我们往任务队列中放入任务时，需要唤醒等待的线程来处理任务，如果我们使用C++标准库中的函数**notify_all()来唤醒线程**，则会将所有的线程都唤醒，然后最终只有一个线程可以获得任务的处理权，其他线程在此陷入睡眠，因此产生惊群问题。









##### detach和join区别、如何销毁线程

**detach和join区别：**

（1）当调用join()，主线程等待子线程执行完之后，主线程才可以继续执行，此时主线程会释放掉执行完后的子线程资源。主线程等待子线程执行完，可能会造成性能损失。

（2）当调用detach()，主线程与子线程分离，他们成为了两个独立的线程遵循cpu的时间片调度分配策略。子线程执行完成后会自己释放掉资源。分离后的线程，主线程将对它没有控制权。当你确定程序没有使用共享变量或引用之类的话，可以使用detch函数，分离线程。

**如何销毁线程：**

- **等待线程结束：**int pthread_join(pthread_t tid, void** retval);

  主线程调用，等待子线程退出并回收其资源，类似于进程中wait/waitpid回收僵尸进程，调用pthread_join的线程会被阻塞。

  - tid：创建线程时通过指针得到tid值。
  - retval：指向返回值的指针。

- **结束线程：**pthread_exit(void *retval);

  子线程执行，用来结束当前线程并通过retval传递返回值，该返回值可通过pthread_join获得。

- **分离线程：**int pthread_detach(pthread_t tid);

  主线程、子线程均可调用。主线程中pthread_detach(tid)，子线程中pthread_detach(pthread_self())，调用后和主线程分离，子线程结束时自己立即回收资源





##### 线程同步有哪些

线程同步的方式：信号量、条件变量、读写锁、互斥锁、自旋锁

**信号量**：**信号量可以看作是一个计数器**，当有线程访问资源时信号量加1，访问完成时信号量减1(这个修改是原子操作)，当信号量为0时，访问资源的线程被阻塞

**条件变量**：**条件变量是线程间的一种通知机制**，一个条件变量可以阻塞多个线程，这些线程会组成一个等待队列，直到条件成立后，向等待队列中的一个或所有线程发送“条件成立”的信号，解除它们的“被阻塞”状态。

**锁类（互斥锁、读写锁、自旋锁）：**

**锁**：锁主要是为了保证多线程条件下，共享资源的安全性。

**互斥锁**：

互斥锁是一种独占锁，当前线程加锁成功时，其他线程会加锁失败被阻塞，在多线程条件下可以防止共享资源被并发访问，造成数据错乱。

​       1、一次只能一个线程拥有互斥锁，其他线程只有等待

​       2、互斥锁是在抢锁失败的情况下主动放弃CPU进入睡眠状态直到锁的状态改变时再唤醒，而操作系统负责线程调度，为了实现锁的状态发生改变时唤醒阻塞的线程或者进程，需要把锁交给操作系统管理，所以**互斥锁在加锁操作时**涉及上下文的切换。互斥锁加锁的时间大概100ns左右，而实际上互斥锁的一种可能的实现是**先自旋一段时间，当自旋的时间超过阀值之后再将线程投入睡眠中**，因此在并发运算中使用互斥锁（每次占用锁的时间很短）的效果可能不亚于使用自旋锁

**读写锁**：

读写锁是一把锁，它有读和写两个状态，当“写锁”没有被线程持有的时候，多个线程可以共同持有”读锁“，读取共享资源。一旦“写锁”被线程持有，那些想要获取“读锁”的线程和想要获取“写锁”的线程都会被阻塞，任意时刻只允许一个线程持有写锁，在读多写少的时候应该选择读写锁。

- 多个读者可以同时进行读
- 写者必须互斥（只允许一个写者写，也不能读者写者同时进行）
- 写者优先于读者（一旦有写者，则后续读者必须等待，唤醒时优先考虑写者）

**自旋锁**：

如果当前线程想要使用自旋锁，但发现锁资源被其他线程持有，当前调用线程会一直忙等待，不停地循环判断锁资源是否被释放，直到被释放让当前线程获得为止。如果别的线程长时期占有锁，那么自旋就是在浪费CPU做无用功，**在确定加锁部分的代码执行时间很短时，使用自旋锁比较合适。**



###### **用条件变量不用互斥锁**

- 互斥锁一个明显的缺点是他只有两种状态：锁定和非锁定。
- 而条件变量通过允许线程阻塞和等待另一个线程发送信号的方法弥补了互斥锁的不足，他常和互斥锁一起使用，以免出现竞态条件。
- 当条件不满足时，线程往往解开相应的互斥锁并阻塞线程然后等待条件发生变化。一旦其他的某个线程改变了条件变量，他将通知相应的条件变量唤醒一个或多个正被此条件变量阻塞的线程。总的来说**互斥锁是线程间互斥的机制，条件变量则是同步机制。**







###### 互斥锁和读写锁、自旋锁、CAS原理

**互斥锁和读写锁的区别：**

对于**互斥锁，无论是读线程还是写线程，任意时刻都只能有一个线程获得锁资源**，加锁失败的线程会被阻塞，直到锁资源被释放。

**读写锁是比互斥锁粒度更小的锁**，它是一把锁具有读锁和写锁两种状态，**它允许多个线程同时持有读锁**，这样可以提高共享资源的访问效率，增加读操作的并发性，但是在写锁状态下只允许有一个线程可以持有，所以**读写锁适合于读多写少的场景。**



**互斥锁和自旋锁的区别：**

(1)互斥锁**加锁失败以后**，线程会释放cpu资源，让给其他线程，自己被阻塞。

​    自旋锁加锁失败以后，线程会忙等待，不停的去检查锁资源是否被释放，直到锁被释放，该线程获得锁资源。

(2)**互斥锁的开销比自旋锁大**，如果对互斥锁加锁失败，需要进行线程切换来应对，操作系统会把加锁失败的线程由运行态设置为睡眠态，再把CPU切换给其他线程运行，等到锁被释放了，操作系统把处于睡眠态的阻塞线程变为就绪态，再把CPU切换给该线程使用。

​    而**自旋锁一般是通过CPU提供的CAS函数在用户态完成加锁和解锁操作的，开销比较小。**



**CAS原理**

CAS(比较并交换）比较并交换。自旋锁或乐观锁的核心操作实现就是CAS。

**原理：**CAS是一个原子指令，一般是用在多线程环境中实现同步。 它包括三个操作数：**内存位置(V)、预期原值(A)、新值(B)**。**如果内存位置的值与预期原值相匹配(说明没被其他线程修改，现在可改)，那么处理器会自动将该位置值更新为新值，否则的话，不做任何操作**。这个过程是作为**单个原子操作完成**的。原子性保证了这个新值是最新的，并且这个新值在被读取到操作完成过程中不会被其它线程修改。

**使用场景**

- CAS 适合简单对象的操作，比如布尔值、整型值等
- CAS 适合冲突较少的情况，如果太多线程在同时自旋，那么长时间循环会导致 CPU 开销很大

**问题**

CAS 存在一个问题，就是一个值从 A 变为 B ，又从 B 变回了 A，这种情况下，CAS 会认为值没有发生过变化，但实际上是有变化的。对此，并发包下倒是有 `AtomicStampedReference` 提供了根据版本号判断的实现，可以解决一部分问题。





###### 条件变量

pthread_cond_wait被调用的时候会阻塞，所以一般流程是将线程放在条件变量的请求队列后，**内部解锁**；线程等待被pthread_cond_broadcast信号唤醒或者pthread_cond_signal信号唤醒，唤醒后去竞争锁；若竞争到互斥锁，**内部再次加锁**。

使用方式：

```c++
pthread_mutex_lock(&mutex);
while(线程执行条件是否成立)
{
    pthread_cond_wait(&cond, &mutex);
}

pthread_mutex_unlock(&mutex);
```

使用前先上锁是为了保证线程安全，因为多线程访问所以要上锁。

pthread_cond_wait内部会解锁是因为pthread_cond_wait被调用的时候会阻塞，此时它还有锁，所以要先解锁别的线程才能访问公共资源。

内部为什么要再加锁？是因为解锁了，别的锁获得了互斥锁，所以再上锁，保证线程安全。







##### HTTP长短连接、应用场景

**区别：**

**HTTP短连接**

- 在 HTTP/1.0 中默认使⽤短连接。也就是说，客户端和服务器每进⾏⼀次 HTTP 操作，就建⽴⼀次连接，任务结束就中断连接。

- 当客户端浏览器访问的某个 HTML 或其他类型的 Web 页中包含有其他的 Web 资源（如 JavaScript ⽂件、图像⽂件、CSS ⽂件等），每遇到这样⼀个 Web 资源，浏览器就会重新建⽴⼀个 HTTP 会话。


**HTTP长连接**

- ⽽从 HTTP/1.1 起，默认使⽤长连接，⽤以保持连接特性。使⽤长连接的 HTTP 协议，会在响应头加⼊这行代码：Connection:keep-alive

- 在使用长连接的情况下，当⼀个⽹页打开完成后，客户端和服务器之间⽤于传输 HTTP 数据 的 TCP 连接不会关闭，客户端再次访问这个服务器时，会继续使⽤这⼀条已经建⽴的连接。

- Keep-Alive 不会永久保持连接，它有⼀个保持时间，可以在不同的服务器软件（如 Apache）中设定这个时间。实现长连接需要客户端和服务端都⽀持长连接。 HTTP 协议的长连接和短连接，实质上是 TCP 协议的长连接和短连接。


**应用场景**：

- 长连接多⽤于操作频繁，点对点的通讯，⽽且连接数不能太多情况。每个 TCP 连接都需要三步握⼿，这需要时间，如果每个操作都是先连接，再操作的话那么处理速度会降低很多， 所以每个操作完后都不断开，下次处理时直接发送数据包就 OK 了，不⽤建⽴ TCP 连接。

- 例如： **数据库的连接⽤长连接**， 如果⽤短连接频繁的通信会造成 socket 错误，⽽且频繁的 socket创建也是对资源的浪费。

- ⽽像 WEB ⽹站的 http 服务⼀般都⽤短连接，因为长连接对于服务端来说会耗费⼀定的资源，⽽像Web⽹站这么频繁的成千上万甚⾄上亿客户端的连接⽤短连接会更省⼀些资源， 如果⽤长连接，⽽且同时有成千上万的⽤户，如果每个⽤户都占⽤⼀个连接的话，那可想⽽知吧。所以并发量⼤，但每个⽤户无需频繁操作情况下需用短连接。


通过**解析http请求头中的信息**，通过**connection字段判断是否为长连接状态**，如果是长连接，标记m_linger为true,在后续传输字节为0时，重新初始化http连接，不进行释放。

> - CHECK_STATE_HEADER
> - - 调用parse_headers函数解析请求头部信息
>   - 判断是空行还是请求头，若是空行，进而判断content-length是否为0，如果不是0，表明是POST请求，则状态转移到CHECK_STATE_CONTENT，否则说明是GET请求，则报文解析结束。
>   - 若解析的是请求头部字段，则主要分析connection字段，content-length字段，其他字段可以直接跳过，各位也可以根据需求继续分析。
>   - **connection字段判断是keep-alive还是close，决定是长连接还是短连接**
>   - **content-length字段，这里用于读取post请求的消息体长度**



##### 死锁、解决、死锁检测

**多个进程在执行过程中，因为争夺资源出现了进程相互之间无限期阻塞等待的状态就是死锁。**

**产生死锁：**

产生死锁必须满足四个条件：

(1)**互斥条件**，这个资源一次只能一个进程使用，其他进程想使用需等待该进程释放。

(2)**占用等待条件**，每个进程都会占用一个资源，同时每个进程还在等待另外一个被其他进程所占用的资源。

(3)**非抢占条件**，进程被分配的资源，只能自己释放，不能被强制抢占

(4)**循环等待条件**，多个进程之间形成了一个环形的资源等待的关系。



**解决死锁：**

**解决死锁只需要破坏其中一个条件就行：**

**最常见的就是使用资源有序分配法，来破环循环等待条件**。

还可以资源一次性全部分配，让每个进程都得到他想要的资源；或者终止某些进程，释放它占有的资源



**死锁检测：**

**可以gdb调试死锁**：先attach进程，(调试正在运行的进程)，再info threads看各个线程的状态看是哪些线程在等待锁，再进入线程使用bt命令，看线程堆栈情况，可以查看具体哪些函数有问题再去查看代码

**检测**：可以给每个进程和每个资源都制定唯一编号，创建一张资源分配表，记录各进程与占用资源之间的关系。再创建一张进程等待表，记录各进程与要申请资源之间的关系，如果出现环路等待情况，就是发生了死锁。











#### 3  I/O多路复用

##### LT与ET区别

1. LT：⽔平触发模式，  **LT模式下只要内核缓冲区还有数据可读便会用epoll_wait提醒**（哪怕已经提醒过，针对同一事件可以多次提醒）。

    - 当被监控的 Socket 上有可读事件发生时，**服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束**。

2. ET：使用边缘触发模式ET时，当被监控的 Socket 描述符上有可读事件发生时，**服务器端只会从 epoll_wait 中苏醒一次**，每一次事件到来只通知一次（针对一个事件只提醒一次而不是提醒多次），即使进程没有调用 read 函数从内核读取数据，也依然只苏醒一次，缓冲区数据如果不一次性读完，不会再通知，因此要使用while循环读取缓冲区直到返回-1和EAGAIN来判断是否读完所有数据；

    **每个使用ET模式的文件描述符都应该是非阻塞**的，如果是阻塞的那么读写事件会因为没有后续的事件而一直处于阻塞状态，**使用 I/O 多路复用时，最好搭配非阻塞 I/O 一起使用**。

 **一般来说，ET的效率比LT的效率要高，因为边缘触发可以减少 epoll_wait 的系统调用次数，系统调用也是有一定的开销的的，毕竟也存在上下文的切换。**

 项目中ET和LT最大不同就是LT干事情只干一次，ET是while循环干直到出问题break或者return false。



**epolloneshot事件（只允许一个线程处理数据）**

**epolloneshot事件:**即使**可以使用 ET 模式**，一个socket 上的某个事件还是可能被触发多次。这在并发程序中就会引起一个问题。**比如一个线程在读取完某个 socket 上的数据后开始处理这些数据，而在数据的处理过程中该 socket 上又有新数据可读（EPOLLIN 再次被触发）**，此时另外一个线程被唤醒来读取这些新的数据。于是就出现了两个线程同时操作一个 socket 的局面。一个socket连接在任一时刻都只被一个线程处理，可以使用 epoll 的 EPOLLONESHOT 事件实现。

  对于注册了 EPOLLONESHOT 事件的文件描述符，**操作系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次**，**除非我们使用 epoll_ctl 函数重置该文件描述符上注册的 EPOLLONESHOT 事件**。这样，当一个线程在处理某个 socket 时，其他线程是不可能有机会操作该 socket 的。但反过来思考，注册了 EPOLLONESHOT 事件的 socket 一旦被某个线程处理完毕， **该线程就应该立即重置这个 socket 上的 EPOLLONESHOT 事件，以确保这个 socket 下一次可读时，其 EPOLLIN 事件能被触发**，进而让其他工作线程有机会继续处理这个 socket。



  **EPOLLRDHUP**

  - 通过EPOLLRDHUP属性，来**判断是否对端已经关闭，这样可以减少一次系统调用**。在2.6.17的内核版本之前，只能再通过调用一次recv函数返回0来判断对端是否关闭。如果是-1，其errno为EAGAIN，说明没有数据读了，需要等待。





###### LT、ET适用场景

LT适用于并发量小的情况，ET适用于并发量大的情况。

- **原因：**
- ET在通知用户**之后**，就会将fd从就绪链表中删除，
- 而LT不会，它会一直保留，这就会导致随着fd增多，就绪链表越大，每次都要从头开始遍历找到对应的fd，所以并发量越大效率越低。ET因为会删除所以效率比较高。

**解决LT的缺点：**

- LT模式下，**可写状态的fd会一直触发事件**，该怎么处理这个问题

- 方法1：每次要写数据时，将fd绑定EPOLLOUT事件，写完后将fd同EPOLLOUT从epoll中移除。

- 方法2：方法一中每次写数据都要操作epoll。如果数据量很少，socket很容易将数据发送出去。

- 可以考虑改成：**数据量很少时直接send**，数据量很多时在采用方法1.


###### LT、ET触发时机

**触发时机：**

- **水平触发（LT）**：在任何时候只要满足条件（例如输入缓冲区有数据），就会触发通知。
- **边缘触发（ET）**：只有当状态发生变化时（例如输入缓冲区由无数据变为有数据），才会触发通知。

**LT（水平触发）模式：**

- 默认的工作模式。
- 当epoll_wait()检测到其监视的文件描述符上有事件发生并将这个事件通知给应用程序后，它会继续通知这个事件，直到相应的文件描述符被处理。也就是说，只要有数据可读，epoll_wait()就会不断地返回该文件描述符。
- LT模式更加友好，因为它不会因为一次未处理完就不再通知事件，减少了事件遗漏的可能性。

**ET（边缘触发）模式：**

- 高性能模式。
- 在这种模式下，当epoll_wait()检测到其监视的文件描述符上的事件发生并将这个事件通知给应用程序后，它不会再次通知此事件直到该事件的状态再次发生变化。换句话说，它只会告诉应用程序状态变化了一次。
- 在ET模式下，必须非阻塞地读取数据，直到返回EAGAIN错误（表示没有更多数据可以读取），以确保处理所有的事件，否则可能会错过某些数据。





###### ET和LT什么时候触发可写事件

**EPOLLOUT可写事件**通常被用来表示对应的文件描述符已经准备好进行写操作。具体地，在网络编程中，当socket的发送缓冲区有足够的空间可以存放新的数据时，就会触发EPOLLOUT事件。

对于边沿触发（ET，Edge Triggered）模式和水平触发（LT，Level Triggered）模式在触发EPOLLOUT时有些不同。

1. **水平触发（LT）模式下**：**只要socket的发送缓冲区有足够的空间可以存放新的数据，epoll就会返回EPOLLOUT事件。**也就是说，你可以任何时候写入数据，只要发送缓冲区没有满。如果你持续地忽视这个事件，epoll会持续地通知你。
2. **边沿触发（ET）模式下**：**EPOLLOUT事件只在socket的发送缓冲区从无法写入数据（例如，缓冲区满）到可以写入数据（例如，缓冲区有足够空间）的转换时触发一次。**这就意味着，如果你看到了EPOLLOUT事件，然后写入了一些数据，但是没有填满整个缓冲区，那么你可能需要自己记住这个状态，**因为epoll不会再次通知你可以进行写操作。**

这两种模式各有优点，具体使用哪种模式取决于你的具体需求和应用场景。总的来说，水平触发模式的行为更加直观和简单，但**边沿触发模式在处理大量并发连接时可以提供更高**的效率。



###### UDP有LT和ET吗

-  UDP也可以使用select/epoll，因为select等IO多路复用的操作对象就是文件操作符fd；
-  **大多数情况下，TCP服务器是并发的，UDP的服务器是迭代的。**因为 UDP 是非面向连接的，没有一个客户端可以老是占住服务端，不需要对每个应用保持一个线程。只要处理过程不是死循环，或者耗时不是很长，服务器对于每一个客户机的请求在某种程度上来说是能够满足
-  所以说，UDP没有必要使用多路复用。因此也就没有LTET使用之说，UDP使用recvfrom进行接受数据

**UDP需要LT和ET：**

- 使用水平触发模式时，当被监控的 Socket 上有可读事件发生时，**服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束**
- 使用边缘触发模式时，当被监控的 Socket 描述符上有可读事件发生时，**服务器端只会从 epoll_wait 中苏醒一次**，即使进程没有调用 read 函数从内核读取数据，也依然只苏醒一次，缓冲区数据如果不一次性读完，不会再通知，因此要使用while循环读取缓冲区直到返回-1和EAGAIN来判断是否读完所有数据；**每个使用ET模式的文件描述符都应该是非阻塞**的，如果是阻塞的那么读写事件会因为没有后续的事件而一直处于阻塞状态，**使用 I/O 多路复用时，最好搭配非阻塞 I/O 一起使用**
- 一般来说，边缘触发的效率比水平触发的效率要高，因为边缘触发可以减少 epoll_wait 的系统调用次数，系统调用也是有一定的开销的的，毕竟也存在上下文的切换





###### LT可以fd阻塞，ET不能

- LT模式不需要每次读完数据，只要有数据可读，epoll_wait()就会⼀直通知。所以 LT模式下去读的话，内核缓冲区肯定是有数据可以读的，不会造成没有数据读而阻塞的情况。
- 因为ET模式是当fd有可读事件时，epoll_wait()只会通知⼀次，如果没有⼀次把数据读完，那么要到下⼀次fd有可读事件epoll才会通知。⽽且在ET模式下，在触发可读事件后，需要循环读取信息，直到把数据读完。如果把这个fd设置成阻塞，**数据读完以后read()就阻塞在那**了。无法进行后续请求的处理。

- 为什么ET模式下一定要设置非阻塞。

-  因为ET模式下是**无限循环读**，直到出现错误为EAGAIN或者EWOULDBLOCK，这两个错误表示socket为空，不用再读了，然后就停止循环了。**如果是阻塞，循环读在socket为空的时候就会阻塞到那里，主线程的read（）函数一旦阻塞住，当再有其他监听事件过来就没办法读了，给其他事情造成了影响，所以必须要设置为非阻塞。**

- **备用：**因为ET模式下我们必须要用while循环来把客户端文件描述符这次可读的数据全都读出来，否则之后epoll_wait将不会再通知我们去处理这部分数据，正是因为我们的读操作是在while循环中进行的所以一定要将客户端文件描述符设置为非阻塞，**如果设置为阻塞，while循环中的readv函数读不到数据就会将线程一直阻塞在这里，不往下进行，直到再有数据到达，才继续执行，非阻塞下readv会返回一个值，然后继续往下进行。**



###### ET 用非阻塞 IO

　当使用边缘触发（ET）模式的 `epoll` 时，通常需要配合非阻塞I/O操作来使用。因为ET模式下的事件通知是状态变化时只通知一次，如果使用阻塞I/O，可能会在没有处理完所有数据的情况下阻塞程序，导致部分数据还在缓冲区中时，程序没有继续的读取操作，因此丢失了后续的事件通知。

在ET模式下，正确的做法是在文件描述符上执行非阻塞读写操作。这意味着当你尝试从socket读数据或向socket写数据时，如果没有数据可读或者不能立即写入，调用将立即返回而不是阻塞等待。这样，你的程序就能在一个循环中读取所有的数据，直到遇到`EAGAIN`或`EWOULDBLOCK`错误，这两个错误表明所有数据已经被读取完毕。



个人认为使用 阻塞IO 潜在的问题在于，使用 阻塞 IO 去读的时候，会导致在没有数据可读的时候，导致当前工作线程阻塞不工作。而 ET 模式与 LT 模式都是在有数据的情况下触发，只不过触发的时机不同。假定读缓冲区 50b，而收到的包为 100b，有如下情况：

阻塞 IO

　　LT 模式下，由于只要有数据就会触发读，因此不会有问题，但是在 ET 模式下，由于在新的数据到来之前，都不会触发读事件，因此会导致剩下的 50b 没有读取到，所以为了保证能够读取到完整的包，需要使用 while(1) 之类的循环去读，这就会导致在数据读完之后，最后一次 read 阻塞，因为所有的数据都已经读完了。

- 非阻塞 IO　　

　　在 LT 模式下，使用非阻塞 IO 的效果与阻塞 IO 差不多，在 ET 模式下，处理的逻辑与上面类似，但是由于使用的 非阻塞 IO ，因此不会导致最后一次 read 阻塞，而是会返回 EAGAIN 。

　









##### epoll相关

###### **epoll 底层如何实现**

- Linux epoll机制是通过红黑树和双向链表实现的。 
- 一个是用于**存放待检测文件描述符的红黑树**，
- 另一个是**存放有事件发生的就绪文件描述符的双向链表**。
- 首先通过epoll_create()系统调用在内核中创建一个eventpoll类型的句柄，其中包括红黑树根节点和双向链表头节点。
- 然后通过epoll_ctl()系统调用，向epoll对象的红黑树结构中添加、删除、修改感兴趣的事件，返回0标识成功，返回-1表示失败。
- 最后通过epoll_wait()系统调用判断双向链表（就绪文件描述符）是否为空，如果为空则阻塞。当文件描述符状态改变，fd上的回调函数被调用，该函数将fd加入到双向链表中，此时epoll_wait函数被唤醒，返回就绪好的事件。







###### **选择epoll原因**

- 1、epoll直接在内核区创建事件表，可减少将事件表从用户态拷贝到内核态的开销。
-  2、不用每次都遍历事件表来检测就绪事件，就绪的事件会自动存放到epoll_wait传出参数的那个数组里面

-  3、能够检测的文件描述符没有最大数量限制

-  4、支持ET和LT两种模式




###### epoll使用红黑树

- 红黑树是个高效的数据结构，增删查一般时间复杂度是 O(logn)，通过对这棵黑红树的管理，不需要像 select/poll 在每次操作时都传入整个 Socket 集合，**减少了内核和用户空间大量的数据拷贝和内存分配**。
- 每一个epoll对象都有一个独立的eventpoll结构体，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来



###### epoll_ctl线程安全

- **epoll是通过锁来保证线程安全的, epoll中粒度最小的自旋锁ep->lock(spinlock)用来保护就绪的队列, 互斥锁ep->mtx用来保护epoll的重要数据结构红黑树**
- epoll_wait和epoll_ctl都是线程安全的, 前者带acquire语意, 后者带release语意, 换句话说, 如果epoll_wait后能拿到某个新fd的事件, 那么对应的epoll_ctl前发生的内存修改都可见。
- 当你在一个线程中用 `epoll_wait` 等待事件时，它具有 acquire 语义，保证你看到的所有事件都是最新的状态，不会错过由其他线程通过 `epoll_ctl` 所做的更改。
- 当你在一个线程中用 `epoll_ctl` 修改 epoll 实例的监控列表时，它具有 release 语义，确保在这个调用之后，其他线程通过 `epoll_wait` 看到的将是包含了这些修改的最新事件列表。



###### epoll一定快吗

- 看事件的触发率，如果所有的一直都是触发的，那就不存在浪费了，因为处理的都是一定发生的，所以select也不一定不好
- **epoll不一定好**。
- 对于select和poll来说，所有文件描述符都是**在用户态被加入其文件描述符集合的**，**每次调用都需要将整个集合拷贝到内核态**；
- epoll则**将整个文件描述符集合维护在内核态，每次添加文件描述符的时候都需要执行一个系统调用。系统调用的开销是很大的，而且在有很多短期活跃连接的情况下，epoll可能会慢于select和poll由于这些大量的系统调用开销。**
- **当监测的fd数量较小，且各个fd都很活跃的情况下，建议使用select和poll；当监听的fd数量较多，且单位时间仅部分fd活跃的情况下，使用epoll会更好**。
- **epoll不好情况**
- **文件描述符数量较少**： 当需要监视的文件描述符数量较少时，`select` 或 `poll` 的性能与 `epoll` 相差不大，因为它们在小规模上的开销并不高。此时，`epoll` 的优势不明显。
- **短连接应用**： 如果应用程序主要处理短暂的连接，频繁地进行添加和删除文件描述符操作，`epoll` 的性能优势可能被降低。因为在 `epoll` 中，每次新连接建立时，需要执行额外的 `epoll_ctl` 调用来注册新的文件描述符。
- **单线程应用**： 在单线程应用中，由于不存在线程切换的开销，`select` 或 `poll` 可能已经足够高效。而 `epoll` 的优势主要是在多线程环境下减少锁竞争和减少系统调用的次数。
- **I/O操作非常快**： 如果I/O操作非常快，或者I/O的延迟非常低，比如在高速本地网络或者内存中的I/O，`epoll` 相较于其他机制带来的性能提升可能不会那么明显。
- **使用不当**： 错误地使用 `epoll` 也可能导致性能问题，例如使用边缘触发（ET）模式但没有正确地处理非阻塞I/O可能导致错过事件，需要额外的错误处理代码来保证程序的正确性。
- **系统调用和上下文切换**： 当事件非常频繁时，不断地进行系统调用（`epoll_wait`）和从内核态切换到用户态可能会导致性能下降。



###### **epoll判数据已读取完**

- epoll ET模式，才需要关注数据是否读取完毕了，**必须要一次性将数据读取完，使用非阻塞I/O，读取到出现eagain**。使用select或者epoll的LT模式，不用关注，select/epoll检测到有数据可读去读就OK了

- 针对TCP，调用recv方法，根据recv的返回值。如果返回值小于我们设定的recv buff的大小，那么就认为接收完毕
- TCP、UDP都适用，将socket设为NOBLOCK状态（使用fcntl函数），然后select该socket可读的时候，使用read/recv函数读取数据。当返回值为-1，并且errno是EAGAIN或EWOULDBLOCK的时候，表示数据读取完毕。

**读取数据读完：**

- **ET边缘触发模式**
- epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序必须立即处理该事件
  - **必须要一次性将数据读取完，使用非阻塞I/O，读取到出现eagain**。

- **LT水平触发模式**
- epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序可以不立即处理该事件。
  - 当下一次调用epoll_wait时，epoll_wait还会再次向应用程序报告此事件，直至被处理。



##### I/O多路复用

###### **I/O多路复用技术**

- IO复用指的是应用程序通过**IO复用函数**向内核注册一组事件，内核通过IO复用函数通知应用程序就绪事件进程可以通过一个系统调用函数从内核中获取多个事件。

  I/O 多路复用通过一种“事件通知”机制，允许单个进程/线程监视多个 I/O 流（如 sockets）。当某个 I/O 流准备好读写时，该流会通知操作系统，操作系统再将这个事件通知给应用程序。这样，应用程序可以在多个 I/O 流之间高效地“切换”，只在数据准备好时才进行操作，避免了阻塞等待，并且减少了对多线程/多进程的需求。

###### **select,poll,epoll区别**

- 首先select,poll,epoll都是IO多路复用机制，主要是为了解决：阻塞网络IO如果需要支持多个客户端只能使用多进程和多线程的模式，操作系统负担大的问题；采用一个进程维护多个socket，进程可以通过一个系统调用函数从内核中获取多个事件
- **select** 
- 实现多路复用的方式是，将已连接的 Socket 都放到一个**文件描述符集合**，然后调用 select 函数**将文件描述符集合拷贝到内核里**，然后通过遍历文件描述符判断是否有事件发生，有的话还需要**将整个文件描述符拷贝到用户态**中，用户再通过遍历找到对应的Socket
- **poll** 
- 和 select 并没有太大的本质区别，**都是使用线性结构存储进程关注的 Socket 集合，因此都需要遍历文件描述符集合来找到可读或可写的 Socket，也需要在用户态与内核态之间拷贝文件描述符集合**，这种方式随着并发数上来，性能的损耗会呈指数级增长
- **epoll**
- 则解决了select和poll的轮询缺点，**使用了一组函数来完成任务**而不是单个函数
  - epoll 在内核里使用**红黑树**来关注进程所有待检测的 Socket，需要监控的 socket 通过 `epoll_ctl()` 函数加入内核中的红黑树里，不需要像 select/poll 在每次操作时都传入整个 Socket 集合，**减少内核和用户空间大量的数据拷贝和内存分配**(epoll_wait 调用了 `__put_user` 函数，将数据从内核拷贝到用户空间)
  - epoll 使用事件驱动的机制，内核里维护了**一个链表来记录就绪事件**，只**将有事件发生的 Socket 集合传递给应用程序**，当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个集合，大大提高了检测的效率
  - 而且，epoll 支持边缘触发和水平触发的方式，而 select/poll 只支持水平触发，一般而言，边缘触发的方式会比水平触发的效率高











###### IO多路复用是同步还是异步

- **从 I/O 层面来看的话， epoll 是同步的**，因为 `epoll()` 本身就需要阻塞等待操作系统返回值。
- **从消息处理层面来看， epoll 是异步的**，因为每个调用 `epoll()` 的事件，不需要阻塞等待 epoll 的返回值，只需要 epoll 返回后，通知事件即可。
- **同步io和异步io区别？为什么不⽤异步io**
- **同步I/O：内核向应用程序通知就绪事件，由应用程序自身来完成I/O的读写操作**
  **异步I/O：由内核来完成I/O的读写后向应用程序通知完成事件** 
- linux本身提供的**asio⽬前只⽀持⽂件fd，不⽀持⽹络fd**，但是可以⽤⼀个线程去模拟异步io的操作




3.8 怎么判断客户断开连接

项目中检测用于通信的文件描述符对应的epoll事件是**EPOLLRDHUP**时，就说明客户端关闭了连接



3.9 主线程调用不同函数进行注册，两次注册是否没有必要，直接主线程循环读取然后封装放请求队列不就行了么？

不对，如果数据一直没来，直接进行循环读取就会持续在这里发生阻塞，这就是同步IO的特点，所以一定要注册一下然后等通知，这样就可以避免长期阻塞等候数据。

同步：它主线程使用epoll向内核注册读事件。但是这里内核不会负责将数据从内核读到用户缓冲区，最后还是要靠主线程也就是用户程序read（）函数等负责将内核数据循环读到用户缓冲区。

Epoll对文件操作符的操作有两种模式: `LT`(电平触发)、`ET`(边缘触发)，二者的区别在于当你调用`epoll_wait`的时候内核里面发生了什么：

> sleep和wait的区别：
> 1、sleep是Thread的静态方法，wait是Object的方法，任何对象实例都能调用。
> 2、**sleep不会释放锁，它也不需要占用锁。wait会释放锁**，但调用它的前提是当前线程占有锁(即代码要在synchronized中)。当调用wait()方法的时候，线程会放弃对象锁，进入等待此对象的等待锁定池，只有针对此对象调用notify()方法后本线程才进入对象锁定池准备。
> 3、它们都可以被interrupted方法中断。







##### 什么是零拷贝

**零拷贝技术，因为我们没有在内存层面去拷贝数据，也就是说全程没有通过 CPU 来搬运数据，所有的数据都是通过 DMA 来进行传输的。**。

零拷贝技术的文件传输方式相比传统文件传输的方式，减少了 2 次上下文切换和数据拷贝次数，**只需要 2 次上下文切换和数据拷贝次数，就可以完成文件的传输，而且 2 次的数据拷贝过程，都不需要通过 CPU，2 次都是由 DMA 来搬运。**

所以，总体来看，**零拷贝技术可以把文件传输的性能提高至少一倍以上**。











######  

#### 4.并发模型

1. 【中兴⼀⾯】讲⼀下项目中的⽹络框架？为什么要采⽤这个框架？有了解其他的⽅案吗？为什么不采⽤？
2. 【经纬恒润⼀⾯】为什么选择主从reactor+线程池的模式（将IO操作和业务处理解耦）？epoll配合什么IO使
   ⽤？简单介绍下epoll？了解proactor吗，讲⼀下区别？
3. 【荣耀⼆⾯】介绍⼀下项⽬中的⽹络框架？为什么选择这个⽹络框架？

#####  reactor、proactor、模拟Proactor

1. Reactor 是非阻塞同步网络模式，感知的是就绪可读写事件。在每次感知到有事件发生（比如可读就绪事件）后，就需要应用进程主动调用read 方法来完成数据的读取，也就是要应⽤进程主动将 socket 接收缓存中的数据读到应用进程内存中，这个过程是**同步的**，读取完数据后应用进程才能处理数据。
2. Proactor 是异步网络模式， 感知的是已完成的读写事件。在发起异步读写请求时，需要传⼊数据缓冲区的地址（⽤来存放结果数据）等信息，这样系统内核才可以⾃动帮我们把数据的读写⼯作完成，这⾥的读写⼯作全程由操作系统来做，并不需要像 Reactor 那样还需要应⽤进程主动发起 read/write 来读写数据，操作系统完成读写⼯作后，就会通知应用进程直接处理数据。

**Reactor模式**

- reactor模式中，主线程(I/O处理单元)只负责监听文件描述符上是否有事件发生，有的话立即通知工作线程(逻辑单元 )，读写数据、接受新连接及处理客户请求均在工作线程中完成。通常由同步I/O实现。

- 工作流程：主线程往 epoll 内核事件表中注册 socket 上的读就绪事件。主线程调用 epoll_wait 等待 socket 上有数据可读。当 socket 上有数据可读时， epoll_wait 通知主线程。主线程则将 socket 可读事件放入请求队列。
- 睡眠在请求队列上的某个工作线程被唤醒，它从 socket 读取数据，并处理客户请求，然后往 epoll内核事件表中注册该 socket 上的写就绪事件。主线程调用 epoll_wait 等待 socket 可写。当 socket 可写时，epoll_wait 通知主线程。主线程将 socket 可写事件放入请求队列。睡眠在请求队列上的某个工作线程被唤醒，它往 socket 上写入服务器处理客户请求的结果。

**模拟Proactor模式**

- proactor模式中，主线程和内核负责处理读写数据、接受新连接等I/O操作，工作线程仅负责业务逻辑，如处理客户请求。通常由异步I/O实现。由于异步I/O并不成熟，实际中使用较少，使用同步I/O模拟实现proactor模式。使用同步 I/O 方式模拟出 Proactor 模式。

- 原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一”完成事件“。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。


**Proactor模式**

- Proactor 模式将所有 I/O 操作都交给主线程和内核来处理（进行读、写），工作线程仅仅负责业务逻辑。使用异步 I/O 模型（以 aio_read 和 aio_write 为例）实现的 Proactor 模式的工作流程是：
  主线程调用 aio_read 函数向内核注册 socket 上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序（这里以**信号**为例）。主线程继续处理其他逻辑。当 socket 上的数据被读入用户缓冲区后，内核将向应用程序发送一个信号，以通知应用程序数据已经可用。应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求后，调用 aio_write 函数向内核注册 socket 上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序。
- 主线程继续处理其他逻辑。当用户缓冲区的数据被写入 socket 之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭 socket。





**主从 反应器 ( Reactor ) 多线程 模式 :**

1 . 主反应器 ( MainReactor ) : 运行在独立的 Reactor 主线程中 , 该线程中只负责与客户端的连接请求 ;
2 . 从反应器 ( SubReactor ) : 运行在独立的 Reactor 子线程中 , 该线程中负责与客户端的读写操作 ; 在该子线程中 , 从反应器 ( Reactor ) 监听多个客户端的请求事件 , 如果监听到客户端的数据发送事件 , 将对应的业务逻辑转发给 处理器 ( Handler 进行处理 ) ;
3 . 主反应器 ( MainReactor ) 与 从反应器 ( SubReactor ) 对应关系 : 主反应器在服务器端只有一个 , 但是从反应器在服务器端 , 不只一个 , 可以运行多个 Reactor 子线程 , 每个子线程中运行一个 从反应器 ;
之后的操作与 单反应器 ( Reactor ) 多线程的处理机制一样 ;
4 . 服务器端 处理者 ( Handler ) : Handler **只负责响应业务处理的请求事件** , 不处理具体的与客户端交互的业务逻辑 , 因此不会长时间阻塞 , 其调用 read 方法读取客户端数据后 , 将业务逻辑交给 线程池 ( Worker ) 处理相关业务逻辑 , 处理完毕后 , 将结果返回 , Handler 将该结果写出到客户端 ；
5 . 服务器端 线程池 ( Worker ) : 接收 处理者 ( Handler ) 的请求 , 为将**请求对应业务逻辑操作** , 分配给某个独立线程完成 , 执行完成后的结果再次返回给处理者 ( Handler ) ,
**( Handler 读取客户端数据 -> Worker 线程池分配线程执行业务处理操作 -> Handler 将结果回送给客户端 )**
6 . 从反应器 ( SubReactor ) 与 线程池 ( Worker ) 对应关系 : 每个从反应器对应一个线程池 , 也就是说每个主反应器 对应多个 从反应器及配套的线程池 。



###### 为什么不用Proactor

-  在 Linux 下的异步 I/O 是不完善的， aio 系列函数是由 POSIX 定义的异步操作接⼝，不是真正的操作系统级别⽀
- 持的，⽽是在⽤户空间模拟出来的异步，并且仅仅⽀持基于本地⽂件的 aio 异步操作，⽹络编程中的 socket 是不⽀持的，也有考虑过使⽤模拟的proactor模式来开发，但是这样需要浪费⼀个线程专⻔负责 IO 的处理。
- ⽽ Windows ⾥实现了⼀套完整的⽀持 socket 的异步编程接⼝，这套接⼝就是 IOCP ，是由操作系统级别实现的异步 I/O，真正意义上异步 I/O，因此在 Windows ⾥实现高性能网络程序可以使⽤效率更⾼的 Proactor ⽅案。





###### reactor模型单多线程

Reactor模型是⼀个针对同步I/O的⽹络模型，主要是使⽤⼀个reactor负责监听和分配事件，将I/O事件分派给对应的Handler。新的事件包含连接建⽴就绪、读就绪、写就绪等。reactor模型中又可以细分为单reactor单线程、单reactor多线程、以及主从reactor模式。

1. **单reactor单线程模型**就是使⽤ I/O 多路复⽤技术，当其获取到活动的事件列表时，就在reactor中进行读取请
    求、业务处理、返回响应，这样的好处是整个模型都使⽤⼀个线程，不存在资源的争夺问题。但是如果⼀个事
    件的业务处理太过耗时，会导致后续所有的事件都得不到处理。
2. **单reactor多线程**就是⽤于解决这个问题，这个模型中reactor中只负责数据的接收和发送，reactor将业务处
    理分给线程池中的线程进⾏处理，完成后将数据返回给reactor进⾏发送，避免了在reactor进⾏业务处理，但
    是 IO 操作都在reactor中进⾏，容易存在性能问题。⽽且因为是多线程，线程池中每个线程完成业务后都需要
    将结果传递给reactor进⾏发送，还会涉及到共享数据的互斥和保护机制。
3. **主从reactor就是将reactor分为主reactor和从reactor，主reactor中只负责连接的建⽴和分配，读取请求、**
    **业务处理、返回响应等耗时的操作均在从reactor中处理，能够有效地应对⾼并发的场合。**





###### **IO是什么**

- 在计算机中，输入/输出（即IO）是指信息处理系统（比如计算机）和外部世界（可以是人或其他信息处理系统）的通信。输入是指系统接收的信号或数据，输出是指从系统发出的数据或信号。（数据从网卡或硬盘读到内核缓冲区）

**异步IO与IO多路复用区别：**

- 多路复用IO，阻塞IO，非阻塞IO都是同步IO，具体表现为，当事件发生时，会通知cpu，需要cpu中断来处理到来的数据，该过程会占用cpu的时间（在 read 调用时，**内核将数据从内核空间拷贝到用户空间的过程都是需要等待的，也就是说这个过程是同步的**）
- 异步IO则是由内核来接收数据的处理过程，完成之后通知cpu，可以直接获取数据，少了处理数据的时间（内核数据准备好和数据从内核态拷贝到用户态这**两个过程都不用等待**，应用程序并不需要主动发起拷贝动作）





###### 五种I/O模型

**a.阻塞 ** 调用者调用了某个函数，等待这个函数返回，期间什么也不做，不停的去检查这个函数有没有返回，必须等这个函数返回才能进行下一步动作。

**b.非阻塞  (NIO)** 非阻塞等待，每隔一段时间就去检测IO事件是否就绪。没有就绪就可以做其他事。**非阻塞I/O执行系统调用总是立即返回，不管事件是否已经发生，若事件没有发生，则返回-1，此时可以根据 errno 区分这两种情况，对于accept，recv 和 send，事件未发生时，errno 通常被设置成 EAGAIN。**

**c.IO复用，** Linux 用 select/poll/epoll 函数实现 IO 复用模型，这些函数也会使进程阻塞**，但是和阻塞IO所不同的是这些函数可以同时阻塞多个IO操作。而且**可以同时对多个读操作、写操作的IO函数进行检测。直到有数据可读或可写时，才真正调用IO操作函数。作用：一次性可以检测多线程多进程

***IO复用和阻塞IO不一样，IO复用程序阻塞与IO复用系统调用，对IO本身读写是不阻塞的，但是阻塞IO程序阻塞在读写函数***

**d.信号驱动** Linux 用套接口进行信号驱动 IO，安装一个信号处理函数，进程继续运行并不阻塞，当IO事件就绪，进程收到**SIGIO 信号**，然后处理 IO 事件。内核在第一个阶段是异步，在第二个阶段是同步；与非阻塞IO的区别在于它提供了消息通知机制，不需要用户进程不断的轮询检查，减少了系统API的调用次数，提高了效率。

**e.异步** Linux中，可以调用 aio_read 函数告诉内核描述字缓冲区指针和缓冲区的大小、文件偏移及通知的方式，然后立即返回，当内核将数据拷贝到缓冲区后，再通知应用程序。

多路复用使内核们监听多个文件描述符，阻塞在监听的函数比如select， 拷贝数据也是阻塞的，多路复用只是防止进程在某个io阻塞后，不能及时处理其他io的事件。信号驱动则是先登记信号处理函数，当数据准备完毕后由内核发送信号给进程，让进程处理。信号驱动不阻塞在数据准备过程，但阻塞在数据拷贝，所以两者都是同步IO



###### 同步、异步IO

- **同步I/O指内核向应用程序通知的是就绪事件**，比如只通知有客户端连接，要求用户代码自行执行I/O操作。
- **异步I/O是指内核向应用程序通知的是完成事件**，比如读取客户端的数据后才通知应用程序，由内核完成I/O操作。

- **同步IO需要使用系统调用将内核中的数据拷贝到指定位置，异步IO则由内核自动将数据拷贝到指定内存中去。**



- **并发模式中的同步和异步：**
- 在并发模式中，同步和异步的主要区别是**功能完成的流程是否是顺序化的，是否需要等待** 
- 并发模式中的同步和异步：**同步指的是程序完全按照代码序列的顺序执行**，**异步指的是程序的执行需要由系统事件驱动**
- **同步：**当遇到阻塞任务时，会一直等待，直到该任务处理完成，程序完全按照代码顺序执行；
  **异步：**程序的执行需要由**系统事件驱动**，程序的执行是不确定的，没有顺序上的要求 
  按同步方式运行的称为同步线程；按异步方式运行的称为异步线程 



###### 半同步半异步+半同步半反应堆

**半同步/半异步**

- 同步线程用于处理客户逻辑，异步线程用于处理I/O事件

- 异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中，请求队列将通知某个工作在**同步模式的工作线程**来读取并处理该请求对象
- 以**Reactor事件处理模式**为例：

- **异步线程监听到客户请求**后，将其封装成**请求对象**并插入请求队列中；
  请求队列将通知某个工作在**同步模式的工作线程**来**读取并处理该请求对象**；
  具体选择哪个同步线程主要取决于请求队列的设计，如轮流选取的Round Robin算法、使用条件变量或信号量等随机选取





**半同步/半反应堆（Reactor）：**

- 在该模式中，**异步线程只有主线程一个**，由**主线程负责监听所有socket上的事件**
- **主线程充当异步线程，负责监听所有socket上的事件**。

- **若有新请求到来，主线程接收之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。**
- **如果连接socket上有读写事件发生，主线程从socket上接收数据，并将数据封装成请求对象插入到请求队列中。**
- **所有工作线程睡眠在请求队列上，当有任务到来时，通过竞争（如互斥锁）获得任务的接管权**。
- 这种竞争机制使得只有空闲的工作线程才有机会来处理新任务，避免了忙闲不均的情况。




**半同步/半反应堆模式的缺点：**

**主线程和工作线程共享请求队列**，**主线程**往请求队列中添加**新连接socket任务**，**工作线程**从请求队列中**取出任务是互斥**的操作，需要对请求队列进行加锁保护，**导致白白浪费CPU时间**；
每个工作线程在同一时间内只能处理一个客户请求，如果客户较多而工作线程较少，就会导致**任务队列中堆积大量的任务对象**，此时客户端的响应将越来越慢。而如果通过**增加工作线程**来解决这个问题，就会因为**大量的上下文切换**导致消耗大量的CPU时间



**两者区别：**

半同步/半异步**有多个异步线程**，而半同步/半反应堆模式只有**主线程一个异步线程**；
半同步/半反应堆模式将有**事件发生的socket**放进共享的请求队列，需要通过竞争来获得任务（半同步/半异步模式的请求队列是共享的吗？）







#### 5.⽇志系统

1. 异步⽇志系统是怎么实现异步的？业务线程的写⼊是怎么同步的？（加锁），了解⼀下无锁队列

2. 如果服务器在运行的过程中，实际存储的⽂件被其他用户修改了，会发⽣什么？有什么优化的想法吗？

  （想法⼀（被否决）：定时和实际⽂件对⽐；想法二（被否决）：进程修改后通知缓存；

     想法三：类似于内存页面的换入换出，每次从cache读⼊时，先判断实际⽂件去对⽐，可以对⽐最后修改时间或者通过摘要算法判断）

3. 日志系统的异步体现在什么地方？与同步的区别？为什么设计成双缓冲⽽不⽤更多的缓冲区？

    （前端缓冲不⾜时会⾃动扩展，但是双缓冲⾜够应付使⽤场景，因为⽇志只记录必要的信息，并不会太多）

4. 讲⼀下你的⽇志系统的怎么做的？异步指的是什么？异步和同步的区别？如果服务器崩溃的话，你的⽇志系统会发⽣什么？

  （未写⼊⽂件的所有⽇志都会丢失，但是会有时间戳）

  这个问题应该怎么解决？

  （如果是进程crash的话，可以使⽤⼀个⽇志进程来写⼊⽇志；如果是主机宕机的话可以考虑分布式⽇志，将⽇志分发到不同的主机上写⼊）



##### 为什么做⽇志系统

**日志是由服务器自动创建，并记录运行状态，错误信息，访问数据的文件。**在开发其他模块的时候主要用来记录模块的信息，通过日志判断模块功能是否正常。在服务器运行期间主要记录服务器设定的参数、新客户的建立与断开、异常事件的发生（并发数量达到上限、套接字初始化失败等）。便于对服务器运行过程中一些不明情况的分析。



##### 日志系统怎么做的

- 使用单例模式创建日志系统，对服务器运行状态、错误信息和访问数据进行记录，该系统可以实现按天分类，超行分类功能，可以根据实际情况分别使用同步和异步写入两种方式。

- 其中异步写入方式，使用双缓冲机制体现。

- 在多线程程序中写 Log 无非就是前端往后端写，后端往硬盘写，⾸先将LogStream 的内容写到了 AsyncLogging 缓冲区⾥，也就是前端往后端写，这个过程通过 append 函数实现，后端实现通过 threadfunc 函数，两个线程的同步和等待通过**互斥锁和条件变量**来实现，具体实现使⽤了双缓冲技术。

**双缓冲思路**、好处

- 准备两块 buffer 1 和 2; 

- 前端负责往 buffer 1 填数据(日志信息)；
- 后端负责把 buffer 2 的数据写入硬盘的文件。
- 当 buffer 1 写满之后，交换 1 和 2，让后端将 buffer 1 的数据写入文件，
- 而前端则往 buffer 2 填入新的日志信息，如此反复。

使⽤两个 buffer 的好处是在新建日志消息的时候不必等待磁盘⽂件操作，也避免每条新日志消息都触发后端日志线程。换句话说，前端不是将一条条日志消息分别送给后端，而是将多条⽇
志消息拼接成⼀个⼤的 buffer 传送给后端，相当于批处理，减少了线程唤醒的开销。

**异步方式采用双缓冲机制**，具有较高的并发能力。**异步日志，将所写的日志内容先存入阻塞队列，写线程从阻塞队列中取出内容，写入日志。可以提高系统的并发性能。**

- **用双缓冲：**好处是：在大部分的时间中，前台线程和后台线程不会操作同一个缓冲区，这也就意味着前台线程的操作，不需要等待后台线程缓慢的写文件操作(因为不需要锁定临界区)。通过双缓冲技术，很好地解决了生产者和消费者之间的异步操作和速度不匹配问题，提高了日志系统的整体吞吐率。

- 超行、按天分文件逻辑，具体的，日志写入前会判断当前day是否为创建日志的时间，行数是否超过最大行限制；
- 若为创建日志时间，写入日志，否则按当前时间创建新log，更新创建时间和行数；若行数超过最大行限制，在当前日志的末尾加count/max_lines为后缀创建新log。







日志系统代码实现

- 1、 logStream类的实现， logStream 类的主要作⽤是将各个类型的数据转换为 char 的形式放入字符数组中（也就是前端⽇志写⼊ Buffer A 的这个过程），⽅便后端线程写⼊硬盘。
- 2、logStream 类只持有⼀个缓冲区 Buffer ，然后重载对流式运算符 << 进⾏了重载，由于⽇志输⼊的类型繁多，为了后端写⼊⽅便（也为了缓冲区的格式统⼀），需要将输⼊的数据转换为 char 字符类型再进⾏写⼊。
- 3、AsynLogging 类，这个类的作⽤则是将从前端获得的 Buffer A 放⼊ 后端的 Buffer B中，并且将 Buffer B的内容最终写⼊到磁盘中（也就是整个后端所作的内容）
- 4、**参数：**
- 1. thread_ ：后端通过⼀个单独的线程来实现将⽇志写⼊⽂件的，这就是那个线程对象。

  2. currentBuffer_ 和 nextBuffer_ ：双缓冲区，减少前端等待的开销

  3. buffers_ ：缓冲区队列，实际写⼊⽂件的内容也是从这⾥拿的

- 5、**后台线程线程函数做的事情：**
- 1. 每隔3s，或者 Buffer B 满了，就将 Buffer B 放⼊ buffers_ （缓冲区队列）中，这⾥涉及到多个线程的同步，需要加互斥锁，还需要判断缓冲区队列是否已经满了。

  2. 最后再将 buffers_ （缓冲区队列）⾥的每个 Buffer 放到⽂件输出区 output⾥，再 flush ⼀下就可以了。
  3. 之后进⼀步将 logStream 类和 AsynLogging 类封装为 Logger 类就可以完成我们了日志库。

- 6、通过宏定义 #define LOG Logger(__FILE__,__LINE__).stream() 使得我们在使⽤日志写⼊语句 LOG << info; 之类的内容时，先构建⼀个临时的 Logger对象，在构造时生成详细的日志信息，由于临时对象会在语句结束后析构，我们可以在析构的时候再将日志真正地写⼊⽂件，来保证实现的简洁性



**什么时候切换写到另⼀个⽇志⽂件**

- 前⼀个 Buffer 已经写满了，则交换两个 Buffer（写满的 Buffer 置空）。
- ⽇志串写⼊过多，⽇志线程来不及消费，怎么办？直接丢掉多余的⽇志 Buffer，腾出内存，防⽌引起程序故障。
- **什么时候唤醒⽇志线程从** **Buffer** **中取数据？**
- 其一是超时，其⼆是前端写满了⼀个或者多个 Buffer。



###### 日志系统为什么异步

- 同步方式写入日志时会产生比较多的系统调用，若是某条日志信息过大，会阻塞日志系统，造成系统瓶颈。

- **异步方式采用双缓冲机制**，具有较高的并发能力。**异步日志，将所写的日志内容先存入阻塞队列，写线程从阻塞队列中取出内容，写入日志。可以提高系统的并发性能。**
  
- **用双缓冲：**在大部分的时间中，前台线程和后台线程不会操作同一个缓冲区，这也就意味着前台线程的操作，不需要等待后台线程缓慢的写文件操作(因为不需要锁定临界区)。通过双缓冲技术，很好地解决了生产者和消费者之间的异步操作和速度不匹配问题，提高了日志系统的整体吞吐率。
  
  - **不用生产者-消费者模型**，并发编程中的经典模型。以多线程为例，为了实现线程间数据同步，生产者线程与消费者线程共享一个缓冲区，其中生产者线程往缓冲区中push消息，消费者线程从缓冲区中pop消息。
  
  - 阻塞队列将生产者消费者模型进行封装，使用循环数组实现队列，作为两者共享的缓冲区）
  
    
  
- **同步日志，**日志写入函数与工作线程串行执行，由于涉及到I/O操作，当单条日志比较大的时候，同步模式会阻塞整个处理流程，服务器所能处理的并发能力将有所下降，尤其是在峰值的时候，写日志可能成为系统的瓶颈。

- 写入方式通过初始化时是否设置队列大小（表示在队列中可以放几条数据）来判断，若队列大小为0，则为同步，否则为异步。若异步,则将日志信息加入阻塞队列,同步则加锁向文件中写





5.5 服务器运⾏中，实际存储的⽂件被其他⽤户修改了，会发⽣什么？有什么优化的想法？

内存⻚⾯的换⼊换出，每次从cache读⼊时，先判断实际⽂件去对⽐，可以对⽐最后修改时间或者通过摘要算法判断

**服务器运行所需的配置文件**导致**服务器的行为改变，**或者在某些情况下，服务器可能无法正常工作。如果服务器有缓存这些配置文件的机制，可能直到下次服务器重启，才会采用新的配置。
**服务器正在读取或写入的数据文件**，会导致**数据丢失或损坏**。例如，如果文件被修改为一个无效的格式，那么服务器可能无法正确读取文件中的数据。如果服务器正在写入数据到文件，那么可能会导致数据无法正确保存。
**优化：**比如**限制用户的权限，防止他们修改重要的文件**。另外，你也应该**定期备份重要的文件**，以防止数据丢失。如果出现问题，你应该立即检查服务器的日志，以找出问题的原因，然后采取适当的措施来修复问题。

**性能优化：**
代码优化：确保你的代码是高效和优化的。这可能包括减少不必要的计算，使用更快的算法和数据结构，或者使用并发和多线程技术。
使用负载均衡：如果你的服务器负载很大，可能需要使用负载均衡技术将请求分发到多个服务器。这可以帮助提高处理能力，并且如果一个服务器出现问题，其他服务器可以接管。
数据库优化：如果你的服务器使用了数据库，那么你需要确保数据库是优化的。这可能包括使用索引来加速查询，定期清理和优化数据库，以及使用缓存来减少数据库访问。
**安全优化：**
限制权限：只给必要的用户和进程提供必要的权限，可以减少安全风险。
定期更新和打补丁：保持你的服务器和所有的软件更新到最新版本，可以帮助防止新的安全威胁。
备份和恢复策略：定期备份重要的数据，并确保你有一个有效的灾难恢复策略。
**日志和监控：**维护详细的日志，并使用监控工具来检查你的服务器的健康和性能。如果出现问题，这将帮助你快速定位和解决问题。



5.5 **为什么设计成双缓冲⽽不⽤更多的缓冲区？**

前端缓冲不⾜时会⾃动扩展，但是双缓冲⾜够应付使⽤场景，因为⽇志只记录必要的信息，并不会太多



5.6  为什么将生产者消费者模式改 => 双缓冲机制？

- 生产者消费者模式存在的问题：消费者从消息队列每读取一条日志信息就写入文件系统，但是写文件操作是很耗时的。频繁的从消息队列中获取数据，而且每次都要上锁，一定会对生产者的写日志效率产生影响，因为生产者也要对消息队列上锁才能把日志信息插入队列的头部，如果此时消息队列正好被消费者锁住了，那么生产者就必须等待了，这样就会很大影响到日志系统整体的吞吐率
- 双缓冲机制的基本思路是：准备两块 buffer: 1 和 2; 前端负责往 buffer 1 填数据(日志信息)；后端负责把 buffer 2 的数据写入文件。当 buffer 1 写满之后，交换 1 和 2，让后端将 buffer 1 的数据写入文件，而前端则往 buffer 2 填入新的日志信息，如此反复。
- **缓冲区交换：**直接交换两个缓冲区的地址， 把生产者在写入数据时的指向缓冲区1的指针重新指向缓冲区2， 把消费者读取数据时指向的缓冲区2的指针重新指向缓冲区1，这样就达到了交换缓冲区的目的了。
- 双缓冲区的好处是：在大部分的时间中，前台线程和后台线程不会操作同一个缓冲区，这也就意味着前台线程的操作，不需要等待后台线程缓慢的写文件操作(因为不需要锁定临界区)。通过双缓冲技术，很好地解决了生产者和消费者之间的异步操作和速度不匹配问题，提高了日志系统的整体吞吐率。



5.7 什么时候切换写到另⼀个⽇志⽂件？

⽇志串写⼊过多、唤醒⽇志线程

​      前⼀个 Buffer 已经写满了，则交换两个 Buffer（写满的 Buffer置空）。

1. ###### ⽇志串写⼊过多，⽇志线程来不及消费，怎么办？

    直接丢掉多余的⽇志 Buffer，腾出内存，防⽌引起程序故障。
2. ###### 什么时候唤醒⽇志线程从 Buffer中取数据？

    其⼀是超时，其⼆是前端写满了⼀个或者多个 Buffer
    
    

日志系统线程安全

线程安全问题，日志系统需要记录多个连接运行的情况，也就是说日志系统被多个线程拥有，这个时候需要考虑线程安全的问题，**通过内部关键操作（涉及临界区资源的部分）进行加锁**，实现每个线程对日志对象的访问不会产生冲突，避免日志写入的混乱。

在效率方面因为同步模式可能效率比较低，所以设计了一个异步写日志的方式

- 日志写入前会判断当前day是否为创建日志的时间，行数是否超过最大行限制
- - 若为创建日志时间，写入日志，否则按当前时间创建新log，更新创建时间和行数
  - 若**行数超过最大行限制**，在当前日志的末尾加count/max_lines为后缀创建新log









##### 异步日志原理

- 日志库分采用前后端分离。多个前端线程写日志到大缓冲区（要加锁），**一个后端线程**把前端线程的总日志写入文件，不用加锁，但是唤醒后端线程后交换两个缓冲区队列过程要加锁。
- **日志库采用双缓冲技术**，准备两个缓冲区，前后端各有一个4MB的Buffer，buffer缓冲区A 和 buffer缓冲区B（**一端实际各有两个，有个备用缓冲区，还有个缓冲区队列**），前端负责向 A 中写入日志消息，后端负责将 B 中的数据写入文件。
- 前端**当前Buffer**写满了之后，唤醒**后端线程**，后端线程加锁，让后端缓冲区队列与前端写满缓冲区队列**互换**（只交换其指针的指向），互换完成后**释放锁**，让**前端线程**继续向互换后的Buffer缓冲区中写入日志，而不必等待后端线程把日志全部写入文件后再释放锁，同时也避免每条新日志消息到来都唤醒**后端日志线程**。大大**减小了锁的粒度**，这也是muduo异步日志高效的关键原因之一。

###### 后端线程唤醒条件

**后端线程**的唤醒有两个条件：

- **buffer 写满**唤醒：前端**当前缓冲区Buffer**写满了，**唤醒后台线程**互换Buffer，并把Buffer中的日志写入文件，减少线程被唤醒的频率，降低系统开销。
- **超时被唤醒**：**为了及时把前端Buffer中的日志写入文件，日志库也会每3秒执行一次交换Buffer的操作**。防止系统故障导致内存中的日志消息丢失，超过规定的时间阈值，即使 **前端当前缓冲区**buffer 未满，也会立即将 buffer 中的数据写入。





缓冲区 A 和 B，分别对应前台日志缓冲队列 buffers 和后台日志缓冲队列 buffersToWrite。

- muduo前后端**各有两个大小为4MB的Buffer**，是FixedBuffer.h中定义的4MB大小的**字符数组**。



##### 双缓冲过程

- 在0~3秒之间，**前端线程**调用宏`LOG_INFO << 日志信息;`，会开辟一个4KB内存空间，放一条log日志信息，然后存储到前端Buffers_**4M写满缓冲区队列**（黄色Buffer)中，同时析构自己刚开辟的4KB的空间。考虑到多个前端线程会往4M大缓冲区队列放一条日志，当前前端线程需要加锁处理，保证线程安全<br />
- 在**3秒超时**或者**前端的当前小缓冲区写满**之后，后端线程被唤醒，为了让**前端线程**阻塞时间更短，后端线程也需要加锁，把后端待写缓冲区队列Buffer和前端写满缓冲区队列Buffer互换，互换后后端线程释放锁，这样前端线程又可以向红色Buffer中写入新的日志信息。后端线程也可以把来自前端的黄色Buffer中的数据写入文件。
- <a name="ferif"></a>

![](../%E6%A1%8C%E9%9D%A2/%E8%87%AA%E5%B7%B1%E6%95%B4%E7%90%86/%E5%9B%BE%E7%89%87/%E6%97%A5%E5%BF%97.png)<br />





##### 异步日志流程

异步日志前端产生日志和同步是一致的，异步LogStream::buffer_小缓冲区的数据是怎么输出到指定位置。异步日志不是默认的方式，需要自行开启，下从开启异步日志开始讲。
<a name="X6Yob"></a>

开启异步日志

`AsycnLogging`只提供了向**文件写入的功能**，由于`Logger`默认输出位置是`stdout`，所以需要像同步日志那样调用`Logger::setOutput(OutputFunc out)`接口，把**日志输出位置改为文件**。核心代码如下：

```cpp
AsyncLogging* g_asyncLog = nullptr;//异步日志对象
void asyncOutput(const char* msg, int len)
{
	g_asyncLog->append(msg, len);
}

int main()
{
    off_t kRollSize = 500*1000*1000;
    AsyncLogging log(“basename”, kRollSize);
    log.start();		// 启动异步日志的后台线程
    g_asyncLog = &log;
    Logger::setOutput(asyncOutput);//更改日志输出位置
    LOG_INFO << "Hello 0123456789" << " abcdefghijklmnopqrstuvwxyz";
    return 0;
}
```

<a name="YL97Z"></a>

###### 前端线程日志写入

- `AsyncLogging::append()`函数原理：前端生成一条日志消息。先将**日志信息存储到用户自己开辟的4M小缓冲区**，当**当前小缓冲区写满或3s超时**后，唤醒后台线程，加锁，将两个**缓冲区队列**互换后，再将**写满的缓冲区的数据**写入文件。
- 前端准备一个**前台缓冲区队列 `buffers_`和两个 buffer**。前台缓冲队列 `buffers_`用来存放多个小缓冲区的日志消息。两个 buffer，一个是当前缓冲区 `currentBuffer`，追加的日志消息存放于此；另一个作为备份缓冲区，即 `nextBuffer`，减少内存的开销。

**函数执行逻辑如下：**

判断当前缓冲区 `currentBuffer_`是否已经写满。

- 若4MB当前缓冲区未满，追加日志消息到当前缓冲，这是最常见的情况
- 若**当前缓冲区写满**，首先，把它移入前台缓冲队列 `buffers_`。其次，尝试把预备缓冲区 `nextBuffer_`移用为当前缓冲，若失败则**创建新的缓冲区作为当前缓冲**。最后，追加日志消息并唤醒后端日志线程开始写入日志数据。



- `currentBuffer_`有4MB大小，是为了**多积累一些日志信息**，然后**一并交给后端**写入文件，**避免频繁通知后台写文件**。`currentBuffer_->append()`是只把**一条日志信息写入`currentBuffer_`**中，而不是像`LogFile`的`apend()`是将**数据写入文件**中。

###### 前端线程代码

```cpp
// 前端开启异步日志，调用append，把日志信息写入currentBuffer_当前4M缓冲区中
void AsyncLogging::append(const char* logline, int len)
{
    // apend会被多个线程中调用，多个线程会同时向currentBuffer_中写数据，因此要加锁
    std::lock_guard<std::mutex> lock(mutex_);
    // 当前缓冲区剩余空间足够则直接写入
    if (currentBuffer_->avail() > len)
    {
        currentBuffer_->append(logline, len);
    }
    else
    {
        // 当前缓冲区空间不够写，将新信息写入备用缓冲区
        // 将写满的当前缓冲区currentBuffer_装入写满缓冲区队列vector中，即buffers_中
        // 注意currentBuffer_是独占指针，move后自己就指向了nullptr空内存
        buffers_.push_back(std::move(currentBuffer_));
        // nextBuffer_备用缓冲区不空，说明nextBuffer_指向的内存还没有被currentBuffer_抢走，也就是
        // nextBuffer_还没开始使用
        if (nextBuffer_) 
        {
            // 把nextBuffer_指向的内存区域交给currentBuffer_，nextBuffer_也被置空
            currentBuffer_ = std::move(nextBuffer_);
        } 
        else 
        {
            // 备用缓冲区也不够时，重新分配缓冲区，这种情况很少见
            currentBuffer_.reset(new Buffer);
        }
        //将新日志信息加到当前缓冲区
        currentBuffer_->append(logline, len);
        // 唤醒一个后端线程去将日志写入磁盘
        cond_.notify_one();
    }
}
```

<a name="mMD1b"></a>

###### 后端线程日志落盘

- `AsyncLogging::threadFunc()`：后端日志落盘线程的执行函数。
- 后端同样也准备了一个后台缓冲区队列 `buffersToWrite` 和两个备用 buffer。后台缓冲区队列 `buffersToWrite` 存放待写入磁盘的数据。两个备用 buffer，`newBuffer1`和`newBuffer2`，分别用来替换前台的当前缓冲和预备缓冲，而这两个备用 buffer 最后会被`buffersToWrite`内的两个 buffer 重新填充，减少了内存的开销。

函数执行逻辑如下，注意思考如何锁的粒度如何减小，起到了什么作用。

- 唤醒日志落盘线程（超时或写满buffer），交换前台缓冲队列和后台缓冲队列。加锁
- 日志落盘，将后台缓冲队列的所有 buffer 写入文件。不加锁，这样做的好处是日志落盘不影响前台缓冲队列的插入，不会出现阻塞问题，极大提升了系统性能。

`buffersToWrite`**后端缓冲区队列**用于接管前端的所有日志信息，提前释放锁，以便**前端可以继续写日志**，`buffersToWrite`接管后，在慢慢的往文件中写，写入文件的核心代码是`output.append(buffer->data(), buffer->length());`
<a name="xOoWC"></a>



###### 后端线程代码

```cpp
void AsyncLogging::threadFunc()
{
    // output输出位置有写入磁盘的接口，异步日志后端的日志信息只会来自buffersToWrite待写缓冲区队列。
    // 不会来自其他线程，不必考虑线程安全，所以output的threadSafe设置为false。
      // logFile 类负责将数据写入磁盘
    LogFile output(basename_, rollSize_, false);    
    // 后端小缓冲区1、2，用于归还 前端的缓冲区（当前、备用），currentBuffer nextBuffer
    BufferPtr newBuffer1(new Buffer);
    BufferPtr newBuffer2(new Buffer);
    newBuffer1->bzero();
    newBuffer2->bzero();
    // 后端缓冲区队列数组置为16个，用于和前端缓冲区数组进行交换
    BufferVector buffersToWrite;
    buffersToWrite.reserve(16);
     // 异步日志开启，则循环执行
    while (running_)
    {
        {
 // <---------- 交换前台缓冲队列和后台缓冲队列 ---------->
            // 后端线程 交换缓冲区队列加锁，交换后释放锁
            // 互斥锁保护，前端线程在这段时间内无法向前端Buffer缓冲区队列数组写入数据
            // 1、多线程加锁，线程安全，注意锁的作用域
            std::unique_lock<std::mutex> lock(mutex_);
            // 2、判断前台缓冲队列 buffers 是否有数据可读
            if (buffers_.empty())
            // 前端缓冲区队列为空，超过3s，也会唤醒后端线程
            {
                 // 触发日志的落盘 (唤醒) 的两个条件：1.超时，等待三秒也会解除阻塞 or 2.被唤醒，即前台写满 buffer
                cond_.wait_for(lock, std::chrono::seconds(3));
            }
            // 只要触发日志落盘，不管当前的 buffer 是否写满都必须取出来，写入磁盘。此时正使用的前端当前缓冲区buffer也放入buffer数组中（没写完也放进去，避免等待太久才刷新一次）
            // currentbuffer 被锁住 -> currentBuffer 被置空  
 buffers_.push_back(std::move(currentBuffer_));
             // 4、将空闲的 newbuffer1 移为当前缓冲，复用已经分配的空间
            currentBuffer_ = std::move(newBuffer1);// currentbuffer 需要内存空间
            //  5、核心：后端缓冲区队列和前端缓冲区队列交换
            buffersToWrite.swap(buffers_);
            // 若预备缓冲为空，则将空闲的 newbuffer2 移为预备缓冲，复用已经分配的空间
            if (!nextBuffer_)
            {
                nextBuffer_ = std::move(newBuffer2);
            }
        }// 注意这里加锁的粒度，日志落盘的时候不需要加锁了，主要是双队列的功劳
        ...省略归还缓冲区的代码...
            
         // <-------- 日志落盘，将buffersToWrite中的所有buffer写入文件 -------->
     assert(!buffersToWrite.empty());
     // 6、异步日志消息堆积的处理。缓冲区队列25个缓冲区
     // 同步日志，阻塞io，不存在堆积问题；异步日志，直接删除多余的日志，并插入提示信息
     if (buffersToWrite.size() > 25) {
       printf("Dropped\n");
       // 插入提示信息
       char buf[256];
       snprintf(buf, sizeof buf, "Dropped log messages at %s, %zd larger buffers\n",
 Timestamp::now().toFormattedString().c_str(),
                buffersToWrite.size()-2);    
       fputs(buf, stderr);
       output.append(buf, static_cast<int>(strlen(buf)));
       // 只保留2个buffer(默认4M)
       buffersToWrite.erase(buffersToWrite.begin()+2, buffersToWrite.end());   
     }

        // 遍历后端缓冲区队列的所有 buffer，将其写入文件
        for (const auto& buffer : buffersToWrite)
        {
            // 内部封装 fwrite，将 buffer中的一行日志数据，写入用户缓冲区，等待写入文件
            output.append(buffer->data(), buffer->length());
        }
    // 8、刷新数据到磁盘文件？这里应该保证数据落到磁盘，但事实上并没有，需要改进 fsync
     // 内部调用flush，只能将数据刷新到内核缓冲区，不能保证数据落到磁盘（断电问题）
        output.flush(); //刷新缓冲区的文件到磁盘
    }
    
    
    // 9、重新填充 newBuffer1 和 newBuffer2
     // 改变后台缓冲队列的大小，始终只保存两个 buffer，多余的 buffer 被释放
     // 为什么不直接保存到当前和预备缓冲？这是因为加锁的粒度，二者需要加锁操作
     if (buffersToWrite.size() > 2) {
        // 只保留2个buffer，分别用于填充备用缓冲 newBuffer1 和 newBuffer2
       buffersToWrite.resize(2);  
     }
     // 用 buffersToWrite 内的 buffer 重新填充 newBuffer1
     if (!newBuffer1) {
       assert(!buffersToWrite.empty());
       newBuffer1 = std::move(buffersToWrite.back()); // 复用 buffer
       buffersToWrite.pop_back();
       newBuffer1->reset();    // 重置指针，置空
     }
     // 用 buffersToWrite 内的 buffer 重新填充 newBuffer2
     if (!newBuffer2) {
       assert(!buffersToWrite.empty());
       newBuffer2 = std::move(buffersToWrite.back()); // 复用 buffer
       buffersToWrite.pop_back();
       newBuffer2->reset();   // 重置指针，置空
     }
     // 清空 buffersToWrite
     buffersToWrite.clear();  
   }
    output.flush();
}
```







##### **coredump 查找未落盘的日志**

异步日志实现的日志消息并不是生成后立刻就会写入文件，而是先存放在**前台当前缓冲区 `currentbuffer` 或者前台缓冲区队列 `buffers`**中。每过一段时间后才会将缓冲区中的日志消息写到日志文件中。这样就会产生问题：如果程序在中途 core dump 了，那么**在缓冲区中还未来得及写出的日志消息**该如何找回？

构造场景：**主线程开启日志线程**，写入100w条日志，当写到第50w条时人为往空指针写数据制造异常退出，引发 core dump。

```c
 void testCoredump() {
     AsyncLogging log("coredump", 200*1000*1000);
     log.start();   //开启日志线程
     g_asyncLog = &log;
     int msgcnt = 0;
     Logger::setOutput(asyncOutput); //设置日志输出函数
  
     // 写入100万条日志消息
     for(int i = 0; i < 1000000; ++i)   {
       LOG_INFO << "testCoredump" << ++msgcnt;
       if(i == 500000) {
         int *ptr = NULL;
         *ptr = 0x1234;  // 人为制造异常
       }
     } 
 }
```

###### gdb 升级

```bash
 # 编译 gdb 过程中需要 texinfo，安装 texinfo
 sudo apt-get install texinfo
 
 # 下载gdb11
 wget http://ftp.gnu.org/gnu/gdb/gdb-11.1.tar.gz
 tar -zxvf gdb-11.1.tar.gz
 # 编译
 cd gdb-11.1 
 ./configure 
 make 
 sudo make install
 
 # 将 gdb 放到 bin 目录
 mv /usr/bin/gdb /usr/bin/gdb_bak
 cp /opt/gdb-11.1/gdb /usr/bin/gdb
```

###### 生成 core

用 coredump 生成的 **core 文件**来找还未写出的日志消息。运行程序命令，生成了 **core 文件和一个 .log 日志文件**。

```bash
 # 查看 core 文件是否开启，默认不生成，0
 ulimit -c
 # 开启 core 文件生成，unlimited指的是core文件的最大大小，可以设置为其它数字
 ulimit -c unlimited
```

使用 `tail -f`（显示文件的最后几行（默认是10行）命令查看到日志文件中有 460132 条日志消息，其余日志消息未来得及写出。下面**通过 core 文件查找剩下的日志消息**。

###### gdb 调试 core 文件

使用 gdb 执行 coredump 文件。`gdb [execfile可执行文件] [corefile 文件]`

```bash
 gdb main_log_test core
```

![img](https://pic4.zhimg.com/80/v2-762df119d38e0972a0c70d7812d5e6cf_720w.webp)

通过 gdb 信息可以看到，`Program terminated with signal SIGSEGV, Segmentation fault`。`LWP` 是线程的标识，这里当前线程的 LWP 为4614，共有两个线程：LWP 4614 和 4615。崩溃是在主线程。





用 **thread info**查看线程信息，用 `thread id` **切换线程栈**。

使用 `thread 2`切换查看**后端日志线程**，可以看到线程2位于 pthread_cond_timewait

![img](https://pic3.zhimg.com/80/v2-7dec857ec1cdf904ce48af16fb010832_720w.webp)

使用 `bt (backtrace)` 查看**线程的堆栈信息**。`frame id` 切换栈空间。

![img](https://pic4.zhimg.com/80/v2-699f1ba916a9228864c445d0f84cc4b3_720w.webp)

- 此时相当于是 **append 函数的栈帧**未写出的日志消息，只可能存在于 `currentBuffer_`或`buffers_`中。
- 可以通过`currentBuffer.get()`获取该 unique_ptr 所指向的**LogStream 。可以用 print 打印**。由于打印信息数量受到 FixedBuffer 的 max-value-size 限制，所以需要先设置 max-value-size 为无限大

```bash
 set max-value-size unlimited
 print *currentBuffer_.get()
```

![img](https://pic2.zhimg.com/80/v2-f3385647320d50f82fd9097c682da9dd_720w.webp)

print *currentBuffer_.get()

可以看到显示了一部分日志信息，还是有很多被省略了，因为 gdb 的终端输出长度有限制，默认为200个字符，**可以修改这个限制。这样可以在屏幕上全部显示。**

```bash
 show print elements # 可以看到限制200个字符
 set print elements unlimited
 show print elements # 修改成 unlimited
```

将终端上的所有打印的信息输出到指定文件

```bash
 set logging file gdbinfo.txt # 指定日志输出文件
 set logging on   # 开启日志拷贝
 set logging off  # 关闭日志拷贝
```

接下来，`print *currentBuffer_.get()`，**一直按回车**，打印 buffer 所有的数据，**所有终端上的打印信息都会拷贝到 gdbInfo.txt 中。**



vim 打开 gdbInfo.txt，所有数据都被当作了一行，这是因为在拷贝时，将 '\n' 作为了两个普通的字符而不是换行符。在 vim 中将其查找替换即可，在命令模式下输入：`%s/\\n/\r/g`，回车

![img](https://pic4.zhimg.com/80/v2-29840817a77b2e83a9a1ca65454e5907_720w.webp)

可以查找到后台日志线程写入的第50w条日志。

###### **高性能日志原因**

**如何实现高性能的日志**

- **批量写入：**存够数据，一次写入
- **唤醒机制**：**写满通知唤醒** `notify` + **超时唤醒** `wait_timeout`
- **锁的粒度**：**刷新磁盘时，日志接口不会阻塞**。这是通过双队列实现的，前台队列实现日志接口，后台队列实现刷新磁盘。
- **内存分配**：移动语义，避免深拷贝；双缓冲，前台后台都设有。







##### **单例模式实现日志系统**

-  单例模式：构造和析构函数私有化
-  静态成员函数：返回类指针
-  静态成员函数：类外释放对象
-  静态成员：指向本类的指针
-  类外初始化静态成员
   Mylogger *Mylogger::_pInstance = getInstance(); // 饿汉模式



-  1、设置日志的格式: Layout
       %c %d日期 %p优先级 %m消息 %n换行
-  2、日志的目的地: Appender 参数: 名称(没用) + 流名字
-  3、日志的种类

```c++
/* MyLogger.h */
 #ifndef __MYLOGGER_H__
 #define __MYLOGGER_H__
 #include <log4cpp/Category.hh>
 using namespace log4cpp;
 ​
 // 单例模式：构造和析构函数私有化
 class Mylogger {
 public:
     // 静态成员函数：返回类指针
     static Mylogger *getInstance();
     // 静态成员函数：类外释放对象
     static void destroy();
     void warn(const char * msg);
     void error(const char * msg);
     void debug(const char * msg);
     void info(const char * msg);
 ​
 private:
      Mylogger();
     ~Mylogger();
 private:
     // 静态成员：指向本类的指针
     static Mylogger *_pInstance;
     // Category对象的引用
     Category &_myCat; 
 };
 #endif
 /* MyLogger.cc */
 #include "MyLogger.h"
 #include <ilog4cpp/PatternLayout.hh>
 #include <log4cpp/FileAppender.hh>
 #include <log4cpp/OstreamAppender.hh>
 #include <log4cpp/Priority.hh>
 #include <iostream>
 using std::cout;
 using std::endl;
 using namespace log4cpp;
 ​
 // 类外初始化静态成员
 Mylogger *Mylogger::_pInstance = getInstance(); // 饿汉模式
 ​
 Mylogger::Mylogger()
 : _myCat(Category::getRoot().getInstance("mycat"))
 {
     cout << "Mylogger()" << endl;
     // 1、设置日志的格式: Layout
     // %c %d日期 %p优先级 %m消息 %n换行
     PatternLayout *ppl1 = new PatternLayout();
     ppl1->setConversionPattern("%d %c [%p] %m%n");
     PatternLayout *ppl2 = new PatternLayout();
     ppl2->setConversionPattern("%d %c [%p] %m%n");
 ​
     // 2、日志的目的地: Appender 参数: 名称(没用) + 流名字
     OstreamAppender *pos = new OstreamAppender("OstreamAppender", &cout);
     pos->setLayout(ppl1);
     RollingFileAppender *pfa = new RollingFileAppender("RollingFileAppender", name, 3 * 1024, 3);
     pfa->setLayout(ppl2);
     
     // 3、日志的种类
     _myCat.addAppender(pos);
     _myCat.addAppender(pfl);
     _myCat.setPriority(Priority::DEBUG);
 }
 Mylogger::~Mylogger() {
     cout << "~Mylogger()" << endl;
     // 回收资源
     Category::shutdown();
 }
 ​
 Mylogger *Mylogger::getInstance() {
     if(nullptr == _pInstance) {
         _pInstance = new Mylogger();
     }
     return _pInstance;
 }
 ​
 void Mylogger::destroy() {
     if(_pInstance) {
         delete _pInstance;
         _pInstance = nullptr;
     }
 }
 ​
 void Mylogger::warn(const char * msg) {
     _myCat.warn(msg);
 }
 ​
 void Mylogger::error(const char * msg) {
     _myCat.error(msg);
 }
 ​
 void Mylogger::debug(const char * msg) {
     _myCat.debug(msg);
 }
 ​
 void Mylogger::info(const char * msg) {
     _myCat.info(msg);
 }
 /* MyLoggertest.cc */
 #include "MyLogger.h"
 #include <iostream>
 #include <string>
 using std::cout;
 using std::endl;
 using std::string;
 ​
 #define prefix(msg) (string(__FILE__) + string("  ") \
         + string(__FUNCTION__) + string("  ") \
         + string (std::to_string(__LINE__)) \
         + string("  ") + msg ).c_str()
 ​
 #define LogError(msg) Mylogger::getInstance()->error(prefix(msg))
 #define LogInfo(msg) Mylogger::getInstance()->info(prefix(msg))
 #define LogWarn(msg) Mylogger::getInstance()->warn(prefix(msg))
 #define LogDebug(msg) Mylogger::getInstance()->debug(prefix(msg))
 ​
 string func(const string &msg) {
     string s1 = string(__FILE__) + string("  ") 
         + string(__FUNCTION__) + string("  ")
         + string (std::to_string(__LINE__)) 
         + string("  ") + msg;
 ​
     return s1;
 }
 ​
 void test() {
     Mylogger *pml = Mylogger::getInstance();
 ​
     logInfo("The log is info message");  
     logError("The log is error message");
     logWarn("The log is warn message");
     logDebug("The log is debug message");
 }
 ​
 int main(int argc, char **argv)
 {
     test();
     return 0;
 }
```





##### 异步日志、同步日志区别

**异步日志：**把日志**先保存到缓存中**，当日志达到一定数量后，或者缓冲区存满后再统一写入终端或者文件。

- 优点：这种积累日志的方式**大大减少了日志IO操作的次数**，几乎不耽误网络IO。
- 缺点：不能把日志实时写入文件，当**意外断电**，很可能缓存中大量日志还没来得及写入文件，导致大量日志丢失。
  <a name="SxuW1"></a>

日志的作用：便于我们观察服务器当前状态，也有利于排查服务器上的错误，便于维护服务器<br />日志分为同步和异步日志：<br />**同步日志：**有日志信息产生的话立刻输出到终端或者文件

- 优点：可以马上观察到服务器的状态
- 缺点：如果日志过多，就会频繁写数据，写是一种IO操作，比较耗时，日志过多后，线程大部分时间都浪费在写日志上了，而耽误了网络IO的处理，也就限制了高并发。







##### 各个类的作用

| 类名         | 功能                                                         |
| ------------ | ------------------------------------------------------------ |
| FixedBuffer  | 封装了一个**固定长度的字符数组**当作buffer，具有向buffer存取数据的功能。（该类定义在LogStream.h中） |
| LogStream    | 此类为**多种数据类型重载了`<<`运算符**，重载的`<<`函数内部都是将要`<<`的数据存储到它**自己的成员变量`buffer_`中**，这个`buffer_`是`FixedBuffer`类型，大小是**4KB**。 |
| FileUtil     | 封装的**文件工具类**，是个封装过的文件指针。具有将`char*`指向的数据写入文件的功能，其类也有个**64KB的缓冲区**供写文件时候使用。 |
| LogFile      | 具有**利用`FileUtil`把日志写入文件**的功能，也可以根据时间和日志文件大小创建新日志文件的功能。 |
| AsyncLogging | 前端有两个**4M的大缓冲区**（`FixedBuffer`类型）用于存储日志信息，后端有个子线程，也有两个4M的缓冲区，用于和前端写了日志的缓冲区进行交换，交换后，**后端就借助`LogFile`可以把日志**信息往文件中写入。 |
| Logging      | 能够设置日志级别，能够设定`LogStream`获得的日志信息输出到什么位置（输出到stdout还是文件） |

<a name="duajv"></a>











##### 同步日志流程

<a name="mLSse"></a>

产生日志

下面以前端线程调用`LOG_INFO << "abcd" << 15;`为例，看看到底内部发生了什么。

```cpp
#define LOG_INFO if (logLevel() <= Logger::INFO) \
	Logger(__FILE__, __LINE__).stream()
```

首先从上面`LOG_INFO`的宏定义可见，`LOG_INFO`实际上是创建了一个**`Logger`临时对象**，并调用其成员函数`Stream()`，`Stream()`返回的是一个`LogStream`对象的引用，`LogStream`和`cout`很像， 该类重载了`<<`的多种形式，以便`LOG_INFO`能够`<<`多种类型数据。**`LogStream`有一个4KB的`buffer_`**，其所有重`<<`载的函数都只做一件事：把日志信息（此例中为`“abcd”`和`15`），添加（`apend`）到`LogStream`的`buffer_`中。由此也可见`LogStream`**类的**`buffer_**尽管有4KB大小，但是只存储一条日志信息。**
<a name="K4f8a"></a>

日志输出到指定位置

<a name="V8rgM"></a>

日志输出到`stdout标准流`

把在**`LogStream`类的`buffer_`**中存储的的**一条日志**写输出到指定位置（以`stdout`为例）<br />从上面`LOG_INFO`的宏定义可见，`LOG_INFO << "abcd" << 15;`执行结束后就会析构`Logger`对象，**`buffer_`中的数据也正是`Logger`析构的时候被输出到`stdout`**，`Logger`析构函数会调用`defaultOutput`函数（定义在`Logging.cc`中），改函数代码如下：

```cpp
static void defaultOutput(const char* data, int len)
{
    fwrite(data, len, sizeof(char), stdout);
}
```

其中，`data`就是`LogStream`的**`buffer_`中存储的数据**。
<a name="Tnt7A"></a>

日志输出到文件

如果同步日志想把**日志信息fwrite到文件**中，需要手动完成以下步骤：<br />

- FileUtil`中的`buffer_(64KB大小，是FileUtil文件指针fp_对应的缓冲区)，
- LogStream中的buffer_（4KB大小，只存储一条日志），我们在buffer_上加上类名。

1. 创建一个`LogFile`对象。`LogFile`类有个成员函数`apend(char *data, int len)`，只要给他日志信息，即给他`LogStream::buffer_`中的`char *`数据，他就能把日志信息写入文件。所以需要先创建有个**`LogFile`对象**，才能写文件。
2. 让**`Logger`类的输出位置指向文件**。由于`Logger`默认输出位置是`stdout`，所以需要**改变`Logger`的输出位置**。由于`Logger`类提供了`Logger::setOutput(OutputFunc out)`接口，因此可以通过如下方式将日志输出位置改为文件：

```cpp
#include "Logging.h"
#include "LogFile.h"

std::unique_ptr<LogFile> g_logFile;//日志文件对象
// msg就是LogStream::buffer_小缓冲区中存储的一条日志信息，把这条日志信息apend到FileUtil的buffer_大缓冲区中
void dummyOutput(const char* msg, int len)
{	
	if (g_logFile) {
    	g_logFile->append(msg, len);
    }
}
int main() {
	g_logFile.reset(new LogFile("test_log_st", 500*1000*1000, false));
    Logger::setOutput(dummyOutput);	// 改变Logger的输出位置
    LOG_INFO << "Hello 0123456789" << " abcdefghijklmnopqrstuvwxyz";
    return 0;
}
```

此后，`LOG_INFO << data`创建的临时`Logger`对象在析构时，`Logger`类的`g_output`（函数指针）就不再指向`defaultOutput`函数而是指向`dummyOutput`函数了，即，就不会调用`defaultOutput`输出到`stdout`上了，而是调用上述代码的`dummyOutput`函数，把日志`LogStream::buffer_`中**存储的一条日志通过`Logfile::apend()`函数写入到文件中**。`Logging`析构函数核心代码如下：

```cpp
Logger::~Logger()
{
    ...省略代码...
	// 得到LogStream的buffer_
    const LogStream::SmallBuffer& buf(stream().buffer());
    // 输出(默认向终端输出)
    g_output(buf.data(), buf.length());
    // FATAL情况终止程序
    if (impl_.level_ == FATAL)
    {
        g_flush();  // 把缓冲区的数据强制写入指定的位置（默认是写入stdout）
        abort();
    }
}

void Logger::setOutput(OutputFunc out)  
{
    g_output = out;
}
```

<a name="QTgDR"></a>



疑问解答

##### LogFile类的append什么时候需要加锁

- 同步的时候需要，因为多个线程都可能产生日志，即`LOG_INFO << something`，他们都同时要写入同一个文件，所以需要加锁，
- 异步的后端线程写入文件不需要加锁。
- 但产生日志的**前端线程**有很多，在写入**用户开辟的大缓冲区**时确实需要**加锁**，但是这只是写入缓冲区，并不是写入文件，**当大缓冲区被写满后**，**唤醒后端线程**就会接管这个写满的大缓冲区，然后把**写满的大缓冲区的日志通过`LogFile`写入到文件**中，也就是说，异步日志开启的情况下，写文件的永远之后异步日志的后台线程，根本**不存在竞争**，所以不需要**加锁**。

```cpp
void LogFile::append(const char* logline, int len)
{
    if (mutex_)		//同步日志需要加锁
    {
        std::unique_lock<std::mutex> lock(*mutex_);
        append_unlocked(logline, len);
    }
    else			// 异步日志不需要加锁
    {
    	append_unlocked(logline, len);
    }
}

void LogFile::flush()
{
    if (mutex_)		//同步日志需要加锁
    {
        std::unique_lock<std::mutex> lock(*mutex_);
        file_->flush();
    }
    else			// 异步日志不需要加锁
    {
    	file_->flush();
    }
}
```



<a name="TkPUb"></a>

`FileUtil`为什么要开辟一个64KB的`buffer_`

`FileUtil`的作用是把**给定的数据写入到指定的文件**中，回答这个问题首先需要了解IO缓冲区。
<a name="geVjF"></a>

IO工作流程

当我们向**文件指针`fp_`指向的文件**写入数据时，数据并不是立即被写入到文件，而是系统提供了一个缓冲区（这个缓冲区我们不可见，当然我们也可以通过`setbuffer`来为`fp_`指定缓冲区），当我们**调用`fwrite`时**，数据先被**写入缓冲区**，当**缓冲区满**后，才会把**数据写入文件**。另外，如果不希望缓冲区写满后再写入文件，可以调用**`fflush`刷新缓冲区**，即把缓冲区的数据立即写入到文件中。
<a name="VmQhG"></a>

`FileUtil`为什么要开辟一个64KB的`buffer_`

- 系统提供的**缓冲区小**，如果日志很多且密集，那**缓冲区容易写满**，也就会很频繁的向文件中写入数据，IO是比较耗时的，如果**频繁向文件写入数据**那时间都**花在写文件上**了，而没时间去处理业务了，
- 所以可以手动为**`FileUtil`类的文件指针`fp_`指定一个比较大的缓冲区**，这样当日志很密集时，也不会很频繁的写入文件。<br />注意`FileUtil`的`apend`函数尽管**通过循环不断向文件指针`fp_`指向的文件写数据**，但不代表数据一定被写入到文件了，因为我们在`FileUtil`构造函数中为`fp_`指定了一块很大的缓冲区（64KB），
- `apend`尽管通过循环调用`write`一直向`fp_`指向的文件写数据，直到把给的`data`写完为止时，**很可能缓冲区还是没写满**，因此，不会立马写入到文件，而是**暂时缓存在我们指定的缓冲区**中（看起来像是数据全部写入文件了），**当调用`flush`或者`write`时缓冲区满了，那就会立即写入文件**，不过这些都是`wirte`函数调用的`fwrite`帮我们处理的事，不需要我们管，我们只需要不断调用`fwrite`，感觉数据全部写入了文件就行，至于数据是暂时放在缓冲区还是写入文件，就由`fwrite`替我们操心去吧。

```cpp
void FileUtil::append(const char* data, size_t len)
{
    // 记录已经写入的数据大小
    size_t written = 0;
    while (written != len)                          	// 只要没写完就一直写
        {
            // 还需写入的数据大小
            size_t remain = len - written;
            size_t n = write(data + written, remain);   // 返回成功写如入了n字节
            if (n != remain)                            // 剩下的没有写完就看下是否出错，没出错就继续写
            {
                int err = ferror(fp_);
                if (err)
                {
                    fprintf(stderr, "FileUtil::append() failed %s\n", getErrnoMsg(err));
                }
            }
            // 更新写入的数据大小
            written += n;
        }
    // 记录目前为止写入的数据大小，超过限制会滚动日志(滚动日志在LogFile中实现)
    writtenBytes_ += written;
}
```

<a name="trk8s"></a>



###### 服务器崩溃，⽇志系统发⽣什么、解决

**发生情况：**

1. **日志丢失**： 如果日志系统配置为缓存日志信息并且定期写入磁盘，那么崩溃时还未写入磁盘的日志信息可能会丢失。
2. **日志不完整**： 服务器在写入日志时崩溃，可能会导致最后一部分日志信息不完整。
3. **文件损坏**： 如果崩溃发生在日志文件写入操作的过程中，可能会导致日志文件损坏。
4. **性能下降**： 如果服务器崩溃后自动重启，并且大量的日志写入操作在启动时进行，可能会暂时影响服务器的性能。

**解决：**

1. **日志分割**： 定期分割日志文件，减少单个日志文件的大小，这样即使某个日志文件损坏，也只影响一小部分数据。
2. **数据恢复策略**： 定期备份日志文件，以便在文件损坏时可以从备份中恢复。
3. **事务日志**： 对于数据库等系统，使用事务日志可以在发生崩溃时恢复到最后一次一致的状态。
4. **使用写前日志（Write-Ahead Logging, WAL）**： 在更新数据之前先写入日志，确保即使发生故障，也能够恢复到故障发生前的状态。
5. **系统自动重启后的日志处理**： 确保系统崩溃重启后，有清晰的日志处理流程，比如先处理残留的旧日志数据，再开始正常的日志记录。
6. **错误处理机制**： 为日志系统实现错误处理机制，比如在捕获到写入错误时，可以重试或将日志写入备用存储。





===========================================

#### 6.缓存机制LFU

1. 讲⼀下你项⽬中缓存池的作⽤？为什么要选择 LFU ⽽不是 LRU？LFU有什么缺点？（最近加⼊
    的数据因为起始的频率很低，容易被淘汰，⽽早期的热点数据会⼀直占据缓存）可以进⾏优化吗？（LFUaging，或者使⽤LRU+LFU的形式）
2. 缓存机制为什么选⽤LFU？（主要考虑热点⻚⾯）热点⻚⾯指的什么？怎么知道是热点⻚⾯？有
    考虑过其他的缓存算法吗？（LRU，ARC）为什么不选⽤？（balabala）LFU什么缺陷吗，可以怎么进⾏优
    化？（LFU-aging，window-LFU，ARC）
3. 为什么⽤LFU，不⽤LRU？（保证热点⻚⾯的响应）LFU的实现可以讲⼀下吗？（双链表+双
    哈希）
4. 讲⼀讲为什么要加⼊缓存机制，常⽤的缓存算法有哪些？LFU的缺点是什么？怎么优化？
    （LFU-aging）



##### LFU缓存淘汰实现

- **LFU主要两个函数实现：**

- ​         bool LFUCache::**get**(string& key, string& val) 和 void LFUCache::**set**(string& key, string& val) 。 

- ​        **get(2)** 就是先在 FreqHash 中找有没有 key 对应的节点，

- ​        如果没有，就是缓存未命中，由⽤户调⽤ **set(2)** 向缓存中添加节点，如果缓存已满的话就在频度最低的⼩链表中删除最后⼀个节点；如果有，就是缓存命中，此时需要更新当前节点的频度（先将 KeyNode 节点从当前频度的⼩链表中删除，然后加⼊下⼀频度的⼩链表），加⼊下⼀频度的⼩链表的头部，这部分与LRU类似。

- ​       ⽽对于 WebServer 来说，key就是对应的⽂件名，value就是对应的⽂件内容（通过 mmap() 映射），这样来建⽴⻚⾯缓存还是⽐较简单的。

- **LFU主要对象有四个：**

- ⼀个⼤链表 FreqList ，⼀个⼩链表 KeyList ，⼀个 key-FreqList 的哈希表 FreqHash 以及⼀个 key->KeyNode 的哈希表 KeyNodeHash ，

- **FreqList** ：链表的每个节点都是⼀个⼩链表附带⼀个值表示频度，频度⽤来记录每个key的访问次数

- **KeyList** ：同⼀频度下的key value节点 KeyNode ，只要找到 key 对应的 KeyNode ，就可以找到对应的value

- **FreqHash** ：key到⼤链表节点的映射，这个主要是为了更新频度，因为更新频度需要先将 KeyNode 节点从当前频度的⼩链表中删除，然后加⼊下⼀频度的⼩链表（通过遍历⼤链表即可找到）

- **KeyNodeHash** ：key到⼩链表节点的映射，这个Hash主要是为了根据key来找到对应的 KeyNode ，从⽽获得对应的value

  





##### 缓存池的作⽤

**缓存池的作用：**存储频繁访问的页数据，以减少从慢速存储（磁盘）读取数据的次数，从而提高性能。通过将热点数据存储在缓存中，可以显著减少I/O操作，节省处理器时间，提高响应速度。缓存池有多种不同的替换策略，其中最常见的是最近最少使用（LRU）和最少频繁使用（LFU）。LRU策略假定最近使用过的数据在将来可能会再次被使用，而LFU策略则假定最频繁使用的数据在将来可能会再次被使用。

###### **为什么⽤LFU⽽不LRU**

- 保证热点⻚⾯的响应
- LRU：最近最少使⽤，发⽣淘汰的时候，淘汰访问时间最旧的⻚⾯。
  LFU：最近不经常使⽤，发⽣淘汰的时候，淘汰频次低的⻚⾯。
  对于**时间相关度较低**（⻚⾯访问完全是随机，与时间⽆关） WebServer 来说，经常被访问的⻚⾯在下⼀次有更⼤的可能被访问，此时使⽤LFU更好，并且**LFU能够避免周期性或者偶发性的操作导致缓存命中率下降**的问题；
- ⽽对于**时间相关度较⾼**（某些⻚⾯在特定时间段访问量较⼤，⽽在整体来看频率较低）的 WebServer 来说，在特定的时间段内，最近访问的⻚⾯在下⼀次有更大的可能被访问，此时使⽤LRU更好。⽬前设计的 WebServer 时间相关度较低，所以选择LFU。

- 采⽤的是双重链表+hash表的经典做法。线程安全，**LFU本身是线程安全的**：这部分的线程安全暂时没有好的想法，简单的在 get(2) 和 set(2) 加了互斥锁
  MutexLockGuard lock(mutex_);



###### LFU有什么缺点、优化

- 最近加⼊的数据因为起始的频率很低，容易被淘汰，⽽早期的热点数据会⼀直占据缓存、LFUaging，或者使⽤LRU+LFU的形式

- **缺点：**由于LFU的淘汰策略是淘汰访问次数最小的数据块，但是新插⼊的数据块的访问次数为1，这样就会产⽣缓存污染，使得新数据块被淘汰。换句话说就是，最近加⼊的数据因为起始的频率很低，容易被淘汰，而早期的热点数据会⼀直占据缓存，从而降低缓存的效率。对热点数据的访问会导致 freq ⼀直递增，我⽬前使⽤ int 表示 freq_ ，实际上会有溢出的风险。
- **优化：**

- **1、LFU-Aging：**在LFU算法之上，引⼊访问次数平均值概念，当平均值⼤于最⼤平均值限制时，将所有节点的访
  问次数减去最⼤平均值限制的⼀半或⼀个固定值。相当于热点数据“⽼化”了，这样可以避免频度溢出，也能缓
  解缓存污染

- 2、混合使用LRU和LFU：另一种策略是结合LRU和LFU的优点，使用一个算法，如LRU-K或者LFU-LRU。例如，在LFU-LRU中，首先使用LFU找到访问频率最小的一组项目，然后在这组项目中使用LRU将最旧的项目替换出去。

- 3、使用过期策略：一种可能的解决方案是为每个项目设置一个过期时间。在过期时间之后，项目的访问计数将被重置。这样，长期不被访问的项目的访问计数将会降低，从而有更大的可能性被替换。



#####  缓存为什么⽤LFU、热点页面是什么

- 使用最少频繁使用（LFU）算法主要是考虑到热点页面。**热点页面是指在一定时间内访问频率较高的网页**，它们经常被用户访问，因此将它们保存在缓存中可以显著提高服务器的性能和响应速度。

- **为什么：**
- LFU算法是一种以访问频率为基础的缓存替换策略，它更倾向于保留那些被访问次数多的数据项。当有新的数据需要加入到缓存中，而缓存已满时，LFU算法会优先替换掉那些被访问次数最少的数据。因此，对于热点页面这类被频繁访问的数据，LFU算法能有效地将它们保留在缓存中，从而提高其读取速度。
- 通过实现LFU缓存淘汰策略，可以优化高并发下热点页面被频繁请求的问题。这是因为**热点页面通常会被频繁访问**，它们的访问频率会增加，从而使它们更有可能**被保留在缓存**中。这有助于**提高服务器的响应速度**，因为不需要再次**从数据库或磁盘加载热点数据**，而是可以直接**从缓存中获取**。它确保了**缓存中始终包含最常被访问的数据项**，从而减少了**不必要的数据加载**和提高了系统的响应速度。

- **如何知道是热点页面？**
- 这通常通过**统计页面的访问频率**来判断。在LFU算法中，会为**每个页面维护一个计数器**，每次访问页面时，就将该页面的计数器加一。计数器的值越大，说明页面的访问频率越高，也就越可能是热点页面。
- 补：值得注意的是，LFU算法虽然在处理热点页面方面效果良好，但也有其局限性，例如可能出现**缓存污染问题，即长期不再被访问的页面占据缓存**。因此，实际中可能需要通过一些优化策略，如LFU-aging，来改进LFU算法，使其更适应数据访问模式的变化。








#### 7、测试相关

⼀些问题

1. 【华为⼀⾯】在开发过程中有进⾏过测试吗？（做了单元测试、集成测试、功能测试、压⼒测试）压⼒测试是
   怎么做的？（分⽹络框架、⽇志系统、存储引擎三部分去讲）
2. 【联想⼀⾯】在做这个项⽬的过程中做了哪些测试，发现了什么问题？有针对遇到的问题去做优化吗？（主要
   讲了下⽇志系统的并发性问题）





##### 项⽬测试

- 云服务器ubuntu系统，**12核32G**。
- 1000客户端4个线程，短连接2WQPS，长连接5WQPS。长连接因为不需要频繁的创建新套接字去响应请求，然后发送数据读取数据，关闭套接字等操作，所以比短连接QPS高很多。
- 压力测试使用开源压测工具 WebBench，开启1000个进程，访问服务器60s，过程是客户端发出请求，然后服务器读取并解析，返回响应报文，客户端读取。

**对项⽬的测试:**

1. ⽇志系统
   写⼊⽇志类型、等级
   多个线程同时写⼊的响应
   短期大量⽇志进行写⼊
2. ⽹络框架
   正确接收 client 连接和分发
   正确感知 client 连接的读写事件
   正确对 client 连接进⾏读写
   ⾼并发、多请求（webbench）
3. 线程池
   能否将任务放入任务队列，任务是否能够正确执⾏
   多个线程同时往任务队列放⼊任务
4. 缓存机制
   能否成功命中缓存
   多个线程读取缓存





##### webbench

**webbench 的原理**:

- **原理：**父进程fork若干子进程，每个子进程在规定时间内对目标Web服务器循环发出访问请求，父子进程通过管道进行通信，子进程通过管道写端向父进程传递若干访问完毕后记录的总信息，父进程通过管道读端读取子进程发来的相关信息，子进程到时后结束，父进程将所有子进程退出后统计并显示最后的测试结果，然后退出。Webbench最多可以模拟3万个并发连接去测试网站的负载能力。
- Webbench -c 1000 -t 10。-c表示客户端fork出的子进程数量，代表同时有这么多客户端访问服务器；-t表示访问事件。
- 使用webbench对服务器进行压力测试，创建1000个客户端，并发访问服务器60s，正常情况下有接近5.2万个HTTP请求访问服务器。 
- 通过结果可以知道服务器**每分钟输出的页面数**和**每秒传输的字节数**(10w或百万级别)，以及**成功建立的连接数量**和**失败的连接数量**。


###### **使用原因**

由于之前使⽤的 webbench 存在 connect() 失败时 sockfd 泄漏的bug，以及读取响应报⽂时读完了依然 read导致阻塞的 bug （因为是BIO，读完了再读就会阻塞了）。我使⽤ C++11 标准重新实现了⼀遍 webbench 



###### **重构代码实现**

- 下⾯是 webbench 重构后的代码，
  可以看到，webbench 中定义了三个接⼝：
  ParseArg(2) ：解析命令⾏参数
  BuildRequest(1) ： 构造 URL 请求
  WebBench() ：执⾏实际的压测
  然后是**两个核⼼函数**
  ConnectServer(2) 、 Worker(3) ，其中 ConnectServer(2) 的作⽤是与 URL 所在的服务端建⽴ TCP 连接， 
      Worker(3) 则是在子进程中创建客户端与服务器发送信息。
      Worker(3) 则针对长连接和短连接进⾏不同的操作，短连接需要在每次收发完数据后即与服务器断开连接，然后再执⾏
  ConnectServer(2) ，⽽长连接则不⽤，最后在收到闹钟信号 is_expired 后，停⽌连接，并将当前⼦进程的结
  果发给⽗进程统计。



###### **存在问题**

- webbench的原理就是父进程同时fork很多个子进程来向服务器发起连接，然后大量客户端同时发起连接导致服务器不能正常工作（测试服务器能支持多少连接）。编写代码完成后，开一两个浏览器区访问服务器时，没有问题。

- 当进行压力测试时，结果出现了问题，报了一个错误信息：**fork failed**。出现这个错误的第一反应就是服务器并发处理大量请求时，如果**TCP全连接队列太小，默认情况下就容易溢出，后续的请求会丢失**，于是就会出现服务器请求数量上不去的情况。可以通过同时增大**somaxconn**和**backlog**来增大全连接队列，somaxconn是内核参数，通过 `/proc/sys/net/core/somaxconn` 来设置其值（默认128）；backlog是`listen()`函数的第二个参数.于是将somaxconn增大到256，backlog增大到256发现还是不行。




###### **抓包tcpdump**

- 了解到**Linux可用tcpdump来抓包**，再使用**Wireshark来分析这个数据包**，看到很多客户端到服务器的连接的**标志位只有SYN，没有ACK**，说明**三次握手没有成功**，连接没有完全建立，也就是半连接队列满了。然后通过执行**`netstat -nat | grep SYN_RECV`**来查看处于**半连接队列状态的连接数**。 默认值128，多次观察，如果一直是128不变，说明半连接队列满了。同时，**通过`netstate -tupln | gerp 1316`(服务器端口号)**，来查看服务器的端口状态，发现处于Listen状态，再次说明错误应该是发生在三次握手完成之前。调用`accept()`时返回的fd是已经三次握手的连接，当前三次握手没有完成，于是我去accept函数附近查看代码。**定位到是因为在ET模式下，读操作没有放在while循环读导致的。**

- **因为我的epoll使用的是ET模式，当负责监听的文件描述符listenfd触发读事件时，就证明有连接来了，于是调用accept进行接收，如果同时有很多新连接到了，因为没有使用while循环读，ET模式下listenfd只会触发一次，也就是只将一个新连接接受处理，大量的连接没有被处理导致全连接队列满了，然后又导致半连接队列满了，兵败如山倒，所有就有后续一系列的bug**。

-  在ET模式下，当监听的文件描述符listenfd触发读事件时，内核会通知应用程序有新的连接请求到来，此时应用程序需要使用while循环读取所有的连接请求，直到读取完所有的连接请求或者读取返回EAGAIN错误码为止。 如果没有使用while循环读取，会导致全连接队列满，再导致半连接队列满，因为在ET模式下，内核只会在文件描述符状态发生变化时通知应用程序，如果应用程序没有读取所有的连接请求，**那么这些连接请求就会一直留在全连接队列中，占用系统资源，最终导致半连接队列满。当半连接队列满时，新的连接请求将会被拒绝，客户端会收到ECONNREFUSED错误码**。




###### **解决**

**将accept使用while循环接收直到返回错误为EAGAIN或EWOULDBLOCK为止**。



##### 项目难点

###### **传输大文件出错**

 使用writev函数在一次函数调用中写多个非连续缓冲区。

```c++
#include <sys/uio.h>
ssize_t writev(int filedes, const struct iovec *iov, int iovcnt);
```

 iov为io向量机制结构体

```c++
struct iovec {
    void      *iov_base;      /* 指向数据地址 */
    size_t    iov_len;        /* 数据长度 */
};
```

- 需要传输文件时，由于返回的http响应消息（包括http响应行（版本、状态码、状态信息等）和响应头（包括发送的数据长度、数据类型、是否是长连接等））和需要发送的文件不在同一个连续的缓冲区，需要使用`writev()`函数来发送不连续的缓冲区（由于操作系统和文件描述符的设置不同，`writev()`发送数据的总大小是有一定限制的）因此需要使用while循环来发送数据。

- **当请求小文件时，可能调用一次`writev()`函数就可以将数据全部发送出去，正常运行，不会出错；当向服务器请求大文件时，需要多次调用`writev()`函数**，此时就出现了问题，不是文件显示不全就是无法显示。经过对查询资料后，发现问题出现在`writev()`的m_iv结构体成员（`iov_base`-数据的地址/指针、`iov_len`-需要发送数据的长度）中，了解到每次`writev()`函数发送数据后，不会自动偏移文件指针和需要传输数据的长度，还是按照原有的指针和原有长度来发送数据，导致错误。

- 在项目中，使用了iov数组，包含两个对象，第一个存储响应消息的缓冲区（包括响应行与响应头），第二个存储需要发送的文件。因此，做以下**修改：由于响应消息较小，可能第一次`writev()`传输后就发送过去了，因此判断一下发送数据的长度是否超过响应消息的长度，超过则将`iov[0].iov_len=0`，后续只需传输文件，同时每次传输后都更新下次传输的文件指针和长度。这样就解决了大文件传输。**




**2、由于涉及IO操作，当单条日志比较大时，同步模式会阻塞整个处理流程。**

 使用异步日志模式。异步模式中，建立写线程，使用生产者-消费者模型封装一个阻塞队列，将工作线程要写的日志信息push（生产者）到阻塞队列中，使用条件变量，当阻塞队列中有日志则时，写线程通过回调函数调用pop（消费者）将日志异步写入文件中。



**3、.Linux 性能瓶颈检测工具**

vmstat

vmstat提供了`processes, memory, paging, block I/O, traps`和CPU的活动状况.

下边是vmstat的输出样式

各输出列的含义：

Process

> – r: 等待runtime的进程数
>
> – b: 在不可打断的休眠状态下的进程数

Memory

> – swpd: 虚拟内存使用量（KB）
>
> – free: 闲置内存使用量（KB）
>
> – buff: 被当做buffer使用的内存量（KB）

Swap

> – si: swap到磁盘的内存量（KBps）
>
> – so: 从磁盘swap出去的内存量（KBps）

##### 定时器双链表，最小堆原理，优化

**定时器双链表：**

定时器建立在双向链表上的

| 位置         | 添加 | 删除 |
| ------------ | ---- | ---- |
| 刚好在头节点 | O(1) | O(1) |
| 刚好在尾节点 | O(n) | O(1) |
| 平均         | O(n) | O(1) |

**Notes**

添加在为节点时间复杂度为O(n)，因为项目的逻辑是先从头遍历新定时器在链表的位置，如果位置恰好在最后，则插入的时间复杂度O(N)

a.在双向链表的基础上优化：

添加在尾节点的时间复杂度可以优化：在添加新的定时器的时候，**除了检测新定时器是否在小于头节点定时器的时间外，再先检测新定时器是否在大于尾节点定时器的时间，都不符合再使用常规插入。**

b.不使用双向链表，使用最小堆结构可以进行优化。

**定时器最小堆原理：**

时间复杂度：

添加：O(logn)

删除：O(1)

**工作原理：**

将所有定时器中超时时间最小的一个定时器的超时值作为alarm函数的定时值。这样，一旦定时任务处理函数tick()被调用，超时时间最小的定时器必然到期，我们就可以在tick 函数中处理该定时器。然后，再次从剩余的定时器中找出超时时间最小的一个（堆），并将这段最小时间设置为下一次alarm函数的定时值。如此反复，就实现了较为精确的定时。

路由问题可以改进。

**优化：**

1. **时间轮（Time Wheel）优化**：时间轮是一种基于循环队列的数据结构，用于管理定时器任务。时间轮将时间分割成一系列的槽(slot)，每个槽表示一个时间单元。定时器任务被放置在对应的槽中，随着时间轮的旋转，任务的触发时间逐渐逼近。这种结构对于管理大量的定时器任务非常有效。
2. **红黑树（Red-Black Tree）优化**：红黑树是一种自平衡二叉搜索树，也可以用来管理定时器任务。在某些情况下，红黑树的查找和插入操作比堆操作更高效，特别是当需要在定时器任务中进行动态调整优先级等操作时。



#### 8、主从状态机与HTTP请求

##### 有限状态机原理

有限状态机包含一组定义良好的状态和在这些状态之间转移的规则。在HTTP的上下文中，这些状态可能对应于HTTP请求/响应处理的不同阶段，例如开始行解析、头部解析、体解析等。状态转移通常由输入（如接收到的字符）触发，并且可能会导致某些操作的执行（如保存头部字段的名称和值）。

1. **状态**： 有限状态机具有一系列预定义的状态，这些状态表示HTTP解析过程中的特定点。例如，“解析请求行”，“解析头部”，“等待数据”等。
2. **转移**： 每个状态都定义了在给定输入上转移到另一个状态的规则。例如，在解析请求行的状态下，接收到换行符（`\n`）可能会触发状态转移到“解析头部”。
3. **事件**： 事件是触发状态转移的动作。在HTTP中，这通常是接收到的数据，如请求或响应字节。
4. **动作**： 进入状态或离开状态时，状态机可以执行动作。这些动作可能包括解析数据、存储信息、触发回调等。
5. **初始状态**： 状态机有一个明确的开始状态，在HTTP解析器中，通常是“开始”或“解析请求行”。
6. **终止状态**： 这些状态表示解析器完成了处理，可以是成功完成响应或者遇到错误。

###### **HTTP状态机示例**

- 开始状态：这是FSM的初始状态。当接收到新的HTTP请求时，FSM从此状态开始。

- **解析请求行：**
- 从请求报文开始解析请求行。
- 确定HTTP方法（如GET、POST、PUT等）。
- 获取请求的URL。
- 识别HTTP版本。
- **解析头部：**
- 从请求行下一行开始，逐行解析HTTP头部字段，直到遇到空行为止。
- 根据字段名称（如“Host”、“User-Agent”等）将头部内容存储在哈希表中。
- **解析消息体：**
- 如果HTTP请求方法（如POST或PUT）需要包含消息体，FSM将从空行后开始解析消息体。
- 根据“Content-Length”或“Transfer-Encoding”头部确定消息体的长度或如何读取。
- 处理可能的错误：
- 如果在解析过程中发现格式错误或非法值，FSM将转移到错误状态，并可能返回相应的HTTP错误响应。
- **完成状态：**
- 当HTTP请求完整且正确解析后，FSM将转移到完成状态，此时可以开始处理请求并生成响应。
- 在整个解析过程中，有限状态机保持对当前解析位置和状态的跟踪，以确保正确地从请求报文中提取所需信息。使用FSM进行HTTP请求解析可以确保高效、准确且安全地处理来自客户端的请求。

###### 为什么用状态机

因为HTTP请求包括不同的部分，有请求头，请求体，数据，根据不同的部分去执行不同的逻辑代码，可以更加高效。从状态机每次读取一行，返回状态包括读取正常，没读完，读取错误，然后读取正常，主状态机就开始解析，一开始主状态机的状态是解析请求头，然后处理完再处理请求体，如果是post请求最后还会处理数据部分。

###### 什么用到主从状态机

因为HTTP请求包括不同的部分，有请求头，请求体，数据，根据不同的部分去执行不同的逻辑代码，可以更加高效。从状态机每次读取一行，返回状态包括读取正常，没读完，读取错误，然后读取正常，主状态机就开始解析，一开始主状态机的状态是解析请求头，然后处理完再处理请求体，如果是post请求最后还会处理数据部分。

###### 主从状态机解析流程

项目中使用**主从状态机**的模式进行解析，从状态机（`parse_line`）负责读取报文的一行，主状态机负责对该行数据进行解析，**主状态机内部调用从状态机，从状态机驱动主状态机**。每解析一部分都会将整个请求的`m_check_state`状态改变，状态机也就是根据这个状态来进行不同部分的解析跳转的：

![图片](https://img-blog.csdnimg.cn/img_convert/5278e2a441ac9e91a46ead33402eb0da.webp?x-oss-process=image/format,png)

**主状态机**

三种状态，标识解析位置。

- CHECK_STATE_REQUESTLINE，解析请求行
- CHECK_STATE_HEADER，解析请求头
- CHECK_STATE_CONTENT，解析消息体，**仅用于解析POST请求**

**从状态机**

三种状态，标识解析一行的读取状态。

- LINE_OK，完整读取一行
- LINE_BAD，报文语法有误
- LINE_OPEN，读取的行不完整

```cpp
void http_conn::process()
{
    HTTP_CODE read_ret = process_read();
    if(read_ret == NO_REQUEST) {
        modfd(m_epollfd, m_sockfd, EPOLLIN);
        return;
    }
    bool write_ret = process_write(read_ret);
    if(!write_ret) close_conn();
    modfd(m_epollfd, m_sockfd, EPOLLOUT);
}
```

HTTP请求报文：请求行（request line）、请求头部（header）、空行和请求数据

响应报文:状态行、消息报头、空行和响应正文。



######  状态机解析报文整体流程

 **从状态机负责读取报文的一行，主状态机负责对该行数据进行解析**。每解析一部分都会将整个请求的 `m_check_state` 状态改变，状态机也就是根据这个状态来进行不同部分的解析跳转的：

- 主状态机有三种状态：CHECK_STATE_REQUESTLINE，
- 解析请求行；CHECK_STATE_HEADER，
- 解析请求头；CHECK_STATE_CONTENT，
- 解析消息体，仅用于解析POST请求；

- `parse_request_line(text)`，解析请求行，通过请求行的解析我们可以判断该HTTP请求的类型（GET/POST），而请求行中最重要的部分就是URL部分，我们会将这部分保存下来用于后面的生成HTTP响应。
- `parse_headers(text)`：解析请求头部，就是GET和POST中空行以上，请求行以下的部分。
- `parse_content(text)`：解析请求数据，对于GET来说这部分是空的，因为这部分内容已经以明文的方式包含在了请求行中的URL部分了；只有POST的这部分是有数据的，项目中的这部分数据为**用户名和密码**，我们会根据这部分内容**做登录和校验**。



 **解析报文整体流程：**

 process_read通过while循环，将主从状态机进行封装，对报文的每一行就行循环处理

- while循环判断条件：

  ```
  while（（m_check_state == CHECK_STATE_CONTENT && line_status == LINE_OK） || (lien_status = parse_line()) == LINE_OK）
  ```

  - 从状态机转移到LINE_OK，针对GET请求
  - 主状态机转移到CHECK_STATE_CONTENT两者为或关系，为真则循环



######  **为什么GET、Post判断条件写这样呢？**

 针对GET请求，每一行都是\r\n作为结束的，对报文解析解析时，仅用从状态机的状态`(lien_status = parse_line()) == LINE_OK`即可。

 针对POST请求，消息体的末尾没有任何字符，所以不能使用从状态机的状态，转而使用主状态机的转台作为循环入口，后面的`&& line_status == LINE_OK`是因为当消息体解析完成，报文的完整解析完成了，但此时的主状态机状态还是`CHECK_STATE_CONTENT`，也就是符合循环入口，会再次循环，而这时我们不希望的。因此，增加该语句，在消息体解析完后，将`line_status`改为`LINE_OPEN`，用来跳出循环，完成报文解析。





##### 有限状态机搭正则表达式解析HTTP报文

使用有限状态机搭配正则表达式的方式来做HTTP报文的解析，通过有限状态机标志位的改变来执行报文不同部分的解析，请求首行、请求头、请求体。利用regex_match函数根据定义的正则规则去匹配，[]:表示匹配中括号中的字符，[^]表示匹配除了中括号中^后面字符以外的字符，*表示匹配前面的子表达式零次或多次，?表示匹配前面的子表达式零次或一次。

```c++
    //解析请求头：GET / HTTP/1.1
    regex patten("^([^ ]*) ([^ ]*) HTTP/([^ ]*)$");//利用正则来解析，就是定义一个规则(正则表达式)去匹配字符串
    smatch subMatch;
//bool regex_match():在整个字符串中匹配到符合整个表达式的整个字符串时返回true，也就是匹配的是整个字符串。
    if(regex_match(line, subMatch, patten)) {//line是我们要去匹配的字符串，我们匹配的结果数据会保存到subMatch中，patten是定义的正则表达式(按照怎样的规则匹配)
        method_ = subMatch[1];//请求首行中的请求方法
        path_ = subMatch[2];//请求首行中的请求路径
        version_ = subMatch[3];//请求首行中的协议版本
        state_ = HEADERS;//解析完状态首行后状态发生改变
        return true;
    }

//解析请求头:regex patten("^([^:]*): ?(.*)$");
```



##### 如何判断数据读完

**判读读完：**如果是LT模式，就是有数据就去读，所以一定会读完，如果是ET模式，是用while死循环去读数据，直到读完或者被关闭连接了，所以也可以保证数据读完了。



###### 收到不完整请求报文怎么办

目前没有考虑接受不全的问题，因为服务器只是完成基础的get和post请求，不会有过多的数据量发送过去，TCP缓冲区可以足够接收，一次性的把请求数据全部接受。但对于这种情况，如果未来想要实现的话，我觉得可以这样，就是我**先把这次读到自定义用户缓冲区的请求报文进行解析，如果发现报文不完整，可以把缓冲区的读位置索引还原回去**(read指针)，然后收到下一次数据再继续解析。

目前只是简单的解析了表单中的信息，暂时不支持文件上传。



###### **Recv接收函数调用**

- Recv是将接受到的数据copy到buf中，特别地：**返回值**<0时并且(errno == EINTR || errno == EWOULDBLOCK || errno == EAGAIN)的情况下认为连接是正常的，就是没有数据了。**返回值为0就是对方调用了close API来关闭连接**

- 函数原型： ssize_t recv(int sockfd, void *buf, size_t len, int flags);第一个参数是用于通信的套接字，第二个参数就是要拷贝到的缓冲区，第三个参数就是缓冲区的大小，第四个参数一般设为0。

- 比如：bytes_read = recv(m_sockfd, m_read_buf + m_read_idx, READ_BUFFER_SIZE - m_read_idx, 0);就是把接收到的数据拷贝到了m_read_buf这个缓冲区中，从m_read_buf+m_read_idx开始，m_read_idx代表已经读入客户端的数据的最后一个位置的下一个位置，后一个参数代表的是缓冲区的大小，就是从最末尾到前面那个开始都是缓冲区。返回接受的数据的大小。下一个位置就从m_read_idx += bytes_read开始，这里其实就是加的间隔。

  




##### HTTP方法

GET，POST，PUT，HEAD，DELETE，OPTIONS



###### GET和POST对比

- get重点在**从服务器上获取资源**；post重点在**向服务器发送数据**；
- get传输数据是**通过URL请求**；post传输数据**通过Http的post机制**，**将字段与对应值封存在请求实体中**发送给服务器，这个过程对用户是不可见的；
- get是**不安全的**，因为URL是可见的，可能会泄露私密信息，如密码等； **post较get安全性较高**；
- get方式**只能支持ASCII字符**，向服务器传的中文字符可能会乱码。 post**支持标准字符集，可以正确传递中文字符。**
- **GET和POST请求报文的区别之一是有无消息体部分，GET请求没有消息体，当解析完空行之后，便完成了报文的解析。**
- - GET请求在URL中传送的参数是有长度限制。（大多数）浏览器通常都会限制url长度在2K个字节，而（大多数）服务器最多处理64K大小的url。
  - GET产生一个TCP数据包；POST产生两个TCP数据包。对于GET方式的请求，浏览器会把http header和data一并发送出去，服务器响应200（返回数据）；而对于POST，浏览器先发送header，服务器响应100（指示信息—表示请求已接收，继续处理）continue，浏览器再发送data，服务器响应200 ok（返回数据）。





######  HTTP的状态码

- 常见的状态码有 

- 200-请求成功；301-资源被永久转移到其他URL；

- 400-请求报文语法有误，服务器无法识别；403-请求的对应资源禁止被访问；
- 404-请求的资源不存在；500-内部服务器错误；503-服务器正忙

- 响应状态码分为5类；1开头的状态码**表示信息，服务器接到该请求，需要客户端的进一步操作**；2开头**代表成功，操作被成功接受并且处理**；3开头**代表重定向，需要进一步的操作**；4开头**代表客户端错误，包括无法完成请求和语法错误**；5开头代表**服务器错误，指明服务器在处理请求的过程中出错。**

- 本项目实现了200，404，403，500。








##### http请求怎么做、如何保证http请求完整解析

该项目使用线程池（主从Reactor模式）并发处理用户请求，主线程负责读写，工作线程（线程池中的线程）负责处理逻辑（HTTP请求报文的解析等等）

在HTTP报文中，每一行的数据由\r\n作为结束字符，空行则是仅仅是字符\r\n。因此，可以通过查找\r\n将报文拆解成单独的行进行解析，项目中便是利用了这一点。

**主从状态机可以保证完整解析。**

- (存储读取的请求报文数据)判断当前字节是否为\r
- - 接下来的字符是\n，将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK
  - 接下来达到了buffer末尾，表示buffer还需要继续接收，返回LINE_OPEN
  - 否则，表示语法错误，返回LINE_BAD
- 当前字节不是\r，判断是否是\n（**一般是上次读取到\r就到了buffer末尾，没有接收完整，再次接收时会出现这种情况**）
- 如果前一个字符是\r，则将\r\n修改成\0\0，将m_checked_idx指向下一行的开头，则返回LINE_OK
- 当前字节既不是\r，也不是\n，表示接收不完整，需要继续接收，返回LINE_OPEN



###### http连接请求处理

- 当浏览器端发出http连接请求，主线程**创建http类对象数组**用来接收请求并将所有数据读入各个对象对应buffer，然后将该对象插入任务队列；具体来说，通过内核事件表 如果是连接请求，那么就将他注册到内核事件表中（通过静态成员变量完成）。线程池中的工作线程从任务队列中取出一个任务进行处理（解析请求报文）。

- Web服务器端通过`socket监听`来自用户的请求。

- 远端的很多用户会尝试去`connect()`这个Web Server正在`listen`的这个`port`，而监听到的这些连接会排队等待被`accept()`.由于用户连接请求是随机到达的异步时间，所以监听`socket(lisenfs)` lisen到的新的客户连接并且加入监听队列，当`accept`这个连接时候，会分配一个逻辑单元来处理这个用户请求。



1. **创建监听Socket**： 服务器首先需要创建一个socket，并绑定到一个端口上监听客户端的连接请求。
2. **等待连接**： 服务器端的socket进入监听状态，等待客户端的连接请求。
3. **接受连接**： 当客户端尝试连接到服务器时，服务器端的socket会接受连接并为每个客户端创建一个新的socket连接。
4. **读取数据**： 在新建立的连接上，服务器通过读取操作来接收客户端发送的HTTP请求报文。
5. **解析请求**： 服务器解析HTTP请求报文的内容，包括请求行（请求方法、URI、HTTP版本）、请求头和可能的请求体。
6. **处理请求**： 服务器根据解析出的信息处理请求，并准备相应的HTTP响应报文。
7. **发送响应**： 服务器通过连接向客户端发送HTTP响应报文。
8. **关闭连接**： 一旦响应发送完毕，根据HTTP头部的`Connection`字段以及HTTP版本的规范，服务器可能会关闭连接或保持连接以便复用。



###### 如何响应收到HTTP请求的报文

![图片](https://img-blog.csdnimg.cn/img_convert/406d5e710fbf64d133466b3587865dd8.jpeg)

- http响应报文处理流程

- 当上述报文解析完成后，服务器子线程调用process_write完成响应报文，响应报文包括

- 1.状态行：http/1.1 状态码 状态消息；
- 2.消息报头，内部调用add_content_length和add_linger函数

- content-length记录响应报文长度，用于浏览器端判断服务器是否发送完数据

- connection记录连接状态，用于告诉浏览器端保持长连接

- 3.空行

- 1. 响应正文

- 随后注册epollout事件。服务器主线程检测写事件，并调用http_conn::write函数将响应报文发送给浏览器端。至此整个http请求和响应全部完成。




###### HTTP请求是否完整：

 类似于上面的方式，HTTP请求也有一个`Content-Length`头字段，用于表示请求体的长度。服务器可以根据这个字段来判断请求是否完整。当服务器接收到请求时，它会首先读取`Content-Length`字段，并根据该字段的值来判断请求体的大小。然后，服务器会根据已接收的数据大小和`Content-Length`的值来判断是否接收到了完整的请求。



###### HTTP响应是否完整：

 在HTTP协议中，每个响应都有一个`Content-Length`头字段，用于表示响应体的长度。浏览器可以根据这个字段来判断响应是否完整。当浏览器接收到响应时，它会首先读取`Content-Length`字段，并根据该字段的值来判断响应体的大小。然后，浏览器会根据已接收的数据大小和`Content-Length`的值来判断是否接收到了完整的响应。如果接收到的数据大小等于`Content-Length`的值，就说明响应完整，可以进行渲染。









##### Web浏览器与服务器通信过程

1. **解析域名**，找到主机 IP。

2. 浏览器利用 IP 直接与网站主机通信，**三次握手**，建立 TCP 连接。浏览器会以一个随机端口向服务端的 web 程序 80 端口发起 TCP 的连接。

3. 建立 TCP 连接后，浏览器向主机发起一个HTTP请求。

4. 服务器**响应请求**，返回响应数据。

5. 浏览器**解析响应内容，进行渲染**，呈现给用户。

   









#### 内存池

======================

基于哈希桶的内存池是一种常见的内存管理技术，用于优化高并发下短连接频繁建立连接的问题。以下是实现基于哈希桶的内存池的一般步骤：

1. 初始化内存池：在系统启动时，预先分配一定数量的内存块作为内存池，并将这些内存块按照哈希值映射到不同的哈希桶中。每个哈希桶包含多个内存块。
2. 连接建立时：当有新的连接需要建立时，从内存池中获取一个空闲的内存块来存储连接所需的资源。这可以通过遍历哈希桶来寻找空闲的内存块，或者使用一些高效的分配算法（例如，使用空闲链表）来实现。
3. 连接释放时：当连接关闭后，释放该连接占用的内存块，并将该内存块标记为空闲状态，以便后续连接可以重用。在哈希桶中查找要释放的内存块，并进行相应的标记处理。
4. 哈希桶冲突处理：由于哈希值的冲突可能导致多个内存块映射到同一个哈希桶中，因此需要处理冲突。可以使用链表等数据结构将哈希值冲突的内存块链接起来，确保在哈希桶中所有的内存块都能被正确管理。
5. 扩容与收缩：随着连接的不断建立和释放，内存池的大小可能需要动态调整。可以实现扩容机制，当内存池中的内存块不够用时，自动扩大内存池的大小。同样，如果内存池中的空闲内存块过多，也可以考虑收缩内存池的大小，以节省资源。

需要注意的是，实现基于哈希桶的内存池还需要考虑线程安全性，因为高并发场景下可能涉及到多个线程同时操作内存池。因此，在实现内存池时需要使用适当的同步机制，例如使用互斥锁或无锁数据结构，以保证内存池的线程安全性。

总结起来，基于哈希桶的内存池是一种用于优化高并发下短连接频繁建立连接问题的内存管理技术，通过预先分配一定数量的内存块并使用哈希桶进行管理，可以有效地提高内存分配和释放的性能，从而改善系统的并发处理能力。



==============

为什么要加⼊内存池？项⽬中所有的内存申请都走内存池吗？
【oppo⼀⾯】连接对象和缓存对象都⾛内存池，这⾥挖坑了

相⽐于 new 的性能提升在哪？（避免了系统调⽤的开销）， new 的系统调⽤开销有多少？（纳秒级），可
以忽略吗？（不能，在⾼并发场景下，越往后的连接延迟越明显）
new 的主要开销在哪？（系统调⽤和构造函数）
new ⼀定会陷⼊内核态吗？

（不⼀定，因为底层是 malloc ， malloc 根据分配内存的大小不同有两种分配方式，小于128k使用 brk() ，大于 128k 使⽤ mmap() ）
那你内存池对于小内存的申请相⽐ new 还有优势吗？（到这⾥才明⽩⾯试官挖的坑，赶紧sorrymaker，考
虑不周）

但是对于缓存的对象是有必要⾛内存池的，下去再好好理⼀理。（学到了）

=============



7.1 内存池什么设计的？缺点？优化？

内存池中哈希桶的思想借鉴了STL allocator，主要就是维护⼀个哈希桶 MemoryPools ，⾥⾯每项对应⼀个内存池 MemoryPool ，哈希桶
中每个内存池的块⼤⼩ BlockSize 是相同的（4096字节，当然也可以设置为不同的），但是每个内存池⾥每个块分割的⼤⼩（槽⼤⼩） SlotSize 是不同的，依次为8,16,32,...,512字节（需要的内存超过512字节就⽤ new/malloc ），这样设置的好处是可以保证内存碎⽚在可控范围内。

**内存池的基本设计：**

预先分配一大块内存，然后在需要的时候，从这个预分配的内存中分配小块内存给程序使用。当程序不再需要这些小块内存时，它们会被归还给内存池，以供将来再次使用，而不是直接释放。

1. **预分配内存**：内存池通常在启动时分配一块大内存，称为内存池。然后，根据需求，这块大内存被分成多个固定大小的小块，供应用程序使用。
2. **内存块列表**：内存池通常会维护一个空闲内存块列表，当应用程序请求内存时，内存池会从此列表中分配一个空闲块。当内存被释放时，它将被返回到空闲列表。
3. **内存重用**：当内存块被释放时，它们被放回到空闲列表，以便于重新分配。这样可以有效地重用内存，避免频繁的系统调用。

**线程安全**
内存池本身是线程安全的：在分配和删除 Block 和获取 Slot 时加锁 MutexLockGuard lock(mutex_);

**主要的对象有：**指向第⼀个可⽤内存块的指针 Slot currentBlock （图中的ptr to firstBlock ），被释放对象的slot链表 Slot freeSlot ，未使⽤的slot链表 Slot* currentSlot 
具体的作⽤：
**Slot currentBlock ：**内存池实际上是⼀个⼀个的 Block 以链表的形式连接起来，每⼀个 Block 是⼀块⼤的内存，当内存池的内存不⾜的时候，就会向操作系统申请新的 block 加⼊链表。
**Slot freeSlot ：**链表⾥⾯的每⼀项都是对象被释放后归还给内存池的空间，内存池刚创建时 freeSlot是空的。⽤户创建对象，将对象释放时，把内存归还给内存池，此时内存池不会将内存归还给系统（ delete/free ），⽽是把指向这个对象的内存的指针加到 freeSlot 链表的前⾯（前插），之后⽤户每次申请内存时， memoryPool 就先在这个 freeSlot 链表⾥⾯找。
**Slot curretSlot ：**⽤户在创建对象的时候，先检查 freeSlot 是否为空，不为空的时候直接取出⼀项作为分配出的空间。否则就在当前 Block 将 currentSlot 所指的内存分配出去，如果 Block 里面的内存已经使用完，就向操作系统申请⼀个新的 Block 。



**缺陷：**
内存池⼯作期间的内存只会增长，不释放给操作系统。直到内存池销毁的时候，才把所有的 block 释放掉。
这样在经过短时间大量请求后，会导致程序占⽤⼤量内存，然而这内存大概率是不会再⽤的（除非再有短时间的大量请求）。

1. **内存浪费**：内存池通常预先分配一大块内存，如果内存使用率不高，可能会造成内存浪费。同时，由于内存池通常将内存分配为固定大小的块，如果请求的内存大小小于这个块的大小，也会造成内存浪费。
2. **内存池大小**：决定预先分配多大的内存池是一个挑战，如果内存池过小，可能无法满足应用的需求；如果过大，可能会浪费内存。



**优化：**

MemoryPool 中使⽤了额外空间（⼀个链表）来维护 freed slots ，实际上这部分是可以优化的：**将struct Slot 改为 union Slot** ，⾥⾯存放 Slot next 和 T value ，这样可以保证每个 Slot 在没被分配/被释放时，在内存中就包含了指向下个可⽤ Slot 的指针，对于已经分配了的 Slot ，就存放⽤户申请的值。这样可以避免建⽴链表的开销。

针对前⾯所说的缺陷，还可以在服务器空闲（怎么判断呢？）时，将空闲内存释放（释放多少呢？释放哪部分呢？还可以结合⻚⾯置换算法？），归还给操作系统

1. **内存池大小调整**：根据程序的实际需求动态调整内存池的大小，可以有效地减少内存浪费和提高内存利用率。
2. **多级内存池**：为了应对不同大小的内存请求，可以设计多级内存池，每级内存池管理不同大小的内存块。这种设计可以提高内存利用率，减少内存浪费。
3. **内存对齐**：在分配内存时进行内存对齐，可以提高访问效率。
4. **并发控制**：在多线程环境中，需要设计合适的并发控制策略，以避免竞态条件，同时又不影响性能。



7.2  为什么要加入内存池？项⽬中所有的内存申请都⾛内存池吗？

**为什么：**

1. **性能提升**：当需要频繁地进行小块内存的分配和释放时，直接使用系统的内存分配new可能会引起大量的系统调用，导致性能下降。内存池通过预先分配一大块内存，然后从这个大块内存中分配小块内存，可以避免频繁地进行系统调用，从而提高性能。
2. **减少内存碎片**：系统的内存分配器可能会导致内存碎片化，特别是在需要频繁地分配和释放不同大小的内存时。内存池通过固定内存块的大小，可以减少内存碎片化。
3. **提高内存使用效率**：内存池通常会重用已经释放的内存，这样可以提高内存的使用效率，减少内存的浪费。

连接对象和缓存对象不一定走内存池。如果有**大量的小对象需要频繁地创建和销毁，那么使用内存池**是有利的。但如果内存申请的大小和频率变化较大，那么使用内存池可能会带来额外的复杂性和开销。



7.3 new 的性能提升（内存池）在哪？new 的系统调⽤开销有多少？可以忽略吗？new 的主要开销在哪？new ⼀定会陷⼊内核态吗？内存池对于小内存的申请相⽐ new 还有优势吗

（避免了系统调⽤的开销）（纳秒级）（不能，在⾼并发场景下，越往后的连接延迟越明显）（开销：系统调⽤和构造函数）
不⼀定，因为底层是 malloc ， malloc 根据分配内存的⼤⼩不同有两种分配⽅式，⼩于128k使⽤ brk() ，⼤于 128k 使⽤ mmap()系统调用进入内核态 ）
但是对于**缓存的对象**是有必要⾛内存池的。

 **new 的性能提升在哪？**

`new`操作符的性能提升可能源于其使用的上下文。在许多情况下，使用内存池或自定义内存管理器（它们可能重载`new`和`delete`操作符）可能会提高性能，尤其是在**处理大量小对象的创建和销毁时**。

**new 的系统调用开销有多少？可以忽略吗？**

`new`操作符在执行时可能需要进行系统调用以获取更多内存。这个系统调用本身是昂贵的，因为它涉及到用户态和内核态之间的上下文切换。**如果内存分配请求是罕见的或间隔较长的，这个开销可能可以被忽略**。然而，对于那些需要**频繁分配和释放内存的应用程序**，系统调用的开销就不能被忽略。

**new 的主要开销在哪？**

`new`的主要开销主要在于：

1. 系统调用：如上所述，如果需要向系统请求更多的内存，`new`操作符会引发一个系统调用，这是昂贵的。
2. 内存管理器的开销：分配和管理内存是一个复杂的过程，涉及到在内存管理器的数据结构中搜索可用内存块，以及可能的内存碎片化。
3. 对象构造：如果`new`用于非基本类型的对象，那么它还会涉及到对象的构造，这是一个额外的开销。

**new 一定会陷入内核态吗？**

不⼀定，因为底层是 malloc ， malloc 根据分配内存的⼤⼩不同有两种分配⽅式，⼩于128k使⽤ brk() ，⼤于 128k 使⽤ mmap()系统调用进入内核态 ）

`new`操作符本身并不一定会陷入内核态。它只有在需要向操作系统**请求更多内存时才会进行系统调用**，从而触发用户态和内核态之间的上下文切换。然而，如**果分配请求可以通过已经在用户态分配的内存**来满足，那么`new`就可以完全在用户态完成。

**内存池对于小内存的申请相比 new 还有优势吗？**

是的，内存池对于小内存的申请有优势。内存池通过预先分配一大块内存并将其划分为多个小块，从而**避免了频繁的系统调用**。此外，**内存池通常更能有效地处理内存碎片化问题**。当释放对象时，内存池可以立即重用这些内存块，而无需等待内存管理器的干预。这些因素通常使内存池在处理小内存分配时比`new`更有效。



7.4 内存映射是有名还是匿名？

​       **内存映射**是一种将文件的部分内容映射到进程的虚拟地址空间的技术。在Linux中，内存映射可以通过`mmap`系统调用实现。它可以用于实现匿名映射（anonymous mapping）和具名映射（named mapping）。

- 匿名映射是一种将一块匿名内存映射到进程的地址空间，它没有对应的文件，通常用于进程间通信或者在进程中进行临时数据存储。
- 具名映射是将一个文件的内容映射到进程的地址空间，通过提供文件路径来创建映射，它允许进程通过内存直接访问文件内容，类似于常规的文件I/O。







#### 跳表skiplist

1. 【中兴⼀⾯】讲⼀下跳表的原理？使⽤场景？如何保证查询效率？

    （通过**概率近似保证当前层的索引数**是上⼀层的⼀半）

2. 【远景三⾯】存储引擎的定时写⼊是怎么实现的？每次都将内存中的所有数据写⼊⽂件，没有改变的数据
    是否会重复写⼊？

    （会，可以通过在⽂件写⼊实际操作来避免）

    那实际操作在系统中会出现相互抵消的情况，如何优化？

    （开⼀个后台线程，定时合并实际操作，并将实际操作应⽤到⽂件中）

3. 【联想⼆⾯】介绍⼀下你web服务器的存储引擎，为什么要做这么⼀个存储引擎？

    （学习数据库的时候接触到了，⽐较感兴趣所以模仿做了⼀下）

    讲⼀下跳表的原理，查找是怎么做的（从顶层节点开始找，⽅式类似于⼆分查找）

    插⼊节点的过程是怎么样的

    （先将节点插⼊最底层，然后为了保证查找效率，上⼀层的节点数需要近似为下⼀层的1/2，因此通过随机数的⽅式模拟这个概率，如果是偶数则在当前层记录节点，否则继续进⼊下⼀层）

    删除节点

    （从最底层往上层去删除即可）

    B+树了解吗，说⼀下B+树的底层实现，说⼀下
    B+树和跳表的区别？

8.1 讲⼀下跳表的原理，插入？特点？使⽤场景？如何保证查询效率？

**跳表原理：**

跳表，即跳跃表（**Skip List**），是基于并联的链表数据结构，快速查找操作效率可以达到**O(logN)**，对并发友好。通过在标准的有序链表基础上增加了多级索引的方式，使得我们可以快速跳过那些不可能存在我们目标元素的位置，从而实现了快速查找。所以跳表以空间换时间，缩短了操作跳表所需要花费的时间

在普通的有序链表中，如果我们想查找一个元素，可能需要遍历链表中的大部分元素。但在跳表中，元素被存储在多个层次的链表中。每个层次的链表都包含了下一层链表的一部分元素，最底层包含了所有的元素。

**跳表也叫跳跃表，是一种动态的数据结构**。如果我们需要在有序链表中进行查找某个值，需要遍历整个链表，二分查找对链表不支持，二分查找的底层要求为数组，遍历整个链表的时间复杂度为O(n)。我们可以把链表改造成B树、红黑树、AVL树等数据结构来提升查询效率，但是B树、红黑树、AVL树这些数据结构实现起来非常复杂，里面的细节也比较多。跳表就是为了提升有序链表的查询速度产生的一种动态数据结构，跳表相对B树、红黑树、AVL树这些数据结构实现起来比较简单，但时间复杂度与B树、红黑树、AVL树这些数据结构不相上下，时间复杂度能够达到O(logn)。

**跳表的插入**

理解了跳表的查找后，在来看跳表的插入就相对来说比较简单，我们先找出最后一个小于等于插入值的节点，我们将值插入到该节点的右边。除了最底层链表上的插入外，每个新插入的节点都会有一个随机的层高，我们还需要维护层高之间的节点关系，因为新节点的左边第一个元素不一定有层高，所以我们要往左边查找到第一个`node.up`不为空的节点，维护节点之间的关系，然后将节点作为新的节点，继续往上加层。

**特点：**

跳表是多层（level）链表结构；
跳表中的每一层都是一个有序链表，并且按照元素升序（默认）排列；
跳表中的元素会在哪一层出现是随机决定的，但是只要元素出现在了第k层，那么k层以下的链表也会出现这个元素；
跳表的底层的链表包含所有元素；
跳表头节点和尾节点不存储元素，且头节点和尾节点的层数就是跳表的最大层数；
跳表中的节点包含两个指针，一个指针指向同层链表的后一节点，一个指针指向下层链表的同元素节点。

**使用场景：**

跳表通常用在需要进行快速查找的场景中，比如Redis就用到了跳表来实现有序集合数据类型。

1. **数据库索引**：数据库需要快速地找到满足特定条件的记录，而跳表提供了一种有效的方法来维护有序的数据，并且提供快速查找，区间查询和顺序访问记录的能力。因此，许多数据库系统会使用跳表作为索引的数据结构。例如，Redis中的有序集合（Sorted Set）就是用跳表来实现的。
2. **文件系统**：一些文件系统可能会用跳表来存储文件的元数据，使得寻找文件变得更快。
3. **网络路由**：网络设备，如路由器或交换机，需要快速地找到最合适的路径将数据包转发出去。跳表提供了一种高效的方式来进行这种查找。
4. **内存管理**：一些内存管理系统，例如垃圾回收器，可能会使用跳表来跟踪内存块的使用情况，以快速找到空闲的内存块。

**如何保证查询效率：**

跳表通过多级索引实现快速查找。在每个层次的链表中，元素都按照大小排序。每个元素在每一层的链表中都可能有一个或多个指针指向下一层的元素。所以当我们进行查找时，**可以从顶层开始**，根据元素的大小跳过不可能存在目标元素的位置。

**跳表的查询效率取决于索引的层数和每层索引的元素数量**。如果索引设置得当，跳表的查询效率可以达到对数级别，即O(log n)。

**为了保持跳表的查询效率，我们在插入和删除元素时需要适当调整索引**。这通常是通过随机化过程来完成的，每次插入元素时，都通过抛硬币的方式决定将这个元素插入到哪一层。这样可以保证每一层元素的数量都符合预期，从而保证查询效率。

**1.空间复杂度**
假设原始链表的长度是 N，第一级索引大约是 N/2，第二级索引大约是 N/4，以此类推，每一层减少一半，直至剩下两个点，其实就是一个等比数列，计算可以得到：所以跳表的空间复杂度是 O(n)，也就是说如果将n个节点的单链表构成跳表，需要额外将近n个节点的空间。如果我们使用3个节点或者5个节点： 求和得到的节点，大约是 ，尽管空间复杂度还是O(n)，但是存储空间减少了一半。

注意，当原始链表中存储的数据很多的时候，索引节点只需要存储关键值和几个指针，并不需要存储对象，所以当对象比索引节点大很多的时候，索引节点的存储空间就可以忽略不计了。

**2.时间复杂度**
按照每两个节点抽出一个节点作为上一级索引的节点，那么第一级索引节点大约是 n/2 个，第二级的索引大约是 n/4 个，以此类推，第 k 级索引的节点个数是第 k-1 级索引的节点个数的 1/2，那么第 k 级索引点的个数：。

在跳表查询某个数据的时候，如果每一层都要遍历m个节点，那在跳表查询一个数据的时间复杂度是：在每两个节点建立一个索引的极限情况下，每层最多遍历三个节点，因此m的最大值为3。所以在跳表中查找任意数据的时间复杂度是 O(logn)，查找的时间复杂度和二分查找是一样的。这种查找效率的提升，前提是建立了很多级索引。



8.2 介绍⼀下web服务器的存储引擎，为什么要做这么⼀个存储引擎？

**为什么：**

Web服务器的存储引擎主要负责数据的存储、检索、更新和删除。在许多情况下，数据处理的速度和效率直接取决于存储引擎的性能。以下是创建Web服务器存储引擎的一些原因：

1. **数据持久化：** 存储引擎能够保证数据的持久化，即数据在存储后即使在服务器断电或崩溃后仍然存在。这是通过将数据写入非易失性存储介质（如硬盘或SSD）实现的。
2. **数据访问和查询：** 存储引擎提供了对数据的快速访问和查询能力，这通常是通过各种索引结构（如跳表、B树、哈希索引等）实现的。
3. **事务处理：** 许多存储引擎支持事务处理，能够保证一系列操作（一个事务）要么全部成功执行，要么全部不执行。这是通过日志和锁等机制实现的。
4. **并发控制：** 存储引擎也负责处理并发操作，即同时有多个请求访问和修改数据时如何保证数据的一致性。
5. **数据恢复：** 在系统崩溃或数据被误删除等情况下，存储引擎可以通过日志或备份等方式恢复数据。
6. **数据压缩和分区：** 一些高级的存储引擎还提供数据压缩和分区等功能，以提高存储效率和查询性能。

**跳表CRUD操作**：

- **创建（Create）：** 在跳表中插入元素时，我们首先需要生成一个随机的层数，这个层数应小于或等于跳表的最大层数。然后我们从最高层开始，逐层向下找到插入位置（插入位置的下一个节点的值应大于插入值），然后将新节点插入到每一层中。
- **读取（Read）：** 从跳表中读取一个元素时，我们同样从最高层开始，如果当前节点的下一个节点的值小于目标值，我们就向右走；如果当前节点的下一个节点的值大于或等于目标值，我们就向下走。直到找到要查询的节点，或者确定该值不存在。
- **更新（Update）：** 在跳表中更新一个元素通常可以看作是先删除旧节点，然后再插入新节点。
- **删除（Delete）：** 从跳表中删除一个元素时，我们首先需要找到该元素，然后逐层删除节点，并更新各层节点的指针。最后，如果最高层没有节点，那么我们将跳表的层数减一。

**定时数据落盘**：

定期落盘是一种将内存中的数据定期保存到磁盘上的策略，主要目的是防止因服务器突然宕机导致的数据丢失，从而实现数据的持久化存储。这是一种重要的数据备份和恢复策略，被广泛应用在许多数据库和存储系统中。

以下是实现定期落盘策略的一般步骤：

1. **设置定时器或后台线程：** 首先，我们需要设置一个定时器或启动一个后台线程，每隔一定时间（如每分钟、每小时等）触发一次。
2. **数据序列化：** 当定时器触发或后台线程运行时，我们需要将当前内存中的数据转换为可以保存到磁盘的格式，这个过程通常称为数据序列化。这可以通过序列化跳表的方式实现，例如，我们可以将每一个键值对转换为一行文本，然后写入到文件中。
3. **写入磁盘：** 然后，我们将序列化后的数据写入到磁盘中。写入磁盘的过程应尽可能快速和高效，以避免影响到服务器的正常运行。
4. **数据恢复：** 如果服务器突然宕机，那么在服务器重启时，我们可以从磁盘中读取最近一次保存的数据，然后将数据反序列化为跳表，从而恢复数据。

定期落盘策略可以有效地防止数据丢失，但它也有一些缺点。需要注意的是，在写入硬盘时，可能会有并发写入的问题，**因此我们需要使用锁或者其他同步机制来保证数据的一致性**。例如，如果在两次落盘之间服务器宕机，那么这段时间内的数据可能会丢失。此外，写入磁盘的过程可能会占用大量的I/O资源，影响到服务器的性能，因此我们需要合理设置写入的频率，或者使用异步写入、批量写入等技术来提高性能。

**定时写入的工作流程：**

1. **写操作请求**：当有新的数据需要写入硬盘时，这个写操作首先被记录在内存中的缓冲区，这样可以减少频繁地对硬盘进行物理写入。
2. **标记"脏"数据**：内存中被修改过的数据页被标记为"脏"数据。这个标记用于追踪哪些数据需要被写入硬盘。
3. **后台进程**：存储引擎将启动一个后台进程或者线程，这个进程负责周期性地检查内存缓冲区中的"脏"数据，并将其写入硬盘。这个周期通常可以根据系统的负载和性能需求来调整。
4. **写入硬盘**：当到达定时写入的时间点，或者内存中"脏"数据达到一定的数量或者大小时，后台进程会将这些数据写入硬盘。

数据落盘：

InnoDB内存缓冲池中的数据page要完成持久化的话，是通过两个流程来完成的，一个是脏页落
盘；一个是预写redo log日志。

**保证数据的持久性**
当缓冲池中的页的版本比磁盘要新时，数据库需要将新版本的页从缓冲池刷新到磁盘。但是如果
每次一个页发送变化，就进行刷新，那么性能开发是非常大的，于是InnoDB采用了Write Ahead
Log（WAL）策略和Force Log at Commit机制实现事务级别下数据的持久性。

WAL要求数据的变更写入到磁盘前，首先必须将内存中的日志写入到磁盘；
Force-log-at-commit要求当一个事务提交时，所有产生的日志都必须刷新到磁盘上，如果日志刷
新成功后，缓冲池中的数据刷新到磁盘前数据库发生了宕机，那么重启时，数据库可以从日志中 恢复数据。
为了确保每次日志都写入到重做日志文件，在每次将重做日志缓冲写入重做日志后，必须调用一
次fsync操作，将缓冲文件从文件系统缓存中真正写入磁盘。 可以通过 innodb_flush_log_at_trx_commit
来控制重做日志刷新到磁盘的策略。
**脏页落盘**
在数据库中进行读取操作，将从磁盘中读到的页放在缓冲池中，下次再读相同的页时，首先判断
该页是否在缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁 盘上的页。

对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。
页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为CheckPoint的 机制刷新回磁盘。
**重做日志落盘**
InnoDB存储引擎会首先将重做日志信息先放入重做日志缓冲中，然后再按照一定频率将其刷新到
重做日志文件。重做日志缓冲一般不需要设置得很大，因为一般情况每一秒钟都会讲重做日志缓 冲刷新到日志文件中。可通过配置参数
innodb_log_buffer_size 控制，默认为8MB。



8.3 讲⼀下跳表的原理，查找，插⼊节点，删除节点？

（查找：从顶层节点开始找，⽅式类似于⼆分查找）、

（插⼊：先将节点插⼊最底层，然后为了保证查找效率，上⼀层的节点数需要近似为下⼀层的1/2，因此通过随机数的⽅式模拟这个概率，如果是偶数则在当前层记录节点，否则继续进⼊下⼀层），

（删除：从最底层往上层去删除即可）

跳表（Skip List）是一种基于链表的数据结构，用于在有序链表中高效地进行查找、插入和删除操作。它的设计灵感来自于平衡树，但相对于平衡树来说，实现起来更加简单，效率也较高。

**跳表的原理如下：**

1. 基本结构：跳表由多层链表组成，最底层是原始有序链表，每一层都是原始链表的一个子集，每个节点在每一层都有可能出现。最顶层是整个跳表的首节点，称为头节点（head）。
2. 节点结构：每个节点包含两个基本部分：一个值（value），表示节点的数据；多个指向下一层节点的指针，表示节点在不同层级的连接关系。通常，节点的第 0 层指针指向下一个节点（与普通链表类似），第 1 层指针跨越两个节点，第 2 层指针跨越四个节点，以此类推。

**查找操作的过程：**

**在跳表中搜索一个元素时，需要从顶层开始，逐层向下搜索。搜索时遵循如下规则。**

- **目标值大于当前节点的后一节点值时，继续在本层链表上向后搜索；**
- **目标值大于当前节点值，小于当前节点的后一节点值时，向下移动一层，从下层链表的同节点位置向后搜索；**
- **目标值等于当前节点值，搜索结束。**

1. 从头节点开始，沿着最高层的指针，逐层向右移动，直到找到目标值或者遇到比目标值大的值为止。在每一层，如果当前节点的下一个节点值大于等于目标值，就跳转到下一层；如果当前节点的下一个节点值小于目标值，则继续在当前层向右移动。
2. 当目标值出现在最底层时，跳表中的查找操作完成。

**插入节点的过程：**

**每一个元素添加到跳表中时，首先需要随机指定这个元素在跳表中的层数，如果随机指定的层数大于了跳表的层数，则在将元素添加到跳表中之前，还需要扩大跳表的层数，而扩大跳表的层数就是将头尾节点的层数扩大。下面给出需要扩大跳表层数的一次添加的过程。**

1. 执行查找操作，找到合适的位置插入节点。这与查找操作类似，沿着最高层逐层向右移动，找到应该插入的位置。
2. 插入节点后，需要随机决定该节点是否要出现在更高层，以维持跳表的平衡性。具体做法是：在插入节点后，以一定概率决定将该节点插入到更高层，直到抛出一个硬币决定不再插入为止。这样可以保证节点的分布在不同层次上是随机的。

**删除节点的过程：**

**当在跳表中需要删除某一个元素时，则需要将这个元素在所有层的节点都删除，具体的删除规则如下所示。**

- **首先按照跳表的搜索的方式，搜索待删除节点，如果能够搜索到，此时搜索到的待删除节点位于该节点层数的最高层；**
- **从待删除节点的最高层往下，将每一层的待删除节点都删除掉，删除方式就是让待删除节点的前一节点直接指向待删除节点的后一节点。**

1. 执行查找操作，找到要删除的目标节点。与查找操作类似，沿着最高层逐层向右移动，找到目标节点。
2. 从最高层开始，逐层向下删除目标节点。在每一层，将目标节点从链表中移除即可。注意，删除节点可能会破坏跳表的平衡性，因此在删除节点后，需要相应地调整指针，确保跳表仍然保持良好的性质。



**总结： 跳表通过添加多层索引，使得在有序链表中的查找、插入和删除操作可以在平均情况下达到 O(log n) 的时间复杂度，其中 n 是元素的数量。相比于平衡树，跳表的实现更加简单，且有较好的性能表现，特别适用于在单链表上执行高效的查找操作。**



8.4 存储引擎的定时写⼊是怎么实现的？每次都将内存中的所有数据写⼊⽂件，没有改变的数据是否会重复写⼊？那实际操作在系统中会出现相互抵消的情况，如何优化？

（会，可以通过在⽂件写⼊实际操作来避免）（开⼀个后台线程，定时合并实际操作，并将实际操作应⽤到⽂件中）

**存储引擎的定时写入**：也就是常见的写回策略，存储引擎的定时写入通常是通过缓冲区和一些后台进程实现的。数据首先写入内存缓冲区，然后定时或当缓冲区满时，后台进程会将数据写入硬盘。这种方法可以提高写入性能，减少硬盘I/O操作，因为内存的读写速度比硬盘快得多，而且可以将多次写操作集中在一次磁盘I/O中。

**定时写入的优势在于它可以将多个独立的写操作聚合成一个批量的硬盘I/O操作**，**这样可以提升硬盘的写入效率**。然而，这种策略也有其风险，如果在数据还未被写入硬盘之前，系统发生崩溃或者断电，那么内存中的这部分数据就会丢失。因此，很多存储引擎采用了日志系统或者事务机制来确保数据的持久性和一致性。例如，存储引擎可能会使用Write-Ahead Logging（先写日志）策略，即在修改数据之前，先将修改操作写入一个持久化的日志中，这样即使系统发生崩溃，也可以通过重播日志来恢复数据。



**重复写入：**在某些系统中，可能会选择定期将内存中的所有数据写入硬盘，**包括未修改的数据**。但这并不是最有效的方式。理想情况下，**只有被修改（"dirty"）的数据需要被写入硬盘，因为没有修改的数据在硬盘上已经有了副本**。这可以通过在每次数据修改时，将其标记为"dirty"，然后在写入硬盘时，只选择这些被标记为"dirty"的数据来实现。



**定时写入的工作流程：**

1. **写操作请求**：当有新的数据需要写入硬盘时，这个写操作首先被记录在内存中的缓冲区，这样可以减少频繁地对硬盘进行物理写入。
2. **标记"脏"数据**：内存中被修改过的数据页被标记为"脏"数据。这个标记用于追踪哪些数据需要被写入硬盘。
3. **后台进程**：存储引擎将启动一个后台进程或者线程，这个进程负责周期性地检查内存缓冲区中的"脏"数据，并将其写入硬盘。这个周期通常可以根据系统的负载和性能需求来调整。
4. **写入硬盘**：当到达定时写入的时间点，或者内存中"脏"数据达到一定的数量或者大小时，后台进程会将这些数据写入硬盘。





8.5 B+树了解吗，说⼀下B+树的底层实现，说⼀下，B+树和跳表的区别？



**B+ 树是一种平衡多路搜索树**，广泛应用于数据库和文件系统等数据存储领域。它的底层实现如下：

B+ 树的基本结构和 B 树类似，区别在于 B+ 树只有叶子节点存储数据，而非叶子节点只存储索引信息，不存储实际数据。每个节点都有一个关键字数组，按照从小到大的顺序存储，并且有指向子节点的指针。

**特点：**

1. 所有关键字都在叶子节点上，并且叶子节点之间通过指针连接形成有序链表，便于范围查询。
2. 非叶子节点只存储索引，不存储实际数据，使得每个节点能够容纳更多的索引，减少树的高度，提高查询效率。
3. 叶子节点之间通过指针连接，可以方便地进行范围查询和顺序遍历。

**与跳表的区别：**

1. 存储结构：B+ 树是一种树形结构，它在非叶子节点和叶子节点都存储数据，而跳表是基于链表的数据结构，只在最底层的链表节点存储数据。
2. 高度：B+ 树的高度通常较低，查询效率较高，是一种树状结构。而跳表的高度通常较高，但在平均情况下，查询效率也能达到 O(log n)，是一种链表结构。
3. 查找效率：B+ 树的查找效率较高，对于范围查询尤为优越，而跳表的查找效率也不错，但在范围查询时需要在底层链表上遍历。
4. 插入和删除：B+ 树的插入和删除操作相对复杂，因为需要维护树的平衡性；而跳表的插入和删除操作相对简单，只需要调整指针。

总体而言，**B+ 树适用于磁盘存储等需要频繁范围查询的场景，而跳表适用于内存存储等需要高效查找的场景**。B+ 树在数据库和文件系统中广泛应用，而跳表常用于实现高效的集合或索引结构。

 **B+ 树的底层实现**

我们需要考虑节点的数据结构、插入与删除操作、以及查找操作。

1. 节点的数据结构： B+ 树的节点分为两种类型：非叶子节点和叶子节点。它们的数据结构如下：

   - 非叶子节点（Internal Node）：
     - 包含一个关键字数组，按照从小到大的顺序存储关键字。
     - 包含一个指针数组，存储指向对应关键字范围的子节点的指针。
     - 指针的个数比关键字的个数多 1，因为一个节点有 n 个关键字时，会有 n+1 个子节点。
   - 叶子节点（Leaf Node）：
     - 包含一个关键字数组，按照从小到大的顺序存储关键字和对应的实际数据。
     - 包含一个指向下一个叶子节点的指针。

   在实际实现中，可以使用数组和指针或者链表来表示节点的结构。

2. 插入操作：

   - 从根节点开始查找要插入的位置，根据关键字的大小决定向左子树或右子树移动。
   - 如果插入位置是叶子节点，直接将关键字和数据插入到叶子节点的有序数组中，然后调整该节点和父节点的关键字，确保节点的关键字个数不超过规定的阶数。
   - 如果插入位置是非叶子节点，继续向下查找合适的叶子节点。
   - 如果叶子节点满了，需要进行分裂操作，将一半的关键字和数据移到新的节点中，并更新父节点的关键字和指针。

3. 删除操作：

   - 从根节点开始查找要删除的关键字所在的叶子节点。
   - 在叶子节点中删除关键字和数据，然后对该节点进行合并或借用关键字的操作，保持节点的关键字个数不低于规定的阶数。
   - 如果删除关键字后叶子节点为空，还需要更新父节点的关键字和指针。

4. 查找操作：

   - 从根节点开始查找，根据关键字的大小决定向左子树或右子树移动，直到找到目标关键字所在的叶子节点。
   - 在叶子节点中进行简单的线性查找，找到目标关键字对应的数据。

需要注意的是，为了保持 B+ 树的平衡性，插入和删除操作可能需要进行节点的分裂和合并操作。这样可以确保 B+ 树的高度保持较低，提高查找效率。同时，B+ 树的节点阶数（即关键字的最大个数）是根据实际情况设定的，一般会根据存储设备的特性和数据规模来选择合适的阶数。



8.6 为什么Redis选择使用跳表而不是红黑树来实现有序集合？

Redis 中的有序集合(zset) 支持的操作：

1. 插入一个元素
2. 删除一个元素
3. 查找一个元素
4. 有序输出所有元素
5. 按照范围区间查找元素（比如查找值在 [100, 356] 之间的数据）

其中，前四个操作红黑树也可以完成，且时间复杂度跟跳表是一样的。但是，按照区间来查找数据这个操作，红黑树的效率没有跳表高。按照区间查找数据时，跳表可以做到 O(logn) 的时间复杂度定位区间的起点，然后在原始链表中顺序往后遍历就可以了，非常高效。



8.7 工业上其他使用跳表的场景

HBase 属于 LSM Tree 结构的数据库，LSM Tree 结构的数据库有个特点，实时写入的数据先写入到内存，内存达到阈值往磁盘 flush 的时候，会生成类似于 StoreFile 的**有序文件**，而跳表恰好就是天然有序的，所以在 flush 的时候效率很高，而且跳表查找、插入、删除性能都很高，这应该是 HBase MemStore 内部存储数据使用跳表的原因之一。HBase 使用的是 java.util.concurrent 下的 ConcurrentSkipListMap()。

Google 开源的 key/value 存储引擎 LevelDB 以及 Facebook 基于 LevelDB 优化的 RocksDB 都是 LSM Tree 结构的数据库，他们内部的 MemTable 都是使用了跳表这种数据结构。



8.7 epoll实现一个单线程的FTP文件传输器

Epoll是一个用于处理I/O多路复用的机制，主要用于处理网络事件。单线程的FTP文件传输器可以使用Epoll来监听网络连接的事件，接收和发送文件数据。

- 使用`epoll_create`创建epoll对象。
- 使用`epoll_ctl`将FTP服务器的监听套接字添加到epoll中，监听连接事件。
- 进入循环，使用`epoll_wait`等待事件发生。
- 当有连接事件发生时，使用`accept`接受新的连接，并将新的连接套接字添加到epoll中，监听读事件。
- 当有读事件发生时，使用`recv`接收文件数据，并写入到文件中。
- 当传输完成后，关闭连接套接字，并从epoll中删除。
- 循环继续，等待下一个事件。







#### 定时器相关

11.1 为什么使用定时器？

 为了删除非活跃链接，防止连接资源浪费。`非活跃`，是指浏览器与服务器端建立连接后，长时间不交换数据，一直占用服务器端的文件描述符，导致连接资源的浪费。

 `定时事件`，是指固定一段时间之后触发某段代码，由该段代码处理一个事件，如从内核事件表删除事件，并关闭文件描述符，释放连接资源。



11.2 说一下定时器的工作原理？

 服务器主循环为每一个连接创建一个定时器，并对每个连接进行定时。利用小根堆来储存所有定时器，通过alarm设置定时器（如果一开始小根堆有定时器，则取小根堆的根节点的定时时间设置定时器，若没有则设置默认定时时间）。设置信号函数，仅关注SIGTERM和（触发）SIGALRM两个信号的触发。会预先创建管道，其中**写端写入信号值，读端通过IO复用系统检测读事件**，看是否有超时任务处理。

当检测到超时任务时，不立即处理，减少对主程序的影响（由主循环执行信号对应的逻辑代码）。处理超时任务时，判断当前小根堆的队顶定时器是否超时，如果超时，则执行回调函数，将该文件描述符从epoll_wait的内核事件表中删除，并关闭客户端的文件描述符，再把当前节点从小根堆删去，调整小根堆结构，再次获取定时时间最小的节点，重新调用alarm设置定时时间。

 **为什么写端要设置非阻塞？send是将信息发送给套接字缓冲区，如果缓冲区满了，则会阻塞，这时候会进一步增加信号处理函数的执行时间**，为此，将其修改为非阻塞。



11.3 定时任务处理函数的逻辑

 使用**统一事件源**，SIGALARM信号每次被触发，（通过epoll_wait）主循环调用一次**定时任务处理函数**，处理到时的定时器。

定时器处理非活动连接模块，主要分为两部分，其一为定时方法与信号通知流程，其二为定时器及其容器设计、定时任务的处理。

**定时器超时时间 = 浏览器和服务器连接时刻 + 固定时间(TIMESLOT)，可以看出，定时器使用绝对时间作为超时值，这里alarm设置为5秒，连接超时为15秒。**

```
//设置绝对超时时间
timer->expire = cur + 3 * TIMESLOT;
```



11.4 如何使用定时器

代码中在WebServer：：timer那里。

浏览器与服务器连接时，创建该连接对应的定时器，并将该定时器添加到链表上；

处理异常事件时，执行定时事件，服务器关闭连接，从链表上移除对应定时器；

处理定时信号时，将定时标志设置为true；

处理读事件时，若某连接上发生读事件，将对应定时器向后移动，否则，执行定时事件；

处理写事件时，若服务器通过某连接给浏览器发送数据，将对应定时器向后移动，否则，执行定时事件。



11.5 时间堆相关

堆**从内存视角来看，它是一个连续存放的数组**，而**从逻辑视角来看它是一颗顺序存储的完全二叉树**，堆又**分为大根堆和小根堆**：大根堆是指在完全二叉树当中，**所有子树的根节点值都要大于等于它的左右子树节点的值**。**小根堆**是指完全二叉树当中，所有子树的根节点值都要小于等于它的左右子树节点的值。删除和插入都是O(logn)的时间复杂度。

**具体实现：**采用小根堆的结构，将超时时间最小的定时器的超时值作为基准去找超时的定时器，超时时间最小的那个定时器必然超时，然后就可以去处理它，然后再从剩余的继续找最小的，并将这个最小时间再设置为间隔，如此反复就实现了比较精准的定时。其余的定时器都是以固定的频率去检查是否有定时器超时。小根堆插入新节点时，就up操作不断地和它父节点比较，直到满足条件；删除一个节点就是把它跟堆尾交换，然后把交换后的堆尾删掉，再把交换的那个节点down操作，down操作是每次比较和儿子节点中较小的值，不满足就交换，一直交换到不需要交换为止。



11.6 信号处理流程

Linux下的信号采用的**异步处理机制**，信号处理函数和当前进程是两条不同的执行路线。具体的，**当进程收到信号时，操作系统会中断进程当前的正常流程，转而进入信号处理函数执行操作，完成后再返回中断的地方继续执行。**

为避免信号竞态现象发生，信号处理期间系统不会再次触发它。所以，为确保该信号不被屏蔽太久，**信号处理函数需要尽可能快地执行完毕。**

一般的信号处理函数需要处理该信号对应的逻辑，当该逻辑比较复杂时，信号处理函数执行时间过长，会导致信号屏蔽太久。

这里的解决方案是，**信号处理函数仅仅发送信号通知程序主循环，将信号对应的处理逻辑放在程序主循环中，由主循环执行信号对应的逻辑代码。**



11.7 统一事件源

**统一事件源，是指将信号事件与其他事件一样被处理。**

具体的，**信号处理函数使用管道将信号传递给主循环，信号处理函数往管道的写端写入信号值，主循环则从管道的读端读出信号值，使用I/O复用系统调用(epoll_wait)来监听管道读端的可读事件，这样信号事件与其他文件描述符都可以通过epoll来监测，从而实现统一处理。**



11.8 信号处理机制

**信号的接收**

**接收信号的任务是由内核代理的**，当内核接收到信号后，会将其放到对应进程的信号队列中，同时向进程发送一个中断，使其陷入内核态。注意，此时信号还只是在队列中，对进程来说暂时是不知道有信号到来的。

**信号的检测**

**进程从内核态返回到用户态前进行信号检测，进程在内核态中，从睡眠状态被唤醒的时候进行信号检测**，当发现有新信号时，便会进入下一步，信号的处理。

**信号的处理**

- 信号处理函数是运行在用户态的，**调用处理函数前，内核会将当前内核栈的内容备份拷贝到用户栈上，并且修改指令寄存器（eip）将其指向信号处理函数。**
- 接下来进程返回到用户态中，执行相应的信号处理函数。
- 信号处理函数执行完成后，还需要返回内核态，**检查是否还有其它信号未处理。**
- 所有信号都处理完成，就会**将内核栈恢复（从用户栈的备份拷贝回来）**，同时**恢复指令寄存器将其指向中断前的运行位置，最后回到用户态继续执行进程。**



11.9 信号通知逻辑

创建管道，其中**管道写端写入信号值，管道读端通过I/O复用系统监测读事件**

设置信号处理函数SIGALRM（时间到了触发）和SIGTERM（kill会触发，Ctrl+C）

 **通过struct sigaction结构体和sigaction函数注册信号捕捉函数**

 在结构体的handler参数设置信号处理函数，具体的，**从管道写端写入信号的名字**

**利用I/O复用系统监听管道读端文件描述符的可读事件**

**信息值传递给主循环**，主循环再根据接收到的信号值**执行目标信号对应的逻辑代码**



11.10 为什么管道写端要非阻塞？

信号处理函数中send是将信息发送给套接字缓冲区，如果缓冲区满了，则会阻塞，这时候会进一步增加信号处理函数的执行时间，为此，将其修改为非阻塞。

**没有对非阻塞返回值处理，如果阻塞是不是意味着这一次定时事件失效了？**

是的，但定时事件是非必须立即处理的事件，可以允许这样的情况发生。



11.11管道传递的是什么类型？

信号本身是整型数值，管道中**传递的是ASCII码表中整型数值对应的字符。**

```
//将信号值从管道写端写入，传输字符类型，而非整型
send(u_pipefd[1], (char*)&msg, 1, 0);
```

**switch-case的变量冲突？**

switch的变量一般为字符或整型，当switch的变量为字符时，case中可以是字符，也可以是字符对应的ASCII码。



#### 设计模式相关

##### 单例模式概念、什么时候初始化、条件变量、手撕单例模式

**单例模式概念：**

 **单例模式**：单例对象的类**只允许一个实例存在，并提供一个全局访问点，该实例被所以模块共享**。主要解决一个**全局使用的类频繁创建和销毁的问题**。

 单例模式三要素：**只能有一个实例**；单例类**必须自己创建自己唯一的实例**。单例类必须**给所有其他对象提供这一个实例**。

 **优点**：

- 保证内存里**只有一个实例，减少了内存的开销**。
- **避免资源的多重占用。（比如写文件）**。
- 单例模式设置全局访问点，可以优化和共享资源的访问。

 **缺点**：

- 一般没有接口，**不能继承，扩展困难**。
- 在并发测试中**，不利于代码调试**。调试过程中，如果单例模式中的代码没有执行完，不能模拟生成一个新的对象。如果要扩展，只能修改原来的代码，**破坏了开放封闭原则**。
- 单例模式的功能代码通常写在一个类中，如果功能设计不合理，**很容易违背单一职责的原则**。



##### 单例在什么时候初始化

- C++中static对象的初始化
- non-local static 对象的初始化发生在main函数执行之前，也即main函数之前的单线程启动阶段，所以不存在线程安全问题。但C++没有规定多个non-local static 对象的初始化顺序，尤其是来自多个编译单元的non-local static对象，他们的初始化顺序是随机的
- 对于local static 对象，其**初始化发生在控制流第一次执行到该对象的初始化语句**时。多个线程的控制流可能同时到达其初始化语句。
  - 在C++11之前，在多线程环境下local  static对象的初始化并不是线程安全的:如果一个线程正在执行local static对象的初始化语句但还没有完成初始化，此时若其它线程也执行到该语句，那么这个线程会认为自己是第一次执行该语句并进入该local static对象的构造函数中。这会造成这个local static对象的重复构造，进而产生**内存泄露**问题
  - 而C++11则在语言规范中解决了这个问题。C++11规定，在一个线程开始local static 对象的初始化后到完成初始化前，其他线程执行到这个local static对象的初始化语句就会等待，直到该local static 对象初始化完成







##### 手撕单例模式

 通过将**构造函数私有化**，因此无法通过调用调用该类的构造函数来实例化类对象，只能**通过该类提供的公有的静态方法来得到该类唯一的实例**。通过**局部静态变量只初始化一次的特点，返回静态对象成员**。

**懒汉模式(内部静态变量线程安全版本)：**获取该类的对象 时 才创建该类的实例

```c++
// 使用函数内的局部静态对象无需加锁和解锁，因为C++11后编译器可以保证内部静态变量的线程安全性
class single{
private:
	single(){}
	~single(){}
public:
    single(const single&) = delete;
    single& operator=(const single&) = delete;
	static single* getinstance();
};
single* single::getinstance(){
	static single obj;
	return &obj;
}
```

**懒汉模式加锁版本**

```c++
class single{
private:
	static pthread_mutex_t lock;
	static single* obj;
	single(){}
	~single(){}
public:
    //使用双检测的原因：如果只检测一次，每一调用获取实例时都需要加锁，严重影响程序性能。
    //双层检测仅在第一次创建单例的时候加锁，其他时候都不再符合NULL == p的情况，直接返回已创建好的实例。
    static single* getinstance(){
        if(obj == nullptr){
            pthread_mutex_lock(&lock);
            if(obj == nullptr){
                obj = new single();
            }
            pthread_mutex_unlock(&lock);
        }
        return obj;
    }
};
//静态变量类内声明，类外初始化
single* single::obj = nullptr;
pthread_mutex_t single::lock;
```

**饿汉模式**:：获取该类的对象之 前 已经创建好该类的实例

 饿汉模式存在的问题：**non_local static变量（函数外的static对象）在不同编译单元中的初始化顺序是未定义的。如果在初始化完成前调用 getInstance() 方法会返回一个未定义的实例**。

```c++
class single{
private:
	static single *obj;
	single(){}
	~single(){}
public:
	static singel* getinstance(){};
}
single* single::obj = new single();
single* single::getinstance(){
	return obj;
}
//C++11局部静态变量线程安全懒汉模式
class Single{
private:
    Single(){}
    ~Single(){}
public:
    static Single& getinstance();
}
static Single::Single& getinstance(){
    static Single::Single& obj;
    return *obj;
};

//双检测锁的线程安全懒汉模式
class Single{
private:
    static pthread_mutex_t lock;
    static Single* obj;
    Single(){}
    ~Single(){}
public:
    static Single* getinstance(){
		if(obj == nullptr){
            pthread_mutex_lock(&lock);
            if(obj == nullptr){
				obj = new Single();
            }
            pthread_mutex_unlock(&lock);
        }
        return obj;
    }
};
pthread_mutex_t Single::lock;
Single* Single::obj = nullptr;

//使用call_once实现懒汉单例模式
class Single{
private:
    static std::unique_ptr<Single*> obj;
    Single(){}
    ~Single(){}
public:
    static Single* getinstance();
};
std::unique_ptr<Single>Single::obj = nullptr;
Signle* Single::getinstance(){
    static std::once_flag s_flag;
    std::call_once(s_flag, [&](){
        obj = new Single();
    });
    return obj;
}

//饿汉模式，线程安全
class Single{
private:
    static Single* obj;
    Single(){}
    ~Single(){}
public:
    static Single* getinstance();
}
Single* Single::obj = new Single();
Single* Single::getinstance(){
	return obj;
}
```



##### Makefile  && GDB调试

Makefile文件就是为了自动化编译，使用方便，要不然要一个一个文件编译

```c++
目标...: 依赖...
	命令....

$@ 表示目标的完整名称
$< 表示第一个依赖文件的名称
$^ 表示所有的依赖文件
CXX 表示C++编译器名称默认g++
$(变量) 代表获取变量值
var = hello, 定义变量-变量名=变量值
?= 代表用来给变量赋值，但只有在该变量之前未被赋值时（包括环境变量和Makefile中的赋值），这个赋值操作才会执行。
ifeq (arg1, arg2)
# 如果arg1和arg2相等，那么执行这里的内容
else
# 如果arg1和arg2不相等，那么执行这里的内容
endif
`clean`定义了一个名为clean的伪目标。当你运行make clean时，它将执行下面的命令。
CXX ?= g++

DEBUG ?= 1
ifeq ($(DEBUG), 1)
    CXXFLAGS += -g
else
    CXXFLAGS += -O2

endif

server: main.cpp  ./timer/lst_timer.cpp ./http/http_conn.cpp ./log/log.cpp ./CGImysql/sql_connection_pool.cpp  webserver.cpp config.cpp
	$(CXX) -o server  $^ $(CXXFLAGS) -lpthread -lmysqlclient

clean:
	rm  -r server
```

GDB调试

![img](https://gitee.com/linkeda666/C-add-add/raw/master/%E5%9B%BE%E7%89%87/GDB%E8%B0%83%E8%AF%95_03.png)

![img](https://gitee.com/linkeda666/C-add-add/raw/master/%E5%9B%BE%E7%89%87/GDB%E8%B0%83%E8%AF%95_04.png)

![img](https://gitee.com/linkeda666/C-add-add/raw/master/%E5%9B%BE%E7%89%87/GDB%E8%B0%83%E8%AF%95_05.png)



#### 数据库登录注册相关

13.1 什么是数据库连接池？为什么要创建数据库连接池？

 执行数据库语句的过程包括：**TCP三次握手**、**MySQL认证三次握手**（**握手初始化、登录认证、认证结果**）、**执行SQL**、**close MySQl**、**TCP四次握手**。

 缺点：频繁的建立连接与关闭连接，导致临时对象多；**频繁关闭连接导致大量的TIME_WAIT的TCP状态**；**响应时间较长**；网络IO较大。

 **建立连接池后**，只需要建立一次连接，后面的访问会复用创建的连接。减少了网络开销且没有了TIME_WAIT状态的TCP，同时响应时间快。

 池是资源的容器，这组资源在服务器启动之初就被完全创建好并初始化，本质上是**对资源的复用**。

 当系统开始处理客户请求的时候，如果它需要相关的资源，可以直接从池中获取，**无需动态分配**；当服务器处理完一个客户连接后,可以把相关的资源放回池中，**无需执行系统调用释放资源**。

 使用**单例模式和链表**创建数据库连接池，实现对数据库连接资源的复用。连接池的功能主要有：初始化连接池，获取一个连接，将连接放回连接池，销毁连接处。**数据库连接的获取与释放通过RAII机制封装，避免手动释放**。具体来说：**构造函数时初始化获取一个连接，析构函数释放连接**。



13.2 数据库连接池RAII说一下

 **采用单例模式+ 链表创建数据库连接池**，项目采用的是list容器，数据库这一部分用到了信号量保证原子操作

**数据库连接的获取与释放通过RAII机制封装，避免手动释放**

这里需要注意的是，在获取连接时，通过有参构造对传入的参数进行修改。其中数据库连接本身是指针类型，所以参数需要通过双指针才能对其进行修改。

```
class connectionRAII{
public:
	connectionRAII(MYSQL **con, connection_pool *connPool);
	~connectionRAII();
private:
	MYSQL *conRAII;
	connection_pool *poolRAII;
};
connectionRAII::connectionRAII(MYSQL **SQL, connection_pool *connPool)
{
	*SQL = connPool->GetConnection();
	conRAII = *SQL;
	poolRAII = connPool;
}
```

**在数据库连接池(list容器)中取出一个连接时，采用信号量机制同时由于多线程操作数据库连接池，所以还要用互斥锁**

**二级指针做参数，是为了给函数外定义的指针赋值，比如说我在外面定义了个指针：MYSQL\* mysql = NULL;再用二级指针做参数从而操作这个一级指针**

二级指针作为形参传递给函数时，实参的地址被拷贝到形参中。由于形参和实参指向同样的内存地址，函数中修改形参的值的时候会对实参产生影响。因此，二级指针作为形参可以修改实参。

```
connectionRAII mysqlcon(&mysql, connPool);
connectionRAII::connectionRAII(MYSQL* *SQL, connection_pool *connPool)
{
	*SQL = connPool->GetConnection();//这里是取出了数据库连接池list容器的front
	conRAII = *SQL;
	poolRAII = connPool;
}
```

**post请求报文中，content部分是用户名和密码，格式是这样：username=root&password=123456**



13.3 登录、注册说一下？（登陆注册是POST请求）

 将数据库MySQL中的用户名和密码加载到服务器的**map中**，**key为用户名，value为密码**。服务器解析请求报文时，当POST请求，提取出请求报文的消息体的用户名和密码。**POST请求中最后是用户名和密码，用&隔开。分隔符&，前是用户名，后是密码**。

 登录：将浏览器输入的优化命和密码在数据库中查找，直接判断。

 注册：往数据库中插入数据，需要判断是否有重复的用户名。



13.4 登录中的用户名和密码你是load到本地，然后使用map匹配的，如果有10亿数据，即使load到本地后hash，也是很耗时的，你要怎么优化？

 **利用hash建立多级索引的方式来加快用户验证**。具体如下：

- 将10亿的用户信息，利用大致缩小1000倍的hash算法进行hash，这时就获得了100万的hash数据，每一个hash数据代表着一个**用户信息块（一级）**；
- 而后，再分别对这100万的hash数据再进行hash，例如最终剩下1000个**hash数据（二级）**。

 服务器只需要保存1000个二级hash数据，当用户请求登录的时候，先对用户信息进行一次hash，找到对应信息块（二级），在读取其对应的一级信息块，最终找到对应的用户数据。



13.5 用户名保存相关

这个可以用HTTP/1.1的cookie技术回答

**HTTP cookie**(也称为web cookie,网络cookie,浏览器cookie或者简称cookie)是网站发送的一个小的数据片段，当用户浏览时，会通过用户的浏览器保存在用户的电脑上。通过Cookie这种可靠的机制，网站可以记录状态信息(例如在电商网站中放到购物车中的物品)或者记录用户的浏览行为(包括特殊按钮的点击，登录或者记录之前访问的页面)。也可以用来记录用户之前输入的字段例如姓名，地址，密码，信用卡号码。



13.6  当write()或read()阻塞时，webserver是否就会被阻塞在这一步无法处理下一个请求和新的就绪事件

对的，当使用阻塞性的 `read()` 或 `write()` 操作时，如果这些操作阻塞了，web服务器就会被阻塞在那一步，无法处理下一个请求和新的就绪事件。这种情况下，web服务器的吞吐量将会大大降低。

在这种情况下，通常我们会选择非阻塞 I/O 或者多线程/多进程方式来处理这个问题。

1. **非阻塞 I/O**：通过将 `read()` 和 `write()` 操作设置为非阻塞，当操作无法立即完成时，这些调用会立即返回一个错误而不是阻塞。然后，你可以使用轮询、select、poll、epoll、kqueue 等 I/O 多路复用技术来检查哪些文件描述符已经就绪，然后再进行 `read()` 或 `write()`。
2. **多线程/多进程**：你可以创建多个线程或进程，每个线程/进程处理一个或多个请求。这样，即使一个线程/进程被 `read()` 或 `write()` 阻塞，其他的线程/进程还可以继续处理新的请求。但是，这种方法也会引入新的复杂性，如线程同步、进程间通信等问题。
3. **异步 I/O**：对于一些操作系统和库，你可以使用异步 I/O (AIO)。这意味着你可以启动一个 `read()` 或 `write()` 操作，然后立即返回，当操作完成时，操作系统会通知你。这允许你在等待 I/O 操作完成的同时做其他事情。

每种方法都有其优点和缺点，所以你需要根据你的具体需求来选择最合适的方法。



13.7 怎样实现同一个账户同一时间只能在一个终端登录

- 在账户表的基础上，新建了一个账户account_session表，用来记录登录账户的account_id和最新一次登录成功用户的session_id，然后首先要修改登录方法：
- 每次登录成功后，要将登录用户信息写入Session的同时还要更新account_session表里相应账户的session_id（当然，如果是第一次登录时，进行的便是插入动作），然后要修改获取当前用户信息的方法，在里面要做两重判断，
- 首先，看当前会话是否存在登录用户信息，如果没有，则肯定是未登录，不再赘述，如果有，还要再进一步要用当前会员里存的account_id去account_session表查询最新的session_id，与当前会员中的session_id作比较，如果是一致的，说明当前会话是最新的会话，登录状态正常，如果不一致，说明在当前登录会话创建后，被新的登录会话覆盖掉了，当前的登录会话已经失效，这时候，服务器应该删除当前的登录会话并返回提示给客户端，至此，限制账户同一时间单终端登录功能便实现了





```c++



```

## 问题分割线-----------

1、inet_ntop()与inet_pton:

 inet_ntop()：将网络字节序转换为点分十进制；inet_pton()：将点分十进制转换为网络字节序；



1.1 如果处于全连接队列中的连接对应的客户端出现网络异常（掉线）或提前退出，那么服务器调用accept()是否成功？

 会成功。accept只是从全连接队列中取出链接，不关心连接处于何种状态（不管是established状态还是close_wait状态），更不关心网络状况的变化。



2、close与shutdown关闭连接的区别：

 close并非总是立马关闭连接，而是将fd的引用计数-1，当fd的引用计数为0时，才会真正关闭连接；而shutdown系统调用会立刻关闭连接，其第二个参数SHUT_RD、SHUT_WR与SHUT_RDWR分别表示立即关闭该文件描述符的读端、写端与读写端。



3、如何设置接收缓冲区与发送缓冲区的大小?

 1.setsockopt()的第三个参数选择SO_RCVBUF和SO_SNDBUF来设置，系统会将其值加倍；tcp接收缓冲区的最小值为256字节，发送缓冲区的最小值为2048字节（不同系统可能不同）。

有最小值的目的：确保一个tcp连接有**足够多的空闲缓冲区来处理拥塞**（比如快速重传算法就期望tcp接收缓冲区能至少容纳4个大小为SMSS的tcp报文段）。

 2.通过直接修改内核参数/proc/sys/net/ipv4/tcp_rmem和/proc/sys/net/ipv4/tcp_wmem强制修改接收与发送缓冲区，且没有最小值限制。



4、如何创建守护进程？僵尸进程？孤儿进程？

 守护进程是以**后台进程的形式运行，没有控制终端，不会收到用户输入，也不会输出到控制终端**。其**父进程为init进程**(PID为1)

 1、**执行一个fork()，退出父进程**

（1、**创建会话的进程不能是首进程**。2、**防止父进程被杀死后，shell显示一个shell提示符**，而守**护进程不能在终端有显示**），

   子进程继续执行。

 **2、子进程调用setsid()开启一个新会话**。创建**一个新的会话，是没有控制终端的，只要控制终端没有和其连接，就没有控制终端，从而脱离控制终端**。

**为什么要子进程调用setsid()？**

因为若是父进程调用，且父进程为首进程，则会**产生两个组id和会话id都一样的会话和组，冲突**。子进程就有自己的进程id，创建的会话和组已有的不同。

 3、**清除进程的umask以确保守护进程创建文件与目录所需要的权限**。

 4、**修改进程的工作目录为根目录**。

 5、**关闭从父进程继承来的所有打开的文件描述符**。

 6、**关闭标准输入、输出设备和标准错误输出设备文件描述符0、1、2，并使用dup2将标准输入、输出设备和标准错误输出设备重定向到/dev/null文件，写到该文件会丢弃**。

 

**僵尸进程与孤儿进程：**

 每个进程结束后，都会**释放自己地址空间的用户区数据**，**内核的PCB无法自己释放，需要父进程释放**。因此子进程终止时，父进程尚未回收子进程残留的PCB资源存放在内存当中，变为僵尸进程。且僵尸进程不能被kill -9杀死。

 **僵尸进程**的危害：如果**父进程不调用wait()或waitpid()的话，子进程内核PCB的信息不会被释放，进程号会一直占着，而系统的进程号是有限的，如果僵尸进程过多，系统就没有可用的进程号导致不能产生新进程**。

 **孤儿进程**：

**父进程运行结束，子进程还在运行**。内核会把孤儿进程的**父进程设置为init进程**，而init进程会循环等待wait其结束的子进程。因此，当孤儿进程产生时，会将其交给党和人名政府。因此**孤儿进程不会产生什么危害**。

 进程是资源分配的基本单位，线程时调度的基本单位。



5、 pthread_creat()注意事项：

 其第三个参数为**处理线程函数的指针，要求为静态函数**。如果处理线程函数为类成员函数时，需要将其设置为**静态成员函数**。若线程函数为类成员函数，则this指针会作为默认的参数被传进函数中，从而和线程函数参数`(void*)`不能匹配，不能通过编译。

静态成员函数就没有这个问题，里面没有this指针。



6、手撕定时器

**定时器：**

 1.需要的数据结构：1.有序；2.增删改后仍有序；3.找到最近要被触发的定时任务

 2.实现触发机制：不需要占用线程（管理异步任务）来触发任务（不使用sleep）

**任务如何组织：**

```
struct TImernode{
	time_t expire;//定时器在什么时刻执行，不能唯一确定一个TimeNode
    unit64_t id;
	callback func;//使用函数对象传递上下文，且调用回调函数
}
```

解决不能唯一确定一个TimeNode方法： 1.堆上创建 timenode*来唯一确定TimeNode 2.生成唯一id的方式

**存储方式(STL中选择)：**

 使用map：map<TimeNodeBase, TimeNode>，需重载<,但是有重复存储的数据，不使用map

```
struct TimeNodeBase{
    time_t expire;	//addTimer	now()+time;
    uint64_t id;	//自增唯一标识TimeNode，用uint64_t确保id不会重复
}
```

 使用set：set只需实现重载一个比较TimeNode的<；为什么不用

```c++
struct TimeNodeBase{
    time_t expire;	//addTimer	now()+time;
    uint64_t id;	//自增唯一标识TimeNode，用uint64_t确保id不会重复
};

struct TimerNode : public TimerNodeBase{
    //使用闭包，就不需要传递上下文参数
    using Callback = std::function<void(const TimerNode& node)>;
    Callback func;
    TimeNode(time_t expire, uint64_t id, Callback func) : func(std::move(func)){
        this->expire = expire;
        this->id = id;
    }
};

//引用时具备多态的
//TimerNode		TimerNodeBase		或者两者之间都可以调用该函数
bool operator < (const TimerNodeBase &lhd, const TimerNodeBase &rhd){
    if(lhd.expire < rhd.expire) return true;
    else if(lhd.expire > rhd.expire) return false;
    else
    {
        return lhd.id < rhd.id;	//用来确保先插入的先执行
    }
}

//我们需要暴露给用户什么对象
//c++14等价key的特性
class Timer{
public:
    //返回最近系统启动到现在的时间 ms
    static inline time_t GetTick(){
        return chrono::duration_cast<chrono::milliseconds>(chrono::steady_clock::now().time_since_epoch()).count();
    }
    
    TimerNodeBase ADDTimer(int time, TimeNode::Callback func){
        time_t expire = GetTick() + time;
        auto pairs = timeouts.emplace(expire, GenID(), std::move(func));
        return static_cast<TimeNodeBase>(*pair.first);
    }
    
    void DelTimer(TimeNodeBase &node){
        auto iter = timeouts.find(node);
        if(iter != timeouts.end())
            timeouts.erase(iter);
    }
private:
    static inline uint64_t GenID(){
        return gid++;
    }
    static uint64_t gid;
	set<TimeNode, std::less<>>timeouts;
};
uint64_t Timer:: gid = 0;
```

**timefd 来管理定时器**

 通过timerfd_creat()创建timerfd，同时注册epoll事件（EPOLLIN|EPOLLET），利用epoll_ctl()将timerfd添加给epoll事件交给内核管理；通过timerfd_settime()来启动定时器；内核检测到定时器超时了就提醒用户，删除定时器。



7、手撕高性能线程池：

**什么是线程池：**

 生产者消费者模型，不希望生产者线程执行任务，任务的共性：耗时（IO相关、复杂的cpu运算）。

 线程池的构成部分：消费者线程；如何传递任务？用队列，因为加锁粒度小。

 **异步任务：**函数返回后，该任务没有完成，由线程池的其他线程其余时间完成。

**为什么需要线程池：**

 1.生产者不想执行耗时操作，从而影响其他任务

 2.性能优化

**线程池队列设计：**

 1、先进先出结构，保证生产任务与获取任务的先后顺序，执行任务的顺序不一定

 2、锁的粒度小。粒度就是加锁与解锁之间程序消耗的时间。锁 原子操作<自旋锁<互斥锁

消费者线程->线程池：如果队列没有任务，线程休眠，有任务唤醒。根据队列的数据状态，使用条件变量与互斥锁来解决



8、为什么需要用户态的网络缓冲区？TCP与UDP的缓冲区有什么区别？

1、TCP为什么需要？

 tcp有粘包的问题，用户层可能收到的不是一个完整的数据包。需要界定一个完整数据包的规则解决：

1、以特殊字符进行分割；2、固定长度分割。

 多余的数据需要保存起来。生产者--消费者模型。

2、UDP为什么需要？

 基于报文传输，在ip层分片，进行ip重组，没有粘包的问题，内核没有发送缓冲区，在用户态也不需要发送缓冲区，无确认机制。内核与用户态有接收缓冲区

 **不能确定一次系统调用收到一个完整的数据包，需要处理粘包；对缓冲区而言，生产者的速度大于消费者的速度。**



9、阻塞io/reactor/proactor（IOCP）缓冲区设计有什么差异？无差异

 发送数据与接收数据是否有差异？差异释放影响缓冲区的设计？

 阻塞IO是通过阻塞线程的方式等待数据的到达。

 reactor：IO多路复用等待数据的到达，通过事件回调的方式处理数据读取

 IOCP(完成端口)：投递一个接收数据的请求，准备一个buffer，在内核中检测并操作IO，直接拷贝到buffer中去，以事件信号的方式通知用户态数据已经读到了用户态。

**具体缓冲区的设计：**

RingBuffer环形缓冲区，避免挪数据（通过两个指针），还是**有定长的缺点**，同时**增加了系统调用的次数**，因为**空间不连续（Linux下通过writev发送不连续的空间）**。

```
int ringbuffer[8*1024];
int* head;//通过取余rindex % (8 * 1024)，逻辑环形,与消费数据相关，读数据
int* tail;//与生产数据相关，写数据
```

**chainbuffer**

缓冲区的设计：数组（定长且需要将为读取的数据移动到数组头部）

--> RingBuffer（通过读写指针避免移动数据，但还是定长，且由于空间不连续，需要增加系统调用的次数来发送不连续空间的数据，Linux下可以通过writev一次发送不连续空间的数据）

--> ChainBuffer（通过多个缓冲区按照链式连接而成，每个缓冲区内部是连续的内存块，通过指针连接多个缓冲区，实现缓冲区动态变化）



10、服务器只能实现5-7万的并发连接QPS，什么突破到20万

达到20万的并发连接QPS是一个相当具有挑战性的目标，但它确实是可能的。下面是一些可能的方向，您可以尝试将其整合到您的架构中以实现这一目标：

1. **硬件优化**：32G16核CPU
   - 升级服务器硬件，例如使用更多核心、更快的CPU，更大的内存。
   - 使用更快的网络硬件和配置，以减少网络延迟。
2. **负载均衡**：
   - 通过引入负载均衡器将请求分发到多个服务器，从而扩展横向扩展能力。
   - 使用具有高性能和低延迟的负载均衡算法。
3. **软件优化**：
   - 使用异步编程，确保服务器能够在等待某些资源（如数据库连接）时继续处理其他请求。
   - 优化代码和数据库查询，减少每个请求的处理时间。
   - 考虑使用更高效的编程语言和框架。
4. **缓存优化**：
   - 使用缓存来存储经常访问的数据，减少对慢速存储的访问。
   - 选择正确的缓存策略和过期机制。
5. **数据库优化**：
   - 分析和优化数据库查询，确保它们尽可能快速。
   - 考虑数据库分片或使用读/写分离以提高吞吐量。
6. **分布式系统和微服务架构**：
   - 将应用程序分解为更小的、可独立扩展的部分，可以更灵活地调整不同部分的资源。
   - 使用分布式数据库和分布式缓存解决方案。
7. **监控和调优**：
   - 使用性能监控工具持续监控系统性能，并及时识别和解决瓶颈。
   - 根据实际负载调整服务器和应用程序参数。
8. **流量控制**：
   - 在必要时应用限流和退化策略，确保系统不会被突然的流量激增所压垮。
9. **内容分发网络（CDN）**：
   - 如果适用，可以使用CDN来缓存和更快地提供静态内容。
10. **协议优化**：

- 考虑使用更有效的通信协议，如HTTP/2或gRPC。







### 存放线程执行任务的结构体或者类型是什么(std::function )

`std::function` 函数封装器。一个模板类，可以存储几乎任何可调用对象，包括函数指针、lambda表达式、bind表达式和其他函数对象。被用于多线程编程作为任务的表示方式。

```c++
cppCopy codestd::function<void()> myTask = []() { 
    // ... do something ... 
};
```

### 线程 A如何向线程B发起异步请求并获取到处理结果、接口是什么

**std::promise and std::future**:

`std::promise` 用于在一个线程中存储某种数据值，以便将来在其他线程中使用。与 `std::promise` 配套使用的是 `std::future`，它可以从与 `std::promise` 对象关联的 `std::future` 对象中检索值。

**std::async**:

`std::async` 是一个更简单的方法，可以异步地启动任务，并返回一个 `std::future`，用于之后获取结果。

### 线程池有的线程出现问题比如说崩溃了 怎么解决的

**异常处理**：确保线程的任务有适当的异常处理机制。捕获并适当处理异常可以避免线程崩溃。

**健康检查**：线程池可以定期进行线程健康检查。如果检测到线程不再响应或已经死亡，线程池可以关闭该线程并启动一个新线程来替换它。

### 线程池中某个线程被持续占用怎么办

设置请求大小:避免过大的请求 (请求体大小通常限制在 1~2MB 左右)

设置超时时间:超时则直接释放线程资源

异步 I/0: 后台线程异步执行 (比如 NIO 或 Netty)（C++中如何捕获异步结果了 future和promise）
调整线程池大小:当线程全部被占用时、创建更多的线程处理请求(使用完放回线程池)

请求队列:新的请求可以放入队列、等待有线程可用时再处理



### 如果在现有的并发量基础上访问量突然增多怎么解决

1. **动态调整线程池大小**：线程池根据工作负载动态地增加或减少线程数。当检测到负载增加时，可以增加线程数量来处理更多的任务。
2. **请求队列**：如果所有线程都在忙碌状态，新来的任务可以被放在一个队列中等待。当有线程可用时，它会从队列中取出任务来执行。
3. **负载均衡**：如果你的系统分布在多个服务器上，使用负载均衡器可以确保每个服务器都处理合适的请求量。
4. **优先级队列**：为不同的任务设置优先级。关键任务可以被优先处理，而低优先级的任务在高峰时期可以被延迟或丢弃。
5. **缓存策略**：对于常见和重复的请求，使用缓存可以大大减少实际的处理负载。



###  用户名保存相关

这个可以用HTTP/1.1的cookie技术回答

**HTTP cookie**(也称为web cookie,网络cookie,浏览器cookie或者简称cookie)是网站发送的一个小的数据片段，当用户浏览时，会通过用户的浏览器保存在用户的电脑上。通过Cookie这种可靠的机制，网站可以记录状态信息(例如在电商网站中放到购物车中的物品)或者记录用户的浏览行为(包括特殊按钮的点击，登录或者记录之前访问的页面)。也可以用来记录用户之前输入的字段例如姓名，地址，密码，信用卡号码。



### 当write()、read()阻塞，服务器是否会被阻塞无法处理下一个请求和新的就绪事件

对的，当使用阻塞性的 `read()` 或 `write()` 操作时，如果这些操作阻塞了，web服务器就会被阻塞在那一步，无法处理下一个请求和新的就绪事件。这种情况下，web服务器的吞吐量将会大大降低。

在这种情况下，通常我们会选择非阻塞 I/O 或者多线程/多进程方式来处理这个问题。

1. **非阻塞 I/O**：通过将 `read()` 和 `write()` 操作设置为非阻塞，当操作无法立即完成时，这些调用会立即返回一个错误而不是阻塞。然后，你可以使用轮询、select、poll、epoll、kqueue 等 I/O 多路复用技术来检查哪些文件描述符已经就绪，然后再进行 `read()` 或 `write()`。
2. **多线程/多进程**：你可以创建多个线程或进程，每个线程/进程处理一个或多个请求。这样，即使一个线程/进程被 `read()` 或 `write()` 阻塞，其他的线程/进程还可以继续处理新的请求。但是，这种方法也会引入新的复杂性，如线程同步、进程间通信等问题。
3. **异步 I/O**：对于一些操作系统和库，你可以使用异步 I/O (AIO)。这意味着你可以启动一个 `read()` 或 `write()` 操作，然后立即返回，当操作完成时，操作系统会通知你。这允许你在等待 I/O 操作完成的同时做其他事情。

每种方法都有其优点和缺点，所以你需要根据你的具体需求来选择最合适的方法。