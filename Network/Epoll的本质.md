### 1.技术概述

epoll作为linux下高性能网络服务器开发的必备技术，nginx、redis等大部分服务器都是用到了这一多路复用技术。本文将会从网卡接收到数据的流程说起，串联CPU中断，操作系统进程调度等知识，分析阻塞接受数据，select到epoll的变化过程

### 2.网卡接收数据

首先查看计算机结构图，从硬件了解计算机如何接收数据

![流程图](http://p9.pstatp.com/large/pgc-image/022d17643ae64ea6b0ec09baaac3751c)

- 网卡接收到传来的数据，数据来源可以通过网线等

- 通过硬件电路的传输

- 将数据写到内存的某个地址

这个过程涉及到了DMA传输、IO通路选择，但是核心就是网卡会把数据写入内存

### 3.如何知道接收了数据

程序执行的时候，会有优先级的需求。一般而言，由硬件产生的信号需要CPU立马做出回应，所以这类程序的优先级就很高。CPU理应中断正在执行的程序，去做出相应，当CPU完成硬件相应后，再重新执行用户程序

![中断流程](http://p1.pstatp.com/large/pgc-image/80f6a55288af44439825c83c5e4a4ae0)

因此，当网卡把数据写入内存后，网卡向CPU发出一个中断信号，操作系统就知道有新的数据进来，再通过网卡中断程序去处理数据


### 4.进程阻塞为什么不占用CPU

这一步从操作系统角度来看接收数据。阻塞是进程调度的关键一环，指的是进程在等待某事件(如接收到网络数据)发生之前的等待状态，recv、select和epoll都是阻塞方法，这里select和epoll调用的时候，如果没有连接进入，就会进入等待队列，不会浪费CPU，而有连接的时候，会发出中断程序，然后通知到CPU进行调用该进程

    //创建socket
    int s = socket(AP_INET, SOCKET_STREAM, 0)
    //绑定端口
    bind(s, ...)
    listen(s, ...)
    //接收客户端连接
    int c = accpet(s, ...)
    //接收客户端数据
    recv(c, ...)

这段基础的网络编程代码，最后调用的recv接收数据方法就是一个阻塞方法，每次当程序运行到这就会一直等待下去，知道收到数据才会往下执行

#### 工作队列

操作系统为了支持多任务，实现进程调度的功能，会把进程分为“运行”和“等待”几种状态。运行状态是获取到CPU的使用权，等待状态是阻塞状态。如上面说到的recv方法，程序会从运行状态转为等待状态，接收到数据后又转为运行状态，操作系统分时执行每个运行状态的进程，由于速度比较快，看起来像是同时工作

下图是计算机运行的A、B、C三个进程，其中A执行着上面的网络程序，一开始这三个进程在工作队类中，处于运行状态会分时执行

![image](http://p3.pstatp.com/large/pgc-image/d61c7d54df2748109f15613db6575ce4)

#### 等待队列

当进程A指定到创建socket语句。操作系统会创建一个由文件系统管理的socket对象。这个socket对象包括了发送缓存区、接收缓存区、等待队列等。等待队列是非常重要的结构，它指向了要等待该socket事件的进程

![image](http://p1.pstatp.com/large/pgc-image/edc8a2ae7be0448e98420503620e3182)

当程序执行到recv时，操作系统会把A从工作队列移动到socket的等待队列。由于工作队列中只有B、C。所以进程A会被阻塞，不会执行后面的代码，因此也不会占用CPU资源

![image](http://p3.pstatp.com/large/pgc-image/594b281e0de143d981eb0b76a8475870)

ps：操作系统添加等待队列只是添加了对这个“等待中”进程的引用，以便在接收到数据时获取进程对象、将其唤醒，而非直接将进程管理纳入自己之下。上图为了方便说明，直接将进程挂到等待队列之下。

#### 唤醒进程

当socket接收到数据后，操作系统会将该socket的等待队列的进程重新放回到执行队列中，该进程就会成为运行状态，继续至此那个。而且socket接收缓存区中有数据，recv就可以接收到数据了执行后续的代码

### 5.内核接收网络数据的全过程

这一步贯穿了网卡，中断，进程调度等知识，叙述了recv下接收数据的全过程

进程在recv阻塞期间，计算机接收到数据，数据经由网卡传输到内存，然后网卡中断信号通知cpu数据的到达，cpu执行中断程序。中断程序有两个功能，首先是把接收到的数据写入到Socket缓存区中，在唤醒进程A，重新把A放入工作队列

![image](http://p1.pstatp.com/large/pgc-image/08ea962b22624832af24aefbc5fe405a)

![image](http://p1.pstatp.com/large/pgc-image/9b9828c3348f4ca7ae0080c4d2d72b25)

#### 操作系统如何知道网络数据对应哪个socket

因为socket对应了一个端口号，而网络数据包中包含了ip和端口号等信息，内核通过端口号找到对应的socket。当然操作系统为了提高处理速度，会维护端口号到socket的索引结构，加速读取

### 6.如何同时监听多个socket

服务器端管理了多个客户端连接，而recv只能监视一个socket。在这种冲突下，人们开始寻找监听多个socket的方法，epoll就是监听多个socket的一个实现。

首先是select的方式，即建立多个socket，维护一个socket列表，如果socket列表都没有数据，挂起进程，这里的select会导致挂起，直到有一个socket接收到数据然后才唤醒进程

    int s = socket(AF_INET, SOCK_STREAM, 0)
    bind(s, ...)
    listen(s, ...)
    int fds[] = 需要监听的socket
    while(true) {
        int n = select(..., fds, ...)
        for(int i = 0; i <= n; i ++) {
            if (FD_ISSET(fds[i], ...)) {
                // fds[i]的数据处理
            }
        }
    }

select的思路很简单，假如通知监听sock1, sock2, sock3，则流程如下

![image](http://p3.pstatp.com/large/pgc-image/6043034546d5496da06254d13027d19a)

当sock2接收到数据的时候，就会调用中断程序将进程唤醒

![image](http://p9.pstatp.com/large/pgc-image/8e56ee6a4c184d88811a1481103e2937)

![image](http://p1.pstatp.com/large/pgc-image/f027fa8566d1498fa7349438d7cbb840)

经由这些步骤，当A被唤醒后，它至少知道有一个socket接收到了数据，程序只需要遍历一次socket列表获取就绪socket

这种实现方式比较简单，几乎所有系统都有实现。但是区别有如下：

- 每次调用select都需要将进程加入到所监视socket的等待列表，每次唤醒都需要从队列中移除。这里面涉及到两次遍历，而且每次都需要把整个fds列表传递给内核，开销较大，因此规定select的最大监听数量，默认为1024

- 进程被唤醒后，并不知道哪些socket收到了数据，需要再遍历一次

ps：本节只解释了select的一种情形。当程序调用select时，内核会先遍历一遍socket，如果有一个以上的socket接收缓冲区有数据，那么select直接返回，不会阻塞。这也是为什么select的返回值有可能大于1的原因之一。如果没有socket有数据，进程才会阻塞。

### 7.epoll的思路

epoll是select和poll的增强版本，主要使用了以下措施

1.功能分离

select的低效原因之一就是维护“等待队列”和“阻塞进程”两个步骤合二为一，每次调用select都需要调用这两步操作。然而大多数场景下，socket是固定的，不需要每次都修改，epoll将这两个操作分来，先用epoll_ctl维护等待队列，在调用epoll_wait阻塞进程

![image](http://p1.pstatp.com/large/pgc-image/387e1d8065d647e58142340924813f8d)

    int s = socket(AF_INET, SOCK_STREAM, 0)
    bind(s, ...)
    listen(s, ...)
    int epfd = epoll_create(...)
    // 将需要监听的socket添加到epfd中
    epoll_ct(epfd, ...)
    while(true) {
        int n = epoll_wait(...)
        for(接收到的数据的socket) {
            // 进行处理
        }
    }

2.就绪队列

select低效的另一个原因就是程序不知道哪些socket接收到了数据，只能一个一个遍历。如果内核维护一个“就绪队列”，引用的接收到数据的socket，就不需要遍历。

![image](http://p3.pstatp.com/large/pgc-image/274f1b21172c449aa4f74d7adf7281b0)


#### epoll的原理和流程

当某个进程调用epoll_create方法时，内核会创建一个eventpoll对象（也就是程序中epdf）对象。eventpoll对象也就是文件系统的一员，和socket一样，它也会有等待队列

![image](http://p3.pstatp.com/large/pgc-image/b286a8e620584935b50952148beb979c)

创建eventpoll对象后，可以使用epoll_ctl添加或者删除需要监听的socket。

![image](http://p3.pstatp.com/large/pgc-image/55827041ed234556870392671bf829c6)

当socket接收到数据的时候，中断程序会执行eventpoll对象，而不是直接操作进程。接收到数据后，中断程序会给eventpoll对象的就绪队列中添加socket引用

![image](http://p1.pstatp.com/large/pgc-image/ae0520a0dbcd44308eac41a9ef5f68c4)

当程序执行epoll_wait到时候，如果rdList已经引用了socket，那么epoll_wait直接返回，如果为空，则阻塞线程。

当进程运行到了epoll_wait, 内核就会把进程A放入到eventpoll的等待队列

![image](http://p9.pstatp.com/large/pgc-image/8dbfc0f0a8504f0b98f9d55c2a700248)

当socket接收到数据，中断程序一方面修改rdList,一方面唤醒等待的进程，进程会再次进入运行态，也因为rdList的存在，进程A知道了哪些socket发生了改变

![image](http://p1.pstatp.com/large/pgc-image/1bd98d20ef2b4efda2c388463e723103)


### 8.结论


| 系统调用                | select                                                                       | poll                                                           | epoll                                                                                         |
|---------------------|------------------------------------------------------------------------------|----------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| 事件集合                | 用户通过3个参数分别传入感兴趣的可读、可写以及异常等事件，内核通过对这些参数的修改来反馈其中的就绪时间，这使得用户每次调用select都要重置这三个参数 | 统一处理所有的事件类型，因此只需要一个事件集合，用户通过pollfd.events传入感兴趣的事件，内核修改事件返回就绪事件 | 内核通过一个事件表直接管理用户感兴趣的所有事件，因此每次调用epoll_wait时候，不需要重复出传入用户感兴趣的事件，epoll_wait系统调用的参数events仅用来反馈就绪的事件 |
| 应用程序索引就绪文件描述符的事件复杂度 | O(n)                                                                         | O(n)                                                           | O(1)                                                                                          |
| 最大支持文件描述符数量         | 一般有最大值限制1024                                                                 | 65535                                                          | 65535                                                                                         |
| 工作模式                | LT                                                                           | LT                                                             | 支持ET高效模式                                                                                      |
| 内核实现和工作效率           | 采用轮询的方式来检测就绪事件                                                               | 采用轮询的方式来检测就绪事件                                                 | 采用回调方式来检测就绪事件                                                                                 |

