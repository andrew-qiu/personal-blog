---
title: Intellij IDEA自定义代码模板
date: 2018-02-17 14:01:40
categories: Misc
tags: [Java, Intellij IDEA, Live Template, SLF4J, Logger]
#cover: banner-write-in-es6.jpg
---

相信Java码农们大多都有过反复敲写这段代码的心酸经历：

```java
public static void main(String[] args) {
    System.out.println("Hello World");
}
```

不过，这种糟心的状况在各位接触IDE工具以后得到了较好改善。的确现在只需输入`psvm`、`sout`等提示性关键字，再按下`TAB`键，便能在短短几秒内自动生成上述代码段。是不是觉得黑科技感十足？接下来，就一起探寻下 **Live Template** 是如何影响程序狗的日常工作哒。

# 初识神器

首先打开菜单 **File -> Settings**，然后找到 **Editor** 目录下的 **Live Template** 子项，便可找到`psvm`、`sout`等关键字的代码模板定义。是不是发现也没想象中的那么神秘 ⊙▂⊙

![Live Template](settings-live-template.jpg)

任意点选其中一项，不难注意到以下几个关键信息：

- `Abbreviation`表示模板缩略关键字（或称“缩写”），与`psvm`、`sout`作用等同
- `Description`用于编写对该项模板的详细描述内容，方便日后检索识别
- `Template text`用于编写代码模板

# 实例挑战

由于IDEA将缩写词`log`默认定义为如下形式：

```java
private static final Logger logger = Logger.getLogger($CLASS$.class)
```

如果直接使用它，IDE通常会尝试导入标准库`java.util.logging`中的Logger类。但在大多数应用场景下，Slf4j+Logback这类框架组合还是要更为流行一些。这时如果再沿用这段代码模板，将让努力变成徒劳。那么问题就来了，如何对模板进行改造才好使呢？

## 预定义变量

代码模板中使用到的预定义变量（以美元符号夹在中间的形式表现出来），都不是凭空创造出来的，而是在下图所示的地方自行定义滴：

![Live Template Variables](live-template-variables.jpg)

其中，

- `Name`代表模板变量名称，例如：这里定义为**CLASS_NAME**
- `Expression`代表模板变量与之对应的[取值表达式](https://www.jetbrains.com/help/idea/2016.2/live-template-variables.html)，例如：通过**className()**获取当前文件的完整类名
- `Default value`代表模板变量的默认值，它会在**取值表达式**为空时生效

## 动态模板

紧接着引用上述的`$CLASS_NAME$`模板变量，根据实际采用的日志框架可将模板内容修改为：

* **Slf4j**

```java
private final static org.slf4j.Logger LOGGER = org.slf4j.LoggerFactory.getLogger($CLASS_NAME$.class);
```

* **Log4j**

```java
private static final org.apache.log4j.Logger log = org.apache.log4j.Logger.getLogger($CLASS_NAME$.class);
```

至此，自定义代码模板的操作方法阐述完毕，赶紧亲自动手尝试下吧！

# 参考文章

* [Generate log line with Intellij and live templates](https://agileek.github.io/intellij/java/2015/03/01/intellij-live-template-log/)
* [Stack OverFlow: Intellij Live Template](https://stackoverflow.com/questions/6097646/intellij-live-template)
