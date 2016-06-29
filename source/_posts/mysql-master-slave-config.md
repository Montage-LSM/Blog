---
title: Windows下多个Mysql实例配置主从
date: 2015-01-08
tags: [Mysql,win]
other: true
---



序：
    网上有很多类似的文章，也是各种百度出来的，但是对于多数刚开始接触MYSQL主从的小白来说，网上文章的代码里面很多技术点都没有理解，有跌打误撞碰上的，但多数都是这篇文章卡主了，换篇文章接着卡。- -。
    下面真正开始写教程之前，我希望你能够先完整的看完，再去敲代码。
    方法适用于MYSQL 5.1之后的版本。之前的版本，自行百度。
Mysql的主从是个什么德行我就不解释了。不然你也不会搜不到这篇文章。
<!--more-->

环境：
w7 64位。
    mysql 5.5.24...（也就是多数大家装的wamp包里面的版本）
其实应该是要在 linux里面去做这件事的，但是仅仅是为了了解，学习这个主从，大多数人还是windows下的平台，So...不解释。
首先你要在你的windows下再装一个mysql实例（不要妄想着一个Mysql实例，里面弄两个库然后他们配置主从，这个我可没玩过，有兴趣的同学可以尝试一下），意味着你要分配不同的端口。
windows下安装多个mysql的过程看下面这篇文章就好了。
http://blog.csdn.net/zuxianghaung/article/details/7272557

这是用到的软件包
http://yunpan.cn/cySt9WkiiTDPa 提取码 42e8
（看我多么良心，连软件都给你准备好了，不用你去各大垃圾下载站去下载了。再次注意你的环境，保证跟我的一样，以及数据库版本）


OK。我就当你已经配置好了第二个mysql实例。
下面两个bat 文件代码，用来帮你快速启动关闭你的新服务
startmysql.bat

@ECHO OFF
net start mysql5.5
pause

stopmysql.bat

@ECHO OFF
net stop mysql5.5

pause

这个mysql5.5 是你第二个实例的服务名称，stop停止服务,start 开启服务，不解释了。
别忘了进去你第二个Mysql实例瞅瞅。


进入正餐：
因为我们是在一个windows下配置的，所以没有网上那些主从 IP。 都是localhost
主数据库  my.ini添加如下

在[mysqld]下添加配置数据：
server-id=1     #配一个唯一的ID编号，1至32。 手动设定
log-bin=mysql-bin  #二进制文件存放路径 ，不要在意为啥没有路径名，你就这样写

#设置要进行或不要进行主从复制的数据库名，同时也要在Slave（也就是你的从库） 上设定。
binlog-do-db=进行主从数据库名1 ,进行主从数据库名2
binlog-ignore-db=不参与主从的数据库名,不参与主从的数据库名2
保存，重启数据库服务。

 上面的这些配置的含义:
    - server-id 顾名思义就是服务器标识id号了
    - log-bin 指定日志类型
    - binlog-do-db 是你需要复制的数据库名称，如果有多个就用逗号“,”分开
    - binlog-ignore-db 是不需要复制的数据库名称，如果有多个就用逗号“,”分开
在主库中建立一个用户（专门用给从库连接的，注意这是在主库里面建立的，可别迷迷糊糊的到从库的命令界面敲）：

1.mysql>grant replication slave,reload,super on *.* to lisimin@localhost identified by 'root' ; 

2.mysql>flush privileges; 

3.mysql>show master status; # 找到File 和 Position 的值记录下来；

![image](http://o9isa6w17.bkt.clouddn.com/20150108101611819.png)



简单解释一下第一句。
创建了一个用来复制的账号。
“@”前面的“lisimin”是用户名，后面的是有效的域，这里因为是本地，所以是写的是Localhost,如果是其他地址，对应填写上IP即可，不过应该不需要考虑端口问题，我创建的时候就没写端口。by 后面的“root”是密码。账户密码自己定义，不要跟root，以及你当前的用户名冲突。。

2.flush....刷新权限。
3.这个就是你的日志值
下面这篇文章是介绍创建用户对应分配权限问题的，简单了解一下就行。

http://blog.chinaunix.net/uid-20639775-id-3249105.html

从库配置:
从数据库配置my.ini:
[mysqld]
server-id=2     #唯一
#设置要进行或不要进行主从复制的数据库名，同时也要在Master 上设定。
replicate-do-db=进行主从数据库名1 ,数据库名2
replicate-ignore-db=不进行数据库名1 ,数据库名2

多个数据库之间用 , 分割。其实也可以这样写
replicate-do-db=进行主从数据库名1 
replicate-do-db=进行主从数据库名2
上面的那个写法也是。

其实你只需要写进行主从的库名称就可以了。
对了。假如你的主库叫 A 。那你的从库 最好也叫A。叫别的也可以，不过一定要是存在的。

下面登陆你的从库：
mysql>change master to master_host='127.0.0.1',master_user='slave',master_password='slave', master_log_file='mysql-bin.000001',master_log_pos=107;

master_host= 这里填你主库的IP。
master_user='lisimin'  刚才我们创建的那个用户。
master_user='root'  ..不解释。
这就是我们刚才 在主库里面 show  master status；得到的值了。自行根据实际情况填写
master_log_file='mysql-bin.000015' 
master_log_pos=107

如果你的主库还有是其他端口的话，
master_port=端口号   
master里面常用的就这些参数了，其余的自行百度。
常见的有,让你先 stop slave 。。
那你就先 mysql>stop slave 。再执行上面的代码。其他的错误，容易出现在语法，标点符号上，
然后 mysql>start slave ;
mysql>show slave status\G;
如果出现：
Slave_IO_Runing:Yes
Slave_SQL_Running :yes
那就说明成功了，如果出现错误，一般都是连接出问题。重新检查一下你是否正确的输入了刚才创建的用户名和密码。好了基本上就是这些问题了。

尊重原创，一些资料，也是从下面两篇文章中参照的、
http://blog.csdn.net/zhangking/article/details/5662545
http://blog.sina.com.cn/s/blog_6954c2c401017vvp.html


最后说点注意事项，从库要先创建好，以及里面的表结构，反正我是先创建好表结构的，如果你说以后涉及到添加，删除字段问题，那就是以后的事了。

还有，如果你真正部署到服务器的话，一般是linux一定要写好了定时删除 日志文件的脚本文件，这个估计是以后的事了。不然，日志文件可是非常大的。定期做个备份啥的。

OK，这样你就可以在你的主数据库里添加一条记录试试，看看你的从库有木有记录。（别忘开从库的服务啊、、）
以及，可以把主库里面的表设计为 innodb。从库设计为myisam。。来提高性能。

不啰嗦了。
