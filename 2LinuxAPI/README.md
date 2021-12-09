第5~7章

## 第5章 Linux网络编程基础API

**socket地址API**

字节序分为大端字节序和小端字节序。大端字节序是指一个整数的高位字节存储在内存的低地址处，低位字节存储在内存的高地址处。小端字节序则是指整数的高位字节存储在内存的高地址处，而低位字节则存储在内存的低地址处。  
现代PC大多采用小端字节序，因此小端字节序又被称为主机字节序。发送端总是把要发送的数据转化为大端字节序数据后再发送，而接收端知道对方传送过来的数据总是采用大端字节序，所以接收端可以根据自身采用的字节序决定是否对接收到的数据进行转换。因此大端字节序也称为网络字节序。

socket网络编程中有通用和专用（不同协议族）的表示地址的结构体，并提供了IP地址转换函数。

**创建socket**

UNIX/Linux的一个哲学是：所有东西都是文件。socket也不例外，它就是可读、可写、可控制、可关闭的文件描述符。

```c
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```
domain告诉系统使用哪个底层协议族，type指定服务类型，protocol通常设置为0，表示使用前两个参数确定的默认协议。  
socket系统调用成功时返回一个socket文件描述符。

**命名socket**

创建socket时，我们给它指定了地址族，但是并未指定使用该地址族中的哪个具体socket地址。将一个socket与socket地址绑定称为给socket命名。命名socket的系统调用是bind，其定义如下：

```
#include <sys/types.h>
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr* my_addr, socklen_t addrlen);
```
bind将my_addr所指的socket地址分配给未命名的sockfd文件描述符，addrlen参数指出该socket地址的长度。  
bind成功时返回0，失败时返回-1并设置errno。其中两种常见的errno是EACCES和EADDRINUSE。
- EACCES，被绑定的地址是受保护的地址，仅超级用户能够访问。比如普通用户将socket绑定到知名服务端口（0~1023）上时，bind将返回EACCES错误。
- EADDRINUSE，被绑定的地址正在使用中。比如将socket绑定到一个处于TIME_WAIT状态的socket地址。

**监听socket**

socket被命名之后，还不能马上接受客户连接，我们需要创建一个监听队列以存放待处理的客户连接。

```
#include <sys/socket.h>
int listen(int socket, int backlog);
```

**接受连接**

下面的系统调用从listen监听队列中接受一个连接：

```
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```
sockfd参数是执行过listen系统调用的监听socket（我们把执行过listen调用、处于LISTEN状态的socket称为监听socket，而所有处于ESTABLISHED状态的socket则称为连接socket）。addr参数用来获取被接受连接的远端socket地址，该socket地址的长度由addrlen参数指出。accept成功时返回一个新的连接socket，该socket唯一地标识了被接受的这个连接，服务器可通过读写该socket来与被接受连接对应的客户端通信。accept失败时返回-1并设置errno。  
accept只是从监听队列中取出连接，而不论连接处于何种状态（ESTABLISHED或者CLOSE_WAIT状态），更不关心任何网络状况的变化。

**发起连接**

如果说服务器通过listen调用来被动接受连接，那么客户端需要通过如下系统调用来主动与服务器建立连接：

```
#include <sys/types.h>
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *serv_addr, socklen_t addrlen);
```
sockfd参数由socket系统调用返回一个socket。serv_addr参数是服务器监听的socket地址，addrlen参数则指定这个地址的长度。

**关闭连接**

1. close：并非立即关闭一个连接，而是将fd的引用计数减1.只有当fd的引用计数为0时才真正关闭连接。且close关闭连接时只能将socket上的读和写同时关闭。
2. shutdown：如果无论如何都要立即终止连接，可以使用shutdown系统调用。shutdown能够分别关闭socket上的读或写，或者都关闭。

## 第6章 高级I/O函数

1. pipe函数可用于创建一个管道，以实现进程间通信。通过pipe函数创建的两个文件描述符分别构成管道的两端，fd[0]只能用于从管道读出数据，fd[1]则只能用于往管道写入数据，而不能反过来使用。如果要实现双向的数据传输，就应该使用两个管道。socket的基础API中有一个socketpair函数，它能够方便地创建双向管道。  
2. dup函数创建一个新的文件描述符，该文件描述符和原有文件描述符file\_descriptor指向相同的文件、管道或者网络连接。并且dup返回的文件描述符总是取系统当前可用的最小整数值。dup2和dup类似，不过它将返回第一个不小于file\_descriptor\_two的整数值。
3. readv函数和writev函数：readv函数将数据从文件描述符读到分散的内存块中，即分散读；writev函数则将多块分散的内存数据一并写入文件描述符中，即集中写。
4. sendfile函数在两个文件描述符之间直接传递数据，从而避免了内核缓冲区和用户缓冲区之间的数据拷贝，效率很高，这被称为零拷贝。函数中有两个参数：in\_fd,out\_fd，in\_fd参数是待读出内容的文件描述符，out\_fd参数是待写入内容的文件描述符。in\_fd必须指向真实的文件，不能是socket和管道；而out\_fd必须是一个socket。
5. mmap函数用于申请一段内存空间，我们可以将这段内存作为进程间通信的共享内存，也可以将文件直接映射到其中。munmap函数则释放由mmap创建的这段内存空间。
6. splice函数用于在两个文件描述符之间移动数据，也是零拷贝操作。使用splice函数时，fd\_in和fd\_out必须至少有一个是管道文件描述符。
7. tee函数在两个管道文件描述符之间复制数据，也是零拷贝操作。它不消耗数据，因此源文件描述符上的数据仍然可以用于后续的读操作。
8. fcntl函数，提供了对文件描述符的各种控制操作。在网络编程中，fcntl函数通常用来将一个文件描述符设置为非阻塞的。

## 第7章 Linux服务器程序规范
需要处理日志、用户信息、进程间关系、系统资源限制、工作目录等。细节将在后面章节展示。
