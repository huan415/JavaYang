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
 <properties>
        <yangyc.version>1.0.0</message.version>
        <revision>${yangyc.version}-SNAPSHOT</revision>
        <maven.build.timestamp.format>yyyyMMdd</maven.build.timestamp.format>
        <java.version>1.8</java.version>
        <spring-cloud.version>2020.0.3</spring-cloud.version>
    </properties>
```

## flatten-maven-plugin 插件

flatten-maven-plugin 作用是在生成压缩包的.flattened-pom.xml（此时将`${revision}`等占位符替换掉），在isntall或deploy时用.flattened-pom.xml替换原有的pom.xml

build-helper-maven-plugin作用：${maven.build.timestamp}默认是UTC时区，使用build-helper-maven-plugin自定义时区和时间格式

```
<build>
        <plugins>
         <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>build-helper-maven-plugin</artifactId>
                <version>1.8</version>
                <executions>
                    <execution>
                        <id>timestamp-property</id>
                        <goals>
                            <goal>timestamp-property</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <name>build.time</name>
                    <pattern>yyyyMMdd</pattern>
                    <timeZone>GMT+8</timeZone>
                    <regex/>
                    <source/>
                    <value/>
                </configuration>
            </plugin>
             <!--替换占位符-->
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

![maven-version-controller-1](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/maven-version-controller-1.jpg)

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
            <revision>${yangyc.version}-SNAPSHOT</revision>
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
            <!--${maven.build.timestamp}默认是UTC时区，使用build-helper-maven-plugin自定义时区和时间格式-->
            <revision>${yangyc.version}-${maven.build.timestamp}-beta</revision>
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
            <revision>${yangyc.version}</revision>
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

![maven-version-controller-2](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/maven-version-controller-2.jpg)



## 踩坑

1. <revision> 是关键字，不能随意替换
   ![maven-version-controller-3](https://raw.githubusercontent.com/huan415/JavaYang/master/assets/maven-version-controller-3.jpg)
2.  <build>没有引用flatten-maven-plugin，业务组件在引入jar后, install时会报错，原因上面有说明

```java
Could not transfer artifact com.javayang.test.yang:service:pom:${revision} from/to nexus (http://xx.xx.xx.xx:xxxx/repository/public/): Failed to transfer file http://xx.xx.xx.xx:xxxx/repository/public/com/javayang/test/yang/service/$%7Brevision%7D/service-$%7Brevision%7D.pom with status code 400

```

