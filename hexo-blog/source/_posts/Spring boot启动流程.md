---
title: Spring Boot 启动流程
date: 2018-08-10
tags: [Spring]
---

# Spring boot启动流程

参考文档：

- [SpringBoot启动流程解析](https://blog.csdn.net/q547550831/article/details/73441052)

- [Spring Boot启动流程详解（一）](https://www.cnblogs.com/exmyth/p/7128485.html)

## 启动流程
-  调用SpringApplication的run方法

````java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        // 程序启动入口
        // 启动嵌入式的 Tomcat 并初始化 Spring 环境及其各 Spring 组件
        SpringApplication.run(Application.class,args);
    }
}
````
-  执行SpringApplication的重载的run方法

````java
public static ConfigurableApplicationContext run(Object source, String... args) {
        return run(new Object[]{source}, args);
}
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
        return (new SpringApplication(sources)).run(args);
}
````

<!-- more -->

- 重载的run方法首先n根据传入的object，即Application.class，构造SpringApplication对象

````java
public SpringApplication(Object... sources) {
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = new HashSet();
        this.initialize(sources);
    }
````

- 构造的过程中对属性进行了操作，并根据传入的sources调用初始化方法

````java
private void initialize(Object[] sources) {
        if(sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
        }

        this.webEnvironment = this.deduceWebEnvironment();
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        this.mainApplicationClass = this.deduceMainApplicationClass();
}
````

- 初始化方法首先调用this.deduceWebEnvironment方法来判断是否是web环境

````java
private boolean deduceWebEnvironment() {
        String[] var1 = WEB_ENVIRONMENT_CLASSES;
        int var2 = var1.length;

        for(int var3 = 0; var3 < var2; ++var3) {
            String className = var1[var3];
            if(!ClassUtils.isPresent(className, (ClassLoader)null)) {
                return false;
            }
        }

        return true;
}

private static final String[] WEB_ENVIRONMENT_CLASSES = new String[]{"javax.servlet.Servlet", "org.springframework.web.context.ConfigurableWebApplicationContext"};
````

可以看到webEnvironment是一个boolean，该成员变量用来表示当前应用程序是不是一个Web应用程序。那么怎么决定当前应用程序是否Web应用程序呢，是通过在classpath中查看是否存在WEB_ENVIRONMENT_CLASSES这个数组中所包含的类，如果存在那么当前程序即是一个Web应用程序，反之则不然。

- 设置初始化构造器Initializers以及监听器Listeners，其中，两个set方法都用到了this.getSpringFactoriesInstances方法

````java
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
        return this.getSpringFactoriesInstances(type, new Class[0], new Object[0]);
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        LinkedHashSet names = new LinkedHashSet(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        List instances = this.createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
        AnnotationAwareOrderComparator.sort(instances);
        return instances;
}
````

可以看到，首先调用重载的方法，重载的方法首先获取当前线程的类加载器，然后用该类加载器加载传入的class类，其中SpringFactoriesLoader.loadFactoryNames代码如下

````java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();

        try {
            Enumeration ex = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
            ArrayList result = new ArrayList();

            while(ex.hasMoreElements()) {
                URL url = (URL)ex.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
                String factoryClassNames = properties.getProperty(factoryClassName);
                result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
            }

            return result;
        } catch (IOException var8) {
            throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
        }
}
````

可以简单的理解为去META-INF/spring.factories下面查找类名并加载。最后回到getSpringFactoriesInstances方法，执行this.createSpringFactoriesInstances，创建spring工厂实例

- 回到初始化initialize方法，来看this.deduceMainApplicationClass方法代码

````java
private Class<?> deduceMainApplicationClass() {
        try {
            StackTraceElement[] stackTrace = (new RuntimeException()).getStackTrace();
            StackTraceElement[] var2 = stackTrace;
            int var3 = stackTrace.length;

            for(int var4 = 0; var4 < var3; ++var4) {
                StackTraceElement stackTraceElement = var2[var4];
                if("main".equals(stackTraceElement.getMethodName())) {
                    return Class.forName(stackTraceElement.getClassName());
                }
            }
        } catch (ClassNotFoundException var6) {
            ;
        }

        return null;
}
````

通过获取当前栈，来获取main方法的入口，简单的说就是获取Application类，并将这个类赋值给this.mainApplicationClass

- 初始化完成之后，我们来到run方法

````java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Object analyzers = null;
        this.configureHeadlessProperty();
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        try {
            DefaultApplicationArguments ex = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, ex);
            Banner printedBanner = this.printBanner(environment);
            context = this.createApplicationContext();
            new FailureAnalyzers(context);
            this.prepareContext(context, environment, listeners, ex, printedBanner);
            this.refreshContext(context);
            this.afterRefresh(context, ex);
            listeners.finished(context, (Throwable)null);
            stopWatch.stop();
            if(this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, listeners, (FailureAnalyzers)analyzers, var9);
            throw new IllegalStateException(var9);
        }
}
````

主要是用来监测程序执行时间，可以理解为System.out.println(endTime - startTime)。

- 设置headless模式

```java
private void configureHeadlessProperty() {
    System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, System.getProperty(
            SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```

实际上就是设置headless属性为true，没有图形化界面的意思，对于headless有兴趣的可以自行google

- 获取监听器

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
        Class[] types = new Class[]{SpringApplication.class, String[].class};
        return new SpringApplicationRunListeners(logger, this.getSpringFactoriesInstances(SpringApplicationRunListener.class, types, new Object[]{this, args}));
}
```

可以看到调用的是之前介绍过的this.getSpringFactoriesInstances方法，通过查找spring.factories配置，加载监听器，随后调用starting方法启动监听器。

- 根据SpringApplicationRunListeners以及参数来准备环境变量

```java
DefaultApplicationArguments(String[] args) {
    Assert.notNull(args, "Args must not be null");
    this.source = new Source(args);
    this.args = args;
}

private static class Source extends SimpleCommandLinePropertySource {

    Source(String[] args) {
        super(args);
    }
    ...
}


public SimpleCommandLinePropertySource(String... args) {
    super(new SimpleCommandLineArgsParser().parse(args));
}

private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        this.configureEnvironment(environment, applicationArguments.getSourceArgs());
        listeners.environmentPrepared(environment);
        if(this.isWebEnvironment(environment) && !this.webEnvironment) {
            environment = this.convertToStandardEnvironment(environment);
        }

        return environment;
 }
 
 private ConfigurableEnvironment getOrCreateEnvironment() {
        return (ConfigurableEnvironment)(this.environment != null?this.environment:(this.webEnvironment?new StandardServletEnvironment():new StandardEnvironment()));
}

```

由于我们的run方法传入的args为空，所以DefaultApplicationArguments没有实际内容。在准备环境过程中，会首先获取或者创建一个环境，environment=true，返回StandardServletEnvironment。

- 通过方法this.configureEnvironment来配置环境

```java
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
        this.configurePropertySources(environment, args);
        this.configureProfiles(environment, args);
}

protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
        MutablePropertySources sources = environment.getPropertySources();
        if(this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
            sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
        }

        if(this.addCommandLineProperties && args.length > 0) {
            String name = "commandLineArgs";
            if(sources.contains(name)) {
                PropertySource source = sources.get(name);
                CompositePropertySource composite = new CompositePropertySource(name);
                composite.addPropertySource(new SimpleCommandLinePropertySource(name + "-" + args.hashCode(), args));
                composite.addPropertySource(source);
                sources.replace(name, composite);
            } else {
                sources.addFirst(new SimpleCommandLinePropertySource(args));
            }
        }

}

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
        environment.getActiveProfiles();
        LinkedHashSet profiles = new LinkedHashSet(this.additionalProfiles);
        profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
        environment.setActiveProfiles((String[])profiles.toArray(new String[profiles.size()]));
}
```

这里有两步，1.配置Property Sources。2.配置Profiles。然后调用listeners.environmentPrepared来准备环境，spring boot启动流程中，监听器最为复杂，这边不具体展开。Spring Application的Environment代表着程序运行的环境，主要包含了两种信息，一种是profiles，用来描述哪些bean definitions是可用的；一种是properties，用来描述系统的配置，其来源可能是配置文件、JVM属性文件、操作系统环境变量等等。configurePropertySources首先查看SpringApplication对象的成员变量defaultProperties，如果该变量非null且内容非空，则将其加入到Environment的PropertySource列表的最后。然后查看SpringApplication对象的成员变量addCommandLineProperties和main函数的参数args，如果设置了addCommandLineProperties=true，且args个数大于0，那么就构造一个由main函数的参数组成的PropertySource放到Environment的PropertySource列表的最前面(这就能保证，我们通过main函数的参数来做的配置是最优先的，可以覆盖其他配置）。在我们的例子中，由于没有配置defaultProperties且main函数的参数args个数为0，所以这个函数什么也不做。

configureProfiles首先会读取Properties中key为spring.profiles.active的配置项，配置到Environment，然后再将SpringApplication对象的成员变量additionalProfiles加入到Environment的active profiles配置中。在我们的例子中，配置文件里没有spring.profiles.active的配置项，而SpringApplication对象的成员变量additionalProfiles也是一个空的集合，所以这个函数没有配置任何active profile。

- 调用this.printBanner方法打印bunner

````java
private Banner printBanner(ConfigurableEnvironment environment) {
        if(this.bannerMode == Mode.OFF) {
            return null;
        } else {
            Object resourceLoader = this.resourceLoader != null?this.resourceLoader:new DefaultResourceLoader(this.getClassLoader());
            SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter((ResourceLoader)resourceLoader, this.banner);
            return this.bannerMode == Mode.LOG?bannerPrinter.print(environment, this.mainApplicationClass, logger):bannerPrinter.print(environment, this.mainApplicationClass, System.out);
        }
}
````

如果mode为off，则返回null，如果为log，则打印到log中，不然就输出。

- 创建上下文

````java
protected ConfigurableApplicationContext createApplicationContext() {
        Class contextClass = this.applicationContextClass;
        if(contextClass == null) {
            try {
                contextClass = Class.forName(this.webEnvironment?"org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext":"org.springframework.context.annotation.AnnotationConfigApplicationContext");
            } catch (ClassNotFoundException var3) {
                throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
            }
        }

        return (ConfigurableApplicationContext)BeanUtils.instantiate(contextClass);
}
````

如果是web环境，上下文类名为AnnotationConfigEmbeddedWebApplicationContext，否则为AnnotationConfigApplicationContext，然后通过全类名反射创建上下文。

- 注册异常分析器

```java

public FailureAnalyzers(ConfigurableApplicationContext context) {
    this(context, (ClassLoader)null);
}

FailureAnalyzers(ConfigurableApplicationContext context, ClassLoader classLoader) {
    Assert.notNull(context, "Context must not be null");
    this.classLoader = classLoader == null?context.getClassLoader():classLoader;
    this.analyzers = this.loadFailureAnalyzers(this.classLoader);
    this.prepareFailureAnalyzers(this.analyzers, context);
}

private List<FailureAnalyzer> loadFailureAnalyzers(ClassLoader classLoader) {
        List analyzerNames = SpringFactoriesLoader.loadFactoryNames(FailureAnalyzer.class, classLoader);
        ArrayList analyzers = new ArrayList();
        Iterator var4 = analyzerNames.iterator();

        while(var4.hasNext()) {
            String analyzerName = (String)var4.next();

            try {
                Constructor ex = ClassUtils.forName(analyzerName, classLoader).getDeclaredConstructor(new Class[0]);
                ReflectionUtils.makeAccessible(ex);
                analyzers.add((FailureAnalyzer)ex.newInstance(new Object[0]));
            } catch (Throwable var7) {
                logger.trace("Failed to load " + analyzerName, var7);
            }
        }

        AnnotationAwareOrderComparator.sort(analyzers);
        return analyzers;
}

private void prepareFailureAnalyzers(List<FailureAnalyzer> analyzers, ConfigurableApplicationContext context) {
    Iterator var3 = analyzers.iterator();

    while(var3.hasNext()) {
        FailureAnalyzer analyzer = (FailureAnalyzer)var3.next();
        this.prepareAnalyzer(context, analyzer);
    }

}

private void prepareAnalyzer(ConfigurableApplicationContext context, FailureAnalyzer analyzer) {
     if(analyzer instanceof BeanFactoryAware) {
            ((BeanFactoryAware)analyzer).setBeanFactory(context.getBeanFactory());
     }

}
```

可以看到，loadFailureAnalyzers方法同样使用了SpringFactoriesLoader.loadFactoryNames方法去获取analyzerNames，即去spring.factories文件中查找全类名，然后通过反射创建实例。并放入bean工厂。

- 准备上下文,为ApplicationContext加载environment，之后逐个执行ApplicationContextInitializer的initialize()方法来进一步封装ApplicationContext，
并调用所有的SpringApplicationRunListener的contextPrepared()方法，【EventPublishingRunListener只提供了一个空的contextPrepared()方法】，之后初始化IoC容器，并调用SpringApplicationRunListener的contextLoaded()方法，广播ApplicationContext的IoC加载完成，这里就包括通过**@EnableAutoConfiguration**导入的各种自动配置类。


```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    this.postProcessApplicationContext(context);
    this.applyInitializers(context);
    listeners.contextPrepared(context);
    if(this.logStartupInfo) {
        this.logStartupInfo(context.getParent() == null);
        this.logStartupProfileInfo(context);
    }

    context.getBeanFactory().registerSingleton("springApplicationArguments", applicationArguments);
    if(printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    Set sources = this.getSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    this.load(context, sources.toArray(new Object[sources.size()]));
    listeners.contextLoaded(context);
}

protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    if(this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton("org.springframework.context.annotation.internalConfigurationBeanNameGenerator", this.beanNameGenerator);
    }

    if(this.resourceLoader != null) {
        if(context instanceof GenericApplicationContext) {
            ((GenericApplicationContext)context).setResourceLoader(this.resourceLoader);
        }

        if(context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader)context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }

}

protected void applyInitializers(ConfigurableApplicationContext context) {
    Iterator var2 = this.getInitializers().iterator();

    while(var2.hasNext()) {
        ApplicationContextInitializer initializer = (ApplicationContextInitializer)var2.next();
        Class requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }

}


```

- 刷新上下文

````java
private void refreshContext(ConfigurableApplicationContext context) {
    this.refresh(context);
    if(this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        } catch (AccessControlException var3) {
            ;
        }
    }

}
````

- 遍历所有注册的ApplicationRunner和CommandLineRunner，并执行其run()方法。该过程可以理解为是SpringBoot完成ApplicationContext初始化前的最后一步工作，我们可以实现自己的ApplicationRunner或者CommandLineRunner，来对SpringBoot的启动过程进行扩展。

````java
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {
    this.callRunners(context, args);
}

private void callRunners(ApplicationContext context, ApplicationArguments args) {
    ArrayList runners = new ArrayList();
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    AnnotationAwareOrderComparator.sort(runners);
    Iterator var4 = (new LinkedHashSet(runners)).iterator();

    while(var4.hasNext()) {
        Object runner = var4.next();
        if(runner instanceof ApplicationRunner) {
            this.callRunner((ApplicationRunner)runner, args);
        }

        if(runner instanceof CommandLineRunner) {
            this.callRunner((CommandLineRunner)runner, args);
        }
    }

}

private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
    try {
        runner.run(args);
    } catch (Exception var4) {
        throw new IllegalStateException("Failed to execute ApplicationRunner", var4);
    }
}

````

- 最后调用监听器的finished方法，结束整个流程

















