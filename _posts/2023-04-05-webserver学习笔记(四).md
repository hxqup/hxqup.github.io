---
layout:     post
title:      "webserver笔记(四）"
subtitle:   " \"http连接处理\""
date:       2023-04-05 22:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(四)

## http报文示例

> * 请求报文：
>
>   ```C++
>   GET /index.html HTTP/1.1
>   Host: www.example.com
>   Connection: keep-alive
>   Cache-Control: max-age=0
>   User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.101 Safari/537.36
>   Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
>   Accept-Encoding: gzip, deflate, br
>   Accept-Language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7
>   ```
>
>   ```C++
>    POST / HTTP1.1
>    Host:www.wrox.com
>    User-Agent:Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1; .NET CLR 2.0.50727; .NET CLR 3.0.04506.648; .NET CLR 3.5.21022)
>    Content-Type:application/x-www-form-urlencoded
>    Content-Length:40
>    Connection: Keep-Alive
>    空行
>    name=Professional%20Ajax&publisher=Wiley
>   ```
>
> * 响应报文
>
>   ```C++
>   HTTP/1.1 200 OK
>   Date: Mon, 05 Apr 2023 00:00:00 GMT
>   Server: Apache/2.4.6 (CentOS)
>   Content-Type: text/html; charset=UTF-8
>   Content-Length: 1234
>   Connection: close
>   
>   <!DOCTYPE html>
>   <html>
>   <head>
>   	<title>Example Page</title>
>   </head>
>   <body>
>   	<h1>Hello, World!</h1>
>   	<p>This is an example page.</p>
>   </body>
>   </html> 
>   ```

---

## http状态码

HTTP有5种类型的状态码，具体的：

* 1xx：指示信息--表示请求已接收，继续处理。其中，如果http请求为post，那么服务端可能会中途返回一个100状态码，意为继续
* 2xx：成功--表示请求正常处理完毕。

> 常见的有：
>
> 1. 200 OK：客户端请求被正常处理
> 2. 205 Partial content：客户端进行了范围请求

* 3xx：重定向--要完成请求必须进行更进一步的操作

> 常见的有：
>
> 1. 301 Moved Permanently：永久重定向，该资源已被永久移动到新位置，将来任何对该资源的访问都要使用本响应返回的若干个URL之一
> 2. 302 Found：临时重定向，请求的资源可以临时从不同URL获得

* 4xx:客户端错误--请求有语法错误，服务端无法处理

> 常见的有：
>
> 1. 400 Bad Request:是一个比较笼统的错误，请求报文无法解析
> 2. 403 Forbidden：请求被服务器拒绝
> 3. 404 Not Found：请求不存在，服务器上找不到请求的资源

* 5xx：服务端错误——服务器处理请求出错

> 常见的有：
>
> 1. 500 Internal Server error：服务器在执行请求时出现错误

## http连接处理

> 处理流程
>
> * 浏览器端发出http连接请求，主线程创建http对象接收请求并将所有数据读入对应buffer，将该对象插入任务队列，工作线程从任务队列中取出一个任务进行处理
> * 工作线程取出任务后，调用read函数，通过状态机对请求报文进行解析
> * 解析完之后，生成响应报文，并写入buffer，返回给浏览器

### 读取请求

首先创建一个std::string对象作为inbuffer,然后调用readn函数读取请求，readn的实现如下

```C++
ssize_t readn(int fd,std::string &inbuffer,bool &zero){
	ssize_t nread;
	ssize_t readSum=0;
	while(true){
		char buff[MAXLINE];
		if((nread=read(fd,buff,MAXLINE))<0){
			if(EINTR==errno)
				continue;
			else if(EAGAIN==errno)
				return readSum;
			else{
				fprintf(stderr,"read error\n");
				return -1;
			}
		}	
		// 表示
		else if(!nread){
			zero=true;
			break;
		}
		readSum+=nread;
		inbuffer+=std::string(buff,buff+nread);
	}
	return readSum;
}
```

循环调用read函数读取请求，MAXLINE表示一次读取的字节数，每次读取完，readSum要增加，同时把buffer中的数据移动到inbuffer中。

### 解析数据

主要函数代码如下：

```C++
void Http_conn::parse(){
	bool zero=false;
	int readsum;
	if(getconf().getssl())
		readsum=SSL_readn(channel->getssl().get(),inbuffer,zero);
	else
    	readsum=readn(channel->getFd(),inbuffer,zero);
    if(readsum<0||zero){
        initmsg();
        channel->setDeleted(true);
        channel->getLoop().lock()->addTimer(channel,0);
	return;
        //读到RST和FIN默认FIN处理方式，这里的话因为我不太清楚读到RST该怎么处理，就一起这样处理好了
    }
	while(inbuffer.length()&&~inbuffer.find("\r\n",pos))
		parsestate=handleparse[parsestate]();
}
```

关键之处就在于最后那个while循环，会更换主状态机的状态，这里相当于更改函数指针，从而循环调用，一步步解析报文。

### 生成响应报文

主要代码如下：

```C++
void Http_conn::send(){
	string outbuffer="";
	if(METHOD_POST==method){

	}
	else if(METHOD_GET==method){
		if(keepalive){
			outbuffer="HTTP/1.1 200 OK\r\n";
			outbuffer+=string("Connection: Keep-Alive\r\n")+"Keep-Alive: timeout="+to_string(getconf().getkeep_alived())+"\r\n";
			channel->getLoop().lock()->addTimer(channel,getconf().getkeep_alived());
		}
		else{
			outbuffer="HTTP/1.0 200 OK\r\n";
			channel->getLoop().lock()->addTimer(channel,0);
		}
		outbuffer += "Content-Type: " + filetype + "\r\n";
		outbuffer += "Content-Length: " + to_string(size) + "\r\n";
        	outbuffer += "Server: WWQ's Web Server\r\n";
		outbuffer += "\r\n";
		if(!(getCache().get(path,outbuffer))){
			int src_fd=Open((storage+path).c_str(),O_RDONLY,0);
			char *src_addr=(char *)mmap(NULL,size,PROT_READ,MAP_PRIVATE,src_fd,0);
			Close(src_fd);
			string context(src_addr,size);
			outbuffer+=context;
			getCache().set(path,context);
			munmap(src_addr,size);
		}
	}
	const char *buffer=outbuffer.c_str();
	if(getconf().getssl()){
		if(!SSL_writen(channel->getssl().get(),buffer,outbuffer.length()))
			LOG<<"ssl writen error";
	}
	else{
		if(!writen(channel->getFd(),buffer,outbuffer.length()))
			LOG<<"writen error";
	}
	initmsg();
	channel->setRevents(EPOLLIN|EPOLLET);
	channel->getLoop().lock()->updatePoller(channel);
}
```

这个部分可以分为两步：

1. 生成响应报文：

> ```C++
> 		if(keepalive){
> 			outbuffer="HTTP/1.1 200 OK\r\n";
> 			outbuffer+=string("Connection: Keep-Alive\r\n")+"Keep-Alive: timeout="+to_string(getconf().getkeep_alived())+"\r\n";
> 			channel->getLoop().lock()->addTimer(channel,getconf().getkeep_alived());
> 		}
> 		else{
> 			outbuffer="HTTP/1.0 200 OK\r\n";
> 			channel->getLoop().lock()->addTimer(channel,0);
> 		}
> 		outbuffer += "Content-Type: " + filetype + "\r\n";
> 		outbuffer += "Content-Length: " + to_string(size) + "\r\n";
>         outbuffer += "Server: WWQ's Web Server\r\n";
> 		outbuffer += "\r\n";
> 		if(!(getCache().get(path,outbuffer))){
> 			int src_fd=Open((storage+path).c_str(),O_RDONLY,0);
> 			char *src_addr=(char *)mmap(NULL,size,PROT_READ,MAP_PRIVATE,src_fd,0);
> 			Close(src_fd);
> 			string context(src_addr,size);
> 			outbuffer+=context;
> 			getCache().set(path,context);
> 			munmap(src_addr,size);
> 		}
> ```
>
> 其中，如果缓存中没有，那么就先通过文件描述符和内存进行映射，再把这段响应报文传入outbuffer，并放入缓存中。

2.发送给客户端

> ```C++
> ssize_t writen(int fd,const void *vptr,size_t n){
> 	size_t nleft;
> 	ssize_t nwritten;
> 	const char *ptr=(char *)vptr;
> 	nleft=n;
> 	while(nleft>0){
> 		if((nwritten=write(fd,ptr,nleft))<=0){
> 			if(nwritten<0&&EINTR==errno)
> 				nwritten=0;
> 			else
> 				return -1;
> 		}
> 		nleft-=nwritten;
> 		ptr+=nwritten;
> 	}
> 	return n;
> }
> ```
>
> 就是循环写，直到write的返回值<0