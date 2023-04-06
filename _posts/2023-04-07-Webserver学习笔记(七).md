---
layout:     post
title:      "webserver笔记(七）"
subtitle:   " \"定时器类\""
date:       2023-04-06 16:30:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# WebServer学习笔记(七)

## 定时器

定时器设计采用一个**小顶堆**，每当SIGALRM信号到来，超时时间最小的定时器必然到期，然后再从剩余定时器中找到超时时间最小的定时器中，并将这一最小时间设置为心搏间隔。

> 最小堆与最大堆
>
> 最小堆是指每个节点的值都小于其左右孩子值的一棵完全二叉树。在C++ STL中，priority_queue底层即是用堆这一数据结构来实现的。
>
> 不过这一数据结构有一个“奇怪”的特性，就是其模板参数
>
> ```C++
> template <typename T, typename Container=std::vector<T>, typename Compare=std::less<T>> class priority_queue
> ```
>
> 第三个参数可以用一个仿函数，默认是采用less<T\>,但这会实现一个最大堆。而如果采用greater<T\>则实现的是最小堆。
>
> 这是因为在堆的调整过程中，对于大顶堆，如果当前插入的节点值大于其父节点，那么就应该向上调整。其父节点索引小于当前插入节点的索引，也就是父节点是左数，插入节点是右值，可以看到，左数小于右数时，要向上调整，也就是Compare函数应该返回true，正好是less。



下面具体看下TimerManager类的实现

```C++
struct TimerCmp{
	bool operator()(SP_TimerNode &a,SP_TimerNode &b)const{
		return a->getExpiredtime()>b->getExpiredtime();
	}
};

class TimerManager{
private:
	std::priority_queue<SP_TimerNode,std::vector<SP_TimerNode>,TimerCmp>timerheap;
	std::unordered_map<int,SP_TimerNode>timermap;

public:
	void addTimer(SP_Channel channel,int timeout);
	void handleExpiredEvent();
};

typedef std::shared_ptr<TimerManager> SP_TimerManager;
```

可以看到这个类中，直接用到了STL封装好的priority_queue,其实也可以自己实现一个最小堆



而主要的函数包括两个，

```C++
void TimerManager::addTimer(SP_Channel channel,int timeout){
	//SP_TimerNode timernode(new TimerNode(channel,timeout));
	SP_TimerNode timernode(newElement<TimerNode>(channel,timeout),deleteElement<TimerNode>);
	int cfd=channel->getFd();
	//if(timermap.find(cfd)==timermap.end())
	// 判断该节点是否是第一次添加
	if(channel->isFirst()){
		timerheap.push(timernode);
		channel->setnotFirst();
	}
	timermap[cfd]=std::move(timernode);
}

void TimerManager::handleExpiredEvent(){
	while(!timerheap.empty()){
		SP_TimerNode now=timerheap.top();
		if(now->isDeleted() || !now->isValib()){
			timerheap.pop();
			SP_TimerNode timernode=timermap[now->getChannel()->getFd()];
			if(now==timernode){
				timermap.erase(now->getChannel()->getFd());
				// 对应的定时器节点已经被更新过了
				now->getChannel()->handleClose();
			}
			else
				timerheap.push(timernode);
		}
		else
			break;
	}
}
```

其中addTimer中会判断这个节点是否是第一次添加，也就是timermap中是否有这个节点，如果有的话，就直接根据文件描述符来修改就好了。

而handleExpiredEvent则用来处理超时事件，从堆顶取出元素，判断其是否满足超时条件，如果满足的话，就将其从堆中删除。此外，如果该文件描述符的时间没有更新过，则直接断开连接。