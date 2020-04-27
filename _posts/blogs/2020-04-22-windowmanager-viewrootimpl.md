---
layout: post_no_cmt
title:  "记录 WindowManager，Window，Activity，DecorView，ViewRootImpl"
date:   2020-04-22 08:00:00 +0800
categories: [android]
---

# Window，Activity，View 的关系
- Activity 持有成员变量 PhoneWindow（继承自 Window）
- PhoneWindow 持有一个 DecorView（继承自 Framelayout）
- DecorView 内部持有两个 ViewGroup 成员变量 mContentRoot，mContentParent，mContentRoot 会根据 Widow Features 的属性选择合适的系统布局资源，inflate 之后添加到 DecorView 里面
- mContentRoot 的布局里面都会有一个 id 为 android.id.content 的 ViewGroup，这个会被 mContentParent 成员变量引用
- Activity#setContentView 实际上是把 View 加到了 mContentParent 里面

详细参考源码，或很久之前写的 Blog：[Activity、Window、View的关系](https://blog.csdn.net/wyzxk888/article/details/51443406) 


# ActivityThread 启动 Activity
```
ActivityThread#handleLaunchActivity {
    final Activity a = performLaunchActivity(r, customIntent);
}
 | 
 ∨ 
ActivityThread#performLaunchActivity {
	activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
	activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);
}
 |
 ∨
Activity#attach {
	mWindow = new PhoneWindow(this);
	mWindow.setWindowManager((WindowManager)context.getSystemService(Context.WINDOW_SERVICE), ...)
	mWindowManager = mWindow.getWindowManager();
}
 |
 ∨
Window#setWindowManager {
	mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}
Window#getWindowManager {
    return mWindowManager;	
}
```
到此，Activity，Window，WindowManager 关系以及何时实例化都很清楚了。

- ActivityThread 在 performLaunchActivity() 方法里实例了将要启动的 Activity
- 在 Activity 实例的 attach 方法里面创建了 PhoneWindow 对象，赋值于成员变量 mWindow，
- 再通过系统服务创建 WindowManager 对象
- 最后通过 mWindow 的 setWindowManager() 和 getWindowManager()，使得 Window 和 Activity 都持有关联WindowManager 对象
- 当然，在 setWindowManager() 的时候，WindowManager 对象也持有了当前的 Window 对象（实际上是 WindowManagerGlobal 持有）

## 回到 ActivityThread#performLaunchActivity
在 `Activity#attach` 之后

```
mInstrumentation.callActivityOnCreate -> Activity#onCreate
```
# DecorView 关联 WindowManager
在 `ActivityThread#handleLaunchActivity` 执行之后，接下去执行：

```
public void handleStartActivity(ActivityClientRecord r, PendingTransactionActions pendingActions)  {

	Activity#performStart -> mInstrumentation.callActivityOnStart(this) -> Activity#onStart
	
	(可能执行) mInstrumentation.callActivityOnRestoreInstanceState -> Activity#onRestoreInstanceState
	
	mInstrumentation.callActivityOnPostCreate -> Activity#onPostCreate
}


public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward, String reason) {
	ActivityThread#performResumeActivity -> Activity#onResume()
	
    if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            
            ......

            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }
            
            ......
            
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
}
```
到此，Window 里面的 DecorView 也出现了，并且添加到了 WindowManager 里面

一处小细节：DecorView 最开始是 INVISIBLE，添加到管理后才通过 `activity.makeVisible()` 变成了 VISIBLE

## 深入 WindowManager 
WindowManager 的实现类 WindowManagerImpl#addView 实际上是把 View 交给了 WindowManagerGlobal

```
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }
```
## WindowManagerGlobal#addView

```
root = new ViewRootImpl(view.getContext(), display);

view.setLayoutParams(wparams);

mViews.add(view);
mRoots.add(root);
mParams.add(wparams);

// do this last because it fires off messages to start doing things
try {
    root.setView(view, wparams, panelParentView);
}
```
WindowManagerGlobal 最后是把 DecorView 交给 ViewRootImpl

# ViewRootImpl 
> The top of a view hierarchy, implementing the needed protocol between View and the WindowManager. This is for the most part an internal implementation detail of WindowManagerGlobal.

层次结构的顶部，实现 View 和 WindowManager 之间需要的协议。这是 WindowManagerGlobal 内部实现内容的最大部分

理解上述描述，先要理解 WindowManager 是什么

每个 WindowManager 实例会绑定一个特定的 Display，Display 可以理解为屏幕的逻辑类，或者抽象类。结合之前的分析，Activity 启动过程中新建了 Window，Activity 通过 setContentView() 方法使 Window 创建了 DecorView 和其子 ViewGroup，最后在 ActivityThread 执行 handleResumeActivity 过程中（Activity#onReume 之后），又把 DecorView 全部交给了 WindowManger，显然是需要 WindowManger 作为中间人把 View 显示到屏幕上

因此描述可以理解为，WindowManger 会调用 ViewRootImpl 来告诉 View 要测量，绘制，以及最后通知系统服务，安排展示

## ViewRootImpl 源码小结
```
ViewRootImpl#setView 
 |
 ∨ 
ViewRootImpl#requestLayout 
 |
 ∨ 
ViewRootImpl#checkThread() 检查是否是当前线程
ViewRootImpl#scheduleTraversals 
 |
 ∨
mChoreographer.postCallback(mTraversalRunnable) 异步 Handler 请求
 |
 ∨ 
mTraversalRunnable.run().doTraversal()
 |
 ∨ 
ViewRootImpl#performTraversals 完成测量，布局，绘制。
三个重要的方法：
	- performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	- performLayout(lp, desiredWindowWidth, desiredWindowHeight);
	- performDraw();

```
之后 ViewRootImpl#setView 内部调用 mWindowSession#addToDisplay，进行一次 IPC，告诉 WindowMangerService 展示 Window

## ViewRootImpl 解释了几个问题
1. 为什么 Activity#onCreate 的时候，View#getMeasured* 为 0。因为测量布局绘制的过程是在 Activity#onResume 之后才提交的，且是异步的。
2. 那 Activity#onResume 里面调用 View#getMeasured* 就不为 0 了吗，不一定，因为是异步的，这是不可靠的
3. 那什么时候可以确定 View#getMeasured* 测量的完成。测量之后，绘画之前会执行：            `mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();` 对应可用回调：
	
	```
	view.getViewTreeObserver().addOnGlobalLayoutListener(new OnGlobalLayoutListener() {
	    @Override
	    public void onGlobalLayout() {
	    }
});
	```
4. Activity#onCreate() 里面其他线程更新 UI 不会抛出异常，但是 view 显示后，其他线程更新 UI 会抛出异常。原因是检查线程的判断在 ViewRootImpl 里，但是 ViewRootImpl 实例的创建却在 ActivityThread#handleResumeActivity
5. View#getViewRootImpl 是否公用同一个对象。是的

```
public ViewRootImpl getViewRootImpl() {
    if (mAttachInfo != null) {
        return mAttachInfo.mViewRootImpl;
    }
    return null;
}
```
ViewRootImpl 对象保存在 mAttachInfo 里面，AttachInfo是保存 view 和 Window 之间信息的类，在 ViewRootImpl 构造器里面创建。通过 mView#dispatchAttachedToWindow(mAttachInfo, 0) 传给 DecorView 及其子 View

```
ViewRootImpl.setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
	...
	mView = view;
	...
}

ViewRootImpl.performTraversals() {
	...
	View host = mView
	host.dispatchAttachedToWindow(mAttachInfo, 0);
	...
}
```
