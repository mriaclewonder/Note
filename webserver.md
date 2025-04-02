# **WebServer**

服务器程序通常需要处理三类事件：I/O事件，信号及定时事件。有两种事件处理模式：

- `Reactor`模式：要求主线程（I/O处理单元）只负责监听文件描述符上是否有事件发生（可读、可写），若有，则立即通知工作线程（逻辑单元），将socket可读可写事件放入请求队列，交给工作线程处理。**这个过程是同步的，读取完数据后应用进程才能处理数据。**
- `Proactor`模式：**将所有的I/O操作都交给主线程和内核来处理**（进行读、写），工作线程仅负责处理逻辑，如主线程读完成后users[sockfd].read()，选择一个工作线程来处理客户请求pool->append(users + sockfd)。

**Reactor 可以理解为「来了事件操作系统直接通知，自己啥也不干，让子线程来处理读写」，而 Proactor 可以理解为「来了事件操作系统来处理，处理完再通知主线程」。这里的「事件」就是有新连接、有数据可读、有数据可写的这些 I/O 事件。这里的「处理」包含从驱动读取到内核以及从内核读取到用户空间。**

举个实际生活中的例子，Reactor 模式就是快递员在楼下，给你打电话告诉你快递到你家小区了，你需要自己下楼来拿快递。而在 Proactor 模式下，快递员直接将快递送到你家门口，然后通知你。

无论是 Reactor，还是 Proactor，都是一种基于「事件分发」的网络编程模式，区别在于 Reactor 模式是基于「待完成」的 I/O 事件，而 Proactor 模式则是基于「已完成」的 I/O 事件。

`理论上来说Proactor是更快的`。由于Proactor模式需要异步I/O的一套接口，而在Linux环境下，没有异步AIO接口，`想要实现只有用同步I/O模拟实现Proactor`，即主线程完成读写后通知工作线程。但意义不大，所以大部分都是采用`Reactor`。



## 什么是心跳机制？

就是每隔几分钟发送一个固定信息给服务端，服务端收到后回复一个固定信息如果服务端几分钟内没有收到客户端信息则视客户端断开。

发包方：可以是客户也可以是服务端，看哪边实现方便合理。
心跳包之所以叫心跳包是因为：它像心跳一样每隔固定时间发一次，以此来告诉服务器，这个客户端还活着。事实上这是为了保持长连接，至于这个包的内容，是没有什么特别规定的，不过一般都是很小的包，或者只包含包头的一个空包。心跳包主要也就是用于长连接的保活和断线处理。一般的应用下，判定时间在30-40秒比较不错。如果实在要求高，那就在6-9秒。

### 心跳包的发送，通常有两种技术：

**1.应用层自己实现的心跳包**
由应用程序自己发送心跳包来检测连接是否正常，服务器每隔一定时间向客户端发送一个短小的数据包，然后启动一个线程，在线程中不断检测客户端的回应， 如果在一定时间内没有收到客户端的回应，即认为客户端已经掉线；同样，如果客户端在一定时间内没有收到服务器的心跳包，则认为连接不可用。

**2.使用SO_KEEPALIVE套接字选项**
在TCP的机制里面，本身是存在有心跳包的机制的，也就是TCP的选项. 不论是服务端还是客户端，一方开启KeepAlive功能后，就会自动在规定时间内向对方发送心跳包， 而另一方在收到心跳包后就会自动回复，以告诉对方我仍然在线。因为开启KeepAlive功能需要消耗额外的宽带和流量，所以TCP协议层默认并不开启默认的KeepAlive超时需要7,200，000 MilliSeconds， 即2小时，探测次数为5次。对于很多服务端应用程序来说，2小时的空闲时间太长。因此，我们需要手工开启KeepAlive功能并设置合理的KeepAlive参数

### 开启KeepAlive选项后会导致的三种情况：

1、对方接收一切正常：以期望的ACK响应，2小时后，TCP将发出另一个探测分节
2、对方已崩溃且已重新启动：以RST响应。套接口的待处理错误被置为ECONNRESET，套接口本身则被关闭。
3、对方无任何响应：套接口的待处理错误被置为ETIMEOUT，套接口本身则被关闭.

一个服务器通常会连接多个客户端，因此由用户在应用层自己实现心跳包，代码较多 且稍显复杂。用TCP／IP协议层为内置的KeepAlive功能来实现心跳功能则简单得多。心跳包在按流量计费的环境下增加了费用.但TCP得在连接闲置2小时后才发送一个保持存活探测段，所以通常的方法是将保持存活参数改小，但这些参数按照内核去维护，而不是按照每个套接字维护，因此改动它们会影响所有开启该选项的套接字。

## tcp粘包

#### 1.Q：什么是TCP粘包问题？

TCP粘包就是指发送方发送的若干包数据到达接收方时粘成了一包，从接收缓冲区来看，后一包数据的头紧接着前一包数据的尾，出现粘包的原因是多方面的，可能是来自发送方，也可能是来自接收方。

#### 2.Q：造成TCP粘包的原因

（1）发送方原因

TCP默认使用Nagle算法（主要作用：减少网络中报文段的数量），而Nagle算法主要做两件事：

1. 只有上一个分组得到确认，才会发送下一个分组
2. 收集多个小分组，在一个确认到来时一起发送

Nagle算法造成了发送方可能会出现粘包问题

（2）接收方原因

TCP接收到数据包时，并不会马上交到应用层进行处理，或者说应用层并不会立即处理。实际上，TCP将接收到的数据包保存在接收缓存里，然后应用程序主动从缓存读取收到的分组。这样一来，如果TCP接收数据包到缓存的速度大于应用程序从缓存中读取数据包的速度，多个包就会被缓存，应用程序就有可能读取到多个首尾相接粘到一起的包。

#### 3.Q：什么时候需要处理粘包现象？

1. 如果发送方发送的多组数据本来就是同一块数据的不同部分，比如说一个文件被分成多个部分发送，这时当然不需要处理粘包现象
2. 如果多个分组毫不相干，甚至是并列关系，那么这个时候就一定要处理粘包现象了

#### 4.Q：如何处理粘包现象？

（1）发送方

对于发送方造成的粘包问题，可以通过关闭Nagle算法来解决，使用TCP_NODELAY选项来关闭算法。

（2）接收方

接收方没有办法来处理粘包现象，只能将问题交给应用层来处理。

（2）应用层

应用层的解决办法简单可行，不仅能解决接收方的粘包问题，还可以解决发送方的粘包问题。

解决办法：循环处理，应用程序从接收缓存中读取分组时，读完一条数据，就应该循环读取下一条数据，直到所有数据都被处理完成，但是如何判断每条数据的长度呢？

1. 格式化数据：每条数据有固定的格式（开始符，结束符），这种方法简单易行，但是选择开始符和结束符时一定要确保每条数据的内部不包含开始符和结束符。
2. 发送长度：发送每条数据时，将数据的长度一并发送，例如规定数据的前4位是数据的长度，应用层在处理时可以根据长度来判断每个分组的开始和结束位置。

#### 5.Q：UDP会不会产生粘包问题呢？

TCP为了保证可靠传输并减少额外的开销（每次发包都要验证），采用了基于流的传输，基于流的传输不认为消息是一条一条的，是无保护消息边界的（保护消息边界：指传输协议把数据当做一条独立的消息在网上传输，接收端一次只能接受一条独立的消息）。

**UDP则是面向消息传输的，是有保护消息边界的，接收方一次只接受一条独立的信息，所以不存在粘包问题。**

举个例子：有三个数据包，大小分别为2k、4k、6k，如果采用UDP发送的话，不管接受方的接收缓存有多大，我们必须要进行至少三次以上的发送才能把数据包发送完，但是使用TCP协议发送的话，我们只需要接受方的接收缓存有12k的大小，就可以一次把这3个数据包全部发送完毕。



## 线程同步机制封装类

> 封装锁，信号量，条件变量

#### **锁机制的功能**

- 实现多线程同步，通过锁机制，确保任一时刻只能有一个线程能进入关键代码段.

#### **封装的功能**

- 类中主要是Linux下三种锁进行封装，将锁的创建于销毁函数封装在类的构造与析构函数中，实现RAII机制

实现RAII

### **信号量**

>linux sem 信号量是一种特殊的变量，访问具有原子性， 用于解决进程或线程间共享资源引发的同步问题。
>
>用户态进程对 sem 信号量可以有以下两种操作：
>
>- 等待信号量
>  当信号量值为 0 时，程序等待；当信号量值大于 0 时，信号量减 1，程序继续运行。
>- 发送信号量
>  将信号量值加 1
>
>通过对信号量的控制，从而实现共享资源的顺序访问。

```c
#include <semaphore.h>

int sem_init(sem_t *sem, int pshared, unsigned int value);
```

该函数初始化由 sem 指向的信号对象，并给它一个初始的整数值 value。

pshared 控制信号量的类型，`值为 0` 代表该信号量用于**多线程间的同步**，值如果`大于 0` 表示可以共享，用于**多个相关进程间的同步**

参数 pshared > 0 时指定了 sem 处于共享内存区域，所以可以在进程间共享该变量



## 日志类

![image-20250322190850545](D:/Code/TYPORA/Note/webserver.assets/image-20250322190850545.png)



### 为什么如果是异步日志，就把日志放进队列里让其他线程处理

在这段日志系统中，“异步日志”与“同步日志”的核心区别在于 **写入文件操作由谁执行、何时执行**。

- **同步日志**：
   每次调用 `write_log()`，都会立即调用 `fputs()` 等函数将日志内容写入磁盘文件，直到写完才返回。这样会让调用日志的线程（往往是业务线程）在写磁盘 IO 时被阻塞，增加响应时间。
- **异步日志**：
   在初始化时，若指定了 `max_queue_size >= 1`，则开启异步模式（`m_is_async = true`），此时日志系统会：
  1. **将日志内容压入阻塞队列**（`m_log_queue`），立刻返回，不做实际的文件写入。
  2. **由另一个专门的日志线程**（通过 `pthread_create(&tid, NULL, flush_thread_log, NULL)` 创建）不断从队列中取出日志内容并写入磁盘文件。

这样做的原因是：

1. **减少主线程阻塞**
    主线程（或者说业务线程）只需要把日志消息打包好并压入队列，立刻就可以返回继续做其他工作，而不是等待磁盘 IO 完成。磁盘 IO 是相对耗时的操作，如果所有线程都在写日志时同步阻塞，可能会显著降低系统吞吐量。
2. **提升系统吞吐量**
    由一个单独的日志线程统一处理所有日志的写入，主线程和其他工作线程不会因日志写入而停顿，系统整体可以处理更多请求、执行更多业务逻辑。
3. **实现方式**
   - 在 `init()` 函数中，如果 `max_queue_size >= 1`，就创建了 `m_log_queue`（一个阻塞队列）以及专门用于写日志的线程（`flush_thread_log`）。
   - 每次调用 `write_log()` 时，若异步模式开启且队列未满，就把格式化好的日志字符串 `log_str` **push** 到 `m_log_queue`。
   - 日志线程则在 `flush_thread_log` 中不断从 `m_log_queue` 中 **pop** 出日志内容并写入文件。

因此，**“为什么如果是异步日志，就把日志放进队列里让其他线程处理？”** —— 正是为了让主线程快速返回，不被磁盘 IO 阻塞，从而提高系统整体的并发能力和性能。



## **数据库连接池**

```cpp
#include "sql_connection_pool.h"

connection_pool::connection_pool()
{
    m_CurConn = 0;
    m_FreeConn = 0;
}

connection_pool *connection_pool::GetInstance()
{

    static connection_pool instance;
    return &instance;
}

void connection_pool::init(std::string url, std::string User, std::string PassWord, std::string DatabaseName, int port, int MaxConn, int close_log)
{
    m_url = url;
    m_User = User;
    m_PassWord = PassWord;
    m_DatabaseName = DatabaseName;
    m_close_log = close_log;
    for (int i = 0; i < MaxConn; i++)
    {
        MYSQL *conn = NULL;
        conn = mysql_init(conn);
        if (conn == NULL)
        {
            LOG_ERROR("Mysql Error");
            exit(1);
        }
        conn = mysql_real_connect(conn, url.c_str(), User.c_str(), PassWord.c_str(), DatabaseName.c_str(), port, NULL, 0);
        if (conn == NULL)
        {
            LOG_ERROR("Mysql Error");
            exit(1);
        }

        connList.push_back(conn);
        ++m_FreeConn;
    }
    reserve = sem(m_FreeConn);
    m_MaxConn = m_FreeConn;
}

// 获取一个 连接池 里面的连接
MYSQL *connection_pool::GetConnection()
{
    MYSQL *con = NULL;
    if (0 == connList.size())
    {
        return NULL;
    }

    reserve.wait();

    lock.lock();

    con = connList.front();
    connList.pop_front();

    ++m_CurConn;
    --m_FreeConn;

    lock.unlock();
    return con;
}

int connection_pool::GetFreeConn()
{
    return this->m_FreeConn;
}

// 释放链接
bool connection_pool::ReleaseConnection(MYSQL *conn)
{
    if (conn == NULL)
        return false;
    lock.lock();

    connList.push_back(conn);
    ++m_FreeConn;
    --m_CurConn;

    lock.lock();

    reserve.post();
    return true;
}

void connection_pool::DestroyPool()
{
    lock.lock();
    if (connList.size() > 0)
    {
        std::list<MYSQL *>::iterator it;
        for (it = connList.begin(); it != connList.end(); it++)
        {
            MYSQL *con = *it;
            mysql_close(con);
        }
        m_CurConn = 0;
        m_FreeConn = 0;
        connList.clear();
    }

    lock.unlock();
}

connection_pool ::~connection_pool()
{
    DestroyPool();
}

connectionRAII::connectionRAII(MYSQL **SQL, connection_pool *connPool)
{
    *SQL = connPool->GetConnection();
    conRAII = *SQL;
    poolRAII = connPool;
}

connectionRAII::~connectionRAII()
{
    poolRAII->ReleaseConnection(conRAII);
}
```

RAII（Resource Acquisition Is Initialization）思想体现在 `connectionRAII` 类的设计中，通过对象的生命周期来自动管理数据库连接资源。下面详细介绍创建对象和调用函数的过程，以及 RAII 在其中的体现。

------

### 1. RAII 在 `connectionRAII` 类中的体现

### 对象构造阶段（资源获取）

- 构造函数：

  当你创建一个 connectionRAII对象时

  ```cpp
  connectionRAII connRAII(&conn, pool)
  ```

  构造函数会被自动调用。

  1. 获取连接：

     在构造函数内部，调用传入的连接池对象 connPool的 GetConnection()方法，从连接池中获取一个 MySQL 连接。

     ```cpp
     *SQL = connPool->GetConnection();
     ```

     此时，*SQL（即传入的 MYSQL 指针）和 connectionRAII对象内部的 conRAII成员都会保存这个连接指针。

  2. **保存连接池指针：**
      构造函数还保存了连接池的指针到 `poolRAII` 成员变量，以便后续在析构时调用释放函数。

这样，在对象创建时就“获取”了资源，也就是数据库连接，这正是 RAII 的核心思想之一：**资源的获取绑定到对象的初始化阶段**。

### 对象析构阶段（资源释放）

- 析构函数：

  当 connectionRAII 对象生命周期结束（例如，函数执行完毕或出现异常导致提前退出），析构函数会被自动调用。

  1. 释放连接：

     在析构函数中，会调用之前保存的连接池对象的 ReleaseConnection()方法，将连接归还给连接池：

     ```cpp
     poolRAII->ReleaseConnection(conRAII);
     ```

     这样，无论函数是否正常结束，都保证了数据库连接能够被正确释放，避免了资源泄露问题。

### 2. 创建对象和函数调用过程详解

假设你在某个函数中需要使用数据库连接，可以按如下步骤使用 RAII 模式管理连接资源：

1. **获取连接池实例：**
    先通过 `connection_pool::GetInstance()` 获取单例连接池对象，并调用 `init()` 方法初始化连接池，预先创建一定数量的 MySQL 连接。

   ```cpp
   connection_pool* pool = connection_pool::GetInstance();
   pool->init(url, User, PassWord, DatabaseName, port, MaxConn, close_log);
   ```

2. **使用 RAII 类创建对象：**
    在需要操作数据库的函数中，定义一个 `MYSQL*` 指针，并创建 `connectionRAII` 对象。

   ```cpp
   MYSQL* conn;
   connectionRAII connRAII(&conn, pool);
   ```

   - **构造函数调用：**
      此时，`connectionRAII` 的构造函数被调用，会自动调用 `pool->GetConnection()` 获取一个数据库连接，并赋值给 `conn`。
   - **此时，你就可以直接使用 `conn` 进行数据库操作。**

3. **自动释放资源：**
    当函数执行完毕或者发生异常，局部变量 `connRAII` 会自动销毁：

   - **析构函数调用：**
      在 `connectionRAII` 的析构函数中，会自动调用 `pool->ReleaseConnection(conn)` 将连接归还到连接池，从而实现资源的自动释放。

------

### 3. 函数作用概述

#### 在 `connection_pool` 类中

- **`GetConnection()`：**
  - **作用：** 从连接池中取出一个 MySQL 连接。
  - 过程：
    1. 判断连接池是否为空。
    2. 调用信号量 `wait()`，如果没有可用连接则阻塞等待。
    3. 进入临界区（通过互斥量 `lock` 加锁），从连接列表中取出一个连接，并更新连接数量。
    4. 释放锁并返回连接指针。
- **`ReleaseConnection(MYSQL \*conn)`：**
  - **作用：** 将一个 MySQL 连接归还给连接池。
  - 过程：
    1. 进入临界区，使用互斥量保护连接列表。
    2. 将连接加入连接列表，更新连接计数。
    3. 调用信号量 `post()`，通知等待的线程有新连接可用。
    4. 释放锁。
- **`DestroyPool()`：**
  - **作用：** 关闭所有连接，销毁连接池。
  - 过程：
    1. 进入临界区后遍历所有连接并调用 `mysql_close()`。
    2. 清空连接列表，重置连接计数。

#### 在 `connectionRAII` 类中

- **构造函数：**
  - **作用：** 获取数据库连接并初始化 RAII 对象。
  - 调用过程：
    1. 调用连接池的 `GetConnection()` 获取连接。
    2. 将获取到的连接指针传给外部变量和对象内部成员。
- **析构函数：**
  - **作用：** 自动释放数据库连接，将其归还给连接池。
  - 调用过程：
    1. 当对象生命周期结束时，调用连接池的 `ReleaseConnection()`。

### 总结

- **RAII 的核心思想：** 将资源的获取（数据库连接）绑定到对象的初始化，将资源的释放绑定到对象的销毁。
- 使用 `connectionRAII`：
  - 在对象构造时，通过调用 `connection_pool::GetConnection()` 获取连接并分配给变量。
  - 在对象析构时，自动调用 `connection_pool::ReleaseConnection()` 归还连接，无需手动管理资源释放。
- 优点：
  - 简化资源管理，避免因异常或提前返回导致的资源泄露问题。
  - 代码清晰，责任明确，易于维护和调试。

这种设计模式在 C++ 中非常常见，不仅可以用在数据库连接管理上，还可以应用于文件、内存等其他资源的管理。



## 线程池

以下是对 **半同步/半反应堆线程池** 及其代码实现的详细解析：



#### **1. 半同步/半反应堆（Half-Sync/Half-Reactive）**
- **同步层**：工作线程池同步处理任务（如业务逻辑、数据库操作）。  
- **反应堆层**：主线程（或I/O线程）异步监听事件（如网络请求），并将就绪的任务投递到同步层。  
- **特点**：  
  - 主线程仅负责事件通知，不阻塞业务处理。  
  - 工作线程竞争任务，实现并发处理。

#### **2. Proactor 模式**
- **定义**：异步I/O模式，主线程（或操作系统）完成I/O操作（如数据读写），工作线程直接处理结果。  
- **模拟方式**：  
  - 主线程 **同步阻塞** 完成I/O（如 `read`/`write`），然后将任务交给线程池处理。  
  - 虽然I/O是同步的，但业务处理是异步的，从而模拟Proactor的流程。



以下代码实现了 **半同步/半反应堆线程池**，并通过同步I/O模拟Proactor模式：

#### **1. 线程池初始化**
```cpp
template <typename T>
threadpool<T>::threadpool(int actor_model, connection_pool *connPool, int thread_number, int max_requests)
    : m_actor_model(actor_model), m_connPool(connPool), m_thread_number(thread_number),
      m_max_requests(max_requests), m_threads(NULL), m_stop(false) {
    // 创建线程并启动 worker 函数
    for (int i = 0; i < thread_number; i++) {
        pthread_create(m_threads + i, NULL, worker, this); // 工作线程启动
    }
}
```
- **主线程**：负责监听I/O事件（如 `epoll_wait`，代码未展示）。  
- **工作线程**：通过 `worker` 函数调用 `run()`，从任务队列获取任务。

---

#### **2. 主线程投递任务（模拟Proactor）**
```cpp
// 主线程同步完成数据读取后，调用 append 添加任务
template <typename T>
bool threadpool<T>::append(T *request) {
    locker_guard lock(m_queuelocker); // RAII 加锁
    if (m_workqueue.size() >= m_max_requests) return false;
    m_workqueue.push_back(request);  // 将任务加入队列
    m_requestat.post();              // 信号量通知工作线程
    return true;
}
```
- **流程**：  
  1. 主线程通过同步I/O（如 `read`）读取数据。  
  2. 封装数据为 `request` 对象，调用 `append` 将其加入任务队列。  
  3. 通过信号量 `m_requestat` 唤醒工作线程处理任务。  

---

#### **3. 工作线程处理任务（同步层）**
```cpp
template <typename T>
void threadpool<T>::run() {
    while (!m_stop) {
        m_requestat.wait();        // 等待信号量（任务到来）
        m_queuelocker.lock();
        T *request = m_workqueue.front();
        m_workqueue.pop_front();
        m_queuelocker.unlock();

        // Proactor 模式：直接处理已读取的数据
        connectionRAII mysqlconn(&request->mysql, m_connPool);
        request->process(); // 业务处理（如生成HTTP响应）
    }
}
```
- **Proactor 模拟**：  
  - 主线程完成I/O后，任务数据已就绪（如 `request` 包含完整HTTP请求）。  
  - 工作线程只需处理业务逻辑（如调用 `process()` 生成响应）。  

---

#### **4. Reactor 模式对比**
```cpp
if (1 == m_actor_model) { // Reactor 模式
    if (0 == request->m_state) { // 读阶段
        if (request->read_once()) { // 工作线程自己读取数据
            request->process();     // 处理数据
        }
    } else { // 写阶段
        request->write(); // 工作线程自己发送数据
    }
}
```
- **Reactor 模式**：  
  - 主线程仅通知事件就绪（如可读/可写）。  
  - 工作线程需自行完成I/O操作（如 `read_once()` 和 `write()`）。

---

### **模式对比总结**
| **特性**       | **Proactor（模拟）**          | **Reactor**        |
| -------------- | ----------------------------- | ------------------ |
| **I/O 操作**   | 主线程同步完成（如 `read`）   | 工作线程异步完成   |
| **业务处理**   | 工作线程处理                  | 工作线程处理       |
| **代码复杂度** | 主线程简单，工作线程无I/O逻辑 | 工作线程需处理I/O  |
| **适用场景**   | 高并发短连接（如HTTP）        | 长连接或自定义协议 |

---

### **关键设计点**
1. **任务队列解耦**  
   - 主线程与工作线程通过队列通信，避免直接耦合。  
   - 互斥锁（`m_queuelocker`）和信号量（`m_requestat`）保证线程安全。

2. **资源管理**  
   - `connectionRAII` 类自动管理数据库连接，确保资源释放。  

3. **模式切换**  
   - 通过 `m_actor_model` 灵活选择 Reactor/Proactor 模式。

---

### **总结**
- **半同步/半反应堆**：主线程异步监听事件，工作线程同步处理任务。  
- **Proactor 模拟**：主线程同步完成I/O，工作线程仅处理业务，实现高效分工。  
- **代码价值**：通过线程池和任务队列，平衡I/O与CPU密集型操作，提升并发性能。

### 如何用同步 I/O 模拟 Proactor？

#### **代码中的模拟逻辑**

- **主线程职责**：
  1. 通过 **同步阻塞 I/O**（如 `read`）读取数据。
  2. 将读取完成的数据封装为任务，投递到线程池队列。
- **工作线程职责**：
  直接处理数据（无需关心 I/O 操作）。

```cpp
#ifndef THREADPOOL_H
#define THREADPOOL_H
#include "../CGIMysql/sql_connection_pool.h"
#include "../lock/locker.h"

#include <list>
#include <cstdio>
#include <exception>
#include <pthread.h>

template <typename T>
class threadpool
{
public:
    threadpool(int actor_model, connection_pool *connPool, int thread_number = 8, int max_requests = 1000);
    ~threadpool();

    bool append(T *request, int state);
    bool append(T *request);

private:
    // 静态工作线程函数，每个线程启动后调用worker
    static void *worker(void *arg);

    // 真正执行任务处理的函数
    void run();

private:
    int m_thread_number;        // 线程的数量
    int m_max_requests;         // 请求队列允许的最大请求数
    pthread_t *m_threads;       // 线程池数组
    std::list<T *> m_workqueue; // 请求队列
    locker m_queuelocker;
    sem m_requestat;
    connection_pool *m_connPool; // 数据库连接池
    int m_actor_model;           // 模型切换标志
};

template <typename T>
threadpool<T>::threadpool(int actor_model, connection_pool *connPool, int thread_number, int max_requests) : m_actor_model(actor_model), m_connPool(connPool), m_thread_number(thread_number), m_max_requests(max_requests), m_threads(NULL)
{
    // 判断 线程池的 大小
    if (thread_number <= 0 || max_requests <= 0)
        throw std::exception();

    m_threads = new pthread_t[m_thread_number];
    if (!m_threads)
        throw std::exception();

    for (int i = 0; i < thread_number; i++)
    {
        // 创建线程
        if (pthread_create(m_threads + i, NULL, worker, this) != 0)
        {
            delete[] m_threads;
            throw std::exception();
        }
        if (pthread_detach(m_threads[i]))
        {
            delete[] m_threads;
            throw std::exception();
        }
    }
}

template <typename T>
threadpool<T>::~threadpool()
{
    delete[] m_threads;
}

template <typename T>
bool threadpool<T>::append(T *request, int state)
{
    m_queuelocker.lock();
    if (m_workqueue.size >= m_max_requests)
    {
        m_queuelocker.unlock();
        return false;
    }

    request->m_state = state;
    m_workqueue.push_back(request);
    m_queuelocker.unlock();
    m_requestat.post();
    return true;
}

template <typename T>
bool threadpool<T>::append(T *request)
{
    m_queuelocker.lock();
    if (m_workqueue.size() >= m_max_requests)
    {
        m_queuelocker.unlock();
        return false;
    }

    m_workqueue.push_back(request);
    m_queuelocker.unlock();
    m_requestat.post();
    return true;
}

template <typename T>
void *threadpool<T>::worker(void *arg)
{
    threadpool *pool = (threadpool *)arg;
    pool->run();
    return pool;
}

template <typename T>
void threadpool<T>::run()
{
    while (true)
    {
        m_requestat.wait();
        m_queuelocker.lock();

        if (m_workqueue.empty())
        {
            m_queuelocker.unlock();
            continue;
        }

        T *request = m_workqueue.front();
        m_workqueue.pop_front();
        m_queuelocker.unlock();

        if (!request)
        {
            continue;
        }

        if (1 == m_actor_model) // Reactor
        {
            if (0 == request->m_state) // 读
            {
                if (request->read_once())
                {
                    request->improv = 1;
                    connectionRAII mysqlconn(&request->mysql, m_connPool); // 管理连接
                    request->process();                                    // 处理请求
                }
                else
                {
                    request->improv = 1;
                    request->timer_flag = 1;
                }
            }
            else // 写
            {
                if (request->write())
                {
                    request->improv = 1;
                }
                else
                {
                    request->improv = 1;
                    request->timer_flag = 1;
                }
            }
        }
        else // Proactor模式
        {
            connectionRAII mysqlconn(&request->mysql, m_connPool); // 管理连接
            request->process();                                    // 处理请求
        }
    }
}

#endif
```

## 定时器

### **1. EPOLLONESHOT**

#### **作用**

- **单次触发模式**：当使用 `epoll_ctl` 为文件描述符设置 `EPOLLONESHOT` 事件标志后，该描述符的指定事件（如可读或可写）**只会触发一次通知**，直到通过 `epoll_ctl` 重新注册该事件。

#### **典型用途**

- **多线程环境**：防止多个线程同时处理同一个文件描述符的事件（避免数据竞争）。
- **精准控制**：确保事件被处理后再重新监听（如处理完数据后再重新注册读事件）。

```c
// 注册 EPOLLIN 事件，并设置 EPOLLONESHOT
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLONESHOT;
ev.data.fd = sockfd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);

// 处理完事件后需重新注册
epoll_ctl(epoll_fd, EPOLL_CTL_MOD, sockfd, &ev);  // 重新激活事件
```

###  作用

EPOLLRDHUP

当 `epoll_wait()` 监听的 `fd`（套接字）触发 `EPOLLRDHUP` 事件时，意味着：

- **对端调用了 `shutdown(fd, SHUT_WR)` 或 `close(fd)`**，即对方不再发送数据了。
- **本端仍然可以继续发送数据**，但对方不会再发送数据过来了。

相当于 **对方关闭了写入方向（write shutdown），但你的 socket 仍然可以写数据**。



### **`SIGALRM`**定时器信号

#### **功能**

- **触发条件**：由 `alarm()` 或 `setitimer()` 等函数设置的定时器超时触发。
- **默认行为**：终止进程（Terminate）。
- 常见用途
  - 实现超时机制（如网络请求超时）。
  - 周期性任务调度（如心跳检测）。
  - 资源使用时间限制（如防止代码死循环）。

>定时器处理非活动连接
>
>===============
>
>由于非活跃连接占用了连接资源，严重影响服务器的性能，通过实现一个服务器定时器，处理这种非活跃连接，释放连接资源。利用alarm函数周期性地触发SIGALRM信号,该信号的信号处理函数利用管道通知主循环执行定时器链表上的定时任务.
>
>\> * 统一事件源
>
>\> * 基于升序链表的定时器
>
>\> * 处理非活动连接

## http

### Http 请求的头部部分

 HTTP 请求的头部部分如下:

```c
GET /index.html HTTP/1.1
Host: www.example.com
Connection: keep-alive
Content-length: 123
```

### 函数

**strspn** 函数用于检索字符串中连续匹配字符集的长度。它返回字符串 *str1* 中从开头连续包含在 *str2* 中的字符数量

strchr() 用于查找字符串中的一个字符，并返回该字符在字符串中第一次出现的位置。

**atol（将字符串转换成长整型数）** 



### **何时会触发 `EAGAIN`？**

`EAGAIN` 是 Linux/Unix 系统编程中一个常见的错误码，尤其在非阻塞 I/O 操作中频繁出现。它的核心含义是：**当前操作无法立即完成，但稍后重试可能成功**。以下是深入解析：

在非阻塞（non-blocking）文件描述符上执行 I/O 操作时：

- **读操作**：内核缓冲区无数据可读，立即返回 `EAGAIN`。
- **写操作**：内核缓冲区已满，无法写入更多数据，立即返回 `EAGAIN`。

此时程序不应阻塞等待，而是应该通过事件驱动（如 `epoll`）监听文件描述符状态，待其就绪后再重试。



### `struct iovec` 

`struct iovec` 是 Linux/Unix 系统中用于 **分散-聚集 I/O（Scatter-Gather I/O）** 的核心数据结构，允许程序通过单个系统调用（如 `writev` 或 `readv`）高效处理多个非连续内存缓冲区的读写操作。它在高性能网络编程（如 HTTP 服务器）和文件 I/O 优化中广泛应用。

#### 核心作用

通过 `struct iovec` 数组，可以将多个不连续的内存块（如 HTTP 响应头和文件内容）组合成一个逻辑连续的流，**减少系统调用次数**，提升 I/O 性能。

#### 示例场景

HTTP 响应发送

- `iov[0]`：存储 HTTP 头部（如 `"HTTP/1.1 200 OK\r\nContent-Length: 1000\r\n\r\n"`）。
- `iov[1]`：指向内存映射的文件内容（如 `mmap` 映射的 HTML 文件）。

- 调用 `writev(fd, iov, 2)` 一次性发送头和文件内容，避免两次 `write` 调用。

------

####  **关键系统调用**

- **`writev(int fd, const struct iovec *iov, int iovcnt)`**
   将 `iov` 数组中的多个缓冲区按顺序写入文件描述符 `fd`。
- **`readv(int fd, const struct iovec *iov, int iovcnt)`**
   从文件描述符 `fd` 读取数据到 `iov` 数组中的多个缓冲区。

#### 总结

`struct iovec` + `writev` 是高性能服务器开发中的“黄金组合”，通过减少系统调用次数和内存拷贝，显著提升 I/O 效率。在你的 HTTP 服务器代码中，它完美实现了响应头和文件内容的高效整合发送，是 Reactor 模式中非阻塞 I/O 的关键优化手段。



### **状态机设计思想**

- **主状态机**：在 `process_read()` 中通过 `m_check_state` 控制解析阶段，对应HTTP协议的请求行、请求头和内容体。
- **从状态机**：通过 `parse_line()` 解析单行数据，确保主状态机处理的数据是完整的行。
- **协作**：主状态机依赖从状态机提供完整的行数据，逐步推进解析流程，直到完成整个HTTP请求的解析

1. **分阶段处理**：
   - 将HTTP请求拆解为**请求行 → 请求头 → 内容体**三个阶段，每个阶段对应一个状态。
   - 每个状态专注处理特定部分，代码结构清晰。
2. **非阻塞处理**：
   - 适应ET/LT模式，当数据不完整时（如`LINE_OPEN`），保存当前状态，等待下次触发读事件继续解析。
3. **错误处理**：
   - 每个解析函数返回错误码（如`BAD_REQUEST`），状态机提前终止并返回错误响应。





## webserver

###  为什么这样实现能够区分模式

代码通过检测全局或成员变量 `m_actormodel` 的值来选择执行不同的逻辑分支，从而实现两种模式的区分：

- **Reactor 模式（m_actormodel == 1）**
  - IO 事件（如读事件）只是触发一个信号，实际的读操作和数据处理被延迟到工作线程中。
  - 使用 **improv** 来等待工作线程完成处理。
  - 主线程主要负责事件分发、定时器调整和等待状态同步。
- **Proactor 模式（m_actormodel != 1）**
  - IO 操作（如读取数据）在事件触发时就已经完成，数据已经被操作系统异步读入。
  - 主线程直接检测读操作结果，成功后将任务提交到线程池处理，失败则直接处理定时器。
  - 这种模式下，异步 IO 的“主动”部分已经完成，不需要等待工作线程设置标志位。

总结来说，通过 `m_actormodel` 的不同值，代码明确区分了两种模式：

- 在 **Reactor 模式** 下，任务的实际执行由工作线程异步完成，主线程需等待状态同步（由 **improv** 控制）。
- 在 **Proactor 模式** 下，IO 数据的读取已经异步完成，主线程只负责分发任务给线程

```cpp
void WebServer::dealwithread(int sockfd)
{
    util_timer *timer = users_timer[sockfd].timer;

    //reactor
    if (1 == m_actormodel)
    {
        if (timer)
        {
            adjust_timer(timer);
        }

        //若监测到读事件，将该事件放入请求队列
        m_pool->append(users + sockfd, 0);

        while (true)
        {
            if (1 == users[sockfd].improv)
            {
                if (1 == users[sockfd].timer_flag)
                {
                    deal_timer(timer, sockfd);
                    users[sockfd].timer_flag = 0;
                }
                users[sockfd].improv = 0;
                break;
            }
        }
    }
    else
    {
        //proactor
        if (users[sockfd].read_once())
        {
            LOG_INFO("deal with the client(%s)", inet_ntoa(users[sockfd].get_address()->sin_addr));

            //若监测到读事件，将该事件放入请求队列
            m_pool->append_p(users + sockfd);

            if (timer)
            {
                adjust_timer(timer);
            }
        }
        else
        {
            deal_timer(timer, sockfd);
        }
    }
}
```

