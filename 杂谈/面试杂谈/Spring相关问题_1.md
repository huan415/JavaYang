## Spring相关问题杂谈

## Spring单例Bean会不会有线性安全问题

* 会有线性安全问题。
  因为单例Bean，即bean是一个实例。多线程同时对这个对象的非静态属性操作，会有线性安全问题。
* 解决办法：
  1. 避免可变的成员变量（不建议）
  2. ThreadLocal。将可变的成员变量保存在ThreadLocal里（建议）

## Spring Bean的生命周期



## 声明bean的注解

* @Controller
* @Service
* @Repository
* @Component
* @Bean

## @Component与@Bean区别

|              |                          @Component                          |                            @Bean                             |
| ------------ | :----------------------------------------------------------: | :----------------------------------------------------------: |
| 作用对象     |                           作用于类                           |                          作用于方法                          |
| 生成Bean方式 | @ComponentScan声明扫描路径，扫描到有@Component的类，进行实例化 |                                                              |
| 灵活性       |                                                              | @Bean比较灵活，类型想把第三方的类实例化成bean，<br />需要用@Bean |

