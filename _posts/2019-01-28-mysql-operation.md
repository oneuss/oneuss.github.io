---
layout: post
title:  "MySQL基本运维"
subtitle: "MySQL的安装等"
date:   2019-01-28
background: '/img/imac_bg.png'
---
# MySQL安装
**1. 查看linux版本：file /sbin/init 或者 file /bin/ls**

**2. 查看系统是否已经安装了mysql的其他版本：**
 ps：yum与rpm等改天系统学一下
```
[root@leo usr]#rpm -qa|grep mysql
mysql-libs-5.1.52-1.e16_0.1.x86_64
[root@leo usr]#yum -y remove mysql-libs*
....
```

**3. oracle官网下载到mysql发现压缩包里有一大波安装文件，如下：**
 MySQL-client         客户端组件
 MySQL-debuginfo      调试MySQL的组件
 MySQL-devel          想针对于MySQL编译安装PHP等依赖于MySQL的组件包
 MySQL-embedded       MySQL的嵌入式版本
 MySQL-server         服务端
 MySQL-shared         共享库
 MySQL-shared-dompat  为了兼容老版本的共享库
 MySQL-test           MySQL的测试组件（在线处理功能）

**4. 安装其中的 server，devel，client**
使用rpm：
```
rpm -ivh mysql-community-server-5.7.18-1.el6.x86_64.rpm
```
遇到问题，显示：

```
warning: mysql-community-server-5.7.18-1.el6.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
error: Failed dependencies:
	mysql-community-client(x86-64) >= 5.7.9 is needed by mysql-community-server-5.7.18-1.el6.x86_64
	mysql-community-common(x86-64) = 5.7.18-1.el6 is needed by mysql-community-server-5.7.18-1.el6.x86_64
```
发现是包的依赖关系问题，得先安装client和common，最后安装来安装去，发现得这样顺序安装（根据包依赖猜的）：

```
mysql-community-common-5.7.18-1.el6.x86_64.rpm
mysql-community-libs-5.7.18-1.el6.x86_64.rpm
mysql-community-client-5.7.18-1.el6.x86_64.rpm
mysql-community-server-5.7.18-1.el6.x86_64.rpm
```

**5. 安装成功；**

**6. 启动 mysql服务：**
  安装完成在 /etc/init.d/mysqld 下运行 `./mysqld start --user=root`
  （这里注意，我第一次运行时候未加 --user=root参数，运行时候报错，查看日志 /var/log/mysqld.log得知是没有对某个文件的操作权限，加上该参数就可以 了），其实这里有点儿含糊，改天布置集群后，在其他主机安装mysql时候再彻底研究一下。

**7. 连接MySQL**
这里安装的是MySQL5.7，该版本在安装时候会默认生成密码，查看默认密码方式：`grep 'temporary password' /var/log/mysqld.log`（MySQL安装已经是半年之前了现在才想起来重新学习，这里我在启动时候盯着日志看也没找到有默认密码的字样，所以猜测MySQL初始化是我在几个月前完成的）。注意第一次连接上数据库之后，需要手动修改密码 `alter user root@localhost identified by 'your_newPassword'`。

**8. 远程连接失败？**
我这儿使用的是虚拟机，MySQL安装完成后我在Windows主机上使用Navicat连接MySQL失败，遂用cmd `telnet ip 3306`  发现端口未开发，开放默认的3306端口的方法：
```
vi /etc/sysconfig/iptables
```

在 -A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT 后写上

```
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j
-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j
```
顺便把tomcat端口也给开放了，然后重启网卡：

```
service iptables restart
```
查看端口情况：

```
service iptables status
```
完成上面步骤之后，还是没办法访问，网上搜索得知虚拟机系统还得设置端口转发，仔细想想感觉这样是因为害怕是与主机的端口发生冲突了，但是ip也不一样啊，难道是说，只有在主机访问虚拟机时候才会这样，其他ip的主机并不会？想让其他人试试，但是想想又得设置内网穿透，闲言少叙，如何设置端口转发：
查了半天只有nat模式可以设置端口转发

**9. 那可咋整？**
   再查了下，发现根本不是端口转发的问题，而是MySQL里默认不允许远程远程登录，修改方式：
```
use mysql;
update user set host = '%' where user = 'root';
flush privileges;
```
再试试，成功。

**10. 在此遇到安装问题**
阿里云上安装了同样版本的MySQL，安装完成但是没办法启动。决定查看官方文档：http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html （my.cnf配置文件中找到该URL）。官方的rpm包安装启动方法是

```
sudo service mysqld start
```

但是启动之后发现如下错误：

```
mysqld: Can't change dir to '/var/lib/mysql/' (OS errno 13 - Permission denied)
```
随之将该路径用户组及所有者改为mysql，继续启动还是报错：

```
mysqld.service: main process exited, code=exited, status=1/FAILURE
```
查看日志有如下错误：

```
Unable to lock ./ibdata1 error: 11
```
又在下面找打这么一段话：
Could not open or create the system tablespace. If you tried to add new data files to the system tablespace, and it failed here, you should now edit innodb_data_file_path in my.cnf back to what it was, and remove the new ibdata files InnoDB created in this failed attempt. InnoDB only wrote those files full of zeros, but did not yet use them in any way. But be careful: do not remove old data files which contain your precious data!
应该是自己什么时候乱搞删了东西了，后来又重新装mysql。我重新装了一次，启动，成功。汗颜。

**11. 后续维护**
      运行一段时间，想再配置一些东西，但是忘记了配置文件位置怎么办？
      可以执行如下命令，查看启动时候是否制定了配置文件
```  
ps -ef|grep mysql|grep 'my.cnf'
```
如果还是没有结果，则可能是按照mysql默认的配置文件查找位置，具体位置可以参考文档或者是执行帮助查看：
```
mysql --help|grep 'my.cnf'
```

# 忘记root密码？

1. 在MySQL配置文件：/etc/my.cnf文件中，[mysqld]最后添加一行数据skip-grant-tables```vim /etc/my.cnf```,在[mysqld]最后：```skip-grant-tables```

2. 重启mysql服务```service mysqld restart```


3. 进入MySQL：不需要密码，直接进入mysql```mysql -uroot```

4. 刷新权限```flush privileges;```

5. 修改密码```alter user 'root'@'localhost' IDENTIFIED BY 'new_pwd';```

6. 还原配置文件：/etc/my.cnf，删除```skip-grant-tables```，然后重启mysql。
