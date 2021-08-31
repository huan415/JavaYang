# Jmap命令

## Jmap命令

1. 查哪个对象占用比较多
    jmap -histo pid
2. 查看各个代内存大小
   jmap -heap pid
3. 打印堆栈信息
   jmap -dump:format=b,file=canal.hprof pid
4. finalizerinfo 打印正等候回收的对象的信息
   jmap -finalizerinfo pid

## Jstat

       1. gc信息
          jstat -gc pid
       2. gc信息（百分比）
          jstat -gcutil pid
       3. 三个代空间分配大小
          jstat -gccapacity pid
       4. 年轻代信息
          jstat -gcnew pid
       5. 年轻代信息及占用量
          jstat -gcnewcapacity pid
       6. 老年代信息
          jstat -gcold pid
       7. 老年代信息及占用量
          stat -gcoldcapacity pid
       8. perm信息及占用量
          jstat -gcpermcapacity pid
       9. vm执行信息
          jstat -printcompilation pid
       10. class数量及空间情况
           jstat –class pid
       11. vm实时编译信息
           jstat -compile pid

## Jstack

       1. 查看CPU高内存的堆栈信息
          步骤：
          (1) top 查看CPU最高的进程
          (2) top -Hp pid 查看哪个线程占用的CPU最高
          (3)printf "%x\n" tid  线程ID转为16进制格式
          (4)jstack -l tid  > ./filename.stack
          (5)jstack pid |grep tid -A 30   查看线程堆栈信息

## Jinfo

1. jvm的全部参数和系统属性
   jinfo pid
2. 输出某属性的参数
   jinfo -flag name pid





https://blog.csdn.net/chizizhixin/article/details/87877265