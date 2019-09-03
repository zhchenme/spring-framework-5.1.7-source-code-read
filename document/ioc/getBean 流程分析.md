### 一、概述

Spring 的源码也看了有两个多月的时间了，大概的功能模块基本上也都了解一些，想要把所有模块串起来可能还需要很长时间，本来也打算花大概七八个月甚至更长时间来撸 Spring 的源码，所以关于进展这一块也算是预料之中的事情。

自己是对着网上的博客文章一步步跟着看下来的，虽然很多点都了解一些，但是理解都不是很深刻，甚至有很多不懂的地方。自己的计划安排是先了解大概，回过头来再温习一遍，最后找一本不错的 Spring 书籍再进行巩固。

现在正处在第二阶段，想着出一些分析文章，可能写的不好，但是还是想尝试一下，别人总结的再好，哪怕看懂了，还是觉得心里不踏实。下面对源码分析的时候会参考部分博主的文章（已经经得博主许可），个人建议看 Spring 源码分析的文章且不可眼高手低，一定要对着源码自己看一遍，要不然很容易就忘记了。

多余的话就说到这里，下面开始进入正题。

### 二、熟悉而陌生的 getBean

这里默认大家熟悉 `BeanFactory`、`FactoryBean`、`BeanDefinition` 的用途，如果感觉这些概念比较迷惑的话建议先了解一些相关的概念后再来看。

**2.1 getBean 方法预览**

相信使用过 Spring 的开发者们一定写过这样的代码 `applicationContext.getBean(alias)`，

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

**2.3 获取单例 bean**

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





### 参考



[Spring IOC 容器源码分析 - 获取单例 bean](http://www.tianxiaobo.com/2018/06/01/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E8%8E%B7%E5%8F%96%E5%8D%95%E4%BE%8B-bean/) by 田小波 <br>
[死磕 Spring 系列](http://cmsblogs.com/?cat=206) by 小明哥 <br>



