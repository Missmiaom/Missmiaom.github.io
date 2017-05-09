---
layout:     post
title:      "STL智能指针源码浅析"
subtitle:   " \"STL Smart Pointer\""
date:       2017-4-12
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
    - STL
---

> 本文从源码入手，浅析 STL中各种智能指针。源码版本： VS2015 C++

### auto_ptr

---

以下是 auto\_ptr 的部分源码：

```
template<class _Ty>
	class auto_ptr
		{	// wrap an object pointer to ensure destruction
public:
	typedef auto_ptr<_Ty> _Myt;
	typedef _Ty element_type;

	explicit auto_ptr(_Ty *_Ptr = 0) _THROW0()
		: _Myptr(_Ptr)
		{	// construct from object pointer
		}

	auto_ptr(_Myt& _Right) _THROW0()
		: _Myptr(_Right.release())
		{	// construct by assuming pointer from _Right auto_ptr
		}

	···

	template<class _Other>
		auto_ptr(auto_ptr<_Other>& _Right) _THROW0()
		: _Myptr(_Right.release())
		{	// construct by assuming pointer from _Right
		}

	_Myt& operator=(_Myt& _Right) _THROW0()
		{	// assign compatible _Right (assume pointer)
		reset(_Right.release());
		return (*this);
		}

	~auto_ptr() _NOEXCEPT
		{	// destroy the object
		delete _Myptr;
		}

	···

	_Ty *release() _THROW0()
		{	// return wrapped pointer and give up ownership
		_Ty *_Tmp = _Myptr;
		_Myptr = 0;
		return (_Tmp);
		}

	void reset(_Ty *_Ptr = 0)
		{	// destroy designated object and store new pointer
		if (_Ptr != _Myptr)
			delete _Myptr;
		_Myptr = _Ptr;
		}

private:
	_Ty *_Myptr;	// the wrapped object pointer
	};
```

auto\_ptr 只有一个指针成员，指向用于包裹的对象（资源），auto\_ptr 的特点在于拷贝构造函数 和 `operator=` 的实现。介绍这些之前首先来看一下 release 和 reset 两个成员函数。 release 成员函数作用是返回包裹资源的指针，并将成员指针指向空。reset 函数则是将成员指针释放，然后指向传入的资源（即重新设置指针）。

#### 资源独占性

auto\_ptr 的拷贝构造函数和 `operator=` 都调用了被复制对象的 release 的函数去设置自身的成员指针，所以在拷贝完成之后，被复制对象的成员指针将会指向空！这也就是说 **auto\_ptr 对象对资源的所有权是独占的！**，可以简单理解为不可能有两个或者以上的 auto\_ptr 指向同一个资源对象，它的复制也伴随着所有权的转移。

#### 无法用于数组指针

再看 auto\_ptr 的析构函数，是用 `delete _Myptr;` 直接释放成员指针，所以 **auto\_ptr 无法包裹数组指针！** ，否则在释放时则会引起内存泄漏。

#### 不能在STL容器中使用

由于 STL 中容器的元素必须具备拷贝构造和可赋值的特性，也就是说容器中对象可以进行安全的赋值操作，可以将一个对象拷贝到另一个对象，从而获得两个独立的，逻辑上相同的拷贝（这一点被运用到例如swap、sort等等函数中）。所以，在设计上 **auto\_ptr 是不能用在 STL容器中的！**，STL 容器也有相关措施去保障这一点，不要尝试写出类似 `std::vector<std::auto_ptr>` 的代码，编译器会主动报错。


### unique_ptr

---

auto\_ptr 确保同一时间资源只能被一个智能指针引用，但是复制过程中隐式地将原来指针失效的做法十分容易引起误会，并且由于删除操作的固定（直接使用delete释放指针）， auto\_ptr 并不支持包裹数组指针或者是包裹非动态申请的对象。C++ 11 新标准中引入了 *unique\_ptr* ，除了保证资源的独占性，它还有以下三个方面的特征。

```
template<class _Ty,
	class _Dx>	// = default_delete<_Ty>
	class unique_ptr
		: public _Unique_ptr_base<_Ty, _Dx>
	{	// non-copyable pointer to an object
public:
	typedef unique_ptr<_Ty, _Dx> _Myt;
	typedef _Unique_ptr_base<_Ty, _Dx> _Mybase;
	typedef typename _Mybase::pointer pointer;
	typedef _Ty element_type;
	typedef _Dx deleter_type;

	using _Mybase::get_deleter;

	constexpr unique_ptr() _NOEXCEPT
		: _Mybase(pointer())
		{	// default construct ···}

	constexpr unique_ptr(nullptr_t) _NOEXCEPT
		: _Mybase(pointer())
		{	// null pointer construct ···}

	_Myt& operator=(nullptr_t) _NOEXCEPT
		{	// assign a null pointer ···}

	explicit unique_ptr(pointer _Ptr) _NOEXCEPT
		: _Mybase(_Ptr)
		{	// construct with pointer
		static_assert(!is_pointer<_Dx>::value,
			"unique_ptr constructed with null deleter pointer");
		}

	unique_ptr(pointer _Ptr,
		typename _If<is_reference<_Dx>::value, _Dx,
			const typename remove_reference<_Dx>::type&>::type _Dt) _NOEXCEPT
		: _Mybase(_Ptr, _Dt)
		{	// construct with pointer and (maybe const) deleter&
		}

	···

	unique_ptr(unique_ptr&& _Right) _NOEXCEPT
		: _Mybase(_Right.release(),
			_STD forward<_Dx>(_Right.get_deleter()))
		{	// construct by moving _Right
		}

	_Myt& operator=(_Myt&& _Right) _NOEXCEPT
		{	// assign by moving _Right
			···
		}

	~unique_ptr() _NOEXCEPT
		{	// destroy the object
		if (get() != pointer())
			this->get_deleter()(get());
		}

	···

	unique_ptr(const _Myt&) = delete;
	_Myt& operator=(const _Myt&) = delete;
	};
```

#### 允许自定义删除器

首先，类的声明为 `template<class _Ty, class _Dx> class unique_ptr` 模板参数多出了一个类型 `_Dx` ，这个参数是用来作为删除器的仿函数类类型。 unique\_ptr 允许用户传入一个仿函数作为删除器，自定义智能指针在析构时所需要进行的操作。如果缺省，_Dx 的类型为 `default_delete<_Ty>`， 这是 stl 自定义的一个仿函数，功能就是简单的调用 `delete` 操作符删除指针参数。

我们也可以创建一个仿函数自定义删除器，去实现特定的功能，unique\_ptr 自定义删除器实现区域互斥锁：

```
//自定义删除器
struct  mutex_deleter
{
	void operator() (std::mutex* x)
	{
		x->unlock();    //解锁
	}
};

···
{
	//传入删除器类型
	std::unique_ptr<std::mutex, mutex_deleter> mutex_ptr(new std::mutex()); 

	mutex_ptr->lock();    //上锁
	···                   //互斥操作
	                      //不需要手动调用unlock
}//离开区域，智能指针调用删除器进行自动解锁
```

#### 不允许使用左值引用进行拷贝

在上面部分源码的最后两行，unique\_ptr 使用左值引用的拷贝构造函数和 `operator=` 操作符都被删除掉了（令 函数声明 = delete 即删除函数，C++ 11 的新特性）。只保留了直接通过裸指针和右值引用进行拷贝的接口，用户必须通过 `std::move` 函数来将左值引用转换为右值引用，才可以进行拷贝操作，被复制的智能指针一样会失效，但这就明确地告知用户被拷贝的指针将会失效。这类构造和赋值通常被称为 “move构造和move赋值”

```
//正确的构造方法
std::unique_ptr<int> ptr(new int(88));   //使用构造函数赋值
std::unique_ptr<int> ptr = new int(88);  //使用 “=” 进行拷贝赋值

//错误的拷贝用法
std::unique_ptr<int> copy_ptr(ptr);      //报错！不能使用左值引用进行拷贝
std::unique_ptr<int> copy_ptr = ptr;     //报错！不能使用左值引用行拷贝

//正确的拷贝用法，使用 std::move 使左值引用变右值引用
std::unique_ptr<int> copy_ptr(std::move(ptr));        // ptr 失效
std::unique_ptr<int> copy_ptr = std::move(ptr);       // ptr 失效
```

也由于拥有move构造和move赋值，**unique\_ptr 是可以放入 STL 容器**中的：

```
std::unique_ptr<int> up(new int(88));

std::vector < std::unique_ptr<int> > vec_ptr;
vec_ptr.push_back(std::move(up));                //使用move构造

std::cout << *vec_ptr[0].get() << std::endl;

if (up.get() != NULL)
	std::cout << *up.get() << std::endl;         
else
	std::cout << "NULL" << std::endl;            
```

#### 支持数组指针

unique\_ptr 还有另外的模板类型声明，`template<class _Ty,class _Dx> class unique_ptr<_Ty[], _Dx>`，添加了对数组的支持，相应缺省的删除器也会使用 `delete[]` 操作去释放整个数组的空间。

```
std::auto_ptr<int> autoptr(new int[3]);  //报错！
std::unique_ptr<int> unptr(new int[3]);  //正确
```

### _Ref_count_base

---

\_Ref\_count\_base 是实现引用计数的纯虚类，它被  shared\_ptr 和 weak\_ptr 两大引用计数智能指针的父类包含。它封装了引用计数的方法，并提供了销毁和删除的抽象接口。在看 shared\_ptr 和 weak\_ptr 的源码前有必要了解其大致的实现思路，以下是它的源码（省去了部分操作函数和重载函数）：

```
class _Ref_count_base
	{	// common code for reference counting
private:
	virtual void _Destroy() _NOEXCEPT = 0;
	virtual void _Delete_this() _NOEXCEPT = 0;

private:
	_Atomic_counter_t _Uses;
	_Atomic_counter_t _Weaks;

protected:
	_Ref_count_base()
		{	// construct
		_Init_atomic_counter(_Uses, 1);
		_Init_atomic_counter(_Weaks, 1);
		}

public:
	virtual ~_Ref_count_base() _NOEXCEPT
		{}	// ensure that derived classes can be destroyed properly
		
	void _Incref()
		{	// increment use count
		_MT_INCR(_Uses);
		}

	void _Incwref()
		{	// increment weak reference count
		_MT_INCR(_Weaks);
		}

	void _Decref()
		{	// decrement use count
		if (_MT_DECR(_Uses) == 0)
			{	// destroy managed resource, decrement weak reference count
			_Destroy();
			_Decwref();
			}
		}

	void _Decwref()
		{	// decrement weak reference count
		if (_MT_DECR(_Weaks) == 0)
			_Delete_this();
		}

	long _Use_count() const _NOEXCEPT
		{	// return use count
		return (_Get_atomic_count(_Uses));
		}

	bool _Expired() const _NOEXCEPT
		{	// return true if _Uses == 0
		return (_Use_count() == 0);
		}

	virtual void *_Get_deleter(const _XSTD2 type_info&) const _NOEXCEPT
		{	// return address of deleter object
		return (0);
		}
	};

```

#### 引用计数操作的原子性

\_Ref\_count\_base 包含了两个成员变量 \_Uses 和 \_Weaks ，他们均是 `unsigned long` 类型的整型数值，分别表示被共享引用的次数和weak引用的次数。有意思的是，他们是通过别名 \_Atomic\_counter\_t 来声明的，很明显提示这两个计数器是原子的。当构造时，\_Uses 和 \_Weaks 都被初始化为了 1 。

\_Incref, \_Incwref, \_Decref, \_Decwref 方法分别是用于增加和减少计数的接口，主要是调用了 MT\_INCR 和 \_MT\_DECR 两个宏，其定义如下：

```
#define _MT_INCR(x) \
_InterlockedIncrement(reinterpret_cast<volatile long *>(&x))
#define _MT_DECR(x) \
_InterlockedDecrement(reinterpret_cast<volatile long *>(&x))
```

可见，这两个宏其实是调用了 Windows 下原子增操作和原子减操作的接口，除了这些，设置初值和得到返回值也均是原子操作，所以计数操作都是原子操作，即 **share\_ptr 和 weak\_ptr 的引用计数操作是线程安全的！**

#### 销毁和删除接口

需要注意的是，计数减操作函数会做一个判断，如果执行减操作之后，引用计数值变为 0 时，则要调用相应的销毁函数或者删除函数来执行指针释放操作。而这个操作是纯虚的，提供给子类的接口，具体在 share\_ptr 和 weak\_ptr 中再讲解这两个接口。

```
virtual void _Destroy() _NOEXCEPT = 0;
virtual void _Delete_this() _NOEXCEPT = 0;
```

### shared_ptr

---

