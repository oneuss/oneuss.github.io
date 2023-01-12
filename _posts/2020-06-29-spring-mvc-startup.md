---
layout: post
title:  "SpringWeb容器启动过程"
subtitle: ""
date:   2020-06-29
background: '/img/imac_bg.png'
---

# 概念

要理解IoC、AOP等你只需要理解JavaSE就可以，知道IoC等解决的是什么问题。但是如果要理解SpringWeb、Spring Web-MVC，你还需要了解JavaEE，理解Servlet。
需要理解以下概念：
1. 什么是`web application`?
Web应用程序是可以从Web访问的应用程序。 Web应用程序由Web组件（如Servlet，JSP，Filter等）以及其他元素（如HTML，CSS和JavaScript）组成。 Web组件通常在Web服务器中执行并响应HTTP请求。
2. 什么是`Servlet`？
Servlet是在服务器上运行的小程序。 这个词是对应于Java applet创造的。Java applet是一个小程序，与Web（HTML）页面一起作为单独的文件发送，在客户端上运行。
Servlet技术用于创建Web应用程序（位于服务器端并生成动态Web页面）。
根据上下文的不同，可以用多种方式描述Servlet。[[来源：https://www.javatpoint.com/servlet-tutorial]](https://www.javatpoint.com/servlet-tutorial)
    * Servlet是一种用于创建Web应用程序的技术。
    * Servlet是提供许多接口和类（包括文档）的API。
    * Servlet是创建任何Servlet都必须实现的接口。
    * Servlet是扩展服务器功能并响应传入请求的类。 它可以响应任何请求。
    * Servlet是一个Web组件，已部署在服务器上以创建动态网页。
3. 什么是`ServletContext`？[【来源】](https://www.quora.com/What-is-ServletContext)
ServletContext是在启动Web应用程序时创建的配置对象。 它包含可以在web.xml中配置的不同初始化参数。
ServletContext是一个有助于与其他Servlet进行通信的接口。 它包含有关Web应用程序和容器的信息。 这是一种应用程序环境。 使用上下文，一个servlet可以获取对资源的URL引用，并存储上下文中其他servlet可以使用的属性。
    * 和`ServletConfig `有什么不同？
        - ServletConfig是每个servlet一个，而ServletContext是每个Web应用程序一个。
        - ServletContext可用于Web应用程序中的所有servlet和jsp，而ServletConfig仅可用于特定的servlet。

4. 什么是`Servlet Container`?
Servlet容器是一个程序，可以接收来自网页的请求并将这些请求重定向到Servlet对象。Servlet负责生成传递到容器的网页文本，容器再将网页返回到发出请求的浏览器。

5. `Web Server`和`Application Server`区别？
    1. `Web Server`仅包含Web或Servlet容器。 它可以用于servlet，jsp，struts，jsf等。不能用于EJB。这是一台可以存储Web内容的计算机。 通常，Web服务器可用于托管网站，但也使用其他一些Web服务器，例如FTP，电子邮件，存储，游戏等。Web服务器包括：Apache Tomcat和Resin。
    2. `Application Server`包含Web和EJB容器。 它可以用于servlet，jsp，struts，jsf，ejb等。它是基于组件的产品，位于以服务器为中心的体系结构的中间层。应用服务器包括：JBoss：来自JBoss社区的开源服务器；Glassfish：由Sun Microsystem提供，现在被甲骨文收购；Weblogic：由Oracle提供，它更安全；Websphere：由IBM提供。

# 一些和启动相关的接口和类

### 看源码过程中首先发现这几个接口/类，似乎与启动相关。如下：

1. ServletContainerInitializer
>该接口允许一个库/运行时被通知Web应用程序的启动阶段，并执行任何所需的servlet、过滤器和监听器的程序化注册以响应它。
这个接口的实现可以用HandlesTypes进行注解，以便接收（在它们的onStartup(java.util.Set<java.lang.Class<?>>, javax.servlet.ServletContext)方法中）实现、扩展或已被注解为由注解指定的类类型的应用类的Set。
如果这个接口的实现没有使用HandlesTypes注解，或者没有一个应用程序类与注解指定的类相匹配，容器必须向onStartup(java.util.Set<java.lang.Class<?>，javax.servlet.ServletContext)传递一个空的类集。
当检查应用程序的类以查看它们是否符合 ServletContainerInitializer 的 HandlesTypes annontation 所指定的任何标准时，如果缺少了应用程序的任何可选的 JAR 文件，容器可能会遇到类加载问题。因为容器无法决定这些类型的类加载失败是否会阻止应用程序正常工作，所以它必须忽略它们，同时提供一个配置选项来记录它们。
该接口的实现必须由位于META-INF/services目录内的JAR文件资源声明，并以该接口的完全限定类名命名，并将使用运行时的服务提供者查找机制或与之语义等同的容器特定机制来发现。在这两种情况下，来自web片段JAR文件的ServletContainerInitializer服务被排除在绝对排序之外的服务必须被忽略，这些服务被发现的顺序必须遵循应用程序的类加载委托模型。
2. SpringServletContainerInitializer
>运作机制
该类将被加载和实例化，并在容器启动期间由任何符合 Servlet 3.0 的容器调用其 onStartup(java.util.Set<java.lang.Class<?>，javax.servlet.ServletContext)方法，假设 spring-web 模块 JAR 存在于 classpath 上。这通过 JAR 服务 API ServiceLoader.load(Class) 方法检测 spring-web 模块的 META-INF/services/javax.servlet.ServletContainerInitializer 服务提供者配置文件来实现。完整的细节请参见 JAR 服务 API 文档以及 Servlet 3.0 Final Draft 规范的 8.2.4 节。
...
与Spring的WebApplicationInitializer的关系
Spring的WebApplicationInitializer SPI只由一个方法组成：WebApplicationInitializer.onStartup(ServletContext)。其签名有意与ServletContainerInitializer.onStartup(Set, ServletContext)非常相似：简单地说，SpringServletContainerInitializer负责将ServletContext实例化并委托给任何用户定义的WebApplicationInitializer实现。然后，每个WebApplicationInitializer负责完成初始化ServletContext的实际工作。委托的具体过程在下面的onStartup文档中详细描述。
一般说明
一般来说，这个类应该被看作是为更重要的、面向用户的WebApplicationInitializer SPI提供支持性基础设施。利用这个容器初始化器也是完全可选的：虽然这个初始化器确实会在所有Servlet 3.0+运行时下被加载和调用，但是否在classpath上提供任何WebApplicationInitializer实现，仍然是用户的选择。如果没有检测到WebApplicationInitializer类型，这个容器初始化器将没有任何效果。
注意这个容器初始化器和WebApplicationInitializer的使用并没有以任何方式与Spring MVC "绑定"，除了这些类型被装在spring-web模块JAR中。相反，它们可以被认为是通用的，因为它们能够方便地对ServletContext进行基于代码的配置。换句话说，任何Servlet、监听器或过滤器都可以在WebApplicationInitializer中注册，而不仅仅是Spring MVC特定的组件。
这个类既不是为扩展而设计的，也不打算被扩展。它应该被认为是一个内部类型，WebApplicationInitializer是面向公众的SPI。

3. WebApplicationInitializer
>将在Servlet 3.0+环境中实现的接口，以便以编程方式配置ServletContext--与传统的基于web.xml的方法相反（或可能与之结合）。
这个SPI的实现将被SpringServletContainerInitializer自动检测到，它本身是由任何Servlet 3.0容器自动引导的。关于这个引导机制的细节，请参见其Javadoc。

![image.png](https://upload-images.jianshu.io/upload_images/13572633-286b36e5f0f94885.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上面的这几个类/接口的Java doc，这几个类在启动时候执行过程如下（首先你需要理解Java SPI）：
容器初始化过程中，当加载`Spring-web`这个jar包时，在jar文件的`META-INF`文件夹下名为`javax.servlet.ServletContainerInitializer`的文件中，指定了`ServletContainerInitializer`的实现类的全限定名`org.springframework.web.SpringServletContainerInitializer`，在该类`@HandlesTypes`注解指定了一个接口`WebApplicationInitializer`。在容器启动时候，`onStartup`方法会被运行，`WebApplicationInitializer`接口的实现类会当作参数传入`onStartup`方法。`SpringServletContainerInitializer`的`onStartup`方法如下：
```java
@Override
public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
		throws ServletException {
  	List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();
  	if (webAppInitializerClasses != null) {
  		for (Class<?> waiClass : webAppInitializerClasses) {
  			if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
  					WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
  				try {
  					initializers.add((WebApplicationInitializer) waiClass.newInstance());
  				}
  				catch (Throwable ex) {
  					throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
  				}
  			}
  		}
  	}
  	if (initializers.isEmpty()) {
  		servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
  		return;
  	}
  	servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
  	AnnotationAwareOrderComparator.sort(initializers);
  	for (WebApplicationInitializer initializer : initializers) {
  		initializer.onStartup(servletContext);
  	}
}
```
`WebApplicationInitializer`接口的非接口和抽象类的子类，它们的`onStartup`方法就会被执行。
似乎Spring容器启动时依靠的就是这几个，但世实际上，看到Spring-web中`WebApplicationInitializer`接口的几个实现类都是抽象类，而且项目中也没写过这个接口的实现类，看来一般都不用这个方式。

### 那再来看看咱们平时在`web.xml`中经常配置的`ContextLoaderListener`类以及与它相关的这几个类：

这几个类的UML图如下：
![image.png](https://upload-images.jianshu.io/upload_images/13572633-faff3c29d13c8762.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. ServletContextListener
 >用于接收ServletContext生命周期变化通知事件的接口。
用于接收ServletContext生命周期变化通知事件的接口。
为了接收这些通知事件，实现类必须在 Web 应用程序的部署描述符中声明，用 WebListener 注解，或者通过 ServletContext 上定义的 addListener 方法之一注册。
该接口的实现在其contextInitialized(javax.servlet.ServletContextEvent)方法中按照声明的顺序被调用，在其contextDestroyed(javax.servlet.ServletContextEvent)方法中按照相反的顺序被调用。
2. ContextLoaderListener
>Bootstrap监听器用于启动和关闭Spring的根WebApplicationContext。简单地委托给ContextLoader以及ContextCleanupListener。
从Spring 3.1开始，ContextLoaderListener支持通过ContextLoaderListener(WebApplicationContext)构造函数注入根Web应用上下文，允许在Servlet 3.0+环境中进行编程配置。参见WebApplicationInitializer的使用示例。
3. ContextLoader
>为根应用程序上下文执行实际的初始化工作，由ContextLoaderListener调用。
在web.xml context-param级别寻找 "contextClass "参数来指定上下文类的类型，如果没有找到，则返回到XmlWebApplicationContext。通过默认的ContextLoader实现，任何指定的上下文类都需要实现ConfigurableWebApplicationContext接口。
处理 "contextConfigLocation "上下文参数，并将其值传递给上下文实例，将其解析为可能的多个文件路径，这些路径可以用任意数量的逗号和空格分隔，例如 "WEB-INF/applicationContext1.xml，WEB-INF/applicationContext2.xml"。也支持ant式的路径模式，例如 "WEB-INF/*Context.xml,WEB-INF/spring*.xml "或 "WEB-INF/**/*Context.xml"。如果没有明确指定，上下文的实现应该使用一个默认的位置（XmlWebApplicationContext："/WEB-INF/applicationContext.xml"）。
注意：在多个配置位置的情况下，以后的bean定义将覆盖之前加载的文件中定义的位置，至少在使用Spring的默认ApplicationContext实现之一时是如此。这可以通过一个额外的XML文件来故意覆盖某些bean定义。
除了加载根应用上下文之外，该类还可以选择加载或获取共享的父上下文并将其挂到根应用上下文上。更多信息请参见 loadParentContext(ServletContext) 方法。

看来`ContextLoaderListener`和SpringMVC的启动密不可分了。根据使用经验，一般`web.xml`中也会做这样的配置：
```xml
<listener>
  <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```
来看下`ServletContextListener`。
```java
public interface ServletContextListener extends EventListener {
    default public void contextInitialized(ServletContextEvent sce) {}
    default public void contextDestroyed(ServletContextEvent sce) {}
}
```
可以看出，这儿是观察者模式（监听器模式、发布订阅模式）的应用。
这里以tomcat的启动为例，启动过程是这样的：web.xml配置了Spring的监听器`ContextLoaderListener`，tomcat等容器启动时候解析web.xml时，将配置的监听器保存起来，在启动阶段调用`listenerStart`方法时，实例化这些监听器，并调用`contextInitialized`方法。我把关键代码择出来，如下：

*tomcat*源码
`ContextConfig`类的`webConfig`及`configureContext`方法
```java
protected void webConfig() {
   //解析web.xml
   //...
   // Step 9. Apply merged web.xml to Context
  if (ok) {
      configureContext(webXml);
  }
}
private void configureContext(WebXml webxml)  {
    //...
    for (String listener : webxml.getListeners()) {
        context.addApplicationListener(listener);
    }
    //...
}
```
`StandardContext`类的方法：
```java
/**
上面的configureContext里面调用的就是这个方法
*/
@Override
public void addApplicationListener(String listener) {
    synchronized (applicationListenersLock) {
        String results[] = new String[applicationListeners.length + 1];
        for (int i = 0; i < applicationListeners.length; i++) {
            if (listener.equals(applicationListeners[i])) {
                log.info(sm.getString("standardContext.duplicateListener",listener));
                return;
            }
            results[i] = applicationListeners[i];
        }
        results[applicationListeners.length] = listener;
        applicationListeners = results;
    }
    fireContainerEvent("addApplicationListener", listener);
}

public boolean listenerStart() {
    //...
    // 实例化所需的监听器
    //这里findApplicationListeners方法获取的listeners[]就是上面addApplicationListener里添加的
    String listeners[] = findApplicationListeners();
    Object results[] = new Object[listeners.length];
    boolean ok = true;
    for (int i = 0; i < results.length; i++) {
        String listener = listeners[i];
        results[i] = getInstanceManager().newInstance(listener);
    }
    //...
    setApplicationLifecycleListeners(lifecycleListeners.toArray());
    //发送应用程序启动事件
    Object instances[] = getApplicationLifecycleListeners();
    if (instances == null || instances.length == 0) {
        return ok;
    }
    ServletContextEvent event = new ServletContextEvent(getServletContext());
    ServletContextEvent tldEvent = null;
    //...
    for (Object instance : instances) {
        if (!(instance instanceof ServletContextListener)) {
            continue;
        }
        ServletContextListener listener = (ServletContextListener) instance;
        //...
        listener.contextInitialized(event);
        //...
    }
    return ok;
}
```
至此容器初始化过程中，`contextInitialized`方法的调用栈比较清楚了，接下来咱们再具体到`contextInitialized`方法看看这个方法的具体执行。
```java
/**
 * Initialize the root web application context.
 */
@Override
public void contextInitialized(ServletContextEvent event) {
	initWebApplicationContext(event.getServletContext());
}
```
这里提到这个方法初始化的是**根应用上下文**，具体各种应用上下文[戳这里](https://www.jianshu.com/p/bd884dd795a1)了解下。
继续看`initWebApplicationContext`方法，这个方法是`ContextLoader`中的方法，上面`ContextLoaderListener`的JavaDoc也说了：
>简单地委托给ContextLoader以及ContextCleanupListener。

先看`initWebApplicationContext`方法的Java doc信息：
>使用构造时提供的应用上下文，或根据 "contextClass "和 "contextConfigLocation " 这两个环境变量参数（就是web.xml中的context-params） 创建一个新的上下文，为给定的servlet上下文初始化Spring的web应用上下文。

这是什么意思呢？就是常用的这个配置：
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/spring-*.xml</param-value>
</context-param>
```
接着看代码：
``` java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    //...
    // 将上下文存储在本地实例变量中，以保证在ServletContext关闭时可用。
    if (this.context == null) {
        this.context = createWebApplicationContext(servletContext);
    }
    if (this.context instanceof ConfigurableWebApplicationContext) {
        ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
        if (!cwac.isActive()) {
            // 上下文尚未刷新-->提供设置父上下文、设置应用上下文id等服务。
            if (cwac.getParent() == null) {
                // 上下文实例在没有显式父级的情况下被注入 -> 如果有的话，确定根Web应用上下文的父级。
                ApplicationContext parent = loadParentContext(servletContext);
                cwac.setParent(parent);
            }
            //关键代码
            configureAndRefreshWebApplicationContext(cwac, servletContext);
        }
    }
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
    ClassLoader ccl = Thread.currentThread().getContextClassLoader();
    if (ccl == ContextLoader.class.getClassLoader()) {
        currentContext = this.context;
    }
    else if (ccl != null) {
        currentContextPerThread.put(ccl, this.context);
    }
    return this.context;
}
```
这里再看下`createWebApplicationContext`方法：
```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    Class<?> contextClass = determineContextClass(sc);
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
    	throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
    			"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

`configureAndRefreshWebApplicationContext`方法
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
    	// The application context id is still set to its original default value
    	// -> assign a more useful id based on available information
    	String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
    	if (idParam != null) {
    		wac.setId(idParam);
    	}
    	else {
    		// Generate default id...
    		wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
    				ObjectUtils.getDisplayString(sc.getContextPath()));
    	}
    }

    wac.setServletContext(sc);
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
    	wac.setConfigLocation(configLocationParam);
    }

    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
    	((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }

    customizeContext(sc, wac);
    wac.refresh();
}
```


### 但是有时候，我们甚至没有做`ContextLoaderListener`相关配置，好像也可以用，那咱们再看下这个机制。
