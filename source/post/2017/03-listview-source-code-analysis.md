```toml
title = "ListView源码流程图解与浅析"
slug = "03-listview-source-code-analysis"
desc = "03 listview source code analysis"
date = "2017-03-21 11:03:41"
update_date = "2017-03-21 11:03:41"
author = "wangganxin"
thumb = ""
tags = ["ListView"]
```

>目录<br>
>一、前言<br>
>二、RecycleBin回收机制<br>
>三、第一次Layout<br>
>四、第二次Layout<br>
>五、滑动加载更多<br>
>参考资料<br>

###一、前言

近期时间比较松散，决定研究下我们熟悉已久的ListView，作为菜鸟，自然是不敢独自摸索，Google了相关资料，紧跟大神的分析套路前行。记得任玉刚主席曾说过，看源码的时候我们应该侧重对流程的把握，刚开始时不要纠结太多的代码细节，否则容易深陷其中而无法自拔。

废话不多说，先看看ListView的继承结构图吧：

![listview_type_hierarchy](/media/2017/listview_type_hierarchy.png)

AbsListView有两个实现类，一个是ListView，另一个是GridView，所以ListView和GridView的工作原理和实现上有很多共同点的。

<!--more-->

###二、RecycleBin回收机制

RecycleBin是AbsListView的一个内部类，主要作用是对View的回收与重用。

其中两个变量：

- View[] mActiveViews：存放布局开始时活动在屏幕上的视图，从第一个到屏幕最后一个View，默认是第一个View
- ArrayList<View>[] mScrapViews ：用来存放适配器中转换过来的废弃的View，该ArrayList为无序

由此衍生出来几个比较重要的方法：

- fillActivityViews()：调用这个方法，就会根据传入的参数来将ListView中的指定元素存储到mActiveViews数组当中
- getActivityView()：用于从mActiveViews数组当中获取数据，mActivityViews中的View一旦被获取后，下次再获取同样位置的View将会返回null
- addScrapView()：将一个废弃的View进行缓存
- getScrapView()：从废弃缓存中取出一个View
- setViewTypeCount()：在Adapter中可以重写getViewTypeCount()来表示ListView中有几种类型的数据项，而此处的setViewTypeCount()作用就是为每种类型的数据项单独开一个缓存机制


###三、第一次Layout

ListView最终是继承于View，所以它的执行流程还是会分为三步，onMeasure测量View的大小，onLayout用于确定View的布局，onDraw用于将View绘制到界面上。
在ListView中，onMeasure通常是占用整个屏幕，onDraw的过程其实是交给了ListView中的子元素绘制，所以重点分析的地方在于onLayout。

![first-layout](/media/2017/first-layout.png)

###四、第二次Layout

第二次Layout的过程和第一次的基本流程类似，这里有一点区别需要说明的是，在第一次Layout过程的最后调用的是addViewInLayout方法，用于向ViewGroup中添加一个新的子View；第二次Layout过程的最后调用了attachViewToParent方法，该方法是将一个已经detach的View重新attach到ViewGroup上。

![second-layout](/media/2017/second-layout.png)

###五、滑动加载更多

ListView的滑动机制从它的父类AbsListView的onTouchEvent方法开始，传递和分发一系列的事件给子类ListView，在trackMotionScroll方法中调用了addScrapView方法（子View被移出屏幕，便加入到废弃缓存中），而在obtainView方法逻辑中调用了getScrapView方法，尝试从废弃缓存中获取View，这里形成了生产者和消费者模式，也正因为这样可以使ListView在滑动的过程中，不管有多少条数据需要显示，移出屏幕的View被缓存后，很快又从屏幕中重新利用起来，从而避免OOM，内存不会暴增的情况。

![scrolling](/media/2017/scrolling.png)

###参考资料

- ListView Android-23源码 
- [Android ListView工作原理完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/44996879) 
