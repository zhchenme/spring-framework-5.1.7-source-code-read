在 [getBean 流程分析.md](getBean%20流程分析.md) 中主要介绍了获取单例 bean 与 `FactoryBean` 的主要流程，这篇文章来介绍下当单例 bean 不存在时创建单例 bean 的过程。

### 一、创建单例 bean

**1.1 单例 bean 创建入口**

创建单例 bean 的入口在 `AbstractBeanFactroy` 类中的 `doGetBean` 方法中，如果单例缓存中没有命中，且父容器也没有创建过该 bean， 则走创建单例 bean 的流程。如下

```java
AbstractBeanFactroy

				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							// 创建 bean
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
```

上面的代码只是 `doGetBean` 方法中的一小部分，如果想要了解整个过程可以参考上篇文章的总结 [getBean 流程分析.md](getBean%20流程分析.md)。首先调用 `createBean` 方法创建了一个实例，然后这个实例作为 `getSingleton` 方法的参数返回了 bean 实例。

**1.2 createBean 方法**

```java
AbstractAutowireCapableBeanFactory

protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		// 合并过父与子的 BeanDefinition
		RootBeanDefinition mbdToUse = mbd;

		// 从 beanDefinition 获取 class 属性
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// overrides：lookup-method、replace-method
		try {
			// 重载方法（meta、lookup-method、replace-method）校验，判断是否该方法是否重载
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 实例化对象前的处理，且可以创建 bean 对象
			// 这不是意味着在后置处理过程中可以创建 bean 实例？
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 创建 bean
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```