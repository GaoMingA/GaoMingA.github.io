---
layout:     post
title:      "Android ANR分析"
subtitle:   "Android ANR"
navcolor:   "invert"
date:       2017-05-31
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android 
    - ANR 
---
# 概述

  ANR（Application Not Responding）是Android开发常见的一种问题，尤其是在稳定性测试中非常容易出现。我们可以理解ANR是Android的一种机制，Android应用程序是一个进程，该进程中有个main Thread，这个主线程中通过Looper维护着一个MessageQueue，主要负责响应用户事件和处理UI相关的一些事情。那么，如果用户事件或UI事件响应的很慢，官方这样说：[Generally, 100 to 200ms is the threshold beyond which users will perceive slowness in an application](https://developer.android.com/training/articles/perf-anr.html#Reinforcing)。既然100到200毫秒用户就感觉慢了，如果在主线程处理耗时任务，用户体验是不是就非常差呢，所以，Android官方强烈要求处理耗时任务不能在主线程中。但是，有可能我们在写代码时将一个耗时操作（IO、网络请求、数据操作等）放到了UI线程，或者，多个线程产生了死锁，导致主线程无法继续执行等等，在这种情况下如果没有有效的机制提醒用户并恢复进程状态，那么，用户可能认为是系统卡死或者定屏问题，从而怒砸手机。于是，在系统设计中便加入了ANR机制，会弹出一个dialog提示用户。

  跟所有知识点一样，我们通过思维导图将整个知识架构先搭起来：

![ANR知识架构](https://github.com/GaoMingA/blogger/blob/master/img/android/Android_ANR.png?raw=true)

# ANR机制触发原理

  anr机制在三种情况下会触发:

1. ServiceTimeout(20seconds)
2. BroadcasetTimeout(10seconds)
3. KeyDispatchTimeout(5seconds)

## Service超时

```
/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
```

[Service启动]()会走到`realStartServiceLocked`方法。

```Java
final ActivityManagerService mAm; 
// How long we wait for a service to finish executing.
static final int SERVICE_TIMEOUT = 20*1000;
// How long we wait for a service to finish executing.
static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
```

```java
 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
   		...
   		bumpServiceExecutingLocked(r, execInFg, "create");
   		...
 }
```

```Java
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
  		...
        scheduleServiceTimeoutLocked(r.app);
  		...
}
```

```Java
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        long now = SystemClock.uptimeMillis();
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        mAm.mHandler.sendMessageAtTime(msg,
                proc.execServicesFg ? (now+SERVICE_TIMEOUT) : (now+ SERVICE_BACKGROUND_TIMEOUT));
    }
```

mAm是ActivityManagerService的对象，向ActivityManagerService的内部Handler子类MainHandler发送`SERVICE_TIMEOUT_MSG`消息。这里发送的消息是delay消息，如果是前台进程中运行的Service，那么delay时间是`now+SERVICE_TIMEOUT`从当前时间计算20秒后，如果是后台进程中运行的Service，delay时间是`now+ SERVICE_BACKGROUND_TIMEOUT`200秒后。

消息发送后，处理在ActivityManagerService：

```Java
final ActiveServices mServices;
```

```Java
final class MainHandler extends Handler {
        public MainHandler(Looper looper) {
            super(looper, null, true);
        }
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case SERVICE_TIMEOUT_MSG: {
                if (mDidDexOpt) {
                    mDidDexOpt = false;
                    Message nmsg = mHandler.obtainMessage(SERVICE_TIMEOUT_MSG);
                    nmsg.obj = msg.obj;
                    mHandler.sendMessageDelayed(nmsg, ActiveServices.SERVICE_TIMEOUT);
                    return;
                }
                mServices.serviceTimeout((ProcessRecord)msg.obj);
            } break;
            ...
            }
        }
}
```

handler在收到`SERVICE_TIMEOUT_MSG`消息后，判断是不是在做odex的动作，如果是，再延迟20秒发送消息，因为odex操作比较耗时，如果不是做odex操作，直接调用到`mServices.serviceTimeout((ProcessRecord)msg.obj);`：

```Java
void serviceTimeout(ProcessRecord proc) {
        String anrMessage = null;

        synchronized(mAm) {
            if (proc.executingServices.size() == 0 || proc.thread == null) {
                return;
            }
            final long now = SystemClock.uptimeMillis();
            final long maxTime =  now -
                    (proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
            ServiceRecord timeout = null;
            long nextTime = 0;
            //寻找超时的Service
            for (int i=proc.executingServices.size()-1; i>=0; i--) {
                ServiceRecord sr = proc.executingServices.valueAt(i);
                if (sr.executingStart < maxTime) {
                    timeout = sr;
                    break;
                }
                if (sr.executingStart > nextTime) {
                    nextTime = sr.executingStart;
                }
            }
            //如果Service超时的进程不在最近运行的进程，忽略这个ANR
            if (timeout != null && mAm.mLruProcesses.contains(proc)) {
                Slog.w(TAG, "Timeout executing service: " + timeout);
                StringWriter sw = new StringWriter();
                PrintWriter pw = new FastPrintWriter(sw, false, 1024);
                pw.println(timeout);
                timeout.dump(pw, "    ");
                pw.close();
                mLastAnrDump = sw.toString();
                mAm.mHandler.removeCallbacks(mLastAnrDumpClearer);
                mAm.mHandler.postDelayed(mLastAnrDumpClearer, LAST_ANR_LIFETIME_DURATION_MSECS);
                anrMessage = "executing service " + timeout.shortName;
            } else {
                Message msg = mAm.mHandler.obtainMessage(
                        ActivityManagerService.SERVICE_TIMEOUT_MSG);
                msg.obj = proc;
                mAm.mHandler.sendMessageAtTime(msg, proc.execServicesFg
                        ? (nextTime+SERVICE_TIMEOUT) : (nextTime + SERVICE_BACKGROUND_TIMEOUT));
            }
        }

        if (anrMessage != null) {
            mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage);
        }
    }
```

anrMessage不为空，最后调用到`mAm.mAppErrors.appNotResponding(proc, null, null, false, anrMessage);`，appNotResponding方法就开始打印ANR发生时的一些日志，这时ANR已经被触发了。按照我们刚才的思路，那不是所有的Service运行就会触发ANR吗？当然不是，在延时发送`SERVICE_TIMEOUT_MSG`时，设计了取消该message发送的方法，也就是说在`SERVICE_TIMEOUT_MSG`发送前的delay时间内，Service完成响应，则取消该ANR信息的发送，那么整个Service ANR机制就完善了：

```Java
   private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        ...
        if (r.executeNesting <= 0) {
            if (r.app != null) {
                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE,
                        "Nesting at 0 of " + r.shortName);
                r.app.execServicesFg = false;
                r.app.executingServices.remove(r);
                if (r.app.executingServices.size() == 0) {
                    if (DEBUG_SERVICE || DEBUG_SERVICE_EXECUTING) Slog.v(TAG_SERVICE_EXECUTING,
                            "No more executingServices of " + r.shortName);
                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
					...
                }
        ...
   }
```

判断如果当前Service所在进程中无正在运行的进程，则调用`removeMessages`移除`SERVICE_TIMEOUT_MSG`。

## Broadcast超时



## InputEvent超时



# ANR触发后报告输出



# 分析ANR需要的LOG

  既然ANR发生了，那么我们就要检测是什么原因触发了ANR，需要的log日志如下(示例来自[官网](https://source.android.com/source/read-bug-reports#anrs-deadlocks))：

**event log:**

```
grep "am_anr" bugreport-2015-10-01-18-13-48.txt
```

grep "am_anr" 从event log中查找哪个进程发生了ANR，可以看到发生ANR时的时间点、pid、组件包名。

```verilog
10-01 18:12:49.599  4600  4614 I am_anr  : [0,29761,com.google.android.youtube,953695941,executing service com.google.android.youtube/com.google.android.apps.youtube.app.offline.transfer.OfflineTransferService]
10-01 18:14:10.211  4600  4614 I am_anr  : [0,30363,com.google.android.apps.plus,953728580,executing service com.google.android.apps.plus/com.google.android.apps.photos.service.PhotosService]
```

如上例子，第一条ANR信息中，发生anr的时间是 10-01 18:12:49 、pid是29761、进程为com.google.android.youtube。第二条ANR信息中，发生anr的时间是10-01 18:14:10.211、pid是30363、进程为com.google.android.apps.plus

**logcat 日志**

 logcat 日志中包含关于发生 ANR 时是什么在占用 CPU 以及发生anr的Reason等更多信息。

在日志中  grep "ANR in"

```
grep "ANR in" bugreport-2015-10-01-18-13-48.txt
```

```verilog
10-01 18:13:11.984  4600  4614 E ActivityManager: ANR in com.google.android.youtube
10-01 18:14:31.720  4600  4614 E ActivityManager: ANR in com.google.android.apps.plus
10-01 18:14:31.720  4600  4614 E ActivityManager: PID: 30363
10-01 18:14:31.720  4600  4614 E ActivityManager: Reason: executing service com.google.android.apps.plus/com.google.android.apps.photos.service.PhotosService
10-01 18:14:31.720  4600  4614 E ActivityManager: Load: 35.27 / 23.9 / 16.18
10-01 18:14:31.720  4600  4614 E ActivityManager: CPU usage from 16ms to 21868ms later:
10-01 18:14:31.720  4600  4614 E ActivityManager:   74% 3361/mm-qcamera-daemon: 62% user + 12% kernel / faults: 15276 minor 10 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   41% 4600/system_server: 18% user + 23% kernel / faults: 18597 minor 309 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   32% 27420/com.google.android.GoogleCamera: 24% user + 7.8% kernel / faults: 48374 minor 338 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   16% 130/kswapd0: 0% user + 16% kernel
10-01 18:14:31.720  4600  4614 E ActivityManager:   15% 283/mmcqd/0: 0% user + 15% kernel
...
10-01 18:14:31.720  4600  4614 E ActivityManager:   0.1% 27248/irq/503-synapti: 0%
10-01 18:14:31.721  4600  4614 I ActivityManager: Killing 30363:com.google.android.apps.plus/u0a206 (adj 0): bg anr
```

> 注：当ANR不是发生在system server进程时，mian log会有关键字"ANR in”，如果anr发生在 system server进程，则main log一般不会记录到关键字"ANR in”
>
> 可以通过main log关注anr发生到结束前一段时间（2s）的系统运行情况。

**/data/anr/tarces.txt**

trace文件用于查看anr发生时主线程的堆栈信息。trace文件中会有很多堆栈信息，需要确保找到正确的信息，就是通过event log中发生anr的时间点和pid信息来确认。

```verilog
------ VM TRACES AT LAST ANR (/data/anr/traces.txt: 2015-10-01 18:14:41) ------

----- pid 30363 at 2015-10-01 18:14:11 -----
Cmd line: com.google.android.apps.plus
Build fingerprint: 'google/angler/angler:6.0/MDA89D/2294819:userdebug/dev-keys'
ABI: 'arm'
Build type: optimized
Zygote loaded classes=3978 post zygote classes=27
Intern table: 45068 strong; 21 weak
JNI: CheckJNI is off; globals=283 (plus 360 weak)
Libraries: /system/lib/libandroid.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/libwebviewchromium_loader.so libjavacore.so (7)
Heap: 29% free, 21MB/30MB; 32251 objects
Dumping cumulative Gc timings
Total number of allocations 32251
Total bytes allocated 21MB
Total bytes freed 0B
Free memory 9MB
Free memory until GC 9MB
Free memory until OOME 490MB
Total memory 30MB
Max memory 512MB
Zygote space size 1260KB
Total mutator paused time: 0
Total time waiting for GC to complete: 0
Total GC count: 0
Total GC time: 0
Total blocking GC count: 0
Total blocking GC time: 0

suspend all histogram:  Sum: 119.728ms 99% C.I. 0.010ms-107.765ms Avg: 5.442ms Max: 119.562ms
DALVIK THREADS (12):
"Signal Catcher" daemon prio=5 tid=2 Runnable
  | group="system" sCount=0 dsCount=0 obj=0x12c400a0 self=0xef460000
  | sysTid=30368 nice=0 cgrp=default sched=0/0 handle=0xf4a69930
  | state=R schedstat=( 9021773 5500523 26 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xf496d000-0xf496f000 stackSize=1014KB
  | held mutexes= "mutator lock"(shared held)
  native: #00 pc 0035a217  /system/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char> >&, int, char const*, art::ArtMethod*, void*)+126)
  native: #01 pc 0033b03b  /system/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&) const+138)
  native: #02 pc 00344701  /system/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+424)
  native: #03 pc 00345265  /system/lib/libart.so (art::ThreadList::RunCheckpoint(art::Closure*)+200)
  native: #04 pc 00345769  /system/lib/libart.so (art::ThreadList::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+124)
  native: #05 pc 00345e51  /system/lib/libart.so (art::ThreadList::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+312)
  native: #06 pc 0031f829  /system/lib/libart.so (art::Runtime::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char> >&)+68)
  native: #07 pc 00326831  /system/lib/libart.so (art::SignalCatcher::HandleSigQuit()+896)
  native: #08 pc 003270a1  /system/lib/libart.so (art::SignalCatcher::Run(void*)+324)
  native: #09 pc 0003f813  /system/lib/libc.so (__pthread_start(void*)+30)
  native: #10 pc 00019f75  /system/lib/libc.so (__start_thread+6)
  (no managed stack frames)

"main" prio=5 tid=1 Suspended
  | group="main" sCount=1 dsCount=0 obj=0x747552a0 self=0xf5376500
  | sysTid=30363 nice=0 cgrp=default sched=0/0 handle=0xf74feb34
  | state=S schedstat=( 331107086 164153349 851 ) utm=6 stm=27 core=3 HZ=100
  | stack=0xff00f000-0xff011000 stackSize=8MB
  | held mutexes=
  kernel: __switch_to+0x7c/0x88
  kernel: futex_wait_queue_me+0xd4/0x130
  kernel: futex_wait+0xf0/0x1f4
  kernel: do_futex+0xcc/0x8f4
  kernel: compat_SyS_futex+0xd0/0x14c
  kernel: cpu_switch_to+0x48/0x4c
  native: #00 pc 000175e8  /system/lib/libc.so (syscall+28)
  native: #01 pc 000f5ced  /system/lib/libart.so (art::ConditionVariable::Wait(art::Thread*)+80)
  native: #02 pc 00335353  /system/lib/libart.so (art::Thread::FullSuspendCheck()+838)
  native: #03 pc 0011d3a7  /system/lib/libart.so (art::ClassLinker::LoadClassMembers(art::Thread*, art::DexFile const&, unsigned char const*, art::Handle<art::mirror::Class>, art::OatFile::OatClass const*)+746)
  native: #04 pc 0011d81d  /system/lib/libart.so (art::ClassLinker::LoadClass(art::Thread*, art::DexFile const&, art::DexFile::ClassDef const&, art::Handle<art::mirror::Class>)+88)
  native: #05 pc 00132059  /system/lib/libart.so (art::ClassLinker::DefineClass(art::Thread*, char const*, unsigned int, art::Handle<art::mirror::ClassLoader>, art::DexFile const&, art::DexFile::ClassDef const&)+320)
  native: #06 pc 001326c1  /system/lib/libart.so (art::ClassLinker::FindClassInPathClassLoader(art::ScopedObjectAccessAlreadyRunnable&, art::Thread*, char const*, unsigned int, art::Handle<art::mirror::ClassLoader>, art::mirror::Class**)+688)
  native: #07 pc 002cb1a1  /system/lib/libart.so (art::VMClassLoader_findLoadedClass(_JNIEnv*, _jclass*, _jobject*, _jstring*)+264)
  native: #08 pc 002847fd  /data/dalvik-cache/arm/system@framework@boot.oat (Java_java_lang_VMClassLoader_findLoadedClass__Ljava_lang_ClassLoader_2Ljava_lang_String_2+112)
  at java.lang.VMClassLoader.findLoadedClass!(Native method)
  at java.lang.ClassLoader.findLoadedClass(ClassLoader.java:362)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:499)
  at java.lang.ClassLoader.loadClass(ClassLoader.java:469)
  at android.app.ActivityThread.installProvider(ActivityThread.java:5141)
  at android.app.ActivityThread.installContentProviders(ActivityThread.java:4748)
  at android.app.ActivityThread.handleBindApplication(ActivityThread.java:4688)
  at android.app.ActivityThread.-wrap1(ActivityThread.java:-1)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1405)
  at android.os.Handler.dispatchMessage(Handler.java:102)
  at android.os.Looper.loop(Looper.java:148)
  at android.app.ActivityThread.main(ActivityThread.java:5417)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:726)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:616)

  ...
  Stacks for other threads in this process follow
  ...
```

根据pid 30363 at 2015-10-01 18:14:11来确定这条信息。

> 在MTK平台，日志会被打包成db文件，利用gat工具打开，对应的文件分别为event-SYS_ANDROID_EVENT_LOG、main log-SYS_ANDROID_LOG、traces-SWT_JBT_TRACES.
>
> 系统DB位置/data/system/dropbox 包含CPU使用情况和traces文件信息等

# 问题分析和分类



## app层引发的ANR



## System 引发的ANR



# 关于ANR问题分析的总结





