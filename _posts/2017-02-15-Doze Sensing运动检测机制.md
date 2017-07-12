---
layout:     post
title:      Doze Sensing AnyMotionDetector运动检测机制
subtitle:   
date:       2017-02-15
author:     WMY
header-img: img/20.jpg
catalog: true
tags:
    - 工作
---
 
## 前言

 暂未开始

## AnyMotion机制  

检测运动的AnyMotionDetector会在DeviceIdleController的onBootPhase阶段初始化：

    @Override
    public void onBootPhase(int phase) {
                         。。。。。。。。

                mAnyMotionDetector = new AnyMotionDetector(
                        (PowerManager) getContext().getSystemService(Context.POWER_SERVICE),
                        mHandler, mSensorManager, this, angleThreshold); 
 
在Doze执行完pending阶段后会调用如下程序，检测设备是否运动：

	case STATE_IDLE_PENDING:
	    mState = STATE_SENSING;
	    if (DEBUG) Slog.d(TAG, "Moved from STATE_IDLE_PENDING to STATE_SENSING.");
	    EventLogTags.writeDeviceIdle(mState, reason);
	    //设置一个计时器，运动检测超时后会重回Inactive阶段
	    scheduleSensingTimeoutAlarmLocked(mConstants.SENSING_TIMEOUT);
	    cancelLocatingLocked();
	    mNotMoving = false;
	    mLocated = false;
	    mLastGenericLocation = null;
	    mLastGpsLocation = null;
	    mAnyMotionDetector.checkForAnyMotion();//注意这里开始检测运动 

这里设置运动检测超时alarm方法实现如下：

    void scheduleSensingTimeoutAlarmLocked(long delay) {
        if (DEBUG) Slog.d(TAG, "scheduleSensingAlarmLocked(" + delay + ")");
        mNextSensingTimeoutAlarmTime = SystemClock.elapsedRealtime() + delay;
        mAlarmManager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, mNextSensingTimeoutAlarmTime,
            "DeviceIdleController.sensing", mSensingTimeoutAlarmListener, mHandler);
    } 
如果超过后，mSensingTimeoutAlarmListener会监听到该alarm触发，并执行如下程序：

    private final AlarmManager.OnAlarmListener mSensingTimeoutAlarmListener
            = new AlarmManager.OnAlarmListener() {
        @Override
        public void onAlarm() {
            if (mState == STATE_SENSING) {
                synchronized (DeviceIdleController.this) {
                   //调用如下方法会重回到Inactive阶段
                    becomeInactiveIfAppropriateLocked();
                }
            }
        }
    }; 

因此从上可以看到，在规定的时间内，如果无法确定设备是否运动的话，设备将重新回到Inactive阶段，下面来看检测设备是否运动的checkForAnyMotion方法实现原理。

### AnyMotionDetector的checkForAnyMotion运动检测机制 

checkForAnyMotion方法中，首先会判断运动传感器是否在检测中，假如不在检测中，则持有一个wakelock锁，并调用startOrientationMeasurementLocked方法，检测设备运动状态

     /*
 	  * Acquire accel data until we determine AnyMotion status.
     */
    public void checkForAnyMotion() {
        if (DEBUG) {
            Slog.d(TAG, "checkForAnyMotion(). mState = " + mState);
        }
        if (mState != STATE_ACTIVE) {//说明运动传感器不在检测中
            synchronized (mLock) {
                mState = STATE_ACTIVE;
                if (DEBUG) {
                    Slog.d(TAG, "Moved from STATE_INACTIVE to STATE_ACTIVE.");
                }
                mCurrentGravityVector = null;
                mPreviousGravityVector = null;
                mWakeLock.acquire();
                startOrientationMeasurementLocked();
            }
        }
    }

再来看startOrientationMeasurementLocked方法实现：

    private void startOrientationMeasurementLocked() {
        if (DEBUG) Slog.d(TAG, "startOrientationMeasurementLocked: mMeasurementInProgress=" +
            mMeasurementInProgress + ", (mAccelSensor != null)=" + (mAccelSensor != null));
        if (!mMeasurementInProgress && mAccelSensor != null) {
			//调用传感器mAccelSensor，通过mListener监听SAMPLING_INTERVAL_MILLIS * 1000时间内侧采样数据，mListener的实现详见1.1
            if (mSensorManager.registerListener(mListener, mAccelSensor,
                    SAMPLING_INTERVAL_MILLIS * 1000)) {
                mMeasurementInProgress = true;
                mRunningStats.reset();
            } 
            //同时启动一个Handler超时方法，在ACCELEROMETER_DATA_TIMEOUT_MILLIS时间内，若未获取到有效设备信息将会触发mMeasurementTimeout线程执行，mMeasurementTimeout的实现见1.2
            Message msg = Message.obtain(mHandler, mMeasurementTimeout);
            msg.setAsynchronous(true);
            mHandler.sendMessageDelayed(msg, ACCELEROMETER_DATA_TIMEOUT_MILLIS);
        }
    } 

####  1.1 mListener的实现如下：

    private final SensorEventListener mListener = new SensorEventListener() {
        @Override
        public void onSensorChanged(SensorEvent event) {
            int status = RESULT_UNKNOWN; //初始状态为RESULT_UNKNOWN
            synchronized (mLock) {
                Vector3 accelDatum = new Vector3(SystemClock.elapsedRealtime(), event.values[0], event.values[1], event.values[2]);
				//获取传感器返回的坐标信息，通过如下方法确定设备位置
                mRunningStats.accumulate(accelDatum); 
                // 如果获取到足够多的采样数据，则停止检测
                if (mRunningStats.getSampleCount() >= mNumSufficientSamples) {
					//stopOrientationMeasurementLocked方法实现见1.3
                    status = stopOrientationMeasurementLocked();
                }
            }
            if (status != RESULT_UNKNOWN) {
				//如果最终获取到有效运动状态信息，则将结果反馈给DeviceIdleController
                mCallback.onAnyMotionResult(status);
            }
        } 
		.............
    }; 

####  1.2 mMeasurementTimeout的实现如下：

    private final Runnable mMeasurementTimeout = new Runnable() {
      @Override
      public void run() {
          int status = RESULT_UNKNOWN;
          synchronized (mLock) {
              if (DEBUG) Slog.i(TAG, "mMeasurementTimeout. Failed to collect sufficient accel " +
                      "data within " + ACCELEROMETER_DATA_TIMEOUT_MILLIS + " ms. Stopping " +
                      "orientation measurement.");
              status = stopOrientationMeasurementLocked();
          }
          if (status != RESULT_UNKNOWN) {
              mCallback.onAnyMotionResult(status);
          }
      }
  }; 

####  1.3 stopOrientationMeasurementLocked的实现如下：

    private int stopOrientationMeasurementLocked() {
        if (DEBUG) Slog.d(TAG, "stopOrientationMeasurement. mMeasurementInProgress=" +
                mMeasurementInProgress);
        int status = RESULT_UNKNOWN;
        if (mMeasurementInProgress) {
            mSensorManager.unregisterListener(mListener);
            mHandler.removeCallbacks(mMeasurementTimeout);
            long detectionEndTime = SystemClock.elapsedRealtime();
            mMeasurementInProgress = false;
            mPreviousGravityVector = mCurrentGravityVector;
            mCurrentGravityVector = mRunningStats.getRunningAverage();
            if (DEBUG) {
                Slog.d(TAG, "mRunningStats = " + mRunningStats.toString());
                String currentGravityVectorString = (mCurrentGravityVector == null) ?
                        "null" : mCurrentGravityVector.toString();
                String previousGravityVectorString = (mPreviousGravityVector == null) ?
                        "null" : mPreviousGravityVector.toString();
                Slog.d(TAG, "mCurrentGravityVector = " + currentGravityVectorString);
                Slog.d(TAG, "mPreviousGravityVector = " + previousGravityVectorString);
            }
            mRunningStats.reset();
            //通过getStationaryStatus获取当前设备位置状态，见1.4节
            status = getStationaryStatus();
            if (DEBUG) Slog.d(TAG, "getStationaryStatus() returned " + status);
            if (status != RESULT_UNKNOWN) {//若设备位置状态不为RESULT_UNKNOWN，即已确定当前设备位置状态，则将WakeLock锁释放，传感器状态置为STATE_INACTIVE
                mWakeLock.release();
                if (DEBUG) {
                    Slog.d(TAG, "Moved from STATE_ACTIVE to STATE_INACTIVE. status = " + status);
                }
                mState = STATE_INACTIVE;
            } else {
                /*
                 * Unknown due to insufficient measurements. Schedule another orientation
                 * measurement.
                 */
                if (DEBUG) Slog.d(TAG, "stopOrientationMeasurementLocked(): another measurement" +
                        " scheduled in " + ORIENTATION_MEASUREMENT_INTERVAL_MILLIS +
                        " milliseconds.");

                //若不能确定当前位置状态，则延迟ORIENTATION_MEASUREMENT_INTERVAL_MILLIS后，执行mSensorRestart线程方法，重新检测运动状态，注意这种情况下并没有释放WakeLock锁，即此时设备并没有进入深睡眠
                Message msg = Message.obtain(mHandler, mSensorRestart);
                msg.setAsynchronous(true);
                mHandler.sendMessageDelayed(msg, ORIENTATION_MEASUREMENT_INTERVAL_MILLIS);
            }
        }
        return status;
    }

####  1.4 判断设备位置状态的getStationaryStatus方法实现如下：

    /*
     * Updates mStatus to the current AnyMotion status.
     */
    public int getStationaryStatus() {
        if ((mPreviousGravityVector == null) || (mCurrentGravityVector == null)) {
            return RESULT_UNKNOWN;
        }
        Vector3 previousGravityVectorNormalized = mPreviousGravityVector.normalized();
        Vector3 currentGravityVectorNormalized = mCurrentGravityVector.normalized();
        float angle = previousGravityVectorNormalized.angleBetween(currentGravityVectorNormalized);
        if (DEBUG) Slog.d(TAG, "getStationaryStatus: angle = " + angle
                + " energy = " + mRunningStats.getEnergy());
        if ((angle < mThresholdAngle) && (mRunningStats.getEnergy() < THRESHOLD_ENERGY)) {
            return RESULT_STATIONARY;
        } else if (Float.isNaN(angle)) {
          /**
           * Floating point rounding errors have caused the angle calcuation's dot product to 
           * exceed 1.0. In such case, we report RESULT_MOVED to prevent devices from rapidly
           * retrying this measurement.
           */
            return RESULT_MOVED;
        }
        long diffTime = mCurrentGravityVector.timeMillisSinceBoot -
                mPreviousGravityVector.timeMillisSinceBoot;
        if (diffTime > STALE_MEASUREMENT_TIMEOUT_MILLIS) {
            if (DEBUG) Slog.d(TAG, "getStationaryStatus: mPreviousGravityVector is too stale at " +
                    diffTime + " ms ago. Returning RESULT_UNKNOWN.");
            return RESULT_UNKNOWN;
        }
        return RESULT_MOVED;
    }

以上方法主要是通过比较前后两次设备的坐标值差异来判断设备当前的状态，注意监测设备运动状态至少会调用两次运动传感器，因为第一次比较时，上一次的坐标状态mPreviousGravityVector == null；
假如运动感器确定设备状态后，会回调DeviceIdleController的onAnyMotionResult方法，该方法具体实现如下： 

	   @Override
	    public void onAnyMotionResult(int result) {
	        if (DEBUG) Slog.d(TAG, "onAnyMotionResult(" + result + ")");
	        if (result != AnyMotionDetector.RESULT_UNKNOWN) {
	            synchronized (this) {
	                cancelSensingTimeoutAlarmLocked();
	            }
	        }
	        if (result == AnyMotionDetector.RESULT_MOVED) {
	            if (DEBUG) Slog.d(TAG, "RESULT_MOVED received.");
	            synchronized (this) {
	                handleMotionDetectedLocked(mConstants.INACTIVE_TIMEOUT, "sense_motion");
	            }
	        } else if (result == AnyMotionDetector.RESULT_STATIONARY) {
	            if (DEBUG) Slog.d(TAG, "RESULT_STATIONARY received.");
	            if (mState == STATE_SENSING) {
	                // If we are currently sensing, it is time to move to locating.
	                synchronized (this) {
	                    mNotMoving = true;
	                    stepIdleStateLocked("s:stationary");
	                }
	            } else if (mState == STATE_LOCATING) {
	                // If we are currently locating, note that we are not moving and step
	                // if we have located the position.
	                synchronized (this) {
	                    mNotMoving = true;
	                    if (mLocated) {
	                        stepIdleStateLocked("s:stationary");
	                    }
	                }
	            }
	        }
	    }



  
