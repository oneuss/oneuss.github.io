---
layout: post
title:  "一个XML解析错误的排查"
subtitle: "使用JAXB解析xml"
date:   2019-04-30
background: '/img/imac_bg.png'
---
项目里面需要用到XML解析，我权衡之后使用了JAXB，因为：1. 对接系统返回的XML比较复杂，如果使用时候使用DOM4J之类的，代码与XML格式强耦合，所以使用JAXB，直接使用注解映射XML和JavaBean；2. JDK原生支持的。
但是烦恼就此开始，XML和JavaBean转换时倒是容易：
```
    public static <T> T convertXmlToJavaBean(Class<T> clazz, String xmlString) throws Exception {
        try(StringReader stringReader = new StringReader(xmlString)) {
            return JAXB.unmarshal(stringReader, clazz);
        } catch (Exception e) {
            e.printStackTrace();
            throw new Exception("XML 格式错误!");
        }
    }

    public static <T> String convertJavabeanToXml(T bean) throws Exception {
        try(StringWriter stringWriter = new StringWriter()) {
            JAXB.marshal(bean, stringWriter);
            return stringWriter.toString();
        } catch (Exception e) {
            e.printStackTrace();
            throw new Exception(e.getMessage());
        }

    }
```
不过，根据每个接口的XML格式写JavaBean类把我累够呛。好在最后还是写完了。
JAXB相关的注解参考这里，比较详细，[JAXB常用注解讲解(超详细)]([https://blog.csdn.net/wn084/article/details/80853587](https://blog.csdn.net/wn084/article/details/80853587)
)。
这些倒还好说，就是体力活，不过最后测试时候出现了这个错误。
```Feature 'http://javax.xml.XMLConstants/feature/secure-processing' is not recognized.```
队列，忘记说了，项目是Maven项目，且是我接手别人的项目，项目依赖了不少包。我在另一个非Maven项目里面执行时候是没有问题，所以显然是执行环境的问题。
Debug进去，发现了报错的位置：
```com.sun.xml.internal.bind.v2.util.XmlFactory.createParserFactory()```
```factory.setFeature("http://javax.xml.XMLConstants/feature/secure-processing", !isXMLSecurityDisabled(disableSecureProcessing));```
正是此句代码。```SAXParserFactory```是个抽象工厂类，由```newInstance```提供实现类。
```
    public static SAXParserFactory newInstance() {
        return FactoryFinder.find(
                /* The default property name according to the JAXP spec */
                SAXParserFactory.class,
                /* The fallback implementation class name */
                "com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl");
    }
```
find方法比较长，本次错误发生时候工厂实现类是由这句提供：
```
T provider = findServiceProvider(type);
if (provider != null) {
   return provider;
}
```
所以直接进到这个方法里面
```
    /*
     * Try to find provider using the ServiceLoader API
     *
     * @param type Base class / Service interface  of the factory to find.
     *
     * @return instance of provider class if found or null
     */
    private static <T> T findServiceProvider(final Class<T> type) {
        try {
            return AccessController.doPrivileged(new PrivilegedAction<T>() {
                public T run() {
                    final ServiceLoader<T> serviceLoader = ServiceLoader.load(type);
                    final Iterator<T> iterator = serviceLoader.iterator();
                    if (iterator.hasNext()) {
                        return iterator.next();
                    } else {
                        return null;
                    }
                 }
            });
        } catch(ServiceConfigurationError e) {
            // It is not possible to wrap an error directly in
            // FactoryConfigurationError - so we need to wrap the
            // ServiceConfigurationError in a RuntimeException.
            // The alternative would be to modify the logic in
            // FactoryConfigurationError to allow setting a
            // Throwable as the cause, but that could cause
            // compatibility issues down the road.
            final RuntimeException x = new RuntimeException(
                    "Provider for " + type + " cannot be created", e);
            final FactoryConfigurationError error =
                    new FactoryConfigurationError(x, x.getMessage());
            throw error;
        }
    }
```
这个方法就是在当前线程的ContextClassLoader路径上找需要的类。

而在执行成功的环境上执行的这句的结果是：
```T provider = findServiceProvider(type);``` null，所以进行到这句：
```return newInstance(type, fallbackClassName, null, true);```

也就是说，区别在于findServiceProvider(type)方法的执行结果，也就是由```ServiceLoader.load(type)```执行结果决定。
就是与类加载有关系。
翻看小项目依赖，找到了一个叫```XOM:1.0```的解析包，这个包依赖于```xercesImpl:2.6.2```，```ServiceLoader.load(type)```就是就是加载到了这个包的类，所以解决办法就是去掉这个依赖，或者是和我一样升级到了```XOM:1.3.2```，```xercesImpl```对应升级到了```xercesImpl:2.8.0```，问题便解决了。
问题现在是解决了，但是我有个疑问，根据双亲委派机制，应该会先到JDK路径找的吧，为什么会先加载扩展包的实现类呢？我对双亲委派模型本来就有点迷，更加迷了，改日细细研究一番。
