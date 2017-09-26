---
layout:     post
title:      Android源码中JNI用法总结
subtitle:   JNI两种注册过程
date:       2017-09-21
author:     WMY
header-img: img/02.jpg
catalog: 	 true
tags: 
    - 工作
---


# JNI学习总结


##0、环境配置

a.local.properties 中添加NDK路径，如：ndk.dir=D\:\\ProgramFiles\\NDK\\android-ndk-r14b

b.在工程app文件目录下的build.gradle中添加如下代码：

	  defaultConfig {
	         。。。。。。
	        ndk{
	            moduleName "JniUtil" //生成的so包名
	            ldLibs "log" //添加依赖库，因为有log等打印
	          //abiFilters "armeabi", "armeabi-v7a" //输出指定abi体系结构下的so库，目前可有可无。
	        }
	           sourceSets.main{
	            jni.srcDirs = []
	            jniLibs.srcDir "src/main/libs"
	        }
	     。。。。
	 ｝

c. 在gradle.properties文件中添加：android.useDeprecatedNdk=true 


##1、JNI传统静态注册方式

传统的开发方法是先在java层声明好native方法，然后通过命令生成.h头文件，根据生成的头文件中的方法名再用本地的C来实现对应的方法。
传统方式java中的native方法和C/C++中方法对应是这样的：Java_完整包名类名_方法名()， 例如：

	C:\Users\Administrator\AndroidStudioProjects\test0\app\src\main\java>
	javah -jni com.example.administrator.test0.JniUtil

	结果会生成如下的头文件： com_example_administrator_test0_JniUtil.h

根据生成的头文件中的方法名再用本地的C来实现对应的方法即可。

用C/C++实现完方法后，将(c/c++)文件编译成动态库，点击： Android Studio -->Build -->Make Project 这样会在路径： app --> build-->intermediates -->ndk-->debug-->lib 下面生成各so库文件。

**静态注册弊端：**

a. 后期类名、文件名改动，头文件所有函数将失效，需要手动改，超级麻烦易出错

b. 代码编写不方便，由于JNI层函数的名字必须遵循特定的格式，且名字特别长；

c. 会导致程序员的工作量很大，因为必须为所有声明了native函数的java类编写JNI头文件；

d. 程序运行效率低，因为初次调用native函数时需要根据根据函数名在JNI层中搜索对应的本地函数，然后建立对应关系，这个过程比较耗时。

静态注册JNI弊端多多，因此使用动态注册JNI十分有必要。


##2、JNI_Onload方式 


###2.1、动态注册的原理
 
相对于传统上层APP开发JNI调用方式，android系统源码则采用另一种方式来定义其native函数，此方式无需生成.h头文件，它使用了java和c函数的映射数组，并在其中描述了函数的参数及返回值, 该映射表会注册给 JVM，这样 JVM 就可以用函数映射表来调用相应的函数，而不必通过函数名来查找相关函数(这个查找效率很低，函数名超级长)。实现过程：

a. 利用结构体JNINativeMethod保存Java Native函数和JNI函数的对应关系；

b. 在一个JNINativeMethod数组中保存所有native函数和JNI函数的对应关系；

c. 在Java中通过System.loadLibrary加载完JNI动态库之后，调用JNI_OnLoad函数，开始动态注册；

d. JNI_OnLoad中会调用AndroidRuntime::registerNativeMethods函数进行函数注册；

e. AndroidRuntime::registerNativeMethods中最终调用jniRegisterNativeMethods完成注册。


####2.1.1 结构体JNINativeMethod

 JNINativeMethod的作用是通过这个结构体从而使Java与jni建立联系，该结构体的定义如下：

	typedef struct {
	const char* name; //Java中函数的名字
	const char* signature;//符号签名，描述了函数的参数和返回值
	void* fnPtr;//函数指针，指向一个被调用的c函数
	} JNINativeMethod;


其中比较难理解的是第二个参数，例如：
“()V” “(II)V”  “(Ljava/lang/String;Ljava/lang/String;)V”
实际上这些字符是与函数的参数一一对应的“（）”中的字符表示参数，后面的则代表返回值，eg：
用“()V”表示void Func(); “(II)V”表示 void Func(int,int);

事实上第二个参数在上述执行：javah -jni com.example.administrator.test0.JniUtil 时生成的.h头文件中已经包含，如：
	
	/*
	 * Class:     com_example_administrator_test0_JniUtil
	 * Method:    getJniAdd
	 * Signature: (II)I //注意此值即为第二个参数值
	 */
	JNIEXPORT jint JNICALL Java_com_example_administrator_test0_JniUtil_getJniAdd
	  (JNIEnv *, jclass, jint, jint);



我们看上面定义的结构体数组：

	JNINativeMethod nativeMethod[] = \{\{"getJniAdd", "(II)I", (void *) getJniAddNative\}\};
 
可以看出，里面有一个成员，该成员第一个参数 "getJniAdd",java 函数名；第二个参数“(II)I;",是签名符号，对应java中的native方法：int getJniAdd(int a, int b)的参数及返回值。第三个参数就是要调用的 native 方法。

####2.1.2 JNI_OnLoad()函数

在开篇我们就提过当 java 通过 System.loadLibrary 加载完 JNI 动态库后，紧接着会调用 JNI_OnLoad 的函数。这个函数主要有两个作用:

a.指定Jni版本

告诉JVM该组件使用哪一个jni版本（若未提供JNI_Onload函数，JVM会默认使用最老的JNI1.1版本），如果要使用新的版本的JNI，如JNI 1.4版本，则必须由JNI_OnLoad()函数返回常量JNI_VERSION_1_4(该常量定义在jni.h中)来告知JVM初始化.
当JVM执行到System.loadLibrary() 函数时，会立即调用 JNI_OnLoad() 方法，因此在该方法中进行各种资源的初始化操作最为恰当，

	JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *vm, void *reserved)
	{
	    LOGD("-------------JNI_OnLoad into.--------\n");
	    JNIEnv *env = NULL;
	    jint result = -1;
	    if ((*vm)->GetEnv(vm, (void **) &env, JNI_VERSION_1_4) != JNI_OK) {
	        return -1;
	    }
	    assert(env != NULL);
	    //动态注册，自定义函数
	    if (!registerNatives(env)) {
	        return -1;
	    }
	
	    return JNI_VERSION_1_4;
	}

b.动态注册

RegisterNatives,动态注册就是在这里完成的，函数原型在jni.h中:

	 RegisterNatives(jclass clazz, const JNINativeMethod* methods,jint nMethods)

该函数有3个参数含义如下：

clazz :java类名，通过FindClass得到; 

methods: JNINativeMethod的结构体指针

mMethods: 方法个数

在该例子中代码如下:
 
	static jint getJniAddNative(JNIEnv *env, jclass cls, jint a, jint b) {
	    return a + b;
	}	
 
	//方法数组，正是这个，可以动态调用任意 native 方法
	JNINativeMethod nativeMethod[] = \{\{"detJniAdd", "(II)I", (void *) getJniAddNative\} \};	
	
	static int registerNativeMethods(JNIEnv *env, const char *className, JNINativeMethod *gMethods,
	                                 int numMethods) {
	    jclass clazz;
	    clazz = (*env)->FindClass(env, className);
	    if (clazz == NULL) {
	        return JNI_FALSE;
	    }	
	    if ((*env)->RegisterNatives(env, clazz, gMethods, numMethods) < 0) {
	        return JNI_FALSE;
	    }	
	    return JNI_TRUE;
	}
	
	//注册Native
	static int registerNatives(JNIEnv *env) {
	     const char *className = "com/example/administrator_test0/JniUtil"; //指定注册的类
	    return registerNativeMethods(env, classPathName, nativeMethod,
	                                 sizeof(nativeMethod) / sizeof(nativeMethod[0]));
	}
 

其他： 

对于JNIEnv *env来说，在C中调用：(*env)->NewStringUTF(env, "Hello from JNI!");

而在C++中如果按照上述调用则会发生'base operand of '->' has non-pointer type '_JNIEnv''错误，需要如下调用：env->NewStringUTF("Hello from JNI!");

原因：参见jni.h中对于JNIEnv的定义：

	#if defined(__cplusplus)
	typedef _JNIEnv JNIEnv;
	#else
	typedef const struct JNINativeInterface* JNIEnv;
	#endif






参考文档：
http://blog.csdn.net/xsf50717/article/details/54693802


