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

### 4.Web 服务器如何处理和响应接收到的HTTP 请求报文？

1. 用线程池 并发处理 用户请求
2. 主线程负责读写，工作线程(线程池中的线程)负责处理逻辑(HTTP请求报文的解析等等)

过程：

通过之前的代码，我们将listenfd上到达的connection通过 accept()接收，并返回一个新的 socket 文件描述符 connfd 用于和用户通信，并对用户请求返回响应，同时将这个connfd注册到内核事件表中，等用户发来请求报文。

这个过程是：通过 epoll_wait 发现这个 connfd 上有可读事件了（EPOLLIN），主线程就将这个HTTP的请求报文读进这个连接socket的读缓存中 users[sockfd].read()，然后将该任务对象（指针）插入线程池的请求队列中 pool->append(users + sockfd); (当前文档的第二个代码块中存在相关的使用)

其中，线程池的实现还需要依靠 **锁机制以及信号量机制** 来实现**线程同步**，保证操作的原子性。

HTTP请求是如何被处理的？

```c++

void http_conn::process() {
    HTTP_CODE read_ret = process_read();
    if(read_ret == NO_REQUEST) {
        modfd(m_epollfd, m_sockfd, EPOLLIN);
        return;
    }
    bool write_ret = process_write(read_ret);
    if(!write_ret)
        close_conn();
    modfd(m_epollfd, m_sockfd, EPOLLOUT);
}
```

首先，调用 process_read 函数(自己编写的，位于http_conn.cpp)，对读入 connfd 读缓冲区的请求报文进行解析。

process_read 函数的作用就是 将 get 和 post 的报文进行解析 了解用户的请求内容。HTTP请求报文由请求行（request line）、请求头部（header）、空行和请求数据四个部分组成有。get 和 post 都有相对应的报文, Get 发送一个报文 其中包含 header 和 data。 Post 会发送两个报文 先发送header 之后发送data 。发送header 后会收到 100 的回复报文(表示请求已收到)，之后发送data 会后收到200 的回复报文。

使用**主从状态机**进行检测，**从状态机** 读取报文的一行，主状态机负责对该行数据进行解析。主状态机内部调用从状态机，从状态机驱动主状态机。

每解析一部分都会将整个请求的m_check_state状态改变，状态机也就是根据这个状态来进行不同部分的解析跳转的：

- parse_request_line(text)： 解析请求行(通常是请求报文的第一行)，解析这次的请求是get 还是 post。请求行中重要的部分是URL，将 URL 保存下来用于后面的生成HTTP响应。

- parse_headers(text); 解析请求头部，GET和POST中 空行 以上，请求行以下的部分。

- parse_content(text);，解析请求数据，对于GET来说这部分是空的，因为这部分内容已经以明文的方式包含在了请求行中的URL部分了；只有POST的这部分是有数据的，项目中的这部分数据为用户名和密码，我们会根据这部分内容做**登录和校验**，并**涉及到与数据库的连接**。

OK，经过上述解析，当**得到一个完整的，正确的HTTP请求**时，就到了do_request代码部分，我们需要首先对GET请求和不同POST请求（登录，注册，请求图片，视频等等）做不同的预处理，然后分析目标文件的属性，若**目标文件存在**、**对所有用户可读且不是目录**时，则使用**mmap**将其**映射到内存地址 m_file_address** 处，并告诉调用者获取文件成功。

#### 相关例子

假设已经做好了相关的服务器，然后输入localhost:9000, 之后相当于你向服务器 进行了Get 请求。然后服务端解析这个Get 请求，之后返回一个HTML 界面。如果是post 如何进行，judge 界面中 存在新用户和 已有账户两个 button 元素，点击相应的button 就是一个POST 操作。之后检测 action 来判断POST 的操作是什么(这个在html 中展现 button 使用form 实现，里面标志着action 是什么，form 中可以使用 method 表示 post)

### 5. 数据库连接池是如何运行的

在处理用户注册，登录请求的时候，我们需要将这些用户的**用户名**和**密码**保存下来用于新用户的注册及老用户的登录校验，这种功能是服务器端通过用户键入的用户名密码和数据库中已记录下来的用户名密码数据进行校验实现的。

若每次用户请求我们都需要新建一个数据库连接，请求结束后我们释放该数据库连接，当用户请求连接过多时，这种做法过于低效，所以**类似线程池**的做法，我们构建一个数据库连接池，预先生成一些数据库连接放在那里供用户请求使用。

- 使用mysql_init()初始化连接
- 使用mysql_real_connect()建立一个到mysql数据库的连接
- 使用mysql_query()执行查询语句
- 使用result = mysql_store_result(mysql)获取结果集
- 使用mysql_num_fields(result)获取查询的列数，mysql_num_rows(result)获取结果集的行数
- 通过mysql_fetch_row(result)不断获取下一行，然后循环输出
- 使用mysql_free_result(result)释放结果集所占内存
- 使用mysql_close(conn)关闭连接

对于一个数据库连接池来讲，就是预先生成多个这样的数据库连接，然后放在一个链表中，同时维护最大连接数MAX_CONN，当前可用连接数FREE_CONN和当前已用连接数CUR_CONN这三个变量。同样注意在对连接池操作时（获取，释放），要用到锁机制，因为它被所有线程共享。

### 6.CGI 校验

TODO：未完成

当点击新用户按钮时，服务器对这个POST请求的响应是：返回用户一个登录界面；当你在用户名和密码框中输入后，你的POST请求报文中会连同你的用户名密码一起发给服务器，然后我们拿着你的用户名和密码在数据库连接池中取出一个连接用于mysql_query()进行查询，逻辑很简单，**同步线程校验SYNSQL**方式相信大家都能明白，还有另外两种检验方式，**CGI** 和 **???**

#### 同步线程校验 SYNSQL

#### CGI

CGI（通用网关接口），它是一个**运行在Web服务器上的程序**，在编译的时候将**相应的.cpp文件编程成.cgi文件**并在主程序中调用即可（通过社长的makefile文件内容也可以看出）。这些CGI程序通常通过客户在其浏览器上点击一个button时运行。这些程序通常用来**执行一些信息搜索、存储等任务**，而且通常会**生成一个动态的HTML网页来响应客户的HTTP请求**。我们可以发现项目中的sign.cpp文件就是我们的CGI程序，将用户请求中的用户名和密码保存在一个**id_passwd.txt**文件中，通过将数据库中的用户名和密码**存到一个map中用于校验**。在主程序中通过execl(m_real_file, &flag, name, password, NULL);这句命令来执行这个CGI文件，这里**CGI程序仅用于校验，并未直接返回给用户响应**。这个CGI程序的运行通过多进程来实现，根据其返回结果判断校验结果（使用pipe进行父子进程的通信，子进程将校验结果写到pipe的写端，父进程在读端读取）。

### 7. 生成HTTP 响应并返回给用户

对读到的请求进行了处理，目标文件的属性进行分析，若目标文件存在、**对用户可读**且不是目录时，则使用mmap将其映射到内存地址m_file_address处，并告诉调用者获取文件成功FILE_REQUEST。 接下来要做的就是**根据读取结果对用户做出响应了**，也就是到了 **process_write(read_ret);** 这一步，该函数根据process_read()的返回结果来判断应该返回给用户什么响应，我们最常见的就是404错误了，说明客户请求的文件不存在。然后呢，假设用户请求的文件存在，而且已经被mmap到m_file_address这里了，那么我们就将做如下写操作，将响应写到这个connfd的写缓存m_write_buf中去：

```c++
case FILE_REQUEST: {
    add_status_line(200, ok_200_title);
    if(m_file_stat.st_size != 0) {
        add_headers(m_file_stat.st_size);
        m_iv[0].iov_base = m_write_buf;
        m_iv[0].iov_len = m_write_idx;
        m_iv[1].iov_base = m_file_address;
        m_iv[1].iov_len = m_file_stat.st_size;
        m_iv_count = 2;
        bytes_to_send = m_write_idx + m_file_stat.st_size;
        return true;
    }
    else {
        const char* ok_string = "<html><body></body></html>";
        add_headers(strlen(ok_string));
        if(!add_content(ok_string))
            return false;
    }
}
```

TODO：第三点设置为EPOLLOUT 代码中未体现

1. 首先将状态行写入写缓存
2. 响应头也是要写进connfd的写缓存（HTTP类自己定义的，与socket无关）中的
3. 对于请求的文件，我们已经直接将其映射到m_file_address里面，然后将**该connfd文件描述符上修改为EPOLLOUT**（可写） 事件，然后epoll_Wait监测到这一事件后，使用writev来将响应信息和请求文件聚集写到TCP Socket本身定义的发送缓冲区（这个缓冲区大小一般是默认的，但我们也可以通过setsockopt来修改）中，交由内核发送给用户。OVER！

### 8. 服务器优化：定时器处理非活动链接

TODO： 细节未看

项目中，我们预先分配了MAX_FD个http连接对象：

```c++

// 预先为每个可能的客户连接分配一个http_conn对象
http_conn* users = new http_conn[MAX_FD];

```

如果某一用户connect()到服务器之后，长时间不交换数据，一直占用服务器端的文件描述符，导致连接资源的浪费。这时候就应该**利用定时器把这些超时的非活动连接释放掉**，关闭其占用的文件描述符。这种情况也很常见，当你登录一个网站后长时间没有操作该网站的网页，再次访问的时候你会发现需要重新登录。

项目中使用的是 **SIGALRM 信号来实现定时器**，利用**alarm函数周期性的触发SIGALRM信号**，信号处理函数利用**管道通知主循环**，主循环接收到该信号后**对升序链表上所有定时器进行处理**，若该段时间内没有交换数据，则将该连接关闭，释放所占用的资源。（具体请参考Linux高性能服务器编程 第11章 定时器和社长庖丁解牛：07定时器篇），我们接下来看项目中的具体实现。

```c++

/* 定时器相关参数 */
static int pipefd[2];
static sort_timer_lst timer_lst

/* 每个user（http请求）对应的timer */
client_data* user_timer = new client_data[MAX_FD];
/* 每隔TIMESLOT时间触发SIGALRM信号 */
alarm(TIMESLOT);
/* 创建管道，注册pipefd[0]上的可读事件 */
int ret = socketpair(PF_UNIX, SOCK_STREAM, 0, pipefd);
/* 设置管道写端为非阻塞 */
setnonblocking(pipefd[1]);
/* 设置管道读端为ET非阻塞，并添加到epoll内核事件表 */
addfd(epollfd, pipefd[0], false);

addsig(SIGALRM, sig_handler, false);
addsig(SIGTERM, sig_handler, false);

```

alarm函数会定期触发SIGALRM信号，这个信号交由sig_handler来处理，每当监测到有这个信号的时候，都会将这个信号写到pipefd[1]里面，传递给主循环：

```c++

// 主循环：
/* 处理信号 */
else if(sockfd == pipefd[0] && (events[i].events & EPOLLIN)) {
    int sig;
    char signals[1024];
    ret = recv(pipefd[0], signals, sizeof(signals), 0);
    if(ret == -1) {
        continue;  // handle the error
    }
    else if(ret == 0) {
        continue;
    }
    else {
        for(int j = 0; j < ret; ++j) {
            switch (signals[j]) {
                case SIGALRM: {
                    timeout = true;
                    break;
                }
                case SIGTERM: {
                    stop_server = true;
                }
            }
        }
    }
}

```

当我们在读端pipefd[0]读到这个信号的的时候，就会将timeout变量置为true并跳出循环，让timer_handler()函数取出来定时器容器上的到期任务，该定时器容器是通过升序链表来实现的，从头到尾对检查任务是否超时，若超时则调用定时器的回调函数cb_func()，关闭该socket连接，并删除其对应的定时器del_timer。

#### 计时器的优化

这个基于升序双向链表实现的定时器存在着其固有缺点：

- 每次遍历添加和修改定时器的效率偏低(O(n))，使用最小堆结构可以降低时间复杂度降至(O(logn))。
- 每次以固定的时间间隔触发SIGALRM信号，调用tick函数处理超时连接会造成一定的触发浪费，举个例子，若当前的TIMESLOT=5，即每隔5ms触发一次SIGALRM，跳出循环执行tick函数，这时如果当前即将超时的任务距离现在还有20ms，那么在这个期间，SIGALRM信号被触发了4次，tick函数也被执行了4次，可是在这4次中，前三次触发都是无意义的。对此，我们可以动态的设置TIMESLOT的值，每次将其值设置为当前最先超时的定时器与当前时间的时间差，这样每次调用tick函数，超时时间最小的定时器必然到期，并被处理，然后在从时间堆中取一个最先超时的定时器的时间与当前时间做时间差，更新TIMESLOT的值。

### 9. 服务器优化：日志

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

### epoll / select / poll

#### epoll 要快于 select

TODO：未完全了解为什么epoll 要快于 select

假设有200 个 连接数。 select 需要将进程添加到等待队列中，每个连接一个，等待队列总共为200个。之后需要 thunk 将进程附加到等待列表中，每个连接一个。(thunk 主要表示为延迟计算的执行，以节约资源，只在必要时进行计算)

该进程只需使用一个 thunk 就可以被放入一个等待列表中。当进程唤醒时，只需将其从一个等待列表中删除，并且只需释放一个 thunk。

#### 复制问题

对于select和poll来说，所有文件描述符都是在用户态被加入其文件描述符集合的，**每次调用都需要将整个集合拷贝到内核态**；epoll则将整个文件描述符集合维护在内核态，**每次添加文件描述符的时候都需要执行一个系统调用**。系统调用的开销是很大的，而且在有很多短期活跃连接的情况下，epoll可能会慢于select和poll由于这些大量的系统调用开销。

#### 数据结构

select使用线性表描述文件描述符集合，文件描述符有上限；poll使用链表来描述；

epoll底层通过红黑树来描述，并且维护一个ready list，将事件表中已经就绪的事件添加到这里，在使用epoll_wait调用时，仅观察这个list中有没有数据即可。

#### 寻找是否有就绪文件描述符的方式

select和poll的最大开销来自内核判断是否有文件描述符就绪这一过程：每次执行select或poll调用时，它们会采用遍历的方式，遍历整个文件描述符集合去判断各个文件描述符是否有活动；epoll则不需要去以这种方式检查，当有活动产生时，会自动触发epoll回调函数通知epoll文件描述符，然后内核将这些就绪的文件描述符放到之前提到的ready list中等待epoll_wait调用后被处理。

#### 支持的模式

select和poll都只能工作在相对低效的LT模式下，而epoll同时支持LT和ET模式。

#### 总结

当监测的fd数量较小，且各个fd都很活跃的情况下，建议使用select和poll；当监听的fd数量较多，且单位时间仅部分fd活跃的情况下，使用epoll会明显提升性能。

### epoll 对文件操作符的两种模式

#### LT

LT（电平触发）：类似select，LT会去遍历在epoll事件表中每个文件描述符，来观察是否有我们感兴趣的事件发生，如果有（触发了该文件描述符上的回调函数），epoll_wait就会以非阻塞的方式返回。

若该epoll事件没有被处理完（没有返回EWOULDBLOCK），该事件还会被后续的epoll_wait再次触发。

#### ET

ET（边缘触发）：ET在发现有我们感兴趣的事件发生后，立即返回，并且sleep这一事件的epoll_wait，不管该事件有没有结束。

在使用ET模式时，必须要保证该文件描述符是非阻塞的（确保在没有数据可读时，该文件描述符不会一直阻塞）；并且每次调用read和write的时候都必须等到它们返回EWOULDBLOCK（确保所有数据都已读完或写完）。

### 线程池

所谓线程池，就是一个 pthread_t 类型的普通数组，通过 pthread_create() 函数创建 m_thread_number 个线程，通过 worker() 函数以执行每个请求处理函数（HTTP请求的process函数），通过 pthread_detach() 将线程设置成脱离态（detached）后，当这一线程运行结束时，它的资源会被系统自动回收，而不再需要在其它线程中对其进行 pthread_join() 操作。

操作工作队列一定要加锁（locker），因为它被所有线程共享。

我们用信号量来标识请求队列中的请求数，通过 m_queuestat.wait(); 来等待一个请求队列中待处理的HTTP请求，然后交给线程池中的空闲线程来处理。

#### 为什么要使用线程池？

当你需要限制你应用程序中同时运行的线程数时，线程池非常有用。因为启动一个新线程会带来性能开销，每个线程也会为其堆栈分配一些内存等。为了任务的并发执行，我们可以将这些任务任务传递到线程池，而不是为每个任务动态开启一个新的线程。

#### 线程池中的线程数量是依据什么确定的？

在StackOverflow上面发现了一个还不错的回答，意思是：

线程池中的线程数量最直接的限制因素是中央处理器(CPU)的处理器(processors/cores)的数量N

如果你的CPU是4-cores的

对于CPU密集型的任务(如视频剪辑等消耗CPU计算资源的任务)来说，那线程池中的线程数量最好也设置为 4（或者+1防止其他因素造成的线程阻塞）；

对于IO密集型的任务，一般要多于CPU的核数，因为**线程间竞争的不是CPU的计算资源而是IO**，IO的处理一般较慢，多于cores数的线程将为CPU争取更多的任务，以防 线程处理IO的过程造成CPU空闲导致资源浪费，公式：最佳线程数 = CPU当前可使用的Cores数 \* 当前CPU的利用率 \* (1 + CPU等待时间 / CPU处理时间)（还有回答里面提到的Amdahl准则可以了解一下）

### RAII 是什么？

RAII全称是“Resource Acquisition is Initialization”，直译过来是“资源获取即初始化”.

在构造函数中申请分配资源，在析构函数中释放资源。因为C++的语言机制保证了，当一个对象创建的时候，自动调用构造函数，当对象超出作用域的时候会自动调用析构函数。所以，在RAII的指导下，我们应该使用类来管理资源，将资源和对象的生命周期绑定