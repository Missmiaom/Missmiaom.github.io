---
layout:     post
title:      "问题总结Q&A"
subtitle:   " \"Summary of problems\""
date:       2016-08-09
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - ROS
---

> 将最近一段时间的问题和解决方法记录下来

### Arduin & rosserial

#### Q：使用 Arduino Leonardo 和 rosserial 与上位机通信出现同步失败的错误提示。

**A：**

1. 在 loop 函数最后，一定要调用 *spinOnce* 函数。

2. 加上 *#define USE_USBCON* 编译预处理。

#### Q：使用 Leonardo 的定时器功能

**A：**

1. Arduino 官方库中没有定时器的接口，需要使用 *TimerOne* 库或者 *MsTimer2* 库，它们的区别是一个用了定时器1，一个用了定时器2。

2. Leonardo 板子中官方库函数中使用了定时器1，所以如果再使用 *TimerOne* 库，就会导致同步失败的问题或者一些不可预知的错误。推荐使用 *MsTimer2* 库

### ubuntu

#### Q：永久提升用户权限为ROOT用户

**A：**

1. 打开用户自动登录设置：System Settings -> User Accounts -> Automatic Login -> ON 。

2. 执行`sudo gedit /etc/passwd` 。

3. 将以用户名开头的那一行中的 *1000* 改为 *0* ，重启。

#### Q：VMware虚拟机网络桥接模式的设置

**A：**

1. 打开虚拟机的网络编辑器：编辑 -> 虚拟网络编辑器 -> 更改设置。

2. 选择 VMnet0 ， 选择桥接模式， 桥接到主机所使用的网卡。如下图：

<img src="http://leiyiming.com/img/in-post/post-STL/桥接模式设置.png"/>

3. 虚拟机 -> 设置 -> 网络适配器 -> 网络链接 -> 选择桥接模式即可。

<img src="http://leiyiming.com/img/in-post/post-STL/桥接模式设置2.png"/>

#### Q：依赖包无法下载导致下载包失败

**A：**

1. 更新软件源，推荐使用网易源或者中科大源

2. 如果更新软件源之后也无法解决问题，一般是由于某个依赖包版本过高，而 apt 在解决依赖问题时只会安装未安装的包和升级旧版本的包，不会降级已安装的包。所以当需要安装的包依赖于某个低版本的包时，就会出现依赖包无法安装导致安装失败。

3. 解决办法：用 apt-get install 下载无法安装的依赖包，直到出现 *需要 XXX 是 xxx 版本，而已安装的是 xxx 版本*，这样就可以定位到需要降级的包的名称和需要降级到的版本。

4. 打开新立得包管理器（Synaptic Package Manager），搜索需要降级的包名，然后点击 Package -> Force Version ， 选择 apt 提示的版本，然后点击 Apply 。最后安装时就不会出现依赖的问题了。

<img src="http://leiyiming.com/img/in-post/post-STL/降低版本.png"/>


### ROS & tk1

#### Q: 多机通信的设置

**A：**

1. 连接同一个局域网，记录每台机器的IP

2. 互 ping，保证两两之间可以ping通，**注意**：使用虚拟机时需要将网络设置为桥接模式，在同一个网段下才可以互ping成功。

3. 选择一台机器作为主机（运行roscore），其他机器为从机，并在所有机器的 *～/.bashrc* 文件末尾写入 *export ROS_MASTER_URI=http://MASTER_IP:11311* 。

4. 在每台机器上的 */etc/hosts* 文件中，加入以 **IP 主机名** 格式的标示。有几个机器就加几行。如下图：

<img src="http://leiyiming.com/img/in-post/post-STL/hosts.png"/>



#### Q： 使用 catkin 编译某一个包并下载

**A：**

`-DCATKIN_WHITELIST_PACKAGES="package1;package2"`

`-DCMAKE_INSTALL_PREFIX=/opt/ros/indigo`

#### Q： 查看 tk1 的 L4T 版本

**A：**

` head -n 1 /etc/nv_tegra_release `

#### Q： 报错：Jetson TK1 - modprobe: FATAL: Module nvidia not found.

**A：**

` echo "alias nvidia nvhost_vi" >> /etc/modprobe.d/nvidia.conf `

#### Q： 在 tk1 上编译 opencv2.4.12 并在编译选项中加入 cuda6.5

**A：**

` cmake -DWITH_CUDA=ON -DCUDA_ARCH_BIN="3.2" -DCUDA_ARCH_PTX="" -DBUILD_TESTS=OFF -DBUILD_PERF_TESTS=OFF CUDA_GENERATION=Kepler .. `

#### Q： 查看 opencv 版本

**A：**

` pkg-config --modversion opencv `

#### Q： 编译 rtabmap 配置 CMakeLists（在opencv2.4.12下）

**A：**

`FIND_PACKAGE(OpenCV 2.4.12 REQUIRED QUIET)`

`INCLUDE_DIRECTORIES("/usr/local/include")`

`LINK_DIRECTORIES("/usr/local/lib")`

#### Q： 使用双目相机跑 rtabmap

**A：**

1. 相机坐标系与世界坐标系：相机坐标系是：x轴指向正右，y轴指向正下，z轴指向正前。而 rtabmap 需要的坐标系为世界坐标系，即：x轴指向正前，y轴指向正左，z轴指向正上。

2. 利用 *static_transform_publisher* 发布坐标转换关系。绕x轴旋转-90°，绕z轴旋转-90°

```
<arg name="pi/2" value="1.5707963267948966" />
<arg name="optical_rotate" value="0 0 0 -$(arg pi/2) 0 -$(arg pi/2)" />
<node pkg="tf" type="static_transform_publisher" name="camera_base_link"
     args="$(arg optical_rotate) base_link camera_link 100" />
```

#### Q： 在 rtabmap 中使用 SURF 算法

**A：**

```

<node pkg="rtabmap_ros" type="rtabmap" name="rtabmap">
   <param name="SURF/GpuVersion" type="string" value="true"/>
</node>

```

#### Q： rtabmap 开启自定位模式，不建图

**A：**

```
<param name="Mem/IncrementalMemory" type="string" value="false"/>
```

#### Q： 查看 rtabmap 所有参数

**A：**

`rosrun rtabmap_ros rtabmap --params-all`

#### Q： 更改 rtabmp 显示参数

**A：**

修改 *launch/rgbd_gui.ini* 文件中对应参数。
