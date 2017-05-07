---
layout:     post
title:      "Effective C++ 读书笔记"
subtitle:   " \"Effective C++\""
date:       2016-12-19
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - C++
    - 读书笔记
---

### 让自己习惯C++

#### 尽量以 const，enum，inline 替换 #define

> Prefer consts,enums, and inlines to #define

使用 `#define` 定义常量时（`#define PI 3.14`），编译器在预处理阶段即将常量名（PI）替换为常量值（3.14），而不是将常量名（PI）放入符号表中。当运用此常量如果获得一个编译错误，错误信息中提到的是常量值（3.14）而不是常量名（PI），这会浪费因为追踪它而花费的时间。而使用 `const` 定义常量时则不会出现上述情况。

当需要创建一个 class 专属常量时，可以使用私有枚举类型来实现，因为枚举类型的数值可以充当 `int` 而被使用。

```
class GamePlayer {
private:
  enum { NumTurns = 5 };   //"the enum hack" —— 令 NumTurns 成为 5 的一个记号名称
  int scores[NumTurns];    //合法
};
```

函数宏因为是直接替换参数，所以很容易发生不安全的行为：

```
#define CALL_WITH_MAX(a, b) f( (a) > (b) ? (a) : (b) )   //以a和b的较大值调用f

int a = 5, b = 0;
CALL_WITH_MAX(++a, b);             //a被累加两次
CALL_WITH_MAX(++a, b+10);          //a被累加一次，a的递增次数不确定！
```

谨记：

* 对于单纯常量，最好以 `const` 对象或 `enum`s 替换 `#define`s 。

* 对于形似函数的宏（macros），最好改用 `inline` 函数替换 `#define` 。

#### 尽可能使用 const

> Use const whenever possible

```
char* p = "Hello";              //non-const pointer, non-const data
const char* p = "Hello"         //non-const pointer, const data
char* const p = "Hello"         //const pointer, non-const data
const char* const p = "Hello"   //const pointer, const data
```

#### 确定对象被使用前已先被初始化

> Make sure that objects are initialized before they're used

谨记：

* 为内置型对象进行手工初始化，因为C++不保证初始化它们。

* 构造函数最好使用成员初值列，而不要在构造函数本体内使用赋值操作。初值列列出的成员变量，其排列次序应该和它们在class中的声明次序相同。

* 为免除“跨编译单元之初始化次序”问题，请以 local static 对象替换 non-local static对象。

### 构造、析构、赋值运算

#### C++隐式声明的函数

> Know what functions C++ silently writes and calls

当一个类没有显式地声明 *构造函数*，*copy构造函数*，*copy assignment操作符* 和 *析构函数* 任意一个或几个时，编译器会隐式地为其生成相应 **公有的** **非虚** 函数。

那么，编译器所隐式声明的函数做了什么呢？

 *default构造函数* 和 *析构函数* 依次调用 base class 和 non-static 成员变量的构造函数和析构函数。而 *copy构造函数* 和 *copy assignment操作符* 只是单纯地将对象的每一个 non-static 成员变量拷贝到目标对象。

 **例外**：当类中含有引用成员时，即使没有显式地声明 *copy assignment操作符*，编译器也不会为其隐式地声明该函数。因为 *copy assignment操作符* 会将一个引用值赋值给一个已经初始化的引用，这是非法的！（C++禁止引用改指向不同对象）。

 另外，当一个类的 *copy assignment操作符* 声明为 private 时，编译器不会为其所有子类隐式声明 *copy assignment操作符* ，因为编译器无权调用基类的 *copy assignment操作符*。

#### 拒绝C++隐式声明的拷贝构造函数和赋值操作符

> Explicitly disallow the use of compiler-generated functions you do not want

当一个类的设计初衷就是禁止拷贝的话，C++隐式声明的 *copy构造函数*，*copy assignment操作符* 函数将会违背初衷。

解决办法：

1. 显式地只声明 *copy构造函数*，*copy assignment操作符* 为 private ，而不做任何实现。

    ```
    class NonCopy{
    public:
      ...
    private:
      ...
      NonCopy(const NonCopy&);                 //只有声明
      NonCopy& operator=(const NonCopy&);      //外部无法调用，内部函数调用时，会因没有实现而在编译期报错
    }
    ```

2. 私有继承满足第一点的基类。

    ```
    class Example：private NonCopy{
      ...                                       //无需做任何动作
    }
    ```

#### 为多态基类声明 virtual 析构函数

> Declare destructors virtual in polymorphic base classes

C++ 允许使用基类指针指向派生类，并通过基类指针访问派生类中的函数，典型的例子便是factory（工厂）模式。如果指针由 `new` 在 heap 上动态生成的话，必须使用 `delete` 去释放动态生成的空间，否则很可能造成内存泄漏。

派生类被释放时，总是先调用最下层派生类的析构函数，然后从下往上依次调用各个基类的析构函数。使用指向派生类的 **基类指针** ，当其被释放时，首先调用的是基类的析构函数。所以，如果基类的析构函数是 **非虚** （non-virtual）的话，**子类的析构函数将不会被调用，即子类的成员将不会被释放从而导致内存泄漏。**

谨记：

* 带有多态性质的基类应该声明一个 virtual 析构函数。如果一个类带有任何 virtual 函数，它就应该拥有一个 virtual 析构函数。

* 相反，如果一个类的设计目的不是作为基类，那么，它就不应该声明 virtual 析构函数。

#### 析构函数不要抛出异常

> Prevent exceptions from leaving destructors

谨记：

* 析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕获异常，然后吞下它们（不传播）或结束程序

* 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作。


#### 绝不在构造和析构过程中调用 virtual 函数

> Never call virtual functions during construction or destruction

如果基类的构造函数或者析构函数中调用了虚函数会出现什么情况？

子类在构造时会先调用基类的构造函数，如果基类的构造函数中调用了虚函数，由于此时并没有调用子类的构造函数，所以虚函数不会下降至子类，而是调用基类的虚函数。这就会导致不明行为，如果基类构造函数调用的是纯虚函数的话，程序还会因为找不到纯虚函数的实现而导致连接出错。

在基类的构造函数被执行期间，子类对象会被C++认为是基类类型（因为此时子类并没有被构造），所以调用虚函数时，C++会调用基类的虚函数。基类析构函数中调用虚函数的问题也是如此，在子类的析构过程中，总是先调用子类的析构函数，再调用基类的构造函数。当基类的构造函数被调用时，子类已经被析构，所以C++会认为该对象是基类类型，如果此时调用虚函数，则是调用基类的虚函数，而不是下降调用子类的虚函数。

解决方法：

将在基类中调用的虚函数改为非虚，并要求子类的构造函数传递必要信息给基类构造函数。

```
class Base
{
public:
  Base(string& info)
  {
    func(info);               //non-virtual
  }

  void func(string& info);    //non-virtual
};

class Derived : public Base
{
public:
  Derived(parameters)
   : Base(createInfo(parameters))         
  {}

private:
  static string createInfo(parameters);   //使用static函数，避免传递参数时与子类对象未构造的问题
};
```

#### 令 operator= 返回一个 *this 的引用

> Have assignment operators return a reference to *this

赋值是可以写成连锁形式的，比如：`x = y = z = 15` ，`=` 其实采用右结合律，所以刚才的式子也可以写成 `x = (y = (z = 15))` 。所以为了实现连锁赋值，你为class实现 `operator=` 时就应该返回一个 `*this` 的引用。例如：

```
class Widget
{
public:
  Widget& operator=(const Widget& rhs)
  {
    ···
    return *this;
  }
};
```

同样，例如 `operator+=` 等其他赋值相关运算都使用。这只是个协议，并无强制性。不过标准库提供的类型均遵守了这个协议。

#### 在 operator= 中处理“自我赋值”

> Handle assignment to self in operator=

当类中包含动态申请的资源时，自我赋值可能会掉入“在停止使用资源之前意外释放了它”的陷阱，例如：

class Widget
{
  ···
private:
  Resource* ptr;   //指针，指向一个从heap分配而得的对象
};

Widget& Widget::operator=(const Widget& rhs)  //不安全的赋值
{
  delete ptr;                                 //释放当前的Resource
  ptr = new Resource(*rhs.ptr);               //使用rhs的Resource副本
  return *this;
}

如果使用上面的赋值操作实现自我赋值时，就会出现使用一个已经被释放的指针！

解决办法：

一、先进行 “证同测试”

只需要在最前面添加上 `if(this == &rhs) return *this` 即可，如果是自我赋值则不做任何事。这种方式可以解决问题，但是自我赋值的几率非常低的时候，证同测试会带来效率上的下降。

二、在复制ptr所指向的资源之前不要删除ptr

```
Widget& Widget::operator=(const Widget& rhs) 
{
  Resource* pp = ptr;                         //记住原先的ptr
  ptr = new Resource(*rhs.ptr);               //令ptr指向rhs的Resource的一个副本
  delete pp;                                  //删除原先的ptr
  return *this;
}
```

三、copy and swap

class Widget
{
  ···
  void swap(Widget& rhs);                     //交换*this和rhs的数据
};

Widget& Widget::operator=(const Widget& rhs)  //不安全的赋值
{
  Widget temp(rhs);                           //为rhs数据制作一份副本
  swap(temp);                                 //交换*this数据和上述副本数据
  return *this;
}

谨记：

确保当对象自我赋值时 `operator=` 有自我良好行为。其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及copy and swap。

确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

#### 复制对象时勿忘其每一个成分

> Copy all parts of an object

当你尝试显式地声明拷贝构造函数和拷贝复制操作符时，一定要记住复制每一个成分，否则会导致使用了不明确的行为，并且这样做不会引起编译器报错。

同样，在子类中显示声明拷贝构造函数和拷贝复制操作符时，也必须拷贝父类中的成员，即调用父类的拷贝复制操作符进行父类成员的拷贝。例如：

```
class Base
{
public:
    ···
    Base& operator=Base(const Base& rhs){}
};
class Derived : public Base
{
public:
    ···
    Derived& Derived::operator=(const Derived& rhs)
    {
        // 防止自赋值
        if(this == &rhs)
            return *this;

        // 调用父类赋值操作符的第一种方法
        Base::operator=(rhs);

        // 调用父类赋值操作符的第二种方法，对*this的Base部分赋值
        static_cast<Base&>(*this) = rhs;

        // 子类成员赋值
        ···

        return *this;
    }
}
```

谨记：

* 拷贝函数应该确保复制 **对象内的所有成员变量** 以及 **所有父类成员**

* 不要尝试以某个拷贝函数实现另一个拷贝函数，应该讲共同的代码放进第三个函数中，由两个拷贝函数共同调用。

### 资源管理

#### 以对象管理资源

> Use objects to manage resources

C++中常常会出现因为忘记释放动态申请的资源而导致内存泄漏的问题，问题的解决办法是引入一种以对象管理资源的概念，**RAII**。

**RAII**：Resource Acquisition Is Initialization。字面上的意思是“资源获取即是初始化”。它有两个关键的想法：

1. 获得资源后立刻放进管理对象内。我们可以在获取资源之后初始化某个管理对象，也可以拿来赋值某个管理对象。

2. 管理对象运用析构函数确保资源被释放。不论控制流如何离开区块，一旦对象被销毁（例如对象离开作用域），其析构函数自然会被自动调用。所以将资源的释放放入管理对象的析构函数中，即可保证资源被释放。

谨记：

* 为防止资源泄漏，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放资源。

* 两个常被是用的 RAII class 分别是 shared\_ptr 和 auto\_ptr 。前者通常是较优选择，因为其copy行为比较直观。如选择 auto\_ptr ，复制动作会使被复制的 ptr 指向 null。

#### 在资源管理类中小心 copying 行为

auto\_ptr 和 shared\_ptr 适合管理 *heap-based* 资源上，即动态申请的资源，因为它们最终需要使用 `delete` 来释放资源。而针对于不是 *heap-based* 资源，就需要建立自己的资源管理类。比如：想建立一个 RAII 的互斥锁类，在类对象被构造时即上锁，在被析构时解锁。

```
class Lock{
public:
  explicit Lock(Mutex* pm)
    : mutexPtr(pm)
    { lock(mutexPtr); }         //获得资源（上锁）

  ~Lock(){ unlock(mutexPtr); }  //释放资源（解锁）

private:
  Mutex *mutexPtr;
};
```

当建立自己的资源管理类时，必须小心复制行为。有以下四种针对复制行为的解决办法：

**禁止复制**：许多时候允许 RAII 对象被复制并不合理，所以可以将 copying 操作声明为 private 。

**对底层资源使用“”引用计数法**：有时候我们希望自己建立的管理类能够像 shared\_ptr 一样当它最后一个使用者被销毁时自动释放资源。 但是 shared\_ptr 的缺省行为是“引用次数为0时删除其所指物”，上面的例子中，我们需要的是解除锁定而不是删除。

shared\_ptr 在构造时可以传入一个函数作为删除操作，如果缺省即为 `delete` 操作，我们可以传入指定的删除操作来达到目的。

```
class Lock{
public:
  explicit Lock(Mutex* pm)
    : mutexPtr(pm, unlock)        //以unlock函数为删除操作  
    { lock(mutexPtr.get()); }     //获得资源（上锁）

private:
  shared_ptr<Mutex> mutexPtr;     //以 shared_ptr 替换纯指针
};
```

注意，此例中 Lock 类不需要再声明析构函数，因为当 mutexPtr 的引用计数为0时，会自动调用 unlock 来解除锁定，这样也使得 Lock 类可以被安全地复制。

**复制底部资源**：即使用深度拷贝复制一个一模一样的副本，这样只要管理类在被析构时确保资源得到释放，即可保证每个副本资源的安全释放。

**转移底部资源的拥有权**：在某些特殊的场合你可能希望永远只有一个 RAII 对象指向一个资源，如果 RAII 对象被复制时，资源的拥有权会从被源对象转移到目标对象。这即是 auto\_ptr 奉行的复制意义。

