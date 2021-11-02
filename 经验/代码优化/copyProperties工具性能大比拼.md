# copyProperties工具性能大比拼

## 背景

接了个优化接口的活儿，从链路跟踪发现项目中有很多的 PropertyUtils.copyProperties()，严重影响性能。
对象复制时：

1. 大量的属性get、set堆积，代码可读性差
2. PropertyUtils.copyProperties() 和 BeanUtils.copyProperties 一句代码解决，很方便，殊不知有个隐藏的坑--影响性能
3. 建议改成cglib的BeanCopier

## copy工具

1. Apache BeanUtils 性能最差
   Apache PropertyUtils 性能比BeanUtils 好
2. springframework BeanUtils  性能：优于Apache BeanUtils
                                                                  略于Apache PropertyUtils
3. cglib的BeanCopier 性能最好，注意：新建BeanCopier耗性能，要复用新建完的copier

原因：

1. BeanUtils、PropertyUtils 反射实现，性能差，尤其是method.invoke()   (Apache 和springframework )
2. Apache BeanUtils主要集中了各种丰富的功能(日志、转换、解析等等),导致性能变差，而springframework的BeanUtils没有这些，所以springframework比Apache 性能好
3. 通过字节码方式转换，性能最好

## 测试性能

![image-20211102163607800](D:\project\aopensource\github\JavaYang\经验\代码优化\assets\image-20211102163607800.png)

```java
package com.javayang.test.test1.copypertiesperf;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.commons.beanutils.PropertyUtils;
import org.springframework.cglib.beans.BeanCopier;

import java.lang.reflect.InvocationTargetException;
import java.util.Date;

/**
 * @author 杨艺聪
 * @version 1.0
 * @description:  验证属性不同: Student-age: int
 *                            Student3-age: String
 *                https://blog.csdn.net/qq_32317661/article/details/84393465
 * @date 2021/11/2
 **/
public class TestCopyAttribute {
    public static void main(String[] args) throws InvocationTargetException, IllegalAccessException, NoSuchMethodException {
        Student source = new Student();
        source.setAge(18);
        source.setName("杨艺聪");
        source.setBirthDate(new Date());
        // 能复制
        testBeanUtil(source);
        // 报错 argument type mismatch
        testPropertyUtil(source);
        // 属性不一样不会复制，例如 age: int -> String
        testBeanCopier(source);

    }

    public static void testBeanUtil(Student source) throws InvocationTargetException, IllegalAccessException {
        Student3 target = new Student3();
        BeanUtils.copyProperties(target,source);
        System.out.println(source.toString());
        System.out.println(target.toString());
    }

    public static void testPropertyUtil(Student source) throws InvocationTargetException, IllegalAccessException, NoSuchMethodException {
        Student3 target = new Student3();
        PropertyUtils.copyProperties(target,source);
        System.out.println(source.toString());
        System.out.println(target.toString());
    }

    public static void testBeanCopier(Student source) {
        Student3 target = new Student3();
        BeanCopier copier = BeanCopier.create(Student.class, Student3.class,false);
        copier.copy(source, target,null);
        System.out.println(source.toString());
        System.out.println(target.toString());
    }
}


```



### 测试配置转换

```java
package com.javayang.test.test1.copypertiesperf;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.commons.beanutils.PropertyUtils;
import org.springframework.cglib.beans.BeanCopier;

import java.lang.reflect.InvocationTargetException;

/**
 * @author 杨艺聪
 * @version 1.0
 * @description: 参考：https://www.jianshu.com/p/bcbacab3b89e
 *                    https://blog.csdn.net/u012500848/article/details/100412631
 * @date 2021/11/2
 **/
public class TestCopyPerf {
    static int count = 10 * 10000;
    public static void main(String[] args) throws InvocationTargetException, IllegalAccessException, NoSuchMethodException {
        Student source = new Student();
        // 性能最差  耗时：3815
        testBeanUtil(source);
        // 性能优于BeanUtil 耗时：287
        // 很明显可以看出有很多debug校验日志： 10:57:36.450 [main] DEBUG org.apache.commons.beanutils.converters.ArrayConverter - Converting 'long[]' value '[J@51cdd8a' to type 'long[]'
        testPropertyUtil(source);
        //优于apche的BeanUtil，略于apache的PropertyUtil   耗时：610
        testSpringBeanUtil(source);
        // 性能最好    耗时：112
        testBeanCopier(source);
    }

    public static void testSpringBeanUtil(Student source) throws InvocationTargetException, IllegalAccessException {
        long start = System.currentTimeMillis();
        Student2 target = new Student2();
        for (int i = 0; i < count; i++) {
            org.springframework.beans.BeanUtils.copyProperties(target,source);
        }
        System.out.println("springframework BeanUtils.copyProperties() copy "+count+"次耗时: "+(System.currentTimeMillis() - start));
    }

    public static void testBeanUtil(Student source) throws InvocationTargetException, IllegalAccessException {
        long start = System.currentTimeMillis();
        Student2 target = new Student2();
        for (int i = 0; i < count; i++) {
            BeanUtils.copyProperties(target,source);
        }
        System.out.println("apache BeanUtils.copyProperties() copy "+count+"次耗时: "+(System.currentTimeMillis() - start));
    }

    public static void testPropertyUtil(Student source) throws InvocationTargetException, IllegalAccessException, NoSuchMethodException {
        long start = System.currentTimeMillis();
        Student2 target = new Student2();
        for (int i = 0; i < count; i++) {
            PropertyUtils.copyProperties(target,source);
        }
        System.out.println("apache PropertyUtils.copyProperties() copy "+count+"次耗时: "+(System.currentTimeMillis() - start));
    }

    public static void testBeanCopier(Student source) {
        long start = System.currentTimeMillis();
        Student2 target = new Student2();
        BeanCopier copier = BeanCopier.create(Student.class, Student2.class,false);
        for (int i = 0; i < count; i++) {
            copier.copy(source, target,null);
        }
        System.out.println("cglib BeanCopier.copy() copy "+count+"次耗时: "+(System.currentTimeMillis() - start));
    }

}

```



### BeanCopier原理

1. BeanCopier.create 生成source -> target代理类，放入jvm中 （这个最耗时，可以缓存）
2. source 的 get 方法，PropertyDescriptor[] getters
3. target 的 set 方法 ，PropertyDescriptor[] setters
4. 遍历 setters 的属性
5. 按setters的name生成sourceClass的所有setter方法 -->  PropertyDescriptor getter
6. PropertyDescriptor[] setters -->  PropertyDescriptor setter
7. setter和getter名字和类型 配对，生成代理类的拷贝方法









参考：https://www.jianshu.com/p/bcbacab3b89e   （性能）
            https://blog.csdn.net/qq_32317661/article/details/84393465  （BeanCopier原理）
           https://blog.csdn.net/u012500848/article/details/100412631 (避免一直BeanCopier.create)