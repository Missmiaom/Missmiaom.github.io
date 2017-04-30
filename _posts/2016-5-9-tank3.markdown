---
layout:     post
title:      "搭建基于ROS的自主机器人平台（三）——底层功能的开发"
subtitle:   " \"Tank\""
date:       2016-05-09
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Tank
    - ROS
---

> 上篇博客中，我介绍了 mbed 以及 rosserial，这边博客就利用这两个工具来进行机器人底层功能的开发。当然，这里提到了全部都是有关 ROS 的开发，像 PWM 控速， 码盘数据读取等开发我就省略了，只要自己做过小车的应该都会。

---

### 速度控制与反馈

ROS 的 Navigation Stack 能够根据记录的地图自动进行路径规划，产生的速度控制命令话题 /cmd_vel 。同时，底盘也需要将实际速度反馈给上层，这个反馈的话题是 /odom

### /cmd_vel

/cmd_vel 话题发布的消息类型是 *geometry_msgs/Twist* ，结构如下：

```
geometry_msgs/Vector linear
  float64 x
  float64 y
  float64 z

geometry_msgs/Vector angular
  float64 x
  float64 y
  float64 z
```

*linear* 表示线速度， *angular* 表示角速度。我们需要接收 /cmd_vel 话题，并将数据转换为左右轮速。这里我使用的是轨迹推演算法，具体推导请看 [princeton courses](https://www.cs.princeton.edu/courses/archive/fall11/cos495/COS495-Lecture5-Odometry.pdf)。

##### 设置轮速

由线速度和角速度推导出差速轮轮速的公式如下：

<img src="http://leiyiming.com/img/in-post/post-tank/轨迹推演.gif"/>

这里的 *Vl* ， *Vr* 为计算得到的左右轮速， *l* 为轮轴距， *V* 为 X 轴线速度， *w* 为 Z 轴角速度。

**注意**：ROS中的坐标系是右手坐标系，简单地说就是以机器人正前方为 *X* 轴正向，正左方为 *Y* 轴正向，正上方为 *Z* 轴正向。右手法则中四指的方向为旋转正向，即逆时针方向。如下图：

<img src="http://leiyiming.com/img/in-post/post-tank/右手法则.png"/>

OK，了解完这些，我们就可以开始了动手了。

```
//定义一个接受者，接收 cmd_vel 话题，回调函数是 subscribeVelocityCommand
ros::Subscriber<geometry_msgs::Twist> cmd_vel_sub("cmd_vel", &subscribeVelocityCommand);

/********cmd_vel 的回调函数********/
void subscribeVelocityCommand(const geometry_msgs::Twist& msg)
{
	double vl, vr;                                      //左右轮速
	double linear = msg.linear.x;                       //X轴线速度，单位，m/s
	double angular = msg.angular.z;                     //Z轴角速度，单位，rad/s

	double tmp = (angular * tank.getBias()) / 2.0f;     
	vr = 100.0f * ( linear + tmp );
	vl = 100.0f * ( linear - tmp );                     //计算左右轮速,单位，cm/s

	uint8_t vl_dir = Forward;                           //这里我用 uint8_t 类型的枚举量来表示前进后退方向。   
	uint8_t vr_dir = Forward;                           //前进为Forward， 后退为Backward，停止为Stop
                                                            //这里的思路是先初始化为前进，然后判断速度正负，
	if(vl < 0)                                          //如果为负，方向设为后退，并取其相反数
	{                                                   //因为我这里速度参数都需要是正数
		vl_dir = Backward;
		vl = -vl;
	}
	if(vr < 0)
	{
		vr_dir = Backward;
		vr = -vr;
	}

	if(vl < 0.1) vl_dir = Stop;                          //如果速度绝对值小于0.1cm/s，则不执行，停止
	if(vr < 0.1) vr_dir = Stop;

	tank.MotorSpeed_Control(Left_Motor, vl_dir, vl);     //设置轮速的函数
	tank.MotorSpeed_Control(Right_Motor, vr_dir, vr);
}
```

只要接收到 cmd_vel 话题，就转换为差速轮轮速，并马上设置电机速度，到达控制的效果。

##### 反馈轮速

同时，为后面的发布 /odom 做准备，我们需要发布反馈速度话题 cmd_vel_feedback 。周期和编码器采集周期相同，我这里设置的是20ms，每当编码器数据更新新，就立即计算为 Twist 消息，然后发布出去。

公式如下：

<img src="http://leiyiming.com/img/in-post/post-tank/轨迹推演逆.gif"/>

代码：

```
  //声明发布者 cmd_vel_feedback, 以及消息 feedback
  geometry_msgs::Twist feedback;
  ros::Publisher cmd_vel_feedback("cmd_vel_feedback", &feedback);                 

  //利用 mbed API声明一个Ticker， 每0.02秒自动调用其依附的函数 getSpeed
  Ticker Ticker_speed;
	Ticker_speed.attach(&getSpeed, 0.02);
	void getSpeed()
	{
		tank.Get_Encoder_Speed();

		double Speedcms_Left = tank.getSpeedcms_Left();                          //由底层接口获得速度值
		double Speedcms_Right = tank.getSpeedcms_Right();

		/***********发布速度反馈***********/
		feedback.linear.x = (Speedcms_Left + Speedcms_Right) / 2.0 / 100.0;      //计算速度值，单位，m/s
		feedback.linear.y = 0;
		feedback.angular.z = (Speedcms_Right - Speedcms_Left) / tank.getBias() / 100.0;
	}
```


### /odom

除了接收速度话题，我们还需要发布 /odom 话题，来告诉上层目前机器人的一个粗略的定位。话题消息为 *nav_msgs/Odometry* ，格式如下：

```
std_msgs/Header header
string child_frame_id
geometry_msgs/PoseWithCovariance pose
geometry_msgs/TwistWithCovariance twist
```

*header* 是ROS常用的信息头，*child_frame_id* 是子坐标系， *pose* 是 header.frame_id 坐标系中，即机器人初始位置（0,0,0）的一个三维坐标，轮式机器人就只有二维信息了。 *twist* 是机器人当前的速度信息，处于 child_frame_id 坐标系中。之所以说这个是粗略的定位，是因为在实际情况中可能会碰到轮子打滑，地面不平整等因素的干扰。（当然上层会通过激光信息来匹配校准）

由于发布频率比较快，消息量比较大，这部分放到底层做会使串口中断过长，导致编码器采集遗漏的问题，所以这部分我们放到上位机来做。上位机程序我们需要接收上面提到的 cmd_vel_feedback 话题，然后进行累加计算。

```
class Odom{
public:
  Odom() :
    x(0), y(0), th(0)
  {
    odom_pub = nh.advertise < nav_msgs::Odometry > ("odom", 10);
    vel_sub = nh.subscribe(std::string("cmd_vel_feedback"), 1, &Odom::feedbackCB, this);
  }

private:
  void feedbackCB(const geometry_msgs::TwistConstPtr& msg)
  {
    double dt = 0.02;
    double delta_x = (msg->linear.x * cos(th) - msg->linear.y * sin(th)) * dt;
    double delta_y = (msg->linear.x * sin(th) + msg->linear.y * cos(th)) * dt;
    double delta_th = msg->angular.z * dt;

    x += delta_x;
    y += delta_y;
    th += delta_th;

    //since all odometry is 6DOF we'll need a quaternion created from yaw
    geometry_msgs::Quaternion odom_quat = tf::createQuaternionMsgFromYaw(th * 1.125);

    //first, we'll publish the transform over tf
    geometry_msgs::TransformStamped odom_trans;
    odom_trans.header.stamp = ros::Time::now();
    odom_trans.header.frame_id = "odom";
    odom_trans.child_frame_id = "base_link";

    odom_trans.transform.translation.x = x;
    odom_trans.transform.translation.y = y;
    odom_trans.transform.translation.z = 0.0;
    odom_trans.transform.rotation = odom_quat;

    //send the transform
    odom_broadcaster.sendTransform(odom_trans);

    //next, we'll publish the odometry message over ROS
    nav_msgs::Odometry odom;
    odom.header.stamp = ros::Time::now();
    odom.header.frame_id = "odom";

    //set the position
    odom.pose.pose.position.x = x * 1.2468;
    odom.pose.pose.position.y = y;
    odom.pose.pose.position.z = 0.0;
    odom.pose.pose.orientation = odom_quat;

    //set the velocity
    odom.child_frame_id = "base_link";
    odom.twist.twist.linear.x = msg->linear.x;
    odom.twist.twist.linear.y = msg->linear.y;
    odom.twist.twist.angular.z = msg->angular.z;

    //publish the message
    odom_pub.publish(odom);
  }

  ros::NodeHandle nh;
  ros::Publisher odom_pub;
  ros::Subscriber vel_sub;
  tf::TransformBroadcaster odom_broadcaster;

  double x;
  double y;
  double th;
};
```

这里我整个功能封装为一个 Odom 类，主要参考 roswiki 的 [发布Odometry教程](http://wiki.ros.org/navigation/Tutorials/RobotSetup/Odom)， vel_sub 是接收 cmd_vel_feedback 的接受者，在其回调函数 feedbackCB 中计算出 odometry 信息，并发布出去。**注意**，这里还需要广播一个 tf 消息，用以实时更新 *base_link* 与 *odom* 坐标系之间的转换关系。

---

### 后记

本篇博客主要介绍了底盘的开发接口以及实现方法，主要代码参考我的github。
