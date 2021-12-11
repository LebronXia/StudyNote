# Flutter状态管理-Provider
在开发中经常会遇到状态管理的问题。一般原则是：如果状态跨组件共享，则该状态应该由各个组件共同的父元素来管理。目前已经有很多专门用于状态管理的包了，`Provider`、`Redux`、`bloc`和`Scoped Model`。现在我们就看一下google推荐使用的Provider。

我们要做的是针对项目中登录状态、主题、国际化进行状态管理。

1.添加依赖

像之前一样，在`pubspec.yaml`文件汇总引入

```
  provider: ^3.0.0+1    #跨组件状态共享
```
2.创建数据Model,继承自`ChangeNotifier`

```dart
//共享状态
class ProfileChangeNotifier extends ChangeNotifier{

  Profile get _profile => Global.profile;

  @override
  void notifyListeners() {
    Global.saveProfile();  //保存profile变更
    super.notifyListeners();  ////通知依赖的Widget更新
  }
}

//用户状态
class UserModel extends ProfileChangeNotifier {
  User get user => _profile.user;

  // APP是否登录(如果有用户信息，则证明登录过)
  bool get isLogin => user != null;

  set user(User user){
    if(user?.login != _profile.user?.login){
      _profile.lastLogin = _profile.user?.login;
      _profile.user = user;
      notifyListeners();
    }
  }
}

//App主题状态
class ThemeModel extends ProfileChangeNotifier{
  //默认使用蓝色主题
  ColorSwatch get theme => Global.themes.
  firstWhere((e) => e.value == _profile.theme, orElse: () => Colors.blue);

  //主题改变后，通知其依赖项，新主题立即生效
  set theme(ColorSwatch color){
    if(color != theme){
      _profile.theme = color[500].value;
      notifyListeners();
    }
  }

}

//国际化
class LocaleModel extends ProfileChangeNotifier{

  //获取当前用户的App语言配置Local类，如果为null，则语言跟随系统语言
  Locale getLocale(){
    if(_profile.locale == null) return null;
    var t = _profile.locale.split("_");
    return Locale(t[0], t[1]);
  }

  String get locale => _profile.locale;

  // ignore: non_constant_identifier_names
  set locale(String locale){
    if(locale != _profile.locale){
      _profile.locale = locale;
      notifyListeners();
    }
  }
}
```
拿登录状态来说。这里我们通过`get user`和`get isLogin`把`user`和`islogin`值暴露出来，并提供`set user`方法用于更改数据。

当调用`notifyListeners()`时，它糊通知所有观察者（Consumer）进行刷新。

3.创建顶层共享数据

在单个状态的时候，你可以在顶层这样写。

我们在main方法中初始化全局数据。

```
void main() {
  final userModel = UserModel();
  final textSize = 63;

  runApp(
    Provider<int>.value(
      value: textSize,
      child: ChangeNotifierProvider.value(
        value: userModel,
        child: MyApp(),
      ),
    ),
  );
}

```
通过 `Provider<T>.value `能够管理一个恒定的数据，并提供给子孙节点使用。我们只需要将数据在其 value 属性中声明即可。

如果你提供了多个状态，可以使用`MultiProvider`。

```
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  Global.init().then((e) => runApp(MyApp()));
}

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    //它将主题、用户、语言三种状态绑定到了应用的根上
    //https://blog.csdn.net/unicorn97/article/details/99877867
    return MultiProvider(providers: <SingleChildCloneableWidget>[
      //使用ChangeNotifierProvider来将Model与Widget相关联
      ChangeNotifierProvider.value(value: ThemeModel()),
      ChangeNotifierProvider.value(value: UserModel()),
      ChangeNotifierProvider.value(value: LocaleModel()),
    ],
      //定义监听者Consumer，获取Model的值来更新UI
      child: (),
    );
  }
}
```
4.进行监听Consumer，获取Model来更新UI

```
 	//定义监听者Consumer，获取Model的值来更新UI
      child: Consumer2<ThemeModel, LocaleModel>(
          // ignore: non_constant_identifier_names
          builder: (BuildContext context, themeModel, localeModel, Widget Child){
            return MaterialApp(
              theme: ThemeData(
                primarySwatch: themeModel.theme,
              ),
              //生成标题
              onGenerateTitle: (context){
                //国际化  此时context在Localizations的子树中
                return GmLocalizations.of(context).title;
              },
              home: HomeRoute(),
              locale: localeModel.getLocale(),
              //支持美国英语和中文简体
              supportedLocales: [
                const Locale('en', 'US'),
                const Locale('zh', 'CN'),
              ],
              localizationsDelegates: [
                //本地化接口的多语言实现，
                GlobalMaterialLocalizations.delegate,
                GlobalWidgetsLocalizations.delegate,
                GmLocalizationsDelegate()
              ],

              //监听系统语言切换
              localeResolutionCallback:
                // ignore: missing_return
                (Locale _locale, Iterable<Locale> supportedLocales){
                  if(localeModel.getLocale() != null){
                    //如果已经选定语言，则不跟随系统
                    return localeModel.getLocale();
                  } else {

                    Locale locale;
                    if(supportedLocales.contains(_locale)){
                      locale = _locale;
                    } else {
                      locale= Locale('en', 'US');
                    }
                    return locale;
                  }
                },

                routes: <String, WidgetBuilder>{
                    "login": (context) => LoginRoute(),
                  "language": (context) => LanguageRoute(),
                  "themes": (context) => ThemeChangeRoute(),
                },

            );
          }),
```

Consumer widget 唯一必须的参数就是 `builder`。当 ChangeNotifier 发生变化的时候会调用 builder 这个函数。

Consumer和Consumers的区别:

- Consumer 的 builder 实际上就是一个 Function，它接收三个参数 (BuildContext context, T model, Widget child)。
- 使用方式基本上和 Consumer<T> 一致，只不过范型改为了两个，并且 builder 方法也变成了 Function(BuildContext context, A value, B value2, Widget child)。


5.我们也可以通过Peovider.of来更新数据

虽然我们可以使用 Consumer 来实现这个效果，不过这么实现有点浪费。因为我们让整体框架重构了一个无需重构的 widget。所以这里我们可以使用 Provider.of。

```
 Widget _buildBody(BuildContext context){
    //我们的根widget是MultiProvider，它将主题、用户、语言三种状态绑定到了应用的根上，
    // 如此一来，任何路由中都可以通过Provider.of()来获取这些状态，也就是说这三种状态是全局共享的！
    UserModel userModel = Provider.of<UserModel>(context);
    if(!userModel.isLogin){
      //用户未登录  显示登录按钮
      return Center(
          child: RaisedButton(
            child: Text(GmLocalizations.of(context).login),
            onPressed: (){
              Navigator.of(context).pushNamed("login");
            }

          ),
      );
    }
   }
```
在`LanguageRoute`中

```
class LanguageRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {

    var color = Theme.of(context).primaryColor;
    var localeModel = Provider.of<LocaleModel>(context);
    var gm = GmLocalizations.of(context);

    //语言选项
    Widget _buildLanguageItem(String lan, value){
      return ListTile(
        title: Text(
          lan,
          style: TextStyle(color: localeModel.locale == value ? color : null),
        ),
        trailing:
            //trailing 可以放 Icon 也可以放别的 Widget
        localeModel.locale == value ? Icon(Icons.done, color: color,) : null,
        onTap: (){
          // 更新locale后MaterialApp会重新build
          localeModel.locale = value;
        },
      );
    }

    return new Scaffold(
      appBar: new AppBar(
        title: new Text(gm.language),
      ),
      body: ListView(
        children: <Widget>[
          _buildLanguageItem("中文简体", "zh_CN"),
          _buildLanguageItem("English", "en_US"),
          _buildLanguageItem(gm.auto, null),
        ],
      ),
    );
  }
}
```
设置类似,这里我们可以使用 Provider.of，并且将 listen 设置为 false。在 build 方法中使用上面的代码，当 notifyListeners 被调用的时候，并不会使 widget 被重构。
```
 Provider.of<Counter>(context, listen: false).increment(),
```

