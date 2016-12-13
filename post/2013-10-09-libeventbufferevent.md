---
layout: post
title: libevent之bufferevent
category: 网络
description:
---
**基本概念**  

很多时候，除了响应事件之外，应用还希望做一定的数据缓冲。比如说，写入数据的时候，通常的运行模式是：  

1. 决定要向连接写入一些数据，把数据放入到缓冲区中
2. 等待连接可以写入
3. 写入尽量多的数据
4. 记住写入了多少数据，如果还有更多数据要写入，等待连接再次可以写入  

这种缓冲IO模式很通用，libevent为此提供了一种通用机制，即bufferevent。bufferevent由一个底层的传输端口（如套接字），一个读取缓冲区和一个写入缓冲区组成。与通常的事件在底层传输端口已经就绪，可以读取或者写入的时候执行回调不同的是，bufferevent在读取或者写入了足够量的数据之后调用用户提供的回调。  
有多种共享公用接口的bufferevent类型：  

- 基于套接字的bufferevent：使用event_*接口作为后端，通过底层流式套接字发送或者接收数据的bufferevent  
- 异步IObufferevent：使用WindowsIOCP接口，通过底层流式套接字发送或者接收数据的bufferevent  
- 过滤型bufferevent：将数据传输到底层bufferevent对象之前，处理输入或者输出数据的bufferevent：比如说，为了压缩或者转换数据。  
- 成对的bufferevent：相互传输数据的两个bufferevent。  

每个bufferevent都有一个输入缓冲区和一个输出缓冲区，它们的类型都是“struct evbuffer。有数据要写入到bufferevent时，添加数据到输出缓冲区；bufferevent中有数据供读取的时候，从输入缓冲区抽取数据。  

每个bufferevent有两个数据相关的回调：一个读取回调和一个写入回调。默认情况下，从底层传输端口读取了任意量的数据之后会调用读取回调；输出缓冲区中足够量的数据被清空到底层传输端口后写入回调会被调用。通过调整bufferevent的读取和写入“水位（watermarks）”可以覆盖这些函数的默认行为。  
每个bufferevent有四个水位：  

- 读取低水位：读取操作使得输入缓冲区的数据量在此级别或者更高时，读取回调将被调用。默认值为0，所以每个读取操作都会导致读取回调被调用。  
- 读取高水位：输入缓冲区中的数据量达到此级别后，bufferevent将停止读取，直到输入缓冲区中足够量的数据被抽取，使得数据量低于此级别。默认值是无限，所以永远不会因为输入缓冲区的大小而停止读取。  
- 写入低水位：写入操作使得输出缓冲区的数据量达到或者低于此级别时，写入回调将被调用。默认值是0，所以只有输出缓冲区空的时候才会调用写入回调。  
- 写入高水位：bufferevent没有直接使用这个水位。它在bufferevent用作另外一个bufferevent的底层传输端口时有特殊意义。  

bufferevent也有“错误”或者“事件”回调，用于向应用通知非面向数据的事件，如连接已经关闭或者发生错误。定义了下列事件标志：  

- BEV_EVENT_READING：读取操作时发生某事件。  
- BEV_EVENT_WRITING：写入操作时发生某事件。  
- BEV_EVENT_ERROR：操作时发生错误。关于错误的更多信息，请调用EVUTIL_SOCKET_ERROR()。  
- BEV_EVENT_TIMEOUT：发生超时。  
- BEV_EVENT_EOF：遇到文件结束指示。  
- BEV_EVENT_CONNECTED：请求的连接过程已经完成。  

默认情况下，bufferevent的回调在相应的条件发生时立即被执行。在依赖关系复杂的情况下，这种立即调用会制造麻烦。比如说，假如某个回调在evbufferA空的时候向其中移入数据，而另一个回调在evbufferA满的时候从中取出数据。这些调用都是在栈上发生的，在依赖关系足够复杂的时候，有栈溢出的风险。要解决此问题，可以请求bufferevent（或者evbuffer）延迟其回调。条件满足时，延迟回调不会立即调用，而是在event_loop（）调用中被排队，然后在通常的事件回调之后执行。  

创建bufferevent时可以使用一个或者多个标志修改其行为。可识别的标志有：  

- BEV_OPT_CLOSE_ON_FREE：释放bufferevent时关闭底层传输端口。这将关闭底层套接字，释放底层bufferevent等。  
- BEV_OPT_THREADSAFE：自动为bufferevent分配锁，这样就可以安全地在多个线程中使用bufferevent。  
- BEV_OPT_DEFER_CALLBACKS：设置这个标志时，bufferevent延迟所有回调，如上所述。  
- BEV_OPT_UNLOCK_CALLBACKS：默认情况下，如果设置bufferevent为线程安全的，则bufferevent会在调用用户提供的回调时进行锁定。设置这个选项会让libevent在执行回调的时候不进行锁定。  

**基于套接字的bufferevent**  

基于套接字的bufferevent是最简单的，它使用libevent的底层事件机制来检测底层网络套接字是否已经就绪，可以进行读写操作，并且使用底层网络调用（如readv、writev、WSASend、WSARecv）来发送和接收数据。  

可以使用bufferevent_socket_new()创建基于套接字的bufferevent。  

	struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);  

base是event_base，options是表示bufferevent选项（BEV_OPT_CLOSE_ON_FREE等）的位掩码，fd是一个可选的表示套接字的文件描述符。如果想以后设置文件描述符，可以设置fd为-1。成功时函数返回一个bufferevent，失败则返回NULL。  

如果bufferevent的套接字还没有连接上，可以启动新的连接。  

	int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);  

address和addrlen参数跟标准调用connect()的参数相同。如果还没有为bufferevent设置套接字，调用函数将为其分配一个新的流套接字，并且设置为非阻塞的。如果已经为bufferevent设置套接字，调用bufferevent_socket_connect()将告知libevent套接字还未连接，直到连接成功之前不应该对其进行读取或者写入操作。连接完成之前可以向输出缓冲区添加数据。如果连接成功启动，函数返回0；如果发生错误则返回-1。  

常常需要将解析主机名和连接到主机合并成单个操作，libevent为此提供了：  

	int bufferevent_socket_connect_hostname(struct bufferevent *bev,
    struct evdns_base *dns_base, int family, const char *hostname,
    int port);
	int bufferevent_socket_get_dns_error(struct bufferevent *bev);  

这个函数解析名字hostname，查找其family类型的地址（允许的地址族类型有AF_INET,AF_INET6和AF_UNSPEC）。如果名字解析失败，函数将调用事件回调，报告错误事件。如果解析成功，函数将启动连接请求，就像bufferevent_socket_connect()一样。dns_base参数是可选的：如果为NULL，等待名字查找完成期间调用线程将被阻塞，而这通常不是期望的行为；如果提供dns_base参数，libevent将使用它来异步地查询主机名。  
跟bufferevent_socket_connect()一样，函数告知libevent，bufferevent上现存的套接字还没有连接，在名字解析和连接操作成功完成之前，不应该对套接字进行读取或者写入操作。函数返回的错误可能是DNS主机名查询错误，可以调用bufferevent_socket_get_dns_error()来获取最近的错误。返回值0表示没有检测到DNS错误。  

**过滤型bufferevent**  

有时候需要转换传递给某bufferevent的所有数据，这可以通过添加一个压缩层，或者将协议包装到另一个协议中进行传输来实现。  

	enum bufferevent_filter_result {
        BEV_OK = 0,
        BEV_NEED_MORE = 1,
        BEV_ERROR = 2
	};
	typedef enum bufferevent_filter_result (*bufferevent_filter_cb)(
    struct evbuffer *source, struct evbuffer *destination, ev_ssize_t dst_limit,
    enum bufferevent_flush_mode mode, void *ctx);

	struct bufferevent *bufferevent_filter_new(struct bufferevent *underlying,
        bufferevent_filter_cb input_filter,
        bufferevent_filter_cb output_filter,
        int options,
        void (*free_context)(void *),
        void *ctx);  

有时候在给出了对的一个成员时，需要获取另一个成员，这时候可以使用bufferevent_pair_get_partner()。如果bev是对的成员，而且对的另一个成员仍然存在，函数将返回另一个成员；否则，函数返回NULL。  
过滤型bufferevent有时候需要转换传递给某bufferevent的所有数据，这可以通过添加一个压缩层，或者将协议包装到另一个协议中进行传输来实现。  
接口bufferevent_filter_new（）函数创建一个封装现有的底层bufferevent的过滤bufferevent。所有通过底层bufferevent接收的数据在到达过滤bufferevent之前都会经过输入过滤器的转换；所有通过底层bufferevent发送的数据在被发送到底层bufferevent之前都会经过输出过滤器的转换。向底层bufferevent添加过滤器将替换其回调函数。可以向底层bufferevent的evbuffer添加回调函数，但是如果想让过滤器正确工作，就不能再设置bufferevent本身的回调函数。  

**成对的bufferevent**  

有时候网络程序需要与自身通信。比如说，通过某些协议对用户连接进行隧道操作的程序，有时候也需要通过同样的协议对自身的连接进行隧道操作。当然，可以通过打开一个到自身监听端口的连接，让程序使用这个连接来达到这种目标。但是，通过网络栈来与自身通信比较浪费资源。替代的解决方案是，创建一对成对的bufferevent。这样，写入到一个bufferevent的字节都被另一个接收（反过来也是），但是不需要使用套接字。  

	int bufferevent_pair_new(struct event_base *base, int options,
    struct bufferevent *pair[2]);  

调用bufferevent_pair_new()会设置pair[0]和pair[1]为一对相互连接的bufferevent。除了BEV_OPT_CLOSE_ON_FREE无效、BEV_OPT_DEFER_CALLBACKS总是打开的之外，所有通常的选项都是支持的。  

	struct bufferevent *bufferevent_pair_get_partner(struct bufferevent *bev)  

有时候在给出了对的一个成员时，需要获取另一个成员，这时候可以使用bufferevent_pair_get_partner()。如果bev是对的成员，而且对的另一个成员仍然存在，函数将返回另一个成员；否则，函数返回NULL。  

**通用bufferevent操作**  

	void bufferevent_free(struct bufferevent *bev);  

这个函数释放bufferevent。bufferevent内部具有引用计数，所以，如果释放bufferevent时还有未决的延迟回调，则在回调完成之前bufferevent不会被删除。如果设置了BEV_OPT_CLOSE_ON_FREE标志，并且bufferevent有一个套接字或者底层bufferevent作为其传输端口，则释放bufferevent将关闭这个传输端口。  

	typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
	typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);

	void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);

	void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);  

bufferevent_setcb()函数修改bufferevent的一个或者多个回调。readcb、writecb和eventcb函数将分别在已经读取足够的数据、已经写入足够的数据，或者发生错误时被调用。每个回调函数的第一个参数都是发生了事件的bufferevent，最后一个参数都是调用bufferevent_setcb()时用户提供的cbarg参数：可以通过它向回调传递数据。事件回调的events参数是一个表示事件标志的位掩码。  

	void bufferevent_enable(struct bufferevent *bufev, short events);
	void bufferevent_disable(struct bufferevent *bufev, short events);

	short bufferevent_get_enabled(struct bufferevent *bufev);  

可以启用或者禁用bufferevent上的EV_READ、EV_WRITE或者EV_READ|EV_WRITE事件。没有启用读取或者写入事件时，bufferevent将不会试图进行数据读取或者写入。没有必要在输出缓冲区空时禁用写入事件：bufferevent将自动停止写入，然后在有数据等待写入时重新开始。类似地，没有必要在输入缓冲区高于高水位时禁用读取事件：bufferevent将自动停止读取，然后在有空间用于读取时重新开始读取。默认情况下，新创建的bufferevent的写入是启用的，但是读取没有启用。可以调用bufferevent_get_enabled()确定bufferevent上当前启用的事件。  

	void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);  

bufferevent_setwatermark()函数调整单个bufferevent的读取水位、写入水位，或者同时调整二者。（如果events参数设置了EV_READ，调整读取水位。如果events设置了EV_WRITE标志，调整写入水位）  

如果只是通过网络读取或者写入数据，而不能观察操作过程，是没什么好处的。bufferevent提供了下列函数用于观察要写入或者读取的数据。  

	struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
	struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);  

这两个函数提供了非常强大的基础：它们分别返回输入和输出缓冲区。  

	int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
	int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);  

这些函数向bufferevent的输出缓冲区添加数据。bufferevent_write()将内存中从data处开始的size字节数据添加到输出缓冲区的末尾。bufferevent_write_buffer()移除buf的所有内容，将其放置到输出缓冲区的末尾。成功时这些函数都返回0，发生错误时则返回-1。  

	size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
	int bufferevent_read_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);  

这些函数从bufferevent的输入缓冲区移除数据。bufferevent_read()至多从输入缓冲区移除size字节的数据，将其存储到内存中data处。函数返回实际移除的字节数。bufferevent_read_buffer()函数抽空输入缓冲区的所有内容，将其放置到buf中，成功时返回0，失败时返回-1。  

跟其他事件一样，可以要求在一定量的时间已经流逝，而没有成功写入或者读取数据的时候调用一个超时回调。  

	void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);  

试图读取数据的时候，如果至少等待了timeout_read秒，则读取超时事件将被触发。试图写入数据的时候，如果至少等待了timeout_write秒，则写入超时事件将被触发。  

	int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);  

清空bufferevent要求bufferevent强制从底层传输端口读取或者写入尽可能多的数据，而忽略其他可能保持数据不被写入的限制条件。函数的细节功能依赖于bufferevent的具体类型。  

**类型特定的bufferevent操作**  

这些bufferevent函数不能支持所有bufferevent类型。  

	int bufferevent_priority_set(struct bufferevent *bufev, int pri);
	int bufferevent_get_priority(struct bufferevent *bufev);  

设置与获取bufev的优先级。  

	int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
	evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);  

这些函数设置或者返回基于fd的事件的文件描述符。只有基于套接字的bufferevent支持setfd()。两个函数都在失败时返回-1；setfd()成功时返回0。  

	struct event_base *bufferevent_get_base(struct bufferevent *bev);  

这个函数返回bufferevent的event_base。  

	struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);  

这个函数返回作为bufferevent底层传输端口的另一个bufferevent。  

**手动锁定和解锁**  

有时候需要确保对bufferevent的一些操作是原子地执行的。为此，libevent提供了手动锁定和解锁bufferevent的函数。  

	void bufferevent_lock(struct bufferevent *bufev);
	void bufferevent_unlock(struct bufferevent *bufev);  

用这个函数锁定bufferevent将自动同时锁定相关联的evbuffer。这些函数是递归的：锁定已经持有锁的bufferevent是安全的。当然，对于每次锁定都必须进行一次解锁。  

