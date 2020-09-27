---
layout:     post
title:      "AIDL中in/out/inout的差别"
date:       2020-09-25
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android
---



ManagerService.java

```java
public class ManagerService extends Service {
    private IManager.Stub mManager;
    private List<Book> mBooks = new ArrayList<>();

    @Override
    public void onCreate() {
        super.onCreate();
        mManager = new ManagerServiceImpl();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mManager;
    }

    private class ManagerServiceImpl extends IManager.Stub {
        private static final String TAG = "Server";

        @Override
        public void addBookIn(Book book) {
            book.price = 1024;
            mBooks.add(book);
            Log.i(TAG, "addBookIn: " + mBooks.toString());
        }

        @Override
        public void addBookOut(Book book) {
            book.price = 1024;
            mBooks.add(book);
            Log.i(TAG, "addBookOut: " + mBooks.toString());
        }

        @Override
        public void addBookInout(Book book) {
            book.price = 1024;
            mBooks.add(book);
            Log.i(TAG, "addBookInout: " + mBooks.toString());
        }
    }
}
```



Book.java

```java
public class Book implements Parcelable {
    public String name;
    public int price;

    public Book() {
    }

    public Book(String name, int price) {
        this.name = name;
        this.price = price;
    }

    protected Book(Parcel in) {
        name = in.readString();
        price = in.readInt();
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(price);
    }

    // 需要手动补充此方法
    public void readFromParcel(Parcel parcel) {
        name = parcel.readString();
        price = parcel.readInt();
    }

    @Override
    public int describeContents() {
        return 0;
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public String toString() {
        return "Book{" +
                "name='" + name + '\'' +
                ", price=" + price +
                '}';
    }
}
```



```java
public class MainActivity extends AppCompatActivity implements View.OnClickListener {
    private IManager manager;
    private static final String TAG = "Client";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        findViewById(R.id.buttonAddBookIn).setOnClickListener(this);
        findViewById(R.id.buttonAddBookOut).setOnClickListener(this);
        findViewById(R.id.buttonAddBookInout).setOnClickListener(this);

        Intent i = new Intent(this, ManagerService.class);
        bindService(i, new Connection(), BIND_AUTO_CREATE);
    }

    @Override
    public void onClick(View view) {
        if (manager == null) return;

        try {
            Book book = new Book("Handbook", 128);
            switch (view.getId()) {
                case R.id.buttonAddBookIn: {
                    Log.i(TAG, "addBookIn start: " + book.toString());
                    manager.addBookIn(book);
                    Log.i(TAG, "addBookIn end: " + book.toString());
                    break;
                }

                case R.id.buttonAddBookOut: {
                    Log.i(TAG, "addBookOut start: " + book.toString());
                    manager.addBookOut(book);
                    Log.i(TAG, "addBookOut end: " + book.toString());
                    break;
                }

                case R.id.buttonAddBookInout: {
                    Log.i(TAG, "addBookInout start: " + book.toString());
                    manager.addBookInout(book);
                    Log.i(TAG, "addBookInout end: " + book.toString());

                    break;
                }
            }
        } catch (RemoteException ignore) {
        }
    }

    private class Connection implements ServiceConnection {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            manager = IManager.Stub.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            manager = null;
        }
    }
}
```



Book.aidl

```java
package com.phantomvk.playground.aidl;
parcelable Book;
```



IManager.aidl

```java
package com.phantomvk.playground.aidl;
import com.phantomvk.playground.aidl.Book;

interface IManager {
    void addBookIn(in Book book);
    void addBookOut(out Book book);
    void addBookInout(inout Book book);
}
```



AndroidManifest.xml

```xml
<service
    android:name=".ManagerService"
    android:process=":remote" />
```



```
Client: addBookIn start: Book{name='Handbook', price=128}
Server: addBookIn:      [Book{name='Handbook', price=1024}]
Client: addBookIn end:   Book{name='Handbook', price=128}
```



```
Client: addBookOut start: Book{name='Handbook', price=128}
Server: addBookOut:      [Book{name='null', price=1024}]
Client: addBookOut end:   Book{name='null', price=1024}
```



```
Client: addBookInout start: Book{name='Handbook', price=128}
Server: addBookInout:      [Book{name='Handbook', price=1024}]
Client: addBookInout end:   Book{name='Handbook', price=1024}
```



> private static class Proxy implements com.phantomvk.playground.aidl.IManager

```java
@Override public void addBookIn(com.phantomvk.playground.aidl.Book book) throws android.os.RemoteException
{
  android.os.Parcel _data = android.os.Parcel.obtain();
  android.os.Parcel _reply = android.os.Parcel.obtain();
  try {
    _data.writeInterfaceToken(DESCRIPTOR);
    if ((book!=null)) {
      _data.writeInt(1);
      book.writeToParcel(_data, 0);
    }
    else {
      _data.writeInt(0);
    }
    mRemote.transact(Stub.TRANSACTION_addBookIn, _data, _reply, 0);
    _reply.readException();
  }
  finally {
    _reply.recycle();
    _data.recycle();
  }
}
```

```java
@Override public void addBookOut(com.phantomvk.playground.aidl.Book book) throws android.os.RemoteException
{
  android.os.Parcel _data = android.os.Parcel.obtain();
  android.os.Parcel _reply = android.os.Parcel.obtain();
  try {
    _data.writeInterfaceToken(DESCRIPTOR);
    mRemote.transact(Stub.TRANSACTION_addBookOut, _data, _reply, 0);
    _reply.readException();
  }
  finally {
    _reply.recycle();
    _data.recycle();
  }
}
```
```java
@Override public void addBookInout(com.phantomvk.playground.aidl.Book book) throws android.os.RemoteException
{
  android.os.Parcel _data = android.os.Parcel.obtain();
  android.os.Parcel _reply = android.os.Parcel.obtain();
  try {
    _data.writeInterfaceToken(DESCRIPTOR);
    if ((book!=null)) {
      _data.writeInt(1);
      book.writeToParcel(_data, 0);
    }
    else {
      _data.writeInt(0);
    }
    mRemote.transact(Stub.TRANSACTION_addBookInout, _data, _reply, 0);
    _reply.readException();
  }
  finally {
    _reply.recycle();
    _data.recycle();
  }
}
```
