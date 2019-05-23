---
layout: page
title: "milestone"
description: "代表作"
header-img: "img/zhihu.jpg"
---

[toc]
# 一. 理论概述
## 概述
本案例操作的是针对于MySQL的复制类型中的**全同步复制**,对几种复制类型简单总结下:
1. **异步复制**:MySQL默认的复制类型就是异步复制,主库在执行完客户端提交的事务后会立即将结果返回给客户端.可能出现的问题就是,数据在主库上写入成功,主库返回给客户端结果,主库在这时crash掉了,那很可能,该crash掉的主库数据还没有完全同步到从库,随后,从库提升为新的主库,客户端查询刚才的请求结果没有.
2. **全同步复制**:主库执行完一个事务,所有的从库都执行了该事务才返回给客户端,不足之处是返回客户端的响应速度会变慢
3. **半同步复制**:介于前两者之间主库执行完事务后不会立刻返回客户端,而是至少等待一个从库接收到,并且该从库写到其relay log 中才返回给客户端结果.降低了出错的肯能,同事也相对全同步复制来说减少了一些响应时间,但是也有其不足之处.
    -  问题:客户端事务在存储引擎提交后,在得到从库确认的过程中宕机了,可能有两种情况:
        - **事务还没有发送到从库上:** 客户端收到事务提交失败信息,再次重新提交到新的主库上,宕机修复好的主库重新启动之后,是以从库的身份加入到集群中,但是该事务之前该事务就有了,随后又会同步一次.有了两份
        - **事务已经发送到从库上:**此时从库已同步完成该该事务,但是由于主库宕机,客户端再次提交事务到新的主库.有了两份
4. 随之,基于之前版本基础上了loss-less半同步复制,MySQL5.7.2之后的版本支持,引入了一个新的参数进行控制rpl_semi_sync_master_wait_point.
- 我一直在想可能是修改一些配置就OK了,但是逻辑完全出问题了.由于我要配置的是全同步,网上各大网站的文档,上来就贴配置,思路有没有写,重点是**只要设置为相应的模式即是在使用相应的同步策略.**
- 而全同步同步策略的配置方式是基于组复制(GR)实现的**接下来就研究一下GR,对GR的实现在大脑里有点思路**
1. MySQL的组复制可以配置为单主模型和多主模型两种工作模式，它们都能保证MySQL的高可用。以下是两种工作模式的特性简介：
    1. 单主模型：从复制组中众多个MySQL节点中自动选举一个master节点，只有master节点可以写，其他节点自动设置为read only。当master节点故障时，会自动选举一个新的master节点，选举成功后，它将设置为可写，其他slave将指向这个新的master。
    2. 多主模型：复制组中的任何一个节点都可以写，因此没有master和slave的概念，只要突然故障的节点数量不太多，这个多主模型就能继续可用。

[参考文章](https://www.cnblogs.com/f-ck-need-u/p/9203154.html)




**本文主要针对于全同步复制**
# 二. 部署

**二进制包MySQL5.7.24版本**

主机名|IP地址|角色
--|--|--
master|192.168.111.3|主库
slave1|192.168.111.4|从库
slave2|192.168.111.5|从库
```
[root@localhost ~]# vim /etc/hosts

127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.111.3   master
192.168.111.4   slave1
192.168.111.5   slave2

[root@localhost ~]# scp /etc/hosts root@192.168.111.4:/etc/
root@192.168.111.4's password: 
hosts                                                                                                             100%  221   170.1KB/s   00:00    
[root@localhost ~]# scp /etc/hosts root@192.168.111.5:/etc/

[root@localhost ~]# hostname master
[root@localhost ~]# bash
[root@master ~]# uname -n
master

[root@localhost ~]# hostname slave1
[root@localhost ~]# bash
[root@slave1 ~]# uname -n
slave1

[root@localhost ~]# hostname slave2
[root@localhost ~]# bash
[root@slave2 ~]# uname -n
slave2

安装mysql二进制安装
[root@localhost ~]# yum -y install libaio
[root@localhost ~]# wget http://mirrors.sohu.com/mysql/MySQL-5.7/mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
[root@localhost ~]# tar zxf mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz 
[root@localhost ~]# mv mysql-5.7.24-linux-glibc2.12-x86_64 /usr/local/mysql

[root@localhost mysql]# /usr/local/mysql/bin/mysqld --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data --initialize
将随机生成的登录密码记录下来
2019-05-18T08:43:11.094845Z 1 [Note] A temporary password is generated for root@localhost: 2Gk75Zvp&!-y

```
- 配置
```
主库操作
新建配置文件
[root@localhost ~]# vim /etc/my.cnf 

[root@localhost ~]# uuidgen
5f21777b-9a91-457e-9f0a-33ceba505a24
#查看uuid
[root@localhost ~]# /etc/init.d/mysqld stop
#所有机器关掉服务
[root@localhost ~]# rm -rf /usr/local/mysql/data/*
#全部如此操作,删除数据目录下所有文件
[root@localhost ~]# vim /etc/my.cnf 

[mysqld]
server-id=1
#这些设置将server配置为使用唯一标识号，以启动全局事物标示符号
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
gtid_mode=ON
 #打开gtid保证半同步,因为组复制基于GTID，所以必须开启gtid_mode和enforce_gtid_consistency
enforce_gtid_consistency=ON
#强制持续打开
relay_log_info_repository=TABLE
master_info_repository=TABLE
#master的信息存放在系统表中，而不是文件中;组复制要将master和relay log的元数据写入到mysql.slave_master_info和mysql.slave_relay_log_info中
binlog_checksum=NONE
#它设置 server 打开二进制日志记录,使用基于行的格式并禁用二进制日志事件校验和;由于MySQL对复制事件校验的设计缺陷，组复制不能对他们校验，所以设置binlog_checksum=none
log-bin=mysql-binlog
#二进制日志文件功能开启
sync-binlog=1
#sync_binlog是为了保证每次事务提交都立刻将binlog刷盘，保证出现故障也不丢失日志。
binlog_format=ROW
#二进制文件格式
log-slave-updates=ON
#组中的每个节点都保留了完整的数据副本，它是share-nothing的模式。所以所有节点上都必须开启log_slave_updates，这样新节点随便选哪个作为donor都可以进行异步复制。
transaction_write_set_extraction=XXHASH64
#server必须为每个事务收集写集合，并使用 XXHASH64哈希算法将其编码为散列。
loose-group_replication_group_name="5f21777b-9a91-457e-9f0a-33ceba505a24"
#告知插件,正在加入或创建的组要命名为5f21777b-9a91-457e-9f0a-33ceba505a24
loose-group_replication_start_on_boot=off
#server 启动时不自动启动组复制。
loose-group_replication_local_address="192.168.111.3:24901"
#插件使用的ip地址，端口用于接受来自组中其他成员的传入连接
loosegroup_replication_group_seeds="192.168.111.3:24901,192.168.111.4:24902,192.168.111.5:24903" 
#种子成员,集群内所有成员的IP及端口,注意端口一样的话报错,不知为啥
loose-group_replication_bootstrap_group=off
#插件是否自动引导，注意此选项在任何时候只能在一个 server 实例上使用,通常是首次引导组时(或在整个组被崩溃然后恢复的情况下)。 如果您多次引导组,例如,当多个 server 实例设置了此选项,则它们可能会人为地造成脑裂的情况,其中存在两个具有相同名称的不同组。 在第一个server 实例加入组后禁用此选项。组中的所有 server 成员的配置都非常相似。(此配置项如报错,就删除)
loose-group_replication_member_weigth = 40
#设置该节点的权重,MySQL5.7.20以后版本才支持,否则就不配置,会根据uuid来选举master



[root@localhost ~]# /usr/local/mysql/support-files/mysql.server start
Starting MySQL.Logging to '/usr/local/mysql/data/mysql.log'.
 SUCCESS! 
[root@localhost ~]# netstat -lnpt | grep 3306
tcp6       0      0 :::3306                 :::*                    LISTEN      64636/mysqld 


[root@localhost mysql]# cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
#添加到系统服务

[root@localhost mysql]# ln -s /usr/local/mysql/bin/* /usr/local/bin/
#便于使用

[root@master ~]# mysqladmin -u root -p'2Gk75Zvp&!-y' password '123456'
mysqladmin: [Warning] Using a password on the command line interface can be insecure.
Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.
[root@master ~]# mysql -uroot -p123456
#更改密码,连接数据库

mysql> grant replication slave on *.* to rep@'192.168.111.%' identified by'123456';
Query OK, 0 rows affected, 1 warning (0.30 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
#授权用户
```
- 配置节点加组时的通道。这是组复制的一个关键。
```
在新节点加入组时，首先要选择donor。新节点和donor之间的异步复制就是通过一个名为group_replication_recovery的通道(通道名固定，不可使用自定义通道)进行数据恢复的，经过数据恢复后，新节点填充了它缺失的那部分数据，这样就和组内其他节点的数据保持了同步。
mysql> change master to master_user='rep',master_password='123456' for channel 'group_replication_recovery';
#也可以称之为专门的术语：通道凭据(channel credentials)
#当执行完上面的语句后，就生成了一个该通道的relay log文件(注意称呼：该通道的relay log，后面还有另一个通道的relaylog)。如下，其中前缀"relay-log"是配置文件中"relay_log"选项配置的值。
#group_replication_recovery通道的relay log用于新节点加入组时，当新节点联系上donor后，会从donor处以异步复制的方式将其binlog复制到这个通道的relaylog中，新节点将从这个recovery通道的relay log中恢复数据。
```
- 前面配置文件中已经指定了master和relay log的元数据信息要记录到表中，所以这里可以先查看下关于relay log的元数据。
```
mysql> select * from mysql.slave_relay_log_info\G;
*************************** 1. row ***************************
  Number_of_lines: 7
   Relay_log_name: ./master-relay-bin-group_replication_recovery.000001
    Relay_log_pos: 4
  Master_log_name: 
   Master_log_pos: 0
        Sql_delay: 0
Number_of_workers: 0
               Id: 1
     Channel_name: group_replication_recovery
1 row in set (0.01 sec)
```
- 开启mysql实例的组复制功能
```
mysql> install plugin group_replication soname 'group_replication.so';
Query OK, 0 rows affected (0.03 sec)
#安装该插件
mysql> set @@global.group_replication_bootstrap_group=on;
Query OK, 0 rows affected (0.00 sec)
#这表示稍后启动的组复制功能将引导组，也就是创建组并配置组，这些都是自动的。配置引导变量为ON后，再开启组复制插件功能，也就是启动组复制。最后将引导变量设回OFF，之所以要设置回OFF，是为了避免下次重启组复制插件功能时再次引导创建一个组，这样会存在两个名称相同实际却不相同的组。
mysql> start group_replication;
Query OK, 0 rows affected (2.35 sec)
#启动组复制
mysql> set @@global.group_replication_bootstrap_group=off;
Query OK, 0 rows affected (0.00 sec)
```
**这几个过程不适合放进配置文件中，强烈建议手动执行它们的。否则下次重启mysql实例时，会自动重新引导创建一个组。同理，除了第一个节点，其他节点启动组复制功能时，不应该引导组，所以只需执行其中的start语句，==千万不能开启group_replication_bootstrap_group变量引导组==。**

这里的几个过程，应该形成一个习惯，在启动第一个节点时，这3条语句同时执行，在启动其他节点时，只执行start语句。

至此，这个节点的组复制已经配置完成了。现在需要查看这个节点是否成功加入到组中，成功加入组的标志是被设置为"ONLINE"。只要没有设置为ONLINE，就表示组中的这个节点是故障的。
```
mysql> select * from performance_schema.replication_group_members\G;
*************************** 1. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: f7a5022e-7948-11e9-a40e-000c2935c4a6
 MEMBER_HOST: master
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
1 row in set (0.00 sec)
#member_host，它是对外连接的地址
```
**稍后加入其余两个节点,加入之前顺便测试一下开启组复制后，必须使用InnoDB、表中必须有主键的限制。**

下面创建4个表：t1和t4是InnoDB表，t3和t4具有主键。
```
mysql> create database test;
mysql> use test;
Database changed
mysql> create table t1(id int);
mysql> create table t2(id int)engine=myisam;
mysql> create table t3(id int primary key)engine=myisam;
mysql> create table t4(id int primary key);
mysql> insert into t1 values(1);
ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.
mysql> insert into t2 values(1);
ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.
mysql> insert into t3 values(1);
ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.
mysql> insert into t4 values(1);
Query OK, 1 row affected (0.00 sec)

#不支持非innodb存储引擎,和没有主键
```
### 向组加入新节点
- 生产环境有许多需要考虑的因素:对于一台已经运行许久的数据库,该数据库已经有大量数据文件,依靠组复制,复制大量数据,速度会非常慢,可以先通过备份恢复的方式,恢复到本地,在依靠组复制进行

**slave2:**
```
[root@slave1 ~]# vim /etc/my.cnf 

[mysqld]
server-id=2 #修改
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log-bin=mysql-binlog
log-slave-updates=ON
binlog_format=ROW
sync-binlog=1
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5f21777b-9a91-457e-9f0a-33ceba505a24"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address="192.168.111.4:24902" #修改为本机
loose-group_replication_group_seeds="192.168.111.3:24901,192.168.111.4:24902,192.168.111.5:24903"
loose-group_replication_member_weigth = 20 #修改
[mysqld_safe]
log-error=/usr/local/mysql/data/mysql.log
pid-file=/usr/local/mysql/data/mysql.pid

mysql> change master to master_user='rep',master_password='123456' for channel 'group_replication_recovery';

#到master上查看新加入的节点
mysql> select * from mysql.slave_master_info\G;
*************************** 1. row ***************************
       Number_of_lines: 25
       Master_log_name: 
        Master_log_pos: 4
                  Host: <NULL>
             User_name: 
         User_password: 
                  Port: 0
         Connect_retry: 60
           Enabled_ssl: 0
                Ssl_ca: 
            Ssl_capath: 
              Ssl_cert: 
            Ssl_cipher: 
               Ssl_key: 
Ssl_verify_server_cert: 0
             Heartbeat: 30
                  Bind: 
    Ignored_server_ids: 0
                  Uuid: 
           Retry_count: 86400
               Ssl_crl: 
           Ssl_crlpath: 
 Enabled_auto_position: 1
          Channel_name: group_replication_applier
           Tls_version: 
*************************** 2. row ***************************
       Number_of_lines: 25
       Master_log_name: 
        Master_log_pos: 4
                  Host: 
             User_name: rep
         User_password: 123456
                  Port: 3306
         Connect_retry: 60
           Enabled_ssl: 0
                Ssl_ca: 
            Ssl_capath: 
              Ssl_cert: 
            Ssl_cipher: 
               Ssl_key: 
Ssl_verify_server_cert: 0
             Heartbeat: 0
                  Bind: 
    Ignored_server_ids: 0
                  Uuid: 
           Retry_count: 86400
               Ssl_crl: 
           Ssl_crlpath: 
 Enabled_auto_position: 0
          Channel_name: group_replication_recovery
           Tls_version: 
mysql> install plugin group_replication soname 'group_replication.so';
mysql> start group_replication;

```
**slave3:**
```
[root@slave2 ~]# vim /etc/my.cnf 

[mysqld]
server-id=3
gtid_mode=ON
enforce_gtid_consistency=ON
master_info_repository=TABLE
relay_log_info_repository=TABLE
binlog_checksum=NONE
log-bin=mysql-binlog
log-slave-updates=ON
binlog_format=ROW
sync-binlog=1
datadir=/usr/local/mysql/data
socket=/tmp/mysql.sock
transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="5f21777b-9a91-457e-9f0a-33ceba505a24"
loose-group_replication_start_on_boot=off
loose-group_replication_local_address="192.168.111.4:24903"
loose-group_replication_group_seeds="192.168.111.3:24901,192.168.111.4:24902,192.168.111.5:24903"
loose-group_replication_member_weigth = 10
[mysqld_safe]
log-error=/usr/local/mysql/data/mysql.log
pid-file=/usr/local/mysql/data/mysql.pid

mysql> change master to master_user='rep',master_password='123456' for channel 'group_replication_recovery';

#到master上查看新加入的节点


mysql> change master to master_user='rep',master_password='123456' for channel 'group_replication_recovery';
mysql> select * from mysql.slave_master_info\G;
mysql> install plugin group_replication soname 'group_replication.so';
mysql> start group_replication;


mysql> select * from performance_schema.replication_group_members\G
*************************** 1. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 72d428ee-7980-11e9-b073-000c2935c4a6
 MEMBER_HOST: master
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 2. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 73d663df-62c0-11e9-9efa-000c2919fb90
 MEMBER_HOST: slave2
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
*************************** 3. row ***************************
CHANNEL_NAME: group_replication_applier
   MEMBER_ID: 7895461d-6984-11e9-bd58-000c294b0234
 MEMBER_HOST: slave1
 MEMBER_PORT: 3306
MEMBER_STATE: ONLINE
3 rows in set (0.00 sec)

```
- slave1和slave2查看数据是否同步
```
mysql> show tables from test;
+----------------+
| Tables_in_test |
+----------------+
| t1             |
| t2             |
| t3             |
| t4             |
+----------------+
4 rows in set (0.00 sec)
```
## 测试
- 查找主节点
```
mysql> select b.member_host the_master,a.variable_value master_uuid
    ->     from performance_schema.global_status a
    ->     join performance_schema.replication_group_members b
    ->     on a.variable_value = b.member_id
    ->     where variable_name='group_replication_primary_member';
+------------+--------------------------------------+
| the_master | master_uuid                          |
+------------+--------------------------------------+
| master     | 72d428ee-7980-11e9-b073-000c2935c4a6 |

```
- 节点离组和选举新节点(最少3个节点的环境)


当前是master是主,我将网卡关掉模拟故障
```
[root@master ~]# ifdown ens32

slave1:上查看日志

2019-04-28T07:53:59.922705Z 0 [Warning] Plugin group_replication reported: 'Member with address master:3306 has become unreachable.'
2019-04-28T07:54:00.179220Z 0 [Note] Plugin group_replication reported: '[GCS] Removing members that have failed while processing new view.'
2019-04-28T07:54:00.927598Z 0 [Warning] Plugin group_replication reported: 'Members removed from the group: master:3306'
2019-04-28T07:54:00.927632Z 0 [Note] Plugin group_replication reported: 'Primary server with address master:3306 left the group. Electing new Primary.'2019-04-28T07:54:00.927750Z 0 [Note] Plugin group_replication reported: 'A new primary with address slave2:3306 was elected, enabling conflict detection until the new primary applies all relay logs.'2019-04-28T07:54:00.927928Z 28 [Note] Plugin group_replication reported: 'This server is working as secondary member with primary member address slave2:3306.'2019-04-28T07:54:00.928013Z 0 [Note] Plugin group_replication reported: 'Group membership changed to slave2:3306, slave1:3306 on view 15581931615697484:7.'

mysql> select b.member_host the_master,a.variable_value master_uuid from performance_schema.global_status a join performance_schema.replication_group on a.variable_value = b.member_id   
+------------+--------------------------------------+
| the_master | master_uuid                          |
+------------+--------------------------------------+
| slave2     | 73d663df-62c0-11e9-9efa-000c2919fb90 |
+------------+--------------------------------------+
1 row in set (0.00 sec)

```
可以看到slave2已经切换为主了

# 三.总结
1. 本来就想着做一个MySQL的全同步复制类型,毕竟对于业务数据安全要求极其苛刻情况下,这点延时是可以接受的
2. 结果就做了mysql组复制,非常感谢[这篇文档](https://www.cnblogs.com/f-ck-need-u/p/9203154.html)受益匪浅.
3. 最后测试,个人感觉也是没有做的很到位,首先,这点时间延迟,在测试环境中不太容易测出.
4. 个人对于整个**组复制**高可用集群还是理解不够透彻,据说是mysql官方极力推荐的,如果以后用到,一定下点成本.






