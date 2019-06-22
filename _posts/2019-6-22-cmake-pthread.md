---
layout:     post
title:      "cmake中链接pthread报未定义错误的原因"
subtitle:   " \"cmake\""
date:       2019-6-22
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - cmake
---

### 解决方法

---

在CMakeFile中添加 `SET(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} "-pthread")` 可解决。

### 原因

---

>  可见编译选项中指定 -pthread 会附加一个宏定义 -D_REENTRANT，该宏会导致 libc 头文件选择那些thread-safe的实现；链接选项中指定 -pthread 则同 -lpthread 一样，只表示链接 POSIX thread 库。由于 libc 用于适应 thread-safe 的宏定义可能变化，因此在编译和链接时都使用 -pthread 选项而不是传统的 -lpthread 能够保持向后兼容，并提高命令行的一致性。
> --------------------- 
> 作者：createchance 
> 来源：CSDN 
> 原文：https://blog.csdn.net/createchance/article/details/11843015 
> 版权声明：本文为博主原创文章，转载请附上博文链接！

