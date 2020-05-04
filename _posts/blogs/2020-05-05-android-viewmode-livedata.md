---
layout: post_no_cmt
title:  "Android MVVM 之 ViewMode，LiveData 原理"
date:   2020-05-05 01:00:00 +0800
categories: [android]
---

# 问题
Android 框架中 Activity/Fragment 属于界面管理器。Android 系统会管理界面管理器的生命周期，因此，由 Android 系统控制界面管理器的销毁和重建会影响某些用户操作，设备事件

具体一点，**控制界面管理器的销毁和重建会导致临时性界面相关数据的丢失**。例如，Activity 里面可能包含用户列表。因配置更改或者系统回收重建 Activity 后，新 Activity 必须重新提取用户列表。

对于简单的数据，Activity 可以使用 onSaveInstanceState() 保存（序列化）数据，并在 onCreate() 中恢复数据。**但是这个方法只适合少量数据，而不适合数量较大的数据，如列表和位图**

另一个问题，界面控制器会经常进行异步调用，调用会需要一些时间才能返回结果，**界面控制器需要管理异步调用调用，确保系统在销毁界面控制器之后清理这些调用以免潜在内存泄露**。这个管理工作十分巨大。

界面控制器的异步调用还有一个问题，**界面控制器因系统原因重建的情况下，异步调用会重新发送，造成资源浪费**

界面控制器主要用于显示界面数据、对用户操作做出响应或处理操作系统通信（如权限请求），再加上如果负责从数据库或网络加载数据，会使类越发膨胀。**界面控制器逻辑中分离出数据操作（业务层）会减少测试难度和增加代码可维护性**

## 总结为了解决的问题

- 解决较大数据，如列表和位图，在界面管理器因系统原因销毁重建的情况下状态保存的问题。同时，可以避免重新获取数据
- 简化异步操作（获取数据或者业务操作）的管理，关联生命周期，避免内存泄露
- 分离数据业务逻辑层，易于测试和代码维护

# 例子

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

## ViewModel 之 ViewModelProvider 与 ViewModelStore

```
public ViewModelProvider(@NonNull ViewModelStoreOwner owner, @NonNull Factory factory) {
    this(owner.getViewModelStore(), factory);
}

public ViewModelProvider(@NonNull ViewModelStore store, @NonNull Factory factory) {
    mFactory = factory;
    mViewModelStore = store;
}
```
- `AppCompatActivity` 的父类 `ComponentActivity` 实现了 `ViewModelStoreOwner` 接口

`ComponentActivity#getViewModelStore`：

```
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("Your activity is not yet attached to the "
                + "Application instance. You can't request ViewModel before onCreate call.");
    }
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}
```
- 揭示了获取 `ViewModelStore` 的两种方式，从 NonConfigurationInstances 里面恢复， new 一个新的
- `ViewModelStore`：实际上就是一个 Value 为 ViewMode 的 HashMap 的包装类

```
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
接着回到 `ViewModelProvider#get`：

```
@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull Class<T> modelClass) {
    String canonicalName = modelClass.getCanonicalName();
    if (canonicalName == null) {
        throw new IllegalArgumentException("Local and anonymous classes can not be ViewModels");
    }
    return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
}

@NonNull
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }
    if (mFactory instanceof KeyedFactory) {
        viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
    } else {
        viewModel = (mFactory).create(modelClass);
    }
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```
- key 为固定固定字符串："androidx.lifecycle.ViewModelProvider.DefaultKey" + ViewMode 实现类全名
- 对有缓存的情况进行判断，缓存的 ViewMode 是当前要获取的类型的实例，直接返回，否则使用 Factory 去创建
- 无缓存的情况也是使用 Factory 去创建
- 这里直接用了最简单的内置的 NewInstanceFactory 来创建，即直接通过反射创建 modelViewClass.newInstance()
- 返回创建好 ViewMode 之前，还需要缓存到 mViewModelStore 里面
- 补充除了这里看到的 KeyedFactory 和最简单的 NewInstanceFactory，ViewModelProvider 还实现了一个 AndroidViewModelFactory，这个工厂是为了创建 Application 的 ViewMode，因此它也是单例工厂

## ViewModel 保存数据的原理

通过上面的代码可以知道，具体实现的 ViewMode 实际上是保存在 mViewModelStore 里面，而 mViewModelStore 除了第一次的 new，还可以从 NonConfigurationInstances 恢复，这是重点。

```
// Activity#getLastNonConfigurationInstance
public Object getLastNonConfigurationInstance() {
    return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
}
```
对应的方法是：`ComponentActivity#onRetainNonConfigurationInstance` 以及调用它的 `Activity#retainNonConfigurationInstances`

- ComponentActivity 的 NonConfigurationInstances 保存着 mViewModelStore，而 Activity 的 NonConfigurationInstances 把 ComponentActivity 的 NonConfigurationInstances 保存在其 NonConfigurationInstances.activity 里
- 即，Activity.NonConfigurationInstances.activity.viewModelStore 是 mViewModelStore
- `Activity#retainNonConfigurationInstances` 会在 Activity 的 Destroy 阶段被调用，保存到对应的 ActivityClientRecord 里，这在后面的 relaunch 的执行里面会使用同一个 ActivityClientRecord 来进行 Activity 的重启动，并获取到之前保存的 lastNonConfigurationInstances

至此，ViewModel 的工作原理已经清楚了，在 Activity 因系统原因，destroy-create 的过程中，ViewModel 一直保存在 NonConfigurationInstances，而 NonConfigurationInstances 保存在ActivityClientRecord 中，并未销毁

## 基于 ViewModel 的通信
ViewModel 可以用于 Activity 和 Fragment，以及 Fragment 和 Fragment 之间的通信

**前提**：通过 `ViewModelProvider#get` ViewModel 时候，ViewModelProvider 的构造器的 lifecycleOwner 都需要用同一个 Activity 的

在了解原理后其实很好解释：因为同一个 Activity（lifecycleOwner） 同一个 ViewModelStore

## ViewModel
综上，ViewModel 的特性似乎和其本身并没有多大关系，其内部代码，唯一会被明确使用到的就是清理方法 `ViewModel#clear`，会在 （Activity/Fragment）LifecycleOnwer 在 Destroy 状态下且非配置变化的情况下调用

```
// ComponentActivity.java

getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            if (!isChangingConfigurations()) {
                getViewModelStore().clear();
            }
        }
    }
});

// ViewModelStore.java

public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.clear();
    }
    mMap.clear();
}

// ViewModel.java
protected void onCleared() {
}

@MainThread
final void clear() {
    mCleared = true;
    // Since clear() is final, this method is still called on mock objects
    // and in those cases, mBagOfTags is null. It'll always be empty though
    // because setTagIfAbsent and getTag are not final so we can skip
    // clearing it
    if (mBagOfTags != null) {
        synchronized (mBagOfTags) {
            for (Object value : mBagOfTags.values()) {
                // see comment for the similar call in setTagIfAbsent
                closeWithRuntimeException(value);
            }
        }
    }
    onCleared();
}
```

# LiveData
例子中使用到的是 `MutableLiveData<T>`，继承于 `LiveData<T>`

```
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```
上面都是设置数值的方法

- postValue 从主或者非主线程上设置值（估计底层使用了 MainLooper 的 Handler）
- setValue 从主线程上设置值


例子中还涉及到一个的异步获取数据的方法：

```
mainViewMode.mLiveData.observe(this, new Observer<List<User>>() {
    @Override
    public void onChanged(List<User> users) {
        // update ui
    }
});
```

## 代码分析

### 获取值
```
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}
```
- 调用该方法的线程合法性判断，必须是主线程
- 当前 LifecycleOwner 的 Lifecycle 状态为 DESTROYED，return
- 构造观察者包装
- 以“观察者-包装”的形式加入到队列，同时判断观察者是否在观察多个 LifecycleOwner，这是非法的
- 已经添加到队列过，合法的情况（同一个 Activity 调用多次 observe），return
- 未添加过，观察者包装注册 LifecycleOwner 的 Lifecycle，后续 Lifecycle 的状态变化会通知到观察者包装
 
#### 深入 Lifecycle 的观察者模式

`owner.getLifecycle()` 获取到的实际上是

```
// ComponentActivity.java
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

// LifecycleRegistry.java
public LifecycleRegistry(@NonNull LifecycleOwner provider) {
    mLifecycleOwner = new WeakReference<>(provider);
    mState = INITIALIZED;
}

@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```
- ComponentActivity 使用 LifecycleRegistry 管理生命周期，构造参数是本 Activity，且在 LifecycleRegistry 内保存为弱应用
- 添加观察者方法：
    - 初始化状态，第一次是 INITIALIZED
    - 将传入的观察者（LiveData 里面的 LifecycleBoundObserver）和状态一起构造成“带状态的观察者”
    - 以“观察者-带状态的观察者”的格式加入队列
    - 第一次添加观察者：
        - isReentrance：false
        - targetState：INITIALIZED
        - mAddingObserverCounter：1
        - while 不执行
        - 执行 sync()：isSynced()为 true，while 不执行
        - mAddingObserverCounter：0

##### 接收 Lifecycle 事件
Activity Lifecycle 的监听使用了 ReportFragment

```
// ComponentActivity.java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mSavedStateRegistryController.performRestore(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
    if (mContentLayoutId != 0) {
        setContentView(mContentLayoutId);
    }
}

// ReportFragment.java
public static void injectIfNeededIn(Activity activity) {
    // ProcessLifecycleOwner should always correctly work and some activities may not extend
    // FragmentActivity from support lib, so we use framework fragments for activities
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```
通过在 ReportFragment 的生命周期里面调用 `dispatch(Lifecycle.Event)` 来更新 Activity.mLifecycleRegistry，调用`LifecycleRegistry#handleLifecycleEvent`

```
// ReportFragment.java

···
@Override
public void onStart() {
    super.onStart();
    dispatchStart(mProcessListener);
    dispatch(Lifecycle.Event.ON_START);
}
···

private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

除了 ReportFragment 生命周期方法里来更新 Activity.mLifecycleRegistry，Activity#onSaveInstanceState 里面也进行了一次更新

```
// ComponentActivity.java

@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
    Lifecycle lifecycle = getLifecycle();
    if (lifecycle instanceof LifecycleRegistry) {
        ((LifecycleRegistry) lifecycle).setCurrentState(Lifecycle.State.CREATED);
    }
    super.onSaveInstanceState(outState);
    mSavedStateRegistryController.performSave(outState);
}
```

##### 处理 Lifecycle 事件

```
// LifecycleRegistry.java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```
如果Activity 才启动，当前状态 mState 赋值 CREATED；调用 `sync()`，isSynced()：false，执行 while

```
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```
满足执行 forwardPass() 方法的条件

```
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}
```
遍历“观察者-带状态的观察者”的队列，调用“带状态的观察者”分发当前状态即将进入生命周期事件：

```
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```
接之前例子的状态：INITIALIZED，将会分发 ON_CREATE 事件，调用 `ObserverWithState#dispatchEvent`

```
// ObserverWithState.java
ObserverWithState(LifecycleObserver observer, State initialState) {
    mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
    mState = initialState;
}

void dispatchEvent(LifecycleOwner owner, Event event) {
    State newState = getStateAfter(event);
    mState = min(mState, newState);
    mLifecycleObserver.onStateChanged(owner, event);
    mState = newState;
}
```
继而调用了 `LifecycleBoundObserver#onStateChanged`

```
@Override
public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    activeStateChanged(shouldBeActive());
}
```
接之前的例子，mOwner 生命周期当前状态是 CREATED

shouldBeActive() 为判断当前“观察者包装”是否应该是活跃的，实现上就是比较当前观察的 LifecycleOwner 状态大于等于 STARTED，即 STARTED，RESUMED，如果是 CREATED，那么 shouldBeActive() 即为 false

后期，LifecycleBoundObserver 被通知，即 `onStateChanged` 被调用，同时 `mOwner.getLifecycle().getCurrentState()` 得到大于等于 STARTED 的值，shouldBeActive() 为 true， `activeStateChanged(true);` 就会被调用

```
// ObserverWrapper.class

void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        onActive();
    }
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
    }
    if (mActive) {
        dispatchingValue(this);
    }
}
```
最终会调用 `LiveData.dispatchingValue(this)`，详细看下面的“[派发和通知](#dispatch_notify)”

### 设置数据
```
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};

@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```
- postValue 最终还是会执行 setValue 的代码
- ArchTaskExecutor#postToMainThread 实现上是调用了 DefaultTaskExecutor 对象的 postToMainThread()，而该方法的实现是创建了 Handler(Looper.getMainLooper())，继而调用 Handler#post
- setValue 做线程合法性检查，必须是主线程
- 操作版本 ++
- 设置实际的 Data 
- 派发

<a id="dispatch_notify"/>

#### 派发数据

```
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            considerNotify(initiator);
            initiator = null;
        } else {
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```
- dispatchingValue 被调用的情况分两种，setValue 导致的 `dispatchingValue(null)`; 由于 LifecycleOwner 生命周期的变化，导致 LifecycleBoundObserver 被通知，即 `LifecycleBoundObserver#onStateChanged` 被调用，然后调用 `activeStateChanged(true)`，继而调用 `dispatchingValue(this)`
- 上述两种情况区别条件就是代码中的 `if (initiator != null)`，然后分别执行：
    - 当前被通知的“观察者包装”为入参的 `considerNotify(initiator)`
    - 加入队列的“观察者包装”为入参的 `considerNotify(iterator.next().getValue())`
    - 注意，执行队列中的通知并不会删除队列里面的“观察者-观察者包装”

##### 队列里面的“观察者-观察者包装” 清除
这个事件不需要开发者自己去处理。或者说，开发自己去处理这个事件的情况只发生在调用`observeForever` 的情况，调用这个方法的情况下，并无注册（观察）任何 LifecycleOwner，因此也就没有基于生命周期的自管理。

自管理的情况，即有注册（观察）LifecycleOwner 的情况，当“观察者包装”被通知状态改变的时候，且为 DESTROYED 的时候，会进行观察者队列里面“观察者-观察者包装”的清除

```
// LifecycleBoundObserver.class

@Override
public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
        removeObserver(mObserver);
        return;
    }
    activeStateChanged(shouldBeActive());
}
```

#### 通知数据

```
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```
- `activeStateChanged(true)` 激活 LifecycleBoundObserver
- 每调用一次 `LifecycleBoundObserver.mObserver.onChanged` 通知一次数据，`LifecycleBoundObserver.mLastVersion` 就更新为最近的操作

# 调用总图
![LiveData 调用总图](https://img-blog.csdnimg.cn/20200505014540217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70)

