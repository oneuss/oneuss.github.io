---
layout: post
title:  "疑似MySQL Java驱动的bug"
subtitle: ""
date:   2020-09-07
background: '/img/imac_bg.png'
---


# 背景
项目组最近遇到了一个问题，各应用执行SQL偶尔会变慢，而且无规律可循。因为一些原因，最近数据库环境进行了迁移，访问数据库中间多了一层硬件防火墙，应用服务器也安装了一些杀毒软件。查看慢SQL和GC日志，发现并无异常。

# 描述
## 项目环境
JDK：1.8
数据库：MySQL 5.5
数据库连接池：alibaba Druid 1.1.5/1.1.23
MySQL驱动版本：mysql-connector-java 5.1.38/8.0.21
ps：上面列出两个版本是想说明最新版本有同样的问题。

## 数据库连接池配置

```xml
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
      init-method="init" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!--或者使用～>8.0.0版本的下面这个驱动类-->
    <!--<property name="driverClassName" value="com.mysql.jdbc.Driver"/>-->
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>

    <!-- 配置初始化大小、最小、最大 -->
    <property name="initialSize" value="10"/>
    <property name="minIdle" value="10"/>
    <property name="maxActive" value="50"/>

    <property name="timeBetweenEvictionRunsMillis" value="60000"/>
    <property name="minEvictableIdleTimeMillis" value="300000"/>

    <property name="testWhileIdle" value="true"/>
    <property name="testOnBorrow" value="false"/>
    <property name="testOnReturn" value="false"/>

    <property name="poolPreparedStatements" value="true"/>
    <property name="maxOpenPreparedStatements" value="20"/>

    <!-- 配置监控统计拦截的filters，wall用于防止sql注入，stat用于统计分析 -->
    <property name="filters" value="wall,stat"/>
</bean>
```

需要注意的是，按照官方说明，这里的配置是有问题的，后面会说到。具体配置请看文档：[DruidDataSource配置属性列表](https://github.com/alibaba/druid/wiki/DruidDataSource%E9%85%8D%E7%BD%AE%E5%B1%9E%E6%80%A7%E5%88%97%E8%A1%A8)

## 异常描述
### 异常1
SQL执行时间变长前后的日志有如下的异常栈：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902213813200.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
可以看到线程`Druid-ConnectionPool-Create-xxx`在执行`SocketInputStream#socketRead0`这个本地方法时超时，这里是在读取响应数据的时候超时的，也就是请求数据已经发送出去了，猜测是连接已经建立了，等待读取数据时候超时的？

### 异常2
后续又发现，应用服务启动时候会阻塞在某个地方导致启动失败，失败时候的线程堆栈dump如下（使用的是[perfma的在线分析工具](https://thread.console.perfma.com/)）：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902232551817.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)

通过结果可以看出启动线程同样卡在`SocketInputStream#socketRead0`方法里。

对于以上现象的分析：
上面两种情况都是卡在了读取数据时候，所以猜测是服务器返回的包在中间某个环节丢失了？（中间加了硬件防火墙和服务器的防火墙，所以肯定是这两个环节的问题）后来就将其中一个应用服务器节点上防火墙停掉了，结果情况并没有改善，所以排除了服务器防火墙的原因。而因为别的原因，硬件防火墙没办法查看分析，提议运维抓包又一直没抓，所以只能瞎猜原因。后面看到的现象是如果多启动几次应用服务会启动成功，所以猜测是某些端口的原因，这个问题没办法只能搁置了，不过总得解决，这个后续还要继续分析，等待更新。这个不是本文重点，重点是接下来的点。

### 异常3
在发现异常1的时候，开始时候分析是数据库连接池的连接已经中断，继续使用连接导致的（现在看来不是这个原因），而druid配置的检查连接是否有效的时间间隔太长甚至是配置可能未生效导致的，所以通过查看Druid的文档，发现我们的配置确实是有问题：

> validationQuery：用来检测连接是否有效的sql，要求是一个查询语句，常用select 'x'。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会起作用。

从上面看到，需要加上`validationQuery`这个参数。加上之后debug发现`Destroy线程会检测连接`依然没有检查连接是否有效，所以决定查看源码，`Destroy线程`线程的核心方法是`shrink(boolean checkTime, boolean keepAlive)`方法，该方法中与检查连接有效性相关的代码片段如下（部分代码）：
```java

int keepAliveCount = 0;
for (int i = 0; i < poolingCount; ++i) {
    DruidConnectionHolder connection = connections[i];
    if ((onFatalError || fatalErrorIncrement > 0) && (lastFatalErrorTimeMillis > connection.connectTimeMillis))  {
         keepAliveConnections[keepAliveCount++] = connection;
         continue;
    }
    long idleMillis = currentTimeMillis - connection.lastActiveTimeMillis;
    if (idleMillis < minEvictableIdleTimeMillis
           & idleMillis < keepAliveBetweenTimeMillis
    ) {
        break;
    }
    if (keepAlive && idleMillis >= keepAliveBetweenTimeMillis) {
        keepAliveConnections[keepAliveCount++] = connection;
    }
}
if (keepAliveCount > 0) {
    // keep order
    for (int i = keepAliveCount - 1; i >= 0; --i) {
        DruidConnectionHolder holer = keepAliveConnections[i];
        Connection connection = holer.getConnection();
        holer.incrementKeepAliveCheckCount();

        boolean validate = false;
        try {
            this.validateConnection(connection);
            validate = true;
        } catch (Throwable error) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("keepAliveErr", error);
            }
            // skip
        }

        boolean discard = !validate;
        if (validate) {
            holer.lastKeepTimeMillis = System.currentTimeMillis();
            boolean putOk = put(holer, 0L);
            if (!putOk) {
                discard = true;
            }
        }

        if (discard) {
            try {
                connection.close();
            } catch (Exception e) {
                // skip
            }

            lock.lock();
            try {
                discardCount++;

                if (activeCount + poolingCount <= minIdle) {
                    emptySignal();
                }
            } finally {
                lock.unlock();
            }
        }
    }
    this.getDataSourceStat().addKeepAliveCheckCount(keepAliveCount);
    Arrays.fill(keepAliveConnections, null);
}
```
很明显，需要配置合适的`minEvictableIdleTimeMillis`和`keepAliveBetweenTimeMillis`值，另外还需要配置`keepAlive = true`。通过上述配置，`Destroy`线程的确走到了检查连接有效性的代码部分，看起来问题似乎是解决的。

### 异常4
上面说到，检查连接有效性的配置生效了，我又连接本地数据库简单测试了一下，方法是在应用启动之后关闭MySQL数据库服务，测试结果却和预期不太一样。测试结果显示，数据库服务关闭之后，`Destroy`线程第一次检查完连接，连接依然有效。这是怎么回事？
点进去`this.validateConnection(connection);`方法查看一下，代码如下：
```java
    public void validateConnection(Connection conn) throws SQLException {
        String query = getValidationQuery();
        if (conn.isClosed()) {
            throw new SQLException("validateConnection: connection closed");
        }

        if (validConnectionChecker != null) {
            boolean result = true;
            Exception error = null;
            try {
                result = validConnectionChecker.isValidConnection(conn, validationQuery, validationQueryTimeout);

                if (result && onFatalError) {
                    lock.lock();
                    try {
                        if (onFatalError) {
                            onFatalError = false;
                        }
                    } finally {
                        lock.unlock();
                    }
                }
            } catch (SQLException ex) {
                throw ex;
            } catch (Exception ex) {
                error = ex;
            }

            if (!result) {
                SQLException sqlError = error != null ? //
                    new SQLException("validateConnection false", error) //
                    : new SQLException("validateConnection false");
                throw sqlError;
            }
            return;
        }

        if (null != query) {
            Statement stmt = null;
            ResultSet rs = null;
            try {
                stmt = conn.createStatement();
                if (getValidationQueryTimeout() > 0) {
                    stmt.setQueryTimeout(getValidationQueryTimeout());
                }
                rs = stmt.executeQuery(query);
                if (!rs.next()) {
                    throw new SQLException("validationQuery didn't return a row");
                }

                if (onFatalError) {
                    lock.lock();
                    try {
                        if (onFatalError) {
                            onFatalError = false;
                        }
                    }
                    finally {
                        lock.unlock();
                    }
                }
            } finally {
                JdbcUtils.close(rs);
                JdbcUtils.close(stmt);
            }
        }
    }
```
仔细看下代码，进去之后先获取参数里设置的`validationQuery`，似乎是按照文档描述那样在运行，但是继续往下看发现发现首先判断的却是`validConnectionChecker`，这个是什么东西？看名称是一个有效连接检查器，再看这个变量是在`DruidDataSource#initValidConnectionChecker`方法设置进去的，代码如下：
```java
private void initValidConnectionChecker() {
    if (this.validConnectionChecker != null) {
        return;
    }

    String realDriverClassName = driver.getClass().getName();
    if (JdbcUtils.isMySqlDriver(realDriverClassName)) {
        this.validConnectionChecker = new MySqlValidConnectionChecker();
    }
    // 其余省略
}
```
只要驱动是MySQL的驱动，这个变量就赋值为`MySqlValidConnectionChecker`对象，而这个方法是在`DruidDataSource#init`方法中调用的，也就是说这个变量几乎一定是指向`MySqlValidConnectionChecker`对象。这就是说无论你配置还是不配置`validationQuery`，如果不做特殊处理，几乎一定是走`validConnectionChecker#isValidConnection`方法来判断连接的有效性，也就是说在这儿，连接的检查没有使用`validationQuery`，配置不配置不影响（比较好奇别的地方会不会影响，为了不影响文章连贯性，这部分放在最后吧。）
那就到`validConnectionChecker`来看看。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906174956613.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
我这里使用的是8.0.23版本的MySQL驱动，该版本`ConnectionImpl`类的`pingInternal`方法是存在的（我看了下，5.1.x版本这个方法就存在了），所以这里将反射执行`pingInternal`方法来进行连接的检查。后面省略的代码部分是该方法不存在时候使用`validationQuery`来进行检查。经过debug发现，问题就出在反射执行`pingInternal`这里。该方法会在MySQL服务刚刚关闭时候出现空指针异常，后续走`pingInternal`方法每次都会抛出NPE（因为对MySQL驱动并不了解，暂时不清楚抛出NPE是否有意为之，但是从代码看应该不是，这个地方咱们后面再谈）。如果这儿出现NPE会导致什么问题呢？咱们来看下这儿反射执行的代码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906180706587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
这里会抛出`NPE`，再回到上层调用方法看下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906192836440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
可以看到这里对除`SQLException`外的异常没有直接抛出，本意是在下面判断`result`为`false`时，包装成`SQLException`再抛出，却因为`result`默认值是`true`，而`validConnectionChecker#isValidConnection`却因为异常的抛出而没有将`result`的默认值进行修改，导致这个尴尬的结果。
但是你如果继续测试执行sql，你会发现并没有出现意外的情况，这是为什么？
来看一下获取连接的方法：
```java
    @Override
    public DruidPooledConnection getConnection() throws SQLException {
        return getConnection(maxWait);
    }

    public DruidPooledConnection getConnection(long maxWaitMillis) throws SQLException {
        init();

        if (filters.size() > 0) {
            FilterChainImpl filterChain = new FilterChainImpl(this);
            return filterChain.dataSource_connect(this, maxWaitMillis);
        } else {
            return getConnectionDirect(maxWaitMillis);
        }
    }
```
简单看下`getConnection`方法，这里是责任链模式的实际应用。点进去会发现最终还是会进入`getConnectionDirect`方法，直接看这个方法吧：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906231918976.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
具体的配置我已经标出来了，`testOnBorrow`默认值是false，如果配置成true，和下面的代码走一样的校验。而`testWhileIdle`建议是配置为`true`，那如果配置为true，并且满足空闲时间大于`timeBetweenEvictionRunsMillis`，来看下对应的代码：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906233343510.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
同样抛出异常，这里直接返回来false，再到上层就丢弃这个连接了，所以这种情况下是不会有问题的。但是正如获取连接的方法注释的那样：
> ![if (idleMillis >= timeBetweenEvictionRunsMillis
                            || idleMillis < 0 // unexcepted branch
                            )](https://img-blog.csdnimg.cn/20200906233606623.png#pic_center)


因为`timeBetweenEvictionRunsMillis`同样是`Destroy`线程休眠的时间，理想情况下，`idleMillis`应该和`timeBetweenEvictionRunsMillis`差不多，但是如果连接长时间不用，这个值肯定还是略大于`idleMillis`，所以是会走到这个逻辑里的。如果刚好在这个时间间隙中获取连接呢？也就是说，`destroy`线程检查连接时候抛出了NPE，这个连接实际无效了，但是连接的`lastActiveTimeMillis`在上一次检查时更新了，这时候程序来拿连接了会怎么样？下面是测试方法，我们来试下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907004002593.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
发现连接拿到了，`Statement`语句对象也创建了，直到执行sql时候报`CommunicationsException`，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907004217727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
虽然执行失败了，也算是没有再报NPE了。此时查看`DruidDataSource`对象，会发现剩余的连接依然有效，`Destroy`线程依然在忙碌的进行无效的检查（无用功）。
以上的测试代码都在我的GitHub仓库里，地址如下：[https://github.com/wangzzleo/druid-bug-demo](https://github.com/wangzzleo/druid-bug-demo)
### 异常5
以上部分虽然看起来是`Druid`的校验出了问题，虽然`Druid`的有代码默认值的问题，但其实主要问题不在这儿，主要还是`MySQL-Connectot/J`以不期望的方式抛出了一个NPE，这个问题我提到了MySQL的bug系统里，链接如下：[https://bugs.mysql.com/bug.php?id=97824&thanks=3&notify=67](https://bugs.mysql.com/bug.php?id=97824&thanks=3&notify=67)，说实话，我其实不是很了解人家的代码，不知道是否有意为之，但是感觉NPE一般都不是有意为之的。这个bug并不是我提的，是另一个人提交的，我就是添加了一条评论告诉别人怎么复现。MySQL停机的问题应该不常见，但是在MySQL集群环境下有MySQL服务挂掉应该也是比较常模拟的故障，如果大家有这方面经验也可以告诉我下，战五渣的我其实这方面经验不是很多。后面我会给出我自己的理解，今天就先更新到这儿。另外，Druid是不是也可以提个`issue`什么的，有读者看到可以帮忙提交一下，或者我改天提下。

上面说到`MySQL-Connectot/J`以不期望的方式抛出了一个NPE，咱们就来看下这是怎么回事吧。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907164519552.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
方法调用栈如上所示，这里的重点就是`NativeProtocol#sendCommand`方法。该类接口上说明如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907164647904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)
这应该是一个核心方法，与数据库的交互很关键。NPE也是这儿抛出的，接下来看下什么情况下会抛出NPE。`com.mysql.cj.Session`方法上有这样的说明：
>Session exposes logical level which user API uses internally to call  Protocol methods. It's a higher-level abstraction than MySQL server session (ServerSession). Protocol and ServerSession methods should never be used directly from user API.
>Session暴露了逻辑层次，用户API在内部使用它来调用协议方法。它是比MySQL服务器会话（ServerSession）更高层次的抽象。Protocol和ServerSession方法不应该直接从用户API中使用。

所以`Protocol`里面的方法是不应该直接调用的，而是调用`Session`提供的方法。（但是不是一般都不直接调用`Session`，而是调用`Connection`提供的方法？）在这里看下这个方法都是哪些地方调用了，为什么看这个呢，这个后面会说到。（方法是在方法上面右键，再点击`Find Usages`）
这里可以看到这个方法只在`NativeProtocol`类内部和上面提到的`NativeSession#sendCommand`方法里面调用。而除了`NativeSession#sendCommand`方法可以设置`timeoutMillis`外，其余的地方`timeoutMillis`都是0。再接着看`NativeSession#sendCommand`方法的调用，会发现除了上面提到的`NativeSession#ping`方法外，其余所有调用的地方这个值设置的都是0。（无图言j）
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020090717203550.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzM0ODc1MTA2,size_16,color_FFFFFF,t_70#pic_center)

这个超时时间是读取数据时的超时时间，执行SQL的时间是没法预料的，所以猜测是这个原因，除了`ping`外，所有执行SQL地方的执行时间均为0。这个地方先记住。
现在终于到了关键的方法`NativeProtocol#sendCommand`了：
```java
    public final NativePacketPayload sendCommand(Message queryPacket, boolean skipCheck, int timeoutMillis) {
        int command = queryPacket.getByteBuffer()[0];
        this.commandCount++;

        if (this.queryInterceptors != null) {
            NativePacketPayload interceptedPacketPayload = (NativePacketPayload) invokeQueryInterceptorsPre(queryPacket, false);

            if (interceptedPacketPayload != null) {
                return interceptedPacketPayload;
            }
        }

        this.packetReader.resetMessageSequence();

        int oldTimeout = 0;

        if (timeoutMillis != 0) {
            try {
            	// 设置超时时间
                oldTimeout = this.socketConnection.getMysqlSocket().getSoTimeout();
                this.socketConnection.getMysqlSocket().setSoTimeout(timeoutMillis);
            } catch (IOException e) {
                throw ExceptionFactory.createCommunicationsException(this.propertySet, this.serverSession, this.getPacketSentTimeHolder(),
                        this.getPacketReceivedTimeHolder(), e, getExceptionInterceptor());
            }
        }

        try {

            checkForOutstandingStreamingData();

            // Clear serverStatus...this value is guarded by an external mutex, as you can only ever be processing one command at a time
            this.serverSession.setStatusFlags(0, true);
            this.hadWarnings = false;
            this.setWarningCount(0);

            //
            // Compressed input stream needs cleared at beginning of each command execution...
            //
            if (this.useCompression) {
                int bytesLeft = this.socketConnection.getMysqlInput().available();

                if (bytesLeft > 0) {
                    this.socketConnection.getMysqlInput().skip(bytesLeft);
                }
            }

            try {
                clearInputStream();
                this.packetSequence = -1;
                // 发送数据包
                send(queryPacket, queryPacket.getPosition());

            } catch (CJException ex) {
                // don't wrap CJExceptions
                throw ex;
            } catch (Exception ex) {
                throw ExceptionFactory.createCommunicationsException(this.propertySet, this.serverSession, this.getPacketSentTimeHolder(),
                        this.getPacketReceivedTimeHolder(), ex, getExceptionInterceptor());
            }

            NativePacketPayload returnPacket = null;

            if (!skipCheck) {
                if ((command == NativeConstants.COM_STMT_EXECUTE) || (command == NativeConstants.COM_STMT_RESET)) {
                    this.packetReader.resetMessageSequence();
                }
				// 检查回复包中是否有错误，如果没有，则返回回复包，准备读取。
                returnPacket = checkErrorMessage(command);

                if (this.queryInterceptors != null) {
                    returnPacket = (NativePacketPayload) invokeQueryInterceptorsPost(queryPacket, returnPacket, false);
                }
            }

            return returnPacket;
        } catch (IOException ioEx) {
            this.serverSession.preserveOldTransactionState();
            throw ExceptionFactory.createCommunicationsException(this.propertySet, this.serverSession, this.getPacketSentTimeHolder(),
                    this.getPacketReceivedTimeHolder(), ioEx, getExceptionInterceptor());
        } catch (CJException e) {
            this.serverSession.preserveOldTransactionState();
            throw e;

        } finally {
        	// 关键就在这个地方，不太清楚这个地方为什么要设置
            if (timeoutMillis != 0) {
                try {
                    this.socketConnection.getMysqlSocket().setSoTimeout(oldTimeout);
                } catch (IOException e) {
                    throw ExceptionFactory.createCommunicationsException(this.propertySet, this.serverSession, this.getPacketSentTimeHolder(),
                            this.getPacketReceivedTimeHolder(), e, getExceptionInterceptor());
                }
            }
        }
    }
```
debug发现（debug时候记得设置超时时间，不然不会走到这儿），抛出空指针异常的地方 在最后`finally`语句块的`this.socketConnection.getMysqlSocket().setSoTimeout(oldTimeout);`这条语句里，这个地方`getMysqlSocket()`获取`mySqlSocket`时候抛出了NPE。继续研究发现，正是在`returnPacket = checkErrorMessage(command);`检查回复包错误时候将`mySqlSocket`设置为`null`了。具体的过程不再赘述，读者可以直接进入`SimplePacketReader#readHeader`查看，正是在这里读取消息头时候抛出`IOException`，继而进入`catch`语句块强制关闭连接，将`mySqlSocket`设置为`null`。为什么`finally`代码块要这么写呢？去GitHub看下这段代码的提交记录，似乎最开始这个类还叫`MysqlaProtocol `时候，这个地方就这么写了。具体是什么原因我也不清楚，有看到的朋友可以给我解释下。而正是这里的空指针异常让`Druid`检查连接的地方错误将失效连接判断为有效，再次验证时候刚进入`sendCommnd`方法走到`// 设置超时时间
                oldTimeout = this.socketConnection.getMysqlSocket().getSoTimeout();`，就会抛出NPE，所以每次验证都无法正确判断。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200907173321752.png#pic_center)
有的朋友可能会问，这个方法每次与数据库交互都会使用，为什么从来没出错过，这就是为什么上面提到，除了`ConnectionIml#ping`方法外，别的地方超时时间都是0，不会走到这段逻辑里。

至此，异常分析都结束了，那为什么会出现这样的情况呢？

# 推测

### 验证
下面是我当时验证的过程：

`NativeProtocol#sendCommand`出错的情况：
设置超时时间>0，`send(queryPacket, queryPacket.getPosition())` 发包，但是没有报异常，因为`skipCheck`在这里固定是`false`，此时走到`checkErrorMessage`方法，将`socket`设置为`null`，再进入`finally`，`setSoTimeout`报空指针异常

步骤如下：
1. 关闭数据库，此时服务器`TCP`端口处于`FIN_WAIT_2`状态，可以接受数据但是不能返回响应；
2. 接下来，`send(queryPacket, queryPacket.getPosition())`发送数据，发送成功（可能是进入缓冲区，也可能是因为服务器此时还可以接受数据）
3. 此时进入`checkErrorMessage(command)`方法，该方法最终进入`SimplePacketReader#readHeader`方法，读取响应数据未读取到抛出异常，catch语句块捕获异常，并执行`mysqlSocket = null`。
4. 此时再回到`NativeProtocol#sendCommand`,进入`finally`语句块，该语句块不知为何重新设置了超时时间，`this.socketConnection.getMysqlSocket().setSoTimeout(oldTimeout);`，由3可知，此时`socket`为空，所以会抛出`NPE`。



这种情况出现的要点是：`send(queryPacket, queryPacket.getPosition());`函数不报错，调用`checkErrorMessage#checkErrorMessage`，内部调用`readMessage(this.reusablePacket);`，再调用`this.packetReader.readHeader();`，此时出现`IOException`或`CJPacketTooBigException`会关闭连接。



### 推测
那为什么在服务器关闭后，第一条访问的send不会报错？因为服务器处于FIN_WAIT_2状态还是可以接受数据?
checkErrorMessage 为什么异常？ 因为没有读到数据？
(TCP的知识我需要加强一下了)



【附1】：
来看看获取连接时候的代码，代码片段如下：
```java
	@Override
    public DruidPooledConnection getConnection() throws SQLException {
        return getConnection(maxWait);
    }

    public DruidPooledConnection getConnectionDirect(long maxWaitMillis) throws SQLException {
    int notFullTimeoutRetryCnt = 0;
    for (;;) {
        // handle notFullTimeoutRetry
        DruidPooledConnection poolableConnection;
        try {
            poolableConnection = getConnectionInternal(maxWaitMillis);
        } catch (GetConnectionTimeoutException ex) {
            if (notFullTimeoutRetryCnt <= this.notFullTimeoutRetryCount && !isFull()) {
                notFullTimeoutRetryCnt++;
                if (LOG.isWarnEnabled()) {
                    LOG.warn("get connection timeout retry : " + notFullTimeoutRetryCnt);
                }
                continue;
            }
            throw ex;
        }

        if (testOnBorrow) {
            boolean validate = testConnectionInternal(poolableConnection.holder, poolableConnection.conn);
            if (!validate) {
                if (LOG.isDebugEnabled()) {
                    LOG.debug("skip not validate connection.");
                }

                discardConnection(poolableConnection.holder);
                continue;
            }
        } else {
            if (poolableConnection.conn.isClosed()) {
                discardConnection(poolableConnection.holder); // 传入null，避免重复关闭
                continue;
            }
            if (testWhileIdle) {
                final DruidConnectionHolder holder = poolableConnection.holder;
                long currentTimeMillis             = System.currentTimeMillis();
                long lastActiveTimeMillis          = holder.lastActiveTimeMillis;
                long lastExecTimeMillis            = holder.lastExecTimeMillis;
                long lastKeepTimeMillis            = holder.lastKeepTimeMillis;
                if (checkExecuteTime
                        && lastExecTimeMillis != lastActiveTimeMillis) {
                    lastActiveTimeMillis = lastExecTimeMillis;
                }
                if (lastKeepTimeMillis > lastActiveTimeMillis) {
                    lastActiveTimeMillis = lastKeepTimeMillis;
                }
                long idleMillis                    = currentTimeMillis - lastActiveTimeMillis;
                long timeBetweenEvictionRunsMillis = this.timeBetweenEvictionRunsMillis;
                if (timeBetweenEvictionRunsMillis <= 0) {
                    timeBetweenEvictionRunsMillis = DEFAULT_TIME_BETWEEN_EVICTION_RUNS_MILLIS;
                }
                if (idleMillis >= timeBetweenEvictionRunsMillis
                        || idleMillis < 0 // unexcepted branch
                        ) {
                    boolean validate = testConnectionInternal(poolableConnection.holder, poolableConnection.conn);
                    if (!validate) {
                        if (LOG.isDebugEnabled()) {
                            LOG.debug("skip not validate connection.");
                        }

                        discardConnection(poolableConnection.holder);
                         continue;
                    }
                }
            }
        }
        // 省略后面代码
    }

    protected boolean testConnectionInternal(DruidConnectionHolder holder, Connection conn) {
        String sqlFile = JdbcSqlStat.getContextSqlFile();
        String sqlName = JdbcSqlStat.getContextSqlName();

        if (sqlFile != null) {
            JdbcSqlStat.setContextSqlFile(null);
        }
        if (sqlName != null) {
            JdbcSqlStat.setContextSqlName(null);
        }
        try {
            if (validConnectionChecker != null) {
                boolean valid = validConnectionChecker.isValidConnection(conn, validationQuery, validationQueryTimeout);
                long currentTimeMillis = System.currentTimeMillis();
               // ...
                return valid;
            }

            if (conn.isClosed()) {
                return false;
            }

            if (null == validationQuery) {
                return true;
            }

            Statement stmt = null;
            ResultSet rset = null;
            try {
                stmt = conn.createStatement();
                if (getValidationQueryTimeout() > 0) {
                    stmt.setQueryTimeout(validationQueryTimeout);
                }
                rset = stmt.executeQuery(validationQuery);
                if (!rset.next()) {
                    return false;
                }
            } finally {
                JdbcUtils.close(rset);
                JdbcUtils.close(stmt);
            }
			// ...
            return true;
        } catch (Throwable ex) {
            // skip
            return false;
        } finally {
            if (sqlFile != null) {
                JdbcSqlStat.setContextSqlFile(sqlFile);
            }
            if (sqlName != null) {
                JdbcSqlStat.setContextSqlName(sqlName);
            }
        }
    }
 ```
可以看到获取连接的逻辑和`Destroy`类似，`validConnectionChecker`不为空就和`validationQuery`没什么关系了。这个其实没啥，可能是文档没及时更新吧。
