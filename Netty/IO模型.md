# I/O 模型

## 阻塞与非阻塞、同步与异步
* 阻塞与非阻塞
  * 菜没好，要不要死等-> 数据就绪前要不要等待
  * 阻塞：没有数据传过来时，读会阻塞直到有数据；缓冲区满时，写操场也会阻塞，非阻塞遇到这些情况，都是直接返回

* 同步与异步
  * 菜好了，谁端-> 数据就绪后，数据操作谁完成
  * 数据就绪后需要自己去读是同步，数据就绪直接读好再回调给程序是异步

## IO模型
Unix系统有5种IO模型：
* 阻塞式I/O
* 非阻塞式I/O
* I/O复用(select 和poll)
* 信号驱动式I/O(SIGIO)
* 异步I/O(POSIX的aio_系列函数)

一个输入操作通常包括2个不同的阶段：  
* 等待数据准备好
* 从内核向进程复制数据

对于一个套接字来说：第一步涉及等待数据从网络中到达，当所等待分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核缓冲区复制到相应进程缓冲区。  
## 阻塞式I/O模型

![](./img/IO1.png)
默认情况下所有套接字都是阻塞的，把recvfrom函数视为系统调用，区分应用进程和内核；进程调用recvfrom，其系统调用直到数据报到达且被复制到应用进程的缓冲区中或者发生错误才返回。  

## 非阻塞式I/O

进程把一个套接字设置成非阻塞是在通知内核：当所有请求的I/O操作非得把本进程投入睡眠才能完成时，不要把本进程投入睡眠，而是返回一个错误。 
![](./img/IO2.png)
前三次调用recvfrom时没有数据返回，因此内核转而立即返回一个EWOULDBLOCK错误，第四次调用recvfrom时已有一个数据报准备好，它被复制到应用进程缓冲区，于是recvfrom成功返回。当一个应用进程对非阻塞描述符循环调用recvfrom时，我们称之为轮询(polling)，应用进程程持续轮询内核，查看某个操作是否就绪。耗费大量CPU时间。 
## I/O复用模型

可以调用select或poll，阻塞在这两个系统调用中的一个之上，而不是阻塞在真正的系统调用之上。
![](./img/IO3.png)

## 信号驱动式I/O 模型
可以用信号，内核在描述符就绪时发送SIGIO信号通知我们， 
![](img/IO4.png)
这种优势在于等待数据报到达期间进程不被阻塞，

## 异步I/O模型
工作机制是：告知内核启动某个操作，并让内核在整个操作完成后(包括将数据从内核复制到我们自己的缓冲区)完成后通知我们。与信号驱动模型的主要区别在于：信号驱动式I/O由内核通知我们何时可以启动一个I/O操作，而异步I/O模型是由内核通知我们I/O操作何时完成。
![](img/IO5.png)

## 各种I/O模型比较
![](img/IO6.png)
调用blocking和non-blocking的区别：  
调用blockingIO会一直block对应的进程直到操作完成，而non-blocking在kernel准备数据的情况下立刻返回。  
前四种模型都是同步I/O模型，真正操作I/O操作recvfrom将阻塞进程；同步、异步的说法是对于数据获取的过程而言的，前面几种最后获取数据的read操作调用，都是同步的，在read调用时，内核将数据从内核空间拷贝到应用程空间，这个过程是在read函数中同步进行的，如果内核实现的拷贝效率差，read调用就会在同步过程中消耗比较长的时间。  
而异步操作发起read之后，内核自动将数据从内核空间拷贝到应用程序空间，拷贝是异步的，内核自动完成，

## I/O多路复用之select、poll、epoll详解
 本质上select、poll、epoll都是同步IO，他们都需要在读写事件就绪后自己负责进行读写，就是说这个读写过程是阻塞的。  
### select
```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```
select函数监视的文件描述符分3类，分别是writefds、readfds、exceptfds.调用后select函数会阻塞，直到有描述符就绪(有数据可读、可写、或者except)，或者超时，函数返回，。当函数返回后，可以通过遍历fdset，来找到就绪的描述符。  
select几乎所有平台都支持，良好的跨平台。缺点是：单个进程能够监视的文件描述符的数量存在最大限制，linux上一般是1024,可以通过修改宏定义甚至重新编译内核。

### poll
```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```
与select使用三个位图表示fdset不同，poll使用一个pollfd的指针实现。  
```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```
pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值"传递的方式，同时pollfd没有最大数量限制，但是数量过大后，性能会下降。和select函数一样，poll返回需要轮询pollfd来获取就绪的描述符。  
> select和poll都需要在返回后，通过遍历文件描述符来获取已就绪的socket。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降


### epoll
epoll是内核2.6中提出的，是之前select、epoll增强版，epoll更加灵活，没有描述符限制。<strong>epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这个事件基于红黑树实现的，大量IO请求下，插入和删除性能都要比select/poll的fs_set数组要好，</strong>这样用户控件和内核空间的copy只需一次。  
epoll操作过程需要三个接口：
```
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
* int epoll_create(int size)  
  创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听fd+1的值，<font color="red">参数size并不限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议</font>。创建好epoll句柄后，就会占用一个fd值，linux下查看/proc/进程id/fd/,能够看到fd值，所以在使用完epoll后，必须调用close()关闭,否则可能导致fd被耗尽。  

* int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)

函数是对指定描述符fd执行op操作。
> epfd：是epoll_create()的返回值。  
> op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。  
> fd：是需要监听的fd（文件描述符）  
> epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：
```
truct epoll_event {
  __uint32_t events;  /* Epoll events */
  epoll_data_t data;  /* User data variable */
};

//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```
* int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout); 等待epfd上的IO事件，最多返回maxevents个事件。  参数events用来从内核中得到事件的集合，maxevents告知内核这个events有多大，这个maxevents的值不能大于创建epoll_create的size。参数timeout是超时时间(毫秒，0立即返回-1将不确定，)；返回需要处理的事件数目，如返回0表示已经超时。  

在select/poll中，进程只有调用一定的方法后，内核才对所有监视的文件描述符进行扫描，<strong>而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait()时便得到通知。</strong>  
epoll优点：  
* 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件数目，具体数目可以cat /proc/sys/fs/file-max查看，一般来说，这个数目和系统内存关系很大。select的最大缺点就是进程打开fd是数量有限制的。
* IO的效率不会随着监视文件的数量增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现。

> 如果没有大量的idle-connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是遇到大量idle-connection,就会发现epoll效率大大高于select/poll.


## 其他
### 文件描述符
文件描述符(File descriptor)是一个用于表述指向文件的引用的抽象化概念。   
文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符；文件描述符一概念往往适用于UNIx、Linux这样的操作系统。 

### Windows下的IOCP和Proactor模式

和Linux不同，Windows下实现了一套完整的支持套接字的异步编程接口，这套接口一般被叫做IOCompletetionPort(IOCP).  
和reactor模式一样，Proactor模式也存在一个无限循环运行的eventloop线程，但是不同与Reactor模式，<strong>这个线程并不负责处理I/O调用，它只负责对应的read、write操作完成的情况下，分发完成事件到不同的处理函数</strong>。  
这里HTTP举例：  
* 客户端发起一个GET请求
* 这个GET请求对应的字节流被内核读取完成，内核将这个完成事件放置到一个队列中
* event loop线程，也就是proactor这个队列获取事件，根据事件类型，分发到不同的处理函数上，比如一个http handle的onMessage解析函数；
* Http request 解析函数完成报文解析
* 业务逻辑处理，比如读取数据库的记录
* 业务逻辑处理完成，开始encode，完成之后，发起一个异步写操作
* 这个异步写操作被内核执行，完成之后这个异步操作被放置到内核的队列中
* Proactor线程获取这个完成事件，分发到Http Handler 的onWriteCompled方法执行。  


Proactor不会再像reactor一样，每次感知事件后再调用read、write方法完成数据的读写，只负责感知事件完成，并有对应的Handler发起异步读写请求，I/O操作本身是由系统内核完成的。需要传入数据缓冲区的地址信息，系统内核才可以自动帮我们把数据的读写工程做完成。  
无论是reactor还是Proactor模式，<font color="red">都是一种基于事件分发的网络编程模式</font>；Reactor是基于待完成的I/O事件，而Proactor模式则是基于已完成的I/O事件。

## 零拷贝
在I/O复用模型中，执行读写I/O操作依然是阻塞的，执行读写I/O操作时，存在多次内存拷贝和上下文切换，给系统增加了性能开销。  
零拷贝是一种避免多次内存复制的技术，用来优化I/O操作。  
网络编程中，read、write来完成一次I/O读写操作，每一次I/O操作都需要完成四次内存拷贝，路径是I/O设备->内核空间->用户空间->内核空间->其他I/O设备。 
<font color="red">Linux中的mmap函数可以替代read、write的I/O读写操作，实现用户空间和内核空间共享一个缓存数据。</font>mmap函数将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址，不管是用户空间还是内核都是虚拟地址，最终都要通过地址映射到物理内存地址。这种方式避免了内核空间与用户空间的数据交换。I/O复用中的epoll函数使用mmap减少了内存拷贝。 

在javaNIO编程中，则是使用到了Direct Buffer来实现零拷贝。java在JVM内存空间之外开辟了一个物理内存空间，这样内核进程和用户进程都能共享一份缓存数据。 

## 线程模型优化
NIO在用户层做了优化升级，NIO是基于事件驱动模型来实现I/O操作。Reactor模型是同步I/O事件处理的一种常见模型，其核心思想是将I/O注册到多路复用器上，一旦有I/O事件触发,多路复用就将事件分发到事件处理器中，执行就绪的I/O事件操作。三个组件：
* 事件接收器Acceptor：主要负责接收请求连接
* 事件分离器Reactor：接收请求后，会将建立的连接注册到分离器中，依赖与循环监听多路复用器selector，一旦监听到事件，就会将事件dispatch到事件处理器。
* 事件处理器Handler：事件处理器主要是完成相关的事件处理，比如读写I/O操作

### 单线程的reactor线程模型
NIO是基于单线程实现，所有I/O操作都会在NIO线程上完成，由于NIO是非阻塞的IO，理论上是可以完成所有的IO操作
NIO不能真正地实现非阻塞IO操作，因为读写IO操作时用户进还是处于阻塞状态，这种高负载、高并发情况下，存在性能瓶颈。
![](img/re-1.png)

### 多线程Reactor线程模型
为了解决搞负载，性能瓶颈，使用线程池
Tomcat和Netty中都是用一个Acceptor线程来监听连接请求时间，当连接成功后将建立的连接注册到多路复用器中，一旦监听到事件，就交个worker线程池负责处理。大多数情况下能满足性能需求，一个Acceptor线程会存在瓶颈，如果大量客户端请求。 
![](img/re-2.png)

### 主从Reactor线程模型

现在主流通信框架中的 NIO 通信框架都是基于主从 Reactor 线程模型来实现的。在这个模型中，Acceptor 不再是一个单独的 NIO 线程，而是一个线程池。Acceptor 接收到客户端的 TCP 连接请求，建立连接之后，后续的 I/O 操作将交给 Worker I/O 线程。
![](img/re-3.png)