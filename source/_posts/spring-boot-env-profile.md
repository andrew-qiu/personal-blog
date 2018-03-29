---
title: Spring Boot多环境配置资源装载方案
date: 2018-02-23 21:39:42
categories: Java
tags: [Spring Boot, Profile, YAML]
#cover: banner-write-in-es6.jpg
---

Spring Boot配置文件除了存在公有部分外，有时还要根据不同环境作差异化定制，例如：数据库连接、应用端口号等等。以下是常见的配置结构：

|        字段名         |      用途      |
| :------------------: | :-----------: |
|    applcation.yml    | 与环境无关的配置 |
| application-dev.yml  |    开发环境    |
| application-beta.yml |    测试环境    |
| application-prod.yml |    生产环境    |

另外一种情况是，Logback、Dubbo这种中间件配置通常独立存在，即无法整合到`applcation-*.yml`文件中，得结合Maven Profile构建机制来实现差异化定制。那么问题就来了，如果要在应用中激活某个Profile，如何让Spring和Maven同步生效呢？

![Maven目录结构](spring_boot_dir.png)

# 实现原理分析

如果要在Spring Boot中激活某个Profile，一般在`applcation.yml`中指定`spring.profiles.active`配置项。例如：

```yaml
spring:
  profiles: 
    active: beta
```

表示Beta环境配置（applcation-beta.yml）将在运行期间生效。同样的，要在Maven中激活Beta环境资源，可在`pom.xml`文件中预先指定如下内容：

```xml
<properties>
  <profiles.dir>src/profiles</profiles.dir>
</properties>

<profiles>
  <profile>
    <id>beta</id>
    <build>
      <resources>
          <resource>
              <directory>${profiles.dir}/beta</directory>
          </resource>
      </resources>
    </build>
  </profile>
</profiles>
```

如果要实现两者保持统一，`applcation.yml`文件中的`spring.profiles.active`就必须抽取成变量形式。然后在初次选择Maven Profile、或发生变更时，同步地替换为指定标识（比如这里的`beta`字样）。

## Spring配置改造

首先，将`spring.profiles.active`抽取成占位符形式：

```xml
spring:
  profiles:
    active: @spring.profiles.active@
```

其中，`@`符号可以替换为任意值。这里使用它是为了与`$`占位符形成区分，避免标识符冲突发生。

## Maven配置改造

然后，利用[Maven Resources Plugin插件](http://maven.apache.org/plugins/maven-resources-plugin/)中的过滤机制，替换资源文件中指定的占位符。具体做法是，在`pom.xml`中引入maven-resources-plugin插件：

```xml
<properties>
  <resource.delimiter>@</resource.delimiter>
</properties>

<build>
  <plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-resources-plugin</artifactId>
        <version>2.6</version>
        <configuration>
            <delimiters>
                <delimiter>${resource.delimiter}</delimiter>
            </delimiters>
            <useDefaultDelimiters>false</useDefaultDelimiters>
        </configuration>
    </plugin>
  </plugins>
</build>
```

这里设置了期望分隔符为`@`，也就是说`applcation.yml`文件中类似`@...@`的字符串会被替换。至于它们会被替换成什么值，这完全取决于properties节点下这些字段是如何指定的：

```xml
<profiles>
    <profile>
        <id>beta</id>
        <!--
          maven-resources-plugin插件扫描到 @spring.profiles.active@ 占位符时，会在这里搜寻是否存在可用的键值对，
              如果存在则用它进行替换
        -->
        <properties>
            <spring.profiles.active>beta</spring.profiles.active>
        </properties>
        <build>
            <resources>
                <resource>
                    <directory>${profiles.dir}/beta</directory>
                </resource>
            </resources>
        </build>
    </profile>
</profiles>
```

到此位置，就基本实现了占位符替换功能，能够保持Maven/Spring Profile同时激活生效、或切换生效。

# 性能优化

虽然上述方案能够基本满足需求，但是Maven Resources Plugin插件会试图替换`<resources>...</resources>`和`<testResources>...</testResources>`目录中的所有资源文件，可能造成编译阶段的性能问题。因此，要将插件的占位符替换行为限定在YAML文件上：

```xml
<build>
  <finalName>nova</finalName>
  <resources>
    <!-- 对于指定文件开启占位符替换，并将替换结果拷贝到target目录下面 -->
    <resource>
      <directory>src/main/resources</directory>
      <filtering>true</filtering>
      <includes>
        <include>**/*.yml</include>
      </includes>
    </resource>
    <!-- 其他文件直接复制到target目录下面 -->
    <resource>
      <directory>src/main/resources</directory>
      <filtering>false</filtering>
    </resource>
  </resources>
</build>
```

# 参考文章

* [Spring Boot Profile使用](http://blog.javachen.com/2016/02/22/profile-usage-in-spring-boot.html)
* [Maven Resources Plugin - Filtering](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)
