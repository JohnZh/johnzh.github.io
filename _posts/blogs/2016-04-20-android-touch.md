---
layout: post_no_cmt
title:  "Android Touch 事件传递流程小结"
date:   2016-04-20 08:00:00 +0800
categories: [android]
---
# Touch 事件传递流程
``` 
User touch
 |
 ∨
Screen
 |
 ∨
System
 |
 ∨                  Activity#dispatchTouchEvent return true 
App                                                ∧   
 |                                                 |
...                                  Activity#onTouch 
 |                                            ∧    |          
 ∨                                            |    |
Activity#dispatchTouchEvent <-----------------+----+
 |                                            |    |
 ∨                                            |    |
Window#superDispatchTouchEvent <--------------+----+
 |                                            |    |
 |                                         false   true  
 ∨                                            |    |
DecorView#superDispatchTouchEvent <-----+-----+----+
 |                                            |    |
 ∨                                            |    |
ViewGrop#dispatchTouchEvent <-----------------+----+
 |                                            |    |
 ∨                                            |    |
ViewGrop#onInterceptTouchEvent -- true +      |    |
 |                                     |      |    |
false                                  |      ∧    ∧
 |                                     |      |    |
 ∨                                     |      |    |
find child view (1) -- (not found)     |    false  true
 |                           |         ∨      |    |
(Child View)       +-- (ViewGroup as View) <--+    |
 |                 |                          |    |
 ∨                 ∨                          |    | 
View#dispatchTouchEvent <---------------------+----+
 |                                            |    |
mOnTouchListener != null                      |    |
&& View.enable                                |    |
 |      |                                     |    |
 |  mOnTouchListener.onTouch() -- true -------|--->|
 |      |                                     |    |
 |     false                                  |    |
 |<-----+                                     |    |
 ∨                                            |    |
View#onTouch -- false (event not comsume) ----+    |
 |                                                 |
return true --- true (event has consumed) ---------+
```
(1) 处：在 ViewGrop#onInterceptTouchEvent 中，查找 touch 范围内的子 View，未找到的情况就会直接调用 super#dispatchTouchEvent，ViewGrop 继承 View

简单的说
- 事件从 Activity#dispatchTouchEvent 开始向下传，通过 ViewGrop#dispatchTouchEvent 传到 ViewGroup 层（**注意，第一个 ViewGroup 其实是 DecorView，然后通过 find Child View 继续向下面的子 ViewGroup 传**）
- ViewGroup 可以通过 ViewGroup#onInterceptTouchEvent 对事件进行拦截标记，return true 即表示这个事件当前 ViewGroup 拦截了，但是拦截之后依然要 Override 这个 ViewGroup#onTouch 进行处理
- ViewGroup 不拦截（默认），会继续向 ViewGroup 里合适的 ChildView 传，**注意这个 ChildView 可以是 ViewGroup 也可以是 View，最终通过 ChildView#dispatchTouchEvent 传入**
- 最后一层
    - 已经是 View 了，调用 onTouch()
    - ViewGrop已经找不到可传的 ChildView 了，调用父类的 dispatchTouchEvent，进而调用 onTouch
- onTouch 返回值逆行，不论返回值是 true 还是 false 都会 return 会上一层
- 最后一层是 View 的，返回 true，已消费的情况，基本上是直接一层一层向上返回；返回 false，未消费的情况，返回到上一层 ViewGroup 的时候，ViewGroup 会先去执行父类的 dispatchTouchEvent，进而再执行自己的 onTouch()，并向上传该返回值，如果上层的 ViewGroup 没有任何 onTouch 消费，最终就回到 Activity#dispatchTouchEvent 里面了
- 调用Activity#onTouch，并return Activity#onTouch
