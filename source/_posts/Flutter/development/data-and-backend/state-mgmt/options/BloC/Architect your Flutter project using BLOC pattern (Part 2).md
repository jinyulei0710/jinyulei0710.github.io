---
layout: post
title:  "(译)使用BloC模式来架构你的Flutter项目(第二部分)"
date:   2019-04-15 22:26:17 +0800
categories: 
   - Flutter 开发
tags:
    - 状态管理
    - BLoC
---

[原文链接](https://medium.com/flutterpub/architect-your-flutter-project-using-bloc-pattern-part-2-d8dd1eca9ba5)

嗨伙计！本文是我之前的文章“构建您的Flutter项目”的延续。正如我在上一篇文章中所承诺的那样，我将解决当前架构设计中的一些缺陷，并为我们正在构建的应用程序添加一些新功能。在开始旅程之前，让我向你展示四个我们要涉及的地方并对它们进行学习。以下是我们将在本文中介绍的话题。

## 我们将讨论的话题：
1. 解决当前架构设计中的缺陷
2. 单实例与范围有限的实例（BLoC访问）
3. 导航
4. RxDart的变形
注意：在看下去之前。我强烈建议您阅读我之前的文章，以便更好地了解我们正在构建的应用程序以及我们正在遵循的架构设计（BLoC模式）。

<!--more-->


## 当前架构设计中的缺陷
在解决问题或讨论变化之前。如果你们读过上篇文章的话，我想你们都一定在会有一个问题，让我来回答这个问题。

为什么你没有更新上一篇文章而不是写一篇新的文章？

是!当然我当然可以这样做的。我可以直接向您展示一个可以让您满意的可以工作和无错误的代码。但在这里，我想告诉你在构建这个应用程序时所经历的一切。我遇到的问题或失败。通过这种方式，您可以构建任何Flutter应用程序，并在完成整个过程时调试或解决您遇到的任何问题。不要废话，让我们回到我们的目标。

如果你已经看过我之前的文章和代码，第一个缺陷是我在MoviesBloc类中创建了一个名为dispose()的方法。此方法负责关闭所有打开的流以避免内存泄漏。我创建了该方法，但从未在movie_list.dart文件中的任何位置调用它，这会导致内存泄漏。另一个主要缺陷是我在构建方法中进行网络请求，这是非常危险的。让我们试着解决这两个主要缺陷。

目前，MovieList类是一个StatelessWidget，而StatelessWidget的工作方式是，添加到Widget树上的时候build()方法被调用，它的所有属性都是不可变的。build方法是入口，由于配置更改会被多次调用。所以它不是进行任何网络请求的好地方（我在之前的文章中做过）。我们甚至在StatelessWidget中没有一个可以在其中调用bloc的dispose方法。我们必须找到一个可以进行网络请求的地方，最后调用dispose方法。

要点是StatelessWidget像StatefulWidget那样提供initState和dispose方法。首先调用StatefulWidget中的initState方法来分配资源，并在处理这些分配的资源时调用dispose方法(更多详细内容请看[这里](https://docs.flutter.io/flutter/widgets/StatefulWidget-class.html)）。因此，让我们将MovieList类从StatelessWidget转换为StatefulWidget，并在StatefulWidget内的initState()作出网络请求，在StatefulWidget内的dispose()调用MovieBloc的dispose()。

只需用下面的实现替换movie_list.dart代码即可。

```dart
import 'package:flutter/material.dart';
import '../models/item_model.dart';
import '../blocs/movies_bloc.dart';

class MovieList extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return MovieListState();
  }
}

class MovieListState extends State<MovieList> {
  @override
  void initState() {
    super.initState();
    bloc.fetchAllMovies();
  }

  @override
  void dispose() {
    bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
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
          return GridTile(
            child: Image.network(
              'https://image.tmdb.org/t/p/w185${snapshot.data
                  .results[index].poster_path}',
              fit: BoxFit.cover,
            ),
          );
        });
  }
}
```
在以上的代码中，我在initState()中调用了bloc.fetchAllMovies()，在MovieListState类的dispose()中调用了bloc.dispose()。运行该应用程序，你可以像往常一样看到加载电影列表的应用程序。你不会看到任何视觉上的变化，但在内部你已经确定，不会有任何多次的网络请求，也不会有任何内存泄漏。哇！这看起来很整洁。 😍

主题演讲：永远不要在构建方法中进行任何网络或数据库调用，并始终确保关闭处于开启状态的流。

## 新的功能实现

现在是我们为现有应用添加新功能的时候了。在我们开始讨论或实现新功能之前。让我向你展示一下。这是一个小视频。

如你所见，我们添加了一个的页面，你可以在其中查看从列表中选择的特定电影的详细信息。

## 让我们规划下应用流程

在向应用添加任何新功能之前，最好先做一些案头工作(决定流程)。所以在这里我分享了我在分析应用程序的所有功能后想出的应用程序流程。

![](https://cdn-images-1.medium.com/max/800/1*dkKlfFewf6CBVbwFIeCNHQ.png)

我想大多数人在看完图表后都会很容易理解这个流程，因为我使用的新术语很少。但是我仍然要解释下上面的图表。

1. 电影列表页面：这是你可以看到所有电影的网格列表的页面。
2. 电影列表BloC：这是根据需要从存储库获取数据并将其传递到电影列表页面的桥梁(随后我将对单例机进行解释)。
3. 电影详细信息页面：在此页面中，您将看到从列表屏幕中选择的电影的详细信息。在这里你可以看到电影的名称，评级，发布日期，描述和预告片（我随后将解释范围有限的实例）。
4. 仓库：这是控制数据流的中心点。
5. API提供者：持有了网络请求的实现。

现在你一定在想。什么是图中的“单例和范围有限的实例”。让我们详细了解它们。

## 单例与范围有限的实例

如图所示，两个页面都可以访问各自的BLoC类。您可以通过两种方式将这些BLoC类暴露给各自的页面，即单例或范围有限的实例。当我说Single Instance时，我的意思是BLoC类的单个引用（Singleton）将暴露给页面。可以从应用程序的任何部分访问此类型的BLoC类。任何屏幕都可以使用单实例BLoC类。

但是Scoped Instance BLoC类具有有限的访问权限。我的意思是它只对与它相关联页面或这暴露给的页面是可访问的。这是有一个对其进行解释的小图。

![](https://cdn-images-1.medium.com/max/800/1*rID6hLMzUfIwSA6m8UkM2g.png)

正如你在上图中所看到的，只有widget和页面下方的其他2个自定义widget能访问bloc。我们使用的是InheritedWidget，它将BLoC保存在其中。InheritedWidget将 Screen widget进行包装，让Screen widget及其下方的widget可以访问BLoC。Screen Widget的父widget都无法访问BLoC。

希望您了解单例和范围有限的实例之间的区别。当您使用小型应用程序时，单例访问BLoC的方式非常有用。但是如果你正在开发一个大型项目，那么范围有限的实例是首选方式。

## 添加详情页

是时候将详情页添加到我们的应用程序中。详情页背后的逻辑是，用户将点击电影列表中的电影条目。用户将进入详情页，用户可以在其中查看电影的详细信息。一些细节（电影名称，评级，发布日期，描述，海报）将从列表页面传递到详情页。预告片将从服务器加载。让我们把购物车部分放在一边，先专注于显示从列表页面传递而来的数据。

在创建文件之前，我希望你使用与上一篇文章中提到相同的项目结构。如果是一样的结构，则在ui包内创建一个名为movie_detail.dart的文件。复制将以下代码粘贴到文件中。

```dart
import 'package:flutter/material.dart';

class MovieDetail extends StatefulWidget {
  final posterUrl;
  final description;
  final releaseDate;
  final String title;
  final String voteAverage;
  final int movieId;

  MovieDetail({
    this.title,
    this.posterUrl,
    this.description,
    this.releaseDate,
    this.voteAverage,
    this.movieId,
  });

  @override
  State<StatefulWidget> createState() {
    return MovieDetailState(
      title: title,
      posterUrl: posterUrl,
      description: description,
      releaseDate: releaseDate,
      voteAverage: voteAverage,
      movieId: movieId,
    );
  }
}

class MovieDetailState extends State<MovieDetail> {
  final posterUrl;
  final description;
  final releaseDate;
  final String title;
  final String voteAverage;
  final int movieId;

  MovieDetailState({
    this.title,
    this.posterUrl,
    this.description,
    this.releaseDate,
    this.voteAverage,
    this.movieId,
  });

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        top: false,
        bottom: false,
        child: NestedScrollView(
          headerSliverBuilder: (BuildContext context, bool innerBoxIsScrolled) {
            return <Widget>[
              SliverAppBar(
                expandedHeight: 200.0,
                floating: false,
                pinned: true,
                elevation: 0.0,
                flexibleSpace: FlexibleSpaceBar(
                    background: Image.network(
                  "https://image.tmdb.org/t/p/w500$posterUrl",
                  fit: BoxFit.cover,
                )),
              ),
            ];
          },
          body: Padding(
            padding: const EdgeInsets.all(10.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: <Widget>[
                Container(margin: EdgeInsets.only(top: 5.0)),
                Text(
                  title,
                  style: TextStyle(
                    fontSize: 25.0,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Container(margin: EdgeInsets.only(top: 8.0, bottom: 8.0)),
                Row(
                  children: <Widget>[
                    Icon(
                      Icons.favorite,
                      color: Colors.red,
                    ),
                    Container(
                      margin: EdgeInsets.only(left: 1.0, right: 1.0),
                    ),
                    Text(
                      voteAverage,
                      style: TextStyle(
                        fontSize: 18.0,
                      ),
                    ),
                    Container(
                      margin: EdgeInsets.only(left: 10.0, right: 10.0),
                    ),
                    Text(
                      releaseDate,
                      style: TextStyle(
                        fontSize: 18.0,
                      ),
                    ),
                  ],
                ),
                Container(margin: EdgeInsets.only(top: 8.0, bottom: 8.0)),
                Text(description),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

正如您所看到的，此类的构造函数需要很少的参数。这些数据将从列表屏幕提供给此类。下一步是实现导航逻辑，将我们从列表页面跳转到详情页。

## 导航

在Flutter中如果要从一个页面跳转到另一个页面，我们使用的是Navigator类。让我们在movie_list.dart文件中实现导航逻辑。

因此，想法是，在点击每个网格项目的时候，我们将打开详情页并显示我们从列表页面传递到详情页的内容。这是movie_list.dart文件的代码。

```dart
import 'package:flutter/material.dart';
import '../models/item_model.dart';
import '../blocs/movies_bloc.dart';
import 'movie_detail.dart';

class MovieList extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return MovieListState();
  }
}

class MovieListState extends State<MovieList> {
  @override
  void initState() {
    super.initState();
    bloc.fetchAllMovies();
  }

  @override
  void dispose() {
    bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
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
          return GridTile(
            child: InkResponse(
              enableFeedback: true,
              child: Image.network(
                'https://image.tmdb.org/t/p/w185${snapshot.data
                    .results[index].poster_path}',
                fit: BoxFit.cover,
              ),
              onTap: () => openDetailPage(snapshot.data, index),
            ),
          );
        });
  }

  openDetailPage(ItemModel data, int index) {
    Navigator.push(
      context,
      MaterialPageRoute(builder: (context) {
        return MovieDetail(
          title: data.results[index].title,
          posterUrl: data.results[index].backdrop_path,
          description: data.results[index].overview,
          releaseDate: data.results[index].release_date,
          voteAverage: data.results[index].vote_average.toString(),
          movieId: data.results[index].id,
        );
      }),
    );
  }
}
```

在以上代码中，您可以看到openDetailPage()方法具有导航逻辑。我们将传递将在详情页中显示的数据。运行该应用程序，您可以跳转到新页面。

哇！我可以跳转了😍
现在是时候在详情页面显示预告片了。让我们熟悉下从服务器获取预告片所需的API。点击以下链接就能获得JSON响应体。

https://api.themoviedb.org/3/movie/<movie_id>/videos?api_key=your_api_key

在上面的API中，我们必须输入两个东西。第一个是movie_id，第二个是api密钥。这是点击链接后返回的响应体。

```json
{
  "id": 299536,
  "results": [
    {
      "id": "5a200baa925141033608f5f0",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "6ZfuNTqbHE8",
      "name": "Official Trailer",
      "site": "YouTube",
      "size": 1080,
      "type": "Trailer"
    },
    {
      "id": "5a200bcc925141032408d21b",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "sAOzrChqmd0",
      "name": "Action...Avengers: Infinity War",
      "site": "YouTube",
      "size": 720,
      "type": "Clip"
    },
    {
      "id": "5a200bdd0e0a264cca08d39f",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "3VbHg5fqBYw",
      "name": "Trailer Tease",
      "site": "YouTube",
      "size": 720,
      "type": "Teaser"
    },
    {
      "id": "5a7833440e0a26597f010849",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "pVxOVlm_lE8",
      "name": "Big Game Spot",
      "site": "YouTube",
      "size": 1080,
      "type": "Teaser"
    },
    {
      "id": "5aabd7e69251413feb011276",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "QwievZ1Tx-8",
      "name": "Official Trailer #2",
      "site": "YouTube",
      "size": 1080,
      "type": "Trailer"
    },
    {
      "id": "5aea2ed2c3a3682bf7001205",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "LXPaDL_oILs",
      "name": "\"Legacy\" TV Spot",
      "site": "YouTube",
      "size": 1080,
      "type": "Teaser"
    },
    {
      "id": "5aea2f3e92514172a7001672",
      "iso_639_1": "en",
      "iso_3166_1": "US",
      "key": "PbRmbhdHDDM",
      "name": "\"Family\" Featurette",
      "site": "YouTube",
      "size": 1080,
      "type": "Featurette"
    }
  ]
}
```
对于上述响应体，我们需要有一个POJO类。让我们先构建它。在models包中创建名为trailer_model.dart的文件。复制粘贴下面的代码。

```dart
class TrailerModel {
  int _id;
  List<_Result> _results = [];

  TrailerModel.fromJson(Map<String, dynamic> parsedJson) {
    _id = parsedJson['id'];
    List<_Result> temp = [];
    for (int i = 0; i < parsedJson['results'].length; i++) {
      _Result result = _Result(parsedJson['results'][i]);
      temp.add(result);
    }
    _results = temp;
  }

  List<_Result> get results => _results;

  int get id => _id;
}

class _Result {
  String _id;
  String _iso_639_1;
  String _iso_3166_1;
  String _key;
  String _name;
  String _site;
  int _size;
  String _type;

  _Result(result) {
    _id = result['id'];
    _iso_639_1 = result['iso_639_1'];
    _iso_3166_1 = result['iso_3166_1'];
    _key = result['key'];
    _name = result['name'];
    _site = result['site'];
    _size = result['size'];
    _type = result['type'];
  }

  String get id => _id;

  String get iso_639_1 => _iso_639_1;

  String get iso_3166_1 => _iso_3166_1;

  String get key => _key;

  String get name => _name;

  String get site => _site;

  int get size => _size;

  String get type => _type;
}

```

现在让我们在movie_api_provider.dart实现网络请求。复制并粘贴一下内容到文件中。

```dart
import 'dart:async';
import 'package:http/http.dart' show Client;
import 'dart:convert';
import '../models/item_model.dart';
import '../models/trailer_model.dart';

class MovieApiProvider {
  Client client = Client();
  final _apiKey = '802b2c4b88ea1183e50e6b285a27696e';
  final _baseUrl = "http://api.themoviedb.org/3/movie";

  Future<ItemModel> fetchMovieList() async {
    final response = await client.get("$_baseUrl/popular?api_key=$_apiKey");
    if (response.statusCode == 200) {
      // If the call to the server was successful, parse the JSON
      return ItemModel.fromJson(json.decode(response.body));
    } else {
      // If that call was not successful, throw an error.
      throw Exception('Failed to load post');
    }
  }

  Future<TrailerModel> fetchTrailer(int movieId) async {
    final response =
        await client.get("$_baseUrl/$movieId/videos?api_key=$_apiKey");

    if (response.statusCode == 200) {
      return TrailerModel.fromJson(json.decode(response.body));
    } else {
      throw Exception('Failed to load trailers');
    }
  }
}

```

fetchTrailer（movie_id）是我们请求API并将JSON响应转换为TrailerModel对象并返回Future<TrailerModel>的方法。

现在让我们通过添加这个新的网络请求实现来更新repository.dart文件。复制并粘贴repository.dart文件中的代码。

```dart
import 'dart:async';
import 'movie_api_provider.dart';
import '../models/item_model.dart';
import '../models/trailer_model.dart';

class Repository {
  final moviesApiProvider = MovieApiProvider();

  Future<ItemModel> fetchAllMovies() => moviesApiProvider.fetchMovieList();

  Future<TrailerModel> fetchTrailers(int movieId) => moviesApiProvider.fetchTrailer(movieId);
}
```

现在是实现Scoped Instance BLoC方法的时候了。在blocs包中创建一个新文件movie_detail_bloc.dart。在同一个blocs包中再创建一个文件movie_detail_bloc_provider.dart。

这是movie_detail_bloc_provider.dart文件的代码。

```dart
import 'package:flutter/material.dart';
import 'movie_detail_bloc.dart';
export 'movie_detail_bloc.dart';

class MovieDetailBlocProvider extends InheritedWidget {
  final MovieDetailBloc bloc;

  MovieDetailBlocProvider({Key key, Widget child})
      : bloc = MovieDetailBloc(),
        super(key: key, child: child);

  @override
  bool updateShouldNotify(_) {
    return true;
  }

  static MovieDetailBloc of(BuildContext context) {
    return (context.inheritFromWidgetOfExactType(MovieDetailBlocProvider)
            as MovieDetailBlocProvider)
        .bloc;
  }
}
```

此类扩展了InheritedWidget，并通过of(context)方法提供对bloc的访问。正如你所见，of(context）期望将context作为参数。此context属于InheritedWidget包装了的页面。在我们的例子中，它是电影详情页。

让我们编写movie_detail_bloc.dart的代码。将以下代码复制粘贴到bloc文件中。

```dart
import 'dart:async';

import 'package:rxdart/rxdart.dart';
import '../models/trailer_model.dart';
import '../resources/repository.dart';

class MovieDetailBloc {
  final _repository = Repository();
  final _movieId = PublishSubject<int>();
  final _trailers = BehaviorSubject<Future<TrailerModel>>();

  Function(int) get fetchTrailersById => _movieId.sink.add;
  Observable<Future<TrailerModel>> get movieTrailers => _trailers.stream;

  MovieDetailBloc() {
    _movieId.stream.transform(_itemTransformer()).pipe(_trailers);
  }

  dispose() async {
    _movieId.close();
    await _trailers.drain();
    _trailers.close();
  }

  _itemTransformer() {
    return ScanStreamTransformer(
      (Future<TrailerModel> trailer, int id, int index) {
        print(index);
        trailer = _repository.fetchTrailers(id);
        return trailer;
      },
    );
  }
}

```

让我解释一下上面的代码。从服务器获取预告片列表背后的思想是，我们必须将movieId传递给预告片API，它将返回给我们预告片列表。为了实现这个，我们将使用RxDart的一个重要特性，即[变形](https://pub.dartlang.org/documentation/rxdart/latest/rx_transformers/rx_transformers-library.html)。

## 变形

变形的主要作用是帮助链接两个或更多个[Subject](https://pub.dartlang.org/documentation/rxdart/latest/rx_subjects/rx_subjects-library.html)并获得最终的结果。思想是，如果你想在对数据执行某些操作后将数据从一个[Subject](https://pub.dartlang.org/documentation/rxdart/latest/rx_subjects/rx_subjects-library.html)传递到另一个[Subject](https://pub.dartlang.org/documentation/rxdart/latest/rx_subjects/rx_subjects-library.html)。我们将使用变换器对来自第一个[Subject](https://pub.dartlang.org/documentation/rxdart/latest/rx_subjects/rx_subjects-library.html)的输入数据执行操作，并将其传递给下一个[Subject](https://pub.dartlang.org/documentation/rxdart/latest/rx_subjects/rx_subjects-library.html)。

在我们的应用程序中，我们将movieId添加到_movieId，这是一个[PublishSubject](https://pub.dartlang.org/documentation/rxdart/latest/rx/PublishSubject-class.html)。我们将movieId传递给ScanStreamTransformer，然后ScanStreamTransformer将请求预告片API并获取结果并将其传递给_trailers，这是一个[BehaviorSubject](https://pub.dartlang.org/documentation/rxdart/latest/rx/BehaviorSubject-class.html)。这是一张阐释我的说明的小图。

![](https://cdn-images-1.medium.com/max/800/1*8EKpbLKC61gkS8jstJ_9kQ.png)


最后一步是，使MovieDetail页面可以访问movieDetailBloc。为此，我们需要更新openDetailPage()方法。这是movie_list.dart文件的更新代码。

```dart
import 'package:flutter/material.dart';
import '../models/item_model.dart';
import '../blocs/movies_bloc.dart';
import 'movie_detail.dart';
import '../blocs/movie_detail_bloc_provider.dart';

class MovieList extends StatefulWidget {
  @override
  State<StatefulWidget> createState() {
    return MovieListState();
  }
}

class MovieListState extends State<MovieList> {
  @override
  void initState() {
    super.initState();
    bloc.fetchAllMovies();
  }

  @override
  void dispose() {
    bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
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
          return GridTile(
            child: InkResponse(
              enableFeedback: true,
              child: Image.network(
                'https://image.tmdb.org/t/p/w185${snapshot.data
                    .results[index].poster_path}',
                fit: BoxFit.cover,
              ),
              onTap: () => openDetailPage(snapshot.data, index),
            ),
          );
        });
  }

  openDetailPage(ItemModel data, int index) {
    Navigator.push(
      context,
      MaterialPageRoute(builder: (context) {
        return MovieDetailBlocProvider(
          child: MovieDetail(
            title: data.results[index].title,
            posterUrl: data.results[index].backdrop_path,
            description: data.results[index].overview,
            releaseDate: data.results[index].release_date,
            voteAverage: data.results[index].vote_average.toString(),
            movieId: data.results[index].id,
          ),
        );
      }),
    );
  }
}
```

正如您在MaterialPageRoute中看到的那样，我们将返回MovieDetailBlocProvider（InheritedWidget）并将MovieDetail页面包装到其中。这样MovieDetailBloc类就可以在详情页面以及它下面的所有widget中访问。

最后，这是movie_detail.dart文件的代码。

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import '../blocs/movie_detail_bloc_provider.dart';
import '../models/trailer_model.dart';

class MovieDetail extends StatefulWidget {
  final posterUrl;
  final description;
  final releaseDate;
  final String title;
  final String voteAverage;
  final int movieId;

  MovieDetail({
    this.title,
    this.posterUrl,
    this.description,
    this.releaseDate,
    this.voteAverage,
    this.movieId,
  });

  @override
  State<StatefulWidget> createState() {
    return MovieDetailState(
      title: title,
      posterUrl: posterUrl,
      description: description,
      releaseDate: releaseDate,
      voteAverage: voteAverage,
      movieId: movieId,
    );
  }
}

class MovieDetailState extends State<MovieDetail> {
  final posterUrl;
  final description;
  final releaseDate;
  final String title;
  final String voteAverage;
  final int movieId;

  MovieDetailBloc bloc;

  MovieDetailState({
    this.title,
    this.posterUrl,
    this.description,
    this.releaseDate,
    this.voteAverage,
    this.movieId,
  });

  @override
  void didChangeDependencies() {
    bloc = MovieDetailBlocProvider.of(context);
    bloc.fetchTrailersById(movieId);
    super.didChangeDependencies();
  }

  @override
  void dispose() {
    bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: SafeArea(
        top: false,
        bottom: false,
        child: NestedScrollView(
          headerSliverBuilder: (BuildContext context,
              bool innerBoxIsScrolled) {
            return <Widget>[
              SliverAppBar(
                expandedHeight: 200.0,
                floating: false,
                pinned: true,
                elevation: 0.0,
                flexibleSpace: FlexibleSpaceBar(
                    background: Image.network(
                      "https://image.tmdb.org/t/p/w500$posterUrl",
                      fit: BoxFit.cover,
                    )),
              ),
            ];
          },
          body: Padding(
            padding: const EdgeInsets.all(10.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: <Widget>[
                Container(margin: EdgeInsets.only(top: 5.0)),
                Text(
                  title,
                  style: TextStyle(
                    fontSize: 25.0,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Container(margin: EdgeInsets.only(top: 8.0,
                    bottom: 8.0)),
                Row(
                  children: <Widget>[
                    Icon(
                      Icons.favorite,
                      color: Colors.red,
                    ),
                    Container(
                      margin: EdgeInsets.only(left: 1.0,
                          right: 1.0),
                    ),
                    Text(
                      voteAverage,
                      style: TextStyle(
                        fontSize: 18.0,
                      ),
                    ),
                    Container(
                      margin: EdgeInsets.only(left: 10.0,
                          right: 10.0),
                    ),
                    Text(
                      releaseDate,
                      style: TextStyle(
                        fontSize: 18.0,
                      ),
                    ),
                  ],
                ),
                Container(margin: EdgeInsets.only(top: 8.0,
                    bottom: 8.0)),
                Text(description),
                Container(margin: EdgeInsets.only(top: 8.0,
                    bottom: 8.0)),
                Text(
                  "Trailer",
                  style: TextStyle(
                    fontSize: 25.0,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                Container(margin: EdgeInsets.only(top: 8.0,
                    bottom: 8.0)),
                StreamBuilder(
                  stream: bloc.movieTrailers,
                  builder:
                      (context, AsyncSnapshot<Future<TrailerModel>> snapshot) {
                    if (snapshot.hasData) {
                      return FutureBuilder(
                        future: snapshot.data,
                        builder: (context,
                            AsyncSnapshot<TrailerModel> itemSnapShot) {
                          if (itemSnapShot.hasData) {
                            if (itemSnapShot.data.results.length > 0)
                              return trailerLayout(itemSnapShot.data);
                            else
                              return noTrailer(itemSnapShot.data);
                          } else {
                            return Center(child: CircularProgressIndicator());
                          }
                        },
                      );
                    } else {
                      return Center(child: CircularProgressIndicator());
                    }
                  },
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }

  Widget noTrailer(TrailerModel data) {
    return Center(
      child: Container(
        child: Text("No trailer available"),
      ),
    );
  }

  Widget trailerLayout(TrailerModel data) {
    if (data.results.length > 1) {
      return Row(
        children: <Widget>[
          trailerItem(data, 0),
          trailerItem(data, 1),
        ],
      );
    } else {
      return Row(
        children: <Widget>[
          trailerItem(data, 0),
        ],
      );
    }
  }

  trailerItem(TrailerModel data, int index) {
    return Expanded(
      child: Column(
        children: <Widget>[
          Container(
            margin: EdgeInsets.all(5.0),
            height: 100.0,
            color: Colors.grey,
            child: Center(child: Icon(Icons.play_circle_filled)),
          ),
          Text(
            data.results[index].name,
            maxLines: 1,
            overflow: TextOverflow.ellipsis,
          ),
        ],
      ),
    );
  }
}
```

这里有很多事情需要注意。我们之所以在didChangeDependencies()中初始化MovieDetailBloc的原因是[这个](https://docs.flutter.io/flutter/widgets/SingleTickerProviderStateMixin/didChangeDependencies.html)。你还可以看到StreamBuilder的快照数据包含Future<TrailerModel>，它只能由FutureBuilder使用。

我不会深入解释这些内容，因为这篇文章已经变得越来越大，我已经介绍了很多新东西。如果您需要任何帮助，请随时通过LinkedIn，Twitter与我联系或在下面发表您的评论。

我希望你喜欢这些内容并学到很多新东西。我只想告诉你，起初很难理解它。相信我，如果你进一步阅读我在这里触及的所有话题，你会发现它非常简单。请大声👏

如果你想要完整的代码。这是项目的[github地址](https://github.com/SAGARSURI/MyMovies)。