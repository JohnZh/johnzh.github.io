---
layout: post_no_cmt
title:  "Android View 1：深入 XML 创建 View"
date:   2015-05-05 08:00:00 +0800
categories: [android]

---

# XML 创建 View 流程

不论是 Activity#setContentView()，还是 LayoutInflater#inflate，还是 View#inflate

最后都会调用 

```java
LayoutInflater.inflate(resouce: int, root: ViewGroup, attachToRoot: boolean): View
|
V
final XmlResourceParser parser = res.getLayout(resource)
|
V			
LayoutInflater.inflate(parser: XmlResourceParser, 
				root: ViewGroup, attachToRoot: boolean): View
```

## LayoutInflater.inflate

```java
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
3. 非 `<merge>`，调用 `LayoutInflater#createViewFromTag` 创建根节点 temp: View
4. 如果传入 root:ViewGroup != null，生成匹配 root 的布局参数 params，并且如果 attachToRoot:boolean == false，布局参数 params 会设置到 temp 里面
5. 调用 `LayoutInflater#rInflateChildren` 进行子 View 的创建，实际调用了 `LayoutInflater#rInflate`
6. 最后 root != null 且 attachToRoot，则 temp 会加到 root 里面。
7. 或者，root == null 或者不需要 attachToRoot，那么 temp 会作为结果返回



## LayoutInflater.rInflate 递归创建 View

```java
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

解析：

- 最外层是 white 循环，跳出条件是当前 xml 定位已经是最外层，且是 xml 元素的结束点，或者文档结尾，总之就是 xml 布局文件遍历完成。因此当 parent:View 为某个组件的时候，eg. `<TextView />` 并不会进入 while 循环

- 会优先判断并处理 `<requestFocus>`, `<tag>`, `<include>`,  `<merge>` 的情况

- 非以上情况创建子调用 `LayoutInflater#createViewFromTag` 创建子 View，同时会生成 parentView 的布局参数，AddView 的时候一起设置

- 继续调用 `LayoutInflater#rInflateChildren` 递归创建并添加这个 View 的子 View

- 处理完所有的标签后会调用 `parentView.onFinishInflate`

  > 注意：使用<merge/>和 attachToRoot 的时候，finishInflate 值为 false，意味着这个parentView.onFinishInflate 不会被调用



## LayoutInflater.createViewFromTag 从 xml 标签创建 View

```java
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

解析：

- 会先处理`<view>`，系统主题的应用，`<blink>`(通过 handler/message 实现闪烁的布局)
- 接着，使用 `LayoutInflater.mFactory`，`LayoutInflater.mFactory2` 的 `onCreateView` 来创建 View。
  - 这两个 Factory 是抽象类，在系统控件的样式兼容上使用非常多。具体查看 `AppCompatActivity#onCreate` 的 `delegate.installViewFactory()` 以及 delegate 的实现类 `AppCompatDelegateImpl`
  - 开发者也可以使用 Factory 自定义全局的控件属性
- 最后，Factory 如果不存在，或者没有创建 View，那么调用 `LayoutInflater#onCreateView`，继而调用到 `LayoutInflater#createView` 通过 classLoader 加载 View 的反射类，再通过反射类的构造器实例化 View 对象。详细内容参考以下代码：



## LayoutInflater.onCreateView & createView

```java
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



# layoutInflater.inflate 方法差异

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