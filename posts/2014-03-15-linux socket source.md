---
layout: post
title: 从内核角度认识socket
category: linux
description:
---
**socket与文件系统**  

Linux内核为所有的与socket有关的操作的API，提供了一个统一的系统调用入口sys_socketcall，参见[linux socket 编程基础](http://chencheng.sn.cn/categories/linux/linux-socket-base)。对于用户态而言，一个socket，就是一个特殊的，已经打开的文件。为了对socket抽像出文件的概念，内核中为socket定义了一个专门的文件系统类型sockfs:  

	static struct vfsmount *sock_mnt; 
	static struct file_system_type sock_fs_type = {
         .name =           "sockfs",
         .get_sb =         sockfs_get_sb,
         .kill_sb =        kill_anon_super,
	 };  

在系统初始化的时候，安装该文件系统。创建一个socket，就是在sockfs文件系统中创建一个文件节点(inode)，并建立起为了实现socket功能所需的一整套数据结构，包括struct inode、struct file、struct socket等。当一个struct socket数据结构被分配空间后，再将其与一个已打开的文件建立映射关系。这样，用户态就可以用抽像的文件的概念来操作socket了。  
inode与socket是如何关联的呢？它们通过struct socket_alloc封装:  

	struct socket_alloc {
         struct socket socket;
         struct inode vfs_inode;
	 };  

struct socket结构:  

	struct socket {
          socket_state                state;
 		  unsigned long				  flags;
          struct proto_ops        	  *ops;
          struct fasync_struct        *fasync_list;
          struct file                 *file;
          struct sock                 *sk;
          wait_queue_head_t        	  wait;
          short                       type;
	};  

在内核中，用struct file结构描述一个已经打开的文件，要把一个socket与一个已打开的文件建立映射，首先要做的就是为socket分配一个struct file，并申请分配一个相应的文件描述符fd。然后通过目录项struct dentry把两者关联起来:  

	struct file {
          struct list_head        f_list;
          struct dentry           *f_dentry;
		  struct file_operations  f_op;
		  ......
	}  
	
	struct dentry {
		  ......
		  struct inode *d_inode;
	}  

d_inode成员指向了与之对应的inode节点，而之前已经创建了一个inode节点和与之对应的socket。   

对于socket有两个重要的函数集指针一个是文件系统struct file中的f_op指针，它指向了对应的用户态调用的read,write等操调用，但不支持open:  

	static struct file_operations socket_file_ops = {
		 .owner = 		  	THIS_MODULE,
         .llseek =        	no_llseek,
         .aio_read =      	sock_aio_read,
         .aio_write =     	sock_aio_write,
         .poll =          	sock_poll,
         .unlocked_ioctl =	sock_ioctl,
  		 .mmap = 		  	sock_mmap,
         .open =          	sock_no_open,
         .release =         sock_close,
         .fasync =        	sock_fasync,
         .readv =        	sock_readv,
         .writev =        	sock_writev,
         .sendpage =        sock_sendpage
	};   

另一个是 struct socket 结构sock的ops指针， 指向具体协议类型的 ops, 例如，inet_stream_ops、inet_dgram_ops 或者是inet_sockraw_ops等等。它用来支持上层的socket的其它API调用。sys_socketcall中的函数都是通过调用sock的ops指向的函数集来完成socket的底层操作。  

**协议介绍**  

内核中有多种协议族，INET只是其中的一种：  

	#define AF_UNSPEC   0  
	#define AF_UNIX     1   /* Unix domain sockets      */  
	#define AF_LOCAL    1   /* POSIX name for AF_UNIX   */  
	#define AF_INET     2   /* Internet IP Protocol     */  
	#define AF_IPX      4   /* Novell IPX           */  
	#define AF_X25      9   /* Reserved for X.25 project    */  
	#define AF_INET6    10  /* IP version 6         */  
	#define AF_DECnet   12  /* Reserved for DECnet project  */  
	#define AF_NETBEUI  13  /* Reserved for 802.2LLC project*/  
	#define AF_IRDA     23  /* IRDA sockets         */  
	#define AF_PPPOX    24  /* PPPoX sockets        */  
	#define AF_LLC      26  /* Linux LLC            */  
	#define AF_CAN      29  /* Controller Area Network      */  
	#define AF_BLUETOOTH    31  /* Bluetooth sockets        */  
	#define AF_ISDN     34  /* mISDN sockets        */  
	#define AF_PHONET   35  /* Phonet sockets       */  
	#define AF_IEEE802154   36  /* IEEE802154 sockets       */  
	#define AF_NFC      39  /* NFC sockets          */  
	#define AF_MAX      40  /* For now.. */
	......  

INET协议族对应该结构的定义如下：  

	static const struct net_proto_family inet_family_ops = {  
    	.family = PF_INET,  
    	.create = inet_create,  
    	.owner  = THIS_MODULE,  
	};  

 
内核用static struct inet_protosw inetsw_array[]结构维护AF_INET协议族下支持的协议类型。数据结构struct inet_protosw封装了一个socket协议类型（如SOCK_STREAM、SOCK_DGRAM等）与ip协议中对应的传输层协议:  

	struct inet_protosw {
          	struct list_head 			list;
			unsigned short         		type;
            int                 		protocol;
            struct proto         		*prot;
            struct proto_ops 			*ops;
            int              			capability;
            char             			no_check;
            unsigned char         		flags;
      };   

type是协议类型，对于ipv4而言，就是SOCK_STREAM、SOCK_DGRAM或者是SOCK_RAW之一。protocol是传输层的协议号。 proto用于描述一个具体的传输层协议，而ops指向当前协议族的操作集。 
 
下面是协议族操作集结构体定义：  

	struct proto_ops {  
    int     family;  
    struct module   *owner;  
    int     (*release)   (struct socket *sock);  
    int     (*bind)      (struct socket *sock,  
                      struct sockaddr *myaddr,  
                      int sockaddr_len);  
    int     (*connect)   (struct socket *sock,  
                      struct sockaddr *vaddr,  
                      int sockaddr_len, int flags);  
    int     (*socketpair)(struct socket *sock1,  
                      struct socket *sock2);  
    int     (*accept)    (struct socket *sock,  
                      struct socket *newsock, int flags);  
    int     (*getname)   (struct socket *sock,  
                      struct sockaddr *addr,  
                      int *sockaddr_len, int peer);  
    unsigned int    (*poll)      (struct file *file, struct socket *sock,  
                      struct poll_table_struct *wait);  
    int     (*ioctl)     (struct socket *sock, unsigned int cmd,  
                      unsigned long arg);  
    int     (*listen)    (struct socket *sock, int len);  
    int     (*shutdown)  (struct socket *sock, int flags);  
    int     (*setsockopt)(struct socket *sock, int level,  
                      int optname, char __user *optval, unsigned int optlen);  
    int     (*getsockopt)(struct socket *sock, int level,  
                      int optname, char __user *optval, int __user *optlen);  
    int     (*sendmsg)   (struct kiocb *iocb, struct socket *sock,  
                      struct msghdr *m, size_t total_len);  
    int     (*recvmsg)   (struct kiocb *iocb, struct socket *sock,  
                      struct msghdr *m, size_t total_len,  
                      int flags);  
    int     (*mmap)      (struct file *file, struct socket *sock,  
                      struct vm_area_struct * vma);  
    ssize_t     (*sendpage)  (struct socket *sock, struct page *page,  
                      int offset, size_t size, int flags);  
};  

针对不同的socket类型，定义了不同的ops：  

	struct proto_ops inet_stream_ops = { 
        .family =       PF_INET,
		.owner =		THIS_MODULE,
	    .release =      inet_release,
	    .bind =         inet_bind,
        .connect =      inet_stream_connect,
        .socketpair =   sock_no_socketpair,
        .accept =       inet_accept,
        .getname =      inet_getname,
        .poll =         tcp_poll,
        .ioctl =        inet_ioctl,
        .listen =       inet_listen,
        .shutdown =     inet_shutdown,
        .setsockopt =   sock_common_setsockopt,
        .getsockopt =   sock_common_getsockopt,
        .sendmsg =      inet_sendmsg,
        .recvmsg =      sock_common_recvmsg,
	 	.mmap = 		sock_no_mmap,
        .sendpage =     tcp_sendpage
	};

	struct proto_ops inet_dgram_ops = {
        .family =       PF_INET,
		.owner = 		THIS_MODULE,
        .release =      inet_release,
        .bind =         inet_bind,
        .connect =      inet_dgram_connect,
        .socketpair =   sock_no_socketpair,
        .accept =       sock_no_accept, 
        .getname =      inet_getname, 
        .poll =         udp_poll, 
        .ioctl =        inet_ioctl, 
        .listen =       sock_no_listen, 
        .shutdown =     inet_shutdown, 
        .setsockopt =   sock_common_setsockopt, 
        .getsockopt =   sock_common_getsockopt, 
        .sendmsg =      inet_sendmsg, 
        .recvmsg =      sock_common_recvmsg,
	    .mmap = 		sock_no_mmap, 
        .sendpage =     inet_sendpage,
	};  
 
	static struct proto_ops inet_sockraw_ops = { 
        .family =        PF_INET,
	    .owner = 		 THIS_MODULE, 
        .release =       inet_release, 
        .bind =          inet_bind, 
        .connect =       inet_dgram_connect, 
        .socketpair =    sock_no_socketpair, 
        .accept =        sock_no_accept, 
        .getname =       inet_getname, 
        .poll =          datagram_poll, 
        .ioctl =         inet_ioctl, 
        .listen =        sock_no_listen, 
        .shutdown =      inet_shutdown, 
        .setsockopt =    sock_common_setsockopt, 
        .getsockopt =    sock_common_getsockopt, 
        .sendmsg =       inet_sendmsg, 
        .recvmsg =       sock_common_recvmsg,
	    .mmap = 		 sock_no_mmap, 
        .sendpage =      inet_sendpage,
	};  

inet_stream_ops、inet_dgram_ops、inet_sockraw_ops属于协议族层次。  

下面是传输层的协议操作集结构体定义：

	struct proto {
         void                       (*close)(struct sock *sk,long timeout);
         int                        (*connect)(struct sock *sk,struct sockaddr *uaddr,int addr_len); 
         int                        (*disconnect)(struct sock *sk, int flags); 
         struct sock *              (*accept) (struct sock *sk, int flags, int *err); 
         int                        (*ioctl)(struct sock *sk, int cmd,unsigned long arg);
         int                        (*init)(struct sock *sk); 
         int                        (*destroy)(struct sock *sk); 
         void                       (*shutdown)(struct sock *sk, int how);
         int                        (*setsockopt)(struct sock *sk, int level,int optname,char __user *optval,int optlen); 
         int                        (*getsockopt)(struct sock *sk, int level,int optname,char __user *optval,int__user*option);
		 int						(*sendmsg)(struct kiocb*iocb,struct sock*sk,struct msghdr *msg,size_t len);
		 int						(*recvmsg)(struct kiocb*iocb,struct sock*sk,struct msghdr *msg,size_t len,int noblock, int flags,int *addr_len);
		 int						(*sendpage)(struct sock*sk,struct page* page,int offset, size_t size, int flags); 
         int                        (*bind)(struct sock *sk,struct sockaddr *uaddr, int addr_len); 
         int                        (*backlog_rcv) (struct sock *sk,struct sk_buff *skb);
         void                       (*hash)(struct sock *sk); 
         void                       (*unhash)(struct sock *sk);
         int                        (*get_port)(struct sock *sk, unsigned short snum);
         void                       (*enter_memory_pressure)(void); 
         atomic_t                	*memory_allocated;
         atomic_t                	*sockets_allocated;
         int                        *memory_pressure;
         int                        *sysctl_mem;
         int                        *sysctl_wmem;
         int                        *sysctl_rmem;
         int                        max_header;
         kmem_cache_t               *slab; 
		 unsigned int 				obj_size;
         struct module              *owner; 
         char                       name[32];
         struct list_head        	node;
         struct {
                  int inuse;
                  u8  __pad[SMP_CACHE_BYTES - sizeof(int)];
         } stats[NR_CPUS];
	};   

该结构体和proto_ops的区别是：该结构体和具体的传输层协议相关，其中的函数指针指向对应的协议的相应的操作函数。  

TCP协议的操作集定义如下：  

	struct proto tcp_prot = {  
    .name           = "TCP",  
    .owner          = THIS_MODULE,  
    .close          = tcp_close,  
    .connect        = tcp_v4_connect,  
    .disconnect     = tcp_disconnect,  
    .accept         = inet_csk_accept,  
    .ioctl          = tcp_ioctl,  
    .init           = tcp_v4_init_sock,  
    .destroy        = tcp_v4_destroy_sock,  
    .shutdown       = tcp_shutdown,  
    .setsockopt     = tcp_setsockopt,  
    .getsockopt     = tcp_getsockopt,  
    .recvmsg        = tcp_recvmsg,  
    .sendmsg        = tcp_sendmsg,  
    .sendpage       = tcp_sendpage,  
    .backlog_rcv        = tcp_v4_do_rcv,  
    .hash           = inet_hash,  
    .unhash         = inet_unhash,  
    .get_port       = inet_csk_get_port,  
    .enter_memory_pressure  = tcp_enter_memory_pressure,  
    .sockets_allocated  = &tcp_sockets_allocated,  
    .orphan_count       = &tcp_orphan_count,  
    .memory_allocated   = &tcp_memory_allocated,  
    .memory_pressure    = &tcp_memory_pressure,  
    .sysctl_mem     = sysctl_tcp_mem,  
    .sysctl_wmem        = sysctl_tcp_wmem,  
    .sysctl_rmem        = sysctl_tcp_rmem,  
    .max_header     = MAX_TCP_HEADER,  
    .obj_size       = sizeof(struct tcp_sock),  
    .slab_flags     = SLAB_DESTROY_BY_RCU,  
    .twsk_prot      = &tcp_timewait_sock_ops,  
    .rsk_prot       = &tcp_request_sock_ops,  
    .h.hashinfo     = &tcp_hashinfo,  
    .no_autobind        = true,  
	};  

UDP协议的操作集则为：  

	struct proto udp_prot = {  
    .name          = "UDP",  
    .owner         = THIS_MODULE,  
    .close         = udp_lib_close,  
    .connect       = ip4_datagram_connect,  
    .disconnect    = udp_disconnect,  
    .ioctl         = udp_ioctl,  
    .destroy       = udp_destroy_sock,  
    .setsockopt    = udp_setsockopt,  
    .getsockopt    = udp_getsockopt,  
    .sendmsg       = udp_sendmsg,  
    .recvmsg       = udp_recvmsg,  
    .sendpage      = udp_sendpage,  
    .backlog_rcv       = __udp_queue_rcv_skb,  
    .hash          = udp_lib_hash,  
    .unhash        = udp_lib_unhash,  
    .rehash        = udp_v4_rehash,  
    .get_port      = udp_v4_get_port,  
    .memory_allocated  = &udp_memory_allocated,  
    .sysctl_mem    = sysctl_udp_mem,  
    .sysctl_wmem       = &sysctl_udp_wmem_min,  
    .sysctl_rmem       = &sysctl_udp_rmem_min,  
    .obj_size      = sizeof(struct udp_sock),  
    .slab_flags    = SLAB_DESTROY_BY_RCU,  
    .h.udp_table       = &udp_table,  
    .clear_sk      = sk_prot_clear_portaddr_nulls,  
	};    
 
**socket内核结构**  

每一个socket套接字，都有一个对应的struct socket结构来描述(内核中一般使用名称为sock)，但是同时又有一个struct sock结构（内核中一般使用名称为sk），两者之间是一一对应的关系。socket结构和sock结构实际上是同一个事物的两个方面。socket结构是面向进程和系统调用界面的侧面，而sock结构则是面向底层驱动程序的侧面。设计者把socket套接字中，与文件系统关系比较密切的那一部份放在socket结构中，而把与通信关系比较密切的那一部份，则单独成为一个数结结构，那就是sock结构。由于这两部份逻辑上本来就是一体的，所以要通过指针互相指向对方，形成一对一的关系。  

sock结构中，有三个重要的双向队列，分别是sk_receive_queue、sk_write_queue和sk_error_queue。队列并非采用通用的list_head来维护，而是使用skb_buffer队列：  

	struct sk_buff_head {
         struct sk_buff        *next;
         struct sk_buff        *prev;
         __u32                  qlen;
         spinlock_t        		lock;
	};  

队列中指向的每一个skb_buffer，就是一个数据包，分别是接收、发送和投递错误。  
 
sk是如何与协议栈交互的呢？对于每一个类型的协议，为了与sk联系起来，都定义了一个struct XXX_sock结构，XXX是协议名，例如:  

	struct tcp_sock {
         struct inet_connection_sock        inet_conn;
		 int					 			tcp_header_len;
		 ......
	};  

	struct udp_sock {
         struct 		inet_sock inet;
	  	 int			pending;
         unsigned int   corkflag;
         __u16          encap_type;
         __u16          len;
	};   

	struct raw_sock {
         struct inet_sock   inet;
         struct icmp_filter filter;
	};  

inet是一个struct inet_sock结构类型，包括AF_INET的一般属性：   

	struct inet_sock {
         struct sock         sk;
         ......
	};  

最后对内核中出现的套接字结构做个总结：  

- struct socket，这个是BSD层的socket，应用程序会用过系统调用首先创建该类型套接字，它和具体协议无关。
- struct inet_sock，是INET协议族使用的socket结构，可以看成位于INET层，是struct sock的一个扩展。它的第一个属性就是struct sock结构。
- struct sock，是与具体传输层协议相关的套接字，所有内核的操作都基于这个套接字。
- struct tcp_sock ， 是TCP协议的套接字表示 ， 它是对struct inet_connection_sock的扩展 ， 其第一个属性就是        struct inet_connection_sock inet_conn。
- struct raw_sock，是原始类型的套接字表示，ICMP协议就使用这种套接字，其是对struct sock的扩展。
- struct udp_sock，是UDP协议套接字表示，其是对struct inet_sock套接字的扩展。
- struct inet_connetction_sock，是所有面向连接协议的套接字，是对struct inet_sock套接字扩展。  


