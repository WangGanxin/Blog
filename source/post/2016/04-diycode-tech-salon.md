```toml
title = "DiyCode技术沙龙"
slug = "diycode-tech-salon"
desc = "diycode tech salon"
date = "2016-11-15 20:57:31"
update_date = "2016-11-28 20:57:31"
author = "wangganxin"
thumb = ""
tags = ["diycode"]
```

最近事情比较多，很遗憾这是一篇迟来的分享。

[DiyCode技术沙龙](http://www.diycode.cc/) 是由广州社区的成员合力举办的一场Android技术分享会，因为获得了一张免费的门票，加上分享的主题都比较感兴趣，所以去参加学习一下，总的来说干货满满，见识了很多的技术大牛。

![diycode-tech-salon](/media/2016/diycode-tech-salon.png)

### 内存管理优化


### 动态更新
随着App日益庞大以及越来越复杂的逻辑，不仅需要团队多人的协作开发还有方法数越界等问题，因此，插件化以此衍生出而来，它的好处有以下几个方面：

- 插件模块动态升级
- 改进大型App的架构
- 实现多团队协作开发，各自发布

因为对这一块接触了解的并不是很多，在此不作更详细的记录。


### hotfix
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

### App安全

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

密钥：

TOKEN




### 一些总结
