# SpringApplication 相关源码注释

`SpringApplication.run(SpringBootTestApplication.class, args);`方法主要是通过`new SpringApplication(null, SpringBootTestApplication.class).run(args);`实现的。

## SpringApplication的构造方法

构造一个SpringApplication类，为其初始化启动时所需的配置信息

```Java
private List<ApplicationContextInitializer<?>> initializers;
private List<ApplicationListener<?>> listeners;
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // resourceLoader一般为null，primarySources为SpringApplication.run中传入的第一个参数
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");?

    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 查询应用的类型，定义在WebApplicationType枚举类中，包括：NONE， SERVLET， REAACTIVE三种
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 从spring.factories工厂中分别获取Initializer组件类列表和Listener组件类列表，并对它们进行实例化之后配置到当前应用中
    // setInitializers(initializers) = this.initializers = new ArrayList<>(initializers);
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // setListeners(listeners) = this.listeners = new ArrayList<>(listeners);
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 获取main方法所在的启动类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

相关参考链接：

<!-- no toc -->
- [webapplicationtype枚举类的deducefromclasspath方法](#webapplicationtype枚举类的deducefromclasspath方法)
- [springapplication类的getspringfactoriesinstances方法](#springapplication类的getspringfactoriesinstances方法)
- [springapplication的deducemainapplicationclass方法](#springapplication的deducemainapplicationclass方法)

## SpringApplication类的run方法

```Java
public ConfigurableApplicationContext run(String... args) {
    // StopWatch（秒表）类用来监控启动任务的启动时长
    StopWatch stopWatch = new StopWatch();
    // 开启秒表，本质上是long startTimeNanos = System.nanoTime();
    stopWatch.start();
    // 初始化 可配置的应用上下文
    ConfigurableApplicationContext context = null;
    // 初始化 启动失败的分析报告器
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // 配置是否使用headless模式（系统的一种配置模式。在该模式下，系统缺少了显示设备、键盘或鼠标，一般是在程序开始激活headless模式，告诉程序，现在你要工作在Headless  mode下，就不要指望硬件帮忙了，你得自力更生，依靠系统的计算能力模拟出这些特性来），即System.setProperty("java.awt.headless", System.getProperty("java.awt.headless", true));
    configureHeadlessProperty();
    // 获取类型为SpringApplicationRunListener，构造参数类型为SpringApplication.class和String[].class的全部实例，并封装为SpringApplicationRunListeners类
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 调用所有的SpringApplicationRunListener实例的starting方法，启动其监听
    listeners.starting();
    try {
        // 通过启动参数生成应用参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备可配置的环境，在ApplicationContext中使用该环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 从环境中获取spring.beaninfo.ignore并配置
        configureIgnoreBeanInfo(environment);
        // 打印横幅
        Banner printedBanner = printBanner(environment);
        // 创建ApplicationContext
        context = createApplicationContext();
        // 从spring。factories中获取失败分析报告器，用于失败时详细分析失败原因并报告
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        // 准备ApplicationContext，包括设置环境、注册一些必要的组件等
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 执行context的refresh方法，初始化启动ApplicationContext
        refreshContext(context);
        // 刷新之后执行一些方法
        afterRefresh(context, applicationArguments);
        // 关闭秒表，本质上是long lastTime = System.nanoTime() - this.startTimeNanos;
        stopWatch.stop();
        if (this.logStartupInfo) {
            // 打印启动日志
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 调用启动完成的事件回调
        listeners.started(context);
        // 从应用上下文中获取ApplicationRunner和CommandLineRunner类型的Bean并调用它们的run方法
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        // 失败时调用必要的逻辑，失败分析报告器就用在这里
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 正常启动，调用运行中事件回调
        listeners.running(context);
    }
    catch (Throwable ex) {
        // 失败时调用必要的逻辑
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    // 返回创建的ApplicationContext
    return context;
}
```

<!-- no toc -->
- [springapplication类的createapplicationcontext方法](#springapplication类的createapplicationcontext方法)
- [SpringApplication类的prepareEnvironment方法](#SpringApplication类的prepareEnvironment方法)
- [springapplication类的refreshcontext方法](#springapplication类的refreshcontext方法)

## SpringApplication类的prepareEnvironment方法

主要完成以下功能：

- 通过`System.getProperties()`获取JVM相关变量，通过Java启动参数`-Dkey=name`指定。
- 通过`System.getenv()`获取当前系统的环境变量，Linux通过`export key=value`指定；Windows通过`set key=value`指定
- 通过命令行参数获取，使用应用启动参数`-key=value`来配置
- 通过`resources`或者`resources/config`目录下的配置文件`application.yml`或者`application.properties`指定。

```Java
/**
 * 准备应用环境
 * @param listeners Spring应用的启动监听器，用于处理一些事件回调
 * @param applicationArguments 启动时传入的启动参数的封装，一般以“--参数名=参数值”的方式配置参数
 * @return 返回可配置的应用环境
 */
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 根据情况获取或者创建环境。SpringApplication构造时可以主动指定环境。
    // 如果是自动获取的环境，则会根据当前Web类型返回不同的环境。SERVLET环境返回StandardServletEnvironment，SERVLET环境返回StandardReactiveWebEnvironment，其他返回StandardEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置引用环境，添加一些默认的PropertySource属性源，尝试通过属性源获取活动的profiles
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // attach一些属性源到环境中
    ConfigurationPropertySources.attach(environment);
    // 调用全部监听器的环境已准备事件
    listeners.environmentPrepared(environment);
    // 环境初始化完毕，绑定环境中的一些属性到当前SpringApplication中
    bindToSpringApplication(environment);
    // 如果当前不是Web环境，则把环境做转换
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    // attach一些属性源到环境中
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

## SpringApplication类的createApplicationContext方法

根据创建SpringApplication的应用上下文实例对象

```Java
// 基于注解配置的Servlet Web服务应用上下文
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext";
// 基于注解配置的Reactive Web服务应用上下文
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";
// 基于注解配置的非Web应用上下文
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context.annotation.AnnotationConfigApplicationContext";
// 根据环境创建对应类型的ApplicationContext
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    // 如果当前配置中的applicationContextClass为空，则根据环境自动获取，否则直接以配置为准
    if (contextClass == null) {
        try {
            // 根据Web类型返回不同的ApplicationContext类型
            switch (this.webApplicationType) {
            case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
        }
    }
    // 实例化applicationContext类，并返回
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

## SpringApplication类的方法prepareContext方法

```Java
/*
 * 准备应用上下文
 * @param context 创建好的可配置应用上下文
 * @param environment 创建好的可配置环境
 * @param listeners Spring应用的启动监听器，用于处理一些事件回调
 * @param applicationArguments 启动时传入的启动参数的封装，一般以“--参数名=参数值”的方式配置参数
 * @param printedBanner 可打印的启动横幅，可以按照自己的喜好修改
 */
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 把环境设置到应用上下文中
    context.setEnvironment(environment);
    // 设置完做一些处理
    postProcessApplicationContext(context);
    // 调用应用初始化组件，这些组件在SpringApplication的构造方法中通过setInitializers方法指定的。本质上是遍历全部initializer，并调用其initialize方法
    applyInitializers(context);
    // 调用监听器的上下文已准备事件
    listeners.contextPrepared(context);
    // 打印启动日志
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    // 直接注册特殊类型的启动参数单例到BeanFactory中，通过这种方式不会触发实例化和销毁bean的回调函数
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        // 直接注册可打印类型的Banner单例到BeanFactory中，通过这种方式不会触发实例化和销毁bean的回调函数
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        // 设置是否允许注册同名bean，并自动覆盖，默认为true，即允许，如果为false，则当注册同名bean时，会抛出异常
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) { // lazyInitialization默认为false
        // 添加延迟初始化Bean工厂后处理器
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // Load the sources
    // 加载并获取当前应用的配置资源,本质上是[SpringBootTestApplication.class]
    Set<Object> sources = getAllSources();
    // 保证资源不为空
    Assert.notEmpty(sources, "Sources must not be empty");
    // 调用BeanDefinitionLoader的loader方法，加载配置资源到应用上下文中
    load(context, sources.toArray(new Object[0]));
    // 调用监听器的上下文已加载回调事件
    listeners.contextLoaded(context);
}
```

## SpringApplication类的refreshContext方法

```Java
private boolean registerShutdownHook = true;
private void refreshContext(ConfigurableApplicationContext context) {
    // 最终调用父类ConfigurableApplicationContext的refresh方法
    refresh((ApplicationContext) context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}
```

## SpringApplication类的getSpringFactoriesInstances方法

从spring.factories配置文件中获取有type类型的实例对象集合

```Java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    // 从spring.factories工厂中获取所有type类型的实例对象
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    // 获取当前上下文加载器，用于Classpath下的加载资源文件
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 调用SpringFactoriesLoader.loadFactoryNames获取指定类的所有类型名，使用Set保证没有重复类型名
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 通过调用createSpringFactoriesInstances将names生成对应的已初始化的类实例列表
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 通过AnnotationAwareOrderComparator排序规则（依据@Priority、@Ordear注解和PriorityOrdeared、Ordered接口）对instances中的实例进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

相关参考链接：

<!-- no toc -->
- [SpringFactoriesLoader的loadFactoryNames方法](#SpringFactoriesLoader的loadFactoryNames方法)
- [springapplication类的createspringfactoriesinstances方法](#springapplication类的createspringfactoriesinstances方法)

## SpringApplication类的createSpringFactoriesInstances方法

使用classLoader生成names中的类名对应类实例，并使用args进行相应的初始化

```Java
@SuppressWarnings("unchecked")
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
        ClassLoader classLoader, Object[] args, Set<String> names) {
    // instances是相应的实例对象列表
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            // 通过类名加载类实例
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            // 断言instanceClass是否是type的子类
            Assert.isAssignable(type, instanceClass);
            // 获取其相应参数的构造器方法，并进行初始化，然后将该实例加入instances列表
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

## SpringApplication的deduceMainApplicationClass方法

用来推断main方法所在的启动类

```Java
private Class<?> deduceMainApplicationClass() {
    try {
        // 通过创建一个异常，获取当前方法执行的调用栈信息
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            // 从底到顶遍历栈，找到方法名是main的栈，尝试推断该栈中的Class就是启动类
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```

## ServletWebServerApplicationContext类的refresh方法

Spring的Servlet环境下，其准备的上下文是ServletWebServerApplicationContext类的子类。调用其子类refresh方法时，调用的是从ServletWebServerApplicationContext类的refresh方法。该方法负责应用上下文的刷新，从而启动整个Spring应用

```Java
@Override
public final void refresh() throws BeansException, IllegalStateException {
    try {
        super.refresh();
    }
    catch (RuntimeException ex) {
        WebServer webServer = this.webServer;
        if (webServer != null) {
            webServer.stop();
        }
        throw ex;
    }
}
```

## WebApplicationType枚举类的deduceFromClasspath方法

该方法主要实现应用类型的判断，其通过查找上下文中有没有特定类实现的

```Java
private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";
private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";
private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";
private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet", "org.springframework.web.context.ConfigurableWebApplicationContext" };
static WebApplicationType deduceFromClasspath() {
    // 如果只存在Reactive相关类，则判断为Reactive类型
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    // 如果不存在任意一个Web相关环境的依赖，则判断为非Web环境
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    // 否则都是Servlet环境
    return WebApplicationType.SERVLET;
}
```

## SpringFactoriesLoader的loadFactoryNames方法

查找到classLoader类加载器下factoryType工厂类型对应的全部工厂实现类的类名集合

```Java
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
// loadSpringFactories方法得到给定类加载器下的全部spring.factories文件中定义的工厂类和实现类名的映射表。可能不同的spring的jar包其spring.factories文件中配置了同一个类，则最终结果会存在：相应的工厂类型其实现类列表有重复的情况
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 先从缓存中查找相应的数据，没有再加载
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // 使用默认类加载器或者给定的类加载器，获取全部名为FACTORIES_RESOURCE_LOCATION（即spring.factories）的文件的URL类
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            // 根据URL创建相应的Properties，并读取其中相应的工厂类型和实现类映射表
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        // 做缓存，下次调用时直接使用缓存中的输入
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```
