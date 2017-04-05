---
layout:     post
title:      "Android 按键拦截处理"
subtitle:   "按键事件上报流程"
navcolor:   "invert"
date:       2017-04-05
author:     "gaoming"
header-img: "/img/website/home-bg.jpg"
catalog: true
tags:
    - Android 
    - keyevent 
    - 按键 
---

# Android 按键拦截处理

​	这段时间收到一个需求，某客户为了手机售后debug，需要一个Service来获取、记录用户在使用手机时所有按键事件（Back、 Home 、Recent app 、volume_up， volume_down和Power键），何时按下什么键，记录到数据库。保存到数据库后怎么上传服务器，怎么清除本地文件，由客户自己实现，我们这里暂不关心。本文的重点是按键上报。

## 解决方案

​	这里，本文先将最后的处理方案写出来，在后面的章节详细分析整个过程的处理思路。由于客户给出的接收按键的方式是一个广播接收器，所以，本文将截获的按键事件，都是通过广播的形式发送出去的。

### 广播接收器：

```java
class KeyWatcherReceiver extends BroadcastReceiver{
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            Log.d("KeyWatcherReceiver","getAction =" + action);
            if(action.equals("com.action.lgdb.key_event")){
                Bundle bundle = intent.getExtras();
                Log.d("KeyWatcherReceiver","bundle =" + bundle);
                if (bundle == null){
                    return;
                }
                int type = bundle.getInt("lgdb_key_event",-1);
//              String type = bundle.getString("lgdb_key_event",null);
                Log.d("KeyWatcherReceiver","type = " + type);
                if (type > 0){
                    String strType = String.valueOf(type);
                    Toast.makeText(context,strType,Toast.LENGTH_SHORT).show();
                }
             }
         }
  }
```

​	广播接收器最后获得的type值为按键对应的值，然后进行相关处理即可。

### Back、Home、Volume_up、Volum_down和Power

​	以上这五个按键，我们在在 ```PhoneWindowManager.java``` 的中的```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```方法中处理，该方法是在按键入队之前对其进行系统级处理，后面我们在详细流程中来分析系统是如何走到该方法的。

```
frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java 
```

```java
 public int interceptKeyBeforeQueueing(KeyEvent event, int policyFlags) {

   ......
     
        //add by gaoming for LG_MLT Menu key Monitor--start{
        if (keyCode > 0 ){
           if (down) {
             Intent intent = new Intent();
             intent.setAction(ACTION_MLT_KEY_EVENT); //ACTION_MLT_KEY_EVENT = "com.action.lgdb.key_event"
             Bundle bundle = new Bundle();
             /*if ((keyCode == KeyEvent.KEYCODE_POWER) || (keyCode ==KeyEvent.KEYCODE_HOME) || (keyCode ==KeyEvent.KEYCODE_BACK)
                   || (keyCode ==KeyEvent.KEYCODE_MENU) || (keyCode ==KeyEvent.KEYCODE_VOLUME_UP)|| (keyCode ==KeyEvent.KEYCODE_VOLUME_DOWN)){
                 bundle.putInt("lgdb_key_event", keyCode);
                 Log.d("gaoming", " send lgdb key event value: " + keyCode);
             }*/
             if (KeyEvent.KEYCODE_POWER == event.getKeyCode()){
                 bundle.putInt("lgdb_key_event", 26);
                 Log.d("gaoming", " send lgdb key event Power: " + "MLK_KEY_POWER");
             }else if (KeyEvent.KEYCODE_HOME == event.getKeyCode()){
                 bundle.putInt("lgdb_key_event", 3);
                 Log.d("gaoming", " send lgdb key event Home: " + "MLK_KEY_HOME");
             }else if (KeyEvent.KEYCODE_BACK == event.getKeyCode()){
                 bundle.putInt("lgdb_key_event", 4);
                 Log.d("gaoming", " send lgdb key event BACK: " + "MLK_KEY_BACK");
             //}else if (KeyEvent.KEYCODE_APP_SWITCH == event.getKeyCode()){
             //    bundle.putInt("lgdb_key_event", 187);
             //    Log.d("gaoming", " send lgdb key event APP_SWITCH: " + "KEYCODE_APP_SWITCH");
             }else if (KeyEvent.KEYCODE_VOLUME_UP == event.getKeyCode()){
                 bundle.putInt("lgdb_key_event", 24);
                 Log.d("gaoming", " send lgdb key event VOLUME_UP: " + "MLK_KEY_VOLUME_UP");
             }else if (KeyEvent.KEYCODE_VOLUME_DOWN == event.getKeyCode()){
                 bundle.putInt("lgdb_key_event", 25);
                 Log.d("gaoming", " send lgdb key event VOLUME_DOWN: " + "MLK_KEY_VOLUME_DOWN");
             }
             intent.putExtras(bundle);
             mContext.sendBroadcast(intent);
           }
        }
        //end--}
   
  ......
    
 }
```

​	可以看到 ```KeyEvent.KEYCODE_APP_SWITCH``` 是被注释掉了，Recent app 按键监听，我们在```PhoneStatusBar.java ```实现.

### Recent app

```
frameworks/base /packages/SystemUI/src/com/android/systemui/statusbar/phone/PhoneStatusBar.java
```

```Java
private View.OnClickListener mRecentsClickListener = new View.OnClickListener() {
        public void onClick(View v) {
        //add by gaoming for LG_MLT Menu key Monitor--start{
             Intent intent = new Intent();
             intent.setAction("com.action.lgdb.key_event");
             Bundle bundle = new Bundle();
             bundle.putInt("lgdb_key_event", 187);
             Log.d("gaoming", " system ui send lgdb key event MENU: " + "KEYCODE_APP_SWITCH");
             intent.putExtras(bundle);
             mContext.sendBroadcast(intent);
       //--end}
            awakenDreams();
            toggleRecentApps();
        }
    };
```

> 思考：这里为什么Recent app键无法在```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```方法中监听的到呢？我们在后面Recent_app处理流程详细说明
>

## 问题处理思路回顾

​	首先想到的是能不能利用Android提供的SDK解决问题呢？

​	我们来看看```onKeyDown()```方法，查看该方法的实现，该方法是接口```KeyEvent.Callback```中的抽象方法，所有的View全部实现了该接口并重写了该方法，该方法用来捕捉手机键盘被按下的事件。也就是说，如果我们要在Service里面来重写该方法是不可能的，所以这个方法不能用。而且这个方法只能监听到，```volume_up```、```volume_down```和```back```这几个按键的消息，无法监听到```home```、```power```和```recent_app```，这是因为这几个按键在framework层已经被系统处理了，没有再dispatch到view层处理。

​	另外，```home```，```recent_app```键可以通过OnRecieve系统广播 ```Intent.ACTION_CLOSE_SYSTEM_DIALOGS ```来获取，但是需求要监听很多按键，其他按键并没有系统广播，或者action类型比较多，处理起来比较麻烦，而且客户规定了他接受广播的action和key。

​	所以，本文采用了在framework层发送按键广播的形式来解决问题，就是在第一节中的解决方案。

## 系统按键处理流程

 	 按键处理设计的整体思路是驱动层会有一个消息队列来存放事件，会有一个Reader来不停的读取事件，一个Dispatcher来分发消息队列中的事件。Dispatcher分发的事件最后会通过jni上报到InputManagerService，然后通过接口最后传递给PhoneWindowManager，这里再根据不同的按键事件类型来做不同的处理。上层能做的修改基本上都是从PhoneWindowManager中开始的。

![按键处理类图](https://github.com/GaoMingA/blogger/blob/master/img/website/android/keyevent_class_chart.png?raw=true)

上图引用自： [Android电源管理之关机流程](http://chendongqi.me/2017/02/21/pm_shutdown_flow/)

​	   可以参考上面的链接，并结合源码了解整个启动过程：

​	  SystemServer启动InputManagerService，然后依次创建了NativeInputManager、InputManager、InputReader、InputDispatcher这几个关键的对象以及InputReaderThread和InputDispatcherThread这两个关键线程。然后让这个两个thread启动起来，在InputReaderThread无限循环运行时，通过InputReader从EventHub中不断读取events然后通知InputDispatcher将事件入队。而InputDispatcherThread则通过InputDispatcher不停的将队列中的事件分发出去，这就是整个input系统的基本机制。

## Recent_app处理流程

 	 通过了解按键处理流程，我们知道```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```在Java层代码中，具有最高优先级，也就是说在按键事件未入队列，还不到dispatch之前就已经对其进行了处理，比如power键、Home键等系统响应事件都是在这个方法提前拦截并处理的。那么，为什么Recent app按键在此处无法生效呢？因为Home、Back和Recent_app都是在SystemUI里面绘制的虚拟按键，当SystemUI绘制了这三个按键后，走的流程又是怎么样的呢，Home和Back为什么可以被```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```拦截，二Recent_app不行呢，所以我们需要从SystemUI里面寻找答案。

首先对比一下home.xml和recent_apps.xml，这两个文件分别是Home键和Recent_app键绘制的xml文件

home.xml

```xml
<com.android.systemui.statusbar.policy.KeyButtonView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/home"
    android:layout_width="@dimen/navigation_key_width"
    android:layout_height="match_parent"
    android:layout_weight="0"
    android:src="@drawable/ic_sysbar_home"
    systemui:keyCode="3"
    android:scaleType="center"
    android:contentDescription="@string/accessibility_home"
    android:paddingStart="@dimen/navigation_key_padding"
    android:paddingEnd="@dimen/navigation_key_padding"
    />
```

recent_app.xml

```xml
<com.android.systemui.statusbar.policy.KeyButtonView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/recent_apps"
    android:layout_width="@dimen/navigation_key_width"
    android:layout_height="match_parent"
    android:layout_weight="0"
    android:src="@drawable/ic_sysbar_recent"
    android:scaleType="center"
    android:contentDescription="@string/accessibility_recent"
    android:paddingStart="@dimen/navigation_key_padding"
    android:paddingEnd="@dimen/navigation_key_padding"
    />
```

有一个很明显的区别  systemui:keyCode="3" ， keyCode=3是Home键在KeyEvent.java定义的系统value值，Recent_app是没有这个属性的。

> 注： 以上代码是Android N的实现代码，在之前的系统代码中 是在navigation_bar.xml里面定义的

看到区别后，我们来看看KeyButtonView这个类是怎么处理的。该类继承自ImageView，先看看其构造方法：

```java
    public KeyButtonView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs);
        ......
        mCode = a.getInteger(R.styleable.KeyButtonView_keyCode, 0);
        ......
    }
```

这里如果在xml中定义了systemui:keyCode属性，mCode就被赋值。当点击事件发生时，回调onTouchEvent()方法，该方法又调用到sendEvent()方法：

```java
void sendEvent(int action, int flags, long when) {
        final int repeatCount = (flags & KeyEvent.FLAG_LONG_PRESS) != 0 ? 1 : 0;
        final KeyEvent ev = new KeyEvent(mDownTime, when, action, mCode, repeatCount,
                0, KeyCharacterMap.VIRTUAL_KEYBOARD, 0,
                flags | KeyEvent.FLAG_FROM_SYSTEM | KeyEvent.FLAG_VIRTUAL_HARD_KEY,
                InputDevice.SOURCE_KEYBOARD);
        InputManager.getInstance().injectInputEvent(ev,
                InputManager.INJECT_INPUT_EVENT_MODE_ASYNC);
    }
```

可以看到，mCode已KeyEvent()构造函数参数的形式传给了KeyEvent，这个参数即KeyCode的value值，然后，新定义的KeyEvent对象ev以参数的形式传给了InputManager的injectInputEvent()方法:

```java
 /**
     * Injects an input event into the event system on behalf of an application.
     * The synchronization mode determines whether the method blocks while waiting for
     * input injection to proceed.
     ......
 **/
    public boolean injectInputEvent(InputEvent event, int mode) {
        if (event == null) {
            throw new IllegalArgumentException("event must not be null");
        }
        if (mode != INJECT_INPUT_EVENT_MODE_ASYNC
                && mode != INJECT_INPUT_EVENT_MODE_WAIT_FOR_FINISH
                && mode != INJECT_INPUT_EVENT_MODE_WAIT_FOR_RESULT) {
            throw new IllegalArgumentException("mode is invalid");
        }

        try {
            return mIm.injectInputEvent(event, mode);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
```

该方法的作用参考源码的注释，是将一个input event事件添加到system，那么是如何添加的呢，```return mIm.injectInputEvent(event, mode);```这句里面```mIm```是这样定义的对象```private final IInputManager mIm;```，所以是通过aidl的方式调用到了InputManagerService.java的```injectInputEvent()```方法。

```java
    @Override // Binder call
    public boolean injectInputEvent(InputEvent event, int mode) {
        return injectInputEventInternal(event, Display.DEFAULT_DISPLAY, mode);
    }

    private boolean injectInputEventInternal(InputEvent event, int displayId, int mode) {
        ......
        try {
            result = nativeInjectInputEvent(mPtr, event, displayId, pid, uid, mode,
                    INJECTION_TIMEOUT_MILLIS, WindowManagerPolicy.FLAG_DISABLE_KEY_REPEAT);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        ......
    }
```

交给InputManagerService.java后又通过JNI方式交给了native层处理，详细可根据本文第二张的input 事件流程分析，最后再次交给PhoneWindowManager.java的```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```处理。

​	  分析了这么多，上面的情况是Home有systemui:keyCode="3"属性时的情况，那么Recent_app就比较明确了，Recent_app没有systemui:keyCode这个属性，所以它不会有以上流程：将keyCode交给InputManager添加到system，所以Recent_app就直接在SystemUI里面处理其事件了，这也就是我们在“解决方案”一节为什么将Recent_app直接在SystemUI里面处理的原因。

​	  为了证实我们的想法，我们将recent_app.xml添加systemui:keyCode="187"属性，然后不在SystemUI里面监听recent_app事件，去掉第一节在PhoneWindowManager.java里注释内容，这样我们在```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```方法中依然可以监听Recent_app事件了。

## 总结

​	  本文以这个简单的需求为切入点，结合源码分析了Android按键事件的上报流程及SystemUI虚拟按键交给System处理的流程，以后遇到按键事件相关需求可做为参考。

