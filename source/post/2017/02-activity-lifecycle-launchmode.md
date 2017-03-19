```toml
title = "Activity的生命周期和启动模式"
slug = "02-activity-lifecycle-launchmode"
desc = "02 activity lifecycle launchmode"
date = "2017-03-19 00:36:15"
update_date = "2017-03-19 00:36:15"
author = "wangganxin"
thumb = ""
tags = ["Lifecycle","LaunchMode"]
```

>目录<br>
>一、Activity生命周期<br>
>二、Activity的LaunchMode<br>
>三、IntentFilter匹配规则<br>
>参考资料<br>

网上类似的分析资料有很多，然而终究只是别人写出来的分享，自己写下来算是再次重温吧。

### 一、Activity生命周期

在正常情况下，Activity的生命周期：

- **onCreate**:Activity正在被创建
- **onRestart**:Activity正在重新启动
- **onStart**:Activity正在被启动
- **onResume**:Activity已经可见
- **onPause**:Activity正在停止，此时可以做一些存储数据，停止动画等操作，但注意不能太耗时，因为这会影响到新Activity的显示，onPause必须先执行完，新Activity的onResume才会执行
- **onStop**:Activity即将停止
- **onDetory**:Activity即将被销毁

<!--more-->

说明：

1. 当用户打开新的Activity或者切换到桌面的时候，回调如下：onPause->onStop。这里有一种特殊情况，如果新的Activity采用了透明主题，那么当前的Activity不会回调onStop。
2. 从整个生命周期来说，onCreate和onDestory是配对的，分别标识着Activity的创建和销毁，并且只可能有一次调用。从Activity是否可见来说，onStart和onStop是配对的，随着用户的操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次；从Activity是否在前台来说，onResume和onPause是配对的，随着用户操作或者设备屏幕的点亮和熄灭，这两个方法可能被调用多次。

Q1：onStart和onResume、onPause和onStop从描述上来看差不多，对我们来说有什么实质的不同？

>从实际使用过程中来说，onStart和onResume、onPause和onStop看起来的确差不多，甚至我们可以只保留其中的一对，比如只保留onStart和onStop，既然如此，为什么Android系统还要提供看起来重复的接口呢？根据上面的分析，这两个配对的回调分别表示不同的意义，onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调的，除了这种区别，在实际使用中没有其他明显区别。

Q2：假设当前Activity为A，如果这时用户打开一个新的Activity B，那么B的onResume和A的onPause哪个先执行呢？

>这一点可以从Android源码中得到解释。Activity的启动过程的源码相当复杂，涉及Instrumentation、ActivityThread和ActivityManagerService（简称AMS）。这里不详细分析这一过程，简单理解：启动Activity的请求由Instrumentation来处理，然后它通过Binder向AMS发送请求，AMS内部维护着一个ActivityStack并负责栈内的Activity的状态同步，AMS通过ActivityThread去同步Activity的状态从而完成生命周期的调用。

Q3：关于子线程更新UI，下段代码是否可以正常工作？

```Java

public class MainActivity extends AppCompatActivity {

    private TextView tv;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        tv=findViewById(R.id.main_tv);

        new Thread(new Runnable() {
            @Override
            public void run() {
                tv.setText("Other Thread");
            }
        }).start();
    }
}

```

>可以，首先应该知道，检查线程的工作是由ViewRoot来完成的，当访问UI的时候，ViewRoot会进行检查工作，如果不是在UI线程访问的程序就会抛出异常，但在OnCreate时候，ViewRoot还没有创建，无法检查上面线程访问UI控件，因此程序并不会报错。ViewRoot是在Activity的onResume之后才创建的。Android之所以要求子线程中不能更新UI，是因为UI访问是没有加锁的，在多线程访问的情况下是线程不安全的。


在异常情况下Activity生命周期：

- 资源相关的系统配置发生改变，导致activity被杀死并重新创建

>onSaveInstanceState的调用时机是在onStop之前；onRestoreInstanceState的调用时机是在onStart之后

- 资源不足，导致低优先级的activity被杀死

>前台Activity——正在和用户交互的Activity，优先级最高<br/>
可见但非前台Activity——比如Activity中弹出了一个对话框，导致Activity可见但是位于后台无法与用户直接交互<br/>
后台Activity——已经被暂停的Activity，优先级最低<br/>

### 二、Activity的LaunchMode
- standard，标准模式，也是系统的默认模式，每次调用创建一个新Activity的实例

		适用于普通的页面内容展示

- singleTop，栈顶复用，如果新Activity已经位于任务栈的栈顶，不会重新创建，同时onNewIntent方法被回调；否则，重新创建该Activity实例

		适用于接收通知启动的内容显示页面。例如某个新闻客户端的新闻内容页面，
		如果收到10多个新闻推送，每次都打开一个新闻内容页面是很烦人的

- singleTask，栈内复用，任务栈中没有该类型的Activity实例，则重新创建；否则，onNewIntent方法回调+clearTop

		适合作为程序入口点。例如浏览器的主界面，不管从多少个应用启动浏览器，
		只会启动主界面一次，其余情况都会走onNewIntent，并且会清空主界面上面的其他页面

- singleInstance，单实例模式，任务栈中只有一个该Activity实例，没有其他Activity，后续请求均不会创建新的Activity

		适合需要与程序分离开的页面。例如闹铃提醒，将闹铃提醒与闹铃设置分离，
		singleInstance不要用于中间页面，如果用于中间页面， 跳转会有问题 ，
		比如：A -> B (singleInstance) -> C，完全退出后，在此启动，首先打开的是B


###三、IntentFilter匹配规则

启动Activity分为两种，显式调用和隐式调用。显式调用需要明确指定被启动对象的组件信息，包括包名和类名，而隐式调用则不需要明确指定组件信息。原则上一个Intent不应该既是显式调用又是隐式调用，如果两者共存，以显式调用为主。另外，一个Activity中可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可成功启动对应的Activity。

```xml
        <activity android:name=".framework.sample.SampleListActivity">
            <intent-filter>
                <action android:name="android.intent.action.SEND"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data android:mimeType="text/plain"/>
            </intent-filter>

            <intent-filter>
                <action android:name="android.intent.action.SEND"/>
                <action android:name="android.intent.action.SEND_MULTIPLE"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data android:mimeType="application/vnd.google.panorama360+jpg"/>
                <data android:mimeType="image/*"/>
                <data android:mimeType="video/*"/>
            </intent-filter>
        </activity>
```

- action的匹配规则

> 1. action是一个字符串，系统预定义了一些action，我们也可以在应用中定义自己的action
> 2. 一个匹配规则中可以有多个action
> 3. action的匹配要求Intent中的action存在且必须和过滤规则长得其中一个action相同
> 4. action区分大小写，大小写不同字符串相同的action会匹配失败

- category的匹配规则

> 1. category同样是一个字符串，系统预定义了一些category，也可以在应用中定义自己的
> 2. 要求Intent中可以没有category，但是一旦有category，不管有几个，每个都要能够和过滤规则中的任何一个category相同
> 3. 为什么跳转activity的时候不设置category也可以匹配呢？原因是系统在调用startActivity或者startctivityForResult的时候默认为Intent加上了“android.intent.catetgory.DEFAULT”，同时为了我们的activity能够接受隐式调用，就必须在intent-filter中指定“android.intent.catetgory.DEFAULT”这个category。

- data的匹配规则

> 1. 匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data


### 参考资料

- 《Android开发艺术探索》—任玉刚