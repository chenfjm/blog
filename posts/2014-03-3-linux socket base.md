---
layout: post
title: linux socket 编程基础
category: linux
description:
---

**socket介绍**  

要弄清楚什么是socket，首先得理解系统是如何唯一标识一个网络中的进程的，进程间是如何通信的。网络层的"ip地址"可以唯一标识网络中的主机，而传输层的"协议+端口"可以唯一标识主机中的应用程序（进程）。这样利用三元组（ip地址，协议，端口）就可以标识网络的进程了，网络中的进程通信就可以利用这个标志与其它进程进行交互。而目前英特网中的进程间交互遵循的协议叫TCP/IP协议。使用TCP/IP协议的应用程序通常采用应用编程接口UNIX BSD的套接字（socket）来实现网络进程之间的通信。  

上面我们已经知道网络中的进程是通过socket来通信的，那什么是socket呢？socket是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用以实现进程在网络中通信。  
有如下三种类型的socket:  

- 流套接字（SOCK_STREAM）,流套接字用于提供面向连接、可靠的数据传输服务。这是一个使用最多的socket类型，这个socket是使用TCP来进行传输。
- 数据报套接字（SOCK_DGRAM）,数据报套接字提供了一种无连接的服务，使用UDP协议进行数据的传输。
- 原始套接字(SOCK_RAW),原始套接字可以读写内核没有处理的IP数据包，而流套接字只能读取TCP协议的数据，数据报套接字只能读取UDP协议的数据。  

**socket API**  

socket()  

	int socket(int domain, int type, int protocol);

参数:  

- domain：即协议域，又称为协议族（family）。常用的协议族有，AF_INET、AF_INET6、AF_LOCAL（AF_UNIX）、AF_ROUTE。
- type: socket类型，常用类型有SOCK_STREAM、SOCK_DGRAM、SOCK_RAW。
- protocol: 具体协议,常用的协议有，IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC等，它们分别对应TCP传输协议、UDP传输协议、STCP传输协议、TIPC传输协议。  

socket()用于创建一个socket描述符，它唯一标识一个socket。这个socket描述字跟文件描述字一样，后续的操作都有用到它，把它作为参数，通过它来进行一些读写操作。  

bind()  

	int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);  

参数:  

- sockfd：即socket描述字，它是通过socket()函数创建了，唯一标识一个socket。
- addr：一个const struct sockaddr *指针，指向要绑定给sockfd的协议地址。这个地址结构根据地址创建socket时的地址协议族的不同而不同。  

ipv4地址结构:  

	struct sockaddr_in {
    	sa_family_t    sin_family; /* address family: AF_INET */
    	in_port_t      sin_port;   /* port in network byte order */
    	struct in_addr sin_addr;   /* internet address */
	};

	/* Internet address. */
	struct in_addr {
    	uint32_t       s_addr;     /* address in network byte order */
	};  

ipv6地址结构:  

	struct sockaddr_in6 { 
    	sa_family_t     sin6_family;   /* AF_INET6 */ 
    	in_port_t       sin6_port;     /* port number */ 
    	uint32_t        sin6_flowinfo; /* IPv6 flow information */ 
    	struct in6_addr sin6_addr;     /* IPv6 address */ 
    	uint32_t        sin6_scope_id; /* Scope ID (new in 2.4) */ 
	};

	struct in6_addr { 
    	unsigned char   s6_addr[16];   /* IPv6 address */ 
	};  

Unic域地址结构:  

	#define UNIX_PATH_MAX    108
	struct sockaddr_un { 
    	sa_family_t sun_family;               /* AF_UNIX */ 
    	char        sun_path[UNIX_PATH_MAX];  /* pathname */ 
	};  

- addrlen：对应的是地址的长度。 
 
bind()函数把一个地址族中的特定地址赋给socket。例如对应AF_INET、AF_INET6就是把一个ipv4或ipv6地址和端口号组合赋给socket。  

listen()  

	int listen(int sockfd, int backlog);  

参数:  

- sockfd,要监听的socket描述字。
- backlog,socket可以排队的最大连接个数。  

如果作为一个服务器，在调用socket()、bind()之后就会调用listen()来监听这个socket，如果客户端这时调用connect()发出连接请求，服务器端就会接收到这个请求。socket()函数创建的socket默认是一个主动类型的，listen函数将socket变为被动类型的，等待客户的连接请求。  

connect()  

	int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);  

参数：   

- sockfd,客户端的socket描述字。
- addr,服务器的socket地址。
- addrlen,socket地址的长度。 
 
客户端通过调用connect函数来建立与TCP服务器的连接。  

accept()  

	int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);  

参数:  

- sockfd,服务器的socket描述字。
- addr,指向struct sockaddr *的指针，用于返回客户端的协议地址。
- 协议地址的长度。  

TCP服务器端依次调用socket()、bind()、listen()之后，就会监听指定的socket地址了。TCP客户端依次调用socket()、connect()之后就想TCP服务器发送了一个连接请求。TCP服务器监听到这个请求之后，就会调用accept()函数取接收请求，这样连接就建立好了。之后就可以开始网络I/O操作了。  

网络IO函数  

	#include <unistd.h>
	ssize_t read(int fd, void *buf, size_t count);
    ssize_t write(int fd, const void *buf, size_t count);

    #include <sys/types.h>
    #include <sys/socket.h>

    ssize_t send(int sockfd, const void *buf, size_t len, int flags);
    ssize_t recv(int sockfd, void *buf, size_t len, int flags);

    ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
    const struct sockaddr *dest_addr, socklen_t addrlen);
    ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
    struct sockaddr *src_addr, socklen_t *addrlen);

    ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
    ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);  

close()  

	int close(int fd);  

在服务器与客户端建立连接之后，会进行一些读写操作，完成了读写操作就要关闭相应的socket描述字，好比操作完打开的文件要调用fclose关闭打开的文件。  

**socket系统调用**  

Linux下，用户空间调用的socket API由glibc实现，glibc实现系统调用同名函数通常使用INT 0x80 + 系统调用号的方式陷入内核，一般来说，read对应sys_read,write对应sys_write.但是，socket系列却不是这样，为了节约系统调用号，将所有的socket系列的接口使用同一个系统调用号陷入内核，即socketcall:  

	SYSCALL_DEFINE2(socketcall, int, call, 
                   unsigned long __user *, args)
	{
		...
    	switch (call) {
    	case SYS_SOCKET:
        	err = sys_socket(a0, a1, a[2]);
        	break;
    	case SYS_BIND:
        	err = sys_bind(a0, (struct sockaddr __user *)a1, a[2]);
        	break;
    	case SYS_CONNECT:
        	err = sys_connect(a0, (struct sockaddr __user *)a1, a[2]);
        	break;
    	case SYS_LISTEN:
        	err = sys_listen(a0, a1);
        	break;
    	case SYS_ACCEPT:
        	err = sys_accept4(a0, (struct sockaddr __user *)a1,
                  (int __user *)a[2], 0);
        	break;
		...
	}  

socketcall函数用一个简单的逻辑分枝将不同的socket调用分发到对应的内核函数中。  

 
