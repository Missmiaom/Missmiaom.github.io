---
layout:     post
title:      "C++ STL 之 deque"
subtitle:   " \"Standard Template Library\""
date:       2016-07-07
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
    - STL
---

> STL 常用容器类 deque 的详解

### deque 概述

顾名思义，*deque* 是一种双向队列， *vector* 是一种单项开口的连续空间， *deque* 则是双向开口的连续空间，即 *deque* 可以在其头尾两端插入或者删除元素。（ *vector* 理论上也可以，但是效率很低）

### deque 的基本操作

#### deque 的中控器，迭代器和缓冲区

*deque* 的实现引入了中控器的概念，**中控器** 是一个按序存储缓冲区的头指针的连续空间。简言之， *deque* 的中控器实际上是一个指针数组，其中每个元素称为 **node** 。 *node* 指向的是大小相同用以存储数据的缓冲区，缓冲区为连续存储空间。这样的设计很容易联想到十字链表，只不过 *deque* 中控器中的指针指向的是连续存储空间而不是链表。

由于 *deque* 的中控器设计，其迭代器比较复杂， *deque* 的迭代器需要指明当前数据所属的 *node* ，以及在缓冲区的位置 **cur** 。当迭代器进行递增和递减时，都需要判断是否需要从一个缓冲区调到下一个或者上一个缓冲区，所以迭代器也需要维护其所在缓冲区的头节点 **first** 和尾节点 **last** 。

<img src="https://leiyiming.com/img/in-post/post-STL/deque中控器.png"/>

#### deque 的数据结构

*deque* 拥有一个指向中控器的指针，以便管理维护。初始构造时， *deque* 的中控器会根据初始数据的数量来决定创建多少个 *node* （申请多大空间）。最少是8个，最多是"最大node数+2"，保证头尾各有一个 *node* 来保证双向扩充。

**例如**：缓冲区大小设置为32字节时，初始存放20个int型数据。初始数据总共占 20 * 4 / 32 = 3 （向上取整） 个缓冲区，即需要分配三个 *node*，此时不足8个，中控器创建8个 *node* （申请8个单位大小空间），然后选择正中间位置的三个 *node* 来存放三个缓冲区的指针。这样做的目的是尽量保证头尾都有相同的空间来扩展。

<img src="https://leiyiming.com/img/in-post/post-STL/deque初始.png"/>

*deque* 维护两个迭代器 **start** 和 **finish** ， 用于指向目前第一个数据和最后一个数据。当向头插入元素时， *start* 减一，如果 *start* 在一个 *node* 的边界时， *start* 就需要跳转到前一个 *node* 。 当 *start* 已经处于中控器的最左边的 *node* 时，如果这时往头部添加数据，就需要创建新的更大容量的中控器，并将原中控器内容拷贝到新中控器中。

*deque* 支持随即访问，通过[]操作符可以快速访问某个数据。其利用缓冲区之间的跳转以及缓冲区的顺序存储特性可以很方便的实现。

#### deque 的数据操作

```
void push_front(const T& x);              //在头部插入数据
void push_back(const T& x);               //在尾部插入数据
iterator erase(iterator position);        //移除迭代器所指数据，并返回下一个节点的迭代器
void pop_front();                         //移除头部数据
void pop_back();                          //移除尾部数据
void clear(...);                          //移除某个节数据或某个区间的数据
iterator insert(iterator position， const T& value);
                                          //在指定位置插入值为value的节点
```

---

### 后记

*deque* 在功能上合并了 *vector* 和 *list* ， 既支持随即访问，也支持两端插入和删除。其内部的插入删除操作简单，但因为中控器的存在，占用内存相对偏多，结构偏复杂。
