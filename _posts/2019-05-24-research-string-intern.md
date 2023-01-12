---
layout: post
title:  "从intern函数产生的疑问"
subtitle: ""
date:   2019-05-24
background: '/img/imac_bg.png'
---
我之前说过：
> 在阅读周明耀老师的《深入理解JVM ＆ G1 GC》该书时，对本书p31~33关于String的部分内容产生疑问，遂通过google等搜索引擎以及Stack Overflow和知乎(R大)[[https://www.zhihu.com/people/rednaxelafx/answers?order_by=vote_num](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.zhihu.com%2Fpeople%2Frednaxelafx%2Fanswers%3Forder_by%3Dvote_num)]的解答研究了一番，最终理解还是有欠缺。

这次系统地把这个“债”还上。
### 先看下涉及到的两本书的内容：

1. 节选自《深入理解JVM ＆ G1 GC》P31+ （内容来自京东本书试读的截图，侵删）
>![31](https://upload-images.jianshu.io/upload_images/13572633-dd2a5004391a9136.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![32](https://upload-images.jianshu.io/upload_images/13572633-efa374a6bcc666fe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![33](https://upload-images.jianshu.io/upload_images/13572633-786a712213831fc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![34](https://upload-images.jianshu.io/upload_images/13572633-ccc019b08b56de61.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2. 节选自《深入理解Java虚拟机 JVM高级特性与最佳实践 第2版》P57
>![P57](https://upload-images.jianshu.io/upload_images/13572633-503fa56f177d8f40.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3. `intern`函数如下：

```java
    /**
     * Returns a canonical representation for the string object.
     * <p>
     * A pool of strings, initially empty, is maintained privately by the
     * class {@code String}.
     * <p>
     * When the intern method is invoked, if the pool already contains a
     * string equal to this {@code String} object as determined by
     * the {@link #equals(Object)} method, then the string from the pool is
     * returned. Otherwise, this {@code String} object is added to the
     * pool and a reference to this {@code String} object is returned.
     * <p>
     * It follows that for any two strings {@code s} and {@code t},
     * {@code s.intern() == t.intern()} is {@code true}
     * if and only if {@code s.equals(t)} is {@code true}.
     * <p>
     * All literal strings and string-valued constant expressions are
     * interned. String literals are defined in section 3.10.5 of the
     * <cite>The Java&trade; Language Specification</cite>.
     *
     * @return  a string that has the same contents as this string, but is
     *          guaranteed to be from a pool of unique strings.
     */
    public native String intern();
```

关于`intern`函数的`Javadoc`描述是说：
>字符串常量池有String类单独维护，它最初是空的。当intern方法被调用时，如果池中已经包含了一个由equals方法判断为true的字符串对象，则会将池中的string返回。否则这个string对象会被添加到池中并且返回这个对象的引用。所有的文字字符串和字符串值常量表达式都是内部的。

### 验证
本着怀疑精神，我下载了`Oracle JDK6`，`JDK7`和`JDK8`，分别验证书中结果。
1. 《深入理解JVM ＆ G1 GC》内容验证

```java
    public static void main(String[] args) {
        String s = new String("1");
        s.intern();
        String s2 = "1";
        System.out.println(s == s2);

        String s3 = new String("1") + new String("1");
        s3.intern();

        String s4 = "11";
        System.out.println(s3 == s4);
    }
```
  - JDK6

```bash
D:\Java>D:\Java\jdk1.6\bin\java -version
java version "1.6.0_45"
Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
Java HotSpot(TM) 64-Bit Server VM (build 20.45-b01, mixed mode)

D:\Java>D:\Java\jdk1.6\bin\javac StringInternDemo.java

D:\Java>D:\Java\jdk1.6\bin\java StringInternDemo
false
false
```
  - JDK7

```shell
D:\Java>D:\Java\jdk1.7\bin\java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)

D:\Java>D:\Java\jdk1.7\bin\javac StringInternDemo.java

D:\Java>D:\Java\jdk1.7\bin\java StringInternDemo
false
true
```
  - JDK8

```
D:\Java>java -version
java version "1.8.0_202"
Java(TM) SE Runtime Environment (build 1.8.0_202-b08)
Java HotSpot(TM) 64-Bit Server VM (build 25.202-b08, mixed mode)

D:\Java>javac StringInternDemo.java

D:\Java>java StringInternDemo
false
true
```
2. 《深入理解Java虚拟机 JVM高级特性与最佳实践 第2版》内容验证

```
public class RuntimeConstantPoolOOM {
    public static void main(String[] args) {
        String str1 = new StringBuilder("计算机").append("软件").toString();
        System.out.println(str1.intern() == str1);

        String str2 = new StringBuilder("ja").append("va").toString();
        System.out.println(str2.intern() == str2);
    }
}
```
- JDK6

```
D:\Java>D:\Java\jdk1.6\bin\javac  -encoding UTF-8 RuntimeConstantPoolOOM.java

D:\Java>D:\Java\jdk1.6\bin\java RuntimeConstantPoolOOM
false
false
```
- JDK7

```
D:\Java>D:\Java\jdk1.7\bin\javac  -encoding UTF-8 RuntimeConstantPoolOOM.java

D:\Java>D:\Java\jdk1.7\bin\java RuntimeConstantPoolOOM
true
false
```
- JDK8

```
D:\Java>javac -encoding UTF-8 RuntimeConstantPoolOOM.java

D:\Java>java RuntimeConstantPoolOOM
true
false
```
书中的结果没问题，这个也不至于，代码明明白白。
### 我的疑问

1. 《深入理解JVM ＆ G1 GC》有这么一段表述：
![](https://upload-images.jianshu.io/upload_images/13572633-abe583b9b64031f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我的疑惑主要有两点：
    1.1 `new String("1");``是创建了两个对象吗？
这个疑问可以参考R大的这篇帖子 [请别再拿“``String s = new String("xyz");``创建了多少个String实例”来面试了吧](https://rednaxelafx.iteye.com/blog/774673)
可以说创建两个对象这个说法是笼统的，具体的情况比这个要复杂。我说下看完之后我的理解，可能不是特别准确，我最怕说完之后被R大看到说 你要这么理解的话，这个帖子就白看了，哈哈。。

具体来说，SunJDK1.6版本中，“1”这个字符串字面量，在编译后、运行代码前类加载的resolve 阶段，字面量“1”的`String`实例会存放于`PermGen`，其引用会被保存到位于`NativeMemory`的`StringTable`中。而``new String("1")``执行后，会在 `JavaHeap`创建另一个`String`实例。
> [请别再拿“``String s = new String("xyz");``创建了多少个String实例”来面试了吧](https://rednaxelafx.iteye.com/blog/774673)
>>根据上文引用的规范的内容，符合规范的JVM实现应该在类加载的过程中创建并驻留一个String实例作为常量来对应"xyz"字面量；具体是在类加载的resolve阶段进行的。这个常量是全局共享的，只在先前尚未有内容相同的字符串驻留过的前提下才需要创建新的String实例。

而在`SunJDK`更高版本中，类加载时候创建的`String`实例位置在`JavaHeap`中，其他没有变化。

  1.2 JDK1.6 字符串常量池中创建的字符串实例是在Perm 区吗？
是的，参照[关于`class loader`的一点疑惑？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/29996850/answer/47429324) 倒数第二段话。按理说标准答案需要参照JDK1.6的Java语言标准和JVM标准，不过很惭愧，我不甚能看懂。因此参照R大的回答，R大老师对这两个标准的研究比较多。

另外关于字符串相加再额外说一点，`new String("1") + new String("1");`编译时会被优化为`new StringBuilder("1").appeng("1");`。另外，字面量直接相加时，发生编译时常量折叠，也就是`"1"+"1"`相当于`"11"`。且会当做常量对待 ☞ *“当+运算符的左右两个操作数都是编译时常量时，这个+表达式也会被任务是编译时常量表达式”。*
附JDK1.6 main方法javap结果：

```java
public static void main(java.lang.String[]);
  Code:
   Stack=4, Locals=3, Args_size=1
   0:   new     #2; //class java/lang/StringBuilder
   3:   dup
   4:   invokespecial   #3; //Method java/lang/StringBuilder."<init>":()V
   7:   new     #4; //class java/lang/String
   10:  dup
   11:  ldc     #5; //String 1
   13:  invokespecial   #6; //Method java/lang/String."<init>":(Ljava/lang/String;)V
   16:  invokevirtual   #7; //Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
   19:  new     #4; //class java/lang/String
   22:  dup
   23:  ldc     #5; //String 1
   25:  invokespecial   #6; //Method java/lang/String."<init>":(Ljava/lang/String;)V
   28:  invokevirtual   #7; //Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
   31:  invokevirtual   #8; //Method java/lang/StringBuilder.toString:()Ljava/lang/String;
   34:  astore_1
   35:  aload_1
   36:  invokevirtual   #9; //Method java/lang/String.intern:()Ljava/lang/String;
   39:  pop
   40:  ldc     #10; //String 11
   42:  astore_2
   43:  getstatic       #11; //Field java/lang/System.out:Ljava/io/PrintStream;
   46:  aload_1
   47:  aload_2
   48:  if_acmpne       55
   51:  iconst_1
   52:  goto    56
   55:  iconst_0
   56:  invokevirtual   #12; //Method java/io/PrintStream.println:(Z)V
   59:  return
```
JDK1.7 main方法：

```java
public static void main(java.lang.String[]);
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=4, locals=3, args_size=1
       0: new           #2                  // class java/lang/StringBuilder
       3: dup
       4: invokespecial #3                  // Method java/lang/StringBuilder."<init>":()V
       7: new           #4                  // class java/lang/String
      10: dup
      11: ldc           #5                  // String 1
      13: invokespecial #6                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
      16: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: new           #4                  // class java/lang/String
      22: dup
      23: ldc           #5                  // String 1
      25: invokespecial #6                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
      28: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      31: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      34: astore_1
      35: aload_1
      36: invokevirtual #9                  // Method java/lang/String.intern:()Ljava/lang/String;
      39: pop
      40: ldc           #10                 // String 11
      42: astore_2
      43: getstatic     #11                 // Field java/lang/System.out:Ljava/io/PrintStream;
      46: aload_1
      47: aload_2
      48: if_acmpne     55
      51: iconst_1
      52: goto          56
      55: iconst_0
      56: invokevirtual #12                 // Method java/io/PrintStream.println:(Z)V
      59: return
```

2. 再看《深入理解Java虚拟机 JVM高级特性与最佳实践 第2版》

>![04d1debbf71cddd1a0757f39e396055.png](https://upload-images.jianshu.io/upload_images/13572633-82b85a37209f89b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这段代码为什么·`str1.intern()==str1` 在JDK1.6是false，JDK1.7是true，上文已经做了回答。而关于第二点`java`是什么时候加载的，依旧参照R大回答。

>如何理解《深入理解java虚拟机》第二版中对`String.intern()``方法的讲解中所举的例子？ - RednaxelaFX的回答 - 知乎
https://www.zhihu.com/question/51102308/answer/124441115

### 结语
俗话说，尽信书则不如无书。不能全部相信书上的结果，但是也别产生怀疑就马上轻视别人。要自己验证结果，另外要尽量明白其中的道理。有疑问可以查阅官方文档，也可以虚心向人求教。

另外参考链接:
[知乎:Java 中``new String("字面量")`` 中 "字面量" 是何时进入字符串常量池的?](https://www.zhihu.com/question/55994121/answer/147296098)
