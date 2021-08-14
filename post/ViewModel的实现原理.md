## 一、ViewModel的使用

在页面（Activity/Fragment）很简单的情况下，通常我们会将UI交互，数据获取与处理等相关业务逻辑，全部写在页面中，但是在页面复杂的情况下，这样做是不合适的，它不符合“**单一责任**”原则。页面只应该负责接收用户的交互，以及将数据展示到屏幕上，相关数据应该单独存放和处理。

为此，Android为我们提供了**ViewModel**类，专门用于存放应用程序页面所需的数据。它将页面所需的数据从页面中剥离出来，页面只需要处理用户交互，以及负责展示数据的工作。

![v2-48b149d7b2f75c79173ff6f7a32b62db_720w](https://user-images.githubusercontent.com/19853475/129429940-26022154-bb28-43a4-839b-849ca9f0bf2a.jpeg)

另外，如果我们的应用程序支持横竖屏切换，当用户旋转手机屏幕时，我们还需要考虑数据的存储与恢复。如果数据不进行存储，那么通常我们还需要重新去获取一次。

而ViewModel能为我们解决这个问题，它独立于配置变化。也就是说，屏幕旋转导致的Activity重建，并不会影响到ViewModel的生命周期。

![img](https://user-gold-cdn.xitu.io/2019/3/21/1699bd67cfefec24?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 1. 引入ViewModel

目前最新版本为2.3.1

> implementation "androidx.lifecycle:lifecycle-extensions:2.3.1"



### 2.一个简单例子

（1）定义一个User数据类

```java
class User implements Serializable {
    public int age;
    public String name;
}
```

（2）ViewModel结合LiveData

```java
public class UserModel extends ViewModel {

    public final MutableLiveData<User> mUserLiveData = new MutableLiveData<>();

    public UserModel() {
        //模拟从网络加载用户信息
        mUserLiveData.postValue(new User(1, "name1"));
    }

    //模拟 进行一些数据骚操作
    public void doSomething() {
        User user = mUserLiveData.getValue();
        if (user != null) {
            user.age = 15;
            user.name = "name15";
            mUserLiveData.setValue(user);
        }
    }

}
```

（3）这时候在Activity中就可以使用ViewModel了. 其实就是一句代码简单实例化,然后就可以使用ViewModel了.

```java
public class MainActivity extends FragmentActivity {

    private TextView mContentTv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mContentTv = findViewById(R.id.tv_content);

        //构建ViewModel实例
        final UserModel userModel = new ViewModelProvider(this).get(UserModel.class);

        //让TextView观察ViewModel中数据的变化,并实时展示
        userModel.mUserLiveData.observe(this, new Observer<User>() {
            @Override
            public void onChanged(User user) {
                mContentTv.setText(user.toString());
            }
        });

       findViewById(R.id.btn_test).setOnClickListener(v->{
            //点击按钮  更新User数据  观察TextView变化
            userModel.doSomething();
       });
    }
}
```

此时点击按钮后TextView的内容变为15，旋转屏幕后数据依然不变。可见在Activity销毁之后,它才会自动销毁(所以别让ViewModel持有Activity引用,会内存泄露的).

### 3.ViewModel与Fragment通信

Activity与Fragment可以共享一个ViewModel,因为Fragment是依附在Activity上的,在实例化ViewModel时将Activity传入ViewModelProvider,它会为该Activity已创建好ViewModel,这个Fragment可以方便的访问该ViewModel中的数据.在Activity中修改userModel数据后,该Fragment就能拿到更新后的数据.

```java
public class MyFragment extends Fragment {
     public void onStart() {
        //这里拿到的ViewModel实例,其实是和Activity中创建的是一个实例
         UserModel userModel = new ViewModelProvider(getActivity()).get(UserModel.class);
     }
 }
```

## 二、ViewModel的实现原理



###  1.ViewModelProvider的构造方法



```java
public ViewModelProvider(@NonNull ViewModelStoreOwner owner) {
    this(owner.getViewModelStore(), owner instanceof HasDefaultViewModelProviderFactory
            ? ((HasDefaultViewModelProviderFactory) owner).getDefaultViewModelProviderFactory()
            : NewInstanceFactory.getInstance());
}
```

ViewModelProvider构造方法接受一个ViewModelStoreOwner类型的参数。ViewModelStoreOwner是一个接口，内部只有一个getViewModelStore方法。接下来在构造方法中又调用了重载的构造方法，并且第一个调用了ViewModelStoreOwner的getViewModelStore方法来获取ViewModelStore，第二个参数通过ower判断类型来实例化ViewModelProviderFacory

在中ComponentActivity实现ViewModelStoreOwner接口和HasDefaultViewModelProviderFactory，ComponentActivity重写了getViewModelStore和getDefaultViewModelProviderFactory方法。

```java
public class ComponentActivity extends androidx.core.app.ComponentActivity implements
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory{
    @NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        // 确保ViewModelStore不为空
        ensureViewModelStore();
        return mViewModelStore;
    }

    @NonNull
    @Override
    // 创建了一个默认的ViewModelFactory
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mDefaultFactory == null) {
            mDefaultFactory = new SavedStateViewModelFactory(
                    getApplication(),
                    this,
                    getIntent() != null ? getIntent().getExtras() : null);
        }
        return mDefaultFactory;
    }

}
```

ensureViewModelStore方法中如果ViewModelStore为null，则会尝试恢复或者实例化ViewModelStore，代码如下：

```java
void ensureViewModelStore() {
    if (mViewModelStore == null) {
        // 获取上次配置数据实例
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // 从NonConfigurationInstances中恢复ViewModelStore
            mViewModelStore = nc.viewModelStore;
        }
        // NonConfigurationInstances中没有保存那么就实例化ViewModelStore
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
}
```

ViewModelProvider的构造方法最终调用了下边的这个重载的构造方法：

```java
public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}
```

从上述代码可以看到，在创建ViewModel的时候，Activity中会首先尝试恢复或者创建一个ViewModelStore。并将其保存到ViewModelProvider中待用。



### 2.ViewModelProvider的get方法

实例化了ViewModelProvider后会调用ViewModelProvider的get方法。代码如下：

```java
    public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
				// ... 省略校验
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) mFactory).create(key, modelClass);
        } else {
            viewModel = mFactory.create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
```

get方法中首先调用了mViewModelStore的get方法，并通过key来获取ViewModel。可见ViewModel本身应该是一个K-V的集合类。暂且不管ViewModelStore。接下来，如果modelClass已经实例化了，就直接返回ViewModel，否则就调用Factory来创建ViewModel，并将ViewModel存入ViewModelStore中。很显然，ViewModel的实例化时在Factory中进行的。ViewModelProvider的构造方法是允许我们自己传入一个工厂的，如果不传也会创建默认的工厂。本章的第一小节已经看到会默认实例化一个SavedStateViewModelFactory，这个工厂继承了KeyedFactory，我们来看下它的create是如何实现的：

```java
public <T extends ViewModel> T create(@NonNull String key, @NonNull Class<T> modelClass) {
    // 判断是不是AndroidViewModel类型
    boolean isAndroidViewModel = AndroidViewModel.class.isAssignableFrom(modelClass);
    Constructor<T> constructor;
    // 获取ViewModel的构造方法
    if (isAndroidViewModel && mApplication != null) {
        // 查找ANDROID_VIEWMODEL_SIGNATURE对应的构造方法
        constructor = findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE);
    } else {
        // 查找VIEWMODEL_SIGNATURE的构造方法
        constructor = findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE);
    }
    // doesn't need SavedStateHandle
    if (constructor == null) {
        // 通过Factory去创建ViewModel
        return mFactory.create(modelClass);
    }

    SavedStateHandleController controller = SavedStateHandleController.create(
            mSavedStateRegistry, mLifecycle, key, mDefaultArgs);
    try {
        T viewmodel;
        // 通过构造方法反射实例化ViewModel
        if (isAndroidViewModel && mApplication != null) {
            viewmodel = constructor.newInstance(mApplication, controller.getHandle());
        } else {
            viewmodel = constructor.newInstance(controller.getHandle());
        }
        viewmodel.setTagIfAbsent(TAG_SAVED_STATE_HANDLE_CONTROLLER, controller);
        return viewmodel;
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Failed to access " + modelClass, e);
    } catch (InstantiationException e) {
        throw new RuntimeException("A " + modelClass + " cannot be instantiated.", e);
    } catch (InvocationTargetException e) {
        throw new RuntimeException("An exception happened in constructor of "
                + modelClass, e.getCause());
    }
}
```

这里的mFactory是在SavedStateViewModelFactory实例化时创建的，可能是AndroidViewModelFactory类型或者NewInstanceFactory类型。

```java
public SavedStateViewModelFactory(@Nullable Application application,
        @NonNull SavedStateRegistryOwner owner,
        @Nullable Bundle defaultArgs) {
		// ...
    mFactory = application != null
            ? ViewModelProvider.AndroidViewModelFactory.getInstance(application)
            : ViewModelProvider.NewInstanceFactory.getInstance();
}
```

以AndroidViewModelFactory为例来看create

```java
public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
    if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
        //noinspection TryWithIdenticalCatches
        try {
            return modelClass.getConstructor(Application.class).newInstance(mApplication);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InstantiationException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        } catch (InvocationTargetException e) {
            throw new RuntimeException("Cannot create an instance of " + modelClass, e);
        }
    }
    return super.create(modelClass);
}
```

其实就是通过构造方法反射创建了ViewModel而已。



总体来看ViewModelProvider的任务就是创建ViewModel并将ViewModel存储到ViewModelStore中。而ViewModel的创建都是通过Model.class的构造方法反射创建的。





### 3. ViewModelStore

上一节猜测ViewModelStore是一个存储K-V结构的数据类型，那么来看下它的源码。

```java
public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    /**
     *  Clears internal storage and notifies ViewModels that they are no longer used.
     */
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

原来ViewModelStore内部就是封装了一下HashMap，实例化的ViewModel都被存储到了HashMap中。另外注意到ViewModelStore中还有一个clear方法，用来清理ViewMode和Map集合。在哪里对ViewModelStore进行清楚呢？这个还需要到Activity中找答案了，注意到在ComponentActivity的构造方法中有如下代码：

```java
public ComponentActivity() {
    Lifecycle lifecycle = getLifecycle();

    if (Build.VERSION.SDK_INT >= 19) {
        getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_STOP) {
                    Window window = getWindow();
                    final View decor = window != null ? window.peekDecorView() : null;
                    if (decor != null) {
                        decor.cancelPendingInputEvents();
                    }
                }
            }
        });
    }
    getLifecycle().addObserver(new LifecycleEventObserver() {
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            // Activity的销毁事件
            if (event == Lifecycle.Event.ON_DESTROY) {
                // Clear out the available context
                mContextAwareHelper.clearAvailableContext();
                // And clear the ViewModelStore
                if (!isChangingConfigurations()) {
                    getViewModelStore().clear();
                }
            }
        }
    });
    getLifecycle().addObserver(new LifecycleEventObserver() {
        @Override
        public void onStateChanged(@NonNull LifecycleOwner source,
                @NonNull Lifecycle.Event event) {
            ensureViewModelStore();
            getLifecycle().removeObserver(this);
        }
    });


}
```

全局搜素发现只有在ComponentActivity构造方法中有一处调用了ViewModelStore的clear方法，很显然，这里是通过LifeCycle监听Activity的销毁事件，然后才对ViewModel进行了清理。

### 4.ViewModelStore的状态保存

我们知道，屏幕旋转时候Activity会经历销毁和重建的过程，那么ViewModelStore是什么时候被保存的呢？其实是在ComponentActivity的onRetainNonConfigurationInstance方法中，代码如下：

```java
// ComponentActivity
public final Object onRetainNonConfigurationInstance() {
    // Maintain backward compatibility.
    Object custom = onRetainCustomNonConfigurationInstance();

    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // No one called getViewModelStore(), so see if there was an existing
        // ViewModelStore from our last NonConfigurationInstance
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }

    if (viewModelStore == null && custom == null) {
        return null;
    }

    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

ViewModelStore的恢复则是在getLastNonConfigurationInstance中，代码如下：

```java
// Activity
public Object getLastNonConfigurationInstance() {
    return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
}
```

getLastNonConfigurationInstance的调用其实前边代码已经有提到，就是在ensureViewModelStore中。ensureViewModelStore除了在getViewModelProvider中调用外，还会在生命周期的回调中被调用，代码在上边也有贴出。



最后，还有一点比较重要，就是onRetainNonConfigurationInstance的调用时机，这点涉及到的东西也比较多，就不再这里赘述了，可以参考[ViewModel源码研究之聊聊onSaveInstanceState和onRetainNonConfigurationInstance的区别](https://juejin.cn/post/6987566061499449357)这篇文章





参考：

https://juejin.cn/post/6987566061499449357

https://juejin.cn/post/6844903801044877325

https://zhuanlan.zhihu.com/p/76361500