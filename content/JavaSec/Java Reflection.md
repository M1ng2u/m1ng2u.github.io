---
title: Java Reflection
subtitle:
date: 2025-01-29T06:32:30+08:00
slug: java-reflection
draft: false
author:
  name: M1ng2u
  link:
  email:
  avatar:
description:
keywords:
license:
comment: true
weight: 0
tags:
  - Java
  - Reflection
categories:
  - JavaSec
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRelated: false
hiddenFromFeed: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: true
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

# Java Reflection 学习

jdk 1.8.0_72

## 反射机制简述

java 在运行时：

1. 对于任意一个类，能获取它的所有方法、属性、构造器

2. 任意一个对象，能调用它的所有方法、修改它的属性（包括私有属性）

这种动态获取信息，动态调用对象方法的功能，就是反射机制

通过反射，就将 Java 这类静态语言附加上了**动态特性**

那么什么是动态特性？

> p牛：一段代码，改变其中的变量，将会导致这段代码产生功能性的变化，我称之为动态特性。

举例说明

```java
public void execute(String className, String methodName) throws Exception {
    Class clazz = Class.forName(className);
    clazz.getMethod(methodName).invoke(clazz.newInstance());
}
```

这段代码中改变传入参数的值，将会调用不同类的不同方法

## forName 

forName 有两个函数重载

`Class<?> forName(String name)`

`Class<?> forName(String name, boolean initialize, ClassLoader loader)`

第一种相当于第二种的一个封装，等价于：

`Calss<?> forName(String name, true, currentLoader)`

即自动初始化，并根据类名加载类

关于初始化这个概念需要理解一下：

在 forName 时，类的构造函数并不会执行，这个初始化其实是“类初始化”

```java
public class Test {
    {
        System.out.println("Instance Initialization Block.");
    }
    
    static {
        System.out.println("Static Initialization Block.");
    }
    
    public Test() {
        System.out.println("Constructor.");
    }
}
```

三个块的执行顺序是：

- 类初始化时 `static {}` 先被调用
- `{}` 会在构造函数之前执行，因为他的代码会放在构造函数的 `super()` 后面，构造函数的内容前面

所以如果执行 `Test test = new Test();` ，将会输出：

```text
Static Initialization Block.
Instance Initialization Block.
Constructor.
```

而如果执行 `Class.forName()` ，将会输出

```text
Static Initialization Block.
```

`initialize` 设为 `false` 时，类不会初始化，无输出

所以对于上面的 `execute()` 函数，可以仅利用第一句，将恶意类的恶意代码放在 `static {}` 内，从而实现执行

```java
public void execute(String className, String methodName) throws Exception {
    Class clazz = Class.forName(className);
    // clazz.getMethod(methodName).invoke(clazz.newInstance());
}
```

## 内部类和单例

Java 的普通类 C1 中可以编写内部类 C2 ，在编译时生成两个文件： C1.class 和 C1$C2.class ，可通过 `Class.forName("C1$C2")` 加载内部类，然后可以使用 `newInstance()` 调用类的无参构造函数实例化这个类

有时候会遇到使用 `newInstance` 成功不了，原因可能是：

- 使用的类没有无参构造函数
- 使用的类的构造函数是私有的

例如， `java.lang.Runtime` ，执行以下代码将会报错

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.newInstance(), "id");
```

因为 `Runtime` 类的无参构造方法是私有的，我们可以看到， `Runtime` 类是典型的饿汉式单例：

```java
public class Runtime {
    private static Runtime currentRuntime = new Runtime();

    public static Runtime getRuntime() {
        return currentRuntime;
    }

    /** Don't let anyone else instantiate this class */
    private Runtime() {}
```

所以只能通过 `getRuntime()` 获取 `Runtime` 对象了，修改 payload

```java
Class clazz = Class.forName("java.lang.Runtime");
clazz.getMethod("exec", String.class).invoke(clazz.getMethod("getRuntime").invoke(clazz), "id");
```

> 设计模式之 “单例模式”
>
> 举例说明，对于 web 应用，数据库连接只需要建立一次，而不需要每次使用数据库时都重新连接，所以为了实现这样的目标，我们可以将数据库连接所使用的类设置为私有，然后编写静态方法来获取，从而实现：只有类初始化时执行一次构造函数，之后只允许通过 `getInstance()` 获取该对象，避免重复建立数据库连接
>
> 核心思想：
>
> - 唯一性：限制类只能被实例化一次
> - 全局访问：提供统一的访问入口，方便其他对象使用该实例
>
> 实现方式：
>
> 1. 饿汉式（Eager Initialization）
>
>    - 饿汉式
>
>      类加载时立即创建实例（线程安全）
>
>    ```java
>    public class Database {
>        private static final Database INSTANCE = new Database();
>        
>        private Database() {}
>        
>        public static Database getInstance() {
>            return INSTANCE;
>        }
>    }
>    ```
>
>    - Enum
>
>      防止反射和序列化破坏单例（线程安全）
>
>    ```java
>    public class Database {
>        enum Db {
>            INSTANCE;
>            private Db() {}
>        	public void connect() {
>            	// 数据库连接
>        	}
>        }
>    }
>    ```
>
> 2. 懒汉式（Lazy Initialization）
>
>    - 双重检验锁方式（Double-Checked Locking）
>
>      延迟实例化，首次调用时创建（需要处理线程安全）
>
>    ```java
>    public class Database {
>        private static volatile Database INSTANCE;
>                                
>        private Database() {}
>                                
>        public static Database getInstance() {
>            if (INSTANCE == null) {
>                synchronized (Database.class) {
>                    if (INSTANCE == null) {
>                        INSTANCE = new Database();
>                    }
>                }
>            }
>            return INSTANCE;
>        }
>    }
>    ```
>
>    - 静态内部类方式（Holder Class）
>
>      利用类加载机制保证线程安全，且延迟加载
>
>    ```java
>    public class Database {
>        private Database() {}
>                                
>        private static class Holder {
>            static final Database INSTANCE = new Database();
>        }
>                                
>        public static Database getInstance() {
>            return Holder.INSTANCE;
>        }
>    }
>    ```

---

回到主线，看一下这两种情况下怎么通过反射来实例化类？

### 一：没有无参构造函数

使用 `getConstructor` 方法获取构造函数，然后使用 `newInstance` 实例化

以 `ProcessBuilder` 为例

```java
public final class ProcessBuilder
{
    private List<String> command;
    private File directory;
    private Map<String,String> environment;
    private boolean redirectErrorStream;
    private Redirect[] redirects;

    public ProcessBuilder(List<String> command) {
        if (command == null)
            throw new NullPointerException();
        this.command = command;
    }

    public ProcessBuilder(String... command) {
        this.command = new ArrayList<>(command.length);
        for (String arg : command)
            this.command.add(arg);
    }

    public ProcessBuilder command(List<String> command) {
        if (command == null)
            throw new NullPointerException();
        this.command = command;
        return this;
    }

    public List<String> command() {
        return command;
    }
    
    // 略
}
```

对于第一个构造函数 `public ProcessBuilder(List<String> command) {}`

payload （强制类型转换）

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(List.class).newInstance(Arrays.asList("id"))).start();
```

payload （完全使用反射）

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(List.class).newInstance(Arrays.asList("id")));
```

对于第二个构造函数 `public ProcessBuilder(String... command) {}` 等价于 `public ProcessBuilder(String[] command) {}`

payload （强制类型转换）

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
((ProcessBuilder)clazz.getConstructor(String[].class).newInstance(new String[][]{{"id"}})).start();
```

payload （完全使用反射）

```java
Class clazz = Class.forName("java.lang.ProcessBuilder");
clazz.getMethod("start").invoke(clazz.getConstructor(String[].class).newInstance(new String[][]{{"id"}}));
```

### 二：构造函数私有

使用 `getDeclaredMethod` 和 `getDeclaredConstructor` 方法，或者先获取单例

以 `Runtime` 为例

payload （使用 `getDeclared`）

```java
Class clazz = Class.forName("java.lang.Runtime");
Constructor constructor = clazz.getDeclaredConstructor();
constructor.setAccessible(true); // 修改作用域
Process process = (Process) clazz.getMethod("exec", String.class).invoke(constructor.newInstance(), "id");
```

payload （获取单例，跟单例介绍前面是一样的）

```java
Object runtimeInstance = Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(null);
Class.forName("java.lang.Runtime").getMethod("exec", String.class).invoke(runtimeInstance, "id");
```

> 怎么确定命令是否执行成功？（以上面这两句为例）
>
> ```java
> // 可以打印结果，也可以调用一个可执行的程序，例如 calc.exe
> Process process = (Process) Class.forName("java.lang.Runtime").getMethod("exec", String.class).invoke(runtimeInstance, "calc");
> InputStream inputStream = process.getInputStream();
> BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
> String line;
> while ((line = reader.readLine()) != null) {
>     System.out.println(line);
> }
> ```

## 读取修改公有私有属性和被 `final` 或 `static final` 修饰的成员变量

```Java
package com.mingzux;

import java.lang.reflect.Field;
import java.lang.reflect.Modifier;

public class TestReflection {
    public static void main(String[] args) throws Exception {
        // 获取类和对象
        Class mz = SDUer.class;
        SDUer mingzu = (SDUer) mz.getDeclaredConstructor().newInstance();

        // 获取继承的 public 字段 - name
        Field name = mz.getField("name");
        System.out.println(name);
        // 读取 name 的值, 需要该类的对象, 而不是类
        System.out.println(name.get(mingzu));
        // 修改 name 的值, 需要该类的对象, 而不是类
        name.set(mingzu, "M!ng2u");
        System.out.println(name.get(mingzu));

        // 获取 private 字段 - sduId
        Field sduId = mz.getDeclaredField("sduId");
        System.out.println(sduId);
        // 读取 sduId 的值，同样需要对象
        sduId.setAccessible(true); // private 需要 setAccessible
        System.out.println(sduId.get(mingzu));
        // 修改 sduId 的值
        sduId.set(mingzu, "2333333");
        System.out.println(sduId.get(mingzu));

        // 获取继承的 protected 字段 - id
        Field id = mz.getSuperclass().getDeclaredField("id");
        System.out.println(id);
        // 读取 id 的值
        System.out.println(id.get(mingzu));
        // 修改 id 的值
        id.set(mingzu, "31415926535");
        System.out.println(id.get(mingzu));

        // 获取继承的被 final 修饰的字段 - species
        Field species = mz.getField("species");
        System.out.println(species);
        // 读取 species 的值
        System.out.println(species.get(mingzu));
        // 修改 species 的值
        // 获取 species 的 modifiers
        Field speciesModifiers = species.getClass().getDeclaredField("modifiers");
        System.out.println(speciesModifiers);
        // 读取 modifiers 的值
        speciesModifiers.setAccessible(true);
        System.out.println(speciesModifiers.get(species));
        // 将 modifiers 中的 final 去掉
        speciesModifiers.setInt(species, species.getModifiers() & ~Modifier.FINAL);
        System.out.println(speciesModifiers.get(species));
        // 这时候就可以修改 species 的值了
        species.setAccessible(true);
        species.set(mingzu, "Visitor");
        System.out.println(species.get(mingzu));

        // 获取被 static final 修饰的字段 - university
        Field university = mz.getDeclaredField("university");
        System.out.println(university);
        // 读取 university 的值
        university.setAccessible(true);
        System.out.println(university.get(mingzu));
        university.setAccessible(false); //这里不设 false 会寄, 可能是下面权限检查过不了
        // 修改 university 的值
        // 获取 university 的 modifiers
        Field universityModifiers = university.getClass().getDeclaredField("modifiers");
        System.out.println(universityModifiers);
        // 读取 modifiers 的值
        universityModifiers.setAccessible(true);
        System.out.println(universityModifiers.get(university));
        // 将 modifiers 中的 static final 去掉
        universityModifiers.setInt(university, university.getModifiers() & ~Modifier.FINAL);
        System.out.println(universityModifiers.get(university));
        // 修改 university 的值
        university.set(mingzu, "minihash");
        System.out.println(university.get(mingzu));

        System.out.println(mingzu.university); // 由于 JVM 内联优化, 包含 final 修饰的 university 的这句会被改成
        // System.out.println("SDU");
        // 所以只能用上面的 Field.get(Object obj) 方法获取
    }
}

class SDUer extends Person {
    public static final String university = "SDU";
    private String sduId = "202322171145";

    @Override
    public String toString() {
        return "SDUer [sduId=" + sduId + ", name=" + name + ", id=" + id + ", university=" + university + "]";
    }
}

class Person {
    public String name = "Mingzu";
    public String gender = "M";
    protected String id = "1145141919810";
    public final String species = "Human";
}

```

输出结果为

```txt
------------------------------------------------------
public java.lang.String com.mingzux.Person.name
Mingzu
M!ng2u
------------------------------------------------------
private java.lang.String com.mingzux.SDUer.sduId
202322171145
2333333
------------------------------------------------------
protected java.lang.String com.mingzux.Person.id
1145141919810
31415926535
------------------------------------------------------
public final java.lang.String com.mingzux.Person.species
Human
private int java.lang.reflect.Field.modifiers
17
1
Visitor
------------------------------------------------------
public static final java.lang.String com.mingzux.SDUer.university
SDU
private int java.lang.reflect.Field.modifiers
25
9
minihash
SDU
```

