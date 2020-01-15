

### 一、BeanDefinition

#### 1.1 什么是 BeanDefinition

在传统的 Spring 项目中，一般通过 XML 的方式配置 bean，而 `BeanDefinition` 就是 XML 配置属性的载体，XML 文件首先会被转化成 `Document` 对象，然后解析 `Document`，把 XML 中 `<bean />` 标签转化成 `BeanDefinition` 供 IoC 容器创建 bean 时使用。

我们可以来做个测试。

```xml
    <bean id="typeMismatch" class="org.springframework.tests.sample.beans.TestBean" scope="prototype">
        <property name="name"><value>typeMismatch</value></property>
        <property name="age"><value>34x</value></property>
        <property name="spouse"><ref bean="rod"/></property>
    </bean>
```

下面是 debug 后抓到的 `BeanDefinition` 属性。

![](https://raw.githubusercontent.com/zchen96/spring-framework-5.1.7-source-code-read/master/image/ioc/BeanDefinition2.png)

#### 1.2 BeanDefinition 初始化流程图

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

测试类中定义的是 `InputStreamResource`，下面 `InputStreamResource` 中 `getInputStream()` 的实现。

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

#### 2.2 转化 `Document` 对象


```java
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            // 获取 XML 对应 document 实例
            Document doc = doLoadDocument(inputSource, resource);
            // 调用 registerBeanDefinitions 方法注册 BeanDefinitions
            int count = registerBeanDefinitions(doc, resource);
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + count + " bean definitions from " + resource);
            }
            return count;
        }
        // 下面本来有一堆异常处理，阅读体验不太好直接干掉了
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "Unexpected exception parsing XML document from " + resource, ex);
        }
    }
```

`doLoadBeanDefinitions` 方法中调用 `doLoadDocument` 初始化 `Document` 对象，内部实现比较简单，下面一起来看一下。

```java
    protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
        return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
                getValidationModeForResource(resource), isNamespaceAware());
    }
```

上面涉及到几个关于 XML 的知识点，下面简单的介绍一下，最后面有列出参考文章，有兴趣的可以深入理解。

- `EntityResolver`：XML 文件解析器
- `errorHandler`：解析出错处理机制
- `getValidationModeForResource()` 方法： 获取 XML 验证格式，XML 一般支持 DTD 与 XSD，也可以自定义，主要用来约束与验证 XML 文档格式
- `isNamespaceAware()`：判断 XML 解析器是否支持 XML ,`<beans xmlns=""/>` 其中 `xmlns` 就是命名空间

```java
    @Override
    public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
            ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

        // 创建 DocumentBuilderFactory
        DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
        if (logger.isTraceEnabled()) {
            logger.trace("Using JAXP provider [" + factory.getClass().getName() + "]");
        }
        // 创建一个 DocumentBuilder
        DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
        return builder.parse(inputSource);
    }
```

上面 `DocumentBuilderFactory` 与 `DocumentBuilder` 都是 JDK 中提供的类，根据 XML 输入流获取 `Document` 的过程没有深入跟踪，这里就不展开分析了。

#### 2.3 处理 `Document` 节点

```java
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        // 通过反射创建一个 BeanDefinitionDocumentReader 对象
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        // 获取已经注册的 BeanDefinition 的数量，-> beanDefinitionMap 的 size
        int countBefore = getRegistry().getBeanDefinitionCount();
        // 创建 XmlReaderContext，注册 BeanDefinitions
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        // 返回最新注册的 bean 的数量
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }


    @Override
    public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        doRegisterBeanDefinitions(doc.getDocumentElement());
    }

    protected void doRegisterBeanDefinitions(Element root) {
        // 记录当前 BeanDefinitionParserDelegate 对象（旧的）
        BeanDefinitionParserDelegate parent = this.delegate;
        // 创建一个新的 BeanDefinitionParserDelegate 对象
        this.delegate = createDelegate(getReaderContext(), root, parent);

        // 检查 <beans> 标签的命名空间是否为空，或者是 http://www.springframework.org/schema/beans
        if (this.delegate.isDefaultNamespace(root)) {
            // 获取 profile 的值，beans 标签可以设置 profile 属性用于多环境配置管理
            String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                // 处理 profile 多个值
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                // 判断是否有默认启用的 profile
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                    }
                    return;
                }
            }
        }

        // 解析前处理，空实现，可自定义
        preProcessXml(root);
        // 解析 document 实例
        parseBeanDefinitions(root, this.delegate);
        // 解析后处理，空实现，可自定义
        postProcessXml(root);

        this.delegate = parent;
    }

    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        // 根节点使用默认名称空间，执行默认解析 <bean id="" class="" />
        if (delegate.isDefaultNamespace(root)) {
            // 获取所有的子节点
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    // 同上判断
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            // <tx:annotation-driven>
            delegate.parseCustomElement(root);
        }
    }

    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        // import
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            importBeanDefinitionResource(ele);
        }
        // alias
        else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            processAliasRegistration(ele);
        }
        // bean
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        }
        // beans
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // recurse，递归
            doRegisterBeanDefinitions(ele);
        }
    }
```


### 参考阅读

[XML中DTD,XSD的区别与应用](https://www.jianshu.com/p/13b205ba2175) <br>
[xsd，dtd，tld有什么区别和联系？](https://www.zhihu.com/question/38843167) <br>