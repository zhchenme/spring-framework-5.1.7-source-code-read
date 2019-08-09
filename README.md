### Spring 源码分析

[解决 Spring 循环依赖](http://www.imooc.com/article/34150) by 田小波

### TAG

- 获取 bean
    - `org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean`
- 创建 bean
    - `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])`
    - `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean`
    - `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)`
   
``` java
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

bean 的生命周期图解([图源](http://www.tianxiaobo.com/2018/01/19/Spring-bean%E7%9A%84%E7%94%9F%E5%91%BD%E6%B5%81%E7%A8%8B/))：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/bean%e5%ae%9e%e4%be%8b%e5%8c%96%e8%bf%87%e7%a8%8b.png)



### IoC 流程

入口：`org.springframework.context.support.AbstractApplicationContext.refresh`
test：`org.springframework.web.context.XmlWebApplicationContextTests.withoutMessageSource`