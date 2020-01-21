在看 Spring IoC 源码之前，建议大家先熟悉一下 IoC 容器中常见的一些配置属性，因为源码中涉及到这些属性的解析，如果不知道这些属性的作用的话，源码看起来会很费劲。

下面大部分的特性总结来源于 [Spring IOC 容器源码分析系列文章导读](http://www.tianxiaobo.com/2018/05/30/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/)，在这篇文章的基础上，我又新增了几个属性。

#### 1 alias

alias 的中文意思是“别名”，在 Spring 中，我们可以使用 alias 标签给 bean 起个别名。比如下面的配置：

```java
<bean id="hello" class="xyz.coolblog.service.Hello">
    <property name="content" value="hello"/>
</bean>
<alias name="hello" alias="alias-hello"/>
<alias name="alias-hello" alias="double-alias-hello"/>
```

这里我们给hello这个 beanName 起了一个别名alias-hello，然后又给别名alias-hello起了一个别名double-alias-hello。我们可以通过这两个别名获取到hello这个 bean 实例，比如下面的测试代码：

```java
public class ApplicationContextTest {

    @Test
    public void testAlias() {
        String configLocation = "application-alias.xml";
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
        System.out.println("    alias-hello -> " + applicationContext.getBean("alias-hello"));
        System.out.println("double-alias-hello -> " + applicationContext.getBean("double-alias-hello"));
    }
}
```

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15275252140497.jpg)

#### 2 autowire

autowire 即自动注入的意思，通过使用 autowire 特性，我们就不用再显示的配置 bean 之间的依赖了。把依赖的发现和注入都交给 Spring 去处理，省时又省力。autowire 几个可选项，比如 byName、byType 和 constructor 等。autowire 是一个常用特性，相信大家都比较熟悉了，所以本节我们就 byName 为例，快速结束 autowire 特性的介绍。

当 bean 配置中的 autowire = byName 时，Spring 会首先通过反射获取该 bean 所依赖 bean 的名字（beanName），然后再通过调用 BeanFactory.getName(beanName) 方法即可获取对应的依赖实例。autowire = byName 原理大致就是这样，接下来我们来演示一下。

```java
public class Service {

    private Dao mysqlDao;

    private Dao mongoDao;

    // 忽略 getter/setter

    @Override
    public String toString() {
        return super.toString() + "\n\t\t\t\t\t{" +
            "mysqlDao=" + mysqlDao +
            ", mongoDao=" + mongoDao +
            '}';
    }
}

public interface Dao {}
public class MySqlDao implements Dao {}
public class MongoDao implements Dao {}
```

配置如下：

```java
<bean name="mongoDao" class="xyz.coolblog.autowire.MongoDao"/>
<bean name="mysqlDao" class="xyz.coolblog.autowire.MySqlDao"/>

<!-- 非自动注入，手动配置依赖 -->
<bean name="service-without-autowire" class="xyz.coolblog.autowire.Service" autowire="no">
    <property name="mysqlDao" ref="mysqlDao"/>
    <property name="mongoDao" ref="mongoDao"/>
</bean>

<!-- 通过设置 autowire 属性，我们就不需要像上面那样显式配置依赖了 -->
<bean name="service-with-autowire" class="xyz.coolblog.autowire.Service" autowire="byName"/>
```

测试代码如下：

```java
String configLocation = "application-autowire.xml";
ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
System.out.println("service-without-autowire -> " + applicationContext.getBean("service-without-autowire"));
System.out.println("service-with-autowire -> " + applicationContext.getBean("service-with-autowire"));
```

测试结果如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15276394476647.jpg)

#### 3 FactoryBean

FactoryBean？看起来是不是很像 BeanFactory 孪生兄弟。不错，他们看起来很像，但是他们是不一样的。FactoryBean 是一种工厂 bean，与普通的 bean 不一样，FactoryBean 是一种可以产生 bean 的 bean，好吧说起来很绕嘴。FactoryBean 是一个接口，我们可以实现这个接口。下面演示一下：

```java
public class HelloFactoryBean implements FactoryBean<Hello> {

    @Override
    public Hello getObject() throws Exception {
        Hello hello = new Hello();
        hello.setContent("hello");
        return hello;
    }

    @Override
    public Class<?> getObjectType() {
        return Hello.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

配置如下：

```java
<bean id="helloFactory" class="xyz.coolblog.service.HelloFactoryBean"/>
```

测试代码如下：

```java
public class ApplicationContextTest {

    @Test
    public void testFactoryBean() {
        String configLocation = "application-factory-bean.xml";
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
        System.out.println("helloFactory -> " + applicationContext.getBean("helloFactory"));
        System.out.println("&helloFactory -> " + applicationContext.getBean("&helloFactory"));
    }
}
```

测试结果如下：

![](https://blog-pictures.oss-cn-shanghai.aliyuncs.com/15275264045762.jpg)