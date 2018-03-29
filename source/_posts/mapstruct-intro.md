---
title: MapStruct进阶踩坑指南
date: 2018-02-12 13:06:36
categories: Java
tags: [Java, Bean, Maven, Gradle]
# cover: "/images/mapstruct.png"
---

虽然[CGLIB](https://github.com/cglib/cglib)等基于运行时代码生成的解决方案相比Apache BeanUtils等反射类框架具有明显的性能优势，但在某些应用场景下，比如：利用Converter接口实现定制化Bean转换，还是与Getter/Setter的手工拷贝形式存在性能差距。

A. Rey近期做过的一份[O2O Mapping框架评测报告](https://github.com/arey/java-object-mapper-benchmark)显示，MapStruct由于采用了编译时生成代码（Compile-Time Code Generator）的处理方案，使其最终获得了媲美纯手工拷贝的性能得分。而它注解式的编码风格、以及强大的[IDE外围插件支持](http://mapstruct.org/documentation/ide-support/)，让博主有了兴趣尝试一番。

# 依赖安装

> 由于MapStruct[官方安装指导](http://mapstruct.org/documentation/installation/)坑到无底线，所以根据实战经验重新整理了一波安装步骤，避免大家再次掉进去。

首先，务必在编译生命周期内引入Mapstruct Processor，以保证`@Mapper`、`@Mapping`等注解能被正确识别、处理：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.5.1</version>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后，根据实际情况选择引入JDK版本对应的依赖包：

* JDK1.8+（强烈推荐）

```xml
<properties>
    <java.version>1.8</java.version>    <!-- or higher, depending on your project -->
    <org.mapstruct.version>1.2.0.Final</org.mapstruct.version>
</properties>
...
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-jdk8</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>

    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>
```

* JDK1.6/JDK1.7

```xml
<properties>
    <java.version>1.6</java.version>    <!-- Sepecify JDK Version, depending on your project -->
    <org.mapstruct.version>1.2.0.Final</org.mapstruct.version>
</properties>
...
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>

    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>
```

# 同名字段的自动映射

> 准备[PayOrderEntity](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-payorderentity-java)和[LoanApplyDto](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-loanapplydto-java)两个测试类，确保两者均生成了字段对应的`Getter`、`Setter`方法，否则会导致拷贝失败。

编写带有`@Mapper`注解的一个映射接口和对应抽象方法，标记转换关系是由PayOrderEntity映射到LoanApplyDto的。这里要特别注意，MyBatis中也存在`@Mapper`同名注解，导入包路径时不要搞错了！

```java
import org.mapstruct.Mapper;

@Mapper
public interface TradeInfoMapper {

    TradeInfoMapper INSTANCE = Mappers.getMapper(TradeInfoMapper.class);

    // Mapping JavaBean: PayOrderEntity --> LoanApplyDto
    LoanApplyDto tradeInfoTrans(PayOrderEntity payOrderEntity);
}
```

Mapstruct Processor能够在编译阶段实现这类接口，自动对同名字段适用恰当的（隐式）[转换关系](http://mapstruct.org/documentation/stable/reference/html/#datatype-conversions)。这样，便可在调用方引用接口中声明的（静态）成员变量`INSTANCE`和`tradeInfoTrans`方法实施转换，例如：

```java
PayOrderEntity source = new PayOrderEntity();
source.setOrderNo("PAY20170810120512");
source.setPayAmount(new BigDecimal("300.00"));
source.setReduceAmount(new BigDecimal("10.00"));

LoanApplyDto target = TradeInfoDefaultMapper.INSTANCE.tradeInfoTrans(source);
```

执行结果：

|    字段名     |       源对象       |       目标对象     |      字段类型     |
| :----------: | :---------------: | :---------------: | :-------------: |
|   orderNo    | PAY20170810120512 | PAY20170810120512 | 同名字段，显式转换 |
|  payAmount   |       300.00      |       300.00      | 同名字段，显式转换 |
| reduceAmount |       10.00       |       10.00       | 同名字段，隐式转换 |
|    remark    |         -         |        null       | 非同名字段，不转换 |

# 定制转换方法

MapStruct具备非常强大的同名字段映射能力，能够较为合理地处理不同类型间的转换诉求。与此同时，它针对非同名字段转换、字段值修改等个性化诉求也提供了一套定制手段。

## 非同名字段映射转换

接上节，假设我们不使用自动映射、而采用手工指定的策略时，可以引入`@Mappings`或`@Mapping`注解来指定源/目标字段的具体映射关系：

### 单个字段

```java
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

@Mapper
public interface TradeInfoMapper {

    TradeInfoMapper INSTANCE = Mappers.getMapper(TradeInfoMapper.class);

    @Mapping(source = "orderNo", target = "remark")
    LoanApplyDto tradeInfoTrans(PayOrderEntity payOrderEntity);
}
```

预期结果：

|    字段名     |       源对象       |       目标对象     |      字段类型     |
| :----------: | :---------------: | :---------------: | :-------------: |
|   orderNo    | PAY20170810120512 | PAY20170810120512 | 同名字段，显式转换 |
|  payAmount   |       300.00      |       300.00      | 同名字段，显式转换 |
| reduceAmount |       10.00       |       10.00       | 同名字段，隐式转换 |
|    remark    |         -         | PAY20170810120512 |    指定映射关系   |

### 多个字段

```java
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Mappings;

@Mapper
public interface TradeInfoMapper {

    TradeInfoMapper INSTANCE = Mappers.getMapper(TradeInfoMapper.class);

    @Mappings({
        @Mapping(source = "reduceAmount", target = "payAmount"),
        @Mapping(source = "payAmount", target = "reduceAmount")
    })
    LoanApplyDto tradeInfoTrans(PayOrderEntity payOrderEntity);
}
```

预期结果：

|    字段名     |       源对象       |       目标对象     |          字段类型       |
| :----------: | :---------------: | :---------------: | :--------------------: |
|   orderNo    | PAY20170810120512 | PAY20170810120512 |  同名字段，显式转换       |
|  payAmount   |       300.00      |       10.00       |  同名字段，但指定映射关系  |
| reduceAmount |       10.00       |       300.00      |  同名字段，但指定映射关系  |
|    remark    |         -         |        null       | 非同名字段，且未指定映射关系 |

### 特殊逻辑定制

如果要实现字段转换中的一些特殊逻辑，比如：折扣金额（reduceAmount）要根据下单金额（payAmount）的0.1倍计算而来，就要用到默认方法（Java8+）、或抽象类方法来实现。具体可参考[官方手册：增加定制方法](http://mapstruct.org/documentation/stable/reference/html/#adding-custom-methods)章节，这里就不再赘述。

# Mapper实例获取

前面我们都是从`getMapper()`方法获取的Mapper实例，虽然具备单例、线程安全等诸多优点，但还是与Spring Framework、CDI这类依赖注入框架的使用习惯相背离。因此，MapStruct现在也加强了对它们的支持力度。例如：

```java
@Mapper(componentModel = "spring")
public interface TradeInfoMapper {

    // Mapping JavaBean: PayOrderEntity --> LoanApplyDto
    LoanApplyDto tradeInfoTrans(PayOrderEntity payOrderEntity);
}
```

便省去了声明`INSTANCE`成员变量的相关语句，仅仅依靠`@Resource`、`@Autowired`、`@Inject`这类JSR规范注解就能被加载到。当然前提是依赖注入框架支持这类注解使用。

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class TradeInfoMapperTest {

    @Autowired
    private TradeInfoMapper tradeInfoMapper;

    @Test
    public void componentModel() throws Exception {
        PayOrderEntity source = new PayOrderEntity();
        source.setOrderNo("PAY20170810120512");
        source.setPayAmount(new BigDecimal("300.00"));
        source.setReduceAmount(new BigDecimal("10.00"));

        LoanApplyDto target =  tradeInfoMapper.tradeInfoTrans(source);
    }
}
```

# 参考文章

* [MapStruct Reference Guide](http://mapstruct.org/documentation/stable/reference/html/)
* [MapStruct Examples](https://github.com/mapstruct/mapstruct-examples)
