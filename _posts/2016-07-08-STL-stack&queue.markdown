---
layout:     post
title:      "C++ STL 之 stack & queue"
subtitle:   " \"Standard Template Library\""
date:       2016-07-08
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
    - STL
---

> STL 常用容器 stack & queue 的详解

### stack & queue 概述

**stack** 是一种先进后出（First In Last Out， FILO）的数据结构。 **queue** 是一种先进先出（First In First Out, FIFO）的数据结构。 *stack* 只有一个出入口，它允许在其最顶端压入弹出数据。 *queue* 有一个入口与一个出口，它只允许在尾端压入数据，在首端弹出数据。 它们相同点都是不允许遍历，所以它们都没有迭代器。其次，它们都是以一个 *deque* 作为底层容器来实现特殊的功能，是一种 **adapter** ， 所以它们并不被视为容器类，而是容器适配器（container adapter）。

<img src="https://leiyiming.com/img/in-post/post-STL/stack.png"/>

<img src="https://leiyiming.com/img/in-post/post-STL/queue.png"/>

**stack & queue 实现**： *deque* 是一个双向开口的容器，所以只要将一端封闭，就可以很简单地实现 *stack* 所需的功能。 将 *deque* 一端的插入封闭，另一端的弹出封闭，就实现了 *queue* 的功能。

### stack & queue 基本操作

#### stack

```
reference top();                  //返回顶部的引用
void push(const value_type& x);   //压栈
void pop();                       //弹栈
size_type size();                 //返回栈大小
bool empty();                     //返回是否为空
```

#### queue

```
reference front();                //返回前端的引用
reference back();                 //返回后端的引用
void push(const value_type& x);   //压入队列
void pop();                       //弹出队列
size_type size();                 //返回队列大小
bool empty();                     //返回是否为空
```

---

### 后记

*stack* & *queue* 都是算法设计中常用的数据结构，STL 均默认基于 *deque* 实现它们。当然也可以用 *list* 作为底层容器实现它们，只需要类似  `stack<int, list<int>> istack` 的声明就可以了
