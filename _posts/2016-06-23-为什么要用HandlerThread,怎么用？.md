---
layout: post
title: "为什么要用HandlerThread,怎么用？"
subtitle: "Android异步线程处理的最佳实践"
tags: [Android, Handler, HandlerThread, Looper]
comments: false
---

>作为一个Android开发，Handler机制是一定要了解的。在我面试过程中，发现很多人对Handler和Looper机制非常了解，对答如流，但是却不知道何为HandlerThread。本文我们就来简单聊一下它

HandlerThread是Thread的子类，它的作用很明确，文档说的也很清楚
>Handy class for starting a new thread that has a looper. The looper can then be used to create handler classes. Note that start() must still be called.

意思就是说HandlerThread帮我们创建好了Looper。
我们都知道，Android主线程的Looper是自动创建的，其他线程是没有创建Looper的，需要我们自己创建。一般做法很简单
```
@Override 
public void run() {
    Looper.prepare();
    Looper.loop();
}
```
``prepare()``和``loop()``两个方法不再赘述，我们先来看一个不用HandlerThread的例子：

```
Thread newThread = new Thread(new Runnable()
  {    
    @Override    
    public void run() {        
      Looper.prepare();        
      Looper.loop();    
    }
  });
newThread.start();
Handler handler = new Handler(newThread.getLooper());
```
相信不少人会用上面的方式创建一个异步线程的Handler，有没有问题呢？肯定有。
``newThread``的looper是在这个线程运行之后创建的，所以，当执行到``Handler handler = new Handler(newThread.getLooper());``的时候，``newThread``的looper可能还没有创建好！
****这就是为什么我们需要HandlerThread，并不仅仅是因为它帮我们创建了一个looper，更重要的是它为我们处理了这个异步问题。****
来看下HandlerThread的实现:
```
@Override
public void run() {    
  mTid = Process.myTid();    
  Looper.prepare();    
  synchronized (this) {        
    mLooper = Looper.myLooper();        
    notifyAll();    
  }    
  Process.setThreadPriority(mPriority);    
  onLooperPrepared();    
  Looper.loop();    
  mTid = -1;
}

public Looper getLooper() {    
  if (!isAlive()) {        
    return null;    
  }        
  // If the thread has been started, wait until the looper has been created.    
  synchronized (this) {        
    while (isAlive() && mLooper == null) {            
      try {                
        wait();            
      } 
      catch (InterruptedException e) { }        
    }    
  }    
  return mLooper;
}

```

简单的一个加锁帮我们做了最重要的事情。
>有人问我在``run()``方法里面，mTid开始赋值``Process.myTid()``,为什么后来又复制-1了呢？仔细想一下就有答案了，因为``Looper.loop()``是个死循环啊，执行到``mTid = -1``的时候，就是looper退出的时候。

---
插一句，这个mTid是干嘛的？
```
/** * Returns the identifier of this thread. See Process.myTid(). */
public int getThreadId() {    
  return mTid;
}
```
``Thread``里面是没有``getThreadId()``方法的，``Process.myTid() ``方法定义如下：
```
/** * Returns the identifier of the calling thread, which be used with * {@link #setThreadPriority(int, int)}. */
public static final int myTid() {    
  return Libcore.os.gettid();
}
```
原来``tid``是修改线程优先级、调度策略时用来做线程唯一标识的。那么在HandleThread中，把mTid置为－1是几个意思？笔者的理解是，HandlerThread本身是为looper服务的，looper终止以后，线程也会马上终止，为防止开发者错误使用，所以将mTid置为－1。

---

关于HandlerThread另外一个容易忽略的问题就是退出Looper。Looper通过``quit()``和``quitSafely()``方法退出(记得，Looper.loop()之后是一个死循环)，看看Looper的``loop()``方法的注释
```
/** * Run the message queue in this thread. Be sure to call * {@link #quit()} to end the loop. */
public static void loop()
```
Looper使用完毕执行quit本身就是一件容易忽略的事情，如果放到HandlerThread中，更是容易忘得一干二净。所以HandlerThread为我们提供了两个相同的方法：
```
public boolean quit(){}
public boolean quitSafely(){}
```
****不过说到底，维护Looper的生命周期还是我们每个开发者自己的事情，HandlerThread只不过是封装了一下，帮我们处理一个异步问题罢了。**** 