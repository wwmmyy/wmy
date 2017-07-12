---
layout:     post
title:      Battery historian tool 使用说明
subtitle:    
date:       2016-12-14
author:     WMY
header-img: img/11.jpg
catalog: 	 true
tags: 
    - 工作工具
---


# Battery historian tool 使用说明 #
   
  

## 1、Battery Historian 是什么？能干什么？ ##
一款由Google提供的Android系统电量分析工具.能够以网页形式展示手机的电量消耗过程：

	查看手机耗电量走势
	哪些 APP 在前台运行
	哪些应用持唤醒锁，以及手机相应的清醒时间
	进入doze时刻及时长
	不同时刻网络连接情况、屏幕状态、CPU运行状态等等。

  可以通过此了解唤醒频率、有谁发起，持续多长时间，从而协助找出手机耗电量快的原因
 
## 2、打开及相关命令介绍 ##

### 2.1、打开git bash工具 运行如下命令 ###

 	1、 cd $GOPATH/src/github.com/google/battery-historian 进入其目录下方 

 	2、 运行Battery Historian 
	  // a. go run setup.go 
	   b. go run cmd/battery-historian/battery-historian.go	 
 
 	3、检查/battery-historian是否运行，登录网址 http://localhost:9999查看
 
 
### 2.2、导出手机的Bugreport日志 ###
 
	1.输入指令 adb bugreport > bugreport.txt导出。
	2. pwd 显示导出位置
 
### 2.3、上传bugreport.txt文件至? [http://localhost:9999](http://localhost:9999) ###

	退出deepidle的原因,关键字： becomeActiveLocked, reason

### 2.4、其他说明 ###

	重置命令：adb shell dumpsys batterystats --reset

   1.Wakelock 分析，默认情况下Android不记录每个具体应用的持锁时间戳，因此需要开启

	开启命令：adb shell dumpsys batterystats --enable full-wake-history

   2.Kernel trace analysis

   实现记录kernel wakeup source and kernel wakelock activities:

	$ adb root
	$ adb shell
	
	# Set the events to trace.
	$ echo "power:wakeup_source_activate" >> /d/tracing/set_event
	$ echo "power:wakeup_source_deactivate" >> /d/tracing/set_event
	
	# The default trace size for most devices is 1MB, which is relatively low and might cause the logs to overflow.
	# 8MB to 10MB should be a decent size for 5-6 hours of logging.
	
	$ echo 8192 > /d/tracing/buffer_size_kb
	
	$ echo 1 > /d/tracing/tracing_on

	Then, use the device for intended test case.
	Finally, extract the logs:
	$ echo 0 > /d/tracing/tracing_on
	$ adb pull /d/tracing/trace <some path>
	
	# Take a bug report at this time.
	$ adb bugreport > bugreport.txt

	# 重置电量信息
	adb shell dumpsys batterystats --reset
	
	# 让系统记录所有的的WakeLock信息
	adb shell dumpsys batterystats --enable full-wake-history
	
	# 测试完成后，导出bugreport
	adb bugreport > name_of_bugreport.txt
	
	# 设置手机进入不充电的状态 <- 这样我们才能连着电脑收集数据
	adb shell dumpsys battery unplug
	
	# adb shell dumpsys battery -h
	# Dump current battery state, or:
	#   set [ac|usb|wireless|status|level|invalid] <value>
	#   unplug
	#   reset


## 3、界面模块介绍 ##

### 3.1导入单个bugreport.txt文件分析 ###

成功[导入](http://23.251.148.173/)bugreport.txt文件后界面显示:

 ![](http://i.imgur.com/iqSdlFS.jpg) 

**kernel唤醒源、时间及次数：**  

 ![](http://i.imgur.com/ALrp9Id.jpg)
 
**耗电量排行:**  

 ![](http://i.imgur.com/msqVuL2.jpg)

**每个app使用的移动流量:**  

 ![](http://i.imgur.com/gFGQIvd.jpg)

**每个APP占用移动网络时间及次数:**  

 ![](http://i.imgur.com/tsWR2Oa.jpg) 

**APP持锁时间及次数:**  

  ![](http://i.imgur.com/e5DA3uq.jpg)


### 3.2导入两个bugreport.txt文件对比分析 ###

同时导入两个bugreport.txt文件做对比分析：
 
**（1）以图表的方式对比展示：**

 ![](http://i.imgur.com/ypXybN8.jpg)

**（2）从系统状态维度对比展示：**

 ![](http://i.imgur.com/dhslSus.jpg)

**（3）从历史状态维度对比展示：**

  ![](http://i.imgur.com/2AnMVtp.jpg)

### 3.3导入bugreport.txt + Kernel Wakesource Trace文件对比分析 ###


## 4、案例分析 ##

### 4.1 案例1 ###

用户手机晚上待机后，发现手机依然发热，电池耗电量很快，需要找出导致耗电量快的原因。
 第一步导出手机bugreport.txt并上传，观察耗电量统计图：

![](http://i.imgur.com/Ucxg7AE.jpg)

从图中可以清晰的看到，当6PM左右，屏幕关闭时耗电量非常快，说明有应用在后天持锁，导致系统一直在后台运行，将光标移动到UserSpace WakeLock行，会显示如下信息：

 ![](http://i.imgur.com/xnPe9WX.jpg)

从图中可以看到 "net_scheduler" 很可能是手机保持唤醒的原因，但是它表示什么意思呢？打开"System stats"面板下的"Userspace Wakelocks"模块：
 
![](http://i.imgur.com/XchOO1I.png)

从中可以看到对应的YouTube music app在持锁，可能就是CPU保持唤醒的原因。可以通过关闭或卸载该应用，再次待机测试进行验证确认。

### 4.2 案例2 ###

手机使用及休眠过程中电池消耗量过快,将bugreport.txt并上传分析：

![](http://i.imgur.com/k25F2Fl.jpg)
![](http://i.imgur.com/k4MfZEW.jpg)
![](http://i.imgur.com/nxk7cr4.jpg)

通过观察上图可以发现，在灭屏待机后手机耗电量依然很快，说明灭屏状态下没有进去睡眠。
针对这种情况首先我们要确认audio等已关闭，观察灭屏后的UserSpace WakeLock等信息，对可疑的异常持锁的app关闭或卸载后测试验证。若仍然异常，将手机的关闭wifi、蓝牙、GPS、LTE等关闭，排除这方面的异常，继续待机观察：

 ![](http://i.imgur.com/OBq63Zh.jpg)

![](http://i.imgur.com/o8RYsuA.jpg)
     
通过电量曲线发现，待机7小时电量消耗30%，明显不正常，此时已经排除了wifi、蓝牙、LTE关闭等影响，也可能是硬件导致，此时单凭Battery-Historian tool无法找到原因，需要结合MTKlog，systrace等工具继续分析。






