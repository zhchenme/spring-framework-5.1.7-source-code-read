在 [getBean 流程分析.md](getBean%20流程分析.md) 中主要介绍了获取单例 bean 与 `FactoryBean` 的主要流程，这篇文章来介绍下当单例 bean 不存在时创建单例 bean 的过程。

### 一、创建单例 bean

**1.1 单例 bean 创建入口**

创建单例 bean 的入口在 `AbstractBeanFactory` 类中的 `doGetBean` 方法中，如果单例缓存中没有命中，且父容器也没有创建过该 bean， 则走创建单例 bean 的流程。如下

```java
AbstractBeanFactory

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
			// 初始化前置处理，如果不为 null 则直接返回，实现可 BeanPostProcessor 接口
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

上面代码的逻辑比较清晰，下面简单来总结一下：

- 通过 `beanDefinition` 解析 bean 的 class 类型
- 判断是否覆盖了 methodOverrides（meta、lookup-method、replace-method）方法
- 如果实现了 `BeanPostProcessor` 接口，则执行前置处理，如果返回 bean 不为 null 会直接返回，PS `BeanPostProcessor` 是全局的
- 执行 `doCreateBean` 方法创建 bean

比较重要的方法是 `doCreateBean`，下面来仔细分析一下。

### 1.3 `doCreateBean` 方法


```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		// 使用 BeanWrapper 包装创建的 bean
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			// TODO factoryBeanInstanceCache 作用是什么？
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		// 使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化
		if (instanceWrapper == null) {
			/**
			 * 创建 bean 实例，并将实例包裹在 BeanWrapper 实现类对象中返回。createBeanInstance
			 * 中包含三种创建 bean 实例的方式：
			 *   1. 通过工厂方法创建 bean 实例
			 *   2. 通过构造方法自动注入（autowire by constructor）的方式创建 bean 实例
			 *   3. 通过无参构造方法方法创建 bean 实例
			 *
			 * 若 bean 的配置信息中配置了 lookup-method 和 replace-method，则会使用 CGLIB 增强 bean 实例
			 */
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		// 获取包装的 bean 实例对象（暂未填充属性）与 class 对象
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			// 设置目标类型
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					// TODO 后置处理修改 BeanDefinition
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		/**
		 * earlySingletonExposure 表示是否提前曝光 bean，可以用来解决循环依赖，如果是单例 bean 一般为 true
		 */
		boolean earlySingletonExposure = (mbd.isSingleton() // 单例模式
				&& this.allowCircularReferences &&          // 允许循环依赖
				isSingletonCurrentlyInCreation(beanName));  // 当前 bean 正在被创建
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 将工厂对象放入三级缓存 singletonFactories 中
			// TODO 19-07-18 可以解决循环依赖的情况
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//  对 bean 进行填充，将各个属性值注入，可能存在依赖于其他 bean 的属性
			// TODO 19-07-18 与上面循环依赖对应
			populateBean(beanName, mbd, instanceWrapper);
			/**
			 * 初始化：
			 * 1. 判断 bean 是否实现了 BeanNameAware、BeanFactoryAware、BeanClassLoaderAware 等接口，并执行接口方法
			 * 2. 应用 bean 初始化前置操作
			 * 3. 如果 bean 实现了 InitializingBean 接口，则执行 afterPropertiesSet 方法。如果用户配置了 init-method，则调用相关方法执行自定义初始化逻辑
			 * 4. 应用 bean 初始化后置操作
			 */
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			// 从缓存中获取提前暴露的 bean
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					// 这个 bean 可能不是完整的 bean
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		// 注册并返回 bean
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

`doCreate` 方法比较复杂，下面总结一下：

- 从 `factoryBeanInstanceCache` 缓存中判断是否命中 `BeanWrapper`，如果命中且会删除
- `BeanWrapper` 在缓存中不存在，调用 `createBeanInstance` 方法创建
- 判断是否有后置处理器对 `BeanDefinition` 进行合并
- 判断是否需要提前曝光 bean，如果需要则把 bean 加入到 `singletonFactories` 缓存中（还未填充属性）
- 填充 bean 属性
- 调用 `initializeBean` 完成剩下的初始化操作
- 注册销毁逻辑

`createBeanInstance` 方法会具体分析，下面来简单看下 `initializeBean`方法。

1.4 **`initializeBean` 方法**


```java
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				// 激活 Aware 方法，对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			// 前置处理，用户可以实现 BeanPostProcessor 进行自定义业务处理
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			// 如果用户实现了 InitializingBean 接口或者自定义了 init 方法，进行处理
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			// 后置处理
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

