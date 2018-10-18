---
layout:     post
title:      "wireshark 实时解密TLS数据方法"
subtitle:   " \"wireshark\""
date:       2018-10-18
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - tls
---

### 必要条件

---

* 能够获取服务器私钥文件
* 不使用 **ECDHE** 作为秘钥交换算法



### wireshark配置步骤

---

1.编辑 -> 首选项 ->  Protocols -> SSL。

![](http://pbs8zp0hz.bkt.clouddn.com/blog/image/wireshak-tls/1.jpg)

选择 SSL debug file，点击browse.. 选择debug文件保存位置。随便新建一个空文件即可。这个文件中会记录wireshark解密相关的log，如果解密失败的话可以尝试查看该文件。

2.RSA keys list -> edit.. 

![](http://pbs8zp0hz.bkt.clouddn.com/blog/image/wireshak-tls/2.jpg)

* IP address：服务器ip
* Port: 服务器TLS port
* Protocol：这里由于我需要解密TCP数据，所以填写 `tcp` 。如果需要解密https数据填写 `http`
* Key File：服务器秘钥文件

3.设置完毕之后再进行抓包，会发现原本的 TLS 数据变为了 TCP数据，并且在原本的Secure Sockets Layer下还有一条标记的TCP数据，点击后便可以在数据框看到解密之后的数据了。

![](http://pbs8zp0hz.bkt.clouddn.com/blog/image/wireshak-tls/3.jpg)



### Poco 不使用 ECDHE 设置方法

---

cipher-suit包含四个部分(将摘要算法并入认证算法的话也可以看作三个部分)，第一是密钥交换算法，第二是签名算法/验签算法，第三是摘要算法，第四是对称加密算法。

加密套件的协商会在TLS的握手阶段完成，客户端会在 `Client Hello` 将其支持的所有加密套件发送给服务端，服务端选择一组后在 `Certificate, Server Hello Done` 中传回给客户端。协商后的结果如下：

```
Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

Poco 设置不使用 ECDHE 秘钥交换算法的设置方法是构造 Context 时传入加密套件字符串中包含`!ECDHE` ：

````
new Poco::Net::Context(Poco::Net::Context::CLIENT_USE, "", "", "",Poco::Net::Context::VERIFY_STRICT, 9, false, "ALL:!ADH:!LOW:!EXP:!MD5:!ECDHE:@STRENGTH");
````

