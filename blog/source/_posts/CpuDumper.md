---
title: Android获取CPU使用
date: 2018/5/28
comments: true
categories: performance
tags: performance

---

# 获取CPU情况方法
CPU是性能优化其中一环，对常规的CPU收集手段整理并进行沉淀。
代码：[CpuDumper](https://github.com/NgWaiKong/CpuDumper)
## 一、CPU占用
### 1、proc/stat 
- proc/stat节点记录的是系统进程整体CPU的统计信息
- 文件中的时间单位，sysconf(_SC_CLK_TCK)一般地定义为jiffies(一般地等于10ms)
- 总的cpu时间 = 前七个变量（user, nice, system, idle, iowait, irq, softirq）之和
- 上述时间为一个累计时间，需要在两个时间点分别读一下cpu快照，设为total_time_old 和 total_time_new，则两个值相减即为这段时间内的总CPU时间total_time_delta，然后想办法读一个进程或线程在相同时间段内的cpu时间proc_time_delta, 则该进程或线程的cpu使用率即为（ proc_time_delta / total_time_delta ）* 100% 
- 项目代码：CpuDumper.dumpSystemRate

<!-- more -->

### 2、/proc/[pid]/stat
- 记录的是pid进程整体的CPU统计信息
- 总的cpu时间 = 第14、15个变量（utime：该进程处于用户态的时间、stime：该进程处于内核态的时间)之和
- 项目代码：CpuDumper.dumpAppRate

### 3、/proc/[pid]/task
- 记录的是pid进程中各线程的CPU统计信息
- 各线程总的cpu时间 = 第14、15、16、17个变量（utime + stime + cutime + cstime）
- 项目代码：CpuDumper.dumpAppThreadCpu

### 4、统计
计算某一进程或线程的CPU使用率：
- 在不同时间点，取得上述3个文件的记录的CPU时间， 各自相减，获得deltaSys，deltaApp，deltaThread
- 进程使用率：deltaApp/deltaSys，线程使用率：deltaThread/deltaSys

### 5、附：top 命令
- 获取当前系统各进程或其中线程CPU占用情况
- 常用：
adb shell top -s cpu -m 10 -t
adb shell top -s cpu -m 10 -t |findstr packagename 

- Usage: top [ -m max_procs ] [ -n iterations ] [ -d delay ] [ -s sort_column ] [-t ] [ -h ] 、
    -m num  Maximum number of processes to display. 最多显示多少个进程
    -n num  Updates to show before exiting.  刷新次数 
    -d num  Seconds to wait between updates. 刷新间隔时间（默认5秒）
    -s col  Column to sort by (cpu,vss,rss,thr). 按哪列排序 
    -t      Show threads instead of processes. 显示线程信息而不是进程
    -h      Display this help screen.  显示帮助文档 

- 项目代码：cpuDumper.dumpTop(5000, stream);
## 二、堆栈获取
参考系统ANR时，通过kill -3 pid 和debuggerd pid来获取堆栈信息
### 1、获取Java堆栈（1）
- 接口Thread.getAllStackTraces().entrySet()
- 项目代码：cpuDumper.dumpJavaStack()

### 2、获取Java堆栈（2）
- 命令adb shell su kill -3 pid
- 获取Java线程堆栈
- 需要root
- Dump的堆栈信息保存在/data/anr/traces.txt中
- 项目代码： kill3Cmd.dump()

### 3、获取Native堆栈
- 需要root
- 命令adb shell su debuggerd -b pid
- -b为输出到console，否则输出到/data/tombstones/(该目录只会存放10个文件，多出会进行覆盖)
- 项目代码：cpuDumper.dumpNativeStack()

## 三、Android Profiler：
- AndroidStudio 3.0工具
- 对CPU部分进行双击，可以直观看到应用内线程和系统线程的占用
- 提供对线程dump堆栈方法
- 参考https://developer.android.com/studio/profile/cpu-profiler?hl=zh-cn

## 四、附件
- nexus手机root工具可以使用Nexus Root tools
- debuggerd命令环境配置：root 后，若debuggerd 无法执行。可以尝试替换可用的debuggerd文件。
方法如下：
1、进入/data/system/bin/中，对debuggerd文件进行挂载mount（若嫌麻烦，可以下载re文件管理器http://android.myapp.com/detail.htm?apkName=com.speedsoftware.rootexplorer，然后对文件挂载）
2、拷贝附件中的debuggerd_kelvin文件到/data/system/bin/覆盖原先的debuggerd

## 五、参考
- https://www.google.com/search?q=android+cpu&oq=android+cpu&aqs=chrome..69i57j69i60l2j69i65j69i60l2.1761j0j4&sourceid=chrome&ie=UTF-8
- https://blog.csdn.net/nonmarking/article/details/77924477
- http://gityuan.com/2016/11/27/native-traces/
- http://gityuan.com/2016/06/15/android-debuggerd/
- http://taobaofed.org/blog/2015/12/04/cpu-allocation-profiler/
- http://gityuan.com/2016/12/02/app-not-response/
- http://gityuan.com/2016/11/26/art-trace/