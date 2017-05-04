```toml
title = "Android性能优化摘录"
slug = "05-android-performance-optimization"
desc = "android performance optimization"
date = "2016-12-14 14:32:31"
update_date = "2016-12-14 14:32:31"
author = "wangganxin"
thumb = ""
tags = ["性能优化"]
```
>目录<br>
>一、View的过度绘制（OverDraw）<br>
>二、View的绘制流程<br>
>三、三种常用布局的比较<br>
>四、RecyclerView VS ListView 之View层级关系<br>
>五、高效布局标签<br>
>六、去掉window的背景<br>
>七、去掉其他不必要的背景<br>
>八、ClipRect & QuickReject<br>
>九、善用draw9patch<br>
>十、慎用Alpha<br>
>十一、应该早点知道的API<br>
>十二、其他<br>


本文是[有心课堂-性能优化合辑](http://stay4it.com/course/26) 视频的学习笔记，也翻阅过网上的相关资料，整理了个人认为比较重要的知识点。

### 一、View的过度绘制（OverDraw）

OverDraw，是指在一帧的时间内（16.67ms）像素被绘制了多次，理论上一个像素只绘制一次是最优的，但由于重叠的布局导致一些像素被重复绘制多次，而每次绘制都会对应到CPU的一组绘图命令和GPU的一些操作，当这个操作超过16.67ms时就会出现掉帧的现象，即我们常说的卡顿，所以对重叠不可见元素的重复绘制会产生额外的开销，我们需要尽量减少OverDraw的发生。

Android提供了测量OverDraw的选项，在开发者选项->调试GPU过度绘制（Show GPU OverDraw）,打开该选项就可以看到当前页面OverDraw的状态。

- 没有颜色 ： 没有OverDraw。像素只画了一次。
- 蓝色：OverDraw 1倍。像素绘制了两次，大片的蓝色是可以接受的（若整个窗口都是蓝色，可以摆脱一层）。
- 绿色：OverDraw 2倍。像素绘制了三次，中等大小的绿色区域是可以接受的，但你应该尝试去优化、减少它们。
- 浅红：OverDraw 3倍。像素绘制了四次，小范围可接受。
- 暗红：OverDraw 4倍。像素绘制了五次或更多，这是错误的，要修复它们。

![GPU-OverDraw](/media/2016/gpu-overdraw.png)

### 二、View的绘制流程

- measure :为整个View树计算实际的大小，即设置实际的高（对应属性：mMeasuredHeight）和宽(对应属性：mMeasureWidth)，每个View的控件的实际宽高都是由父视图和本身的视图决定的。
- layout：为将整个根据子视图的大小以及布局参数将View树放到合适的位置上。
- draw：利用前两步得到的参数，将视图显示在屏幕上。

<!--more-->

### 三、三种常用布局的比较

RelativeLayout：

	优点：
	1. View树扁平化，减少View层级
	2. 使用场景广
	缺点：
	1. 测量效率稍差
	2. 使用略复杂

LinearLayout：

	优点：
	1. 使用非常简单
	2. 测量效率高
	缺点：
	1. 嵌套过多，易导致View层级复杂
	2. 使用场景相对较窄

FrameLayout：

	使用场景特殊，有些场景下可以替代RelativeLayout

选择布局容器的基本准则：

- 尽可能的使用RelativeLayout以减少View层级，使View树趋于扁平化
- 在不影响层级深度的情况下，使用LinearLayout和FrameLayout，而不是RelativeLayout

### 四、RecyclerView VS ListView 之View层级关系

		
					 | --- RecyclerView
		ViewGroup----|                                    |--- ListView
					 | --- AdapterView --- AbsListView ---|
														  |--- GridView

从View层级关系，我们可以看出，在实际开发中更推荐使用RecyclerView。


### 五、高效布局标签

- Merge标签：减少视图的层级结构。

Merage标签一般配合FrameLayout和LinearLayout使用，当然RelativeLayout也可以用Merge标签，只是RelativeLayout本身比较复杂，如果我们也通过Merge标签去添加一些子View的时候很容易弄巧成拙。

另外Merge标签只能作为XML布局的根标签使用，当Inflate以<merge/>开头的布局文件时，必须指定一个父ViewGroup，并且必须设定attachToRoot为true。

- ViewStub标签：高效占位符。

我们经常会遇到这的情况，运行时动态根据条件来决定显示哪个View或者布局。常用的做法是把View都写在上面，先把他们的可见性都设为View.GONE，然后在代码中动态的更改它的可见性。这样的做法的优点是逻辑简单而且控制起来比较灵活。但是它的缺点就是耗费资源。虽然把View的初始化可见View.GONE，但是在Inflate布局的时候View仍然会被Inflate，也就是说仍然会创建对象，会被实例化，会被设置属性，也就是说会耗费内存等资源。

ViewStub是一个轻量级的View，它是一个看不见的，不占布局位置，占用资源非常小的控件，可以为ViewStub指定一个布局，在Inflate布局的时候，只有ViewStub会被初始化，然后当ViewStub被设置为可见的时候或是调用了ViewStub.inflate()的时候，ViewSub所指向的布局会被inflateh和实例化，然后ViewStub的布局属性都会传给它所指向的布局。

	<ViewStub
    android:id="@+id/stub_view"
    android:inflatedId="@+id/panel_stub"
    android:layout="@layout/progress_overlay"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="bottom" />

当想加载布局时，可以使用下面其中的一种方法：

	((ViewStub) findViewById(R.id.stub_view)).setVisibility(View.VISIBLE);
	或
	View importPanel = ((ViewStub) findViewById(R.id.stub_view)).inflate();

- Space标签：空白控件。

### 六、去掉window的背景

在我们使用了Android自带的一些主题时，window会被默认添加一个纯色的背景，这个背景是被DecorView持有的，当我们的自定义布局时又添加一张背景图或者设置背景色，那么DecorView的background此时对我们来说是无用的，但是它会产生一次OverDraw，带来绘制性能损耗。

去掉window的背景可以在onCreate中的setContentView之后调用

	getWindow().setBackgroundDrawable(null);

或者在theme中添加

	android:windowbackground="null";

### 七、去掉其他不必要的背景

- 有时候为了方便会先给Layout设置一个整体的背景，再给子View设置背景，会造成重叠，如果子View宽度match_parent，可以看到完全覆盖了Layout的一部分，这里就可以通过分别设置背景来减少重绘
- 如果采用的是selector的背景，将normal状态的color设置为"@android:color/transparent"也同样可以解决问题。

这里只简单举两个例子，我们在开发过程中的一些习惯性的思维定式会带来不经意的OverDraw，所以开发过程中我们为某个View或者ViewGroup设置背景的时候，先思考下是否真的有必要，或者思考下这个背景能不能分段设置在子View上，而不是图方便直接设置在根View上。

### 八、ClipRect & QuickReject

为了解决OverDraw的问题，Android系统会通过避免绘制那些完全不可见的组件来尽量减少消耗，但不幸的是，对于那些过于复杂的自定义View（通常重写了OnDraw方法），Android系统无法检测在OnDraw里面具体会执行什么操作，系统无法监控并自动优化，也就无法避免OverDraw了。但是我们可以通过canvas.clipRect()来帮助系统识别那些可见的区域，这个方法可以指定一块矩形区域，只有在这个区域内才会被绘制，其他的区域会被忽视，这个API可以很好的帮助那些有多组重叠组件的自定义View来控制显示的区域，同时clipRect方法还可以帮助节约CPU与GPU资源，在clipRect区域之外的绘制指令都不会被执行，那些部分内容在矩形区域内的组件，任然会得到绘制。除了clipRect方法之外，我们还可以使用canvas.quickReject()来判断是否没和某个矩形相交，从而跳过那些非矩形区域内的绘制操作。

### 九、善用draw9patch

给ImageView加一个边框，遇到这种需求，通常在ImageView后面设置一张背景图，露出边框便完美解决问题，此时这个ImageView，设置了两次drawable，底下一层仅仅是为了作为图片的边框而已，但是两层drawable的重叠区域却绘制了两次，导致OverDraw。

优化方案：将背景drawable制作成draw9patch，并且将和前景重叠的部分设置为透明，由于Android的2D渲染器会优化draw9patch中的透明区域，从而优化了这次OverDraw。但是背景图片必须制作成draw9patch才行，因为Android 2D渲染器只对draw9patch有这个优化，否则，一张普通的png，就算你把中间的部分设置成透明，也不会减少这次OverDraw.

### 十、慎用Alpha

假如对一个View做Alpha转化，需要先将View绘制出来，然后做Alpha转化，最后将转换的效果在界面上。通俗得说，做Alpha转化就需要对当前View绘制两遍，可想而知，绘制效率会的大打折扣，耗时会翻倍，所以Alpha还是慎用。如果一定要做Alpha转化的话，可以采用缓存的方式。

	view.setLayerType(LAYER_TYPE_HARDWARE);
	doSmoeThing();
	view.setLayerType(LAYER_TYPE_NONE);

通过setLayerType方式可以将当前界面缓存在GPU中，这样不需要每次绘制原始界面，但是GPU内存是相当宝贵的，所以用完要马上释放掉。

### 十一、应该早点知道的API

1. android：lineSpacingExtra
   
	设置行间距，如“3dp”

2. android：lineSpacingMultiPlier

	设置行间距的倍数，如“1.2”

3. android:includeFontPadding="false"

	TextView默认上下是有一定的padding的，有时候我们可能不需要上下这部分留白，加上它即可

4. android:titleMode（BitmapDrawable）

	可以指定图片使用重复填充的模式

5. android:fillViewport

	设置ScrollView撑满父容器

6. android:clipChildren

	定义它的子控件是否在他应有的边界内进行绘制，默认情况为true，也就是不允许进行扩展绘制

7. android:clipToPadding

	定义ViewGroup是否允许在padding中绘制，默认情况为true，也就是把padding中的值都进行裁切。该属性一般配合padding使用

8. ArgbEvaluator类

	实现丰富的色彩效果，提高体验度。譬如：滑动Viewpager时，背景色渐变；随着EditText输入框的长度变化背景色等等

### 十二、其他

- 尽量避免过多的使用static变量
- Avtivity和Activity之间或Fragment和Fragment之间使用Intent、Bundle传递数据
- 常量提取到一个单独的类中，并注意命名规范。如Intent传值的key
- 善用 Gradle

有时候我们的App会把HOST_URL、DEBUG等常量写在Constants中，这样我们在打正式包或测试包除了要修改代码外，studio还需要重新build一次，是比较耗时的，所以我们可以把一些常量的设置可以定义到build.gradle文件中

```gradle

    buildTypes {
        debug {
            buildConfigField("String", "HOST_URL", "\"http://api-test.ganxin.com\"")
            buildConfigField("Boolean", "LOG_DEBUG", "true")
        }
        release {
            buildConfigField("String", "HOST_URL", "\"http://api.ganxin.com\"")
            buildConfigField("Boolean", "LOG_DEBUG", "false")
        }
    }

```

又或者在AndroidManifest文件中的元素在不同的场景下需要替换不同的值的，可以尝试用productFlavors，如 ：

```gradle

    productFlavors {
        productive{
            manifestPlaceholders =[
                  LAUCHER_LOGO:"@mipmap/ic_logo_normal" ,
                  CHANNEL_NAME : "normal"
            ]
        }

        customA{
            manifestPlaceholders =[
                    LAUCHER_LOGO:"@mipmap/ic_logo_custom" ,
                    CHANNEL_NAME : "customA"
            ]
        }
    }

```

在AndroidManifest文件中的引用方式：

	android:icon="${LAUCHER_LOGO}"

    <meta-data
        android:name="UMENG_CHANNEL"
        android:value="${CHANNEL_NAME}" />

- 使用递归方法的时候避免造成OOM（内存溢出）
- 注意使用Handler造成的内存泄漏

一段简单的使用Handler示例

```Android

Handler mHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
        mImageView.setImageBitmap(mBitmap);
    }
}

```

a. 当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用（不然怎么可能通过Handler来操作Activity中的View ？）。而Handler通常会伴随一个耗时的后台线程（例如网络拉取图片）一起出现，这个后台线程在任务执行完毕之后，通过消息机制通知Handler，然后Handler把图片更新到界面，然后用户在网络请求过程中关闭了Activity，在正常情况下，Activity不再被使用，它就可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用，这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄漏），直到网络请求结束。

b.另外，如果执行了Handler的postDelayed()方法，该方法会将你的Handler装入一个Message，并把这条Message推到MesageQueue中，那么在你设定的delay到达之前，会有一条MessageQueue->Message->Handler->Activity的链，导致你的Activity被引用而无法被回收。

---
解决方案：

a. 通过程序逻辑来进行保护

	- 在关闭Activity的时候停掉你的后台线程。线程停掉了，就相当于切断了Handler和外部连接的线，Activity自然会在合适的时候被回收
	- 如果你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息对象移除就行
	
	关于Handler.remove方法：
	removeCallbacks(Runnable r) —— 清除r匹配上的Message
	removeCallbacks(Runnable r, Object token) —— 清除r匹配且匹配token(Message.obj)的Message，token为空时，只匹配r 
	removeCallbacksAndMessage(Object token) —— 清除token匹配的Message
	removeMessages(int what) —— 按what来匹配
	removeMessages(int what,Object object) —— 按what来匹配

	如果需要清除以Handler为target的所有Message（包括Callback），调用如下方法即可：
	handler.removeCallbacksAndMessages(null);

b. 将Handler声明为静态类

静态类不持有外部类的对象，所以Activity可以随意回收。

```Java

static class MyHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        mImageView.setImageBitmap(mBitmap);
    }
}

```

但使用了以上代码后，会发现，由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity的对象了，所以你需要在Handler中增加一个对Activity的弱引用（WeakReference）:

```Java

static class MyHandler extends Handler {
    WeakReference<Activity > mActivityReference;

    MyHandler(Activity activity) {
        mActivityReference= new WeakReference<Activity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        final Activity activity = mActivityReference.get();
        if (activity != null) {
            mImageView.setImageBitmap(mBitmap);
        }
    }
}

```