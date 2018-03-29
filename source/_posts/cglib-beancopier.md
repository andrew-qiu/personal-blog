---
title: CGLIB BeanCopier实现高效拷贝的方法总结
date: 2018-01-30 20:00:00
categories: Java
tags: [Java, Bean, ASM]
#cover: banner-write-in-es6.jpg
---

> CGLIB（Byte Code Generation Library）作为一个强大的Java代码生成工具库，实现了对ASM字节码工具的封装与操作简化，可在运行期动态扩展类和实现接口。

由于业务系统分层设计的缘故，经常会遇到PO/BO/VO/DTO等领域对象之间的相互转换，实现各层模型间的信息交换。根据有关博文的评测结果，CGLIB BeanCopier在多数应用场景下具有明显优于DozerMapper、Apache BeanUtils等工具的转换性能，因此决心对其功能点探索一波。

# 基本功能

BeanCopier对象实例化方法如下：

```java
BeanCopier copier = BeanCopier.create(Class source, Class target, boolean useConverter);
```

其中，第三个参数useConverter表示是否引入Converter方式进行转换。

- 如果useConverter等于`false`，仅对属性名和类型完全相同的两种变量进行拷贝，不能针对某几个属性单独拷贝
- 如果useConverter等于`true`，可对某些特定属性值进行定制化转换，比如：四舍五入等操作

下面针对以上两种模式分别举例说明用法：

## Converter关闭模式

> 准备[PayOrderEntity](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-payorderentity-java)和[LoanApplyDto](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-loanapplydto-java)两个测试类，确保两者均生成了字段对应的`Getter`、`Setter`方法，否则会导致拷贝失败。

首先，将PayOrderEntity实例化成源对象、并对各字段赋值：

```java
PayOrderEntity source = new PayOrderEntity();
source.setOrderNo("PAY20170810120512");
source.setPayAmount(new BigDecimal("300.00"));
source.setReduceAmount(new BigDecimal("10.00"));
```

然后，将LoanApplyDto实例化成目标对象，保持各字段值初始为空：

```java
LoanApplyDto target = new LoanApplyDto();
```

最后，利用BeanCopier进行字段拷贝：

```java
final BeanCopier copier = BeanCopier.create(PayOrderEntity.class, LoanApplyDto.class, false);
copier.copy(source, target, null);
```

执行结果：

|    字段名     |       源对象       |       目标对象     |      字段类型    |
| :----------: | :---------------: | :---------------: | :------------:  |
|   orderNo    | PAY20170810120512 | PAY20170810120512 |  同属性名、同类型  |
|  payAmount   |       300.00      |       300.00      |  同属性名、同类型  |
| reduceAmount |       10.00       |        null       | 同属性名、不同类型 |
|    remark    |         -         |        null       | 不同属性名、不同类型 |

不难看出，对于源/目标对象对于同名、同类型字段可以实现正确拷贝，而不满足此条件的字段则无法进行相互拷贝。

## Converter开启模式

通过实现Converter接口，可利用此自定义转换器实现源/目标对象的定制化属性值转换：

```java
package net.sf.cglib.core;  
  
public interface Converter {  
    Object convert(Object value, Class target, Object context);  
}  
```

其中，value表示源对象属性值，target表示目标对象属性类，context表示目标对象setter方法名。

* 具体转换样例如下：

```java
final BeanCopier copier = BeanCopier.create(PayOrderEntity.class, LoanApplyDto.class, true);
copier.copy(source, target, new Converter() {
    @Override
    public Object convert(Object value, Class target, Object context) {
        if (value instanceof BigDecimal && target.isAssignableFrom(String.class)) {
            return value.toString();
        } else {
            return value;
        }
    }
});
```

执行结果：

|    字段名     |       源对象       |       目标对象     |      字段类型    |
| :----------: | :---------------: | :---------------: | :------------:  |
|   orderNo    | PAY20170810120512 | PAY20170810120512 |  同属性名、同类型  |
|  payAmount   |       300.00      |       300.00      |  同属性名、同类型  |
| reduceAmount |       10.00       |       10.00       | 同属性名、不同类型 |
|    remark    |         -         |        null       | 不同属性名、不同类型 |

不难看出，reduceAmount字段已经成功从源对象中拷贝到了目标对象，即便两个对象中定义的属性类型并不相同。

> 一旦使用Converter，BeanCopier只使用Converter定义的规则去拷贝属性，因此convert方法中要考虑所有属性的转换逻辑。

# 扩展功能

CGLIB工具包为了更灵活地满足需求，还提供了BulkBean和BeanMap实现单个属性的定制化拷贝，而非BeanCopier那般把所有属性拷贝一遍。

```java
BulkBean bulkBean = BulkBean.create(Class target, String[] getters, String[] setters, Class[] types);
BeanMap beanMap = BeanMap.create(Object bean);
```

## Bulk Bean

BulkBean将整个拷贝动作拆分为getPropertyValues、setPropertyValues两个方法，使得Bean的取值/修改更加灵活可定制。这里依然引入[PayOrderEntity](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-payorderentity-java)作为样例：

```java
PayOrderEntity source = new PayOrderEntity();
source.setOrderNo("PAY20170810120512");
source.setPayAmount(new BigDecimal("300.00"));
```

利用BulkBean定义测试类中哪些字段可被操作、以及映射关系。例如，PayOrderEntity中允许

```java
final String[] getters = new String[] {"getOrderNo", "getPayAmount"};
final String[] setters = new String[] {"setOrderNo", "setPayAmount"};
final Class[] clazzs = new Class[] {String.class, BigDecimal.class};

final BulkBean bulkBean = BulkBean.create(PayOrderEntity.class, getters, setters, clazzs);
```

* 调用`source`对象中orderNo和payAmount字段的Getter方法，输出结果分别为`[PAY20170810120512, 300.00]`：

```java
Object[] result = bulkBean.getPropertyValues(source);
```

* 调用`source`对象中orderNo和payAmount字段的Setter方法：

```java
bulkBean.setPropertyValues(bean, new Object[] {"CREDIT20180810121", new BigDecimal("150.00")});
```

执行结果：

|    字段名     |       源对象       |       目标对象     |
| :----------: | :---------------: | :---------------: |
|   orderNo    | PAY20170810120512 | CREDIT20180810121 |
|  payAmount   |       300.00      |       150.00      |
| reduceAmount |       10.00       |       10.00       |

## Bulk Map

业务系统经常会碰到POJO转换Map对象的应用场景，比如：报文加签/验签，这时需要用到BeanMap工具类来实现。这里依然引入[PayOrderEntity](https://gist.github.com/andrew-qiu/64de481b1305d5f48c7fc22ef58625d6#file-payorderentity-java)作为样例：

```java
PayOrderEntity source = new PayOrderEntity();
source.setOrderNo("PAY20170810120512");
source.setPayAmount(new BigDecimal("300.00"));
```

然后构建映射对象：

```java
BeanMap beanMap = BeanMap.create(source);
```

此时，beanMap就是期望生成的Map对象，可使用`beanMap.get("orderNo")`获取orderNo字段属性值。

# 参考文章

* [CGLIB Tutorial](https://github.com/cglib/cglib/wiki/Tutorial)
* [关于BeanCopier的一些思考](https://github.com/alibaba/tamper/wiki/%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95)
* [CGLIB中BeanCopier源码实现](https://www.jianshu.com/p/f8b892e08d26)
* [cglib BeanCopier使用](http://blog.csdn.net/liangrui1988/article/details/41802275)
