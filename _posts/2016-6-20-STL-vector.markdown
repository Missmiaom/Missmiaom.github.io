---
layout:     post
title:      "C++ STL 之 vector"
subtitle:   " \"Standard Template Library\""
date:       2016-06-20
author:     "Leiym"
header-img: "img/post-bg-2015.jpg"
tags:
    - C++
    - STL
---

> STL库中常用的容器类 vector 详解，包括基础用法以及内存空间的操作。

### vector 概述

*vector* 的数据安排以及操作方式与数组类似，唯一的区别在与数组是静态空间，一旦声明之后就无法自动改变空间大小，需要手动扩充空间大小。而 *vector* 是动态空间，随着元素的加入，其内部机制会自行扩充空间以容纳新元素。

### vector 基本操作

#### vector 迭代器

其迭代器为普通指针。

```
template <class T, class Alloc = alloc>
class vector{
...
protected:
  iterator start;             //表示目前使用空间的头
  iterator finish;            //表示目前使用空间的尾（最后一个元素的下一个地址）
  iterator end_of_storage;    //表示目前可用空间的尾
...
}
```

其中，*end_of_storage* 为 *vector* 的可用空间尾部，而不一定指向最后一个元素的下一个位置。详解见后续内存操作中。

#### vector 方法

```
iterator begin();        //返回使用空间头的迭代器
iterator end();          //返回使用空间尾的迭代器
size_type size();        //返回使用空间大小
size_type capacity();    //返回可用空间（已申请）大小
bool empty();            //返回是否为空
reference front();       //返回头元素的引用
reference back();        //返回尾元素的引用

void push_back();        //将元素压入末尾
void pop_back();         //将尾端向前移动一个单位，放弃最后一个元素
iterator insert( iterator loc, const TYPE &val );
                         //在指定位置前插入元素val
void insert( iterator loc, size_type num, const TYPE &val );
                         //在指定位置前插入num个元素val
void insert( iterator loc, input_iterator start, input_iterator end );
                         //在指定位置前插入[start,end]区间所有元素

iterator erase(iterator pos);
                         //清楚某位上的元素
void resize(size_type new_size);
                         //重新分配大小
void clear();            //清空所有元素
```

### vector 的内存管理

*vector* 的使用空间大小与占用空间大小不一样，当往其尾部添加数据时，如果空间不足，它并不只会申请要添加的数据占用的空间大小，而是申请已有空间大小的二倍。所以占用空间总是大于等于已使用空间大小的。

#### vector 内存申请规则

利用一段代码来解释其内存空间申请规则，为了方便观察，我将运行结果写在了每行代码之后，下同：

```
int main()
{
	int i;
	vector<int> tv(2,9);
	cout<<"size= "<<tv.size()<<endl;                           //size =  2
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 2

	tv.push_back(1);
	cout<<"size= "<<tv.size()<<endl;                           //size =  3
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 4

	tv.push_back(2);
	cout<<"size= "<<tv.size()<<endl;                           //size =  4
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 4

	tv.push_back(3);
	cout<<"size= "<<tv.size()<<endl;                           //size =  5
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 8

	tv.push_back(4);
	cout<<"size= "<<tv.size()<<endl;                           //size =  6
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 8

	return 0;
}
```

可以很直观地得到以下结果： **当其初始化时，容量等于大小。当加入新元素之后的大小小于容量时，无需改变空间大小。当加入新元素之后的大小大于容量时，容量会扩大到原容量的两倍。** 这样做避免了频繁地申请内存，移动数据，释放原内存。值得注意的是： **当 vector 的容量发生变化时，并不是在原有的空间接上新的空间，而是申请另外一块两倍大小的空间，所以指向原 vector 的迭代器会全部失效。**

#### vector 内存空间的释放

继续来看一段代码：

```
int main()
{
	vector<int> tv(4,9);
	cout<<"size= "<<tv.size()<<endl;                           //size =  4
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 4


	tv.erase(tv.begin());
	cout<<"size= "<<tv.size()<<endl;                           //size =  3
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 4

	tv.clear();
	cout<<"size= "<<tv.size()<<endl;                           //size =  0
	cout<<"capacity= "<<tv.capacity()<<endl;                   //capacity = 4
	return 0;
}
```

可以从运行结果发现： **不论是 erase ，还是 clear 方法，都只能删除元素，无法删除占有的内存。** 这样的设计是为了避免频繁的申请空间，所以当反复使用一个 vector 时，其优点就显现出来了。

这样不会导致内存的泄露吗？答案有可能会，虽然在 vector 的析构函数中会自动释放其申请的空间，但是当 vector 存放的是指针时，析构函数只会释放存放指针的空间，而不会释放指针指向的空间。所以如果要手动地释放内存空间怎么办呢？

##### swap 方法交换空间

```
	vector<int>(tv).swap(tv);

  vector<int>().swap(tv);
```

当使用 vector 的构造函数并且旧容器当参数时，生成的新容器只会申请旧容器所有元素大小的空间，并不会申请和旧容器占有空间一样大的空间，也就是没有申请旧容器没有使用的空间。然后使用 swap 方法，交换新旧空间，这样就可以达到释放未使用空间的目的。

同理，和空 vector 交换空间就可以达到释放旧容器的所有空间。

##### 遍历 vector 释放释放指针空间

当 vector 存放的是指针时，遍历 vector ，然后一个一个释放其存放的指针指向的空间也可以达到目的。

```
for (vector<int *>::iterator it = tv.begin(); it != tv.end(); it ++)
    if (NULL != *it)
    {
        delete *it;
        *it = NULL;
    }
v.clear();
```

---

### 后记

这篇博文记录了STL库的最常用的容器 vector 的用法和内存释放方法。
