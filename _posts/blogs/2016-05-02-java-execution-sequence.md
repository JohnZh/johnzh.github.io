---
layout: post_no_cmt
title:  "Java：类执行顺序和变量赋值顺序"
date:   2016-05-02 00:06:00 +0800
categories: java
---
# 类执行顺序
内容：静态代码块，代码块，构造器的执行顺序，存在继承的情况

非继承的情况：
1. 静态代码块
2. 代码块
3. 构造器

继承的情况：
1. 父类静态代码块
2. 子类静态代码块
3. 父类代码块
4. 父类构造器
5. 子类代码块
6. 子类构造器

测试代码：

```
public class Son extends Parent {
    static {
        System.out.println("Son static block");
    }

    {
        System.out.println("Son block");
    }

    public Son() {
        System.out.println("Son constructor");
    }
}

public class Parent {
    static {
        System.out.println("Parent static block");
    }

    {
        System.out.println("Parent block");
    }

    public Parent() {
        System.out.println("Parent constructor");
    }
}

// main():
Son son = new Son();

// output:
Parent static block
Son static block
Parent block
Parent constructor
Son block
Son constructor
```

# 变量赋值顺序
内容：成员变量声明直接赋值，代码块赋值，构造器赋值，存在继承的情况
基于类执行顺序的结论，可以得出：**构造器赋值肯定在最后**。而声明直接赋值必然早于代码块赋值。因此顺序为：

1. 父类变量声明直接赋值
2. 父类代码块赋值
3. 父类构造器赋值
4. 子类变量声明直接赋值
5. 子类代码块赋值
6. 子类构造器赋值


测试代码：

```
public class Parent {
    String name = "parent";

    public Parent() {
        System.out.println("Parent constructor: " + name);
    }

    {
        System.out.println("Parent block: " + name);
        name = "parent block";
    }
}
public class Son extends Parent {
    String name = "son";

    {
        System.out.println("Son block: " + name);
        name = "son block";
    }

    public Son() {
        System.out.println("Son constructor: " + name);
    }
}

// main:
Son son = new son();

// output:
Parent block: parent
Parent constructor: parent block
Son block: son
Son constructor: son block

// 如果块赋值早于声明直接赋值，那么第一个 block 的输出就应该是 null
```

> 声明时直接赋值和构造器赋初始值在使用上因为前者不是很符合面向对象的思想，所有并不被提倡
