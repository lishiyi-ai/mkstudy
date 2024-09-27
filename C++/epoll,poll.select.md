## select

~~~cpp
#include <sys/select.h>
#include <sys/time.h>
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
// return：表示此时有多少个监控的描述符就绪，若超时则为0，出错为-1。
// nfds：表示表示文件描述符最大的数目+1，这个数目是指读事件和写事件中数目最大的，+1是为了全面检查
// readfds：表示需要监视的会发生读事件的fd，没有设置为NULL
// writefds：表示需要监视的会发生写事件的fd，没有设置为NULL
// exceptfds：表示异常处理的，暂时没用到。。。
// timeout：表示阻塞的时间，如果是0表示非阻塞模型，NULL表示永远阻塞，直到有数据来
struct timeval {
   long    tv_sec;         /* seconds */
   long    tv_usec;        /* microseconds */
};
// 通用模型
int main() {


  fd_set read_fs, write_fs;
  struct timeval timeout;
  int max_sd = 0;  // 用于记录最大的fd，在轮询中时刻更新即可
  
  /*
   * 这里进行一些初始化的设置，
   * 包括socket建立，地址的设置等,
   * 同时记得初始化max_sd
   */

  // 初始化比特位
  FD_ZERO(&read_fs);
  FD_ZERO(&write_fs);

  int rc = 0;
  int desc_ready = 0; // 记录就绪的事件，可以减少遍历的次数
  while (1) {
    // 这里进行阻塞
    rc = select(max_sd + 1, &read_fd, &write_fd, NULL, &timeout);
    if (rc < 0) {
      // 这里进行错误处理机制
    }
    if (rc == 0) {
      // 这里进行超时处理机制
    }

    desc_ready = rc;
    // 遍历所有的比特位，轮询事件
    for (int i = 0; i <= max_sd && desc_ready; ++i) {
      if (FD_ISSET(i, &read_fd)) {
        --desc_ready;
        // 这里处理read事件，别忘了更新max_sd
      }
      if (FD_ISSET(i, &write_fd)) {
        // 这里处理write事件，别忘了更新max_sd
      }
    }
  }
}

~~~

**select 调用过程**

1）用户进程需要监控某些资源 fds，在调用 select 函数后会阻塞，操作系统会将用户线程加入这些资源的等待队列中。

2）直到有描述副就绪（有数据可读、可写或有 except）或超时（timeout 指定等待时间，如果立即返回设为 null 即可），函数返回。

3）select 函数返回后，中断程序唤起用户线程。用户可以遍历 fds，通过 FD_ISSET 判断具体哪个 fd 收到数据，并做出相应处理。

select 函数优点明显，实现起来简单有效，且几乎所有操作系统都有对应的实现。

**select 的缺点**

1.通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。

1）每次调用 select 都需要**将进程加入到所有监视 fd 的等待队列**，每次唤醒都需要从每个队列中移除。 这里**涉及了两次遍历**，而且每次都要将整个 fd_set 列表传递给内核，有一定的开销。

2）当函数返回时，系统会将**就绪描述符写入 fd_set 中，并将其拷贝到用户空间**。**进程被唤醒后，用户线程并不知道哪些 fd 收到数据，还需要遍历一次**。

## poll

可以认为`poll`是一个增强版本的`select`，因为`select`的比特位操作决定了一次性最多处理的读或者写事件只有1024个，而`poll`使用一个新的方式优化了这个模型。

~~~cpp
struct pollfd {
	int fd;          // 需要监视的文件描述符
	short events;    // 需要内核监视的事件
	short revents;   // 实际发生的事件
};

#include <poll.h>
int poll(struct pollfd* fds, nfds_t nfds, int timeout);
// fds：一个pollfd队列的队头指针，我们先把需要监视的文件描述符和他们上面的事件放到这个队列中
// nfds：队列的长度
// timeout：事件操作，设置指定正数的阻塞事件，0表示非阻塞模式，-1表示永久阻塞。
// 时间的数据结构：
struct timespec {
	long    tv_sec;         /* seconds */
    long    tv_nsec;        /* nanoseconds */
};
// 先宏定义长度
#define MAX_POLLFD_LEN 200  

int main() {
  /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

  int rc = 0;
  pollfd fds[MAX_POLL_LEN];
  memset(fds, 0, sizeof(fds));
  int ndfs  = 1;  // 队列的实际长度，是一个随时更新的，也可以自定义其他的
  int timeout = 0;
  /*
   * 在这里进行一些感兴趣事件的注册，
   * 每个pollfd可以注册多个类型的事件，
   * 使用 | 操作即可，就行博文提到的那样。
   * 根据需要设置阻塞时间
   */

  int current_size = ndfs;
  int compress_array = 0;  // 压缩队列的标记
  while (1) {
    rc = poll(fds, nfds, timeout);
    current_size = ndfs;
    if (rc < 0) {
    // 这里进行错误处理
    }
    if (rc == 0) {
    // 这里进行超时处理
    }

    for (int i = 0; i < current_size; ++i) {
      if (fds[i].revents == 0){  // 没有事件可以处理
        continue;
      }
      if (fds[i].revents & POLLIN) {  // 简单的例子，比如处理写事件
      
      }
      /*
       * current_size 是为了降低复杂度的，可以随时进行更新
       * ndfs如果要更新，应该是最后统一进行
       */
    }

    if (compress_array) {  // 如果需要压缩队列
      compress_array = 0;
      for (int i = 0; i < ndfs; ++i) {
        for (int j = i; j < ndfs; ++j) {
          fds[i].fd = fds[j + i].fd;
        }
        --i;
        --ndfs;
      }
    }
  }
}

~~~

poll 函数与 select 原理相似，都需要来回拷贝全部监听的文件描述符，不同的是：

1）poll 函数采用链表的方式替代原来 select 中 fd_set 结构，因此可监听文件描述符数量不受限。

2）poll 函数返回后，可以通过 pollfd 结构中的内容进行处理就绪文件描述符，相比 select 效率要高。

3）新增**水平触发**：也就是通知程序 fd 就绪后，这次没有被处理，那么下次 poll 的时候会再次通知同个 fd 已经就绪。

**poll 缺点**

和 select 函数一样，poll 返回后，需要轮询 pollfd 来获取就绪的描述符。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。

## epoll

~~~cpp
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
typedef union epoll_data {
	void *ptr;
	int fd;
	uint32_t u32;
	uint64_t u64;
} epoll_data_t;
// epoll_data是一个union类型。fd很容易理解，是文件描述符；而文件描述符本质上是一个索引内核中资源地址的一个下标描述，因此也可以用ptr指针代替；同样的这些数据可以用整数代替。
struct epoll_event {
	uint32_t events;
	epoll_data_t data;
};
// 再来看epoll_event，有一个data用于表示fd，之后又有一个events表示注册的事件。
//events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
 
// 使用模板
#define MAX_EVENTS 10
int main() {
	struct epoll_event ev, events[MAX_EVENTS];
    int listen_sock, conn_sock, nfds, epollfd;

    /* Code to set up listening socket, 'listen_sock',
     (socket(), bind(), listen()) omitted */

	epollfd = epoll_create1(0);
	if (epollfd == -1) {
		perror("epoll_create1");
      	exit(EXIT_FAILURE);
	}

	ev.events = EPOLLIN;
	ev.data.fd = listen_sock;
	if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
		perror("epoll_ctl: listen_sock");
		exit(EXIT_FAILURE);
	}

	for (;;) {
	    // 永久阻塞，直到有事件
		nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
		if (nfds == -1) {  // 处理错误
			perror("epoll_wait");
			exit(EXIT_FAILURE);
		}

		for (n = 0; n < nfds; ++n) {
			if (events[n].data.fd == listen_sock) {
				conn_sock = accept(listen_sock, (struct sockaddr *) &addr, &addrlen);
				if (conn_sock == -1) {
					perror("accept");
					exit(EXIT_FAILURE);
				}
				setnonblocking(conn_sock);
				ev.events = EPOLLIN | EPOLLET;
				ev.data.fd = conn_sock;
				if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock, &ev) == -1) {
					perror("epoll_ctl: conn_sock");
					exit(EXIT_FAILURE);
				}
			} else {
				do_use_fd(events[n].data.fd);
			}
		}
	}
	return 0;
}

~~~

通过 epoll_ctl 函数添加进来的事件都会被放在红黑树的某个节点内，所以，重复添加是没有用的。

当把事件添加进来的时候时候会完成关键的一步，那就是该事件都会与相应的设备驱动程序建立回调关系，当相应的事件发生后，就会调用这个回调函数，该回调函数在内核中被称为 ep_poll_callback，这个回调函数其实就所把这个事件添加到 rdllist 这个双向链表中。一旦有事件发生，epoll 就会将该事件添加到双向链表中。那么当我们调用 epoll_wait 时，epoll_wait 只需要检查 rdlist 双向链表中是否有存在注册的事件，效率非常可观。这里也需要将发生了的事件复制到用户态内存中即可。

所有 **FD 集合采用红黑树存储**，**就绪 FD 集合使用链表存储**。这是因为就绪 FD 都需要处理，业务优先级需求，最好的选择便是线性数据结构。

**工作模式**
**1）LT模式**

LT（level triggered）模式：也是默认模式，即当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件，并且下次调用 epoll_wait 时，会再次响应应用程序并通知此事件。

**EPOLLIN触发条件：**

1. 处于可读状态。
2. 从不可读状态变为可读状态。

**EPOLLOUT触发条件:**

1. 处于可写状态。
2. 从不可写状态变为可写状态。

**就绪事件包含EPOLLIN的条件：**

- 刚建立连接
- 有数据可读
- 断开连接

**就绪事件包含EPOLLOUT的条件：**

- 内核写缓冲区未满，可写

**2）ET模式**

ET（edge-triggered）模式：当 epoll_wait 检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

ET 是一种高速工作方式，很大程度上减少了 epoll 事件被重复触发的次数。epoll 工作在 ET 模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

为何高效**
1） epoll 精巧的使用了 3 个方法来实现 select 方法要做的事，分清了频繁调用和不频繁调用的操作。

epoll_ctrl 是不太频繁调用的，而 epoll_wait 是非常频繁调用的。而 **epoll_wait 却几乎没有入参**，这比 select 的效率高出一大截，而且，它也不会随着并发连接的增加使得入参越发多起来，导致内核执行效率下降。

2） **mmap 的引入**，将用户空间的一块地址和内核空间的一块地址同时映射到相同的一块物理内存地址（不管是用户空间还是内核空间都是虚拟地址，最终要通过地址映射映射到物理地址），使得这块物理内存对内核和对用户均可见，减少用户态和内核态之间的数据交换。

3）**红黑树将存储 epoll 所监听的 FD。**高效的数据结构，本身插入和删除性能比较好，时间复杂度O(logN)。

#### ET模式
LT模式比较好理解，关键是ET模式。

**EPOLLIN触发条件：**
	从不可读状态变为可读状态。
	内核接收到新发来的数据。

**EPOLLOUT触发条件：**
	从不可写状态变为可写状态。

**就绪事件包含EPOLLIN的条件：**
	刚建立连接
	内核接收到新数据
	断开连接

**就绪事件包含EPOLLOUT的条件：**
	每次注册EPOLLOUT，下一次epoll_wait返回就绪事件包含EPOLLOUT（刚注册当然可写）
	内核写缓冲区从不可写变成可写
	同时注册了EPOLLIN和EPOLLOUT，如果接收到新数据时可写则就绪事件既包含EPOLLIN也包含EPOLLOUT



# epoll底层原理深究

#### eventpoll结构体

在epoll的使用中，我们经常需要对文件描述符集合进行添加、删除等操作，同时对触发的事件类型进行处理，回调IO事件中的工作函数。这其中离不开两个数据结构的帮助------epitem与eventpoll。

~~~cpp
// file：fs/eventpoll.c
struct eventpoll { 	
    //sys_epoll_wait用到的等待队列
    wait_queue_head_t wq;
    //接收就绪的描述符都会放到这里
    struct list_head rdllist;
    //每个epoll对象中都有一颗红黑树
    struct rb_root rbr;
    ......
}
~~~

**其中wq为等待队列链表**，软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。（epoll_wait若就绪队列中无数据，会将当前进程加入到等待队列中）

**rdllist标识就绪的文件描述符链表**，当有的连接就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。

**rbr为一颗红黑树**，采用红黑树的结构为epoll高效的处理海量数据的增删改查，这个红黑树用于管理用户进程添加进来的所有socket连接。

#### epitem结构体

epitem是每一个IO所对应的的事件，比如epoll_ctl()中使用EPOLL_CTL_ADD操作的时候，就需要创建一个epitem。

~~~cpp
// file：fs/eventpoll.c
struct epitem {
    //红黑树节点
    struct rb_node rbn;
    //socket文件描述符信息
    struct epoll_filefd ffd;
    //所归属的 eventpoll 对象
    struct eventpoll *ep;
    //等待队列
    struct list_head pwqlist;
}
~~~

