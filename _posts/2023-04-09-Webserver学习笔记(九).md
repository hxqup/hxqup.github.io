---
layout:     post
title:      "webserver笔记(九）"
subtitle:   " \"ssl加密通信\""
date:       2023-04-09 15:00:00
author:     "Hxq"
header-img: "img/post-bg-2015.jpg"
tags:
    - 网络编程
---

# Webserver学习笔记(九)

> 为什么要使用ssl连接
>
> 因为该web服务器设计到登录、注册操作，如果仅仅使用http1.1进行传输，传输过程中信息的安全性得不到保证，因此考虑通过openssl库接入ssl进行加密通信

## ssl连接

### 创建CA(证书认证机构)

首先创建一个 myCA 目录来存放自建 CA 的相关信息：

```shell
# 创建用户目录下的 https 目录作为实验目录，在其中创建 myCA 目录存放 CA 相关信息
# myCA/signedcerts：存放经 CA 认证的证书副本
# myCA/private：存放 CA 私钥
cd && mkdir -p https/myCA/signedcerts && mkdir https/myCA/private && cd https/myCA

# 创建证书库，这两个文件存放了 CA 每一次颁发证书的记录
echo '01' > serial && touch index.txt
```

使用任何你熟悉的编辑器在 myCA 目录下新建一个 caconfig.cnf 文件来配置 CA 信息，其中需要注意的部分有：

* username：修改为自己的用户名

```t
# My sample caconfig.cnf file.
#
# Default configuration to use when one is not provided on the command line.
#
[ ca ]
default_ca      = local_ca
#
#
# Default location of directories and files needed to generate certificates.
#
[ local_ca ]
dir             = /home/{username}/https/myCA    # CA 目录
certificate     = $dir/cacert.pem
database        = $dir/index.txt
new_certs_dir   = $dir/signedcerts
private_key     = $dir/private/cakey.pem
serial          = $dir/serial
#      
#
# Default expiration and encryption policies for certificates.
# 认证其他服务器证书设置
default_crl_days        = 365                  # 默认吊销证书列表更新时间
default_days            = 1825                 # 默认证书有效期
default_md              = sha256               # 默认对证书签名时的摘要算法
#      
policy          = local_ca_policy
x509_extensions = local_ca_extensions
#      
#
# Default policy to use when generating server certificates.  The following
# fields must be defined in the server certificate.
#
[ local_ca_policy ]
commonName              = supplied
stateOrProvinceName     = supplied
countryName             = supplied
emailAddress            = supplied
organizationName        = supplied
organizationalUnitName  = supplied
#      
#
# x509 extensions to use when generating server certificates.
#
[ local_ca_extensions ]
basicConstraints        = CA:false
#      
#
# The default root certificate generation policy.
# 生成 CA 根证书设置
[ req ]
default_bits    = 2048                                          # 默认生成证书请求时的秘钥长度
default_keyfile = /home/{username}/https/myCA/private/cakey.pem # 默认私钥存放位置
default_md      = sha256                                        # 默认证书签名时使用的摘要算法
#     
prompt                  = no
distinguished_name      = root_ca_distinguished_name
x509_extensions         = root_ca_extensions
#
#
# Root Certificate Authority distinguished name.  Change these fields to match
# your local environment!
#
[ root_ca_distinguished_name ]
commonName              = myCA              # CA 机构名
stateOrProvinceName     = JS                # 所在省份
countryName             = CN                # 所在国家（仅限两字符）
emailAddress            = example@qq.com    # 邮箱
organizationName        = USTC              # 组织名
organizationalUnitName  = SE                # 单位名
#      
[ root_ca_extensions ]
basicConstraints        = CA:true
```

然后生成自签名的 CA 根证书和秘钥，CA 的根证书是自签名的，也就是用自己的私钥对自己的证书签名，因为它是可信机构，不需要别人认证：

```
export OPENSSL_CONF=~/https/myCA/caconfig.cnf

openssl genpkey -algorithm RSA -out ca.key -aes256

openssl req -new -x509 -days 3650 -key ca.key -out ca.crt
```

### 创建服务器证书并用CA签名

```
openssl genpkey -algorithm RSA -out server.key -aes256

openssl req -new -key server.key -out server.csr

openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
```

之后，把server.crt和server.key移动到代码目录下的ssl文件夹里面，开启ssl通信后，可以看到浏览器中显示如图所示的信息。

![image-20230408200758189](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20230408200758189.png)

之所以，浏览器仍然提示连接不安全，是因为使用的是自签名证书，而自签根证书是1024位的，而几乎所有浏览器都不认可1024位证书

![image-20230408200816989](C:\Users\ASUS\AppData\Roaming\Typora\typora-user-images\image-20230408200816989.png)

## ssl与C++

### ssl初始化

```C++
	if(getconf().getssl()){
		SSL_load_error_strings ();
		SSL_library_init ();
		ctx = SP_SSL_CTX(SSL_CTX_new(SSLv23_method()),SSL_CTX_free);
		int r = SSL_CTX_use_certificate_file(ctx.get(), getconf().getsslcrtpath().c_str(), SSL_FILETYPE_PEM);
		if(r <= 0){
			LOG<<"ssl_ctx_use_certificate_file failed";
		}
		r = SSL_CTX_use_PrivateKey_file(ctx.get(), getconf().getsslkeypath().c_str(), SSL_FILETYPE_PEM);
		if(r <= 0)
			LOG<<"ssl_ctx_user_privatekey_file failed";
		r = SSL_CTX_check_private_key(ctx.get());
		if(!r)
			LOG<<"ssl_ctx_check_private_key failed";
	}
```

主要分为两个部分，一个是获取证书文件，一个是获取私钥，然后检验证书文件的正确性

### ssl握手

TLS/SSL握手，分为四步，这里以RSA算法实现密钥交换为例

* 第一步：客户端发送[client hello]消息。消息里面有客户端使用的 TLS 版本号、支持的密码套件列表，以及生成的**随机数（\*Client Random\*）**，这个随机数会被服务端保留，它是生成对称加密密钥的材料之一。
* 第二步：当服务端收到客户端的「Client Hello」消息后，会确认 TLS 版本号是否支持，和从密码套件列表中选择一个密码套件，以及生成**随机数（\*Server Random\*）**。接着，返回「**Server Hello**」消息，消息里面有服务器确认的 TLS 版本号，也给出了随机数（Server Random），然后从客户端的密码套件列表选择了一个合适的密码套件。随后，服务端发了「**Server Hello Done**」消息，目的是告诉客户端，我已经把该给你的东西都给你了，本次打招呼完毕。
* 第三步：客户端验证完证书后，认为可信则继续往下走。接着，客户端就会生成一个新的**随机数 (\*pre-master\*)**，用服务器的 RSA 公钥加密该随机数，通过「**Client Key Exchange**」消息传给服务端。生成完「会话密钥」后，然后客户端发一个「**Change Cipher Spec**」，告诉服务端开始使用加密方式发送消息。然后，客户端再发一个「**Encrypted Handshake Message（Finishd）**」消息，把之前所有发送的数据做个**摘要**，再用会话密钥（master secret）加密一下，让服务器做个验证，验证加密通信「是否可用」和「之前握手信息是否有被中途篡改过」。
* 第四步：服务器也是同样的操作，发「**Change Cipher Spec**」和「**Encrypted Handshake Message**」消息，如果双方都验证加密和解密没问题，那么握手正式完成。

而在代码中，就是一个函数SSL_do_handshake

### ssl读写

在进行http通信时，需要把read和write函数改成SSL_read和SSL_write