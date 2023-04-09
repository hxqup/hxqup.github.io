---
layout:     post
title:      "webserver笔记(十一）"
subtitle:   " \"post的处理\""
date:       2023-04-09 16:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(十一)

> post和get区别
>
> * 根据 RFC 规范，**GET 的语义是从服务器获取指定的资源**，这个资源可以是静态的文本、页面、图片视频等。GET 请求的参数位置一般是写在 URL 中，URL 规定只能支持 ASCII，所以 GET 请求的参数只允许 ASCII 字符 ，而且浏览器会对 URL 的长度有限制（HTTP协议本身对 URL长度并没有做任何规定）。
> * 根据 RFC 规范，**POST 的语义是根据请求负荷（报文body）对指定的资源做出处理**，具体的处理方式视资源类型而不同。POST 请求携带数据的位置一般是写在报文 body 中，body 中的数据可以是任意格式的数据，只要客户端与服务端协商好即可，而且浏览器不会对 body 大小做限制。

## 登录、注册操作

在对http请求信息进行处理时，也需要对请求方法为get和post的情况分别进行处理

如果请求方法为post

```C++
    if(method == METHOD_POST && header["Content-Type"] == "application/x-www-form-urlencoded") {
        ParseFromUrlencoded_();
        if(DEFAULT_HTML_TAG.count(path)) {
            int tag = DEFAULT_HTML_TAG.find(path)->second;
            if(tag == 0 || tag == 1) {
                bool isLogin = (tag == 1);
                if(UserVerify(post["username"], post["password"], isLogin)) {
                    path = "welcome.html";
                } 
                else {
                    path = "error.html";
                }
            }
        }
```

### 什么是application/x-www-from-urlencoded

通过测试发现可以正常访问接口，在Chrome的开发者工具中可以看出，表单上传编码格式为`application/x-www-form-urlencoded`(Request Headers中)，参数的格式为`key=value&key=value`。

![img](https:////upload-images.jianshu.io/upload_images/5863464-eefb916cd00ff9f9.png?imageMogr2/auto-orient/strip|imageView2/2/w/813/format/webp)

我们可以看出，服务器知道参数用符号`&`间隔，如果参数值中需要`&`，则必须对其进行编码。编码格式就是`application/x-www-form-urlencoded`（**将键值对的参数用&连接起来，如果有空格，将空格转换为`+`加号；有特殊符号，将特殊符号转换为`ASCII HEX`值**）。`application/x-www-form-urlencoded`是浏览器默认的编码格式。

### 解析post表单数据

首先要对浏览器编码格式进行转换，转换成不含任何格式符号的形式

```C++
void Http_conn::ParseFromUrlencoded_() {
    if(body_.size() == 0) { return; }

    string key, value;
    int num = 0;
    int n = body_.size();
    int i = 0, j = 0;

    for(; i < n; i++) {
        char ch = body_[i];
        switch (ch) {
        case '=':
            key = body_.substr(j, i - j);
            j = i + 1;
            break;
        case '+':
            body_[i] = ' ';
            break;
        case '%':
            num = ConverHex(body_[i + 1]) * 16 + ConverHex(body_[i + 2]);
            body_[i + 2] = num % 10 + '0';
            body_[i + 1] = num / 10 + '0';
            i += 2;
            break;
        case '&':
            value = body_.substr(j, i - j);
            j = i + 1;
            post[key] = value;
            break;
        default:
            break;
        }
    }
    assert(j <= i);
    if(post.count(key) == 0 && j < i) {
        value = body_.substr(j, i - j);
        post[key] = value;
    }
}
```

上面这个函数对浏览器编码过后的body进行处理，分别对各个符号进行了讨论：

* “=”符号

​		将当前位置之前的字符串作为键（key）保存下来，同时将下一个字符的位置标记为值（value）的起始位置。

* “+"符号

​		将其替换为空格（' '），以便正确解析URL编码。

* “%"符号

​		如果当前字符为百分号（'%'），将其后面两个字符解析为十六进制数，然后将其转换为相应的字符，以便正确解析URL编码。

* “&"符号

​		将当前位置之前的字符串作为值，保存到以键为索引的POST参数中。

* 默认情况

​		什么也不做

### 验证用户数据

前面已经得到了用户传输来的用户名和密码，现在要从数据库中取出对应的密码，并进行比较，从而返回响应的html网页

```C++
PARSESTATE Http_conn::parseHeader(){
    if(inbuffer[pos] == '\r' && inbuffer[pos + 1] == '\n')
        return PARSE_BODY;
    string key,value;
    if(!Read(key,": "))
        return PARSE_ERROR;
    if(!Read(value,"\r\n"))
        return PARSE_ERROR;
    header[key] = value;
    return PARSE_HEADER;
}

bool Http_conn::UserVerify(const string &name, const string &pwd, bool isLogin) {
    if(name == "" || pwd == "") { return false; }
    MYSQL* sql;
    SqlConnRAII(&sql,SqlConnPool::Instance());
    assert(sql);
    
    bool flag = false;
    unsigned int j = 0;
    char order[256] = { 0 };
    MYSQL_FIELD *fields = nullptr;
    MYSQL_RES *res = nullptr;
    if(!isLogin) { flag = true; }
    /* 查询用户及密码 */
    snprintf(order, 256, "SELECT username, passwd FROM user WHERE username='%s' LIMIT 1", name.c_str());

    if(mysql_query(sql, order)) { 
        mysql_free_result(res);
        return false; 
    }
    res = mysql_store_result(sql);
    j = mysql_num_fields(res);
    fields = mysql_fetch_fields(res);

    while(MYSQL_ROW row = mysql_fetch_row(res)) {
        string password(row[1]);
        /* 注册行为 且 用户名未被使用*/
        if(isLogin) {
            if(pwd == password) { 
                flag = true; 
            }
            else {
                flag = false;
            }
        } 
        else { 
            flag = false; 
        }
    }
    mysql_free_result(res);

    /* 注册行为 且 用户名未被使用*/
    if(!isLogin && flag == true) {
        bzero(order, 256);
        snprintf(order, 256,"INSERT INTO user(username, password) VALUES('%s','%s')", name.c_str(), pwd.c_str());
        if(mysql_query(sql, order)) { 
            flag = false; 
        }
        flag = true;
    }
    return flag;
}
```

先判断用户的操作是登录还是注册，如果是登录的话，则用mysql查找对应用户名下的密码，并与用于传来的密码进行比较，判断正确性。由于创建表时，选择的数据库为Innodb，所以使用的是b+数查找，时间复杂度为O(logn);

如果是注册的话，也用户名未被使用，则直接往表中insert数据就ok了。