# deploy失败

# 错误信息

```java
ERROR] Failed to execute goal org.apache.maven.plugins:maven-deploy-plugin:2.8.2:deploy (default-deploy) on project yangyc-test: Deployment failed: repository element was not specified in the POM inside distributionManagement element or in -DaltDeploymentRepository=id::layout::url parameter -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
[ERROR] For more information about the errors and possible solutions, please read the following articles:
[ERROR] [Help 1] http://cwiki.apache.org/confluence/display/MAVEN/MojoExecutionException
```



## 配置

### setting.xml

```xml
<servers>
	<server>
		<id>nexus</id>
		<username>yangyc</username>
		<password>yangyc</password>
	</server>
  </servers>

<mirrors>
	<mirror>
        <id>nexus</id>
        <mirrorOf>central</mirrorOf>
        <url>http://xx.xx.xx.xx:xxxx/repository/public/</url>
    </mirror>
	<mirror>
        <id>aliyunmaven</id>
        <mirrorOf>central</mirrorOf>
        <name>aliyun maven</name>
        <url>https://maven.aliyun.com/repository/public </url>
    </mirror>
  </mirrors>
```

### pom.xml

```xml
<groupId>com.javayang.test</groupId>
<artifactId>yangyc-test</artifactId>
<version>1.0.0-20210714-release</version>

..............胜率....................

<distributionManagement>
        <!--正式版本-->
        <!--<repository>
            <id>nexus</id>
            <name>Archiva Releases</name>
            <url>http://xx.xx.xx.xx:xxxx/repository/release/</url>
        </repository>-->
        <!--快照版本-->
        <snapshotRepository>
            <id>nexus</id>
            <name>Nexus Snapshot Repository</name>
            <url>http://xx.xx.xx.xx:xxxx/repository/snapshots/</url>
        </snapshotRepository>
    </distributionManagement>
```

pom.xml中的id要与setting.xml中的id匹配（打包时根据id到setting.xml中找用户名密码）



## deploy 失败原因分析

snapshot快照仓库、release正式仓库

maven打包时根据<version>是否包含-SNAPSHOT，来判断是快照版本或者正式版本，然后将其发布到对应的仓库中（-SNAPSHOT发布到快照仓库，否则发布到release正式仓库）。

snapshot快照仓库:  每次都从镜像服务器中下载最新版本（即使不更改版本），这样方便开发

release正式仓库： 某版本已经存在只会使用本地，不会从镜像仓库下载，所以要下载新的版本要清本地仓库或者更改版本

错误原因：如上pom.xml所示，<version>1.0.0-20210714-release</version> 是正式版本，而<repository>被注释掉，所以报错，放开注释即可。

​                   或者改为<version>1.0.0-SNAPSHOT</version> 

