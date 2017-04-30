---
layout:     post
title:      "搭建基于ROS的自主机器人平台（二）——基于mbed的STM32F4系列控制器"
subtitle:   " \"Tank\""
date:       2016-05-06
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Tank
    - ROS
---

> 从这篇博文开始我会详细地介绍各个部分开发过程。第一步介绍基于mbed的STM32F4系列控制器的底盘功能的开发。

---

### 总体思路

底盘的开发比较耗时的部分绝对是串口通信，特别是在linux下还没有windows下各种方便串口调试的工具。如果把大量的时间花在了串口通信显然是不划算的。这里我们用到了 **rosserial** 包，它是用python写的一个串口工具，能够很方便的实现底层与上位机的串口通讯，并且通信方式与ROS传统方式无异。

用这种方法的话，底盘完全就可以当成一个 node ，直接接收和发布相关的通信节点，对于整个系统的架构是非常清晰的。不过 rosserial 常被用于 *Arduino* ，STM系列的控制器只能使用 **mbed** 进行开发,才可以顺利地使用 rosserial 。

### mbed 简介

mbed是一个面向ARM处理器的原型开发平台，它具体包括免费的软件库（SDK），硬件参考设计（HDK）和在线工具（Web）三部分内容，各个部分的具体介绍如下：

SDK：mbed设计了一个硬件抽象层，从而屏蔽了不同 mcu 厂商提供了微处理之间的差异，对于用户来说，他只需要和这个硬件抽象层打交道即可，也就是说，用户基于mbed开发的应用可以很方便地更换使用不同厂商的arm微处理器，从而留给用户更多的选择。

HDK：HDK是mbed提供的硬件参考设计，它是面向用户开发设计的，所以HDK提供了统一了程序上载接口，单步调试接口，串口调试接口，用户无需购买其它硬件就可以开始软件开发工作。

WEB：为了省去用户开发环境安装的麻烦，mbed提供了一个完备的基于浏览器的微处理器软件开发环境，包括代码编写，程序编译，版本控制等功能，用户只要上网就可以开发，编译结果只要下载保存到mbed开发板上即可工作，非常方便。

mbed 本身提供了比较好用的网页编辑器，能够针对不同平台调用相应的硬件库，并且只需要一键编译就可以得到二进制文件，文件可通过浏览器下载。
[选择平台网址](https://developer.mbed.org/platforms/)
<img src="http://leiyiming.com/img/in-post/post-tank/mbedonline.png"/>

上图就是选择控制器的页面，mbed目前支持的只是一小部分的控制器。对于初学者来说，在linux下有这样类似IDE的环境是很不错的。

#### gcc4mbed

老司机还是得离线用，然后自己编译呀，要不然总是不放心的～～而且离线之后就可以针对自己的板子来修改库文件达到移植的目的了。

我们需要下载一个编译工具—— **gcc4mbed** ，[gcc4mbed下载地址](https://github.com/adamgreen/gcc4mbed)， 在这个网页下方有具体的安装方法，使用的时候只需要按照linux下安装的方法即可。

移植到新的设备： [添加新的设备到mbed](https://github.com/adamgreen/gcc4mbed/blob/master/notes/new_devices.creole#adding-new-devices-to-gcc4mbed)

gcc4mbed 例子：[例子](https://github.com/adamgreen/gcc4mbed/tree/master/samples)

makefile 配置选项： [makefile](https://github.com/adamgreen/gcc4mbed/blob/master/notes/makefile.creole#make-variables)

mbed API手册： [handbook](https://developer.mbed.org/handbook/Homepage)

下载工具则选用普通的JLink即可， 调试也是用JLink官网的 Debuger。具体下载地址可自行搜索，这里就不在赘述。

#### 移植时需要注意的点

我根据自己的板子型号来具体介绍一下移植过程中需要注意的点。板子芯片型号是 *STM32F405RG* 。

在移植的时候，我们需要在 gcc4mbed-master/external/mbed/libraries/mbed/targets/hal/TARGET_STM/TARGET_STM32F4 文件夹中新建一个针对我自己平台的文件夹。命名规则是 *TARGET_BROADNAME* BROADNAME 是给你的板子的一个命名，在写 makefiel 时也必须用这个名字。我这里选择了 *TARGET_RED_F405RG* 。

建立好了文件夹之后，把相近的 *TARGET_NUCLEO_F401RE* 文件夹中的内容拷贝过来。修改 PinNames.h 文件添加你需要重命名的引脚口。比如需要添加一个电机1的ADC，可以写成 `MOTOR1ADC   = PC_0 ` ， 这样程序中使用 MOTOR1ADC 就相当于在使用 PC_0 端口了。

同样的方法，我们在 gcc4mbed-master/external/mbed/libraries/mbed/targets/cmsis/TARGET_STM/TARGET_STM32F4 中再使用一次。不过这次添加的是跟硬件有关的信息了。比如时钟频率等等。对于几个重要的属性，一定要仔细修改，否则程序运行的时候会出莫名其妙的错误。（亲身经历：忘记修改时钟频率，导致波特率不对，串口连接失败，而且根本没有提示信息，搞了好久。。）

### rosserial

简单地介绍完 mbed ，接下来的重点则是 **rosserial_mbed** 了，参考网址：[rosserial_mbed配置方法](http://wiki.ros.org/rosserial_mbed/Tutorials/rosserial_mbed%20Setup)。

第一步，先下载 **rosserial** 包至可用的工作空间，并编译。

第二步，执行以下命令，生成 rosserial_mbed 库。**注意**：这里的 *ros-lib-dir* 是要存放库的地址，后面会用到。

`rosrun rosserial_mbed make_libraries.py ros-lib-dir`

第三步，添加相关环境变量。请分别把 *gcc4mbed-dir*, *ros-lib-dir* 替换为你的 gcc4mbed 所在的文件夹，和 rosserial_mbed 库所在文件夹。

```
$ export GCC4MBED_DIR=gcc4mbed-dir
$ export ROS_LIB_DIR=ros-lib-dir
```

第四步，修改 makefile ，添加外部库，示例如下：

```
PROJECT         := rosserial_mbed_HelloWorld
DEVICES         := NUCLEO_F401RE
GCC4MBED_DIR    := $(GCC4MBED_DIR)
USER_LIBS       := $(ROS_LIB_DIR)
NO_FLOAT_SCANF  := 1
NO_FLOAT_PRINTF := 1

include $(GCC4MBED_DIR)/build/gcc4mbed.mk
```

**PROJECT** 是项目名称，编译链接后生成的二进制文件会以其命名。

**DEVICES** 是目标平台，必需是已有的目标平台，而且可以添加多个，用空格符分隔。可以添加自己的设备，方法参考上面的网址。

**GCC4MBED_DIR** 是 gcc4mbed 的文件夹，这里用环境变量的值代替。

**USER_LIBS** 是用户使用的库的文件夹，用到了什么库，就需要在这里声明库的文件夹。

剩下的不需要改动，进行编译之后，会生成一个或多个和目标平台名相同的文件夹，其中就包含了编译链接产生的文件，其中就有可以下载到板子里的 *.bin* 或者 *.hex* 文件。

---

### 后记

测试用例就不详解了，不管是 mbed 还是 rosserial 中都有大量的例子，参照着写没有太大问题。这篇博文主要介绍了前期的准备工作，接下来就给大家详细介绍底盘的开发过程。
