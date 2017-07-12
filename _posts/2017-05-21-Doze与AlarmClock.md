---
layout:     post
title:      Doze 与 AlarmClock 的关系
subtitle:    
date:       2017-5-21
author:     WMY
header-img: img/01.jpg
catalog: 	 true
tags: 
    - alarm
---


## Doze 与 AlarmClock 的关系 ##


### 1、 Doze模式下AlarmClock类型的alarm会正常触发，不受影响

通常情况下，进入Doze的idle阶段，普通非白名单应用设置的alarm会被放在pendingList数组中挂起，只有退出idle阶段才会继续起作用，但是对于API21+(Android5.0以上)，新API允许使用setAlarmClock()方法来设置一个可见的定时器：系统UI通过getNextAlarmClock()更新时间和图标。注意setAlarmClock()可以在设备/应用休眠idle时仍然生效（类似于setExactAndAllowWhileIdle()）:更形象的描述是类似来电唤醒。对于上层用户来说设置AlarmClock方法如下：

	AlarmManager am = (AlarmManager) context.getSystemService(Context.ALARM_SERVICE);
	Intent intent = new Intent(ALARM_ALERT_ACTION);
	PendingIntent sender = PendingIntent.getBroadcast(context, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT);
	am.setAlarmClock(new AlarmManager.AlarmClockInfo(time, sender), sender);

接下来我们看下setAlarmClock的代码实现原理:

	代码路径：base\core\java\android\app\AlarmManager.java:

    public void setAlarmClock(AlarmClockInfo info, PendingIntent operation) {
        setImpl(RTC_WAKEUP, info.getTriggerTime(), WINDOW_EXACT, 0, 0, operation,
                null, null, null, null, info);
    }

    private final IAlarmManager mService;

    private void setImpl(int type, long triggerAtMillis, long windowMillis, long intervalMillis,
            int flags, PendingIntent operation, final OnAlarmListener listener, String listenerTag,
            Handler targetHandler, WorkSource workSource, AlarmClockInfo alarmClock) {
 		.............
       
        try {
            mService.set(mPackageName, type, triggerAtMillis, windowMillis, intervalMillis, flags,
                    operation, recipientWrapper, listenerTag, workSource, alarmClock);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

通过binder机制，最后会调用对应的AlarmManagerService 的set方法:

	代码路径：base\services\core\java\com\android\server\AlarmManagerService.java:
 
    private final IBinder mService = new IAlarmManager.Stub() {
        @Override
        public void set(String callingPackage,
                int type, long triggerAtTime, long windowLength, long interval, int flags,
                PendingIntent operation, IAlarmListener directReceiver, String listenerTag,
                WorkSource workSource, AlarmManager.AlarmClockInfo alarmClock) {
		............. 
          // If this alarm is for an alarm clock, then it must be standalone and we will
            // use it to wake early from idle if needed.
            if (alarmClock != null) {
            //注意：如果设置的是alarmClock类型的alarm，会对其flags添加如下标记，这个在后面的判断很关键
                flags |= AlarmManager.FLAG_WAKE_FROM_IDLE | AlarmManager.FLAG_STANDALONE;
 
		.............
        }

 
 
紧接着会走如下代码，判断将此次设置的alarmClock保存到mNextWakeFromIdle中：

    private void setImplLocked(Alarm a, boolean rebatching, boolean doValidate) { 
		.............

	    } else if ((a.flags&AlarmManager.FLAG_WAKE_FROM_IDLE) != 0) {
	            // 如果此次设置的alarmClock触发时间早于上一次设置的alarmClock触发时间，那么会把此次的AlamClock更新保存到全局的mNextWakeFromIdle中，mNextWakeFromIdle很关键，doze的状态机切换判断会调用到此alarm信息
	        if (mNextWakeFromIdle == null || mNextWakeFromIdle.whenElapsed > a.whenElapsed) {
	            mNextWakeFromIdle = a;
	            // If this wake from idle is earlier than whatever was previously scheduled,
	            // and we are currently idling, then we need to rebatch alarms in case the idle
	            // until time needs to be updated.
	            if (mPendingIdleUntil != null) {
	                needRebatch = true;
	            }
	        }
	    }
		.............. 
	}


此外在进入doze的idle阶段后，系统会重新将之前保存到 mAlarmBatches 中的所有alarms重新rebatch，在此过程中会走如下代码，此时会判断每个alarm的flags类型，若不是如下类型的flags那么对应alarm会被暂存到 mPendingWhileIdleAlarms 中，不会向下设置：

    private void setImplLocked(Alarm a, boolean rebatching, boolean doValidate) {
		..................
         //刚进入doze idle阶段时系统会设置一个将要退出idle的alarm，并保存到 mPendingIdleUntil 中，因此会走去下代码
        } else if (mPendingIdleUntil != null) {
            // We currently have an idle until alarm scheduled; if the new alarm has
            // not explicitly stated it wants to run while idle, then put it on hold.
            // 注意 从上述代码可以看到，alarmClock的flags包含AlarmManager.FLAG_WAKE_FROM_IDLE标记，因此不会进入下面的if语句，会按照正常流程设置下去，因此不会收到doze idle的影响；
            if ((a.flags&(AlarmManager.FLAG_ALLOW_WHILE_IDLE
                    | AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED
                    | AlarmManager.FLAG_WAKE_FROM_IDLE))
                    == 0) {
                mPendingWhileIdleAlarms.add(a);
                return;
            }
        }
		..................
     }

综上所述，在进入Doze的idle阶段，AlarmClock类型的alarm仍然会正常运行，不受影响。



### 2、 AlarmClock 可能会影响Doze状态机的切换时间等

#### 2.1、 AlarmClock 类型的alarm可能会影响处在Doze idle的时间长短

在刚进入Doze的idle阶段，系统会设置一个flags类型为FLAG_IDLE_UNTIL的alarm（此alarm的触发时间即为退出doze idle的时间），并保存到全局变量mPendingIdleUntil中，在 设置此alarm的过程中会走如下代码：

    private void setImplLocked(Alarm a, boolean rebatching, boolean doValidate) {
        // 因为设置doze的状态机idle alarm会对其flags添加FLAG_IDLE_UNTIL标记，因为会进入如下判断：
        if ((a.flags&AlarmManager.FLAG_IDLE_UNTIL) != 0) {
            // This is a special alarm that will put the system into idle until it goes off.
            // The caller has given the time they want this to happen at, however we need
            // to pull that earlier if there are existing alarms that have requested to
            // bring us out of idle at an earlier time.
            // 如果此标记退出idle的alarm 触发时间大于mNextWakeFromIdle（注意之前alarmClock类型的alarm就是设置到mNextWakeFromIdle ）的触发时间，那么：退出doze idle的时间就是mNextWakeFromIdle的触发时间；
            if (mNextWakeFromIdle != null && a.whenElapsed > mNextWakeFromIdle.whenElapsed) {
                a.when = a.whenElapsed = a.maxWhenElapsed = mNextWakeFromIdle.whenElapsed;
            }
		............. 
     }
 
#### 2.2、 AlarmClock 类型的alarm可能会影响处在Doze 状态机切换判断 

在Doze每次切换状态机的时候会经过如下判断：

 	 Doze代码路径： base\services\core\java\com\android\server\DeviceIdleController.jav
 
    void stepIdleStateLocked(String reason) {
        。。。。。
        final long now = SystemClock.elapsedRealtime();
            //下面的这句话表明，在doze状态机的切换过程中，最近的alarmClock触发时间小于：now+mConstants.MIN_TIME_TO_ALARM时间，那么Doze会直接返回active阶段，状态机不会再往下走;
        if ((now+mConstants.MIN_TIME_TO_ALARM) > mAlarmManager.getNextWakeFromIdleTime()) {
            // Whoops, there is an upcoming alarm.  We don't actually want to go idle.
            if (mState != STATE_ACTIVE) {
                becomeActiveLocked("alarm", Process.myUid());
                becomeInactiveIfAppropriateLocked();
            }
            return;
        }
        。。。。。
    }

    MIN_TIME_TO_ALARM = mParser.getLong(KEY_MIN_TIME_TO_ALARM,!COMPRESS_TIME ? 60 * 60 * 1000L : 6 * 60 * 1000L);


我们再看下mAlarmManager.getNextWakeFromIdleTime（）的实现,其实现对应到AlarmManagerService.java 的getNextWakeFromIdleTime（）方法：  
	
	@Override
	public long getNextWakeFromIdleTime() {
	    return getNextWakeFromIdleTimeImpl();
	}
	
	long getNextWakeFromIdleTimeImpl() {
		synchronized (mLock) {
		    return mNextWakeFromIdle != null ? mNextWakeFromIdle.whenElapsed : Long.MAX_VALUE;
		}
	}
  
从之前的判断我们知道mNextWakeFromIdle就是最近将要触发的alarmClock的alarm；因此，在doze状态机的切换过程中，最近的alarmClock触发时间如果小于：now+mConstants.MIN_TIME_TO_ALARM时间，那么Doze会直接返回active阶段，状态机不会再往下走，也就不会进入idle；







