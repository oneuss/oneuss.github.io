---
layout: post
title:  "项目中使用了哪些技术？"
subtitle: "常被问到"
date:   2020-04-22
background: '/img/imac_bg.png'
---
***
# Spring

1. 为什么使用Spring？
    这里引用Spring官网的描述：

    > Spring使Java编程变得更快、更简单、更安全。Spring对速度、简单性和效率的关注使其成为世界上最受欢迎的Java框架。
    > Spring makes programming Java quicker, easier, and safer for everybody. Spring’s focus on speed, simplicity, and productivity has made it the [world's most popular](https://snyk.io/blog/jvm-ecosystem-report-2018-platform-application/) Java framework.
    > "我们使用了很多Spring框架自带的工具，并收获了很多**开箱即用的解决方案，而且不用担心编写大量的额外代码**--所以这确实为我们节省了一些时间和精力。"
    > 西恩-格雷厄姆，应用转型领导，迪克体育用品公司的应用转型负责人
    > “We use a lot of the tools that come with the Spring framework and reap the benefits of having a lot of the out of the box solutions, and not having to worry about writing a ton of additional code—so that really saves us some time and energy.”
    >SEAN GRAHAM, APPLICATION TRANSFORMATION LEAD, DICK’S SPORTING GOODS

    > 1. Spring is everywhere
    >     Spring 框架无处不在，全世界的开发人员，各种公司都在使用Spring，这从侧面反映了Spring框架的稳定性。另外，广泛的使用也使得这个框架可以更早地发现其中隐藏的bug，促进了框架的安全性。同时也使得Spring开发社区十分活跃，使用过程中发现问题可以更及时、高效地解决，入门也更迅速。
    > 2. Spring is flexible
    >     Spring 是灵活的。Spring拥有众多的扩展，以及可以灵活地使用第三方库。
    > 3. Spring is productive
    >    微服务相关的描述，我并没有使用到。
    > 4. Spring is fast
    >    速度很快。包括几个方面，首先是性能方面，速度快，其次是开发效率快，上手快。最后是可以快速构建项目（使用Spring Boot）。
    > 5. Spring is secure
    >    Spring是安全的。这也包括两方面。一是安全漏洞的修补和版本更新迭代。二是Spring的安全框架。
    > 6. Spring is supportive
    >    社区完善，这点第一点里面提到过。社区也很重要，以前我使用过IBM的产品，出问题在互联网上很难找到合适的解决方案。

2. Spring MVC
    1. 说下Spring MVC从接受请求到返回数据的整个流程
        客户端发送请求，根据web.xml的配置，请求到了DispatcherServlet；
        DispatcherServlet根据请求路径，调用的HandlerMapping解析出handler；

3. IOC、context
    1. 讲下什么是IOC？IOC相比new有什么优缺点
    2. 对IOC容器的理解，IOC容器是怎么工作的
    3. Spring容器初始化过程
    4. Bean创建过程
    5. Spring如何解决循环引用
    6. BeanFactory和ApplicationContext的区别
    7. Bean的生命周期
        找到配置中bean的定义，利用反射创建bean实例，如果有一些属性值需要设置，则调用set方法设置。如果bean实现了BeanNameAware接口，则调用setBeanName方法，实现了BeanFactoryAware，调用setBeanFactory方法，其他aware接口类似。如果有和加载这个Bean的Spring容器相关的BeaPostProcessor对象，执行postProcessBeforeInitialization（）方法。如果bean实现了InitializingBean，则执行afterPropertiesSet（）方法。接下来执行配置文件定义的init-method指定的方法。如果有和加载这个 Bean的 Spring 容器相关的 BeanPostProcessor 对象，执行postProcessAfterInitialization() 方法。当要销毁Bean的时候，如果 Bean 实现了 DisposableBean 接口，执行 destroy() 方法。最后会执行配置文件定义的destory-method方法。
4. [AOP、spring-aspects](https://docs.spring.io/spring/docs/4.1.2.RELEASE/spring-framework-reference/html/aop.html#aop-introduction)
    1. 针对API：进行入参、返回结果的打印，对异常的返回结果进行进行封装。
    2. 集群环境下，为避免定时任务重复执行，针对定时任务执行前后进行加锁，解锁操作。（使用针对函数的自定义注解，参数1：缓存的key，参数２：过期时间）
    3. JDK动态代理的原理：获取代理对象时候传入类加载器，所有实现的接口数组，代理对象InvocationHandler（里面有个invoke方法，里面写入具体想执行的代码），Proxy类会生成一个实现了代理类实现的接口的，继承了Proxy类的子类，具体执行方法时候，这个子类的方法里会调用proxy的invocationHandler对象的invoke方法执行，这样就达到了增强的效果。
    4. Spring Aop和AspecJ Aop有什么区别
5. Spring-Data-Redis
    这个不用多说了，RedisTemplate。
6. Spring-TX
    1. 事务隔离级别
    2. Spring事务的实现
        Spring使用AOP（面向切面编程）来实现声明式事务，AOP有5种通知方式，声明式事务的实现就是通过环绕通知的方式，在目标方法执行之前开启事务，在目标方法执行之后提交或者回滚事务。
    3. 事务的传播特性
    4. 事务不生效的原因
        是否数据库引擎不支持事务，入口方法是否是public，Spring事务只对运行时异常好滚，是不是没有开启注解事务，是不是同类中方法调用，没有调用到代理方法等。

***

# MyBatis

1. 为什么只写了接口，就可以执行sql？

***

# MySQL
1. 如何优化SQL（sql优化的着手点）
    1. 注意适当地建立索引，这样插入时候将相邻数据存储在一起，将随机IO变成顺序IO，提高效率。
    2. 查询时候如果可以的花，查询字段在索引字段里面，这样覆盖索引，走二级索引查询即可，不会回表，提高了效率。
    3. 查询条件不要使用函数或者表达式，否则不能命中索引
    4. 对多个列对查询，建立多列联合索引，查询时遵循最左前缀原则。
    5. 复杂的查询，用explain分析索引执行情况，根据结果来优化查询语句。
    6. 切分大查询，避免将buffer pool占满，影响别的查询。
2. MySQL调优做过哪些
3. Explain的结果一定是最优的吗
4. 有没有做主从之类的
    1. 主从复制，涉及三个线程
        - binlog线程：负责将主服务器上的数据更改写入二进制日志中。
        - I/O线程：负责从主服务器上读取二进制日志，并写入从服务器的中继日志（Replay log）。
        - SQL线程：负责读取中继日志，解析出主服务器傻姑已经执行的数据更改并在从服务器上重放（Replay）。
      2. 读写分离，主服务器处理写操作，以及实时性强的读操作，从服务器处理读操作，优点如下：
          - 主从服务器负责各自的读写，极大长度缓解了锁的争用；
          - 从服务器可以使用MyISAM。提高查询性能以及节省系统开销；
          - 增加冗余，提高可用性。
          - 实现：用代理方式实现，代理服务器接受应用层的读写请求，决定转发到哪个服务器。
5. 事务隔离级别
    - READ-UNCOMMITTED
       - 读未提交
      - 造成脏读问题
      - 性能并不比其他的好太多，因此很少使用
    - READ-COMMITTED
      - 读已提交
      - 不可重复读问题：事务1查询第一次，结果1，事务2修改了内容，事务1再查询结果不同，同一次事务中查询出不同的结果。不可重复读是读取到的记录，再读取时候，记录发生了改变。
      - 大多数数据库默认的隔离级别，但不是MySQL的。
    - REPEATABLE-READ
      - 可重复读
      - 有可能造成幻读问题：事务1查询结果，发现不存在，这时事务2插入了同样的数据，事务1再读发现有了，幻读重点强调了读取到了之前读取没有获取到的记录。InnoDB通过MVCC解决了该问题。
      - 这是MySQL的默认隔离级别。
    - Serializable
      - 可串行化
      - 啥问题没有，就是慢。

***

# Redis

1. 使用它做什么？
2. 分布式锁如何实现的？
3. 遇到过什么问题？锁过期时间设置了吗？如果设置了过期还未执行完怎么办？（看门狗）可重入是怎么做的？（使用redission可以实现，具体是）
4. 有没有用过本地缓存
5. 二八定律、热数据与冷数据、缓存学霸、缓存击穿、缓存预热、缓存更新、缓存降级。
6. Redis数据结构、如何做持久化？
    1. 数据结构 - 基础数据结构：
        1. string（字符串）
            - redis字符串是动态字符串，可以修改
            - 内部结构类似于ArrayList，采用预分配冗余空间的方式减少内存频繁分配。
            - 字符串<1M时，扩容都是加倍现在的空间。>1M时，一次只会多扩1M空间。
            - 字符串最大：512M。
        2. list（列表）
            - 相当于LinkedList。所以插入删除都是O(1)的时间复杂度。索引定位O(n)
            - 底层具体存储的是quicklist结构。元素较少时使用连续的内存，这个结构是ziplist（压缩列表）
            - 数据量多时候，改成quicklist：将链表和ziolist组合起来->使用双向指针将ziplist串起来。
        3. set（集合）
            - 相当于HashSet
        4. hash（字典）
            - 相当于HashMap，数组+链表，但是字段的值只能是字符串
            - 为例保证性能，rehash采用渐进式rehash的策略：渐进式rehash会在rehash同时，保留新旧两个hash结构，查询时同时查询两个hash结构，然后后续逐渐将旧的迁移到新的hash结构，完成后就只用新的。
        5. zset（有序集合）
            - redis最有意思的数据结构，类似与SortedSet和HashMap结合体
            - 有唯一的value，另外每个value都有score，代表value的排序权重
            - score使用double存储，存在精度问题
            - 内部排序是通过跳跃列表来实现的：最下面一层所有的元素都会串起来，然后每隔几个元素挑出来一个代表，再将这几个代表用指针串起来，然后在代表中再挑选出代表串起来，一层层类似金字塔。“代表”是随机策略选取，概率逐渐减半，100%->50%->25%->12.5%->..
    2. 位图（本质也是string（或者说string的本质也是byte数组，谁又不是呢），通过setbit/getbit 位图操作）
    3. HyperLogLog（pfadd/pfcount/pfmerge 操作）
        - 不精确的去重解决方案
        - 统计UV
    4. 布隆过滤器
        - 简单理解为不怎么精确的set结构：某个值存在时，可能并不存在；某个之不存在时，一定不存在。  
        - redis 4.0之后，使用插件实现
        - 具体原理是：由一个大型的位数组和几个不一样的无偏hash函数组成。添加key时，多个hash函数对key计算hash，再对位数组取模得到多个位置，将这个位置设为1.当向布隆过滤器询问key是否存在时，也要进行系统的过程，看这几个位置是否都为1，只要有一个不为1，就说明不存在，如果都是1，说明极有可能存在。具体涉及到概率论，不懂。。
7. Redis单线程为什么这么快？
8. 有没有做高可用
    1. 主从
9. 缓存读写，数据库和缓存如何协同工作
10. Redis集群巴拉巴拉
11. 持久化
12. 连接redis用的什么？jedis是线程安全的吗？  
      - 项目里是用工具类对Spring的StringRedisTemplate进行了一些方法的封装。stringRedisTemplate是用JedisConnectionFactory连接工厂。

***

# 使用了什么高并发技术？

1. CountDownLatch
    - 定时任务，执行查询时候使用线程池执行，定时任务的主方法里面使用了countdownlatch。
    - 实现原理：aqs
2. 使用了什么高并发集合？
3. 使用的什么类型的线程池？用什么拒绝策略？
    Spring的ThreadPoolTaskExecutor，本质是ThreadPoolExecutor
4. 用过什么锁，有什么区别
5. 锁升级过程
    首先需要了解对象头的组成。对象头有锁标志位，bit位上不同值代表不同的锁状态。（注意无锁和偏向锁标志位都是01，但是后面有个是否偏向锁的一个bit位的标识）。
    - 无锁->偏向锁：
    - 偏向锁->轻量级锁
        进入同步代码块时，如果此同步对象没有被锁定，虚拟机首先在当前栈帧中建一个Lock Record的玩意儿，先把对象头的markword 拷贝到这个区域，然后尝试将对象头的Mark Word指向线程的Lock Record指针，如果更新成功，就拥有了这个锁，并将标志位变为00（轻量级锁），如果更新失败先检查Mark Word是否指向当前线程的Lock Record,是就说明拥有了。
        解锁：CAS操作尝试将Mark Word替换回来，替换成功完成。替换失败说明有竞争，
    - 轻量级锁->重量级锁

6. synchronized原理

***

# 用到了哪些设计模式？

1. 工厂模式，单例模式，建造者，代理模式，策略模式之类
2. 简单实用了适配器模式：使用JAXB转换进行Bean和xml格式转换时候，一些日期格式或者别的与默认的转换格式不一致时候，使用实现了XmlAdapter 接口的自定义的适配器进行类型转换。
3. 责任链
4. 装饰器模式
5. 设计模式举例
    - 单例模式：某个类只能有一个实例，提供一个全局的访问点。
   - 简单工厂：一个工厂类根据传入的参量决定创建出那一种产品类的实例。
   - 工厂方法：定义一个创建对象的接口，让子类决定实例化那个类。
   - 抽象工厂：创建相关或依赖对象的家族，而无需明确指定具体类。
   - 建造者模式：封装一个复杂对象的构建过程，并可以按步骤构造。
   - 原型模式：通过复制现有的实例来创建新的实例。
   - 适配器模式：将一个类的方法接口转换成客户希望的另外一个接口。
   - 组合模式：将对象组合成树形结构以表示“”部分-整体“”的层次结构。
   - 装饰模式：动态的给对象添加新的功能。
   - 代理模式：为其他对象提供一个代理以便控制这个对象的访问。
   - 亨元（蝇量）模式：通过共享技术来有效的支持大量细粒度的对象。
   - 外观模式：对外提供一个统一的方法，来访问子系统中的一群接口。
   - 桥接模式：将抽象部分和它的实现部分分离，使它们都可以独立的变化。
   - 模板模式：定义一个算法结构，而将一些步骤延迟到子类实现。
   - 解释器模式：给定一个语言，定义它的文法的一种表示，并定义一个解释器。
   - 策略模式：定义一系列算法，把他们封装起来，并且使它们可以相互替换。
   - 状态模式：允许一个对象在其对象内部状态改变时改变它的行为。
   - 观察者模式：对象间的一对多的依赖关系。
   - 备忘录模式：在不破坏封装的前提下，保持对象的内部状态。
   - 中介者模式：用一个中介对象来封装一系列的对象交互。
   - 命令模式：将命令请求封装为一个对象，使得可以用不同的请求来进行参数化。
   - 访问者模式：在不改变数据结构的前提下，增加作用于一组对象元素的新功能。
   - 责任链模式：将请求的发送者和接收者解耦，使的多个对象都有处理这个请求的机会。
   - 迭代器模式：一种遍历访问聚合对象中各个元素的方法，不暴露该对象的内部结构。
6. 策略模式

***

# Dubbo

1. 调用超时怎么办
    - 我们使用的集群容错是：failover，但是retries重试次数设置的是0，也就是失败后就抛异常了。
2. 怎么做负载均衡的
    - 使用缺省配置：random，也就是随机的。权重也没有配置，每台机器都一样。
    - dubbo的负载均衡总共有这几种：
      - Random LoadBalance：随机，按权重设置随机概率。默认负载均衡策略。权重算法：取权重和的随机数，看落在哪个范围，比如：第一台机器权重10，第二台20，第三台30，那第一台机器的区域是1-10，第二胎11-30，第三台31-60，再取随机数，看落在哪台机器的范围内。这个算法以前我做游戏抽奖时候用过，哈哈。中奖率低的一批。
      - RoundRobin LoadBalance：轮询，按公约后的权重设置轮询比率。这个算法会严格按照权重比例来分配，比如三台机器权重如下{a:1,b:2,c:3}，如果有6次请求，负载结果如下：{a,b,c,b,c,c}。
      - LeastActive LoadBalance：最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。
      - ConsistentHash LoadBalance：一致性 Hash，相同参数的请求总是发到同一提供者。当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。虚拟节点数量，默认160。
3. 集群容错怎么做的
    - 使用默认的：failover，但是retries是0，所以和failfast没区别，感觉。
    - dubbo集群容错：
        - Failover Cluster：失败自动切换，当出现失败，重试其它服务器 。通常用于读操作，但重试会带来更长延迟。可通过 `retries="2"` 来设置重试次数(不含第一次)。缺省配置。
        - Failfast Cluster：快速失败，只发起一次调用，失败立即报错。通常用于非幂等性的写操作，比如新增记录。
        - Failsafe Cluster：失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。
        - Failback Cluster：失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。
        - Forking Cluster：并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。
        - Broadcast Cluster：广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。

4. SPI
5. 是用Zookeeper做注册中心的注册，发现过程,以`barService`为例
    1. 服务提供者启动时: 向 /dubbo/com.foo.BarService/providers 目录下写入自己的 URL 地址
    2.  服务消费者启动时: 订阅 /dubbo/com.foo.BarService/providers 目录下的提供者 URL 地址。并向 /dubbo/com.foo.BarService/consumers 目录下写入自己的 URL 地址
    3. 监控中心启动时: 订阅 /dubbo/com.foo.BarService 目录下的所有提供者和消费者 URL 地址。
    4. 我们来验证以下：
        1. 是用zkclient连接zookeeper:`./zkCli.sh -server 127.0.0.1:2181`
        2. 查看`dubbo`节点的所有子节点：`ls /dubbo`，结果如下：  
      ![image.png](https://upload-images.jianshu.io/upload_images/13572633-d1b4e93f7225b6ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
        3. 随便查看一个service路径下的内容，可以看到提供者消费者都在下面。
![image.png](https://upload-images.jianshu.io/upload_images/13572633-56fbeb787177f0c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. 都用过哪些配置？？？
    - 这个问题。。
    - service：服务提供者暴露服务配置。
    - reference：服务消费者引用服务配置。
    - protocol ：服务提供者协议配置。对应的配置类： org.apache.dubbo.config.ProtocolConfig。如果需要支持多协议，可以声明多个 <dubbo:protocol> 标签，并在 <dubbo:service> 中通过 protocol 属性指定使用的协议。
    - registry：注册中心配置。对应的配置类： org.apache.dubbo.config.RegistryConfig。同时如果有多个不同的注册中心，可以声明多个 <dubbo:registry> 标签，并在 <dubbo:service> 或 <dubbo:reference> 的 registry 属性指定使用的注册中心。
    - provider ：服务提供者缺省值配置。对应的配置类： org.apache.dubbo.config.ProviderConfig。同时该标签为 <dubbo:service> 和 <dubbo:protocol> 标签的缺省值设置。
    - cosumer：服务消费者缺省值配置。配置类： org.apache.dubbo.config.ConsumerConfig 。同时该标签为 <dubbo:reference> 标签的缺省值设置。
    - monitor：监控中心配置。
    - application：应用信息配置。对应的配置类：org.apache.dubbo.config.ApplicationConfig
    - module：模块信息配置。对应的配置类 org.apache.dubbo.config.ModuleConfig
    - method：方法级配置。对应的配置类： org.apache.dubbo.config.MethodConfig。同时该标签为 <dubbo:service> 或 <dubbo:reference> 的子标签，用于控制到方法级。
    - argument：方法参数配置。对应的配置类： org.apache.dubbo.config.ArgumentConfig。该标签为 <dubbo:method> 的子标签，用于方法参数的特征描述。
    - parameter：选项参数配置。对应的配置类：java.util.Map。同时该标签为<dubbo:protocol>或<dubbo:service>或<dubbo:provider>或<dubbo:reference>或<dubbo:consumer>的子标签，用于配置自定义参数，该配置项将作为扩展点设置自定义参数使用。
    - config：配置中心。对应的配置类：org.apache.dubbo.config.ConfigCenterConfig
7. 一个接口有多个实现时候怎么办啊？
    配置group，分组
8. 希望访问指定的服务时候怎么办？
    reference中配置指定url，点对点直连服务提供者地址，将绕过注册中心

***

# Zookeeper

1. 用它做什么？
    分布式协调服务。做dubbo的注册中心，服务注册发现。也可以用它做分布式锁。
2. 为什么选择它？dubbo还有其他选择吗？
    高可用
3. 有没有做高可用？做了，三台机器
4. zookeeper的数据结构？
    像文件系统的目录树。数据存储基于节点，称为Znode的节点。Znode包含了数据，子节点应用，访问权限等。具体如下：
   - data：  
  Znode存储的数据信息。
   - ACL：
记录Znode的访问权限，即哪些人或哪些IP可以访问本节点。
   - stat：
包含Znode的各种元数据，比如事务ID、版本号、时间戳、大小等等。
   - child：
当前节点的子节点引用，类似于二叉树的左孩子右孩子。
5. 如何实现分布式一致性

6. zk选举过程，协议
    ZooKeeper 集群中始终确保其中的一台为 leader 的角色，并通过 *ZAB (Zookeeper Atomic Broadcast Protocol) <sup>[[1]](http://dubbo.apache.org/zh-cn/blog/dubbo-zk.html#fn1)</sup>* 协议确保所有节点上的信息的一致。客户端可以访问集群中的任何一台进行读写操作，而不用担心数据出现不一致的现象。

7. Zookeeper如何实现分布式锁
    临时顺序节点。临时节点的特性可以让锁有过期时间，顺序节点可以在锁上有个排队，保证公平性。客户端进来创建一个临时顺序节点，如果顺序排在第一位，就拿到了锁，如果不排在第一位，就在前一个节点上加一个监听器，当上一个节点被删除时候，再判断是不是第一个节点，是就加锁成功。

***

# 领域驱动设计

[https://kb.cnblogs.com/page/522125/](https://kb.cnblogs.com/page/522125/)

***

# 数据结构

1. 项目中使用过哪些数据结构
2. 对每个数据结构的了解，每个数据结构的特点，Java对他们是如何实现的
3. HashMap 数据结构，JDK8之后的改变，为什么线程不安全，并发会有什么问题
    1. JDK7中HashMap并发put会造成循环链表，导致get时出现死循环
    2. put和get并发时，可能导致get为null（resize时候，如果第一个线程刚刚创建了一个空的hash表，第二个此时来get，get不到）
    3. 多线程的put可能导致元素的丢失（形成链表情况下，如果两个线程同时执行到新建新节点的地方，后执行的把先执行的替换了）

***

# 关于项目

1. 与第三方交互如何保证通信安全？
    1.

***

# 哪里使用到了反射

1. 项目里不同的场景需要使用不同的第三方支付渠道，我将支付渠道实现类bean name写到枚举类里面，里面有支付渠道ID和bean name，再在创建场景时候，将资金与支付渠道在数据库里关联起来，这样不同的场景支付时候，先查询到支付渠道的ID，再通过id找到bean name，反射执行支付方法。

***

# Nginx

1. 负载均衡算法
2. 有没有做限流？怎么做的
3. Nginx有没有做高可用？用什么模型
4. 一致性hash算法？

***

# FastDFS

***

# 网络
1. 三次握手与四次挥手的原因
    1. 三次握手：最少需要3次，客户端和服务端双方都能确认对方和自己的发送接受能力都没有问题。
     2. 四次挥手：双方各自发送需要关闭连接的请求FIN，并且确认对方关闭连接的请求ACK。
![图片来源：http://blog.dothinkings.com/wp/?p=884](https://upload-images.jianshu.io/upload_images/13572633-73a1ca9027a2fc03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

***

# 对项目对详细介绍

1. 项目如何避免超放？
2. 简历上写的要清楚，不要说 “我只是用过而已，不太清楚”
3. 并发量多少，如何进行并发量的计算，估计？有没有做熔断限流，重试

***

# 算法

1. 3sum
2. 最长连续数
3. 有哪些常见的hash算法

***

# JVM

1. 项目中使用了说明JVM调优手段？
2. 谈下类加载过程
    - 加载：根据查找路径找class文件导入到虚拟机。
    - 验证：验证class文件，是不是符合规范，比如检查魔数，版本号等。
    - 准备：在方法区给类变量分配内存，设置零值（如果是常量，就会把初始值直接设置给它）。
    - 解析：虚拟机将常量池中符号引用替换为直接引用
    - 初始化：静态变量赋值，和执行静态代码块。
    - ps：
      - 生命周期：加载、连接（验证（检查）、准备、解析）、初始化、使用、卸载。
      - 加载的时机：1：new实例化对象，读取或者设置类的静态字段，调用类静态方法时候。2:反射调用时候。3： 初始化类时，发现父类未初始化，先触发其父类的初始化。4:虚拟机启动是，初始化主类（main。。。）。5:
3. 对象头有什么，对象大小，对象常见过程，对象分配在哪儿？栈上分配了解吗？TLAB是什么？
    1. 首先，对象由对象头（Header），实例数据（Instance Data）和对齐填充（Padding）组成。Hotspot的对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）
        - Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。这些信息都是与对象自身定义无关的数据，所以Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据。它会根据对象的状态复用自己的存储空间，也就是说在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。这部分在32位和64位虚拟机中分别是32bit和64bit。
        - Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
        - 对了，如果是数组，还有有额外的部分存储数组长度。
4. 工具[arthas](https://alibaba.github.io/arthas/)  [在线分析](http://fastthread.io/)
5. 双亲委派模型
6. 对象创建过程：
    - 虚拟机执行到new关键字之后的，先看下后面那个类有没有加载，没加载就加载
    - 之后分配内存空间，分配完之后，
7. JVM由哪几部分构成
    - 类加载器
    - 运行时数据区
      - 堆
      - 栈：描述Java方法执行的数据结构。每个方法执行时，都会创建一个栈帧，用于保存局部变量表，方法出口，操作数栈，动态链接等。
      - 方法区：存储已被虚拟机加载的类信息、常量、静态变量、即时编译后的代码等数据。
      - 本地方法栈
      - 程序计数器：线程私有，唯一没有规定oom的区域。
    - 执行引擎
    - 本地接口库
    - >工作过程：jvm首先需要把字节码通过一定的方式 类加载器（ClassLoader） 把文件加载到内存中运行时数据区（Runtime Data Area），而字节码文件是jvm的一套指令集规范，并不能直接交个底层操作系统去执行，因此需要特定的命令解析器 执行引擎（Execution Engine） 将字节码翻译成底层系统指令再交由CPU去执行，而这个过程中需要调用其他语言的接口 本地库接口（Native Interface）来实现整个程序的功能
>
>作者：Java中文社群
>链接：https://juejin.im/post/5cad272a5188254eb942fabe
8. Java线程是如何实现的？

***

# Java基础

1. HashMap是线程安全的吗？并发情况下会产生什么问题？JDK1.8之后有什么改进？
