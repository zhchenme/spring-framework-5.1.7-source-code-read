### 一、概述

Spring 的源码也看了有两个多月的时间了，大概的功能模块基本上也都了解一些，想要把所有模块串起来可能还需要很长时间，本来也打算花大概七八个月甚至更长时间来撸 Spring 的源码，所以关于进展这一块也算是预料之中的事情。

自己是对着网上的博客文章一步步跟着看下来的，虽然很多点都了解一些，但是理解都不是很深刻，甚至有很多不懂的地方。自己的计划安排是先了解大概，回过头来再温习一遍，最后找一本不错的 Spring 书籍再进行巩固。

现在正处在第二阶段，想着出一些分析文章，可能写的不好，但是还是想尝试一下，别人总结的再好，哪怕看懂了，还是觉得心里不踏实。下面对源码分析的时候会参考部分博主的文章（已经经得博主许可），个人建议看 Spring 源码分析的文章且不可眼高手低，一定要对着源码自己看一遍，要不然很容易就忘记了。

多余的话就说到这里，下面开始进入正题。

### 二、熟悉而陌生的 getBean

这里默认大家熟悉 `BeanFactory`、`FactoryBean`、`BeanDefinition` 的用途，如果感觉这些概念比较迷惑的话建议先了解一些相关的概念后再来看。

**2.1 getBean 方法预览**

相信使用过 Spring 的开发者们一定写过这样的代码 `applicationContext.getBean(alias)`，调用过程很简单，过程确是很复杂的，下满我们一起来看一下。

```java
AbstractBeanFactory

    @Override
    public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        return doGetBean(name, requiredType, null, false);
    }
```

`getBean` 方法里就一行代码，更多的实现逻辑都在 `goGetBean` 中，相信很多人看了下面这个方法都望而却步了，代码真的太多了，Spring 的源码就是这样，模块很多，代码也很多，如果想要吃透可能要花很长时间，坚持吧。

```java
AbstractBeanFactory

    /**
     * @param name          beanName or alias
     * @param requiredType  bean 的类型
     * @param args          创建 bean 时携带的参数
     * @param typeCheckOnly 是否为类型检查
     */
    @SuppressWarnings("unchecked")
    protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
            @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
        /**
         * 这里的 name 有三种情况：
         * 1.beanName 本身，即：<id="" class=""/> 中 id 的值
         * 2.'&' 开头，表示获取 FactoryBean 本身
         * 3.alias 别名
         *
         * 如果 beanName 以 `&` 开头，则去除 `&` 前缀，如果是 alias 则替换为对应的 beanName，
         * 这里为什么要获取转化为 beanName 呢？
         * 原因是要根据 beanName 去缓存中查找是否当前 bean 已经被初始化了，如果初始化了则命中缓存直接返回
         */
        final String beanName = transformedBeanName(name);
        Object bean;

        
        // 单例模式的 Bean 在整个过程中只会被创建一次，第一次创建后会将该 Bean 加载到缓存中，再获取 Bean 就会从单例缓存中获取
        Object sharedInstance = getSingleton(beanName);
        if (sharedInstance != null && args == null) {
            if (logger.isTraceEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            /**
             * 注意这个 name 是没有经过处理的，有可能以 `&` 开头，即获取 FactoryBean 本身，而不是对应的 bean 实例
             * 当然如果是获取 FactoryBean 本身则会直接返回
             */
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }
        // 单例缓存中没有实例，则可能表明该实例还没有创建或者该实例在父容器中已经创建了，所以需要先检查一次父容器
        else {
            /**
             * 因为 Spring 只解决单例模式下得循环依赖，在原型模式下如果存在循环依赖则会抛出异常
             * Spring 只能解决单例模式的循环依赖，为什么呢？因为单例模式下有对象缓存
             */
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // 如果容器中没有找到，则从父类容器中加载
            BeanFactory parentBeanFactory = getParentBeanFactory();
            // parentBeanFactory 不为空且 beanDefinitionMap 中已经保存过 beanName 对应的 BeanDefinition
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // 获取 name 对应的 beanName，如果 name 是以 & 字符开头，则返回 & + beanName
                String nameToLookup = originalBeanName(name);
                // 如果，父类容器为 AbstractBeanFactory ，直接递归查找返回
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                            nameToLookup, requiredType, args, typeCheckOnly);
                }
                // 用明确的 args 从 parentBeanFactory 中，获取 Bean 对象
                else if (args != null) {
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                // 用明确的 requiredType 从 parentBeanFactory 中，获取 Bean 对象
                else if (requiredType != null) {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
                // 直接使用 beanName 从 parentBeanFactory 获取 Bean 对象
                else {
                    return (T) parentBeanFactory.getBean(nameToLookup);
                }
            }

            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                // 合并父 BeanDefinition 与子 BeanDefinition
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                // 检查给定的合并的 beanDefinition
                checkMergedBeanDefinition(mbd, beanName, args);

                // 获取依赖的 bean
                String[] dependsOn = mbd.getDependsOn();
                /**
                 * 每个 Bean 都不是单独工作的，它会依赖其他 Bean，其他 Bean 也会依赖它
                 * 对于依赖的 Bean ，它会优先加载，所以，在 Spring 的加载顺序中，
                 * 在初始化某一个 Bean 的时候，首先会初始化这个 Bean 的依赖
                 *
                 *
                 * <bean id="beanA" class="BeanA" depends-on="beanB">
                 * <bean id="beanB" class="BeanB" depends-on="beanA">
                 *
                 * 对于上面这种依赖关系会抛出异常
                 */
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        // 如果存在循环依赖，则抛出异常
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        // 注入依赖的 bean，A 依赖 B，B 依赖 A 都需要被记下，存储在对应的 map 中，记录彼此依赖的关系
                        registerDependentBean(dep, beanName);
                        try {
                            // 加载 depends-on 依赖的 bean
                            getBean(dep);
                        }
                        catch (NoSuchBeanDefinitionException ex) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                        }
                    }
                }

                /**
                 * Spring Bean 的作用域默认为 singleton ，当然，还有其他作用域，如 prototype、request、session 等
                 * 不同的作用域会有不同的初始化策略
                 * 如果是单例模式，因为刚开始是从单例缓存中获取，如果缓存中不存在，则需要从头开始加载
                 * 后面会对单例模式的 createBean 方法进行分析
                 */
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, () -> {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            destroySingleton(beanName);
                            throw ex;
                        }
                    });
                    // 处理 FactoryBean 类型的 bean
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }

                // 原型模式
                else if (mbd.isPrototype()) {
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
                
                // 从指定的 scope 下创建 bean
                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, () -> {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // 检查 bean 的实际类型是否符合需要的类型
        if (requiredType != null && !requiredType.isInstance(bean)) {
            try {
                T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
                if (convertedBean == null) {
                    throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
                }
                return convertedBean;
            }
            catch (TypeMismatchException ex) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
```

代码很长，但是整体的流程理解下来也不是很难，下面来总结一下 `doGetBean` 方法的流程：

 1. 转换 name 为 beanName（用于从缓存中获取单例 bean）
 2. 根据 beanName 从单例缓存中获取 bean，如果 bean 不为空，则调用 `getObjectForBeanInstance` 方法，下面会分析到
 3. 缓存中没有则获取父容器，父容器存在时尝试从父容器中获取 bean
 4. 父容器不存在时，合并父子的 `BeanDefinition`，接着处理 `depends-on` 的依赖关系
 5. 根据不同的类型创建 bean
 6. 创建 bean 时如果制定了类型，则对 bean 进行类型转换

上面对 `getBean` 大致的流程进行了分析，因为调用了很多其他的方法，下面我们就针对这些方法做一些更详细的分析。

**2.2 beanName 转换**

```java
AbstractBeanFactory

    protected String transformedBeanName(String name) {
        return canonicalName(BeanFactoryUtils.transformedBeanName(name));
    }
```

转换成 beanName 调用了两个方法，我们分别来看下：

```java
BeanFactoryUtils

    public static String transformedBeanName(String name) {
        Assert.notNull(name, "'name' must not be null");
        // 不以 '&' 开头，直接返回
        if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
            return name;
        }
        // 如果以 '&' 开头，进行截取，比如 '&&&&aaa' 返回 `aaa`
        return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
            do {
                beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
            }
            while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
            return beanName;
        });
    }
```

`transformedBeanName` 主要用于处理 `&` 开头的 name，为什么会有以 `&` 开头的 name 呢？原因就是要获取 `FactoryBean` 本身而不是获取其内部的 bean。

```java
AliasRegistry

    public String canonicalName(String name) {
        String canonicalName = name;
        String resolvedName;
        do {
            // 可能是根据别名获取 bean，因此要从别名集合中获取 beanName
            resolvedName = this.aliasMap.get(canonicalName);
            if (resolvedName != null) {
                canonicalName = resolvedName;
            }
        }
        while (resolvedName != null);
        return canonicalName;
    }
```

上面这个方法主要用于处理别名，别名都被放在一个 map（K:alias V:beanName）中，因为别名没有限制(别名 A 指向别名 B，别名 B 指向名称为 C 的 bean，则返回 C)，因此要通过循环的方式找到最终的别名，并返回对应的 beanName。

**2.3 从缓存获取单例 bean**

单例 bean 最大的特点就是一单创建就会存储在缓存中，以后再获取单例 bean 时都会先检查一下缓存。

```java
DefaultSingletonBeanRegistry

    public Object getSingleton(String beanName) {
        return getSingleton(beanName, true);
    }
```

`getSingleton` 调用了它的一个重载方法：

```java
DefaultSingletonBeanRegistry

    /**
     * @param beanName            beanName
     * @param allowEarlyReference 是否允许提前曝光未创建完全的 bean
     */
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 从 singletonObjects 缓存中获取，singletonObjects 中存放的都是已经初始化完全的 bean
        Object singletonObject = this.singletonObjects.get(beanName);
        // 如果 bean 还没有创建好且正处在创建状态
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                // 从 earlySingletonObjects 中获取单例对象，提前曝光 bean，用于解决循环依赖问题
                singletonObject = this.earlySingletonObjects.get(beanName);
                // 如果 earlySingletonObjects 缓存中没有，切允许提前曝光，则从工厂中获取 bean，用于解决循环依赖
                if (singletonObject == null && allowEarlyReference) {
                    // 获取单例缓存工厂
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        // 从工厂中获取 bean），提前曝光 bean
                        singletonObject = singletonFactory.getObject();
                        // 添加 bean 到 earlySingletonObjects 中
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        // 从 singletonFactories 中移除对应的 ObjectFactory，因为已经加入到了 earlySingletonObjects 缓存
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }
```

`getSingleton` 代码并不多，除了获取缓存中的单例 bean 以外还用来解决循环依赖问题，关于这三个缓存的 map，下面简单总结一下，循环依赖的问题，后面会单独拿出来分析
。

- `singletonObjects`：用于存放完全初始化好的 bean，从该缓存中取出的 bean 可以直接使用
- `earlySingletonObjects`：用于存放还在初始化中的 bean，用于解决循环依赖
- `singletonFactories`：用于存放 bean 工厂。bean 工厂所产生的 bean 是还未完成初始化的 bean，bean 工厂所生成的对象最终会被缓存到 `earlySingletonObjects` 中

**2.4 从 FactoryBean 中获取 bean**

`FactoryBean` 与 `BeanFactory` 经常会被拿出来作比较，简单来说 `FactoryBean` 是一种用于生产 bean 接口，用户可以通过实现它来生成 bean，而 `BeanFactory` 是用来生产 bean 的工厂，IoC 容器中大多数 bean（`FactoryBean 除外`）都是 `BeanFactory` 生成的。

关于 `FactoryBean` 的使用可以参考 [容器源码分析系列文章导读](http://www.tianxiaobo.com/2018/05/30/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/) 中的例子，下面我们来看代码：

```java
AbstractBeanFactory

    protected Object getObjectForBeanInstance(
            Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

        // 如果 name 与工厂相关以 '&' 开头，表示获取 FactoryBean 本身，而不是对应的 bean 实例
        if (BeanFactoryUtils.isFactoryDereference(name)) {
            // 如果是 NullBean 直接返回
            if (beanInstance instanceof NullBean) {
                return beanInstance;
            }
            // 如果不是 FactoryBean 类型直接抛出异常
            if (!(beanInstance instanceof FactoryBean)) {
                throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
            }
        }

        /**
         * !(beanInstance instanceof FactoryBean) 为 true 时表示这是一个普通的 bean（非 FactoryBean 类型），直接返回
         * BeanFactoryUtils.isFactoryDereference(name) 为 true 时表示要获取的 FactoryBean 本身，也直接返回
         * BeanFactoryUtils.isFactoryDereference(name) 方法判断 name 是否以 `&` 开头，是则返回 true
         */
        if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
            return beanInstance;
        }

        // 代码如果走到这里，则表明 beanInstance 一定是 FactoryBean 类型的
        Object object = null;
        /**
         * 如果 beanDefinition 为 null，则从 factoryBeanObjectCache 缓存中获取 bean
         * FactoryBean 生成的单例 bean 会被缓存在 factoryBeanObjectCache 集合中，不用每次都创建
          */
        if (mbd == null) {
            object = getCachedObjectForFactoryBean(beanName);
        }
        // 使用 FactoryBean 获得 Bean 对象
        if (object == null) {
            FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
            // 如果 mbd 为空，判断当前 FactroyBean 的 beanName 是否已经注册到 beanDefinitionMap 中
            if (mbd == null && containsBeanDefinition(beanName)) {
                // 合并 BeanDefinition
                mbd = getMergedLocalBeanDefinition(beanName);
            }
            // 检测是用户定义的还是程序本身定义的
            boolean synthetic = (mbd != null && mbd.isSynthetic());
            // 获取 FactoryBean 对象
            object = getObjectFromFactoryBean(factory, beanName, !synthetic);
        }
        return object;
    }
```

`getObjectForBeanInstance` 方法中，首先会判断 `beanInstance` 的类型，如果要获取的是 `FactoryBean` 本身或者普通的 bean，则直接返回，如果是要获取 `FactoryBean` 中对应的实例则调用 `getObjectFromFactoryBean` 方法获取。

```java
FactoryBeanRegistrySupport

    protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
        /**
         * FactryBean 默认为单例模式，对于单例 bean 在创建后会放到缓存中
         * 原型模式并没有缓存，每次都会重新创建对象
         */
        if (factory.isSingleton() && containsSingleton(beanName)) {
            synchronized (getSingletonMutex()) {
                // 从 factoryBeanObjectCache 缓存中获取指定的 Bean
                Object object = this.factoryBeanObjectCache.get(beanName);
                if (object == null) {
                    // 缓存中为空，则从 FactoryBean 中获取
                    object = doGetObjectFromFactoryBean(factory, beanName);
                    Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                    if (alreadyThere != null) {
                        object = alreadyThere;
                    }
                    else {
                        if (shouldPostProcess) {
                            // 当前 bean 正在创建
                            if (isSingletonCurrentlyInCreation(beanName)) {
                                // 暂时返回非后处理对象，而不是存储它
                                return object;
                            }
                            // 前置处理与后置处理至关重要，把 bean 加入到正在创建的集合中
                            // 他们记录着 Bean 的加载状态，是检测当前 Bean 是否处于创建中的关键之处，对解决 Bean 循环依赖起着关键作用
                            beforeSingletonCreation(beanName);
                            try {
                                // 对从 FactoryBean 获取的对象进行后处理，默认直接返回对象，可自定义实现类
                                // 生成的对象将暴露给 bean 引用
                                object = postProcessObjectFromFactoryBean(object, beanName);
                            }
                            catch (Throwable ex) {
                                throw new BeanCreationException(beanName,
                                        "Post-processing of FactoryBean's singleton object failed", ex);
                            }
                            finally {
                                // 后置处理， 把 bean 从正在创建的集合中移除
                                afterSingletonCreation(beanName);
                            }
                        }
                        // 添加到 factoryBeanObjectCache 中，进行缓存
                        if (containsSingleton(beanName)) {
                            this.factoryBeanObjectCache.put(beanName, object);
                        }
                    }
                }
                return object;
            }
        }
        // 非单例模式
        else {
            // 跳过缓存，直接从从 FactoryBean 中获取对象
            Object object = doGetObjectFromFactoryBean(factory, beanName);
            // 判断是否需要后续处理
            if (shouldPostProcess) {
                try {
                    // 对从 FactoryBean 获取的对象进行后置处理
                    object = postProcessObjectFromFactoryBean(object, beanName);
                }
                catch (Throwable ex) {
                    throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
                }
            }
            return object;
        }
    }
```

`FactoryBean` 有单例与原型两种模式，默认为单例模式，如果为单例模式，则先尝试从缓存中获取，如果缓存中为空则调用 `doGetObjectFromFactoryBean` 方法创建，创建成功后需要添加到缓存中去，避免下次获取时重新创建。如果为非单例模式则直接调用 `doGetObjectFromFactoryBean` 方法，生成的对象并不会缓存，当然在这个过程中还涉及到后置处理器相关的内容，这里就不展开了。

`doGetObjectFromFactoryBean` 方法实现如下：

```java
FactoryBeanRegistrySupport

    private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
            throws BeanCreationException {

        Object object;
        try {
            // 有系统安全权限，直接使用特权获取 bean
            if (System.getSecurityManager() != null) {
                AccessControlContext acc = getAccessControlContext();
                try {
                    object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
                }
                catch (PrivilegedActionException pae) {
                    throw pae.getException();
                }
            }
            // 不需要权限验证，直接从 FactoryBean 中获取 bean
            else {
                object = factory.getObject();
            }
        }
        catch (FactoryBeanNotInitializedException ex) {
            throw new BeanCurrentlyInCreationException(beanName, ex.toString());
        }
        catch (Throwable ex) {
            throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
        }
        
        if (object == null) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(
                        beanName, "FactoryBean which is currently in creation returned null from getObject");
            }
            object = new NullBean();
        }
        return object;
    }
```

上面的方法中我们只需要关注 `factory.getObject()` 即可，比较简单。

整个流程较长，下面我们来总结一下：

 1. 判断 bean 的类型，如果是 `FactoryBean` 本身或普通的 bean，则直接返回，如果不是上面两种情况都不满足，则说明一定是 `FactoryBean` 类型的实例
 2. 判断缓存中是否有该实例，有则直接返回
 3. 如果缓存中没有，则判断是否有父 `BeanDefinition`，如果有，则与子 `BeanDefinition` 进行合并
 4. 判断 `FactoryBean` 是否为单例模式，如果是单例模式从 `factoryBeanObjectCache `缓存中获取 bean，缓存为空直接从 `FactoryBean` 中获取，中间有后置处理，获取后放入到缓存
 5. 原型模式也是直接从 `FactoryBean` 中直接获取，不会放入到缓存

**2.5 合并 BeanDefinition**

Spring 支持配置继承，在标签中可以使用parent属性配置父类 bean。这样子类 bean 可以继承父类 bean 的配置信息，同时也可覆盖父类中的配置。比如下面的配置（这段完全拷贝[田小波](http://www.tianxiaobo.com/2018/06/01/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E8%8E%B7%E5%8F%96%E5%8D%95%E4%BE%8B-bean/#24-%E5%90%88%E5%B9%B6%E7%88%B6-beandefinition-%E4%B8%8E%E5%AD%90-beandefinition)的案例分析）：

```java
    <bean id="hello" class="xyz.coolblog.innerbean.Hello">
        <property name="content" value="hello"/>
    </bean>

    <bean id="hello-child" parent="hello">
        <property name="content" value="I`m hello-child"/>
    </bean>
```

如上所示，hello-child 配置继承自 hello。hello-child 未配置 class 属性，这里我们让它继承父配置中的 class 属性，通过指定父 bean，子 bean 也可以实例化成功。看完了例子下面我们来看代码：

```java
AbstractBeanFactory

    protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
        // 如果 beanDefinition 已经合并了，获取后直接返回
        RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
        if (mbd != null) {
            return mbd;
        }
        // beanName 与 beanDefinition 合并
        return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
    }
```

合并前会先检查之前 `BeanDefinition` 是否合并过，如果合并过则直接返回合并后的 `BeanDefinition`。

```java
AbstractBeanFactory

    protected RootBeanDefinition getMergedBeanDefinition(
            String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
            throws BeanDefinitionStoreException {

        synchronized (this.mergedBeanDefinitions) {
            RootBeanDefinition mbd = null;

            // 这里 containingBd 为 null，个人感觉是为了适配其他的调用
            if (containingBd == null) {
                mbd = this.mergedBeanDefinitions.get(beanName);
            }

            if (mbd == null) {
                // 如果没有父配置，则拷贝 RootBeanDefinition 或将 BeanDefinition 升级为 RootBeanDefinition
                if (bd.getParentName() == null) {
                    if (bd instanceof RootBeanDefinition) {
                        mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
                    }
                    else {
                        mbd = new RootBeanDefinition(bd);
                    }
                }
                // 有父配置
                else {
                    // 将子 BeanDefinition 的数据与父 BeanDefinition 进行合并
                    BeanDefinition pbd;
                    try {
                        // 获取父的 beanName，这里会进行转化，上面我们已经分析过了
                        String parentBeanName = transformedBeanName(bd.getParentName());
                        // 判断父 beanName 与子 beanName 是否相同
                        if (!beanName.equals(parentBeanName)) {
                            // 父 beanDefinition 可能也有 parent，如果有需要继续合并父 beanDefinition与爷爷 beanDefinition
                            pbd = getMergedBeanDefinition(parentBeanName);
                        }
                        else {
                            // 名字相同特殊处理，并合并父 beanDefinition 与爷爷 beanDefinition
                            BeanFactory parent = getParentBeanFactory();
                            if (parent instanceof ConfigurableBeanFactory) {
                                pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                            }
                            else {
                                throw new NoSuchBeanDefinitionException(parentBeanName,
                                        "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
                                        "': cannot be resolved without an AbstractBeanFactory parent");
                            }
                        }
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
                                "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
                    }
                    // 将父 beanDefinition 深拷贝到 RootBeanDefinition
                    mbd = new RootBeanDefinition(pbd);
                    // 子 beanDefinition 覆盖父 beanDefinition 属性
                    mbd.overrideFrom(bd);
                }

                // 如果没有指定，默认设置为单例模式
                if (!StringUtils.hasLength(mbd.getScope())) {
                    mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
                }

                if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
                    mbd.setScope(containingBd.getScope());
                }

                if (containingBd == null && isCacheBeanMetadata()) {
                    // 将 beanName 与 RootBeanDefinition 进行关联，下次可直接获取
                    this.mergedBeanDefinitions.put(beanName, mbd);
                }
            }

            return mbd;
        }
    }
```

`getMergedBeanDefinition` 相对来说比较简单，先检查是否有父 `BeanDefinition`，如果没有则升级为 `RootBeanDefinition`，如果有则父 `BeanDefinition` 也可能有父 `BeanDefinition`，有的话继续合并。接着将父 `BeanDefinition` 中的属性全部深拷贝到 `RootBeanDefinition` 中，然后子属性覆盖父属性，最后将 beanName 与 `RootBeanDefinition` 进行关联，下次可直接从缓存中获取。

### 总结

这里我们只讲了从缓存获取单例 bean 与获取 `FactoryBean` 的过程，当单例 bean 没有创建时则需要从头开始加载创建 bean，为了防止篇幅过长，单独放到下一篇文章里里面讲解。Spring 单例 bean 还涉及到另外一个问题：循环依赖，关于循环依赖，后面也会单独讲解。

关于 Spring 总结性的文章，参考了网上的一些文章（见参考），感谢！

### 参考

[Spring IOC 容器源码分析 - 获取单例 bean](http://www.tianxiaobo.com/2018/06/01/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E8%8E%B7%E5%8F%96%E5%8D%95%E4%BE%8B-bean/) by 田小波 <br>
[死磕 Spring 系列](http://cmsblogs.com/?cat=206) by 小明哥 <br>



