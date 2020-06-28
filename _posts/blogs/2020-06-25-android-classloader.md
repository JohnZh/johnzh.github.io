---
layout: post_no_cmt
title:  "ClassLoader 归纳总结"
date:   2020-06-25 01:00:00 +0800
categories: [android]
---

# 前言
ClassLoader 在热修复和插件化技术中都是非常核心的技术，因此此文对类加载器进行一些归纳总结

# 主要类加载器
- ClassLoader：抽象类，顶层父类
- BaseDexClassLoader：继承 ClassLoader，主要的子类之一
- BootClassLoader：继承 ClassLoader，和 ClassLoader 定义在同一文件下，用于加载 Android framework 里面的类，比如 Activity
- PathClassLoader：继承 BaseDexClassLoader，负责 App 开发中的类加载，比如依赖中的类，appcompat 下的 AppCompatActivity
- DexClassLoader：继承 BaseDexClassLoader，额外的类加载器，例如插件的 apk，dex 等

## 关系图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200628122825873.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d5enhrODg4,size_16,color_FFFFFF,t_70#pic_center)

## 调用关系
在使用 PathClassLoader 或者 DexClassLoader 加载类的时候（两者在 Android 8.0 之后源码除了构造器其他都是一样的）会进行如下的过程：

PathClassLoader#loadClass 或 DexClassLoader#loadClass 都调用 ClassLoader#loadClass

ClassLoader#loadClass:

```
0. 从 vm 里面找已经加载的类
1. vm 里面找不到，parent 不为 null，调用 parent#loadClass
    1.1 PathClassLoader 的 parent 是 BootClassLoader，DexClassLoader 的 parent 一般设为 PathClassLoader
    1.2 BootClassLoader#loadClass 也是先去 vm 里面找已经加载的类，找不到再调用 BootClassLoader#findClass
    1.3 BootClassLoader#findClass 调用了 Class#classForName
    1.4 Class#classForName 是一个 native 方法
2. parent 为 null，或者 parent#loadClass 返回 null，调用 ClassLoader#findClass
3. ClassLaoder#findClass 是一个 protected 方法，直接抛出异常，因此其实现应该是交给子类来完成的
4. PathClassLoader 或者 DexClassLoader 并没有 findClass 的实现，findClass 的实现在 DexBaseClassLoader 里面
5. DexBaseClassLoader#findClass 调用了成员变量 pathList: DexPathList 的 findClass 方法
6. DexPathList#findClass 变量成员变量 dexElements: Element[]，调用每个数组成员的 Element#findClass
7. Element#findClass 会调用其成员变量 dexFile: DexFile 的 loadClassBinaryName
8. DexFile#loadClassBinaryName 调用 DexFile#defineClass
9. DexFile#defineClass 调用 DexFile#defineClassNative，该方法为 native 方法
```

**步骤 1 即双亲委托机制，目的是为了避免重复加载，以及安全性考虑，防止核心 api 被篡改**

# 热修复，插件化
在需要重启的热修复和插件化的技术中，有一个核心技术就是将 app 的 classLoader（PathClassLoader）里面的 Element[] 替换成修复的 dex 构成的 Element[] + 原来的 Element[] 的新数组，或者是插件 apk 构成的 Element[] + 原来的 Element[] 的新数组

- 重启的热修复：
    - 制作 dexFiles
    - 反射获取 makeDexElements 方法（各个 Android 版本会有差异）并构造出 Element[]
    - 反射获取原始的 Element[]
    - 利用 Array 工具构造出新的 Element[]，将上述两个 Element[] 复制进去，**dexFiles 在前，原始 Element[] 在后**
    - 将新的 Element[] 通过反射赋值到原始的 Element[] 的成员变量上

- 插件化：
    - 打包插件 module 的 apk
    - 通过 DexClassLoader 加载插件 apk
    - 反射获取 DexClassLoader 里面的 Element[]（之前要先获取 DexPathList）
    - 反射获取原始的 Element[]
    - 利用 Array 工具构造出新的 Element[]，将上述两个 Element[] 复制进去，**原始 Element[] 在前，插件的在后**
    - 将新的 Element[] 通过反射赋值到原始的 Element[] 的成员变量上