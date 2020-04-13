---
layout: post
title:  "(译)使用BloC模式来架构你的Flutter项目"
date:   2019-04-15 20:20:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - BLoC
---

[原文链接](https://medium.com/flutterpub/architecting-your-flutter-project-bd04e144a8f1)

大家好。我带着一篇新的关于Flutter的文章回来了。这一次，我将向你展示“如何架构你的Flutter项目”。这样您就可以轻松维护，扩展和测试Flutter项目。在深入探讨实际话题之前。我想分享一个小故事，说明为什么我们应该专注于为我们的项目构建一个坚实的架构。

更新：本文的第2部分已发布，当前设计中有一些更改，以解决一些问题并展示一些惊人的实现。链接在[这里](Architect-your-Flutter-project-using-BLOC-pattern-(Part-2).html)。

<!--more-->


## 为什么你需要对你的项目进行架构

多年前的2015年，我是还是一名求胜心切的新手程序员，当时我在学习Android应用程序开发。作为一个求胜心切的程序员，我只关心我编写的程序的输出和效率。我从未想过对我编写的程序或项目进行架构。这种趋势和风格也反映在我的android项目中。我是用求胜心切的程序员的心态编写Android应用程序。起初我一个写项目的时候，一切都很好，因为从来没有老板或经理可以给我添加新功能或更改应用程序中现有功能的需求。但是当我开始在创业公司工作并为他们构建Android应用程序时。我总是花很多时间来更改应用中的现有功能。不仅如此，我甚至在构建应用程序的过程中还附加了bug。所有这些问题的主要根源是“我从未遵循任何架构模式或从未架构过我的项目”。随着时间的推移，我开始了解软件世界，我将自己从求胜心切的程序员转变为软件工程师。今天，每当我开始一个新项目时，我的主要目标是建立一个坚固的项目结构或架构。这帮助我成为一个更好，更没有压力的软件工程师😄。

结束我无聊的故事😅。让我们开始研究本文的主要目标。 “使用BLOC模式构建您的Flutter项目”

## 我们的目标

我将构建一个非常简单的应用程序。它将有一个页面，您可以在这个页面中看到条目的网格列表。这些条目将从服务器获取。条目列表将是从The Movies DB网站获取的热门电影。

注意：在进一步之前，我假定您了解[Widgets](https://docs.flutter.io/flutter/widgets/Widget-class.html)，[如何在Flutter中进行网络调用](https://medium.com/flutterpub/making-a-network-call-in-flutter-f712f2137109)以及在Dart中具有中级了解。本文将有点冗长，并且链向了大量的其他资源，从而你能够更多地了解具体的话题。

开始我们的表演。 😍

在直接深入代码之前。让我给你看下架构图，我们会遵照这个图来架构这个应用。
![](https://cdn-images-1.medium.com/max/800/1*MqYPYKdNBiID0mZ-zyE-mA.png)

Bloc 模式

上图展示了数据是如何从UI流向数据层，以及如何从UI层流向数据层。 BLOC永远不会持有UI页面中的任何widget。UI页面只会观察来自BLOC类的变化。让我们通过问答的形式来理解这个图：

### 1.什么是BLOC模式？
它是Google开发人员推荐的Flutter状态管理系统。它有助于管理状态并从项目的中心位置访问数据。

### 2.我可以将此架构与其他任何架构联系起来吗？
当然可以。MVP和MVVM就是一个很好的例子。唯一变化的是：BLOC取代了MVVM中的ViewModel。

### 3. BLOC的底层是什么？或者在一处管理状态的核心是什么？
以STREAMS或REACTIVE的方法。一般而言，数据将以流的形式从BLOC流向UI或从UI流向BLOC。如果你从未听说过流。阅读此[Stack Overflow上的回答](https://stackoverflow.com/a/1216397/8327394)。

希望这个简短的问答环节消除了你的一些疑虑。

让我们开始用BLOC模式构建项目

1.首先创建一个新项目并清除main.dart文件中的所有代码。在终端中输入以下命令：

```
flutter create myProjectName
```

2.在你的main.dart的文件中写下以下代码：

```dart
import 'package:flutter/material.dart';
import 'src/app.dart';

void main(){
  runApp(App());
}
```

第二行会有错误。我们将在下一步中解决它。

3.在lib包下创建一个src包。在src包中创建一个文件并将其命名为app.dart。将以下代码复制到app.dart文件中。

```dart
import 'package:flutter/material.dart';
import 'ui/movie_list.dart';

class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return MaterialApp(
        theme: ThemeData.dark(),
        home: Scaffold(
          body: MovieList(),
        ),
      );
  }
}
```

4.在src包下创建一个新的包并将其命名为resources。

现在创建一些新的包，即blocs，models，resources和ui，如下图所示，然后我们就设置好项目的框架：

![](https://cdn-images-1.medium.com/max/800/1*ptxnhV5gsEyKAj6wliRMaw.png)

blocs包将保存我们的BLOC实现相关文件。 models包将保存POJO类或我们将从服务器获取的JSON响应体的模型类。 resources包将保存存储库类和网络调用实现类。 ui包将保存我们的屏幕，用户可以看到它们。

5.我们要补充的最后一件事就是RxDart第三方库。打开pubspec.yaml文件并添加rxdart：^ 0.18.0，如下所示：

```yaml

dependencies:
  flutter:
    sdk: flutter

  # The following adds the Cupertino Icons font to your application.
  # Use with the CupertinoIcons class for iOS style icons.
  cupertino_icons: ^0.1.2
  rxdart: ^0.18.0
  http: ^0.12.0+1
```
同步你的项目或在终端输入以下命令。确保在项目目录中执行此命令。

```
flutter packages get
```

6.现在我们完成了项目的骨架。是时候处理项目的最底层，即网络层。让我们熟悉下我们将要使用的API端点。点击[此链接](https://www.themoviedb.org/account/signup)，您将进入电影数据库API站点。注册并从“设置”页面获取API密钥。我们将点击以下网址获取回复：

ttp://api.themoviedb.org/3/movie/popular?api_key="your_api_key”

将您的API密钥放在上面的链接中并点击（删除双引号）。你可以看到这样的JSON响应体：

```json
{
  "page": 1,
  "total_results": 19772,
  "total_pages": 989,
  "results": [
    {
      "vote_count": 6503,
      "id": 299536,
      "video": false,
      "vote_average": 8.3,
      "title": "Avengers: Infinity War",
      "popularity": 350.154,
      "poster_path": "\/7WsyChQLEftFiDOVTGkv3hFpyyt.jpg",
      "original_language": "en",
      "original_title": "Avengers: Infinity War",
      "genre_ids": [
        12,
        878,
        14,
        28
      ],
      "backdrop_path": "\/bOGkgRGdhrBYJSLpXaxhXVstddV.jpg",
      "adult": false,
      "overview": "As the Avengers and their allies have continued to protect the world from threats too large for any one hero to handle, a new danger has emerged from the cosmic shadows: Thanos. A despot of intergalactic infamy, his goal is to collect all six Infinity Stones, artifacts of unimaginable power, and use them to inflict his twisted will on all of reality. Everything the Avengers have fought for has led up to this moment - the fate of Earth and existence itself has never been more uncertain.",
      "release_date": "2018-04-25"
    },
```


7.让我们为这种类型的响应构建一个模型或POJO类。在模型包中创建一个新文件，并将其命名为item_model.dart。将以下代码复制并粘贴到item_model.dart文件中：

```dart
class ItemModel {
  int _page;
  int _total_results;
  int _total_pages;
  List<_Result> _results = [];

  ItemModel.fromJson(Map<String, dynamic> parsedJson) {
    print(parsedJson['results'].length);
    _page = parsedJson['page'];
    _total_results = parsedJson['total_results'];
    _total_pages = parsedJson['total_pages'];
    List<_Result> temp = [];
    for (int i = 0; i < parsedJson['results'].length; i++) {
      _Result result = _Result(parsedJson['results'][i]);
      temp.add(result);
    }
    _results = temp;
  }

  List<_Result> get results => _results;

  int get total_pages => _total_pages;

  int get total_results => _total_results;

  int get page => _page;
}

class _Result {
  int _vote_count;
  int _id;
  bool _video;
  var _vote_average;
  String _title;
  double _popularity;
  String _poster_path;
  String _original_language;
  String _original_title;
  List<int> _genre_ids = [];
  String _backdrop_path;
  bool _adult;
  String _overview;
  String _release_date;

  _Result(result) {
    _vote_count = result['vote_count'];
    _id = result['id'];
    _video = result['video'];
    _vote_average = result['vote_average'];
    _title = result['title'];
    _popularity = result['popularity'];
    _poster_path = result['poster_path'];
    _original_language = result['original_language'];
    _original_title = result['original_title'];
    for (int i = 0; i < result['genre_ids'].length; i++) {
      _genre_ids.add(result['genre_ids'][i]);
    }
    _backdrop_path = result['backdrop_path'];
    _adult = result['adult'];
    _overview = result['overview'];
    _release_date = result['release_date'];
  }

  String get release_date => _release_date;

  String get overview => _overview;

  bool get adult => _adult;

  String get backdrop_path => _backdrop_path;

  List<int> get genre_ids => _genre_ids;

  String get original_title => _original_title;

  String get original_language => _original_language;

  String get poster_path => _poster_path;

  double get popularity => _popularity;

  String get title => _title;

  double get vote_average => _vote_average;

  bool get video => _video;

  int get id => _id;

  int get vote_count => _vote_count;
}
```

我希望你可以吧这个文件和JSON响应体一一对应。如果不能一一对应的话，我们最感兴趣的是Results类中的poster_path，这是你需要进一步知道的。我们将在主UI中显示所有热门电影的海报。 fromJson()方法仅仅是获取解码后的json并将值放入正确的变量。

8.是时候开始网络实现方面的工作了。在资源包中创建一个文件，并将其命名为movie_api_provider.dart。将以下代码复制并粘贴到文件中，我将向你解释：

```dart
import 'dart:async';
import 'package:http/http.dart' show Client;
import 'dart:convert';
import '../models/item_model.dart';

class MovieApiProvider {
  Client client = Client();
  final _apiKey = 'your_api_key';

  Future<ItemModel> fetchMovieList() async {
    print("entered");
    final response = await client
        .get("http://api.themoviedb.org/3/movie/popular?api_key=$_apiKey");
    print(response.body.toString());
    if (response.statusCode == 200) {
      // If the call to the server was successful, parse the JSON
      return ItemModel.fromJson(json.decode(response.body));
    } else {
      // If that call was not successful, throw an error.
      throw Exception('Failed to load post');
    }
  }
}
```
注意：请将您的API密钥放在movie_api_provider.dart文件中的变量_apiKey中，否则它是不能工作的。

fetchMovieList()方法对API作出了网络请求。网络请求完成后，如果网络请求成功，它将返回Future ItemModel对象，否则将抛出异常。

9.接下来，我们将在资源包中创建一个新文件，并将其命名为repository.dart。将以下代码复制并粘贴到文件中：

```dart
import 'dart:async';
import 'movie_api_provider.dart';
import '../models/item_model.dart';

class Repository {
  final moviesApiProvider = MovieApiProvider();

  Future<ItemModel> fetchAllMovies() => moviesApiProvider.fetchMovieList();
}
```

我们正在导入movie_api_provider.dart文件并调用其fetchMovieList()方法。此Repository类是数据流向BLOC的中心点。

10.现在有点复杂。实现bloc逻辑。让我们在blocs包中创建一个新文件，并将其命名为movies_bloc.dart。复制下面的代码粘贴，我将向您详细解释代码：

```dart
import '../resources/repository.dart';
import 'package:rxdart/rxdart.dart';
import '../models/item_model.dart';

class MoviesBloc {
  final _repository = Repository();
  final _moviesFetcher = PublishSubject<ItemModel>();

  Observable<ItemModel> get allMovies => _moviesFetcher.stream;

  fetchAllMovies() async {
    ItemModel itemModel = await _repository.fetchAllMovies();
    _moviesFetcher.sink.add(itemModel);
  }

  dispose() {
    _moviesFetcher.close();
  }
}

final bloc = MoviesBloc();
```

我们正在导入一个包导入'包：rxdart/rxdart.dart';最终将导入此文件中所有与RxDart相关的方法和类。在MoviesBloc类中，我们创建了Repository类对象，该对象将用于访问fetchAllMovies()。我们创建一个PublishSubject对象，该对象的职责是以ItemModel对象的形式添加从服务器获取的数据，并将其作为流传递给UI页面。要将ItemModel对象作为流传递，我们创建了另一个方法allMovies()，其返回类型为[Observable](https://youtu.be/XbOuCBuQepI)（如果您不理解Observable，请观看此视频）。如果你看到最后一行,我们创建了bloc对象。这样我们就可以将一个MoviesBloc类的实例访问能力给到到UI页面。

如果你不知道什么是响应式编程。请看下这个简单的解释。简而言之，只要有来自服务器的新数据，我们就必须更新UI屏幕。为了使这个更新任务变得简单，我们告诉UI页面继续观察来自MoviesBloc类的任何变更并相应地更新你的内容。这种“观察”新数据的工作可以使用RxDart完成。

11.现在是最后一节。在ui包中创建一个新文件，并将其命名为movie_list.dart。复制粘贴以下代码：

```dart
import 'package:flutter/material.dart';
import '../models/item_model.dart';
import '../blocs/movies_bloc.dart';

class MovieList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    bloc.fetchAllMovies();
    return Scaffold(
      appBar: AppBar(
        title: Text('Popular Movies'),
      ),
      body: StreamBuilder(
        stream: bloc.allMovies,
        builder: (context, AsyncSnapshot<ItemModel> snapshot) {
          if (snapshot.hasData) {
            return buildList(snapshot);
          } else if (snapshot.hasError) {
            return Text(snapshot.error.toString());
          }
          return Center(child: CircularProgressIndicator());
        },
      ),
    );
  }

  Widget buildList(AsyncSnapshot<ItemModel> snapshot) {
    return GridView.builder(
        itemCount: snapshot.data.results.length,
        gridDelegate:
            new SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 2),
        itemBuilder: (BuildContext context, int index) {
          return Image.network(
            'https://image.tmdb.org/t/p/w185${snapshot.data
                .results[index].poster_path}',
            fit: BoxFit.cover,
          );
        });
  }
}
```

这个类的最好以及有趣的部分是，我没有使用StatefulWidget。但我使用的是StreamBuilder，它将执行StatefulWidget所做的更新UI的工作。

有一点需要指出，我在构建方法中进行网络请求，这是不应该像build(context)方法那样多次调用。文章更新中会有更好的方法。但是现在，随着文章变得越来越复杂，我保持简单，即在build(Context)方法中进行网络请求。

正如我告诉你的，我们的MoviesBloc类将新数据作为流传递。因此，为了处理流，我们有一个很好的内置类，即StreamBuilder，它将监听传入的流并相应地更新UI。 StreamBuilder期待一个流参数，我们在它返回流时传递MovieBloc的allMovies()方法。因此，当有数据流出现时，StreamBuilder将使用最新数据重新渲染widget。这里快照数据持有着ItemModel对象。现在你可以使用任意Widget来呈现对象中的内容了（这里就靠你的想象力了）。我使用GridView呈现了ItemModel列表中的所有海报。
如果你想要完整的代码。这是项目的[github地址](https://github.com/SAGARSURI/MyMovies)。