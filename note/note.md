# note

## 介绍

线程池 + 非阻塞 socket + epoll + 事件处理 并发模型

状态机解析HTTP 请求报文，解析 GET 和 POST 请求。

访问服务器数据库实现web端用户 注册、登录功能，请求服务器图片和视频文件

实现同步/异步 日志系统，记录服务器运行状态

webbech 压力测试 实现 上万的并发连接数据交换。

## 准备

unix 环境高级编程

unix 网络编程

## 内容细节

### 3.web 服务器如何接受客户端发来的HTTP 请求报文

> 用户如何与你的Web服务器进行通信

通常用户使用**Web浏览器**与**相应服务器**进行通信。在浏览器中键入“域名”或“IP地址:端口号”，浏览器则先将你的域名解析成相应的IP地址或者直接根据你的IP地址向对应的Web服务器发送一个HTTP请求。这一过程**首先要通过TCP协议的三次握手建立与目标Web服务器的连接**，然后HTTP协议生成针对目标Web服务器的HTTP请求报文，通过TCP、IP等协议发送到目标Web服务器上。

> Web服务器如何接收客户端发来的HTTP请求报文呢?

Web服务器端通过socket监听来自用户的请求。

远端的很多用户会尝试去connect()这个Web Server上正在listen的这个port，而监听到的这些连接会排队等待被accept()。由于用户连接请求是随机到达的异步事件，每当监听socket（listenfd）listen到**新的客户连接并且放入监听队列**，我们都需要告诉我们的Web服务器有连接来了，accept这个连接，并分配一个逻辑单元来处理这个用户请求。

```c++
#include <sys/socket.h>
#include <netinet/in.h>
/* 创建 监听socket 文件描述符 */
int listenfd = socket(PF_INET, SOCK_STREAM, 0);
/* 创建监听socket的TCP/IP的IPV4 socket地址 */
struct sockaddr_in address;
bzero(&address, sizeof(address));
address.sin_family = AF_INET;
address.sin_addr.s_addr = htonl(INADDR_ANY);  /* INADDR_ANY：将套接字绑定到所有可用的接口 htonl:host to network long 将本地字节序的 32 位整数转换为网络字节序的 32 位整数*/
address.sin_port = htons(port); // port 是对应的端口号 htons：host to network short (16位)

int flag = 1;
/* SO_REUSEADDR 允许端口被重复使用 */
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
/* 绑定socket和它的地址 */
ret = bind(listenfd, (struct sockaddr*)&address, sizeof(address));  
/* 创建监听队列以存放待处理的客户连接，在这些客户连接被accept()之前 */
ret = listen(listenfd, 5);
```

而且，我们在处理这个请求的同时，还需要继续监听其他客户的请求并分配其另一逻辑单元来处理（并发，同时处理多个事件，后面会提到使用**线程池实现并发**）。这里，服务器通过**epoll这种I/O复用技术**（还有select和poll）来实现对监听socket（listenfd）和连接socket（客户请求）的同时监听。注意I/O复用虽然可以同时监听多个文件描述符，但是它**本身是阻塞**的，并且当有多个文件描述符同时就绪的时候，如果不采取额外措施，程序则只能按顺序处理其中就绪的每一个文件描述符，所以为提高效率，我们将在这部分通过线程池来实现并发（多线程并发），为每个就绪的文件描述符分配一个逻辑单元（线程）来处理。

EPOLL 中放入的是 需要监视的 socket 文件描述符

```c++
#include <sys/epoll.h>
/* 将fd上的EPOLLIN和EPOLLET事件注册到epollfd指示的epoll内核事件中

EPOLLIN : 表示你希望监视文件描述符上的可读事件
EPOLLET : 表示你希望使用边缘触发（edge-triggered）方式来监视文件描述符
EPOLLLT ： 希望使用水平触发
EPOLLOUT : 监视文件描述符上的可写事件
EPOLLERR : 表示你希望监视文件描述符上的错误事件
EPOLLHUP : 表示你希望监视文件描述符上的挂起事件，由 本地端进行发起 的关闭事件
EPOLLPRI : 表示你希望监视文件描述符上的高优先级事件
EPOLLONESHOT : 表示你希望使用**一次性事件模式**来监视文件描述符；原理：该文件描述符只会在第一次触发事件后从 epoll 队列中删除。每次事件触发后，你需要重新将该文件描述符添加到 epoll 实例中以监视下一次事件。
EPOLLEXCLUSIVE : 表示你希望以**独占模式**监视文件描述符
EPOLLRDHUP : 本地端 检测 对端 断开连接
*/
// 这个函数位于 自己编写的 lst_timer.cpp内 
void addfd(int epollfd, int fd, bool one_shot) {
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
    /* 针对connfd，开启EPOLLONESHOT，因为我们希望每个socket在任意时刻都只被一个线程处理 */
    if(one_shot)
        event.events |= EPOLLONESHOT;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);/* epoll_ctl 函数功能： 添加文件描述符到 epoll 实例 
    第一个参数 epoll 实例的文件描述符
    第二个参数 操作的类型op 是操作类型，可以是以下之一：
        EPOLL_CTL_ADD：将文件描述符添加到 epoll 实例中。
        EPOLL_CTL_MOD：修改文件描述符的监视选项。
        EPOLL_CTL_DEL：从 epoll 实例中删除文件描述符。
    第三个参数 添加、修改或删除的文件描述符
    第四个参数指定要监视的事件类型和其他选项
    */
    setnonblocking(fd);
}
/* 创建一个额外的文件描述符来唯一标识内核中的epoll事件表 （创建了一个epoll 实例） 参数表示的是 size （表示容纳文件描述符的数目）。 返回的是文件描述符，内核2.6.8开始，size 会被忽略，大于0即可*/
int epollfd = epoll_create(5);  
/* 用于存储epoll事件表中就绪事件的event数组，即存储检测到有相关事件的socket 连接*/
/*
struct epoll_event 的原型 : 
*/
epoll_event events[MAX_EVENT_NUMBER];
/* 主线程往epoll内核事件表中注册 监听socket 事件，当listen到新的客户连接时，listenfd变为就绪事件 */
addfd(epollfd, listenfd, false);  
/* 主线程调用epoll_wait等待一组文件描述符上的事件，并将当前所有就绪的epoll_event复制到events数组中 */
/* epoll_wait 函数原型:
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
edfd : epoll 实例的文件描述符
events : struct epoll_event 结构的数组指针，用于存储 event 的信息;
maxevents : event 数组的大小
timeout : 超时值，ms 为单位。如果设置为负数，epoll_wait 将会一直等待事件发生，直到有事件发生为止。如果设置为0，如果没有事件发生，epoll_wait 将会立即返回。如果设置为正数，epoll_wait 将等待指定的时间，然后返回，不管是否有事件发生。
返回值 : 表示发生事件的数量，如果没有事件发生并且超时时间到达，返回值为0，如果发生错误，返回值为-1。
*/
int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
/* 然后我们遍历这一数组以处理这些已经就绪的事件 */
/* event 的 类型为 epoll_events，epoll_events 的原型为
struct epoll_event
{
  uint32_t events;  // Epoll events 
  epoll_data_t data;  // User data variable  这个是 union 结构，union 结构只会存储当前的一个值而不是多个值，这些值之间是共享内存的 。
}
*/

for(int i = 0; i < number; ++i) {
    int sockfd = events[i].data.fd;  // 事件表中就绪的socket文件描述符
    if(sockfd == listenfd) {  // 当listen到新的用户连接，listenfd上则产生就绪事件
        struct sockaddr_in client_address;
        socklen_t client_addrlength = sizeof(client_address);
        /* ET模式 就是边缘触发,需要一直accept 直到空为止*/
        while(1) {
            /* accept()返回一个新的socket文件描述符用于send()和recv() */
            int connfd = accept(listenfd, (struct sockaddr *) &client_address, &client_addrlength);
            /* 并将connfd注册到内核事件表中 */
            users[connfd].init(connfd, client_address);
            /* ... */
        }
    }
    else if(events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)) {
        // 如有异常，则直接关闭客户连接，并删除该用户的timer
        /* ... */
    }
    else if(events[i].events & EPOLLIN) {
        /* 当这一sockfd上有可读事件时，epoll_wait通知主线程。*/
        if(users[sockfd].read()) { /* 主线程从这一sockfd循环读取数据, 直到没有更多数据可读 */
            pool->append(users + sockfd);  /* 然后将读取到的数据封装成一个请求对象并插入请求队列 */
            /* ... */
        }
        else
            /* ... */
    }
    else if(events[i].events & EPOLLOUT) {
        /* 当这一sockfd上有可写事件时，epoll_wait通知主线程。主线程往socket上写入服务器处理客户请求的结果 */
        if(users[sockfd].write()) {
            /* ... */
        }
        else
            /* ... */
    }
}
```

服务器程序通常需要处理三类事件：**I/O事件**，**信号**及**定时事件**。有两种事件处理模式：

Reactor模式：要求主线程（I/O处理单元）只负责监听文件描述符上是否有事件发生（可读、可写），若有，则立即通知工作线程（逻辑单元），将socket可读/可写事件放入请求队列，交给工作线程处理。

Proactor模式：将所有的I/O操作都交给主线程和内核来处理（进行读、写），**工作线程仅负责处理逻辑**，如主线程读完成后users[sockfd].read()，选择一个工作线程来处理客户请求pool->append(users + sockfd)。

通常使用同步I/O模型（如epoll_wait）实现Reactor。

使用异步I/O（如aio_read和aio_write）实现Proactor。（异步IO,指发起IO之后立刻返回继续进行程序接下来的操作，「内核数据准备好」和「数据从内核态拷贝到用户态」这两个过程由内核自动完成的，和前面的同步操作不一样，应用程序并不需要主动发起拷贝动作）

Proactor 模式：1. 创建 handler 2. 创建Proactor 3. 注册handler 和 proactor 进入到 内核的Asynchronous Operation Processor 中 4. Asynchronous Operation Processor 完成 I/O 操作后通知 Proactor 5. 回调不同的handler 进行业务处理 6. handler 完成事务

Reactor 可以理解为「来了事件操作系统通知应用进程，让应用进程来处理」，而 Proactor 可以理解为「来了事件操作系统来处理，处理完再通知应用进程」。

## 其他相关细节

### socket 函数

socket 函数 位于库 <sys/socket.h> 中 socket 的原型为 socket(int domain, int type, int protocol)

domain 表示套接字的协议域或地址族。

type 表示套接字的类型

- SOCK_STREAM：流套接字，用于可靠的、面向连接的TCP通信。
- SOCK_DGRAM：数据报套接字，用于不可靠的、无连接的UDP通信。
- SOCK_RAW：原始套接字，用于直接访问底层网络协议，通常需要特权。

socket 创建套接字，bind 绑定ip ， listen 监听， accept 接受连接

protocol :参数0 即为 自动选择

### socket linger

socket linger 是一个 控制 套接字关闭 的选项 包括两个成员：

- l_onoff: 0/1; 设置为1 时，开启套接字延迟关闭。设置为0时，关闭。开启了套接字延迟关闭，使用linger 成员配置延迟关闭的行为。
- l_linger: 表示延迟关闭的时间（以秒为单位）。如果 l_onoff 设置为1，并且 l_linger 设置为一个大于0的值，那么在套接字关闭时，系统会等待一段时间，直到发送或接收缓冲区中的数据被发送完毕或等待时间到期，然后再真正关闭套接字。

要作用是确保在关闭套接字之前，尽可能多的数据能够被发送或接收。

|l_onoff|l_linger|close socket 行为| 发送队列| 底层行为|
|----|----|----|----|----|
|0|非0|直接关闭|？（丢弃还是保留）|？|
|1|0|直接关闭|立即放弃|直接使用RST 进行关闭socket，不经过2MSL 时间|
|1|>=1|阻塞知道时间超时或者数据发送完成|超时时间内持续发送，超时后立即放弃|超时同情况2，否则发送成功|

### socket close 和 shutdown

### socket SO_REUSEADDR 参数

当你启用 SO_REUSEADDR 选项时，它允许你的套接字绑定到一个本地地址（IP地址和端口），即使在该地址上已经存在一个处于 TIME_WAIT 状态的套接字。通常情况下，如果尝试绑定到一个已经被占用的地址，操作系统会拒绝这个绑定请求。但启用 SO_REUSEADDR 选项后，即使之前的套接字仍处于 TIME_WAIT 状态，你的套接字也可以成功地绑定到相同的地址。

表示允许 重用 本地地址。即使在该地址上已经存在一个处于 TIME_WAIT 状态的套接字。通常情况下，如果尝试绑定到一个已经被占用的地址，操作系统会拒绝这个绑定请求。但启用 SO_REUSEADDR 选项后，即使之前的套接字仍处于 TIME_WAIT 状态，你的套接字也可以成功地绑定到相同的地址。

### 为什么选择 epoll 而不是 select 和 poll

