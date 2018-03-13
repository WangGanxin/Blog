```toml
title = "【React Native填坑之旅】撸一个简易聊天表情组件-ChatUI"
slug = "01-react-native-chatui"
desc = "01 react native chatui"
date = "2018-03-13 15:00:00"
update_date = "2018-03-13 15:00:00"
author = "wangganxin"
thumb = ""
tags = ["React Native","ChatUI"]
```

>目录<br>
>
>一、需求<br>
>二、实现思路<br>
>
>参考资料<br>

### 一、需求
笔者是做直播类App的，近期项目准备用React Native对直播间进行改造，其中涉及一个基础的功能点就是在直播间中点击一个按钮，弹出输入框进行快速发言，这个输入框有自定义的表情。Google了下由于没有现成的库拿来用，只能撸起袖子自己干了！

### 二、实现思路

讲解之前，先来看看最终的效果图：

![ChatUI](/media/2018/rn-chatui.png)

<!--more-->

##### 1、Emoji表情

由于App自有一套表情体系，这里找了微信聊天表情来演示，建立一个公共的DataSource

```JavaScript

//符号->图片路径
export const EMOTIONS_DATA = {
  '/{weixiao': require('./emotions/weixiao.png'), /* 微笑 */
  '/{piezui': require('./emotions/piezui.png'), /* 撇嘴 */
  '/{se': require('./emotions/se.png'), /* 色 */
  ......
};

//符号->中文
export const EMOTIONS_ZHCN = {
  '/{weixiao':'[微笑]',
  '/{piezui': '[撇嘴]',
  '/{se': '[色]',
  ......
};

//反转对象的键值
export const invertKeyValues = obj =>
  Object.keys(obj).reduce((acc, key) => {
    acc[obj[key]] = key;
    return acc;
  }, {});

```

`EMOTIONS_DATA` ：变量与图片资源的映射

`EMOTIONS_ZHCN` ：变量与中文语义之间的映射，因为React Native中TextInput不支持插入图片，这么做一是为了兼容以前的App版本，二是为了消除用户对形如‘/{abc’特殊字符的歧义

`invertKeyValues` ： 反转对象的键值，也即反转上面的`EMOTIONS_ZHCN`，这个后面在图文混排显示时会用到

##### 2、输入框区域

输入框区域布局比较简单,用flexbox，这里着重讲ChatInputBar，

```JavaScript

  render() {
    if (Platform.OS === 'android'){
      return this._renderAndroidView();
    }

    return this._renderIosView();

  }

```

可以看到render方法中，根据平台作了适配，渲染不同View。之所以要区分得从一个需求点说起，注意到需求是点击一个按钮弹出一个输入框，不同于IM聊天界面输入框是固定在底部的，所以这里自然想到用弹窗Dialog，而React Native官方提供了一个叫`Modal`的组件，该组件可以用来覆盖包含React Native根视图的原生视图，但在实际开发过程中遇到了一个坑，即如果在Android系统上用原生的`Modal`，里面包裹ViewPagerAndroid组件，手机主动关屏再次亮屏后，会出现无法滑动ViewPagerAndroid的情况，具体原因还不清楚，后续会继续研究下，至于解决方案是我采用了第三方库[react-native-modalbox](https://github.com/maxs15/react-native-modalbox)，来看看`_renderAndroidView()`里面的代码

```JavaScript

_renderAndroidView(){
    return <ModalBox
      swipeToClose={false}
      backdropOpacity={0}
      backButtonClose={true}
      onClosed={() => this._onModalBoxClosed()}
      style={[styles.container]} position={'bottom'} ref={'modal'}>

      <TouchableWithoutFeedback onPress={() => this.closeInputBar()}>
        <View style={styles.box_container}/>
      </TouchableWithoutFeedback>
      <View style={styles.inputContainer}>
        <View style={styles.textContainer}>
          <TouchableWithoutFeedback
            onPress={() => this._toogleShowEmojiView()}>
            <Image style={styles.emojiStyle} source={require('./emotions/ic_emoji.png')}/>
          </TouchableWithoutFeedback>

          <TextInput
            ref="textInput"
            style={styles.inputStyle}
            underlineColorAndroid="transparent"
            multiline = {true}
            autoFocus={true}
            editable={true}
            placeholder={'说点什么'}
            placeholderTextColor={'#bababf'}
            onSelectionChange={(event) => this._onSelectionChange(event)}
            onChangeText={(text) => this._onInputChangeText(text)}
            onFocus={() => this._onFocus()}
            defaultValue={this.state.inputValue}/>

          <TouchableWithoutFeedback onPress={() => this._onSendMsg()}>
            <View ref="sendBtnWrapper" style={[styles.sendBtnStyle]}>
              <Text ref="sendBtnText" style={[styles.sendBtnTextStyle]}>发送</Text>
            </View>
          </TouchableWithoutFeedback>
        </View>
        <Line lineColor={'#bababf'}/>
        {
          this.state.isEmotionsVisible &&
          <EmotionsView onSelected={(code) => this._onEmojiSelected(code)}/>
        }
      </View>
    </ModalBox>;
  }
  
```

这里比较复杂的是`TextInput`组件，遇到两个比较坑的问题，首先我们知道在Android中EditText是可以通过`SpannableString`来插入图片的，而React Native中`TextInput`不支持插入图片，只能插入纯文本，既然不能插入表情图只能用表情符号替代了；另外的，`TextInput`没有直接获取当前正在编辑文本光标所在位置的方法，怎么办呢，自己保存一个全局的cursorIndex变量呗....，经过摸索，可以通过`onSelectionChange `来获取第一次点击时光标所在的位置，之后如果通过输入法或者表情插入文本时，就需要自己维护这个光标索引了，以下是引用其文档的解释

>  onSelectionChange function 
> 
> 长按选择文本时，选择范围变化时调用此函数，传回参数的格式形如 { nativeEvent: { selection: { start, end } } } 

在代码中通过`cursorIndex`状态变量来保存

```JavaScript

  _onSelectionChange(event){

    this.setState({
      cursorIndex:event.nativeEvent.selection.start,
    });
  }
  
```

`_renderIosView()`和`_renderAndroidView()`里面的代码大致相同，只是用了原生的`Modal`，

```JavaScript

_renderIosView(){
    return <Modal
      animationType={'slide'} transparent={true} visible={this.state.modalVisible}>
      ......
      <KeyboardAvoidingView behavior={'position'}>
        ......

            <TextInput
              ref="textInput"
              style={styles.inputStyle}
              underlineColorAndroid="transparent"
              multiline = {true}
              autoFocus={true}
              editable={true}
              keyboardType={'default'}
              selectionColor={'#56b2f0'}
              returnKeyType={'send'}
              placeholder={'说点什么'}
              enablesReturnKeyAutomatically={true}
              placeholderTextColor={'#bababf'}
              onSelectionChange={(event) => this._onSelectionChange(event)}
              onChangeText={(text) => this._onInputChangeText(text)}
              onFocus={() => this._onFocus()}
              defaultValue={this.state.inputValue}/>
         ......
      </KeyboardAvoidingView>
    </Modal>;
  } 

```

还有个不同的地方是`_renderIosView()`中使用了`KeyboardAvoidingView `进行包裹，该组件用于解决一个常见的尴尬问题：手机上弹出的键盘常常会挡住当前的视图。本组件可以自动根据键盘的位置，调整自身的position或底部的padding，以避免被遮挡，那为什么`_renderAndroidView()`没有用该组件包裹呢？说多了都是泪，因为不起作用啊。。。不过你在Android平台运行ChatUI项目后发现，输入法并没有遮挡住输入框视图，原因是ChatUI项目的Andorid工程里，AndroidManifest注册的MainActivity设置了`android:windowSoftInputMode="adjustResize"`属性，这个属性可以让视图根据软键盘的弹出自动调整，所以注意了，如果你的项目是原生和React Native混合开发的，在React Native依赖的那个Activity中，必须设置相应的`android:windowSoftInputMode`属性来适配相关的需求，哈哈哈！😝


##### 3、表情选择区域

分页显示的表情视图，在网上找了一个开源库[rn-viewpager](https://github.com/zbtang/React-Native-ViewPager)，简单浏览了下源码，在IOS端是通过ScrollView是实现的，Android端还是封装了React Native的`ViewPagerAndroid`组件，使用第三方库的好处是能够快速适配两个端，贴切需求，但坑也是大大滴！先来看一段`ViewPagerAndroid`文档中的一段话：

>ViewPagerAndroid
>
>一个允许在子视图之间左右翻页的容器。每一个ViewPagerAndroid的子容器会被视作一个单独的页，并且会被拉伸填满。
>注意所有的子视图都必须是纯View，而不能是自定义的复合容器。

注意，**所有的子视图都必须是纯View，而不能是自定义的复合容器**，

以下是我的`EmotionsView`修改前：

```JavaScript

  _renderPagerView(){
   ......
   
    viewItems.push(<EmotionsChildView key={0} />);
    viewItems.push(<EmotionsChildView key={1} />);
    viewItems.push(<EmotionsChildView key={2} />);
    return viewItems;
  }

  render() {
    return (
      <IndicatorViewPager
        style={styles.wrapper}
        indicator={this._renderDotIndicator()}>
        { this._renderPagerView() }
      </IndicatorViewPager>
    );
  }

```

修改后：

```JavaScript

  _renderPagerView(){
   ......
   
    viewItems.push(<View key={0}><EmotionsChildView/></View>);
    viewItems.push(<View key={1}><EmotionsChildView/></View>);
    viewItems.push(<View key={2}><EmotionsChildView/></View>);
    return viewItems;
  }

  render() {
    return (
      <IndicatorViewPager
        style={styles.wrapper}
        indicator={this._renderDotIndicator()}>
        { this._renderPagerView() }
      </IndicatorViewPager>
    );
  }

```

其中`EmotionsChildView `是我自己封装的一个复合表格View，**如果在`ViewPagerAndroid`中的子View不是纯View，而是一个复合容器的话，将会导致子View内部无法测量正确的高度！**

##### 4、表情图文混排显示

`RichTextWrapper` 封装的一个富文本组件，为ChatInputBar而生，仅有一个`textContent `属性

```JavaScript

    <RichTextWrapper textContent={this.state.chatMsg}/>
    
```

内部的实现：

``` JavaScript
    
  let emojiReg = new RegExp('\\/\\{[a-zA-Z_]{1,18}'); //表情符号正则表达式
    
  componentWillReceiveProps(nextProps) {

    this.state.Views.length=0;

    let textContent = nextProps.textContent;
    this._matchContentString(textContent);

  }

  _matchContentString(textContent){

    // 匹配得到index并放入数组中
    let emojiIndex = textContent.search(emojiReg);

    let checkIndexArray = [];

    // 若匹配不到，则直接返回一个全文本
    if (emojiIndex === -1) {
      this.state.Views.push(<Text key ={'emptyTextView'+(Math.random()*100)}>{textContent}</Text>);

    } else {

      if (emojiIndex !== -1) {
        checkIndexArray.push(emojiIndex);
      }

      // 取index最小者
      let minIndex = Math.min(...checkIndexArray);

      // 将0-index部分返回文本
      this.state.Views.push(<Text key ={'firstTextView'+(Math.random()*100)}>{textContent.substring(0, minIndex)}</Text>);

      // 将index部分作分别处理
      this._matchEmojiString(textContent.substring(minIndex));
    }
  }

  _matchEmojiString(emojiStr) {

    let castStr = emojiStr.match(emojiReg);
    let emojiLength = castStr[0].length;

    this.state.Views.push(<Image key={emojiStr} style={styles.subEmojiStyle resizeMethod={'auto'3} source={EMOTIONS_DATA[castStr]}/>);

    this._matchContentString(emojiStr.substring(emojiLength));

  }
     
```

`this.state.Views`是一个View数组，在`constructor()`中定义，用于存放截取的各个子View；`componentWillReceiveProps()`接收外部传递进来的字符串`textContent`，具体匹配在`_matchContentString ()`进行匹配：将一个字符串按表情符正则匹配规则查找，如果没有表情符index返回-1，直接push一个`Text`子View到Views数组；如果有表情符，取出checkIndexArray中index最小者，将字符串拆分成三部分，0-index，index，index-end，其中0-index直接返回纯文本，index部分就是表情符，这里转换成`Image`子View并push到Views数组中，将index-end部分进行再次递归，直到最终的index为-1返回纯文本为止。


最后，奉上源码，有需要的童鞋自行去下载咯~ 

GitHub:[https://github.com/WangGanxin/react-native-chatUI](https://github.com/WangGanxin/react-native-chatUI)

### 参考资料
- [ReactNative实现emoji表情图文混排方案](http://blog.csdn.net/gang544043963/article/details/70245850) 
- [React Native自定义View解析Emoji](https://www.jianshu.com/p/b8a6da804479)