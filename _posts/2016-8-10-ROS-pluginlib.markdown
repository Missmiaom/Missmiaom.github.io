---
layout:     post
title:      "pluginlib的使用总结"
subtitle:   " \"the pluginlib of ROS\""
date:       2016-08-10
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - ROS
---

> ROS 中十分重要的包 pluginlib 的解释

### 概述

**pluginlib** 包提供了一系列工具使得 ROS 程序可以动态的加载或者卸载插件。这里的插件一般指动态库中的功能类，插件的目的往往是类似的，但是达到目的的算法却不尽相同，装载不同的插件就可以使程序获得不同的表现。为了方便， *pluginlib* 允许程序根据插件名以及插件注册的路径动态地加载。这一过程是不需要重新编译的，所以配合 ROS 的动态参数的机制，可以很容易的实现只更改参数文件中装载的插件名，就可使用不同的算法，从而达到快速地对比各个算法的优缺点，以及找到最合适某个场景的算法的目的。

### 插件的编写方法

*pluginlib* 利用多态的特性，不同的插件如果实现统一接口，便可以互相更换。用户在使用插件时，调用统一的接口函数，更换插件时，不需要更改程序，也不需要重新编译。

所以，使用 *pluginlib* 的编写插件的方法可总结为以下四步：

1. 创建插件基类，统一接口。（若为现成的接口编写插件，则跳过该步）
2. 编写插件类，继承插件基类，实现统一接口。
3. 注册插件，并编译为动态库。
4. 使插件可用于 ROS 工具链。

这篇博文中，我分别将两个int型的加法运算和乘法运算编写为插件，并使用动态参数修改得到不同的运算结果。

#### 第一步：创建插件基类

创建 *calculate_base.h* 并声明插件基类 *Calculate*，定义接口函数，分别为 *initialize* 初始化函数，以及 *result* 运算函数。

```
namespace calculate_base
{
  class Calculate
  {
  public:
    virtual void initialize(int x, int y) = 0;
    virtual int result() = 0;
    virtual ~Calculate(){}

  protected:
    Calculate(){}
  };
}
```

#### 第二步：编写插件类

在　calculate_plugin.h　中，实现了两个插件类，分别是 *Add* 和 *Multiply*，它们均实现了基类 *Calculate* 的接口函数。这里为了方便，我将两个插件的实现写到了一个文件中，这并不影响它们成为两个插件，看完接下来的教程会发现，**插件是基于类的**，而不是基于动态库。

```
#include "calculate_base.h"

namespace calculate {
  class Add : public calculate_base::Calculate
  {
  public:
    Add(){}

    void initialize(int x, int y)
    {
      this->x = x;
      this->y = y;
    }

    int result()
    {
      return x + y;
    }

  private:
    int x;
    int y;
  };

  class Multiply : public calculate_base::Calculate
  {
  public:
    Multiply(){}

    void initialize(int x, int y)
    {
      this->x = x;
      this->y = y;
    }

    int result()
    {
      return x * y;
    }
  private:
    int x;
    int y;
  };
}
```

#### 第三步：注册插件，并编译为动态链接库

插件的实现完成后需要利用 *pluginlib*　提供的宏操作来注册为可使用的插件，并且需要编译为动态链接库。

在 *calculate_plugin.cpp* 中，　利用　*PLUGINLIB_EXPORT_CLASS* 宏来注册插件类。

```
#include <pluginlib/class_list_macros.h>
#include "../include/plugin_test/calculate_base.h"
#include "../include/plugin_test/calculate_plugin.h"

PLUGINLIB_EXPORT_CLASS(calculate::Add, calculate_base::Calculate)
PLUGINLIB_EXPORT_CLASS(calculate::Multiply, calculate_base::Calculate)
```

使用 *PLUGINLIB_EXPORT_CLASS* 必须包含头文件 *pluginlib/class_list_macros.h*　。**宏操作的第一个参数为插件类的全名，第二个参数为插件类的基类全名**，这两个名字都　**包括命名空间！**　。

然后将其编译为动态链接库，在　*CMakeLists.txt*　中加入：

```
add_library(calculate_plugin
  src/calculate_plugin.cpp
)
```

然后执行 *catkin_make* 就生成了动态链接库，默认生成库路径为工作空间下 *devel/lib/* 文件夹。

#### 第四步：使插件可用于 ROS 工具链

这一步需要编写 XML 文件，以便 ROS 工具链能够找到插件的路径。新建 *calculate_plugin.xml* 内容如下：

```
<library path="lib/libcalculate_plugin">
  <class type="calculate::Add" base_class_type="calculate_base::Calculate">
    <description>This is a add plugin.</description>
  </class>
  <class type="calculate::Multiply" base_class_type="calculate_base::Calculate">
    <description>This is a multiply plugin.</description>
  </class>
</library>
```

*library* 标签指出了包含插件的动态库的相对路径，这里为 *lib* 文件夹下名为 *libcalculate_plugin.so* 动态链接库，*library* 标签中省去了库的后缀 *.so*。

*class* 标签指出了期望从动态库中导出的插件类。各个参数的含义： *type*，指明插件类的空间名::类名。 *base_class* ，指明插件的基类的空间名::类名。 *description* ，插件的描述。*name*，使用包名+插件名的方法标识插件。

编写完 *calculate_plugin.xml* 之后，还需要在包含插件的包的 *package.xml* 添加下面一行：

```
<export>
  <plugin_test plugin="${prefix}/calculate_plugin.xml" />
</export>
```

上面的标签名 *plugin_test* 必须是包含插件 **基类** 的包名， *plugin* 参数指明上面所编写的插件的 xml 文件。

到此为止，插件的编写编译注册已经全部完成，在 *source* 工作空间之后，执行一下命令，如果能看到插件的名称以及路径，就表明插件已经可以使用了。

```
rospack plugins --attrib=plugin pluginlib_test
```

<img src="http://leiyiming.com/img/in-post/post-ros/查看插件.png"/>


注： *pluginlib_test* 为我的插件所在的包名，换成你想查找的包名，就可以查看这个包中的插件。

### 使用插件

 *plugin_loader.cpp* ：

```
#include <ros/ros.h>
#include <pluginlib/class_loader.h>
#include "../include/plugin_test/calculate_base.h"

int main(int argc, char** argv)
{
  ros::init(argc, argv, "plugin_test");
  ros::NodeHandle nh;

  int x = 3, y = 5;
  std::string cal;
  if(!nh.getParam("/calculate", cal))
  {
    ROS_ERROR("Get calculate ERROR");
    return 0;
  }

  pluginlib::ClassLoader<calculate_base::Calculate> calculate_loader("plugin_test", "calculate_base::Calculate");

  try
  {
    boost::shared_ptr<calculate_base::Calculate> calculate = calculate_loader.createInstance(cal);
    calculate->initialize(x, y);

    ROS_INFO("Result : %d", calculate->result());
  }
  catch(pluginlib::PluginlibException& ex)
  {
    ROS_ERROR("The plugin failed to load for some reason. Error: %s", ex.what());
  }

  return 0;
}
```

下面是分段解释：

```
#include <pluginlib/class_loader.h>
#include "../include/plugin_test/calculate_base.h"
```

必须包含 *pluginlib/class_loader.h* 和插件基类的头文件。

```
pluginlib::ClassLoader<calculate_base::Calculate> calculate_loader("plugin_test", "calculate_base::Calculate");
```

构造一个 *ClassLoader* 对象去装载插件，两个参数分别为 1.插件所在包名；2.插件基类全名。

```
boost::shared_ptr<calculate_base::Calculate> calculate = calculate_loader.createInstance(cal);
calculate->initialize(x, y);
```

调用 *ClassLoader* 对象的 *createInstance* 函数根据 rosparam 设置的参数获得不同的插件的实例。最后调用相关方法，来得到结果。

<img src="http://leiyiming.com/img/in-post/post-ros/plugin运行结果.png"/>

可以看到上图中，只设置的参数 *calculate* 的值便得到了不同的结果。即使用了不同的插件。

---

### 后记

记录了 *pluginlib* 的使用方法和机制，希望能够帮助到大家！
