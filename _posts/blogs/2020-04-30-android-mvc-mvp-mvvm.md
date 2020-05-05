---
layout: post_no_cmt
title:  "Android 开发 与 MVC，MVP，MVVM"
date:   2020-05-05 23:00:00 +0800
categories: [android]
---
# 设计模式、架构模式
- 设计模式 Design Pattern
- 架构模式 Architecture Pattern

设计模式是一种代码级别设计思路的经验与总结。目的是为了代码的可复用，可维护，可读，稳健及安全。

架构模式，软件系统的架构风格，代码组织方式，目的是在于功能和模块间的解耦，代码块和功能块的复用，通常由多个设计模式组成。

> 设计模式共 23 种：常用的有策略模式，观察者模式，装饰者模式，工厂方法模式，抽象工厂模式，单例模式，命令模式，适配器模式，模板方法模式，迭代器模式，组合模式，状态模式等。参考《[设计模式](http://c.biancheng.net/design_pattern/)》

# 网络中关于 Android 开发中的架构模式
网络上提及的 Android 架构模式是 MVC（Model-View-Controller） 的，其中 C 是 Activity/Fragment，View 是 xml 布局，M 是业务相关模型数据类。

也有说应该使用 MVP（Model-View-Presenter） 模式，因为 Activity/Fragment 里面有大量处理用户输入和输出的操作，而这些职责理应属于 View，因此要把 Activity/Fragment 作为 View，另外加一个 Presenter 并且把业务逻辑放里面，同时以接口的形式负责 View 的更新，M 依旧是业务相关模型数据类

同时 Google 官方又提出了 MVVM（Model-View-ViewModel）

# 了解 MVC MVP
Model-View-Controller

Model-View-Presenter

- M：业务模型，用于处理应用程序数据逻辑和业务逻辑的部分
- V：用户界面
- C：控制器，接收用户事件，传递事件到业务模型，组织调度不同的 V 和 M
- P：控制器（没错，它也是控制器）

图示：
![MVC](https://img-blog.csdnimg.cn/20200501125323708.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70)

其实在 MVC 模式的定义中，C 的功能其实非常薄，并不涉及到业务逻辑的处理，业务逻辑的处理应该都是在 M。其实，MVP 也是一样，P 也应该是很薄的存在。MVP 本质上只是解耦了 V 和 M，本质也是 MVC 模式，因而 MVP 是区别于传统的 MVC 另外的叫法

## MVC 和 Design Pattern
一般情况下，MVC 会采用三种设计模式来实现

- 组合模式，Composite Pattern：实现图形界面 V 的树形层次结构
- 观察者模式，Observer Pattern：M 的改变会通知 V，引起 V 的改变
- 策略模式，Strategy Pattern：V 上发生的事件可能会引发不同的 C 的响应，或者说 C 的不同实现，亦或者是 C 对不同 M 的调用

# 回头看 Android 的架构模式

不做任何分层的情况下事件的流向：

1. xml 文件构成图形界面（底层是通过解析反射生成类添加到 PhoneWindow 的 DecorView 中）这里都是 V
2. 用户事件的输入发生在 Activity/Fragment 中控件的事件监听，这里涉及 C 和 V 
3. Activity/Fragment 中使用业务数据模型对象进行业务处理（数据库操作，网络请求），涉及 C 和 M 
4. 业务处理结果通过调用 Activity/Fragment 中的控件对象进行页面的更新，涉及 C 和 V

从事件流向上看下来，Android 开发的 MVC 架构模式其实是之前提及的 MVP（没有 M 对 V 的变化通知），但是并没有非常明确的层次划分或者说是模块划分

## MVP 的实现思路
Activity/Fragment 里面其实 M、V、P 的代码都有涉及，其中 M 部分的代码可以非常清楚地提取出去。关键是看 Activity/Fragment 里面 V、P 的代码要如何提取。

实现上，其实把 Activity/Fragment 作为 V 或者作为 P，然后提取并创建 P 或 V 的代码都是可以的

### Activity/Fragment 作为 V
- 就和网络上流行的模式一样，Activity/Fragment 会作为 V 实现接口 IView
- 创建  Presenter，将 Activity/Fragment 作为传入或作为构造器参数
- 同时，Presenter 可能需要关联 Activity/Fragment  的生命周期
- Presenter 里面根据需要创建 M 所需要的模型对象，操作业务数据结构 bean，M 里面定义接口用于与 Presenter 的通信
- 如果有需要，Presenter 里面 M 的实现也可以使用抽象

### Activity/Fragment 作为 P/C
- 关键在于提取所有的 V 的元素到单独的类 mvcView，暴露 View 的一些操作
- 在 mvcView 里面定义接口 Listener，留给 Activity/Fragment 实现，用于用户事件的传递。同时，在 Activity/Fragment 的生命周期方法里面关联 mvcView
- 在 Activity/Fragment 里面创建M 所需要的模型对象，操作业务数据结构 bean，M 里面定义接口用于与 Activity/Fragment 的通信
- 如果有需要，M 的实现也可以使用抽象

### 比较以上两种
个人更偏向于第二种，将 Activity / Fragment 作为 P/C，理由：

- Activity/Fragment 本身就不是顶层视图类，PhoneWindow 里面的 DecorView 才是
- Activity 继承了 Context，很多业务逻辑依赖 Context，例如权限请求。
- 上面这个理由其实很明显，因为将 Activity/Fragment 作为 V 的实现里，Presenter 的创建也需要 Activity/Fragment 实例，其实是需要 Context，并且还需要绑定 Activity/Fragment 的生命周期，那为什么不直接让 Activity/Fragment 成为 P 呢

# MVVM 
## 相关类

- LiveData 及其子类
- ViewMode 以及开发者自己继承的类

> 参考[《Android MVVM 之 LiveData，ViewMode 原理》]({% post_url/blogs/2020-05-05-android-viewmode-livedata %})

```
public class MyViewModel extends ViewModel {
  
    private MutableLiveData<List<User>> mLiveData;

    private void loadUsers() {
        // Do an asynchronous operation to fetch users.
        WebService.fetchUser(users -> {
            mLiveData.setValue(users);
        });
    }
}

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    
    MyViewModel mainViewMode
            = new ViewModelProvider(this, new ViewModelProvider.NewInstanceFactory())
            .get(MyViewModel.class);

    mainViewMode.loadUser();

    mainViewMode.mLiveData.observe(this, new Observer<List<User>>() {
        @Override
        public void onChanged(List<User> users) {
            // update ui
        }
    });
}
```
在我看来，Android 的 MVVM 架构模式并不是独立 MVP/MVC 而独立存在的一个模式，Activity/Fragment 依然作为接收 View 传递来的用户事件和调用业务模型层的控制器的存在。

只不过由于独立出来的 ViewMode 层的关系，作为连接 View 和 Mode 的桥梁，在这个模式下，**Activity/Fragment 作为控制器的工作看似变少了，似乎不再担任业务模型处理完数据后通知 View 层更新的工作**

但是本质上调用和事件关系依然是 
```
View -> Activtiy/Fragment -> ViewModel -> Model -> Observer -> Activtiy/Fragment 作为内部类调用 View 更新
```