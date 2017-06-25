---
layout:     post
title:      AlarmManagerService Time_tick 
subtitle:  
date:       2017-02-04
author:     BY
header-img: img/13.jpg
catalog: true
tags:
    - 工作
---

> “🙉🙉🙉 ”


## 前言

WMY 的 Blog 就这么开通了。

后面会整理资料介绍 AlarmManagerService Time_tick的运行机制


##1、TIME_TICK alarm的初始化:

在AlarmManagerService 初始化onStart过程中，会初始化一个action为ACTION_TIME_TICK的Intent：  

        mTimeTickSender = PendingIntent.getBroadcastAsUser(getContext(), 0,
                new Intent(Intent.ACTION_TIME_TICK).addFlags(
                        Intent.FLAG_RECEIVER_REGISTERED_ONLY
                                | Intent.FLAG_RECEIVER_FOREGROUND), 0,
                UserHandle.ALL);  
                
                .........
        // now that we have initied the driver schedule the alarm        
        mClockReceiver = new ClockReceiver();
        mClockReceiver.scheduleTimeTickEvent();


紧接着ClockReceiver 通过调用 scheduleTimeTickEvent（）方法启动intent为 mTimeTickSender的alarm： 
 
        public void scheduleTimeTickEvent() {
            final long currentTime = System.currentTimeMillis();
            final long nextTime = 60000 * ((currentTime / 60000) + 1);

            // Schedule this event for the amount of time that it would take to get to
            // the top of the next minute.
            final long tickEventDelay = nextTime - currentTime;

            final WorkSource workSource = null; // Let system take blame for time tick events.
            setImpl(ELAPSED_REALTIME, SystemClock.elapsedRealtime() + tickEventDelay, 0,
                    0, mTimeTickSender, null, null, AlarmManager.FLAG_STANDALONE, workSource,
                    null, Process.myUid(), "android");
        }  

亮屏情况下一分钟内该alarm会触发，然后发送action为：ACTION_TIME_TICK 的广播。StatusBarManagerService 中会接收到该广播，然后更新状态栏时间；
那么该alarm是如何实现循环设置触发的呢？ 接下来看下ClockReceiver的定义  

    class ClockReceiver extends BroadcastReceiver {
        public ClockReceiver() {
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_TIME_TICK);
            filter.addAction(Intent.ACTION_DATE_CHANGED);
            getContext().registerReceiver(this, filter);
     }

        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals(Intent.ACTION_TIME_TICK)) {
                if (DEBUG_BATCH) {
                    Slog.v(TAG, "Received TIME_TICK alarm; rescheduling");
                }
                Slog.v(TAG, "mSupportAlarmGrouping = " + mSupportAlarmGrouping +
                        "  mAmPlus = " + mAmPlus);
                scheduleTimeTickEvent();
            }  

从上可以看出AlarmManagerService中的ClockReceiver 会接收ACTION_TIME_TICK意图广播事件，当接收到该事件后会继续调用scheduleTimeTickEvent()设置新的intent为mTimeTickSender的alarm，从而实现循环触发，更新状态栏时间.  

##2、亮灭屏阶段time_tick的响应机制


###2.1 灭屏阶段time_tick alarm的触发机制

在AlarmManagerService初始化走到onStart()阶段，会初始化启动一个叫AlarmThread的线程，该线程会阻塞式的监听底层alarm的触发动作，并根据底层触发返回值result，来遍历Alarm列表找到上层对应的alarm及其所在的batch组，假如其所在的组内不存在能唤醒设备的类型的alarm，那么该batch内的alarm会被暂存到mPendingNonWakeupAlarms数组中，等下一次能够唤醒设备的alarm触发时在一起触发;
当alarm触发时，通过调用triggerAlarmsLocked来获取alarm所在的batch组及其是否存在wakeup类型的alarm：
 
    boolean triggerAlarmsLocked(ArrayList<Alarm> triggerList, final long nowELAPSED,
            final long nowRTC) {
        boolean hasWakeup = false;
        // batches are temporally sorted, so we need only pull from the
        // start of the list until we either empty it or hit a batch
        // that is not yet deliverable
        while (mAlarmBatches.size() > 0) {
            Batch batch = mAlarmBatches.get(0);
            if (batch.start > nowELAPSED) {
                // Everything else is scheduled for the future
                break;
            }
            // We will (re)schedule some alarms now; don't let that interfere
            // with delivery of this current batch
            mAlarmBatches.remove(0);

            final int N = batch.size();
            for (int i = 0; i < N; i++) {
                Alarm alarm = batch.get(i);
  
				............

                triggerList.add(alarm);
 
				............

                if (alarm.wakeup) {
                    hasWakeup = true;
                }

				............
            }
        }
 
		............

        return hasWakeup;
    }


从上可以看出，通过triggerAlarmsLocked方法便可以将该小于该alarm触发时间及该alarm其所在的batch组内的alarms取出来放到 triggerList中，并判断是否存在wakeup类型的alarm； 

当底层alarm 触发时AlarmThread线程会执行如下代码：

		//判断是否存在能够唤醒设备的alarm存在
        boolean hasWakeup = triggerAlarmsLocked(triggerList, nowELAPSED, nowRTC);
		//若不存在唤醒设备的alarm，且延迟时间没有超过最大范围则进入如下判断
        if (!hasWakeup && checkAllowNonWakeupDelayLocked(nowELAPSED)) {
            // if there are no wakeup alarms and the screen is off, we can
            // delay what we have so far until the future.
            if (mPendingNonWakeupAlarms.size() == 0) {
                mStartCurrentDelayTime = nowELAPSED;
                mNextNonWakeupDeliveryTime = nowELAPSED
                        + ((currentNonWakeupFuzzLocked(nowELAPSED)*3)/2);
            }
			//将要触发的alarm放在mPendingNonWakeupAlarms数组中延迟触发
            mPendingNonWakeupAlarms.addAll(triggerList);
            mNumDelayedAlarms += triggerList.size();
            rescheduleKernelAlarmsLocked();
            updateNextAlarmClockLocked();
        } else {
            // now deliver the alarm intents; if there are pending non-wakeup
            // alarms, we need to merge them in to the list.  note we don't
            // just deliver them first because we generally want non-wakeup
            // alarms delivered after wakeup alarms.
			//当出现能唤醒设备的alarm触发时，会顺带把之前暂存在mPendingNonWakeupAlarms的alarm也拿出来一起触发
            rescheduleKernelAlarmsLocked();
            updateNextAlarmClockLocked();
            if (mPendingNonWakeupAlarms.size() > 0) {
                calculateDeliveryPriorities(mPendingNonWakeupAlarms);
                triggerList.addAll(mPendingNonWakeupAlarms);
                Collections.sort(triggerList, mAlarmDispatchComparator);
                final long thisDelayTime = nowELAPSED - mStartCurrentDelayTime;
                mTotalDelayTime += thisDelayTime;
                if (mMaxDelayTime < thisDelayTime) {
                    mMaxDelayTime = thisDelayTime;
                }
                mPendingNonWakeupAlarms.clear();
            }
            deliverAlarmsLocked(triggerList, nowELAPSED);
        }
    }

此外由于time_tick alarm是type为3，也就是在系统休眠时不会唤醒系统的类型的alarm，假如其触发时间前面的batch内以及其所在的batch组内不存在可以唤醒设备的alarm，那么该time_tick的alarm触发时就会被放到mPendingNonWakeupAlarms数组中，延迟触发；

###2.2 亮屏阶段time_tick alarm的触发机制

在AlarmManagerService类中，有个监听亮灭屏的广播接收者InteractiveStateReceiver，当屏幕状态发生变化时，它会接收到该信息并执行相应的操作：

    class InteractiveStateReceiver extends BroadcastReceiver {
        public InteractiveStateReceiver() {
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_SCREEN_OFF);
            filter.addAction(Intent.ACTION_SCREEN_ON);
            filter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);
            getContext().registerReceiver(this, filter);
        }

        @Override
        public void onReceive(Context context, Intent intent) {
            synchronized (mLock) {
                interactiveStateChangedLocked(Intent.ACTION_SCREEN_ON.equals(intent.getAction()));
            }
        }
    }

interactiveStateChangedLocked方法实现如下：

    void interactiveStateChangedLocked(boolean interactive) {
        if (mInteractive != interactive) {
            mInteractive = interactive;
            final long nowELAPSED = SystemClock.elapsedRealtime();
            if (interactive) { //当亮屏时interactive 为true
                if (mPendingNonWakeupAlarms.size() > 0) {
                    final long thisDelayTime = nowELAPSED - mStartCurrentDelayTime;
                    mTotalDelayTime += thisDelayTime;
                    if (mMaxDelayTime < thisDelayTime) {
                        mMaxDelayTime = thisDelayTime;
                    }
					//将暂存到mPendingNonWakeupAlarms中的alarm（包括time_tick alarm）取出来一起触发
                    deliverAlarmsLocked(mPendingNonWakeupAlarms, nowELAPSED);
                    mPendingNonWakeupAlarms.clear();
                }
                if (mNonInteractiveStartTime > 0) {
                    long dur = nowELAPSED - mNonInteractiveStartTime;
                    if (dur > mNonInteractiveTime) {
                        mNonInteractiveTime = dur;
                    }
                }
            } else {
                mNonInteractiveStartTime = nowELAPSED;
            }
        }
    }

   通过以上流程可以看出，当灭屏一段时间后，time_tick alarm触发时可能会被临时暂存在mPendingNonWakeupAlarms数组中，延迟触发；当亮屏时InteractiveStateReceiver广播接收者会接收到亮屏广播，调用interactiveStateChangedLocked方法，将暂存在mPendingNonWakeupAlarms数组中的alarm取出来集中触发，这时time_tick alarm就算之前是暂存在mPendingNonWakeupAlarms数组中，这时也会触发执行对应的操作，不会影响状态栏的时间更新；


