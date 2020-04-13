---
layout: post
title:  "Async in Dart"
date:   2019-04-27 14:16:17 +0800
categories: 
   - Flutter 开发
tags:
    - Async
    - Dart
    - Dart Developer Summit
---
https://www.youtube.com/watch?v=MUDOIAssBDs&list=PLOU2XLYxmsIIQorIS8gagUiMau9S84vZV&index=2

## Outline

* Introduction
	* synchronous vs asynchronus
	* dart:async
* Async/Await
* Debugging
* Summary

<!--more-->


## Introduction

Synchronous:

* Wait

Asynchronous

* Yield(don't wait)


## Parallel Synchronous

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/web_Parallel_Synchronous.png)

## Problems - Synchronous

* Race conditions
* Threads can be expensive
* Inefficient usd of threads

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/dart_system_call.png)

## Async Programming

"Don't wait for us. We will call you back."

## Shared Threads

Multiple tasks share the same thread.

parallel Synchronous

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/parallelSynchronous.png)

Asynchronus

![](https://qiantun.oss-cn-beijing.aliyuncs.com/img/dart_Asynchronous.png)

## Yield to Event Loop

Typical asynchronous pattern:

void main(){
  doSomethingSlow(callbackWhenDone);
}

## Async Programming-2

* Better use of threads
* Cooperative
* Callbacks
* Event Loop
* More reative

Fits well with event-driven programs

## Asynchronous in Dart

Asynchronous from the start.

* Support in dart:async
* Consistently used in dart:io,dart:convert,dart:html and dart:isolate


## Fundamental Types

One return value.

A collection of values.

||Synchronous|Asynchronous|
|---|----|---|
|Single|T|Future<T>|
|Multiple|Iterable<T>|Stream<T>|

## Future

A Future represents a value that is not yet available.

abstract class Future {
   Future then(onValue(value),{onError});
   Future catchError(onError(error));
   Future whenComplete(onCompletion());
}

## Future-Example

Example:

```dart
import 'dart:io';
main(){
   new File('/tep/file.text')
   .create()//Return a future.
   .then((_){ print('Created');})
   .catchError((e){print('couldn't create.$e');});
}
```

## Stream

Asynchronous Iterable

* sequence of values
* operations to manipulate and filter values
* Iterables are pulled. Streams push

## Pull

Iterables are pulled:

var iterator=[1,2,3].iterator;
while(iterator.moveNext()){
   print(iterator.current);
} 

## Push

Streams push:

var stream= element.onClick;
stream.listen(print);

## Iterable-API

```dart
abstract class Iterable<E>{
  Iterator<E> get iterator;
  
  Iterable map(Function f);
  Iterable<E> where(Function f);
  E           get first;
  void  forEach(void f(E element));
  ...
}
```

## Stream- Api

```dart
abstract class Iterable<E>{
  listen(void onData(E data),...);
  
  Stream map(Function f);
  Stream<E> where(Function f);
  Future<E>  get first;
  void  forEach(void f(E element));
  ...
}
```
## Async/Await

* Improve syntax
* Make it easier to reason about asynchronous programs

## Stream & Futures- Example

Small web-server in Dart(no error-handing):

```dart
runServer(){
   var future=HttpServer.bind(‘127.0.0.1’,4040);
   future.then((HttpServer server){// a Stream.
     server.listen((HttpRequest request){
        request.response.write('Hello, world!');
        request.response.close();
     });
    }); 
}
```

## Synchronous Web Server

```dart
runServer(){
   var server= HttpServer.bind('127.0.0.1',4040);
   for(HttpRequest request in server.requests){
      request.response.write('Hello,world!');
      request.response.close();
   }
}
```

## Async/Await- Syntactic Sugar

Syntactic Sugar

* Modifier on the body
* Function local
* Explicit


## Syntactic Sugar-Modifier

```dart
runServer() async {
   var future = HttpServer.bind('127.0.0.1',4040);
   future.then((HttpServer server){
      server.listen((HttpRequest request){
         request.response.write('Hello, world!');
         request.response.close();
        });
       });  
}
```

## Syntactic sugar - await

```dart
runServer() async{
  var server= await HttpServer.bind('127.0.0.1',4040);
  server.listen((HttpRequest request){
    request.response.write('Hello,world!');
    request.response.close();
  });
}

run Server async{
  var server= await HttpServer.bind('127.0.0.1',4040);
  await for(HttpRequest request in server){
     request.response.write('Hello, world!');
     request.response.close();
  }
}
```
Nitty Gritty Details

## Static Type - Await

The static type of an await is the generic type of the future.

API:
   Future<List<String>> File.readAsLines();

Async use:

   List<String> lines = await file.readAsLines();
   
## Await on non-Futrue value

* Await wait on any value
* Wraps value in Future if necessary

await 499

== 

await new Future.value(499);

## Closures

Modifiers works on closures.

foo(() async=> ...);
foo(() async {...});

## Bodywrap

Body of async functions is wrapped into Future.microtask.

Future runServer() async { ... }

==

Future runServer()=> new Future.microtask((){
 ...
});

## Bodywrap-2

Consequences:

* Errors are captured and return in Future
* Functions yield at entry
* Returned Futures are chained
* Return Type is Future

## Return Type

async functions always return a Future.

Future runServer() async {...}

Completed with the return value or error.

*

||Synchronous|Asynchronous
|---|----|---|
|Single|T|Future<T>|
|Multiple|Iterable<T>|Stream<T>|

## Sync*

Create Iterabels with sync*.

Iterable range(int from, int to) sync*{
   for(int i=form;i<to;i++){
      yield i;
   }
}

range(3,6).forEach(print);//Prints 3 4 5

## Yield

* Yield pushes one value
* yield* pipes through another iterable

```dart

Iterable recRange(int from,int to) sync*{
   if(from<to){
      yield from;
      yield* recRange(from+1,to)
   }
}

```

## Async*

Create Streams with async*

```dart
Stream bigFiles(List<String> fileNames) async*{
   for(var fileName in fileNames){
     var stat = await new File(fileName).stat();
     if(stat.size>10000) yield fileName;
   }
}
```

## Yield + Await

The async*  modifier enables:

* yield
* yield*
* awiat
* awiat for


## Async*- Details

Good to know:

* Function is not started before listen
* Finallies are executed on cancel
* Yield eventually returns to event loop

while(true) {yield 499;}

## Problem

Asynchronous debugging is hard.

* interleaved execution
* stack frames are lost

No Sliver Bullet

## Debugging - Example

```dart
foo() async {
  await 'asynchronous wait';
  throw 'bad';
}
bar()=> foo();
gee()=> bar();
main(){
  gee();
}

```

##  Cleaned Stacktrace

Stacktrace contains a lot of noisy internal frames.

After cleanup:

```dart
Unhandled exception:
Uncaught Error: bad
Stack Trace:
\#0 foo(example.dart:5:3)
<asynchronous entry>
```

## Zones

Zones:

*  Reason for noisy stack traces
*  Provide hooks for sophisticated stack-trace tracking


## Zone

* create ZoneDescription
* fork zone
* intercept all asynchronous calls
* wrap targets and store tracing information
* install that information when coming back
* ...

## Stack_trace package

Already done:

https://pub.flutter-io.cn/packages/stack_trace

## Debugging- Original

```dart
foo() async {
  await 'asynchronous wait';
  throw 'bad';
}
bar()=> foo();
gee()=> bar();
main(){
  gee();
}
```

```dart
import 'package:stack_trace/stack_trace.dart';
...
foo() async {
  await 'asynchronous wait';
  throw 'bad';
}
bar()=> foo();
gee()=> bar();
main(){
  Chain.capture(gee);
}
```

```dart
import 'package:stack_trace/stack_trace.dart';
...
foo() async {
  await 'asynchronous wait';
  throw 'bad';
}
bar()=> foo();
gee()=> bar();
main(){
  Chain.capture(gee,onError:(e.chain){print(chain.terse);});
}
```

## Summary

* Asynchronous programming well supported in Dart
* Async / await tremendously simplifies asynchronous programming
* stack_trace package helps in debugging



## Credits

Futures:

* As old as 1976
* Lots of JavaScript packages

Streams:

* Reactive Extensions(Rx)

Aysnc/await:

* .NET Framework 4.5


