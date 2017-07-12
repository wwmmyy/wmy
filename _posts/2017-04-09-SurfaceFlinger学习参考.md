---
layout:     post
title:      SurfaceFlinger 学习参考
subtitle:   
date:       2017-04-09
author:     WMY
header-img: img/01.jpg
catalog: true
tags:
    - 工作
---   

# android Gui系统之SurfaceFlinger 


**本文转自:【Joyfulmath---A Software Designer 博客】**

**原文地址：http://blog.csdn.net/jinzhuojun/article/details/17293325**



GUI 是任何系统都很重要的一块。

android GUI大体分为4大块。

1）SurfaceFlinger

2）WMS

3）View机制

4）InputMethod

这块内容非常之多，但是理解后，可以触类旁通，其实现在主流的系统，包括andorid，ios在构架上，都是有很多相识之处。

我们先来讲SurfaceFlinger


## 1.OpenGL & OpenGL ES  

OPenGL ES 是android系统绘画的基础。关于OpenGL部分，可以百度了解下。

先来看一个OpenGL & SurfaceFlinger之间的框架图：

![](http://i.imgur.com/npci4QF.png)

从底层往上看：
1）linux内核提供统一的设备驱动，/dev/graphics/fb*

2) Android HAL 提供2个接口 Gralloc & fb
fb 负责打开framebuffer，提供接口操作。gralloc负责管理帧缓冲区的分配和释放。composer是HAL中另一个重要的功能，它主要是给厂商定制UI合成。SurfaceFlinger中负责HWComposer会用到这个功能。而且关键是HWComposer还负责产生VSync信号，这是本期SurfaceFlinger的重点。

3）由于OpenGL是一套通用的库（大部分就是接口），所以它需要一个本地的实现。andorid平台OpenGL有2个本地窗口，FrameBufferNativeWindow & Surface。

4）OpenGL可以有软件 或者依托于硬件实现，具体的运行状态，就是由EGL来配置。

5）SurfaceFlinger持有一个成员数组mDisplays来支持各种显示设备。DisplayDevices在初始化的时候调用EGL来搭建OpenGL的环境。

## 2.Android的硬件接口HAL  

HAL需要满足android系统和厂商的要求

### 2.1硬件接口的抽象 

从面向对象角度来讲，接口的概念就是由C++非常容易实现，但是HAL很多代码是C语言描述的。
这就需要一种技巧来实现面向对象。
定义一种结构，子类的成员变量第一个类型是父类的结构就可以了。抽象方法可以用函数指针来实现。
其实这个就是C++多态实现的基本原理，具体可参考《深入理解C++对象模型》

### 2.2接口的稳定性 

Android已经把各个硬件都接口都统一定义在：

libhardware/include/hardware/ 具体代码可以参考：https://github.com/CyanogenMod/android_hardware_libhardware/tree/cm-12.0/include/hardware

## 3.Android显示设备：Gralloc & FrameBuffer 

FrameBuffer是linux环境下显示设备的统一接口。从而让用户设备不需要做太多的操作，就可以适配多种显示设备。

FramwBuffer本质上就是一套接口。android系统不会直接操作显示驱动，而通过HAL层来封装。而HAL中操作驱动的模块就是gralloc。

### 3.1Gralloc模块的加载 

gralloc通过FrameBufferNativeWindow 来加载的：
	
	FramebufferNativeWindow::FramebufferNativeWindow() 
	    : BASE(), fbDev(0), grDev(0), mUpdateOnDemand(false)
	{
	    hw_module_t const* module;
	    if (hw_get_module(GRALLOC_HARDWARE_MODULE_ID, &module) == 0) {
	        int stride;
	        int err;
	        int i;
	        err = framebuffer_open(module, &fbDev);
	        ALOGE_IF(err, "couldn't open framebuffer HAL (%s)", strerror(-err));
	        
	        err = gralloc_open(module, &grDev);
	        ALOGE_IF(err, "couldn't open gralloc HAL (%s)", strerror(-err));
	
	        // bail out if we can't initialize the modules
	        if (!fbDev || !grDev)
	            return;
	        
	        mUpdateOnDemand = (fbDev->setUpdateRect != 0);
	        
	        // initialize the buffer FIFO
	        if(fbDev->numFramebuffers >= MIN_NUM_FRAME_BUFFERS &&
	           fbDev->numFramebuffers <= MAX_NUM_FRAME_BUFFERS){
	            mNumBuffers = fbDev->numFramebuffers;
	        } else {
	            mNumBuffers = MIN_NUM_FRAME_BUFFERS;
	        }
	        mNumFreeBuffers = mNumBuffers;
	        mBufferHead = mNumBuffers-1;
	
	        /*
	         * This does not actually change the framebuffer format. It merely
	         * fakes this format to surfaceflinger so that when it creates
	         * framebuffer surfaces it will use this format. It's really a giant
	         * HACK to allow interworking with buggy gralloc+GPU driver
	         * implementations. You should *NEVER* need to set this for shipping
	         * devices.
	         */
	#ifdef FRAMEBUFFER_FORCE_FORMAT
	        *((uint32_t *)&fbDev->format) = FRAMEBUFFER_FORCE_FORMAT;
	#endif
	
	        for (i = 0; i < mNumBuffers; i++)
	        {
	                buffers[i] = new NativeBuffer(
	                        fbDev->width, fbDev->height, fbDev->format, GRALLOC_USAGE_HW_FB);
	        }
	
	        for (i = 0; i < mNumBuffers; i++)
	        {
	                err = grDev->alloc(grDev,
	                        fbDev->width, fbDev->height, fbDev->format,
	                        GRALLOC_USAGE_HW_FB, &buffers[i]->handle, &buffers[i]->stride);
	
	                ALOGE_IF(err, "fb buffer %d allocation failed w=%d, h=%d, err=%s",
	                        i, fbDev->width, fbDev->height, strerror(-err));
	
	                if (err)
	                {
	                        mNumBuffers = i;
	                        mNumFreeBuffers = i;
	                        mBufferHead = mNumBuffers-1;
	                        break;
	                }
	        }
	
	        const_cast<uint32_t&>(ANativeWindow::flags) = fbDev->flags; 
	        const_cast<float&>(ANativeWindow::xdpi) = fbDev->xdpi;
	        const_cast<float&>(ANativeWindow::ydpi) = fbDev->ydpi;
	        const_cast<int&>(ANativeWindow::minSwapInterval) = 
	            fbDev->minSwapInterval;
	        const_cast<int&>(ANativeWindow::maxSwapInterval) = 
	            fbDev->maxSwapInterval;
	    } else {
	        ALOGE("Couldn't get gralloc module");
	    }
	
	    ANativeWindow::setSwapInterval = setSwapInterval;
	    ANativeWindow::dequeueBuffer = dequeueBuffer;
	    ANativeWindow::queueBuffer = queueBuffer;
	    ANativeWindow::query = query;
	    ANativeWindow::perform = perform;
	
	    ANativeWindow::dequeueBuffer_DEPRECATED = dequeueBuffer_DEPRECATED;
	    ANativeWindow::lockBuffer_DEPRECATED = lockBuffer_DEPRECATED;
	    ANativeWindow::queueBuffer_DEPRECATED = queueBuffer_DEPRECATED;
	}

我们继续深入看：

galloc的父类，最终是：

libhardware\include\hardware\hardware.h

	typedef struct hw_module_methods_t {
	    /** Open a specific device */
	    int (*open)(const struct hw_module_t* module, const char* id,
	            struct hw_device_t** device);
	
	} hw_module_methods_t;

只有一个open方法，也就是所有的厂商都需要实现开启设备的方法。

看下fb的打开的代码：

libhardware\modules\gralloc\framebuffer.cpp
	
	int fb_device_open(hw_module_t const* module, const char* name,
	        hw_device_t** device)
	{
	    int status = -EINVAL;
	    if (!strcmp(name, GRALLOC_HARDWARE_FB0)) {
	        /* initialize our state here */
	        fb_context_t *dev = (fb_context_t*)malloc(sizeof(*dev));
	        memset(dev, 0, sizeof(*dev));
	
	        /* initialize the procs */
	        dev->device.common.tag = HARDWARE_DEVICE_TAG;
	        dev->device.common.version = 0;
	        dev->device.common.module = const_cast<hw_module_t*>(module);
	        dev->device.common.close = fb_close;
	        dev->device.setSwapInterval = fb_setSwapInterval;
	        dev->device.post            = fb_post;
	        dev->device.setUpdateRect = 0;
	
	        private_module_t* m = (private_module_t*)module;
	        status = mapFrameBuffer(m);
	        if (status >= 0) {
	            int stride = m->finfo.line_length / (m->info.bits_per_pixel >> 3);
	            int format = (m->info.bits_per_pixel == 32)
	                         ? (m->info.red.offset ? HAL_PIXEL_FORMAT_BGRA_8888 : HAL_PIXEL_FORMAT_RGBX_8888)
	                         : HAL_PIXEL_FORMAT_RGB_565;
	            const_cast<uint32_t&>(dev->device.flags) = 0;
	            const_cast<uint32_t&>(dev->device.width) = m->info.xres;
	            const_cast<uint32_t&>(dev->device.height) = m->info.yres;
	            const_cast<int&>(dev->device.stride) = stride;
	            const_cast<int&>(dev->device.format) = format;
	            const_cast<float&>(dev->device.xdpi) = m->xdpi;
	            const_cast<float&>(dev->device.ydpi) = m->ydpi;
	            const_cast<float&>(dev->device.fps) = m->fps;
	            const_cast<int&>(dev->device.minSwapInterval) = 1;
	            const_cast<int&>(dev->device.maxSwapInterval) = 1;
	            *device = &dev->device.common;
	        }
	    }
	    return status;
	}

首先check设备名是否正确。

分配dev的空间，这是一个壳。
然后初始化dev。
提供fb的核心接口
内存映射

	status = mapFrameBuffer(m);
然后是建立壳 & 核心间的关系。

这样就打开了fb设备。

在回到FrameBufferNativeWindow 可以看到：

        err = framebuffer_open(module, &fbDev);
        ALOGE_IF(err, "couldn't open framebuffer HAL (%s)", strerror(-err));
        
        err = gralloc_open(module, &grDev);
        ALOGE_IF(err, "couldn't open gralloc HAL (%s)", strerror(-err));
fb打开的驱动信息在fbDev，gralloc打开的信息在grDev中。

fbDev负责的是主屏幕，grDev负责图形缓冲去的分配和释放。

所以FrameBufferNativeWindow控制这SurfaceFlinger的基础。

## 4.FrameBufferNativeWindow 

### 4.1FramebufferNativeWindow  

在OpenGL中，我们不断提及本地窗口的概念，在Android中，native window一共由2个。
一个是面向管理者（SurfaceFlinger）的 FramebufferNativeWindow
另一个是面像APP的，surface。
先来看第一种：
首先看下定义的地方：

	class FramebufferNativeWindow 
	    : public ANativeObjectBase<
	        ANativeWindow, 
	        FramebufferNativeWindow, 
	        LightRefBase<FramebufferNativeWindow> >
	{
ANativeWindow是什么东西？

ANativeWindow是OpenGL 在android平台的显示类型。

所以FramebufferNativeWindow就是一种Open GL可以显示的类型。

FramebufferNativeWindow的构造函数上面已经贴出来了，进一步分析如下：

1）加载module，上面已经分析过了。

2）打开fb & gralloc，也已经分析过了。

3）根据fb的设备属性，获得buffer数。这个buffer后面会解释。

4）给每个buffer初始化，并分配空间。这里new NativeBuffer只是指定buffer的类型，或者分配了一个指针，但是没有分配内存，所以还需要alloc操作。

5）为本地窗口属性赋值。

目前buffer默认值是在2~3,后面会介绍3缓冲技术，就会用到3个buffer。

双缓冲技术：

把一组图画，画到屏幕上，画图是需要时间的，如果时间间隔比较长，图片就是一个一个的画在屏幕的，看上去就会卡。

如果先把图片放在一个缓冲buffer中，待全部画好后，把buffer直接显示在屏幕上，这就是双缓冲技术。

### 4.2dequeuebuffer 

	int FramebufferNativeWindow::dequeueBuffer(ANativeWindow* window, 
	        ANativeWindowBuffer** buffer, int* fenceFd)
	{
	    FramebufferNativeWindow* self = getSelf(window);
	    Mutex::Autolock _l(self->mutex);
	    framebuffer_device_t* fb = self->fbDev;
	
	    int index = self->mBufferHead++;
	    if (self->mBufferHead >= self->mNumBuffers)
	        self->mBufferHead = 0;
	
	    // wait for a free non-front buffer
	    while (self->mNumFreeBuffers < 2) {
	        self->mCondition.wait(self->mutex);
	    }
	    ALOG_ASSERT(self->buffers[index] != self->front);
	
	    // get this buffer
	    self->mNumFreeBuffers--;
	    self->mCurrentBufferIndex = index;
	
	    *buffer = self->buffers[index].get();
	    *fenceFd = -1;
	
	    return 0;
	}
代码不多，但是却是核心功能，通过它来获取一块可渲染的buffer。

1）获取FramebufferNativeWindow对象。为什么没有使用this 而是使用了传入ANativeWindow的方式，此处我们并不关心。

2）获得一个Autolock的锁，函数结束，自动解锁。

3）获取mBufferHead变量，这里自增，也就是使用下一个buffer，一共只有3个，（原因上面已经解释），所以循环取值。

4）如果没有可用的缓冲区，等待bufferqueue释放。一旦获取后，可用buffer就自减

## 5. Surface 

Surface是另一个本地窗口，主要和app这边交互。注意：app层java代码无法直接调用surface，只是概念上surface属于app这一层的。

首先Surface是ANativeWindow的一个子类。

可以推测，surface需要解决如下几个问题：

1）面向上层（java层）提供画板。由谁来分配这块内存

2）与SurfaceFlinger是什么关系

	Surface::Surface(
	        const sp<IGraphicBufferProducer>& bufferProducer,
	        bool controlledByApp)
sp<IGraphicBufferProducer>& bufferProducer 是分配surface内存的。它到底是什么呢？

![](http://i.imgur.com/l2SYMDQ.png)

先来看看从ViewRootImpl到获取surface的过程。
ViewRootImpl持有一个java层的surface对象，开始是空的。
后续的流程见上面的流程图。也就是-说ViewRootImpl持有的surface对象，最终是对SurfaceComposerClient的创建的surface的一个“引用”。
由此分析可以看到 一个ISurfaceClient->ISurfaceComposerClient->IGraphicBufferProducer.当然binder需要一个实名的server来注册。
在ServiceManager中可以看到，这些服务查询的是“SurfaceFlinger”。
也就是，这些东东都是SurfaceFlinger的内容。

	SurfaceFlinger::SurfaceFlinger()
	    :   BnSurfaceComposer(),

SurfaceFlinger是BnSurfaceComposer的一个子类。也就是ISurfaceComposer的一个实现。

surface虽然是为app层服务的，但是本质上还是由SurfaceFlinger来管理的。

SurfaceFlinger怎么创建和管理surface，需要通过BufferQueue，将在下一篇讨论。

## 6 BufferQueue 

BufferQueue是SurfaceFlinger管理和消费surface的中介

每个应用 可以由几个BufferQueue？

应用绘制UI 所需的内存从何而来？

应用和SurfaceFlinger 如何互斥共享资源的访问？

### 6.1 Buffer的状态 
 

	const char* BufferSlot::bufferStateName(BufferState state) {
	    switch (state) {
	        case BufferSlot::DEQUEUED: return "DEQUEUED";
	        case BufferSlot::QUEUED: return "QUEUED";
	        case BufferSlot::FREE: return "FREE";
	        case BufferSlot::ACQUIRED: return "ACQUIRED";
	        default: return "Unknown";
	    }
	} 

状态变迁如下：FREE->DEQUEUED->QUEUED->ACQUIRED->FREE

![](http://i.imgur.com/R1OKUTZ.png)


producer & comsumer 分别是什么？

应用程序需要刷新UI，所以就会产生surface到BufferQueue，so，producer 可以认为就是应用程序，也可以是上一篇里面介绍的ISurfaceComposerClient。
即 producer 为：应用程序

comsumer不用看也知道，就是SurfaceFlinger
因此大致流程为：：：
1）当应用程序刷新UI，获取buffer的缓冲区，这个操作就是dequeue。经过dequeue后，该buffer被producer锁定，其他owner就不能插手了。

2）然后把surface写入到该buffer里面，当producer认为写入结束后，就执行queue的操作，并把buffer归还给bufferqueue，Owner也变为bufferqueue。

3）当一段buffer里面有数据以后，comsumer就会收到消息，然后去获取该buffer。

	void BufferQueue::ProxyConsumerListener::onFrameAvailable(
	        const android::BufferItem& item) {
	    sp<ConsumerListener> listener(mConsumerListener.promote());
	    if (listener != NULL) {
	        listener->onFrameAvailable(item);
	    }
	}

bufferqueue里面就是comsumerlistener，当有可以使用的buffer以后，就会通知comsumer使用：

	// mGraphicBuffer points to the buffer allocated for this slot or is NULL
	    // if no buffer has been allocated.
	    sp<GraphicBuffer> mGraphicBuffer;

可以看到注释，bufferqueue的mSlot[64],并不是都有内容的，也就是mSlot存的是buferr的指针，如果没有，就存null，slot的个数在andorid5.0 里面定义在BufferQueueDefs.h里面，

	enum { NUM_BUFFER_SLOTS = 64 };


### 6.2 Buffer内存的出处 

既然producer是主动操作，所以如果dequeue的时候，已经获取了内存，后面的操作也就不需要分配内存了：

	status_t BufferQueueProducer::dequeueBuffer(int *outSlot,
	        sp<android::Fence> *outFence, bool async,
	        uint32_t width, uint32_t height, uint32_t format, uint32_t usage) {
	    ATRACE_CALL();
	    { // Autolock scope
	        Mutex::Autolock lock(mCore->mMutex);
	        mConsumerName = mCore->mConsumerName;
	    } // Autolock scope
	
	    BQ_LOGV("dequeueBuffer: async=%s w=%u h=%u format=%#x, usage=%#x",
	            async ? "true" : "false", width, height, format, usage);
	
	    if ((width && !height) || (!width && height)) {
	        BQ_LOGE("dequeueBuffer: invalid size: w=%u h=%u", width, height);
	        return BAD_VALUE;
	    }
	
	    status_t returnFlags = NO_ERROR;
	    EGLDisplay eglDisplay = EGL_NO_DISPLAY;
	    EGLSyncKHR eglFence = EGL_NO_SYNC_KHR;
	    bool attachedByConsumer = false;
	
	    { // Autolock scope
	        Mutex::Autolock lock(mCore->mMutex);
	        mCore->waitWhileAllocatingLocked();
	
	        if (format == 0) {
	            format = mCore->mDefaultBufferFormat;
	        }
	
	        // Enable the usage bits the consumer requested
	        usage |= mCore->mConsumerUsageBits;
	
	        int found;
	        status_t status = waitForFreeSlotThenRelock("dequeueBuffer", async,
	                &found, &returnFlags);
	        if (status != NO_ERROR) {
	            return status;
	        }
	
	        // This should not happen
	        if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
	            BQ_LOGE("dequeueBuffer: no available buffer slots");
	            return -EBUSY;
	        }
	
	        *outSlot = found;
	        ATRACE_BUFFER_INDEX(found);
	
	        attachedByConsumer = mSlots[found].mAttachedByConsumer;
	
	        const bool useDefaultSize = !width && !height;
	        if (useDefaultSize) {
	            width = mCore->mDefaultWidth;
	            height = mCore->mDefaultHeight;
	        }
	
	        mSlots[found].mBufferState = BufferSlot::DEQUEUED;
	
	        const sp<GraphicBuffer>& buffer(mSlots[found].mGraphicBuffer);
	        if ((buffer == NULL) ||
	                (static_cast<uint32_t>(buffer->width) != width) ||
	                (static_cast<uint32_t>(buffer->height) != height) ||
	                (static_cast<uint32_t>(buffer->format) != format) ||
	                ((static_cast<uint32_t>(buffer->usage) & usage) != usage))
	        {
	            mSlots[found].mAcquireCalled = false;
	            mSlots[found].mGraphicBuffer = NULL;
	            mSlots[found].mRequestBufferCalled = false;
	            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
	            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
	            mSlots[found].mFence = Fence::NO_FENCE;
	
	            returnFlags |= BUFFER_NEEDS_REALLOCATION;
	        }
	
	        if (CC_UNLIKELY(mSlots[found].mFence == NULL)) {
	            BQ_LOGE("dequeueBuffer: about to return a NULL fence - "
	                    "slot=%d w=%d h=%d format=%u",
	                    found, buffer->width, buffer->height, buffer->format);
	        }
	
	        eglDisplay = mSlots[found].mEglDisplay;
	        eglFence = mSlots[found].mEglFence;
	        *outFence = mSlots[found].mFence;
	        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
	        mSlots[found].mFence = Fence::NO_FENCE;
	    } // Autolock scope
	
	    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
	        status_t error;
	        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
	        sp<GraphicBuffer> graphicBuffer(mCore->mAllocator->createGraphicBuffer(
	                    width, height, format, usage, &error));
	        if (graphicBuffer == NULL) {
	            BQ_LOGE("dequeueBuffer: createGraphicBuffer failed");
	            return error;
	        }
	
	        { // Autolock scope
	            Mutex::Autolock lock(mCore->mMutex);
	
	            if (mCore->mIsAbandoned) {
	                BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
	                return NO_INIT;
	            }
	
	            mSlots[*outSlot].mFrameNumber = UINT32_MAX;
	            mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
	        } // Autolock scope
	    }
	
	    if (attachedByConsumer) {
	        returnFlags |= BUFFER_NEEDS_REALLOCATION;
	    }
	
	    if (eglFence != EGL_NO_SYNC_KHR) {
	        EGLint result = eglClientWaitSyncKHR(eglDisplay, eglFence, 0,
	                1000000000);
	        // If something goes wrong, log the error, but return the buffer without
	        // synchronizing access to it. It's too late at this point to abort the
	        // dequeue operation.
	        if (result == EGL_FALSE) {
	            BQ_LOGE("dequeueBuffer: error %#x waiting for fence",
	                    eglGetError());
	        } else if (result == EGL_TIMEOUT_EXPIRED_KHR) {
	            BQ_LOGE("dequeueBuffer: timeout waiting for fence");
	        }
	        eglDestroySyncKHR(eglDisplay, eglFence);
	    }
	
	    BQ_LOGV("dequeueBuffer: returning slot=%d/%" PRIu64 " buf=%p flags=%#x",
	            *outSlot,
	            mSlots[*outSlot].mFrameNumber,
	            mSlots[*outSlot].mGraphicBuffer->handle, returnFlags);
	
	    return returnFlags;
	}

step1：BufferQueueProducer::waitForFreeSlotThenRelock 循环的主要作用就是查找可以使用的slot。

step2：释放不需要的buffer，并且统计已分配的内存。

 		// Free up any buffers that are in slots beyond the max buffer count
        for (int s = maxBufferCount; s < BufferQueueDefs::NUM_BUFFER_SLOTS; ++s) {
            assert(mSlots[s].mBufferState == BufferSlot::FREE);
            if (mSlots[s].mGraphicBuffer != NULL) {
                mCore->freeBufferLocked(s);
                *returnFlags |= RELEASE_ALL_BUFFERS;
            }
        }

step2：释放不需要的buffer，并且统计已分配的内存。

	for (int s = 0; s < maxBufferCount; ++s) {
            switch (mSlots[s].mBufferState) {
                case BufferSlot::DEQUEUED:
                    ++dequeuedCount;
                    break;
                case BufferSlot::ACQUIRED:
                    ++acquiredCount;
                    break;
                case BufferSlot::FREE:
                    // We return the oldest of the free buffers to avoid
                    // stalling the producer if possible, since the consumer
                    // may still have pending reads of in-flight buffers
                    if (*found == BufferQueueCore::INVALID_BUFFER_SLOT ||
                            mSlots[s].mFrameNumber < mSlots[*found].mFrameNumber) {
                        *found = s;
                    }
                    break;
                default:
                    break;
            }
        }

如果有合适的，found 就是可以使用的buffer编号。

如果dequeue too many，but comsumer还来不及消耗掉，这个时候，有可能会导致OOM，所以，判断是否在队列里面有过多的buffer。

等待comsumer消耗后，释放互斥锁。

        if (tryAgain) {
            // Return an error if we're in non-blocking mode (producer and
            // consumer are controlled by the application).
            // However, the consumer is allowed to briefly acquire an extra
            // buffer (which could cause us to have to wait here), which is
            // okay, since it is only used to implement an atomic acquire +
            // release (e.g., in GLConsumer::updateTexImage())
            if (mCore->mDequeueBufferCannotBlock &&
                    (acquiredCount <= mCore->mMaxAcquiredBufferCount)) {
                return WOULD_BLOCK;
            }
            mCore->mDequeueCondition.wait(mCore->mMutex);
        }

在返回dqueueBuffer这个方法：如果没有找到free的slot，就直接返回错误。当然正常情况下是不会发生的。

        // This should not happen
        if (found == BufferQueueCore::INVALID_BUFFER_SLOT) {
            BQ_LOGE("dequeueBuffer: no available buffer slots");
            return -EBUSY;
        }
 .

	 mSlots[found].mBufferState = BufferSlot::DEQUEUED;

把找到的buffer的状态设为DEQUEUE。

在判断了mSlot[found]的属性以后，它可能是空的，也有可能不符合当前需要的buffer的size，就给mSlot[found]分配新的属性和内存

	if ((buffer == NULL) ||
                (static_cast<uint32_t>(buffer->width) != width) ||
                (static_cast<uint32_t>(buffer->height) != height) ||
                (static_cast<uint32_t>(buffer->format) != format) ||
                ((static_cast<uint32_t>(buffer->usage) & usage) != usage))
        {
            mSlots[found].mAcquireCalled = false;
            mSlots[found].mGraphicBuffer = NULL;
            mSlots[found].mRequestBufferCalled = false;
            mSlots[found].mEglDisplay = EGL_NO_DISPLAY;
            mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
            mSlots[found].mFence = Fence::NO_FENCE;

            returnFlags |= BUFFER_NEEDS_REALLOCATION;
        }

        if (CC_UNLIKELY(mSlots[found].mFence == NULL)) {
            BQ_LOGE("dequeueBuffer: about to return a NULL fence - "
                    "slot=%d w=%d h=%d format=%u",
                    found, buffer->width, buffer->height, buffer->format);
        }

        eglDisplay = mSlots[found].mEglDisplay;
        eglFence = mSlots[found].mEglFence;
        *outFence = mSlots[found].mFence;
        mSlots[found].mEglFence = EGL_NO_SYNC_KHR;
        mSlots[found].mFence = Fence::NO_FENCE;
    } // Autolock scope

    if (returnFlags & BUFFER_NEEDS_REALLOCATION) {
        status_t error;
        BQ_LOGV("dequeueBuffer: allocating a new buffer for slot %d", *outSlot);
        sp<GraphicBuffer> graphicBuffer(mCore->mAllocator->createGraphicBuffer(
                    width, height, format, usage, &error));
        if (graphicBuffer == NULL) {
            BQ_LOGE("dequeueBuffer: createGraphicBuffer failed");
            return error;
        }

        { // Autolock scope
            Mutex::Autolock lock(mCore->mMutex);

            if (mCore->mIsAbandoned) {
                BQ_LOGE("dequeueBuffer: BufferQueue has been abandoned");
                return NO_INIT;
            }

            mSlots[*outSlot].mFrameNumber = UINT32_MAX;
            mSlots[*outSlot].mGraphicBuffer = graphicBuffer;
        } // Autolock scope
    }

这样buffer对应的内存就是在producer，dequeue操作的时候分配内存的。（if need）

### 6.3应用程序和BufferQueue的关系 

首先看一张surface各类之间的关系：

![](http://i.imgur.com/Rar5kBT.png)

这里多了一个Layer的东西，Layer代表一个画面的图层（在android app的角度以前是没有图层的概念的，虽然framework有）。SurfaceFlinger混合，就是把所有的layer做混合。

我们先来看createlayer：

	status_t SurfaceFlinger::createLayer(
        const String8& name,
        const sp<Client>& client,
        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp)

这里关键是handler & gbp这2个参数。

我们看下去代码：layer 是从createNormalLayer里面来的
	
	status_t SurfaceFlinger::createNormalLayer(const sp<Client>& client,
	        const String8& name, uint32_t w, uint32_t h, uint32_t flags, PixelFormat& format,
	        sp<IBinder>* handle, sp<IGraphicBufferProducer>* gbp, sp<Layer>* outLayer)
	{
	    // initialize the surfaces
	    switch (format) {
	    case PIXEL_FORMAT_TRANSPARENT:
	    case PIXEL_FORMAT_TRANSLUCENT:
	        format = PIXEL_FORMAT_RGBA_8888;
	        break;
	    case PIXEL_FORMAT_OPAQUE:
	        format = PIXEL_FORMAT_RGBX_8888;
	        break;
	    }
	
	    *outLayer = new Layer(this, client, name, w, h, flags);
	    status_t err = (*outLayer)->setBuffers(w, h, format, flags);
	    if (err == NO_ERROR) {
	        *handle = (*outLayer)->getHandle();
	        *gbp = (*outLayer)->getProducer();
	    }
	
	    ALOGE_IF(err, "createNormalLayer() failed (%s)", strerror(-err));
	    return err;
	}

producer跟踪源代码可以看到：

	class MonitoredProducer : public IGraphicBufferProducer

通过bind机制，可以认为就是BufferQueue的一个子类。

所以，每一个bufferqueue对应的都是一个layer。

看下handler：

	sp<IBinder> Layer::getHandle() {
	    Mutex::Autolock _l(mLock);
	
	    LOG_ALWAYS_FATAL_IF(mHasSurface,
	            "Layer::getHandle() has already been called");
	
	    mHasSurface = true;
	
	    /*
	     * The layer handle is just a BBinder object passed to the client
	     * (remote process) -- we don't keep any reference on our side such that
	     * the dtor is called when the remote side let go of its reference.
	     *
	     * LayerCleaner ensures that mFlinger->onLayerDestroyed() is called for
	     * this layer when the handle is destroyed.
	     */
	
	    class Handle : public BBinder, public LayerCleaner {
	        wp<const Layer> mOwner;
	    public:
	        Handle(const sp<SurfaceFlinger>& flinger, const sp<Layer>& layer)
	            : LayerCleaner(flinger, layer), mOwner(layer) {
	        }
	    };
	
	    return new Handle(mFlinger, this);
	}

没有什么东西，就是LayerCleaner，

它的设计目的就是SurfaceFlinger来清除图层。

所以我们可以得出结论

1）一个app对应一个surfaceFlinger,可以有多个layer，从而对应多个bufferqueue

2）surface的缓冲区内存是BufferQueue在进行dequeue的时候分配的，属于client端。

3）App & SurfaceFlinger都通过bufferQueue来分配和使用缓冲区，所以互斥操作是由BufferQueue来实现。


## 7.SurfaceFlinger 

SurfaceFlinger在前面的篇幅了，多有涉及。

SurfaceFlinger是GUI刷新UI的核心，所以任何关于SurfaceFlinger的改进都会对android UI系统有重大影响。

SurfaceFlinger主要分为4个部分

1）黄油计划---project butter

2）启动过程

3）SurfaceFlinger & BufferQueue的关系

4）Vsync信号的处理

### 7.1黄油计划 

就是给android系统，图上一层“黄油”。我们来看看andorid是怎么给SurfaceFlinger涂上这层黄油的。

butter 由2个组成部分，Vsync & Triple buffer。

Triple buffer：

上面讲到双缓冲区技术，也提到FrameBufferNativeWindow 在申请buffer的时候，可以是2，或者是3.

这个3 就是马上要讲到的Triple Buffer技术。

我们先会过来看看双缓冲技术。

之前说 双缓冲，是把一个buffer放在bitmap上，等到这个所有元素都准备好以后，在把bitmap刷到屏幕上。

这样会解决卡顿的感觉。

我们考虑一种情况，假设屏幕刷新频率是66Hz，CPU频率是100Hz.

之前已经讲了双缓冲技术，这里简单过一下。

如上面的假设，UI的刷新是0.015s，而buffer的准备是0.01s

一个Frame Buffer代表一帧图像。0.01s：

此时，buffer已经准备好数据，而显示器只显示了图像的2/3 0.015s

显示器显示了第一帧图像，而buffer已经填充了第二帧的1/3 0.02s

Buffer已经准备好了第二帧，而显示器出现了问题，1/3的内容属于第二帧，2/3的内容属于第一帧。

这就是android引入双缓冲技术的原因。

如果buffer准备的时间，比屏幕刷新图像的速度慢呢？

显示屏的每一次刷新，就是对显示器屏幕的扫描，但是它是有间隔的（物理设备嘛，肯定有这个间隔）。

典型的PC显示器屏幕刷新频率是60Hz，这是因为一秒60帧，从人的角度看，就会觉得很流畅。

所以间隔1/60秒，也就是16ms 如果我们准备时间<=16ms,那就可以做到“无缝连接”。画面就很流程。

这段空隙称为VBI。 这个时间就是交换缓冲区最佳的时间。而这个交换的动作就是Vsync 也是SurfaceFlinger的重点。

如果我们图像准备时间<=16ms. OK,画面是很流畅的，但是我们无法保证设备性能一定很very good。所以也有可能画面准备时间超过16ms

![](http://i.imgur.com/YQK3cAE.png)

我们看看这张图。

刚开始buffer里面有数据A，这时候，可以直接显示在屏幕上。
过了16ms以后，数据B还没准备好，屏幕只能继续显示A。这样就浪费了依次交换的机会。
到下一次交换，B被显示在屏幕上。 这里有段时间被浪费了。
等到下一次A的时候，过了16ms，还是没有准备好，继续浪费。所以双缓冲区技术，也有很大浪费。
有没有办法规避呢，
比如上图 B & A之间的这段时间，如果我增加一个buffer，C。
这样B准备好以后，虽然C没有好，但是B可以显示在屏幕上，等到下一次16ms到了以后，C已经准备好了，这样可以很大程度上减少CPU时间的浪费。
也就是空间换时间的一种思想。
所以多缓冲区就是，就是可以根据系统的实际内存情况，来判断buffer的数量。


### 7.2 SurfaceFlinger的启动 

SurfaceFlinger 我们前面已经说了，它其实就是一个service。

	void SurfaceFlinger::onFirstRef()
	{
	    mEventQueue.init(this);
	}

初始化事件队列。

	void MessageQueue::init(const sp<SurfaceFlinger>& flinger)
	{
	    mFlinger = flinger;
	    mLooper = new Looper(true);
	    mHandler = new Handler(*this);
	}

创建了looper & Handler

但是这个looper什么时候起来的呢

	void MessageQueue::waitMessage() {
	    do {
	        IPCThreadState::self()->flushCommands();
	        int32_t ret = mLooper->pollOnce(-1);
	        switch (ret) {
	            case Looper::POLL_WAKE:
	            case Looper::POLL_CALLBACK:
	                continue;
	            case Looper::POLL_ERROR:
	                ALOGE("Looper::POLL_ERROR");
	            case Looper::POLL_TIMEOUT:
	                // timeout (should not happen)
	                continue;
	            default:
	                // should not happen
	                ALOGE("Looper::pollOnce() returned unknown status %d", ret);
	                continue;
	        }
	    } while (true);
	}

可以看到最终会调用looper启动函数。可以看到Looper::POLL_TIMEOUT: android什么都没做，尽管它们不应该发生。

其实handler兜了一圈，发现最后还是回到surfaceflinger来处理：
	
	void SurfaceFlinger::onMessageReceived(int32_t what) {
	    ATRACE_CALL();
	    switch (what) {
	        case MessageQueue::TRANSACTION: {
	            handleMessageTransaction();
	            break;
	        }
	        case MessageQueue::INVALIDATE: {
	            bool refreshNeeded = handleMessageTransaction();
	            refreshNeeded |= handleMessageInvalidate();
	            refreshNeeded |= mRepaintEverything;
	            if (refreshNeeded) {
	                // Signal a refresh if a transaction modified the window state,
	                // a new buffer was latched, or if HWC has requested a full
	                // repaint
	                signalRefresh();
	            }
	            break;
	        }
	        case MessageQueue::REFRESH: {
	            handleMessageRefresh();
	            break;
	        }
	    }
	}



### 7.3 client 

任何有UI界面App都在surfaceflinger里面有client。

所以是一个app对应一个surfaceflinger里面的client（ISurfaceComposerClient）。

- ![](http://i.imgur.com/Nro6pvY.png)

下面我们来分析surfaceflinger的2个重要函数：


	sp<ISurfaceComposerClient> SurfaceFlinger::createConnection()
	{
	    sp<ISurfaceComposerClient> bclient;
	    sp<Client> client(new Client(this));
	    status_t err = client->initCheck();
	    if (err == NO_ERROR) {
	        bclient = client;
	    }
	    return bclient;
	}

返回ISurfaceComposerClient，也就是client的bind对象实体。

其实就上面标红的一句，进行必要的有效性检查，现在代码：

	status_t Client::initCheck() const {
	    return NO_ERROR;
	}

有了clinet以后，看下surface的产生。

	status_t Client::createSurface(
	        const String8& name,
	        uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
	        sp<IBinder>* handle,
	        sp<IGraphicBufferProducer>* gbp)
	{
	    /*
	     * createSurface must be called from the GL thread so that it can
	     * have access to the GL context.
	     */
	
	    class MessageCreateLayer : public MessageBase {
	        SurfaceFlinger* flinger;
	        Client* client;
	        sp<IBinder>* handle;
	        sp<IGraphicBufferProducer>* gbp;
	        status_t result;
	        const String8& name;
	        uint32_t w, h;
	        PixelFormat format;
	        uint32_t flags;
	    public:
	        MessageCreateLayer(SurfaceFlinger* flinger,
	                const String8& name, Client* client,
	                uint32_t w, uint32_t h, PixelFormat format, uint32_t flags,
	                sp<IBinder>* handle,
	                sp<IGraphicBufferProducer>* gbp)
	            : flinger(flinger), client(client),
	              handle(handle), gbp(gbp),
	              name(name), w(w), h(h), format(format), flags(flags) {
	        }
	        status_t getResult() const { return result; }
	        virtual bool handler() {
	            result = flinger->createLayer(name, client, w, h, format, flags,
	                    handle, gbp);
	            return true;
	        }
	    };
	
	    sp<MessageBase> msg = new MessageCreateLayer(mFlinger.get(),
	            name, this, w, h, format, flags, handle, gbp);
	    mFlinger->postMessageSync(msg);
	    return static_cast<MessageCreateLayer*>( msg.get() )->getResult();
	}


## 8.Vsync 

### 8.1 Vsync概论 

VSYNC（Vertical Synchronization）是一个相当古老的概念，对于游戏玩家，它有一个更加大名鼎鼎的中文名字—-垂直同步。

“垂直同步(vsync)”指的是显卡的输出帧数和屏幕的垂直刷新率相同，这完全是一个CRT显示器上的概念。其实无论是VSYNC还是垂直同步这个名字，

因为LCD根本就没有垂直扫描的这种东西，因此这个名字本身已经没有意义。但是基于历史的原因，这个名称在图形图像领域被沿袭下来。
在当下，垂直同步的含义我们可以理解为，使得显卡生成帧的速度和屏幕刷新的速度的保持一致。举例来说，如果屏幕的刷新率为60Hz，那么生成帧的速度就应该被固定在1/60 s。

### 8.2 VSync信号的产生和分发 

VSync信号的产生在android_frameworks_native\services\surfaceflinger\DisplayHardware\HWComposer.cpp里面定义：

	HWComposer::HWComposer(
	        const sp<SurfaceFlinger>& flinger,
	        EventHandler& handler)
	    : mFlinger(flinger),
	      mFbDev(0), mHwc(0), mNumDisplays(1),
	      mCBContext(new cb_context),
	      mEventHandler(handler),
	      mDebugForceFakeVSync(false),
	      mVDSEnabled(false)
	{
	    for (size_t i =0 ; i<MAX_HWC_DISPLAYS ; i++) {
	        mLists[i] = 0;
	    }
	
	    for (size_t i=0 ; i<HWC_NUM_PHYSICAL_DISPLAY_TYPES ; i++) {
	        mLastHwVSync[i] = 0;
	        mVSyncCounts[i] = 0;
	    }
	
	    char value[PROPERTY_VALUE_MAX];
	    property_get("debug.sf.no_hw_vsync", value, "0");
	    mDebugForceFakeVSync = atoi(value);
	
	    bool needVSyncThread = true;
	
	    // Note: some devices may insist that the FB HAL be opened before HWC.
	    int fberr = loadFbHalModule();
	    loadHwcModule();
	
	    if (mFbDev && mHwc && hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1)) {
	        // close FB HAL if we don't needed it.
	        // FIXME: this is temporary until we're not forced to open FB HAL
	        // before HWC.
	        framebuffer_close(mFbDev);
	        mFbDev = NULL;
	    }
	
	    // If we have no HWC, or a pre-1.1 HWC, an FB dev is mandatory.
	    if ((!mHwc || !hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))
	            && !mFbDev) {
	        ALOGE("ERROR: failed to open framebuffer (%s), aborting",
	                strerror(-fberr));
	        abort();
	    }
	
	    // these display IDs are always reserved
	    for (size_t i=0 ; i<NUM_BUILTIN_DISPLAYS ; i++) {
	        mAllocatedDisplayIDs.markBit(i);
	    }
	
	    if (mHwc) {
	        ALOGI("Using %s version %u.%u", HWC_HARDWARE_COMPOSER,
	              (hwcApiVersion(mHwc) >> 24) & 0xff,
	              (hwcApiVersion(mHwc) >> 16) & 0xff);
	        if (mHwc->registerProcs) {
	            mCBContext->hwc = this;
	            mCBContext->procs.invalidate = &hook_invalidate;
	            mCBContext->procs.vsync = &hook_vsync;
	            if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))
	                mCBContext->procs.hotplug = &hook_hotplug;
	            else
	                mCBContext->procs.hotplug = NULL;
	            memset(mCBContext->procs.zero, 0, sizeof(mCBContext->procs.zero));
	            mHwc->registerProcs(mHwc, &mCBContext->procs);
	        }
	
	        // don't need a vsync thread if we have a hardware composer
	        needVSyncThread = false;
	        // always turn vsync off when we start
	        eventControl(HWC_DISPLAY_PRIMARY, HWC_EVENT_VSYNC, 0);
	
	        // the number of displays we actually have depends on the
	        // hw composer version
	        if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_3)) {
	            // 1.3 adds support for virtual displays
	            mNumDisplays = MAX_HWC_DISPLAYS;
	        } else if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1)) {
	            // 1.1 adds support for multiple displays
	            mNumDisplays = NUM_BUILTIN_DISPLAYS;
	        } else {
	            mNumDisplays = 1;
	        }
	    }
	
	    if (mFbDev) {
	        ALOG_ASSERT(!(mHwc && hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1)),
	                "should only have fbdev if no hwc or hwc is 1.0");
	
	        DisplayData& disp(mDisplayData[HWC_DISPLAY_PRIMARY]);
	        disp.connected = true;
	        disp.format = mFbDev->format;
	        DisplayConfig config = DisplayConfig();
	        config.width = mFbDev->width;
	        config.height = mFbDev->height;
	        config.xdpi = mFbDev->xdpi;
	        config.ydpi = mFbDev->ydpi;
	#ifdef QCOM_BSP
	        config.secure = true; //XXX: Assuming primary is always true
	#endif
	        config.refresh = nsecs_t(1e9 / mFbDev->fps);
	        disp.configs.push_back(config);
	        disp.currentConfig = 0;
	    } else if (mHwc) {
	        // here we're guaranteed to have at least HWC 1.1
	        for (size_t i =0 ; i<NUM_BUILTIN_DISPLAYS ; i++) {
	            queryDisplayProperties(i);
	        }
	    }
	
	    // read system property for VDS solution
	    // This property is expected to be setup once during bootup
	    if( (property_get("persist.hwc.enable_vds", value, NULL) > 0) &&
	        ((!strncmp(value, "1", strlen("1"))) ||
	        !strncasecmp(value, "true", strlen("true")))) {
	        //HAL virtual display is using VDS based implementation
	        mVDSEnabled = true;
	    }
	
	    if (needVSyncThread) {
	        // we don't have VSYNC support, we need to fake it
	        mVSyncThread = new VSyncThread(*this);
	    }
	#ifdef QCOM_BSP
	    // Threshold Area to enable GPU Tiled Rect.
	    property_get("debug.hwc.gpuTiledThreshold", value, "1.9");
	    mDynThreshold = atof(value);
	#endif
	}

是否需要模拟产生VSync信号，默认是开启的。

  // Note: some devices may insist that the FB HAL be opened before HWC.
    int fberr = loadFbHalModule();
    loadHwcModule();

打开fb & hwc设备的HAL模块。

        if (mHwc->registerProcs) {
            mCBContext->hwc = this;
            mCBContext->procs.invalidate = &hook_invalidate;
            mCBContext->procs.vsync = &hook_vsync;
            if (hwcHasApiVersion(mHwc, HWC_DEVICE_API_VERSION_1_1))
                mCBContext->procs.hotplug = &hook_hotplug;
            else
                mCBContext->procs.hotplug = NULL;
            memset(mCBContext->procs.zero, 0, sizeof(mCBContext->procs.zero));
            mHwc->registerProcs(mHwc, &mCBContext->procs);
        }

        // don't need a vsync thread if we have a hardware composer
        needVSyncThread = false;

 硬件VSync信号启动，不需要软件模拟。

    if (needVSyncThread) {
        // we don't have VSYNC support, we need to fake it
        mVSyncThread = new VSyncThread(*this);
    }

如果需要，启动VSyncThread线程来开启软件模拟。

#### 8.2.1硬件产生 

	mHwc->registerProcs(mHwc, &mCBContext->procs);
hwc会产生回调：procs.vsync & procs.invalidate 信号。

	void HWComposer::vsync(int disp, int64_t timestamp) {
	    if (uint32_t(disp) < HWC_NUM_PHYSICAL_DISPLAY_TYPES) {
	        {
	            Mutex::Autolock _l(mLock);
	
	            // There have been reports of HWCs that signal several vsync events
	            // with the same timestamp when turning the display off and on. This
	            // is a bug in the HWC implementation, but filter the extra events
	            // out here so they don't cause havoc downstream.
	            if (timestamp == mLastHwVSync[disp]) {
	                ALOGW("Ignoring duplicate VSYNC event from HWC (t=%" PRId64 ")",
	                        timestamp);
	                return;
	            }
	
	            mLastHwVSync[disp] = timestamp;
	        }
	
	        char tag[16];
	        snprintf(tag, sizeof(tag), "HW_VSYNC_%1u", disp);
	        ATRACE_INT(tag, ++mVSyncCounts[disp] & 1);
	
	        mEventHandler.onVSyncReceived(disp, timestamp);
	    }
	}
 最终会通知mEventHandler的消息，这个handler从那里来的呢？

	void SurfaceFlinger::init(){
	
	 mHwc = new HWComposer(this,
	            *static_cast<HWComposer::EventHandler *>(this));
	}
Yes，handler就是SurfaceFlinger，对嘛。SurfaceFlinger就是Surface合成的总管，所以这个信号一定会被它接收。

	class SurfaceFlinger : public BnSurfaceComposer,
                       private IBinder::DeathRecipient,
                       private HWComposer::EventHandler


#### 8.2.2软件模拟信号 

	bool HWComposer::VSyncThread::threadLoop() {
	    { // scope for lock
	        Mutex::Autolock _l(mLock);
	        while (!mEnabled) {
	            mCondition.wait(mLock);
	        }
	    }
	
	    const nsecs_t period = mRefreshPeriod;
	    const nsecs_t now = systemTime(CLOCK_MONOTONIC);
	    nsecs_t next_vsync = mNextFakeVSync;
	    nsecs_t sleep = next_vsync - now;
	    if (sleep < 0) {
	        // we missed, find where the next vsync should be
	        sleep = (period - ((now - next_vsync) % period));
	        next_vsync = now + sleep;
	    }
	    mNextFakeVSync = next_vsync + period;
	
	    struct timespec spec;
	    spec.tv_sec  = next_vsync / 1000000000;
	    spec.tv_nsec = next_vsync % 1000000000;
	
	    int err;
	    do {
	        err = clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &spec, NULL);
	    } while (err<0 && errno == EINTR);
	
	    if (err == 0) {
	        mHwc.mEventHandler.onVSyncReceived(0, next_vsync);
	    }
	
	    return true;
	}
判断系统Vsync信号开关。然后计算下一次刷新的时间点。

 	const nsecs_t period = mRefreshPeriod;

刷新间隔，CLOCK_MONOTONIC是从系统开机后的时间间隔（tick累加）

得到需要等待的时间sleep，和下一次vsync信号的时间点。

然后一个do while的操作，来等待信号时间点的到来。

最后，发出这个信号。

这里还有个情况，就是一开始的地方，mEnable变量，系统可以设置enable来控制vsync信号的产生。

	void HWComposer::VSyncThread::setEnabled(bool enabled) 


### 8.3 SurfaceFlinger处理Vsync信号 

在4.4以前，Vsync的处理通过EventThread就可以了。但是KK再次对这段逻辑进行细化和复杂化。Google真是费劲心思为了提升性能。

先来直接看下Surfaceflinger的onVSyncReceived函数：

	void SurfaceFlinger::onVSyncReceived(int type, nsecs_t timestamp) {
	    bool needsHwVsync = false;
	
	    { // Scope for the lock
	        Mutex::Autolock _l(mHWVsyncLock);
	        if (type == 0 && mPrimaryHWVsyncEnabled) {
	            needsHwVsync = mPrimaryDispSync.addResyncSample(timestamp);//mPromaryDisplays是什么？
	        }
	    }
	
	    if (needsHwVsync) {
	        enableHardwareVsync();//做了什么
	    } else {
	        disableHardwareVsync(false);//做了什么
	　　} 
	
	}
虽然很短，但是乍一看还是一头雾水

mPrimaryHWVsyncEnabled是什么时候被赋值的？

mPrimaryDispSync是什么，addResyncSample又做了什么？

enableHardwareVsync &disableHardwareVsync在干什么？

要解答这些问题，就从SurfaceFlinger::init开始看
	
	void SurfaceFlinger::init() {
	    ALOGI(  "SurfaceFlinger's main thread ready to run. "
	            "Initializing graphics H/W...");
	
	    status_t err;
	    Mutex::Autolock _l(mStateLock);
	
	    /* Set the mask bit of the sigset to block the SIGPIPE signal */
	    sigset_t sigMask;
	    sigemptyset (&sigMask);
	    sigaddset(&sigMask, SIGPIPE);
	    sigprocmask(SIG_BLOCK, &sigMask, NULL);
	
	    // initialize EGL for the default display
	    mEGLDisplay = eglGetDisplay(EGL_DEFAULT_DISPLAY);
	    eglInitialize(mEGLDisplay, NULL, NULL);
	
	    // Initialize the H/W composer object.  There may or may not be an
	    // actual hardware composer underneath.
	    mHwc = new HWComposer(this,
	            *static_cast<HWComposer::EventHandler *>(this));
	
	    // get a RenderEngine for the given display / config (can't fail)
	    mRenderEngine = RenderEngine::create(mEGLDisplay, mHwc->getVisualID());
	
	    // retrieve the EGL context that was selected/created
	    mEGLContext = mRenderEngine->getEGLContext();
	
	    LOG_ALWAYS_FATAL_IF(mEGLContext == EGL_NO_CONTEXT,
	            "couldn't create EGLContext");
	
	    // initialize our non-virtual displays
	    for (size_t i=0 ; i<DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES ; i++) {
	        DisplayDevice::DisplayType type((DisplayDevice::DisplayType)i);
	        // set-up the displays that are already connected
	        if (mHwc->isConnected(i) || type==DisplayDevice::DISPLAY_PRIMARY) {
	#ifdef QCOM_BSP
	            // query from hwc if the non-virtual display is secure.
	            bool isSecure = mHwc->isSecure(i);;
	#else
	            // All non-virtual displays are currently considered secure
	            bool isSecure = true;
	#endif
	            createBuiltinDisplayLocked(type, isSecure);
	            wp<IBinder> token = mBuiltinDisplays[i];
	
	            sp<IGraphicBufferProducer> producer;
	            sp<IGraphicBufferConsumer> consumer;
	            BufferQueue::createBufferQueue(&producer, &consumer,
	                    new GraphicBufferAlloc());
	
	            sp<FramebufferSurface> fbs = new FramebufferSurface(*mHwc, i,
	                    consumer);
	            int32_t hwcId = allocateHwcDisplayId(type);
	            sp<DisplayDevice> hw = new DisplayDevice(this,
	                    type, hwcId, mHwc->getFormat(hwcId), isSecure, token,
	                    fbs, producer,
	                    mRenderEngine->getEGLConfig());
	            if (i > DisplayDevice::DISPLAY_PRIMARY) {
	                // FIXME: currently we don't get blank/unblank requests
	                // for displays other than the main display, so we always
	                // assume a connected display is unblanked.
	                ALOGD("marking display %zu as acquired/unblanked", i);
	                hw->setPowerMode(HWC_POWER_MODE_NORMAL);
	            }
	            mDisplays.add(token, hw);
	        }
	    }
	
	    // make the GLContext current so that we can create textures when creating Layers
	    // (which may happens before we render something)
	    getDefaultDisplayDevice()->makeCurrent(mEGLDisplay, mEGLContext);
	
	    // start the EventThread
	    sp<VSyncSource> vsyncSrc = new DispSyncSource(&mPrimaryDispSync,
	            vsyncPhaseOffsetNs, true, "app");
	    mEventThread = new EventThread(vsyncSrc);
	    sp<VSyncSource> sfVsyncSrc = new DispSyncSource(&mPrimaryDispSync,
	            sfVsyncPhaseOffsetNs, true, "sf");
	    mSFEventThread = new EventThread(sfVsyncSrc);
	    mEventQueue.setEventThread(mSFEventThread);
	
	    mEventControlThread = new EventControlThread(this);
	    mEventControlThread->run("EventControl", PRIORITY_URGENT_DISPLAY);
	    android_set_rt_ioprio(mEventControlThread->getTid(), 1);
	
	    // set a fake vsync period if there is no HWComposer
	    if (mHwc->initCheck() != NO_ERROR) {
	        mPrimaryDispSync.setPeriod(16666667);
	    }
	
	    // initialize our drawing state
	    mDrawingState = mCurrentState;
	
	    // set initial conditions (e.g. unblank default device)
	    initializeDisplays();
	
	    // start boot animation
	    startBootAnim();
	}
有2个几乎一样的DispSyncSource，它的目的是提供了Vsync的虚拟化。关于这块的分析，可以参考Android 4.4(KitKat)中VSync信号的虚拟化

在三缓冲的框架下，对于一帧内容，先等APP UI画完了，SurfaceFlinger再出场整合到FrameBuffer

而现在google就是让它们一起跑起来。

然后搞了2个延时，这样就不会有问题。对应vsyncSrc（绘图延时） & sfVsyncSrc（合成延时）

#### 8.3.1 EventThread 

	bool EventThread::threadLoop() {
	    DisplayEventReceiver::Event event;
	    Vector< sp<EventThread::Connection> > signalConnections;
	    signalConnections = waitForEvent(&event);
	
	    // dispatch events to listeners...
	    const size_t count = signalConnections.size();
	    for (size_t i=0 ; i<count ; i++) {
	        const sp<Connection>& conn(signalConnections[i]);
	        // now see if we still need to report this event
	        status_t err = conn->postEvent(event);
	        if (err == -EAGAIN || err == -EWOULDBLOCK) {
	            // The destination doesn't accept events anymore, it's probably
	            // full. For now, we just drop the events on the floor.
	            // FIXME: Note that some events cannot be dropped and would have
	            // to be re-sent later.
	            // Right-now we don't have the ability to do this.
	            ALOGW("EventThread: dropping event (%08x) for connection %p",
	                    event.header.type, conn.get());
	        } else if (err < 0) {
	            // handle any other error on the pipe as fatal. the only
	            // reasonable thing to do is to clean-up this connection.
	            // The most common error we'll get here is -EPIPE.
	            removeDisplayEventConnection(signalConnections[i]);
	        }
	    }
	    return true;
	}
 EventThread会发送消息到surfaceflinger里面的MessageQueue。 
MessageQueue处理消息：
	
	int MessageQueue::eventReceiver(int /*fd*/, int /*events*/) {
	    ssize_t n;
	    DisplayEventReceiver::Event buffer[8];
	    while ((n = DisplayEventReceiver::getEvents(mEventTube, buffer, 8)) > 0) {
	        for (int i=0 ; i<n ; i++) {
	            if (buffer[i].header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
	#if INVALIDATE_ON_VSYNC
	                mHandler->dispatchInvalidate();
	#else
	                mHandler->dispatchRefresh();
	#endif
	                break;
	            }
	        }
	    }
	    return 1;
	}

如果Event的类型是

	DisplayEventReceiver::DISPLAY_EVENT_VSYNC
正是我们需要的类型，所以，就有2中处理方式：

UI进程需要先准备好数据（invalidate），然后Vsync信号来了以后，就开始刷新屏幕。

SurfaceFlinger是在Vsync来临之际准备数据然后刷新，还是平常就准备当VSync来临是再刷新。

先来看dispatchInvalidate,最后处理的地方就是这里。

        case MessageQueue::INVALIDATE: {
            bool refreshNeeded = handleMessageTransaction();
            refreshNeeded |= handleMessageInvalidate();
            refreshNeeded |= mRepaintEverything;
            if (refreshNeeded) {
                // Signal a refresh if a transaction modified the window state,
                // a new buffer was latched, or if HWC has requested a full
                // repaint
                signalRefresh();
            }
            break;
        }
我们来看下handleMessageRefresh：
	
	void SurfaceFlinger::handleMessageRefresh() {
	    ATRACE_CALL();
	    preComposition();　　//合成前的准备
	    rebuildLayerStacks();//重新建立layer堆栈
	    setUpHWComposer();//HWComposer的设定
	#ifdef QCOM_BSP
	    setUpTiledDr();
	#endif
	    doDebugFlashRegions();
	    doComposition();　　//正式合成工作
	    postComposition();　//合成的后期工作
	}


#### 8.3.2handleMessageTransaction 

 handleMessageTransaction在简单判断后直接调用handlerTransaction。

可以看到里面的handleTransactionLocked才是代码真正处理的地方。 
		
	void SurfaceFlinger::handleTransactionLocked(uint32_t transactionFlags)
	{
	    const LayerVector& currentLayers(mCurrentState.layersSortedByZ);
	    const size_t count = currentLayers.size();
	
	    /*
	     * Traversal of the children
	     * (perform the transaction for each of them if needed)
	     */
	
	    if (transactionFlags & eTraversalNeeded) {
	        for (size_t i=0 ; i<count ; i++) {
	            const sp<Layer>& layer(currentLayers[i]);
	            uint32_t trFlags = layer->getTransactionFlags(eTransactionNeeded);
	            if (!trFlags) continue;
	
	            const uint32_t flags = layer->doTransaction(0);
	            if (flags & Layer::eVisibleRegion)
	                mVisibleRegionsDirty = true;
	        }
	    }
	
	    /*
	     * Perform display own transactions if needed
	     */
	
	    if (transactionFlags & eDisplayTransactionNeeded) {
	        // here we take advantage of Vector's copy-on-write semantics to
	        // improve performance by skipping the transaction entirely when
	        // know that the lists are identical
	        const KeyedVector<  wp<IBinder>, DisplayDeviceState>& curr(mCurrentState.displays);
	        const KeyedVector<  wp<IBinder>, DisplayDeviceState>& draw(mDrawingState.displays);
	        if (!curr.isIdenticalTo(draw)) {
	            mVisibleRegionsDirty = true;
	            const size_t cc = curr.size();
	                  size_t dc = draw.size();
	
	            // find the displays that were removed
	            // (ie: in drawing state but not in current state)
	            // also handle displays that changed
	            // (ie: displays that are in both lists)
	            for (size_t i=0 ; i<dc ; i++) {
	                const ssize_t j = curr.indexOfKey(draw.keyAt(i));
	                if (j < 0) {
	                    // in drawing state but not in current state
	                    if (!draw[i].isMainDisplay()) {
	                        // Call makeCurrent() on the primary display so we can
	                        // be sure that nothing associated with this display
	                        // is current.
	                        const sp<const DisplayDevice> defaultDisplay(getDefaultDisplayDevice());
	                        defaultDisplay->makeCurrent(mEGLDisplay, mEGLContext);
	                        sp<DisplayDevice> hw(getDisplayDevice(draw.keyAt(i)));
	                        if (hw != NULL)
	                            hw->disconnect(getHwComposer());
	                        if (draw[i].type < DisplayDevice::NUM_BUILTIN_DISPLAY_TYPES)
	                            mEventThread->onHotplugReceived(draw[i].type, false);
	                        mDisplays.removeItem(draw.keyAt(i));
	                    } else {
	                        ALOGW("trying to remove the main display");
	                    }
	                } else {
	                    // this display is in both lists. see if something changed.
	                    const DisplayDeviceState& state(curr[j]);
	                    const wp<IBinder>& display(curr.keyAt(j));
	                    if (state.surface->asBinder() != draw[i].surface->asBinder()) {
	                        // changing the surface is like destroying and
	                        // recreating the DisplayDevice, so we just remove it
	                        // from the drawing state, so that it get re-added
	                        // below.
	                        sp<DisplayDevice> hw(getDisplayDevice(display));
	                        if (hw != NULL)
	                            hw->disconnect(getHwComposer());
	                        mDisplays.removeItem(display);
	                        mDrawingState.displays.removeItemsAt(i);
	                        dc--; i--;
	                        // at this point we must loop to the next item
	                        continue;
	                    }
	
	                    const sp<DisplayDevice> disp(getDisplayDevice(display));
	                    if (disp != NULL) {
	                        if (state.layerStack != draw[i].layerStack) {
	                            disp->setLayerStack(state.layerStack);
	                        }
	                        if ((state.orientation != draw[i].orientation)
	                                || (state.viewport != draw[i].viewport)
	                                || (state.frame != draw[i].frame))
	                        {
	#ifdef QCOM_BSP
	                            int orient = state.orientation;
	                            // Honor the orientation change after boot
	                            // animation completes and make sure boot
	                            // animation is shown in panel orientation always.
	                            if(mBootFinished){
	                                disp->setProjection(state.orientation,
	                                        state.viewport, state.frame);
	                                orient = state.orientation;
	                            }
	                            else{
	                                char property[PROPERTY_VALUE_MAX];
	                                int panelOrientation =
	                                        DisplayState::eOrientationDefault;
	                                if(property_get("persist.panel.orientation",
	                                            property, "0") > 0){
	                                    panelOrientation = atoi(property) / 90;
	                                }
	                                disp->setProjection(panelOrientation,
	                                        state.viewport, state.frame);
	                                orient = panelOrientation;
	                            }
	                            // Set the view frame of each display only of its
	                            // default orientation.
	                            if(orient == DisplayState::eOrientationDefault and
	                                    state.frame.isValid()) {
	                                qdutils::setViewFrame(disp->getHwcDisplayId(),
	                                    state.frame.left, state.frame.top,
	                                    state.frame.right, state.frame.bottom);
	                            }
	#else
	                            disp->setProjection(state.orientation,
	                                state.viewport, state.frame);
	#endif
	                        }
	                        if (state.width != draw[i].width || state.height != draw[i].height) {
	                            disp->setDisplaySize(state.width, state.height);
	                        }
	                    }
	                }
	            }
	
	            // find displays that were added
	            // (ie: in current state but not in drawing state)
	            for (size_t i=0 ; i<cc ; i++) {
	                if (draw.indexOfKey(curr.keyAt(i)) < 0) {
	                    const DisplayDeviceState& state(curr[i]);
	
	                    sp<DisplaySurface> dispSurface;
	                    sp<IGraphicBufferProducer> producer;
	                    sp<IGraphicBufferProducer> bqProducer;
	                    sp<IGraphicBufferConsumer> bqConsumer;
	                    BufferQueue::createBufferQueue(&bqProducer, &bqConsumer,
	                            new GraphicBufferAlloc());
	
	                    int32_t hwcDisplayId = -1;
	                    if (state.isVirtualDisplay()) {
	                        // Virtual displays without a surface are dormant:
	                        // they have external state (layer stack, projection,
	                        // etc.) but no internal state (i.e. a DisplayDevice).
	                        if (state.surface != NULL) {
	                            configureVirtualDisplay(hwcDisplayId,
	                                    dispSurface, producer, state, bqProducer,
	                                    bqConsumer);
	                        }
	                    } else {
	                        ALOGE_IF(state.surface!=NULL,
	                                "adding a supported display, but rendering "
	                                "surface is provided (%p), ignoring it",
	                                state.surface.get());
	                        hwcDisplayId = allocateHwcDisplayId(state.type);
	                        // for supported (by hwc) displays we provide our
	                        // own rendering surface
	                        dispSurface = new FramebufferSurface(*mHwc, state.type,
	                                bqConsumer);
	                        producer = bqProducer;
	                    }
	
	                    const wp<IBinder>& display(curr.keyAt(i));
	                    if (dispSurface != NULL && producer != NULL) {
	                        sp<DisplayDevice> hw = new DisplayDevice(this,
	                                state.type, hwcDisplayId,
	                                mHwc->getFormat(hwcDisplayId), state.isSecure,
	                                display, dispSurface, producer,
	                                mRenderEngine->getEGLConfig());
	                        hw->setLayerStack(state.layerStack);
	                        hw->setProjection(state.orientation,
	                                state.viewport, state.frame);
	                        hw->setDisplayName(state.displayName);
	                        // When a new display device is added update the active
	                        // config by querying HWC otherwise the default config
	                        // (config 0) will be used.
	                        int activeConfig = mHwc->getActiveConfig(hwcDisplayId);
	                        if (activeConfig >= 0) {
	                            hw->setActiveConfig(activeConfig);
	                        }
	                        mDisplays.add(display, hw);
	                        if (state.isVirtualDisplay()) {
	                            if (hwcDisplayId >= 0) {
	                                mHwc->setVirtualDisplayProperties(hwcDisplayId,
	                                        hw->getWidth(), hw->getHeight(),
	                                        hw->getFormat());
	                            }
	                        } else {
	                            mEventThread->onHotplugReceived(state.type, true);
	                        }
	                    }
	                }
	            }
	        }
	    }
	
	    if (transactionFlags & (eTraversalNeeded|eDisplayTransactionNeeded)) {
	        // The transform hint might have changed for some layers
	        // (either because a display has changed, or because a layer
	        // as changed).
	        //
	        // Walk through all the layers in currentLayers,
	        // and update their transform hint.
	        //
	        // If a layer is visible only on a single display, then that
	        // display is used to calculate the hint, otherwise we use the
	        // default display.
	        //
	        // NOTE: we do this here, rather than in rebuildLayerStacks() so that
	        // the hint is set before we acquire a buffer from the surface texture.
	        //
	        // NOTE: layer transactions have taken place already, so we use their
	        // drawing state. However, SurfaceFlinger's own transaction has not
	        // happened yet, so we must use the current state layer list
	        // (soon to become the drawing state list).
	        //
	        sp<const DisplayDevice> disp;
	        uint32_t currentlayerStack = 0;
	        for (size_t i=0; i<count; i++) {
	            // NOTE: we rely on the fact that layers are sorted by
	            // layerStack first (so we don't have to traverse the list
	            // of displays for every layer).
	            const sp<Layer>& layer(currentLayers[i]);
	            uint32_t layerStack = layer->getDrawingState().layerStack;
	            if (i==0 || currentlayerStack != layerStack) {
	                currentlayerStack = layerStack;
	                // figure out if this layerstack is mirrored
	                // (more than one display) if so, pick the default display,
	                // if not, pick the only display it's on.
	                disp.clear();
	                for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
	                    sp<const DisplayDevice> hw(mDisplays[dpy]);
	                    if (hw->getLayerStack() == currentlayerStack) {
	                        if (disp == NULL) {
	                            disp = hw;
	                        } else {
	                            disp = NULL;
	                            break;
	                        }
	                    }
	                }
	            }
	            if (disp == NULL) {
	                // NOTE: TEMPORARY FIX ONLY. Real fix should cause layers to
	                // redraw after transform hint changes. See bug 8508397.
	
	                // could be null when this layer is using a layerStack
	                // that is not visible on any display. Also can occur at
	                // screen off/on times.
	                disp = getDefaultDisplayDevice();
	            }
	            layer->updateTransformHint(disp);
	        }
	    }
	
	
	    /*
	     * Perform our own transaction if needed
	     */
	
	    const LayerVector& layers(mDrawingState.layersSortedByZ);
	    if (currentLayers.size() > layers.size()) {
	        // layers have been added
	        mVisibleRegionsDirty = true;
	    }
	
	    // some layers might have been removed, so
	    // we need to update the regions they're exposing.
	    if (mLayersRemoved) {
	        mLayersRemoved = false;
	        mVisibleRegionsDirty = true;
	        const size_t count = layers.size();
	        for (size_t i=0 ; i<count ; i++) {
	            const sp<Layer>& layer(layers[i]);
	            if (currentLayers.indexOf(layer) < 0) {
	                // this layer is not visible anymore
	                // TODO: we could traverse the tree from front to back and
	                //       compute the actual visible region
	                // TODO: we could cache the transformed region
	                const Layer::State& s(layer->getDrawingState());
	                Region visibleReg = s.transform.transform(
	                        Region(Rect(s.active.w, s.active.h)));
	                invalidateLayerStack(s.layerStack, visibleReg);
	            }
	        }
	    }
	
	    commitTransaction();
	
	    updateCursorAsync();
	}
这里处理3中情况，过程类似，我们只看eTraversalNeeded这种情况。

	uint32_t trFlags = layer->getTransactionFlags(eTransactionNeeded);

获取各个layer的标志位，然后做const uint32_t flags = layer->doTransaction(0);的操作各layer计算各自的可见区域，mVisibleRegionsDirty记录可见区域变化。
以下代码：

	mCurrentState.layersSortedByZ

surfaceFlinger有2个记录layer变化的全局变量
	
	State mDrawingState;
	State mCurrentState;
	
一个记录上一次的状态，后者记录当前的状态，这样就可以判断layer的变化状态。layersSortedByZ 可见，layer是通过Z-order排列的。
这个变量记录了所有的layer。   
首先判断size是否有修改,然后

	mSurfaceFlingerConsumer->setDefaultBufferSize

重新获取大小。

	if (c.sequence != s.sequence) {
        // invalidate and recompute the visible regions if needed
        flags |= eVisibleRegion;

Sequence是个什么东西？

当Layer的position，Zorder，alpha,matrix,transparent region,flags,crops.等发生变化的时候，sequence就会自增。
也就是，当这些属性发生变化是，页面在Vsync信号触发的时候，根据sequence来判断是否需要属性页面。
新增layer，对比2个state中的layer队列，就可以知道新增的layer。
移除layer，也是比较2个layer队列，就可以找到移除的layer。提交transaction，主要就是同步2个state。然后currentstate继续跟踪layer变化，如此往复。

Vsync 是SurfaceFlinger模块最核心的概念，所以这块将会分多次讲解。

## 9.Vsync第二部分  

在上一篇中我们讲到，视图的刷新需要很多步骤，
	
	void SurfaceFlinger::handleMessageRefresh() {
	    ATRACE_CALL();
	    preComposition();　　//合成前的准备
	    rebuildLayerStacks();//重新建立layer堆栈
	    setUpHWComposer();//HWComposer的设定
	#ifdef QCOM_BSP
	    setUpTiledDr();
	#endif
	    doDebugFlashRegions();
	    doComposition();　　//正式合成工作
	    postComposition();　//合成的后期工作
	}

本文将继续分析这些过程。

### 9.1 handlerMessageInvalidate  

invalidate 字面意思就是使无效，更进一步就是当前的buffer已经无效，请刷新界面。

啥也没干，buffer已经无效，我换下一个，就是handlePageFlip

	void SurfaceFlinger::handleMessageInvalidate() {
	    ATRACE_CALL();
	    handlePageFlip();
	}
再来看这个函数：handlePageFlip
	
	bool SurfaceFlinger::handlePageFlip()
	{
	    Region dirtyRegion;
	
	    bool visibleRegions = false;
	    const LayerVector& layers(mDrawingState.layersSortedByZ);
	    bool frameQueued = false;
	
	    // Store the set of layers that need updates. This set must not change as
	    // buffers are being latched, as this could result in a deadlock.
	    // Example: Two producers share the same command stream and:
	    // 1.) Layer 0 is latched
	    // 2.) Layer 0 gets a new frame
	    // 2.) Layer 1 gets a new frame
	    // 3.) Layer 1 is latched.
	    // Display is now waiting on Layer 1's frame, which is behind layer 0's
	    // second frame. But layer 0's second frame could be waiting on display.
	    Vector<Layer*> layersWithQueuedFrames;
	    for (size_t i = 0, count = layers.size(); i<count ; i++) {
	        const sp<Layer>& layer(layers[i]);
	        if (layer->hasQueuedFrame()) {
	            frameQueued = true;
	            if (layer->shouldPresentNow(mPrimaryDispSync)) {
	                layersWithQueuedFrames.push_back(layer.get());
	            }
	        }
	    }
	    for (size_t i = 0, count = layersWithQueuedFrames.size() ; i<count ; i++) {
	        Layer* layer = layersWithQueuedFrames[i];
	        const Region dirty(layer->latchBuffer(visibleRegions));
	        const Layer::State& s(layer->getDrawingState());
	        invalidateLayerStack(s.layerStack, dirty);
	    }
	
	    mVisibleRegionsDirty |= visibleRegions;
	
	    // If we will need to wake up at some time in the future to deal with a
	    // queued frame that shouldn't be displayed during this vsync period, wake
	    // up during the next vsync period to check again.
	    if (frameQueued && layersWithQueuedFrames.empty()) {
	        signalLayerUpdate();
	    }
	
	    // Only continue with the refresh if there is actually new work to do
	    return !layersWithQueuedFrames.empty();
	}
	
@step1:layer->latchBuffer(visibleRegions) 通过该函数锁定各layer的缓冲区。可以理解这个函数一定与BufferQueue有关。

	status_t updateResult = mSurfaceFlingerConsumer->updateTexImage(&r,
                mFlinger->mPrimaryDispSync);
上面是latchBuffer的核心语句。SurfaceFlingerConsumer前文已经提过，是client端操作bufferqueue的一个端口。所以这个函数一定是操作bufferqueue的。


	// Acquire the next buffer.
    // In asynchronous mode the list is guaranteed to be one buffer
    // deep, while in synchronous mode we use the oldest buffer.
    err = acquireBufferLocked(&item, computeExpectedPresent(dispSync));
    if (err != NO_ERROR) {
        if (err == BufferQueue::NO_BUFFER_AVAILABLE) {
            err = NO_ERROR;
        } else if (err == BufferQueue::PRESENT_LATER) {
            // return the error, without logging
        } else {
            ALOGE("updateTexImage: acquire failed: %s (%d)",
                strerror(-err), err);
        }
        return err;
    }


    // We call the rejecter here, in case the caller has a reason to
    // not accept this buffer.  This is used by SurfaceFlinger to
    // reject buffers which have the wrong size
    int buf = item.mBuf;
    if (rejecter && rejecter->reject(mSlots[buf].mGraphicBuffer, item)) {
        releaseBufferLocked(buf, mSlots[buf].mGraphicBuffer, EGL_NO_SYNC_KHR);
        return NO_ERROR;
    }

    // Release the previous buffer.
    err = updateAndReleaseLocked(item);
    if (err != NO_ERROR) {
        return err;
    }

看了注释，基本解释了大体的过程。

1)  请求新的buffer

2）通过rejecter来判断申请的buffer是否满足surfaceflinger的要求。

3）释放之前的buffer

具体流程可以参考如下：

![](http://i.imgur.com/e5N6ZPl.png)

@step2：SurfaceFlinger:invalidateLayerStack来更新各个区域。

### 9.2 preComposition 合成前的准备 

 首先来看2个Vsync Rate相关的代码：

	virtual void setVsyncRate(uint32_t count)
	virtual void requestNextVsync() 
当count为1时，表示每个信号都要报告，当count =2 时，表示信号 一个间隔一个报告，当count =0时，表示不自动报告，除非主动触发requestNextVsync

	void SurfaceFlinger::preComposition()
	{
	    bool needExtraInvalidate = false;
	    const LayerVector& layers(mDrawingState.layersSortedByZ);
	    const size_t count = layers.size();
	    for (size_t i=0 ; i<count ; i++) {
	        if (layers[i]->onPreComposition()) {
	            needExtraInvalidate = true;
	        }
	    }
	    if (needExtraInvalidate) {
	        signalLayerUpdate();
	    }
	}
代码很简单，其实一共就3步，

1）获取全部的layer

2）每个layer onPrecomposition

3) layer update

	bool Layer::onPreComposition() {
	    mRefreshPending = false;
	    return mQueuedFrames > 0 || mSidebandStreamChanged;
	}
也就是说，当layer里存放被queue的frame以后，就会出发layer update.

	void SurfaceFlinger::signalLayerUpdate() {
	    mEventQueue.invalidate();
	}
最终会调用：

	void EventThread::Connection::requestNextVsync() {
	    mEventThread->requestNextVsync(this);
	}

	void EventThread::requestNextVsync(
	        const sp<EventThread::Connection>& connection) {
	    Mutex::Autolock _l(mLock);
	    if (connection->count < 0) {
	        connection->count = 0;
	        mCondition.broadcast();//通知对vsync感兴趣的类
	    }
	}

那么谁在等待这个broadcast呢？
还是EventThread

	void EventThread::onVSyncEvent(nsecs_t timestamp) {
	    Mutex::Autolock _l(mLock);
	    mVSyncEvent[0].header.type = DisplayEventReceiver::DISPLAY_EVENT_VSYNC;
	    mVSyncEvent[0].header.id = 0;
	    mVSyncEvent[0].header.timestamp = timestamp;
	    mVSyncEvent[0].vsync.count++;
	    mCondition.broadcast();
	}


### 9.3可见区域rebuildlayerStack 

	void SurfaceFlinger::rebuildLayerStacks() {
	#ifdef QCOM_BSP
	    char prop[PROPERTY_VALUE_MAX];
	    property_get("sys.extended_mode", prop, "0");
	    sExtendedMode = atoi(prop) ? true : false;
	#endif
	    // rebuild the visible layer list per screen
	    if (CC_UNLIKELY(mVisibleRegionsDirty)) {
	        ATRACE_CALL();
	        mVisibleRegionsDirty = false;
	        invalidateHwcGeometry();
	
	        const LayerVector& layers(mDrawingState.layersSortedByZ);
	        for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
	            Region opaqueRegion;
	            Region dirtyRegion;
	            Vector< sp<Layer> > layersSortedByZ;
	            const sp<DisplayDevice>& hw(mDisplays[dpy]);
	            const Transform& tr(hw->getTransform());
	            const Rect bounds(hw->getBounds());
	            int dpyId = hw->getHwcDisplayId();
	            if (hw->isDisplayOn()) {
	                SurfaceFlinger::computeVisibleRegions(dpyId, layers,
	                        hw->getLayerStack(), dirtyRegion, opaqueRegion);
	
	                const size_t count = layers.size();
	                for (size_t i=0 ; i<count ; i++) {
	                    const sp<Layer>& layer(layers[i]);
	                    const Layer::State& s(layer->getDrawingState());
	                    Region drawRegion(tr.transform(
	                            layer->visibleNonTransparentRegion));
	                    drawRegion.andSelf(bounds);
	                    if (!drawRegion.isEmpty()) {
	                        layersSortedByZ.add(layer);
	                    }
	                }
	            }
	            hw->setVisibleLayersSortedByZ(layersSortedByZ);
	            hw->undefinedRegion.set(bounds);
	            hw->undefinedRegion.subtractSelf(tr.transform(opaqueRegion));
	            hw->dirtyRegion.orSelf(dirtyRegion);
	        }
	    }
	}

前文提到mVisibleRegionsDirty这个变量是标记要刷新的可见区域的，我们按字面意思解释下：脏的可见区域，顾名思义，这就是要刷新的区域，因为buffer已经“脏”了。

@step1：系统的display可能不止一个，存在与mDisplays中。

@step2：computeVisibleRegions这个函数根据所有的layer状态，得到2个重要的变量。opaqueRegion & dirtyRegion

dirtyRegion是需要被刷新的。 opaqueRegion 不透明区域，应为layer是按Z-order排序的，左右排在前面的opaqueRegion 会挡住后面的layer。

@step3：Region drawRegion(tr.transform( layer->visibleNonTransparentRegion));程序需要进一步得出每个layer 绘制的区域。

系统的display（显示器）可能不止一个，但是所有的layer都记录在layersSortedByZ里面。记录每个layer属于那个display的变量是 hw->getLayerStack()

@step4：将结果保存到hw中。

这里的不透明区域 是很有意义的一个概念，就是我们在Z-order 上，越靠近用户的时候，值越大，所以是递减操作。

### 9.4 setUpHWComposer 搭建环境 

用于合成surface的功能模块可以有2个，OpenGL ES & HWC，它的管理实在HWC里面实现的。

setUpHWComposer 总的来说就干了3件事情。

1）构造Worklist，并且给DisplayData:list 申请空间

2）填充各layer数据

3）报告HWC（有其他地方决定使用那个）
 
### 9.5 doCompostion  

关键地方来了，上面的setUpHWComposer 只是交给HWC来负责显示，真正显示的地方就在这里。

1）如何合成

2）如何显示到屏幕上
 
![](http://i.imgur.com/BBsonhs.png)

合成的流程大体如上图。

有了概念后，分析源码：
	
	void SurfaceFlinger::doComposition() {
	    ATRACE_CALL();
	    const bool repaintEverything = android_atomic_and(0, &mRepaintEverything);
	    for (size_t dpy=0 ; dpy<mDisplays.size() ; dpy++) {
	        const sp<DisplayDevice>& hw(mDisplays[dpy]);
	        if (hw->isDisplayOn()) {
	            // transform the dirty region into this screen's coordinate space
	            const Region dirtyRegion(hw->getDirtyRegion(repaintEverything));
	
	            // repaint the framebuffer (if needed)
	            doDisplayComposition(hw, dirtyRegion);
	
	            ++mActiveFrameSequence;
	
	            hw->dirtyRegion.clear();
	            hw->flip(hw->swapRegion);
	            hw->swapRegion.clear();
	        }
	        // inform the h/w that we're done compositing
	        hw->compositionComplete();
	    }
	    postFramebuffer();
	}

变量mRepaintEverything用于说明，是否需要全部重绘所有内容。如果为true的化，那么dirtyRegion就是displaydevice的 width & height构成的RECT。

否则由dirtyRegion转换而来。

doDisplayComposition是每个Display来处理，有可能会用到OpenGL 的接口来交换buffer。

hw->compositionComplete(); 通知HWC合成结束了。

postFramebuffer HWC的Set接口调用。


#### 9.5.1 doDisplayComposition 

	void SurfaceFlinger::doDisplayComposition(const sp<const DisplayDevice>& hw,
	        const Region& inDirtyRegion)
	{
	    // We only need to actually compose the display if:
	    // 1) It is being handled by hardware composer, which may need this to
	    //    keep its virtual display state machine in sync, or
	    // 2) There is work to be done (the dirty region isn't empty)
	    bool isHwcDisplay = hw->getHwcDisplayId() >= 0;
	    if (!isHwcDisplay && inDirtyRegion.isEmpty()) {
	        return;
	    }
	
	    Region dirtyRegion(inDirtyRegion);
	
	    // compute the invalid region
	    hw->swapRegion.orSelf(dirtyRegion);
	
	    uint32_t flags = hw->getFlags();
	    if (flags & DisplayDevice::SWAP_RECTANGLE) {
	        // we can redraw only what's dirty, but since SWAP_RECTANGLE only
	        // takes a rectangle, we must make sure to update that whole
	        // rectangle in that case
	        dirtyRegion.set(hw->swapRegion.bounds());
	    } else {
	        if (flags & DisplayDevice::PARTIAL_UPDATES) {
	            // We need to redraw the rectangle that will be updated
	            // (pushed to the framebuffer).
	            // This is needed because PARTIAL_UPDATES only takes one
	            // rectangle instead of a region (see DisplayDevice::flip())
	            dirtyRegion.set(hw->swapRegion.bounds());
	        } else {
	            // we need to redraw everything (the whole screen)
	            dirtyRegion.set(hw->bounds());
	            hw->swapRegion = dirtyRegion;
	        }
	    }
	
	    if (CC_LIKELY(!mDaltonize && !mHasColorMatrix)) {
	        if (!doComposeSurfaces(hw, dirtyRegion)) return;
	    } else {
	        RenderEngine& engine(getRenderEngine());
	        mat4 colorMatrix = mColorMatrix;
	        if (mDaltonize) {
	            colorMatrix = colorMatrix * mDaltonizer();
	        }
	        mat4 oldMatrix = engine.setupColorTransform(colorMatrix);
	        doComposeSurfaces(hw, dirtyRegion);
	        engine.setupColorTransform(oldMatrix);
	    }
	
	    // update the swap region and clear the dirty region
	    hw->swapRegion.orSelf(dirtyRegion);
	
	    // swap buffers (presentation)
	    hw->swapBuffers(getHwComposer());
	}

传入的参数inDirtyRegion，这就是要刷新的“脏”区域，but，我们的刷新机制，决定了必须是矩形的区域。

So，需要一个最小的矩形，能够包裹inDirtyRegion的区域。

SWAP_RECTANGLE：系统支持软件层面的部分刷新，就需要计算这个最小矩形。

PARTIAL_UPDATES：硬件层面的部分刷新，同理需要这个最小矩形。

最后就是重绘这个区域。 
		
	bool SurfaceFlinger::doComposeSurfaces(const sp<const DisplayDevice>& hw, const Region& dirty)
	{
	    RenderEngine& engine(getRenderEngine());
	    const int32_t id = hw->getHwcDisplayId();
	    HWComposer& hwc(getHwComposer());
	    HWComposer::LayerListIterator cur = hwc.begin(id);
	    const HWComposer::LayerListIterator end = hwc.end(id);
	
	    Region clearRegion;
	    bool hasGlesComposition = hwc.hasGlesComposition(id);
	    const bool hasHwcComposition = hwc.hasHwcComposition(id);
	    if (hasGlesComposition) {
	        if (!hw->makeCurrent(mEGLDisplay, mEGLContext)) {
	            ALOGW("DisplayDevice::makeCurrent failed. Aborting surface composition for display %s",
	                  hw->getDisplayName().string());
	            eglMakeCurrent(mEGLDisplay, EGL_NO_SURFACE, EGL_NO_SURFACE, EGL_NO_CONTEXT);
	            if(!getDefaultDisplayDevice()->makeCurrent(mEGLDisplay, mEGLContext)) {
	              ALOGE("DisplayDevice::makeCurrent on default display failed. Aborting.");
	            }
	            return false;
	        }
	
	        // Never touch the framebuffer if we don't have any framebuffer layers
	        if (hasHwcComposition) {
	            // when using overlays, we assume a fully transparent framebuffer
	            // NOTE: we could reduce how much we need to clear, for instance
	            // remove where there are opaque FB layers. however, on some
	            // GPUs doing a "clean slate" clear might be more efficient.
	            // We'll revisit later if needed.
	            if(!(mGpuTileRenderEnable && (mDisplays.size()==1)))
	                engine.clearWithColor(0, 0, 0, 0);
	        } else {
	            // we start with the whole screen area
	            const Region bounds(hw->getBounds());
	
	            // we remove the scissor part
	            // we're left with the letterbox region
	            // (common case is that letterbox ends-up being empty)
	            const Region letterbox(bounds.subtract(hw->getScissor()));
	
	            // compute the area to clear
	            Region region(hw->undefinedRegion.merge(letterbox));
	
	            // but limit it to the dirty region
	            region.andSelf(dirty);
	
	
	            // screen is already cleared here
	#ifdef QCOM_BSP
	            clearRegion.clear();
	            if(mGpuTileRenderEnable && (mDisplays.size()==1)) {
	                clearRegion = region;
	                if (cur == end) {
	                    drawWormhole(hw, region);
	                } else if(mCanUseGpuTileRender) {
	                   /* If GPUTileRect DR optimization on clear only the UnionDR
	                    * (computed by computeTiledDr) which is the actual region
	                    * that will be drawn on FB in this cycle.. */
	                    clearRegion = clearRegion.andSelf(Region(mUnionDirtyRect));
	                }
	            } else
	#endif
	            {
	                if (!region.isEmpty()) {
	                    if (cur != end) {
	                        if (cur->getCompositionType() != HWC_BLIT)
	                            // can happen with SurfaceView
	                            drawWormhole(hw, region);
	                    } else
	                        drawWormhole(hw, region);
	                }
	            }
	        }
	
	        if (hw->getDisplayType() != DisplayDevice::DISPLAY_PRIMARY) {
	            // just to be on the safe side, we don't set the
	            // scissor on the main display. It should never be needed
	            // anyways (though in theory it could since the API allows it).
	            const Rect& bounds(hw->getBounds());
	            const Rect& scissor(hw->getScissor());
	            if (scissor != bounds) {
	                // scissor doesn't match the screen's dimensions, so we
	                // need to clear everything outside of it and enable
	                // the GL scissor so we don't draw anything where we shouldn't
	
	                // enable scissor for this frame
	                const uint32_t height = hw->getHeight();
	                engine.setScissor(scissor.left, height - scissor.bottom,
	                        scissor.getWidth(), scissor.getHeight());
	            }
	        }
	    }
	
	    /*
	     * and then, render the layers targeted at the framebuffer
	     */
	
	    const Vector< sp<Layer> >& layers(hw->getVisibleLayersSortedByZ());
	    const size_t count = layers.size();
	    const Transform& tr = hw->getTransform();
	    if (cur != end) {
	        // we're using h/w composer
	#ifdef QCOM_BSP
	        int fbWidth= hw->getWidth();
	        int fbHeight= hw->getHeight();
	        /* if GPUTileRender optimization property is on & can be used
	         * i) Enable EGL_SWAP_PRESERVED flag
	         * ii) do startTile with union DirtyRect
	         * else , Disable EGL_SWAP_PRESERVED */
	        if(mGpuTileRenderEnable && (mDisplays.size()==1)) {
	            if(mCanUseGpuTileRender && !mUnionDirtyRect.isEmpty()) {
	                hw->eglSwapPreserved(true);
	                Rect dr = mUnionDirtyRect;
	                engine.startTileComposition(dr.left, (fbHeight-dr.bottom),
	                      (dr.right-dr.left),
	                      (dr.bottom-dr.top), 0);
	            } else {
	                // Un Set EGL_SWAP_PRESERVED flag, if no tiling required.
	                hw->eglSwapPreserved(false);
	            }
	            // DrawWormHole/Any Draw has to be within startTile & EndTile
	            if (hasGlesComposition) {
	                if (hasHwcComposition) {
	                    if(mCanUseGpuTileRender && !mUnionDirtyRect.isEmpty()) {
	                        const Rect& scissor(mUnionDirtyRect);
	                        engine.setScissor(scissor.left,
	                              hw->getHeight()- scissor.bottom,
	                              scissor.getWidth(), scissor.getHeight());
	                        engine.clearWithColor(0, 0, 0, 0);
	                        engine.disableScissor();
	                    } else {
	                        engine.clearWithColor(0, 0, 0, 0);
	                    }
	                } else {
	                    if (cur->getCompositionType() != HWC_BLIT &&
	                          !clearRegion.isEmpty()) {
	                        drawWormhole(hw, clearRegion);
	                    }
	                }
	            }
	        }
	#endif
	
	        for (size_t i=0 ; i<count && cur!=end ; ++i, ++cur) {
	            const sp<Layer>& layer(layers[i]);
	            const Region clip(dirty.intersect(tr.transform(layer->visibleRegion)));
	            if (!clip.isEmpty()) {
	                switch (cur->getCompositionType()) {
	                    case HWC_CURSOR_OVERLAY:
	                    case HWC_OVERLAY: {
	                        const Layer::State& state(layer->getDrawingState());
	                        if ((cur->getHints() & HWC_HINT_CLEAR_FB)
	                                && i
	                                && layer->isOpaque(state) && (state.alpha == 0xFF)
	                                && hasGlesComposition) {
	                            // never clear the very first layer since we're
	                            // guaranteed the FB is already cleared
	                            layer->clearWithOpenGL(hw, clip);
	                        }
	                        break;
	                    }
	                    case HWC_FRAMEBUFFER: {
	                        layer->draw(hw, clip);
	                        break;
	                    }
	                    case HWC_BLIT:
	                        //Do nothing
	                        break;
	                    case HWC_FRAMEBUFFER_TARGET: {
	                        // this should not happen as the iterator shouldn't
	                        // let us get there.
	                        ALOGW("HWC_FRAMEBUFFER_TARGET found in hwc list (index=%zu)", i);
	                        break;
	                    }
	                }
	            }
	            layer->setAcquireFence(hw, *cur);
	        }
	
	#ifdef QCOM_BSP
	        // call EndTile, if starTile has been called in this cycle.
	        if(mGpuTileRenderEnable && (mDisplays.size()==1)) {
	            if(mCanUseGpuTileRender && !mUnionDirtyRect.isEmpty()) {
	                engine.endTileComposition(GL_PRESERVE);
	            }
	        }
	#endif
	    } else {
	        // we're not using h/w composer
	        for (size_t i=0 ; i<count ; ++i) {
	            const sp<Layer>& layer(layers[i]);
	            const Region clip(dirty.intersect(
	                    tr.transform(layer->visibleRegion)));
	            if (!clip.isEmpty()) {
	                layer->draw(hw, clip);
	            }
	        }
	    }
	
	    // disable scissor at the end of the frame
	    engine.disableScissor();
	    return true;
	}

依次分析：hasGlesComposition需要Open GL来合成的layer，hasHwcComposition需要HWC来合成的layer。

这2各变量不是互斥的，有同时存在需要Open GL layer & HWC layer。

hasHwcComposition在2种情况下是true。

1）layer的类型是HWC_Framebuffer的时候，通常情况下是true。

2)cur ==end 核心实现layer->draw 来完成。

3）cur!=end, 将有hwc来实现。
	
	{
	    ATRACE_CALL();
	
	    if (CC_UNLIKELY(mActiveBuffer == 0)) {
	        // the texture has not been created yet, this Layer has
	        // in fact never been drawn into. This happens frequently with
	        // SurfaceView because the WindowManager can't know when the client
	        // has drawn the first time.
	
	        // If there is nothing under us, we paint the screen in black, otherwise
	        // we just skip this update.
	
	        // figure out if there is something below us
	        Region under;
	        const SurfaceFlinger::LayerVector& drawingLayers(
	                mFlinger->mDrawingState.layersSortedByZ);
	        const size_t count = drawingLayers.size();
	        for (size_t i=0 ; i<count ; ++i) {
	            const sp<Layer>& layer(drawingLayers[i]);
	            if (layer.get() == static_cast<Layer const*>(this))
	                break;
	            under.orSelf( hw->getTransform().transform(layer->visibleRegion) );
	        }
	        // if not everything below us is covered, we plug the holes!
	        Region holes(clip.subtract(under));
	        if (!holes.isEmpty()) {
	            clearWithOpenGL(hw, holes, 0, 0, 0, 1);
	        }
	        return;
	    }
	
	    // Bind the current buffer to the GL texture, and wait for it to be
	    // ready for us to draw into.
	    status_t err = mSurfaceFlingerConsumer->bindTextureImage();
	    if (err != NO_ERROR) {
	        ALOGW("onDraw: bindTextureImage failed (err=%d)", err);
	        // Go ahead and draw the buffer anyway; no matter what we do the screen
	        // is probably going to have something visibly wrong.
	    }
	
	    bool blackOutLayer = isProtected() || (isSecure() && !hw->isSecure());
	
	    RenderEngine& engine(mFlinger->getRenderEngine());
	
	    if (!blackOutLayer) {
	        // TODO: we could be more subtle with isFixedSize()
	        const bool useFiltering = getFiltering() || needsFiltering(hw) || isFixedSize();
	
	        // Query the texture matrix given our current filtering mode.
	        float textureMatrix[16];
	        mSurfaceFlingerConsumer->setFilteringEnabled(useFiltering);
	        mSurfaceFlingerConsumer->getTransformMatrix(textureMatrix);
	
	        if (mSurfaceFlingerConsumer->getTransformToDisplayInverse()) {
	
	            /*
	             * the code below applies the display's inverse transform to the texture transform
	             */
	
	            // create a 4x4 transform matrix from the display transform flags
	            const mat4 flipH(-1,0,0,0,  0,1,0,0, 0,0,1,0, 1,0,0,1);
	            const mat4 flipV( 1,0,0,0, 0,-1,0,0, 0,0,1,0, 0,1,0,1);
	            const mat4 rot90( 0,1,0,0, -1,0,0,0, 0,0,1,0, 1,0,0,1);
	
	            mat4 tr;
	            uint32_t transform = hw->getOrientationTransform();
	            if (transform & NATIVE_WINDOW_TRANSFORM_ROT_90)
	                tr = tr * rot90;
	            if (transform & NATIVE_WINDOW_TRANSFORM_FLIP_H)
	                tr = tr * flipH;
	            if (transform & NATIVE_WINDOW_TRANSFORM_FLIP_V)
	                tr = tr * flipV;
	
	            // calculate the inverse
	            tr = inverse(tr);
	
	            // and finally apply it to the original texture matrix
	            const mat4 texTransform(mat4(static_cast<const float*>(textureMatrix)) * tr);
	            memcpy(textureMatrix, texTransform.asArray(), sizeof(textureMatrix));
	        }
	
	        // Set things up for texturing.
	        mTexture.setDimensions(mActiveBuffer->getWidth(), mActiveBuffer->getHeight());
	        mTexture.setFiltering(useFiltering);
	        mTexture.setMatrix(textureMatrix);
	
	        engine.setupLayerTexturing(mTexture);
	    } else {
	        engine.setupLayerBlackedOut();
	    }
	    drawWithOpenGL(hw, clip, useIdentityTransform);
	    engine.disableTexturing();
	}

里面关键就是drawwithOpenGL，可见是由Open GL来合成layer。





























































