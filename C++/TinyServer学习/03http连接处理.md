## epoll

#### epoll_create函数

~~~C++
#include <sys/epoll>
int epoll_create(int size)

// 创建一个指示epoll内核事件表的文件描述符，该描述符将作用其他epoll系统调用的第一参数
~~~

#### epoll_ctl函数

~~~C++
#include <sys/epoll>
// 该函数用于操作内核事件表监控的文件描述符上的事件：注册、修改、删除
int epoll_ctl(int epfd, in op, in fd, struct epoll_event *event)

// epfd:为epoll_create的句柄
// op: 表示动作，用3个宏表示:
	EPOLL_CTL_ADD (注册新的fd到epfd),
	EPOLL_CTL_MOD (修改已经注册的fd的监听事件),
	EPOLL_CTL_DEL (从epfd删除一个fd);
// event:
struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
// EPOLLIN：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）
// 表示对应的文件描述符可以写
// 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）
// 表示对应的文件描述符发生错误
// 表示对应的文件描述符被挂断；
// 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)而言的
// 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
~~~

#### epoll_wait函数

~~~C++
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeous)

// events:用来存内核事件的集合
// maxevents: 告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size
// time:是超时事件
    //-1:阻塞
	// 0:立即返回，非阻塞
	// >0:指定毫秒
// 返回值：成功返回有多少文件描述符就绪，时间到时返回0，出错返回-1
~~~

## select/poll/epoll

- **调用函数**
  - select和poll都是一个函数，epoll是一组函数
- **文件描述符数量**
  - select通过线性表描述文件描述符集合，文件描述符有上限，一般是1024，但可以修改源码，重新编译内核，不推荐
  - poll是链表描述，突破了文件描述符上限，最大可以打开文件的数目
  - epoll通过红黑树描述，最大可以打开文件的数目，可以通过命令ulimit -n number修改，仅对当前终端有效
- **将文件描述符从用户传给内核**
  - select和poll通过将所有文件描述符拷贝到内核态，每次调用都需要拷贝
  - epoll通过epoll_create建立一棵红黑树，通过epoll_ctl将要监听的文件描述符注册到红黑树上
- **内核判断就绪的文件描述符**
  - select和poll通过**遍历文件描述符集合**，判断哪个文件描述符上有事件发生
  - epoll_create时，内核除了帮我们在**epoll文件系统里建了个红黑树用于存储以后epoll_ctl传来的fd外**，还会**再建立一个list链表，用于存储准备就绪的事件，当epoll_wait调用时，仅仅观察这个list链表里有没有数据**即可。
  - epoll是根据每个fd上面的回调函数(中断函数)判断，只有发生了事件的socket才会主动的去调用 callback函数，其他空闲状态socket则不会，若是就绪事件，插入list
- **应用程序索引就绪文件描述符**
  - select/poll只返回发生了事件的文件描述符的个数，若知道是哪个发生了事件，同样需要遍历
  - epoll返回的发生了事件的个数和结构体数组，结构体包含socket的信息，因此直接处理返回的数组即可
- **工作模式**
  - select和poll都只能工作在相对低效的LT模式下
  - epoll则可以工作在ET高效模式，并且epoll还支持EPOLLONESHOT事件，该事件能进一步减少可读、可写和异常事件被触发的次数。 
- **应用场景**
  - 当所有的fd都是活跃连接，使用epoll，需要建立文件系统，红黑书和链表对于此来说，效率反而不高，不如selece和poll
  - 当监测的fd数目较小，且各个fd都比较活跃，建议使用select或者poll
  - 当监测的fd数目非常大，成千上万，且单位时间只有其中的一部分fd处于就绪状态，这个时候使用epoll能够明显提升性能

## ET、LT、EPOLLONESHOT

- **LT水平触发模式**
  - epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序可以不立即处理该事件。
  - 当下一次调用epoll_wait时，epoll_wait还会再次向应用程序报告此事件，直至被处理
- **ET边缘触发模式**
  - epoll_wait检测到文件描述符有事件发生，则将其通知给应用程序，应用程序必须立即处理该事件
  - 必须要一次性将数据读取完，使用非阻塞I/O，读取到出现eagain
- **EPOLLONESHOT**
  - 一个线程读取某个socket上的数据后开始处理数据，在处理过程中该socket上又有新数据可读，此时另一个线程被唤醒读取，此时出现两个线程处理同一个socket
  - 我们期望的是一个socket连接在任一时刻都只被一个线程处理，通过epoll_ctl对该文件描述符注册epolloneshot事件，一个线程处理socket时，其他线程将无法处理，**当该线程处理完后，需要通过epoll_ctl重置epolloneshot事件**

## 请求报文

~~~C++
// GET报文
GET /562f25980001b1b106000338.jpg HTTP/1.1
Host:img.mukewang.com
User-Agent:Mozilla/5.0 (Windows NT 10.0; WOW64)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36
Accept:image/webp,image/*,*/*;q=0.8
Referer:http://www.imooc.com/
Accept-Encoding:gzip, deflate, sdch
Accept-Language:zh-CN,zh;q=0.8
空行
请求数据为空
    
// POST报文
POST / HTTP1.1
Host:www.wrox.com
User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
Content-Type:application/x-www-form-urlencoded
Content-Length:40
Connection: Keep-Alive
空行
name=Professional%20Ajax&publisher=Wiley
~~~

## 请求报文

HTTP请求报文由**请求行**（request line）、请求头部（header）、**空行**和请求数据四个部分组成。

**请求行**，用来说明请求类型,要访问的资源以及所使用的HTTP版本。
**GET**说明请求类型GET，/562f25980001b1b106000338.jpg(URL)为要访问的资源，该行的最后一部分说明使用的是HTTP1.1版本。

**请求头部**，紧接着请求行（即第一行）之后的部分，用来说明服务器要使用的附加信息。
    **HOST**，给出请求资源所在服务器的域名。
    User-Agent，HTTP客户端程序的信息，该信息由你发出请求使用的浏览器来定义,并且在每个请求中自动发送等。
    **Accept**，说明用户代理可处理的媒体类型。
    **Accept-Encoding**，说明用户代理支持的内容编码。
    **Accept-Language**，说明用户代理能够处理的自然语言集。
    **Content-Type**，说明实现主体的媒体类型。
    **Content-Length**，说明实现主体的大小。
    **Connection**，连接管理，可以是Keep-Alive或close。

**空行**，请求头部后面的空行是必须的即使第四部分的请求数据为空，也必须有空行。

**请求数据也叫主体**，可以添加任意的其他数据。

## 响应报文

HTTP响应也由四个部分组成，分别是：**状态行**、消息报头、**空行**和响应正文。

~~~C++
// GET
GET /562f25980001b1b106000338.jpg HTTP/1.1
Host:img.mukewang.com
User-Agent:Mozilla/5.0 (Windows NT 10.0; WOW64)
AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.106 Safari/537.36
Accept:image/webp,image/*,*/*;q=0.8
Referer:http://www.imooc.com/
Accept-Encoding:gzip, deflate, sdch
Accept-Language:zh-CN,zh;q=0.8
空行
请求数据为空
// POST
POST / HTTP1.1
Host:www.wrox.com
User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
Content-Type:application/x-www-form-urlencoded
Content-Length:40
Connection: Keep-Alive
空行
name=Professional%20Ajax&publisher=Wiley    
~~~

**状态行**，由HTTP协议版本号， 状态码， 状态消息 三部分组成。
**第一行为状态行**，（HTTP/1.1）表明HTTP版本为1.1版本，状态码为200，状态消息为OK。

消息报头，用来说明客户端要使用的一些附加信息。
**第二行和第三行为消息报头**，Date:生成响应的日期和时间；Content-Type:指定了MIME类型的HTML(text/html),编码类型是UTF-8。

**空行**，消息报头后面的空行是必须的。

**响应正文**，服务器返回给客户端的文本信息。空行后面的html部分为响应正文。

## HTTP状态码

HTTP有5种类型的状态码，具体的：

- 1xx：指示信息--表示请求已接收，继续处理。
- 2xx：成功--表示请求正常处理完毕。
  - 200 OK：客户端请求被正常处理。
  - 206 Partial content：客户端进行了范围请求。
- 3xx：重定向--要完成请求必须进行更进一步的操作。
  - 301 Moved Permanently：永久重定向，该资源已被永久移动到新位置，将来任何对该资源的访问都要使用本响应返回的若干个URI之一。
  - 302 Found：临时重定向，请求的资源现在临时从不同的URI中获得。
- 4xx：客户端错误--请求有语法错误，服务器无法处理请求。
  - 400 Bad Request：请求报文存在语法错误。
  - 403 Forbidden：请求被服务器拒绝。
  - 404 Not Found：请求不存在，服务器上找不到请求的资源。
- 5xx：服务器端错误--服务器处理请求出错。
  - 500 Internal Server Error：服务器在执行请求时出现错误。

## 有限状态机

~~~C++
STATE_MACHINE(){
	State cur_State = type_A;
	while(cur_State != type_C){
    	Package _pack = getNewPackage();
        switch(){
        	case type_A:
           		process_pkg_state_A(_pack);
                cur_State = type_B;
                break;
            case type_B:
                process_pkg_state_B(_pack);
                cur_State = type_C;
                break;
        }
    }
}
~~~

## 流程图与状态机

**从状态机负责读取报文的一行，主状态机负责对该行数据进行解析**，主状态机内部调用从状态机，从状态机驱动主状态机。

![640 (2)](E:\MarkDown_study\C++\TinyServer学习\img\640 (2).webp)

#### **主状态机**

三种状态，标识解析位置。

- CHECK_STATE_REQUESTLINE，解析请求行
- CHECK_STATE_HEADER，解析请求头
- CHECK_STATE_CONTENT，解析消息体，仅用于解析POST请求

#### **从状态机**

三种状态，标识解析一行的读取状态。

- LINE_OK，完整读取一行
- LINE_BAD，报文语法有误
- LINE_OPEN，读取的行不完整

#### **HTTP_CODE含义**

表示HTTP请求的处理结果，在头文件中初始化了八种情形，在报文解析时只涉及到四种。

- NO_REQUEST
  - 请求不完整，需要继续读取请求报文数据
- GET_REQUEST
  - 获得了完整的HTTP请求
- BAD_REQUEST
  - HTTP请求报文有语法错误
- INTERNAL_ERROR
  - 服务器内部错误，该结果在主状态机逻辑switch的default下，一般不会触发

#### **解析报文整体流程**

process_read通过while循环，将主从状态机进行封装，对报文的每一行进行循环处理。

- 判断条件
  - 主状态机转移到CHECK_STATE_CONTENT，该条件涉及解析消息体
  - 从状态机转移到LINE_OK，该条件涉及解析请求行和请求头部
  - 两者为或关系，当条件为真则继续循环，否则退出
- 循环体
  - 从状态机读取数据
  - 调用get_line函数，通过m_start_line将从状态机读取数据间接赋给text
  - 主状态机解析text

## 基础API

#### stat

~~~C++
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>

//获取文件属性，存储在statbuf中
int stat(const char *pathname, struct stat *statbuf);

struct stat {
   mode_t    st_mode;        /* 文件类型和权限 */
   off_t     st_size;        /* 文件大小，字节数*/
};
~~~

#### mmap

~~~~C++
void* mmap(void* start,size_t length,int prot,int flags,int fd,off_t offset);
int munmap(void* start,size_t length);

// start：映射区的开始地址，设置为0时表示由系统决定映射区的起始地址
// length：映射区的长度
// prot：期望的内存保护标志，不能与文件的打开模式冲突
	PROT_READ 表示页内容可以被读取
// flags：指定映射对象的类型，映射选项和映射页是否可以共享
	MAP_PRIVATE 建立一个写入时拷贝的私有映射，内存区域的写入不会影响到原文件
// fd：有效的文件描述符，一般是由open()函数返回
// off_toffset：被映射对象内容的起点
~~~~

#### iovec

~~~~C++
定义了一个向量元素，通常，这个结构用作一个多元素的数组。
struct iovec {
	void	*iov_base; /* starting address of buffer */
	size_t	iov_len; /* size of buffer */
};
~~~~

#### writev

~~~~C++
// writev函数用于在一次函数调用中写多个非连续缓冲区，有时也将这该函数称为聚集写。
#include <sys/uio.h>
ssize_t writev(int filedes, const struct iovec *iov, int iovcnt);
// filedes表示文件描述符
// iov为前述io向量机制结构体iovec
// iovcnt为结构体的个数
若成功则返回已写的字节数，若出错则返回-1。writev以顺序iov[0]，iov[1]至iov[iovcnt-1]从缓冲区中聚集输出数据。writev返回输出的字节总数，通常，它应等于所有缓冲区长度之和。

特别注意： 循环调用writev时，需要重新处理iovec中的指针和长度，该函数不会对这两个成员做任何处理。writev的返回值为已写的字节数，但这个返回值“实用性”并不高，因为参数传入的是iovec数组，计量单位是iovcnt，而不是字节数，我们仍然需要通过遍历iovec来计算新的基址，另外写入数据的“结束点”可能位于一个iovec的中间某个位置，因此需要调整临界iovec的io_base和io_len。
~~~~

## 流程图

浏览器端发出HTTP请求报文，服务器端接收该报文并调用`process_read`对其进行解析，根据解析结果`HTTP_CODE`，进入相应的逻辑和模块。

其中，服务器子线程完成报文的解析与响应；主线程监测读写事件，调用`read_once`和`http_conn::write`完成数据的读取与发送。

![640](E:\MarkDown_study\C++\TinyServer学习\img\640.webp)