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
    - Android keyevent 按键 
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

==这里为什么Recent app键无法在```interceptKeyBeforeQueueing(KeyEvent event, int policyFlags)```方法中监听的到呢？我们在后面详细说明==

## 问题处理思路回顾

​	首先想到的是能不能利用Android提供的SDK解决问题呢？

​	我们来看看```onKeyDown()```方法，查看该方法的实现，该方法是接口```KeyEvent.Callback```中的抽象方法，所有的View全部实现了该接口并重写了该方法，该方法用来捕捉手机键盘被按下的事件。也就是说，如果我们要在Service里面来重写该方法是不可能的，所以这个方法不能用。而且这个方法只能监听到，```volume_up```、```volume_down```和```back```这几个按键的消息，无法监听到```home```、```power```和```recent_app```，这是因为这几个按键在framework层已经被系统处理了，没有再dispatch到view层处理。

​	另外，```home```，```recent_app```键可以通过OnRecieve系统广播 ```Intent.ACTION_CLOSE_SYSTEM_DIALOGS ```来获取，但是需求要监听很多按键，其他按键并没有系统广播，或者action类型比较多，处理起来比较麻烦，而且客户规定了他接受广播的action和key。

​	所以，本文采用了在framework层发送按键广播的形式来解决问题，就是在第一节中的解决方案。

## 系统按键处理流程




