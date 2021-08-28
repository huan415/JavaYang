# @Autowired、 @Resource, @Inject 三个注解的区别



##  3种依赖注入

### 基于 field 注入（属性注入）

属性上加入注解
本质：通过反射方式注入到field

```java
@Autowired
private TestService testService;
```

                1. 

### 基于 setter 注入（set方法注入）

set方法上面加注解（spring 4.3以后可以不加@Autowired注解）

```java
@Autowired
public void setTestService(TestService testService) {
    this.testService = testService;
}
```



### 基于 constructor 注入 （构造器注入）

把所需的依赖放到构造方法参数中，并在构造方法中进行初始化（spring 4.3以后可以不加@Autowired注解）

```java
private TestService testService;
@Autowired
public UserService(@Qualifier("testService") TestService testService) {
    this.testService = testService;
}
```



### 区别

field 注入最为常见，其优缺点如下：

优点： 简单（最为常见）
缺点：

                1. 容易违背单一职责原则，容易给一个类里加了很多的依赖（如果使用构造器，参数太多，容易发现问题）
                2.  依赖注入与容器耦合。实例化和依赖注入必须依靠容器，不能自己为其提供依赖（构造器注入或者set注入可以自己new并且提供所需依赖）
                3. final不能使用属性注入

综上所述，Spring建议：

​       依赖注入优先选用构造器注入（方便单一职责发现问题）
​       可选、可变的依赖注入用setter注入
​        不建议使用filed注入

## @Autowired

1. 装配顺序：
   （1）按type匹配
   （2）按name匹配（如果按type发现有多个bean）
   （3）按@Qualifier指定的name匹配（有@Qualifier）
             按变量名匹配（没有@Qualifier）
    （4）以上都匹配不上，则报错（@Autowired(required=false)不会报错）
2. 

## @Resource(name和type)

1. 装配顺序：
   （1）指定name，按name匹配，找不到抛异常
   （2）指定type，按type匹配，找不到抛异常
   （3）指定name和type，按name和type匹配，找不到抛异常
    （4）没有指定name和type，按变量名匹配，找不到抛异常
2. 

## @Inject 

和@Autowired一样（没有required=false）





参考： https://mp.weixin.qq.com/s/bb5ImUZFjjmsOwmcPZR2yw