---
title: Java SPI解读
date: 2019-03-07
tags: [源码]
---
# Java SPI解读

## SPI概念
Service Provider Interface，用于扩展机制应用,有很多的SPI扩展机制应用的实例，比如common-logging，JDBC等等,并且dubbo基于jdk的SPI机制，实现了dubbo自己的SPI


<!-- more -->


## 例子代码
### 接口
/src/main/java/com.wuj.demo.ICar

````java
package com.wuj.demo;

public interface ICar {

    public void run();
}

````
### 实现类

/src/main/java/com.wuj.demo.MotoCar

````java
package com.wuj.demo;

public class MotoCar implements ICar {

    @Override
    public void run(){
        System.out.println("two wheels");
    }

}
````

/src/main/java/com.wuj.demo.PoloCar

````java
package com.wuj.demo;

public class PoloCar implements ICar {

    @Override
    public void run(){
        System.out.println("four wheels");
    }
}
````

### 测试类

/src/main/java/com.wuj.demo.SPITest

````java
package com.wuj.demo;

import java.util.Iterator;
import java.util.ServiceLoader;

public class SPITest {


    public static void main(String[] args) {

        ServiceLoader<ICar> spiLoader = ServiceLoader.load(ICar.class);
        Iterator<ICar> iaIterator = spiLoader.iterator();
        while (iaIterator.hasNext()) {
            iaIterator.next().run();
        }

    }

}

````

### 配置文件

/src/main/resource/META-INF/services/com.wuj.demo.ICar

````java
com.wuj.demo.MotoCar
com.wuj.demo.PoloCar
````

### 输出结果

````java
two wheels
four wheels
````

## Java SPI 流程解读

### 入口

````java
ServiceLoader<ICar> spiLoader = ServiceLoader.load(ICar.class);
````

### ServiceLoader的load方法

````java
public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
````
使用线程上下文的类加载器来创建ServiceLoader


### 带ClassLoader参数的load方法

````java
 public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }
````
为指定的服务使用指定的类加载器来创建一个ServiceLoader

### 类构造器

````java
private ServiceLoader(Class<S> svc, ClassLoader cl) {
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
````
1. 使用指定的类加载器和服务创建服务加载器。
2. 如果没有指定类加载器，使用系统类加载器，就是应用类加载器。
3. 此时，安全管理器没有被设置，所以获取到的acc为null
### reload 方法

````java
public void reload() {
		 //清空缓存中所有已实例化的服务提供者
        providers.clear();
        //新建一个迭代器，该迭代器会从头查找和实例化服务提供者
        lookupIterator = new LazyIterator(service, loader);
    }
````

用于新的服务提供者安装到正在运行的Java虚拟机中的情况。

### 到此，spiLoader已经创建好了，注意：这里没有去META-INF/services目录下查找配置文件，也没有加载具体实现类，做的事情就是封装了我们的接口类型和类加载器，并初始化了一个迭代器。

### iterator.hasNext()方法

````java
public boolean hasNext() {
                if (knownProviders.hasNext())
                    return true;
                return lookupIterator.hasNext();
            }
````
一开始由于没有去读取配置文件，所以knownProviders.hasNext()为false，走的是ServiceLoader里的迭代器的hasNext方法

### lookupIterator.hasNext()方法

````java
public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }
````
可见不管acc是否为null，都会走hasNextService()方法

### hasNextService()方法

````java
private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }
````
之前初始化的时候没有给nextName以及configs赋值，所以需要去读取资源文件。PREFIX的值为写死的值"META-INF/services/"。这里通过文件全名，查找文件。初始化时也并未给pending赋值，所以pending为null，走parse方法。

### parse()方法

````java
InputStream in = null;
        BufferedReader r = null;
        ArrayList<String> names = new ArrayList<>();
        try {
            in = u.openStream();
            r = new BufferedReader(new InputStreamReader(in, "utf-8"));
            int lc = 1;
            while ((lc = parseLine(service, u, r, lc, names)) >= 0);
        } catch (IOException x) {
            fail(service, "Error reading configuration file", x);
        } finally {
            try {
                if (r != null) r.close();
                if (in != null) in.close();
            } catch (IOException y) {
                fail(service, "Error closing configuration file", y);
            }
        }
        return names.iterator();
````
主要是parseLine方法，使用parseLine方法进行解析，未被实例化的服务提供者会被保存到缓存中去

### parseLine（）

````java
		String ln = r.readLine();
        if (ln == null) {
            return -1;
        }
        int ci = ln.indexOf('#');
        if (ci >= 0) ln = ln.substring(0, ci);
        ln = ln.trim();
        int n = ln.length();
        if (n != 0) {
            if ((ln.indexOf(' ') >= 0) || (ln.indexOf('\t') >= 0))
                fail(service, u, lc, "Illegal configuration-file syntax");
            int cp = ln.codePointAt(0);
            if (!Character.isJavaIdentifierStart(cp))
                fail(service, u, lc, "Illegal provider-class name: " + ln);
            for (int i = Character.charCount(cp); i < n; i += Character.charCount(cp)) {
                cp = ln.codePointAt(i);
                if (!Character.isJavaIdentifierPart(cp) && (cp != '.'))
                    fail(service, u, lc, "Illegal provider-class name: " + ln);
            }
            if (!providers.containsKey(ln) && !names.contains(ln))
                names.add(ln);
        }
        return lc + 1;
````
1. 解析服务提供者配置文件中的一行
2. 首先去掉注释校验，然后保存
3. 返回下一行行号
4. 重复的配置项和已经被实例化的配置项不会被保存

### 到这里，hasNextService就走完了，回到test类，接下去走spiLoader.iterator().next()方法

````java
public S next() {
                if (knownProviders.hasNext())
                    return knownProviders.next().getValue();
                return lookupIterator.next();
            }
````
providers并没有被赋值，所以走lookupIterator的next方法

### lookupIterator.next()

````java
public S next() {
            if (acc == null) {
                return nextService();
            } else {
                PrivilegedAction<S> action = new PrivilegedAction<S>() {
                    public S run() { return nextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
        }
````
这边不管acc是不是为null，都会走nextService方法

### nextService()

````java
private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
````
从这里可以看出来，最终还是用了Class.forName方法来加载类。

### 结论
利用java的SPI扩展，定义了标准，可以很好对程序进行扩展。封装好了class.forName方法，可以有效的减少代码中的硬编码，这样当以后替换了接口的实现的时候，只需要更换配置文件，而不需要去改动代码。





