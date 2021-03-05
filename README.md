## Set up Master Slave Replication in MySQL / MariaDB —


### Master, 192.168.43.135 —

**DNS setup -** 

[root@master ~]# nano /etc/hostname
```
master
```
[root@master ~]# nano /etc/hosts

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.43.135    master
192.168.43.168    slave
```
[root@master ~]# ping slave

[root@master ~]# systemctl stop firewalld

[root@master ~]# setenforce 0


### Prepare master with MariaDB —

[root@master ~]# sudo dnf install mariadb-server

**Note :** Default Port 3306

[root@master ~]# sudo systemctl start mariadb

[root@master ~]# sudo systemctl status mariadb

[root@master ~]# sudo systemctl enable mariadb

**Securing Mariadb -**

[root@master ~]# sudo mysql_secure_installation


### Making mysql password file —

[root@master ~]# mysql

```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

[root@master ~]# nano .my.cnf

```
[client]
user=root
password=@20Password
```
[root@master ~]# mysql

MariaDB [(none)]> show schemas;

MariaDB [(none)]> show databases;


### Master configuration —

[root@master ~]# nano /etc/my.cnf

```
[mysqld]
server-id=11
log-bin=mysql-bin
```
[root@master ~]# systemctl restart mysqld

[root@master ~]# mysql

MariaDB [(none)]> show schemas;

MariaDB [(none)]> grant replication slave on *.* to 'root'@'%' identified by '@20Password';

MariaDB [(none)]> flush tables with read lock;

MariaDB [(none)]> unlock tables; (Incase needs to unlock)

MariaDB [(none)]> show master status \G;

```
*************************** 1. row ***************************
            File: mysql-bin.000001
        Position: 521
    Binlog_Do_DB:
Binlog_Ignore_DB:
1 row in set (0.002 sec)
```

### Slave, 192.168.43.168 —

**DNS setup -** 

[root@slave ~]# nano /etc/hostname

```
slave
```
[root@slave ~]# nano /etc/hosts

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.43.168    slave
192.168.43.135    master
```

[root@slave ~]# ping master

[root@slave ~]# systemctl stop firewalld

[root@slave ~]# setenforce 0


### Prepare slave with MariaDB —

[root@slave ~]# sudo dnf install mariadb-server

**Note :** Default Port 3306

[root@slave ~]# sudo systemctl start mariadb

[root@slave ~]# sudo systemctl status mariadb

[root@slave ~]# sudo systemctl enable mariadb

**Securing Mariadb -**

[root@slave ~]# sudo mysql_secure_installation


### Making mysql password file —

[root@slave ~]# mysql
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
```

[root@slave ~]# nano .my.cnf
```
[client]
user=root
password=@20Password
```
[root@slave ~]# mysql

MariaDB [(none)]> show schemas;

MariaDB [(none)]> show databases;


### Slave configuration —

[root@slave ~]# nano /etc/my.cnf

```
[mysqld]
server-id=22
```
[root@slave ~]# sudo systemctl restart mariadb

[root@slave ~]# mysql

```
MariaDB [(none)]> change master to
    -> master_host='master',
    -> master_user='root',
    -> master_password='@20Password',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=521;
Query OK, 0 rows affected (0.009 sec)
```
MariaDB [(none)]> start slave;

MariaDB [(none)]> show processlist;

MariaDB [(none)]> show slave status \G;


**Incase need to reset the slave -**

MariaDB [(none)]> stop slave;

MariaDB [(none)]> reset slave;

MariaDB [(none)]> start slave IO_THREAD;

MariaDB [(none)]> stop slave IO_THREAD;

MariaDB [(none)]> start slave;


### Switch to master — 

**Dump the database and scp to slave -** 

[root@master ~]# mysqldump mysql > mysql.sql

[root@master ~]# scp mysql.sql slave:/root


### Switch to slave —

**Deport the dump taken from master -**

[root@slave ~]# ls -ltr

[root@slave ~]# mysql mysql < mysql.sql


### Switch to master server —

**Install net tools and test the master slave connectivity**

[root@master ~]# yum install net-tools

[root@master ~]# netstat -natp | egrep -i established.*mysql
```
tcp6       0      0 192.168.43.135:3306     192.168.43.168:56632    ESTABLISHED 32449/mysqld
```

[root@master ~]# mysql -e "SHOW PROCESSLIST"

```
| 10 | root        | slave:56632 | NULL | Binlog Dump | 1880 | Master has sent all binlog to slave; waiting for binlog to be updated | NULL       
```

### Create a database to see —

[root@master ~]# mysql

MariaDB [(none)]> show databases;

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.005 sec)
```

MariaDB [(none)]> unlock table;

### Find on slave —

[root@slave ~]# mysql

MariaDB [(none)]> show databases;

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.008 sec)
```

### Create database on master —

MariaDB [(none)]> create database replication;

MariaDB [(none)]> show databases;

```

+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| replication        |
+--------------------+
4 rows in set (0.001 sec)
```

### Check on slave —

MariaDB [(none)]> show databases;

```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| replication        |
+--------------------+
4 rows in set (0.001 sec)
```
.
  
**@ By — Anup Kumar Mondal**
