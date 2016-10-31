```toml
title = "ButterKnife框架使用"
slug = "about-use-butter-knife"
desc = "about use butter knife"
date = "2016-10-31 16:52:33"
update_date = "2016-10-31 16:52:33"
author = "wangganxin"
thumb = ""
tags = ["ButterKnife"]
```

[ButterKnife](https://github.com/JakeWharton/butterknife)(黄油刀)是Android上的一款依赖注入框架，目前已经是8.4版本了，迭代还是蛮快的，最近准备在自己的App上拿来练练手，浏览了网上的一些资料，大多比较陈旧，于是自己稍微归纳了下：

###一、使用姿势

#####step 1 :
在项目的build.gradle文件中，添加“android-apt”插件

	buildscript {
  	repositories {
    	mavenCentral()  //studio一般默认为jcenter(),如果无法加载则添加此句
   	}
  	dependencies {
    	classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
  	}
	}

#####step 2 :
在moudle下的build.gradle文件中 应用“android-apt”插件，并且添加ButterKnife依赖

	apply plugin: 'android-apt'

	android {
  	...
	}
	
	dependencies {
  	compile 'com.jakewharton:butterknife:8.4.0'
  	apt 'com.jakewharton:butterknife-compiler:8.4.0'
	}

#####step 3 :
最后，在Activity中使用

	class ExampleActivity extends Activity {
  	@BindView(R.id.user) EditText username;
  	@BindView(R.id.pass) EditText password;

  	@BindString(R.string.login_error) String loginErrorMessage;

  	@OnClick(R.id.submit) void submit() {
    	// TODO call server...
  	}

  	@Override public void onCreate(Bundle savedInstanceState) {
    	super.onCreate(savedInstanceState);
    	setContentView(R.layout.simple_activity);
    	ButterKnife.bind(this);
    	// TODO Use fields...
  	}
	}

当然除了在Activity中使用之外，你可以提供其他的View Root,来获取对象(执行注入)，如Fragment

<!--more-->

这里有一点需要注意，在ButterKnife 以前的版本中，我们在setContentView之后需要调用`ButterKnife.bind(this)`，同时也要在onDestory中调用`ButterKnife.unbind(this)`来进行解除绑定，现如今新版的Api中已经去除了unbind方法了

#####关于android-butterknife-zelezny
[butterknife-zelezny](https://github.com/avast/android-butterknife-zelezny)是Android Studio/IDEA上的使用ButterKnife框架时的一款插件，可以更方便快速地生成注解代码，具体使用方式可以前往github查看，这里不在赘述。

###二、需要注意的点

1、Activity `ButterKnife.bind(this)`必须在setContentView();之后，且**父类bind绑定后，子类不需要再bind**

2、Fragment `ButterKnife.bind(this, mRootView)`

3、属性布局不能用private or static 修饰，否则会报错

4、setContentView()不能通过注解实现。

5、ButterKnife已经更新到版本8.4，以前的版本中叫做@InjectView、@Bind，而现在改用叫@BindView，更加贴合语义。

6、ButterKnife不能再在library module中使用,这是因为你的library中的R字段的id值不是final类型的，但是你自己的应用module中却是final类型的。针对这个问题，有人在Jack的github上issue过这个问题，他本人也做了回答，[点击这里](https://github.com/JakeWharton/butterknife/issues/100)

###最后
目前还是刚刚熟悉使用阶段，等以后碰到其他问题了，再继续补充吧。
