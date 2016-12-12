```toml
title = "Android性能优化摘录"
slug = "05-android-performance-optimization"
desc = "android performance optimization"
date = "2016-12-12 16:32:31"
update_date = "2016-12-12 16:32:31"
author = "wangganxin"
thumb = ""
tags = ["性能优化"]
```

本文是[有心课堂-性能优化合辑](http://stay4it.com/course/26) 视频的学习笔记。

### View的过度绘制（OverDraw）

OverDraw，是指在一帧的时间内（16.67ms）像素被绘制了多次，理论上一个像素只绘制一次是最优的，但由于重叠的布局导致一些像素被重复绘制多次，而每次绘制都会对应到CPU的一组绘图命令和GPU的一些操作，当这个操作超过16.67ms时就会出现掉帧的现象，即我们常说的卡顿，所以对重叠不可见元素的重复绘制会产生额外的开销，我们需要尽量减少OverDraw的发生。

Android提供了测量OverDraw的选项，在开发者选项->调试GPU过度绘制（Show GPU OverDraw）,打开该选项就可以看到当前页面OverDraw的状态。

- 没有颜色 ： 没有OverDraw。像素只画了一次。
- 蓝色：OverDraw 1倍。像素绘制了两次，大片的蓝色是可以接受的（若整个窗口都是蓝色，可以摆脱一层）。
- 绿色：OverDraw 2倍。像素绘制了三次，中等大小的绿色区域是可以接受的，但你应该尝试去优化、减少它们。
- 浅红：OverDraw 3倍。像素绘制了四次，小范围可接受。
- 暗红：OverDraw 4倍。像素绘制了五次或更多，这是错误的，要修复它们。

![GPU-OverDraw](/media/2016/gpu-overdraw.png)

### View的绘制流程

- measure :为整个View树计算实际的大小，即设置实际的高（对应属性：mMeasuredHeight）和宽(对应属性：mMeasureWidth)，每个View的控件的实际宽高都是由父视图和本身的视图决定的。
- layout：为将整个根据子视图的大小以及布局参数将View树放到合适的位置上。
- draw：利用前两步得到的参数，将视图显示在屏幕上。

### 三种常见布局的比较

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

### RecyclerView VS ListView 之View层级关系

		
					 | --- RecyclerView
		ViewGroup----|                                    |--- ListView
					 | --- AdapterView --- AbsListView ---|
														  |--- GridView

从View层级关系，我们可以看出，在实际开发中更推荐使用RecyclerView。


### 高效布局标签

Merge标签：

减少视图的层级结构。Merage标签一般配合FrameLayout和LinearLayout使用，当然RelativeLayout也可以用Merge标签，只是RelativeLayout本身比较复杂，如果我们也通过Merge标签去添加一些子View的时候很容易弄巧成拙。

ViewStub标签：
高效占位符。

Space标签：
空白控件。

### 去掉window的背景

在我们使用了Android自带的一些主题时，window会被默认添加一个纯色的背景，这个背景是被DecorView持有的，当我们的自定义布局时又添加一张背景图或者设置背景色，那么DecorView的background此时对我们来说是无用的，但是它会产生一次OverDraw，带来绘制性能损耗。

去掉window的背景可以在onCreate中的setContentView之后调用

		getWindow().setBackgroundDrawable(null);

或者在theme中添加

		android:windowbackground="null";

### 去掉其他不必要的背景

- 有时候为了方便会先给Layout设置一个整体的背景，再给子View设置背景，会造成重叠，如果子View宽度match_parent，可以看到完全覆盖了Layout的一部分，这里就可以通过分别设置背景来减少重绘
- 如果采用的是selector的背景，

### 应该早点知道的API

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

6. ArgbEvaluator类

	实现丰富的色彩效果，提高体验度。譬如：滑动Viewpager时，背景色渐变；随着EditText输入框的长度变化背景色等等


