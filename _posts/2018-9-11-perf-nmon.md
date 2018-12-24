---
layout:     post
title:      "性能监控工具nmon & FlameGraph"
subtitle:   " \"performance test\""
date:       2018-9-11
author:     "leiyiming"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Linux
---

## nmon

---

nmon 是一个能够监测CPU、内存、IO等系统资源使用情况的工具，既能够像top那样实时显示，也能支持录制一段时间的数据，并通过excel来绘制图表。

#### 安装(Centos 7)

```shell
$ yum install epel-release && yum install nmon
```

#### 实时显示

运行

```shell
$ nmon
```

会看到下面的显示：

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/45951018.jpg)

下面的按键列表对应着实时显示的系统资源情况，例如，按 `c` 会显示CPU使用情况，按 `m` 会显示内存使用情况，支持多选。例如，选中CPU、Memory、Network，只需要按 `c` 、`m` 、`n`。

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/8445853.jpg)



#### 录制并绘图

数据录制模式下，需要下载专门的VBA分析工具，[下载地址](https://www.ibm.com/developerworks/community/wikis/home?lang=en#!/wiki/Power+Systems/page/nmon_analyser)

解压后有文件 nmon analyser v55.xlsm ，其中包含了解析需要的宏。稍后会用到。

数据收集模式的详细帮助：

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/36478094.jpg)

常用命令：

```shell
$ nmon -s1 -c120 -f -m .
```

* -s1：间隔1秒一次数据快照
* -c120：总共录制120张数据快照
* -f：使用默认的命名方式，即 `<hostname>_YYYYMMDD_HHMM.nmon`
* -m .： 将录制文件保存在当前目录下



再用之前提到的解析工具进行解析，点击 nmon analyser v55.xlsm 中的`Analyze nmon data` ，选中刚刚生产的.nmon文件，即可绘制出图表。

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/87576967.jpg)

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/3370232.jpg)



## perf + FlameGraph

--

#### 1. perf

perf 是linux下用于记录程序运行情况的工具，能够监控堆栈以及各种软硬件事件。下面只记录简单用法

```shell
$ perf record --call-graph dwarf -e cpu-clock -a -g -p 1
```

* -g：额外记录函数的调用关系
* -e：记录事件 cpu-clock，即对应函数占用cpu周期
* -p：指定监控程序的PID

 

程序运行完之后，perf record会生成一个名为perf.data的文件，如果之前已有，那么之前的perf.data文件会被重命名为perf.data.old。

获得这个perf.data文件之后，可以通过 `perf report` 直接进行查看，它会自动加载当前目录下的 perf.data 文件。但是这样显示非常不直观，所以就引入了火焰图。

#### 2、使用火焰图展示结果

1. Flame Graph项目位于GitHub上：<https://github.com/brendangregg/FlameGraph>

2. 可以用git将其clone下来：git clone <https://github.com/brendangregg/FlameGraph.git>

 我们以perf为例，看一下flamegraph的使用方法：

1、第一步

$sudo perf record -e cpu-clock -g -p 1

`Ctrl+c` 结束执行后，在当前目录下会生成采样数据perf.data.

2、第二步

用perf script工具对perf.data进行解析

perf script -i perf.data &> perf.unfold

3、第三步

将perf.unfold中的符号进行折叠：

\#./stackcollapse-perf.pl perf.unfold &> perf.folded

4、最后生成svg图：

./flamegraph.pl perf.folded > perf.svg



生成的火焰图如下：

![](http://pbs8zp0hz.bkt.clouddn.com/18-9-11/71595103.jpg)

从下至上代表函数调用层次，越宽代表占用CPU周期越多。格式是svg，点击某一个方块，可以放大显示其详细信息。



脚本：

```shell
#!/bin/bash

rm -f perf.data
rm -f perf.unfold
rm -f perf.folded
rm -f perf.svg

perf record -e cpu-clock -g -p 1

perf script -i perf.data &> perf.unfold

./FlameGraph/stackcollapse-perf.pl perf.unfold &> perf.folded
./FlameGraph/flamegraph.pl perf.folded > perf.svg
```

