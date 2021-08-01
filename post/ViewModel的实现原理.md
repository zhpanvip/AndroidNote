## 一、ViewModel的使用

在页面（Activity/Fragment）很简单的情况下，通常我们会将UI交互，数据获取与处理等相关业务逻辑，全部写在页面中，但是在页面复杂的情况下，这样做是不合适的，它不符合“**单一责任**”原则。页面只应该负责接收用户的交互，以及将数据展示到屏幕上，相关数据应该单独存放和处理。

为此，Android为我们提供了**ViewModel**类，专门用于存放应用程序页面所需的数据。它将页面所需的数据从页面中剥离出来，页面只需要处理用户交互，以及负责展示数据的工作。

![img](https://pic4.zhimg.com/80/v2-48b149d7b2f75c79173ff6f7a32b62db_720w.jpg)

另外，如果我们的应用程序支持横竖屏切换，当用户旋转手机屏幕时，我们还需要考虑数据的存储与恢复。如果数据不进行存储，那么通常我们还需要重新去获取一次。

而ViewModel能为我们解决这个问题，它独立于配置变化。也就是说，屏幕旋转导致的Activity重建，并不会影响到ViewModel的生命周期。

![img](https://user-gold-cdn.xitu.io/2019/3/21/1699bd67cfefec24?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### 1. 引入ViewModel

> implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"

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





内容来自以下链接：

https://juejin.cn/post/6844903801044877325

https://zhuanlan.zhihu.com/p/76361500


