---
layout: post
title: libevent核心 - event
category: 网络
description:
---
**event结构体**   

Libevent是基于事件驱动（event-driven）的，从名字也可以看到event是整个库的核心。event就是Reactor框架中的事件处理程序组件；它提供了函数接口，供Reactor在事件发生时调用，以执行相应的事件处理，通常它会绑定一个有效的句柄。  
首先给出event结构体的定义，它位于event_struct.h文件中：  

	struct event {
	struct event_callback ev_evcallback;

	/* for managing timeouts */
	union {
		TAILQ_ENTRY(event) ev_next_with_common_timeout;
		int min_heap_idx;
	} ev_timeout_pos;
	evutil_socket_t ev_fd;

	struct event_base *ev_base;

	union {
		/* used for io events */
		struct {
			LIST_ENTRY (event) ev_io_next;
			struct timeval ev_timeout;
		} ev_io;

		/* used by signal events */
		struct {
			LIST_ENTRY (event) ev_signal_next;
			short ev_ncalls;
			/* Allows deletes in callback */
			short *ev_pncalls;
		} ev_signal;
	} ev_;

	short ev_events;
	short ev_res;		/* result passed to event callback */
	struct timeval ev_timeout;
	};  

下面简单解释一下结构体中各字段的含义。  

- ev_evcallback:event的回调函数，被ev_base调用，执行事件处理程序。  
- ev_timeout_pos:管理定时事件。  
- ev_base:该事件所属的反应堆实例，这是一个event_base结构体。  
- ev_:这是一个联合，里面包含两个结构体，ev_io和ev_signal。ev_io是该IO事件在IO事件列表中的位置，ev_signal是该signal事件在signal事件列表中的位置。信号和IO事件不能同时设置，因此它们在一个union中。  
- ev_events：event关注的事件类型，它可以是以下3种类型：  
	- I/O事件： EV_WRITE和EV_READ
	- 定时事件：EV_TIMEOUT
	- 信号：    EV_SIGNAL
	- 辅助选项：EV_PERSIST，表明是一个永久事件。  
- ev_res：记录了当前激活事件的类型。  
- ev_timeout: 如果是超时事件，它是event的超时值。  

**创建事件**  

使用event_new（）接口创建事件。  

	#define EV_TIMEOUT      0x01   
	#define EV_READ         0x02  
	#define EV_WRITE        0x04  
	#define EV_SIGNAL       0x08  
	#define EV_PERSIST      0x10  
	#define EV_ET           0x20  
	typedef void (*event_callback_fn)(evutil_socket_t, short, void *);  
	struct event *event_new(struct event_base *base, evutil_socket_t fd,short what, event_callback_fn cb,void *arg);  
	void event_free(struct event *event);  

event_new()试图分配和构造一个用于base的新的事件。what参数是上述标志的集合。如果fd非负，则它是将被观察其读写事件的文件。事件被激活时，libevent将调用cb函数，传递这些参数：文件描述符fd，表示所有被触发事件的位字段，以及构造事件时的arg参数。发生内部错误，或者传入无效参数时，event_new（）将返回NULL。  

**事件标志**  

- EV_TIMEOUT:这个标志表示某超时时间流逝后事件成为激活的。构造事件的时候，EV_TIMEOUT标志是被忽略的：可以在添加事件的时候设置超时，也可以不设置。超时发生时，回调函数的what参数将带有这个标志。  
- EV_READ:表示指定的文件描述符已经就绪，可以读取的时候，事件将成为激活的。  
- EV_WRITE:表示指定的文件描述符已经就绪，可以写入的时候，事件将成为激活的。  
- EV_SIGNAL:用于实现信号检测。  
- EV_PERSIST:表示事件是“持久的”。  
- EV_ET:表示如果底层的event_base后端支持边沿触发事件，则事件应该是边沿触发的。这个标志影响EV_READ和EV_WRITE的语义。  

**添加事件**  

构造事件之后，在将其添加到event_base之前实际上是不能对其做任何操作的。使用event_add（）将事件添加到event_base。  

	int event_add(struct event *ev, const struct timeval *tv); 
 
在非未决的事件上调用event_add（）将使其在配置的event_base中成为未决的。成功时函数返回0，失败时返回-1。如果tv为NULL，添加的事件不会超时。否则，tv以秒和微秒指定超时值。如果对已经未决的事件调用event_add（），事件将保持未决状态，并在指定的超时时间被重新调度。  

对已经初始化的事件调用event_del（）将使其成为非未决和非激活的。如果事件不是未决的或者激活的，调用将没有效果。成功时函数返回0，失败时返回-1。  

	int event_del(struct event *ev);  

**设置优先级**  

多个事件同时触发时，libevent没有定义各个回调的执行次序。可以使用优先级来定义某些事件比其他事件更重要。每个event_base有与之相关的一个或者多个优先级。在初始化事件之后，但是在添加到event_base之前，可以为其设置优先级。事件的优先级是一个在0和event_base的优先级减去1之间的数值。成功时函数返回0，失败时返回-1。多个不同优先级的事件同时成为激活的时候，低优先级的事件不会运行。libevent会执行高优先级的事件，然后重新检查各个事件。只有在没有高优先级的事件是激活的时候，低优先级的事件才会运行。  

	int event_priority_set(struct event *event, int priority);  

**检查事件状态**  

	int event_pending(const struct event *ev, short what, struct timeval *tv_out);
	#define event_get_signal(ev) /* ... */
	evutil_socket_t event_get_fd(const struct event *ev);
	struct event_base *event_get_base(const struct event *ev);
	short event_get_events(const struct event *ev);
	event_callback_fn event_get_callback(const struct event *ev);
	void *event_get_callback_arg(const struct event *ev);
	int event_get_priority(const struct event *ev);

	void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);  

event_pending（）函数确定给定的事件是否是未决的或者激活的。如果是，而且what参数设置了EV_READ、EV_WRITE、EV_SIGNAL或者EV_TIMEOUT等标志，则函数会返回事件当前为之未决或者激活的所有标志。如果提供了tv_out参数，并且what参数中设置了EV_TIMEOUT标志，而事件当前正因超时事件而未决或者激活，则tv_out会返回事件的超时值。event_get_fd（）和event_get_signal（）返回为事件配置的文件描述符或者信号值。event_get_base（）返回为事件配置的event_base。event_get_events（）返回事件的标志（EV_READ、EV_WRITE等）。event_get_callback（）和event_get_callback_arg（）返回事件的回调函数及其参数指针。 event_get_assignment（）复制所有为事件分配的字段到提供的指针中。任何为NULL的参数会被忽略。  

**配置一次触发事件**  

如果不需要多次添加一个事件，或者要在添加后立即删除事件，而事件又不需要是持久的，则可以使用event_base_once（）。   
 
	int event_base_once(struct event_base *, evutil_socket_t, short,
	void (*)(evutil_socket_t, short, void *), v		oid *, const struct timeval *);  

除了不支持EV_SIGNAL或者EV_PERSIST之外，这个函数的接口与event_new（）相同。安排的事件将以默认的优先级加入到event_base并执行。回调被执行后，libevent内部将会释放event结构。成功时函数返回0，失败时返回-1。不能删除或者手动激活使用event_base_once（）插入的事件：如果希望能够取消事件，应该使用event_new（）或者event_assign（）。  

**手动激活事件**  

极少数情况下，需要在事件的条件没有触发的时候让事件成为激活的。  

	void event_active(struct event *ev, int what, short ncalls);  

这个函数让事件ev带有标志what（EV_READ、EV_WRITE和EV_TIMEOUT的组合）成为激活的。事件不需要已经处于未决状态，激活事件也不会让它成为未决的。  


