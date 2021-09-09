# Common questions about flutter development.
### 说明

* 以下相关问题仅是个人在项目中遇到的，并予以记录
* 以下相关观点仅代表我个人看法，仅供参考。如有疑问可随时提issue



### iOS设备上的输入框，粘贴内容时重复粘贴

1. 项目背景
   * Flutter版本：`1.22.2`
   * 问题点：点击搜索输入框，输入法出现了验证码的QuickType栏
   * 制衡点：电话输入框的keyboardType，只能指定为phone类型
2. 问题原因
   1. iOS输入法上方的提示叫QuickType栏【详见：[QuickType bar](https://developer.apple.com/documentation/security/password_autofill/about_the_password_autofill_workflow#3001199)】
   2. 【项目中】在搜索输入框内就不应该提示验证码的QuickType栏
   3. 短信收到的验证码只会三分钟内在弹出的输入法QuickType栏提示
   4. 有了上面三点基础认识之后，我们首先想到的解决方案就是在搜索栏的输入框内不提示验证码
      * 通过给TextField添加autocorrect:false（禁用自动更正）、enableSuggestions:false（不展示输入建议）解决该问题
        * 网上最初的解决方案：[禁用预测文本](https://stackoverflow.com/questions/55157290/how-to-disable-predictive-text-in-textfield-of-flutter)
        * enableSuggestions在Android上的一些问题：[issue 22828](https://github.com/flutter/flutter/issues/22828)
      * 经过测试过后发现，只在TextField的keyboardType为Text时该设置生效，在keyboardType为number相关的Type时完全不生效，这个问题在flutter的issue中也有出现【⚠️目前暂未解决】
        * 2018年的issue就有这个问题：[issue 22828](https://github.com/flutter/flutter/issues/22828)
        * 最近的issue回复中还未解决：[issue 72537](https://github.com/flutter/flutter/issues/72537)
      * 虽然网上有说通过设置TextField的keyboardType为visiblePassword可以彻底解决该问题，但我们的电话输入框只能指定为phone类型不能替换，所以改解决方案不满足需求
   5. iOS点击QuickType栏重复键入内容，是iOS系统一直存在的问题，目前网上的解决方案大致如下：
      * 目前只有中文输入法有该问题的存在
      * 给输入框限制对应验证码的长度【详见：[iOS自带输入法验证码重复输入解决方案](https://blog.csdn.net/weixin_44363139/article/details/107781625)】
3. 解决方案：给TextField添加autocorrect:false、enableSuggestions:false，这样并不能完全解决该问题，但也能够最大程度降低该问题的出现【⚠️所以目前在电话输入框还会存在该问题】



### 长按底部导航栏/页面返回按钮，出现按钮提示

1. 项目背景

   * Flutter版本：`1.22.2`
   * 问题点：长按底部导航栏/页面返回按钮，出现按钮提示

3. 问题原因

   * 长按后出现的提示叫Tooltip

   * 导航栏：在BottomNavigationBarItem中使用lable时，在\_BottomNavigationBarState的build()函数中，通过\_BottomNavigationTile构建BottomBar的item；而在\_BottomNavigationTile的build()函数中，判断了BottomNavigationBarItem.label不为null，则会在BottomBarItem的外层嵌套Tooltip, Tooltip提示的信息也为label
   
     ![flutter 1.22.2 _BottomNavigationTile.build()](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_cq_1.png)
     
   * 返回按钮：在未指定AppBar的leading按钮时，页面在可以返回的时候展示了系统默认的返回按钮，默认的返回按钮IconButton添加了tooltip，导致了有该提示
   
     ![_AppBarState.build()](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_cq_3.png)
   
     ![BackButton.build()](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_cq_4.png)
   
3. 解决方案：

   * 导航栏：

     * 在BottomNavigationBarItem中使用title替换label【但title已经不推荐使用了】

     * 在Flutter 2.0版本之后，BottomNavigationBarItem将tooltip作为参数暴露出来，所以无需做任何更改

       ![Flutter 2.0 _BottomNavigationTile.build()](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_cq_2.png)

   * 返回按钮：给AppBar添加leading自定义返回按钮



### 在MaterialApp组件中使用Cupertino相关组件

* Flutter版本：`1.22.2`
* 在MaterialApp组件中使用了Cupertino相关组件, 则需要在其外层嵌套Localizations组件【如: 使用CupertinoTabBar替换BottomNavigationBar时】
* 详见：[CupertinoTabBar requires Localizations parent](https://flutter.cn/docs/release/breaking-changes/cupertino-tab-bar-localizations)



### 不展示Android上ScrollView的水波纹效果

1. 项目背景

   * Flutter版本：`1.22.2`
   * 问题点：在Android设备上，项目中的ListView滚动到底部或顶部时出现水波纹

2. 参考组件

   * `ScrollConfiguration` 是Android上提供的默认scroll behavior
   * `BouncingScrollPhysics`是类似iOS上的弹一弹效果
   * `GlowingOverscrollIndicator`, `ScrollConfiguration`提供水波纹效果，这种效果通常只在Android上才有。使用`MaterialApp`时，`GlowingOverscrollIndicator`的水波纹颜色可以通过`ThemeData.accentColor`进行指定.

3. 解决方案

   * 使用`ScrollConfiguration`搭配`自定义ScrollBehavior`，可去掉水波纹

   * 为什么需要自定义

     * 经过测试将`ThemeData.accentColor`设置为`Colors.transparent`时，并不能完全透明
     * 查看`GlowingOverscrollIndicator`的使用位置，发现了`ScrollConfiguration`。 在`ScrollBehavior`中`iOS, linux, macOS, windows`都是直接返回的child，而`android, fuchsia`直接使用了`GlowingOverscrollIndicator`
     * 查看`GlowingOverscrollIndicator`的构造方法，控制水波纹效果是由`showLeading`和`showTrailing`进行控制的，而这两个值默认是true。所以为了控制水波纹效果，自定义了`OverScrollBehavior`

   * 自定义ScrollBehavior

     ``` dart
     class OverScrollBehavior extends ScrollBehavior {
       @override
       Widget buildViewportChrome(
           BuildContext context, Widget child, AxisDirection axisDirection) {
         // When modifying this function, consider modifying the implementation in
         // the base class as well.
         switch (getPlatform(context)) {
           case TargetPlatform.iOS:
           case TargetPlatform.linux:
           case TargetPlatform.macOS:
           case TargetPlatform.windows:
             return child;
           case TargetPlatform.android:
           case TargetPlatform.fuchsia:
             return GlowingOverscrollIndicator(
               child: child,
               // 不显示头部水波纹
               showLeading: false,
               // 不显示尾部水波纹
               showTrailing: false,
               axisDirection: axisDirection,
               color: Theme.of(context).accentColor,
             );
         }
       }
     }
     ```

   * 使用方法如下

     ``` dart
     ScrollConfiguration(
       behavior: OverScrollBehavior(),
       child: ListView() / GridView() / CustomScrollView() / BoxScrollView(),
     )
     ```





