# Hibernate Validator 6.0.13.Final - JSR 380 参考实现：参考文档

Hibernate Validator 6.0.13.Final - JSR 380 Reference Implementation: Reference Guide

Hardy Ferentschik  Gunnar Morling  Guillaume Smet  2018-08-22


# 序言

数据校验功能是一个应用程序的常用功能，它发生在应用的从展示层到持久层的各个层面。我们经常会在各个层面使用相同的校验逻辑，而这样容易导致
时间浪费和容易出错。为了避免进行重复的校验，开发者经常把校验逻辑写在领域模型里面，这样又将领域模型的代码和校验代码杂糅在一起，而领域模型
应该只关乎元数据本身。

![avatar](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers.png)

JSR 380 - Bean Validation 2.0 - 为了实体和方法的校验定义了元数据模型和 API。默认的元数据模型是通过注解来描述的，但是也可以通过XML配
置来重写和拓展元数据。这些 API 并没有限制在某一特定的应用层或者编程模型上，也没有限制在 web 层或持久层。而且既可以用在服务端应用，也可以用
在类似 Swing 这样的客户端。

![avatar](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers2.png)

Hibernate Validator 是 JSR 380 参考实现。Hibernate Validator、Bean Validation API 和 TCK 都是使用了Apache Software License 
2.0。

Hibernate Validator 6 和 Bean Validation 2.0 需要 Java8 或更新版本。 

# 1. 开始使用

这一章节将会告诉你如何开始使用 Hibernate Validator（Bean Validation 的参考实现）。你需要提前准备好这些：

- JDK8
- [Apache Maven](http://maven.apache.org/)
- 联网环境（Maven 需要下载一些必要的库）

## 1.1. 工程设置

为了在 Maven 工程中使用 Hibernate Validator，你只需要在 *pom.xml* 文件中添加下列依赖：

*Example 1.1: Hibernate Validator Maven dependency*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.13.Final</version>
</dependency>
```

这也会将它的依赖 Bean Validation API (javax.validation:validation-api:2.0.1.Final)拉下来。

### 1.1.1. Unified EL 统一表达式语言

Hbernate Validator 需要统一表达式语言（[JSR341](http://jcp.org/en/jsr/detail?id=341)）的实现来动态的评估违反限制的表达式（参见
Section 4.1, “Default message interpolation”）。当你的应用跑在一个类似 JBoss AS 的 Java EE 容器中时，容器已经提供了一个EL的实
现。但是在Java SE环境中，你需要通过在 POM 文件中添加一个依赖来自己添加 EL 的实现。例如，你可以添加以下的依赖来使用 JSR 341 [参考实
现](https://javaee.github.io/uel-ri/):

*Example 1.2: Maven dependencies for Unified EL reference implementation*

```xml
<dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.el</artifactId>
    <version>3.0.1-b09</version>
</dependency>
```

> 提示
>
> 对于那些无法提供 EL 实现的环境，Hibernate Validator提供了 [Section 12.9, “ParameterMessageInterpolator”]()。但是使用interpolator
并不符合 Bean Validation 的规范。

### 1.1.2. CDI

Bean Validation 使用了 CDI（Java<sup>TM</sup> EE 的上下文和依赖注入，[JSR346]()） 定义了整合点。如果你的应用所在的环境无法提供这
个，你可以通过添加一下的 Maven 依赖，来使用使用 Hibernate Validator CDI 可移植拓展。

*Example 1.3: Hibernate Validator CDI portable extension Maven dependency*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-cdi</artifactId>
    <version>6.0.13.Final</version>
</dependency>
```

值得注意的是，大部分跑在 Java EE 环境上的应用是不需要添加这个依赖的。你可以在这 [Section 11.3, “CDI”]() 获取更多关于Bean Validation 
和 CDI 整合的内容。
 
### 1.1.3. 结合安全管理器运行

Hibernate Validator 支持与[安全管理器](http://docs.oracle.com/javase/8/docs/technotes/guides/security/index.html)一起启动，
为了实现这个功能，你必须到Hibernate Validator，Bean Validation API，Classmate and JBoss Logging 和 Bean Validation的代码底层
去分配一些许可权限。下面展示了如何通过一个加工过的 Java 默认 policy 文件 [policy file](http://docs.oracle.com/javase/8/docs/technotes/guides/security/PolicyFiles.html)
来实现这个功能。

*Example 1.4: Policy file for using Hibernate Validator with a security manager*

```
grant codeBase "file:path/to/hibernate-validator-6.0.13.Final.jar" {
    permission java.lang.reflect.ReflectPermission "suppressAccessChecks";
    permission java.lang.RuntimePermission "accessDeclaredMembers";
    permission java.lang.RuntimePermission "setContextClassLoader";

    permission org.hibernate.validator.HibernateValidatorPermission "accessPrivateMembers";

    // Only needed when working with XML descriptors (validation.xml or XML constraint mappings)
    permission java.util.PropertyPermission "mapAnyUriToUri", "read";
};

grant codeBase "file:path/to/validation-api-2.0.1.Final.jar" {
    permission java.io.FilePermission "path/to/hibernate-validator-6.0.13.Final.jar", "read";
};

grant codeBase "file:path/to/jboss-logging-3.3.2.Final.jar" {
    permission java.util.PropertyPermission "org.jboss.logging.provider", "read";
    permission java.util.PropertyPermission "org.jboss.logging.locale", "read";
};

grant codeBase "file:path/to/classmate-1.3.4.jar" {
    permission java.lang.RuntimePermission "accessDeclaredMembers";
};

grant codeBase "file:path/to/validation-caller-x.y.z.jar" {
    permission org.hibernate.validator.HibernateValidatorPermission "accessPrivateMembers";
};
```

[WildFly 应用程序服务器](http://wildfly.org/)包含了开箱即用的 Hibernate Validator。为了升级服务器模块到最新最好的Bean Validation 
API 和 Hibernate Validator，可以使用 WildFly 的补丁。你可以从 [SourceForge](http://sourceforge.net/projects/hibernate/files/hibernate-validator)
上下载补丁文件，也可以使用以下依赖从 Maven 中央仓库下载。 

*Example 1.5: Maven dependency for WildFly 14.0.0.Beta1 patch file*

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-modules</artifactId>
    <version>6.0.13.Final</version>
    <classifier>wildfly-14.0.0.Beta1-patch</classifier>
    <type>zip</type>
</dependency>
```

我们也为 WildFly 13.0.0.Final 提供了一个补丁：

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-modules</artifactId>
    <version>6.0.13.Final</version>
    <classifier>wildfly-13.0.0.Final-patch</classifier>
    <type>zip</type>
</dependency>
```

下载补丁文件后，你可以通过以下命令将它应用于 WildFly 上：

*Example 1.7: Applying the WildFly patch*

```
$JBOSS_HOME/bin/jboss-cli.sh patch apply hibernate-validator-modules-6.0.13.Final-wildfly-14.0.0.Beta1-patch.zip
```

如果你想回退补丁，使用服务器提供的原生 Hibernate Validator 版本，你可以使用以下的命令：

*Example 1.8: Rolling back the WildFly patch*

```
$JBOSS_HOME/bin/jboss-cli.sh patch rollback --reset-configuration=true
```

你可以在[这里](https://developer.jboss.org/wiki/SingleInstallationPatching/)
和[这里](http://www.mastertheboss.com/jboss-server/jboss-configuration/managing-wildfly-and-eap-patches)了解到更多关于 
WildFly 补丁的情况。

### 1.1.5. 在 Java 9 上运行























