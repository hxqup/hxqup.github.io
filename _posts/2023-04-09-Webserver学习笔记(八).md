---
layout:     post
title:      "webserver笔记(八）"
subtitle:   " \"内存池的实现\""
date:       2023-04-09 15:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(八)

## 内存池实现

> 为什么要实现一个内存池？
>
> 典型的例子就是大并发频繁分配和回收内存，会导致进程的内存产生碎片，并且不会立马被系统回收，容易产生内存泄露。
>
> 使用内存池分配内存有几点好处：
>
> * 提升内存分配效率。
>
> * 不需要每次分配内存都执行malloc/alloc等函数。
>
> * 让内存的管理变得更加简单。
>
> * 内存的分配都会在一块大的内存上，回收的时候只需要回收大块内存就能将所有的内存回收，防止了内存管理混乱和内存泄露问题。

### STL中的内存配置

考虑到小型区块可能造成内存碎片，SGI STL 设计了双层配置器，第一层配置器直接使用malloc和free，第二层配置器则视情况采用不同策略：当配置区块超过128bytes，视为足够大，便调用第一级配置器；当配置区块小于128bytes，视为过小，为了降低额外负担，使用复杂的memory pool整理方式。

* SGI第一级配置器

> allocate直接使用malloc，deallocate直接使用free
>
> 模拟C++的set_new_handler()处理内存不足的情况

* SGI第二级配置器

> * 维护16个自由链表，负责这16个区块的次配置能力，内存池以malloc获得，如果内存不足，转而采用第一级配置器
> * 如果需求大于128bytes，转而采用第一级配置器

### 内存池的实现

首先定义了一个结构体，叫做slot(插槽)，用来描述一块空闲内存，实现如下：

```C++
struct Slot{
	Slot* next;
};
```

可以看到，Slot结构体只包含一个元素，就是一个指向下一块内存块的指针，所以Slot就相当于是一个链表节点，用来描述内存池中的内存块之间的关系。

```C++
class MemoryPool{
private:
	int slot_size;	
		
	Slot* currentBlock;
	Slot* currentSlot;
	Slot* lastSlot;
	Slot* freeSlot;
	
	MutexLock m_freeSlot;
	MutexLock m_other;
	size_t padPointer(char* p,size_t align);
	Slot* allocateBlock();
	Slot* nofree_solve();	

public:
	MemoryPool();
	~MemoryPool();
	void init(int size);	
	Slot* allocate();
	void deallocate(Slot* p);

};
```

内存池初始化时，会创建一个大小为64的内存池数组，索引为0的内存池大小为8字节，索引为1的内存池大小为16字节，第三块为32字节，以此类推...

```C++
void init_memorypool(){
	for(int i=0;i<64;++i)
		get_memorypool(i).init((i+1)<<3);
}
```

主要函数有两个：

* 分配内存

```C++
template<class T,class... Args>
T* newElement(Args&&... args){
	T *p;
	if(p=reinterpret_cast<T *>(use_memory(sizeof(T))))
		new(p) T(std::forward<Args>(args)...);
	return p;
}
```

> 这里补充一些C++知识：
>
> * placement new
>
>   如果你想在已经分配的内存中创建一个对象，使用new是不行的。也就是说placement new允许你在一个已经分配好的内存中（栈或堆中）构造一个新的对象。原型中void*p实际上就是指向一个已经分配好的内存缓冲区的的首地址。
>
>   使用 placement new 运算符在已分配的内存位置上构造对象时，必须显式调用对象的构造函数，并将其参数传递给构造函数进行处理，以确保构造和析构过程的正确性和完整性。
>
> * 完美转发
>
>   **std::forward会将输入的参数原封不动地传递到下一个函数中，这个“原封不动”指的是，如果输入的参数是左值，那么传递给下一个函数的参数的也是左值；如果输入的参数是右值，那么传递给下一个函数的参数的也是右值。**
>
> * 可变参数模板
>
>   typename... Args 表示一个参数包（parameter pack），表示可以接受任意数量的类型参数。在使用参数包时，可以使用展开运算符 ... 将其扩展为一系列单独的类型参数，然后对每个类型参数进行处理。
>
> * 万能引用
>
>   C++11 标准中规定，通常情况下右值引用形式的参数只能接收右值，不能接收左值。但对于函数模板中使用右值引用语法定义的参数来说，它不再遵守这一规定，既可以接收右值，也可以接收左值（此时的右值引用又被称为“万能引用”）。其主要用于与完美转发配合使用

use_memory函数：如果请求内存大于512字节，则直接调用malloc，否则，请求从内存池中取出对应大小的内存。

```C++
Slot* MemoryPool::allocate(){
	// 双重检测,保证同一时刻只有一个线程能访问freeSlot
	if(freeSlot){
		{
			MutexLockGuard lock(m_freeSlot);
			if(freeSlot){
				Slot *result = freeSlot;
				freeSlot = freeSlot->next;
				return result;
			}
		}
	}
	return nofree_solve();
}
```

每次申请内存时，先检查一下所需内存池内是否还有空闲插槽，如果没有的话，则调用一个handler(no_free_solve)，如果有的话，则直接返回空闲插槽，空闲插槽是被释放之后得到的内存块。

```C++
Slot* MemoryPool::nofree_solve(){
	if(currentSlot >= lastSlot)
		return allocateBlock();
	Slot* useSlot;
	{
		MutexLockGuard lock(m_other);
		useSlot = currentSlot;
		currentSlot += (slot_size>>3);
	}
	return useSlot;
}
```

nofree_solve先进行判定，如果当前内存池内没有空闲的插槽了，则必须要新分配一个内存块，否则，则直接返回一块插槽就ok。

内存块的分配如下所示：

```C++
Slot* MemoryPool::allocateBlock(){
	char* newBlock=NULL;
	while(!(newBlock=reinterpret_cast<char *>(malloc(BlockSize))));
	char* body=newBlock+sizeof(Slot);
	size_t bodyPadding=padPointer(body,static_cast<size_t>(slot_size));
	Slot *useSlot;
	{
            MutexLockGuard lock(m_other);
            reinterpret_cast<Slot*>(newBlock)->next=currentBlock;
            currentBlock = reinterpret_cast<Slot*>(newBlock);
            currentSlot = reinterpret_cast<Slot*>(body+bodyPadding);
            lastSlot = reinterpret_cast<Slot*>(newBlock+BlockSize-slot_size+1);
            useSlot = currentSlot;
            currentSlot += (slot_size>>3);
	}
    return useSlot;
}
```

> while(!(newBlock=reinterpret_cast<char *>(malloc(BlockSize))))  这相当于也是一个handler，如果分配不出新的内存块，就一直让其循环，相当于一个自旋锁
>
> padPointer相当于让内存块的起始地址要和slot_size的大小对齐，具体实现如下：
>
> ```C++
> inline size_t MemoryPool::padPointer(char* p,size_t align){
> 	// uintptr_t 是 C++ 中的一种无符号整数类型，用于存储指针转换为整数后的值。
> 	uintptr_t result = reinterpret_cast<uintptr_t>(p);
> 	return ((align-result)%align);
> }
> ```

* 释放内存

```C++
template<class T>
void deleteElement(T* p){
	if(p)
		p->~T();
	free_memory(sizeof(T),reinterpret_cast<void *>(p));
}
```

free_memory函数：如果请求的内存大小大于512字节，则直接调用free，否则调用deallocate函数

deallocate实现如下：

```C++
inline void MemoryPool::deallocate(Slot* p){
	if(p){
		MutexLockGuard lock(m_freeSlot);
		p->next = freeSlot;
		freeSlot = p;
	}
}
```

相当于是在一个空闲链表中往前插入一块内存。