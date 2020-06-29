---
layout: post
title:  "解决升级 Android Studio 3.6.1 后无法运行 Java 代码的问题"
date:   2020-03-20 00:06:00 +0800
categories: [android, java]
---


# 问题
最近升级了 Android Studio 到 V3.6.1 后发现之前创建的 Lib Module 无法运行纯 Java 代码了，部分错误信息如下：

```
FAILURE: Build failed with an exception.

* Where:
Initialization script '/private/var/folders/q7/rlfldg551dx7_r90x2_1hcww0000gn/T/MyCode_main__.gradle' line: 20

* What went wrong:
A problem occurred configuring project ':purejava'.
> Could not create task ':purejava:MyCode.main()'.
   > SourceSet with name 'main' not found.
```

因为开发中有时需要用纯 Java 代码做些小测试，所以这个新问题还是得解决

# 解决问题

查到的解决办法，其中
 
 - 使用 Run "XXX.main()" with Coverage 这个方法是可行的，但是太麻烦
 - 给.idea/gradle.xml 添加 `<option name="delegatedBuild" value="false" />
` 没试，原因是根据代码字面意思是不委托 gradle 进行 build，似乎不妥
 - 最后使用的方法。修改了 Lib Module 的 build.gradle 为如下代码：

```
apply plugin: 'java'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    testImplementation 'junit:junit:4.12'
}

```
这其实是完全把 lib 模块当成 java 项目来进行 build 了

# 使用 Lib Module 运行 Java 代码
补充添加 LibModule 用来运行 Java 代码的步骤：

1. 新建 Module 选择 Android Library 名字随意
2. 在 src/main/java/packageName下 新建主类并添加 main 方法
3. 选择 Edit Configurations... 也在主菜单 Run -> Edit Configurations...
4. ‘+’ 号新建一个 Application 的配置，填入 
    - Main class，运行的主类
    - Use classpath of module，当前 lib module
    - JRE，可以选 jre，也可以选 android 的
5. Apply，OK，选择刚刚新建的 Configuration
6. Run

# Android Studio 只运行 Java 代码
原理上和新建 Lib Module 方法类似：

1. 新建空白 Activity 项目
2. 删除 Settings.gradle 里面的 include ':app'。新版本还有 rootProject.name 也删除
3. 删除 app 整个文件路径，虽然不删其实也没事
4. 替换项目的 build.gradle 为：

```
apply plugin: 'java'

repositories {
    google()
    jcenter()
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    testImplementation 'junit:junit:4.12'
}
```
5. 新建运行主文件 src/main/java/packagename/MyCode.java 
6. 新建 Application 运行配置：
    - Main class：packagename/MyCode
    - Use classpath of module：这个时候因为已经没有 Module，因此是选择当前项目
    - JRE，选一个 jre1.8 之类的就行，一般会有默认
7. 选择这个运行配置
8. Run