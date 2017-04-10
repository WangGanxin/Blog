```toml
title = "App公共组件：加载数据Layout，高效开发必备！"
slug = "05-loaddatalayout"
desc = "05 loaddatalayout"
date = "2017-04-10 13:55:53"
update_date = "2017-04-10 13:55:53"
author = "wangganxin"
thumb = ""
tags = ["LoadDataLayout"]
```

>目录<br>
>
>一、前言<br>
>二、使用姿势<br>
>三、一些说明<br>

###一、前言

前不久在简书上看到一篇文章 [《直接拿去用！每个App都会用到的LoadingLayout》](http://www.jianshu.com/p/60a8b73a00f6) ，项目中经常会遇到几种页面：加载中、无网络、无数据、出错四种情况，传统的方式是通过include相关的布局，逐个分情况设置显示或隐藏，这样繁琐的过程一直是个痛点，于是参考了 `Weavey` 的封装套路，自己重新写了一套，进行了一些优化和扩展，虽然说原理不会太复杂（继承FrameLayout，XML渲染完成后，加上四个页面，然后按需控制显示哪一层），但若把它分离出来确实是提升开发效率的一个利器。

下面先来看看效果图：

![demo](/media/2017/loaddatalayout.gif)

<!--more-->

###二、使用姿势

#####step 1：

懒人版：

在你项目的build.gradle文件中添加依赖即可

> compile 'com.ganxin.library:loaddatalayout:1.0.1'

喜欢折腾的：

GitHub源码：[https://github.com/WangGanxin/LoadDataLayout](https://github.com/WangGanxin/LoadDataLayout)

下载源码，爱怎么玩就怎么玩，自由定制度高。

#####step 2：

在需要用到的布局中引入相关的Layout：

普通样式：LoadDataLayout

```XML

<com.ganxin.library.LoadDataLayout
    android:id="@+id/loadDataLayout"
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@color/blue"
        android:gravity="center"
        android:text="contentView"
        android:textColor="@color/white"
        android:textSize="20sp"/>

</com.ganxin.library.LoadDataLayout>

```

仿知乎样式：SwipeLoadDataLayout

```XML

<com.ganxin.library.SwipeLoadDataLayout
    android:id="@+id/swipeLoadDataLayout"
    style="@style/common_swipeLoadDataLayout"
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@color/blue"
        android:gravity="center"
        android:text="contentView"
        android:textColor="@color/white"
        android:textSize="20sp"/>

</com.ganxin.library.SwipeLoadDataLayout>

```

支持的自定义属性：

> loadingText：加载页面显示的文本，可选，仅LoadDataLayout可用
>  
> loadingTextSize:加载页面显示的图片，可选，仅LoadDataLayout可用
> 
> loadingTextColor:加载页面文本的颜色，可选，仅LoadDataLayout可用
> 
> loadingViewLayoutId:自定义加载页面的布局Id，可选，仅LoadDataLayout可用，如果设置了该属性，上面的三个属性失效
> 
> emptyText：无数据页面显示的文本，可选
> 
> emptyImgId：无数据页面显示的图片，可选
> 
> emptyImageVisible：无数据页面图片的可见性，可选，默认为true
> 
> errorText:出错页面显示的文本，可选
> 
> errorImgId:出错页面显示的图片，可选
> 
> errorImageVisible:出错页面图片的可见性，可选，默认为true
> 
> noNetWorkText:无网络页面显示的文本，可选
> 
> noNetWorkImgId:无网络页面显示的图片，可选
> 
> noNetWorkImageVisible:无网络页面图片的可见性，可选，默认为true
> 
> allTipTextColor:无数据、无网络、出错三个页面的文本颜色，可选
> 
> allTipTextSize:无数据、无网络、出错三个页面的文本字体大小，可选
> 
> allPageBackgroundColor:四个页面的背景色，可选
> 
> reloadBtnText:重新加载按钮的文本，可选
> 
> reloadBtnTextSize:重新加载按钮的文本字体大小，可选
> 
> reloadBtnTextColor:重新加载按钮的文本颜色，可选
> 
> reloadBtnBackgroundResource:重新加载按钮的背景资源，可选
> 
> reloadBtnVisible:重新加载按钮的可见性，可选，默认为true
> 
> reloadClickArea:重新加载按钮的点击区域，full代表整个页面，button代表仅button大小区域，默认button
> 


除了在XML设置属性外，支持全局配置，对所有使用的地方起作用，需要在Application中配置，示例：

```Java

public class App extends Application {

    @Override
    public void onCreate() {
        super.onCreate();

        LoadDataLayout.getBuilder()
                .setLoadingText(getString(R.string.custom_loading_text))
                .setLoadingTextSize(16)
                .setLoadingTextColor(R.color.colorPrimary)
                //.setLoadingViewLayoutId(R.layout.custom_loading_view) //如果设置了自定义loading视图,上面三个方法失效
                .setEmptyImgId(R.drawable.ic_empty)
                .setErrorImgId(R.drawable.ic_error)
                .setNoNetWorkImgId(R.drawable.ic_no_network)
                .setEmptyImageVisible(true)
                .setErrorImageVisible(true)
                .setNoNetWorkImageVisible(true)
                .setEmptyText(getString(R.string.custom_empty_text))
                .setErrorText(getString(R.string.custom_error_text))
                .setNoNetWorkText(getString(R.string.custom_nonetwork_text))
                .setAllTipTextSize(16)
                .setAllTipTextColor(R.color.text_color_child)
                .setAllPageBackgroundColor(R.color.pageBackground)
                .setReloadBtnText(getString(R.string.custom_reloadBtn_text))
                .setReloadBtnTextSize(16)
                .setReloadBtnTextColor(R.color.colorPrimary)
                .setReloadBtnBackgroundResource(R.drawable.selector_btn_normal)
                .setReloadBtnVisible(true)
                .setReloadClickArea(LoadDataLayout.FULL);
    }
}


```

为了适应某个界面的“特殊需求”，也支持局部设置各种属性，仅对当前对象生效，不影响全局，示例：

```Java

        loadDataLayout.setEmptyText("暂无数据显示")
                .setErrorText("出错啦")
                .setErrorImage(R.drawable.ic_error)
                .setAllTipTextSize(20)
                .setReloadBtnText("点我重新加载")
                .setReloadClickArea(LoadDataLayout.FULL);
                //and so on...

```

#####step 3：

为ReloadButton设置监听：

```Java

       loadDataLayout.setOnReloadListener(new LoadDataLayout.OnReloadListener() {
            @Override
            public void onReload(View v, int status) {

            }
        });

```

根据实际需要，设置显示的页面：

```Java

        loadDataLayout.setStatus(LoadDataLayout.LOADING); //加载中
        loadDataLayout.setStatus(LoadDataLayout.EMPTY); //无数据
        loadDataLayout.setStatus(LoadDataLayout.ERROR); //出错
        loadDataLayout.setStatus(LoadDataLayout.NO_NETWORK); //无网络
        loadDataLayout.setStatus(LoadDataLayout.SUCCESS); //加载成功

```

###三、一些说明

- LoadDataLayout默认一开始是隐藏contengView的，而SwipeLoadDataLayout中设置status为Loading时，contentView是显示的。
- LoadDataLayout和SwipeLoadDataLayout只能有一个直属子View，类似ScrollView，添加两个直属子View会抛异常：throw new IllegalStateException
