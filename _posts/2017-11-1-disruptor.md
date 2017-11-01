---
layout:     post
title:      "disruptor C++ 用法浅析"
subtitle:   " \"disruptor\""
date:       2017-11-1
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - disruptor
    - C++
---

> disruptor 是一个高性能的异步处理框架，支持多线程间共享数据。disruptor 最初在JAVA上实现，这里介绍了disruptor C++ 版本实现的用法。

## 设计思想

---

disruptor 的底层数据结构是一个首尾相连的环（ring buffer），并且只维护一个用于指向下一个可用的位置的游标。

当需要写入时，disruptor 会分配一个空间，并将指针更新为该空间的最大下标处，这块空间就是申请写入的线程所独享的。当空间分配成功后，写线程可以慢慢写入。所以当有多线程同时写入时，需要同步的地方只是更新游标，而更新游标只是做加操作，利用原子变量的加操作就可以避免使用锁，从而提高效率。

当需要读取时，disruptor 会等待游标更新到想要读取的下标处，然后执行读操作，这个过程同样不需要加锁。disruptor 提供了安全的机制保证写入数据不会覆盖未处理的数据。

## 类结构

---

### Sequence

游标，通常用来记录可使用空间的下标。 

利用了缓冲行填充的方法来避免伪共享，提高效率，具体参见：[伪共享分析以及volatile与缓存行填充的应用](http://leiyiming.com/2017/08/25/falsesharing/)

提供读取、存储和增加值功能，基本是直接调用了`std`原子变量的基本操作。只不过调用的时候指定了内存屏障类型，尽量将内存屏障影响变小。内部外部都使用。

### RingBuffer

固定长度的环形缓冲区，本质就是包装了一个`std`的 `array` ，对外只提供下标取值操作(中括号运算符)，会对超过长度的序号自动取余来实现环形复用。内部使用。

### ClaimStrategy

用来写入数据的辅助工具类，主要负责分配空闲可写入空间，等待写入空间可使用状态。内部使用。 
有两个版本： 
- `SingleThreadedStrategy`：单线程写入版本，写入完成后不做等待。 

- `MultiThreadedStrategy`：多线程写入版本，写入完成后等待之前的空间成为可读状态才继续。**有多个线程写入的时候必须要使用此版本。**

### WaitStrategy

等待读取数据的工具类，主要负责等待可读取游标是否到达要读取的位置，可中断等待。内部使用。 
有四个版本： 
- `BusySpinStrategy`：不放弃cpu死循环等待 
- `YieldingStrategy`：死循环一定次数后调用`yield`放弃`cpu`时间片。 
- `SleepingStrategy`：死循环一定次数后重复调用`yield`放弃cpu时间片。调用`yield`一定次数后重复调用`sleep`睡眠指定时间。 
- `BlockingStrategy`：阻塞等待，内部使用了条件变量，需要写入的时候唤醒。

### SequenceBarrier

`WaitStrategy`的封装，但是保存了队列的游标状态数据。用来在等待可读数据游标到达指定位置，外部使用。

### Sequencer

包含了`RingBuffer`、`Sequence`、`WaitStrategy`、`ClaimStrategy`的封装类。统一封装了对外接口。



## 读写方法

---

### 创建 Sequencer 对象

首先需要构建一个 `Sequencer` 对象。`Sequencer`是一个模板类，可在模板参数中选择存储对象的类型、内部环形队列的长度、写入策略（ `ClaimStrategy` 版本）、读取策略（ `WaitStrategy` 类型）。 
**注意：模板参数中长度参数默认是1024，如果要设为其他值，那么必须明确指明ClaimStrategy类型的模板参数，并且设为同样的长度值**

例如下面的代码声明了一个存储 `int` 类型数据，存储空间为 8 ，写入策略使用 `MultiThreadedStrategy`，读取策略使用 `SleepingStrategy` 的 Sequence 对象。

```C++
constexpr size_t RING_SIZE = 8;
using ClaimStrategy = disruptor::MultiThreadedStrategy<RING_SIZE>;
using WaitStrategy = disruptor::SleepingStrategy<>;
using NumberSequencer = disruptor::Sequencer<int, RING_SIZE, ClaimStrategy, WaitStrategy>;
std::array<int, RING_SIZE> numberArray = { 0 };
NumberSequencer *sequencer = new NumberSequencer(numberArray);
```



### 写入

1. 首先调用 `Sequencer` 对象的 `Claim` 方法传入长度参数分配指定长度的写入空间，此方法返回的是可以写入的数据序号的最大一个。
2. 直接使用序号作为下标访问向队列中填入数据（注意返回的是最大的序号，要先减去长度然后依次写）。
3. 然后调用`Sequencer`对象的`Publish`方法传入刚才`Claim`方法返回值和长度参数，此时就完成了写入数据的完整过程。 

```C++
// 1.调用Claim方法，申请写入空间，返回申请到空间的最大的序列号
int seq = _sequencer->Claim(onceSize);

// 2.使用取值操作直接写入数据
for (int i = seq - onceSize + 1; i <= seq; ++i)
{
  (*_sequencer)[i] = _data;
}

// 3.调用Publish方法，确认已经写入数据的序列号
_sequencer->Publish(seq, onceSize);
```

**注意：因为内部是环形缓冲，如果写入的空间不足并且不加控制就会覆盖最前面的空间，如果其中有未读取的数据就会丢失** ，控制方法见下文。



### 读取

1. 调用`Sequencer`对象的`NewBarrier`方法创建一个用来读取数据的`SequenceBarrier`对象指针。这个对象将用来等待数据的到来和读取数据 。**注意：这个对象最后使用结束后需要手动调用delete删除。** 
2. 调用`SequenceBarrier`对象的`WaitFor`方法，传入期望获取的数据的序号。当队列中的可读数据序号超过或者等于传入的序号的时候，方法就会返回。返回值就是可读数据的最大的序号。使用序号作为下标即可读取数据。


```C++
int64_t seqGeted;

// 1.创建 SequenceBarrier 对象
_barrier = _sequencer->NewBarrier(std::vector<disruptor::Sequence*>());

// 2.调用 WaitFor 方法获取数据 
if ((seqGeted = _barrier->WaitFor(_seqWant)) >= 0)
{
  for (int i = _seqWant; i <= seqGeted; ++i)
  {
    std::cout << std::endl << "Consumer " + _name + " get: " << i;
    Sleep(2000);
  }
}
```
**注意：此处返回的是最大的可读数据序号，但是，没有办法知道从哪个序号开始的数据是没读过的，也就是说有可能此时你期望获取的数据已经被其他线程处理过了或者因为来不及处理已经被覆盖了**， 解决方法见下文



## 控制方法

---

### gating_sequences

门限序列，disruptor 就是它用来控制写入者不要覆盖未读取的数据的。

1. 每一个读取线程都需要维护一个 `Sequence` 对象来记录当前线程的处理进度，表示当前线程处理完的序列号。
2. 然后调用`Sequencer`对象的`set_gating_sequences`方法，传入一个存储每个线程进度的`Sequence`对象指针的`vector`的引用。
3. 在读取数据的地方，每次处理完数据后将已经处理完的数据序号更新到`Sequence`对象里。`Claim`方法内部就会每次检查更新后的 `Sequence` 指针的 vector ，查找其中最小值，然后一直等待此序列号的数据已经被处理完毕后再写入，防止覆盖未处理的数据。 



## 完整实例

---

util.h

```C++
#pragma once

#include "disruptor/sequencer.h"
#include "disruptor/ring_buffer.h"

//Ring buffer size 必须是2的N次方
constexpr size_t RING_SIZE = 8;// disruptor::kDefaultRingBufferSize;

using ClaimStrategy = disruptor::MultiThreadedStrategy<RING_SIZE>;
using WaitStrategy = disruptor::SleepingStrategy<>;
using NumberSequencer = disruptor::Sequencer<int, RING_SIZE, ClaimStrategy, WaitStrategy>;

```



NumberProducer.hpp

```C++
#pragma once
#include <iostream>
#include <thread>
#include <windows.h>
#include "disruptor/sequencer.h"
#include "disruptor/ring_buffer.h"
#include "util.h"

class Producer
{
public:
  	// 传入 Sequencer 指针，进行操作
    Producer(std::string name, NumberSequencer* sequencer, int data)
        : _name(name)
        , _sequencer(sequencer)
        , _stop(false)
        , _data(data)
    {
        _thread = new std::thread(&Producer::Start, this);
    }

    ~Producer()
    {
        _stop = true;
        _thread->join();
    }

    void Start()
    {
        int onceSize = 1;

        while (!_stop)
        {		
			// 1.调用Claim方法，申请写入空间，返回申请到空间的最大的序列号
            int seq = _sequencer->Claim(onceSize);
			
          	// 2.写入数据
            for (int i = seq - onceSize + 1; i <= seq; ++i)
            {
                (*_sequencer)[i] = _data;
                std::cout << std::endl << "Producer " + _name + " write: " + std::to_string(_data);
            }
			
          	// 3.调用Publish方法，确认已经写入数据的序列号
            _sequencer->Publish(seq, onceSize);
            std::cout << std::endl << "Producer " + _name + " write to : " << seq;

            Sleep(1000);
        }
    }

private:
    NumberSequencer                             *_sequencer;
    std::thread                                 *_thread;
    bool                                        _stop;
    int                                         _data;
    std::string                                 _name;
};

```



NumberConsumer.hpp

```C++
#pragma once
#include <iostream>
#include <thread>
#include <string>
#include <windows.h>
#include "disruptor/sequencer.h"
#include "disruptor/ring_buffer.h"
#include "util.h"

class Consumer
{
public:
    Consumer(std::string name, NumberSequencer* sequencer, disruptor::Sequence* handle)
        : _name(name)
        , _stop(false)
        , _handled(handle)								//需要维护的进度序列号
        , _sequencer(sequencer)							//传入 Sequencer 指针进行操作
        , _seqWant(disruptor::kFirstSequenceValue)		//期望读取的序列号
    {
        // 1.创建 SequenceBarrier 对象
        _barrier = _sequencer->NewBarrier(std::vector<disruptor::Sequence*>());
        _thread = new std::thread(&Consumer::Start, this);
    }

    ~Consumer()
    {
        delete _barrier;
        _stop = true;
        _thread->join();
    }

    void Start()
    {
        while (!_stop)
        {
            int64_t seqGeted;
          	// 2.调用 WaitFor 方法获取数据 
            if ((seqGeted = _barrier->WaitFor(_seqWant)) >= 0)
            {
                for (int i = _seqWant; i <= seqGeted; ++i)
                {
                    std::cout << std::endl << "Consumer " + _name + " get: " << i;
                    Sleep(2000);
                }
              	//更新期望序列号
                _seqWant = seqGeted + 1;
              	//维护当前进度序列号
                _handled->set_sequence(seqGeted);
            }
        }
    }

private:
    NumberSequencer                             *_sequencer;
    disruptor::SequenceBarrier<WaitStrategy>    *_barrier;
    disruptor::Sequence*                        _handled;
    int64_t                                     _seqWant;
    std::thread                                 *_thread;
    bool                                        _stop;
    std::string                                 _name;
};
```



main.cpp

```C++
#include "util.h"
#include "NumberConsumer.hpp"
#include "NumberProducer.hpp"

#include <array>
#include <memory>

int main()
{
    std::array<int, RING_SIZE> numberArray = { 0 };
    NumberSequencer *sequencer = new NumberSequencer(numberArray);
	
  	//为两个读线程创建两个进度序列号，初值为-1
    disruptor::Sequence* handle_c1 = new disruptor::Sequence(disruptor::kInitialCursorValue);
    disruptor::Sequence* handle_c2 = new disruptor::Sequence(disruptor::kInitialCursorValue);
	
  	//将 sequencer 的门限序列设置为保存有写线程进度序列号的序列
    std::vector<disruptor::Sequence*> sequences;
    sequences.push_back(handle_c1);
  	sequences.push_back(handle_c2);
    sequencer->set_gating_sequences(sequences);

  	//开启两个写线程
    std::shared_ptr<Producer> p1(new Producer("1", sequencer, 1));
    std::shared_ptr<Producer> p2(new Producer("2", sequencer, 2));
  
  	//开启两个读线程，分别传入进度序列号
    std::shared_ptr<Consumer> c1(new Consumer("1", sequencer, handle_c1));
    std::shared_ptr<Consumer> c2(new Consumer("2", sequencer, handle_c2));

    getchar();
}

```

