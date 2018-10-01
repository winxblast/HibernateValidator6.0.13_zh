# Hibernate Validator 6.0.13.Final - JSR 380 参考实现：参考文档

Hibernate Validator 6.0.13.Final - JSR 380 Reference Implementation: Reference Guide

Hardy Ferentschik  Gunnar Morling  Guillaume Smet  2018-08-22


# 序言

数据校验功能是一个应用程序的常用功能，它发生在应用的从展示层到持久层的各个层面。我们经常会在各个层面使用相同的校验逻辑，而这样容易导致
时间浪费和容易出错。为了避免进行重复的校验，开发者经常把校验逻辑写在领域模型里面，这样又将领域模型的代码和校验代码杂糅在一起，而领域模型
应该只关乎元数据本身。

![avatar](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers.png)

JSR 380 - Bean Validation 2.0 - 为了实体和方法的校验定义了元数据模型和API。默认的元数据模型是通过注解来描述的，但是也可以通过XML配
置来重写和拓展元数据。这些API并没有限制在某一特定的应用层或者编程模型上，也没有限制在web层或持久层。而且既可以用在服务端应用，也可以用
在类似Swing这样的客户端。

![avatar](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers2.png)

Hibernate Validator 是 JSR 380 参考实现。Hibernate Validator、Bean Validation API 和 TCK 都是使用了Apache Software License 
2.0。

Hibernate Validator 6 和 Bean Validation 2.0 需要Java8或更新版本。 