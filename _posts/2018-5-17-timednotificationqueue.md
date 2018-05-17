---
layout:     post
title:      "Poco::TimedNotification 使用小结"
subtitle:   " \"Poco TimedNotification\""
date:       2018-5-17
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
---





## 概览

---

`TimedNotificationQueue` 是一个以时间为优先级的先入先出通知队列。队列中保存的是 `Notification` ，并按照 `Timestamp` 排序。它适用于执行不定周期的任务，比定时器更加灵活。



## 用法

---

### 主要接口

```C++
//将通知以时间戳先后存入队列
void enqueueNotification(Notification::Ptr pNotification, Timestamp timestamp);
void enqueueNotification(Notification::Ptr pNotification, Clock clock);

//取出队列头部的通知。
//如果当前时刻等于或者大于队列头部的通知时刻，那么将返回头部的通知。
//如果当前时刻小于队列头部的通知时刻，那么将返回null。
Notification* dequeueNotification();

//等待并取出头部通知。
//如果当前时刻等于或者大于队列头部的通知时刻，那么将返回头部的通知。
//如果当前时刻小于队列头部的通知时刻，那么将等待直至可以取出头部通知为止。
Notification* waitDequeueNotification();

//多出一个等待时间参数，如果不满足取出条件，不会死等而是等待一定时长。
Notification* waitDequeueNotification(long milliseconds);

bool empty() const;
int size() const;
void clear();
```



### 示例

```C++
//继承Notification，自定义通知
class CustomizedNotification : public Poco::Notification
{
public:
    CustomizedNotification() = default;
    
    void doSomething() {}
};

//声明
Poco::TimedNotificationQueue queue;

//存入通知的线程
...
CustomizedNotification::Ptr pNf = new CustomizedNotification();
Poco::Clock timeStamp;
queue.enqueueNotification(pNf, timeStamp);
...

//取出通知的线程
...
while (!isCancelled())
{
    CustomizedNotification* pNf = dynamic_cast<CustomizedNotification*> (queue.waitDequeueNotification());
  	if (NULL != pNf)
  	{
		pNf->doSomething();
    	pNf->release();
  }
}
...
```



##  实现浅析

---

### 时间优先

`TimedNotificationQueue` 最大的特点就是以时间优先出队列，底层的数据结构其实就是 `multimap` ，利用其自动排序的特性实现时间优先。



### 取出等待

取出的操作无疑就是获取 `mutilmap` 的头部元素，然后比较当前时刻和该元素的时间戳，然后执行等待或者取出。

**Q：** 但是其接口  `waitDequeueNotification` 可以在没有到达头部通知时，阻塞等待至头部通知的时刻到达之后，再返回通知。那如果在等待期间有另外一个通知被插入到头部了呢？ `TimedNotificationQueue`  还是等待到原头部时刻然后取出吗。

**A：** 其内部的等待过程并不是通过sleep来实现的，而是通过 `Poco::Event` 来实现的，利用其 `tryWait()` 方法进行等待。当有新的通知入队列时，会调用 `Event::set()` 方法跳出  `tryWait()` 等待，然后重新进行取出头部、比较时间戳、等待取出的过程。

这里看一下具体的实现代码。

```C++
Notification* TimedNotificationQueue::waitDequeueNotification()
{
	for (;;)
	{
		_mutex.lock();
		NfQueue::iterator it = _nfQueue.begin();   //1.获得头部元素
		if (it != _nfQueue.end())
		{
			_mutex.unlock();
			Clock::ClockDiff sleep = -it->first.elapsed();  //2. 比较当前时刻与头部元素的时间戳大小
			if (sleep <= 0)
			{
				return dequeueOne(it).duplicate();   //3. 如果小于或等于就取出
			}
			else if (!wait(sleep))                   //4. 如果大于就执行wait，跳转到wait函数
			{
				return dequeueOne(it).duplicate();   //5. 等待完成，中间没有新的通知插入，取出
			}
			else continue;                           //6. 等待终止，中间有新的通知插入，进入下一个循环
		}
		else
		{
			_mutex.unlock();
		}
		_nfAvailable.wait();
	}
}

//wait函数
bool TimedNotificationQueue::wait(Clock::ClockDiff interval)
{
	const Clock::ClockDiff MAX_SLEEP = 8*60*60*Clock::ClockDiff(1000000); // sleep at most 8 hours at a time
	while (interval > 0)
	{
		Clock now;
		Clock::ClockDiff sleep = interval <= MAX_SLEEP ? interval : MAX_SLEEP;
		if (_nfAvailable.tryWait(static_cast<long>((sleep + 999)/1000)))  //重点:使用Event的tryWait等待
			return true;
		interval -= now.elapsed();
	}
	return false;
}

//入队列函数
void TimedNotificationQueue::enqueueNotification(Notification::Ptr pNotification, Clock clock)
{
	poco_check_ptr (pNotification);

	FastMutex::ScopedLock lock(_mutex);
	_nfQueue.insert(NfQueue::value_type(clock, pNotification));
	_nfAvailable.set();     //重点：新通知入队列时，调用set，跳出tryWait等待。
}
```

