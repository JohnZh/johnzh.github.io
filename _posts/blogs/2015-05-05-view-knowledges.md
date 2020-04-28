---
layout: post_no_cmt
title:  "Android View 知识点：创建，测量，布局，绘制"
date:   2015-05-05 08:00:00 +0800
categories: [android]
---
Last modified: 2020-04-28

# 从 XML 创建 View
不论是 Activity#setContentView()，还是 LayoutInflater#inflate，还是 View#inflate

最后都会调用 

```
LayoutInflater#inflate(resouce: int, root: ViewGroup, attachToRoot: boolean): View
 |
 |
final XmlResourceParser parser = res.getLayout(resource)
 |
 v
LayoutInflater#inflate(parser: XmlResourceParser, root: ViewGroup, attachToRoot: boolean): View
```
`LayoutInflater#inflate` 核心代码:

```
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;
        
        try {
            // Look for the root node.
            int type;
            while ((type = parser.next()) != XmlPullParser.START_TAG &&
                    type != XmlPullParser.END_DOCUMENT) {
                // Empty
            }

            if (type != XmlPullParser.START_TAG) {
                throw new InflateException(parser.getPositionDescription()
                        + ": No start tag found!");
            }

            final String name = parser.getName();

            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }
                
                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ViewGroup.LayoutParams params = null;

                if (root != null) {
                    if (DEBUG) {
                        System.out.println("Creating params from root: " +
                                root);
                    }
                    // Create layout params that match root, if supplied
                    params = root.generateLayoutParams(attrs);
                    if (!attachToRoot) {
                        // Set the layout params for temp if we are not
                        // attaching. (If we are, we use addView, below)
                        temp.setLayoutParams(params);
                    }
                }

                // Inflate all children under temp against its context.
                rInflateChildren(parser, temp, attrs, true);

                // We are supposed to attach all the views we found (int temp)
                // to root. Do that now.
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }

        }
        
        ...
        
        return result;
    }
}
```
流程说明：
1. 找到根节点开始标签
2. 找到后，如果是 `<merge>`，合法性判断，调用 `LayoutInflater#rInflate` 递归实例化所有的 View
3. 非 `<merge>`，调用 `LayoutInflater#createViewFromTag` 创建根节点 View：temp
4. 如果传入 root:ViewGroup != null，生成匹配 root 的布局参数 params，并且如果 attachToRoot:boolean == false，布局参数 params 会设置到 temp 里面
5. 调用 `LayoutInflater#rInflateChildren` 进行子 View 的创建，实现也是调用了 `LayoutInflater#rInflate`
6. 最后 root != null 且 attachToRoot，则 temp 会加到 root 里面。
7. 或者，root == null 或者 不需要 attachToRoot，那么 temp 会作为结果返回


再深入递归的实例化：

```
void rInflate(XmlPullParser parser, View parent, Context context,
        AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

    final int depth = parser.getDepth();
    int type;
    boolean pendingRequestFocus = false;

    while (((type = parser.next()) != XmlPullParser.END_TAG ||
            parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

        if (type != XmlPullParser.START_TAG) {
            continue;
        }

        final String name = parser.getName();

        if (TAG_REQUEST_FOCUS.equals(name)) {
            pendingRequestFocus = true;
            consumeChildElements(parser);
        } else if (TAG_TAG.equals(name)) {
            parseViewTag(parser, parent, attrs);
        } else if (TAG_INCLUDE.equals(name)) {
            if (parser.getDepth() == 0) {
                throw new InflateException("<include /> cannot be the root element");
            }
            parseInclude(parser, context, parent, attrs);
        } else if (TAG_MERGE.equals(name)) {
            throw new InflateException("<merge /> must be the root element");
        } else {
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }

    if (pendingRequestFocus) {
        parent.restoreDefaultFocus();
    }

    if (finishInflate) {
        parent.onFinishInflate();
    }
}
```
- `LayoutInflater#rInflate`：
    - 最外层是 white 循环，跳出条件是当前 xml 定位已经是最外层，且是 xml 元素的结束点，或者文档结尾，总之就是 xml 布局文件遍历完成
    - 会优先判断并处理 `<requestFocus>`, `<tag>`, `<include>`,  `<merge>` 的情况
    - 非以上情况创建子调用 `LayoutInflater#createViewFromTag` 创建子 View，同时会生成 parentView 的布局参数，AddView 的时候一起设置
    - 继续调用 `LayoutInflater#rInflateChildren` 递归创建并添加这个 view 下面子 View

再看具体单独 View 的实例化
```
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    if (name.equals("view")) {
        name = attrs.getAttributeValue(null, "class");
    }

    // Apply a theme wrapper, if allowed and one is specified.
    if (!ignoreThemeAttr) {
        final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
        final int themeResId = ta.getResourceId(0, 0);
        if (themeResId != 0) {
            context = new ContextThemeWrapper(context, themeResId);
        }
        ta.recycle();
    }

    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    try {
        View view;
        if (mFactory2 != null) {
            view = mFactory2.onCreateView(parent, name, context, attrs);
        } else if (mFactory != null) {
            view = mFactory.onCreateView(name, context, attrs);
        } else {
            view = null;
        }

        if (view == null && mPrivateFactory != null) {
            view = mPrivateFactory.onCreateView(parent, name, context, attrs);
        }

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(parent, name, attrs);
                } else {
                    view = createView(name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    }
    
    ......
}
```
- `LayoutInflater#createViewFromTag`：
    - 会先处理`<view>`，系统主题的应用，`<blink>`(通过 handler/message 实现闪烁的布局)
    - 接着，使用 `LayoutInflater.mFactory`，`LayoutInflater.mFactory2` 的 `onCreateView` 来创建 View。
        - 这两个 Factory 是抽象类，在系统控件的样式兼容上使用非常多。具体查看 `AppCompatActivity#onCreate` 的 `delegate.installViewFactory()` 以及 delegate 的实现类 `AppCompatDelegateImpl`
        - 开发者也可以使用 Factory 自定义全局的控件属性
    - 最后，Factory 如果不存在，或者没有创建 View，那么调用 `LayoutInflater#onCreateView`，继而调用到 `LayoutInflater#createView` 通过 classLoader 加载 View 的反射类，再通过反射类的构造器实例化 View 对象。详细内容参考下方代码

```
protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return createView(name, "android.view.", attrs);
}

public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    if (constructor != null && !verifyClassLoader(constructor)) {
        constructor = null;
        sConstructorMap.remove(name);
    }
    Class<? extends View> clazz = null;

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            // If we have a filter, apply it to cached constructor
            if (mFilter != null) {
                // Have we seen this name before?
                Boolean allowedState = mFilterMap.get(name);
                if (allowedState == null) {
                    // New class -- remember whether it is allowed
                    clazz = mContext.getClassLoader().loadClass(
                            prefix != null ? (prefix + name) : name).asSubclass(View.class);

                    boolean allowed = clazz != null && mFilter.onLoadClass(clazz);
                    mFilterMap.put(name, allowed);
                    if (!allowed) {
                        failNotAllowed(name, prefix, attrs);
                    }
                } else if (allowedState.equals(Boolean.FALSE)) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
        }

        Object lastContext = mConstructorArgs[0];
        if (mConstructorArgs[0] == null) {
            // Fill in the context if not already within inflation.
            mConstructorArgs[0] = mContext;
        }
        Object[] args = mConstructorArgs;
        args[1] = attrs;

        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        mConstructorArgs[0] = lastContext;
        return view;

    }
    
    ......
}

```

浅出：

View view = 
1. layoutInflater.inflate(resouce, root, true)
2. layoutInflater.inflate(resouce, root, false)
3. layoutInflater.inflate(resouce, null, false)
4. layoutInflater.inflate(resouce, root)
5. View.inflate(context, resource, root)

这三个方法效果上的差异：
- 首先，4,5 和 1 等价
- 2 的条件：root 不为 null，但是 attachToRoot == false，resource 的根节点会 `setLayoutParams(params)` 即被设置上 root 的布局参数。效果上就是 `layout_*` 的参数生效，eg. `layout_margin`, `layout_height`。但是返回的只是 resource 布局的 View
- 1 的条件：attachToRoot = true，`layout_*` 的参数生效，但是返回的 resource 布局的 View 外面还包了一层 root
- 3 的条件：root 为 null，首先 attachToRoot 肯定不可能为 true。其次只会返回 resource 布局的 View，且根节点 View 上的 `layout_*` 参数都会失效

# 完全自定义 View
三个过程：
- measure
- layout
- draw

代码位置：

```
ViewRootImpl.performTraversals() {
    measureHierarchy() {
        ...
        performMeasure()
        ...
    }
    
    ...
    performMeasure()
    ...
    performLayout() 
    ...
    performDraw()
    ...
}
```
## measure, onMeasure
- measure：并非直接的测量工作，而是做调度和优化，比如判断是否强重布局，或者是其他需要重布局的情况，重布局意味着需要重测量，例如利用 mMeasureCache 缓存尺寸数据，避免重复测量
- onMeasure()：实际测量工作，计算出自己期望的尺寸，并通过 setMeasuredDimension() 保存，供 parentView 使用
    - onMeasure() 的工作原理，首先计算出 View 自己的尺寸，然后再根据 parentView 提供的 “尺寸限制” **widthMeasureSpec/heightMeasureSpec** 对自己的尺寸进行修正
    - 不同的 View 的子类对 onMeasure() 实现都不同
    - View 的子类 ViewGroup 的子类 (FrameLayout, LinearLayout 等) onMeasure() 实现也都不同，ViewGroup 是抽象类没有实现 onMeasure()
    - ViewGroup 子类的 onMeasure() 通过 getChildMeasureSpec() 得出当前 View 对其 childView 的 “尺寸限制” **childWidthMeasureSpec/childHeightMeasureSpec**，再通过 child.measure(childWidthMeasureSpec, childHeightMeasureSpec) 让 childView 计算自己的尺寸
    - getChildMeasureSpec() 计算 childView 的 “尺寸限制” 是基于当前 view 的 parentView 传来 “尺寸限制” + childView 自身的 “布局参数” MarginLayoutParams 来实现的

## layout, onLayout
- layout：调用 setFrame 保存 View 实际尺寸，该方法会同时触发 onSizeChanged() 通知尺寸更改，继而调用 onLayout()
    - setFrame 实际是设置了 View 的ltrb，即矩形四点信息(left/top/right/bottom)，这其实也是确定了 View 在 parentView 里面的实际宽高，此时 getWidth()/getHeight() 不再是 0
    - 区别 getMeasuredWidth/Height() 该两方法是在 measure 过程中的值，measure 为完成之前会变化，measure 完成之后不会变化，且一般情况下等于getWidth/Height()
- onLayout：onLayout() 在 View 和 ViewGroup 里面都是空方法，一般在自定义 View 里面无 childView，onLayout() 无需实现，在自定义 ViewGroup 和 ViewGroup 子类里面 onLayout() 实现都不同
    - ViewGroup 子类里面的 onLayout() 实现，会调用 childView 的 layout() 保存他们的位置和尺寸

## draw, onDraw
- draw：绘制的总调度方法，分别调用：
    - drawBackground()，绘制背景，私有，不能重写，只能靠 xml 文件或者 setBackground() 方法来设置改变
    - onDraw()，绘制主体内容的方法，通常自定义 View 实现该方法就可以
    - dispatchDraw()，空方法，View 不用实现，ViewGroup 重写调用 childView 的 draw() 时候使用
    - onDrawForeground()，绘制装饰物，前景，滚动条
    - drawDefaultFocusHighlight()，绘制默认的聚焦高亮

## 一般自定义 View 步骤
1. 自定义属性的声明与获取
2. 测量 onMeasure()
3. *布局 onLayout()，仅 ViewGroup 时候*
4. 绘制 onDraw()
5. *派遣绘制 dispatchDraw()，仅 ViewGroup 时候*
6. 绘制前景 onDrawForeground()
7. *拦截触控事件 onInterceptTouchEvent()，仅 ViewGroup 时候*
8. 触控事件 onTouchEvent()

# View 扩展知识

## 宽高的获取
1. onWindowFocusChanged，View 初始化完毕后会被调用，Activity 的窗口得到焦点和失去焦点都会被调用一次，即 Activity 继续执行和暂停执行时，**不推荐**
2. ViewTreeObserver，当 View 树的状态发生改变或者 View 树内部的 View 可见性发现改变时，onGlobalLayout 方法将被回调，且调用次数不止一次，**一般推荐**
3. View.post(new Runnble)，两种情况：**推荐**
    1. View 已经完成测绘，这种直接调用主线程 handler.post(new Runnable) 发送一个 Message 并回调给 Runnble处理
    2. View 没有完成测绘，这种会先将 Runnble 任务通过数组保存下来，当View开始测绘时，会将包存下来的 Runnble 任务通过主线程 handler 进行发送消息，由于消息在 messageQueue中 是串行处理的，所以 view.post 的 Runnble 任务会在 view 测绘完成后在开始执行其自身的消息


## 测量，布局，绘画次数
常规布局下的自定义 View 的绘制和布局初始化阶段：

api < 24 (Android 7) 之前，依次：
1. onMeasure
2. onMeasure 
3. onLayout 
4. onMeasure
5. onLayout 
6. onDraw 

api >= 24 进行了优化：
1. onMeasure
2. onMeasure 
3. onLayout 
4. onDraw

api >= 29 后又进行了优化，效果同上，RenderThread 初始化采用了阻塞机制
参考：[这里](https://www.zhihu.com/question/263393657/answer/918036584)

### 第一次和第二次测量
测量的起点在 ViewRootImpl#performTraversals:

第一次调用 ViewRootImpl#performTraversals 执行了两次测量，第一次发生在
```
[Android 9 (api 28) 源代码]

[line 2122]
// Ask host how big it wants to be
windowSizeMayChange |= measureHierarchy(host, lp, res,
        desiredWindowWidth, desiredWindowHeight);
        
// measureHierarchy(...)
measureHierarchy(final View host, final WindowManager.LayoutParams lp,
            final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    ...
[line 1833]
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    ...
}

```

第二次在
```
[line 2541] 
 // Ask host how big it wants to be
performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

```

接着会在 `ViewRootImpl#performTraversals` 方法里面调用
`performLayout(lp, mWidth, mHeight);`[line 2590]，继而调用 DecorView#layout：

```
host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
```

## 测量的起点
```
childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);

performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);

// getRootMeasureSpec()
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

### getRootMeasureSpec()
- 一般情况下 lp.width 和 lp.height，Window 的参数宽高都是 MATCH_PARENT，见 Window.mWindowAttributes 构造器
- MATCH_PARENT 的情况会构造一个(windowSize, EXACTLY) 的 measureSpec
- WRAP_CONTENT 的情况会构造一个(windowSize, AT_MOST) 的 measureSpec
- Window 大小设置成了固定值是构造 (固定值, EXACTLY) 的 measureSpec

### MeasureSpec 构成
- MeasureSpec 是 int 类型，size 和 mode 采用都保存在了 int 类型里面，且采用了二进制方式
- int 类型 32 位，第 31 和 32 位(index 30，31)，保存 mode，第 1~30 位（index 0~29）保存 size

```
public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
private static final int MODE_SHIFT = 30;
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

...
public static final int EXACTLY     = 1 << MODE_SHIFT;
...
```
### MeasureSpec 的 mode 
- **EXACTLY**: parentView 已经为 childView 确定了准确的大小，childView 必须遵从这些大小，忽略 childView 自己想要大小
    - 对应布局参数 "MATCH_PARENT" 或具体值
- **AT_MOST**: childView 可以想要多大就多大
    - 对应布局参数 "WRAP_CONTENT"
- **UNSPECIFIED**: parentView 没有对 childView 进行任何限制


### performMeasure() 
```
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```
核心是 `mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);` 其中 mView 为 DecorView

#### 调用流程

```
mView.measure(childWidthMeasureSpec, childHeightMeasureSpec)
 |
 ∨
View.measure(childWidthMeasureSpec, childHeightMeasureSpec)
 |
 ∨
Override 
DecorView.onMeasure(widthMeasureSpec, heightMeasureSpec)
 |
super.onMeasure(widthMeasureSpec, heightMeasureSpec)
 |
 ∨
Override 
FrameLayout.onMeasure(widthMeasureSpec, heightMeasureSpec) {
         
    for(View child: mChildren):
    ViewGroup.measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0)
     |
    one of children:
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec)
     |
     ∨
    View.measure(childWidthMeasureSpec, childHeightMeasureSpec)
    
    maxWidth = max(children.width with marginLeft and marginRight)
    maxHeight = max(children.width with marginTop and marginBottom)
    
    maxWidth += padding
    maxHeight += padding
    
    maxWidth = max(maxWidth, getSuggestedMinimumHeight())
    maxHeight = max(maxHeight, getSuggestedMinimumHeight())
    
    .....
    
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
}
```
FrameLayout 在 onMeasure 后就已经有了 measuredWidth/Height


#### 代码块调用关系
![image](https://img-blog.csdnimg.cn/20200426165015667.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70)


#### 部分代码说明
`ViewGroup#measureChildWithMargins` 要求child 测量自己的尺寸，会考虑 parentView 的约束和 padding，以及childView 的 layout_margin* 设置

```
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}

```
说明：
- getChildMeasureSpec(spec, padding, childDimension) 的三个参数分别传入了：
    - spec：parentView 的尺寸
    - padding：parentView 的 padding + child 的 margin + 同方向上已经被用掉的空间（可能是 parentView 下其他的子 view 占据的空间）
    - childDimension：当前 view 自己的布局宽高约束值
- 首先计算出 size，这个 size 一般情况下为 parentView 的尺寸减去传入的 padding。这个 size 会作为大部分情况下的返回值，即 childView 的尺寸 childWidthMeasureSpec, childHeightMeasureSpec 里面的 size
- getChildMeasureSpec 具体逻辑：(结合之前 MeasureSpec 的 mode 的内容)
    - parentView mode 是 EXACTLY，此时 parentView 布局宽高参数为 MATCH_PARENT 或者 固定值
        - childView 的宽高约束为 >= 0 的固定值，childView 尺寸的 size 和 mode 分别为固定值和 EXACTLY
        - childView 的宽高约束为 MATCH_PARENT，childView 尺寸的 size 和 mode 分别为 size 和 EXACTLY
        - childView 的宽高约束为 WRAP_CONTENT，childView 尺寸的 size 和 mode 分别为 size 和 AT_MOST
    - parentView mode 是 AT_MOST，此时 parentView 布局宽高参数为 WARP_CONTENT
        - childView 的宽高约束为 >= 0 的固定值，childView 尺寸的 size 和 mode 分别为固定值和 EXACTLY
        - childView 的宽高约束为 MATCH_PARENT，childView 尺寸的 size 和 mode 分别为 size 和 AT_MOST **(会尊重 parentView 的 mode 也变成 AT_MOST)**
        - childView 的宽高约束为 WRAP_CONTENT，childView 尺寸的 size 和 mode 分别为 size 和 AT_MOST 
    - parentView mode 是 UNSPECIFIED，这是第一次 Measure 时候，parentView 为了让子 View 第一次测量自己尺寸而传的
        - childView 的宽高约束为 >= 0 的固定值，childView 尺寸的 size 和 mode 分别为固定值和 EXACTLY
        - childView 的宽高约束为 MATCH_PARENT，childView 尺寸的 size 和 mode 分别为 (< api23 为 0，否则 size》) 和 UNSPECIFIED
        - childView 的宽高约束为 WRAP_CONTENT，childView 尺寸的 size 和 mode 分别为 (< api23 为 0，否则 size》) 和 UNSPECIFIED
    - 最后组合确定的 size 和 mode 成 MeasureSpec 返回
- 回到 measureChildWithMargins() 调用 child.measure(childWidthMeasureSpec, childHeightMeasureSpec) 继续完成测量