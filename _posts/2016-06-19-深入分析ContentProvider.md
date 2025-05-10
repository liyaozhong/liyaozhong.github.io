---
layout: post
title: "深入分析ContentProvider"
subtitle: "Android跨进程数据访问的实现原理"
tags: [Android, IPC, 源码分析]
comments: false
---

ContentProvider是Android四大组件之一，承担着跨进程数据访问的重要职责。本文就从一次ContentProvider访问入手，分析下它是怎么完成跨进程数据访问的。

既然是跨进程，那就必须有一个客户端进程和一个ContentProvider进程，我们先从客户端进程分析，看它如何访问ContentProvider进程。以Query操作为例，一般情况下，当我们需要访问ContentProvider的时候一般都会执行这么一句：
```
getContentResolver().query(uri, projection, selection, selectionArgs, sortOrder);
```

ContentResolver是什么？Google的解释是"This class provides applications access to the content model"。实际上我们正是通过ContentResolver来获取一个IContentProvider对象，通过IContentProvider对象我们尽可以进行IPC通讯了。``getContentResolver()``方法定义Context类中，实际上Context是一个抽象类，在客户端应用程序中``getContext()``实际上返回的是一个ContextImp对象，``getContentResolver()``方法就定义在ContextImp.java中，并且最终返回ContextImp的内部类ApplicationContentResolver，从名字上看这是一个Application级别的对象。
ApplicationContentResolver我们稍后再说，先看下query方法都干了什么：
```
public final Cursor query(final Uri uri, String[] projection,
            String selection, String[] selectionArgs, String sortOrder,
            CancellationSignal cancellationSignal) {
        IContentProvider unstableProvider = acquireUnstableProvider(uri);
        if (unstableProvider == null) {
            return null;
        }
        IContentProvider stableProvider = null;
        try {
            long startTime = SystemClock.uptimeMillis();

            ICancellationSignal remoteCancellationSignal = null;
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
                remoteCancellationSignal = unstableProvider.createCancellationSignal();
                cancellationSignal.setRemote(remoteCancellationSignal);
            }
            Cursor qCursor;
            try {
                qCursor = unstableProvider.query(uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
            if (qCursor == null) {
                return null;
            }
            // force query execution
            qCursor.getCount();
            long durationMillis = SystemClock.uptimeMillis() - startTime;
            maybeLogQueryToEventLog(durationMillis, uri, projection, selection, sortOrder);
            // Wrap the cursor object into CursorWrapperInner object
            CursorWrapperInner wrapper = new CursorWrapperInner(qCursor,
                    stableProvider != null ? stableProvider : acquireProvider(uri));
            stableProvider = null;
            return wrapper;
        } catch (RemoteException e) {
            // Arbitrary and not worth documenting, as Activity
            // Manager will kill this process shortly anyway.
            return null;
        } finally {
            if (unstableProvider != null) {
                releaseUnstableProvider(unstableProvider);
            }
            if (stableProvider != null) {
                releaseProvider(stableProvider);
            }
        }
    }
```
##上面的代码有四个关键步骤:
1.``acquireUnstableProvider``
2.``unstableProvider.query(......)``
3.``qCursor.getCount();``
4.``return new CursorWrapperInner(......)``

接下来我们一步一步分析这四个关键步骤。

##第一步：acquireUnstableProvider

乍看上去感觉怪怪的，好端端的为什么加上了一个Unstable的标签?难道还有stable的不成？事实确实如此，我们知道此时的ContentResolver实际上是一个ApplicationContentResovler对象，来看下ApplicationContentResovler

```
private static final class ApplicationContentResolver extends ContentResolver {
        private final ActivityThread mMainThread;
        private final UserHandle mUser;

        public ApplicationContentResolver(
                Context context, ActivityThread mainThread, UserHandle user) {
            super(context);
            mMainThread = Preconditions.checkNotNull(mainThread);
            mUser = Preconditions.checkNotNull(user);
        }

        @Override
        protected IContentProvider acquireProvider(Context context, String auth) {
            return mMainThread.acquireProvider(context, auth, mUser.getIdentifier(), true);
        }

        @Override
        protected IContentProvider acquireExistingProvider(Context context, String auth) {
            return mMainThread.acquireExistingProvider(context, auth, mUser.getIdentifier(), true);
        }

        @Override
        public boolean releaseProvider(IContentProvider provider) {
            return mMainThread.releaseProvider(provider, true);
        }

        @Override
        protected IContentProvider acquireUnstableProvider(Context c, String auth) {
            return mMainThread.acquireProvider(c, auth, mUser.getIdentifier(), false);
        }

        @Override
        public boolean releaseUnstableProvider(IContentProvider icp) {
            return mMainThread.releaseProvider(icp, false);
        }

        @Override
        public void unstableProviderDied(IContentProvider icp) {
            mMainThread.handleUnstableProviderDied(icp.asBinder(), true);
        }
    }
```
实际上，是否是stable的，都将调用ActivityThread的acquireProvider方法，区别就是最后的一个参数boolean stable。这个机制是API 16引入的，文章的最后会对此进行说明。现在我们只要知道它最终走到了ActivityThread的acquireProvider就可以了。在ActivityThread的acquireProvider方法中，我们首先会去acquireExistingProvider，从字面上就可以看出这是从一个类缓存的地方读取已经保存的ContentProvider对象，如果不存在，就会调用``ActivityManagerNative.getDefault().getContentProvider(getApplicationThread(), auth, userId, stable);``从这儿开始，ActivityManagerService就要登场了，上面的代码最终会通过binder通信调用ActivityManagerService的getContentProviderImpl，这块的逻辑比较复杂，涉及到了Android组件启动的过程，我们只需知道客户端调用会阻塞在``ActivityManagerNative.getDefault().getContentProvider``，ActivityManagerService启动目标ContentProvider进程后(如果ContentProvider进程已经存在则不必重启)，返回一个目标ContentProvider的实例。在这儿需要说明的是，ContentProvider可以再Manifest中配置一个叫做android:multiprocess的属性，默认值是false，表示ContentProvider是单例的，无论哪个客户端应用的访问都将是一个ContentProvider对象(当然，必须是同一个ContentProvider，即Uri或者Authority name是一个)，如果设为true，系统会为每一个访问该ContentProvider的进程创建一个实例。因为android:multiprocess的默认值是false，所以我们在写自己的ContentProvider的时候还是要注意并发的情况。

扯得有点远了，回到我们之前步骤，我们已经得到了ContentProvider的实例，这个时候第一步就完成了，接下来看第二步。

##第二步：unstableProvider.query(......)

unstableProvider实际上是IContentProvider实例，IContentProvider是进行IPC通讯的接口，这个query实际上调用的是目标ContentProvider中的query方法，当然，在真正调用目标ContentProvider的query方法之前，还需要经过enforceReadPermission方法，这一步主要是看下该ContentProvier有没有export，读写权限等等(enforceReadPermission方法只判断读权限)。随后执行query方法，并且返回一个cursor对象。

以上就是第二步的大致逻辑，不过不要以为这么简单就结束了。IContentProvider的query可是跨进程的，我们知道ContentProvider的query方法可是五花八门，有访问数据库返回SQLiteCursor的，有返回MatrixCursor的等等，那么IContentProvider返回的那个cursor到底是什么呢？我们来看下IContentProvider的这个IPC通信到底是怎么回事

IPC通信需要两端，对于我们的例子，这两端分别是ContentProviderProxy和ContentProviderNative，首先会执行ContentProviderProxy的``query``方法，然后通过binder通信执行ContentProviderNative的``onTransact``方法。ContentProviderProxy的query方法有一下五个主要步骤：

1. new一个BulkCursorToCursorAdaptor对象——adaptor
2. 填充data用于binder通信
3. 调用mRemote.transact，这是一个阻塞的过程，直到ContentProviderNative的onTransact方法返回
4. 读取reply数据，new一个BulkCursorDescriptor并以此初始化adaptor
5. return adaptor

ContentProviderNative的onTransact会调用ContentProvider的query方法，并根据query返回的cursor初始化一个CursorToBulkCursorAdaptor对象，最终将BulkCursorDescriptor对象写入reply中。

至此我们知道了，不管我们的ContentProvider query方法返回的到底是什么样的cursor，最终在客户端进程都将会被封装在一个BulkCursorToCursorAdaptor对象中，那么这个BulkCursorToCursorAdaptor对象是不是就是我们在客户端调用query返回的最终类型呢？别急，往下看。

##第三步：qCursor.getCount();

这一步看似鸡肋，实际上涉及到了SQLiteCursor的一个设计要点，那就是SQLiteCursor的内存共享。getCount会调用SQLiteCursor的fillWindow，在以后的文章中我会在讲到SQLiteCursor，在此我们只要知道它是强制执行数据库query就可以了。

##第四步：return new CursorWrapperInner(......)

哈，看到了吧，我们在客户端调用query最终返回的是一个CursorWrapperInner类型，它是ContentResolver的一个内部类。实际上我们常用的getCount，onMove等一些列方法都是通过BulkCursorToCursorAdaptor和CursorToBulkCursorAdaptor的交互实现的。

到此，ContentProvider的访问流程就结束了，下面说一下开头买的坑:unstable和stable的ContentProvider。

在4.1之前，我们都可能会遇到过这样的场景，我们的应用程序访问了ContentProvider，但是这个ContentProvider意外挂了，这个时候我们的应用程序也将被连带杀死！这是Android处于对数据安全的考虑而做的决定，不过貌似Google也感觉这样的方式不太友好，所以在4.1以后提出了stable和unstable的概念。对于ContentResolver的query方法，我们将默认使用unstable的ContentProvider。看看下面的代码
```
Cursor qCursor;
            try {
                qCursor = unstableProvider.query(uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            } catch (DeadObjectException e) {
                // The remote process has died...  but we only hold an unstable
                // reference though, so we might recover!!!  Let's try!!!!
                // This is exciting!!1!!1!!!!1
                unstableProviderDied(unstableProvider);
                stableProvider = acquireProvider(uri);
                if (stableProvider == null) {
                    return null;
                }
                qCursor = stableProvider.query(uri, projection,
                        selection, selectionArgs, sortOrder, remoteCancellationSignal);
            }
```

上面是ContentResolver query中的一部分，可以看出unstable ContentProvider在query过程中如果发生了DeadObjectExeption则会被捕获，进而重新获取一个stable的ContentProvider。其实深入分析stable和unstable的ContentProvider还会有很多内容，以后有时间再说。 