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

 1. 设置父容器，这里为 null
 2. 设置配置文件的位置
 3. 调用 `resresh` 方法，启动容器

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

 1. 设置启动信息：启动时间，活跃状态
 2. 初始化属性信息
 3. 验证必要的属性信息
 4. 初始化存储监听器的集合，此时并未有监听器注册

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

 1. 如果存在 `beanFactory` 则销毁，并销毁创建的 bean，就是清空各种缓存 map
 2. 创建 `BeanFactory（DefaultListableBeanFactory）`
 3. 设置序列 ID，根据当前对象生成
 4. 设置是否允许循环依赖与覆盖 `beanDefinition`，默认都是不允许的，set 方法没有调用
 5. 加载 `beanDefinition`

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

 1. 1.设置 bean 的类加载器，解析器
 2. 设置与排除对应 `beanPostProcessor`，这里设置了 `ApplicationContextAwareProcessor`、`ApplicationListenerDetector`，`ApplicationContextAwareProcessor` 会处理对应的 ...Aware 接口，可以联想项目中经常使用类继承 `ApplicationAware`，然后获取到 `ApplicationContext`，这个 `ApplicationContext` 就是通过 `BeanPostProcessor` 实现的 
 3. 设置依赖信息，TODO 没看懂
 4. 注册默认的环境 bean 信息，比如：`environment`、`systemProperties`、`systemEnvironment`

在项目里我们经常通过继承 `ApplicationContextAware` 来获取到 `aplicationContext`，从而创建一些辅助 bean 来帮助我们实现一些架构设计，这个 `applicationContext` 就是通过 `BeanPostProcessor` 完成配置的，`BeanPostProcessor` 相关内容我会在后续的文章里讲到。下面我们一起来简单的看一下 `ApplicationContextAwareProcessor` 中的 `postProcessBeforeInitialization` 方法。

```java
    public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
        AccessControlContext acc = null;

        if (System.getSecurityManager() != null &&
                (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
                        bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
                        bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
            acc = this.applicationContext.getBeanFactory().getAccessControlContext();
        }

        if (acc != null) {
            AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                invokeAwareInterfaces(bean);
                return null;
            }, acc);
        }
        else {
            invokeAwareInterfaces(bean);
        }

        return bean;
    }

    private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof EnvironmentAware) {
                ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
            }
            if (bean instanceof EmbeddedValueResolverAware) {
                ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
            }
            if (bean instanceof ResourceLoaderAware) {
                ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
            }
            if (bean instanceof ApplicationEventPublisherAware) {
                ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
            }
            if (bean instanceof MessageSourceAware) {
                ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
            }
            // 设置 applicationContext
            if (bean instanceof ApplicationContextAware) {
                ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
            }
        }
    }
```

### 7.postProcessBeanFactory

```java
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    }
```

这个方法没有任何操作，当代码执行到这里的时候 `beanDefinition` 已经初始化完成，而 bean 实例还没有创建，在这个方法中你可以设置 `BeanPostProcessors` 来达到某种效果。比如在 `StaticWebApplicationContext` 实现类中会设置 `ServletContextAwareProcessor` 来完成 web 相关的一些配置。


### 8.invokeBeanFactoryPostProcessors


```java
    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {

        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

        // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
        // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
    }

    /**
     * 按次序执行仅限于 BeanDefinitionRegistryPostProcessor，beanFactoryPostProcessors 不会按类型区分，而是一次全部执行
     * 主意顺序，先 BeanDefinitionRegistryPostProcessor 后 beanFactoryPostProcessors
     *
     * 1.遍历 beanFactoryPostProcessors，BeanDefinitionRegistryPostProcessor 与 BeanFactoryPostProcessor 区分类别后加入到不同集合
     * 2.找出 BeanDefinitionRegistryPostProcessor 是 PriorityOrdered 的，依次执行
     * 3.找出 BeanDefinitionRegistryPostProcessor 是 Ordered 的，依次执行
     * 4.执行剩下的 BeanDefinitionRegistryPostProcessor 与所有 beanFactoryPostProcessors
     */
    public static void invokeBeanFactoryPostProcessors(
            ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

        // Invoke BeanDefinitionRegistryPostProcessors first, if any.
        Set<String> processedBeans = new HashSet<>();

        if (beanFactory instanceof BeanDefinitionRegistry) {
            BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
            // bean 处理器
            List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
            // beanDefinition 处理器
            List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

            for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
                if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                    BeanDefinitionRegistryPostProcessor registryProcessor =
                            (BeanDefinitionRegistryPostProcessor) postProcessor;
                    registryProcessor.postProcessBeanDefinitionRegistry(registry);
                    registryProcessors.add(registryProcessor);
                }
                else {
                    regularPostProcessors.add(postProcessor);
                }
            }

            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the bean factory post-processors apply to them!
            // Separate between BeanDefinitionRegistryPostProcessors that implement
            // PriorityOrdered, Ordered, and the rest.
            List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

            // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
            String[] postProcessorNames =
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            // 找出所有实现 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessor，按顺序执行
            for (String ppName : postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            // 排序
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            // 排好序后加入到 registryProcessors 集合，重复添加？？
            registryProcessors.addAll(currentRegistryProcessors);
            // BeanDefinitionRegistryPostProcessor 根据排序依次执行
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            // 清空已经排序的 BeanDefinitionRegistryPostProcessor 列表
            currentRegistryProcessors.clear();

            // 实现 Ordered 的依次执行
            // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();

            // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
            // 剩余的依次执行
            boolean reiterate = true;
            while (reiterate) {
                reiterate = false;
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                for (String ppName : postProcessorNames) {
                    // processedBeans 不包含说明不是 Ordered 与 PriorityOrdered 类型，因此没有执行
                    if (!processedBeans.contains(ppName)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                        reiterate = true;
                    }
                }
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
            }

            // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
            invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
        }

        else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

        // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
        // Ordered, and the rest.
        List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
        List<String> orderedPostProcessorNames = new ArrayList<>();
        List<String> nonOrderedPostProcessorNames = new ArrayList<>();
        for (String ppName : postProcessorNames) {
            if (processedBeans.contains(ppName)) {
                // skip - already processed in first phase above
            }
            else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            }
            else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }

        // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
        sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

        // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
        List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : orderedPostProcessorNames) {
            orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        sortPostProcessors(orderedPostProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

        // Finally, invoke all other BeanFactoryPostProcessors.
        List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : nonOrderedPostProcessorNames) {
            nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

        // Clear cached merged bean definitions since the post-processors might have
        // modified the original metadata, e.g. replacing placeholders in values...
        beanFactory.clearMetadataCache();
    }
```

上面的代码很长，其实执行的逻辑很简单，主要用于执行 `BeanDefinitionRegistryPostProcessor` 与 `BeanFactoryPostProcessor`，注意不是 `BeanPostProcessor`。看源码一时间可能不知道这些 `proseccor` 到底有什么用，下面我配置了三种类型的 `processor`，通过一个例子我们直观的来了解一下。

```java
    @Test
    public void beanTest() {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
        System.out.println("user -> " + applicationContext.getBean("user"));
    }
```

![](https://raw.githubusercontent.com/zchen96/spring-framework-5.1.7-source-code-read/master/image/ioc/processors.png)

 - `BeanDefinitionRegistryPostProcessor`： 在 `beanDefinition` 注册之后执行
 - `BeanFactoryPostProcessor`：在 `BeanDefinitionRegistryPostProcessor` 执行完成之后执行
 - `BeanPostProcessor`：在 bean 创建之前执行

### 9.registerBeanPostProcessors

```java
    protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
    }
```

这一个过程和上一个过程很像，不过有一个大的差别是 `registerBeanPostProcessors` 方法是发现并注册所有的 `BeanPostProcessor` 并不会执行。细节代码一大堆这里就不贴出来了。

### 10.initMessageSource

```java
    protected void initMessageSource() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
            this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
            // Make MessageSource aware of parent MessageSource.
            if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
                HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
                if (hms.getParentMessageSource() == null) {
                    // Only set parent context as parent MessageSource if no parent MessageSource
                    // registered already.
                    hms.setParentMessageSource(getInternalParentMessageSource());
                }
            }
            if (logger.isTraceEnabled()) {
                logger.trace("Using MessageSource [" + this.messageSource + "]");
            }
        } else {
            // Use empty MessageSource to be able to accept getMessage calls.
            DelegatingMessageSource dms = new DelegatingMessageSource();
            dms.setParentMessageSource(getInternalParentMessageSource());
            this.messageSource = dms;
            beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
            if (logger.isTraceEnabled()) {
                logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
            }
        }
    }
```

`initMessageSource` 方法用于给容器初始化一个 `messageSource` 用来支持一些国际化的操作。

### 11.initApplicationEventMulticaster

```java
    protected void initApplicationEventMulticaster() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            this.applicationEventMulticaster =
                    beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
            if (logger.isTraceEnabled()) {
                logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
            }
        } else {
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
            beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
            if (logger.isTraceEnabled()) {
                logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
                        "[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
            }
        }
    }
```

给容器初始化一个事件多播器，这个具体有什么作用还没有搞懂。

### 12.onRefresh

```java
    protected void onRefresh() throws BeansException {
        // For subclasses: do nothing by default.
    }
```

`onRefresh` 是一个模板方法，用来自定义一些实现。

### 13.registerListeners

```java
    protected void registerListeners() {
        // Register statically specified listeners first.
        for (ApplicationListener<?> listener : getApplicationListeners()) {
            getApplicationEventMulticaster().addApplicationListener(listener);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let post-processors apply to them!
        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
        for (String listenerBeanName : listenerBeanNames) {
            getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
        }

        // Publish early application events now that we finally have a multicaster...
        // TODO 下面没有看懂作用是什么
        Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
        this.earlyApplicationEvents = null;
        if (earlyEventsToProcess != null) {
            for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                getApplicationEventMulticaster().multicastEvent(earlyEvent);
            }
        }
    }
```

通过 `getBeanNamesForType` 方法获取到所有实现 `ApplicationListener` 的 beanName 的列表，进行注册。

### 14.finishBeanFactoryInitialization

```java
    protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        // Initialize conversion service for this context.
        // 初始化 property 转换服务 bean
        if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
                beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
            beanFactory.setConversionService(
                    beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
        }

        // 设置注解 value 解析器
        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
        }

        // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
        // TODO 这个是干嘛用的
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }

        // Stop using the temporary ClassLoader for type matching.
        // 禁用临时类加载器
        beanFactory.setTempClassLoader(null);

        // Allow for caching all bean definition metadata, not expecting further changes.
        // 冻结配置，不允许修改
        beanFactory.freezeConfiguration();

        // Instantiate all remaining (non-lazy-init) singletons.
        /**
         * 初始化所有非懒加载的单例 bean
         */
        beanFactory.preInstantiateSingletons();
    }
```

 1. 初始化 property 转换服务 bean
 2. 设置注解 value 解析器
 3. `LoadTimeWeaverAware` 目前还没有搞懂是干啥的
 4. 禁用临时类加载器，并冻结配置文件
 5. 遍历所有的 `beanDefinitionNames`，根据 `beanDefinitionNam`e 获取 merged 过的 `beanDefinitio`n，并过滤懒加载与非单例的 `beanDefinition`
 6. 判断是否是 `FactoryBean`，走不同的分支创建 bean（getBean）
 7. 单例 bean 创建完成，判断 bean 是否是 `SmartInitializingSingleton`（实现该接口），如果是则在 bean 创建完成后执行回调

 关于创建 bean 的流程与循环依赖等细节，我会在后续的文章里进行总结，感兴趣的可以留意一下。

### 15.finishRefresh
    
```java
    protected void finishRefresh() {
        // Clear context-level resource caches (such as ASM metadata from scanning).
        // 清除资源缓存
        clearResourceCaches();

        // Initialize lifecycle processor for this context.
        initLifecycleProcessor();

        // Propagate refresh to lifecycle processor first.
        getLifecycleProcessor().onRefresh();

        // Publish the final event.
        publishEvent(new ContextRefreshedEvent(this));

        // Participate in LiveBeansView MBean, if active.
        LiveBeansView.registerApplicationContext(this);
    }
```

 1. 清除资源缓存
 2. 初始化容器生命周期处理器
 3. 设置处理器生命周期开始
 4. 执行所有 `ApplicationListener`，这里可以详细了解一下监听器
 5. `LiveBeansView.registerApplicationContext(this)` 没有看懂什么意思

### 16.resetCommonCaches

```java
    protected void resetCommonCaches() {
        ReflectionUtils.clearCache();
        AnnotationUtils.clearCache();
        ResolvableType.clearCache();
        CachedIntrospectionResults.clearClassLoader(getClassLoader());
    }
```

当所有步骤完成时清除相关缓存。


