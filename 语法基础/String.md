# String

## String长度限制

1. 字符串常量池限制65535（编译期）
2. 长度不能超过Integer.MAX_VALUE（运行期）

## Sting存储位置

   url:  https://mp.weixin.qq.com/s/TeXBo1CSYinevog2TwAaUg

1. String
   * 编译器已经能确定的 ----- 在常量池（“huan415”）
   * 运行期才能够确定的 ----- 在堆中（new String(“huan415”)）
     new 的时候先看看常量池有没有，
           有：就在只在堆中创建对象，并堆指向常量池
            没有：就在堆和常量池中创建对象，并堆指向常量池
2. 基本数据类型
   * 引用和变量-----栈中
   * 常量存----常量池