---
layout:     post
title:      Android Studio导入Android系统源码
subtitle:    
date:       2016-12-20
author:     WMY
header-img: img/11.jpg
catalog: 	 true
tags: 
    - 工作工具
---

 
# 使用Android Studio导入源码工程 #



0、因为在导入源码时需要消耗大量内存，所以先修改IDEA_HOME/bin/studio.vmoptions中-Xms和-Xmx的值。

1、为了成功将源码导入AS，我们需要先生成AS可是别的项目工程配置文件
在源码根目录依次执行 

	source build/ensetup.sh
	make idegen && development/tools/idegen/idegen.sh

	这时会在源码的根目录下生成android.ipr，android.iws和android.iml三个文件
	

	注:生成的文件包括：
	①android.iws 包含工作区的个人设置，比如打开过的文件，版本控制工具的配置，本地修改历史，运行和debug的配置等。
	②android.ipr 一般保存了工程相关的设置，比如modules和modules libraries的路径，编译器配置，入口点等。
	③android.iml 用来描述modules。它包括modules路径、 依赖关系，顺序设置等。一个项目可以包含多个 *.iml 文件。
	
	之后我们在AS中打开源码根目录下新生成的android.ipr 
参考文档：
	http://blog.csdn.net/ritterliu/article/details/50857128
