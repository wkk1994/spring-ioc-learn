# Spring国际化

## Spring国际化的使用场景

* 普通国际化场景：主要使用Spirng国际化的场景获取文本文案，使用比较少。
* Bean Validation：校验国际化文案。
* Web站点页面渲染：根据不同的语言对站点的内容进行本地化的展示。
* Web MVC错误小心的显示：比如在不同的地区或者语言下，对错误信息做本地化的展示。

## Spring国际化接口

Spring国际化的核心接口：`org.springframework.context.MessageSource`，它有两个开箱即用的实现类，分别是：`org.springframework.context.support.ResourceBundleMessageSource`和`org.springframework.context.support.ReloadableResourceBundleMessageSource`。

与接口相关的主要概念有：

* 文案模板编码（code）
* 文案模板参数（args）
* 区域（Locale）

## 层次性的MessageSource

Spring层次性接口回顾：
Spring中提供了很多层次性接口的实现，之间有提到的有：

* org.springframework.beans.factory.HierarchicalBeanFactory

  HierarchicalBeanFactory中提供了getParentBeanFactory方法可以获取到父BeanFactory。

* org.springframework.context.ApplicationContext

  ApplicationContext提供了getParent方法，可以获取到父ApplicationContext。

* org.springframework.beans.factory.config.BeanDefinition

  BeanDefinition提供了getParentName方式可以获取父ParentName。

Spring层次性国际化接口：

Spring国际化接口也提供了层次性的实现，实现类：`org.springframework.context.HierarchicalMessageSource`，它提供了getParentMessageSource方法可以获取到父MessageSource。

> 双亲委派：和Java的ClassLoader的双亲委派机制不同，Spring实现的双亲机制，是从下向上查找。双亲委派并没有规定一定要由上至下查找。

## Java国际化标准实现

Java国际化实现的核心是抽象类`java.util.ResourceBundle`。

ResourceBundle可以看作是资源的提供者，提供了java.util.ResourceBundle#getString方法可以根据key获取到指定的vaue值。它也提供了多个静态并且重载的getBundle方法，可以根据baseName、locale获取对应的ResourceBundle实现。

ResourceBundle默认提供了2种实现类。

* 基于Properties资源实现的`java.util.PropertyResourceBundle`

  它是比较常用的ResourceBundle实现。

* 例举实现的`java.util.ListResourceBundle`

  ListResourceBundle使用的比较少，它是硬编码的形式，通过getContents方法返回一个二维数组作为资源的实现。

**ResourceBundle核心特性**

* key-value的设计模式：资源的内部形式都是key-value的形式，code对应value。

* 层次性设计：和Spring的MessageSource类似也是层次性设计。

* 缓存设计

  在获取ResourceBundle实现的操作中进行了缓存设计，会先从缓存中获取。缓存的实现使用ConcurrentHashMap，保证了线程安全。缓存的key使用`java.util.ResourceBundle.CacheKey`，CacheKey的组成有baseName、locale、ClassLoader。

  > 为什么要加上ClassLoader？
  ResourceBundle的实现加载有些依赖于ClassLoader，不同的ClassLoader可以加载的classpath的路径可能不通，为了避免不同ClassLoader加载了不同的ResourceBundle实现的重复问题，所以要CacheKey要加上ClassLoader。

* 字符编码控制 - java.util.ResourceBundle.Control（@since 1.6）
  
  默认读取的编码方式是ISO-8859-1，可以在ResourceBundle#getBundle中传递Control指定编码方式。

* Control的SPI扩展机制：java.util.spi.ResourceBundleControlProvider（@since 1.8）

  通过SPI机制，实现ResourceBundleControlProvider#getControl方法，就可以根据不同的baseName获取到不同的Control。具体细节参考`java.util.ResourceBundle#getDefaultControl`。

## Java文本格式化

Java文本格式化的核心接口是`java.text.MessageFormat`。

* 基本用法：
  * 设置消息格式模式- new MessageFormat(...)
  * 格式化 - format(new Object[]{...})
* 消息格式模式
  * 格式元素：{ArgumentIndex (,FormatType,(FormatStyle))}
  * FormatType：消息格式类型，可选项，每种类型在 number、date、time 和 choice 类型选其一
  * FormatStyle：消息格式风格，可选项，包括：short、medium、long、full、integer、currency、percent

* 高级特性
  * 重置消息格式模式
  * 重置 java.util.Locale
  * 重置 java.text.Format

MessageFormatDemo示例代码：[MessageFormatDemo.java](https://github.com/wkk1994/spring-ioc-learn/blob/master/i18n/src/main/java/com/wkk/learn/spring/ioc/i18n/MessageFormatDemo.java)

**必须要重置Pattern，新设置的Locale才能生效?**

MesaageForamt 类定义了一个 Format[] formats 数组用于保存当前 pattern 格式化时所使用的 java.text.Format 实例.而这个数组的赋值在 java.text.MessageFormat#makeFormat(） 方法中， 但这个方法的唯一调用者是java.text.MessageFormat#applyPattern()方法。所以当调用applyPattern()方法格式化新的文案，这个方法会根据Locale变量解析好Format并保存在formats 中，当再这个方法之后再调用setLocale()方法更改地区，是不会触发 formats 数组重新赋值的， 只有等到再次调用 applyPattern()时， 设置的 Locale才会生效。**不过如果定义的文本没有指定FormatType，就没有对应的format，这时会根据参数的类型进行格式化，这样新设置的locale会生效。**

```java
if (obj == null) {
    arg = "null";
} else if (formats[i] != null) {
    subFormatter = formats[i];
    if (subFormatter instanceof ChoiceFormat) {
        arg = formats[i].format(obj);
        if (arg.indexOf('{') >= 0) {
            subFormatter = new MessageFormat(arg, locale);
            obj = arguments;
            arg = null;
        }
    }
} else if (obj instanceof Number) {
    // format number if can
    subFormatter = NumberFormat.getInstance(locale);
} else if (obj instanceof Date) {
    // format a Date if can
    subFormatter = DateFormat.getDateTimeInstance(
             DateFormat.SHORT, DateFormat.SHORT, locale);//fix
} else if (obj instanceof String) {
    arg = (String) obj;

} else {
    arg = obj.toString();
    if (arg == null) arg = "null";
}
```