## Activity的启动模式

Android 提供了四种Activity启动方式：

### 1.标准模式：standard

每启动一次Activity，就会创建一个新的Activity实例并置于栈顶。谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中。

特殊情况下，如果在Service或Application中启动一个Activity，其并没有所谓的任务栈，可以使用标记位Flag来解决。解决办法：为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，创建一个新栈。

### 2.栈顶复用模式：singleTop

如果需要新建的Activity位于任务栈栈顶，那么此Activity的实例就不会重建，而是复用栈顶的实例。并回调：

```java
   @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
    }
```

由于不会重建一个Activity实例，则不会回调其他生命周期方法。

应用场景：在通知栏点击收到的通知，然后需要启动一个Activity，这个Activity就可以用singleTop，否则每次点击都会新建一个Activity。

### 3.栈内复用模式：singleTask

该模式是一种单例模式，即一个栈内只有一个该Activity实例。singleTask会具有clearTop特性，会把之上的栈内Activity清除。与singleTop一样，系统也会回调它的onNewIntent方法。

当一个singleTask模式的Activity请求启动后，系统会首先寻找是否存在这个Activity想要的任务栈，如果不存在，就创建一个新的任务栈，然后创建Activity的实例后把这个Activity放入到这个任务栈中。如果存在所需要的任务栈，这时，要看这个Activity是否在栈中有实例存在，如果有，那么系统就会把这个Activity调用到栈顶，并调用它的onNewIntent方法。如果不存在，就会创建这个Activity的实例，并将其压入栈顶。

该模式，可以通过在AndroidManifest文件的Activity中指定该Activity需要加载到哪个栈中，即singleTask的Activity可以指定想要加载的目标栈。singleTask和taskAffinity配合使用，指定开启的Activity加入到哪个栈中。

```xml
<activity android:name=".Activity1"
    android:launchMode="singleTask"
    android:taskAffinity="com.test.task"
    android:label="@string/app_name">
</activity>
```

**关于taskAffinity**：TaskAffinity可以理解为任务相关性，这个参数标识了Activity所需要的任务栈名字。每个Activity都有taskAffinify属性，这个属性指出了它希望进入的任务栈。如果一个Activity没有显式的指明该Activity的taskAffinity，那么它的这个属性就等于Application指明的taskAffinity，如果Application也没有指明，那么该taskAffinity的值就等于包名。

上述配置中，Activity1指定了taskAffinity为“com.test.task"，在这种情况下启动Activity1，如果“com.test.task"栈不存在，则创建这个栈，并把创建的Activity压入这个栈内。如果“com.test.task"栈存在，并且其中没有该Activity实例，则会创建Activity并压入栈顶，如果这个任务栈中有该Activity实例，则clearTop把该Activity实例之上的Activity杀死并出栈，重用并让该Activity实例处在栈顶，然后调用onNewIntent()方法。

**singleTask应用场景：**

对于大部分应用，当我们在主界面点击返回按钮都是退出应用，那么当我们第一次进入主界面之后，主界面位于栈底，以后不管我们打开了多少个Activity，只要我们再次回到主界面，都应该使用将主界面Activity上所有的Activity移除的方式来让主界面Activity处于栈顶，而不是往栈顶新加一个主界面Activity的实例，通过这种方式能够保证退出应用时所有的Activity都能被销毁。

### 4.单例模式：singleInstance

作为栈内复用的加强版，打开该Activity时，直接创建一个新的任务栈，并创建该Activity实例放入栈中。一旦该模式的Activity实例已经存在于某个栈中，任何应用在激活该Activity时都会重用该栈中的实例。

应用场景：呼叫来电界面

## 场景分析

假设当前有两个任务栈，前台任务栈中有AB（自栈底到栈顶，下同）两个Activity，后台任务栈有CD两个Activity。如果CD都被设置了SingleTask，此时，通过A启动D，那么整个后台任务栈都会被切换到前台，可以理解为栈变成了ABCD的顺序，此时按back按键后，D出栈，C显示；再次按back键，C出栈，B显示，以此类推。

如果上述场景中A启动的是C而不是D，那么D会被首先clearTop掉，即任务栈变为ABC，按返回时C先出栈，接着B、A出栈。





## 相关面试题

### 1.一个被设置为standard模式的Activity在被ApplicationContext启动时会什么会报错？

因为standard模式的Activity默认会进入启动它的Activity所属的任务栈，但是由于非Activity类型的Context自身并没有所属的任务栈，因此，此时启动standard模式的Activity就有问题了。可以通过给待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样在启动这个Activity时就会为它创建一个新的任务栈。此时，这个Activity实际上是以singleTask模式启动的。

### 2.有A、B、C三个Activity，其中A启动模式为standard，taskAffinity为默认。B和C的启动模式都设置为singleTask,并且都指定taskAffinity为"com.test.task"。此时，如果在A中点击按钮启动B，在B中点击按钮又启动C，在C中点击按钮又启动A，最后在A中点击按钮启动B。现在按两次返回键会回到哪里？

会回到桌面。第一次A启动B，由于B指定了单独的任务栈“com.test.task",因此会首先创建”com.test.task"任务栈（以下简称该任务栈为T），然后实例化B，并将B加入T任务栈中。此时B启动C，由于C与B指定的都是T任务中，因此C会实例化后加入到T任务栈，接着在C中启动A，由于此时T任务栈中没有A，因此会实例化A并将A加入到T任务栈。接着A再次启动B，由于B已经存在于T任务栈中，因此会执行clearTop，将C和A都出栈，然后显示出B。此时点击返回B会finish掉，此时后台任务栈中的A会显示出来，再次点击返回，A结束回到桌面。





