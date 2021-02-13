layout:     post
title:      Android 热缓解框架解析
subtitle:   
date:       2020-06-22
author:     WMY
header-img: img/17.jpg
catalog: true
tags:
    - 工作
---


## Android 热缓解框架解析

#### (1) 引言：

随着手机功能的不断丰富，算法复杂性、系统核心频率和集成水平不断提高，而设备的形制和尺寸不断缩小，手机热缓解的重要性日益凸显。

为了在手机开始过热时进行有效的热缓解，Android 10 在 Android 框架中引入了热系统，以及一个新的 HAL 版本，用于将热子系统硬件设备的接口抽象化,硬件接口包括设备表面、电池、GPU、CPU 和 USB 端口的温度传感器和热敏电阻。借助该框架，设备制造商和应用开发者可以主动获取这些系统硬件设备的温度数据,或者通过注册的回调函数（位于 PowerManager 类中）接收高温通知，进而在设备开始过热时调整系统及应用执行策略。例如，当系统温度较高时，jobscheduler 作业会受到限制，如有必要，可启动框架热关机。从而可以有效缓解手机发热问题。


#### (2) Android热服务上层接口：

Android 10 中的热服务利用来自 Thermal HAL 2.0 的各种缓解信号不断进行监控，并向其客户端提供有关限制严重级别的反馈，其中包括内部组件和 Android 应用。

（1）首先对于上层Android应用来说，可通过 PowerManager 类来添加IThermalStatusListener监听器回调或主动获取获取热状态等信息，方法如下：

	getCurrentThermalStatus()，以整数形式返回设备的当前热状态（除非该设备正处于受限状态）。
	addThermalStatusListener()，添加监听器。
	removeThermalStatusListener()，移除之前添加的监听器。   

以上监听器触发后或主动获取返回的值为当前状态热状态码信息；在ThermalHal 2.0中，热状态分为如下起个级别：

|热状态代码	| 说明和建议用法 |
| -------------------- | ------------------------- |
| NONE (0x0) | 未限制。 此状态代码可用于实施保护操作 | 
| LIGHT (0x1) | 轻微限制。用户体验不受影响。在此阶段，请对设备采取温和的缓解操作。例如，不要提频或采用低效频率（仅适用于大核心）。 |
| MODERATE (0x2) | 中等限制。用户体验没有受到很大影响。 执行的热缓解操作会影响前台 Activity，因此应用应立即减少耗电量。 |
| SEVERE (0x3) | 严重限制。用户体验在很大程度上受到影响。在此阶段，设备热缓解措施应该会限制系统容量。这可能会导致显示卡顿和音频抖动等负面影响。 |
| CRITICAL (0x4) | 平台已采取一切措施减少耗电量。设备热缓解软件已经以最低容量运行所有组件。|
| EMERGENCY (0x5) | 平台中的关键组件因热状况而即将关闭。设备功能受限。这是系统在设备关机前发出的最后一次警告。在此阶段，调制解调器和移动数据网络等一些功能会完全关闭。 |
| SHUTDOWN (0x6) | 立即关机。鉴于此阶段的严重级别，应用可能无法收到此通知。 |
 
应用层调用以上接口方法，假如收到返回值为0x2，说明当前手机热等级为MODERATE (0x2) 中等限制，应用可以根据对应的热等级建议，调整自身逻辑，降低系统负载，缓解手机发热，进而增强在高温状态下的用户体验。


（2）对于Android系统内部组件来说，可以注册IThermalEventListener回调接口等方法进行监控访问更加详细的热传感器和热事件信息；例如，Android系统内部组件可注册IThermalEventListener事件，监听手机表面温度变化:

    if (mThermalService == null) {
        mThermalService = IThermalService.Stub.asInterface(
            ServiceManager.getService(Context.THERMAL_SERVICE));
    }
    if (mEnableSkinTemperatureWarning) {
        ret = mThermalService.registerThermalEventListenerWithType(
                mSkinThermalEventListener, Temperature.TYPE_SKIN);
    } 

    final class SkinThermalEventListener extends IThermalEventListener.Stub {
        @Override public void notifyThrottling(Temperature temp) {
            int status = temp.getStatus();
            if (status >= Temperature.THROTTLING_EMERGENCY) {
                ......
                mWarnings.showHighTemperatureWarning();
                Slog.d(TAG, "SkinThermalEventListener: notifyThrottling was called "
                        + ", current skin status = " + status
                        + ", temperature = " + temp.getValue());
            }
        }
    }

当表面温度超过系统底层设置的阈值档位时就会触发回调事件，进而弹出 Android 系统界面的警告消息提醒用户。

此外，Android系统内部组件还可以调用thermalHal支持的其他功能接口：

	List<CoolingDevice> getCurrentCoolingDevices(); //返回由于触发温控阈值而被限频的硬件设备信息，如BATTERY，CPU，GPU，MODEM，NPU等。
	CpuUsageInfo[] getCpuUsages()； // 获取CPU每个核的负载
	float[] getDeviceTemperatures(@DeviceTemperatureType int type,@TemperatureSource int source) //获取指定节点设备当前温度值


#### (3) Android热服务处理流程：

在 Android 10 中，框架中的热服务利用来自 Thermal HAL 2.0 的各种缓解信号不断进行监控，并向其客户端提供有关限制严重级别的反馈，其中包括内部组件和 Android 应用。下图为 Android 10 中热缓解处理流程的模型。

![](.\therm_mitigation_flow.png)


下面以其中一个方法的调用流程为例，探究整个调用流程的实现机制；上层应用通过注册如下监听回调获取系统温度触发等级信息。

    PowerManager pm = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
    pm.addThermalStatusListener(new PowerManager.OnThermalStatusChangedListener() {
        @Override
        public void onThermalStatusChanged(int status) {
            Log.d(TAG, "onThermalStatusChanged:" + status);
        }
    });

在PowerManager.java 中该方法内部实现如下：

    public void addThermalStatusListener(@NonNull OnThermalStatusChangedListener listener) {
        ...
            this.addThermalStatusListener(mContext.getMainExecutor(), listener);
        ...
    }

    public void addThermalStatusListener(@NonNull @CallbackExecutor Executor executor,
            @NonNull OnThermalStatusChangedListener listener) {
		    ...
                mThermalService = IThermalService.Stub.asInterface(
                        ServiceManager.getService(Context.THERMAL_SERVICE));
            ...
            IThermalStatusListener internalListener = new IThermalStatusListener.Stub() {
                @Override
                public void onStatusChange(int status) {
                    ...
                        executor.execute(() -> {
                            listener.onThermalStatusChanged(status);
                    ...
                }
            };
            ...
                if (mThermalService.registerThermalStatusListener(internalListener)) 
            ...
        }
    }

接下来在 ThermalManagerService 中看下 registerThermalStatusListener 的实现过程：

        @Override
        public boolean registerThermalStatusListener(IThermalStatusListener listener) {
            synchronized (mLock) {
                ...
                    if (!mThermalStatusListeners.register(listener)) {
                        return false;
                    }
                    // 新的监听者刚注册监听会触发一次回调，返回当前温度等级.
                    postStatusListener(listener);
                ...
            }
        }

    private void postStatusListener(IThermalStatusListener listener) {
        ...
                listener.onStatusChange(mStatus);
        ...
    }

从上可以看到上层的注册监听者会被存放在在mThermalStatusListeners中，它是一个RemoteCallbackList类；

    /** Registered observers of the thermal status. */
    @GuardedBy("mLock")
    private final RemoteCallbackList<IThermalStatusListener> mThermalStatusListeners =
            new RemoteCallbackList<>();

如何触发并调用到存储在mThermalStatusListeners这些注册过来的binder监听者呢，继续看 ThermalManagerService 中的以下调用链：

    private void notifyStatusListenersLocked() {
        ...
        final int length = mThermalStatusListeners.beginBroadcast();//获取监听者数目
        try {
            for (int i = 0; i < length; i++) {
                final IThermalStatusListener listener =
                        mThermalStatusListeners.getBroadcastItem(i);
                postStatusListener(listener); //遍历触发监听者
            }
        } finally {
            mThermalStatusListeners.finishBroadcast();
        }
    }

    private void setStatusLocked(int newStatus) {
        ...
            notifyStatusListenersLocked();
        ...
    }

    private void onTemperatureMapChangedLocked() {
        ...
            setStatusLocked(newStatus);
        ...
    }

    private void onTemperatureChanged(Temperature temperature, boolean sendStatus) {
        shutdownIfNeeded(temperature);
        ...
                notifyEventListenersLocked(temperature);
        ...
    }

    /* HwBinder callback **/
    private void onTemperatureChangedCallback(Temperature temperature) {
        ...
            onTemperatureChanged(temperature, true);
        ...
    }

从以上调用关系看，onTemperatureChangedCallback 最终会调用到 mThermalStatusListeners 触发上层应用的注册监听回调；那么谁会又调用到onTemperatureChangedCallback函数呢？在ThermalManagerService 的初始化启动过程中会执行如下代码：

    @Override
    public void onBootPhase(int phase) {
        ...
            onActivityManagerReady();
        ...
    }

	private void onActivityManagerReady() {
        synchronized (mLock) {
            // Connect to HAL and post to listeners.
            boolean halConnected = (mHalWrapper != null);
            if (!halConnected) {
                mHalWrapper = new ThermalHal20Wrapper();
                halConnected = mHalWrapper.connectToHal();
            }
            ...
            mHalWrapper.setCallback(this::onTemperatureChangedCallback);
		...

以上可看出，ThermalHal20Wrapper 在thermalService服务初始化完成后会设置 onTemperatureChangedCallback 回调函数，ThermalHal20Wrapper类实现如下：

    abstract static class ThermalHalWrapper {
        @FunctionalInterface
        interface TemperatureChangedCallback {
            void onValues(Temperature temperature);
        }

        /** Temperature callback. */
        protected TemperatureChangedCallback mCallback;

        @VisibleForTesting
        protected void setCallback(TemperatureChangedCallback cb) {
            mCallback = cb;
        }
	...

    static class ThermalHal20Wrapper extends ThermalHalWrapper {
        /** Proxy object for the Thermal HAL 2.0 service. */
        @GuardedBy("mHalLock")
        private android.hardware.thermal.V2_0.IThermal mThermalHal20 = null;

        /** HWbinder callback for Thermal HAL 2.0. */
        private final IThermalChangedCallback.Stub mThermalCallback20 =
                new IThermalChangedCallback.Stub() {
                    @Override
                    public void notifyThrottling(
                            android.hardware.thermal.V2_0.Temperature temperature) {
                        Temperature thermalSvcTemp = new Temperature(
                                temperature.value, temperature.type, temperature.name,
                                temperature.throttlingStatus);
                        ...
                            mCallback.onValues(thermalSvcTemp);
                        ...
                    }
                };

        @Override
        protected boolean connectToHal() {
            ...
                mThermalHal20 = android.hardware.thermal.V2_0.IThermal.getService(true);
                mThermalHal20.linkToDeath(new DeathRecipient(), THERMAL_HAL_DEATH_COOKIE);
                mThermalHal20.registerThermalChangedCallback(mThermalCallback20, false,0 /* not used */);
            ...
        }

以上可看出 ThermalHal20Wrapper 会获取Thermal HAL 2.0 service，并调用 registerThermalChangedCallback，当温度触发时会回调到 mThermalCallback20的实现，进而调用到以上onTemperatureChangedCallback方法，最终触发调用到上层监听者。

#### (4) Thermal HAL 2.0 实现机制

从 Android 10 开始引入了 Thermal HAL 2.0，设备制造商必须实现 Thermal HAL 2.0 的 HIDL 中的方法（如 IThermal.hal 中所提供），获取设备温度传感器和限制状态，温度状态变化超过设定阈值时，将这些信息通过IThermalChangedCallback传递给thermalservice，触发以上调用流程，进而上层应用监听者能够获取这些状态信息，从而做出相应决策。 

下面以Google pixel项目为例解析thermal HAL 2.0的实现机制：
以下是thermal HAL 2.0的 IThermal.hal 部分声明：
 
	import IThermalChangedCallback;
	interface IThermal extends @1.0::IThermal {

   /**
    * @param callback the IThermalChangedCallback to use for receiving
    *    thermal events (nullptr callback will lead to failure with status code FAILURE).
    * @param filterType if filter for given sensor type.
    * @param type the type to be filtered.
    *
    * @return status Status of the operation. If status code is FAILURE,
    *    the status.debugMessage must be populated with a human-readable error message.
    */
   registerThermalChangedCallback(IThermalChangedCallback callback,
                                  bool filterType,
                                  TemperatureType type)
       generates (ThermalStatus status);

	
	/**
	 * IThermalChangedCallback send throttling notification to clients.
	 */
	interface IThermalChangedCallback {
	    /**
	     * Send a thermal throttling event to all ThermalHAL
	     * thermal event listeners.
	     *
	     * @param temperature The temperature associated with the
	     *    throttling event.
	     */
	    oneway notifyThrottling (Temperature temperature);
	};


Google pixel 中 thermal HAL 2.0 的回调接口实现如下，它会将注册监听者保存在 callbacks_ 中：

	Return<void> Thermal::registerThermalChangedCallback(const sp<IThermalChangedCallback> &callback,
	                                                     bool filterType, TemperatureType_2_0 type,
	                                                     registerThermalChangedCallback_cb _hidl_cb) {
	    ...
	        callbacks_.emplace_back(callback, filterType, type);
	        LOG(INFO) << "a callback has been registered to ThermalHAL, isFilter: " << filterType
	                  << " Type: " << android::hardware::thermal::V2_0::toString(type);
	    ...
	}
	
callbacks_的实现如下：

	struct CallbackSetting {
	    CallbackSetting(sp<IThermalChangedCallback> callback, bool is_filter_type,
	                    TemperatureType_2_0 type)
	        : callback(callback), is_filter_type(is_filter_type), type(type) {}
	    sp<IThermalChangedCallback> callback;
	    bool is_filter_type;
	    TemperatureType_2_0 type;
	};
 	std::vector<CallbackSetting> callbacks_;



在pixel thermal HAL 2.0 服务刚初始化时，会同时开启一个线程持续地监听底层硬件设备的温度节点变化：      del：当温度超过配置文件的设定阈值时，就会调用如下函数触发回调：
	
	Thermal::Thermal()
	    : thermal_helper_(
	          std::bind(&Thermal::sendThermalChangedCallback, this, std::placeholders::_1)) {}

如下是ThermalHelper部分实现：

	ThermalHelper::ThermalHelper(const NotificationCallback &cb)
	    : thermal_watcher_(new ThermalWatcher(
	              std::bind(&ThermalHelper::thermalWatcherCallbackFunc, this, std::placeholders::_1))),
	      cb_(cb),.. { //将回调函数保存在 cb_ 变量中

		...
		//解析温控节点阈值信息的配置文件，pixel的配置文件在：/vendor/etc/thermal_info_config.json 中
	    ...

	    //根据配置文件中配置信息，监控对应设备节点的温度变化
	    thermal_watcher_->registerFilesToWatch(monitored_sensors, initializeTrip(tz_map)); 
	
	    // Need start watching after status map initialized，
	    is_initialized_ = thermal_watcher_->startWatchingDeviceFiles();//启动线程开始监控设备节点温度变化
	}

在 ThermalWatcher 中，它继承了Looper类，循环监听各个系统设备节点的温度变化

	// A helper class for monitoring thermal files changes.
	class ThermalWatcher : public ::android::Thread {
	  public:
	    ThermalWatcher(const WatcherCallback &cb)
	        : Thread(false), cb_(cb), looper_(new Looper(true)) {}

	...

	bool ThermalWatcher::threadLoop() { //循环执行
	    // Polling interval 2s， 若是polling的方式每个2s读取一次设备节点温度
	    static constexpr int kMinPollIntervalMs = 2000;
	    // Max uevent timeout 5mins 若是uevent的方式监听，最大延迟5min读取一次设备节点温度
	    static constexpr int kUeventPollTimeoutMs = 300000;
	    int fd;
	    std::set<std::string> sensors;
	    auto time_elapsed_ms = std::chrono::duration_cast<std::chrono::milliseconds>(boot_clock::now() -
	                                                                                 last_update_time_)
	                                   .count();
	    int timeout = (thermal_triggered_ || is_polling_) ? kMinPollIntervalMs : kUeventPollTimeoutMs;
	    if (time_elapsed_ms < timeout && looper_->pollOnce(timeout, &fd, nullptr, nullptr) >= 0) {
	        ...
	        parseUevent(&sensors);
	        ...
	    }
	    thermal_triggered_ = cb_(sensors); //这里如果设备节点温度变化会触发回调
	    ...
	}

若是ThermalWatcher监控到设备节点温度发生变化后，会回调到如下方法：

	bool ThermalHelper::thermalWatcherCallbackFunc(const std::set<std::string> &uevent_sensors) {
	    std::vector<Temperature_2_0> temps;
	    bool thermal_triggered = false;	
	    ...
		//中间逻辑大致是 检测当前温度变化是否符合配置文件对应设备温度节点的阈值触发标准等。
	    ...
	    if (!temps.empty() && cb_) {
	        cb_(temps); //满足标准后，调用回调函数触发回调监听
	    }	
	    ...
	}

按照如上回调函数流程，会调用到 Thermal.cpp 中的sendThermalChangedCallback函数，进而触发上层回调监听：

	void Thermal::sendThermalChangedCallback(const std::vector<Temperature_2_0> &temps) {
	    std::lock_guard<std::mutex> _lock(thermal_callback_mutex_);
	    for (auto &t : temps) {
	        LOG(INFO) << "Sending notification: "
	                  << " Type: " << android::hardware::thermal::V2_0::toString(t.type)
	                  << " Name: " << t.name << " CurrentValue: " << t.value << " ThrottlingStatus: "
	                  << android::hardware::thermal::V2_0::toString(t.throttlingStatus);
	        callbacks_.erase( //注意这里的 callbacks_ 中即为 registerThermalChangedCallback 函数中保存的上层注册监听者
	            std::remove_if(callbacks_.begin(), callbacks_.end(),
	                           [&](const CallbackSetting &c) {
	                               if (!c.is_filter_type || t.type == c.type) {
	                                   Return<void> ret = c.callback->notifyThrottling(t);
	                                   return !ret.isOk();
	                               }
	                               ...
	                           }),
	            callbacks_.end());
	    }
	}

经过层层回调，通过hidl 接口回调到 thermalservice 中的 ThermalHal20Wrapper，进而将消息传递给上层的应用监听者，实现android系统设备节点温度监听。


如上中pixel的配置文件存放在：/vendor/etc/thermal_info_config.json 中， 部分配置截取说明如下：


    "Sensors":[
        {
            "Name":"back_therm", #设备节点名称，
            "Type":"SKIN", #设备节点类型
            "HotThreshold":[ #表示温度触发等级，超过该等级值s时触发一次温度上报
                "NAN",
                40.0,
                47.0,
                50.0,
                52.0,
                54.0,
                56.0
            ],
            "VrThreshold":52.0, #
            "Multiplier":1.0, #读取到的节点温度 乘以 该值，表示实际温度值
            "Monitor":true  #是否监听该节点温度变化
        }
    ],
    "CoolingDevices":[
        {
            "Name":"thermal-cpufreq-0", #需要获取的由于温升限频的设备节点名称
            "Type":"CPU"
        }
    ]



	// On init we will spawn a thread which will continually watch for
	// throttling.  When throttling is seen, if we have a callback registered
	// the thread will call notifyThrottling() else it will log the dropped
	// throttling event and do nothing.  The thread is only killed when
	// Thermal() is killed.


限制设备性能的任何因素（包括电池电量限制）都必须通过 Thermal HAL 进行报告。为确保做到这一点，请将可能会指示需要进行缓解操作（根据状态变化）的所有传感器放入 Thermal HAL，并报告所采取的任何缓解操作所对应的严重级别。从传感器读数返回的温度值不一定是实际温度，只要它准确反映相应的严重级别阈值即可。例如，您可以传递不同的数值而非实际温度阈值，也可以在阈值规范中建立保护带以提供迟滞功能。不过，与该值对应的严重级别必须与在相应阈值所需达到的级别一致。（例如，当现实中实际温度为 65°C，且该温度的严重级别对应于您指定的“严重”时，您可以考虑返回 72°C 作为临界温度阈值。）严重级别必须始终准确无误，以便充分发挥热框架的功能。

如需详细了解框架中的阈值级别及其如何与各缓解操作一一对应，请参阅每个热状态代码的用法建议。



### 总结




