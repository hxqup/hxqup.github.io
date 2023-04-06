---
layout:     post
title:      "webserver笔记(六）"
subtitle:   " \"LFU缓存\""
date:       2023-04-06 16:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(六)

## LFU缓存

> LFU与LRU
>
> LFU（Least Frequently Used）和LRU（Least Recently Used）是两种常见的缓存淘汰算法，用于决定在缓存容量限制下，哪些数据需要被移除。它们的区别主要在于以下两个方面：
>
> 1. 淘汰策略不同： LFU会淘汰最少使用次数的数据，即访问频率较低的数据；而 LRU 则会淘汰最近最少使用的数据，即访问时间距离当前时间最远的数据。
> 2. 计算代价不同：LFU需要维护每个缓存块被访问的次数，因此需要额外的计数器并进行更新；而 LRU 只需要维护一个时间戳来记录最近的访问时间。因此，LFU的计算代价比 LRU 更高。
>
> 需要注意的是，虽然 LFU 和 LRU 都可以用作缓存优化算法，但具体使用哪种算法还取决于应用场景、数据特征以及性能需求等因素。

### LFU的实现

实现了两个链表，一个叫**key_list**，另一个叫**freq_list**

其中key_list节点的value为

```C++
struct Key{
	string key,value;
};
```

freq_list节点的value为**key_list**



最终LFUCache类的实现如下：

```C++
class LFUCache{
private:
	freq_node head;
	int capacity;
	MutexLock mutex;	

	std::unordered_map<string,key_node> kmap;//key到keynode的映射
	std::unordered_map<string,freq_node> fmap;//key到freqnode的映射
	
	void addfreq(key_node &nowk,freq_node &nowf);	
	void del(freq_node &node);

public:
	void init();
	~LFUCache();
	bool get(string &key,string &value);//通过key返回value并进行LFU操作
	void set(string &key,string &value);//更新LFU缓存
};
```

这个类的重点在于get和set的实现：

* get

```C++
bool LFUCache::get(string &key,string &v){
	if(!capacity)
		return false;
	MutexLockGuard lock(mutex);
	if(fmap.find(key)!=fmap.end()){
		//命中
		key_node nowk=kmap[key];
		freq_node nowf=fmap[key];
		v += nowk->getValue().value;
		addfreq(nowk,nowf);
		return true;
	}
	return false;
}
```

如果命中了缓存，则返回缓存池中缓存的value值

---

* set

```C++
void LFUCache::set(string &key,string &v){
	if(!capacity)
		return;
	//printf("kmapsize=%d capacity=%d\n",kmap.size(),capacity);
	MutexLockGuard lock(mutex);
	if(kmap.size() == capacity){
		freq_node headnxt = head->getNext();
		key_node last=headnxt->getValue().getLast();
		headnxt->getValue().del(last);
		//printf("key=%d\n",last->getValue().key);
		kmap.erase(last->getValue().key);
		fmap.erase(last->getValue().key);
		//delete last;
		// 释放内存
		deleteElement(last);
		if(headnxt->getValue().isEmpty())
			del(headnxt);
	}
	//key_node nowk=new Node<Key>;
	key_node nowk=newElement<Node<Key>>();
	nowk->getValue().key=key;
	nowk->getValue().value=v;
	addfreq(nowk,head);
	kmap[key]=nowk;
}
```

多了一重判定，看是否达到缓存池的容量大小，如果达到了，则需要删除缓存池中使用频率最低的节点，随后插入新节点。