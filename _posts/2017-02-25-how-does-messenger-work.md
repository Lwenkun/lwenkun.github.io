---
layout:     post
title:      "Messenger 的工作原理"
subtitle:   "How does Messenger work"
date:       2017-02-25
author:     "lwenkun"
header-img: "img/post-bg-how-does-messenger-work.png"
tags:
- android
- barriers
---
# Messenger 的工作原理 #

我们知道，`Handler` 是安卓用来进行线程间通信的工具，但是如果涉及到进程间通信呢，可以用 Handler 吗？答案是否定的。因为 `Handler` 实现的线程共享是基于统一进程的线程共享同一进程空间这一事实的。而不同进程拥有不同的进程空间，所以 `Hander` 机制不能于进程间通信。别太沮丧，安卓为我们提供了一个 Handler 的变体让我们可以通过和 `Handler` 类似的方式实现进程间通信，它就是 `Messenger`。

## IMessenger ##
我们知道进程间通信如果是通过 `Binder` 实现的，`Messenger` 也不例外。既然如此，那么一定会有 `Binder` 接口作为服务端和客户端的通信契约（类似于 AIDL 接口），在这里这个接口就是 `IMessenger`。它的定义如下：

```java
public interface IMessenger {
    void send(Message msg);
}
```

对于这个接口，需要有服务端的实现和客户端的实现。其中服务端的实现就“藏”在 `Handler` 中，它是 `Handler` 的内部类：

```java
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

关于 `Stub` 类是什么，在[安卓进程通信机制之 AIDL](http://liwenkun.me/2016/10/28/android-IPC-AIDL/)已经解释过了，它是一个继承自 `Binder` 的抽象类，内部封装了 `Binder` 通信机制，同时它还实现了 `IMessenger` 接口，但是它自己并没有真正意义的实现，而是留给服务端去实现，这里也就是 `MessengerImpl`。`MessengerImpl` 实现这个接口的方式就是简单地把发送来的消息转发给外部类 `Handler`，由 `Handler` 投递给消息队列。

为什么要把 `IMessenger` 的实现藏在 `Handler` 中，我在后面会将提到。

## Messenger ##

既然我们说 `Messenger` 是 `Handler` 的变体，那么我们自然的会认为 `Messenger` 应该具有这样的功能，或者说使用方法：在客户端进程中投送一个消息给服务端，服务端通过解析这个消息的内容执行相应的动作。那么，`Messenger` 是怎样做到的呢？它和 `IMessenger` 和 `MessengerImpl` 又有什么关联？首先我们从它的构造方法看起：

```java
public Messenger(Handler target) {
    mTarget = target.getIMessenger();
}

public Messenger(IBinder target) {
    mTarget = IMessenger.Stub.asInterface(target);
}
```
其中 `mTarget` 是一个 `IMessenger` 对象；而 `target.getIMessenger()` 就是前面的 `MessengerImpl` 对象。从这里我们很容易发现通过第一个构造方法构造的 `Messenger` 其 `mTarget` 是 `Binder` 接口在服务端的实现；而第二个构造方法构造出的 `Messenger` 其 `mTarget` 是 `Binder` 接口在客户端的实现（不懂的话可以看看我的[这篇文章](http://liwenkun.xyz/2016/10/28/android-IPC-AIDL/))。

现在我们对 `Messenger` 有了一个具体的认知：为了给开发者提供最简单的接口，系统又在 `IMessenger` 的基础之上用 `Messenger` 又封装了一层，将 `Binder` “隐藏”起来。我们通过 `Messenger` 的使用示例来说明：

首先定义服务端，我们在 Service 中实现：

```java
public MessengerServer extends Service {
     private static class MessengerHandler extends Handler {
         @Override
         handlerMessage(Message msg) {
             switch(msg.what) {
                 case TO_MESSENGER_SERVER :
                     Log.d("MessengerServer", );
                     break;
             }
         }    
     }
     
     @Override
     public IBinder onBind(Intent intent) {
         return new Messenger(new MessengerHandler).getBinder();
     }
}
```
然后是客户端的实现，我们放在 `Activity` 中：

```java
public class MessengerClient extends Activity {
    private finanl Messenger server;
    
    private final ServiceConnection conn = new ServiceConnection() {
        @Override 
        public void onServiceConnected(ComponentName className, IBinder service) {
            server = new Messenger(service);
            Message msg = Message.obtain(null, TO_MESSENGER_SERVER);
            Bundle data = new Bundle();
            data.putString("This msg is from client");
            try {
                mServer.send(msg);//发送消息给客户端，进行了一次远程调用
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        
        @Override
        public void onServieDisconnected(ComponentName className) {}
    }
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ......
        Intent i = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }
}
```
这样就通过 `Messenger` 实现了进程间的通信，这和我们用 `Handler` 实现线程间通信非常相似。`Messenger` 在服务端和客户端都有使用，但是使用的构造方法不一样，这也正和前面的分析一致。用一张图来说明 `Messenger` 的工作原理：

![](/img/in-post/post_how_does_messenger_work/how_does_messenger_work.png)

对于这张图说明如下：

- 实现了客户端消息到服务端的传递。
- 整个过程的实质是两端的 mTarget 通过 Binder 通信，其他对象只是起辅助作用。