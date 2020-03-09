```java

    private static final String configLocation = "applicationContext.xml";

    @Test
    public void beanTest() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
        System.out.println("user -> " + applicationContext.getBean("user"));
    }
```

在没有看源码之前，对于上面的例子一直以为是在执行 `applicationContext.getBean("user")` 时创建的 bean 实例，带着这种想法去看了 `getBean` 方法的内部实现，发现有很多地方串不起来，比如 `beanDefinition` 是什么时候初始化的，`beanPostProcessor` 是什么时候设置的，看的过程中一头雾水。

IoC 的源码也跟踪了一段时间，后来在 `ClassPathXmlApplicationContext` 的构造函数中找到了答案，在执行 `new ClassPathXmlApplicationContext(configLocation)` 时 IoC 容器其实已经创建完成了，并会初始化所有的非懒加载配置的单例 bean。下面是看源码过程中的一些记录，如果有不对的地方还望大家指正。由于启动过程中完整的代码非常多，这里只总结一些重要的流程，相关的细节有兴趣的自己 debug 。

### 1.ClassPathXmlApplicationContext 入口

一切都要从 `ClassPathXmlApplicationContext` 的构造函数说起。

```java
    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[] {configLocation}, true, null);
    }

    public ClassPathXmlApplicationContext(
            String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
            throws BeansException {

        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
            refresh();
        }
    }
```

`ClassPathXmlApplicationContext` 的构造函数主要做了 3 件事：

 1.设置父容器，这里为 null
 2.设置配置文件的位置
 3.调用 `resresh` 方法，启动容器

### 2.setConfigLocations

```java
    public void setConfigLocations(@Nullable String... locations) {
        if (locations != null) {
            Assert.noNullElements(locations, "Config locations must not be null");
            this.configLocations = new String[locations.length];
            for (int i = 0; i < locations.length; i++) {
                this.configLocations[i] = resolvePath(locations[i]).trim();
            }
        }
        else {
            this.configLocations = null;
        }
    }
```

遍历配置文件数组，将解析过的文件路径添加到 `configLocations` 数组中，后面初始化 `beanDefitnion` 中会用该该数组。


### 3.refresh 方法一览

```java
@Override
    public void refresh() throws BeansException, IllegalStateException {
        // 线程安全
        synchronized (this.startupShutdownMonitor) {
            // Prepare this context for refreshing.
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
            }

            finally {
                resetCommonCaches();
            }
        }
    }
```

上面简化了一些无用的代码，IoC 容器启动的流程就是通过 `refresh` 方法实现的，流程比较复杂，下面分步来总结。

### 4.prepareRefresh

```java
    protected void prepareRefresh() {
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);

        if (logger.isDebugEnabled()) {
            if (logger.isTraceEnabled()) {
                logger.trace("Refreshing " + this);
            } else {
                logger.debug("Refreshing " + getDisplayName());
            }
        }

        // Initialize any placeholder property sources in the context environment.
        ();

        // Validate that all properties marked as required are resolvable:
        // see ConfigurablePropertyResolver#setRequiredProperties
        getEnvironment().validateRequiredProperties();

        // Store pre-refresh ApplicationListeners...
        // 启动监听器，此时并未注册
        if (this.earlyApplicationListeners == null) {
            this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
        } else {
            // Reset local application listeners to pre-refresh state.
            this.applicationListeners.clear();
            this.applicationListeners.addAll(this.earlyApplicationListeners);
        }

        // Allow for the collection of early ApplicationEvents,
        // to be published once the multicaster is available...
        this.earlyApplicationEvents = new LinkedHashSet<>();
    }
```

1.设置启动信息：启动时间，活跃状态
2.初始化属性信息
3.验证必要的属性信息
4.初始化存储监听器的集合，此时并未有监听器注册

`initPropertySources` 中什么都没做，具体交给子类去实现，用来设置环境属性信息，接着会对属性信息进行校验。

### 5.obtainFreshBeanFactory

```java

    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        refreshBeanFactory();
        // 返回 beanFactory
        return getBeanFactory();
    }

    @Override
    protected final void refreshBeanFactory() throws BeansException {
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }

    @Override
    public final ConfigurableListableBeanFactory getBeanFactory() {
        synchronized (this.beanFactoryMonitor) {
            if (this.beanFactory == null) {
                throw new IllegalStateException("BeanFactory not initialized or already closed - " +
                        "call 'refresh' before accessing beans via the ApplicationContext");
            }
            return this.beanFactory;
        }
    }
```

`refreshBeanFactory` 方法首先调用了 `refreshBeanFactory` 用于刷新 `beanFactory`：

```java
    @Override
    protected final void refreshBeanFactory() throws BeansException {
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }
```

 1.如果存在 beanFactory 则销毁，并销毁创建的 bean，就是清空各种缓存 map
 2.创建 BeanFactory（DefaultListableBeanFactory）
 3.设置序列 ID，根据当前对象生成
 4.设置是否允许循环依赖与覆盖 beanDefinition，默认都是不允许的，set 方法没有调用
 5.加载 beanDefinition

 上面步骤中最重要的是 `loadBeanDefinitions` 方法，该方法用于将 Spring 的配置文件转成 `Document` 对象，然后解析 `Document` 将 XML 文件中的配置转化成 `BeanDefinition`，保存在 `beanDefinitionMap` 中，用于创建 bean 对象。`loadBeanDefinitions` 方法之前已经总结过，大家可以在以前的文章中找到。

### 6.prepareBeanFactory

```java
    protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // Tell the internal bean factory to use the context's class loader etc.
        // 设置类加载器
        beanFactory.setBeanClassLoader(getClassLoader());
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
        beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

        // Configure the bean factory with context callbacks.
        // 设置与取消对应 beanPostProcessor
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
        beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
        beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
        beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

        // BeanFactory interface not registered as resolvable type in a plain factory.
        // MessageSource registered (and found for autowiring) as a bean.
        beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
        beanFactory.registerResolvableDependency(ResourceLoader.class, this);
        beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
        beanFactory.registerResolvableDependency(ApplicationContext.class, this);

        // Register early post-processor for detecting inner beans as ApplicationListeners.
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

        // Detect a LoadTimeWeaver and prepare for weaving, if found.
        if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            // Set a temporary ClassLoader for type matching.
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        // Register default environment beans.
        // 注册默认的环境 bean 信息，比如：environment、systemProperties、systemEnvironment
        if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
        }
        if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
        }
        if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
        }
    }
```
