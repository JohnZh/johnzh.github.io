---
layout: post_no_cmt
title:  "10 个 Java 基础知识点"
date:   2016-05-12 06:00:00 +0800
categories: [java]
---
# 1. final/finally/finalize

1. final 类不能被继承，其成员方法会隐式的指定为 final
2. 普通类 final 方法不能被子类重写，private 方法会隐式的指定为 final
3. final 成员变量
	- 必须要赋初始值，两种方式：声明时候就赋初值；构造器里面赋初值
	- 变量为基础型，变量值不能改	；变量为引用，引用地址值不能改，但是引用指向对象的内容可以改

区别于 finally finalize

- finally: 异常处理模型的一部分，其代码块必须执行，不论异常发生与否
- finalize：Object 方法。于垃圾回收前调用。实际应用中不要依赖该方法去回收短缺资源，因为无法得知该方法的调用时间

# 2. equals 区别于 "=="
- 基础数据比较应该用 "=="，比较的是值
- 类用 == 比较，比较的内存中的地址，只有指向同一个 new 出来的对象才是true
- Object 定义 equals()，初始实现是比较内存地址，有些类做了覆盖，如String，Integer，Date，其 equals 比较内容
- 自己实现的类的两个对象没有覆盖 equals，比较出来依然是false

# 3.  String/StringBuffer/StringBuilder
- String：字符串常量，源码里面成员变量 char value[] 是 final 的
- StringBuilder：字符串变量，char value[] 不是final
- StringBuffer vs StringBuilder 全部方法都 synchronized，线程安全
- 执行速度 StringBuilder > StringBuffer > String
	- 特殊情况，String s = "abc" + "efg" 比  new StringBuilder("abc").append("efg") 速度快，因为 JVM 处理 s 会是 String s = "abcdef" 这样的形式。

# 4. Override/Overload
- Override 覆盖
	- 子类覆盖父类的方法，或者接口方法实现
	- 作用域比父类大，protected 覆盖后只能是 protected/public
	- 抛出异常必须一致，或者更少（子类可以帮父类解决一些问题）
	- 无法覆盖private，‘覆盖’了其实相当于重写一个方法
	
- Overload 重载
	- 同一个方法不同方法签名（参数的类型、个数、顺序）
	- 不同的返回类型，异常，作用域（权限）无法重载
	- 方法的异常数量和类型不会对重载有影响
	- 方法签名不一样，返回值允许不一样
	
# 5. Interface/abstract class
- Interface 多继承 / abstract class 单一继承
- 成员变量：Interface 只能有常量 / abstract class 和普通类一样
- 方法：Interface 默认都是 public/ 非 abstract 方法必须实现
- 设计：强调功能，like-a，轮胎“可滚动” / 强调所属关系，is-a，小轮胎是轮胎

# 6. static/non-static class
- static 修饰类只发生在内部类（正常类用其修饰是错误的）
- (内部)静态类本质是完全独立类，可单独实例化 / 内部(非静态)类本质上持有外部类引用，可以访问外部类所有信息，且无法单独实例化，得先实例化外部类
- 内部类无法存在静态的成员 / 静态类都可以有
- 使用：
	- 操作封装
	- 由于内部类能访问外部类所有信息，配合内部继承可以实现 java 的多继承

# 7. 多态实现原理
运行时指向代码区不同：
Java程序运行时，类的相关信息放在方法区，这些信息中有块叫方法表的区域，
该表包含该类型所有方法的的信息和指向这些方法实际代码的指针
(这些方法可能是覆写的代码，也可能是继承基类的代码)
因此在执行的时候，虽然调用同一个方法名，但是执行的代码区是不同的

# 8. wait() vs sleep()
- wait 是 object 对象，调用后出让对象锁，notify 后恢复运行
- sleep 只是让出 cpu 给其他线程，指定时间恢复，和锁无关

# 9. foreach vs for
- 从生成的.class 文件观察，foreach最终代码会生成效率最高代码。因此对于数组，不论是对象数组还是基础类型数组，最终都会变成 for 循环
- 对于容器类型 foreach 会更加容器类型来判断使用 Iterator 或者 for，比如 LinkedList 和 ArrayList

# 10. 反射的作用与原理
- 原理：就是将内存中类的各种信息（字节码）映射（装载）到 Class 类里
- 作用：可以实现动态配置和加载类，实现类之间的完全解耦