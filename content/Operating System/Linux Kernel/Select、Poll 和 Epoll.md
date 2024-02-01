---
tags:
  - wiki
publish: true
---

# 背景：内核的阻塞和非阻塞

在使用 Linux 内核的 socket 的时候，我们需要考虑阻塞与非阻塞的问题。Kernel 内部维护了定长的 buffer 来保存需要传递的数据和收到的数据。因此，一般情况下，我们写这样的系统调用的时候，都是阻塞的：

```c
int ret = read(fd, &buf, 1024);
int ret = write(fd, &buf, 1024);
```

这意味着，如果内核内部的 buffer 是空的，就会等到有数据时才返回。

但是网络请求与本地的数据传输不一样，网络是可能产生各种错误的。因此，如果采用纯阻塞的设计，会导致 CPU 被白白浪费掉，甚至造成死锁等更严重的问题。考虑到这一点，内核也提供了非阻塞的方式：

```c
int sock = socket(AF_INET, SOCK_STEAM | SOCK_NONBLOCK, 0);

int ret = read(sock, &buffer, 1024);

if (ret > 0) {
	// read some bytes
} else if (ret == 0) {
	// end of file, other side done writing
} else if (errno == EAGAIN) {
	// empty buffer, no data from socket yet
} else {
	// error
}
```

# 问题 1：concurrency

如果使用上面的非阻塞接口，就需要有一个循环反复查询是否成功：

```cpp
while (true) {
	ret = read(sock, &buffer, 1024);
	// handle sock1

	ret = read(sock2, &buffer, 1024);
	// handle sock2
}
```

这种 polling 的方式会非常消耗 CPU 时间，造成能效和性能的下降。这里是一个常规的 tradeoff，我们需要通过异步的方式来降低 polling 带来的开销，例如中断的机制。如果没有请求到来，最好的办法应该是让程序进入睡眠，腾出 CPU 来做别的工作。

# 解决方案 1：select

`select(2)` 的存在就是为了解决上面的问题：

```txt
NAME
       select — synchronous I/O multiplexing

SYNOPSIS
       #include <sys/select.h>

       int select(int nfds, fd_set *restrict readfds,
           fd_set *restrict writefds, fd_set *restrict errorfds,
           struct timeval *restrict timeout);
```

`select` 的机制是，应用可以将需要管理的一系列 fd 都传入 kernel，kernel 负责监听这些 fd，一旦某些 fd 已经准备好（e.g. 有数据可读），就返回应用。应用可以自己检查哪些 fd 需要响应，并执行对应的操作。

```cpp
while (true) {  
    fd_set fds;
    FD_ZERO(&fds);
    FD_SET(fd, &fds);
    FD_SET(fd2, &fds);
  
    // wait
    select(FD_SETSIZE, &fds, nullptr, nullptr, nullptr);

    // react
}
```

当 kernel 返回时，可以通过检查每一个 fd，来确定哪些 fd 有事件发生：

```c
// react  
if (FD_ISSET(sock, &fds)) {  
     ret = read(sock, &buffer, 1024);  
}  
if (FD_ISSET(sock2, &fds)) {  
    ret = read(sock2, &buffer, 1024);  
}
```

`select` 机制带来了良好的 I/O 多路复用能力，且不会造成性能上的副作用。然而，`select` 的最大问题是采用了定值作为 fd 集合的大小，这意味着如果我们在程序中需要处理超过 `fd_set` 所能承受的上限，`select` 就无法使用了。

# 解决方案 2：poll

`poll(2)` 就是为了解决这个问题而实现的：

```
NAME
       poll — input/output multiplexing

SYNOPSIS
       #include <poll.h>

       int poll(struct pollfd fds[], nfds_t nfds, int timeout);
```

`poll` 采用了动态长度的数组来处理 fd，同时在 API 的设计方面也有一些改进，例如引入了输入和输出两个事件，同时通过参数 `revent` 来表示产生的事件，从而提供了更细粒度的事件控制。

```cpp
while (true) {  
    pollfd fds[2] = {  
            pollfd{.fd=sock, .events=POLLIN, .revents = 0},  
            pollfd{.fd=sock2, .events=POLLOUT, .revents = 0},  
    };  
  
    // wait  
    poll(fds, std::size(fds), -1);  

	// react
	if (fds[0].revents == POLLIN) {  
	    ret = read(sock, &buffer, 1024);  
	}  
	if (fds[1].revents == POLLOUT) {  
	    ret = write(sock2, &buffer, 1024);  
	}
}
```

然而，`poll` 也存在一个问题：**每次处理事件，都需要遍历**。内核拿到应用提供的 fd 数组，需要遍历检查哪些 fd 产生了事件；应用从内核返回时，也需要编译所有的数组来确定哪些 fd 产生的消息。当 fd 数组的数目变得非常大的时候，这显然会成为一个性能的瓶颈。

# 解决方案 3：epoll

Linux kernel 为了解决这个问题，提出了 `epoll(2)`:

```
NAME
       epoll - I/O event notification facility

SYNOPSIS
       #include <sys/epoll.h>
```

`epoll` 在内核中维护了一个数据结构，用户与之交互来实现 fd 的插入和删除，两者保持数据的同步。这样，内核就能够直接返回哪些 fd 产生了事件。同时，`epoll` 也保持了 `poll` 所实现的事件分离机制，能够根据不同的事件细粒度地通知。

```cpp
int epfd = epoll_create1(0);

epoll_event evts[2] = {
	epoll_event{
		.events = EPOLLIN,
		.data = epoll_data_t{.fd = sock},
	},
	epoll_event{
		.events = EPOLLOUT,
		.data = epoll_data_t{.fd = sock2},
	},
};

epoll_ctl(epfd, EPOLL_CTL_ADD, sock, &evts[0]);
epoll_ctl(epfd, EPOLL_CTL_ADD, sock2, &evts[1]);

while (true) {
	epoll_event evt;
	
	// wait
	epoll_wait(epfd, &evt, 1, -1);
	
	// react
    if (evt.data.fd == sock) {
		ret = read(sock, &buffer, 1024);
    }
    if (evt.data.fd == sock2) {
		ret = write(sock2, &buffer, 1024);
    }
}
```