```toml
title = "Android多渠道打包"
slug = "06-multi-channel-packaging"
desc = "multi channel packaging"
date = "2016-12-26 15:37:51"
update_date = "2016-12-26 15:37:51"
author = "wangganxin"
thumb = ""
tags = ["多渠道打包"]
```

>目录<br>
>一、Python打包及优化（美团多渠道打包）<br>
>二、Gradle打包<br>
>三、其他打包方案：修改Zip文件的comment<br>
>参考阅读<br>


早几个月前在有心课堂看过Android多渠道打包的视频，觉得蛮有用的，时至今日却发现不少具体的细节已经忘得差不多，于是重新整理了一篇笔记，顺道分享给大家。

### 一、Python打包及优化（美团多渠道打包）

既然是Python打包，那么python环境是必须的，否则无法运行python脚本文件，mac系统下默认安装了Python环境，而Windows系统下则需要自己安装了，这个安装过程相对可以简单，大家可以自行谷歌下，记得配好环境变量。验证是否安装成功的方式是，打开命令行，输入python,如果安装成功的话，会打印python的版本信息。

下面我们以[友盟应用统计](http://dev.umeng.com/analytics/android-doc/integration)为例，进行相关的操作。

#####需要准备的东西：

- 签名好的Apk文件 
- channel.py (python脚本文件
- channel列表文件 （注：必须命名为android_channels.txt，如果不想这样命名，可自行更改python脚本代码
- ChannelUtil.java工具类 

<!--more-->

其中channel列表文件（android_channels.txt）的格式为每一个渠道号换一行，示例：

```Txt

xiaomi
360mobile
wandoujia
baidu

```

另外python脚本文件、ChannelUtil.java这里就不贴代码了，有点长，但还是比较简单容易理解的，大家可以自己下载下来看看，附上 ：[传送门](https://github.com/WangGanxin/Codebase/tree/master/app/src/main/java/com/ganxin/codebase/packaging) 

#####使用姿势：

首先，在签名打包一个Apk之前，需要在程序的入口处添加如下代码：

```Java

        String channel= ChannelUtil.getChannel(mAppContext);  //获取渠道号，内存>SharedPreferences>apk的/META-INF/目录
        MobclickAgent.startWithConfigure(new MobclickAgent.UMAnalyticsConfig(getApplicationContext(),umengAppkey,channel)); //友盟：通过代码的方式设置渠道号

```

生成好以后，将该apk文件和上面所准备的东西放在同一个文件夹，打开终端命令行，进入该文件夹。执行命令：

	python channel.py apk包名

很快就在当前目录下生成一个release文件夹，里面生成了各个渠道所需要的渠道包。

这一条简单的命令背后，到底隐藏着什么不为人知的流程呢？请看下图：

![Python-Workflow](/media/2016/python-packaging-workflow.png)

整理思路来看，就是通过python脚本去读取渠道列表，然后一个for循环将签名好的apk写入渠道号，并在当前目录下，生成一个release文件夹，将写入好的渠道包放进去，那么这个for循环内部究竟是怎样的一个操作呢？它分为以下几步：

1. 复制签名好的signatured.apk到./release文件夹下
2. 重命名signatured.spk 为 signatured_channel_xxx.apk
3. 找到apk/META-INF/目录
4. 新建一个文件名为 channel_xxx的空文件

这种通过python打包的方式有什么优点和缺点呢？

优点：

- 只需要一个签名好的apk
- 速度快（在渠道少的情况下，基本秒开，如果渠道更多，有100多个的话应该也不会超过一分钟）

缺点：

- 依赖Java的签名方式（现有的打包方式，在apk的META-INF目录下添加文件，是不需要重新签名的，但如果谷歌更改了这套签名方式，那么这种方法就不适用了）
- 必须支持只用Java代码写入渠道号

#####zipalign优化：

是否上述的这种打包方式就一定完美了呢？也许你会遇到这样的问题：

- 在Google Play上提交会失败（如果你的应用要上传到该市场的话
- Lollipop系统（Android 5.0.1）安装可能会提示解析安装包错误

所以介绍来要介绍另外一款工具——zipalign，官方的定义是这样子的：

	zipalign is an archive alignment tool that provides important optimization to Android application (APK) files

简单提炼为关键词就是：**优化工具**、**4字节边界对齐**、**减少内存使用**、**提高效率**

何时需要使用这个工具？

- apk签名之后（在开发过程中，更多的时候是eclipse、as自动帮我们使用了这个工具，不需要我们手动去使用它的
- 对apk进行添加或更改的时候

常用的命令：

- zipalign -c -v \<alignment\> existing.apk 
- zipalign [-f] [-v] \<alignment\> infile.apk outfile.apk

-c ：验证apk是否按照某种对齐方式对齐

-f ：覆盖已经存在的文件

-v ：输出verbose级别的信息

\<alignment\>  ：对齐方式，这里我们以4字节的方式对齐

existing.apk ：需要验证的apk文件

infile.apk  ：要打包的apk文件

outfile.apk ：要输出的apk文件


第一条命令是用来验证apk是否有进行过zipalign优化；第二条命令是用来进行zipalign优化


那么zipalign这个工具在哪里呢？

我们可以打开android sdk的安装目录下，在build-tools/版本号文件夹/找到一个zipalign.exe，其实zipalign是在android 1.6之后提供的一个工具，在使用之前我们最好可以把这个路径配置到环境变量里。

在使用的时候，我们通常先用第一条命令判断我们的apk文件是否已经进行过zipalign优化，接着再使用第二条命令是优化我们的apk文件，快捷命令：

首先是验证apk：  
	
	zipalign -c -v 4 demo_channel.apk

进行zipalign优化： 

	zipalign -f -v 4 demo_channel.apk demo_align.apk

有些同学可能会问，你这一个apk优化还好，那如果有很多个渠道包，我们应该怎么去优化呢？这里已经在刚刚的传送门里，给大家准备好了两个脚本文件，分别是mac系统下的zipalign_batch.sh和windows系统下的zipalign_batch.bat文件，并且已经集成到channel.py的python脚本当中，我们在使用的时候，可以根据自己的需要自行开启相关的代码即可：

```Python

    #mac
    #os.system('chmod u+x zipalign_batch.sh')
    #os.system('./zipalign_batch.sh')

    #windows
    #os.system('zipalign_batch.bat')

```

再次提醒下大家，在开启使用这段代码的前，要把zipalign工具配置到你系统的环境变量里，否则在运行python脚本过程中会提示相关的文件找不到哈。

### 二、Gradle打包

相信大部分开发者都已经迁移到AS下进行开发，那么利用gradle进行多渠道打包也是我们必须掌握的一个知识点，下面分别讲解下gradle的多渠道打包和多Apk打包。

#####多渠道打包
多渠道打包，大家应该都知道，这里不解释，同样以友盟统计为例，

第一步：

在AndroidManifest.xml文件中配置渠道ID，`${UMENG_CHANNEL_VALUE}` 为占位符，其中的`UMENG_CHANNEL_VALUE`可以自己任意定义

	<meta-data android:name="UMENG_CHANNEL" android:value="${UMENG_CHANNEL_VALUE}"/>

第二步：

在项目的build.gradle文件中设置打包签名信息 signingConfigs：

```Gradle

    android {
        debug {
            // No debug config
        }

        release {
            storeFile file("../yourapp.jks")
            storePassword "your password"
            keyAlias "your alias"
            keyPassword "your password"
        }
    }

```

接着，设置productFlavors，这里包含你了所需要的渠道号：

```Gradle

    android {
        productFlavors {
            xiaomi {
                manifestPlaceholders = [UMENG_CHANNEL_VALUE: "xiaomi"]
            }
            360 mobile {
                manifestPlaceholders = [UMENG_CHANNEL_VALUE: "360mobile"]
            }
            wandoujia {
                manifestPlaceholders = [UMENG_CHANNEL_VALUE: "wandoujia"]
            }
            baidu {
                manifestPlaceholders = [UMENG_CHANNEL_VALUE: "baidu"]
            }
        }
    }

```

也有另外一种简便的写法，其实就是用Groovy语法执行一个for循环：

```Gradle

    productFlavors {
        xiaomi {}
        360 mobile {}
        wandoujia {}
        baidu {}
    }

    productFlavors.all { flavor ->
        flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
    }

```

最后：

在AS的内置终端Terminal工具中执行命令： 

	./gradlew assembleRelease
	（所有生成的apk在项目的build\outputs\apk下）

好了，接下来可以静静等待打包完成。

如果不想使用命令行的方式进行打包，AS也为我们提供了图形界面的方式，点击菜单栏->Build->Generate Signed APK，输入相关的签名证书路径和密码即可：

![Build-Apk-1](/media/2016/build-apk-1.png)
![Build-Apk-2](/media/2016/build-apk-2.png)

这种打包方式的其中一个好处是可以指定打包后的apk所存放的路径。

#####多Apk打包
什么叫多Apk打包？可能很多人不知道，也没有碰到这样的需求，多Apk打包其实就是根据特定的需求，生成不同类型的apk，譬如说每个apk的有不同的应用名称、应用icon或cpu类型等等。

下面以生成不同的cup类型的apk为例，cup的类型大致可以分为分别是arm、mips、X86这三种：

同样地，我们在build.gradle设置这ProductFlavors即可:

```Gradle

    productFlavors {
        arm {
            ndk{
                abiFilters("armeabi","armeabi-v7a")
            }
        }
        mips {
            ndk{
                abiFilters("mips","mips86")
            }
        }
        x86 {
            ndk{
                abiFilters("x86","x86_64")
            }
        }
    }

```

这里要插句补充下defaultConfig 跟 productFlavors的关系，defaultConfig相当于一个默认的flavor，如果我们没有定义productFlavors，gradle在构建过程中就会只读取defaultConfig的配置信息；如果自定义了productFlavors，那么defaultConfig相当于每一个flavor的基础配置信息。如果flavor和defaultConfig的某些配置项相同，flavor的配置将会覆盖defaultConfig。

#####补充知识点

Gradle常用命令

>- ./gradlew -v ：版本号
>- ./gradlew clean ：清除项目/app目录下的build文件夹
>- ./gradlew build 检查依赖并编译打包
>
	需要注意的是 ./gradlew build 命令会debug、release环境的包都打出来，如果正式发布只需要打Release的包，可以使用assemble命令
- ./gradlew assembleDebug ：编译并打Debug包
- ./gradlew assembleRelease ：编译并打Release的包
- ./gradlew installRelease ：Release模式打包并安装
- ./gradlew uninstallRelease ：卸载Release模式包

>
>我们执行命令前，都会加上`./gradlew`，`./`代表当前目录，`gradlew`代表gradle wrapper，意思是gradle的一层包装，可以理解为在这个项目本地就封装了gradle，即gradle wrapper。在`项目名/gradle/wrapper/gralde-wrapper.properties`文件中声明了它指向的目录和版本.

关于assemble

>在上面的**assemble**命令中，会结合**Build Type**来创建自己的task，譬如：
>
- ./gradlew assembleDebug
- ./gradlew assembleRelease

>除此之外，**assemble**还能和**ProductFlavor** 结合创建新的任务，其实**assemble**是和**Build >Variants** 一起结合使用的，大家可以这么来理解：
>
>	Build Variants = Build Type + Product Flavor
>
>举个栗子：
>
>
>如果我们想打wandoujia渠道的Release版本，可以执行如下命令
>
>	./gradlew assembleWandoujiaRelease
>
>如果我们只打wandoujia渠道版本，则：
>
>	./gradlew assembleWandoujia
>	（此命令会生成wandoujia渠道的Release和Debug版本）
>
>同理，如果想打全部Release版本：
>
	./gradlew assembleRelease
	（这条命令会把Product Flavor下的所有渠道的Release版本都打出来）

>基于以上，总结一下**assemble**创建task有如下用法：
>
>- 允许直接构建一个Variant版本，例如 assembleFlavor1Debug
- 允许构建指定Build Type的所有APK，例如assembleDebug将会构建Flavor1Debug和Flavor2Debug两个Variant版本
- 允许构建指定flavor的所有APK，例如assembleFlavor1将会构建Flavor1Debug和Flavor1Release两个Variant版本

关于BuildVariants

>在AS中有个BuildVariants的概念，Build Variants = Build Type + Product Flavor，譬如如下代码：

```Gradle

android {
    productFlavors {
        xiaomi {}
        _360mobile {}
        wandoujia {}
        baidu {}
    }

    productFlavors.all { flavor ->
        flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
    }
}

```

>会生成以下的BuildVariants:

![AS-BuildVariants](/media/2016/as-buildvariants.png)

>我们就可以看到，productFlavors有多个维度，并且每个Flavor都有debug和release，所以总共生成Build Type\*ProductFlavor个不同的apk，也即2*4=8种不同的APK。


最后附上送一个完整的gradle脚本文件：[传送门](https://github.com/WangGanxin/Codebase/blob/master/app/build.gradle) 

### 三、其他打包方案：修改Zip文件的comment

核心原理：

>Android应用使用的APK文件就是一个带签名信息的ZIP文件，根据 [ZIP文件格式规范](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT) ，每个ZIP文件的最后都必须有一个叫[ Central Directory Record](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html) 的部分，这个CDR的最后部分叫"end of central directory record"，这一部分包含一些元数据，它的末尾是ZIP文件的注释。注释包含Comment Length和File Comment两个字段，前者表示注释内容的长度，后者是注释的内容，正确修改这一部分不会对ZIP文件造成破坏，利用这个字段，我们可以添加一些自定义的数据。

>所以原理很简单，就是**将渠道信息存放在APK文件的注释字段中**

优点：

- 没有解压缩、压缩、重签名过程，打包速度极快，单个包只需要5毫秒左右，甚至可用于网站后台动态生成渠道包

缺点：

- 没有使用Android的productFlavors，无法利用flavor条件编译的功能

相关工具：

- [packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin) 
- [MultiChannelPackageTool](https://github.com/seven456/MultiChannelPackageTool) 


### 参考阅读

- [有心课堂-Android多渠道打包](http://stay4it.com/course/13/reviews/) 
- [美团Android自动化之旅—生成渠道包](http://tech.meituan.com/mt-apk-packaging.html) 
- [ANDROID STUDIO系列教程六--GRADLE多渠道打包](http://stormzhang.com/devtools/2015/01/15/android-studio-tutorial6/) 
- [下一代Android打包工具，100个渠道包只需要10秒钟](https://github.com/mcxiaoke/packer-ng-plugin) 