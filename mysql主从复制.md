# MySQL主从复制 #

怎么安装mysql数据库，这里不详细讲，只说它的主从复制，步骤如下：

## 1、主从服务器分别作以下操作： ##
1.1、版本一致（这里使用mariadb作为数据库，mysql的配置和这个差别不大）

1.2、初始化表，并在后台启动mysql

1.3、修改root的密码（这里使用root作为密码，实际环境中不建议使用这么简单的密码）

主服务器IP：110.65.101.139

从服务器IP：110.65.101.117

## 2、修改主服务器master: ##

	#vim /etc/my.cnf

	[mysqld]
	log-bin=mysql-bin   //[必须]启用二进制日志
	server-id=10  //[必须]服务器唯一ID，默认是1，一般可取IP最后一段


## 3、修改从服务器slave: ##

	#vim /etc/my.cnf

	[mysqld]
	log-bin=mysql-bin   //[不是必须]启用二进制日志
	server-id=20      //[必须]服务器唯一ID，默认是1，一般可取IP最后一段


## 4、重启两台服务器的mysql ##

	/etc/init.d/mysqld restart
	or	/usr/local/mariadb/bin/mysql.server restart


## 5、在主服务器上建立帐户并授权slave: ##

	#/usr/local/mariadb/bin/mysql -uroot -proot
	MariaDB [(none)]>GRANT REPLICATION SLAVE ON *.* to 'mysync'@'%' identified by '123456';
	//一般不用root帐号，'%'表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.xxx.xxx，加强安全。



## 6、登录主服务器的mysql，查询master的状态 ##

	MariaDB [(none)]> show master status;
	+------------------+----------+--------------+------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
	+------------------+----------+--------------+------------------+
	| mysql-bin.000001 |      518 |              |                  |
	+------------------+----------+--------------+------------------+
	1 row in set (0.01 sec)

注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化



## 7、配置从服务器Slave： ##

	MariaDB [(none)]>change master to master_host='110.65.101.139',master_user='mysync',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=518; 
	//注意不要断开，518数字前后无单引号。

	MariaDB [(none)]>start slave;    //启动从服务器复制功能




## 8、检查从服务器复制功能状态： ##

	MariaDB [(none)]> show slave status\G
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 110.65.101.139
	                  Master_User: mysync
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: mysql-bin.000001
	          Read_Master_Log_Pos: 518
	               Relay_Log_File: localhost-relay-bin.000002
	                Relay_Log_Pos: 555
	        Relay_Master_Log_File: mysql-bin.000001
	             Slave_IO_Running: Yes
	            Slave_SQL_Running: Yes
	……


**注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。**

补充：如果出现 Slave_IO_Running: Connecting 往下查看下面错误提示

出现message: Can't connect to MySQL server on '110.65.101.139' (113 "No route to host")

很可能是防火墙阻挡

先关闭防火墙

	systemctl stop firewalld.service

再重复第7步操作


以上操作过程完成后，主从服务器到此配置完成。




## 9、主从服务器测试： ##

主服务器Mysql，建立数据库，并在这个库中建表插入一条数据：

	MariaDB [(none)]> create database ms_db;
	Query OK, 1 row affected (0.00 sec)
	
	MariaDB [(none)]> use ms_db;
	Database changed
	MariaDB [ms_db]> create table ms_tb(id int(4),name char(10));
	Query OK, 0 rows affected (0.06 sec)
	
	MariaDB [ms_db]> insert into ms_tb values(1,'test');
	Query OK, 1 row affected (0.00 sec)
	
	MariaDB [ms_db]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| ms_db              |
	| mysql              |
	| performance_schema |
	| test               |
	+--------------------+
	5 rows in set (0.13 sec)

	MariaDB [ms_db]> select * from ms_tb;
	+------+------+
	| id   | name |
	+------+------+
	|    1 | test |
	+------+------+
	1 row in set (0.02 sec)


从服务器mysql数据库查询：

	MariaDB [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| ms_db              |
	| mysql              |
	| performance_schema |
	| test               |
	+--------------------+
	5 rows in set (0.11 sec)
	
	MariaDB [(none)]> use ms_db;
	Database changed
	MariaDB [ms_db]> select * from ms_tb;
	+------+------+
	| id   | name |
	+------+------+
	|    1 | test |
	+------+------+
	1 row in set (0.01 sec)

可以看到数据已经更新过来并且可以查看



## 10、完成： ##

进阶操作：可以尝试编写一个shell脚本，用nagios监控slave的两个yes（Slave_IO及Slave_SQL进程），如发现只有一个或零个yes，就表明主从有问题了，发短信警报吧。

















