---
layout:     post
title:      "搭建基于ROS的自主机器人平台（一）——开发计划总览"
subtitle:   " \"Tank\""
date:       2016-05-05
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Tank
    - ROS
---

> ROS的学习已经有一年了，大多数时间都在看wiki和各种书，真正动手的机会不多。在老师的带领下，我们就开始了搭建基于ROS的自主机器人平台的项目。接下来的一系列博文中，我会详细介绍从底层到通信层再到上层的架构方法，并且会穿插介绍调试校准的方法，希望对大家搭建自己的机器人有帮助。

---

### 摘要

此次开发的目的是针对履带式底盘开发出完整的自主导航机器人，机器人基于ROS架构，能够实现SLAM，目标识别，交互控制等功能。第一步，针对基于mbed的STM32F4系列控制器开发机器人底层功能，如：控速，反馈，里程测量等；第二步，利用rosserial完成上位机与控制器的通信，并且实现Navigation Stack所需接口；第三步，开发上层功能包，完成交互控制，自主建图，仿真等功能。第四步，针对特定要求，加入目标识别功能，加入GPS加强室外导航准度。

<img src="http://leiyiming.com/img/in-post/post-tank/tank.jpg"/>


### ROS Navigation Stack 简介

#### 功能

Navigation Stack能够根据机器人的里程计数据，传感器数据完成自定位并测绘地图，还能做出路径规划使机器人安全地自主地抵达目标地点。

#### 话题结构图解

<img src="http://leiyiming.com/img/in-post/post-tank/overview_tf.png"/>

上图是Navigation Stack中各个包之间的话题关系。小方框和椭圆代表的是各个部件，箭头上标出了相连部件之间通信的话题名称以及消息类型。中间的大方框代表非常重要的move_base包，其中包含了全局地图，局部地图，全局路径以及局部路径，还有恢复机制，其中的结构我们暂时不需要了解。以大方框为界可以看到有五个话题输入（"tf", "odom", "/map", "sensor topics"），一个话题输出("cmd_vel")。输入的话题对应着五个需要的接口：tf转换关系，odom里程信息，map地图数据，传感器数据；一个输出的话题即底盘的速度控制。

各个部件由不同的颜色划分重要级。**白色** 代表必需但已经实现的部件，**灰色** 代表可选且已经实现的部件，**蓝色** 代表必需并且是针对硬件平台的部件。所以底层开发至少需要实现以下功能：

* 解析cmd_vel话题数据，并将相应的速度命令送达给电机执行

* 将速度反馈累加，计算成odom里程信息，并传达给ROS上层

* 完成机器人的tf构建

* 发布必要的传感器数据，一般是激光数据（sensor_msgs/LaserScan），或者是点云数据（sensor_msgs/PointCloud）

### 底层必需的接口实现

#### 接收 *cmd_vel* 话题，并将其转换为左右轮速

消息类型：geometry_msgs/Twist

消息格式：

```
geometry_msgs/Vector linear

geometry_msgs/Vector angular
```

说明：消息中 linear 和 angular 分别为线速度和角速度，都是3维向量，每一维分别表示x,y,z方向上的值。cmd_vel是 move_base 发布的速度控制话题，底层需要将其解析为左右轮速并转换成相应 PWM 信号发送给电机。

#### 发布 *odom* 话题，提供粗略的里程信息

消息类型：nav_msgs/Odometry

消息格式：

```
std_msgs/Header header

string child_frame_id

geometry_msgs/PoseWithCovariance pose

geometry_msgs/TwistWithCovariance twist
```

说明：实现较为精准的自定位是一个很重要同时难度也很大的问题，Navigation中主要是由激光数据和底盘的里程数据来融合定位。计算里程信息需要将底层反馈的速度值积分得到实时的位姿和速度。 pose 表示的是实时的粗略的位姿， twist 表示的是实时的速度。 header 包含了时间戳，以及父坐标系等信息。 child_frame_id 表示子坐标系。

### 根据底盘配置Navigation Stack

#### TF 配置

系统在运行过程中必须知道各个坐标系之间的转换关系，这一部分由 TF tree 来管理。简单的实现方法是调用 robot_state_publisher 包，输入建模好的 urdf 文件，就可以自动往外广播 TF 消息。所以这部分主要工作在于 urdf 的建模。

#### 传感器数据采集

这里的传感器指激光雷达或者深度相机。由于单片机的性能问题，这部分拿到上位机来做，主要是由相关设备的驱动来完成。所以选择设备时，最好选择有相应的ROS驱动的设备。这部分主要工作在于调试设备的参数。

#### 多传感器融合定位

自定位是很重要的一部分，自定为的精度直接影响到建图导航的准确性。单靠一个传感器可能是不够的，尤其是采用 kinect 之类只有小角度的景深设备。这时候就需要用到多传感器融合的方法，ROS中提供了一个非常好的包—— robot_pose_ekf ，其利用扩展卡尔曼滤波，融合里程，激光，GPS数据来定位。这里的GPS是可选的。

### 调试方法

第一步：只调试底盘。包括测试底盘速度控制以及反馈速度是否准确， IMU ， 码盘等机械参数设置等等

第二步：调试底层和上位机之间的通信，即 rosserial 的工作状况。有时候会产生数据量过大，波特率设置太小导致连接超时等问题。

第三步：确定底层和通信层无误后，针对环境要求，机械要求调试上层包的参数。

---

### 后记

有详细的计划之后，开发起来就不会浪费时间浪费精力。
