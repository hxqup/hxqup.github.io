---
layout:     post
title:      "webserver笔记(十）"
subtitle:   " \"数据库的应用\""
date:       2023-04-09 15:40:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(十)

> 为什么需要数据库连接
>
> 因为涉及到登录、注册操作，需要一个存储单元来保存用户的用户名、密码等信息

## 数据库连接

### 单例模式

单例模式是一种设计模式，它确保一个类只有一个实例，并提供全局访问点以进行该实例。它通常通过将其构造函数设置为私有来实现，从而防止在其他地方创建新的实例，并提供静态方法或属性来返回该唯一实例。这种模式通常用于管理共享资源或跟踪某些状态信息，以确保系统中只有一个实例存在。

它分为懒汉式和饿汉式

* 懒汉式

```C++
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance;
        return instance;
    }

private:
    Singleton() {} // 构造函数私有化，禁止外部实例化
    Singleton(const Singleton&) = delete; // 禁止拷贝构造
    Singleton& operator=(const Singleton&) = delete; // 禁止赋值操作
```

上述代码中，静态方法 `getInstance()` 返回 `Singleton` 类的唯一实例。该实例是通过定义静态变量来实现的，因为静态变量只会在第一次调用时被初始化，所以可以确保只创建一个实例。

需要注意的是，构造函数被声明为私有，这样就禁止了外部实例化。同时，禁止拷贝构造和赋值操作也可以防止复制该实例。

使用该类的代码如下：

```C++
Singleton& instance = Singleton::getInstance();
```

这样就可以获取到 `Singleton` 类的唯一实例了。

* 饿汉式

饿汉式单例模式是指在程序启动时就立即创建对象实例。下面是C++中饿汉式单例模式的一个示例实现：

```C++
cppCopy codeclass Singleton {
public:
    static Singleton& getInstance() {
        return instance;
    }

private:
    Singleton() {} // 构造函数私有化，禁止外部实例化
    Singleton(const Singleton&) = delete; // 禁止拷贝构造
    Singleton& operator=(const Singleton&) = delete; // 禁止赋值操作

    static Singleton instance; // 静态实例变量
};

Singleton Singleton::instance; // 定义静态实例变量
```

上述代码中，静态方法 `getInstance()` 返回 `Singleton` 类的唯一实例。该实例是通过定义静态成员变量来实现的，因为静态成员变量在程序启动时就会被初始化。

需要注意的是，构造函数被声明为私有，这样就禁止了外部实例化。同时，禁止拷贝构造和赋值操作也可以防止复制该实例。

使用该类的代码如下：

```C++
Singleton& instance = Singleton::getInstance();
```

这样就可以获取到 `Singleton` 类的唯一实例了。

**但这样好像有线程安全问题，因为C++不能保证静态变量初始化的顺序。**

### RAII

RAII是Resource Acquisition Is Initialization的缩写，即资源获取即初始化。

RAII是在C++中一种重要的编程技术。它基于构造函数和析构函数的调用时机来管理资源，以确保资源在不再需要时被正确地释放。这种技术适用于任何需要手动管理内存或其他资源的情况，如文件句柄、锁、数据库连接等。

具体实现方式是将资源的生命周期与一个对象的生命周期绑定在一起。当对象创建时，资源被获取并分配给该对象；当对象被销毁时，其对应的资源也会被释放。这样可以避免资源泄漏等问题，同时也让程序更加简洁易懂，提高了代码的可读性和可维护性。

例如，C++标准库中的 `std::unique_ptr` 和 `std::shared_ptr` 就是利用RAII技术实现的智能指针，可以自动管理堆内存的分配和释放，从而避免了内存泄露等问题。

### 常用API

* mysql_real_connect函数是MySQL C API中的一个函数，用于连接到MySQL数据库服务器。它的函数原型如下：

```C++
MYSQL *mysql_real_connect(MYSQL *mysql, const char *host, const char *user, const char *passwd, const char *db, unsigned int port, const char *unix_socket, unsigned long clientflag);
```

>  该函数的参数包括：
>
>  - `mysql`：一个指向MYSQL结构的指针，该结构将保存连接的状态和结果。
>  - `host`：连接的目标主机名或IP地址。
>  - `user`：连接MySQL数据库所需的用户名。
>  - `passwd`：连接MySQL数据库所需的密码。
>  - `db`：要连接的数据库名称。
>  - `port`：MySQL服务器的端口号。
>  - `unix_socket`：如果MySQL服务器使用Unix套接字进行通信，则指定Unix套接字的路径。
>  - `clientflag`：客户端标志。
>
>  该函数返回一个指向MYSQL结构的指针，如果连接失败，则返回NULL。一旦连接成功，可以使用该指针进行后续的MySQL操作。
>
>  需要注意的是，使用该函数连接MySQL数据库时，需要先安装MySQL C API库，并在代码中包含相应的头文件和库文件。

* mysql_query函数是MySQL C API中的一个函数，用于向MySQL服务器发送SQL查询语句。它的函数原型如下：

```C++
int mysql_query(MYSQL *mysql, const char *query);
```

>  该函数的参数包括：
>
>  - `mysql`：一个指向MYSQL结构的指针，该结构保存连接的状态和结果。
>  - `query`：要执行的SQL查询语句。
>
>  该函数返回一个整数值，表示SQL查询的执行结果。如果执行成功，返回0；如果执行失败，返回非零值，并且可以通过调用mysql_error函数获取错误信息。
>
>  需要注意的是，执行SQL查询前必须先使用mysql_real_connect函数连接到MySQL服务器。在执行查询之前，还可以使用mysql_select_db函数选择要使用的数据库。另外，在使用mysql_query函数执行查询时，需要对SQL语句进行适当的编码和防注入处理，以避免安全问题的发生。

* mysql_free_result函数是MySQL C API中的一个函数，用于释放由mysql_store_result函数分配的结果集内存。它的函数原型如下：

```C++
void mysql_free_result(MYSQL_RES *result);
```

> 该函数的参数为一个指向MYSQL_RES结构的指针，该结构保存了MySQL查询的结果集。调用该函数可以释放该结果集所占用的内存。
>
> 需要注意的是，如果在调用mysql_store_result函数之后没有调用mysql_free_result函数，将会导致内存泄漏。因此，在使用mysql_store_result函数获取结果集后，必须在不需要该结果集时调用mysql_free_result函数来释放内存。另外，对于使用mysql_use_result函数获取的结果集，不需要调用mysql_free_result函数来释放内存，而是需要在使用完结果集后调用mysql_free_result函数来清除结果集的缓冲区。

* mysql_store_result函数是MySQL C API中的一个函数，用于将查询的结果集存储在一个MYSQL_RES结构中。它的函数原型如下：

```C++
MYSQL_RES *mysql_store_result(MYSQL *mysql);
```

> 该函数的参数为一个指向MYSQL结构的指针，该结构保存了MySQL连接的状态和结果。调用该函数可以将最后一个查询操作的结果集存储在MYSQL_RES结构中。
>
> 需要注意的是，mysql_store_result函数只能用于获取查询结果集，不能用于获取非查询语句（例如INSERT、UPDATE、DELETE）的返回结果。此外，对于大型结果集或者查询期间网络故障的情况，可以使用mysql_use_result函数来避免一次性将所有结果集加载到内存中。
>
> 另外，在调用mysql_store_result函数之后，必须使用mysql_num_rows函数获取结果集的行数，并使用mysql_fetch_row函数逐行读取结果集的数据。最后，在使用完结果集后，需要调用mysql_free_result函数释放结果集所占用的内存。

* mysql_num_fields函数是MySQL C API中的一个函数，用于获取结果集中的列数。它的函数原型如下：

```C++
unsigned int mysql_num_fields(MYSQL_RES *result);
```

> 该函数的参数为一个指向MYSQL_RES结构的指针，该结构保存了MySQL查询的结果集。调用该函数可以获取结果集中的列数。
>
> 需要注意的是，使用mysql_num_fields函数之前必须先调用mysql_store_result函数获取结果集，并且必须在调用mysql_free_result函数释放结果集之前调用mysql_num_fields函数获取列数。
>
> 在获取结果集的列数后，可以使用mysql_fetch_field函数获取每个字段的详细信息，如字段名、数据类型、长度等等。此外，在处理结果集的时候，还可以使用mysql_fetch_row函数逐行读取数据，并使用mysql_field_count函数获取最后一次执行的查询操作的字段数。

* mysql_fetch_fields函数是MySQL C API中的一个函数，用于获取结果集中每个字段的详细信息。它的函数原型如下：

```C++
MYSQL_FIELD *mysql_fetch_fields(MYSQL_RES *result);
```

> 该函数的参数为一个指向MYSQL_RES结构的指针，该结构保存了MySQL查询的结果集。调用该函数可以获取结果集中每个字段的详细信息，包括字段名、数据类型、长度等等。
>
> 需要注意的是，在调用mysql_fetch_fields函数之前必须先调用mysql_store_result函数获取结果集，并且必须在调用mysql_free_result函数释放结果集之前调用mysql_fetch_fields函数获取字段信息。
>
> mysql_fetch_fields函数返回一个指向MYSQL_FIELD结构的指针数组，每个元素对应结果集中的一个字段，可以通过该指针访问该字段的详细信息。例如，可以通过field_name成员访问字段名，通过type成员访问字段的数据类型，通过length成员访问字段的长度等等。
>
> 需要注意的是，如果结果集中有大量的字段信息需要获取，可能会对系统资源造成负担。因此，在实际使用中应该尽量避免一次性获取所有字段信息，而是根据需要逐个获取每个字段的信息。

* mysql_fetch_row函数是MySQL C API中的一个函数，用于逐行获取查询结果集中的数据。它的函数原型如下：

```C++
MYSQL_ROW mysql_fetch_row(MYSQL_RES *result);
```

> 该函数的参数为一个指向MYSQL_RES结构的指针，该结构保存了MySQL查询的结果集。调用该函数可以逐行获取结果集中的数据，并以MYSQL_ROW类型的指针返回当前行的数据。
>
> 需要注意的是，在调用mysql_fetch_row函数之前必须先调用mysql_store_result函数获取结果集，并且必须在调用mysql_free_result函数释放结果集之前调用mysql_fetch_row函数逐行读取结果集的数据。
>
> mysql_fetch_row函数返回一个指向MYSQL_ROW结构的指针，该结构保存了当前行的数据。MYSQL_ROW是一个字符指针的数组，每个元素对应当前行的一个字段值。可以使用mysql_num_fields函数获取结果集的字段数，从而确定MYSQL_ROW数组的长度，进而逐个访问每个字段的值。
>
> 需要注意的是，mysql_fetch_row函数每次只能返回结果集中的一行数据，并将指针移动到下一行。因此，在处理结果集的时候，需要使用循环结构不断调用mysql_fetch_row函数来逐行读取数据，直到返回NULL表示已经读取完所有数据。

* mysql_close是 MySQL C API 中的一个函数，用于关闭与服务器的连接，并释放由 `mysql_init()` 分配和初始化的所有资源。该函数的原型如下：

```C++
void mysql_close(MYSQL *mysql);
```

> 其中，`mysql` 参数是一个 `MYSQL` 结构体指针，表示与服务器连接的句柄。该函数将释放 `mysql` 对应的资源并关闭连接。
>
> 需要注意的是，调用 `mysql_close` 函数后，不能再使用 `mysql` 句柄进行查询或其他操作。如果需要重新连接数据库，必须重新调用 `mysql_init()` 函数来初始化一个新的 `MYSQL` 结构体。
>
> 在实际使用中，为了避免内存泄漏，通常在程序结束时显式调用 `mysql_close()` 函数来关闭数据库连接。

* 

mysql_library_end()函数的原型如下：

```C++
void mysql_library_end(void);
```

> 这个函数没有参数，也没有返回值。调用它会释放在程序中调用mysql_library_init()函数时分配的所有资源。
>
> 需要注意的是，一旦调用了mysql_library_end()函数，就不能再使用MySQL客户端程序库中的任何函数。因此，一般情况下，程序应该在最后调用mysql_library_end()函数。