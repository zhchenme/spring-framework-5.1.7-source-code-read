

### 一、BeanDefinition

#### 1.1 什么是 BeanDefinition

在传统的 Spring 项目中，一般通过 XML 的方式配置 bean，而 `BeanDefinition` 就是 XML 配置属性的载体，XML 文件首先会被转化成 `Document` 对象，然后解析 `Document`，把 XML 中 `<bean />` 标签转化成 `BeanDefinition` 供 IoC 容器创建 bean 时使用。

我们可以来做个测试

```xml
	<bean id="typeMismatch" class="org.springframework.tests.sample.beans.TestBean" scope="prototype">
		<property name="name"><value>typeMismatch</value></property>
		<property name="age"><value>34x</value></property>
		<property name="spouse"><ref bean="rod"/></property>
	</bean>
```

下面是 debug 后抓到的 `BeanDefinition` 属性

![](https://raw.githubusercontent.com/zchen96/spring-framework-5.1.7-source-code-read/master/image/ioc/BeanDefinition2.png)

#### 1.1 BeanDefinition 初始化流程图

![](https://raw.githubusercontent.com/zchen96/spring-framework-5.1.7-source-code-read/master/image/ioc/BeanDefinition1.png)

### 二、源码分析

Debug 测试类入口：`org.springframework.beans.factory.xml.XmlBeanDefinitionReaderTests#withOpenInputStream`

测试代码如下

```java
	@Test(expected = BeanDefinitionStoreException.class)
	public void withOpenInputStream() {
		// 注意这里初始化的是 SimpleBeanDefinitionRegistry 不具备 BeanFactory 功能，仅仅用来注册 BeanDefinition，不能用来创建 bean
		SimpleBeanDefinitionRegistry registry = new SimpleBeanDefinitionRegistry();
		Resource resource = new InputStreamResource(getClass().getResourceAsStream("test.xml"));
		new XmlBeanDefinitionReader(registry).loadBeanDefinitions(resource);
	}
```

`loadBeanDefinitions` 源码如下

```java
	@Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		// 使用 EncodedResource 包装 Resource，EncodedResource 可以指定字符集编码
		return loadBeanDefinitions(new EncodedResource(resource));
	}

	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		// 获取已经加载过的资源集合
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			// 初始化 currentResources
			currentResources = new HashSet<>(4);
			// 设置初始化的 currentResources
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			// 根据 Resource 获取输入流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				// 核心逻辑，加载 bean 资源
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
```

#### 2.1 获取输入流

测试类中定义的是 `InputStreamResource`，我们来看下 `InputStreamResource` 中 `getInputStream()` 的实现。

```java
	@Override
	public InputStream getInputStream() throws IOException {
		InputStream is;
		if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
		if (is == null) {
			throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
		}
		return is;
	}
```

