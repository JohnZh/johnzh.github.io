---
layout: post
title:  "从 0 到 1 组件化项目，路由原理"
date:   2020-03-19 00:00:00 +0800
categories: [android]
---

# 为什么要组件化	
单一工程架构，随着模块的增加，模块之间耦合度会加大，为了维护和开发新功能，开发人员会需要了解大部分代码，协同开发问题会增加，代码量的增加重新编译运行调试时间成本会上升，单元测试会变得很困难。

于是组件化开发就是为了解决上述的提及的这些问题，而关键点在于“解耦”。

# 单一工程架构
两层架构：

1. 主业务层：A 业务模块，B 业务模块，C 业务模块……
2. 功能依赖：工具库，网络请求库，通用可复用的控件等

明确关系：

1. 业务模块存在互相调用的关系，即 A 调用 BC，B 调用 AC……
2. 业务模块存在调用功能依赖的关系
3. 功能依赖之间存在互相调用关系

# 组件化工程架构
四层架构：

1. **App** 层
2. **Business** 主业务层：A 业务组件，B 业务组件，C 业务组件……（业务组件）
3. **Common** 业务相关功能组件层：通用可复用的控件，和业务相关的支付，扫码，分享，IM等
4. **Dependencies** 业务无关功能组件层：缓存，日志，网络（和业务相关时候就会上升到第 3 层 ），第三方依赖，例如 Glide，Retrofit，Room 等

依赖关系：
在开发阶段，Business 层的各个业务模块作为 App 独立开发，集成阶段作为 aar 集成到 App 层上，即 App 壳依赖 Business 层

Business 层即会依赖 Common 层，又会依赖 Dependencies 层，这是根据 Business 层的业务子模块需求确定的

Common 层一般是 Android Library 除了基于系统或者开发者自己完成的组件和控件，也需要依赖 Dependencies 层 去实现一些业务无关可重用的功能模块

> 补充：对于通用层，有时候根据项目需要会将现在的（3）再分为 Common 和 Basic 层，Basic 层作为提供偏系统 SDK 级别的共享组件层。

为了解耦业务组件，业务组件之间的调用和数据交换都会采用“路由”转发的方式。例如，A 业务组件 -> 路由 -> B 业务组件

# 组件化项目搭建

1. 正常创建项目
2. 为了方便管理创建业务层路径 $Project/business/
3. 创建业务相关组件层，模块类型：Android Library，$Project/common
4. 根据需求在 common 下添加业务无关组件

## 创建业务层子模块
1. New -> Module -> Phone & Tablet Module
2. 选择 Moudle 菜单选择：Refactor -> Move Directory，移动到 business/
3. 在 settings.gradle 里面配置。例如：

```
include ':business:mine' // 我的
include ':business:home' // 主页
include ':business:trade' // 业务功能：交易
```
4. 全局模块化开发配置 gradle.properties:

```
isDev=false // 开发阶段标志，业务模块独立运行调试
compileSdk=28 
minSdk=15
targetSdk=28
// 以上为模块 sdk 统一配置
```

当然，这个配置还有另外两个写法，1）写在 rootProject 的 build.gradle 里面：

```
project.ext {
    isDev = true
    compileSdk=28
    minSdk=15
    targetSdk=28
}

// 引用
rootProject.ext.isDev.toBoolean()
```
2）新建一个 config.gradle 文件，然后内容如 build.gradle，并且要在 build.gradle 里添加 `apply from: 'config.gradle'`

5. app 壳依赖配置，根据是否为开发阶段标志近进行配置

```
dependencies {
    ...
    
    if (!isDev.toBoolean()) {
        implementation project(':business:mine')
        implementation project(':business:trade')
        implementation project(':business:home')
    }
}
```
6. 业务层模块应用插件和 Application Id 配置：

```
if (isDev.toBoolean()) {
    apply plugin: 'com.android.application'
} else {
    apply plugin: 'com.android.library'
}

android {
    defaultConfig {
        ...
        if (isDev.toBoolean()) {
            applicationId "com.john.newtest.mine"
        }
    }
}
```
7. 业务层模块 AndroidManifest.xml 配置：
 
开发阶段，各个模块需要各自的 Application，Launcher Activity 来完成开发和测试，集成阶段各个模块的 AndroidManifest.xml 会被合并到一起，因此两个阶段的AndroidManifest 文件内容是有区别的：

```
android {
    ...
    
    sourceSets {
        main {
            if (isDev.toBoolean()) {
                manifest.srcFile 'src/main/dev/AndroidManifest.xml'
            } else {
                manifest.srcFile 'src/main/AndroidManifest.xml'
            }
        }
    }    
}
```

8. 业务层模块 Application

首先，业务层模块 Application 和主的 Application 有通用部分又有区别部分，因此通常做法都是 Common 层创建共享的 Application，然后各个业务模块分别继承它。

但是各个业务层的 Application 在继承后实际上是不需要打包进去的，就和开发阶段的 AndroidManifest.xml 一样，因此，这个 Application 也只是在开发阶段编译到项目中

```
android {
    ...
    
    sourceSets {
        main {
            if (isDev.toBoolean()) {
                manifest.srcFile 'src/main/dev/AndroidManifest.xml'
                java.srcDirs 'src/main/dev/java'
            } else {
                manifest.srcFile 'src/main/AndroidManifest.xml'
            }
        }
    }    
}
```
同理，对于有些只在开发阶段才需要使用的Java 文件也放在此目录下

9. 依赖管理
之前的步骤统一了 sdk 版本，对于模块都需要依赖的 common 层，或者 dependencies 层里的部分模块，或者自己写的独立工具模块，其实也可以通过 gradle 配置来实现：

举例，本地 module，本地注解解析器，第三方或者官方发布的依赖

```
dependencies { 
	implementation project(path: ':common')
	implementation project(path: ':JRouter')
	implementation project(path: ':JRouterAnnotation')
	
	annotationProcessor project(path: ':JRouterCompiler')
	
	implementation 'androidx.appcompat:appcompat:1.1.0',
	implementation 'androidx.constraintlayout:constraintlayout:1.1.3',
	implementation 'androidx.fragment:fragment:1.2.5'
}
```
改写如下:

```
// 每个业务子模块都需要下面代码
dependencies { 
    rootProject.ext.localModules.each {
        implementation project(it)
    }
    
    rootProject.ext.localProcessor.each {
        annotationProcessor project(it)
    }
    
    implementation rootProject.ext.dependencies
}

// 独立的 config.gradle，或者 build.gradle(.) 
ext {
	localModules = [
            ':common',
            ':JRouter',
            ':JRouterAnnotation'
    ]

    localProcessor = [
            ':JRouterCompiler'
    ]
	
	dependencies = [
	            'androidx.appcompat:appcompat:1.1.0',
	            'androidx.constraintlayout:constraintlayout:1.1.3',
	            'androidx.fragment:fragment:1.2.5'
	]
}
```
此后，共享的依赖就可以全部配置在这里面

这个方法灵活但有点麻烦，还有一种比较简单粗暴的方式。基于以下理由：

- 基本上 Business 层子模块都会依赖 Common 层
- Business 层子模块都会依赖共享的 Dependencies
- Business 层子模块仅自己使用的 Dependencies 只会添加到自己的 build.gradle
- Common 层对共享的 Dependencies 除了自己本身，其他的 Dependencies 也可能会有依赖关系，比如上例的 `androidx.appcompat:appcompat:1.1.0`

**因此最简单最粗暴的做法**：不使用 gradle 管理依赖，而是每个业务子模块都依赖 Common，然后 Common 的 build.gradle 里面使用 api 取代 implementation 依赖共享的 Dependencies

**当然了，这两个方式也可以混着用，一切取决于你的项目需求**

## 模块间协作，路由
路由基础功能：
- 页面的调用 + 数据的传递
- 跨模块组件的调用，eg. A 模块启动 B 模块的 Services，或者只是简单的调用 B 模块的类

可以使用现成框架 [ARouter](https://github.com/alibaba/ARouter)，但是如果觉得觉得框架太大，不易把控，可以自己写个简单的路由完成上面基础功能

> 路由先导知识 Java 反射，注解，APT，Javapoet 编写代码

### 基础路由思路
- JRouter，对外的 Api ，单例，负责定义初始化，跳转，实例化中间服务和 Fragment 的方法
- RouteMsg，帮助 JRouter 包装数据，辅助调转和通信，避免 JRouter 太过臃肿
- RouteRecord，RouteMsg 的基类，路由表里面对应一条记录
- RouteCallback，路由结果，反应模块丢失，找到，到达的情况
- Register，登记中心，登记各个模块里面的类到路由表，管理路由表，从路由表找对应记录
- IRegister，路由注册接口，定义 onRegister 方法
- RouteTables，路由表，管理[path:RouteRecord]表，以及缓存表
- ClassUtils，反射工具集合
- @Router 路径注解

#### 主逻辑
1. JRouter 初始化
2. 调用 JRouter 使用路径构造 RouteMsg，进行路由
3. 登记中心根据 RouteMsg 找到 RouteRecord，根据 RouteRecord 里面的 Class 对象去启动 Activity，实例化 Fragment，中间服务类
4. 中间服务类可以进行缓存，避免重复实

##### JRouter 初始化，即路由表初始化
1. 每个模块需要路由的组件或中间服务都使用 @Router 打上注解，定义 path
2. 使用 APT 技术为每个模块解析 @Router 注解的类和注解值，为每个模块创造一个实现了 IRegister 的工具类，onRegister 方法的实现：调用 Register 进行需要路由的类的注册。要求实现 IRegister 的工具类的包名统一，名字不重复
3. 在公共的 Application 里面进行 Jrouter 的初始化，内部调用 Register 的初始化：通过包名获取这个包名下的所有类，即获取之前为每个模块创建的实现 IRegister 的工具类，假设叫 RegisterHelper
4. 通过反射实例化所有的 RegisterHelper，并调用 onRegister 方法完成注册

上述的思路，对路由 Activity 和实例化 Fragment 已经非常足够了，但是对中间服务类的实现还需要更进一步的优化。

#### 中间服务类
先了解下这个概念，首先明确这个东西的目的是为了模块之间类的调用，即通信或者说是数据传递。其实，要实现数据的传递其实方法挺多，举例：

- 广播：全局广播是基于进程间通信，本地广播是基于 handler 的观察者模式，如果要选可以使用本地广播。
- 基于全局静态表，其实这和路由表是一样的，路由表也是全局静态表。实现区别在于表里面的 Value 值的选择。

实现参考 1：Value 值选择 LiveData<T>，LiveData 是基于组件生命周期和数据变化的双重观察者容器。静态表：`static Map<String, MutableLiveData<T>> map`。但是这个实现和我们的中间服务类就没有半毛钱关系了

实现参考 2：Value 值选择接口。这个就是所谓的中间服务类，也是模块之间可以调用类的方式。原理也不复杂，使用当前的路由表 `Map<path: String, Class>`，Class 要求都是实现统一接口的，比如 `IProvider`（说句实在话，实现统一接口只是为了区分，如果说除了 Activity 和 Fragment 没有其他的考虑了，不实现都没关系）。

然后将被访问的类再抽出一个接口，比如 IService，这个接口会放到共享的模块里面，被访问的类会实现这个接口，其他模块的访问者通过 path 得到这个 Class，然后通过反射实例化，再通过向上转型，使用 IService 对象接收，这样就可以在其他模块访问这个类的方法了。

参考 2 唯一的缺点就是每要访问任何一个模块的类，都必须有一个接口，如果要实现模块的完全解耦，这些接口肯定也不能单纯放在共享模块，比如 Common。**原因：A 模块需要访问 B 模块的类，但是 C 模块不一定需要访问 B 模块的类，因此 C 依赖上 B 模块类的实现接口没有意义**。

**合理的拆分**应该是 A 模块自己暴露可以被访问的类：创建并依赖一个新的 A-api 模块，就放接口类，其他模块如果需要访问 A 里面的类，自己去依赖 A-api 模块

# 关于资源文件命名
由于最后集成阶段资源文件都是会打包到一起的，因此一定要避免资源名称重名的情况，如果app 模块和子模块有两个 Activity layout xml 文件都是 activity_main.xml 最后只会使用 app 模块下面的 activity_main.xml

# 参考代码
[https://github.com/JohnZh/SnippetProject](https://github.com/JohnZh/SnippetProject) 的以下模块

- app
- common
- business/ 
- JRouter 自定义路由框架
- JRouterAnnotation 自定义路由框架注解
- JRouterCompiler 自定义路由框架注解解析器

# 参考
- [ARouter Github](https://github.com/alibaba/ARouter)
- [JavaPoet Github](https://github.com/square/javapoet)
