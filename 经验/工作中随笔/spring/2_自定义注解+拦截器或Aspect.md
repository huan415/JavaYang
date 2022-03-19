# 自定义注解+拦截器或Aspect

## 背景

以前也有使用过自定义拦截器，一般都是参考项目中已有的自定义注解，然后改一改，最近发现项目中有两种写法：1. 拦截器；2：Aspect； 以前只是用，没有过多的去理解，现在记录对比一下



![image-20220319162658908](C:\Users\huan415\AppData\Roaming\Typora\typora-user-images\image-20220319162658908.png)

```java
@Documented
@Inherited
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface YycAnnotation {
}


@Slf4j
@RestController
@RequestMapping("/test")
public class TestController {
    @RequestMapping("/test")
    //@YycAnnotation
    public String query() {
        System.out.println("执行 controller");
        return "Success";
    }
}
```




## 自定义注解+拦截器

步骤：

1. 拦截器实现 HandlerInterceptor
2. **注册拦截器**

```java
@Component
public class TestAnnotationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("执行拦截器 pre TestAnnotationInterceptor");
        return HandlerInterceptor.super.preHandle(request, response, handler);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("执行拦截器 after TestAnnotationInterceptor");
        HandlerInterceptor.super.afterCompletion(request, response, handler, ex);
    }
}
```

```java
@Configuration
@Component
public class InterceptorConfig implements WebMvcConfigurer {

    @Autowired
    public TestAnnotationInterceptor interceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor);
        WebMvcConfigurer.super.addInterceptors(registry);
    }
}
```



## 自定义注解+Aspect

步骤：

1. 声明一个切面类
2. 两种方式：
   * @annotation 指定控制类
   * @annotation 指定的是注解，那么需要在 controller 的方法上加入注解（如上）

```java
@Aspect//表示这是一个切面类
@Component
public class TestAnnotationByAspect {

    @Around("@annotation(YycAnnotation)")//表示在注解处环绕通知
    public Object handler(ProceedingJoinPoint joinPoint) {
        System.out.println("执行切面 start TestAnnotationByAspect");
        Object obj = null;
        try {
            obj = joinPoint.proceed();
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        System.out.println("执行切面 end TestAnnotationByAspect");
        return obj;
    }
}
```

参考：

1. [《SPRING MVC - 你真的懂 【过滤（FILTER）、拦截（INTERCEPTOR）和 切片（ASPECT）】 ？》](https://www.freesion.com/article/1603426305/)
2. [记录请求的耗时（拦截器、过滤器、aspect）](https://www.cnblogs.com/niceyoo/p/10162077.html)