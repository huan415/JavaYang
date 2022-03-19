# 初熟 ImportSelector

## 背景

最近看到 ImportSelector，觉得很陌生，故去了解了一下。

```java
public class XxxImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"com.xx.xx.xx.xxxAnnotationResolver", "cn.xx.xx.xx.xxxListener"};
    }
}
```



## 使用

### @Import

在注解 @Import 指定 class**（一个个枚举出来）**

```java
@Configuration
@Import({Yyc1.class,Yyc2.class})
public class TestImportConfig {
    public TestImportConfig() {
        System.out.println("创建 TestImportConfig");
    }
}
```



### @Import+ImportSelector

在注解 @Import 指定一个 ImportSelector 的实现类**（类似于批量包装）**，在实现类里指定类

```java
@Configuration
@Import(MyImportSelector.class)
public class TestImportSelectorConfig {
    public TestImportSelectorConfig() {
        System.out.println("创建 TestImportSelectorConfig");
    }
}

public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"com.javayang.test.test1.spring.importSelector.Yyc1","com.javayang.test.test1.spring.importSelector.Yyc2"};
    }
}
```



### SpringBoot 中运用（自动装配）

```java
@SpringBootApplication ==>
    @EnableAutoConfiguration ==>
       @AutoConfigurationPackage
```

AutoConfigurationImportSelector 加载 META-INF/spring.factories 里配置的值

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}

```

Registrar 加载 Application 类所在包及其其子包的类

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};
}
```

参考：

1. [《SpringBoot教程(2) @Import和ImportSelector的使用》](https://blog.csdn.net/winterking3/article/details/114537557)