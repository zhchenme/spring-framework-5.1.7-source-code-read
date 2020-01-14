

### 一、什么是 BeanDefinition

在传统的 Spring 项目中，一般通过 XML 的方式配置 bean，而 `BeanDefinition` 就是 XML 配置属性的载体，XML 文件首先会被转化成 `Document` 对象，然后解析 `Document`，把 XML 中 `<bean />` 标签转化成 `BeanDefinition` 供 IoC 容器创建 bean 时使用。

