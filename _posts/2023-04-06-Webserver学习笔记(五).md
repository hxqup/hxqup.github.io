---
layout:     post
title:      "webserver笔记(五）"
subtitle:   " \"异步日志\""
date:       2023-04-06 15:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---


# Webserver学习笔记(五)

> **日志**，由服务器自动创建，并记录运行状态，错误信息，访问数据的文件。
>
> **异步日志**：将所写的日志内容先存入阻塞队列，写线程从阻塞队列中取出内容，写入日志。

## 日志的实现

### FixedBuffer

创建一个buffer类，用于存放日志数据。

```C++
template<int SIZE>
class FixedBuffer:noncopyable{
private:
    const char* end()const;
    char data_[SIZE];
    char* cur;

public:
    FixedBuffer():	cur(data_){}
    ~FixedBuffer(){}
    void append(const char *buf,int len){
		if(avail()>len){
			memcpy(cur,buf,len);
			cur+=len;
		}
	}
    const char* data()const{return data_;}
    int length()const{return cur-data_;}
    char* current(){return cur;}
    int avail()const{return data_+SIZE-cur;}
    void add(int len){cur+=len;}
    void reset(){cur=data_;}
    void bzero(){memset(data_,0,SIZE);}
};
```

这段代码中，end函数用于获取当前缓冲区的末尾，data是用于存放数据的实际容器，而cur则指出了当前所记录到的位置。

> 非类型模板参数
>
> 非类型模板参数是指在C++的模板中，可以使用不仅仅是类型作为参数，还可以使用常量表达式作为参数。这种参数被称为非类型模板参数，也叫做数值模板参数。使用非类型模板参数可以将一些常量或大小信息编译到程序中，从而提高程序的效率和安全性。

---

>Buffer和cache的区别
>
>Buffer和Cache都是常见的计算机术语，用于描述数据在内存中的存储和访问方式。
>
>Buffer通常是指一个临时的数据缓冲区，用于暂存输入或输出数据。在网络编程中，Buffer通常用于存储接收到的网络数据或需要发送的网络数据。例如，可以使用一个读写缓冲区来加速Socket网络通信。当从网络中读取数据时，可以先将读取到的数据存储到该缓冲区中，然后再对缓冲区中的数据进行解析和处理。
>
>Cache则是指一个高速缓存存储区，用于存储经常访问的数据。在计算机系统中，Cache通常用于提高数据访问效率，减少对主存储器的访问次数。例如，CPU中的缓存用于缓存常用的指令和数据，以便快速执行程序；Web服务器中的缓存用于缓存静态网页、图片等资源，以便提高用户访问速度。
>
>虽然Buffer和Cache都是存储数据的缓冲区，但它们的作用和使用场景有所不同。Buffer通常用于临时存储数据，以便进行进一步的处理或传输；而Cache则用于存储经常访问的数据，以便在需要时能够快速访问。另外，Buffer通常是单向的，即只用于输入或输出数据，而Cache则可以是双向的，既可以用于缓存读取的数据，也可以用于缓存写入的数据。

### LogStream类

```C++
class LogStream:noncopyable{
public:
	typedef FixedBuffer<kSmallBuffer> Buffer;
private:
	void staticCheck();
	template<typename T>
	void formatInteger(T);
	template<typename T>
	void formatDecimal(T);
	Buffer buffer_;
	static const int kMaxNumbericSize=32;

public:
	LogStream& operator<<(bool);
	LogStream& operator<<(short);
	LogStream& operator<<(unsigned short);
	LogStream& operator<<(int);
	LogStream& operator<<(unsigned int);
	LogStream& operator<<(long);
	LogStream& operator<<(unsigned long);
	LogStream& operator<<(long long);
	LogStream& operator<<(unsigned long long);
	LogStream& operator<<(float);
	LogStream& operator<<(double);
	LogStream& operator<<(long double);
	LogStream& operator<<(char);
	LogStream& operator<<(const char*);
	LogStream& operator<<(const std::string&);
	void append(const char *data,int len);
	const Buffer& buffer()const;
	void resetBuffer();
};
```

这个类主要就是重载了很多遍，对于各种数据类型都有响应的处理函数。

重点可以看下浮点数与字符串之间的转换：

```C++
template<typename T>
void LogStream::formatDecimal(T v){
	if(buffer_.avail()>=kMaxNumbericSize){
		int len=snprintf(buffer_.current(),kMaxNumbericSize,"%.12g",v);
		buffer_.add(len);
	}
}
```

### FileUtil类

这个类主要涉及到一些文件操作。主要用于将缓冲区中的数据写入文件

```C++
class FileUtil:noncopyable{
private:
	int write(const char *logline,int len);
	FILE *fp;
	char buffer[BUFFERSIZE];	
	
public:
	explicit FileUtil(std::string filename);
	~FileUtil();
	void append(const char *logline,int len);
	void flush();
};
```

> 介绍一些其中用到的一些文件操作函数：
>
> * setbuffer
>
> ```c++
> void setbuffer(FILE * stream,char * buf,size_t size);
> ```
>
> 用于设置文件流的缓冲区
>
> * fwrite_unlocked函数，相当于fwrite的无锁版本，效率更高
>
> ```c++
> size_t fwrite_unlocked(const void *ptr, size_t size, size_t count, FILE *stream);
> ```
>
> > 各个参数的意义如下：
> >
> > 1. const void *ptr：指向要写入的数据缓冲区的指针。可以是任何类型的指针，但由于是二进制写入，通常使用void类型指针。
> > 2. size_t size：每个数据项的大小（以字节为单位）。例如，如果要写入一个int类型的数组，则size应该设置为sizeof(int)。
> > 3. size_t count：要写入的数据项数量。例如，如果要写入10个int类型的数据，则count应该设置为10。
> > 4. FILE *stream：指向要写入数据的文件指针。可以是标准输出、标准错误或用户自定义的文件指针。
> >
> > 函数返回值为成功写入的数据项数量。如果返回值小于count，则表示可能发生了写入错误或者到达了文件结尾。

### LogFile类

```C++
class LogFile:noncopyable{
private:
	void append_unlocked(const char *logline,int len);
	const std::string basename;
	const int flushEveryN;
	int count;
	MutexLock mutex;
	UP_FileUtil file;

public:
	LogFile(const std::string &Basename,int FlushEveryN=1024);
	~LogFile();
	void append(const char *logline,int len);
	void flush();
};
```

这个类基本上主要是实现了将文件flush到磁盘中这一功能，设置了一个数FlushEveryN，当写入文件的次数达到这一标准，就将文件同步到磁盘中

### AsyncLogging类

```C++
class AsyncLogging:noncopyable{
private:
	void threadFunc();
	typedef FixedBuffer<kLargeBuffer> Buffer;
	typedef std::vector<std::shared_ptr<Buffer>> BufferVector;
	typedef std::shared_ptr<Buffer> BufferPtr;
	const int flushInterval;
	bool running;
	std::string basename;
	Thread thread_;
	MutexLock mutex;
	Condition cond;
	BufferPtr currentBuffer;
	BufferPtr nextBuffer;
	BufferVector buffers;
	//CountDownLatch latch;

public:
	AsyncLogging(const std::string logFileName,int FlushInterval=2);
	~AsyncLogging();
	void append(const char *logfile,int len);
	void start();
	void stop();	
};
```

这个类就正式开始写日志了，有一个线程专门用来写日志。

> 主要利用了双缓冲区技术
>
> 双缓冲区技术的基本思想是，使用两个缓冲区来分别存储输入/输出数据和当前正在显示的数据。在处理过程中，程序将输入/输出数据写入到一个缓冲区中，并在缓冲区填满或需要显示时，将该缓冲区与显示缓冲区交换，以便实现快速、平滑地更新显示内容。在该服务器中，每个线程都有一个前端和后端两个日志缓冲区。前端缓冲区用于接收日志信息，一旦缓冲区满了或者到达指定的时间间隔，就会将当前的缓冲区与后端缓冲区交换，从而将缓存的日志信息写入到磁盘中。这样可以避免频繁地向磁盘中写入日志，提高日志系统的效率。

其主要函数实现如下：

```C++
void AsyncLogging::append(const char *logline,int len){
	MutexLockGuard lock(mutex);
	if(currentBuffer->avail()>len)
		currentBuffer->append(logline,len);
	else{
		buffers.emplace_back(currentBuffer);
		if(nextBuffer)
			currentBuffer=std::move(nextBuffer);
		else
			currentBuffer.reset(newElement<Buffer>(),deleteElement<Buffer>);
		currentBuffer->append(logline,len);
		cond.notify();
	}
}

void AsyncLogging::threadFunc(){
	// 前端和后端的buffer
	BufferPtr newBuffer1(newElement<Buffer>(),deleteElement<Buffer>);
	BufferPtr newBuffer2(newElement<Buffer>(),deleteElement<Buffer>);
	BufferVector buffersToWrite;
	LogFile output(basename);
	while(running){
		{
			MutexLockGuard lock(mutex);
			if(buffers.empty())
				cond.waitForSeconds(flushInterval);
			buffers.emplace_back(currentBuffer);
			currentBuffer=std::move(newBuffer1);
			buffersToWrite.swap(buffers);
			if(!nextBuffer)
				nextBuffer=std::move(newBuffer2);
		}
		for(auto &wi:buffersToWrite)
			output.append(wi->data(),wi->length());
		if(!newBuffer1)
			newBuffer1.reset(newElement<Buffer>(),deleteElement<Buffer>);
		if(!newBuffer2)
			newBuffer2.reset(newElement<Buffer>(),deleteElement<Buffer>);
		buffersToWrite.clear();
		output.flush();
	}
	output.flush();
}
```

其不断从currentBuffer中获取信息，并将其放入buffers中，继而将一个空的缓冲区赋给currentBuffer。

之后不断遍历buffersToWrite，将其写入文件中，最后再补充newBuffer1和newBuffer2。

### Logging类

#### Impl类

```C++
class Impl{
public:
	Impl(const char *filename,int line);
	void formatTime();
	std::string getBaseName();
	LogStream& stream();
	int getLine();	

private:
	LogStream stream_;
	int line;
	std::string basename;
};
```



#### Logger类

```C++
class Logger{
private:
	Impl impl;

public:
	static std::string logFileName;
	Logger(const char *filename,int line);
	~Logger();
	LogStream& stream();
	//static void setLogFileName(std::string fileName);
	static std::string getLogFileName();
};
```



**重点在于**

```C++
#define LOG Logger(__FILE__,__LINE__).stream()
```

每次要使用日志功能，都会创建一个logger对象，用完即毁。

在logger的析构函数中，会把数据写入缓冲区中，最后由在AsyncLogging类中创建的线程写入文件。