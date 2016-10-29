---
layout:     post
title:      "AIDL"
subtitle:   "安卓进程通信机制之 AIDL"
date:       2016-10-28
author:     "lwenkun"
header-img: "img/post-bg-android-ipc-aidl.png"
tags:
    - android
    - IPC
    - AIDL
---


# 安卓进程通信机制之 AIDL #

## 什么是 AIDL
AIDL 全称 *Android Interface Definition Language*，即 **安卓接口描述语言**。听起来很深奥，其实它的本质就是生成进程间通信接口的辅助工具。它的存在形式是一种 `.aidl` 文件，开发者需要做的就是在该文件中定义进程间通信的接口，编译的时候 IDE 就会根据我们的 `.aidl` 接口文件生成可供项目使用的 `.java` 文件，这和我们说的“语法糖”有些类似。

AIDL 的语法就是 java 的语法，就是导包上有点细微差别。java 中如果两个类在相同的包中，是不需要进行导包操作的，但是在 AIDL 中，则必须进行导包声明。

## AIDL 详解
构想一个场景：我们有一个图书管理系统，这个系统的通过 CS 模式来实现。具体的管理功能由服务端进程来实现，客户端只需要调用相应的接口就可以。

那么首先定义这个管理系统的 ADIL 接口。

我们在 ／src 新建 `aidl` 包，包中有三个文件 Book.java 、Book.aidl、IBookManager.aidl 三个文件。
 
```java
package com.example.aidl book

public class Book implements Parcelable {
  int bookId;
  String bookName;
  
  public Book(int bookId, String bookName) {
     this.bookId = bookId;
     this.bookName = bookName;
  }
  
  ...
}
```

```java
package com.example.aidl;

Parcelable Book;
```

```java
package com.example.aidl;

import com.example.aidl.Book;

inteface IBookManager {
   List<Book> getBookList();
   void addBook(in Book book);
}
```

下面对这三个文件分别进行说明：

- `Book.java` 是我们定义的实体类，它实现了 `Parcelable` 接口，这样 `Book` 类才能在进程间传输。
- `Book.aidl` 是这个实体类在 AIDL 中的声明。
- `IBookManager` 是服务端和客户端通信的接口。（注意，在 AIDL 接口中除基本类型外，参数前须加方向，`in` 表示输入型参数，`out` 表示输出型参数，`inout` 表示输入输出型参数）

编译器编译后，android studio 为我们的项目自动生成了一个 `.java` 文件，这个文件包含三个类，这三个类分别是 `IBookManager`, `Stub` 和 `Proxy`，这三个类都是静态类型，我们完全可以把他们分开来，三个类定义如下：

- `IBookManager`

```java
public interface IBookManager extends android.os.IInterface {

    public void addBook(net.bingyan.library.Book book) throws android.os.RemoteException;

    public java.util.List<net.bingyan.library.Book> getBookList() throws android.os.RemoteException;
}
```

- `Stub`

```java
public static abstract class Stub extends android.os.Binder implements net.bingyan.library.IBookManager {
        private static final java.lang.String DESCRIPTOR = "net.bingyan.library.IBookManager";

        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an net.bingyan.library.IBookManager interface,
         * generating a proxy if needed.
         */
        public static net.bingyan.library.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof net.bingyan.library.IBookManager))) {
                return ((net.bingyan.library.IBookManager) iin);
            }
            return new net.bingyan.library.IBookManager.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    net.bingyan.library.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = net.bingyan.library.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<net.bingyan.library.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
}
```

- `Proxy`

```java
private static class Proxy implements net.bingyan.library.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public void addBook(net.bingyan.library.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public java.util.List<net.bingyan.library.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<net.bingyan.library.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(net.bingyan.library.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
       }
```

对生成的这三个类的说明如下：

- `IBookManager` 这个类是我们定义的接口，android studio 给它添加了一个父类，让它继承自 android.os.interface 这个接口，这个接口只有一个方法 `IBinder asBinder()`，这样 `IBookManager` 中就有三个带实现的方法了，它是服务端进程和客户端进程通信的窗口。
- `Stub` 是个抽象类，这个类继承自 `android.os.Binder` 类，并且实现的了 `IBookManager` 这个接口。在 `Stub` 中，已经实现了 `asBinder()` 这个接口方法，还有两个是我们定义的 AIDL 接口方法留给继承它的子类去实现。它用在服务端，因此服务端需要实现这两个方法。
- `Proxy` 顾名思义是一个代理类，它是服务端在客户端的一个代理，它也实现了 `IBookManager` 接口，并且实现了 `IBookManager` 中的所有方法。它用在客户端，是服务端在客户端的代理。

现在我们对这三个类逐个分析：

- `IBookManager` 这个类没什么好说的，它只是简单继承了 `asInterface` 这个接口，作用就是将 `IBookManager` 转换成 `IBinder`。

- `Proxy` 这个类上面已经提到过了，它就是进程间通信机制的一个封装类，他的内部实现机制就是 `Binder`，通过构造方法我们也容易看出来。它的构造方法接受一个 `IBinder` 类型的参数，参数名为 `remote`，显然，它代表着服务端。我们看看这个类中的方法 `addBook()` 和 `getBookList()`：


```java
@Override
public void addBook(net.bingyan.library.Book book) throws android.os.RemoteException {
      android.os.Parcel _data = android.os.Parcel.obtain();
      android.os.Parcel _reply = android.os.Parcel.obtain();
      try {
            _data.writeInterfaceToken(DESCRIPTOR)
            if ((book != null)) {
                _data.writeInt(1);
                book.writeToParcel(_data, 0);
            } else {
                _data.writeInt(0);
            }
            mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
            _reply.readException();
       } finally {
            _reply.recycle();
            _data.recycle();
       }
}
```
```java
@Override
public java.util.List<net.bingyan.library.Book> getBookList() throws android.os.RemoteException {
       android.os.Parcel _data = android.os.Parcel.obtain();
       android.os.Parcel _reply = android.os.Parcel.obtain();
       java.util.List<net.bingyan.library.Book> _result;
       try {
             _data.writeInterfaceToken(DESCRIPTOR);
             mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
             _reply.readException();
             _result = _reply.createTypedArrayList(net.bingyan.library.Book.CREATOR);
       } finally {
            _reply.recycle();
            _data.recycle();
       }
       return _result;
}
```

  它们是编译器自动实现的，这两个方法有很多类似之处，可以现在这里透露下：这两个方法就是客户端进程调用服务端进程的窗口。在这两个方法的开始，它们都定义了两个 `Parcel`（中文译名：包裹）对象。`Parcel` 这个类我们看上去很眼熟，是的，`Book` 类中的 `writeToParcel()` 和 `CREATOR` 中的 `createFromParcel()` 的参数就是 `Parcel` 类型的，关于这个类文档中解释如下：

 >Container for a message (data and object references) that can
  be sent through an IBinder.  A Parcel can contain both flattened data
  that will be unflattened on the other side of the IPC (using the various
  methods here for writing specific types, or the general
  {@link Parcelable} interface), and references to live {@link IBinder}
  objects that will result in the other side receiving a proxy IBinder
  connected with the original IBinder in the Parcel.
  
 翻译一下：`Proxy` 是一个可以通过 `IBinder` 进行消息传递的一个容器。一个 `Parcel` 可以包含可序列化的数据，这些数据会在 `IPC` 的另一端被反序列化；它也可以包含指向 `IBinder` 对象的引用，这会使得另一端接收到一个 `IBinder` 类型的代理对象，这个代理对象连接着 `Parcel` 中的原始 `IBinder` 对象。
 
 下面用图来直观的说明：
  ![](/img/in-post/post-android-IPC-AIDL/IPC_1.png)
  
 如图，我们可以很直观的看到服务端以 `Parcel` 作为数据包裹依靠 `Binder` 和客户端进行通信。数据包裹就是序列化之后的对象。
 
 如上所述，这两个方法都定义了两个 `Parcel` 对象，分别叫做 `_data` 和 `_reply`，形象的来说，从客户端的角度来看，`_data` 就是客户端发送给服务端的数据包裹，`_reply` 服务端发送给客户端的数据包裹。

 之后便开始用这两个对象来和服务端进行通信了，我们能够观察到，两个方法中都有这么个方法调用 `mRemote.transact()`，它有四个参数，第一个参数的意义我们后面再讲，第二个参数 `_data` 负责向服务端发送数据包裹比如接口方法的参数，第三个参数 `_reply` 负责从服务端接收数据包裹比如接口方法的返回值。这行代码只有一句简单的方法调用，但是却是 AIDL 通信的最核心部分，它其实进行了一次远程方法调用（客户端通过本地代理 `Proxy` 暴露的接口方法调用服务端 `Stub` 同名方法），所以能想到它是一个耗时操作。

 在我们的例子中：

 - `void addBook(Book book)` 需要借助 `_data` 向服务端发送参数 `Book:book`，发送的方式就是把 `Book` 通过其实现的 `writeToParcel(Parcel out)` 方法打包至 `_data` 中，正如你能想到的，`_data` 其实就是参数 `out`，还记得 `Book` 中的这个方法的实现吗？ 我们是将 `Book` 的字段一个个打包至 `Parcel` 中的。

 - `List<Book> getBookList()` 需要借助 `_reply` 从服务端接收返回值 `List<Book>:books`，方法中的做法是将 `Book` 中的 `CREATOR` 这个静态字段作为参数传入 `_reply` 的 `createTypedArrayList()` 方法中，还记得 `Book` 中的 `CREATOR` 吗？当时你是不是好奇这个静态字段应该怎么用呢？现在一切明了了，我们需要靠这个对象（便于理解我们可以叫它”反序列化器“）来对服务端的数据反序列化从而重新生成可序列化的对象或者对象数组。很明显 `CREATOR` 借助 `_reply` 生成了 `List<Book>:books`。

 当然这两个方法中的 `_data` 和 `_reply` 不仅传递了对象，还传递了一些校验信息，这个我们可以不必深究，但应注意的是，`Parcel` 打包顺序和解包顺序要严格对应。例如，第一个打包的是 `int:i`，那么第一解包的也应该是这个整型值。也即打包时第一次调用的如果是 `Parcel.writeInt(int)`，解包时第一次调用的应该是 `Parcel.readInt()`。
 
 到此，客户端的 `Proxy` 讲解完了，下面我们看看服务端的 Stub。

- `Stub` 中实现了 `IBookManager` 的其中一个方法，这个很简单，就是简单的将自身返回，因为 `Stub` 本身就继承自 `Binder`，而 `Binder` 继承自 `IBinder`，所以没有任何问题。你会问：还有两个方法没实现呢？这两个方法就是我们定义的接口方法，它们留给服务端进程去实现，也就是说，到时候我们在服务端进程中需要定义一个 `Stub` 的实现者。下面对 `Stub` 中的两个重要方法进行分析：

  - `IBookManager asInterface(IBinder obj)`
  
```java
public static net.bingyan.library.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof net.bingyan.library.IBookManager))) {
                return ((net.bingyan.library.IBookManager) iin);
            }
            return new net.bingyan.library.IBookManager.Stub.Proxy(obj);
        }
```
  这个方法的作用是将 `Stub` 类转换成 `IBookManager` 这个接口，方法中有个判断：如果我们的服务端进程和客户端进程是同一进程，那么就直接将 `Stub` 类通过类型转换转成 `IBookManager`；如果不是同一进程，那么就通过代理类 `Proxy` 将 `Stub` 转换成 `IBookManager`。为什么这么做，我们知道如果服务端进程和客户端进程不是同一进程，那么它们的内存就不能共享，就不能通过一般的方式进行通信，但是我们如果自己去实现进程间通信方式，对于普通开发者来说成本太大，因此编译器帮我们生成了一个封装了了进程间通信的工具，也就是这个 `Proxy`，这个类对底层的进程通信机制进行了封装只同时暴露出接口方法，客户端只需要调用这两个方法实现进程间通信（其实就是方法的远程调用）而不需要了解其中的细节。
   
   有了这个方法，我们在客户端可以借助其将一个 `IBinder` 类型的变量转换成我们定义的接口 `IBookManager`，它的使用场景我们会在后面的实例中进行讲解。
     
   - `onTransact(int code, Parcel data, Parcel reply, int flags)`
  
```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
           switch (code) {
               case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
               }
               case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    net.bingyan.library.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = net.bingyan.library.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
               }
               case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<net.bingyan.library.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
           }
           return super.onTransact(code, data, reply, flags);
}
```
   这个方法我们是不是也很熟悉呢？我们在 `Proxy` 中也看到一个类似得方法 `transact(int, Parcel, Parcel, int)`，它们的参数一样，而且它们都是 `Binder` 中的方法，那么它们有什么联系呢？

   前面说了，`transact()` 执行了一个远程调用，如果说 `transact()` 是远程调用的发起，那么 `onTransact()` 就是远程调用的响应。真实过程是客户端发器远程方法调用，android 系统通过底层代码对这个调用进行响应和处理，之后回调服务端的 `onTransact()` 方法，从数据包裹中取出方法参数，交给服务端实现的同名方法调用，最后将返回值打包返回给客户端。
      
   需要注意的是 `onTransact()` 是在服务端进程的 `Binder` 线程池中进行的，这就意味着如果我们的要在 `onTransact()` 方法的中更新 UI，就必须借助 `Handler`。
      
   这两个方法的第一个参数的含义是 AIDL 接口方法的标识码，在 `Stub` 中，定义了两个常量作为这两个方法的标示：
      
```java
   static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
   static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
```
   如果 `code == TRANSACTION_addBook`，那么说明客户端调用的是 `addBook()`；如果 `code == TRANSACTION_getBookList`，那么客户端调用的是 `getBookList()`，然后交由相应的服务端方法处理。
用一张图来表示整个通信过程：
![](/img/in-post/post-android-IPC-AIDL/IPC_2.png)

   了解了 AIDL 的整个过程，接下来就是 AIDL 在安卓程序中的应用了。

## AIDL 的使用 ##

相信大家应该都和清楚 `Service` 的使用了吧，`Service` 虽然称作“服务”，并且运行于后台，但是它们默认还是运行在默认进程的主线程中。其实让 `Service` 运行在默认进程中，有点大材小用了。android 的很多系统服务都运行于单独的进程中，供其他应用调用，比如窗口管理服务。这样做的好处是可以多个应用共享同一个服务，节约了资源，也便于集中管理各个客户端，要注意问题的就是线程安全问题。

那么接下来我们就用 AIDL 实现一个简单的 CS 架构的图书管理系统。

首先我们定义服务端：

- `BookManagerService`

```java
public class BookManagerService extends Service {

    private final List<Book> mLibrary = new ArrayList<>();


    private IBookManager mBookManager = new IBookManager.Stub() {
        @Override
        public void addBook(Book book) throws RemoteException {
            synchronized (mLibrary) {
                mLibrary.add(book);
                Log.d("BookManagerService", "now our library has " + mLibrary.size() + " books");
            }

        }

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mLibrary;
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return mBookManager.asBinder();
    }

}
```
```xml
<service
      android:process=":remote"
      android:name=".BookManagerService"/>
```
服务端我们定义了 `BookManagerService` 这个类，在它里面我们创建了服务端的 `Stub` 对象，并且实现了需要实现的两个 AIDL 接口方法来定义服务端的图书管理策略。在 `onBind()` 方法中我们将 `IBookManager` 对象作为 `IBinder` 返回。我们知道，当我们绑定一个服务时，系统会调用 `onBinder()` 方法得到服务端的 `IBinder` 对象，并将其转换成客户端的 `IBinder` 对象传给客户端，虽然服务端的 `IBinder` 和 客户端的 `IBinder` 是两个 `IBinder` 对象，但他们在底层都是同一个对象。我们在 xml 中注册 `Service` 时给它指定了进程名，这样 `Service` 就能运行在单独的进程中了。

接下来看看客户端的实现：

- `Client`

```java
public class Client extends AppCompatActivity {

    private TextView textView;

    private IBookManager bookManager;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.library_book_manager_system_client);

        Intent i  = new Intent(Client.this, BookManagerService.class);
        bindService(i, conn, BIND_AUTO_CREATE);

        Button addABook = (Button) findViewById(R.id.button);
        addABook.setOnClickListener(v -> {
            if (bookManager == null) return;
            try {
                bookManager.addBook(new Book(0, "book"));
                textView.setText(getString(R.string.book_management_system_book_count, String.valueOf(bookManager.getBookList().size())));
            } catch (RemoteException e) {
                e.printStackTrace();
            }

        });

        textView = (TextView) findViewById(R.id.textView);
    }

    private ServiceConnection conn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d("Client -->", service.toString());

            bookManager = IBookManager.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.d("Client", name.toString());
        }
    };

}
```
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical" android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:weightSum="1"
    android:gravity="center">

    <Button
        android:text="add a book"
        android:layout_width="111dp"
        android:layout_height="wrap_content"
        android:id="@+id/button" />

    <TextView
        android:layout_marginTop="10dp"
        android:text="@string/book_management_system_book_count"
        android:layout_width="231dp"
        android:gravity="center"
        android:layout_height="wrap_content"
        android:id="@+id/textView" />
</LinearLayout>
```

我们的客户端就是一个 `Activity`，`onCreate()` 中进行了服务的绑定，`bindService()` 方法中有一参数 `ServiceConnection:conn`，因为绑定服务是异步进行的，这个参数的作用就是绑定服务成功后回调的接口，它有两个回调方法：一个是连接服务成功后回调，另一个在与服务端断开连接后回调。我们现在关心的主要是 `onServiceConnected()` 方法，在这里我们只做了一件事：将服务端转换过来的 `IBinder` 对象转换成 AIDL 接口，我们定义 `IBookManager:bookManager` 字段来保持对其的引用。这样的话，我们就可以通过这个 `bookManager` 来进行方法的远程调用。我们给客户端的 `Button` 注册事件：每一次点击都会向服务端增加一本书，并且将图书馆现有的图书数量显示出来。

现在我们看看程序的运行效果：

<center> <img src="/img/in-post/post-android-IPC-AIDL/AIDL.gif"/> </center>

每当我们点击按钮，我们就成功的向服务端添加了一本书，说明我们通过 AIDL 跨进程通信成功了。


