```toml
title = "Android中的构建者（Builder）模式"
slug = "design-pattern-for-builder"
desc = "design pattern for builder"
date = "2016-11-07 18:03:41"
update_date = "2016-11-07 18:03:41"
author = "wangganxin"
thumb = ""
tags = ["Builder","设计模式"]
```

最近在使用[Retrofit](https://github.com/square/retrofit)和[OkHttp](https://github.com/square/okhttp)框架的过程中发现创建相关对象时频繁使用到了Builder模式，链式调用的方式让代码变得简洁、易懂，但自己也只是知其然而不知其所以然，所以决定做个笔记加深下印象。

### 一、场景分析

在实际开发中，往往会遇到需要构建一个复杂的对象的代码，像这样的：


```Java

public class User {

    private String name;            // 必传
    private String cardID;          // 必传
    private int age;                // 可选
    private String address;         // 可选
}

```

于是我们起手就是撸了一串这样的代码，譬如：

通过构造函数的参数形式去写一个实现类

```Java

User(String name);

User(String name, String cardID);

User(String name, String cardID,int age);

User(String name, String cardID,int age, String address);

```

又或者通过设置setter和getter方法的形式写一个实现类

<!--more-->

```Java

public class User {
    private String name;            // 必传
    private String cardID;          // 必传
    private int age;                // 可选
    private String address;         // 可选

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getCardID() {
        return cardID;
    }

    public void setCardID(String cardID) {
        this.cardID = cardID;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }
}

```

先说说这两种方式的优劣：

第一种在参数不多的情况下，是比较方便快捷的，一旦参数多了，代码可读性大大降低，并且难以维护，对调用者来说也造成一定困惑；

第二种可读性不错，也易于维护，但是这样子做对象会产生不一致的状态，当你想要传入全部参数的时候，你必需将所有的setXX方法调用完成之后才行。然而一部分的调用者看到了这个对象后，以为这个对象已经创建完毕，就直接使用了，其实User对象并没有创建完成，另外，这个User对象也是可变的，不可变类所有好处都不复存在。

写到这里真想为自己最近封装的表单控件捏一把汗。。。所以有没有更好地方式去实现它呢，那就是接下来要理解的Builder模式了。

### 二、定义
		将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的展示。

Builder模式属于创建型，一步一步将一个复杂对象创建出来，允许用户在不知道内部构建细节的情况下，可以更精细地控制对象的构造流程。

### 三、Builder模式变种-链式调用

####代码实现

```Java

public class User {
    private final String name;         //必选
    private final String cardID;       //必选
    private final int age;             //可选
    private final String address;      //可选
    private final String phone;        //可选

    private User(UserBuilder userBuilder){
        this.name=userBuilder.name;
        this.cardID=userBuilder.cardID;
        this.age=userBuilder.age;
        this.address=userBuilder.address;
        this.phone=userBuilder.phone;
    }

    public String getName() {
        return name;
    }

    public String getCardID() {
        return cardID;
    }

    public int getAge() {
        return age;
    }

    public String getAddress() {
        return address;
    }

    public String getPhone() {
        return phone;
    }

    public static class UserBuilder{
        private final String name;
        private final String cardID;
        private int age;
        private String address;
        private String phone;

        public UserBuilder(String name,String cardID){
            this.name=name;
            this.cardID=cardID;
        }

        public UserBuilder age(int age){
            this.age=age;
            return this;
        }

        public UserBuilder address(String address){
            this.address=address;
            return this;
        }

        public UserBuilder phone(String phone){
            this.phone=phone;
            return this;
        }

        public User build(){
            return new User(this);
        }
    }
}

```

需要注意的点：

- User类的构造方法是私有的，调用者不能直接创建User对象。
- User类的属性都是不可变的。所有的属性都添加了final修饰符，并且在构造方法中设置了值。并且，对外只提供getters方法。
- Builder的内部类构造方法中只接收必传的参数，并且该必传的参数使用了final修饰符。

####调用方式

```Java

        new User.UserBuilder("Jack","10086")
                .age(25)
                .address("GuangZhou")
                .phone("13800138000")
                .build();

```

相比起前面通过构造函数和setter/getter方法两种方式,可读性更强。唯一可能存在的问题就是会产生多余的Builder对象，消耗内存。然而大多数情况下我们的Builder内部类使用的是静态修饰的(static)，所以这个问题也没多大关系。

####关于线程安全

Builder模式是非线程安全的，如果要在Builder内部类中检查一个参数的合法性，必需要在对象创建完成之后再检查，

正确示例：

```Java

public User build() {
  User user = new user(this);
  if (user.getAge() > 120) {
    throw new IllegalStateException(“Age out of range”); // 线程安全
  }
  return user;
}

```

错误示例：

```Java

public User build() {
  if (age > 120) {
    throw new IllegalStateException(“Age out of range”); // 非线程安全
  }
  return new User(this);
}

```

### 四、经典Builder模式

####UML类图

![uml-builder](/media/2016/uml-for-builder-design-pattern.png)

(来源自《Android源码设计模式与解析实战》)

Product : 产品抽象类

Builder : 抽象Builder类，规范产品组建，一般是由子类实现具体的组建过程

ConcreteBuilder : 具体的Builder类

Director : 统一组装过程

####Product角色

```Java

/**
 * 用户抽象类
 */
public abstract class User {
    protected String name;
    protected String cardID;
    protected int age;
    protected String address;

    public void setName(String name) {
        this.name = name;
    }

    public abstract void setCardID();

    public void setAge(int age) {
        this.age = age;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    @Override
    public String toString() {
        return "User [name ="+name+",cardID="+cardID+",age="+age+"," +
                "address="+address+"]";
    }
}

```

####具体的Product类

```Java

/**
 * 具体的Product角色 SysUser
 */
public class SysUser extends User {
    public SysUser() {

    }

    @Override
    public void setCardID() {
        cardID="10086"; //设置默认ID
    }
}

```


####抽象Builder类

```Java

public abstract class Builder {
    public abstract void buildName(String name);
    public abstract void buildCardID();
    public abstract void buildAge(int age);
    public abstract void buildAddress(String address);
    public abstract User create();
}

```

####具体的Builder类

```Java

public class AccountBuilder extends Builder{

    private User user=new SysUser();

    @Override
    public void buildName(String name) {
        user.setName(name);
    }

    @Override
    public void buildCardID() {
        user.setCardID();
    }

    @Override
    public void buildAge(int age) {
        user.setAge(age);
    }

    @Override
    public void buildAddress(String address) {
        user.setAddress(address);
    }

    @Override
    public User create() {
        return user;
    }
}

```

####Director角色，负责构造User

```Java

public class Director {
    Builder mBuilder =null;

    public Director(Builder builder){
        this.mBuilder =builder;
    }

    public void construct(String name,int age,String address){
        mBuilder.buildName(name);
        mBuilder.buildCardID();
        mBuilder.buildAge(age);
        mBuilder.buildAddress(address);
    }
}

```

####测试代码

```Java

public class Test{
    public static void main(String args){
        //构建器
        Builder builder=new AccountBuilder();
        //Director
        Director director=new Director(builder);
        //封装构建过程：Jack，10086,25,GuangZhou
        director.construct("Jack",25,"GuangZhou");
        //打印结果
        System.out.println("Info :" +builder.create().toString());
    }
}

```

####输出结果

		System.out: Info :User [name =Jack,cardID=10086,age=25,address=GuangZhou]


### 六、用到Builder模式的例子

- Android中的AlertDialog.Builder

```Java

private void showDialog(){
        AlertDialog.Builder builder=new AlertDialog.Builder(context);
        builder.setIcon(R.drawable.icon);
        builder.setTitle("Title");
        builder.setMessage("Message");
        builder.setPositiveButton("Button1", new DialogInterface.OnClickListener() {
            @Override
            public void onClick(DialogInterface dialog, int which) {
                //TODO
            }
        });
        builder.setNegativeButton("Button2", new DialogInterface.OnClickListener() {
            @Override
            public void onClick(DialogInterface dialog, int which) {
                //TODO
            }
        });
        
        builder.create().show();
}

```

- OkHttp中OkHttpClient的创建

```Java

OkHttpClient  okHttpClient = new OkHttpClient.Builder()
                 .cache(getCache()) 
                 .addInterceptor(new HttpCacheInterceptor())
                 .addInterceptor(new LogInterceptor())
                 .addNetworkInterceptor(new HttpRequestInterceptor()) 
                 .build();

```

- Retrofit中Retrofit对象的创建

```Java

Retrofit retrofit = new Retrofit.Builder()
         .client(createOkHttp())
         .addConverterFactory(GsonConverterFactory.create())
         .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
         .baseUrl(BASE_URL)
         .build();
```

可见在实际使用中，均省略掉了Director角色，在很多框架源码中，涉及到Builder模式时，大多都不是经典GOF的Builder模式，而是选择了结构更加简单的后者。

### 七、优缺点

优点：

- 良好的封装性，使得客户端不需要知道产品内部实现的细节
- 建造者独立，扩展性强

缺点：

- 产生多余的Builder对象、Director对象，消耗内存

### 参考资料

- [《Android源码设计模式与解析实战》](http://product.dangdang.com/23802445.html)
- [设计模式之Builder模式](http://www.jianshu.com/p/e2a2fe3555b9)