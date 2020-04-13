---
layout: post
title:  "(译)Android 消息传递机制"
date:   2020-03-26 14:16:17 +0800
categories: 
    - Android
tags:
    - 线程
---

目前为止,目前我们使用的线程通讯手段都是常规的 Java 应用所使用的. 管道、内存共享、可阻塞式队列可以应用到 Android 应用 ,但是会引起 UI 线程的问题, 因为这些手段都倾向于堵塞. UI 线程的响应性在使用带有阻塞行为的机制的时候是有风险的,因为这些机制通常可能会使线程挂起.

在 Android 中绝大部份线程之间的通讯是 UI 线程和 工作线程之间的通讯.因此,Android 平台定义了它自己的线程间通讯的方式. UI 线程能够以发送数据消息的方式让长时间运行的任务在后台线程被处理.消息传递机制是非阻塞式的 生产者-消费者模式,在这个模式下无论是生产者线程还是消费者线程,在消息处理的时候都不会被堵塞.


消息处理机制是 Android 平台的基石, API 位于 android.os 包下, 由下图 4-4 中所示的一系列类来实现功能.

![](4-4.api-overview.png)

<!--more-->

android.os.Looper

&emsp;&emsp;&emsp;&emsp; 跟有且只有一个消费者线程关联的消息分发者

android.os.Handler

&emsp;&emsp;&emsp;&emsp; 消费线程消息处理者,生产者线程插入消息到队列的接口. 一个Looper能有多个相关联的Handler,但是它们都会被插入到相同的队列中.

android.os.MessageQueue

&emsp;&emsp;&emsp;&emsp; 在消费者线程被处理的无限的消息列表.每一个Looper以及线程至多有一个消息队列.

android.os.Message

&emsp;&emsp;&emsp;&emsp; 将被在消费者线程执行的消息

如图4-5所示消息由生产者线程插入并且由消费者线程处理.

1. 插入: 生产者线程通过使用与消费者线程相关联的Handler往消息队列中插入消息
2. 获取:运行在消息者线程的Looper按照序列的顺序从队列中获取消息
3. 分发: handler负责在消费者线程处理消息.一个线程可能有多个Handler实例来处理消息,Looper确保了消息被分发到了正确的Handler.

![](4-5.png)

图 4-5 ,多个生产者线程和单个消费者线程之间的消息传递机制.每个消息都指向队列中的下一个消息,图中以左向的箭头指示.

# 例子: 基本的消息传递

在我们进一步详细的分析组件之前,让我们先来看个基本的消息传递的例子来熟悉下:

以下代码实现可能是最常见的使用场景之一,用户按下屏幕上的一个能够触发长时间操作的按钮,例如网络操作.为了不拖慢UI的渲染,一个假的 代表长时间运行的操作doLongRunningOperation()方法 ,需要运行在工作线程.因此,需要仅仅需要对一个生产者线程(UI线程)以及一个工作线程(Looper线程)进行设置.

我们的代码建立了一个消息队列.它在click()回调中处理按钮点击事件,这是在UI线程执行的.在我们的实现中,回调插入了一条假的消息到消息队列当中.为简洁起见,布局和UI组件在我们的代码事例中被省略了:

```
public class LooperActivity extends Activity {

    LooperThread mLooperThread;

    private static class LooperThread extends Thread { //1

        Handler mHandler;

  
        @SuppressLint("HandlerLeak")
        public void run() {
            Looper.prepare();//2
            mHandler = new Handler() {//3
                @Override
                public void handleMessage(@NonNull Message msg) {//4
                    if (msg.what == 0) {
                        doLongRunningOperation();
                    }
                }
            };
            Looper.loop();//5
        }

        private void doLongRunningOperation() {
            //Add long running operation here.
        }

    }

    public void onCreate(Bundle saveInstanceState) {
        super.onCreate(saveInstanceState);
        mLooperThread = new LooperThread();//6
        mLooperThread.start();
    }

    public void onClick(View v) {
        if (mLooperThread.mHandler != null) {//7
            Message msg = mLooperThread.mHandler.obtainMessage(0);//8
            mLooperThread.mHandler.sendMessage(msg);//9
        }
    }


    @Override
    protected void onDestroy() {
        super.onDestroy();
        mLooperThread.mHandler.getLooper().quit();//10
    }
}

```
1. 定义工作线程,作为消息队列的消费者
2. 将 Looper-隐式关联了消息队列与线程关联在一起
3. 设置一个被生产者所使用的Handler,让生产者用来插入消息到队列中.这里我们使用的是默认的构造器,因而会与当前线程的 Looper 绑定在一起. 因此,Handler 只能在 Looper.prepare() 之后创建,不然它没有东西可以绑定了.
4. 当消息被分发到工作线程的时候进入了回调.它会检查 what 参数并执行长时间运行的任务
5. 开始从消息队列向消息者线程分发消息.这是一个阻塞的调用,因此工作现象不会终止.
6. 启动工作线程,从而它能开始处理消息
7. 后台工作线程上handler的设置和handler在UI线程上的使用存在竞态.因此需要验证mHandler是否可用.
8. 初始化一个消息对象并将其what参数设置为0
9. 将消息插入队列
10. 终止后台线程.Looper.quit()的调用终止了 消息的分发并将 Looper.loop 从 阻塞中释放出来,从而run方法能终止,从而导致线程的终止.

# 消息传递所使用的类

现在,让我们更为具体地看看消息传递中所使用的具体组件,以及它们的用途.

## MessageQueue

消息队列由 android.os.MessageQueue 所代表.它由链接的消息构建而成,形成了一个无限的单向链表.生产者线程插入到队列的消息,稍后会被分发到消息者.消息是基于事件戳排序的.队列中时间戳最小的消息,排在分发给消费者的队列的最前面.但是,消息只有在时间戳的值小于当前的时间值的时候才会被分发.如果时间没有到,分发这个动作会等到当前时间超过时间戳.

图4-6 显示了一个有三个等待的消息的消息队列,它们是以时间戳顺序排列的, t1 < t2< t3. 只有一个消息越过了分发栏栅,这个栏栅代表当前时间.能够被分发的消息的时间是小于当前时间的.

![](4-6.png)

图 4-6 队列中的处于等待状态的消息.最右边的消息,是队列中最先被处理的.队列中的消息的箭头指向队列中的下一个消息.

如果没有消息被越过分发栏栅,而Looper以及做好获取下一个消息的转呗,消费者者线程会阻塞. 执行会在消息被发送到分发栏栅的时候恢复.

生产者能能够在任意时间以及队列的任意位置插入一个新的消息.队列中插入的位置是由时间戳决定的.如果跟等待的消息比,新的消息由最小的时间戳,那么它就会占据队列的第一个位置,将被下一个分发.插入总是遵循时间戳的顺序的.关于消息插入的部分会在 Hanlder这节进一步分析.

## MessageQueue.IdleHandler

如果没有消息要处理,消费者线程就会有一些空闲时间.例如,图4-7,显示了当消费者线程空闲的时候,由一段时间槽.默认情况下,消费者线程只会在空闲时间期间等待新的消息,但是处了等待,在空闲槽期间线程能被分配去执行其它任务.这个特性能够让非重要的任务让出它们的执行时间,直到没有其它消息竞争执行时间. 

![](4-7.png)

图 4-7. 如果没有消息被传到分发栏栅,在下一个等待消息被执行之前,就存在一段时间槽能用来执行.

当等待的消息被分发之后,没有其它消息被传到分发栏栅,就会有一段时间槽,期间消费者线程能利用它来执行其它任务.应用通过 android.os.Message.IdleHandler 接口来获取着的时间槽. IdleHanlder 接口,值当线程空闲的时候的一个监听回调.监听被附加到消息队列上并以如下调用的方式从队列上脱离出来:

```
  //Get the message queue of the current thread.
  Message mq=Looper.myQueue();
  //Create and register and idle listener
  MessageQueue.IdleHandler idleHanlder=new MessageQueue.IldeHandler();
  mq.addIdleHandler(idlehandler)
  //Unregister an idle Handler
  mq.removeIdleHanlder(idleHanlder)
```
idle hanlder 接口只包含了一个 callback 方法

```
  interface IdleHandler {
       boolean queueIdle();
  }

```

当消息队列监测到消息队列的空闲时间的时候,它会调用所有注册过 IdleHanlder 实例地方的 queueIdle().实现callback的责任就交给了应用.你应该避免运行长时间运行的任务,因为在它们运行的时候,它们会推迟等待中的消息.

queue() 的实现必须返回一个布尔值,具体含义如下:

true

&emsp;&emsp;&emsp;&emsp; idle hanlder 处于活跃态,它会继续在接下来的时间槽中接收回调.

false

&emsp;&emsp;&emsp;&emsp; idle handler 已经不活跃了,就接下去又时间槽,他也不会接受到回调了.这个效用是更通过 MessageQueue.removeIdleHanlder()移除监听是一样的.

### 例子:使用 Idle Hanlder 来终止一个不再使用的线程

当线程又空闲时间槽的时候,所有在 MessageQueue 上注册过 IdleHanlders 的,都会被调用,期间他在等待要处理的新消息. 空闲时间槽能够发生在第一个消息之前,消息之前,以及最后一个消息之后.如果多个内容提供者应该在消费者线程上按序处理数据, IdleHandler 可以用来终止消费者线程,当消息都处理完了,线程在内存中就没有用处了.有了 IdleHanlder,没有必要记录最后一个插入的消息去得知什么时候线程能被终止.

这只在生产者线程没有延迟地插入消息到队列中才有效,因为这样直到最后一个消息插入之后消费者线程才会进入空闲状态.

ConsumeAndQuitThread 方法展示了 带有 Looper和消息队列的线程的结构,这个线程会在没有消息要处理的时候被终止:

```
private static class ConsumeAndQuitThread extends Thread implements MessageQueue.IdleHandler {

        private static final String THREAD_NAME = "ConsumeAndQuitThread";

        public Handler mConsumerHandler;
        private boolean mIsFirstIdle = true;

        public ConsumeAndQuitThread() {
            super(THREAD_NAME);
        }

        @Override
        public void run() {
            Looper.prepare();

            mConsumerHandler = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    // Consume data
                }
            };
            Looper.myQueue().addIdleHandler(this);//1
            Looper.loop();
        }


        @Override
        public boolean queueIdle() {
            if (mIsFirstIdle) {//2
                mIsFirstIdle = false;
                return true;//3
            }
            mConsumerHandler.getLooper().quit();//4
            return false;
        }

        public void enqueueData(int i) {
            mConsumerHandler.sendEmptyMessage(i);
        }
    }
```

1. 当天启动的时候在后台线程注册IdleHanlder,然后Looper就准备好了,然后消息队列就设置好了.
2. 让第一个 queueIdle 调用通过,因此它会在首个消息到达之前发生
3. 在一次调用的时候返回true,所以IdleHandler仍然是注册着的
4. 终止线程

 消息插入是由多个线程并发插入的,并且模拟一个随机的插入时间:
 
```
final ConsumeAndQuitThread consumeAndQuitThread = new ConsumeAndQuitThread();
        consumeAndQuitThread.start();
        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 0; i < 10; i++) {
                        SystemClock.sleep(new Random().nextInt(10));
                        consumeAndQuitThread.enqueueData(i);
                    }
                }
            }).start();
        }
```

# 消息(Message)

消息队列上的每一项都是 android.os.Message 类,这是一个运送数据或任务,或什么也不运的容器对象.数据由消费者线程处理,然而任务只是在出队列的时候执行,你不需要做其它处理:

消息知道他的接受处理者Hanlder,并通过Message.sendToTarget()将其自身入队:

```
Message m=Message.obtain(hanlder,runnable);
m.sendToTarget();
```

正如我们会在handler这一节会看到的,handler 最常用来进行消息入队,同时就消息插入提供了更多的灵活性.

## 数据消息(Data Message)

数据集有着多个能够被传递给消费者线程的参数,如表 4-2 所示:

表 4-2 消息参数

|参数名称|类型|用途|
|---|---|---|
|what|int|消息标识.消息的通讯意图|
|arg1,arg2|int|用于传递整型的简单数据,如果最多只有两个整数值要被传递到消费者线程,这些参数比分配一个bundle来传递更为高效.|
|obj|Object|任意对象.如果一个对象要传递到另一个进程的线程中,它必须实现 Parcelable|
|data|Bundle|任意数据值的容器|
|replyTo|Messager|指向其它进程中的Hanlder.使跨进程消息通讯成为可能|
|callback|Runnable|在线程上执行的任务.这是一个持有来自Hanlder.post 方法的 Runnable 对象的内部实例域.|

## 任务消息(Task Message)

任务由运行在消费者线程的java.lang.Runnable 对象代表.任务对象除了任务本身之外不能包含任何数据.

消息队列可以包含仍以数据和任务消息的组合.消费者线程以线性的方式对它们进行处理,不管它们是什么类型.如果消息是数据消息,消费者线程会处理数据.任务消息的处理方式是让Runnable执行在消费者线程上,但是消费者线程没有像数据消息一样在Handler.handleMessage(Message)中收到任何消息.

消息的生命周期是简单的:生产者创建消息,并且最终消息由消费者线程处理.这样的描述适用于大部分使用场景,但是出问题的时候,对消息处理的进一步理解就很有价值了.让我们看看在它的生命周期中,在消息身上到底发生了什么,这个过程可以分为如图4-8中的最重要的4步.运行时将消息对象存储到一个应用范围的对象池,从而使得能对之前的消息进行复用.这个避免了在每个处理的时候创建新的实例的消耗.消息对象的处理时间通常是很短的,并且许多消息在在一个时间单位就处理完了.

![](4-8.png)

图 4-8 消息生命周期状态

状态转换部分由应用控制,部分由平台控制.注意状态不是可观察的,并且应用不能观察到从一个状态到另一个状态的变化.因此,应用不应该对于消息的当前状态作出任何假定.

## 初始化(Initialized)

在初始化状态,一个可修改状态的消息对象已经被创建,并且如果它是数据对象的话,已经被数据填充了.应用负责使用以下的调用来创建消息对象.它们能够从对象池中获取一个对象:

* 明确的对象构造器
  
  ```
  Message m=new Message();
  ```
* 工厂方法:
  * 空消息:
  
  		```
  		 Message m=Message.obtain();
  		```
  * 数据消息:
  
      ```
         Message m=Message.obtain(Handler h);
         Message m=Message.obtain(Hanlder h,int what);
         Message m=Message.obtain(Handler h,int what, object o);
         Message m=Message.obtain(Handler h,int what, int arg1, int arg2);
         Message m=Message.obtain(Hanlder,int what,int arg1,int arg2,Object o);
         
      ```
      
   * 任务消息
      
      ```
       Message m=Message.obtain(Handler h,Runnable task);
      ```
      
   *  拷贝构造器
   
      ```
       Message m=Message.obtain(Message originMsg);
      ```

## 等待(Pending)

消息已经由生产者线程插入队列,并且它正等待着被分发到消费者线程.

## 已分发(Dispatched)

在此状态下,Looper已经获取并从队列中移除了消息.消息已经被分发到了消费者线程,并且正在被处理.对于这部操作没有应用api,因为分发是由Looper控制的,而不受应用的影响.当Looper分发一个消息,它会检查消息的分发信息然后分发消息到正确的接受者.一旦分发了,消息是在消费者线程上执行的.

## 已回收(Recycled)

在生命周期的这一时刻,消息的状态已经被清理了并且实例已经被交还给了消息池.当它结束在消费者线程的执行,Looper 处理了消息的回收.消息的回收是由运行时处理的,并且不应该由应用显式地完成.

一旦消息被插入了队列,内容应该不应在被修改了.理论上,在消息处理之前修改内容是合法的.但是,因为状态是不可观察的,在变更数据的时候,消息已经可能被消费者线程处理的,就会导致线程安全问题.更糟糕的是,消息可能已经被回收,因为它已经被交还给了消息池并且被另一个生产者传入到另一个队列中.

# Looper

android.os.Looper 类负责将队列中的消息分发到相关联的 handler.就如图4-6所示,所有的消息都得发送到分发栏栅,从而能够被Looper进行分发.只要队列中有消息能被分发,Looper就能保证消费者线程能收到消息.当没有消息被传到分发栏栅的时候,消费者线程就会阻塞直到有消息传到分发栏栅.

消费者线程并不直接和消息队列交互来获取消息.而是,在Looper添加到线程之后,消息队列才添加到线程.Looper对消息队列进行管理并促进消息到消费者线程的分发.

默认情况下,只有UI线程会有Looper,在应用中创建的线程需要获取looper并显式地进行关联.当Thread上的Looper创建好了,他就被连接到了消息队列.Looper就成了队列和线程之前的中介.设置过程是在线程的run方法中完成的.

```
  class ConsumerThread extends Thread {
       @Override
       public void run(){
          Looper.prepare();//1
          
          //Handler creation omitted.
          
          Looper.loop();//2
       
       }
  }

```

1. 第一步是创建Looper,这一步是由静态的 prepare() 方法完成的;它会创建一个消息队列并将当前的线程与之关联.此时,队列已经做了了消息插入的准备,但是它们没有被分发到消费者线程.
2. 开始在消息队列中处理消息.这是一个阻塞方法,从而确保run()方法不会终止,当run()阻塞的时候,Looper分发消息到消费者线程进行处理.

一个线程只能有一个相关的Looper,当应用尝试设置第二个Looper的时候会发生运行时错误.因此,一个线程只有有一个消息队列,意味着有多个生产者线程发送的消息在消费者线程中是顺序处理的.
因此,当前的消息在它被执行之前都会推迟后续消息的执行.需要长时间执行的消息不能被使用,因为它们可能对队列中其它重要的消息造成延迟.

## Looper 终止

Looper如果要停止执行消息的话,就要使用 quit 或者 quitSafely. quit()让looper停止对队列中的其余消息进行分发,所有队列中等待的消息包括那些已经到到分发栏栅的,都会被丢弃.另一方面,quitSafely,只会丢弃那些没有到达分发栏栅的消息.在Looper终止之前,符合分发条件的等待消息会被处理掉. quitSafely 是在api 18添加的,之前的api只支持quit.


终止Looper并没有终止线程,它只是跳出了Looper.loop()并让线程恢复运行在调用loop的方法.但是你不能启动旧的Looper或新建一个looper,所以线程就不能在入队和处理消息了.如果你调用Looper.prepare(),它会抛出运行时异常,因为线程已经有Looper附加了.如果你调用Looper.loop,它会阻塞,但是没有消息会从队列中分发出来.


## UI 线程 Looper

UI 线程是唯一一个默认就有相关联Looper的线程.和其它由应用本身创建的线程一样,它是一个常规的线程,但是Looper是在应用组件初始化之前就被关联到线程的.

UI 线程 Looper和其它应用线程Looper还是由一些实际区别的:

* 它能通过Looper.getMainLooper()方法从任意地方访问.
* 它不能被终止.调用Looper.quit()会抛出运行时异常.
* 运行时通过Looper.prepareMainLoop()将Looper和UI线程关联在一起.每一个应用只能执行一次.正因如此,尝试附加主线程Looper到其它线程或抛出异常.


# Handler

目前为止,关注点在于Android线程通讯的内部机制,但是应用大部分是和android.os.Hanlder打交道的.它是一个双侧的API,既要处理消息的插入队列也要进行消息的处理.如同 4-5所示,它同时由生产者线程和消费者线程调用来进行以下处理:

* 创建消息
* 插入消息到队列中
* 在消费者线程处理消息
* 队列中消息的管理

## 设置
谈及到Hanlder的职责,handler 与 Looper ,消息队列以及消息都有交互.如图4-4所示:handler唯一直接的关系是和Looper,Looper是和消息队列连接的.如果没有Looper,handler就不能工作,它们不能与队列组合来插入消息,因此它们不能收到任何消息进行处理.因此,handler实例是构造的时候与Looper 实例绑定的:

* 没有显式Looper的构造器绑定的是当前线程的Looper:UI线程由平台内部的类android.app.ActivityThread 管理

   ```
     new Hanlder();
     new Hanlder(Hanlder.Callback)
   ```
* 在构造器中显式指定Looper进行绑定

   ```
     new Handler(Looper);
     new Hanlder(Looper,Hanlder.Callback);
   ```

如果没有显式构造器的Looper在一个没有Looper的线程被调用(例如:它没有调用Looper.prepare()),就没东西让Hanlder绑定了,就会导致运行时异常.一旦handler绑定到了一个Looper,这个绑定是不可变的.

一个线程能够有多个handler,来自这些hanlder的消息共存于消息中,但是能够被分发到正确的Handler实例.如图 4-9所示:


![](4-9.png)

图4-9 多个handler使用同一个Looper,插入消息的handler和处理消息的handler是同一个.

多个hanlder了并没有导致并发执行,消息仍然在一个队列中并按序被处理.

## 消息创建

为简单起见,Handler类提供了如下所示的工产方法封装用来创建Message类的对象:

```
  Message obtainMessage(int what, int arg1,int arg2);
  Message obtainMessage()
  Message obtainMessage(int what,int arg1,int arg2,Object obj)
  Message obtainMessage(int what,Object obj)
```

从Handler获取的message是从消息池获取而来的,并且与请求它的Hanlder实例隐式连接.这个连接使Looper能将每个消息分发到正确的Hanlder成为可能.

## 消息插入

根据消息类型的不同,Hanlderyou着多种插入队列的不同方式.插入任务消息的方法是以post为前缀的,而数据消息是以send为前缀的:

* 添加一个任务到消息队列当中:
  
  ```
  boolean post(Runnable r)
  boolean postAtFrontQueue(Runnable r)
  boolean postAtTime(Runnable r, Object token,long uptimeMillis)
  boolean postAtTime(Runnable r,long uptimeMills)
  boolean postDealyed(Runnable r,long dealyMillis)
  ```
  
* 添加一个数据对象到消息队列中
  
  ```
  boolean sendMessage(Message msg)
  boolean sendMessageAtFrontOfQueue(Message msg)
  boolean sendMessageAtTime(Message msg,long uptimeMillis)
  boolean sendMessageDelayed(Messsage msg,long delayMillis)
  ```

* 添加一个简单数据对象到消息队列中

  ```
  boolean sendEmptyMessage(int what)
  boolean sendEmptyAtTime(int what, long uptimeMillis)
  boolean sendEmptyDelayed(int what, long delayMillis)
  ```

所有的插入方法都会放一个消息到队列中,即使应用没有显式地创建消息对象.post系列中的Runnable, send 系列中的 what,都被包含在Message对象中,因为消息是队列中唯一可以包含的对象.

每个插入消息的队列都有会有个指示消息能够被分发到消费者线程的的时间参数.队列的排序是基于时间参数的,并且踏实应用唯一能影响分发顺序的方式:

default:

&emsp;&emsp;&emsp;&emsp; 立即能够被分发

at_front    

&emsp;&emsp;&emsp;&emsp; 消息在时间为0的时候被分发.因此,它是下一个被分发的消息,除非有另一个消息在它被处理之前插入了.

delay

&emsp;&emsp;&emsp;&emsp; 在能够被分发之前延长一定的时间

uptime

&emsp;&emsp;&emsp;&emsp; 消息能够被分发的绝对时间


尽管显式的延迟和运行时间是可以指定的,但是咩哥消息的处理时间是不确定的.这依赖于已有消息的处理时间和操作系统调度情况.

插入消息到队列中不是保险的.如下表4-3所示的一些常见错误会发生.

表4-3.消息插入错误

|错误|错误返回|常见的应用问题|
|---|---|---|
|消息没有Hanlder|RuntimeException|消息是从一个没有Hanlder的Message.obtain()方法创建的.|
|Message已经被分发了并且已经被处理了|RuntimeExeception|同样的消息被插入了两次|
|Looper已经退出|返回 false|在Looper.quit()被调用之后消息被插入了|


Handler类的dispatchMessage()方法是用来让Looper分发消息到消费者线程的.如果由应用直接使用,消息会在调用的线程处理而不是在消费者线程处理.

## 例子: 双向消息传递

HanlderExampleActivity 模拟了一个长时间运行的操作,这个操作是在用户点击按钮的时候启动的.长时间运行的任务在后台线程运行,与此同时,用户界面上展示了一个进度条,这个进度条会在结果返回UI线程的时候被移除.

首先,设置Activity:

```
   public class HanlderExampleActivity entends Activity{
      
        private final static int SHOW_PROGRESS_BAR=1;
        private final static int HIDE_PROGRESS_BAR=0;
        private BackgroundThread mBackgroundThread;
        
        private TextView mText;
        private Button mButton;
        private ProgressBar mProgressBar;
        
        
        @Override
        public void onCreate(Bundle savedInstanceState){
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_handler_example);
            
            mBackgroundThread=new BackgroundThread();
            mBackgroundThread.start();//1
            
            mText=(TextView)findViewById(R.id.text);
            mProgressBar=(ProgressBar)findViewById(R.id.progress);
            mButton.setOnClickListener(new OnClickListener(){
            
                @Override
                public void onClick(View v){
                    
                    @Override
                    public void click(View v){
                    
                       mBackgroundThread.doWork(); //2
                    }
            
            });
             
        }
      
       @Override
       protected void onDestroy() {
           super.onDestroy();   
           mBackgroundThread.exit();//3
       }
       //... The rest of the Activity is defined futher down
    
   }
```

1. 带有消息队列的后台线程是在HanlderExampleActivity创建的时候启动的.它处理了来自UI线程的任务.
2. 当用户点击按钮,一个新的任务就被发送到后续线程.因为任务在后台线程是线性处理的,在它们被处理完之前,多次按钮点击可能导致排队.
3. 当 HanlderExampleActivity 被销毁的时候,后台线程被终止了.

后台线程是用来承载来自UI线程的任务的.当它运行的时候,在HanlderExampleActivity生命周期期间它就能接受消息.它并没有暴露出它内部的Hanlder, 而是将所有对Handler的访问封装成公有方法 doWork 和 exit:
  
```
   private class BackgroundThread extends Thread {
      
      private Handler mBackgroundHanlder;
      
      public void run(){ //1
         Looper.prepare();
         mBackgroundHanlder=new Hander();//2
         Looper.loop()
      }
      
      public void doWork(){
         mBackgroundHanlder.post(new Runnable(){//3
             @Override
             public void run(){
                Message uiMsg=mUiHanlder.obtainMessage(
                   SHOW_PROGRESS_BAR,0,0,null);//4
                mUiHanlder.sendMessage(uiMsg);//5
                
                Random r=new Random();
                int randomInt=r.next(5000);
                SystemClock.sleep(randomInt);//6
                
                uiMsg=mUiHanlder.obtainMessage(
                     HIDE_PROGERSSS_BAR,randomInt,0,null);//7
                     mUiHanlder.sendMessage(uiMsg);//8
                   
             }
           
         });
        
      }
      
      public void exit(){//9
          mBackgroundHanlder.getLooper().quit()
      }
   
   }
```

1. 将Looper与线程相关联
2. Handler只处理Runnable.因此,它不需要实现Hanlder.handleMessage.
3. 发出一个需要在后台线程执行的长时间任务
4. 创建一个只包含一个what参数以及一个SHOW_PROGRESS_BAR命令的的消息对象给到UI线程,从而它能够显示进度条.
5. 发送消息到UI线程
6. 模拟一个随机时间长度的长时间任务
7. 创建一个带有randomInt的消息对象,它被传入arg1参数.what参数包含了一个命令-HIDE_PROGRESS_BAR 来移除进度条.
8. 最后消息通知UI线程任务结束了,并分发出一个结果
9. 退出Looper,从而线程能终止

UI线程定义了它自己的Hanlder,从而它能接收命令来控制进度条来用来自后台线程的结果来更新ui界.

```
private final Handler mUiHanlder =new Hanlder() {

    public void hanldeMessage(Message msg){
        
        switch(msg.what){
           case SHOW_PROGRESS_BAR:
             mProgressbar.setVisibility(View.VISIBLE);
             break;
           case HIDE_PROGRESS_BAR:
             mText.setText(String.valueOf(msg.arg1));
             mProgress.setVisibility(View.INVISIBLE)
              break;
             
        }
     
    }
}

```

1. 显示进度条
2. 隐藏进度条并用处理好的结果更新TextView

## 消息处理

被Looper分发的消息由Hanlder在消费者线程处理,消息类型决定了处理过程:

*任务消息*

消息对象只包含了一个Runnable并且没有数据.因此,处理过程定义在Runnable的在run()方法中,这个过程自动地是在消费者线程执行的,而不用调用Hanlder.handleMessage().

*数据消息*
  
当消息包含数据的时候,Hanlder是数据的接收者并负责对它进行处理.消费者线程以重载hanlder.handleMessage(Message msg) 方法来处理数据.如下所示有两种方式:

其中一种是将定义handlerMessage作为创建Handler的一部分.这个方法应该在消息队列准备好的时候就定义(在Looper.prepare()调用之后)但是在消息获取开始之前(在Looper.loop()被调用之前).

以下是一个设置数据消息处理的模版:

```
  class ConsumerThread extends Thread {
      Hanlder mHandler;
      @Overide
      public void run(){
          Looper.prepare();
          mHanlder=new Hanlder(){
             public void hanldeMessage(Mssage msg){
                //Process data message here
             }
          };) 
          Looper.loop();
      }
  
  }

```

在这份代码中, Handler是以匿名内部类的形式定义的,但是它也能够被定义成常规的内部类.

有一种替代扩展Handler类的替代方式是使用Hanlder.Callback.它定义了一个handleMessage()方法以及一个hanlder.hanldeMessage()所有没有的返回参数.

```
 public interface Callback{
     public boolean hanldeMessage(Message msg)
 }
```

有了Callback接口,就没有必要去扩展Hanlder类了.实际上,Callback实现能够被传递到Hanlder构造器中,然后它会接收被分发的消息进行处理:

```
  public class HanlderCallbackActivity extends Activity implements Hanlder.Callback {
     Hanlder mUiHanlder;
     
     @Override 
     public void onCreate(Bundle savedInstanceState){
         super.onCreate(savedInstance);
         mUiHanlder=new Hanlder(this);
     }
     
     @Override
     public boolean handleMessage(Message message){
         //Process messages
         return true;
     }
    
  }
```

如果消息被处理了,Callback.HandleMessage()应该返回true,它保证了消息完成处理后不再进行处理.但是,如果返回了false,消息被传入了Handler.hanldeMessage()方法来进一步处理,它添加了一个消息处理者,这个调用过程在Handler自己的方法之前.Callback 处理者能够在Hanlder接收到消息之前对消息进行拦截和修改.以下的代码展示了用Callback()方法拦截消息的原则:

```
  public class HanlderCallbackActivity entends Activity implements Hanlder.Callback{//1
     
     @Override
     public boolean hanldeMessage(Message msg){//2
         switch(msg.what){
            case 1:
               msg.what=11;
               return true;
            default:
                msg.what=22;
               return false;    
         } 
       
     }
     
     //Invoked a button click
     
     public void onHanlderCallback(View v){
         Hanlder hanlder=new Hanlder(this){
            @Override
            public void handleMessage(Message msg){
               //Procress message//3
            }
         };
         handler.sendEmptyMessage(1);//4
         hanlder.sendEmptyMessage(2);//5
           
     }
  
  }
```

1. HanlderCallbackActivity 实现了 Callback接口来拦截消息
2. Callback 实现拦截了消息.如果 msg.what 是1,它就返回true-消息就被处理了.否则的话,它就会将msg.what的值修改为22并返回false,消息没有被处理,所以它被传递到了Hanlder的handle实现.
3. 在第二个Hanlder中处理消息
4. 插入一个msg.what=1的消息,这个消息被Callback拦截并返回true.
5. 插入一个msg.what=2的消息.这个消息被Callback改变然后传递给Hanlder,从属Hanlder 出来的msg=22













