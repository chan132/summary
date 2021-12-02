## Fish Redux框架的使用方式和应用场景



#### 说明

* 文中相关时序图由我个人结合`fish_redux`源码自行总结得出，仅代表我个人理解，仅供参考。
* 如需转载请注明出处，如有疑问可随时提issue
* 本文基于`fish_redux: ^0.3.7`
* GitHub：[fish_redux](https://github.com/alibaba/fish-redux)





### 基础介绍

`fish_redux`是一个视图、状态、逻辑分层很明显的数据管理框架，适用于中大型的复杂应用。



#### 分层

我在使用`fish_redux`框架时，使用了`Android Studio`的`FishReduxTemplate`插件，一键生成了页面所拆分的所有文件。共分成了`page`、`view`、`state`、`action`、`effect`、`reducer`六个文件。

> 注意：在后面的文章中，我们暂且将这六个文件称之为六个层面

* `page`：页面入口。负责注册`effect`、`reducer`、`component`、`adapter`的功能，以及完成类似像`middleware`中间件的其他配置

  ```dart
  class TestPage extends Page<TestState, Map<String, dynamic>> {
    TestPage()
        : super(
              initState: initState,
              effect: buildEffect(),
              reducer: buildReducer(),
              view: buildView,
              dependencies: Dependencies<TestState>(
                  adapter: null,
                  slots: <String, Dependent<TestState>>{
                  }),
              middleware: <Middleware<TestState>>[
              ],);
  }
  ```

  > 默认是每次数据发生变更时组件都会刷新，我们可以通过给`Component`配置`shouldUpdate`参数，去更精确的控制是否刷新组件

* `view`：写页面布局的位置。里面是一个输出`Widget`且与`Context`无关的函数

  ```dart
  /// state: 数据层，页面需要的变量都写在state层
  /// dispatch: 调度器，调用action层中的方法，从而去回调effect、reducer层的方法
  /// viewService: 可以通过其buildComponent('组件名称')方法，调用我们封装的相关组件。可以通过其buildAdapter()，直接创建适配器
  Widget buildView(TestState state, Dispatch dispatch, ViewService viewService) {
    return Scaffold(
      appBar: AppBar(title: Text('Test Page')),
      body: Center(child: Text('current num: ${state.num}')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => dispatch(TestActionCreator.goEffect()),
        child: Icon(Icons.add),
      ),
    );
  }
  ```

* `state`：存放子模块变量的位置。可以在`state`中初始化变量、接受上个页面的参数

  ```dart
  class TestState implements Cloneable<TestState> {
    // 可以定义在页面展示的变量
    int? num;
  
    @override
    TestState clone() => TestState()..num = num;
  }
  
  // 初始化变量
  TestState initState(Map<String, dynamic> args) => TestState()..num = 0;
  ```

* `action`：定义和中转事件的位置。一个事件对应于一个枚举，枚举字段是`effect`、`reducer`层标识的入口

  ```dart
  // 事件名称：一个事件对应于一个枚举，枚举字段是effect、reducer层标识的入口
  enum TestAction { actionGoEffect, actionGoReducer }
  
  // 在此定义中转方法
  class TestActionCreator {
    static Action goEffect() => const Action(TestAction.actionGoEffect);
    
    static Action goReducer(int num) {
      // 1. 通过模版生成的代码有const修饰，如果使用payload传参会报错，所以需要去掉const修饰
      //return const Action(TestAction.action);
      // 2. 通过Action类中的payload将参数传到effect、reducer中去，payload的类型是dynamic
      return Action(TestAction.actionGoReducer, payload: num);
    }
  }
  ```

* `effect`：处理业务逻辑的位置。（如：网络请求）

  ```dart
  Effect<TestState> buildEffect() {
    return combineEffects(<Object, Effect<TestState>>{
      TestAction.actionGoEffect: _onAction,
    });
  }
  
  /// 对应action的处理方法
  ///
  /// action: 可以通过Action拿到payload的值
  /// ctx: 可以拿到State的变量，还可以通过ctx调用dispatch方法，调用Action中的方法【在这里调用dispatch方法，一般都是把处理好的数据，通过action中的方法传到reducer层中更新数据】
  void _onAction(Action action, Context<TestState> ctx) {
    int _num = ctx.state.num + 1;
    return ctx.dispatch(TestActionCreator.goReducer(_num));
  }
  ```

  > 在`effect`中，除开通过`ctx.dispatch()`发送事件出去，主要是当前`Store`内消费事件。也可以通过`ctx.broadcast()`发送广播，就如其名字一样其他页面或组件的`Store`也能收到。

* `reducer`：更新页面数据的位置。主要用来更新数据，也可以写一些简单的逻辑或者和数据有关的逻辑操作

  ```dart
  Reducer<TestState> buildReducer() {
    return asReducer(
      <Object, Reducer<TestState>>{
        TestAction.actionGoReducer: _onAction,
      },
    );
  }
  
  /// 对应action的处理方法
  ///
  /// state: state参数经常使用的就是clone方法
  /// action: 可通过action拿到payload参数
  TestState _onAction(TestState state, Action action) {
    // 克隆一个新的state对象
    final TestState newState = state.clone();
    newState..num = action.payload;
    return newState;
  }
  ```
  
  > `asReducer()`默认情况下每次处理`Action`时，都会遍历一次所有的`Reducer`。通过`filterReducer()`可以根据某些条件过滤一些`Reducer`，从而提高效率（如：我们可以根据`Action`的`type`进行过滤，只有当`type`一致时才会触发`Reducer`对应`Action`的处理方法）



#### 数据在各个层面如何流转

* 在路由构建页面时，会触发`Page`的构造函数。最终响应一个`_PageWidget`组件回去
* 由于`_PageWidget`是一个`StatefulWidget`，所以在其`initState()`触发时，会执行在`page`中传入的`initState()`方法，构建出最初的`state`
* 在`_PageWidget`组件的`_PageState`触发`build()`时，最终会触发在`page`中的`initView()`方法，构建出`view`
* 在`view`中通过`dispatch()`方法，发送一个`effect`对应的`action`出去。只要`effect`监听了该`action`，则会触发相应的方法去处理该事件
* 在`effect`中处理完对应事件后，会将结果放在`reducer`对应的`action`中，通过`dispatch()`方法发送出去。只要`reducer`监听了该`action`，则会触发相应的方法去`clone`一个`State`出来，将结果更新到页面中去

![数据流转流程图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_frl_1.png)

> * 其实流程图中的`action`也算是一个流程，不是一个判断，只是图中画成判断好理解一些
> * 当然我们所有的`action`，不一定都得先到`effect`中，然后再到`reducer`（如：简单的数据操作，也可以直接去到`reducer`）
> * 我们一般在`effect`的事件处理方法中，去跳转下个页面。因为`effect`中更容易拿到`context`（如：我们在`effect`中拿到页面返回的数据后，再通过`action`到`reducer`中，通过`State`更新到页面中）





### 路由管理

路由这块，`fish_redux`没有特殊的进行管理，只不过在命名路由的方式上不同于原生方式。但和`GetX`有类似的地方，在`MaterialApp`的参数中，通过`onGenerateRoute`参数传入了自定义生成路由的方式。由于普通路由跳转方式和`Fluter`原生跳转方式相同，所以下面我们主要通过命名路由，来讲述`fish_redux`的路由管理。



#### 命名路由的定义

在`fish_redux`中路由去构建页面时，最终会通过`AbstractRoutes.buildPage()`方法，生成对应的页面`Widget`。所以我们在定义命名路由时，也会使用`AbstractRoutes`的实现类`PageRoutes`去完成。其内部需要传入一个“路由名关联对应页面”的`Map`进行维护。

```dart
// 路由管理
class RouteConfig {
  static const String homePage = 'home';
  static const String secondPage = 'second';
  
  static final AbstractRoutes routes = PageRoutes(
  	pages: <String, Page<Object, dynamic>> {
      RouteConfig.homePage: HomePage(),
      RouteConfig.secondPage: SecondPage(),
    }
  );
}

// MaterialApp
MaterialApp(
  home: RouteConfig.routes.buildPage(RouteConfig.homePage, null),// 默认启动页
  onGenerateRoute: (settings) {
    return MaterialPageRoute(
      builder: (context) => RouteConfig.routes.buildPage(settings.name, settings.arguments),
      settings: settings,
    );
  },
  ...
)
```



#### 命名路由的使用

在`fish_redux`中命名路由的使用，就是使用`Flutter`的原生`Navigator`完成路由跳转，即`Navigator`中`**Named`相关的方法。一般我们在`effect`中的相关`action`处理方法中，使用`ctx`获取`context`然后完成跳转

```dart
// 普通页面跳转
Navigator.of(ctx.context).pushNamed(RouteConfig.secondPage);

// 页面传参，其中arguments是一个Object
Navigator.of(ctx.context).pushNamed(RouteConfig.secondPage, arguments: {'value': 'blablabla...'});
// 下个页面，可在initState()中通过args获取参数
SecondState initState(Map<String, dynamic> args) {
  return SecondState()..msg = args["value"];
}

// 接收页面返回的结果
var result = await Navigator.of(ctx.context).pushNamed(RouteConfig.secondPage);
```



#### 路由跳转时的加载流程

![路由跳转时序图](https://raw.githubusercontent.com/chan132/summary/main/images/flutter/img_frl_2.png)

* `Flutter`原生命名路由的跳转，都会通过`NavigatorState._routeNamed()`，去生成一个`Route`然后交给原生路由管理。而在`_routeNamed()`内部最终又调用的是`MaterialApp.onGenerateRoute()`生成`Route`
* 而在`fish_redux`中，和`GetX`一样采用了在`MaterialApp`传入了`onGenerateRoute`自定义函数，生成自定义路由。只不过`fish_redux`是传入的`MaterialPageRoute`，并没有自己实现路由的相关管理
* 在`Route`构建页面时，会触发`MaterialPageRoute.builder()`。在其内部也是直接执行了该页面的`buildPage()`方法，最终响应了一个`_PageWidget`
* 由于`_PageWidget`是一个`StatefulWidget`，所以在其`initState()`被触发时，最终会触发`Page`初始化传入的`initState`。在其`build()`被触发时，最终也会触发`Page`初始化传入的`buildView()`





### ListView

在`fish_redux `中使用`ListView`和原生有很大区别，`fish_redux`为了解决传统`ListView`的`分治`问题，在其中引入了类似于`Android`中`Adapter`的概念。

* 一个`ListView`对应于一个`Adapter`，一个`Adapter`可以由多个`Component`和`Adapter`组合而成
* `Adapter`及其子`Adapter`的生命周期，是和`ListView`一致的。而在`Adapter`配置的子`Component`，则与其所对应的`WidgetState`一致



#### 简单用法

这边我们采用一个简单的案例来讲述其用法，然后还是使用`Android Studio`的`FishReduxTemplate`插件，去生成响应的文件。

案例：我们需要将服务端的数据以列表形式呈现给用户。

> 案例描述：我们在页面`initState`时请求`restApi`，结果响应回来之后刷新页面的数据源，进而更新页面列表的展示

##### 创建item的Component

`Component`和`Page`有着相同的分层模型，但我们在使用时并不是每个层面都能用得着，所以在此次案例中之创建我们所使用的`state`和`view`。

> 如果说涉及到`item`内部需要做相应的刷新，也可以按照`page`的那种方式，加上`state`、`action`、`effect`、`reducer`相关文件。然后使用其`dispatch()`方法发送对应的`action`去处理

* 在`state`中我们只需要放入我们的`Model`即可

  ```dart
  class ItemState implements Cloneable<ItemState> {
    ItemModel? item;
  
    ItemState({this.item});
  
    @override
    ItemState clone() => ItemState()..item = item;
  }
  
  ItemState initState(Map<String, dynamic> args) => ItemState();
  ```

* 然后在`view`中根据`state`中的`ItemModel`，构建对应的视图即可

  > 由于`view`不是此次案例的重点，所以不贴源码出来了🐶

* 在`Component`文件中，由于此次案例未涉及到嵌套`Adapter`的相关情况，所以不需要做任何的改动

  ```dart
  class ItemComponent extends Component<ItemState> {
    ItemComponent()
        : super(
            view: buildView,
            dependencies: Dependencies<ItemState>(
              adapter: null,
              slots: <String, Dependent<ItemState>>{},
            ),
          );
  }
  ```

##### 创建Adapter

`Adapter`和`Component`一样，也和`Page`有着相同的分层模型。同样在此次案例中我们只使用到了`Adapter`。

* 在创建`Adapter`时我们选择继承了`SourceFlowAdapter`，里面的泛型需要填入页面的，并在其构造函数的`pool`参数中定义了`item`相关样式，即完成了`Adapter`与`Component`的关联

  ```dart
  // 此处需要传入页面的State，并且页面的State需要继承自MutableSource
  class ListAdapter extends SourceFlowAdapter<ListState> {
    static const String itemStyleA = 'item_style_a';
  
    ListAdapter()
        : super(
          	// 定义item样式
            pool: <String, Component<Object>>{
              itemStyleA: ItemComponent(),
              // 如果还有其他样式，可以在后面追加，如下：
              //itemStyleB: ItemBComponent(),
            },
          );
  }
  ```


##### 创建List对应的页面

这个步骤跟前面创建普通的`Page`类似，由于此次案例涉及到网络请求及页面刷新，所以`Page`的六个层面我们都需要用到。

* 在`page`文件中，通过其构造函数中的`dependencies`参数，去绑定前面创建好的`Adapter`，以完成`Page`和`Adapter`的绑定

  ```dart
  class ListPage extends Page<ListState, Map<String, dynamic>> {
    ListPage()
        : super(
            initState: initState,
            effect: buildEffect(),
            reducer: buildReducer(),
            view: buildView,
            dependencies: Dependencies<ListState>(
              // 绑定Adapter
              adapter: NoneConn<ListState>() + ListAdapter(),
              slots: <String, Dependent<ListState>>{},
            ),
            middleware: <Middleware<ListState>>[],
          );
  }
  ```

  > 暂时不清楚为啥要使用`NoneConn<ListState>() + ListAdapter()`作为`adapter`，但官方推荐这样做！

  ```dart
  /// Use [adapter: NoneConn<T>() + Adapter<T>()] instead of [adapter: Adapter<T>()],
  /// Which is better reusability and consistency.
  Dependencies({
    this.slots,
    this.adapter,
  }) : assert(adapter == null || adapter.isAdapter(), 'The dependent must contains adapter.');
  ```

* 在`state`文件中，放入`ListView`的数据源，然后实现继承自`MutableSource`的方法即可

  ```dart
  class ListState extends MutableSource implements Cloneable<ListState> {
    /// 1. 这个地方一定要注意，List里面的泛型需要定义为ItemState
    /// 2. 如何更新列表数据：只需要更新这个items里面的数据，列表数据就会相应更新
    /// 3. 多样式的情况：请写出List<Object> items;
    List<ItemState>? items;
  
    @override
    ListState clone() {
      return ListState()..items = items;
    }
  
    /// 使用上面定义的List，继承MutableSource，就完成了列表和item的绑定
    @override
    Object getItemData(int index) => items![index];
  
    /// 如果是多样式的情况，在这个地方可以根据items![index]判断其State类型，然后返回其对应Component的样式名称即可
    @override
    String getItemType(int index) => ListItemAdapter.item_style;
  
    @override
    int get itemCount => items!.length;
  
    /// 如果是多样式的情况，在赋值时则不需要转换类型
    @override
    void setItemData(int index, Object data) => items![index] = data as ItemState;
  }
  
  ListState initState(Map<String, dynamic> args) => ListState();
  ```

* 在`view`文件中，没有数据源时我们可以展示`loading`状态。数据源存在时，则通过`viewService.buildAdapter()`获取页面绑定的`Adapter`，然后将其`itemBuilder`和`itemCount`传给`ListView.builder()`完成`ListView`的展示

  ```dart
  Widget buildView(ListState state, Dispatch dispatch, ViewService viewService) {
    return Scaffold(
      appBar: AppBar(title: Text('List Page')),
      body: state.items == null
          ? Center(child: CircularProgressIndicator())
          : ListView.builder(
              itemBuilder: viewService.buildAdapter().itemBuilder,
              itemCount: viewService.buildAdapter().itemCount,
            ),
    );
  }
  ```

* 在`effect`文件中，我们通过监听`Lifecycle.initState`的事件，即可在页面`initState`时请求页面数据

  ```dart
  Effect<ListState> buildEffect() {
    return combineEffects(<Object, Effect<ListState>>{
      /// 进入页面就执行初始化操作
      Lifecycle.initState: _init,
    });
  }
  
  void _init(Action action, Context<ListState> ctx) async {
    // 1. 发送请求
    final response = await RestApi.getList();
    // 2. 解析结果
    List<ItemModel> allItem = [];
    for (dynamic data in response) {
      allItem.add(ItemModel.fromJson(data));
    }
    // 3. 构建符合要求的列表数据
    List<ItemState> items = List.generate(allItem.length, (index) => ItemState(item: allItem[index]));
    // 4. 通知更新数据
    ctx.dispatch(ListActionCreator.updateItem(items));
  }
  ```

  > 此处的`Lifecycle`包含的事件，即`StatefulWidget`中`State`的全部回调：`initState`、`didChangeDependencies`、`build`、`didUpdateWidget`、`deactivate`、`dispose`

* 在`action`文件中，只是将`updateItem`事件转发到`reducer`中去

  > 由于`action`文件跟之前`page`的`action`无太大区别，所以此处不再贴代码🐶

* 在`reducer`文件中，直接`clone`一个`State`出来，然后将`action`中的`payload`更新到`newState`中去，即可完成页面列表的更新

  > 由于`reducer`文件跟之前`page`的`reducer`无太大区别，所以此处不再贴代码🐶



#### Adapter

官方为`Adapter`提供了四种创建方式，分别是`DynamicFlowAdapter`、`StaticFlowAdapter`、`SourceFlowAdapter`、`CustomAdapter`。

##### DynamicFlowAdapter

接受`Model数组`类型的数据驱动。需要注意的是构造函数中的`pool`和`connector`中，通过唯一标识符进行标记时，两边的唯一标识符要一致。

```dart
class ListGroupAdapter extends DynamicFlowAdapter<AdapterTestState> {
  ListGroupAdapter()
      : super(pool: <String, Component<Object>>{
          'header': ListHeaderComponent(),
          'cell': ListCellComponent(),
          'footer': ListFooterComponent()
        }, connector: ListGroupConnector());
}

class ListGroupConnector extends ConnOp<AdapterTestState, List<ItemBean>> {
  @override
  List<ItemBean> get(AdapterTestState state) {
    List<ItemBean> items = [];
    if (state.allItem.length == 0) return items;
    items.add(ItemBean('header', state.headerModel));
    for (var model in state.allItem) {
      items.add(ItemBean('cell', model));
    }
    items.add(ItemBean('footer', state.footerStr));
    return items;
  }
}
```

> 该`Adapter`已经被弃用了，官方建议使用`SourceFlowAdapter`替换

```dart
/// template is a map, driven by array
/// Use [SourceFlowAdapter] instead of [DynamicFlowAdapter]
/// see in example
@deprecated
class DynamicFlowAdapter<T> extends Logic<T> with RecycleContextMixin<T> {...}
```

##### StaticFlowAdapter

接受`Map`类型的数据驱动。该`Adapter`接收一个`Dependent`数组，每个`Dependent`可以是`Component`或者`Adapter + Connector<T, P>`的组合。

```dart
class TestStaticAdapter extends StaticFlowAdapter<PageState> {
  TestStaticAdapter()
    : super(
      slots: [
        TestComponent().asDependent(TestComponenConnectt()),
        TestAdapter().asDependent(TestAdapterConnect()),
      ],
    );
}
```

##### SourceFlowAdapter

跟`DynamicFlowAdapter`一样，也是接受`Model数组`类型的数据驱动，但是不需要再自定义`connector`。也就是在最开始的案例中，为什么我们使用了`SourceFlowAdapter`，然后也可以在`State`中可以使用唯一标识符获取对应的组件。

> 由于前面案例中我们使用了该`Adapter`，所以此处我们不贴代码🐶

##### CustomAdapter

自定义实现`Adapter`，按照自己的想要的方式完成数据驱动。

> 案例相见：[fish_redux官方示例](https://github.com/alibaba/fish-redux/blob/master/doc/concept/custom-adapter-cn.md)



#### Component

`Component`是对视图展现和逻辑功能的封装。我们可以在`Adapter`中绑定使用，也可以在页面中直接绑定使用。在`Adapter`中使用上个案例已经讲了，接下来我们主要看页面中如何使用。

##### 建立页面与Component的关联

* 在页面中使用`Component`，首先需要将`Component`的`State`作为变量放入页面的`State`中

  ```dart
  class TestState implements Cloneable<TestState> {
    ComponentState? componentState;
  
    @override
    TestState clone() => TestState()..componentState = componentState;
  }
  
  TestState initState(Map<String, dynamic> args) => TestState()
    ..componentState = ComponentState(title: 'TestComponent', text: 'TestComponent');
  ```

* 然后创建`Connector`将页面`State`和`Component`的`State`进行关联，以便于在页面中去更改`Component`中的`State`

  ```dart
  class ComponentConnector extends ConnOp<TestState, ComponentState>
      with ReselectMixin<TestState, ComponentState> {
    @override
    AreaItemState computed(TestState state) => state.componentState!.clone();
  
    @override
    void set(TestState state, ComponentState subState) => state.componentState = subState;
  }
  ```

* 最后在`page`中，通过构造函数中的`dependencies`参数绑定`Component`

  ```dart
  class TestPage extends Page<TestState, Map<String, dynamic>> {
    AreaPage()
        : super(
            initState: initState,
            reducer: buildReducer(),
            view: buildView,
            dependencies: Dependencies<TestState>(
                adapter: null,
                slots: <String, Dependent<TestState>>{
                  'component': ComponentConnector() + TestComponent(),
                }),
            middleware: <Middleware<TestState>>[],
          );
  }
  ```

##### 在页面中使用

* 在`view`中使用时，直接使用`viewService.buildComponent()`方法，传入绑定时的唯一标识符，即可完成`Component`的创建

  ```dart
  Center(
  	child: viewService.buildComponent('component'),
  )
  ```

* 如果需要更改`Component`的状态时，只需要跟普通事件一样，在`reducer`中处理时`clone`一个页面的`State`，然后对其`ComponentState`变量进行操作即可

  ```dart
  Reducer<TestState> buildReducer() {
    return asReducer(
      <Object, Reducer<TestState>>{
        TestAction.change: _onAction,
      },
    );
  }
  
  TestState _onAction(TestState state, Action action) {
    final TestState newState = state.clone();
    // 改变TestComponent中的State
    newState.componentState!.text = 'ComponentState: ${Random().nextInt(1000)}';
    return newState;
  }
  ```





### 全局状态

在页面的`state`中写的变量都是在页面中使用的，而我们在开发中避免不了会有全局状态的情况。在`fish_redux`中是通过`Store`去管理一个全局`State`方式实现的，下面我们按照官方更改`ThemeColor`的案例去逐步讲解如何使用。

> 首先我们需要在项目代码目录下创建一个`store`文件夹，然后在其中创建出`state`、`action`、`reducer`、`store`四个文件



#### 定义全局状态

定义`State`是为后续`Store`方便管理，定义`action`和`reducer`是为了后续更改全局状态而设计。

##### 定义State

* 首先需要在`state`文件中定义一个存放全局状态的`Model`，然后定义一个`GlobalBaseState`抽象类，在其中放入我们定义的`Model`。
* 然后定义一个`GlobalState`去实现`GlobalBaseState`，方便后续在`Store`中对其管理。

```dart
abstract class GlobalBaseState {
  StoreModel? storeModel;
}

/// 存放全局状态的Model
///
/// 如有需要增加"全局状态"字段，则在这个Model中新增即可
class StoreModel {
  Color? themeColor;

  StoreModel({this.themeColor});
}

class GlobalState implements GlobalBaseState, Cloneable<GlobalState> {
  /// storeModel变量必须在这实例，不然引用该变量中的字段，会出错
  ///
  /// 这个地方给全局状态赋初始值，应当从缓存、SharedPreference、数据库中取出，以表明为用户选择的全局状态
  @override
  StoreModel? storeModel = StoreModel(themeColor: Colors.lightBlue);

  @override
  GlobalState clone() => GlobalState();
}
```

##### 更改全局状态

* 跟在页面中的`action`一样，在`action`文件中定义好事件名称和方法

  ```dart
  enum GlobalAction { changeThemeColor }
  
  class GlobalActionCreator {
    static Action onChangeThemeColor(Color color) => Action(GlobalAction.changeThemeColor, payload: color);
  }
  ```

* 然后在`reducer`文件中监听更改全局状态的事件，然后在里面放入更改状态的相关逻辑

  ```dart
  Reducer<GlobalState> buildReducer() {
    return asReducer(
      <Object, Reducer<GlobalState>>{
        GlobalAction.changeThemeColor: _onAction,
      },
    );
  }
  
  GlobalState _onAction(GlobalState state, Action action) => state.clone()..storeModel!.themeColor = action.payload;
  ```



#### Store管理

定义`Store`是为了后续可以在外部通过其操作全局状态，然后在`PageRoutes`中通过`visitor`参数，将页面的`State`和全局状态的`State`建立联系。

##### 定义Store

在`store`文件中，通过`createStore()`方法将`GlobalState`和`reducer`传入，创建一个全局的`Store`。

```dart
class GlobalStore {
  static Store<GlobalState>? _globalStore;
  static Store<GlobalState> get store => _globalStore ??= createStore<GlobalState>(GlobalState(), buildReducer());
}
```

##### 建立关联

* 在页面的`State`实现`GlobalBaseState`，就相当于页面拥有了全局状态

  ```dart
  class TestState implements GlobalBaseState, Cloneable<TestState> {
    int? num;
  
    @override
    StoreModel? storeModel;
  
    @override
    TestState clone() => TestState()
      ..num = num
      ..storeModel = storeModel;
  }
  ```

* 创建一个`PageRoutes`的`visitor`类型的方法，在其中完成将全局状态同步到页面`State`中去的相关逻辑

  ```dart
  class StoreConfig {
    static visitor(String path, Page<Object, dynamic> page) {
      /// 全局状态管理：只有特定范围的Page（State继承了全局状态），才需要建立和AppStore的连接关系
      if (page.isTypeof<GlobalBaseState>()) {
        /// 建立AppStore驱动PageStore的单向数据连接
        /// param1：AppStore
        /// param2：当AppStore.state变化时，PageStore.state该如何变化
        page.connectExtraStore(GlobalStore.store, _updateState);
      }
    }
  
    /// 全局状态更新，即将全局状态同步到页面State中去
    static Object _updateState(Object pageState, GlobalBaseState appState) {
      // 1. 如果pageState没有实现GlobalBaseState，或者不是Cloneable的情况，则直接返回当前pageState
      if (!(pageState is GlobalBaseState) || !(pageState is Cloneable)) return pageState;
      // 2. clone一个新的pageState
      GlobalBaseState _newPageState = (pageState as Cloneable).clone() as GlobalBaseState;
      // 3. 首次当前pageState的storeModel为null的情况，将全局状态同步到新的pageState，并返回
      if (pageState.storeModel == null) return _newPageState..storeModel = appState.storeModel;
      // 4. 判断当前pageState的全局状态字段值是否需要更新
      // >> 如果还有其他字段也需要更新，则按照下方相同逻辑去处理即可
      if (pageState.storeModel!.themeColor != appState.storeModel!.themeColor) {
        _newPageState.storeModel!.themeColor = appState.storeModel!.themeColor;
      }
      return _newPageState;
    }
  }
  ```

* 在主入口定义路由的`PageRoutes`中，传入我们刚刚定义好的`visitor`方法，即完成了全局状态与页面之间的关联

  ```dart
  static final AbstractRoutes routes = PageRoutes(
    visitor: StoreConfig.visitor,
    pages: <String, Page<Object, dynamic>>{
      RoutesConfig.homePage: HomePage(),
      ...
    },
  );
  ```



#### 在页面中使用

在页面中跟之前使用`State`中的变量的使用方式一样。只不过去更改全局变量时，需要使用`Store`去更改`GlobalState`中的变量。

```dart
Widget buildView(TestState state, Dispatch dispatch, ViewService viewService) {
  return Scaffold(
    appBar: AppBar(title: Text('Test Page'), backgroundColor: state.storeModel?.themeColor),
    body: Center(child: Text('current num: ${state.num}')),
    floatingActionButton: FloatingActionButton(
      onPressed: () => GlobalStore.store.dispatch(GlobalActionCreator.onChangeThemeColor()),
      child: Icon(Icons.add),
    ),
  );
}
```

