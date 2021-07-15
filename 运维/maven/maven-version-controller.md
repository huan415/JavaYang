# 根据环境打包不同version的jar

## 父级pom.xml

Maven 3.5.0-beta-1版本开始，就可以使用`${revision}`, `${sha1}` 和 `${changelist}`作为占位符来替换pom文件了

```xml
 <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
    </parent>
    <groupId>com.javayang.test</groupId>
    <artifactId>yang</artifactId>
    <version>${revision}</version>
    <packaging>pom</packaging>
    <properties>
        <message.version>1.0.1</message.version>
        <message.package.time>20210411</message.package.time>
        <revision>${yang.version}-SNAPSHOT</revision>
        <java.version>1.8</java.version>
        <spring-cloud.version>2020.0.3</spring-cloud.version>
    </properties>
```

## 自己pom.xml

这样子集引用父级的revision，复用revision，省得到处声明，而且容易改漏

```xml
<parent>
       <groupId>com.javayang.test</groupId>
        <artifactId>yang</artifactId>
        <version>${revision}</version>
    </parent>
    <groupId>com.javayang.test</groupId>
    <artifactId>yang-service</artifactId>
    <version>${revision}</version>
```

## flatten-maven-plugin 插件

flatten-maven-plugin 作用是在生成压缩包的.flattened-pom.xml（此时将`${revision}`等占位符替换掉），在isntall或deploy时用.flattened-pom.xml替换原有的pom.xml

```
<build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>flatten-maven-plugin</artifactId>
                <version>1.1.0</version>
                <configuration>
                    <updatePomFile>true</updatePomFile>
                    <flattenMode>resolveCiFriendliesOnly</flattenMode>
                </configuration>
                <executions>
                    <execution>
                        <id>flatten</id>
                        <phase>process-resources</phase>
                        <goals>
                            <goal>flatten</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>flatten.clean</id>
                        <phase>clean</phase>
                        <goals>
                            <goal>clean</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

效果：

![1](D:\project\huan415\1.jpg)

## 利用环境动态设置revision

不同的环境install、deploy时生成不同版本的jar

```
<profiles>
    <profile>
        <id>snapshot</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <revision>${message.version}-SNAPSHOT</revision>
        </properties>
        <!--不同环境指定不同的版本-->
        <dependencyManagement>
            <dependencies>
               -----------省略---------
            </dependencies>
        </dependencyManagement>
    </profile>
    <profile>
        <id>beta</id>
        <properties>
            <revision>${message.version}-${message.package.time}-beta</revision>
        </properties>
        <!--不同环境指定不同的版本-->
        <dependencyManagement>
            <dependencies>
               -----------省略---------
            </dependencies>
        </dependencyManagement>
    </profile>
    <profile>
        <id>release</id>
        <properties>
            <revision>${message.version}</revision>
        </properties>
        <!--不同环境指定不同的版本-->
        <dependencyManagement>
            <dependencies>
               -----------省略---------
            </dependencies>
        </dependencyManagement>
    </profile>
</profiles>
```

效果：

![2](D:\project\huan415\2.jpg)



## 踩坑

1. <revision> 是关键字，不能随意替换
   ![3](D:\project\huan415\3.jpg)
2.  <build>没有引用flatten-maven-plugin，业务组件在引入jar后, install时会报错，原因上面有说明

```java
Could not transfer artifact com.javayang.test.yang:service:pom:${revision} from/to nexus (http://xx.xx.xx.xx:xxxx/repository/public/): Failed to transfer file http://xx.xx.xx.xx:xxxx/repository/public/com/javayang/test/yang/service/$%7Brevision%7D/service-$%7Brevision%7D.pom with status code 400

```

