---
layout:     post
title:      Alarm 和Doze的关系
subtitle:   
date:       2016-12-13
author:     WMY
header-img: img/02.jpg
catalog: 	 true
tags: 
    - Alarm
---



#  Alarm 和Doze的关系

###  Doze的简介 
为降低移动设备功耗，从Android M开始，增加了Doze机制，当用户不用移动设备，灭屏静止一段时间后，移动设备便会进入Doze的idle状态，此时对于非白名单的第三方应用：  

* APPs not allowed network access.
* App wakelocks ignored
* Alarms deferred, Excludes for alarms that you've set with thesetAlarmClock() method and AlarmManager.setAndAllowWhileIdle()
* WiFi scans are not performed.
* SyncAdaper syncs and JobScheduler jobs deferred until the next maintenance window.
* Apps receiving SMS and MMS messages so they can complete their processing.

每隔一段时间，doze会从idle阶段切换到maintenance阶段，恢复正常让应用联网进行数据同步等，然后再切回到idle阶段。
现在研究当进入doze的idle阶段后，普通的Alarm是如何被处理的呢？


###  进入idle模式后，普通alarm是如何被处理 

当进入Doze的idle阶段，此时设置的普通alarm（不带FLAG_ALLOW_WHILE_IDLE/ FLAG_WAKE_FROM_IDLE标志）会被临时放到mPendingWhileIdleAlarms，mPendingWhileIdleAlarms中的alarm不会被往下设置到底层alarm设备，因此也就不会有唤醒动作。具体流程如下：

a) 首先当进入idle时，会设置一个叫做mPendingIdleUntil 的alarm。DeviceIdleController中进入idle过程会执行如下代码：  

    case STATE_IDLE_MAINTENANCE:
        scheduleAlarmLocked(mNextIdleDelay, true);
        if (DEBUG) Slog.d(TAG, "Moved to STATE_IDLE. Next alarm in " + mNextIdleDelay +
                " ms.");

scheduleAlarmLocked 方法的实现如下：  

    void scheduleAlarmLocked(long delay, boolean idleUntil) {
        if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
        if (mMotionSensor == null) {
            // If there is no motion sensor on this device, then we won't schedule
            // alarms, because we can't determine if the device is not moving.  This effectively
            // turns off normal execution of device idling, although it is still possible to
            // manually poke it by pretending like the alarm is going off.
            return;
        }
        mNextAlarmTime = SystemClock.elapsedRealtime() + delay;
        if (idleUntil) {  //进入idle阶段时会执行如下代码：
            mAlarmManager.setIdleUntil(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                    mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        } else {
            mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP,
                    mNextAlarmTime, "DeviceIdleController.deep", mDeepAlarmListener, mHandler);
        }
    }

跟踪到AlarmManager 中：   

    /**
     * Schedule an idle-until alarm, which will keep the alarm manager * idle until
     * the given time.
     * @hide
     */
    public void setIdleUntil(int type, long triggerAtMillis, String tag, OnAlarmListener listener,
            Handler targetHandler) {
        setImpl(type, triggerAtMillis, WINDOW_EXACT, 0, FLAG_IDLE_UNTIL, null,
                listener, tag, targetHandler, null, null);
    }
 
最终经过各层调用会在AlarmManagerService中执行到setImplLocked()方法,执行过程中会走如下代码：   

        if ((a.flags&AlarmManager.FLAG_IDLE_UNTIL) != 0) {
            if (RECORD_DEVICE_IDLE_ALARMS) {
                if (mPendingIdleUntil == null) {
                    IdleDispatchEntry ent = new IdleDispatchEntry();
                    ent.uid = 0;
                    ent.pkg = "START IDLE";
                    ent.elapsedRealtime = SystemClock.elapsedRealtime();
                    mAllowWhileIdleDispatches.add(ent);
                }
            }
            mPendingIdleUntil = a;
            mConstants.updateAllowWhileIdleMinTimeLocked();
        


此时进入idle模式后，设置了一个 mPendingIdleUntil 的alarm。此Alarm是idle阶段的标志：
   
    // set to null if in idle mode; while in this mode, any alarms we don't want
    // to run during this time are placed in mPendingWhileIdleAlarms
    Alarm mPendingIdleUntil = null;



当mPendingIdleUntil 的Alarm设置之后，系统中凡是设置的Alarm在设置阶段走到setImplLocked 方法中会进行如下的判断，凡是不满足如下条件的Alarm，都会被暂存到mPendingWhileIdleAlarms中，然后return，不会再往下层设置，也就是说在idle阶段设置的普通alarm都被挂起了；   

        } else if (mPendingIdleUntil != null) {
            // We currently have an idle until alarm scheduled; if the new alarm has
            // not explicitly stated it wants to run while idle, then put it on hold.
            if ((a.flags&(AlarmManager.FLAG_ALLOW_WHILE_IDLE
                    | AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED
                    | AlarmManager.FLAG_WAKE_FROM_IDLE))
                    == 0) {
                mPendingWhileIdleAlarms.add(a);
                return;
            }
        }

从上面的代码可以看出，当进入doze的idle阶段，不在白名单中的第三方app设置的普通app都会被临时保存在mPendingIdleUntil中，不会被执行；那么当退出doze的idle阶段这些保存在mPendingIdleUntil中的alarm会被如何处理呢？


###  如何再执行mPendingWhileIdleAlarms中的alarm？ 

 标记idle阶段的mPendingIdleUntil alarm 在到期后将会触发，在alarm触发过程走到triggerAlarmsLocked方法中时，时会判断alarm是否为mPendingIdleUntil alarm，若是则会执行如下代码：  

        if (mPendingIdleUntil == alarm) {
            mPendingIdleUntil = null;
            rebatchAllAlarmsLocked(false); 
            restorePendingWhileIdleAlarmsLocked();
        }

restorePendingWhileIdleAlarmsLocked方法的部分实现如下：  

    void restorePendingWhileIdleAlarmsLocked() {
        if (RECORD_DEVICE_IDLE_ALARMS) {
            IdleDispatchEntry ent = new IdleDispatchEntry();
            ent.uid = 0;
            ent.pkg = "FINISH IDLE";
            ent.elapsedRealtime = SystemClock.elapsedRealtime();
            mAllowWhileIdleDispatches.add(ent);
        }

        // Bring pending alarms back into the main list.
        if (mPendingWhileIdleAlarms.size() > 0) {
            ArrayList<Alarm> alarms = mPendingWhileIdleAlarms;
            mPendingWhileIdleAlarms = new ArrayList<>();
            final long nowElapsed = SystemClock.elapsedRealtime();
            for (int i=alarms.size() - 1; i >= 0; i--) {
                Alarm a = alarms.get(i);
                reAddAlarmLocked(a, nowElapsed, false);
            }
        } 

reAddAlarmLocked方法的实现如下：  

    void reAddAlarmLocked(Alarm a, long nowElapsed, boolean doValidate) {
        a.when = a.origWhen;
        long whenElapsed = convertToElapsed(a.when, a.type);
        final long maxElapsed;
        if (a.windowLength == AlarmManager.WINDOW_EXACT) {
            // Exact
            maxElapsed = whenElapsed;
        } else {
            // Not exact.  Preserve any explicit window, otherwise recalculate
            // the window based on the alarm's new futurity.  Note that this
            // reflects a policy of preferring timely to deferred delivery.
            maxElapsed = (a.windowLength > 0)
                    ? (whenElapsed + a.windowLength)
                    : maxTriggerTime(nowElapsed, whenElapsed, a.repeatInterval);
        }
        a.whenElapsed = whenElapsed;
        a.maxWhenElapsed = maxElapsed;
        setImplLocked(a, true, doValidate);
    }
 
从上可以看出，当标记idle阶段的mPendingIdleUntil alarm 触发后，mPendingWhileIdleAlarms中的alarm会重新拿出来往下设置； setImplLocked(a, true, doValidate) 方法的实现如下：   

    private void setImplLocked(Alarm a, boolean rebatching, boolean doValidate) {

         .......中间处理省略...........

        boolean needRebatch = false;
		//进入doze的idle时设置的alarm flags即为：AlarmManager.FLAG_IDLE_UNTIL
        if ((a.flags&AlarmManager.FLAG_IDLE_UNTIL) != 0) {  
            if (RECORD_DEVICE_IDLE_ALARMS) {
                if (mPendingIdleUntil == null) {
                    IdleDispatchEntry ent = new IdleDispatchEntry();
                    ent.uid = 0;
                    ent.pkg = "START IDLE";
                    ent.elapsedRealtime = SystemClock.elapsedRealtime();
                    mAllowWhileIdleDispatches.add(ent);
                }
            }
            //此时将doze idle阶段标记的alarm保存在mPendingIdleUntil 全局变量中
            mPendingIdleUntil = a;
            mConstants.updateAllowWhileIdleMinTimeLocked();
            //将needRebatch设置为true，这句很关键
            needRebatch = true;
        } 

         .......中间处理省略...........

            //根据上面的设置，进入idle时，便会执行如下代码：
            if (needRebatch) {
                rebatchAllAlarmsLocked(false);
            }
 
rebatchAllAlarmsLocked方法的实现如下：   

    void rebatchAllAlarmsLocked(boolean doValidate) {
        //mAlarmBatches 是用于存在系统内设置的各种未触发的alarm数组
        ArrayList<Batch> oldSet = (ArrayList<Batch>) mAlarmBatches.clone();
        mAlarmBatches.clear();
        Alarm oldPendingIdleUntil = mPendingIdleUntil;
        final long nowElapsed = SystemClock.elapsedRealtime();
        final int oldBatches = oldSet.size();
        for (int batchNum = 0; batchNum < oldBatches; batchNum++) {
            Batch batch = oldSet.get(batchNum);
            final int N = batch.size();
            for (int i = 0; i < N; i++) {
                reAddAlarmLocked(batch.get(i), nowElapsed, doValidate);
            }
        }
        if (oldPendingIdleUntil != null && oldPendingIdleUntil != mPendingIdleUntil) {
            Slog.wtf(TAG, "Rebatching: idle until changed from " + oldPendingIdleUntil
                    + " to " + mPendingIdleUntil);
            if (mPendingIdleUntil == null) {
                // Somehow we lost this...  we need to restore all of the pending alarms.
                restorePendingWhileIdleAlarmsLocked();
            }
        }
        rescheduleKernelAlarmsLocked();
        updateNextAlarmClockLocked();
    }


从上面的代码可以看出，当刚进入doze的idle阶段时，此前系统内设置的各种alarm，会被拿出来重新进行设置，执行reAddAlarmLocked 方法，在reAddAlarmLocked方法的执行过程中，依然会执行如下代码，这样在idle之前系统内设置的未触发的普通alarm，依然会被暂存到mPendingWhileIdleAlarms数组中，这样就不会再idle阶段触发：  

        } else if (mPendingIdleUntil != null) {
            // We currently have an idle until alarm scheduled; if the new alarm has
            // not explicitly stated it wants to run while idle, then put it on hold.
            if ((a.flags&(AlarmManager.FLAG_ALLOW_WHILE_IDLE
                    | AlarmManager.FLAG_ALLOW_WHILE_IDLE_UNRESTRICTED
                    | AlarmManager.FLAG_WAKE_FROM_IDLE))
                    == 0) {
                mPendingWhileIdleAlarms.add(a);
                return;
            }
        }


下图为doze进入idle前后，对alarm处理的简单关系图：


   ![](http://i.imgur.com/ID0aAkR.png)
