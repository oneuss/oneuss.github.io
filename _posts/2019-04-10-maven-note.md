---
layout: post
title:  "Maven笔记"
subtitle: "转载记录自多篇文章"
date:   2019-04-10
background: '/img/imac_bg.png'
---
使用Maven有一年多，但是很多概念一直很含糊，今天稍作总结。 [转载自多篇文章]
1. [Maven是什么？](https://blog.csdn.net/liusong0605/article/details/25654811)
**1.1 优秀的构建工具**
通过简单的命令，能够完成清理、编译、测试、打包、部署等一系列过程。同时，不得不提的是，Maven是跨平台的，无论是在Windows、还是在Linux或Mac上，都可以使用同样的命令。
**1.2 依赖管理工具**
项目依赖的第三方的开源类库，都可以通过依赖的方式引入到项目中来。代替了原来需要首先下载第三方jar，再加入到项目中的方式。从而更好的解决了合作开发中依赖增多、版本不一致、版本冲突、依赖臃肿等问题。
具体是怎么实现的呢？Maven通过坐标系统准确的定位每一个构件，即通过坐标找到对应的Java类库。
**1.3 项目信息管理工具**
能够管理项目描述、开发者列表、[版本控制](http://lib.csdn.net/base/28 "Git知识库")系统地址、许可证等一些比较零散的项目信息。除了直接的项目信息，通过Maven自动生成的站点，以及一些已有的插件，还能够轻松获得项目文档、测试报告、静态分析报告、源码版本、日志报告等非常具有价值的项目信息。

2. [Nexus是什么？](https://blog.csdn.net/liusong0605/article/details/25654811)
Nexus是一种远程仓库，是私服的一种。

3. [pom文件结构](https://www.cnblogs.com/NYfor2018/p/9070028.html)
3.1 项目坐标，信息描述等
```
<modelVersion>4.0.0</modelVersion>
<groupId>com.maven</groupId>
<artifactId>MavenTest</artifactId>
<packaging>war</packaging>
<version>0.0.1-SNAPSHOT</version>
<name>MavenTest Maven Webapp</name>
<url>http://maven.apache.org</url>
```
modelVersion：pom文件的模型版本。
group id：com.公司名.项目名。
artifact id：功能模块名。
packaging：项目打包的后缀，war是web项目发布用的，默认为jar。
version：artifact模块的版本。
name和url：相当于项目描述，可删除。
group id+artifact id+version：项目在仓库中的坐标。
3.2 引入jar包
```
<dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
</dependencies>
```
dependency：引入资源jar包到本地仓库，要引入更多资源就在<dependencies>中继续增加<dependency>。
group id+artifact id+version：资源jar包在仓库的坐标。
scope：作用范围，test指该jar包仅在maven测试时使用，发布时会忽略这个包。需要发布的jar包可以忽略这一配置。
3.3 构建项目
```
<build>  
    <finalName>MavenTest</finalName>  
    <plugins>  
        <plugin>  
            <groupId>org.apache.maven.plugins</groupId>  
            <artifactId>maven-compiler-plugin</artifactId>  
            <version>3.5.1</version>  
            <configuration>  
                <source>1.7</source>  
                <target>1.7</target>  
                <encoding>UTF-8</encoding>  
            </configuration>  
        </plugin>  
    </plugins>  
</build>
```
build：项目构建时的配置。
finalName：在浏览器中的访问路径，如果将它改成test，再执行maven-update，这时运行项目的访问路径是
         http://localhost:8080/MavenTest/     而不是项目名的 http://localhost:8080/test
plugins：插件，第一个插件用<configuration>--<source>和<target>来设置java版本为1.7，第二个插件用<configuration>--<encoding>设置编码为utf-8.
group id+artifact id+version：插件在仓库中的坐标。
configuration：设置插件的参数值。
4. Maven 本地仓库
当用户在pom.xml文件中添加依赖jar包时，maven会先从本地仓库查找，如果这个jar包在本地仓库中找不到，就从中央仓库下载到本地仓库，中央仓库是maven默认的远程仓库。本地仓库默认是在 ```%USER_HOME%\.m2\repository```目录下。可以通过settings.xml文件制定。
5. 镜像
maven默认从中央仓库下载依赖，但是速度比较慢。镜像相当于是中央仓库的一个副本，内容和中央仓库完全一样，下载速度较快。
6. 查看仓库列表时候各项是什么意思？
![nexus仓库列表.png](https://upload-images.jianshu.io/upload_images/13572633-7538d890a8b17f3a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
type列：
1、proxy是代理仓库，如果自己私有库没有对应的资源(jar等)，就会到这里去找。
2、hosted是宿主仓库，是自己的私有库地址。这仓库有release和snapshots两种类型，如果自己在创建依赖jar包的时候，就需要指定，是正式发布(release)，还是发布开发版(snapshots)。
3、group管理组，组是Nexus一个强大的特性，它允许你在一个单独的URL中组合多个仓库，比如默认组合：maven-central、maven-release和maven-snapshots。
Policy列：
snapshots：开发过程中的版本仓库
release：正式发布的版本仓库
7. [SNAPSHOT和RELEASE](https://www.cnblogs.com/wuchanming/p/5484091.html)
在maven的约定中，依赖的版本分为两类——SNAPSHOT和RELEASE。SNAPSHOT依赖泛指以-SNAPSHOT为结尾的版本号，例如1.0.1-SNAPSHOT。除此之外，所有非-SNAPSHOT结尾的版本号则都被认定为RELEASE版本，即正式版，虽然会有beta、rc之类说法，但是这些只是软件工程角度的测试版，对于maven而言，这些都是RELEASE版本。
7.1 SNAPSHOT
如上图，很好地表达了SNAPSHOT的细节，也阐述了一个SNAPSHOT很重要观点——SNAPSHOT不是一个特定的版本，而是一系列的版本的集合，其中HEAD总是指向最新的快照，对外界可见的一般也是最新版，这种给人的假象是新的覆盖了老的，从而使得使用SNAPSHOT依赖的客户端总是通过重新构建（有时候需要-U强制更新）就可以拿到最新的代码。例如：A-->B-1.3.8-SNAPSHOT（理解为A依赖了B的1.3.8-SNAPSHOT版本），那么B-1.3.8-SNAPSHOT更新之后重新deploy到仓库之后，A只需要重新构建就可以拿到最新的代码，并不需要改变依赖B的版本。由此可见，这样达到了变更传达的透明性，这对于开发过程中的团队协作的帮助不言而喻。
7.2 RELEASE
RELEASE版本和SNAPSHOT是相对的，非SANPSHOT版本即RELEASE版本，RELEASE版本是一个稳定的版本号，看清楚咯，是一个，不是一系列，可以认为RELEASE版本是不可变化的，一旦发布，即永远不会变化。
虽然RELEASE版本是稳定不变的，但是仓库还是有策略让这个原则变得可配置，有的仓库会配置成redeploy覆盖，这样RELEASE版本就变成SNAPSHOT了，伪装成RELEASE的SNAPSHOT，会让问题更费解和棘手，我一般称这类人为“挖坑专家”。
记住，RELEASE一旦发布，就不可改变。
如何选择
那么什么时候使用SNAPSHOT？什么时候使用RELEASE?这个可以从他们各自的特性上来看，SNAPSHOT版本的库是一直在变化的，或者说随时都会变化的，这样虽然可以获取到最新的特性，但是也存在不稳定因素，依赖一个不稳定的模块或者库会让模块自身也变得不稳定，尤其是自身对被依赖模块的变化超出掌控的情况。即使可以掌控被依赖模块的变化，也会带来不稳定的因素，因为每次变更都有引入bug的可能性。如果这么说，那么我们是不是要摒弃SANPSHOT了呢？答案肯定是否定的。
想象下，什么情况下，模块会一直变化或者变化比较剧烈？开发新特性的时候，所以对于团队之间协同开发的时候，模块之间出现依赖，变化会非常剧烈，如模块A依赖模块B，模块A必然需要最方便地获取模块B的特性，在开发期间，方便性比稳定性更重要。可以反证下，假设模块B使用RELEASE版本1.0.0，模块A依赖1.0.0，现在模块A出现了bug，需要修复下，那么A就要提供一个版本号1.0.1，这样所有依赖A模块都需要更新版本号，因为开发期间这种事情是如此多，所以会带来巨变。反观SNAPSHOT方案，如果模块B的版本是1.0.0-SNAPSHOT，模块A完全不需要修改版本号即可获取模块B的新特性。当开发进入预发布阶段，为了生产环境的稳定性，依赖应该是RELEASE版本，因为此时SNAPSHOT版本的模块自动获取新特性的特点恰恰会造成生产环境的不稳定性，生产环境上，稳定性重于一切。

8. 将jar安装到本地仓库  
cmd运行```mvn install:install-file -DgroupId=com.wangzz -DartifactId=demo -Dversion=1.0.0 -Dpackaging=jar -Dfile=d:/demo.jar```
