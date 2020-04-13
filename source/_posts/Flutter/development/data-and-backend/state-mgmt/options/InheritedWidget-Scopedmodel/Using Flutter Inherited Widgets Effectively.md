---
layout: post
title:  "(译)高效地使用Inherited widget"
date:   2019-04-18 13:55:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - Inherited Widget
---

[原文链接](https://ericwindmill.com/posts/inherited_widget/)

如果你之前就使用过Flutter,你可能遇到过随处可见的不同类的`of`方法：

<!--more-->


```dart
Theme.of(context).textTheme
MediaQuery.of(context).size
```
这些widget(Theme,MediaQ)是`Inherited widget`。在你应用中几乎所有地方，你都可以访问你的主题，因为它们是继承而来的。

在Flutter中，sdk的每个部分都是暴露给开发者的，所以你自己可以利用`Inherited widget`。 你可以把自定义的InheritedWidget作为内置的中央状态存储，与Redux仓库或Vue的Vuex仓库类似。

当你建立了好像这样的仓库之后，你将能够做到像这样的事情：

```
class RedText extends StatelessWidget {
  // ...
  Widget build(BuildContext context) {
    var state = StateContainer.of(context).state;
    return new Text(
      state.user.username,
      style: const TextStyle(color: Colors.red),
    );
  // ...
```

## 把状态向上抬

当你使用一个`Inherited widget`作为你的状态管理工具，你可能要依赖于叫做`把状态向上抬`架构模式。

考虑下当你新建一个项目时生成的入门Flutter应用。如果你想要把应用分成两个页面，一个展示数字，一个允许你对数字进行更改。突然之间，这个简单的应用就变得令人困惑了。每当变更路由的时候，你需要把这个状态片传过来传传去。

`Inherited widget`通过让整个widget树能访问相同的状态片来解决这个问题。

![](http://res.cloudinary.com/ericwindmill/image/upload/v1518974500/flutter_by_example/medium_tree.png)

有关不同Flutter架构概念的超级精彩详细解释,请看[Brain Egan在 DartConf2018中的演讲](https://www.youtube.com/watch?v=zKXz3pUkw9A&t=1467s)。只是不要看太多，不然你就会被被说服去使用flutter_redux，然后你就一点都不关心这篇文章了。

把状态向上抬来这种模式相较于Redux之类的优势是，使用Inherited Widget的设置和使用非常简单。

注意：毫无疑问，我是Redux、Vuex以及所有‘ux’之类的东西的粉丝。这只是你工具箱的另一件工具，毕竟杀鸡焉用牛刀。

## 何必呢?

此时，您可能会问为什么要使用InheritedWidget。为什么不仅仅应用程序根那里架`stateful widget`呢？

对没错，这就是这里接下来要做的。`InheritedWidget`与`stateful widget`一起使用，并允许您将StatefulWidgets的状态传递给其所有祖先。它是一个实用的widget。因此，您不必在每个类中都写上代码才能把状态传递给其后代。

## 建立一个样板应用

关于这个例子，让我创建一个简单的应用：
![](http://res.cloudinary.com/ericwindmill/image/upload/v1523742041/blog_posts/inherited_test.gif)

基本上这个应用程序的状态被抬到根Widget之上，当你提交表单时，它会在该`inherited widget`上调用`setState`，这个步骤会告诉主页面有新的信息要渲染。

## 1.Material App 根目录

这只是您标准的Flutter应用程序设置部分：

```
void main() {
  runApp(new UserApp());
}

class UserApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      home: new HomeScreen(),
    );
  }
}

```

## 2 HomeScreen Widget

现在这也是非常基本的。这只是当好东西要来的时候你要遵从的样板

```dart
class HomeScreen extends StatefulWidget {
  @override
  HomeScreenState createState() => new HomeScreenState();
}

class HomeScreenState extends State<HomeScreen> {
  
  Widget get _logInPrompt {
    return new Center(
      child: new Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: <Widget>[
          new Text(
            'Please add user information',
            style: const TextStyle(fontSize: 18.0),
          ),
        ],
      ),
    );
  }
  
  // All this method does is bring up the form page.
  void _updateUser(BuildContext context) {
    Navigator.push(
      context,
      new MaterialPageRoute(
        fullscreenDialog: true,
        builder: (context) {
          return new UpdateUserScreen();
        },
      ),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return new Scaffold(
      appBar: new AppBar(
        title: new Text('Inherited Widget Test'),
      ),
      body: _logInPrompt,
      floatingActionButton: new FloatingActionButton(
        onPressed: () => _updateUser(context),
        child: new Icon(Icons.edit),
      ),
    );
  }
}
```
## 3.最后，目前没有做任何事情的表单页面

```dart
class UpdateUserScreen extends StatelessWidget {
  static final GlobalKey<FormState> formKey = new GlobalKey<FormState>();
  static final GlobalKey<FormFieldState<String>> firstNameKey =
  new GlobalKey<FormFieldState<String>>();
  static final GlobalKey<FormFieldState<String>> lastNameKey =
  new GlobalKey<FormFieldState<String>>();
  static final GlobalKey<FormFieldState<String>> emailKey =
  new GlobalKey<FormFieldState<String>>();

  const UpdateUserScreen({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    
    return new Scaffold(
      appBar: new AppBar(
        title: new Text('Edit User Info'),
      ),
      body: new Padding(
        padding: new EdgeInsets.all(16.0),
        child: new Form(
          key: formKey,
          autovalidate: false,
          child: new ListView(
            children: [
              new TextFormField(
                key: firstNameKey,
                style: Theme.of(context).textTheme.headline,
                decoration: new InputDecoration(
                  hintText: 'First Name',
                ),
              ),
              new TextFormField(
                key: lastNameKey,
                style: Theme.of(context).textTheme.headline,
                decoration: new InputDecoration(
                  hintText: 'Last Name',
                ),
              ),
              new TextFormField(
                key: emailKey,
                style: Theme.of(context).textTheme.headline,
                decoration: new InputDecoration(
                  hintText: 'Email Address',
                ),
              )
            ],
          ),
        ),
      ),
      floatingActionButton: new FloatingActionButton(
        child: new Icon(Icons.add),
        onPressed: () {
          final form = formKey.currentState;
          if (form.validate()) {
            var firstName = firstNameKey.currentState.value;
            var lastName = lastNameKey.currentState.value;
            var email = emailKey.currentState.value;

            // Later, do some stuff here

            Navigator.pop(context);
          }
        },
      ),
    );
  }
}
```
[boilder plate的Github Gist](https://gist.github.com/ericwindmill/32e73cc1fbf65114b5aa875500395f5a)

## pt2: 添加`Inherited widget`的功能

### 1.StateContainer以及InheritedStateContainer Widgets

创建一个新的叫做state_container.dart文件。这是所有事情发生的地方。

首先，在该文件中，创建一个名为User的简单类。在真实的应用程序中，这可能是一个更大的类AppState，您可以在其中保存要在整个应用程序中访问的所有属性。

```dart
lass User {
  String firstName;
  String lastName;
  String email;

  User(this.firstName, this.lastName, this.email);
}
```

InheritedWidget通过连接到StatefulWidget作为存储。所以你的StateContainer实际上是三个类：

```dart
class StateContainer extends StatefulWidget
class StateContainerState extends State<StateContainer>
class _InheritedStateContainer extends InheritedWidget
```


`InheritedWidget`和`StateContainer`是最简单的设置，一旦设置它们就不会改变。逻辑主要存在于StateContainerState中。设置前两个:

```dart
class _InheritedStateContainer extends InheritedWidget {
   // Data is your entire state. In our case just 'User' 
  final StateContainerState data;
   
  // You must pass through a child and your state.
  _InheritedStateContainer({
    Key key,
    @required this.data,
    @required Widget child,
  }) : super(key: key, child: child);

  // This is a built in method which you can use to check if
  // any state has changed. If not, no reason to rebuild all the widgets
  // that rely on your state.
  @override
  bool updateShouldNotify(_InheritedStateContainer old) => true;
}

class StateContainer extends StatefulWidget {
   // You must pass through a child. 
  final Widget child;
  final User user;

  StateContainer({
    @required this.child,
    this.user,
  });

  // This is the secret sauce. Write your own 'of' method that will behave
  // Exactly like MediaQuery.of and Theme.of
  // It basically says 'get the data from the widget of this type.
  static StateContainerState of(BuildContext context) {
    return (context.inheritFromWidgetOfExactType(_InheritedStateContainer)
            as _InheritedStateContainer).data;
  }
  
  @override
  StateContainerState createState() => new StateContainerState();
}
```

'of'方法应该永远不会做任何其他事情。实际上，这两个类可以永远保持独立。

### 2. StateContainerState widget

这个Widget是您所有状态和逻辑可以存在的地方。对于这个应用程序，你只需能够存储和操纵你的用户信息。

```dart
class StateContainerState extends State<StateContainer> {
  // Whichever properties you wanna pass around your app as state
  User user;

  // You can (and probably will) have methods on your StateContainer
  // These methods are then used through our your app to 
  // change state.
  // Using setState() here tells Flutter to repaint all the 
  // Widgets in the app that rely on the state you've changed.
  void updateUserInfo({firstName, lastName, email}) {
    if (user == null) {
      user = new User(firstName, lastName, email);
      setState(() {
        user = user;
      });
    } else {
      setState(() {
        user.firstName = firstName ?? user.firstName;
        user.lastName = lastName ?? user.lastName;
        user.email = email ?? user.email;
      });
    }
  }

  // Simple build method that just passes this state through
  // your InheritedWidget
  @override
  Widget build(BuildContext context) {
    return new _InheritedStateContainer(
      data: this,
      child: widget.child,
    );
  }
}
```
如果你以前使用过Redux，你可以看到这里涉及的样板少了多少，看起来是远远不够的，这当然会导致潜在的bug，但对于一个简单的应用程序，这是太棒了。这实际上是设置仓库所需的所有工作。然后，您只需根据需要向该类添加属性和方法。

#3.重构主页面和表单页面

首先，将你的应用包裹在StateContainer中：

```
void main() {
  runApp(new StateContainer(child: new UserApp()));
}

```

就是这样：现在您可以在整个应用程序中访问你的仓库。像这样做：

```dart 
class HomeScreenState extends State<HomeScreen> {
  // Make a class property for the data you want
  User user;

  // This Widget will display the users info:
  
  Widget get _userInfo {
    return new Center(
      child: new Column(
        mainAxisAlignment: MainAxisAlignment.center,
        crossAxisAlignment: CrossAxisAlignment.center,
        children: <Widget>[
          // This refers to the user in your store
          new Text("${user.firstName} ${user.lastName}",
              style: new TextStyle(fontSize: 24.0)),
          new Text(user.email, style: new TextStyle(fontSize: 24.0)),
        ],
      ),
    );
  }

  Widget get _logInPrompt {
    // ...
  }

  void _updateUser(BuildContext context) {
    // ...
  }

  @override
  Widget build(BuildContext context) {
    // This is how you access your store. This container
    // is where your properties and methods live
    final container = StateContainer.of(context);
    
    // set the class's user
    user = container.user;
    
    var body = user != null ? _userInfo : _logInPrompt;
    
    return new Scaffold(
      appBar: new AppBar(
        title: new Text('Inherited Widget Test'),
      ),
      // The body will rerender to show user info
      // as its updated
      body: body,
      floatingActionButton: new FloatingActionButton(
        onPressed: () => _updateUser(context),
        child: new Icon(Icons.edit),
      ),
    );
  }
}
```
这里很简单的变化。表单页面没有太大区别：

```dart
// form_page.dart
// ...
class UpdateUserScreen extends StatelessWidget {
  // ...

  @override
  Widget build(BuildContext context) {
    // get reference to your store
    final container = StateContainer.of(context);
    
    return new Scaffold(
      // the form is the same until here:
      floatingActionButton: new FloatingActionButton(
        child: new Icon(Icons.add),
        onPressed: () {
          final form = formKey.currentState;
          if (form.validate()) {
            var firstName = firstNameKey.currentState.value;
            var lastName = lastNameKey.currentState.value;
            var email = emailKey.currentState.value;

            // This is a hack that isn't important
            // To this lesson. Basically, it prevents 
            // The store from overriding user info
            // with an empty string if you only want
            // to change a single attribute
            if (firstName == '') {
              firstName = null;
            }
            if (lastName == '') {
              lastName = null;
            }
            if (email == '') {
              email = null;
            }

            // You can call the method from your store,
            // which will call set state and rerender
            // the widgets that rely on the user slice of state.
            // In this case, thats the home page
            container.updateUserInfo(
              firstName: firstName,
              lastName: lastName,
              email: email,
            );
            
            Navigator.pop(context);
          }
        },
      ),
    );
  }
}

```
仅此而已！InheritedWidget很简单，对于简单的应用程序，原型等来说，非常可行。

[完成时的Github Gist](https://gist.github.com/ericwindmill/f790bd2456e6489b1ab97eba246fd4c6)
