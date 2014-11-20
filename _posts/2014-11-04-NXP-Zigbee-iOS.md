---
title: NXP Zigbee iOS
layout: post
tags: [iOS,Zigbee]
comments: true
---

最近开发基于Zigbee的网关，需要开发iOS程序，Zigbee方面的库文件使用NXP的。我使用OS X版本是10.9.5，
Xcode版本是6.0.1，SDK版本为iOS8.0，App的最低支持版本是iOS7.0，NXP的库文件版本是1.5。

首先，在NXP网站下载JN-UG-3086和JN-AN-1194，在JN-AN-1194的Source/Host目录下找到libJIP-v1.5，
然后将其Source目录和Include目录懂添加到Xcode中。

1.将TARGETS->Build Settings->Search Paths->Always Search User Paths改为YES

2.在libJIP.c中加入

{% highlight objc %}
#define LIBJIP_VERSION_MAJOR    "1"
#define LIBJIP_VERSION_MINOR    "5"
#define LIBJIP_VERSION          "1.5"
{% endhighlight %}

3.加入libxml库，在TARGET->General->Linked Frameworks and Libraries中添加libxml2.dylib，
然后TARGETS->Build Settings->Search Paths->Header Search Paths中添加${SDK_ROOT}/usr/include/libxml2

4.在TARGET->Build Settings->Apple LLVM 6.0-Preprocessing->Preprocesser Macros中加入
__APPLE_USE_RFC_2292，并在Network.c中加入

{% highlight objc %}

#define IPV6_RECVPKTINFO    IPV6_PKTINFO

{% endhighlight %}

5.在https://gist.github.com/panzi/6856583下载portable_endian.h，将其添加到工程中

6.在Threads.c里面添加#include <unistd.h>，并修改`eJIPLockLockTimed`函数

{% highlight objc %}

teLockStatus eJIPLockLockTimed(tsLock *psLock, uint32_t u32WaitTimeout)
{
    tsLockPrivate *psLockPrivate = (tsLockPrivate *)psLock->pvPriv;
    
#ifndef WIN32
    {
        struct timeval sNow;
        struct timespec sTimeout;
        
        gettimeofday(&sNow, NULL);
        sTimeout.tv_sec = sNow.tv_sec + u32WaitTimeout;
        sTimeout.tv_nsec = sNow.tv_usec * 1000;
        
        DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx time locking: %p\n", pthread_self(), psLock);

        switch (pthread_mutex_timedlock(&psLockPrivate->mutex, &sTimeout))
        {
            case (0):
                DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: time locked: %p\n", pthread_self(), psLock);
                return E_LOCK_OK;
                break;
                
            case (ETIMEDOUT):
                DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: time out locking: %p\n", pthread_self(), psLock);
                return E_LOCK_ERROR_TIMEOUT;
                break;

            default:
                DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: error locking: %p\n", pthread_self(), psLock);
                return E_LOCK_ERROR_FAILED;
                break;
        }
    }
    
#else
    
    
#endif /* WIN32 */
}

{% endhighlight %}

改为

{% highlight objc %}

teLockStatus eJIPLockLockTimed(tsLock *psLock, uint32_t u32WaitTimeout)
{
    tsLockPrivate *psLockPrivate = (tsLockPrivate *)psLock->pvPriv;
    
#ifndef WIN32
    
    int32_t ecode = E_LOCK_ERROR_FAILED;
#if defined(_POSIX_TIMEOUTS) && (_POSIX_TIMEOUTS - 200112L) >= 0L
    /* POSIX Timeouts are supported - option group [TMO] */
    {
        struct timeval sNow;
        struct timespec sTimeout;
        
        gettimeofday(&sNow, NULL);
        sTimeout.tv_sec = sNow.tv_sec + u32WaitTimeout;
        sTimeout.tv_nsec = sNow.tv_usec * 1000;
        
        DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx time locking: %p\n", pthread_self(), psLock);
        
        ecode = pthread_mutex_timedlock(&psLockPrivate->mutex, &sTimeout);
        
    }

    
#else
    
    /* Not OK to use the functions */
    ecode = pthread_mutex_trylock(&psLockPrivate->mutex);
    
    
#endif
    switch (ecode)
    {
    case (0):
        DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: time locked: %p\n", pthread_self(), psLock);
        return E_LOCK_OK;
        break;
        
    case (ETIMEDOUT):
        DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: time out locking: %p\n", pthread_self(), psLock);
        return E_LOCK_ERROR_TIMEOUT;
        break;
        
    default:
        DBG_vPrintf(DBG_LOCKS, "Thread 0x%lx: error locking: %p\n", pthread_self(), psLock);
        return E_LOCK_ERROR_FAILED;
        break;
    }
    
#else
    //wind32
    
#endif /* WIN32 */
}

{% endhighlight %}

{% highlight objc %}

{% endhighlight %}