---
layout:     post
title:      "webserver笔记(三）"
subtitle:   " \"one loop per thread\""
date:       2023-04-05 20:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(三)

> 多线程服务器常用编程模型
>
> * 每个请求创建一个线程，使用阻塞式IO操作。
> * 使用线程池，同样使用阻塞式IO操作
> * 使用non-blocking IO + IO multiplexing
> * leader/Foller等高级模式

## one loop per thread

此种模型下，程序里每个IO线程有一个event loop(或者叫Reactor)，用于处理读写和定时事件。

这种方式的好处是：

* 线程数目基本固定，可以在程序启动时设置，不会频繁创建和销毁
* 可以很方便地在线程间调配负载
* IO事件发生的线程是固定的，同一个TCP连接不必考虑事件并发

总结起来，这种C++多线程服务端编程模式为：one loop per thread + thread pool。

* event loop用作IO multiplexing,配合non-blocking IO和定时器。
* thread pool用来做计算，具体可以是任务队列或者生产者消费者队列。

> Linux能同时启动多少个线程？
>
> 对于32-bit Linux，一个进程的地址空间是4GiB，其中用户态能访问3GiB左右，而一个线程的默认栈大小为10MB,所以一个进程大约最多能同时启动300个线程。
>
> 对于64-bit Linux系统，线程数目可大大增加。

---

### Epoll类

> 该类主要是对epoll进行了封装，将事件的增加，更新，删除操作封装成函数，提高了代码的复用性，可读性
>
> Epoll是EventLoop的间接成员，只供其owner EventLoop在IO线程调用，因此无须加锁，其声明周期与EventLoop相等。

自定义的Epoll类包括以下成员：

> * epollfd，表示epoll文件描述符，它是一个与文件描述符相关联的事件通知对象，它可以监视多个文件描述符上的读、写和异常等时间。
> * vector<epoll_event> events,用一个vector容器来存储epoll_event对象
> * unordered_map<int,SP_Channel> channelmap,是一个epollfd到一个IO通道的映射

包括以下函数：

>* 构造函数和析构函数，分别用于创建和关闭epoll文件描述符
>* add、update、del函数，分别用于在epollfd中添加、更改、删除事件，并同时更新channelmap
>* poll函数，阻塞监听epollfd上的事件，并将这些事件添加到对应的IO channel，最后将这些Channel放到一个vector里面。

### Channel类

Channel类，即通道类，它负责注册读写事件，并保存了fd读写事件发生时调用的回调函数，如果Epoll有读写事件则将这些事件添加到对应的通道中。

> * 一个通道对应唯一EventLoop,因为一个Channel对象只属于某个IO线程，每个Channel对象自始至终只负责一个文件描述符的IO事件分发。
>
> * 一个EventLoop可以有多个通道
>
> * 当有fd返回读写事件时，调用提前注册的回调函数处理读写事件

其主要所示：

**注意**

channel的构造函数是传入一个EventLoop

```C++
class EventLoop;
typedef std::shared_ptr<EventLoop> SP_EventLoop;
typedef std::weak_ptr<EventLoop> WP_EventLoop;
typedef std::shared_ptr<SSL> SP_SSL;

// 描述事件循环中的一个文件描述符 (file descriptor) 对应的事件
class Channel{
private:
	typedef std::function<void()> CallBack;
	int fd;
	int events;
	int revents;
	bool deleted;
	bool First;
	SP_SSL ssl;
	bool sslconnect;
	WP_EventLoop loop; // 指向所属的 EventLoop 对象的弱智能指针；
	CallBack readhandler; // 读事件的回调函数
	CallBack writehandler; // 写事件的回调函数
	CallBack closehandler;	

public:
	Channel(SP_EventLoop Loop);
	~Channel();
	void setReadhandler(CallBack &&readHandler);
	void setWritehandler(CallBack &&writeHandler);
	void setClosehandler(CallBack &&closeHandler);
	void setDeleted(bool Deleted);
	void setssl(SP_SSL SSL);
	void setsslconnect(bool ssl_connect);
	void handleEvent();
	void handleClose();
	void setFd(int Fd);
	void setRevents(int Revents);
	void setEvents(int Events);
	void setnotFirst();
	bool isFirst();
	int getFd();
	int getRevents();
	bool isDeleted();
	SP_SSL getssl();
	bool getsslconnect();
	WP_EventLoop getLoop();
};

typedef std::shared_ptr<Channel> SP_Channel;
```

### EventLoop类

EventLoop主要职责如下：

1. 提供定时执行用户指定任务的方法，支持一次性、周期执行用户任务
2. 提供一个运行循环，每当Epoller监听到有通道对应事件发生时，会将通道加入激活通道列表，运行循环要不断从中取出激活通道，然后调用回调函数处理事件。
3. 每个EventLoop对应一个线程

主要成员函数在于loop函数

```C++
void EventLoop::loop(){
	wakeupchannel=SP_Channel(newElement<Channel>(shared_from_this()),deleteElement<Channel>);
	wakeupchannel->setFd(wakeupfd);
	wakeupchannel->setRevents(EPOLLIN|EPOLLET);
	// std::bind函数，第一个参数为要调用的函数，第二个参数为调用对象地址，类内成员函数都有一个隐藏参数:this
	wakeupchannel->setReadhandler(std::bind(&EventLoop::doPendingFunctors,shared_from_this()));
	addPoller(wakeupchannel);
	std::vector<SP_Channel> temp;
	while(!quit){
		poller->poll(temp);
		for(auto &ti:temp)
			ti->handleEvent();
		temp.clear();
		timermanager->handleExpiredEvent();
	}
}
```

该函数写了一个类似于死循环，一个EventLoop和一个Epoller对应。在循环中，从Epoller中获取事件，并进行处理。

在进入循环前，poller里面只有一个wakeupfd。

### 线程池

> 对于没有IO只有计算任务的线程，使用eventloop有点浪费，所以可以实现一个线程池类

ThreadpoolEventLoop类中，用一个vector<ThreadEventLoop\>作为线程池，之后用一个循环来构造ThreadEventLoop对象。

一个ThredEventLoop类，主要里面包含一个EventLoop成员和一个thread成员，在ThreadEventLoop初始化时，会分别进行loop和thread的初始化，thread初始化时，把类内的loop函数指针传入thread中，进行初始化。

```C++
ThreadEventLoop::ThreadEventLoop()
:	loop(newElement<EventLoop>(),deleteElement<EventLoop>),
	thread(newElement<Thread>(std::bind(&ThreadEventLoop::Loop,this)),deleteElement<Thread>)
{
	
}

void ThreadEventLoop::Loop(){
	loop->loop();
}

```

之后会再需要调用start函数，让线程中的每个线程工作

```C++
void ThreadpoolEventLoop::start(){
	for(auto &evi:elv)
		evi->start();
}

void ThreadEventLoop::start(){
	thread->start();
}

void Thread::start(){
	assert(!started_);
	started_=true;
	ThreadData *data=newElement<ThreadData>(func,name_);
	if(pthread_create(&pthreadId,NULL,&startThread,data)){
		started_=false;
		deleteElement<ThreadData>(data);
	}
}

void* startThread(void *obj){
	ThreadData* data=(ThreadData *)obj;
	data->runInThread();
	deleteElement<ThreadData>(data);
	return NULL;
}
```

> **注意**，这里的startThread是一个全局函数，应该类中的成员函数会自带一个this的参数，而pthread_create的第四个参数为void*类型。
>
> 或者，可以在类中定义一个static函数来实现