## Getx框架的使用方式和应用场景



#### 说明

* 文中相关时序图由我个人结合`GetX`源码自行总结得出，仅代表我个人理解，仅供参考。
* 如需转载请注明出处，如有疑问可随时提issue
* 本文章基于`get: ^4.3.8`
* GitHub：[GetX](https://github.com/jonataslaw/getx)





### 状态管理

在使用`GetX`之前，我们常采用`Provider + Selector/Consumer `的原生实现方式，来完成状态管理，如下：

* 创建`ChangeNotifierProvider`，在其内部完成状态管理
* 为满足多个`Provider`的状态发生变化时都能收到通知，使用`MultiProvider`组件将所有`Provider`都放置在主入口，这样是全局的了
* 在页面中使用`Selector/Consumer`将需要状态同步的组件包在内部，这样就能实现`Provider`中的状态发生变化时，页面中组件就能够及时更改

```dart
// Provider
class Provider extends ChangeNotifier {
  int _count = 0;
  int get count => _count;
  increment() {
    _count++;
    notifyListeners();
  }
}

// MultiProvider
MultiProvider(
	providers: [
    ChangeNotifierProvider(create: (_) => Provider()),
  ],
  child: MaterialApp(...),
)
  
// Selector
Selector<Provider, int>(
	selector: (_, provider) => provider.count,
  builder: (_, count, _) => Text('count is $count'),
)
```

`Provider`核心原理还是采用了`InheritedWidget`的刷新机制，也严重依赖`Widget`树，也就是为什么将`MultiProvider`放在主入口处，就能全局监听到`Provider`的状态变化。



而使用`GetX`，一切都是反应式的，只需要在变量后面加上`.obs`，即完成了可观察的“响应式”变量`Rx`的声明。在使用时只需要将该`Rx`变量包在`Obx()`组件内部即可，这样当该`Rx`变量发生变化时页面也能及时更改。

```dart
// Controller
class Controller extends GetxController {
  var count = 0.obs;
  increment() => count++;
}

// View
final c = Get.put(Controller());
Obx(() => Text('count is ${c.count}'))
```

`GetX`状态管理可以单独使用，不依赖于`GetMaterialApp()`，常用组件：`GetBuilder`、`GetX`、`Obx`、`Worker`等



#### Rx变量

有3种方式将变量变为“可观察的”。

```dart
// 1. Rx{Type}: RxString/RxInt/RxDouble/RxBool/RxList/RxMap
final name = RxString('');
final items = RxList<String>([]);

// 2. Rx<Type>, Type可以是基本类型或自定义类
final name = Rx<String>('');
final items = Rx<List<String>>([]);
final customClass = Rx<CustomClass>();

// 3. 添加'.obs'作为变量属性，变量可以是基本类型或自定义类
final name = ''.obs;
final items = [].obs;
final customClass = CustomClass().obs;
```

> - 虽然Dart中所有类型都是对象，但我们此处暂且将`String/int/double/bool/List/Map`称之为“基本类型”吧
> - `.obs`变量虽然可以打印出对应的值，但是其类型还是对应的`Rx{Type}`。所以像`''.obs`这种就不能使用类似像`.substring()`之类的方法，只能使用`Rx{Type}`部分扩展方法。（如：`Rx{Type}`的`.nil()`、`RxBool`的`.toggle()`）
> - 当`Rx<Type>`变量中的`Type`是自定义类时，不能这样定义：`Rx<CustomClass?> custom = null.obs;`，因为这样相当于是对`Null`做了监听
>
> * `GetX`给`List`扩展了一些方法：`addIf()`、`addAllIf()`。（如果对`List`做了观察，那更改`List`中`item`对象中字段的值，需要主动触发`List`刷新。`Map`同理）
>
>   ```dart
>   // User
>   class User {
>     String name;
>     int age;
>     User({this.name = '', this.age = 0});
>   }
>           
>   // Controller
>   class Controller extends GetxController {
>     var names = <User>[].obs;
>     @override
>     void onInit() {
>       super.onInit();
>       names.addAll([User(name: 'name1', age: 18), User(name: 'name2', age: 20)]);
>     }
>   }
>           
>   // 在页面中更新item中的字段时
>   controller.names[1].age = 30;
>   controller.names.refresh();
>   ```



#### 如何在View中使用

在View中分为`GetBuilder`、`GetX`、`Obx`三种使用方式。

* `GetBuilder/GetX/Obx`只会在所使用的`Rx`变量发生变化时才改变（就算是两次的值一样，也不会触发）。而`Provider`是在其中一个状态发生变化时，其他状态的监听者都会发生改变（即使变量值没有改变）

  ```dart
  // Rx变量的setValue()
  set value(T val) {
    if (subject.isClosed) return;
    if (_value == val && !firstRebuild) return;
    firstRebuild = false;
    _value = val;
  
    subject.add(_value);
  }
  ```

* 默认在首次更新状态时，会触发组件内部重建，即使是相同的值（可以通过`Rx{Type}.firstRebuild = false`关闭该功能）

* 可以通过`controller.type.value`获取到对应`Type`变量的值，也可以通过对`value`赋值去触发组件重建。但需要注意在必须使用字符串的组件中（如：`Text()`组件），不能直接将`Rx`变量传入，需使用其`value`然后按照正常组装方式传入。

  ```dart
  // Controller
  class Controller extends GetxController {
  	var name = ''.obs;
  	var count = 0.obs;
  }
  
  // View中使用时
  final c = Get.find<Controller>();
  // 错误示范
  // Obx(() => Text(c.name))
  // Obx(() => Text(c.count))
  // 正确示范
  Obx(() => Text(c.name.value))
  Obx(() => Text('${c.name}'))
  Obx(() => Text('${c.count}'))
  ```

##### GetBuilder

```dart
// Controller
class Controller extends GetxController {
  var count = 0;
  increment() {
    count++;
    update();
  }
}

// GetBuilder
GetBuilder(
	init: Controller(),
  builder: (c) => Text('count is ${c.count}'),
)
```

> 如果依赖中存在`Controller`的实例，则不会使用`GetBuilder()`中`init`传入的实例；不存在时才会使用`init`传入的实例，同时将其加入到依赖中

##### GetX / Obx

```dart
// Controller
class Controller extends GetxController {
  var count = 0.obs;
  increment() => count++;
}

// GetX
GetX<Controller>(builder: (c) => Text('count is ${c.count}'))

// Obx
final c = Get.find<Controller>();
Obx(() => Text('count is ${c.count}'))
```

##### ValueBuilder / ObxValue

* 这两个组件可以用来在局部组件间，对单个值进行状态管理

* 适用情况：在展示用户电话号码或其他敏感信息时，我们使用了模糊文本，然后用户想要查看完整信息时。此时就可以使用`ValueBuilder / ObxValue`组件，对其可见性状态进行局部管理

  ```dart
  // ValueBuilder
  ValueBuilder<bool?>(
    initialValue: false,
    builder: (value, updateFn) => InkWell(
      	child: Text('131${value! ? '1234' : '****'}5678'),
        onTap: () => updateFn(!value),
      ),
  )
    
  // ObxValue
  ObxValue<RxBool>(
    (data) => InkWell(
      	child: Text('131${data.value ? '1234' : '****'}5678'),
        onTap: () => data.toggle(),
      ),
    false.obs,
  )
  ```

##### GetView / GetWidget

###### GetView

* `GetView`是帮我们省略了`find()`操作的`StatelessWidget`，我们可以直接通过其内部的`controller`直接访问`Controller`内部的属性及方法

* 适用场景：只要我们有用到`Controller`的地方，都可以将页面继承自`GetView`

  ```dart
  class HomePage extends GetView<Controller> {
    @override
    Widget build(BuildContext context) {
  		return Center(child: Text('${controller.value}'));
    }
  }
  ```

###### GetWidget

* `GetWidget`内部缓存了`Controller`，适合与`create()`创建的`Controller`搭配使用
* 适用场景：官方说可以用来保存代办事项列表（如果`item`被重建，`GetWidget`可以保存相同的`Controller`实例），自己暂时还没能想到适用的场景

##### GetxService

`GetxService`就像一个`GetxController`，共享相同的生命周期（`onInit()/onReady()/onClose()`）。跟我们平时在`Android`上看到的`Service`一样，如果说想在应用生命周期内绝对持久化类的实例，那推荐使用`GetxService`。

* `GetxService`与`GetxController`不同的是其内部没有“逻辑”，只通过`GetX`将依赖注入系统，同时不能从内存中删除

  > 删除`GetxService`的唯一方法是`Get.reset()`，该方法类似对`GetX`做“热重启”

* 适用场景：`SharedPerfrence`和`Api`相关操作类，适用`GetxService`将其管理，即可在应用内任何位置使用

  ```dart
  // GetxService
  class SpService extends GetxService {
    var sharedPreferences?;
  	init() async {
  		sharedPreferences = await SharedPreferencesUtil.getInstance();
    }
  }
  
  // 使用
  initService() async {
    await Get.putAsync(SpService()).init();
  }
  main() async {
    await initService();
    runApp(...);
  }
  ```



#### 如何在其他位置/页面使用之前的Controller

``` dart
// 1. 使用GetBuilder
GetBuilder<Controller>(builder: (c) => Text('${c.xxx}'))

// 2. 使用Get.find<S>(tag)
Get.find<Controller>.xxx

// 3. 使用Controller.to
class Controller extends GetxController {
  var xxx = 0.obs;
  static Controller get to => Get.find();
}

Controller.to.xxx
```

如果你的页面只有一个`Controller`，并且需要使用它。那么只需要将页面继承自`GetView`即可，`GetView`帮你省略了`Get.find()`的操作，只需要在布局中直接使用`controller`即可。

```dart
class TestPage extends GetView<Controller> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
    	...
      Obx(() => Text('count is ${controller.count}'))
      ...
    );
  }
}
```



#### Worker

`Worker`将协助你在事件发生时触发特定的回调。

##### ever

* 每次`Rx`变量改变时都会触发回调

* 适用场景：登录状态变化后，触发相应的操作，如下：

  ```dart
  class Controller extends GetxController {
    var isLogin = false.obs;
    
    @override
    onInit() async {
      ever(isLogin, fireRoute);
      isLogin.value = await Preferences.hasToken();
    }
  
    fireRoute(login) => login ? Get.off(HomePage()) : Get.off(LoginPage());
  }
  ```

##### once

仅在`Rx`变量首次改变时触发，暂未联想到适用场景

##### debounce

* 防DDos，用户停止输入多少时间后触发

* 适用场景：搜索功能。在用户停止输入时才去请求api，避免在每次`onChange`时都触发请求

  ```dart
  // Controller
  class Controller extends GetxController {
    var textInput = ''.obs;
    
    @override
    onInit() async {
      debounce(textInput, reqApi, time: Duration(milliseconds: 800));
    }
    
    reqApi(value) => restApi...
  }
  
  // TextField
  final c = Get.find<Controller>();
  TextField(
    ...
  	onChanged: (value) => c.textInput.value = value,
    ...
  )
  ```

##### interval

* 忽略多少时间内的所有更改，暂未联想到适用场景
* 虚拟场景：假设用户通过点击某物可以赚取金币，如果他在同一分钟内点击300次，他将获得300金币。使用`interval`的话，你可以设置一个3秒的时间范围，即使点击300或1000次，他在1分钟内最多只能得到20个硬币



#### GetBuilder、GetX/ObX的刷新机制是如何实现的？

`GetBuilder`和`GetX/Obx`的刷新机制不一样，`GetBuilder`底层是通过`ListNotifier`方式刷新，而`GetX/Obx`底层则是通过`GetStream`方式实现。但最终能在页面完成刷新，都是因为`GetBuilder/GetX/Obx`都是一个`StatefulWidget`，在底层触发刷新回调时，都是执行`setState(() {})`完成页面刷新。

##### GetBuilder刷新机制

* 在`GetBuilderState`的`initState()`内部，将更新组件的方法`getUpdate()`，添加到`ListNotifierMixin`的`_updaters`列表中

  > `getUpdate()`方法是`GetStateUpdaterMixin`中的，方法内部是`if (mounted) setState(() {})`，而`GetBuilderState with GetStateUpdaterMixin`

* 在调用`Controller`的`update()`方法时，最终会通过`ListNotifierMixin`的`_notifyUpdate()`方法，将`_updaters`列表中的所有监听回调方法执行，以此完成刷新

![GetBuilder刷新时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_1.png)

###### GetBuilder内部相关源码如下：

```dart
// GetBuilder, 省略了不相干的一些代码
class GetBuilder<T extends GetxController> extends StatefulWidget {
  @override
  GetBuilderState<T> createState() => GetBuilderState<T>();
}

// GetBuilderState, 省略了不相干的一些代码
class GetBuilderState<T extends GetxController> extends State<GetBuilder<T>>
  with GetStateUpdaterMixin {
  @override
  void initState() {
    ...
      _subscribeToController();
  }

  void _subscribeToController() {
    // _filterUpdate最终也是执行了getUpdate
    _remove?.call();
    _remove = (widget.id == null)
      ? controller?.addListener(
      _filter != null ? _filterUpdate : getUpdate,
    )
      : controller?.addListenerId(
        widget.id,
        _filter != null ? _filterUpdate : getUpdate,
      );
  }
}

// GetStateUpdaterMixin
mixin GetStateUpdaterMixin<T extends StatefulWidget> on State<T> {
  void getUpdate() {
    if (mounted) setState(() {});
  }
}

// GetxController
abstract class GetxController extends DisposableInterface
  with ListenableMixin, ListNotifierMixin {}

// ListNotifierMixin, 省略了不相干的一些代码
mixin ListNotifierMixin on ListenableMixin {
  ...
    @override
    Disposer addListener(GetStateUpdate listener) {
    ...
      _updaters!.add(listener);
    ...
  }
}
```

###### GetxController内部相关源码如下：

```dart
// GetxController
abstract class GetxController extends DisposableInterface
    with ListenableMixin, ListNotifierMixin {
  void update([List<Object>? ids, bool condition = true]) {
    if (!condition) {
      return;
    }
    // refreshGroup(ids)最终也是执行了refresh()
    if (ids == null) {
      refresh();
    } else {
      for (final id in ids) {
        refreshGroup(id);
      }
    }
  }
}

// ListNotifierMixin, 省略了不相干的一些代码
mixin ListNotifierMixin on ListenableMixin {
  @protected
  void refresh() {
    ...
    _notifyUpdate();
  }
  
  void _notifyUpdate() {
    for (var element in _updaters!) {
      element!();
    }
  }
}
```

##### GetX/Obx刷新机制

![Obx刷新状态流转图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_7.png)

![Obx刷新时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_2.png)

###### GetStream的刷新机制：

* `GetStream`的`add()`方法会调用内部的`_notifyData()`方法

* 而`_notifyData()`会将内部`_onData`列表中的所有`LightSubScription`取出执行

  > `LightSubScription`是添加到`GetStream`中的监听回调的封装

###### _ObxState内部：

* 在`initState()`内部，将更新组件的方法`_updateTree()`，通过`GetStream.listen()`方法封装成`LightSubscription`，然后添加到`_ObxState`的`GetStream`的`_onData`列表中

  > `_ObxState`中的`_updateTree()`方法，内部其实也是`if (mounted) setState(() {})`

* 在`build()`内部，会通过`RxInterface`将其`proxy`指定为`_ObxState`的`GetStream`，然后执行组件的`build()`方法

* 在组件的构建过程中，肯定会通过`Rx`变量的`get`方法，去取出`value`

* 而在`value`的`get`方法中，会通过`RxInterface`中`proxy`的`addListener()`方法，将`_ObxState`中的`GetStream.add()`封装成`LightSubscription`，添加到`Rx`变量的`GetStream`的`_onData`列表中

###### Rx变量内部：

* 在更改`Rx`变量的`value`时，肯定会调用`value`的`set`方法
* 而在`value`的`set`方法中，会调用内部的`GetStream`的`add()`方法完成刷新

###### Obx刷新机制（GetX的刷新机制同Obx）：

`Rx`变量`setValue()` ➡️ `Rx`变量的`GetSream.add()` ➡️ `_ObxState`的`GetStream.add()` ➡️ 执行`_ObxState`的刷新方法



#### GetBuilder、GetX、Obx的区别

* `Obx`只有一个`builder`，只适用于简单的使用

* `GetBuilder`和`GetX`都拥有`StatefulWidget`生命周期相关回调函数可以传入，而`Obx`没有

  > 如：`initState`、`didChangeDependencies`、`didUpdateWidget`、`dispose`

* `GetBuilder`可以添加`id`。在向`Controller`添加监听回调时，可以根据`id`分组添加；在`Controller`刷新时，也会根据`id`去执行该组的回调方法（也就是只刷新该`id`组的组件）。而`GetX`和`Obx`是只要使用了该`Rx`变量都会被刷新

* `GetBuilder`可以添加`filter`函数。用来控制在刷新时，`Controller`经过`filter`方法后得到的结果，和上次执行不一样时才刷新组件。而`GetX`和`Obx`没有过滤器

  > `filter`适用场景：比如我们只在变量奇偶类型不一致的情况才刷新（如：“上次是奇数，下次是偶数”才刷新，“上次和下次都是奇数或偶数”则不刷新）

  ```dart
  // Controller
  class Controller extends GetxController {
    var count = 0;
    increment() {
      count++;
      update();
    }
  }
  
  // GetBuilder
  GetBuilder(
  	init: Controller(),
    filter: (c) => c.count % 2 == 0,
    builder: (c) => Text('count is ${c.count}'),
  )
  ```



#### GetxController / GetxService的生命周期（onInit/onReady/onClose）

由于`GetxController`和`GetxService`最终都继承自`GetLifecycle`，而在`GetLifecycle`中才会去触发生命周期回调方法，所以`GetxController`和`GetxService`的生命周期一致。

![GetxController生命周期](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_3.png)

* 在`GetxController/GetxService`实例被创建时，会将`onInit()`和`onClose()`，以`onStart()`和`onDelete()`的回调方式插入到`GetLifecycle`中

* 在`GetxController/GetxService`实例创建好之后，会立即触发其`onStart()`回调，最终执行`onInit()`方法

* 在`onInit()`方法中，会向`WidgetBinding.instance`中插入一条帧回调监听。在下一帧开始绘制时，触发`onReady()`方法

* 当路由被销毁时，最终会触发`GetInstance.delete()`方法，在其方法内部触发`onDelete()`回调，最终执行`onClose()`方法

  > 在`GetInstance.delete()`中，正常情况下只有`GetxController`会触发`onClose()`。`GetxService`只会在强制删除时，才会触发`onClose()`



### 路由管理

在使用`GetX`之前，我们常采用原生`Navigator`或`fluro `的实现方式，来完成路由管理，如下：

* 使用原生`Navigator`：直接在页面中使用原生`Navigator`提供的相关路由跳转方法完成跳转（原生`Navigator`需要传入`Context`）

* 使用`fluro`：

  * 在主入口构建好`FluroRouter`对象，然后通过该对象的`define()`方法，将路由在`FluroRouter`中定义好

  * 在`MaterialApp`中，将`FluroRouter.generator`传给`onGenerateRoute`完成路由注册

  * 最后在页面中使用`FluroRouter.navigateTo()`方法，完成路由跳转（需要注意的是，`FluroRouter.navigateTo()`方法也需要传入`Context`）

    ```dart
    // 定义页面路由
    final router = new FluroRouter();
    router.define('/homePage', handler: homeHandler);
    
    // MaterialApp
    MaterialApp (
      ...
      onGenerateRoute: router.generator,
    	...
    )
    
    // 页面中使用
    router.navigateTo(context, '/homePage');
    ```

不管是使用原生`Navigator`还是`fluro`来管理路由，都需要自己去想办法，如何在任何地方都能完成路由跳转？



而使用`GetX`，我们只需要将`MaterialApp`替换成`GetMaterialApp`，就可以在应用内任何地方通过`Get.to()`方法完成路由跳转。

```dart
// GetMaterialApp
runApp(GetMaterialApp(...));

// 路由跳转
Get.to(HomePage())
```

同时在使用`GetX`的相关路由跳转方法时，我们不需要传入`Context`，`GetX`在底层帮我们完成了`Context`的获取。



#### 普通路由

```dart
// 跳转至下个页面
// 内部使用：Navigator.push(nextRoute)
Get.to(NextPage());

// 关闭当前页面
// 内部使用：Navigator.pop(context)
Get.back();

// 根据predicate规则关闭页面
// 内部使用：Navigator.popUtil(predicate)
// PS：适用于回到首页
Get.until(predicate);

// 跳转至下个页面，然后移除上个页面
// 内部使用：Navigator.pushReplacement(nextRoute)
// PS：适用于闪屏页、登录相关页面
Get.off(NextPage());

// 跳转至下个页面，同时移除之前所有页面
// 内部使用：Navigator.pushAndRemoveUntil(nextRoute, (_) => false)
// PS：适用于首页、购物车相关页面
Get.offAll(NextPage());

// 跳转至下个页面，然后根据predicate规则关闭之前页面（需要自己组装Route、定义predicate）
// 内部使用：Navigator.pushAndRemoveUntil(nextRoute, predicate)
// PS：适用于避免重复打开相同页面
Get.offUtil(nextRoute, predicate);

// 页面传参，及接收返回参数
var data = await Get.to(NextPage(), arguments: 'type is dynamic');

// 带数据返回上个页面
// 内部使用：Navigator.pop(result)
Get.back(result);
```



#### 命名路由

##### 定义路由

使用`GetPage`在`GetMaterialApp`中完成命名路由的定义。

```dart
GetMaterialApp(
  initialRoute: '/',
  getPages: [
    GetPage(name: '/', page: () => HomePage()),
    GetPage(name: '/second', page: () => SecondPage()),
    GetPage(
      name: '/second', 
      page: () => SecondPage(),
      // 过渡方式
      transition: Transition.zoom,
      // 子页面
      children: [
        GetPage(name: '/info', page: () => SecondInfoPage()),
      ],
    ),
  ],
  // 处理未定义的路由
  unknownRoute: GetPage(name: '/notfound', page: () => UnknownRoutePage()),
  ...
)
```

##### 命名路由的使用

```dart
// 跳转至下个页面
// 内部使用：Navigator.pushNamed(routeName)
Get.toNamed('/nextPage');

// 跳转至下个页面，然后移除上个页面
// 内部使用：Navigator.pushReplacementNamed(routeName)
Get.offNamed('/nextPage');

// 跳转至下个页面，同时移除之前所有页面
// 内部使用：Navigator.pushNamedAndRemoveUntil(routeName, (_) => false)
Get.offAllNamed('/nextPage');

// 跳转至下个页面，然后根据predicate规则关闭之前页面（需要自己定义predicate）
// 内部使用：Navigator.pushNamedAndRemoveUntil(routeName, predicate)
// PS：适用于避免重复打开相同页面，如下：
// Get.offNamedUntil(
//   networkErrorPage,
//   predicate: (route) =>
//     route.isCurrent && route.settings.name == networkErrorPage
//       ? false
//       : true,
// )
Get.offNamedUntil(routeName, predicate);

// 先关闭当前页面，然后跳转至下个页面
// 内部使用：Navigator.popAndPushNamed(routeName)
Get.offAndToNamed('/nextPage');

// 页面传参，及接收返回参数
var data = await Get.toNamed("/nextPage", arguments: 'type is dynamic');
```

##### GetMiddleware

`GetPage`有一个`middlewares`字段，可以用来配置页面的中间件`GetMiddleware`。在这个中间件中你可以监听到页面的相关变化。

```dart
GetMaterialApp(
  getPages: [
    GetPage(
      name: '/home', 
      page: HomePage(), 
      middlewares: [
        GetMiddleware(priority: 1),
        GetMiddleware(priority: -1),
      ],
    ),
  ],
  ...
)
```

* 在`GetMiddleware`的构造函数中，有一个`priority`参数用来定义其优先级，优先级越小越先执行

* 如果这个页面只有一个`GetMiddleware`时，那它的所有子页面都会自动拥有相同的中间件`GetMiddleware`

下面对`GetMiddleware()`内部的相关方法进行说明：

###### redirect

* 该方法会在“正在搜索将要跳转的页面路由”时被调用。返回`null`不会重定向，返回`RouteSettings`则重定向至其设置的页面

* 适用场景：我们可以在跳转个人详情页时，判断用户是否登陆，然后重定向至登陆页面

  ```dart
  @override
  RouteSettings? redirect(String? route) => Get.find<Controller>.isLogin.value ? null : RouteSettings(name: '/login');
  ```

###### onPageCalled

* 该方法会在“页面创建任何内容之前调用此页面”时被调用

* 适用场景：打开页面前更改页面的某些内容，或者给换一个页面

  ```dart
  @override
  GetPage? onPageCalled(GetPage? page) => page.copyWith(title: 'Welcome ${Get.find<AuthService>.userName}');
  ```

###### onBindingsStart

* 该方法会在“`Bindings`初始化之前”被调用
* 适用场景：可以在此处更改页面的`Bindings`

###### onPageBuildStart

该方法会在“`Bindings`初始化之后”立即被调用

###### onPageBuilt

该方法会在“页面构建好（`GetPage.page`）之后”立即被调用

###### onPageDispose

该方法会在“页面被释放之后”立即被调用



#### 动态网址Url

##### 普通传参方式

```dart
// 使用方式
Get.toNamed("/nextPage?device=phone&id=354&name=Enzo");

// 获取参数
// PS：在controller/stateful/stateless中都可以这样获取
var id = Get.parameters['id'];
```

##### 根据不同参数跳转不同页面

```dart
// 定义路由
GetMaterialApp(
  initialRoute: '/',
  getPages: [
    // 不带参数时，需在路由名称后面加上“/”，以代表不接受参数
    GetPage(name: '/profile/', page: () => MyProfilePage()),
    // 带参数跳转不同页面，通过以下方式定义
    GetPage(name: '/profile/:user', page: () => UserProfilePage()),
  ],
  ...
)
  
// 使用方式
Get.toNamed("/profile/34954");

// 在页面中获取参数
Get.parameters['user'];

// 发送多个参数
// 方式1：直接在url上跟参数
Get.toNamed("/profile/34954?flag=true&country=italy");
// 方式2：通过路由方法内参数传递
var parameters = <String, String>{"flag": "true", "country": "italy"};
Get.toNamed("/profile/34954", parameters: parameters);
```



#### 监听路由事件

一般我们监听到路由事件，可以用来作为事件埋点，或用于页面拦截。

##### GetMaterialApp

```dart
GetMaterialApp(
  routingCallback: (routing) {
    if(routing.current == '/second'){
      openAds();
    }
  },
  ...
)
```

##### MaterialApp

```dart
// NavigatorObserver
class MiddleWare {
  static observer(Routing routing) {
    if(routing.current == '/second'){
      openAds();
    }
  }
}

// MaterialApp
MaterialApp(
  navigatorObservers: [
    GetObserver(MiddleWare.observer),
  ],
  ...
)
```



#### 弹窗

我们在`flutter`中常见的弹窗有`Snackbar`、`Dialog`、`BottomSheet`，在`GetX`中也将这几个弹窗的调用方式进行了简化。

##### Snackbar

* 调用方式：`Get.snackbar()`
* 内部使用：`Navigator.push(SnackRoute)`
* 如果想完全自定义`SnackBar`，可以使用`Get.rawSnackbar()`

##### Dialog

* 展示`Dialog`：`Get.dialog(DialogWidget())`
* 默认`Dialog`：`Get.defaultDialog(...)`

##### BottomSheet

调用方式：`Get.bottomSheet(BottomSheetWidget()）`

> 注意：在BottomSheet上跳转页面A，然后页面A跳转页面B，从页面B通过popUtil回到BottomSheet时，这个时候GetX框架中的currentRoute还是页面A，原因可以查看以下源码。这是GetX框架的一个Bug，详见[issue-1870](https://github.com/jonataslaw/getx/issues/1870)。
>
> ```dart
> @override
> void didPop(Route route, Route? previousRoute) {
>   // ... 此处省略相关逻辑代码
> 
>   // Here we use a 'inverse didPush set', meaning that we use
>   // previous route instead of 'route' because this is
>   // a 'inverse push'
>   _routeSend?.update((value) {
>     // Only PageRoute is allowed to change current value
>     if (previousRoute is PageRoute) {
>       value.current = _extractRouteName(previousRoute) ?? '';
>     }
> 
>     // ... 此处省略相关逻辑代码
>   });
> 
>   // ... 此处省略相关逻辑代码
> }
> ```

##### 关闭弹窗小组件

在页面中可以通过`Get.overlayContext`拿到弹窗小组件的上下文，然后可根据该`Context`对其进行关闭操作。



#### 路由管理为啥需要GetMaterialApp()，Get.to()和Get.back()为啥不需要context？

* 在`GetMaterialApp`的`build()`中，直接创建了一个`GetBuilder`。而该`GetBuilder`的`init`参数传入了`Get.rootController`，`Get.rootController`是一个`GetMaterialController`，在其中创建了`GlobalKey`供全局路由跳转使用

* 在`GetBuilder`的`build()`方法中，会创建一个`MaterialApp`，并将`GetMaterialController`中的`GlobalKey`，作为`navigateKey`传入

* `GetX`中路由跳转的相关方法，都会通过`global(id)`方法获取`GlobalKey`。默认在未传入`id`时，返回的是`GetMaterialController`中的`GlobalKey`；如果有传入`id`，则会去`GetMaterialController`中的`keys`中找，未找到会抛出异常

  > `keys`是一个`GlobalKey`的`Map`，可通过`Get.nestedKey()`向其添加`key`

* 拿到`GlobalKey`后，即可通过`currentState`获取`NavigatorState`，然后通过原生的路由跳转方法完成路由跳转

![路由跳转时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_4.png)

> * 通过`MaterialApp`的`navigateKey`，来取代原生路由的`Navigator.of(context)`操作
>
> * `Get`的`Snackbar`、`Dialog`、`Bottomsheet`底层都是使用`GlobalKey`，通过`currentState`获取`NavigatorState`，使用`push()`路由的方式完成弹窗展示



#### 路由跳转时的加载流程（含Middleware、Bindings）

![路由跳转时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_5.png)

###### GetPageRoute的创建

* 普通路由：

  * 通常指`Get.to()/Get.off()/Get.offAll()`这三种方式

  * 直接在跳转时通过`GetPageRoute`构造函数创建，然后将其交给原生`Navigator`进行路由管理

    > 注意：这种方式创建的`GetPageRoute`，没有`Middleware`中间件

* 命名路由：

  * 通常指`Get.toNamed()/Get.offNamed()/Get.offAllNamed()/Get.offAndToNamed()`这四种方式

  * 这几种路由跳转方式，最终都会调用`NavigatorState._routeNamed()`创建`Route`，然后将其交给原生`Navigator`进行路由管理

  * 而在`_routeNamed()`方法内部，是通过在`MaterialApp`传入的`onGenerateRoute`函数进行构建。而在`GetMaterialApp`创建`MaterialApp`时，传入的是其内部的`generator()`函数。所以最终是通过`GetMaterialApp.generator()`方法创建`Route`

  * 在`generator()`方法中，通过构造函数的方式创建了`PageRedirect`，然后调用了其`page()`方法。在`page()`方法中会先调用`needRecheck()`方法，然后通过构造函数创建了`GetPageRoute`，并返回上层

    > 在`needRecheck()`方法中，会先后触发`Middleware`的`onPageCalled()`和`redirect()`回调。在触发`onPageCalled()`之前，会将路由树中的`parameters`放到`Get.parameters`中，然后在`onPageCalled()`之后会将当前路由的`parameters`添加到`Get.parameters`中

* `GetPage.createRoute()`：

  * 目前不清楚原生`Navigator`的路由管理会在什么是否触发该方法。从调用路径上看只有`NavigatorState`的`restoreState()`和`_updatePages()`两个方法内部有触发该方法。

    > 目前猜测是在页面被重构时会触发该方法，或者是`Get.to()`这类方法传了`routeName`参数之后，最终也会触发`GetPage.createRoute()`

  * 在该方法中，也是通过构造函数的方式创建了`PageRedirect`，然后调用了其`getPageToRoute()`方法。在`getPageToRoute()`方法中有着和`page()`方法相同的逻辑，也是先调用`needRecheck()`方法，然后通过构造函数创建了`GetPageRoute`

###### Route构建页面内容

* 在`Route`构建页面时，会触发`GetPageRoute.buildContent()`方法，在其内部调用了`_getChild()`方法

* 在`_getChild()`方法中，会先后触发`Middleware`和`Bindings`的回调。详细的顺序如下：

  `Middleware.onBindingsStart()` ➡️ `Bindings.dependencies()` ➡️ `Middleware.onPageBuildStart()` ➡️ `构建页面内容` ➡️ `Middleware.onPageBuilt()`

###### Route被销毁

在`Route`被销毁时，会触发`GetPageRoute.dispose()`方法，在其内部触发了`Middleware`的`onPageDispose()`回调。



### 依赖注入

在使用`GetX`之前，我们的依赖注入基本都通过构造函数完成。在`GetX`中我们可以通过`Get`的`put/putAsync/lazyPut/create`方法，将对象注入到`Get`中，让其帮我们管理。

```dart
// Get.put()
TestClass c = Get.put(TestClass());

// Get.putAsync()
Future<TestClass> c = Get.putAsync(() async {
  await Future.delayed(Duration(seconds: 1));
  return TestClass();
});

// Get.lazyPut()
Get.lazyPut(() => TestClass());

// Get.create()
Get.create(() => TestClass());
```

##### put\<S\>(S)

* 实际执行的是`GetInstance`的`put()`方法
* 在`_insert()`时传入的是“返回值为`S`实例”的`InstanceBuilderCallback`，然后执行`find()`返回`S`的实例
* 适用场景：需要在注入时“立马就将`S`的实例放入内存，同时需要立马使用`S`的实例”的情况

##### putAsync\<S\>(AsyncInstanceBuilderCallback\<S\>)

* 实际执行的是`GetInstance`的`putAsync()`方法

* 在`_insert()`时传入的是异步的`InstanceBuilderCallback`，然后执行`find()`返回`Future<S>`

  > `Future<S>`是`AsyncInstanceBuilderCallback<S>`执行的结果

* 适用场景：需要在注入`S`实例前，做异步操作的情况

  > 虚拟场景：在注入`S`实例前需要请求`Api`，即创建`S`实例依赖网络结果）

##### lazyPut\<S\>(InstanceBuilderCallback\<S\>)

* 实际执行的是`GetInstance`的`lazyPut()`方法
* 在`_insert()`时传入的是同步的`InstanceBuilderCallback`，不执行`find()`，无返回内容
* 适用场景：在注入时不需要立马就将`S`的实例放入内存，同时也不需要立马使用`S`的实例的情况

##### create\<S\>(InstanceBuilderCallback\<S\>)

* 实际执行的是`GetInstance`的`create()`方法

* 在`_insert()`时传入的是同步的`InstanceBuilderCallback`，默认传入的`isSingleton`是`false`，所以在后面每次`find()`的时候，都会执行一次`InstanceBuilderCallback`

* 适用场景：每次获取`S`对象都是一个新的实例

  > 在购物车页面，可以给列表中`item`的`Controller`，适用`create()`方式进行注入。以保证每个`item`的数据唯一，同时也增加了`Controller`的复用性



#### 方法之间的差异

##### 特别说明

* `引用`：在`GetInstance`中`_singl`的`Map`字典中存放的叫引用（毕竟官方也这么叫🐶）

  ```dart
  class GetInstance {
    ...
    // Holds references to every registered Instance when using
    /// `Get.put()`
    static final Map<String, _InstanceBuilderFactory> _singl = {};
    ...
  }
  ```

* `实例`：指的的`S`对象的实例

##### _insert()中的一些字段

* `permanent`：在`GetInstance.delete()`时，是否从`_singl`中移除`引用`或`实例`，为`true`时代表什么都不移除

* `fenix`：在`GetInstance.delete()`时，判断是移除`实例`还是`引用`。为`true`时移除`实例`，为`false`时移除`引用`

  > 移除`引用`：需要重新`_insert()`
  >
  > 移除`实例`：不需要重新`_insert()`，下次`find()`会再次创建`实例`

* `isSingleton`：`实例`是否复用，也就是“是否为单例”。无法通过用户自定义，因为在`Get`的相关注入方法中，未将`_insert()`的该参数暴露出来

##### Get的注入方法

`Get`的所有注入方法，对应到`_insert()`方法中各个字段的值。

| 注入方法   | `permanent` | `fenix`   | `isSingleton` |
| ---------- | ----------- | --------- | ------------- |
| put()      | false       | \<false\> | \<true\>      |
| putAsync() | false       | \<false\> | \<true\>      |
| lazyPut()  | \<false\>   | false     | \<true\>      |
| create()   | true        | \<false\> | \<false\>     |

> 备注：被`<>`框起来的都是默认值，无法更改

从上面的表格中可以看出：

* `put()/putAsync()/lazyPut()`只要`find()`所在的路由未被销毁时，都会共享同一个`实例`
* `create()`是每次`find()`都会创建一个`实例`，并且在路由销毁时，也不会去移除在`_singl`中的`引用`



#### 替换依赖中的实例

子类可以替换父类在依赖中的实例，可通过`Get`的`replace()`或`lazyReplace()`方法进行替换。

```dart
class BaseClass {}
class ClassA extends BaseClass {}
class ClassB extends ClassA {}
// 注入ClassA
Get.put<BaseClass>(ClassA());
// 使用replace()替换成ClassB
Get.replace<BaseClass>(ClassB());
final instance = Get.find<BaseClass>(); // 此时instance是ClassB的实例
// 使用lazyReplace()替换成ClassC
class ClassC extends BaseClass {}
Get.lazyReplace<BaseClass>(ClassC());
final instance = Get.find<BaseClass>(); // 此时instance是ClassC的实例
```



#### Bindings

如果担心忘记，或者不知道依赖在什么位置注入合适，我们可以使用`Bindings`去管理`Get`注入的相关方法。

##### 实现Bindings

* 自定义一个`Binding`类实现`Bindings`，然后实现其`dependencies()`方法
* 在`dependencies()`方法内，即可放置`Get`的相关注入操作

```dart
class CustomBinding implements Bindings {
  @override
  void dependencies() {
    Get.lazyPut<Controller>(() => Controller());
    Get.put<Service>(()=> Api());
  }
}
```

##### 在路由中使用

* 命名路由：只需要在`GetPage`中给相应的页面添加`binding`即可
* 普通路由：只需要在调用`Get`的路由跳转方法中添加`binding`参数即可
* 如果想构建`GetMaterialApp`时就触发`Get`注入，只需要在`GetMaterialApp`中配置`initialBinding`即可

```dart
// 命名路由
GetMaterialApp(
	getPages: [
    GetPage(name: '/', page: () => HomePage(), binding: HomeBinding()),
  ],
  ...
)
  
// 普通路由
Get.to(HomePage(), binding: HomeBinding())
  
// 在GetMaterialApp中的GetBuilder调用initState()时被触发
GetMaterialApp(
	initialBinding: CustomBinding(),
  ...
)
```

##### BindingsBuilder

如果你不想单独写一个类去实现`Bindings`，也可以直接使用`BindingsBuilder`，在回调中完成依赖的注入。

```dart
GetMaterialApp(
	getPages: [
    GetPage(
      name: '/', 
      page: () => HomePage(), 
      binding: BindingsBuilder(() {
        Get.lazyPut<Controller>(() => Controller());
        Get.put<Service>(()=> Api());
      }),
    ),
  ],
  ...
)
```



#### SmartManagement

`SmartManagement`是`GetX`中的依赖管理。`GetX`默认情况下会依赖于路由去触发相关依赖移除操作，在页面被销毁时将该页面，所持有的未使用且未设置`permanent`的依赖，从内存中移除。如果想使用其他的管理模式，可以通过在`GetMaterialApp`中配置`SmartManagement`来实现。

> ⚠️在不熟悉`GetX`框架的情况下，不建议也不推荐更改管理模式

```dart
GetMaterialApp(
  smartManagement: SmartManagement.onlyBuilder,
)
```

`SmartManagement`有三种模式：

* `SmartManagement.full`：默认的管理模式。释放未使用且未设置`permanent`的类

* `SmartManagement.onlyBuilder`：只会释放“在`Binddings`中使用`Get.lazyPut()`方式注入，同时在`GetBuilder/GetX`的`init:`传入”的依赖

  > 使用`Get.put()/Get.putAsync()`或其他方法注入的，`SmartManagement`不会对其管理

* `SmartManagement.keepFactory`：和`SmartManagement.full`模式一样，但`keepFactory`会对其`引用`作保留。所以在下次`find()`时只需要重新创建`实例`即可，不需要再走一遍`_insert()`注入流程

  > 如果有多个`Bindings`请勿使用该模式。它仅被用于“没有`Bindins`”或“只有`GetMaterialApp`的`initialBinding`”的情况



#### 为啥在Get.put()之后，在任何地方都能通过Get.find()找到想要的Controller？

* `Get`的`put()/putAsync()/lazyPut()/create()`方法，最终都会通过`GetInstance._insert()`方法，将实例或创建实例的函数包装成`_InstanceBuilderFactory`引用，然后插入到`_singl`的`Map`中

  > `_singl`使用`S`实例的类型和传入的`tag`作为`key`

* 在`Get`的`find()`中，会从`_singl`中找到对应的`_InstanceBuilderFactory`。然后通过`_initDependencies()`方法，判断是否`实例`已经初始化。

  * 如果已经完成初始化，则直接将该`实例`返回

  * 如果未初始化，则调用`_InstanceBuilderFactory`的`getDependency()`完成初始化，然后将`实例`返回

    > `_initDependencies()`方法会根据`_InstanceBuilderFactory`的`isSingleton`，来判断是否需要将其`isInit`改为true，以此确定下次`find()`是否需要构建新的实例

* `Get`的依赖管理：
  * `GetBuilder`和`GetX`都会在被销毁的时候，通过`GetInstance.delete()`去移除`GetInstance`中的依赖
  * 在`GetPageRoute`被销毁时，会向`WidgetBinding.instance`中插入一条帧回调监听，在下一帧开始绘制时，会将之前路由中所存在的不再使用的依赖，通过`GetInstance.delete()`依次从`GetInstance`中移除

![依赖管理时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_gl_6.png)

> 综合上述流程：只要`Get.put()`的页面未被销毁，在其他任意位置都可通过`Get.find()`获取到该对象的依赖





### 其他工具

#### 国际化

###### Translations

`Translations`内部被定义成了一个键值映射的`Map`。如果需要添加自定义翻译，需要创建一个类去扩展`Translations`。

```dart
class TrMessages extends Translations {
  @override
  Map<String, Map<String, String>> get keys => {
    'en_US': {
      'hello': "Hello",
    },
    'zh_CN': {
      'hello': "你好",
    },
  };
}
```

###### 使用

只需要将`.tr`添加到指定的键上，它就会被翻译。然后使用`Get.local`和`Get.fallbackLocale`指定`Locale`，即可完成配置。

```dart
// .tr
Text('hello'.tr)

// GetMaterialApp
GetMaterialApp(
  translations: TrMessages,
  locale: Locale('zh', 'CN'), // 指定语言环境
  fallbackLocale: Locale('en', 'US'), // 指定备用语言环境
  ...
)
```

> 系统语言环境，通过`Get.deviceLocale`获取

带有参数的`Translations`：

```dart
// Map
Map<String, Map<String, String>> get keys => {
  'en_US': {
    'hello': "Hello @name",
  },
  'zh_CN': {
    'hello': "@name，你好",
  },
};

// Text
Text('hello'.trParams({'name': 'howard'}))
```

###### 更改Locale

只需要通过`Get.updateLocale(locale)`即可更改`Locale`，`Translations`会自动使用新的`Locale`。



#### 主题

在`dark`和`light`主题切换时，只需要使用`Get.changeTheme(theme)`方法即可更改主题，但建议在更改前通过`Get.isDarkMode`验证当前主题是否是`dark`主题。

```dart
Get.changeTheme(Get.isDarkMode ? ThemeData.light() : ThemeData.dark());
```



#### GetConnect

`GetConnect`是`GetX`框架提供的网络框架，可用于`http`请求或`websockets`。

> 目前只验证了简单的使用，能够满足大部分网络请求的使用。针对于项目需要根据具体需求，去衡量是否能够使用`GetConnect`替代之前的网络框架

##### 默认配置

只需要自定义一个类去继承`GetConnect`，即可在内部使用`GET/POST/PUT/DELETE/SOCKET`等方法。

```dart
class ApiConnect extends GetConnect {
  // get
  Future<Response> getUser(int id) => get('https://yourapi/users/$id');
  // post
  Future<Response> postUser(Map data) => post('https://yourapi/users', data);
  // socket
  GetSocket userMessages() => socket('https://yourapi/users/socket');
}
```

##### 自定义配置

可以在`GetConnect`的扩展类的`onInit()`方法中，根据需要自定义`httpClient`相关配置。

```dart
class ApiConnect extends GetConnect {
  @override
  void onInit() {
    // 设置默认解码器
    httpClient.defaultDecoder = ...;
    // 设置baseUrl
    httpClient.baseUrl = ...;
    // 请求拦截器，设置headers
    httpClient.addRequestModifier((request) {
      request.headers['OS_TYPE'] = 'iOS';
      return request;
    });
    // 响应拦截器
    httpClient.addResponseModifier<PayloadModel>((request, response) {
      CasesModel model = response.body;
      if (model.data == null) return;
    });
    // 身份验证
    httpClient.addAuthenticator((request) async {
      final response = await get('http://yourapi/token');
      final token = response.body['token'];
      request.headers['Auth-Token'] = '$token';
      return request;
    });
    // 身份验证最大次数（最多验证多少次，即Authenticator最多被执行几次）
    httpClient.maxAuthRetries = 2;
  }
}
```



#### 其他高级用法

```dart
/// 路由相关
// 获取上个路由的名称
Get.previousRoute
// 获取当前路由的原始路由
Get.rawRoute
// 获取Routing，得到后就可访问例如previousRoute、rawRoute等api
Get.routing
// 移除一个路由，内部使用Navigator.removeRoute(route)
Get.removeRoute(route);

/// 弹窗相关
// 检查Snackbar是否打开
Get.isSnackbarOpen
// 检查Dialog是否打开
Get.isDialogOpen
// 检查Bottomsheet是否打开
Get.isBottomsheetOpen

// 检查当前运行应用程序所在的操作系统
GetPlatform.isAndroid
GetPlatform.isIOS
GetPlatform.isMacOS
GetPlatform.isWindows
GetPlatform.isLinux
GetPlatform.isFuchsia

// 检查设备类型
GetPlatform.isMobile
GetPlatform.isDesktop
GetPlatform.isWeb

// 设备宽高，相当于MediaQuery.of(context).size.width
Get.width
Get.height
  
// 全局配置，也可以在GetMaterialApp中配置
Get.config(
  enableLog = true,
  ...
);
```

