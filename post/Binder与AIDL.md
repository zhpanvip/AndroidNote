## 一、Binder简单使用

使用Binder实现一个客户端连接服务端查询学生成绩的例子。

### 1.服务端提供查询服务

首先，服务端提供查询成绩的服务。GradeService继承Service。

```java
public class GradeService extends Service {
    public static final int REQUEST_CODE=1000;
    private final Binder mBinder = new Binder() {
        @Override
        protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
            if (code == REQUEST_CODE) {
                String name = data.readString();
                // 根据姓名查询学生成绩并将成绩写入到返回数据
                int studentGrade = getStudentGrade(name);
                if (reply != null)
                    reply.writeInt(studentGrade);
                return true;
            }
            return super.onTransact(code, data, reply, flags);
        }
        // 根据姓名查询学生成绩
        public int getStudentGrade(String name) {         
            return StudentMap.getStudentGrade(name);
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}

```

将GradeService设置为单独进程

```xml
<service
    android:name="com.zhpan.sample.binder.server.GradeService"
    android:process=":server">
    <intent-filter>
        <action android:name="android.intent.action.server.gradeservice" />
    </intent-filter>
</service>
```

## 2.客户端连接服务查询成绩

在客户端Activity中绑定GradeService

```Java
public class BinderActivity extends AppCompatActivity {
    // 远程服务的Binder代理
    private IBinder mRemoteBinder;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            // 获取远程服务的Binder代理
            mRemoteBinder = iBinder;
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mRemoteBinder = null;
        }
    };



    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_binder);
        // 绑定服务
        findViewById(R.id.btn_bind_service).setOnClickListener(view -> bindGradeService());
        // 查询学生成绩
        findViewById(R.id.btn_find_grade).setOnClickListener(view -> getStudentGrade("Anna"));
    }
    // 绑定远程服务
    private void bindGradeService() {
        String action = "android.intent.action.server.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }
    // 从远程服务查询学生成绩
    private int getStudentGrade(String name) {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        int grade = 0;
        data.writeString(name);
        try {
            if (mRemoteBinder == null) {
                throw new IllegalStateException("Need Bind Remote Server...");
            }
            mRemoteBinder.transact(REQUEST_CODE, data, reply, 0);
            grade = reply.readInt();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return grade;
    }
```

## 二、代理模式优化Binder

### 1.定义一个查询成绩接口



```java
public interface IGradeInterface {
    int getStudentGrade(String name);
}
```



### 2.被代理的Binder类

自定义GradeBinder继承Binder，并实现IGradeInterface接口



```java
public class GradeBinder extends Binder implements IGradeInterface {

    @Override
    public int getStudentGrade(String name) {
        return StudentMap.getStudentGrade(name);
    }

    @Override
    protected boolean onTransact(int code, @NonNull Parcel data, @Nullable Parcel reply, int flags) throws RemoteException {
        if (code == REQUEST_CODE) {
            String name = data.readString();
            int studentGrade = getStudentGrade(name);
            if (reply != null)
                reply.writeInt(studentGrade);
            return true;
        }
        return super.onTransact(code, data, reply, flags);
    }
}
```



### 3.Binder的代理类



```java
public class BinderProxy implements IGradeInterface {
    // 被代理的Binder
    private final IBinder mBinder;

    private BinderProxy(IBinder binder) {
        mBinder = binder;
    }

  	// 通过Binder查询成绩
    @Override
    public int getStudentGrade(String name) {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        int grade = 0;
        data.writeString(name);
        try {
            if (mBinder == null) {
                throw new IllegalStateException("Need Bind Remote Server...");
            }
            mBinder.transact(REQUEST_CODE, data, reply, 0);
            grade = reply.readInt();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        return grade;
    }
		
    // 实例化Binder代理类的对象
    public static IGradeInterface asInterface(IBinder iBinder) {
        if (iBinder == null) {
            return null;
        } 
        
        if (iBinder instanceof IGradeInterface) {
            LogUtils.e("当前进程");
            // 如果是同一个进程的请求，则直接返回Binder
            return (IGradeInterface) iBinder;
        } else {
            LogUtils.e("远程进程");
            // 如果是跨进程查询则返回Binder的代理对象
            return new BinderProxy(iBinder);
        }
    }

}
```

### 4.客户端发起请求



```java
public class BinderProxyActivity extends BaseViewBindingActivity<ActivityBinderBinding> {
		// 此处可能是BinderProxy也可能是GradeBinder
    private IGradeInterface mBinderProxy;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
          	// 连接服务成功，根据是否跨进程获取BinderProxy或者GradeBinder实例
            mBinderProxy = BinderProxy.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mBinderProxy = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding.btnBindService.setOnClickListener(view -> bindGradeService());
      	// 查询学生成绩点击事件，通过mBinderProxy查询成绩
        binding.btnFindGrade.setOnClickListener(view -> ToastUtils.showShort("Anna grade is " + mBinderProxy.getStudentGrade("Anna")));
    }

		// 绑定服务
    private void bindGradeService() {
        String action = "android.intent.action.server.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }

}
```


## 三、AIDL

AIDL是Android Interface Description Languaged 简写。用于描写客户端/服务端通信接口的一种描述语言。其实说人话就是通过定义aidl接口来生成第二章中的代理类。即AIDL的原理其实跟上一章的代理模式优化Binder的使用是一样的。目的就是为了省去我们自己编写代理的代码。



### 1.定义AIDL接口

```java
package com.zhpan.sample.binder.aidl;

// Declare any non-default types here with import statements

interface IGradeService {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
		// 查询成绩的接口
    int getStudentGrade(String name);
}
```

### 2.Rebuild项目后生成代理代码

```java
// 这个接口相当于上一章中的IGradeInterface接口
public interface IGradeService extends android.os.IInterface {
  
  ...
  
  // Stub是一个Binder，相当于上一章中的GradeBinder
  public static abstract class Stub extends android.os.Binder
      implements com.zhpan.sample.binder.aidl.IGradeService {
    private static final java.lang.String DESCRIPTOR = "com.zhpan.sample.binder.aidl.IGradeService";

    public Stub() {
      this.attachInterface(this, DESCRIPTOR);
    }

    public static IGradeService asInterface(android.os.IBinder obj) {
      
      if ((obj == null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin != null) && (iin instanceof com.zhpan.sample.binder.aidl.IGradeService))) {
        // 如果是当前进程则直接返回当前Binder对象
        return ((com.zhpan.sample.binder.aidl.IGradeService) iin);
      }
      // 跨进程则返回Binder的代理对象
      return new com.zhpan.sample.binder.aidl.IGradeService.Stub.Proxy(obj);
    }

    @Override public android.os.IBinder asBinder() {
      return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
        throws android.os.RemoteException {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code) {
        case INTERFACE_TRANSACTION: {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_basicTypes: {
          data.enforceInterface(descriptor);
          int _arg0;
          _arg0 = data.readInt();
          long _arg1;
          _arg1 = data.readLong();
          boolean _arg2;
          _arg2 = (0 != data.readInt());
          float _arg3;
          _arg3 = data.readFloat();
          double _arg4;
          _arg4 = data.readDouble();
          java.lang.String _arg5;
          _arg5 = data.readString();
          this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
          reply.writeNoException();
          return true;
        }
        case TRANSACTION_getStudentGrade: {
          data.enforceInterface(descriptor);
          java.lang.String _arg0;
          _arg0 = data.readString();
          int _result = this.getStudentGrade(_arg0);
          reply.writeNoException();
          reply.writeInt(_result);
          return true;
        }
        default: {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }
    // Binder的代理类，相当于上一章中的BinderProxy
    private static class Proxy implements com.zhpan.sample.binder.aidl.IGradeService {
      private android.os.IBinder mRemote;

      Proxy(android.os.IBinder remote) {
        mRemote = remote;
      }

      @Override public android.os.IBinder asBinder() {
        return mRemote;
      }

      public java.lang.String getInterfaceDescriptor() {
        return DESCRIPTOR;
      }

      @Override public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
          double aDouble, java.lang.String aString) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeInt(anInt);
          _data.writeLong(aLong);
          _data.writeInt(((aBoolean) ? (1) : (0)));
          _data.writeFloat(aFloat);
          _data.writeDouble(aDouble);
          _data.writeString(aString);
          boolean _status = mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().basicTypes(anInt, aLong, aBoolean, aFloat, aDouble, aString);
            return;
          }
          _reply.readException();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
      }

      @Override public int getStudentGrade(java.lang.String name)
          throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        int _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeString(name);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getStudentGrade, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getStudentGrade(name);
          }
          _reply.readException();
          _result = _reply.readInt();
        } finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }

      public static com.zhpan.sample.binder.aidl.IGradeService sDefaultImpl;
    }

    static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_getStudentGrade = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

    public static boolean setDefaultImpl(com.zhpan.sample.binder.aidl.IGradeService impl) {
      if (Stub.Proxy.sDefaultImpl == null && impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }

    public static com.zhpan.sample.binder.aidl.IGradeService getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }

  /**
   * Demonstrates some basic types that you can use as parameters
   * and return values in AIDL.
   */
  public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble,
      java.lang.String aString) throws android.os.RemoteException;

  public int getStudentGrade(java.lang.String name) throws android.os.RemoteException;
}


```

可以看到，通过AIDL生成的代码和上一章的代码基本原理是一致的，只不过生成的代码都被放在了同一个类中。



### 2.AIDL客户端

AIDL的客户端也很简单

```java
public class AidlActivity extends BaseViewBindingActivity<ActivityBinderBinding> {

    private IGradeService mBinderProxy;

    private final ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
          	// 连接服务后，根据是否跨进程获取Binder或者Binder的代理对象
            mBinderProxy = IGradeService.Stub.asInterface(iBinder);
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            mBinderProxy = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding.btnBindService.setOnClickListener(view -> bindGradeService());
      	// 查询学生成绩
        binding.btnFindGrade.setOnClickListener(view -> getStudentGrade("Anna"));
    }
  
  	// 绑定服务
    private void bindGradeService() {
        String action = "android.intent.action.server.aidl.gradeservice";
        Intent intent = new Intent(action);
        intent.setPackage(getPackageName());
        bindService(intent, mServiceConnection, BIND_AUTO_CREATE);
    }
  
		// 查询成绩
    private void getStudentGrade(String name) {
        int grade = 0;
        try {
            grade = mBinderProxy.getStudentGrade(name);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        ToastUtils.showShort("Anna grade is " + grade);
    }
}
```