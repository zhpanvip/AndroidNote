## 一、MVC (Model View Controller)

MVC要解决的问题是控制层、数据处理层和界面交互进行解耦。

- **Model:**  负责数据的处理和加载
- **View:**  负责界面的展示
- **Controller:** 负责逻辑控制。

MVC的架构即通过 Controller 的控制操作 Model 层的数据，并返回给 View 展示。其关系如下图：

![](https://pic1.zhimg.com/80/v2-9d1b8b206bc3b782bb5dbb103bbb73e4_720w.jpg)

用户触发View层事件通知 Controller 层，Controller 层通过访问服务器或者数据库，然后通知Model更新数据，Model层更新数据后会将数据通知给View层去展示。

### 1. MVC 的优点

- 结构清晰，职责划分清晰
- 降低耦合
- 有利于组件重用

### 2. MVC的缺点

- Android中 Activity/Fragment 承担了View 和Controller两个角色，导致Activity/Fragment中代码庞大。

- View 层与 Model 层存在依赖关系，Model层直接操作View，View的修改会导致Controller和Model都需要改动



## 二、MVP (Model View Presenter)

MVP要解决的问题与MVC大同小异，即控制逻辑，数据处理逻辑以及界面交互解耦，同时，将MVC中的View和Model解耦。

MVP 架构里，将逻辑，数据，界面的处理划分为三个部分，模型(Model)-视图(View)-控制器(Presenter)。各个部分的功能如下：

- **Model模型：** 负责数据的处理和加载
- **View视图：** 负责界面的展示
- **Presenter控制器：**  负责逻辑控制

MVP和MVC最大的不同就是View层和Model层不互相持有，都通过Presenter交互。View产生事件通知Presenter，Presenter中进行逻辑处理后通知Model更新数据，Model更新数据后通知数据给Presenter，Presenter再通知View更新界面。示意图如下：

![](https://pic1.zhimg.com/80/v2-cdfc6c60e8be1a3b8caa7fe0697a2e0c_720w.jpg)



### 1. MVP 的优点

- 结构清晰，职责划分明确
- 模块间充分解耦
- 有利于组件的重用

### 2. MVP 的缺点

- 会引入大量的接口，导致项目文件数量激增
- 增大代码结构的复杂性

### MVVM (Model View ViewModel)

MVVM要解决的问题是将控制逻辑、数据处理逻辑以及界面交互进行解耦，并且能将MVC中的View和Model解耦，还可以把MVP中的Presenter和View也解耦。

MVVM架构中，将逻辑、数据、界面的处理分为三部分，即模型（Model）、视图（View）以及逻辑（ViewModel）。各个部分的功能如下：

- **Model模型**：负责数据的加载和存储
- **View视图**：负责界面的展示
- **ViewModel控制器**：负责逻辑控制

在 MVP 中，就是 View 和 Model 不相互持有，都通过 Presenter 做中转。这样可以使 View 和 Model 解耦。而在MVVM中解耦做的更彻底，ViewModel也不会持有View，其中ViewModel中的改动会自动反馈给View进行界面更新，而View的事件也会自动反馈给ViewModel。

![](https://pic4.zhimg.com/80/v2-f6f7c0e53f42e7d3291c7784b1b2d157_720w.jpg)

要达到这个效果，需要一些辅助工具，比较常用的是DataBinding。在MVVM中，数据流是由View产生事件，自动通知给ViewModel,ViewModel进行逻辑处理后通知Model更新数据。Model更新数据后，通知数据给ViewModel，ViewModel自动通知View更新界面。



https://zhuanlan.zhihu.com/p/83635530

