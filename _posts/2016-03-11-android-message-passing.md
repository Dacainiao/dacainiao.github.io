---
layout: post
date: 2016-03-11 14:13:10 +0800
title:  "Android 的消息传递"

---

到目前为止，关于线程间通信的讨论都是在常规的 Java 语言中，在任何一个 Java 应用中都可以见到。由于 Android 应用中 UI 主线程的阻塞倾向，在将管道，共享内存以及阻塞队列这些机制应用于 Android 应用时，会引起很多问题。当在使用具有阻塞行为的机制时，UI 主线程的响应性会受到威胁，并且很有可能导致其被挂起。

在 Android 应用中，最常见的线程间通信发生在 UI 主线程和工作线程之间。因此，Android 平台定义了自己的消息传递机制来达到线程间通信的目的。UI 主线程可以通过将要处理的数据信息发送给后台线程的方式将耗时的任务移交给后台线程处理。该消息传递机制是一种非阻塞的生产者-消费者模式，因此在消息传递期间，生产者线程和消费者线程都不会发生阻塞。

这种机制在 Android 平台中是一种基本的消息处理机制，相关 API 和实现的一系列类位于 **android.os** 包中，见**图4-4**：

![图4-4](/resources/images/figure-4-4.png)

***图4-4***

**android.os.Looper**

&emsp;&emsp;消息的分发者，与有且只有一个消费者线程相联系。

**android.os.Handler**

&emsp;&emsp;消费者线程的消息处理者，并且作为生产者线程向消息队列中插入消息的接口。一个 Looper 可以与多个 Handler 相联系，但是它们插入消息的消息队列是同一个。

**android.os.MessageQueue**

&emsp;&emsp;在消费者线程上处理的消息的无界链表。每一个 Looper 和线程至多拥有一个 MessageQueue.

**android.os.Message**

&emsp;&emsp;消费者线程上执行的的消息类。

生产者线程插入消息以及消费者线程处理消息的过程如**图4-5**所示：

![图4-5](/resources/images/figure-4-5.png)

***图4-5 展示了多个生产者线程与一个消费者线程之间的消息传递***

1. 插入消息：生产者线程通过与消费者线程连接的 Handler 将消息插入消息队列。
2. 取回消息：运行在消费者线程的 Looper 将消息从消息队列中顺序的取出。
3. 分发消息：Handler 负责处理运行在消费者线程的消息。一个线程可以拥有多个用来处理消息的 Handler 实例；Looper 则保证消息能够分发给正确的 Handler。

## 示例：简单的消息传递

在我们详细的剖析各个组件之前，让我们先通过一个简单的示例来熟悉代码。

接下来的代码实现可能是最常见的消息传递用例之一。用户按下屏幕上的一个按钮，触发一个耗时操作，比如网络请求。为了避免阻塞 UI 的渲染，该耗时操作（此例中用 doLongRunningOperation() 方法代表）必须在工作线程上执行。因此，我们这里只有一个生产者线程（ UI 线程 ）和一个消费者线程（LooperThread）。

这里我们设置了一个消息的队列，它像平常一样在 onClick() 回调中处理按钮点击事件，该操作发生在 UI 线程。在我们的实现中，该回调向我们的消息队列中插入了一条伪消息。简单起见，此处省略了布局和控件相关的代码。

{% highlight java %}
public class LooperActivity extends Activity {

    LooperThread mLooperThread;
    
    private static class LooperThread extends Thread { //1.
        public Handler mHandler;
        public void run() {
            Looper.prepare(); //2.
            mHandler = new Handler() { //3.
            
            	@Override
                public void handleMessage(Message msg) { //4.
                    if (msg.what == 0) {
                        doLongRunningOperation();
                    }
                }
                
            };
            Looper.loop(); //5.
        }
    }
    
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mLooperThread = new LooperThread(); //6.
        mLooperThread.start();
    }

    public void onClick(View v) {
        if (mLooperThread.mHandler != null) { //7.
            Message msg = mLooperThread.mHandler.obtainMessage(0); //8.
            mLooperThread.mHandler.sendMessage(msg); //9.
        }
    }

    private void doLongRunningOperation() {
    	 // Add long running operation here.
    }

    protected void onDestroy() {
        mLooperThread.mHandler.getLooper().quit(); //10.
    }
}
{% endhighlight %}

1. 定义工作线程，作为消息队列的消费者。
2. 把一个 Looper 和一个隐式的 MessageQueue 与线程联系起来。
3. 创建一个 Handler 用来向消息队列插入消息。这儿我们用的是默认构造函数，因此它对当前线程的 Looper 一无所知。所以，这个 Handler 只能在 Looper.prepare() 之后创建。
4. 当消息分发给工作线程时，执行此回调。它检查 what 参数，然后执行耗时操作。
5. 开始将消息队列里的消息分发给消费者线程。这个调用是阻塞的，所以工作线程不会结束。
6. 启动工作线程，准备处理消息。
7. 因为在工作线程中创建 mHandler 和在 UI 线程使用它之间存在竞争条件，所以需要判断 mHandler 是否有效。
8. 用一个设为0的 what 参数初始化一个消息对象。
9. 向消息队列中插入消息。
10. 终结工作线程。Looper.quit() 能够终止消息的分发并且使 Looper.loop() 从阻塞中释放出来，因此，run() 方法可以结束执行，从而导致线程终止。

##  消息传递中使用到的类

现在让我们更细致的观察消息传递中特定的组件以及它们的使用。

### MessageQueue