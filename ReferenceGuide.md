# Hibernate Validator 6.0.13.Final - JSR 380 参考实现：参考文档

Hibernate Validator 6.0.13.Final - JSR 380 Reference Implementation: Reference Guide

Hardy Ferentschik  Gunnar Morling  Guillaume Smet  2018-08-22

# 序言

数据校验功能是一个应用程序的常用功能，它发生在应用的从展示层到持久层的各个层面。我们经常会在各个层面使用相同的校验逻辑，而这样容易导致
时间浪费和容易出错。为了避免进行重复的校验，开发者经常把校验逻辑写在领域模型里面，这样又将领域模型的代码和校验代码杂糅在一起，而领域模型
应该只关乎元数据本身。

![Alt 图片](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers.png)

JSR 380 - Bean Validation 2.0 - 为了实体和方法的校验定义了元数据模型和 API。默认的元数据模型是通过注解来描述的，但是也可以通过XML配
置来重写和拓展元数据。这些 API 并没有限制在某一特定的应用层或者编程模型上，也没有限制在 web 层或持久层。而且既可以用在服务端应用，也可以用
在类似 Swing 这样的客户端。

![Alt 图片](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/images/application-layers2.png)

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

Hibernate Validator 6.0.13.Final 对于 Java 9 和 Java平台模块化系统（JPMS）的支持是实验性质的。这里还没有提供 JPMS 的模块描述器
提供，但是 Hibernate Validator 能够作为一个自动模块。

以下是声明使用`Automatic-Module-Name`头的模块名：

- Bean Validation API: `java.validation`
  
- Hibernate Validator core: `org.hibernate.validator`
 
- Hibernate Validator CDI extension: `org.hibernate.validator.cdi`
  
- Hibernate Validator test utilities: `org.hibernate.validator.testutils`
  
- Hibernate Validator annotation processor: `org.hibernate.validator.annotationprocessor`

这些模块名可能是暂定的，在未来版本提供真实的模块描述器时可能会改名。

> 注意
>
> 当同时使用 Hibernate Validator 和 CDI 时，小心不要激活 JDK 的 `java.xml.ws.annotation` 模块。这个模块包含了一个有 JSR 250 API
（“常见注解”）的子集，但是例如有些注解 `javax.annotation.Priority` 却缺失了。这就造成了 Hibernate Validator 的方法校验拦截器不能
被注册，也就是说：方法校验失效。
>
> 可行的方法是添加完全版的 JSR 250 API 到你的模块（就是添加到classpath），例如添加 *javax.annotation:javax.annotation-api* 的
依赖（当依赖 *org.hibernate.validator:hibernate-validator-cdi* 时，已经有可使用的 JSR 250 API 依赖了）。
>
> 如果你因为某些原因不得不使用 `java.xml.ws.annotation` 模块，你应该在你的 Java 中通过添加 `--patch-module java.xml.ws.annotation=/path/to/complete-jsr250-api.jar`
来打上 API 的完全补丁。

## 1.2. 使用约束

让我们直接通过一个例子来看看如何使用这些约束吧。

*Example 1.9: Class Car annotated with constraints*

```java
package org.hibernate.validator.referenceguide.chapter01;

import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;

public class Car {

    @NotNull
    private String manufacturer;

    @NotNull
    @Size(min = 2, max = 14)
    private String licensePlate;

    @Min(2)
    private int seatCount;

    public Car(String manufacturer, String licencePlate, int seatCount) {
        this.manufacturer = manufacturer;
        this.licensePlate = licencePlate;
        this.seatCount = seatCount;
    }

    //getters and setters ...
}
```

注解 `@NotNull`，`@Size`，`@Min` 应该被用在 Car 类的成员变量上，用来施加以下的约束：

- `manufacture` 必须非空
- `licensePlate` 必须非空，且长度在2~14个字符之间
- `seatCount` 最小值为2

> 提示
>
> 你能在 GitHub 中的 Hibernate Validator 代码库中找到所有例子的完整源代码。

## 1.3. 校验约束

为了校验这些约束，你需要使用一个 `Validator` 实例。让我们来看一个 `Car` 的单元测试：

*Example 1.10: Class CarTest showing validation examples*

```java
package org.hibernate.validator.referenceguide.chapter01;

import java.util.Set;
import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;

import org.junit.BeforeClass;
import org.junit.Test;

import static org.junit.Assert.assertEquals;

public class CarTest {

    private static Validator validator;

    @BeforeClass
    public static void setUpValidator() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    public void manufacturerIsNull() {
        Car car = new Car( null, "DD-AB-123", 4 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals( "must not be null", constraintViolations.iterator().next().getMessage() );
    }

    @Test
    public void licensePlateTooShort() {
        Car car = new Car( "Morris", "D", 4 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals(
                "size must be between 2 and 14",
                constraintViolations.iterator().next().getMessage()
        );
    }

    @Test
    public void seatCountTooLow() {
        Car car = new Car( "Morris", "DD-AB-123", 1 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 1, constraintViolations.size() );
        assertEquals(
                "must be greater than or equal to 2",
                constraintViolations.iterator().next().getMessage()
        );
    }

    @Test
    public void carIsValid() {
        Car car = new Car( "Morris", "DD-AB-123", 2 );

        Set<ConstraintViolation<Car>> constraintViolations =
                validator.validate( car );

        assertEquals( 0, constraintViolations.size() );
    }
}
```

在 `setUpValidator()` 方法中，从 `ValidatorFactory` 中取出了一个 `Validator` 对象。一个 `Validator` 实例是线程安全的，并且可以
被多次使用。所以能把它安全的放在一个静态变量中，然后在测试方法中用它来多次校验不同的 `Car` 实例。

`Validate()` 方法会返回一个 `ConstraintViolation` 实例的集合，你能够遍历这个集合，来查看哪个校验错误发生了。前三个测试方法展示了一
我们可预见的校验错误：

- 在 `manufacturerIsNull()` 中 `manufacture` 违反了非空 `@NotNull` 的约束
- 在 `licensePlateTooShort()` 中 `licensePlate` 违反了大小 `@Size` 的约束
- 在 `seatCountTooLow()` 中 `seatCount` 违反了最小值 `@Min` 的约束

如 `carIsValid()` 中所示，如果对象校验成功，那么 `validate()` 将会返回一个空集。

注意到以上的代码只使用了 `javax.validation` 包中的类。这些是 Bean Validation API 提供的。没有 Hibernate Validator 中的类被直接
使用，这让代码可移植行更好。

## 1.4. 接下来会有什么？

我们结束了对 Hibernate Validator 和 Bean Validation 的5分钟探索。如果需要继续查看代码例子或者看更多的例子，请参考 [第14章，更多阅读]()。

如果你想要了解更多关于 bean 和属性的校验问题，请继续阅读 [第2章，声明并校验 bean 上的约束](#2-声明并校验-bean-上的约束)。如果你想要了解使用 Bean Validation 对
方法进行前置或后置校验，请参考 [第3章，声明并校验方法上的约束]()。如果你想在你的应用中加上自定义的校验，你可以看看 [第6章，创造自定义约束]()。

# 2. 声明并校验 bean 上的约束





















