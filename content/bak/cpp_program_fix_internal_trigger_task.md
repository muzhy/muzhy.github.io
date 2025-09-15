+++
isCJKLanguage = true
title = "C++程序固定时长执行任务"
description = "按固定时间间隔触发任务。"
keywords = ["C++"]
date = 2024-02-03T14:35:41+08:00
authors = ["木章永"]
tags = ["C++"]
categories = []
cover = "/images/Nixie_clock.jpg"
draft = true
+++

# 使用`condition_variable`实现定时执行任务
遇到一个开发任务，需要按一定的时间间隔执行任务，本来是一个简单的功能，直接使用`condition_variable`就可以了（对实时性要求并不是太高，不需要考虑ms级别的误差）

最开始是直接使用`condition_variable`实现的定时触发机制, 代码的大致实现类似于：
```C++
#include <condition_variable>
#include <chrono>

typedef std::chrono::std::chrono::system_clockClockType;
typedef std::chrono::time_point<ClockType> TimePointType;

void main()
{
    TimePointType currTime = ClockType::now();
    TimePointType awaitTimePoint = currTime;
    int m_nCheckInterval = 10;

    std::condition_variable m_cv;
    std::mutex m_mut;
    bool m_bExit;
    
    while(!m_bExit)
    {
        currTime = ClockType::now();
        if (currTime <= m_awaitTimePoint)
        {
            if(m_cv.wait_until(lock, m_awaitTimePoint, [this](){ 
                return m_bExit;
              }))
            {
               if(m_bExit){ break; }
            }
        }

        if(m_bExit){ break; }

        // compute next await timepoint
        m_awaitTimePoint = m_awaitTimePoint + std::chrono::seconds(m_nCheckInterval);
        
        // do something ... 
    };
}
```

# 当系统时间发生变化时，等待间隔发生变化

但是在系统时间发生变化的时候出现了问题：

比如原本打算在16:30分唤醒程序继续执行，当前的系统时间为16:25，此时程序阻塞，等待5分钟后唤醒
然后由于各种原因，比如NTP自动调整时间或者手动设置系统时间，将系统时间修改为16:24
程序还是需要等待到16:30分才会被唤醒，此时需要等待6分钟

在这里不管是使用`wait_for`还是`wait_util`实际上是没有区别的：`wait_for`在实现上是依赖于`wait_util`的
[condition_variable/wait_for](https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for)
> Equivalent to return wait_until(lock, std::chrono::steady_clock::now() + rel_time, std::move(stop_waiting));. This overload may be used to ignore spurious awakenings by looping until some predicate is satisfied (bool(stop_waiting()) == true).

使用系统时钟`std::chrono::system_clock`会有问题，要让时间间隔不受系统时间的影响，应该是可以使用单调时钟来实现的
`std::chrono`提供了三种类型的时钟：
```C++
std::chrono::system_clock;
std::chrono::high_resolution_clock;
std::chrono::steady_clock;
```
其中`std::chrono::steady_clock`就是单调时钟
[std::chrono::steady_clock](https://en.cppreference.com/w/cpp/chrono/steady_clock)
> Class std::chrono::steady_clock represents a monotonic clock. The time points of this clock cannot decrease as physical time moves forward and the time between ticks of this clock is constant. This clock is not related to wall clock time (for example, it can be time since last reboot), and is most suitable for measuring intervals.

但是不幸的是，标准库在POSIX上的实现存在问题——不管使用的时钟类型是什么，实际用的都是`std::chrono::system_clock`

之前已经有人提出过这个问题了：[https://www.open-std.org/jtc1/sc22/wg21/docs/lwg-closed.html#887](https://www.open-std.org/jtc1/sc22/wg21/docs/lwg-closed.html#887)， [Correct C++11 std::condition_variable requires a version of pthread_cond_timedwait that supports specifying the clock](https://austingroupbugs.net/view.php?id=1164)
后来被放在[Adding clockid parameter to functions that accept absolute struct timespec timeouts](https://austingroupbugs.net/view.php?id=1216)，并最终在glibc 2.30 修复


# 使用使用`pthread_cond_timedwait`和`CLOCK_MONOTONIC`定时触发

但是升级glibc比较麻烦，特别是在生产环境上，所以需要使用替代的方案

其实`wait_util`在Linux上的实现是使用`pthread_cond_timedwait`来实现的，而`pthread_cond_timedwait`设置的时钟也有多个选择：
[https://linux.die.net/man/3/clock_gettime](https://linux.die.net/man/3/clock_gettime)
>CLOCK_REALTIME
System-wide realtime clock. Setting this clock requires appropriate privileges.
CLOCK_MONOTONIC
Clock that cannot be set and represents monotonic time since some unspecified starting point.
CLOCK_PROCESS_CPUTIME_ID
High-resolution per-process timer from the CPU.
CLOCK_THREAD_CPUTIME_ID
Thread-specific CPU-time clock.

实际上标准库提供的各种时钟类型也是使用上面的这些时钟类型进行的封装，单调时钟对应的是`CLOCK_MONOTONIC`

所以改成了使用`pthread_cond_timedwait`和`CLOCK_MONOTONIC`实现按照一定的时间间隔定时触发任务

```C++
void main()
{
    pthread_mutex_t m_mutex;
    pthread_condattr_t m_attr;
    pthread_cond_t m_cond;

    pthread_condattr_init(&m_attr);
    pthread_condattr_setclock(&m_attr, CLOCK_MONOTONIC);    
    pthread_cond_init(&m_cond, &m_attr);
    
    int m_nCheckInterval = 10;

    struct timespec start_mono_time;
    clock_gettime(CLOCK_MONOTONIC, &start_mono_time);
    struct timespec target_mono_time = start_mono_time;
    
    while(!m_bExit)
    {
        clock_gettime(CLOCK_MONOTONIC, &start_mono_time);
        if (start_mono_time.tv_sec < target_mono_time.tv_sec)
        {
            pthread_mutex_lock(&m_mutex);
            if(m_bExit){ break; }
            int res = pthread_cond_timedwait(&m_cond, &m_mutex, &target_mono_time);
            if(res != ETIMEDOUT)
            {
                if(m_bExit){ pthread_mutex_unlock(&m_mutex); break; }
            }
            pthread_mutex_unlock(&m_mutex);
        }

        if(m_bExit){ break; }

        // compute next await timepoint
        target_mono_time.tv_sec = target_mono_time.tv_sec + m_nCheckInterval;
        
        // do something ... 
    };
}
```



# 参考资料
https://stackoverflow.com/questions/51005267/how-do-i-deal-with-the-system-clock-changing-while-waiting-on-a-stdcondition-v
https://linux.die.net/man/3/clock_gettime
检测系统时间发信变化，原贴再stack-overflow, 需要利用linux 3.0 新加的特性 https://cloud.tencent.com/developer/ask/sof/102761402
标准库提供的时钟和wait_for,wait_util
https://en.cppreference.com/w/cpp/chrono/system_clock
https://en.cppreference.com/w/cpp/chrono/steady_clock
https://en.cppreference.com/w/cpp/thread/condition_variable/wait_for
利用新特性 TFD_TIMER_CANCEL_ON_SET 检查系统时间变化的参考手册
https://man7.org/linux/man-pages/man2/timerfd_create.2.html