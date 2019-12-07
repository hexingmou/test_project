# mysql 主从复制

## 一、AB主从复制

### 1、主从复制含义

~~~powershell
实现将数据从一台数据库服务器复制到一台到多台数据库服务器
属于异步复制，所以无需维持长连接
~~~

### 2、主从复制搭建思路

~~~powershell
一、在两台计算机中安装相同版本的mysql
二、编写my.cnf配置文件（主master服务器上要开启二进制日志binlog。从slave服务器开启中继日志relaylog）
三、让master与slave进行数据同步，数据统一
四、配置主从
五、开启主从同步（打开同步开关=>start slave）
~~~

前期准备

~~~powershell
♠更改IP地址与UUID编号、主机名称、绑定hosts文件、关闭防火墙与SELinux、关闭NetworkManager，最后还要做一个时间同步。
①master&&slave  #hostnamectl set-hostname master&&slave.he.cn 更改主从服务器主机名
②vim /etc/hosts   在hosts文件中添加两者的IP+主机名    互信
10.1.1.22  master.he.cn
10.1.1.21  slave.he.cn
③关闭NetworkMannager
# systemctl stop NetworkManager
# systemctl disable NetworkManager
④关闭防火墙和selinux
systemctl stop firewalld
systemctl disable firewalld
vim /etc/selinux/conf------->将enforing-->disabled
⑤时间同步   
# ntpdate cn.ntp.org.cn

~~~

主从搭建前期配置

~~~powershell
一、安装mysql  在master上安装
①上传mysql软件到linux服务器上

②使用shell脚本自动安装mysql
# vim mysql.sh
#!/bin/bash
tar -zxf mysql-5.6.35-linux-glibc2.5-x86_64.tar.gz
mv mysql-5.6.35-linux-glibc2.5-x86_64 /usr/local/mysql
useradd -r -s /sbin/nologin mysql
chown -R mysql.mysql /usr/local/mysql
cd /usr/local/mysql
初始化数据库
yum remove mariadb-libs -y
scripts/mysql_install_db --user=mysql
cp support-files/mysql.server /etc/init.d/mysql
service mysql start
#sh mysql.sh
③添加/usr/local/mysql/bin目录到环境变量中
# vim /etc/profile
export PATH=$PATH:/usr/local/mysql/bin
# source /etc/profile
④设置mysql密码
# mysql_secure_installation
除了不禁止root账号的远程登录功能，其他选项都是Y

~~~

~~~powershell
在slave上安装mysql   在从服务器上安装mysql时千万不要初始化（scripts）；
①tar -zxf mysql-5.6.35-linux-glibc2.5-x86_64.tar.gz
②mv mysql-5.6.35-linux-glibc2.5-x86_64 /usr/local/mysql
③useradd -r -s /sbin/nologin mysql   创建mysql账号
④chown -R mysql.mysql /usr/local/mysql    分配权限
⑤yum remove mariadb-libs -y
  rm -rf data/*
⑥cp support-files/mysql.server /etc/init.d/mysql
⑦vim /etc/profile
 export PATH=$PATH:/usr/local/mysql/bin    在最后一行加入就行
⑧source /etc/profile


~~~

主从配置后续配置

~~~powershell
①在master服务器上将data下的所有文件同步到从slave服务器上的data
 #rsync -av /usr/local/mysql/data/* root@10.1.1.21:/usr/local/mysql/data  
②在slave服务器上
#rm -rf data/auto.cnf     这个文件存放的是每个mysql下的UUID
③配置my.cnf
 master服务器上
# vim /usr/local/mysql/my.cnf
[mysqld]
basedir = /usr/local/mysql         
datadir = /usr/local/mysql/data
port = 3306
server_id = 20						=>  必须唯一
socket = /tmp/mysql.sock
character_set_server=utf8mb4
log-bin = /usr/local/mysql/data/binlog		=> 主服务器必须开启二进制日志
log-error = /usr/local/mysql/data/mysql.err   错误日志

slave服务器上   没有进行初始化打开这个文件时里面没有东西（从服务器一定不要初始化）
# vim /usr/local/mysql/my.cnf
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
server_id = 30						=> 必须唯一
socket = /tmp/mysql.sock
character_set_server=utf8mb4
relay-log=/usr/local/mysql/data/relaylog	=> 从服务器必须开启中继日志
log-error = /usr/local/mysql/data/mysql.err

④master&&slave  # service mysql restart  主从服务器都从新启动mysql
⑤创建用于同步的账号 在主服务器master上  因为主从复制从服务器只能读。所以从服务器上不用创建同步账号
mysql> great replication slave on *.* to 'slave'@10.1.1.% identified by '123'
mysql> flush privileges;  刷新
mysql> flush tables with read lock;    刷新表并进行锁表   
mysql> show master status;   查看主服务器状态；找到二进制文件和位置binlog.000001，位置405
⑥在从slave服务器上 使用change master to 进行数据同步
mysql> change master to master_host='10.1.1.20',master_user='slave',master_password='123',master_port=3306,master_log_file='binlog.000001',master_log_pos=405;

master_host ：主机MASTER的IP地址
master_user ：同步的账号
master_password ：同步账号的密码
master_port ：主服务器的端口号
master_log_file ：主服务器正在生效的二进制文件名称
master_log_pos ：主服务器二进制文件的位置
⑦在从slave服务器上打开同步开关
mysql> start slave;         开启同步开关
mysql> show slave status\G    查看同步信息

~~~

![1571549423432](C:\Users\hexingmou\AppData\Roaming\Typora\typora-user-images\1571549423432.png)

~~~powershell 
同步成功后 在主master上 进行解锁（read lock）
mysql> unlock tables; 
~~~

## 二、5.6版本中的基于GTIDs(全局事务标识符)主从复制

### 1、GTIDs数据标识符

-xxxx-xxxx-xxxx-xxxx（唯一）  有一连串数字或字母组成  编号唯一

SQL语句.
xxx-xxxx-xxxx-xxxx-xxxx（唯一）

# xxx-xxxx-xxxx-xxxx-xxxx（唯一）
SQL语句...

2、GTIDs 是完全基于事务的。因此不支持myisam存储引擎
      当使用GTIDs时，每一个事务都可以被==识别并且跟踪== 
搭建GTIDs主从 复制  在AB主从复制基础上进行更改

### 3、从新配置master和slave的my.cnf 文件

~~~powershell
master my.cnf  主
# vim /usr/local/mysql/my.cnf
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
server_id = 22
socket = /tmp/mysql.sock
character_set_server=utf8mb4
log-bin = /usr/local/mysql/data/binlog
log-error = /usr/local/mysql/data/mysql.err
# 添加3行配置  其含义就是开启gtids
gtid-mode=on
log-slave-updates=1
enforce-gtid-consistency

slave  my.cnf  从
# vim /usr/local/mysql/my.cnf
[mysqld]
basedir = /usr/local/mysql
datadir = /usr/local/mysql/data
port = 3306
server_id = 21
socket = /tmp/mysql.sock
log-bin = /usr/local/mysql/data/binlog    
relay-log = /usrlocal/mysql/data/relaylog
log-error = /usr/local/mysql/data/mysql.err
添加4行配置      其含义就是开启gtids
gtid-mode = no
log-slave-updates = 1
enforce-gtid-consistency = 1
skip-slave-start  如果主服务还没有开启。那么从服务器就等着同步。主服务器开启了，就开始进行同步

~~~

4、master&&slave  #  service mysql restart 

### 5、创建一个同步账号

~~~powershell
mysql> grant replication slave on *.* to 'slave'@10.1.1.% identfied by '123';
mysql> flush privileges;
~~~

### 6、搭建主从结构

```POWERSHELL 
在slave中
mysql> change master to master_host='10.1.1.20',master_user='slave',master_password='123',master_port=3306,master_auto_position=1;
mysql> start slave;
mysql> show slave status\G
master_auto_position=1  将基于位点的复制关系升级为 GTIDs 模式
这是AB复制与GTIDS复制最主要的区别

mysql> start slave;         开启同步开关
mysql> show slave status\G    查看同步信息

```

![1571549423432](C:\Users\hexingmou\AppData\Roaming\Typora\typora-user-images\1571549423432.png)

