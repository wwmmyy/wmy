---
layout:     post
title:      Android Add System Service
subtitle:    
date:       2017-04-25
author:     WMY
header-img: img/02.jpg
catalog: 	 true
tags: 
    - 工作
---



## 添加系统级Service ##


本文通过一个demo介绍如何添加一个系统级别的Service，同时实现相应的接口供APP层调用；

### 1.1 创建aidl文件：
 
APP层调用系统内Service属于跨进程访问，需要通过Binder进行跨进程通信，我们首先要实现一个aidl文件：
在源码frameworks/base/core/java/android/os/ 下面新增 一个 IHelloService.aidl

	package android.os;
	/**
	 * 
	 * @author wmy
	 * 如果是非默认类型的参数，需要引入对应包
	 */
	public interface IHelloService {
		
		void setInfo(String key,String value);
		String getInfo(String info);
		void saveLog(String log);
		String dumpLog(); 
	
	}


### 1.2 将aidl添加到mk文件中

在frameworks\base\Android.mk中将IHelloService.aidl文件添加进去，如下所示：

	core/java/android/os/IPowerManager.aidl \
	core/java/android/os/IHelloService.aidl \
	core/java/android/os/IRecoverySystem.aidl \


### 1.3 创建Service文件

在 frameworks/base/services/core/java/com/android/server/ 下面新增一个 HelloService.java 用来实现aidl文件定义的接口。

	package android.os;	 
	import java.io.InputStreamReader;
	import java.io.LineNumberReader;
	import java.lang.*;
	import java.util.HashMap;
	import android.os.RemoteException;
	import android.os.IHelloService;	

	public class HelloService extends IHelloService.Stub{
		private static HashMap<String,String> map=new HashMap<>();
		private static String inner_log=""; 
		    
		@Override
		public void setInfo(String key, String value) {
	 		  map.put(key,value);			
		}
		@Override
		public String getInfo(String info) {
	 		return map.get(key);
		}
		@Override
		public void saveLog(String log) {
	 		inner_log+=log+"\n";
		}
		@Override
		public String dumpLog() {
	 		return inner_log;
		}  
	} 


### 1.4 将HelloService添加到SystemServer 启动进程

将HelloService添加到SystemServer 启动进程后，开机后系统会启动HelloService服务；先在 frameworks/base/core/java/android/content/Context.java 中添加一行 

	public static final String HELLO_SERVICE="helloService";

此外修改 frameworks/base/services/java/com/android/server/SystemServer.java
在  startOtherServices() 函数 的try模块中增加以下代码：

	 try {
            Slog.i(TAG, "HelloService");
            ServiceManager.addService(Context.HELLO_SERVICE, new HelloService());
        } catch (Throwable e) {
            Slog.e(TAG, "Failure starting HelloService", e);

        }

### 1.5 创建HelloManager

在frameworks/base/core/java/android/app/ 下创建HelloManager.java 文件 内容如下： 

	package android.app; 
	import android.annotation.SdkConstant;
	import android.annotation.SystemApi;
	import android.content.Context;
	import android.content.Intent;
	import android.os.Build;
	import android.os.Parcel;
	import android.os.Parcelable;
	import android.os.RemoteException;
	import android.os.IHelloService;
	import android.util.Log;
	
	public class HelloManager {
	    IHelloService mService;
	    public HelloManager(Context ctx,IHelloService service){
	        mService=service;
	    }
	    public void setInfo(String key,String value){
	        try{
	            mService.setInfo(key,value);
	        }catch(Exception e){
	            Log.e("HelloManager",e.toString());
	            e.printStackTrace();
	        }
	
	    }
	    public String getInfo(String key){
	        try{
	            return mService.getInfo(key);
	        }catch(Exception e){
	            Log.e("HelloManager",e.toString());
	            e.printStackTrace();
	        }
	        return null;
	    }
	    public void saveLog(String log){
	        try{
	            mService.saveLog(log);
	        }catch(Exception e){
	            Log.e("HelloManager",e.toString());
	            e.printStackTrace();
	        }
	    } 
	    public String dumpLog(){
	        try{
	            return mService.dumpLog();
	        }catch(Exception e){
	            Log.e("HelloManager",e.toString());
	            e.printStackTrace();
	        }
	        return null;
	    }
	} 

### 1.6 注册到SystemService

修改frameworks/base/core/java/android/app/SystemServiceRegistry.java

在静态代码块中增加
	
	registerService(Context.HELLO_SERVICE, HelloManager.class,
		new CachedServiceFetcher<HelloManager>() {
		    @Override
		    public HelloManager createService(ContextImpl ctx) {
		        IBinder b = ServiceManager.getService(Context.HELLO_SERVICE);
		        IHelloService service = IHelloService.Stub.asInterface(b);
		        return new HelloManager(ctx, service);
		    }});
 
 

### 1.7 修改SePolicy的编译验证

由于SELinux Policy限制，新加一个系统服务需要在如下文件中进行注册。

修改 device\mediatek\common\sepolicy\basic\service.te 在最后一行添加

	type ccc_service, system_api_service, system_server_service, service_manager_type;

然后修改同目录下 \device\mediatek\common\sepolicy\basic\service_contexts 文件，
中间插入一行 

	ccc u:object_r:ccc_service:s0 

### 1.8 重新编译源码

别忘了先添加系统API后需要执行： make update-api。


### 1.9 在Activity中调用新增API 测试
 
	.......
	import android.os.IHelloService;
	
	public class MainActivity extends AppCompatActivity {
	
		HelloManager hm;
	    @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	
	        setContentView(R.layout.activity_main);  
	
	        try {
	
				hm=(HelloManager)getSystemService(Context.HELLO_SERVICE);
				hm.setInfo("hiSystemServie","this is my first new SystemService");
	            Toast.makeText(MainActivity.this,hm.getInfo("hiSystemServie"),Toast.LENGTH_LONG).show();
	        } catch (Exception e) {
				Toast.makeText(MainActivity.this,"getSystemService failed",Toast.LENGTH_LONG).show();
	            e.printStackTrace();
	        }  
	    } 
	 }

参考：
	http://www.cnblogs.com/liam999/p/5933827.html
