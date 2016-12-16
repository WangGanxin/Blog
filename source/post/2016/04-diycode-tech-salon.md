```toml
title = "DiyCode技术沙龙"
slug = "diycode-tech-salon"
desc = "diycode tech salon"
date = "2016-12-06 10:00:31"
update_date = "2016-12-06 10:21:31"
author = "wangganxin"
thumb = ""
tags = ["diycode"]
```

>目录<br>
>一、内存管理优化<br>
>二、动态更新<br>
>三、hotfix<br>
>四、App安全<br>
>五、一些感受<br>

最近事情比较多，很遗憾这是一篇迟来的分享。

[DiyCode技术沙龙](http://www.diycode.cc/) 是由广州社区的成员合力举办的一场Android技术分享会，因为获得了一张免费的门票，加上分享的主题都比较感兴趣，所以去参加学习一下，总的来说干货满满，见识了很多的技术大牛。

![diycode-tech-salon](/media/2016/diycode-tech-salon.png)

### 一、内存管理优化

**Low Memory Killder(LMK)**

常见的进程优先级：

- 前台进程（Foreground Process）
- 可见进程（Visbile Process）
- 服务进程（Service Process）
- 后台进程（Background Process）
- 空进程（Empty Process）

<!--more-->

![process_priority](/media/2016/process_priority.png)

*oom_adj的值越高，优先级越低，越容易被系统回收*

**查看oom_adj**

在adb shell 中，输入如下两个命令即可：

- ps | grep 包名
- cat /proc/进程pid/oom_adj

示例：

		E:\asWorkSpace>adb shell
		shell@hnSCL-Q:/ $ ps | grep com.android.browser
		u0_a7     26894 353   1716624 138140 ffffffff 00000000 S com.android.browser
		u0_a7     26963 353   1538696 49280 ffffffff 00000000 S com.android.browser:service
		shell@hnSCL-Q:/ $ cat /proc/26963/oom_adj
		1
		shell@hnSCL-Q:/ $

*其中的26894和26963分别是浏览器两个进程的pid*

**StrictMode（严苛模式）**

参考资料：[Android性能调优利器StrictMode](http://droidyue.com/blog/2015/09/26/android-tuning-tool-strictmode/index.html)

什么是StrictMode ?

StrictMode（严苛模式）是用来检测程序中违例情况的一个开发者工具，最常用的场景是检测主线程中本地磁盘和网络读写等耗时操作。

能检测哪些？

严苛模式主要用来检测两个问题，一个是线程策略，即ThreadPolicy，另一个是VM策略，即VmPolicy。

ThreadPolicy

- 自定义的耗时调用，使用detectCustomSlowCalls()开启
- 磁盘读取操作，使用detectDiskReads()开启
- 磁盘写入操作，使用detectDiskWrites()开启
- 网络操作，使用detectNetwork()开启

VmPolicy

- Activity泄漏，使用detectActivityLeaks()开启
- 未关闭的Closable对象泄漏，使用detectLeakedClosableObjects()开启
- 泄漏的Sqlite对象，使用detectLeakedSqliteObjects()开启
- 检测实例数量，使用setClassInstanceLimit()开启

在哪里开启？

严苛模式的的开启可以放在Application或者Activity以及其他组件的onCreate方法。为了更好地分析应用中的问题，建议放在Application的onCreate方法。

简单的开启方式？

第一种，通过代码的方式：

```Java

if (IS_DEBUG && Build.VERSION.SDK_INT >= 9) {
    StrictMode.setThreadPolicy(new StrictMode.ThreadPolicy.Builder().detectAll().penaltyLog().build());
  StrictMode.setVmPolicy(new VmPolicy.Builder().detectAll().penaltyLog().build());
}

```

严苛模式需要在Debug模式开启，不要在Release版本中启用。同时，严苛模式自API19开始引入，某些方法也从API11开始引入，使用时需注意API级别。

第二种，通过在手机开发者选项中开启严苛模式，开启之后，如果主线程中有执行时间长的操作，屏幕会闪烁。

**常见问题及优化方案**

- 使用合适的Context，一般注册第三方框架或者SDK时采用Application的Context，除非特地要求传Activity的Context
- 多进程应用需要避免Appication onCreate多次执行引起的重复初始化
- 图片资源放对位置（优先考虑主流设备，优先考虑用户分布多的设备）
- 图片加载优化

		1.inSampleSize(降低采样率)
		2.BitmapRegionDecoder(加载超级大图)
		3.Matrix(小图放大)
		4.LruCache/LinkedHaspMap(缓存控制，避免oom)
        5.选择合适的图片加载框架（UIL、Fresco、Glide、Picasso）
		6.按需显示，优先显示缩略图，需要时显示大图
		7.优化加载图片的时机

- 主动释放内存

		关键函数： onTrimMemory(int level) ，这是4.0以后提供的一个API，系统提供的回调有
		- Application.onTrimMemory()
		- Activity.onTrimMemory()
		- Fragement.OnTrimMemory()
		- Service.onTrimMemory()
		- ContentProvider.OnTrimMemory()
		
		参数level代表不同的内存状态：
		- TRIM_MEMORY_COMPLETE：内存不足，并且该进程在后台进程列表最后一个，马上就要被清理
		- TRIM_MEMORY_MODERATE：内存不足，并且该进程在后台进程列表的中部
		- TRIM_MEMORY_BACKGROUND：内存不足，并且该进程是后台进程
		- TRIM_MEMORY_UI_HIDDEN：内存不足，并且该进程的UI已经不可见了
		以上4个是4.0新增
		- TRIM_MEMORY_RUNNING_CRITICAL：内存不足(后台进程不足3个)，并且该进程优先级比较高，需要清理内存
		- TRIM_MEMORY_RUNNING_LOW：内存不足(后台进程不足5个)，并且该进程优先级比较高，需要清理内存
		- TRIM_MEMORY_RUNNING_MODERATE：内存不足(后台进程超过5个)，并且该进程优先级比较高，需要清理内存
		以上3个是4.1新增

- 选择合适的时机，对View和资源解绑，在合适的时机再恢复
- 释放缓存
- 小心未关闭的Dialog，在Activity和Fragment销毁时记得先dismiss
- 匿名内部类和非静态内部类泄漏，用静态外部类和弱引用的方式取而代之
- 避免创建大量的临时对象而造成内存抖动，考虑复用

		- 高频执行函数中避免创建大量临时对象，如View的onDraw和onTouch等
		- StingBuilder、StringBuffer代替String
	
- WebView造成内存泄漏
		
		Android系统和各家的ROM本身存在的问题造成的，最好的处理方式是将承载WebView的界面独立到其他进程，在合适的时机选择杀死WebView进程

- 自身内存监控（[来自腾讯开发者提供的一种方案](http://dwz.cn/4LiOac)）

		- Runtime.getRuntime().getMaxMemory()
		Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory()

		- 定期检查上边的比例值，达到一定峰值时调用此API触发内存释放
		WindowManagerGlobal.getInstance().startTrimMemory(TRIM_MEMORY_COMPLETE);

- 更多

		- 注册和反注册（BroadcastReceiver、Observer）
		- 关闭资源（Cursor、IO流）
		- 使用优化过的数据结构SparseXXXX
		- 考虑使用Parcelable取代Serializable
		- 建立缓存池，如ListView复用View的思路
		- 开辟多进程（当然，也不是越多越好）
		- 借助lint来规避一些常见的编码问题

### 二、动态更新
随着App日益庞大以及越来越复杂的逻辑，不仅需要团队多人的协作开发还有方法数越界等问题，因此，插件化以此衍生出而来，它的好处有以下几个方面：

- 插件模块动态升级
- 改进大型App的架构
- 实现多团队协作开发，各自发布

因为对这一块接触了解的并不是很多，在此不作更详细的记录。


### 三、hotfix
所谓热补丁（hotfix），它可以让应用在无须重新安装下能够自动更新。对于热补丁的应用，可能我们会存在几个等级的需求：

1. **Bugfix**  : 简单的bugfix，解决一些空指针之类的问题
2. **UI**  : 改变UI，支持资源的变更
3. **features**  : 功能的发布，甚至一定情况下面代替升级

对于现在市面上的热修复方案，大致可以分为两个流派，第一个是native派，它肯定是使用native方式实现，第二个是Java派，如kkfix可以hook系统的方法，robust是AOP的方式。

![hotfix-type](/media/2016/hotfix-type.png)

那么问题又来了，热补丁技术哪家强？其实各个方案都有自己的优缺点，世上没有最好的方案，只有只适合自己的方案

- **AndFix方案**

这套方案主要是在Native层替换方法的实现，关键挑战是兼容性问题，底层的代码每个版本都有改动，可能有些厂商也会去改这块代码，所以这套方案的使用场景存在较大的限制

		优点：1、立刻生效               缺点：1、兼容性
             2、性能影响小                   2、Native定位复杂
             3、补丁相对较小                 3、应用场景首先
             4、支持大部分的加固场景

		总结: 适合Bugfix场景

- **QZone方案**

主要是利用了android classloader查找类的顺序，当出现重复类的时候，优先使用补丁中修改过的类。这套方案有个问题是dalvik在加载修改过的类的时候会出现一个crash，当时的解决方案是用插桩的方式。这套方案的特点是非常简单，而且兼容性不错。

		优点：1、兼容性较好               缺点：1、性能损耗
             2、应用场景广                    2、补丁可能会过大
             3、简单，成功率高                 3、无法立刻生效
             4、支持大部分的加固场景

		总结: 可实现Features发布

- **Tinker方案**

在审视大量方案后，发现gradle的instant run 和 facebook buck exopacakage方案都使用了全量合成的做法，于是想到了推送差异Dex的方式。

		优点：1、兼容性较好               缺点：1、Dalvik Rom体积较大
             2、应用场景广                    2、单独的合成过程
             3、补丁较小                      3、无法立刻生效
             4、性能损耗小                    4、不支持加固（通过回退Qzone方案）

		总结: 可实现Features发布

- **Robust方案**

关键挑战：super方法调用问题；Proguard内联

		优点：1、立即生效               缺点：1、应用场景受限
             2、性能影响小                   2、修改源码
             3、补丁较小                     3、安装包变大（5%左右）
             4、支持大部分加固场景                

		总结: 适合Bugfix场景


各种方案的对比：

![hotfix-compare](/media/2016/hotfix-compare.png)

### 四、App安全

**代码安全**

- 完整性校验：检查classex.dex、apk的完整性，最好将哈希值存储到服务器
- 防逆向分析：代码混淆、加壳保护
- 防进程注入：防止ptrace进行so注入，并hook任意函数

**传输安全**

- HTTPS=HTTP+SSL/TLS
- 苹果已宣布从2017年起所有IOS应用将强制使用HTTPS
- HTTP/2.0也只支持HTTPS
- 防止中间人攻击：SSL Pinning
- 防止降级攻击

关于URL签名的一般处理方式：

1. 将所有参数按参数名进行升序排序；
2. 将排序后的参数名和值拼接成字符串stringParams，格式：key1value1key2value2...；
3. 在上一步的字符串前面拼接上请求URL的Endpoint，字符串后面拼接上AppSecret，即stringUri+StringParams+AppSecret；
4. 使用AppSecret为密钥，对上一步的结果字符串使用HMAC算法计算MAC值，这个MAC值就是签名。

URL示例：

		http://api.domain.com/users/123?appKey=qwer&token=asfe&timestamp=23456789

		AppSecret:nmefj

        参数排序：appKey、timestamp、token

        参数拼接：appKeyqwertimestamp23456789tokenasfe

        URL拼接：users/123appKeyqwertimestamp23456789tokenasfenmefj

        计算MAC值：reRwV

        最终URL：http://api.domain.com/users/123?appKey=qwer&token=asfe&timestamp=23456789&sign=reRwV


**存储安全**

常用的存储方式：

- SharedPreferences
- Internal Storage
- External Storage
- SQlite Databases
- keystore/keychain
- 硬编码
- so文件

存储的安全标准：

- 敏感数据不能存在外部存储器上，无论是否加密
- 私有目录数据正确设置权限
- 铭感数据不能以明文存储在私有目录


**敏感数据安全**

密码：

- 不要在客户端本地保存
- 网络传输加密：MD5、加盐MD5、HMAC、AES、RSA
- 数据库保存：加盐MD5，盐值和MD5分开保存

密钥：

- 保存到SharedPreferences
- 硬编码到代码中
- 加密保存到配置文件中
- NDK开发，放在so文件中，加解密操作都在so文件，并添加签名认证

TOKEN

- 用户登录成功后返回Token,一般有两个：accessToken和和refreshToken
- Token需要设置有效期，accessToken的有效期不能设置太长，最好不超过一周，refreshToken可以长一点
- accessToken过期后，通过refreshToken更新accessToken
- refreshToken过期后，则需要用户重新登录

### 五、一些感受

额，做技术的人好像蛮容易秃顶的...
