---
layout: post_no_cmt
title:  "Java：注解与注解解析器"
date:   2016-05-01 00:06:00 +0800
categories: java
---

# 注解
> An annotation is a form of metadata, that can be added to Java source code. Classes, methods, variables, parameters and packages may be annotated. Annotations have no direct effect on the operation of the code they annotate.

注解是一种能被添加到 java 源代码中的元数据。**类、方法、变量、参数和包**都可以用注解修饰。**注解对于它所修饰的代码并没有直接的影响**。

# 何时会使用注解
注解在开发过程中可能会在下面几个情况下使用：

- 为编译器提供信息：帮助编译器检查错误，或者忽略警告。例如 JDK 内置的几个注解。
- 编译时和发布时的处理：比如在编译和发布阶段利用注解信息生成代码，xml 文件等其他信息。
- 运行时的处理：运行时利用注解信息进行其他操作，通常和反射机制一起实现。例如一些 ORM 框架的实现。

# 自定义注解
除了 JDK 内置的几个注解：(在 java.lang 包中)

```
@Override
@Deprecated
@SafeVarargs (jdk 1.7)
@SuppressWarnings
```
开发者可以自定义注解，但是在自定义注解之前先要了解几个元注解

## 元注解
- @Target 注解用于的地方。参数 ElementType：
	- CONSTRUCTOR：构造器的声明
	- FIELD：域的声明，包括 enum 的实例
	- LOCAL_VARIABLE：局部变量的声明
	- METHOD：方法声明
	- PACKAGE：包声明
	- PARAMETER：参数声明
	- TYPE：类，接口（包括注解类型），enum 声明
	- jdk 1.8 后还加了ANNOTATION_TYPE，TYPE_PARAMETER，TYPE_USE
- @Retention 保留级别。参数 RetentionPolicy：
	- SOURCE：保留在源文件级别。代码被编译后注解将被编译器丢弃
	- CLASS：源文件编译成 CLASS 文件后的级别。CLASS 文件加载到 VM 后被 VM 丢弃
	- RUNTIME：资源加载并允许在 VM 时期也保留的注解。因此可以通过反射机制读取注解的信息
- @Documented 注解会被包含在 javadoc 中
- @Inherited 允许子类继承父类的注解

## 注解元素
自定义注解举例：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Table {
    public String name() default "";
}
```
name 就是注解元素。其特点：

- 类型固定：基本类型(int/float/boolean 等)，String，Class，enum，Annotation 以及以上类型的数组(@Target 的注解元素就是 ElementType[]，多个的时候用逗号分隔)
- 注解元素必须有值，否则必须设置默认值。上例，name 的默认值就是 ""；
- 非基本类型的元素，默认值不能使用 null
- 赋值支持命名为 value 的快捷方式，例如：
	
	```
	@Retention(RetentionPolicy.RUNTIME)
	@Target(ElementType.TYPE)
	public @interface VARCHAR {
		public int value() default 0; // value 为最大长度
		public boolean allowNull default true;
	}
	
	如果我们只需要规定其最大长度，那么可以如下：
	VARCHAR(30) 
	但是如果两个注解元素都需要进行设置，则 value 必须写明：
	VARCHAR(value = 30, allowNull = false)
	
	```

复杂点的注解：包含了枚举和注解的注解元素

```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    ColumnType value() default ColumnType.STRING;
    String name() default "";
    int length() default 0;
    Constraint constraint() default @Constraint;
}
------

public enum ColumnType {
    STRING,
    DOUBLE,
    FLOAT,
    INTEGER,
    LONG
}
------

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Constraint {
    boolean primaryKey() default false;
    boolean allowNull() default true;
    boolean unique() default false;
}
```
注意：@Column 里面 constraint 的默认值就是 @Constraint 所有元素默认值，如果要改变其中一个默认值之后再作为 constraint 默认值可以：`    Constraint constraint() default @Constraint(unique=true);
`

# 注解处理器与反射
正如注解的定义所说，注解其实对其所修饰的代码没有直接影响，因此如果没有注解处理器来解析注解并做相应的操作，注解不会比注释更有用。

通常保留级别为 RUNTIME 的注解都是配合反射进行处理，继 @Table 和 @Column 举一个 ORM 注解处理器的例子：

```
public static void main(String[] args) {
    TableUtils.ConnectionSource source = new TableUtils.ConnectionSource();

    TableUtils.createTable(source, User.class);
    TableUtils.createTable(source, UserOrder.class);
}
```
ConnectionSource 理论上应该是数据库连接资源，这里执行简化为打印 SQL 语句：

```
public static class ConnectionSource {
    public void executeSQL(String sql) {
        System.out.println(sql);
    }
}
```
User 和 UserOrder 为需要映射到 DB 的 Model：

```
@Table
public class User {
    public User(long id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    @Column(value = ColumnType.LONG,
            constraint = @Constraint(primaryKey = true))
    public long id;

    @Column(value = ColumnType.STRING, length = 30,
            constraint = @Constraint(allowNull = false))
    public String name;

    @Column(ColumnType.INTEGER)
    public int age;
}
------

@Table(name = "Order")
public class UserOrder {
    @Column(value = ColumnType.INTEGER, name = "order_id")
    public long orderId;

    @Column(value = ColumnType.STRING,
            length = 10,
            constraint = @Constraint(unique = true, primaryKey = true))
    public String serialNo;

    @Column(value = ColumnType.STRING, length = 50)
    public String name;
}
```
打印结果：

```
CREATE TABLE USER(ID LONG PRIMARY KEY, NAME VARCHAR(30) NOT NULL, AGE INT);
CREATE TABLE ORDER(ORDER_ID INT, SERIALNO VARCHAR(10) PRIMARY KEY, NAME VARCHAR(50));
```
而 TableUtils 在这就作为注解处理器：

```
# createTable 方法：
public static void createTable(ConnectionSource source, Class clazz) {
    StringBuilder sql = new StringBuilder("CREATE TABLE");
    Table table = (Table) clazz.getAnnotation(Table.class);

    if (table == null) {
        return;
    }

    String tableName = table.name();
    if (tableName.equals("")) {
        tableName = clazz.getSimpleName();
    }
    sql.append(" ").append(tableName.toUpperCase()).append("(");

    for (Field field : clazz.getDeclaredFields()) {
        Column column = field.getAnnotation(Column.class);
        if (column == null) continue;

        String columnName = column.name();
        if (columnName.equals("")) {
            columnName = field.getName();
        }
        sql.append(columnName.toUpperCase());

        String columnType = getColumnType(column);
        String columnConstraint = getConstraints(column.constraint());
        sql.append(columnType).append(columnConstraint).append(", ");
    }
    String finalSql = sql.toString();
    finalSql = finalSql.substring(0, finalSql.length() - 2) + ");";

    source.executeSQL(finalSql);
}

# getColumnType 方法：
private static String getColumnType(Column column) {
    StringBuilder sb = new StringBuilder("");
    if (column.value() == ColumnType.STRING) {
        sb.append(" VARCHAR(").append(column.length()).append(")");
    } else if (column.value() == ColumnType.LONG) {
        sb.append(" LONG");
    } else if (column.value() == ColumnType.DOUBLE) {
        sb.append(" DOUBLE");
    } else if (column.value() == ColumnType.FLOAT) {
        sb.append(" FLOAT");
    } else if (column.value() == ColumnType.INTEGER) {
        sb.append(" INT");
    }
    return sb.toString();
} 

# getConstraints 方法：
private static String getConstraints(Constraint constraint) {
    StringBuilder sb = new StringBuilder("");
    if (!constraint.allowNull()) {
        sb.append(" NOT NULL");
    } else if (constraint.primaryKey()) {
        sb.append(" PRIMARY KEY");
    } else if (constraint.unique()) {
        sb.append(" UNIQUE");
    }
    return sb.toString();
}
```

# 注解处理与 APT
见[《Android：Java 注解解析与 APT》]({% post_url /blogs/2020-03-20-java-apt %})
