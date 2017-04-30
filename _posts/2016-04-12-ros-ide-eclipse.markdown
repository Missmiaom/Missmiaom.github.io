---
layout:     post
title:      "ROS IDE之eclipse配置方法"
subtitle:   " \"eclipse setting for ROS\""
date:       2016-04-12
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - ROS
    - eclipse
---


> 这篇博文给大家介绍ROS的重要IDE——Eclipse的配置方法。
***

### 安装eclipse并配置启动文件

不能通过apt-get的方式安装，而需要在官网下载客户端，并做相关配置。下载地址：[eclipse官网下载](http://www.eclipse.org/downloads/packages/eclipse-ide-cc-developers/mars2)。（注意需要选择Eclipse IDE for C/C++ Developers 版本，并根据你的ubuntu系统选择32位或者64位）

下载完成后是没有产生启动图标的，在终端输入eclipse也会提示并没有安装。这时就需要创建一个启动图标，并设置参数。参考：[eclipse启动图标文件配置](http://www.blogs.digitalworlds.net/softwarenotes/?p=54)

完成上述教程后，应该就会生成eclipse图标，点击就可以进入，但这样无法eclipse装载当前环境变量，所以还需要更改一行配置。

- `$sudo gedit /usr/share/applications/eclipse.desktop`

将 *Exec=/opt/eclipse/eclipse* 这一行改为：

- `Exec=bash -i -c /opt/eclipse/eclipse`


### 创建eclipse工程

执行以下命令，记得将 *catkin_ws* 替换成你想要导入eclipse的工程所在的工作空间名：

1. `$ . ~/catkin_ws/devel/setup.bash`

2. `$ cd ~/catkin_ws`

3. `$ catkin_make --force-cmake -G"Eclipse CDT4 - Unix Makefiles" -DCMAKE_BUILD_TYPE=Debug -DCMAKE_ECLIPSE_MAKE_ARGUMENTS=-j8 .`

### 将工程导入eclipse

打开eclipse，在导入项目之前先做一些设置。点击 *Properties --> C/C++ Make Project --> Environment* ，添加以下环境变量：

- ROS_ROOT
- ROS_MASTER_URI
- ROS_PACKAGE_PATH
- PYTHONPATH
- PATH

注意，这些变量最好是在终端中使用echo命令查看，直接复制即可。如：

- $echo ROS_ROOT

然后更改编译选项，选择 *properties -> C/C++ general -> Preprocessor Include Paths, Macros etc*，点击 *Providers* 标签，在 *CDT GCC Built-in Compiler Settings [ Shared ]* 选项打上勾。 如下图：
<img src="http://leiyiming.com/img/in-post/post-ros/eclipse-setting.jpg"/>

最后，点击 *File --> Import --> Existing projects into workspace*，**记住，这里的选择的文件夹一定要是项目所在的工作空间的根目录！**

项目导入完毕后，需要等待一会以便eclipse进行代码分析，在左侧即可看到项目的文件树结构。

### 调试运行

编译的时候像普通项目一样，如果语法有错误就会报错。运行错误则需要单步调试。

选择 *Run –> Run configurations… –> C/C++ Application* ，双击或者点击 *New* ，选择 *Main* 标签，点击 *Search project* 选择已经编译好的二进制文件，**注意这里需要选择的是已经编译好的二进制文件，后缀是.bin，而不是源文件**。

最后点击 *Debug/Run* 就可以在eclipse中调试，并且可以设置断点来检查程序的运行错误。

### DIY eclipse代码格式以及配色

这个就不在赘述啦，网上有很多参考，在这里列出我觉得比较好的博文分享给大家。

[关闭英文拼写检查，关闭xml验证 ](http://blog.sina.com.cn/s/blog_70b623e4010173ce.html)

[Eclipse安装颜色主题，个性化你的IDE，让你的IDE焕然一新](http://www.open-open.com/lib/view/open1389410762742.html)

---

### 后记

eclipse功能强大，但配置方法较为复杂，在编译调试基于ROS的程序时经常出错，解决这些问题又需要一定的编程基础。下一篇博文为大家介绍更为轻量级，更为简单的QT IDE。
